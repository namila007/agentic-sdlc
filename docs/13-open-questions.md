# 13 — Open Questions

Decisions I made a call on so the design would be concrete, but which are genuinely yours. Grouped by how much rework a change costs later.

---

## Expensive to change later — decide before Phase 1

### Q1. Is `develop` really one stage, or should code changes be per-task gates?

**I assumed:** one `develop` stage with internal fan-out across the task graph, converging on a single gate.

**The alternative:** a gate per task node. Much finer-grained review, much more human interruption.

**Why it matters:** the run state machine and the gate schema differ. Retrofitting per-task gates means changing `gates.stage` into `gates.scope` and reworking the UI queue.

**My lean:** one gate, with the *option* of `gate_per_task: true` in project policy for high-risk work. Confirm this is acceptable.

---

### Q2. Where does the code actually live during `develop`?

Three models, and they're not equivalent:

| Model | How | Trade-off |
|---|---|---|
| **A — agent edits the working tree** *(assumed)* | Agent uses its native file tools; ASDLC records the diff as an artifact | Natural for IDE users; ASDLC has no copy until `artifact_put`; conflicts are the developer's problem |
| **B — agent works on a dedicated branch** | ASDLC creates `asdlc/run-x/task-y`, agent commits there | Cleaner isolation, easier rollback; awkward in an IDE, branch-switching mid-session |
| **C — server-side worktree** | Control plane clones and the agent operates via tools | Full control and reproducibility; contradicts [D1](00-overview.md#d1--agents-are-client-side) and loses IDE integration |

**My lean:** A, with B available per-project. But if your users are mostly doing large refactors across many files, B's isolation starts to matter.

---

### Q3. One control plane per team, or per developer?

**I assumed:** one shared instance per team, self-hosted, with multi-project support — driven by your "team/multi-project from day one" answer.

But you also selected "solo/personal use for now." Those pull in different directions:

- **Shared instance** → knowledge and index are shared (the main value), but needs auth, network access, and someone to operate it.
- **Per-developer instance** → trivial to run, but each developer's knowledge center starts empty and diverges. You'd need a sync mechanism, which is a whole other design.

**My lean:** build shared-capable (project scoping everywhere), run it solo on localhost initially. `ASDLC_AUTH_MODE=none` makes solo painless. Sanity-check that this is what you meant.

---

### Q4. Does the `review` stage duplicate your existing PR review?

If your team already does human PR review, ASDLC's `review` stage is a *second* review — one by an agent, one by a human — plus a gate on the agent's review. That can feel like ceremony.

**Options:** (a) keep it, agent review catches different things and is cheap; (b) merge `review` into the `develop` gate as an automated check rather than a stage; (c) make `review` produce comments *on the existing PR* rather than a separate artifact.

**My lean:** (c) is probably the best of both — the review findings land where reviewers already look. Costs a git-host integration.

---

## Moderate cost — decide before Phase 3

### Q5. How much does the knowledge center overlap with what you already have?

If you already have an ADR directory, a Confluence space, or a well-maintained `docs/`, the knowledge center should *index* those rather than become a competing store. If you don't have any of that, the knowledge center becomes the primary home and needs write UX, not just curation UX.

Which is it for your projects? It changes whether the curation UI is a review queue or a full editor.

---

### Q6. What's the actual repo profile?

Sizing, language grammar priority, and whether cross-repo search is on by default all depend on this:

- How many repos per project, roughly?
- Which languages, in order of volume?
- Largest repo, in LOC?
- Monorepo or polyrepo?

I sized for 3–5 repos, ≤500k LOC total, and assumed you want cross-repo search on. Tell me if that's off by an order of magnitude in either direction.

---

### Q7. Air-gapped: is it a real requirement or a nice-to-have?

You selected both "must support self-hosted/air-gapped" and options implying hosted models. These are compatible but the effort differs a lot:

- **Nice-to-have** → design for it (which this doc does: gateway indirection, local embeddings), test it occasionally.
- **Real requirement** → local models become the *primary* configuration, which means evaluating whether a 14B local model can actually produce usable ADRs. It probably can't at the quality of Opus/GPT-5, and that changes what you can promise.

**My lean:** design-for, test-quarterly, don't optimise for. But if there's a specific customer driving this, say so — it changes Phase 6 into Phase 2.

---

## Cheap to change — decide when you get there

### Q8. Six stages, or fewer?

`plan → design → develop → review → test → document` is what you specified. For small changes it's heavy. Worth considering a `--fast` run type that collapses to `plan → develop → review`, chosen at run creation based on a risk heuristic (files touched, dependency changes, whether an ADR is implicated).

Not hard to add later; just noting that it will come up on the first bugfix.

### Q9. Context7: passthrough or local mirror?

Passthrough is one config line and works today. Local mirror is a worker, a schedule, and a snapshot format — but it's the only version that works air-gapped. Depends on Q7.

Also worth knowing: Context7 is Upstash-hosted and self-hosting is enterprise-tier only, so don't build a hard dependency either way.

### Q10. Do you want scheduled/autonomous runs?

Nothing in this design starts a run without a human. But "every Monday, run a `document` pass over undocumented public APIs" or "on every dependabot PR, run a `review` stage" are natural extensions.

Adds a scheduler and a headless execution path (which *does* use server-side inference through the gateway — the one place D1 bends). Worth it? Not in v1, I'd say, but it changes whether the gateway needs to be a first-class inference path or stays a utility.

### Q11. Artifact formats — markdown, or something structured?

I chose markdown for PRDs/ADRs/docs (human-reviewable, git-friendly, renders everywhere) and JSON for task graphs/test cases/findings (machine-checkable). The awkward middle is the PRD: FR traceability would be much easier with structured requirements, but structured requirements are miserable to read and write.

Current design splits the difference: markdown body with a YAML front-matter block listing `FR-*` ids and titles, so traceability is computable without making the document unreadable. Flag if you'd rather go fully structured with a rendered view.

---

## Things I'd want to validate empirically, not decide

These aren't opinions to hold — they're experiments to run in Phase 0/1:

1. **Do Cursor and Copilot agents actually obey a server-served brief as reliably as Claude Code?** My guess: Claude Code best, Cursor close, Copilot noticeably worse. If Copilot can't, the honest answer is "Copilot users get the CLI, not the agent loop."
2. **What's the real gate latency?** If humans take 4 hours to approve, the six-stage pipeline has a 24-hour floor and nobody will use it. Measure early; it may force `notify` policies.
3. **Does knowledge retrieval measurably improve artifacts?** Run 10 features with knowledge on and off, blind-rate the artifacts. If there's no difference, the knowledge center is a very expensive filing cabinet.
4. **What's the token cost per run, actually?** Every claim about savings in [06](06-llm-cache-and-gateway.md) is a hypothesis until measured.
