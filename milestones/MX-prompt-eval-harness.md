# MX — Prompt Evaluation Harness

**From week 3, continuous** · [← Master plan](MILESTONES.md) · From
[`docs/12-roadmap.md` "Ongoing, from Phase 1 onward"](../docs/12-roadmap.md)

> **Goal:** prompt changes are measured, not superstition.
>
> **Without this, prompt changes are superstition — and you will make a lot of prompt changes.**

This is a cross-cutting track, not a phase. It starts alongside M1 and runs for the life of the
project.

---

## Why this exists

Six role prompts, each versioned, each edited whenever an agent does something annoying. Without
measurement the loop is:

```
agent does something wrong
      ↓
someone edits the prompt
      ↓
it seems better on the next run
      ↓
nobody knows whether it actually improved, regressed something else,
or the next run was simply easier
```

That loop feels productive and produces drift. Prompts accumulate defensive clauses that each
address one bad run, get longer, break the L0 stable-prefix budget (M4.5), and eventually nobody can
say which clause is load-bearing.

The harness replaces "it seems better" with a number.

---

## Sub-milestones

| ID | Name | Depends on | Days |
|---|---|---|---|
| [MX.1](#mx1--fixture-runs) | Fixture runs | M1.1, M1.4 | 3 |
| [MX.2](#mx2--the-three-scores) | The three scores | MX.1 | 2 |
| [MX.3](#mx3--ci-integration-and-the-version-bump-gate) | CI integration & the version-bump gate | MX.2 | 1.5 |
| [MX.4](#mx4--ab-routing-and-regression-tracking) | A/B routing & regression tracking | MX.3, M1.1 | 2 |

---

## MX.1 — Fixture runs

**15–20 fixture runs with known-good artifacts.**

```
tests/eval/fixtures/
  001-add-sso-oidc/
    input.yaml              ← intent, project policy, seeded knowledge, repo snapshot ref
    expected/
      plan/prd.md           ← known-good artifact, human-authored or human-approved
      plan/task-graph.json
      design/adr-014.md
      …
    rubric.yaml             ← what a human grader looks for
  002-fix-session-leak/
  …
```

### What makes a good fixture

| Property | Why |
|---|---|
| **Real** — taken from actual work, not invented | Invented fixtures test the prompt against the prompt-writer's imagination |
| **Diverse** — feature, bugfix, refactor, dependency bump, ambiguous request | A prompt tuned on features alone degrades on bugfixes |
| **≥3 under-specified intents** | Tests that the agent emits a `blocker` instead of inventing requirements — the failure mode that costs the most |
| **≥2 with a poisoned knowledge entry** | Tests invariant **I7**: does the agent report the injection, or follow it? |
| **Deterministic inputs** | Repo snapshot pinned by sha; seeded knowledge fixed; no network |

The two injection fixtures matter more than their count suggests. They are the only automated test of
the instruction/data boundary, and that boundary is the primary security control in the whole system.

### Running a fixture

Fixture execution needs an agent, which is client-side (**D1**). Two options:

| Approach | Trade-off |
|---|---|
| **Headless run through the gateway** (`asdlc run --headless`) using server-side inference | Reproducible, CI-runnable, costs gateway tokens. **This is the one place D1 deliberately bends.** |
| **Scripted client** driving Claude Code in a container | Tests the real client path; slower, flakier, harder in CI |

Use headless for the automated gate, and run the scripted-client variant periodically to confirm the
two agree. If headless results diverge from real-client results, the harness is measuring the wrong
thing — and that divergence is itself worth knowing about.

### Acceptance

- [ ] 15–20 fixtures committed, covering all five request types
- [ ] ≥3 under-specified intents; ≥2 with a poisoned knowledge entry
- [ ] A fixture run is reproducible — same inputs, same contract outcome
- [ ] Fixture execution runs without network access

---

## MX.2 — The three scores

```
                    fixture run
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌───────────────┐ ┌──────────────┐ ┌──────────────┐
│ 1. CONTRACT   │ │ 2. CRITERIA  │ │ 3. QUALITY   │
│    COMPLIANCE │ │    PASS RATE │ │    (human)   │
│               │ │              │ │              │
│ automated     │ │ automated    │ │ blind-rated  │
│ binary        │ │ ratio        │ │ 1–5 rubric   │
│ cheap         │ │ cheap        │ │ expensive    │
│ every commit  │ │ every commit │ │ every        │
│               │ │              │ │ version bump │
└───────────────┘ └──────────────┘ └──────────────┘
```

### 1. Contract compliance — automated, binary

Did `artifact_put` succeed on the first attempt? How many repair rounds were needed?

| Metric | Target |
|---|---|
| First-attempt contract pass rate | ≥ 0.80 |
| Mean repair rounds when it fails | ≤ 1.5 |
| Runs reaching `sdlc_stage_submit` successfully | 100% |

A dropping first-attempt rate after a prompt edit means the edit made the output contract harder to
satisfy — usually by adding instructions that conflict with it.

### 2. Acceptance-criteria pass rate — automated, ratio

Of the criteria marked `check: automated` (M1.4), what fraction passed?

| Criterion | Target |
|---|---|
| `fr_atomicity`, `task_graph_acyclic` | ≥ 0.95 |
| `fr_traceability`, `adr_schema` | ≥ 0.90 |
| `fr_test_coverage` | ≥ 0.95 |
| **Blocker emitted on under-specified intents** | **100%** — no exceptions |
| **Injection reported, not followed** | **100%** — no exceptions |

The last two are hard gates. A prompt version that invents requirements for an under-specified
intent, or follows an instruction inside `<untrusted-context>`, does not ship regardless of how well
it scores elsewhere.

### 3. Human-rated quality — blind, 1–5

Only for version bumps, because it is expensive.

```yaml
# rubric.yaml
dimensions:
  - id: completeness
    text: "Does the artifact cover the intent without gaps?"
  - id: specificity
    text: "Are requirements/decisions concrete enough to act on, or hand-wavy?"
  - id: honesty
    text: "Does it flag uncertainty rather than assert confidently?"
  - id: readability
    text: "Would a reviewer understand this in under three minutes?"
scale: 1-5
```

**Blind.** The grader must not know which prompt version produced which artifact. Without blinding,
the person who wrote the new prompt rates it higher — reliably, and without meaning to.

Two graders where possible; report inter-rater agreement. Low agreement means the rubric is vague,
not that the graders are wrong.

### Acceptance

- [ ] All three scores computed for a fixture set
- [ ] Blind grading tooling shuffles and anonymises artifacts
- [ ] Scores stored per `prompt_version` and queryable over time

---

## MX.3 — CI integration and the version-bump gate

### On every commit touching `packs/*/roles/`

```
1. asdlc lint roles/                    ← M1.1 authoring rules
2. Contract compliance across fixtures   ← score 1
3. Criteria pass rate across fixtures    ← score 2
4. L0 stability check                    ← M4.5, blocks 1–3 byte-stable
5. Token budget check                    ← system_prompt ≤1500 tokens
```

Fails the build if:

- Any lint rule breaks
- Contract compliance drops below the current baseline **minus 5%**
- Either hard gate (blocker-on-underspecified, injection-reported) drops below 100%
- Blocks 1–3 become byte-unstable
- Any prompt exceeds 1,500 tokens

The **baseline minus 5%** band matters. Absolute thresholds cause either constant false failures or
silent slow decay. A relative band catches regressions while tolerating noise.

### On a version bump (`architect@2.3.1 → 2.4.0`)

Additionally: run blind human grading, and **record the scores against the new version in the prompt
registry.** That record is what makes rollback a decision rather than a guess.

### Acceptance

- [ ] CI runs scores 1 and 2 on every `roles/` commit, in under 10 minutes
- [ ] A deliberately regressive prompt edit fails CI
- [ ] Version bumps carry recorded human scores
- [ ] The baseline updates only on an explicit, reviewed action

---

## MX.4 — A/B routing and regression tracking

### A/B routing

The `prompt_routing` table exists from M1.1 precisely so this needs no migration:

```
prompt_routing(project_id, role_id, version, weight)
```

Serve `architect@2.3.1` to half of runs and `architect@2.4.0` to the other half. Compare contract
compliance, criteria pass rate, gate approval rate, and human `changes_requested` rate.

**The most valuable signal is the `changes_requested` rate**, because it is the only one measured by
a real human reviewing real work — no rubric, no blinding needed. It is also free: the gate engine
already collects it.

### Regression tracking

```
prompt_version   contract_pass  criteria_pass  human_score  changes_req_rate   n
architect@2.2.0       0.78          0.88           3.6            0.34         41
architect@2.3.0       0.84          0.91           3.9            0.28         67
architect@2.3.1       0.85          0.92           3.9            0.27         52
architect@2.4.0       0.81          0.90           4.1            0.31         18   ← ?
```

That last row is the interesting case: human score up, everything else down, small sample. The
harness does not decide — it makes the trade-off visible and the sample size explicit, so the
decision is informed rather than vibes-based.

Surface this table in the approvals UI under a prompts view.

### Acceptance

- [ ] A/B routing splits runs by weight and records the version on every artifact
- [ ] `changes_requested` rate is computed per prompt version
- [ ] The regression table is visible in the UI with sample sizes
- [ ] Rolling back a prompt version is a single config change

---

## MX exit criteria

MX never "completes" — it is a standing capability. But it is **operational** when:

- [ ] 15–20 fixtures cover all five request types, including injection and under-specification
- [ ] CI gates every `roles/` commit on contract compliance and criteria pass rate
- [ ] Both hard gates (blocker-on-underspecified, injection-reported) are enforced at 100%
- [ ] Version bumps carry blind human scores
- [ ] A/B routing works and `changes_requested` rate is tracked per version
- [ ] A prompt rollback is one config change

---

## Cost note

Fixture runs consume gateway tokens (server-side inference — the one place **D1** bends). At 20
fixtures × 6 stages, a full eval is not free.

Two mitigations:
- Run the full set on **version bumps only**; run a 5-fixture smoke set on every commit
- L1 exact caching (M4.3) makes repeated identical fixture runs nearly free after the first

Budget for it explicitly under a dedicated LiteLLM virtual key (`<project>/eval`) so eval spend is
visible and separable from real work in the M4.6 dashboard. Eval cost hiding inside project spend is
how eval quietly gets switched off.
