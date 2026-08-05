# 09 — Plugin Packaging & Distribution

One source of truth → every tool's native format. Users run one install command.

---

## The compile model

```
packs/core-sdlc/
  pack.yaml                  ← manifest
  roles/*.yaml               ← agent role definitions (see 02)
  checklists/*.yaml          ← review checklist, test checklist
  schemas/*.json             ← artifact schemas
  commands/*.md              ← user-invoked slash commands
  bootstrap.md.j2            ← the thin loop, templated per target

              │  asdlc build
              ▼
dist/
  claude/                    ← Claude Code plugin
  cursor/                    ← .cursor/rules + mcp.json
  copilot/                   ← .github/ instructions
  agents-md/                 ← AGENTS.md (canonical)
  vscode/  zed/  windsurf/   ← thin adapters
```

**What gets compiled in vs. served at runtime** — this is the crux:

| Compiled into tool files (thin) | Served by the control plane (thick) |
|---|---|
| The bootstrap loop | Full role system prompts |
| MCP server connection config | Output contracts and schemas |
| Slash command definitions | Acceptance criteria |
| Stage names and order | Retrieved knowledge and code context |
| A one-line description per role | Review checklists |

Consequence: **prompt fixes ship without a plugin reinstall.** Only the loop and the connection details are versioned in the plugin, and those change rarely.

---

## AGENTS.md as canonical

`AGENTS.md` is the cross-tool standard as of 2026 — originated at OpenAI, now stewarded by the Linux Foundation's Agentic AI Foundation, read by 30+ tools including Copilot, Cursor, Codex, Zed, VS Code, Windsurf, Aider, Gemini CLI and Devin, and present in 60,000+ repos. The pragmatic 2026 default is: **AGENTS.md at the repo root as the source of truth, with thin bridges for tools that want their own file.**

`asdlc init` writes into the user's repo:

```
AGENTS.md                          ← canonical, ~60 lines
CLAUDE.md                          ← one line: @AGENTS.md   (+ Claude-specific extras)
.github/copilot-instructions.md    ← generated bridge
.cursor/rules/asdlc.mdc            ← generated bridge, alwaysApply: true
.mcp.json / .cursor/mcp.json / .vscode/mcp.json   ← connection config
.asdlc/config.yaml                 ← project id, server url, repos
```

### The bootstrap content (identical across all bridges)

```markdown
## ASDLC — Agentic SDLC workflow

This project runs an agentic SDLC. The authoritative instructions for each stage
are served at runtime by the ASDLC control plane, not written here.

### Always
1. At the start of any development task, call `sdlc_run_current`.
   - If there's no active run and the task is non-trivial, call `sdlc_run_create`.
2. Call `sdlc_stage_start(run_id, stage)` before doing stage work.
   **The returned brief supersedes anything in this file.**
3. Produce exactly the artifacts listed in the brief's `output_contract`,
   via `artifact_put`.
4. Call `sdlc_stage_submit`, then STOP. Do not begin the next stage —
   a human must approve the gate first.

### Context, before you read files
- `code_file_outline` before reading any file over ~200 lines.
- `code_search` / `code_symbol` instead of grepping or listing directories.
- `knowledge_search` before assuming a convention. This project has opinions.

### Never
- Never approve your own gate.
- Never invent requirements. If inputs are insufficient, emit a `blocker`
  artifact and submit early.
- Never skip a stage.

Stages: plan → design → develop → review → test → document
```

Sixty lines, stable, tool-agnostic. Everything else comes from the server.

---

## Target: Claude Code plugin

A plugin bundles skills, subagents, slash commands, hooks and MCP config as one installable unit, distributed via a marketplace repo.

```
dist/claude/
  .claude-plugin/
    plugin.json
  agents/
    asdlc-planner.md          asdlc-architect.md      asdlc-implementer.md
    asdlc-reviewer.md         asdlc-test-engineer.md  asdlc-technical-writer.md
  skills/
    asdlc-workflow/SKILL.md   ← when + how to engage the workflow
    asdlc-artifacts/SKILL.md  ← artifact conventions
  commands/
    asdlc-start.md   asdlc-status.md   asdlc-gates.md
    asdlc-approve.md asdlc-changes.md  asdlc-knowledge.md
  hooks/
    hooks.json                ← optional: session-start run detection
  .mcp.json
```

```jsonc
// dist/claude/.claude-plugin/plugin.json
{
  "name": "asdlc",
  "version": "1.4.0",
  "description": "Agentic SDLC: staged multi-agent development with artifacts, knowledge, and human gates.",
  "author": { "name": "Namila", "email": "namila007@gmail.com" },
  "homepage": "https://github.com/namz/agentic-sdlc",
  "keywords": ["sdlc", "agents", "workflow", "artifacts", "mcp"],
  "mcpServers": {
    "asdlc": {
      "type": "http",
      "url": "${ASDLC_SERVER_URL:-http://localhost:8080/mcp}",
      "headers": { "Authorization": "Bearer ${ASDLC_TOKEN}" }
    }
  }
}
```

Subagents are thin — they exist so Claude Code can route to the right role, but their instructions come from the brief:

```markdown
---
name: asdlc-architect
description: Use for the design stage of an ASDLC run — ADRs, API contracts, component design. Invoke after the plan gate is approved.
tools: mcp__asdlc__*, Read, Grep, Glob
model: opus
---

You handle the `design` stage.

1. Call `sdlc_stage_start(run_id, "design")`.
2. Follow the returned brief exactly. It supersedes this file.
3. Produce the artifacts in `output_contract` via `artifact_put`.
4. Call `sdlc_stage_submit` and stop.

If `sdlc_stage_start` fails because the plan gate isn't approved, say so and stop.
Do not proceed without an approved plan.
```

