# 00 — Overview & Principles

## The problem

Teams now run AI agents across several tools at once: Claude Code, Cursor, GitHub Copilot, sometimes Codex or Windsurf. Each tool has its own instruction format, its own memory, its own idea of "context." The result:

- **Prompt drift.** The "spec writer" prompt in `.cursorrules` and the one in `CLAUDE.md` diverge within two weeks.
- **No artifact continuity.** The design doc an agent produced on Monday isn't available to the coding agent on Wednesday, in a different tool, on a different machine.
- **No memory of the project.** Every session re-derives the same architectural facts, burning tokens and getting them subtly wrong.
- **No gate.** Agents produce output; nobody records who approved what, or why.
- **Cost is opaque.** Nobody knows which stage burns the tokens.

## What this system is

A **control plane** that owns everything durable about an agentic SDLC, and stays deliberately out of the inference path.

```
┌─────────────────────────────────────────────────────────────┐
│  DEVELOPER'S TOOL  (Claude Code / Cursor / Copilot / ...)    │
│  ─ runs the LLM, does the reasoning, edits the files         │
│  ─ holds only a thin bootstrap instruction file              │
└───────────────────────────┬─────────────────────────────────┘
                            │ MCP (streamable HTTP)
┌───────────────────────────▼─────────────────────────────────┐
│  ASDLC CONTROL PLANE  (your docker-compose stack)            │
│  stage briefs · artifacts · gates · knowledge · code index   │
│  · LLM gateway + cache · budgets · audit                     │
└─────────────────────────────────────────────────────────────┘
```

## Goals

1. **One definition, every tool.** Agent behaviour is authored once and compiled to each tool's format.
2. **Prompts are served, not shipped.** The bulk of each agent's instructions comes from the server at runtime so it can be fixed centrally.
3. **Every stage output is an artifact** with an ID, a version, a hash, a producer, and a provenance chain.
4. **No stage advances without a human.** Approval is recorded, attributable, and queryable.
5. **Project knowledge accumulates.** What the team learned in run #12 is retrievable in run #47.
6. **Token cost is measured and capped**, per project and per agent role.
7. **Runs on one box with `docker compose up`**, and works fully offline with local models.

## Non-goals (v1)

- Not a replacement for CI/CD. ASDLC produces PRs; your existing CI tests them.
- Not an IDE. No editor, no terminal, no file-writing by the server.
- Not a Kubernetes platform. Single-host Compose is the target; k8s is a later port.
- Not an autonomous "ship it without humans" system. Gates are mandatory by design.
- Not an agent framework. We don't ship an orchestration runtime that runs LLM loops server-side (see [Decision D1](#d1--agents-are-client-side)).

---

## Key decisions

### D1 — Agents are client-side

The developer's tool does the inference. The control plane never calls an LLM to *do SDLC work*.

**Why:** the user is already paying for Claude Max / Copilot / Cursor. Duplicating that spend server-side is wasteful, and it means the agent loses direct file access, terminal access, and the IDE's own context. It also removes any need for us to hold the user's model credentials.

**Consequence (important):** the LLM gateway and its response cache **do not see** the tokens spent by Claude Code or Cursor. They see only server-side inference — embeddings, ingestion summarisation, artifact linting — plus any user who deliberately opts into BYO-key routing. This is spelled out honestly in [06 — LLM cache & gateway](06-llm-cache-and-gateway.md#what-the-gateway-can-and-cannot-see). Client-side token savings come from a different mechanism: serving small, well-ordered, cache-friendly stage briefs instead of dumping whole repos into context.

### D2 — The stage brief is the core primitive

A client calls `sdlc_stage_start(run_id, stage)`. The server returns a **stage brief**: the agent's role prompt, the inputs (prior approved artifacts), pre-retrieved knowledge and code context, the output contract, and acceptance criteria — as one structured payload.

This is what makes "client-side agents" workable across four different tools. The tool-native config file only has to know *how to call the brief and obey it*. Everything else is centralised, versioned, and hot-fixable.

### D3 — Qdrant + Postgres + MinIO

- **Postgres** — runs, stages, gates, artifact metadata, provenance edges, symbol graph, BM25 (`tsvector`), audit log.
- **Qdrant** — dense vectors for knowledge chunks and code chunks, with payload filtering for project scoping.
- **MinIO** — content-addressed artifact blobs (S3 API, so a later move to real S3 is config-only).

Rationale and the alternative (single Postgres+pgvector) are in [01 — Architecture](01-architecture.md#why-three-stores).

### D4 — `AGENTS.md` is the canonical bootstrap format

As of 2026, `AGENTS.md` is read natively by 30+ agent tools including Copilot, Cursor, Codex, Zed and Windsurf, and is stewarded by the Linux Foundation's Agentic AI Foundation. We generate `AGENTS.md` as the source of truth and emit thin bridges: `CLAUDE.md` importing it, plus `.cursor/rules/*.mdc` for Cursor's scoped-rule features and `.github/copilot-instructions.md` for Copilot Chat. Details in [09 — Plugin packaging](09-plugin-packaging.md).

### D5 — Three approval channels, one gate record

Git/PR, web dashboard, and in-IDE MCP tool all write to the same `gate_events` table. The channel is metadata, not a fork in the design. See [08 — Human validation](08-human-validation.md).

### D6 — Multi-project from day one, auth optional

Every row, blob prefix, and vector payload carries `project_id` from the first commit — retrofitting tenancy is expensive. But authentication ships behind a flag (`ASDLC_AUTH_MODE=none|token|oidc`) so solo/local use stays a single `docker compose up`. See [11 — Security](11-security-and-multitenancy.md).

---

## Glossary

| Term | Meaning |
|------|---------|
| **Run** | One pass through the SDLC for one unit of work (a feature, a bug, an epic). Has an ID, a project, and an ordered set of stages. |
| **Stage** | One of `plan`, `design`, `develop`, `review`, `test`, `document`. Owned by one agent role. Ends at a gate. |
| **Agent role** | A named behaviour spec (`planner`, `architect`, `implementer`, `reviewer`, `test-engineer`, `technical-writer`). Authored as YAML, compiled to tool formats. |
| **Stage brief** | The runtime payload the server hands a client when a stage starts: prompt + inputs + context + output contract + acceptance criteria. |
| **Artifact** | Any durable stage output — PRD, ADR, task graph, diff, test plan, doc page. Content-addressed, versioned, provenance-linked. |
| **Gate** | A human decision point at the end of a stage: `approved`, `changes_requested`, or `rejected`. |
| **Knowledge center** | Curated, project-scoped, retrievable facts: ADRs, conventions, domain glossary, past run learnings, external library docs. |
| **Code index** | AST-chunked, embedded representation of the repo, plus an exact symbol graph. |
| **Control plane** | The docker-compose stack. Everything in this repo that isn't a prompt. |
| **Pack** | A distributable bundle of agent roles + MCP config + commands. What becomes a Claude plugin / Cursor ruleset. |

---

## Sources

- [AGENTS.md vs CLAUDE.md vs Cursor Rules vs Copilot (2026)](https://codersera.com/blog/agents-md-vs-claude-md-vs-cursor-rules-comparison-2026/)
- [The AGENTS.md Field Guide, 2026 edition](https://www.iuriio.com/blog/posts/2026/05/agents-md-field-guide-2026)
- [Agentic SDLC: What Changes When Agents Run Development — Augment Code](https://www.augmentcode.com/guides/agentic-sdlc)
- [Create and distribute a plugin marketplace — Claude Code Docs](https://code.claude.com/docs/en/plugin-marketplaces)
