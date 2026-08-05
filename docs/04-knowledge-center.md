# 04 — Knowledge Center

Project-scoped, curated, retrievable memory. The thing that makes run #47 smarter than run #1.

## What belongs here (and what doesn't)

| Belongs | Doesn't belong |
|---|---|
| Architectural decisions and their rationale | Raw source code → [code indexer](05-code-indexer.md) |
| Team conventions ("we use Result types, never exceptions") | Full ticket dumps — ingest summaries, not backlogs |
| Domain glossary ("a *tenant* is not an *org*") | Anything secret (see [11](11-security-and-multitenancy.md)) |
| Constraints ("must run air-gapped for customer X") | Transient run state → Postgres |
| Gotchas learned the hard way | Generic language/framework docs → external docs service |
| External library docs, pinned to your versions | Opinions nobody reviewed |

**Signal-to-noise is the whole game.** A knowledge center that ingests everything retrieves noise, and noisy retrieval is worse than no retrieval — it costs tokens *and* misleads. Bias hard toward curation.

---

## Schema

```sql
CREATE TYPE knowledge_scope  AS ENUM ('global','org','project','repo','run');
CREATE TYPE knowledge_status AS ENUM ('candidate','approved','deprecated','rejected');

CREATE TABLE knowledge_entries (
  id             TEXT PRIMARY KEY,          -- kn_01J...
  project_id     TEXT REFERENCES projects(id),   -- NULL for global/org
  org_id         TEXT,
  repo           TEXT,
  scope          knowledge_scope NOT NULL,
  kind           TEXT NOT NULL,             -- adr|convention|glossary|constraint|
                                            -- gotcha|runbook|external_doc|faq
  title          TEXT NOT NULL,
  body           TEXT NOT NULL,             -- markdown
  summary        TEXT,                      -- ≤200 tokens, used in briefs

  status         knowledge_status NOT NULL DEFAULT 'candidate',
  confidence     REAL DEFAULT 0.8,

  source_type    TEXT,                      -- artifact|url|file|manual|run_harvest|context7
  source_ref     TEXT,
  evidence       JSONB DEFAULT '[]',        -- artifact ids backing the claim

  valid_from     TIMESTAMPTZ DEFAULT now(),
  valid_until    TIMESTAMPTZ,               -- for version-pinned or time-bound facts
  supersedes_id  TEXT REFERENCES knowledge_entries(id),

  tags           TEXT[] DEFAULT '{}',
  tsv            TSVECTOR GENERATED ALWAYS AS (
                   to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))
                 ) STORED,

  created_by     TEXT, approved_by TEXT, approved_at TIMESTAMPTZ,
  created_at     TIMESTAMPTZ DEFAULT now(),
  updated_at     TIMESTAMPTZ DEFAULT now(),

  usage_count    INT DEFAULT 0,             -- times retrieved into a brief
  helpful_count  INT DEFAULT 0,             -- times an agent/human marked it useful
  stale_flags    INT DEFAULT 0
);

CREATE INDEX ON knowledge_entries USING GIN (tsv);
CREATE INDEX ON knowledge_entries (project_id, status, kind);
CREATE INDEX ON knowledge_entries USING GIN (tags);

CREATE TABLE knowledge_chunks (
  id            TEXT PRIMARY KEY,
  entry_id      TEXT NOT NULL REFERENCES knowledge_entries(id) ON DELETE CASCADE,
  ordinal       INT NOT NULL,
  text          TEXT NOT NULL,
  token_count   INT NOT NULL,
  embedding_ref TEXT NOT NULL,     -- Qdrant point id
  model         TEXT NOT NULL,     -- embedding model + version
  created_at    TIMESTAMPTZ DEFAULT now()
);
```

### Qdrant collection

