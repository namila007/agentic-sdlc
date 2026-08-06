# M4 — Gateway, Cache & Cost

**Weeks 14–16** · [← Master plan](MILESTONES.md) · Implements [`docs/06-llm-cache-and-gateway.md`](../docs/06-llm-cache-and-gateway.md)

> **Goal:** measurable cost reduction, and cost visibility.
>
> You can answer "which stage costs the most?" from a dashboard.

---

## Read this before building anything here

**The gateway cannot see the tokens Claude Code or Cursor spend.** Those calls go directly to
Anthropic or OpenAI on the user's own subscription (decision **D1**). They are the vast majority of
tokens in the system, and no amount of response caching on our side touches them.

```
                    WHAT THE GATEWAY SEES              WHAT IT DOES NOT SEE
┌────────────────────────────────────┐  ┌────────────────────────────────────┐
│ · embeddings (indexing, ingestion, │  │ · Claude Code → Anthropic          │
│   every search query)              │  │ · Cursor → Anthropic/OpenAI        │
│ · knowledge summarisation at ingest│  │ · Copilot → GitHub                 │
│ · docstring generation             │  │                                    │
│ · staleness / contradiction judging│  │   the user's own subscription      │
│ · artifact linting & traceability  │  │   the bulk of all tokens spent     │
│ · rerank scoring                   │  │                                    │
│ · headless CI runs                 │  │                                    │
│ · opt-in BYO-key routing           │  │                                    │
└────────────────────────────────────┘  └────────────────────────────────────┘
        L1 + L2 save real money here          L0 + L3 are the only levers here
```

So the milestone splits cleanly:

| Tier | Saves on | Mechanism |
|---|---|---|
| **L0** prompt-prefix cache | **Client-side** | Provider-side caching, earned by deterministic brief ordering |
| **L1** exact response cache | Server-side | Redis, keyed on the exact request |
| **L2** semantic response cache | Server-side | Redis-semantic, restricted allow-list |
| **L3** retrieval cache | **Both** | Built in M2.7; extended to knowledge here |

**Be honest about this when justifying the project.** The value of ASDLC is workflow and
traceability. Cost reduction is real but secondary, and the client-side portion comes from serving
small, well-ordered briefs — not from a response cache.

---

## Scope

| In | Out |
|---|---|
| LiteLLM with Postgres + Redis | Server-side agent execution for SDLC work (contradicts D1) |
| Virtual keys per project/role; budgets and alerts | Multi-model consensus / debate agents |
| L1 exact cache (embeddings first) | Fine-tuned models |
| L2 semantic cache, **allow-list only** | |
| L0 discipline in the brief builder + `--explain-cache` | |
| Spend dashboard: cost per run, stage, role | |

---

## Sub-milestones

