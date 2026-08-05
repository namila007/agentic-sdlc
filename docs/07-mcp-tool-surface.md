# 07 — MCP Tool Surface

One MCP endpoint, 23 tools. Everything the agents can do.

```
POST http://<host>:8080/mcp
Authorization: Bearer <token>
Mcp-Project: <project_id>          # or bound to the token
```

Transport: **streamable HTTP, stateless**. The 2026-07-28 spec made HTTP stateless by default, requires `Mcp-Method` and `Mcp-Name` headers on requests so gateways can route on headers rather than parsing bodies, deprecated the older HTTP+SSE transport, and replaced held-open-stream elicitation with Multi Round-Trip Requests (`resultType: "input_required"`). Build against streamable HTTP; do not implement HTTP+SSE.

---

## Why exactly 23 tools

Tool selection accuracy degrades noticeably past roughly 40 tools, and every tool definition costs context on every single request. The gateway therefore exposes a curated surface and keeps internal RPCs off the MCP boundary. Anything a *human* does (ingest, approve, configure) is UI/CLI, not a tool.

---

## Naming convention

`<domain>_<verb>[_<object>]`, snake_case, no namespace prefix.

Domains: `sdlc_`, `artifact_`, `knowledge_`, `code_`, `gate_`.

Rationale: Claude Code prefixes plugin MCP tools automatically, so adding our own prefix produces `mcp__asdlc__asdlc_run_current`. Keep ours clean.

---

## Complete catalog

### SDLC flow (8)