```jsonc
// collection: knowledge
{
  "vectors": { "dense": { "size": 1024, "distance": "Cosine" } },
  "sparse_vectors": { "bm25": {} },        // optional; see hybrid section
  "quantization_config": { "scalar": { "type": "int8", "always_ram": true } },
  "payload_schema": {
    "project_id": "keyword",   "org_id": "keyword",  "scope": "keyword",
    "kind": "keyword",         "status": "keyword",  "tags": "keyword[]",
    "entry_id": "keyword",     "chunk_id": "keyword",
    "valid_from": "datetime",  "valid_until": "datetime",
    "confidence": "float"
  }
}
```

Every query carries a mandatory filter:

```jsonc
{ "must": [
    { "key": "status", "match": { "value": "approved" } },
    { "should": [
        { "key": "project_id", "match": { "value": "<current>" } },
        { "key": "scope",      "match": { "any": ["org", "global"] } }
    ]}
  ],
  "must_not": [ { "key": "valid_until", "range": { "lt": "<now>" } } ]
}
```

Project scoping is enforced in the service layer, not left to callers. `knowledge_search` never accepts a raw filter from an agent.

---

## Ingestion

```
source ──▶ fetch ──▶ normalise ──▶ chunk ──▶ summarise ──▶ embed ──▶ Qdrant + Postgres
                                                  │
                                          (server-side LLM call
                                           via llm-gateway — cached)
```

**Sources and their handlers**

