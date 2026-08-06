# Agentic SDLC (ASDLC)

A tool-agnostic control plane for running a software development lifecycle with AI agents.

Agents run **inside whatever tool the developer already uses** — Claude Code, Cursor, GitHub Copilot, Windsurf, Codex CLI. The **reasoning happens client-side**. Everything durable — stage prompts, artifacts, approvals, project knowledge, code index, caching, budgets — lives in a **self-hostable server stack** exposed over MCP.

```
Plan → Design → Develop → Review → Test → Document
  └── each stage ends at a human validation gate
```

---

## Document index

| # | Doc | What it covers |
|---|-----|----------------|
| 00 | [Overview & principles](docs/00-overview.md) | Problem, goals, non-goals, glossary, key decisions |
| 01 | [Architecture](docs/01-architecture.md) | Components, data flow, ports, the "stage brief" mechanism |
| 02 | [SDLC flow & agent specs](docs/02-sdlc-flow-and-agents.md) | The 6 stages, per-agent contracts, artifacts, acceptance criteria |
| 03 | [Artifact library](docs/03-artifact-library.md) | Storage, schema, versioning, provenance DAG, attestations |
| 04 | [Knowledge center](docs/04-knowledge-center.md) | Project knowledge, ingestion, hybrid retrieval, curation |
| 05 | [Code indexer](docs/05-code-indexer.md) | Tree-sitter chunking, incremental indexing, symbol graph |
| 06 | [LLM cache & gateway](docs/06-llm-cache-and-gateway.md) | 4 cache tiers, budgets, and where caching actually applies |
| 07 | [MCP tool surface](docs/07-mcp-tool-surface.md) | Full tool catalog, naming, schemas, error model |
| 08 | [Human validation gates](docs/08-human-validation.md) | Gate model, three approval channels, policy tiers |
| 09 | [Plugin packaging](docs/09-plugin-packaging.md) | One source → Claude plugin, Cursor rules, Copilot, AGENTS.md |
| 10 | [Deployment](docs/10-deployment.md) | Docker Compose, profiles, env, air-gapped mode, backups |
| 11 | [Security & multi-tenancy](docs/11-security-and-multitenancy.md) | Project isolation, authn/z, agent-specific threats |
| 12 | [Roadmap](docs/12-roadmap.md) | Phased build plan with exit criteria |
| 13 | [Open questions](docs/13-open-questions.md) | Decisions still needed from you |

---

## The one-paragraph version

You define each SDLC agent **once**, as YAML in this repo. A build step compiles those definitions into every tool's native format (Claude skills/subagents, Cursor `.mdc` rules, Copilot instructions, `AGENTS.md`) and ships them as an installable plugin. But those files stay deliberately thin — they're a bootstrap loop that says *"call `sdlc_stage_start` and do what it tells you."* The real prompt, the retrieved context, the acceptance criteria, and the artifact contract are served at runtime by the control plane, so a prompt fix reaches every user without a plugin reinstall. Agents write outputs back through `artifact_put`, which stores them content-addressed with full provenance and opens a human gate. Nothing advances to the next stage until a human approves — via PR review, a web dashboard, or a slash command in their IDE.

---

## Quick orientation for building this

Read in this order: `00` → `01` → `02` → `07`. Those four define the system. Everything else is a component deep-dive you can defer.

If you want to start coding today, go to **[milestones/](milestones/)** — the roadmap decomposed into executable work. [`MILESTONES.md`](milestones/MILESTONES.md) is the master plan; [`M0 — Walking skeleton`](milestones/M0-walking-skeleton.md) is where the first commit goes.