| Tool | Args | Returns |
|---|---|---|
| `sdlc_run_current` | `repo?` | Active run for this project/repo, or `null`. Cheap; call first. |
| `sdlc_run_create` | `title`, `intent`, `repo?`, `labels?` | `run_id`, first stage, bootstrap instruction. |
| `sdlc_run_status` | `run_id?` | Stage states, gates pending, blockers, cost so far. |
| `sdlc_stage_start` | `run_id`, `stage` | **The stage brief.** See [01](01-architecture.md#the-stage-brief--how-client-side-agents-stay-centrally-controlled). |
| `sdlc_stage_submit` | `run_id`, `stage`, `summary`, `notes?` | Validates the output contract; opens the gate or returns `contract_unsatisfied` with what's missing. |
| `sdlc_task_claim` | `run_id`, `task_id` | Lease on a task node (develop fan-out). `lease_expires_at`. |
| `sdlc_task_complete` | `run_id`, `task_id`, `status`, `reason?` | `done` / `deferred` / `blocked`. |
| `sdlc_check_report` | `run_id`, `stage`, `check`, `result`, `output?` | Report a client-side check result (build, lint, test run) against an acceptance criterion. |

### Artifacts (7)

| Tool | Args | Returns |
|---|---|---|
| `artifact_put` | `run_id`, `stage`, `type`, `name`, `slug?`, `format`, `content` \| `content_ref`, `links?[]`, `metrics?` | `artifact_id`, `version`, or `{ok:false, errors:[{path,message,fix}]}`. |
| `artifact_get` | `artifact_id` \| (`type`,`slug`,`version?`) | Metadata + `inline` (text) or presigned URL. |
| `artifact_list` | `run_id?`, `stage?`, `type?`, `status?`, `limit?` | Previews only. Never full bodies. |
| `artifact_diff` | `artifact_id`, `from_version`, `to_version` | Unified diff. |
| `artifact_link` | `from`, `to`, `relation`, `note?` | Provenance edge. |
| `artifact_trace` | `artifact_id`, `direction`, `depth?` | Provenance subgraph. |
| `artifact_search` | `query`, `type?`, `status?`, `top_k?` | Hybrid search over the library. |

### Knowledge (5)

| Tool | Args | Returns |
|---|---|---|
| `knowledge_search` | `query`, `kinds?[]`, `scopes?[]`, `top_k?`, `max_tokens?` | Summaries + scores + entry ids. |
| `knowledge_get` | `entry_id` | Full body. |
| `knowledge_propose` | `title`, `body`, `kind`, `scope`, `evidence[]`, `confidence?` | Creates a **candidate**. Never approved. |
| `knowledge_feedback` | `entry_id`, `signal`, `note?` | `helpful` \| `stale` \| `wrong`. |
| `knowledge_list` | `kind?`, `scope?`, `tag?`, `limit?` | Browse. |

### Code (6)

| Tool | Args | Returns |
|---|---|---|
| `code_search` | `query`, `repo?`, `lang?`, `path_glob?`, `include_tests?`, `top_k?`, `max_tokens?` | Chunks with `path`, `start_line`, `end_line`, `git_sha`, `score`. |
| `code_symbol` | `name` \| `qualified`, `repo?` | Definition, signature, doc, location. |
| `code_refs` | `symbol`, `repo?` | References + `resolution: exact\|heuristic` + `confidence`. |
| `code_file_outline` | `path`, `repo?` | Symbol tree, no bodies. Cheapest way to understand a file. |
| `code_neighbors` | `path`, `line` \| `symbol` | Structurally adjacent chunks. |
| `code_index_status` | `repo?` | Last indexed sha, staleness, counts. |

### Gates (2)

| Tool | Args | Returns |
|---|---|---|
| `gate_status` | `run_id`, `stage?` | Pending gates, approvers, elapsed time. |
| `gate_decide` | `gate_id`, `decision`, `comment?` | **Human-only.** Rejected unless the caller's token carries `role=approver` *and* `ASDLC_ALLOW_IDE_APPROVAL=true`. See [08](08-human-validation.md#channel-3--in-ide-mcp). |

*(`code_index_sync` is exposed only when `ASDLC_EXPOSE_INDEX_SYNC=true`; it's an operator action, and agents that can trigger full reindexes will.)*

---

## Response conventions

Every tool returns a consistent envelope:

```jsonc
{
  "ok": true,
  "data": { /* tool-specific */ },
  "meta": {
    "tokens_estimate": 1840,      // so agents can budget
    "truncated": false,
    "cache": "l3_hit",            // miss | l3_hit | brief_cache
    "index_version": 412,
    "elapsed_ms": 63
  }
}
```

Errors:

```jsonc
{
  "ok": false,
  "error": {
    "code": "CONTRACT_VIOLATION",
    "message": "Artifact type 'test_case' is not permitted in stage 'design'.",
    "fix": "Produce artifacts of type: adr, api_contract, component_diagram. If you believe a test case is needed, note it in the ADR under Consequences.",
    "retryable": false
  }
}
```

**The `fix` field is not decoration.** Agents recover from a suggested correction far more reliably than from a bare error message, and a bad error message costs a full retry loop. Every error path in the codebase must populate it. This is a lint rule in CI.

---

## Error codes

| Code | Meaning | Retryable |
|---|---|---|
| `UNAUTHENTICATED` | Missing/invalid token | no |
| `PROJECT_SCOPE_DENIED` | Token can't access this project | no |
| `RUN_NOT_FOUND` / `STAGE_NOT_ACTIVE` | Wrong run state | no |
| `STAGE_ORDER_VIOLATION` | Prior stage not approved | no |
| `CONTRACT_VIOLATION` | Artifact type/count/schema wrong | no |
| `SCHEMA_INVALID` | Content failed `schema_ref` | no |
| `GATE_PENDING` | Waiting on a human | yes, with backoff |
| `LEASE_CONFLICT` | Task claimed by another session | yes |
| `INDEX_STALE` | Code index behind head by > threshold | yes after `code_index_sync` |
| `BUDGET_EXCEEDED` | Project/role budget hit | no |
| `RATE_LIMITED` | TPM/RPM | yes, `retry_after_ms` |
| `PAYLOAD_TOO_LARGE` | Artifact > `max_artifact_bytes` | no |

---

## Token discipline

Agents run out of context; the tool surface must actively defend against that.

1. **Every retrieval tool takes `max_tokens`** and truncates server-side, returning `meta.truncated: true` plus a continuation hint. Truncation happens at chunk boundaries, never mid-chunk.
2. **List tools return previews.** `artifact_list` returns ≤4 KB of `inline_preview`, never bodies. Fetching a body is an explicit second call.
3. **Outline before read.** `code_file_outline` is typically 50× cheaper than reading the file. Role prompts for `implementer` and `reviewer` mandate it.
4. **Summaries before bodies.** `knowledge_search` returns ≤200-token summaries; `knowledge_get` is the opt-in for full text.
5. **Default `top_k` is deliberately low** (knowledge 8, code 10). Agents that need more can ask; most don't, and the default is what gets used.
6. **Tool descriptions are short.** Every tool's description is ≤ 60 tokens and its schema has no verbose enum documentation. 23 tools × 200 tokens of schema is 4.4k tokens on *every* request — that's a real cost, and trimming descriptions is free.

---

## Elicitation (asking the human mid-run)

Under MCP 2026-07-28, servers no longer push an `elicitation/create` over a held-open stream. Instead the server returns `resultType: "input_required"` with the requests it needs answered, and the client retries the original call with answers in `inputResponses`.

Used by:

- `sdlc_run_create` when `intent` is too vague to plan from.
- `sdlc_stage_submit` when an acceptance criterion needs a human judgement call.
- `artifact_put` when a declared `implements` target is ambiguous between two tasks.

```jsonc
// server → client
{
  "resultType": "input_required",
  "inputRequests": [{
    "id": "q1",
    "message": "Which task does this change implement?",
    "schema": { "type": "string", "enum": ["T-4: token refresh", "T-7: session store"] }
  }]
}
// client retries the same call with:
{ "inputResponses": { "q1": "T-7: session store" } }
```

**Client support is uneven.** Every elicitation path must degrade to a plain error with a `fix` string telling the agent what to pass explicitly. Never make a flow depend on elicitation working.

---

## Versioning

The tool surface is versioned in the MCP `serverInfo`:

```jsonc
{ "name": "asdlc", "version": "1.4.0", "protocolVersion": "2026-07-28" }
```

- **Additive changes** (new tool, new optional arg) → minor bump, no client action.
- **Breaking changes** (removed tool, changed required arg) → major bump. The gateway keeps N-1 alive at `/mcp/v1` for one release cycle and emits a deprecation warning in `meta`.
- The compiled plugin pins a **minimum** server version and fails loudly at bootstrap if the server is older, rather than failing mysteriously mid-run.

---

## Sources

- [The 2026-07-28 Specification | MCP Blog](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [MCP 2026-07-28 spec: what changed, what breaks](https://stacktr.ee/blog/mcp-2026-spec-changes)
- [MCP goes stateless in the 2026-07-28 specification — Appwrite](https://appwrite.io/blog/post/mcp-goes-stateless-in-the-2026-07-28-specification)
- [MCP Cheat Sheet (2026) — Webfuse](https://www.webfuse.com/mcp-cheat-sheet)
