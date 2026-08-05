# 06 — LLM Cache & Gateway

Four cache tiers plus budget enforcement. Read the scope section first — it determines which tiers can help you at all.

---

## What the gateway can and cannot see

This is the most important paragraph in the document, and it follows directly from [D1: agents are client-side](00-overview.md#d1--agents-are-client-side).

```
┌──────────────────────────────────────────────────────────────┐
│  Claude Code / Cursor / Copilot                              │
│  ─ calls Anthropic/OpenAI DIRECTLY with the user's own       │
│    subscription or key                                        │
│  ─ NEVER routes through your llm-gateway                      │
│  ─ your L1/L2 response caches CANNOT intercept these calls    │
└──────────────────────────────────────────────────────────────┘
                            ▲
        the vast majority of tokens in this system
```

```
┌──────────────────────────────────────────────────────────────┐
│  llm-gateway (LiteLLM :4000) sees:                           │
│  ─ embeddings (indexing, ingestion, every search query)      │
│  ─ knowledge summarisation at ingest                          │
│  ─ docstring generation for undocumented symbols              │
│  ─ contradiction/staleness LLM judge passes                   │
│  ─ artifact linting and traceability judgement                │
│  ─ reranking                                                  │
│  ─ headless CI runs (`asdlc run --headless`), which DO use    │
│    server-side inference                                      │
│  ─ any user who opts into BYO-key routing (see below)         │
└──────────────────────────────────────────────────────────────┘
```

So: **L1 and L2 response caching save real money on server-side workloads and headless runs, not on your Cursor session.** Client-side savings come from L0 (prompt-prefix caching, which the client's provider does for you if you order the prompt correctly) and L3 (retrieval caching, which reduces how much context you send at all).

Anyone who tells you a semantic cache will cut your Claude Code bill is describing a different architecture.

### Optional: BYO-key routing

Users who *want* central caching, budgets, and spend visibility can point their tool at the gateway instead of the provider:

```jsonc
// Cursor / Claude Code custom base URL
{ "baseURL": "http://asdlc.internal:4000/v1", "apiKey": "sk-asdlc-<virtualkey>" }
```

Then all four tiers apply and you get per-project spend dashboards. Trade-off: the user gives up provider-subscription pricing and pays per-token against a key you hold. Offer it; don't require it. It's most attractive for teams on API billing anyway, and for CI.

---

## L0 — Prompt-prefix caching

**Where:** the model provider (Anthropic prompt caching, OpenAI automatic prefix caching). **Applies to:** client-side and server-side. **Requires:** deterministic prompt ordering, which the stage brief controls.

The brief is assembled in strictly decreasing stability order:

```
┌─ block 1  role system prompt              stable per prompt_version   ← cache
├─ block 2  output contract + schemas       stable per prompt_version   ← cache
├─ block 3  project policy + conventions    stable per project          ← cache
├─ block 4  run metadata + prior summaries  stable within a run         ← cache
├─ block 5  input artifacts (approved)      changes per stage
└─ block 6  retrieved knowledge + code      changes per call            ← never cache
```

Rules the brief builder enforces:

1. **Never interleave.** One volatile token near the front invalidates every cached block after it.
2. **Byte-stable serialisation.** Sorted JSON keys, `\n` line endings, no timestamps, no run-specific IDs in blocks 1–3.
3. **Mark boundaries.** For Anthropic clients the brief emits explicit `cache_control: {"type":"ephemeral"}` breakpoints after blocks 2, 3, and 4.
4. **Keep block 1–4 above the provider minimum** (1024 tokens for Sonnet-class, 2048 for Haiku-class) or the cache won't engage at all. Pad the policy block with genuinely useful content rather than filler.

**Expected effect:** on a multi-turn stage where the agent iterates, blocks 1–4 are typically 3–6k tokens and hit cache on every turn after the first. This is the single largest client-side saving available to you, and it costs nothing but discipline in the brief builder.

**Verification:** log `cache_read_input_tokens` vs `cache_creation_input_tokens` where the client reports them; the brief builder has a `--explain-cache` mode that prints block boundaries and byte-stability warnings.

---

## L1 — Exact-match response cache

**Where:** LiteLLM + Redis. **Applies to:** gateway traffic only.

```yaml
litellm_settings:
  cache: true
  cache_params:
    type: redis
    host: redis
    port: 6379
    namespace: "asdlc:l1"
    ttl: 604800          # 7d
    supported_call_types: ["acompletion", "aembedding", "atranscription"]
```

Key = hash of `(model, messages, temperature, tools, response_format, project_id)`. Project ID is in the key deliberately — cross-tenant cache sharing is a data-leak vector, not an optimisation.

**Where it actually earns its keep:** embeddings. Re-indexing an unchanged chunk, re-running the same knowledge query, or re-embedding a repeated search string are all exact hits. Embedding traffic is high-volume and highly repetitive — expect 40–70% hit rates in steady state. Set `ttl: 0` (no expiry) for embeddings specifically, since they're deterministic per model:

```yaml
model_list:
  - model_name: embed
    litellm_params:
      model: openai/BAAI/bge-m3
      api_base: http://embeddings:8081/v1
      api_key: "not-needed"
    model_info: { cache_ttl: 0 }
```

**Do not cache** anything with `temperature > 0` for more than a few minutes, and never cache tool-calling completions where the tool results vary. LiteLLM lets you disable per-request with `cache: {"no-cache": true}` — the brief builder sets this on any call whose output feeds an artifact.

---

## L2 — Semantic response cache

**Where:** LiteLLM Redis-semantic (or Valkey with `valkey-search`). **Applies to:** gateway traffic only.

```yaml
litellm_settings:
  cache: true
  cache_params:
    type: redis-semantic
    host: redis
    port: 6379
    similarity_threshold: 0.95
    redis_semantic_cache_embedding_model: embed
    ttl: 86400            # 24h — shorter than L1, because near-misses are riskier
    namespace: "asdlc:l2"
```

LiteLLM's semantic backends build an HNSW vector index with a tag field isolating each cache key's scope, then run a KNN query at lookup and return the cached response when cosine similarity clears the threshold. Valkey with the `valkey-search` module (`type: valkey-semantic`) gets you this without standing up Redis Stack or a separate vector DB.

### Where to use it — and where not to

| Workload | Semantic cache? | Why |
|---|---|---|
| Knowledge summarisation at ingest | ✅ | Near-duplicate documents are common; a near-miss is harmless. |
| Docstring generation | ✅ | Similar functions → similar docs. Low stakes. |
| Staleness/contradiction judging | ✅ | Repetitive by nature. |
| Rerank scoring | ✅ | High volume, tolerant. |
| **Artifact generation** | ❌ | A 0.95-similar prompt is a *different* requirement. Returning run A's ADR for run B is a correctness bug that looks like a success. |
| **Code generation** | ❌ | Same reason, worse consequences. |
| Anything with a tool-call loop | ❌ | State-dependent. |

Enforced structurally: L2 is enabled per-model-alias, and artifact-producing calls use aliases with `cache: false`. Don't rely on remembering.

**Threshold discipline.** 0.95 is deliberately conservative. Below ~0.93 you start serving answers to materially different questions. Measure before loosening: log every semantic hit with the original and matched prompt, sample 50, and count how many you'd have accepted.

---

## L3 — Retrieval cache

**Where:** Redis, in `knowledge-svc` and `indexer-svc`. **Applies to:** everything, including client-side flows.

This is the tier that most directly reduces client-side tokens, because it makes retrieval fast enough to do well rather than lazily.

```
key = sha256(
        "kn:v1", project_id, normalized_query, filters_canonical,
        top_k, embedding_model, index_version
      )
```

Three sub-caches:

| Sub-cache | Key | TTL | Invalidation |
|---|---|---|---|
| Query embedding | `emb:{model}:{sha256(text)}` | none | on model change |
| Knowledge results | as above | 1 h | bumped `knowledge_index_version` on any approve/deprecate |
| Code results | as above + `git_sha` | until repo head moves | automatic via sha in key |

`index_version` is a monotonic counter per project, incremented whenever the underlying corpus changes. Putting it in the key means **you never invalidate anything** — stale keys just stop being requested and expire naturally. Much simpler and less bug-prone than explicit invalidation.

**Brief-level cache.** The assembled stage brief is itself cached under `(run_id, stage, prompt_version, inputs_hash, index_version)`. Restarting a stage, or two developers picking up the same stage, produces a byte-identical brief — which also maximises L0 hits downstream.

---

## Budgets and spend control

LiteLLM virtual keys, one per `(project, agent_role)`:

```yaml
# generated by `asdlc project init`, one key per role
- key_alias: acme-api/architect
  models: [claude-opus-5, gpt-5.1, embed]
  max_budget: 50            # USD per budget_duration
  budget_duration: 30d
  tpm_limit: 200000
  rpm_limit: 60
  metadata: { project_id: acme-api, role: architect }
```

Budget checks read current spend from a cross-pod Redis counter, so enforcement stays fast and consistent across workers and replicas.

**Alert thresholds** (via LiteLLM callbacks → webhook → approvals-ui + optional Slack):

```yaml
general_settings:
  alerting: ["webhook"]
  alerting_threshold: 300           # slow-request alert, seconds
  budget_alert_thresholds: [0.5, 0.8, 0.95]
```

**Per-run cost attribution.** Every gateway call carries `metadata: {run_id, stage, role, artifact_id?}`. LiteLLM's spend logs then answer the question that actually drives optimisation: *which stage burns the money?* Surfaced on the run timeline in the approvals UI. In practice `develop` and `review` dominate; knowing that tells you where to tighten context budgets.

**Client-side cost attribution** is best-effort: agents report `metrics.token_cost` on `artifact_put` when their tool exposes it. Treat it as indicative, not billing-grade.

---

## Fallbacks and routing

```yaml
router_settings:
  routing_strategy: usage-based-routing-v2
  num_retries: 3
  timeout: 600
  fallbacks:
    - { claude-opus-5: [gpt-5.1, claude-sonnet-5] }
    - { embed: [embed-local] }        # TEI down → Ollama
  context_window_fallbacks:
    - { claude-sonnet-5: [claude-opus-5] }
```

Air-gapped deployments replace the model list entirely with Ollama/vLLM endpoints; nothing else in the stack changes because everything speaks the OpenAI-compatible shape through the gateway. That indirection is the main reason to run a gateway even if you never use its caching.

---

## Measuring whether any of this works

Ship these from day one — cache tuning without them is guesswork:

| Metric | Where | Target |
|---|---|---|
| `l0_cache_read_ratio` | client-reported, per stage | > 0.6 on iterative stages |
| `l1_hit_rate` (embeddings) | LiteLLM | > 0.4 steady state |
| `l1_hit_rate` (completions) | LiteLLM | > 0.15 |
| `l2_hit_rate` + false-positive rate | LiteLLM + sampled review | hits > 0.1, FP < 0.02 |
| `l3_hit_rate` | knowledge/indexer svc | > 0.5 |
| `brief_tokens_p95` | core | < `total_max_tokens` |
| `cost_per_run` by stage | LiteLLM spend logs | tracked, not targeted |

The one to watch hardest is **L2 false-positive rate**. Every other metric being wrong costs money; that one costs correctness.

---

## Sources

- [Caching — In-Memory, Redis, s3, Redis Semantic Cache | LiteLLM](https://docs.litellm.ai/docs/caching/all_caches)
- [Proxy caching configuration | LiteLLM](https://docs.litellm.ai/docs/proxy/caching)
- [Semantic Caching on Valkey and AWS ElastiCache | LiteLLM](https://docs.litellm.ai/blog/valkey_semantic_caching)
- [Budgets, Rate Limits | LiteLLM](https://docs.litellm.ai/docs/proxy/users)
- [Budget and Spend Tracking — DeepWiki/LiteLLM](https://deepwiki.com/BerriAI/litellm/3.3-budget-and-spend-tracking)
- [Scale your LLM gateway with LiteLLM & Redis](https://redis.io/blog/scale-your-llm-gateway/)
