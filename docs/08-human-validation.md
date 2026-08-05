# 08 — Human Validation Gates

Three approval channels, one gate record. Humans stay accountable for what ships.

---

## The gate model

A gate opens when a stage submits and its automated checks pass. It closes on a human decision.

```
stage_submit
     │
     ├─ automated checks fail ──▶ returned to agent (no gate, no human time wasted)
     │
     └─ automated checks pass
              │
         ┌────▼──────────────────────────────────────┐
         │  GATE OPEN                                 │
         │  notified via: web UI · PR · Slack · IDE   │
         └────┬───────────────────────────────────────┘
              │
    ┌─────────┼───────────────────┬──────────────────┐
    ▼         ▼                   ▼                  ▼
 approved  changes_requested   rejected           expired
    │         │                   │                  │
 next stage  back to             run halts       escalate /
            in_progress          (needs           auto-decide
            with feedback        run_restart)     per policy
```

**Automated checks run before humans see anything.** Reviewing an artifact that fails its own schema is a waste of the scarcest resource in the system. This ordering is the main reason gates stay tolerable.

---

## Policy tiers

Adopting the standard three-tier model for agent actions:

| Tier | Behaviour | Default stages |
|---|---|---|
| `auto` | Agent proceeds; decision logged, no human wait | (none by default) |
| `notify` | Agent proceeds; human is notified and can retroactively object within a window | `document` |
| `block` | Agent stops; human must decide before the next stage | `plan`, `design`, `develop`, `review`, `test` |

```yaml
# projects/acme-api/policy.yaml
gates:
  defaults: { policy: block, approvers: ["@namz"], sla_hours: 24 }
  stages:
    document: { policy: notify, objection_window_hours: 48 }
    review:   { policy: block, approvers: ["@namz", "@tech-lead"], quorum: 1 }
  escalation:
    on_sla_breach: notify_and_extend      # notify_and_extend | auto_approve | halt
    max_extensions: 2
  auto_loops:
    max: 2          # reviewer→develop rounds before forcing a human in
```

`auto_approve` on SLA breach is available but **off by default and loudly warned about**. It converts a review system into a delay system.

### Risk-based escalation

Some changes deserve more scrutiny than others. The gate policy can escalate based on artifact content:

```yaml
  escalate_when:
    - { condition: "artifact.type == 'dependency_change'",     approvers: ["@security"] }
    - { condition: "security_findings.severity >= 'major'",     approvers: ["@security"], quorum: 1 }
    - { condition: "diff.files matches 'infra/**'",             approvers: ["@platform"] }
    - { condition: "adr.labels contains 'irreversible'",        quorum: 2 }
```

This is where the "auto for safe reversible actions, block for irreversible or high-stakes" principle earns its keep — most gates stay one-click, and the ones that matter get real attention.

---

## Channel 1 — Git / PR

The default for teams already living in PRs. Zero new UI, plugs into existing review habits and existing branch protection.

**How it works**

1. On `stage_submit`, the worker pushes approved-pending artifacts to `asdlc/<run_slug>` and opens (or updates) a PR.
2. The PR body is generated from the gate payload: what stage, what artifacts, what changed since last version, which acceptance criteria passed automatically, what needs human judgement.
3. A GitHub/GitLab check named `asdlc/gate:<stage>` sits pending.
4. The human reviews. **Approving the PR review** → `approved`. **Requesting changes** → `changes_requested`, with review comments extracted into the feedback payload. **Closing** → `rejected`.
5. A webhook writes the `gate_event`; the check flips to success and the run advances.

```
PR title:  [ASDLC] run/add-sso-oidc — design gate
Labels:    asdlc, asdlc:gate, asdlc:stage/design
Checks:    asdlc/gate:design  ⏳ awaiting human approval
```

**Fallback without webhooks** (air-gapped, or self-hosted git with no hook access): the worker polls. Configurable interval, default 60s.

**Why the PR is the artifact bundle, not the code.** The `develop` stage's code lives in the developer's normal branch and normal PR. The ASDLC PR carries the *documents* — PRD, ADRs, test plan. For `develop` specifically, the gate can be bound to the developer's existing code PR instead (`bind_to_pr: true`), so there's one PR, not two.

**Caveat.** Requires a git host with an API and credentials. It also means approval requires PR-write permission, which is a coarser grant than "can approve a design doc" — worth knowing if your reviewers aren't all committers.

---

## Channel 2 — Web dashboard (`approvals-ui` :3000)

The richest channel, and the only one non-developers will use.

**Pages**

| Page | Contents |
|---|---|
| **Queue** | All gates awaiting me, sorted by SLA remaining. Age, run, stage, requester, risk flags. |
| **Gate detail** | Rendered artifacts (markdown, mermaid diagrams, OpenAPI via Swagger UI, diffs syntax-highlighted). Acceptance-criteria checklist with auto-results. Provenance panel: what this was derived from, which knowledge and code informed it, which prompt version. |
| **Diff view** | Version-to-version diff when a stage is resubmitted after `changes_requested` — reviewers should see *what changed*, not re-read everything. |
| **Run timeline** | Every stage, gate, artifact, and cost. The audit view. |
| **Knowledge curation** | Candidate entries with nearest existing neighbours, batch approve/reject. |
| **Spend** | Cost per run, per stage, per role, from LiteLLM spend logs. |

**Decision actions:** Approve · Request changes (requires a comment) · Reject (requires a reason) · Delegate · Snooze.