| Source | Handler | Notes |
|---|---|---|
| Approved ADRs | automatic on gate approval | Zero-friction, highest quality. |
| `knowledge_candidate` from document stage | queued for curation | See [02](02-sdlc-flow-and-agents.md#knowledge-harvest). |
| Markdown/PDF/docx files | `knowledge_ingest_file` | Tika/`markitdown` for conversion. |
| URLs | `knowledge_ingest_url` | Readability extraction; robots-respecting. |
| Confluence/Notion/Jira | connector (post-v1) | Ingest *summaries*, and only from allow-listed spaces. |
| External library docs | Context7 or local mirror | See below. |
| Manual entry | approvals-ui form | Always available; the escape hatch. |

**Chunking:** heading-aware for markdown (split at `##`, keep the heading path as a prefix on every chunk), target 512 tokens, 15% overlap. Each chunk is prefixed with `entry.title` + heading path — a cheap trick that measurably improves retrieval because chunks stop being context-free fragments.

**Summarisation:** each entry gets a ≤200-token `summary` generated once at ingest. **Briefs use summaries by default**, and only pull full `body` when the agent explicitly calls `knowledge_get`. This roughly halves the context cost of the knowledge block.

### External library docs (Context7)

You mentioned Context7. It's the right tool for "what does the current version of this library actually do" — 9,000+ libraries, version-specific, fetched from source. Two integration modes:

```yaml
external_docs:
  provider: context7           # context7 | local_mirror | none
  context7:
    url: https://mcp.context7.com/mcp
    api_key_env: CONTEXT7_API_KEY
    mode: passthrough          # passthrough | cache_ingest
    libraries:                 # pin to what the project actually uses
      - { id: "/vercel/next.js", version: "15" }
      - { id: "/pgvector/pgvector" }
```

- **`passthrough`** — the gateway proxies the Context7 tool through to the client. Simplest. Requires outbound internet.
- **`cache_ingest`** — a worker fetches the pinned libraries' docs, ingests them as `kind=external_doc, scope=project`, and thereafter serves them from local Qdrant. Slower to set up, but **works air-gapped** and stops repeated remote fetches from burning latency and tokens.

**Caveat worth knowing:** Context7 is an Upstash-hosted service; self-hosting is enterprise-only. For your air-gapped requirement, `cache_ingest` (populated once from a connected machine, then exported) or `local_mirror` (point at vendored docs in the repo) are the workable paths. Don't design a hard dependency on it.

---

## Retrieval

Hybrid, because dense-only retrieval reliably misses exact identifiers (error codes, function names, config keys) and sparse-only misses paraphrase.

```
query
  ├─▶ dense:  embed(query) → Qdrant KNN (filtered)      top 40
  └─▶ sparse: Postgres ts_rank_cd over tsv (filtered)   top 40
                          │
                    RRF fusion (k=60)
                          │
                  optional cross-encoder rerank (bge-reranker-v2-m3)
                          │
                  MMR diversity (λ=0.6)  ← avoids 5 chunks from one doc
                          │
                  token-budget truncation
                          │
                    top_k results
```

**Reciprocal Rank Fusion** rather than score normalisation: score scales from Qdrant cosine and Postgres `ts_rank_cd` aren't comparable, and RRF needs no tuning.

```
RRF(d) = Σ_over_retrievers 1 / (k + rank_r(d)),  k = 60
```

**Reranking** is optional (`RERANK_ENABLED`). It costs ~80ms per query on CPU for 80 candidates and meaningfully improves precision. Turn it on if your knowledge base exceeds ~2,000 entries.

**Recency and confidence** apply as a final multiplicative boost:

```
final = rrf_score × (0.7 + 0.3 × confidence) × recency_decay(updated_at, halflife=180d)
```

Deprecated entries are excluded; superseded entries are excluded unless `include_superseded=true` (used by the "why did we change our mind" query).

---

## Curation

The workflow that keeps quality up.

```
candidate ──review──▶ approved ──┬──▶ (used in briefs)
    │                            │
    │                     stale_flags ≥ 3
    │                            ▼
    └──reject──▶ rejected     needs_review ──▶ deprecated | updated (new version)
```

**Who creates candidates:** the document stage, ingestion of new files, and any agent calling `knowledge_propose`.

**Who approves:** a human, in the approvals UI. Batch approve/reject with a diff view against existing similar entries (the UI surfaces the top-3 nearest existing entries so duplicates and contradictions are caught at the point of decision).

**Staleness detection**, run weekly by a worker:

1. **Contradiction scan** — for each approved entry, find near-duplicate entries with cosine > 0.9 and flag pairs whose bodies disagree (cheap LLM judge call via the gateway, heavily cached).
2. **Code drift** — entries referencing file paths or symbols that no longer exist in the code index get `stale_flags += 1`.
3. **Disuse** — approved entries never retrieved in 180 days get flagged for archive review (not auto-deleted).
4. **Explicit feedback** — `knowledge_feedback(entry_id, helpful|stale|wrong)`, callable by agents mid-run. Cheap, and agents are surprisingly good at noticing "this contradicts what I'm seeing in the code."

**Never auto-approve.** A single wrong approved fact silently degrades every downstream run and is very hard to trace. Curation is the cheapest quality lever in the system.

---

## MCP tools

| Tool | Purpose |
|---|---|
| `knowledge_search` | Hybrid retrieval, project-scoped. Returns summaries + scores + entry ids. |
| `knowledge_get` | Full body of an entry by id. Use after search when the summary isn't enough. |
| `knowledge_propose` | Create a `candidate` entry with evidence. Never writes approved. |
| `knowledge_feedback` | Mark an entry helpful / stale / wrong. |
| `knowledge_list` | Browse by scope/kind/tag. For "what conventions does this project have?" |

`knowledge_ingest_*` and `knowledge_approve` are **not** exposed to agents — they're UI/CLI/worker operations. Agents propose; humans dispose.

---

## Bootstrapping a new project

`asdlc project init` runs a guided pass so the knowledge center isn't empty on day one:

1. Ingest existing `docs/adr/**`, `ARCHITECTURE.md`, `CONTRIBUTING.md`, `AGENTS.md`, `README.md`.
2. Derive conventions from the code index (dominant test framework, error-handling style, directory layout) as **candidates** — surprisingly accurate, and always human-reviewed.
3. Pull the dependency manifest and, if configured, prefetch Context7 docs for the top N direct dependencies.
4. Present everything in the curation UI as a single batch.

Expect ~30 candidates and ~15 minutes of human review for a mature repo. That investment is what makes the first real run good instead of generic.

---

## Sources

- [Context7 — Upstash](https://github.com/upstash/context7)
- [Context7 MCP FAQ (self-hosting availability)](https://context7mcp.com/faq/)
- [Qdrant MCP server implementations](https://mcpservers.org/servers/steiner385/qdrant-mcp-server)