| ID | Name | Depends on | Days |
|---|---|---|---|
| [M4.1](#m41--litellm-deployment-routing-fallbacks) | LiteLLM deployment, routing, fallbacks | M3.2 | 2 |
| [M4.2](#m42--virtual-keys-budgets-spend-attribution) | Virtual keys, budgets, spend attribution | M4.1 | 2 |
| [M4.3](#m43--l1-exact-cache) | L1 exact cache | M4.1 | 1.5 |
| [M4.4](#m44--l2-semantic-cache-with-allow-list) | **L2 semantic cache with allow-list** | M4.3 | 2.5 |
| [M4.5](#m45--l0-brief-cache-discipline) | L0 brief cache discipline & `--explain-cache` | M0.3 | 2.5 |
| [M4.6](#m46--spend-dashboard-and-metrics) | Spend dashboard & metrics | M4.2 | 2.5 |

---

## M4.1 — LiteLLM deployment, routing, fallbacks

**Delivers:** every server-side inference call routes through one proxy, and swapping to local models
is a config change.

### Service

| Service | Image | Port | Depends on |
|---|---|---|---|
| `llm-gateway` | `ghcr.io/berriai/litellm:main-stable` | 4000 (bind to 127.0.0.1) | postgres, redis (healthy) |

Requires a **second Postgres database** (`litellm`), created by `ops/init-db.sql`. Without
`DATABASE_URL` there are no virtual keys, no spend tracking and no admin UI — so it is not optional.

### Config

```yaml
# config/litellm.yaml
model_list:
  - model_name: default
    litellm_params:
      model: anthropic/claude-opus-5
      api_key: "os.environ/ANTHROPIC_API_KEY"

  - model_name: embed
    litellm_params:
      model: openai/BAAI/bge-m3        # openai/ prefix = OpenAI-compatible endpoint
      api_base: http://embeddings:8081/v1
      api_key: none
    model_info: { cache_ttl: 0 }        # embeddings are deterministic — never expire

  - model_name: embed-local
    litellm_params:
      model: openai/nomic-embed-text
      api_base: http://ollama:11434/v1
      api_key: none

  - model_name: judge                   # staleness / contradiction judging — cheap model
    litellm_params:
      model: anthropic/claude-haiku-4-5-20251001
      api_key: "os.environ/ANTHROPIC_API_KEY"

  - model_name: artifact-gen            # NO CACHE — see M4.4
    litellm_params:
      model: anthropic/claude-opus-5
      api_key: "os.environ/ANTHROPIC_API_KEY"

router_settings:
  routing_strategy: usage-based-routing-v2
  num_retries: 3
  timeout: 600
  fallbacks:
    - { default: [claude-sonnet-5] }
    - { embed:   [embed-local] }        # TEI down → Ollama
  context_window_fallbacks:
    - { claude-sonnet-5: [claude-opus-5] }

general_settings:
  master_key: "os.environ/LITELLM_MASTER_KEY"
  database_url: "os.environ/LITELLM_DATABASE_URL"
```

Verify key names against the installed LiteLLM version before deploying — the proxy config surface
moves faster than most. See [APPENDIX](APPENDIX-tech-verification.md).

### Why run a gateway at all, given the caching limits

**The indirection.** For air-gapped deployment (M6.5) you replace the entire `model_list` with
Ollama or vLLM endpoints and every service keeps working, because everything speaks the
OpenAI-compatible shape through one place. That alone justifies the service; caching is a bonus.

### Wiring

Every service that performs inference points at `LLM_GATEWAY_URL=http://llm-gateway:4000`:
`knowledge-svc` (summarisation, judging), `indexer-svc` (embeddings, docstrings), `worker`
(attestation-adjacent linting), `core` (artifact linting, traceability judgement).

**Model API keys live only in the LiteLLM container.** No other service holds a provider credential.

### Acceptance

- [ ] All server-side inference routes through the gateway — asserted by a test that blocks direct provider egress
- [ ] Killing TEI causes embeddings to fall back to Ollama without an application error
- [ ] Replacing `model_list` with local endpoints requires no code change
- [ ] No provider key appears in any service env except `llm-gateway`

---

## M4.2 — Virtual keys, budgets, spend attribution

### One key per (project, role)

Generated by `asdlc project init` via the LiteLLM key-management API:

```jsonc
{
  "key_alias":       "acme-api/architect",
  "models":          ["default", "embed"],
  "max_budget":      50,               // USD per budget_duration
  "budget_duration": "30d",
  "tpm_limit":       200000,
  "rpm_limit":       60,
  "metadata": { "project_id": "acme-api", "role": "architect" }
}
```

Budget checks read current spend from a **cross-pod Redis counter**, so enforcement is consistent
across workers and replicas rather than per-process.

### Per-call attribution

Every gateway call carries metadata:

```jsonc
{ "metadata": { "run_id": "run_01J…", "stage": "design", "role": "architect", "artifact_id": "art_…" } }
```

This is what makes the spend log answer *which stage burns money?* In practice `develop` and
`review` dominate.

### Client-side attribution is best-effort

Agents report `metrics.token_cost` on `artifact_put` when the tool exposes it. Treat this as
**indicative, not billing-grade**, and label it that way in the UI. Showing a self-reported number
next to a measured one without distinction is how dashboards start lying.

### Alerts

```yaml
general_settings:
  alerting: ["webhook"]
  alerting_threshold: 300              # slow-request alert, seconds
  budget_alert_thresholds: [0.5, 0.8, 0.95]
```

Webhook → approvals-ui banner, and optionally Slack.

### Acceptance

- [ ] Exceeding a key budget returns `BUDGET_EXCEEDED` with a `fix` naming the project and role
- [ ] Spend is attributable to run, stage and role for a completed run
- [ ] Budget enforcement holds with three worker replicas running concurrently
- [ ] Client-reported costs are visually distinguished from measured costs

---

## M4.3 — L1 exact cache

```yaml
litellm_settings:
  cache: true
  cache_params:
    type: redis
    host: redis
    port: 6379
    namespace: "asdlc:l1"
    ttl: 604800                        # 7 days
    supported_call_types: ["acompletion", "aembedding", "atranscription"]
```

### Key derivation

`hash(model, messages, temperature, tools, response_format, project_id)`

**`project_id` is in the key deliberately.** A cache shared across tenants is a cross-tenant read
primitive, not an optimisation. Same reasoning as the L3 namespace in M2.7 and the Qdrant payload
filter in M3.1 — three places, one rule.

### Where it earns its keep

**Embeddings.** Re-indexing an unchanged chunk, a repeated search string, a re-embed after a restart:
40–70% hit rates in steady state. Set `cache_ttl: 0` (never expire) per-model for embeddings, since
they are deterministic for a given model.

### Restrictions

- Do not cache anything with `temperature > 0` for more than a few minutes
- Never cache tool-calling completions where tool results vary
- Disable per-request with `cache: {"no-cache": true}` — **the brief builder sets this on any call
  that feeds an artifact**

### Acceptance

- [ ] L1 embedding hit rate > 40% in steady state
- [ ] Two projects embedding identical text produce two cache entries, not one
- [ ] Artifact-feeding calls bypass the cache — asserted by a test

---

## M4.4 — L2 semantic cache with allow-list

**This is the only cache tier that can cost correctness rather than money.** Everything below is a
containment strategy.

```yaml
litellm_settings:
  cache: true
  cache_params:
    type: redis-semantic
    host: redis
    port: 6379
    similarity_threshold: 0.95
    redis_semantic_cache_embedding_model: embed
    ttl: 86400                         # 24h — shorter than L1; near-misses are riskier
    namespace: "asdlc:l2"
```

### The allow-list — structural, not remembered

| Workload | L2 | Why |
|---|---|---|
| Knowledge summarisation at ingest | ✅ | Near-duplicate documents are common; a near-miss is harmless |
| Docstring generation | ✅ | Similar functions → similar docs. Low stakes. |
| Staleness / contradiction judging | ✅ | Repetitive by nature |
| Rerank scoring | ✅ | High volume, tolerant of near-misses |
| **Artifact generation** | ❌ | A 0.95-similar prompt is a **different requirement**. A correctness bug disguised as a cache hit. |
| **Code generation** | ❌ | Same reason, worse consequences |
| Anything in a tool-call loop | ❌ | State-dependent; cannot be replayed |

**Enforcement is structural.** L2 is enabled per-model-alias. Artifact-producing calls route to
aliases configured with `cache: false` (the `artifact-gen` alias in M4.1). Nobody has to remember the
rule at the call site — the routing makes it impossible to get wrong.

### Threshold discipline

0.95 is deliberately conservative. Below roughly 0.93 you are serving answers to materially different
questions.

**Do not loosen it without measuring.** Log every semantic hit with both the original and the matched
prompt. Sample 50. Count how many matches were actually acceptable.

### G4 — the gate

> Is the L2 false-positive rate **< 2%** over 50 manually reviewed hits?

**If it is higher, turn L2 off.** Not "tune it" — off. The other tiers being wrong costs money; this
one costs correctness, and a wrong answer that looks right is the worst failure mode this system has.

Schedule the review as a real calendar task, not a "we'll check eventually".

### Acceptance

- [ ] Artifact-generating calls cannot reach L2 — asserted by routing config, not by convention
- [ ] Every semantic hit is logged with original + matched prompt for review
- [ ] 50-sample FP review completed and recorded
- [ ] G4 decision made and documented

---

## M4.5 — L0 brief cache discipline

**The largest client-side saving available**, and the only cost lever that touches Claude Code and
Cursor at all.

### How it works

Anthropic prompt caching and OpenAI automatic prefix caching both key on a **stable prefix**. If the
first N tokens of a request are byte-identical to a previous request, they are served from the
provider's cache at a fraction of the input cost. Multi-turn stages with agent iteration typically
cache blocks 1–4 at 3–6k tokens, hitting cache on every turn after the first.

The mechanism is entirely under our control, because we build the prompt.

### The rules (established in M0.3, enforced here)

```
┌─ block 1  role system prompt              stable per prompt_version   ← cache
├─ block 2  output contract + schemas       stable per prompt_version   ← cache
├─ block 3  project policy + conventions    stable per project          ← cache
├─ block 4  run metadata + prior summaries  stable within a run         ← cache
├─ block 5  input artifacts (approved)      changes per stage
└─ block 6  retrieved knowledge + code      changes per call            ← never cache
```

1. **Never interleave.** One volatile token near the front invalidates every cached block after it.
   This is the rule that gets broken by accident — a timestamp in the policy block, a run ID in the
   role prompt — and the failure is silent.
2. **Byte-stable serialisation.** Sorted JSON keys, `\n` line endings, no timestamps, no
   run-specific IDs in blocks 1–3.
3. **Mark boundaries.** Explicit `cache_control: {"type":"ephemeral"}` breakpoints after blocks 2, 3
   and 4 for Anthropic clients.
4. **Minimum block size.** Blocks 1–4 must exceed the provider's minimum cacheable length or the
   cache never engages at all. Pad with genuinely useful content — never filler.

### `--explain-cache`

```
$ asdlc stage start design --run run_01J… --explain-cache

BLOCK 1  role system prompt         1,240 tok  ✅ stable    sha 9f2c…
BLOCK 2  output contract + schemas    680 tok  ✅ stable    sha 3ab1…
BLOCK 3  project policy               410 tok  ⚠️  UNSTABLE
         └─ line 12 contains a timestamp: "generated 2026-08-06T09:14:22Z"
            → this invalidates blocks 3–6 on every call
BLOCK 4  run metadata                 320 tok  ✅ stable within run
BLOCK 5  input artifacts            2,100 tok  — volatile by design
BLOCK 6  retrieved context          4,800 tok  — volatile by design

cacheable prefix: 2,330 tok (would be 2,740 if block 3 were fixed)
```

This tool is how L0 stays working. Cache-breaking regressions are invisible without it — the system
behaves identically, just more expensively. Wire the stability check into CI so a PR that introduces
a timestamp into block 3 fails.

### Verification

Log `cache_read_input_tokens` versus `cache_creation_input_tokens` where the client reports them.
Target: **`l0_cache_read_ratio` > 0.6 on iterative stages.**

### Acceptance

- [ ] `--explain-cache` correctly identifies an injected instability
- [ ] CI fails if a brief's blocks 1–3 are not byte-stable across two builds
- [ ] `cache_control` breakpoints appear after blocks 2, 3 and 4
- [ ] Measured cache-read ratio > 0.6 on a multi-turn design stage

---

## M4.6 — Spend dashboard and metrics

**Route:** `/spend` in approvals-ui (stubbed in M1.6), plus a Grafana dashboard under the
`observability` profile (M6.7).

### Views

| View | Answers |
|---|---|
| Cost per run | "What did this feature cost?" |
| Cost per stage | "Which stage burns the most?" — usually `develop` and `review` |
| Cost per role | "Is the architect prompt too expensive?" |
| Cache hit rates L0–L3 | "Are the caches working?" |
| Budget vs. actual per project/role | "Are we about to hit a limit?" |

The run timeline (M1.6) gains a per-stage cost column.

### Metrics to ship from day one

| Metric | Source | Target |
|---|---|---|
| `l0_cache_read_ratio` | client-reported, per stage | > 0.6 on iterative stages |
| `l1_hit_rate` (embeddings) | LiteLLM | > 0.4 steady state |
| `l1_hit_rate` (completions) | LiteLLM | > 0.15 |
| `l2_hit_rate` + **FP rate** | LiteLLM + sampled review | hits > 0.1, **FP < 0.02** |
| `l3_hit_rate` | knowledge / indexer svc | > 0.5 |
| `brief_tokens_p95` | core | < `total_max_tokens` |
| `cost_per_run` by stage | LiteLLM spend logs | tracked, not targeted |

`cost_per_run` is **tracked, not targeted** on purpose. Setting a cost target on a system whose main
value is traceability creates pressure to skip stages, which is exactly the wrong optimisation.

**The metric that matters most is the L2 false-positive rate.** Every other metric being wrong costs
money. That one costs correctness.

### Acceptance

- [ ] "Which stage costs the most?" is answerable from the dashboard in one click
- [ ] All seven metrics are emitted and visible
- [ ] Budget-threshold alerts fire at 50 / 80 / 95%
- [ ] The run timeline shows per-stage cost

---

## M4 exit criteria

- [ ] L1 embedding hit rate > 40% in steady state
- [ ] L0 cache-read ratio > 60% on iterative stages where the client reports it
- [ ] **L2 false-positive rate < 2% over a 50-sample manual review — or L2 is off** ← G4
- [ ] "Which stage costs the most?" is answerable from a dashboard
- [ ] Swapping to fully local models is a `config/litellm.yaml` change with no code change

## The honest accounting

Track the cost of **running the stack** (one box, but not free) alongside the token savings. It is
entirely possible that infrastructure cost exceeds token savings, especially for solo use.

That is fine — but say so plainly when justifying the project. The value here is workflow,
traceability and prompt centralisation. Cost reduction is a real secondary benefit, not the headline.
A dashboard that quietly implies otherwise does the project a disservice.
