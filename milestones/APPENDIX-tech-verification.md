# Appendix — Tech Verification

[← Master plan](MILESTONES.md)

Library APIs and versions gathered via Context7 while writing these milestones, plus **the specific
claims that still need hands-on verification before you write code against them.**

> **Read the caveats.** Documentation aggregators lag real releases, and several of these libraries
> have changed their public API more than once. Where a claim below is marked ⚠️, treat this document
> as a starting point and verify against the package you actually install — not the other way round.

---

## MCP server stack

Used by: [M0.1](M0-walking-skeleton.md), [M0.4](M0-walking-skeleton.md)

### ⚠️ Version and API surface — verify before writing the gateway

The reported picture was **internally inconsistent** across sources: one described FastMCP v3.x with
`mcp.http_app(...)`, another described an official-SDK class named `MCPServer` with
`streamable_http_app(...)` "replacing v1's FastMCP class". These cannot both be current descriptions
of the same ecosystem.

**Do this first, before any gateway code:**

```bash
uv add fastmcp && python -c "import fastmcp; print(fastmcp.__version__)"
python -c "import fastmcp, inspect; print(inspect.signature(fastmcp.FastMCP.http_app))"
```

Then write the gateway against the signature you actually see. Budget half a day for this in M0.1.

### What is well established

**Transport.** Streamable HTTP is the standard; **HTTP+SSE held-open-stream is deprecated.** Do not
implement SSE.

**Stateless mode is required for our design.** With session affinity, multi-worker and multi-replica
deployment fails. The gateway fronts several internal services and will be replicated (M6.8), so
stateless is not optional.

**Session-manager lifespan.** When mounting the MCP app inside a host FastAPI/Starlette app, the
host's lifespan must run the MCP session manager. Skipping this produces failures that look like
transport bugs.

```python
@contextlib.asynccontextmanager
async def lifespan(app):
    async with mcp.session_manager.run():   # required when mounting
        yield
```

**Per-request context.** Both the header dict and any parsed access token are reachable inside a tool
handler through the framework's request-context/dependency mechanism. This is how `Authorization` and
`Mcp-Project` reach the project-scoping logic (M0.4).

**Request isolation.** Under stateless mode, shared state (DB pools, clients) must use context
variables or explicit per-request construction — never module-level mutable globals.

**Elicitation.** Supported for mid-tool input requests, but **client support is uneven**. Every
elicitation path must degrade to a plain error with a `fix` string telling the agent what to pass
explicitly. Never make a flow depend on elicitation working.

**Protocol version.** Read it from the installed SDK. Do not hardcode a version string taken from a
blog post or from this document.

---

## LiteLLM proxy

Used by: [M4.1–M4.4](M4-gateway-cache-cost.md)

Reported stable at time of writing: **v1.81.x**. Verify with `litellm --version` in the pinned image
before relying on any key name below.

### Confirmed config surface

| Section | Purpose |
|---|---|
| `model_list[]` | `model_name` is the public alias; `litellm_params.model` is what is actually called. `os.environ/VAR` references env vars. |
| `router_settings` | `routing_strategy`, `fallbacks`, `context_window_fallbacks`, `num_retries`, `timeout` |
| `litellm_settings` | `cache`, `cache_params`, `drop_params`, callbacks |
| `general_settings` | `master_key`, `database_url`, `alerting` |

**`general_settings.database_url` is required** for virtual keys, spend tracking and the admin UI.
Tables are created on first run. Without it, M4.2 does not work at all.

### Local OpenAI-compatible endpoints

The `openai/` prefix plus `api_base` routes to TEI, Ollama or vLLM:

```yaml
- model_name: embed
  litellm_params:
    model: openai/BAAI/bge-m3
    api_base: http://embeddings:8081/v1
    api_key: none
```

This is the indirection that makes the air-gap swap (M6.5) a config change.

### Caching

**Exact (`type: redis`)** — `host`, `port`, `password`, `namespace`, `ttl`, `supported_call_types`
(`acompletion`, `atext_completion`, `aembedding`, `atranscription`).

**Semantic (`type: redis-semantic`)** — requires an embedding model **declared in `model_list`** and
referenced by `model_name` via `redis_semantic_cache_embedding_model`. `similarity_threshold` ranges
0–1. Valkey- and Qdrant-backed semantic caching also exist.

**Per-request bypass** via the `cache-control: no-cache` header. The brief builder uses this on any
call feeding an artifact (M4.3).

### Virtual keys

