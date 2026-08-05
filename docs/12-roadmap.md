# 12 — Roadmap

Seven phases. Each ends with something usable, not a half-built layer.

**Sequencing principle:** build the thinnest thing that proves the stage-brief loop works end to end, *then* make each component good. The biggest risk in this project isn't any single component — it's discovering after three months that agents in Cursor ignore the brief. Find that out in week two.

---

## Phase 0 — Walking skeleton (week 1–2)

**Goal:** one stage, one tool, one gate, end to end. Prove the mechanism.

Build:
- `core` with a hardcoded `planner` role and a hardcoded brief (no retrieval yet).
- `gateway` exposing 5 tools: `sdlc_run_create`, `sdlc_stage_start`, `artifact_put`, `sdlc_stage_submit`, `gate_status`.
- Postgres only. Artifacts as `TEXT` columns. No MinIO, no Qdrant, no cache, no auth.
- Gate approval via `asdlc gate approve` on the CLI.
- Manual `.mcp.json` config in one repo.

Cut deliberately: retrieval, versioning, provenance, UI, multi-tool.

**Exit criteria**
- In Claude Code *and* in Cursor: `/asdlc start "…"` → brief arrives → PRD written via `artifact_put` → submit → CLI approve → stage marked complete.
- The agent stops at the gate without being told to, in both tools.

**The question this phase answers:** do client-side agents reliably fetch and obey a server-served brief? If the answer is "only in Claude Code," the design needs rework *now*, not later.

---

## Phase 1 — Full pipeline, thin components (week 3–6)

**Goal:** all six stages, real artifact storage, a usable approval surface.

Build:
- All six roles as YAML + the prompt registry + `asdlc lint roles/`.
- MinIO, content addressing, versioning, `is_head`, provenance edges.
- Contract validation on `artifact_put` with the `fix` field on every error.
- Approvals UI: queue + gate detail + markdown/mermaid/diff rendering.
- Git mirror on approval.
- `token` auth mode; `project_id` on everything from the first migration.
- `asdlc` CLI: init, run, artifact, gate, doctor.

Cut deliberately: knowledge center, code index, caching, semantic anything.

**Exit criteria**
- A real feature goes plan → document with six human gates and produces a mergeable PR.
- `asdlc import` rebuilds control-plane state from the git mirror.
- Total human time at gates for a small feature < 20 minutes.

**Watch for:** gate fatigue. If six gates feels like too many on the first real feature, that's data — consider `notify` policy on `document` and `test` before adding more machinery.

---

## Phase 2 — Code indexer (week 7–9)

**Goal:** agents stop reading whole directories.

Build:
- Qdrant + TEI. tree-sitter for your top 5 languages first, not all 40.
- AST chunking, header enrichment, chunk-hash incremental sync.
- Symbol table + heuristic refs.
- `code_search`, `code_symbol`, `code_file_outline`, `code_refs`, `code_index_status`.
- L3 retrieval cache keyed with `git_sha`.
- Wire code context into the brief with a token budget.

**Exit criteria**
- 200k LOC repo indexes in < 10 min; incremental commit < 3 s; search p95 < 150 ms.
- Measured: brief tokens down, and agents call `code_file_outline` before reading.
- A blind comparison on 20 real "where is X" questions beats grep on relevance.

---

## Phase 3 — Knowledge center (week 10–13)

**Goal:** run #20 is measurably better than run #1.

Build:
- `knowledge_entries` + chunks + Qdrant collection with scope filters.
- Hybrid retrieval: dense + Postgres BM25 + RRF + MMR. Rerank behind a flag.
- Auto-ingest of approved ADRs.
- `knowledge_candidate` harvest at the document stage.
- Curation UI with nearest-neighbour duplicate detection.
- `asdlc project init` bootstrap.
- Staleness worker.

**Exit criteria**
- A run demonstrably reuses a decision from an earlier run (traceable via `contextSources` in the attestation).
- Curation load < 10 candidates/week for an active project — if it's more, the harvest prompt is too eager.
- Retrieval precision@5 ≥ 0.7 on a hand-labelled set of 30 project questions.