**Requiring a comment on `changes_requested` is not bureaucracy** — it's the input to the next iteration's brief. A bare rejection produces a re-run that repeats the mistake.

**Design principle for the gate detail page:** a reviewer should be able to make a confident decision in under three minutes without opening another tab. That means rendering, not linking; showing auto-check results inline; and surfacing the specific things the automated checks *couldn't* verify at the top.

---

## Channel 3 — In-IDE (MCP)

For solo developers who never leave the editor. Fastest loop, weakest accountability.

```
/asdlc gates                       → gate_status
/asdlc approve <gate_id> "lgtm"    → gate_decide
/asdlc changes <gate_id> "the TTL should be 30m, not 24h"
```

**Guardrails — this channel is the one with a real failure mode.** The risk is an agent approving its own work, either by being asked to "just approve it" or by hallucinating a call.

1. `gate_decide` is **rejected** unless `ASDLC_ALLOW_IDE_APPROVAL=true` (default `false` in team mode, `true` in solo mode).
2. The calling token must carry `role=approver`. Agent tokens don't.
3. The gate's `produced_by_user` cannot equal the approver unless `allow_self_approval=true` (default `true` solo, `false` team).
4. Every IDE approval is recorded with `channel=ide` and flagged in the audit view — so a team can spot the pattern if it becomes a habit.
5. Slash commands are defined in the plugin as **user-invoked commands**, not tools the agent can call autonomously. In Claude Code that means a `commands/` entry; the agent cannot self-trigger it.

Even so: **in-IDE approval is the weakest control in this system.** It's included because for one developer on their own project the alternative isn't a better review — it's abandoning the workflow. Be honest about the trade in team settings and default it off.

---

## Channel 4 (optional) — Chat

Slack/Teams notification with Approve / Request-changes buttons. Not a separate design — it's a notifier plus a webhook that writes the same `gate_event`. Same guardrails as Channel 3 apply (identity must map to an approver role).

Worth adding once the web UI exists, because notification is where gates actually stall. A gate nobody knows about is indistinguishable from a broken pipeline.

---

## Gate record

```sql
CREATE TYPE gate_decision AS ENUM
  ('approved','changes_requested','rejected','skipped','expired','auto_approved');

CREATE TABLE gates (
  id            TEXT PRIMARY KEY,       -- gat_01J...
  run_id        TEXT NOT NULL REFERENCES runs(id),
  stage         TEXT NOT NULL,
  status        TEXT NOT NULL,          -- open|closed
  policy        TEXT NOT NULL,          -- auto|notify|block
  required_approvers TEXT[] NOT NULL,
  quorum        INT NOT NULL DEFAULT 1,
  artifact_ids  TEXT[] NOT NULL,
  auto_checks   JSONB NOT NULL,         -- [{id, name, result, detail}]
  opened_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  sla_at        TIMESTAMPTZ,
  closed_at     TIMESTAMPTZ
);

CREATE TABLE gate_events (
  id            TEXT PRIMARY KEY,       -- gev_01J...
  gate_id       TEXT NOT NULL REFERENCES gates(id),
  decision      gate_decision NOT NULL,
  actor         TEXT NOT NULL,          -- user id; never an agent
  actor_type    TEXT NOT NULL,          -- human|system
  channel       TEXT NOT NULL,          -- web|git|ide|chat|api
  comment       TEXT,
  feedback      JSONB,                  -- structured, fed into the next brief
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON gate_events (gate_id, created_at);
```

`gate_events` is **append-only** and is the audit spine of the whole system. Never update; never delete. A "changed my mind" is a new event.

---

## Feedback loop

`changes_requested` feedback is structured so the next brief can use it precisely:

```jsonc
{
  "decision": "changes_requested",
  "comment": "Session TTL is wrong and ADR-014 doesn't consider the multi-region case.",
  "feedback": {
    "items": [
      { "artifact_id": "art_adr_014", "severity": "major",
        "locator": "## Decision",
        "issue": "TTL of 24h contradicts the security baseline of 30m for privileged sessions.",
        "expected": "30m sliding, 8h absolute" },
      { "artifact_id": "art_adr_014", "severity": "major",
        "locator": "## Consequences",
        "issue": "No consideration of multi-region Redis replication lag." }
    ],
    "keep": ["The Redis-over-cookie decision is right; don't revisit it."]
  }
}
```

The next `sdlc_stage_start` includes this verbatim plus the rejected artifact as an input. The `keep` array matters: without it, agents frequently "fix" the feedback by rewriting everything, including the parts that were fine.

---

## Notification matrix

| Event | Web | Git | Chat | Email |
|---|---|---|---|---|
| Gate opened | badge + queue | PR + check | message w/ buttons | digest |
| SLA 50% elapsed | — | — | reminder | — |
| SLA breached | banner | check → failure | @mention | immediate |
| Gate decided | queue update | check resolved | thread reply | — |
| Run completed | timeline | PR ready to merge | summary | digest |
| Budget 80% | banner | — | @owner | — |

Digest, not firehose. Per-user notification preferences, default to digest for everything except SLA breach and blockers.

---

## Sources

- [Agentic SDLC — Augment Code](https://www.augmentcode.com/guides/agentic-sdlc)
- [The Agent-Run Loop: Reframing the SDLC as a Continuous Cycle](https://www.augmentcode.com/guides/agent-run-development-loop)
- [What you need to know about Agentic SDLC in 2026](https://www.dronahq.com/agentic-sdlc-guide/)