`POST /key/generate` with master-key auth. Fields: `key_alias`, `max_budget`, `budget_duration`,
`tpm_limit`, `rpm_limit`, `team_id`, `metadata`. Nested `metadata.spend_logs_metadata` fields appear
in spend logs — this is how per-run/stage/role attribution works (M4.2).

⚠️ `budget_duration` accepted values were reported inconsistently (`"30d"` vs `"monthly"`). Test the
exact string against your version.

---

## Qdrant + qdrant-client

Used by: [M2.1](M2-code-indexer.md), [M2.4](M2-code-indexer.md), [M3.1](M3-knowledge-center.md)

### Collection creation

`create_collection(collection_name, vectors_config, hnsw_config, quantization_config, ...)` with
config classes from `qdrant_client.models`:

```
VectorParams · Distance.COSINE · HnswConfigDiff(m, ef_construct)
ScalarQuantization(scalar=ScalarQuantizationConfig(type=ScalarType.INT8, always_ram=True))
SparseVectorParams · MultiVectorConfig
```

Named vectors are passed as a dict: `{"dense": VectorParams(size=1024, distance=Distance.COSINE)}`.

### Payload indexes — mandatory, not optional

`create_payload_index(collection_name, field_name, field_schema)` with
`PayloadSchemaType.{KEYWORD, INTEGER, FLOAT, BOOLEAN, DATETIME, GEO}`.

**Without an index on a filtered field, a filtered search scans every point.** Since `project_id`
appears in every query, an unindexed `project_id` makes the design unusable at scale.

### Multi-tenancy — the recommended pattern

**Single collection + a tenant-marked keyword index on `project_id`.** Qdrant applies
partition-aware optimisations to fields declared this way, materially faster than a plain keyword
index at multi-project scale.

⚠️ The exact parameter for tenant marking was reported as `is_tenant=True` inside
`KeywordIndexParams`. Verify against the installed client — this is a newer option and the shape has
moved.

Collection-per-tenant is only warranted when a tenant needs independent snapshots, independent
embedding models, or hard-deletion guarantees. Up to that point, payload filtering is correct, fast
and simpler.

### Query API

`query_points(collection_name, query, query_filter, limit, with_payload)` is the current universal
entry point — dense, sparse, recommend-by-example and fusion all flow through it. `search()` was
**not reported as formally deprecated**, but `query_points()` is the recommended API for new code.

Filters: `Filter(must=[…], should=[…], must_not=[…])` with
`FieldCondition(key=…, match=MatchValue(...) | MatchAny(...) | range=Range(gte=…, lt=…))`.

### Server-side RRF fusion

```python
query_points(
    prefetch=[Prefetch(query=dense_vec,  using="dense",  limit=50),
              Prefetch(query=sparse_vec, using="sparse", limit=50)],
    query=FusionQuery(fusion=Fusion.RRF),
    limit=10)
```

Worth evaluating in M3.3 as an alternative to fusing in application code — but note our sparse half
lives in **Postgres BM25**, not Qdrant, so application-side RRF is likely to remain necessary
regardless.

### Aliases and snapshots

`update_collection_aliases([DeleteAliasOperation(...), CreateAliasOperation(...)])` performs an
all-or-nothing swap. This is the zero-downtime model-swap mechanism in M2.1/M2.4.

`create_snapshot`, `list_snapshots` for backup — though M6.6 concludes Qdrant should be **rebuilt
rather than restored**, since it is derived data.

### Batch and delete

`batch_update_points(...)` for mixed atomic operations; far faster than sequential upserts.
`delete(points_selector=FilterSelector(filter=…))` for delete-by-filter — which is how M2.2 removes a
deleted file's chunks without tracking point IDs.

---

## tree-sitter (py-tree-sitter)

Used by: [M2.3](M2-code-indexer.md)

### ⚠️ Highest-uncertainty item in this appendix

The Python `Query` / `QueryCursor` API was **not fully documented** in what Context7 returned, and
py-tree-sitter has changed this surface more than once. Specifically unverified:

- `captures()` vs `matches()` return shapes
- Whether queries are built via `language.query(scm)` or a standalone `Query` constructor
- The exact `QueryCursor` methods for byte-range restriction

**Verify against the installed package before writing the chunker**, e.g.:

```bash
python -c "import tree_sitter, inspect; print([n for n in dir(tree_sitter) if not n.startswith('_')])"
```

Budget half a day in M2.3 for this. Do not write the chunker against this document.

### What is established