---

## Phase 4 — Cache & gateway (week 14–16)

**Goal:** measurable cost reduction, and cost visibility.

Build:
- LiteLLM with Postgres + Redis; virtual keys per project/role; budgets and alerts.
- L1 exact cache (embeddings first — that's where the volume is).
- L2 semantic cache, **enabled only on the safe workload list** in [06](06-llm-cache-and-gateway.md#where-to-use-it--and-where-not-to).
- L0 discipline in the brief builder: block ordering, byte-stable serialisation, `cache_control` breakpoints, `--explain-cache`.
- Spend dashboard: cost per run, per stage, per role.

**Exit criteria**
- L1 embedding hit rate > 40% in steady state.
- L0 cache-read ratio > 60% on iterative stages where the client reports it.
- L2 false-positive rate < 2% on a 50-sample manual review. **If it's higher, turn L2 off** — it is the only tier that can cost correctness.
- You can answer "which stage costs the most?" from a dashboard.

---

## Phase 5 — Packaging & distribution (week 17–19)

**Goal:** someone who isn't you can install and use this.

Build:
- `asdlc build` compiling packs → Claude plugin / Cursor `.mdc` / Copilot / AGENTS.md.
- Marketplace repo, versioning, `min_server_version` check.
- `asdlc init` writing bridges into a user repo.
- Getting-started docs and a 5-minute demo.
- Git/PR approval channel + Slack notifier.

**Exit criteria**
- A colleague goes from zero to a completed run in under 30 minutes using only the docs.
- Identical behaviour verified across Claude Code, Cursor, and Copilot on the same run.
- A prompt fix ships to all users with no reinstall.

---

## Phase 6 — Hardening (week 20+)

- OIDC auth, RLS policies applied and tested, egress allow-list.
- Secret scanning on ingest/index/artifact paths.
- Attestation signing.
- Air-gap export/import bundle, tested on a genuinely disconnected host.
- Backup/restore drill.
- Observability profile with the three dashboards.
- Load test: 10 concurrent runs, 5 repos, 1M LOC.

---

## Ongoing, from Phase 1 onward

**Prompt evaluation.** Build a small eval harness early: 15–20 fixture runs with known-good artifacts, scored on contract compliance, acceptance-criteria pass rate, and human-rated quality. Run it on every prompt version bump. Without this, prompt changes are superstition — and you will make a lot of prompt changes.

---

## Deliberate deferrals

| Deferred | Why | Revisit when |
|---|---|---|
| Kubernetes | Compose covers single-host; k8s is weeks of yak-shaving | > 20 concurrent users |
| Server-side agent execution | Contradicts [D1](00-overview.md#d1--agents-are-client-side) | Headless CI runs become a primary use case |
| LSP enrichment of the symbol graph | tree-sitter heuristics are good enough to start | Ref accuracy becomes a real complaint |
| Jira/Confluence connectors | Ingest quality is the bottleneck, not source count | Knowledge center is proven |
| Multi-model consensus / debate agents | Large cost, uncertain benefit | You've exhausted single-agent prompt quality |
| Fine-tuned models | Very large effort | Never, probably |

---

## Where this most likely goes wrong

Worth naming up front, since each has a cheap early check:

1. **Agents ignore the brief.** → Phase 0 tests this in both tools before anything else is built. Mitigation is server-side contract enforcement, not better prompting.
2. **Gate fatigue.** Six gates per feature is a lot. → Measure human time at gates from Phase 1. Be ready to move `test` and `document` to `notify`.
3. **Knowledge center becomes noise.** → Hard curation, precision@5 as an exit criterion, staleness worker from day one.
4. **Semantic cache serves a wrong answer that looks right.** → Restricted workload list, 0.95 threshold, sampled FP review as a Phase 4 exit gate.
5. **The compile-to-four-tools step rots.** → Golden-file tests on `asdlc build` output; CI fails if the generated bootstrap drifts from the pack.
6. **Cost of running the stack exceeds the token savings.** → It's one box; but track it. The value here is workflow and traceability, not primarily cost reduction — be honest about that when justifying it.
