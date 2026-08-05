# 05 — Code Indexer

Semantic + exact search over the repo, so agents stop reading whole files to find one function.

## Why this exists

Without an index, an agent answering "where is session creation handled?" either greps blindly or reads directories into context. Both are expensive; the second is *very* expensive. A good index turns that into a 200-token answer.

**Two complementary halves, deliberately:**

| Half | Answers | Backed by |
|---|---|---|
| **Semantic** | "where do we handle auth token refresh?" | tree-sitter chunks → embeddings → Qdrant |
| **Exact** | "who calls `createSession`?" / "where is `UserRepo` defined?" | tree-sitter symbol extraction → Postgres |

Semantic search alone is bad at exact identifiers. Symbol lookup alone is bad at intent. Ship both; they're cheap together.

---

## Pipeline

```
git repo
   │  (1) discover
   ▼
file list  ── .gitignore, .asdlcignore, size cap, binary/vendor/lockfile exclusion
   │  (2) change detect
   ▼
changed files only  ── git blob sha vs. stored sha  (full scan only on first index)
   │  (3) parse
   ▼
tree-sitter AST  ── 40+ grammars, one process, tolerant of broken syntax
   │  (4) chunk
   ▼
AST chunks  ── split at function/class/method boundaries, never mid-symbol
   │  (5) enrich
   ▼
chunk + header  ── file path, language, enclosing symbol path, imports, docstring
   │  (6) embed            (7) extract symbols
   ▼                            ▼
Qdrant `code`            Postgres symbols + symbol_refs
```

### (1) Discovery

```yaml
index:
  include: ["**/*"]
  exclude:
    - "**/node_modules/**"    - "**/.venv/**"      - "**/vendor/**"
    - "**/dist/**"            - "**/build/**"      - "**/target/**"
    - "**/*.min.js"           - "**/*.lock"        - "**/*.snap"
    - "**/__snapshots__/**"   - "**/*.generated.*"
  max_file_bytes: 1048576     # 1 MB
  respect_gitignore: true
  extra_ignore_file: .asdlcignore
```

Excluding generated code and lockfiles matters more than it sounds — they're huge, semantically empty, and they crowd out real results.

### (2) Incremental change detection

```
for each file: git blob sha (from `git ls-files -s`)
  ├─ sha unchanged  → skip entirely (no parse, no embed)
  ├─ sha changed    → reparse, re-embed only chunks whose text hash changed
  └─ file deleted   → delete points by payload filter, delete symbol rows
```

Chunk-level hashing is the important optimisation: editing one function in a 40-function file re-embeds one chunk, not forty. On a typical commit this is a sub-second job.

**Triggers:** git post-commit hook (optional), a `code_index_sync` MCP call, a filesystem watcher in dev, or a poll interval. Indexing is always async via the worker; `code_index_status` reports progress and staleness.

### (3–4) AST chunking

Tree-sitter, with a per-language query file selecting chunkable nodes:

```scheme
; indexer/queries/python.scm
(function_definition) @chunk
(class_definition)    @chunk
(decorated_definition) @chunk
```

Rules:

- **Never split a symbol.** A function is one chunk even if long.
- **Oversized symbols** (> 1,200 tokens) split at statement boundaries into `part 1/n`, each keeping the full signature as a header.
- **Undersized symbols** (< 60 tokens) merge with siblings up to ~400 tokens — one-line getters as individual chunks are pure noise.
- **Leftovers** (imports, module-level constants, config) become a `module_header` chunk per file.
- **Unsupported languages** fall back to 60-line sliding windows with 10-line overlap. Works fine for config, SQL, and templates.

### (5) Chunk enrichment — the highest-leverage step

Each chunk is embedded *with a header*, not raw:

```
repo: acme-api | path: src/auth/session.ts | lang: typescript
symbol: AuthService.createSession
signature: async createSession(userId: string, opts: SessionOpts): Promise<Session>
imports: redis, jose, ./config
doc: Creates a server-side session and stores it in Redis with a sliding TTL.
---
async createSession(userId: string, opts: SessionOpts): Promise<Session> {
  ...
}
```

