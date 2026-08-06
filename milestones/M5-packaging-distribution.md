# M5 — Packaging & Distribution

**Weeks 17–19** · [← Master plan](MILESTONES.md) · Implements [`docs/09-plugin-packaging.md`](../docs/09-plugin-packaging.md)

> **Goal:** someone who isn't you can install and use this.
>
> A colleague goes from zero to a completed run in under 30 minutes using only the docs.

**Parallelisable.** M5 can start any time after M1.1 freezes the pack format. It does not need M2,
M3 or M4.

---

## The compile model

The whole design rests on one split: **thin files compiled into tools, thick content served at
runtime.**

```
                     packs/core-sdlc/
                            │
                            │  asdlc build
                            ▼
       ┌────────────┬───────────────┬────────────┬─────────────┐
       ▼            ▼               ▼            ▼             ▼
  dist/claude/  dist/cursor/  dist/copilot/  dist/agents-md/  dist/{vscode,zed,windsurf}/
   plugin.json   .cursor/      .github/       AGENTS.md         thin adapters
   agents/       rules/*.mdc   copilot-
   skills/       mcp.json      instructions.md
   commands/
   .mcp.json
```

| Compiled into tool files (**thin**) | Served by the control plane (**thick**) |
|---|---|
| The bootstrap loop (~60 lines) | Full role system prompts |
| MCP server connection config | Output contracts and schemas |
| Slash-command definitions | Acceptance criteria |
| Stage names and order | Retrieved knowledge and code context |
| One-line description per role | Review checklists |

**Consequence:** a prompt fix ships to every user with no reinstall. Only the loop and the connection
details are versioned in the plugin, and those change rarely.

---

## Sub-milestones

