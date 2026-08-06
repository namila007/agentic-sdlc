# M2 — Code Indexer

**Weeks 7–9** · [← Master plan](MILESTONES.md) · Implements [`docs/05-code-indexer.md`](../docs/05-code-indexer.md)

> **Goal:** agents stop reading whole directories.
>
> Measured: brief tokens go down, and agents call `code_file_outline` before reading.

The value here is not "semantic search is cool." It is that reading a 200-line file costs ~2,500
tokens and an outline of the same file costs ~50. Multiply by every file an agent opens
speculatively across six stages. **That** is the saving.

---

## Scope

| In | Out (deliberately) |
|---|---|
| Qdrant + TEI, `code` collection | Knowledge center (M3) |
| tree-sitter for the **top 5 languages**, not all 40 | LSP-based type resolution |
| AST chunking, enrichment headers, incremental sync | Cross-repo dependency graphs |
| Symbol table + heuristic refs | Call-graph reachability analysis |
| 6 code MCP tools + L3 retrieval cache | L1/L2 response caching (M4) |
| Code context wired into the brief with a token budget | Generated docstrings (optional worker pass, deferred) |

**Top 5 languages first.** Pick them from the actual repo profile (Q6). Adding grammar #6 is an
afternoon; getting chunking right for the first five is the work.

---

## Sub-milestones