Raw code embeds poorly — identifiers are terse and the model has no idea what `s.ttl` means. The header supplies natural-language signal and typically lifts retrieval quality substantially for the same embedding cost. Where a docstring is missing and the symbol is public, an optional worker pass generates a one-line description via the LLM gateway (cached, cheap, run once per chunk hash).

### (6) Embedding

| Setting | Default | Notes |
|---|---|---|
| Model | `BAAI/bge-m3` (1024-dim) via TEI | Multilingual, strong on code, permissive licence, runs on CPU. |
| Alternative (air-gapped, small) | `nomic-embed-text` via Ollama | Lower quality, tiny footprint. |
| Alternative (hosted) | `voyage-code-3` | Best code retrieval available; requires internet + key. |
| Batch size | 64 | Tune to your CPU/GPU. |
| Storage | int8 scalar quantization, `always_ram` | ~4× memory reduction, negligible recall loss at this scale. |

**Model changes require a reindex.** `embedding_model` is stored on every chunk row and in the collection alias; the indexer refuses to mix models in one collection and will build a new collection then atomically swap the alias.

### Qdrant collection

```jsonc
// collection: code   (alias -> code_bge-m3_v1)
{
  "vectors": { "dense": { "size": 1024, "distance": "Cosine" } },
  "quantization_config": { "scalar": { "type": "int8", "always_ram": true } },
  "payload_schema": {
    "project_id": "keyword", "repo": "keyword",  "branch": "keyword",
    "path": "keyword",       "lang": "keyword",
    "symbol": "keyword",     "symbol_kind": "keyword",
    "start_line": "integer", "end_line": "integer",
    "git_sha": "keyword",    "chunk_hash": "keyword",
    "is_test": "bool",       "indexed_at": "datetime"
  }
}
```

`is_test` is broken out because "show me the implementation" and "show me the tests" are both frequent and you want to filter, not hope.

### (7) Symbol graph

```sql
CREATE TABLE symbols (
  id          TEXT PRIMARY KEY,
  project_id  TEXT NOT NULL, repo TEXT NOT NULL, branch TEXT NOT NULL,
  path        TEXT NOT NULL,
  name        TEXT NOT NULL,          -- createSession
  qualified   TEXT NOT NULL,          -- AuthService.createSession
  kind        TEXT NOT NULL,          -- function|method|class|interface|type|const|enum
  lang        TEXT NOT NULL,
  signature   TEXT,
  doc         TEXT,
  start_line  INT, end_line INT,
  visibility  TEXT,                   -- public|private|internal (best effort)
  git_sha     TEXT NOT NULL,
  tsv         TSVECTOR GENERATED ALWAYS AS (to_tsvector('simple', qualified)) STORED
);
CREATE INDEX ON symbols (project_id, repo, name);
CREATE INDEX ON symbols USING GIN (tsv);

CREATE TABLE symbol_refs (
  from_path   TEXT NOT NULL,
  from_line   INT  NOT NULL,
  to_name     TEXT NOT NULL,          -- unresolved name, matched by heuristic
  to_symbol_id TEXT,                  -- resolved when unambiguous
  project_id  TEXT NOT NULL, repo TEXT NOT NULL,
  kind        TEXT                    -- call|import|extends|implements
);
CREATE INDEX ON symbol_refs (project_id, to_name);
```

**Honest limitation.** tree-sitter has no type resolution, so `symbol_refs` is name-based and will be wrong on overloads, dynamic dispatch, and same-named methods across classes. That's an acceptable trade for zero-config, 40-language, always-parses indexing. It's mitigated three ways: (a) `to_symbol_id` is only set when the name resolves uniquely in scope, otherwise left NULL and reported as ambiguous; (b) results carry a `confidence` field the agent can see; (c) an optional per-language LSP enrichment pass is on the roadmap for the languages a project actually uses.

Do not present name-matched references to agents as ground truth — the tool response explicitly says `"resolution": "heuristic"`.

---

## Search