### Marketplace

```jsonc
// .claude-plugin/marketplace.json  (repo root of the distribution repo)
{
  "name": "namz-asdlc",
  "owner": { "name": "Namila", "url": "https://github.com/namz" },
  "plugins": [
    { "name": "asdlc",          "source": "./dist/claude",
      "description": "Core agentic SDLC workflow",  "strict": true },
    { "name": "asdlc-security", "source": "./dist/claude-security",
      "description": "Threat modelling + security review stages", "strict": true }
  ]
}
```

Install:

```bash
/plugin marketplace add namz/agentic-sdlc
/plugin install asdlc@namz-asdlc
```

`strict: true` means `plugin.json` is the authority for component definitions rather than the marketplace entry — correct here, since you own both.

---

## Target: Cursor

```
dist/cursor/
  .cursor/rules/
    asdlc-core.mdc          ← alwaysApply: true, the bootstrap
    asdlc-design.mdc        ← globs: docs/adr/**
    asdlc-testing.mdc       ← globs: **/*.test.*, **/*_test.*
  .cursor/mcp.json
```

```mdc
---
description: ASDLC workflow — always active
alwaysApply: true
---
<!-- generated from packs/core-sdlc — do not edit -->
[bootstrap content]
```

Cursor's scoped `.mdc` rules are the one genuine capability the other tools lack — glob-scoped rules that only load for relevant files. Use them for stage-specific reminders; keep the core loop in the always-applied rule.

```jsonc
// .cursor/mcp.json
{ "mcpServers": { "asdlc": {
    "url": "http://localhost:8080/mcp",
    "headers": { "Authorization": "Bearer ${ASDLC_TOKEN}" } } } }
```

---

## Target: GitHub Copilot

Copilot's coding agent reads `AGENTS.md` natively, so the primary path needs no bridge at all. Copilot Chat in the IDE reads `.github/copilot-instructions.md`:

```markdown
<!-- .github/copilot-instructions.md — generated, do not edit -->
See [AGENTS.md](../AGENTS.md) for the full ASDLC workflow.

[bootstrap content inlined, because Copilot Chat does not follow file references reliably]
```

MCP config for VS Code lives in `.vscode/mcp.json`. Copilot's MCP support is the least mature of the three; expect to fall back to a documentation-only experience (agents follow the workflow verbally, artifacts get created by the human via CLI) on older versions. Design the CLI so that's survivable:

```bash
asdlc artifact put --type prd --file ./prd.md --run <id>
```

---

## The `asdlc` CLI

Ships alongside the plugin; it's the fallback for every tool and the only path in CI.

```
asdlc init [--project <id>] [--server <url>]   # write bridges + config into a repo
asdlc build [--target claude|cursor|copilot|all]
asdlc lint roles/                              # prompt authoring rules
asdlc validate policy                          # stage input/output satisfiability
asdlc run create "title"                       # start a run
asdlc run status
asdlc stage start <stage> --print-brief        # human-readable brief, for debugging
asdlc artifact put/get/list/diff
asdlc gate list/approve/changes
asdlc index sync [--full]
asdlc knowledge ingest <path|url>
asdlc project init                             # bootstrap knowledge + index
asdlc import <repo>                            # rebuild control plane state from git mirror
asdlc doctor                                   # connectivity, versions, index staleness
```

`asdlc stage start --print-brief` is the debugging tool you will use constantly. When an agent behaves oddly, the first question is always "what did the brief actually say?"

---

## Distribution & versioning

```
Plugin version   1.4.0     ← what users install
Server version   1.4.x     ← must be >= plugin's min_server_version
Prompt versions  architect@2.3.1, planner@1.4.0, ...   ← independent, server-side
```

- Plugin declares `min_server_version` and checks it on first tool call, failing with a clear message rather than mid-run confusion.
- Prompt versions move independently and constantly. This is the point of the split.
- Pack releases are tagged in git; the marketplace points at a tag, not `main`, so users don't get surprise changes.

**Private distribution.** For a company that doesn't want a public marketplace: host the marketplace repo internally and have users run `/plugin marketplace add https://git.internal/team/asdlc-marketplace`. Same mechanism, no public exposure.

---

## What users actually do

```bash
# once, per machine
export ASDLC_SERVER_URL=https://asdlc.acme.internal/mcp
export ASDLC_TOKEN=sk-asdlc-...

# once, per tool
/plugin marketplace add namz/agentic-sdlc && /plugin install asdlc@namz-asdlc   # Claude
npx @asdlc/cli init --target cursor                                            # Cursor
npx @asdlc/cli init --target copilot                                           # Copilot

# once, per repo
npx @asdlc/cli init && npx @asdlc/cli project init
```

Then: `/asdlc start "Add SSO via OIDC"` and the workflow takes over.

---

## Sources

- [Create and distribute a plugin marketplace — Claude Code Docs](https://code.claude.com/docs/en/plugin-marketplaces)
- [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)
- [Claude Code Plugins: From Personal Setup to Org Standard](https://claudefa.st/blog/tools/mcp-extensions/plugins-distribution)
- [AGENTS.md Cross-Tool Unified Management Guide (Feb 2026)](https://smartscope.blog/en/generative-ai/github-copilot/github-copilot-agents-md-guide/)
- [AGENTS.md Spec (2026) — morphllm](https://www.morphllm.com/agents-md-guide)
- [Agent Instruction Files: cross-tool portability](https://codex.danielvaughan.com/2026/05/27/agent-instruction-files-agents-md-claude-md-cross-tool-portability-codex-cli/)