| ID | Name | Depends on | Days |
|---|---|---|---|
| [M2.1](#m21--vector-and-embedding-infrastructure) | Vector & embedding infrastructure | M1.8 | 2 |
| [M2.2](#m22--discovery-and-incremental-change-detection) | Discovery & incremental change detection | M2.1 | 2 |
| [M2.3](#m23--tree-sitter-ast-chunking-and-enrichment) | tree-sitter AST chunking & enrichment | M2.2 | 3 |
| [M2.4](#m24--embedding-pipeline-and-code-collection) | Embedding pipeline & `code` collection | M2.1, M2.3 | 2 |
| [M2.5](#m25--symbol-graph-and-heuristic-references) | Symbol graph & heuristic references | M2.3 | 2.5 |
| [M2.6](#m26--hybrid-code-retrieval) | Hybrid code retrieval | M2.4, M2.5 | 3 |
| [M2.7](#m27--mcp-tools-l3-cache-and-brief-integration) | MCP tools, L3 cache & brief integration | M2.6 | 2.5 |

---

## The indexing pipeline

```
git repo
   │  (1) discover
   ▼
file list  ── .gitignore · .asdlcignore · size cap · binary/vendor/lockfile exclusion
   │  (2) change detect
   ▼
changed files only  ── git blob sha vs. stored sha   (full scan only on first index)
   │  (3) parse
   ▼
tree-sitter AST  ── one process · tolerant of broken syntax · no compile step
   │  (4) chunk
   ▼
AST chunks  ── split at function/class/method boundaries · NEVER mid-symbol
   │  (5) enrich
   ▼
chunk + header  ── path · language · enclosing symbol · signature · imports · docstring
   │
   ├──────(6) embed────────▶  Qdrant `code` collection
   └──────(7) extract──────▶  Postgres `symbols` + `symbol_refs`
```

Steps 6 and 7 run from the same parse. Parsing twice is the obvious mistake here.

---

## M2.1 — Vector and embedding infrastructure

**Delivers:** Qdrant and a local embedding server in the compose stack, with the `code` collection
provisioned.

### Services added

| Service | Image | Port | Volume | Healthcheck |
|---|---|---|---|---|
| `qdrant` | `qdrant/qdrant` | 6333/6334 (internal) | `qdrantdata:/qdrant/storage` | TCP 6333, 15s/3s/10 |
| `embeddings` | `ghcr.io/huggingface/text-embeddings-inference:cpu-1.8` | 8081 (internal) | `hf-cache:/data` | `curl /health`, 20s/5s/10, **`start_period: 180s`** |
| `indexer` | build `./services/indexer` | 8000 (internal) | `${REPOS_PATH}:/repos:ro` | standard py-health |

The 180-second `start_period` on `embeddings` is not padding — first boot downloads model weights
(~2 GB for bge-m3), and a shorter window makes `make up` fail spuriously on a fresh machine.

`indexer` mounts `/repos` **read-only**. It has no reason to write, and the mount enforces
invariant **I8**.

### Embedding model

| Setting | Default | Note |
|---|---|---|
| Model | `BAAI/bge-m3`, 1024-dim | Multilingual, strong on code, permissive licence, CPU-runnable via TEI |
| Air-gapped / small alternative | `nomic-embed-text` via Ollama | Lower quality, tiny footprint |
| Hosted alternative | `voyage-code-3` | Best code retrieval available; needs internet + API key |
| Batch size | 64 | Tune to available CPU/GPU |
| Quantization | int8 scalar, `always_ram: true` | ~4× memory reduction, negligible recall loss at this scale |

**Model changes require a reindex.** Store `embedding_model` on every chunk row. The indexer must
refuse to mix models in one collection: build a new collection, then atomically swap the alias.

### Collection provisioning

The collection name is an **alias** (`code`) pointing at a versioned physical collection
(`code_bge-m3_v1`). This is what makes zero-downtime reindexing possible.

```
alias:      code  →  code_bge-m3_v1

vectors:            dense · size 1024 · distance Cosine
quantization:       scalar int8, always_ram true
hnsw:               m=16, ef_construct=100
payload indexes:    project_id(keyword, tenant-optimised) · repo(keyword) · branch(keyword)
                    path(keyword) · lang(keyword) · symbol(keyword) · symbol_kind(keyword)
                    is_test(bool) · git_sha(keyword) · chunk_hash(keyword)
                    start_line(int) · end_line(int) · indexed_at(datetime)
```

**Payload indexes are not optional.** Without an index on a filtered field, a filtered search scans
every point in the collection. Since multi-tenancy filtering puts `project_id` in *every* query, an
unindexed `project_id` makes the whole design slow enough to abandon.

Mark `project_id` as a **tenant-optimised keyword index** — Qdrant applies partition-aware
optimisations to fields declared this way, materially faster than a plain keyword index at
multi-project scale. Confirm the exact parameter name against the installed client version; see
[APPENDIX](APPENDIX-tech-verification.md).

`is_test` is a discrete payload field rather than something inferred from the path, because
"show me the implementation" and "show me the tests" are both frequent queries and semantic distance
does not reliably separate them.

### Acceptance

- [ ] `make up` on a clean machine reaches healthy on all services within 5 minutes
- [ ] Collection is created via alias; `code` resolves to `code_bge-m3_v1`
- [ ] Every payload field used in any filter has an index — asserted by a startup check that fails loudly
- [ ] Switching `EMBEDDING_MODEL` and restarting refuses to write into the existing collection

---

## M2.2 — Discovery and incremental change detection

**Delivers:** re-indexing a 200k-LOC repo after a one-line commit takes under 3 seconds.

### Discovery configuration

```yaml
index:
  include: ["**/*"]
  exclude:
    - "**/node_modules/**"
    - "**/.venv/**"
    - "**/vendor/**"
    - "**/dist/**"
    - "**/build/**"
    - "**/target/**"
    - "**/*.min.js"
    - "**/*.lock"
    - "**/*.snap"
    - "**/__snapshots__/**"
    - "**/*.generated.*"
  max_file_bytes: 1048576        # 1 MB
  respect_gitignore: true
  extra_ignore_file: .asdlcignore
```

Excluding generated code and lockfiles is critical, not cosmetic: they are huge, semantically empty,
and they crowd out real results in every retrieval.

### Change detection algorithm

```
for each file:  git blob sha  (from `git ls-files -s`)
  ├─ sha unchanged  → skip entirely — no parse, no embed
  ├─ sha changed    → reparse; re-embed ONLY chunks whose text hash changed
  └─ file deleted   → delete points by payload filter; delete symbol rows
```

**Chunk-level hashing is the optimisation that matters.** Editing one function in a 40-function file
re-embeds one chunk, not forty. Embedding dominates the cost of the whole pipeline; everything else
is bookkeeping.

Renames are handled as delete + add. Git rename detection could preserve the embedding, but the
complexity is not worth it — renames are rare and the re-embed is one chunk.

### Triggers

| Trigger | Mechanism | When |
|---|---|---|
| git post-commit hook | Optional, installed by `asdlc init` | Developer machines |
| `code_index_sync` MCP call | Explicit, operator-gated (`ASDLC_EXPOSE_INDEX_SYNC`) | Manual recovery |
| Filesystem watcher | Dev mode only | Local iteration |
| Poll interval | Default, configurable | Servers without hook access |

**Always async via the worker.** `code_index_status` reports progress and staleness; a synchronous
index would block a tool call for minutes.

### Deletes

Deleting by payload filter (`repo == X AND path == Y`) rather than by point ID means the indexer
never has to track point IDs per file. It requires the payload index on `path` — another reason
M2.1's index list is mandatory.

### Acceptance

- [ ] First index of a 200k-LOC repo completes in < 10 min on 8 vCPU
- [ ] A one-line commit re-indexes in < 3 s
- [ ] Deleting a file removes its chunks and symbol rows within one sync
- [ ] A `.asdlcignore` entry excludes files that `.gitignore` does not

---

## M2.3 — tree-sitter AST chunking and enrichment

**Delivers:** chunks that are semantically whole and embed well.

### Why tree-sitter, and the honest cost

| | tree-sitter | LSP |
|---|---|---|
| Setup | One process, 40+ grammars | A language server per language |
| Broken / partial code | Parses fine | Often fails |
| Compiling project required | No | Usually yes |
| **Type resolution** | **None** | Yes |

The missing type resolution is the real cost, and it is paid in M2.5's heuristic references. LSP
enrichment stays on the deferred list until ref accuracy becomes a genuine complaint.

### Grammar loading

Use the current package-based approach — individual `tree-sitter-<lang>` pip packages (or the
language-pack aggregate), constructing a `Language` from the module's language capsule. The older
`Language.build_library()` compile-at-runtime approach is obsolete.

**py-tree-sitter has changed its API more than once**, including the query-execution surface
(`Query` / `QueryCursor`, `captures()` vs `matches()`). Verify against the installed version before
writing the chunker rather than against any document, including this one. See
[APPENDIX](APPENDIX-tech-verification.md).

### Query files

Per-language `.scm` capture files declare which node types become chunks:

```scheme
; indexer/queries/python.scm
(function_definition)  @chunk
(class_definition)     @chunk
(decorated_definition) @chunk
```

Adding a language is a `.scm` file plus a pip dependency. No code change.

### Chunking rules

| Case | Rule |
|---|---|
| **Normal symbol** | One chunk, even if long. **Never split a symbol.** |
| **Oversized** (> 1,200 tokens) | Split at statement boundaries. Each part labelled `part n/N`. **Each part keeps the full signature in its header.** |
| **Undersized** (< 60 tokens) | Merge with siblings up to ~400 tokens. A one-line getter as its own chunk is pure noise. |
| **Leftovers** (imports, module constants, config) | One `module_header` chunk per file |
| **Unsupported language** | 60-line sliding windows, 10-line overlap. Fine for config, SQL, templates. |

The invariant: **chunk boundaries are always AST boundaries** (or explicit fallback windows), never
arbitrary character offsets.

### Enrichment header — the highest-leverage 20 lines in the indexer

Raw code embeds poorly. Identifiers are terse and the model has no context — what does `s.ttl` mean?
Every chunk is embedded with a natural-language header prepended:

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

This costs nothing extra at embed time and lifts retrieval quality substantially. If G2 fails —
retrieval loses to grep — **fix the header before building more retrieval surface.** It is almost
always the header.

Generating docstrings for undocumented public symbols via the LLM gateway (cached by chunk hash) is
an optional worker pass, deferred until after M4 since it needs the gateway's cache to be affordable.

### Acceptance

- [ ] Top-5 languages parse and chunk; a deliberately syntax-broken file still yields chunks
- [ ] No chunk boundary falls inside a function body — asserted over a fixture corpus
- [ ] A 3,000-token function produces parts each carrying the full signature
- [ ] An unsupported extension falls back to sliding windows without erroring
- [ ] Enrichment headers are present on 100% of chunks

---

## M2.4 — Embedding pipeline and `code` collection

**Delivers:** chunks in Qdrant, incrementally, with correct payloads.

### Payload written per chunk

```
project_id · repo · branch · path · lang
symbol · symbol_kind      -- function|method|class|interface|type|const|enum
start_line · end_line     -- 1-indexed, inclusive
git_sha                   -- blob sha, so the agent can tell whether the snippet is stale
chunk_hash                -- for incremental detection
is_test · indexed_at
```

`git_sha` on every result is what makes staleness visible. Snippets **will** be stale between syncs;
the agent needs to know whether to re-read the file from its own filesystem.

### Batching and throughput

Batch size 64 into TEI. Use batch upsert into Qdrant rather than per-point calls — the difference is
roughly an order of magnitude. Bound the in-flight embedding queue so a full reindex of a 2M-LOC repo
does not exhaust memory.

### Performance targets (single host, 8 vCPU / 16 GB)

| Repo size | First index | Incremental commit | Search p95 |
|---|---|---|---|
| 10k LOC | ~20 s | < 1 s | < 80 ms |
| 200k LOC | ~6 min | 1–3 s | < 150 ms |
| 2M LOC | ~50 min | 3–10 s | < 300 ms |

Dominated by embedding throughput. With a GPU on TEI, divide index times by roughly 8–10.

Memory: ~1.1 KB per chunk in Qdrant with int8 quantization. A 2M-LOC repo ≈ 250k chunks ≈ 280 MB
resident. Comfortable on one box.

### Multi-repo

```yaml
repos:
  - { name: api,   path: /repos/acme-api,   branch: main }
  - { name: web,   path: /repos/acme-web,   branch: main }
  - { name: infra, path: /repos/acme-infra, branch: main, index_only: ["**/*.tf","**/*.yaml"] }
```

All repos share one `code` collection, separated by the `repo` payload field. **Cross-repo search is
on by default** — for microservices this is correct: the API agent needs to see client call sites.

### Acceptance

- [ ] Performance targets met on a real repo of each size class
- [ ] Every chunk carries all payload fields; a startup check fails on any null in a filtered field
- [ ] Re-indexing an unchanged repo writes zero points
- [ ] Cross-repo search returns results from more than one repo when relevant

---

## M2.5 — Symbol graph and heuristic references

**Delivers:** exact symbol lookup, and references that are honest about their uncertainty.

### Schema

```sql
CREATE TABLE symbols (
  id          TEXT PRIMARY KEY,
  project_id  TEXT NOT NULL,
  repo        TEXT NOT NULL,
  branch      TEXT NOT NULL,
  path        TEXT NOT NULL,
  name        TEXT NOT NULL,     -- createSession
  qualified   TEXT NOT NULL,     -- AuthService.createSession
  kind        TEXT NOT NULL,     -- function|method|class|interface|type|const|enum
  lang        TEXT NOT NULL,
  signature   TEXT,
  doc         TEXT,
  start_line  INT, end_line INT,
  visibility  TEXT,              -- public|private|internal (best effort)
  git_sha     TEXT NOT NULL,
  tsv         TSVECTOR GENERATED ALWAYS AS (to_tsvector('simple', qualified)) STORED
);
CREATE INDEX ON symbols (project_id, repo, name);
CREATE INDEX ON symbols USING GIN (tsv);

CREATE TABLE symbol_refs (
  from_path    TEXT NOT NULL,
  from_line    INT  NOT NULL,
  to_name      TEXT NOT NULL,    -- unresolved name, matched heuristically
  to_symbol_id TEXT,             -- set ONLY when the name resolves uniquely; else NULL
  project_id   TEXT NOT NULL,
  repo         TEXT NOT NULL,
  kind         TEXT              -- call|import|extends|implements
);
CREATE INDEX ON symbol_refs (project_id, to_name);
```

`to_tsvector('simple', ...)` — **not** `'english'`. Stemming `createSession` is actively harmful.

### The honesty requirement

tree-sitter has no type resolution, so `symbol_refs` is **name-based**. It will be wrong on
overloads, on dynamic dispatch, and on same-named methods across different classes. That is the
accepted trade for zero-config, 40-language, always-parses indexing.

Three mitigations, all mandatory:

1. `to_symbol_id` is set **only** when the name resolves uniquely in scope. Otherwise it stays NULL
   and the ref is reported as ambiguous.
2. Every result carries a `confidence` field and an explicit `"resolution": "exact" | "heuristic"`.
3. Optional per-language LSP enrichment stays on the roadmap for the languages the project actually uses.

**Agents must never be shown name-matched references as ground truth.** The tool response says
`resolution: "heuristic"` in plain text, and the `reviewer` and `implementer` role prompts must
state that heuristic refs are leads to verify, not facts.

### Acceptance

- [ ] Exact symbol lookup by qualified name is < 20 ms
- [ ] A deliberately overloaded method yields refs with `resolution: "heuristic"` and `to_symbol_id = NULL`
- [ ] No code path emits a ref without a `resolution` field
- [ ] Symbol rows are deleted when their file is deleted

---

## M2.6 — Hybrid code retrieval

**Delivers:** search that beats grep on natural-language questions **and** finds exact identifiers.

### The pipeline

```
query
  ├─▶ dense    Qdrant KNN, filtered by project/repo/branch/lang/is_test    top 40
  ├─▶ symbol   Postgres exact + prefix + trigram on symbols.qualified      top 20
  └─▶ literal  ripgrep over the working tree                              top 20
               (only when the query looks like an identifier,
                an error string, or a config key)
                       │
                 RRF fusion (k = 60)
                       │
                 group by file, cap 3 chunks per file
                       │
                 expand context ±5 lines to symbol boundaries
                       │
                 token-budget truncation (at chunk boundaries, never mid-chunk)
```

### RRF

```
RRF(d) = Σ over retrievers  1 / (k + rank_r(d)),   k = 60
```

RRF rather than score normalisation, because Qdrant cosine scores and Postgres `ts_rank_cd` scores
live on incomparable scales. RRF needs no tuning, which is its main virtue.

### Query routing heuristic

```
if query matches ^[A-Za-z_][A-Za-z0-9_.]*$  OR  is quoted:
      weight symbol + literal higher
else if it reads as a natural-language sentence:
      weight dense higher
```

A cheap heuristic that fixes the most common complaint about pure-semantic code search: it misses
exact identifiers. Someone searching `createSession` wants the definition, not five thematically
similar functions.

### Capping per file

Three chunks per file maximum. Without it, one large file with many similar methods dominates every
result set and the agent sees one file instead of the five it needed.

### Result shape

Always includes `path`, `start_line`, `end_line`, `git_sha` — so the agent can open the exact region
in its own editor rather than trusting a possibly-stale snippet.

### Acceptance

- [ ] Search p95 within the M2.4 targets **with filters applied**, not just unfiltered
- [ ] A bare identifier query returns the definition as result #1
- [ ] No file contributes more than 3 chunks to a result set
- [ ] Truncation never cuts a chunk in half; `meta.truncated` is set when it happens

### G2 — the blind comparison

Take **20 real "where is X" questions** from actual work. Run each through `code_search` and through
`grep`/`rg`. Blind-rate the top-5 relevance of each.

If retrieval does not beat grep: **do not build more retrieval surface.** Fix the enrichment header
(M2.3), then re-run. In practice the header is the cause nearly every time. If it still loses after
a header fix, the honest conclusion is that this repo is better served by exact search, and the
dense half of the pipeline should be de-weighted rather than expanded.

---

## M2.7 — MCP tools, L3 cache and brief integration

**Delivers:** six code tools on the MCP surface, retrieval results cached, and code context flowing
into the brief within a token budget.

### The tools

| Tool | Params | Returns |
|---|---|---|
| `code_search` | `query`, `repo?`, `lang?`, `path_glob?`, `include_tests?`, `top_k?`, `max_tokens?` | `{chunks:[{path,start_line,end_line,lang,symbol,signature,snippet,git_sha,score,source}]}` |
| `code_symbol` | `name` \| `qualified`, `repo?` | `{name,qualified,kind,path,start_line,end_line,signature,doc,visibility,git_sha}` or null |
| `code_refs` | `symbol`, `repo?`, `kind?` | `{references:[{from_path,from_line,kind,confidence,resolution}]}` |
| `code_file_outline` | `path`, `repo?` | `{symbols:[{name,kind,start_line,end_line,doc_short,children:[]}]}` — **no bodies** |
| `code_neighbors` | `path`+`line` \| `symbol` | `{neighbors:[…]}` — same file, same class, importers/imported |
| `code_index_status` | `repo?` | `{last_indexed_sha,staleness_minutes,staleness_commits,chunk_count,symbol_count,pending_jobs}` |

`code_index_sync` is exposed **only** when `ASDLC_EXPOSE_INDEX_SYNC=true`. It is an operator action,
not something an agent should trigger mid-stage.

### `code_file_outline` is the point of the milestone

Roughly 50× cheaper than reading a file, and agents rarely reach for it unprompted. Two things must
happen:

1. The `implementer` and `reviewer` role prompts state explicitly: **outline before read**, for any
   file over ~200 lines.
2. The metric "outline calls per file read" is tracked. If agents are not calling it, the prompt is
   not working — and no amount of tool availability fixes that.

Default `top_k` is deliberately low — **10 for code**. Agents that need more can ask; the default is
what actually gets used.

### L3 retrieval cache

| Sub-cache | Key | TTL | Invalidation |
|---|---|---|---|
| Query embedding | `emb:{model}:{sha256(text)}` | none | on model change |
| Code results | `sha256("code:v1", project_id, normalized_query, filters_canonical, top_k, max_tokens, embedding_model, git_sha)` | until repo head moves | **automatic** — `git_sha` is in the key |

Putting `git_sha` in the key means there is **never an invalidation step**. Stale keys simply stop
being requested and expire naturally. This is simpler and far less bug-prone than explicit
invalidation, which is the kind of mechanism that works for six months and then silently serves
stale results after an edge-case commit.

Cache namespace is prefixed with `project_id` (**I1**) — a shared cache across tenants would be a
cross-tenant read primitive, which is a worse bug than a lower hit rate.

### Brief integration

```yaml
context_policy.code:
  enabled: true
  top_k: 15
  max_tokens: 5000
  prefer: [interfaces, module_boundaries, config]
```

The brief builder calls `code_search` with the stage's intent, truncates to `max_tokens`, and places
results in **block 6** — the last, never-cached block (M0.3). Results are wrapped:

```xml
<untrusted-context source="repository" trust="data-only">
  …retrieved chunks…
</untrusted-context>
```

**This wrapper is invariant I7.** A malicious comment in a vendored dependency is a real injection
vector; the boundary is what turns it into a reportable finding rather than an executed instruction.

### Acceptance

- [ ] All six tools registered; combined schema size < 1,200 tokens
- [ ] L3 hit rate > 0.5 in steady state
- [ ] A commit to the repo changes the cache key and produces fresh results with no explicit flush
- [ ] Brief code block never exceeds `max_tokens`; excess truncated at chunk boundaries
- [ ] Retrieved code is wrapped in `<untrusted-context>` in every brief

---

## M2 exit criteria

- [ ] 200k-LOC repo indexes in < 10 min; incremental commit < 3 s; search p95 < 150 ms
- [ ] **Measured:** brief tokens down versus the M1 baseline on the same runs
- [ ] **Measured:** agents call `code_file_outline` before reading large files
- [ ] **G2:** a blind comparison on 20 real "where is X" questions beats grep on relevance
- [ ] Every `code_refs` result carries `resolution` and `confidence`

## Deferred from M2

| Deferred | Revisit when |
|---|---|
| LSP enrichment of the symbol graph | Ref accuracy becomes a real, repeated complaint |
| Generated docstrings for undocumented symbols | After M4 — needs the gateway cache to be affordable |
| Languages 6–40 | On demand, one `.scm` file at a time |
| Call-graph reachability | Not planned; heuristic refs plus `code_neighbors` covers the actual queries |
