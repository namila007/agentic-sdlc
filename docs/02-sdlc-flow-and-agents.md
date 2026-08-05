# 02 — SDLC Flow & Agent Specs

Six stages, one agent role each, one gate each.

```
 plan ──▶ design ──▶ develop ──▶ review ──▶ test ──▶ document
  │        │          │           │         │        │
  ▼        ▼          ▼           ▼         ▼        ▼
 PRD      ADRs       diffs/PR   findings  test     docs +
 task     API spec   impl notes review    plan +   changelog
 graph    diagram               report    tests    + knowledge
  │        │          │           │         │        │
 [gate]  [gate]     [gate]      [gate]   [gate]   [gate]
```

---

## Agent role definition format

Every role is one YAML file in `packs/core-sdlc/roles/`. This is the single source of truth — compiled into the prompt registry (served in stage briefs) *and* into tool-native files (Claude subagents, Cursor rules).

```yaml
# packs/core-sdlc/roles/architect.yaml
id: architect
version: 2.3.1
stage: design
title: Architect
summary: Turns an approved PRD and task graph into ADRs, API contracts, and a component design.

model_hints:                     # advisory; the client's own model actually runs
  preferred: [claude-opus-5, gpt-5.1, claude-sonnet-5]
  min_context: 100000
  reasoning: high

system_prompt: |
  You are the Architect for this run...
  (full prompt — see "Prompt authoring rules" below)

inputs:
  required: [prd, task_graph]
  optional: [adr, constraint_doc]

context_policy:
  knowledge:
    enabled: true
    scopes: [project, org]
    types: [adr, convention, domain_glossary, constraint]
    top_k: 12
    max_tokens: 5000
  code:
    enabled: true
    top_k: 15
    max_tokens: 5000
    prefer: [interfaces, module_boundaries, config]
  total_max_tokens: 12000

output_contract:
  - type: adr
    min: 1
    max: 8
    format: markdown
    schema_ref: asdlc://schema/adr@1
  - type: api_contract
    min: 0
    max: 3
    format: openapi-3.1
  - type: component_diagram
    min: 1
    max: 1
    format: mermaid

acceptance_criteria:
  - id: AC1
    text: Every FR-* in the PRD is traceable to at least one component.
    check: automated          # core runs a traceability check
  - id: AC2
    text: Each ADR contains Context, Decision, Consequences, Alternatives Rejected.
    check: automated          # schema validation
  - id: AC3
    text: No component introduces a new external dependency without an ADR.
    check: human

tools_allowed:
  - knowledge_search
  - knowledge_get
  - code_search
  - code_symbol
  - code_file_outline
  - artifact_get
  - artifact_put
  - sdlc_stage_submit

gate:
  required: true
  policy: block                  # auto | notify | block
  approver_roles: [tech_lead, architect]

handoff:
  next_stage: develop
  emit_summary: true             # 400-token digest carried forward
```

### Prompt authoring rules

Enforced by a linter in CI (`asdlc lint roles/`):

1. **Second person, imperative.** "You produce…" not "The architect should…".
2. **State the stop condition explicitly.** Every prompt ends with: *"When the output contract is satisfied, call `sdlc_stage_submit` and stop. Do not begin the next stage."*
3. **No hardcoded project facts.** Anything project-specific comes from `context`, never the prompt — otherwise the role isn't reusable.
4. **Stable prefix.** The first ~80% of the prompt must be identical across runs (role, method, contract). Volatile content goes in `context`, which the brief places last. This is what makes client-side prompt caching hit.
5. **Failure instructions.** Every prompt says what to do when inputs are insufficient: emit a `blocker` artifact and submit early rather than inventing requirements.
6. **Max 1,500 tokens.** If a role prompt exceeds this, it's doing two jobs — split it.

---

## Stage 1 — Plan

**Role:** `planner` · **Consumes:** raw intent, tickets, prior runs · **Gate policy:** block

Takes an unstructured request ("add SSO") and produces a specification a machine and a human can both agree on.

**Artifacts produced**

