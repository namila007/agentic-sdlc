# 01 — Architecture

## System diagram

```
 CLIENT SIDE (inference happens here — your subscription, your machine)
┌──────────────────────────────────────────────────────────────────────────┐
│  Claude Code        Cursor         GitHub Copilot        Codex / Zed     │
│  CLAUDE.md +        .cursor/       .github/copilot-      AGENTS.md       │
│  skills + agents    rules/*.mdc    instructions.md                       │
│         └───────────────┴──────────────┴───────────────────┘             │
│                            thin bootstrap loop                            │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │  MCP  (streamable HTTP, Bearer token)
                                 │  one endpoint: /mcp
┌────────────────────────────────▼─────────────────────────────────────────┐
│  asdlc-gateway  :8080          MCP facade                                 │
│  · auth + project scoping · tool registry · rate limit · audit            │
│  · aggregates 5 internal domains into one tool surface                    │
└──┬──────────┬──────────┬──────────┬──────────┬───────────────────────────┘
   │          │          │          │          │
┌──▼───────┐ ┌▼────────┐ ┌▼────────┐ ┌▼───────┐ ┌▼──────────────┐
│ core     │ │artifact │ │knowledge│ │ code   │ │ llm-gateway   │
│ :8000    │ │ svc     │ │ svc     │ │ index  │ │ (LiteLLM)     │
│ runs,    │ │ store,  │ │ ingest, │ │ svc    │ │ :4000         │
│ stages,  │ │ version,│ │ retrieve│ │ parse, │ │ cache+budget  │
│ gates,   │ │ prove-  │ │ curate  │ │ embed, │ │               │
│ briefs   │ │ nance   │ │         │ │ symbols│ │               │
└──┬───────┘ └──┬──────┘ └──┬──────┘ └──┬─────┘ └──┬────────────┘
   │            │           │           │          │
   │         ┌──▼──────┐ ┌──▼───────────▼──┐  ┌────▼─────────┐
   │         │ MinIO   │ │ Qdrant  :6333    │  │ embeddings   │
   │         │ :9000   │ │ knowledge / code │  │ TEI :8081 or │
   │         │ blobs   │ │ collections      │  │ Ollama:11434 │
   │         └─────────┘ └──────────────────┘  └──────────────┘
   │
┌──▼──────────────────┐  ┌───────────────┐  ┌──────────────────┐
│ Postgres :5432      │  │ Redis :6379   │  │ approvals-ui     │
│ state, metadata,    │  │ cache, queues │  │ :3000            │
│ provenance, BM25    │  │               │  │                  │
└─────────────────────┘  └───────────────┘  └──────────────────┘
              ┌──────────────────────────┐
              │ worker (arq)             │  ingestion, indexing,
              │ no exposed port          │  git sync, attestations
              └──────────────────────────┘
```

---

## The stage brief — how client-side agents stay centrally controlled

This is the mechanism that makes the whole design work. Read it carefully; everything else follows.

### The problem it solves

If agent prompts live in `CLAUDE.md` / `.cursorrules` / `copilot-instructions.md`, then:
- fixing a prompt means every user reinstalls the plugin;
- the four files drift;
- prompts can't be conditioned on run state, prior artifacts, or retrieved context.

### The solution

The tool-native files contain only a **bootstrap loop** (~40 lines, identical in spirit across tools):

> You are operating inside an ASDLC run. Never work from memory of these instructions alone.
> 1. Determine the current run: `sdlc_run_current`.
> 2. Fetch your instructions: `sdlc_stage_start(run_id, stage)`.
> 3. Follow the returned brief exactly. It supersedes anything in this file.
> 4. Produce the artifacts named in `output_contract` via `artifact_put`.
> 5. Call `sdlc_stage_submit`. Stop. Do not begin the next stage.

The server responds with the real payload:

```jsonc
// sdlc_stage_start(run_id="run_01J...", stage="design") →
{
  "brief_id": "brf_01J8X...",
  "run": { "id": "run_01J...", "title": "Add SSO via OIDC", "project": "acme-api" },
  "stage": "design",
  "agent_role": "architect",
  "prompt_version": "architect@2.3.1",

  "system_prompt": "You are the Architect for this run. ...",   // full role prompt

  "inputs": [                                    // prior APPROVED artifacts only
    { "artifact_id": "art_...", "type": "prd",       "name": "PRD — SSO",
      "uri": "asdlc://artifact/art_...", "inline": "# PRD\n..." },
    { "artifact_id": "art_...", "type": "task_graph", "name": "Plan v2",
      "uri": "asdlc://artifact/art_...", "inline": "..." }
  ],

  "context": {                                   // pre-retrieved, budgeted
    "knowledge": [ { "id": "kn_...", "title": "ADR-014: auth boundaries",
                     "score": 0.83, "text": "..." } ],
    "code":      [ { "path": "src/auth/session.ts", "symbol": "createSession",
                     "lines": [40, 96], "score": 0.79, "text": "..." } ],
    "budget_used_tokens": 6200,
    "budget_max_tokens": 12000
  },

  "output_contract": [
    { "type": "adr",           "min": 1, "max": 5, "format": "markdown",
      "schema_ref": "asdlc://schema/adr@1" },
    { "type": "api_contract",  "min": 1, "max": 1, "format": "openapi-3.1" },
    { "type": "component_diagram", "min": 1, "max": 1, "format": "mermaid" }
  ],

  "acceptance_criteria": [
    "Every FR-* in the PRD maps to at least one component",
    "Each ADR states context, decision, consequences, and rejected alternatives",
    "The API contract validates against OpenAPI 3.1"
  ],

  "tools_allowed": ["knowledge_search", "code_search", "code_symbol",
                    "artifact_get", "artifact_put", "sdlc_stage_submit"],
  "gate": { "required": true, "policy": "block", "approvers": ["@namz"] }
}
```

### What this buys you