Grammar loading uses **individual `tree-sitter-<lang>` pip packages** (or a language-pack aggregate),
constructing `Language(...)` from the module's language capsule and assigning it to a `Parser`. The
old `Language.build_library()` compile-at-runtime approach is obsolete.

Node byte ranges (`start_byte`, `end_byte`) and point positions (`start_point`, `end_point` as
row/column) are available on every node — these are what M2.3 uses for chunk boundaries and M2.4
writes as `start_line`/`end_line`.

---

## arq

Used by: [M1.7](M1-full-pipeline.md), [M3.6](M3-knowledge-center.md), and the worker throughout

Reported version: **v0.26.x**.

```python
from arq import cron, Retry
from arq.connections import RedisSettings

class WorkerSettings:
    functions = [mirror_to_git, ingest_knowledge, index_repo]
    on_startup = startup
    on_shutdown = shutdown
    redis_settings = RedisSettings(host="redis", port=6379, database=3)
    cron_jobs = [cron(staleness_scan, hour={3}, minute=0)]
```

- Task functions take `ctx` as the first argument (`ctx['job_try']` gives the attempt number)
- `await redis.enqueue_job('name', *args, _job_id=…, _queue_name=…, _defer_by=…, _expires=…)`
- `_job_id` **enforces uniqueness** — this is how M1.7 makes the git-mirror job idempotent
- `job.status()`, `job.info()`, `job.result(timeout=…)`
- Retry with backoff: `raise Retry(defer=ctx['job_try'] * 5)`
- `cron(fn, hour={…}, minute=…)` for the M3.6 weekly staleness worker

Queue splitting (M6.8) uses `_queue_name` with a worker per queue, so a full reindex cannot starve
git-mirror jobs.

---

## Alembic (async)

Used by: [M0.2](M0-walking-skeleton.md) and every migration after

```python
# env.py
async def run_async_migrations():
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.", poolclass=pool.NullPool)
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()

def run_migrations_online():
    asyncio.run(run_async_migrations())
```

- `target_metadata = Base.metadata` enables autogenerate
- `poolclass=pool.NullPool` — migrations should not hold a pool
- Run as `docker compose run --rm core alembic upgrade head`, **never on service start** (concurrent
  replicas would race)
- Programmatic upgrade via `command.upgrade(cfg, "head")` with the connection passed in
  `cfg.attributes["connection"]` — useful for test fixtures

---

## Verification checklist before writing code

Do these in the milestone that first needs them. Each is under an hour, and each prevents a bad day.

| # | Verify | Milestone | Why it matters |
|---|---|---|---|
| 1 | FastMCP package name, version, and HTTP-app factory signature | M0.1 | ⚠️ Reported inconsistently; the gateway is built directly on it |
| 2 | MCP protocol version string from the installed SDK | M0.4 | Goes in `serverInfo`; must not be guessed |
| 3 | py-tree-sitter `Query` / `QueryCursor` API | M2.3 | ⚠️ Least-documented item here; the chunker depends on it |
| 4 | Qdrant tenant-index parameter name | M2.1 | ⚠️ Newer option; the wrong name means a silent perf cliff |
| 5 | `query_points` vs `search` in the installed client | M2.6 | Choose one and use it consistently |
| 6 | LiteLLM config key names in the pinned image | M4.1 | Config surface moves quickly |
| 7 | LiteLLM `budget_duration` accepted values | M4.2 | ⚠️ Reported inconsistently |
| 8 | Provider prompt-cache minimum block size | M4.5 | Below the minimum, L0 silently never engages |

Item 8 has no library to check — it is a provider documentation lookup, and it directly determines
whether the single largest client-side saving works at all.

---

## Confidence summary

| Area | Confidence | Note |
|---|---|---|
| Qdrant core API (collections, filters, payload indexes) | **High** | Consistent across sources |
| arq | **High** | Small, stable API |
| Alembic async | **High** | Long-standing documented pattern |
| LiteLLM config structure | **Medium-high** | Structure solid; individual key names move |
| MCP transport semantics (streamable HTTP, stateless, SSE deprecated) | **High** | Consistent |
| MCP **Python API surface** | ⚠️ **Low** | Sources contradicted each other on class names and versions |
| py-tree-sitter query API | ⚠️ **Low** | Not fully documented in what was returned |
| Qdrant tenant-index parameter | ⚠️ **Medium** | Pattern confirmed; exact parameter name unverified |

The two low-confidence items are both first-week work in their respective milestones (M0.1, M2.3) —
deliberately, so they get verified before anything is built on top of them.