| Type | Format | Notes |
|---|---|---|
| `prd` | markdown | Problem, goals, non-goals, functional requirements `FR-1…n`, non-functional `NFR-1…n`, out of scope, open questions. |
| `task_graph` | JSON | DAG of tasks with `id`, `title`, `depends_on[]`, `estimate`, `fr_refs[]`, `risk`. |
| `blocker` *(conditional)* | markdown | Emitted instead of a PRD when the request is under-specified. Forces a human clarification loop before spending more tokens. |

**Acceptance criteria**
- Every FR is atomic, testable, and numbered.
- Every task in the graph references ≥1 FR (traceability, checked automatically).
- The graph is acyclic and every task is reachable (checked automatically).
- Open questions are listed explicitly rather than silently assumed.

**Design note.** The `blocker` artifact matters more than it looks. The most expensive failure mode in agentic SDLC is an agent confidently inventing requirements and five stages of work being built on them. Making "I don't have enough information" a first-class, cheap, *rewarded* output prevents that.

---

## Stage 2 — Design

**Role:** `architect` · **Consumes:** approved `prd`, `task_graph` · **Gate policy:** block

Full spec above.

**Artifacts:** `adr` (1–8), `api_contract` (OpenAPI 3.1 / protobuf / GraphQL SDL), `component_diagram` (mermaid), optional `data_model` (DBML or SQL DDL).

**Automated checks at `artifact_put`:** OpenAPI validates against 3.1 schema; mermaid parses; ADR has all four required sections; FR→component traceability matrix is complete.

**Knowledge write-back.** Approved ADRs are automatically ingested into the knowledge center at `scope=project, type=adr, status=approved`. This is the main way the system gets smarter across runs — run #47's architect retrieves run #12's decisions.

---

## Stage 3 — Develop

**Role:** `implementer` · **Consumes:** approved `task_graph`, `adr`, `api_contract` · **Gate policy:** block

The only stage that fans out. The client works task-by-task through the graph.

**Artifacts**

| Type | Format | Notes |
|---|---|---|
| `code_change` | unified diff or git ref | One per task node. Stores the diff + the branch/commit if pushed. |
| `impl_note` | markdown | Deviations from the design and *why*. High-value knowledge input. |
| `dependency_change` | JSON | New/updated/removed deps with justification. Feeds security review. |

**Acceptance criteria**
- Every task node is `done`, `deferred` (with reason), or `blocked` (with reason).
- No `code_change` contradicts an ADR without a corresponding `impl_note`.
- Project build/lint command exits 0 (run client-side; result reported via `sdlc_check_report`).

**Where the code actually lives.** The agent edits files in the developer's working tree using its own native file tools — the control plane never writes source. `artifact_put(type="code_change")` records the *diff and its provenance*, not the working copy. On approval, the worker can open a PR from the recorded branch.

**Parallel fan-out.** `sdlc_task_claim(run_id, task_id)` / `sdlc_task_complete` let multiple agents (or one agent in sequence) work the graph without collision. Claims carry a lease TTL so a dead session doesn't deadlock a task.

---

## Stage 4 — Review

**Role:** `reviewer` · **Consumes:** approved `code_change[]`, `adr[]`, `prd` · **Gate policy:** block

Deliberately runs **before** test authoring, so tests are written against reviewed code rather than being rewritten after review changes it.

**Artifacts**

| Type | Format | Notes |
|---|---|---|
| `review_report` | markdown + JSON findings | Findings with `severity` (blocker/major/minor/nit), `file`, `line`, `rationale`, `suggested_fix`. |
| `security_findings` | JSON (SARIF-compatible) | Separate stream so it can feed existing security tooling. |

**Review checklist is data, not prose.** `packs/core-sdlc/checklists/review.yaml` holds the checklist; the brief injects it. Teams fork the checklist without touching the prompt — this is the main customisation point for most orgs.

**Loop rule.** If any finding is `severity=blocker`, the reviewer's `sdlc_stage_submit` automatically requests changes on the *develop* stage rather than opening a review gate. Bounded: `max_auto_loops: 2` in project policy, then it escalates to a human regardless.

