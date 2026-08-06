# Milestones

Implementation plan for the ASDLC control plane. The design lives in [`docs/`](../docs); this folder
turns it into executable work.

## Start here

**[MILESTONES.md](MILESTONES.md)** — the master plan. Milestone map, global invariants, decision
gates, prerequisite decisions, effort model. Read it before opening any individual milestone.

## The milestones

| | File | Weeks | One line |
|---|---|---|---|
| **M0** | [Walking skeleton](M0-walking-skeleton.md) | 1–2 | Prove client-side agents obey a server-served brief, in two tools |
| **M1** | [Full pipeline](M1-full-pipeline.md) | 3–6 | Six stages, real artifacts, gates, UI, git mirror |
| **M2** | [Code indexer](M2-code-indexer.md) | 7–9 | Agents stop reading whole directories |
| **M3** | [Knowledge center](M3-knowledge-center.md) | 10–13 | Run #20 beats run #1 |
| **M4** | [Gateway, cache & cost](M4-gateway-cache-cost.md) | 14–16 | Cost is measured, capped and visible |
| **M5** | [Packaging & distribution](M5-packaging-distribution.md) | 17–19 | Someone else can install it |
| **M6** | [Hardening](M6-hardening.md) | 20+ | Auth, secrets, air-gap, load |
| **MX** | [Prompt eval harness](MX-prompt-eval-harness.md) | 3+ | Prompt changes are measured, not superstition |
| | [Appendix — tech verification](APPENDIX-tech-verification.md) | — | Context7-checked APIs, and what still needs verifying |

## How these are structured

Each major milestone file contains:

- **Scope** — what is in, and what is cut deliberately
- **Sub-milestone table** — `M<n>.<m>`, dependencies, day estimates
- **A diagram** — mermaid or ASCII, for the architecture or flow the milestone builds
- **One section per sub-milestone** — schemas, algorithms, config, thresholds, acceptance criteria
- **Exit criteria** — what must be true to call the milestone done
- **The failure mode it guards against** — and the cheap early check for it

They specify *what to build and how it must behave*, not line-by-line code. Schemas, tool
signatures, algorithms, thresholds and configuration are given concretely enough to implement from.

## Conventions

- **`M<n>.<m>`** — sub-milestone id, stable. Reference these in commits and PRs.
- **`I1`–`I8`** — global invariants, defined in [MILESTONES.md](MILESTONES.md#global-invariants). Breaking one needs an ADR, not a bug fix.
- **`G0`–`G5`** — decision gates. Points where evidence arrives and the remaining plan may change.
- **`Q1`–`Q11`** — open questions from [`docs/13-open-questions.md`](../docs/13-open-questions.md). Q1–Q4 block M1.
- **`D1`–`D6`** — design decisions from [`docs/00-overview.md`](../docs/00-overview.md). A milestone may not silently contradict one.
- ⚠️ — a claim that needs verifying against the installed package before you build on it.

## Before starting M1

Answer **Q1–Q4**. They change the run state machine and the gate schema; deciding them later means
rework rather than a config change. See
[MILESTONES.md § Prerequisite decisions](MILESTONES.md#prerequisite-decisions-block-m1-not-m0).