| ID | Name | Depends on | Days |
|---|---|---|---|
| [M5.1](#m51--pack-format-and-manifest) | Pack format & manifest | M1.1 | 1.5 |
| [M5.2](#m52--asdlc-build-and-golden-file-tests) | `asdlc build` & golden-file tests | M5.1 | 3 |
| [M5.3](#m53--claude-code-plugin-and-marketplace) | Claude Code plugin & marketplace | M5.2 | 2.5 |
| [M5.4](#m54--cursor-copilot-and-agentsmd-targets) | Cursor, Copilot & AGENTS.md targets | M5.2 | 3 |
| [M5.5](#m55--asdlc-init-in-a-user-repo) | `asdlc init` in a user repo | M5.3, M5.4 | 2 |
| [M5.6](#m56--gitpr-approval-channel-and-notifier) | Git/PR approval channel & notifier | M1.5 | 3 |
| [M5.7](#m57--docs-demo-and-the-onboarding-test) | **Docs, demo & the onboarding test** | all above | 2.5 |

---

## M5.1 — Pack format and manifest

```
packs/core-sdlc/
  pack.yaml                  ← manifest
  roles/*.yaml               ← agent role definitions (M1.1)
  checklists/*.yaml          ← review checklist, test checklist
  schemas/*.json             ← artifact schemas
  commands/*.md              ← user-invoked slash commands
  bootstrap.md.j2            ← the thin loop, templated per target
```

```yaml
# pack.yaml
name: core-sdlc
version: 1.4.0
description: Core agentic SDLC workflow
author: { name: "…", email: "…" }
homepage: https://github.com/…
min_server_version: "1.4.0"
roles: [planner, architect, implementer, reviewer, test-engineer, technical-writer]
stages: [plan, design, develop, review, test, document]
commands: [asdlc-start, asdlc-status, asdlc-gates, asdlc-approve, asdlc-changes, asdlc-knowledge]
```

### The checklists are the customisation point

`checklists/review.yaml` is injected into the review brief as **data**. A team forks the checklist
without touching a prompt — which is what makes the pack adoptable by someone with different review
standards. Prompt editing is a fork; checklist editing is configuration.

### Multiple packs

`packs/security/` adds a `threat_model` stage and a `threat-modeler` role. Packs compose: a project
policy lists stages drawn from any installed pack. This is why the stage list is policy rather than
code (M1.1).

### Acceptance

- [ ] `pack.yaml` validates against a JSON Schema
- [ ] A second pack installs alongside `core-sdlc` without collision
- [ ] Forking `checklists/review.yaml` changes review behaviour with no prompt edit

---

## M5.2 — `asdlc build` and golden-file tests

```
asdlc build [--target claude|cursor|copilot|agents-md|all] [--out dist/]
```

Deterministic: same pack in, byte-identical output. Non-negotiable, because golden-file testing
depends on it — no timestamps, no build IDs, no map-iteration order in output.

### Golden-file tests — the anti-rot mechanism

```
tests/golden/
  claude/.claude-plugin/plugin.json
  claude/agents/asdlc-architect.md
  cursor/.cursor/rules/asdlc-core.mdc
  copilot/.github/copilot-instructions.md
  agents-md/AGENTS.md
```

CI runs `asdlc build` and diffs against `tests/golden/`. **Any drift fails the build.**

This mitigates a specific, likely failure: the compile-to-four-tools step rots silently. Someone
edits the bootstrap in `packs/`, three targets regenerate correctly, one has a subtle template bug,
and nobody notices for a month because that tool gets used less. The golden files turn that into a
red CI run.

Updating a golden file is deliberate: `asdlc build --update-golden`, and the diff is reviewed in the PR.

### The bootstrap loop

`packs/core-sdlc/bootstrap.md.j2`, ~60 lines, near-identical across targets:

```markdown
## ASDLC — Agentic SDLC workflow

This project runs an agentic SDLC. The authoritative instructions for each stage are served at
runtime by the ASDLC control plane, not written here.

### Always
1. At the start of any development task, call `sdlc_run_current`.
   - If there's no active run and the task is non-trivial, call `sdlc_run_create`.
2. Call `sdlc_stage_start(run_id, stage)` before doing stage work.
   **The returned brief supersedes anything in this file.**
3. Produce exactly the artifacts listed in the brief's `output_contract`, via `artifact_put`.
4. Call `sdlc_stage_submit`, then STOP. Do not begin the next stage —
   a human must approve the gate first.

### Context, before you read files
- `code_file_outline` before reading any file over ~200 lines.
- `code_search` / `code_symbol` instead of grepping or listing directories.
- `knowledge_search` before assuming a convention. This project has opinions.

### Never
- Never approve your own gate.
- Never invent requirements. If inputs are insufficient, emit a `blocker` artifact and submit early.
- Never skip a stage.

Stages: plan → design → develop → review → test → document
```

Per-target variation is minimal — Claude Code mentions slash commands and agent delegation, Cursor
mentions rule scoping, Copilot mentions the CLI fallback. **The core stays identical**, because M0.6
established that this loop works, and divergence would invalidate that evidence.

### Acceptance

- [ ] Two consecutive builds produce byte-identical output
- [ ] Golden-file tests fail on any generated-file drift
- [ ] `--update-golden` produces a reviewable diff
- [ ] Every generated file carries a `generated — do not edit` header

---

## M5.3 — Claude Code plugin and marketplace

```
dist/claude/
  .claude-plugin/plugin.json
  agents/asdlc-{planner,architect,implementer,reviewer,test-engineer,technical-writer}.md
  skills/asdlc-workflow/SKILL.md
  skills/asdlc-artifacts/SKILL.md
  commands/asdlc-{start,status,gates,approve,changes,knowledge}.md
  hooks/hooks.json                # optional: session-start run detection
  .mcp.json
```

```jsonc
// .claude-plugin/plugin.json
{
  "name": "asdlc",
  "version": "1.4.0",
  "description": "Agentic SDLC: staged multi-agent development with artifacts, knowledge, and human gates.",
  "author": { "name": "…", "email": "…" },
  "keywords": ["sdlc", "agents", "workflow", "artifacts", "mcp"],
  "min_server_version": "1.4.0",
  "mcpServers": {
    "asdlc": {
      "type": "http",
      "url": "${ASDLC_SERVER_URL:-http://localhost:8080/mcp}",
      "headers": { "Authorization": "Bearer ${ASDLC_TOKEN}" }
    }
  }
}
```

### Agent definitions stay thin

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

That is the whole agent file. The real prompt arrives at runtime.

### `gate_decide` must be a command, not a tool

Slash commands are **user-invoked**; an agent cannot self-trigger a `commands/` entry. Exposing gate
approval as a command rather than a tool is a structural guard against agent self-approval
(invariant **I5**) that complements the token-role check.

### Marketplace

```jsonc
// .claude-plugin/marketplace.json  (distribution repo root)
{
  "name": "namz-asdlc",
  "owner": { "name": "…", "url": "…" },
  "plugins": [
    { "name": "asdlc",          "source": "./dist/claude",          "description": "Core agentic SDLC workflow", "strict": true },
    { "name": "asdlc-security", "source": "./dist/claude-security", "description": "Threat modelling + security review stages", "strict": true }
  ]
}
```

Install:

```bash
/plugin marketplace add <owner>/agentic-sdlc
```

Point the marketplace at a **git tag**, never `main` — otherwise every user gets every
mid-development commit.

### Version compatibility

```
Plugin version   1.4.0    ← what users install (git tag)
Server version   1.4.x    ← must be >= plugin's min_server_version
Prompt versions  architect@2.3.1, planner@1.4.0 …   ← independent, server-side, move constantly
```

`min_server_version` is checked on the **first tool call**. It fails loudly with a clear message
rather than failing mysteriously mid-run — the difference between a 30-second fix and a lost
afternoon.

### Acceptance

- [ ] Plugin installs from the marketplace and connects on the first try
- [ ] An outdated server produces a clear version-mismatch message at bootstrap, not mid-run
- [ ] Agents cannot invoke the approve command
- [ ] A prompt-version bump changes agent behaviour with no reinstall

---

## M5.4 — Cursor, Copilot and AGENTS.md targets

### AGENTS.md is canonical (D4)

Generated first; every other bridge derives from it. Read natively by 30+ tools.
`dist/agents-md/AGENTS.md`, ~60 lines.

### Cursor

```
dist/cursor/
  .cursor/rules/asdlc-core.mdc       ← alwaysApply: true
  .cursor/rules/asdlc-design.mdc     ← globs: docs/adr/**
  .cursor/rules/asdlc-testing.mdc    ← globs: **/*.test.*, **/*_test.*
  .cursor/mcp.json
```

```mdc
---
description: ASDLC workflow — always active
alwaysApply: true
---
<!-- generated from packs/core-sdlc — do not edit -->
[bootstrap content inlined]
```

Glob-scoped rules are Cursor's one genuine capability the other tools lack — a rule auto-loads when
matching files are open. Keep the core loop always-applied; use scoped rules for stage-specific
reminders.

### Copilot — two tracks

| Track | Mechanism |
|---|---|
| Copilot **coding agent** | Reads `AGENTS.md` natively. No bridge needed. |
| Copilot **Chat** in the IDE | `.github/copilot-instructions.md`, with the bootstrap **inlined** — Copilot Chat does not follow file references reliably |

```markdown
<!-- .github/copilot-instructions.md — generated, do not edit -->
See [AGENTS.md](../AGENTS.md) for the full ASDLC workflow.

[bootstrap content inlined, because Copilot Chat does not follow file references reliably]
```

MCP config goes in `.vscode/mcp.json`.

**Copilot's MCP support is the least mature of the three.** Design the CLI as the survivable
fallback:

```bash
asdlc artifact put --type prd --file ./prd.md --run <id>
```

This is why M1.8 required CLI feature parity. Copilot is also where M0's deferred conformance test
finally runs — and if it fails, the honest answer is *"Copilot users get the CLI, not the agent
loop,"* stated plainly in the README rather than papered over.

### Cross-tool conformance

Re-run the **M0.6 conformance suite** against all three tools using the generated bridges. Same
metrics, same targets. This is the regression test for the whole packaging layer: it proves the
compiled output actually reproduces the behaviour M0 measured by hand.

### Acceptance

- [ ] `AGENTS.md` is generated first and every bridge derives from it
- [ ] Cursor scoped rules activate on matching file patterns
- [ ] Conformance suite passes on Claude Code and Cursor via generated bridges
- [ ] The Copilot result is recorded honestly, whatever it is, and the README reflects it

---

## M5.5 — `asdlc init` in a user repo

```bash
asdlc init [--project <id>] [--server <url>] [--target claude|cursor|copilot|all]
```

Writes:

```
AGENTS.md                          ← canonical, ~60 lines
CLAUDE.md                          ← bridge: @AGENTS.md + Claude-specific extras
.github/copilot-instructions.md    ← generated bridge
.cursor/rules/asdlc.mdc            ← generated bridge, alwaysApply: true
.mcp.json / .cursor/mcp.json / .vscode/mcp.json
.asdlc/config.yaml                 ← project id, server url, repos
```

Detects installed tools and writes only the relevant bridges by default. Never overwrites an existing
`AGENTS.md` or `CLAUDE.md` without `--force` — it merges its section under a marked, regenerable
block, because these files usually already contain content the user wrote.

### The one-liner

```bash
npx @asdlc/cli init && npx @asdlc/cli project init
```

Then in any tool: `/asdlc start "Add SSO via OIDC"`.

### Acceptance

- [ ] `asdlc init` in a repo with an existing `CLAUDE.md` preserves user content
- [ ] Re-running is idempotent
- [ ] Generated bridges are byte-identical to `asdlc build` output
- [ ] `asdlc doctor` after init reports a working connection

---

## M5.6 — Git/PR approval channel and notifier

**Delivers:** the third approval channel, writing the same `gate_events` row (**D5**).

### PR channel mechanics

```
sdlc_stage_submit
   │
   ├─ worker pushes artifacts to asdlc/<run_slug>
   ├─ opens/updates a PR:
   │     title:  [ASDLC] run/<run_slug> — <stage> gate
   │     labels: asdlc · asdlc:gate · asdlc:stage/<stage>
   │     body:   stage · artifacts · changes since last version
   │             auto-check results · criteria needing human judgement
   └─ check `asdlc/gate:<stage>` → pending
              │
   ┌──────────┴──────────────────────────────┐
   │ human acts on the PR                     │
   ├─ Approve review     → approved           │
   ├─ Request changes    → changes_requested  │  review comments extracted
   │                                          │  into the feedback JSONB
   └─ Close PR           → rejected           │
              │
   webhook → gate_event → check resolves → run advances
```

### Binding to the developer's own PR

For `develop` specifically, `bind_to_pr: true` attaches the gate to the developer's existing code PR
instead of creating a second one. **This is the answer to Q4** — review findings land where reviewers
already look, rather than in a parallel artifact nobody opens.

### Fallback without webhooks

Air-gapped or restricted networks: the worker polls the git API on an interval (default 60 s).
Slower, but it works, and it keeps webhook availability from being a hard requirement.

### Caveat worth stating in the docs

PR approval requires PR-write permission, which is **coarser** than "may approve a design document."
Teams with tight branch protection may find the web channel is a better fit for design-stage gates
even if PR is right for `develop`.

### Notifications

| Event | Web | Git | Chat | Email |
|---|---|---|---|---|
| Gate opened | badge + queue | PR + check | message w/ buttons | digest |
| SLA 50% elapsed | — | — | reminder | — |
| SLA breached | banner | check → failure | @mention | immediate |
| Gate decided | queue update | check resolved | thread reply | — |
| Run completed | timeline | PR ready to merge | summary | digest |
| Budget 80% | banner | — | @owner | — |

**Digest by default, not firehose.** Only SLA breaches and blockers notify immediately. A system that
pings on every event trains people to ignore it — at which point gate latency (G1) gets worse, not
better.

Slack buttons write the same `gate_event` with `channel=chat` and carry the **same guardrails** as
the IDE channel.

### Acceptance

- [ ] A PR approval writes a `gate_event` and advances the run within 30 s
- [ ] Requesting changes extracts review comments into structured feedback
- [ ] The polling fallback works with webhooks disabled
- [ ] `bind_to_pr` produces one PR for the develop stage, not two
- [ ] Notification volume for a full six-stage run is ≤ 3 immediate messages

---

## M5.7 — Docs, demo and the onboarding test

**At this point the docs are the product.**

### Required documentation

| Doc | Answers |
|---|---|
| Getting started | Zero → completed run, in order, with copy-pasteable commands |
| Installing per tool | Claude Code / Cursor / Copilot, one section each |
| Operating the stack | `make` targets, backup, upgrade, doctor |
| Authoring roles | How to fork a checklist, add a stage, bump a prompt |
| Troubleshooting | The ten things that actually go wrong, with the fix |
| What this is not | Non-goals, honest limitations, the Copilot caveat |

"What this is not" is not optional. A tool that oversells itself in its README loses trust on day
two, and that takes far longer to earn back than it took to lose.

### The 5-minute demo

Recorded, scripted, reproducible: `/asdlc start` → brief → PRD → gate → approve → next stage. Under
five minutes. If it cannot be done in five minutes, the onboarding is not ready.

### G5 — the onboarding test

> Hand a colleague the docs and a machine. **Do not help.** Time them.
>
> **Target: zero → completed run in under 30 minutes.**

Watch where they stall. Every stall is a documentation bug, not a user error. Fix the docs, then run
it again with a different person.

This is the highest-signal test in the entire plan. Everything up to here was verified by the person
who built it, which means every implicit assumption still passes. A stranger with the docs is the
only way to find them.

### Acceptance

- [ ] All six documents written
- [ ] Demo recorded, under five minutes
- [ ] **G5 passed with at least two different people**
- [ ] Every stall observed is either fixed or recorded as a known issue

---

## M5 exit criteria

- [ ] A colleague goes from zero to a completed run in under 30 minutes using only the docs
- [ ] Identical behaviour verified across Claude Code, Cursor and Copilot on the same run — or the gap is documented honestly
- [ ] A prompt fix ships to all users with no reinstall
- [ ] Golden-file tests guard every generated file
- [ ] PR approval channel works, including the no-webhook fallback