---

## Stage 5 — Test

**Role:** `test-engineer` · **Consumes:** approved `prd`, `api_contract`, `code_change[]`, `review_report` · **Gate policy:** block

**Artifacts**

| Type | Format | Notes |
|---|---|---|
| `test_plan` | markdown | Strategy, scope, risk-based prioritisation, what is deliberately *not* tested. |
| `test_case` | JSON (Gherkin-ish) | `id`, `fr_refs[]`, `preconditions`, `steps`, `expected`, `level` (unit/integration/e2e), `automated` (bool). |
| `test_code` | diff | Actual test implementation. |
| `test_result` | JSON | Run output: pass/fail/skip counts, failures with traces. |

**Acceptance criteria**
- Every `FR-*` has ≥1 `test_case` (automated traceability check — this is the highest-value automated gate in the whole flow).
- Every `blocker`/`major` review finding has a regression test.
- `test_result` shows a green run, or failures are explicitly accepted with reasons.

**Design note.** Test cases are structured data, not prose, specifically so the FR→test coverage matrix can be computed rather than eyeballed. That matrix is the single most useful thing to show a human at the gate.

---

## Stage 6 — Document

**Role:** `technical-writer` · **Consumes:** everything approved · **Gate policy:** notify (not block, by default)

**Artifacts**

| Type | Format | Notes |
|---|---|---|
| `doc_page` | markdown | User-facing or developer-facing docs, ready to drop into the docs site. |
| `changelog_entry` | markdown | Keep-a-Changelog format. |
| `runbook` *(conditional)* | markdown | If the change introduces an operational surface. |
| `knowledge_candidate` | JSON | **The critical one** — see below. |

### Knowledge harvest

The writer's last job is to propose what this run should teach the *next* run:

```jsonc
{
  "type": "knowledge_candidate",
  "entries": [
    { "scope": "project", "kind": "convention",
      "title": "OIDC state params are stored in Redis, not cookies",
      "body": "...", "evidence": ["art_adr_014", "art_impl_note_3"],
      "confidence": 0.9, "supersedes": null },
    { "scope": "project", "kind": "gotcha",
      "title": "Our reverse proxy strips Authorization on 302 redirects",
      "body": "...", "evidence": ["art_review_report_1"], "confidence": 0.95 }
  ]
}
```

These land as `status=candidate` in the knowledge center. A human promotes them to `approved` in the curation UI. **Never auto-approve knowledge** — an unreviewed wrong "fact" poisons every future run's retrieval, and it's very hard to notice. See [04](04-knowledge-center.md#curation).

---

## Stage summary table

| Stage | Role | Key inputs | Key outputs | Gate | Auto-checks |
|---|---|---|---|---|---|
| plan | planner | intent, tickets | prd, task_graph | block | FR atomicity, DAG validity, traceability |
| design | architect | prd, task_graph | adr, api_contract, diagram | block | OpenAPI schema, ADR sections, FR→component |
| develop | implementer | task_graph, adr, api_contract | code_change, impl_note | block | build/lint exit 0, task completeness |
| review | reviewer | code_change, adr | review_report, security_findings | block | SARIF schema, blocker→loop rule |
| test | test-engineer | prd, api_contract, code_change | test_plan, test_case, test_code, test_result | block | FR→test coverage, green run |
| document | technical-writer | all approved | doc_page, changelog, knowledge_candidate | notify | changelog format, link validity |

---

## Adding your own stages

The stage list is project policy, not code:

```yaml
# projects/acme-api/policy.yaml
stages:
  - plan
  - design
  - threat_model      # custom, from packs/security/roles/threat-modeler.yaml
  - develop
  - review
  - test
  - document
  - release           # custom
```

A custom stage needs one role YAML and any new artifact type schemas. `asdlc validate policy` checks that every stage's `inputs.required` can actually be satisfied by an earlier stage's `output_contract` — catching mis-ordered pipelines before they run.