| Property | How |
|---|---|
| Fix a prompt once | Bump `architect@2.3.1`; every tool gets it on the next `stage_start`. |
| Identical behaviour across tools | Same brief, same contract, regardless of client. |
| Context is *budgeted*, not dumped | Server retrieves and truncates to `budget_max_tokens` before the client ever sees it. This is the single biggest client-side token saving. |
| Prompts are versioned and auditable | `prompt_version` is recorded on every artifact produced. |
| A/B testing prompts | Server can serve `architect@2.3.1` to half of runs. |
| Cache-friendly ordering | Stable blocks (role prompt, contract) first, volatile blocks (retrieved context) last → maximises client-side prompt-cache hits. See [06](06-llm-cache-and-gateway.md#l0--prompt-prefix-caching). |

### Failure mode to design against

A client can ignore the brief — nothing forces an LLM to obey. Mitigation is **contract enforcement at write time**, not trust: `artifact_put` validates against `output_contract` (type allowed? schema valid? required sections present?) and rejects non-conforming artifacts with actionable errors. `sdlc_stage_submit` refuses if the contract isn't satisfied. The server is the authority; the prompt is just a strong hint.

---

## Run state machine

```
                   ┌──────────────────────────────────────────┐
                   │            RUN: created                   │
                   └───────────────────┬──────────────────────┘
                                       │ sdlc_run_start
  ┌────────────────────────────────────▼───────────────────────────────────┐
  │  for each stage in [plan, design, develop, review, test, document]:     │
  │                                                                         │
  │   pending ──stage_start──▶ in_progress ──stage_submit──▶ awaiting_gate  │
  │      ▲                          │                             │         │
  │      │                          │ stage_abandon               │         │
  │      │                          ▼                             │         │
  │      │                      abandoned                          │        │
  │      │                                                         │        │
  │      │   ◀── changes_requested ──────────────────────────────┤        │
  │      │        (re-enters in_progress with reviewer feedback)   │        │
  │      │                                                         ▼        │
  │      │                                              ┌──────────────────┐│
  │      └──────────────── rejected ◀───────────────────│  GATE (human)    ││
  │                        (run halts)                   │ approve /        ││
  │                                                      │ changes / reject ││
  │                                              ┌───────┴──────────────────┘│
  │                                              │ approved                  │
  │                                              ▼                           │
  │                                     stage complete → next stage           │
  └─────────────────────────────────────────────────────────────────────────┘
                                       │ all stages approved
                                       ▼
                                   RUN: completed
```

**Stage skipping.** `review` and `test` can be marked `optional` in project policy for trivial runs; `plan` and a gate on `develop` cannot be skipped. Skips are recorded as `gate_events` with `decision=skipped` and a required reason.

**Parallelism.** `develop` may fan out into parallel *tasks* (one per node in the task graph), each producing its own artifacts, converging at a single stage gate. Stages themselves are strictly sequential in v1.

**Re-entrancy.** `changes_requested` returns the stage to `in_progress` and the next `stage_start` includes the reviewer's feedback plus the rejected artifact as `inputs`, so the agent iterates rather than restarts.

---

## Services

| Service | Image / stack | Port | Responsibility |
|---|---|---|---|
| `asdlc-gateway` | Python 3.12, FastMCP | 8080 | Single MCP endpoint. Authn, project scoping, tool registry, per-tool rate limits, audit log. Stateless. |
| `asdlc-core` | FastAPI + SQLAlchemy | 8000 (internal) | Runs, stages, gates, briefs, prompt registry, contract validation, policy. |
| `artifact-svc` | FastAPI | 8000 (internal) | Artifact CRUD, content addressing, versioning, provenance DAG, attestations, git mirror. |
| `knowledge-svc` | FastAPI | 8000 (internal) | Knowledge ingestion, chunking, hybrid retrieval, curation workflow. |
| `indexer-svc` | FastAPI + tree-sitter | 8000 (internal) | Repo scan, AST chunking, embedding, symbol graph, incremental sync. |
| `worker` | arq (Redis-backed) | — | Async jobs: ingest, index, embed, attest, git push, knowledge promotion. |
| `llm-gateway` | LiteLLM proxy | 4000 | Model routing, exact + semantic response cache, virtual keys, budgets, spend logs. |
| `approvals-ui` | Next.js | 3000 | Gate dashboard, artifact diff viewer, run timeline, knowledge curation. |
| `postgres` | postgres:17 | 5432 | Relational state + full-text (`tsvector`) BM25 half of hybrid search. |
| `qdrant` | qdrant/qdrant | 6333/6334 | Dense vectors: `knowledge`, `code` collections + semantic cache collection. |
| `minio` | minio/minio | 9000/9001 | Content-addressed blob store for artifacts. |
| `redis` | redis:7 | 6379 | Job queue, exact-match caches, rate-limit counters, LiteLLM cross-pod state. |
| `embeddings` | HF text-embeddings-inference | 8081 | Local embedding server (default). Swappable for Ollama or a hosted API. |
| `ollama` | ollama/ollama | 11434 | *Profile `local-llm`.* Local generation for air-gapped server-side inference. |
| `otel-collector` + `grafana` | — | 4317/3001 | *Profile `observability`.* Traces, spend dashboards. |

Only `8080`, `3000`, `9001`, `4000` and (optionally) `3001` need to leave the host. Everything else stays on the compose network.

### Why one MCP gateway instead of five MCP servers

Users would otherwise configure five servers in four tools = twenty config entries. One endpoint means one entry per tool. The gateway also gives a single place to enforce project scoping and to keep the tool count low — clients degrade badly past ~40 tools, so the gateway exposes a curated ~23 and hides internal RPCs.

The MCP `2026-07-28` spec made HTTP transport **stateless by default** and introduced Multi Round-Trip Requests, replacing the held-open-stream model for elicitation. This suits a horizontally-scalable stateless gateway — build against streamable HTTP, not the deprecated HTTP+SSE transport.

### Why three stores

| Concern | Store | Why not Postgres alone |
|---|---|---|
| Relational state, transactions, audit | Postgres | — |
| Dense vector search over ~10⁵–10⁷ chunks | Qdrant | pgvector works to ~10⁶ but payload-filtered HNSW at multi-project scale, named vectors (code vs. docstring), quantisation, and snapshot-per-collection are materially better in Qdrant. Also lets you scale vectors independently. |
| Artifact blobs (diffs, PDFs, images, tarballs) | MinIO | `bytea` in Postgres wrecks backup/restore times and WAL volume. S3 API means a one-line move to real S3 later. |

**If you'd rather start smaller:** Phase 0 can run Postgres + pgvector only, with MinIO replaced by a local volume. The service interfaces (`VectorStore`, `BlobStore`) are abstract precisely so this swap is a config change. Documented in [12 — Roadmap](12-roadmap.md).

---

## Data flow — one complete stage

```
1.  Dev in Cursor:  "/asdlc start 'Add SSO via OIDC'"
      → sdlc_run_create(project, title, intent)         → core → Postgres
2.  Bootstrap rule fires: sdlc_stage_start(run, "plan")
      → core builds brief:
          · loads planner@1.4.0 prompt from prompt registry
          · loads approved inputs (none, first stage) from artifact-svc
          · knowledge_search(intent) → knowledge-svc → Qdrant + Postgres BM25 → RRF
          · code_search(intent)      → indexer-svc  → Qdrant
          · truncates to token budget, orders stable-first
      → returns brief
3.  Cursor's LLM produces a PRD + task graph.
4.  artifact_put(type="prd", ...) ×2
      → artifact-svc: sha256 → MinIO; metadata + provenance → Postgres
      → contract validation against output_contract
5.  sdlc_stage_submit(run, "plan")
      → core: contract satisfied? → stage=awaiting_gate
      → notify: web UI card + (optional) Slack + PR comment
6.  Human opens approvals-ui :3000, reads rendered PRD, clicks Approve.
      → gate_event(decision=approved, by=namz, channel=web)
      → artifacts flip draft → approved
      → worker: mirror approved artifacts to git branch asdlc/run_01J.../
      → stage plan = complete, design = pending
7.  Repeat for design → develop → review → test → document.
8.  On run completion: worker promotes durable learnings into knowledge center
    as `candidate` entries for later curation.
```

---

## Technology choices, with the honest tradeoff

| Choice | Alternative considered | Why this one |
|---|---|---|
| MCP streamable HTTP, stateless | stdio per-tool servers | stdio can't be shared across four tools or reached from CI. Stateless HTTP matches the 2026-07-28 spec direction. |
| FastMCP (Python) | TS MCP SDK | The rest of the stack (tree-sitter bindings, embedding clients, arq) is Python. One language for the backend. |
| LiteLLM | Portkey, custom proxy | OSS, self-hostable, has Redis exact + semantic caching, virtual keys with `max_budget`, per-key TPM/RPM, and spend logs out of the box. No vendor lock. |
| tree-sitter | LSP-based indexing | LSP requires a running language server per language and a compiling project. tree-sitter parses broken/partial code and covers 40+ languages from one process. Tradeoff: no type resolution — mitigated by a lightweight symbol graph. |
| arq | Celery | Redis-only, async-native, ~1/10th the config. Celery's extra features aren't needed here. |
| Next.js for approvals-ui | Server-rendered Jinja | Diff viewers, mermaid rendering, and live gate updates are genuinely easier here. Isolated service; swap freely. |

---

## Sources

- [The 2026-07-28 MCP Specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [MCP 2026-07-28 spec: what changed, what breaks](https://stacktr.ee/blog/mcp-2026-spec-changes)
- [MCP goes stateless in the 2026-07-28 specification — Appwrite](https://appwrite.io/blog/post/mcp-goes-stateless-in-the-2026-07-28-specification)