```
query
  ├─▶ dense    Qdrant KNN, filtered by project/repo/branch/lang/is_test    top 40
  ├─▶ symbol   Postgres exact + prefix + trigram on symbols.qualified      top 20
  └─▶ literal  ripgrep over the working tree (when the query looks like    top 20
               an identifier, error string, or config key)
                        │
                  RRF fusion (k=60)
                        │
                  group by file, cap 3 chunks per file
                        │
                  expand context ±5 lines to symbol boundaries
                        │
                  token-budget truncation
```

**Query routing heuristic:** if the query matches `^[A-Za-z_][A-Za-z0-9_.]*$` or is quoted, weight symbol + literal higher; if it's a natural-language sentence, weight dense higher. Cheap, and it fixes the most common complaint about pure-semantic code search.

**Results are always returned with `path`, `start_line`, `end_line`, and `git_sha`** so an agent can open the exact region in its own editor rather than trusting the snippet. This is important: the index is for *finding*, the client's file tools are for *reading*. Snippets can be stale between index syncs; the git_sha tells the agent whether to re-read.

---

## MCP tools

| Tool | Purpose |
|---|---|
| `code_search` | Hybrid search. Args: `query`, `repo?`, `lang?`, `path_glob?`, `include_tests?`, `top_k?`, `max_tokens?`. |
| `code_symbol` | Look up a symbol by name or qualified name. Returns definition + signature + doc + location. |
| `code_refs` | References to a symbol. Returns `resolution: exact|heuristic` and `confidence`. |
| `code_file_outline` | Symbol tree for a file, no bodies. Extremely cheap way to understand a file — usually 50× smaller than reading it. |
| `code_neighbors` | Chunks structurally adjacent to a given chunk (same file, same class, importers/imported). |
| `code_index_status` | Per-repo: last indexed sha, staleness in commits, chunk/symbol counts, pending jobs. |
| `code_index_sync` | Trigger an incremental (or `full=true`) reindex. Returns a job id. |

`code_file_outline` is the tool agents should reach for first and often don't — the brief's role prompt for `implementer` and `reviewer` explicitly instructs outline-before-read.

---

## Performance targets (single host, 8 vCPU, 16 GB)

| Repo size | First index | Incremental commit | Search p95 |
|---|---|---|---|
| 10k LOC | ~20 s | < 1 s | < 80 ms |
| 200k LOC | ~6 min | 1–3 s | < 150 ms |
| 2M LOC | ~50 min | 3–10 s | < 300 ms |

Dominated by embedding throughput. With a GPU on TEI, divide the index times by roughly 8–10.

**Memory:** ~1.1 KB per chunk in Qdrant with int8 quantization. A 2M LOC repo ≈ 250k chunks ≈ 280 MB resident. Comfortable on a single box.

---

## Multi-repo

A project may have several repos (`api`, `web`, `infra`). All share one `code` collection, separated by the `repo` payload field. Cross-repo search is on by default within a project, which is the right default for microservice work — the API agent needs to see the client's call sites.

```yaml
repos:
  - { name: api,   path: /repos/acme-api,   branch: main }
  - { name: web,   path: /repos/acme-web,   branch: main }
  - { name: infra, path: /repos/acme-infra, branch: main, index_only: ["**/*.tf","**/*.yaml"] }
```

---

## Build vs. buy

Several OSS MCP code indexers exist (tree-sitter + Qdrant + Ollama, local-first, 35–48 languages). Worth evaluating before writing your own.

**Use one off the shelf if** you only want semantic code search and can accept its schema.

**Build the thin service described here if** you need — and you do, given the rest of this design — project scoping shared with the knowledge center, the symbol graph, shared embedding infrastructure, the retrieval cache, and index metadata inside stage briefs. A reasonable middle path for Phase 1: vendor an existing chunker/parser library, keep the service, schema, and tool surface yours.

## Sources

- [mcp-code-indexer — local semantic codebase indexing (Qdrant + SQLite)](https://github.com/groxaxo/mcp-code-indexer)
- [Qdrant MCP server](https://github.com/mhalder/qdrant-mcp-server)
- [Semantic code search MCP servers directory — Glama](https://glama.ai/mcp/servers?query=MCP+server+for+semantic+code+snippet+search)
