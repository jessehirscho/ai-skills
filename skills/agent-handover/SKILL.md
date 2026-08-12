---
name: agent-handover
description: Use whenever a task should be split across multiple LLMs or model tiers with an explicit handover between them — e.g. a strong/expensive model plans, a mid-tier model implements, and a fast/cheap or different-vendor model reviews. Covers when multi-model handover is worth the coordination cost, what to pass at each handover point, how to prevent context loss between models, and how this differs from same-model subagent delegation (`subagents`, `dev-pipeline`).
---

# Multi-LLM Agent Handover

A pattern for routing different phases of one task to different models — chosen for their cost/capability tradeoff or for a genuinely independent second opinion — with a clean, lossless handover between them.

## How this differs from same-model subagent delegation

This repo's `subagents` and `dev-pipeline` skills cover delegating to fresh instances of the *same* underlying model (e.g. Claude planner → Claude builder → Claude reviewer, all Sonnet or a Sonnet/Haiku mix within one vendor's lineup). **Agent handover is about crossing model or vendor boundaries** — e.g. Opus plans, Sonnet implements, a different vendor's model reviews. The same phase-splitting logic applies, but the coordination cost is higher because there's no shared session, no shared tool-call history, and often no shared harness conventions (CLAUDE.md vs. AGENTS.md, different tool names, different context window sizes). Everything below assumes you're the one manually gluing the phases together — no runtime automatically hands work from one model's CLI to another's today.

## When it's worth the coordination overhead

Worth it when:
- A phase genuinely benefits from a stronger (and more expensive) model's judgment — architecture decisions, ambiguous requirements, security-sensitive design — while the bulk of mechanical work doesn't need that strength.
- You want a review pass from a model that didn't see the implementer's reasoning trail, on the theory that a fresh, differently-trained perspective catches different mistakes than the same model grading its own work (this is the strongest, most defensible reason to cross vendors specifically, not just cross tiers).
- Cost matters at scale and the task decomposes cleanly into a small "hard" phase and a large "mechanical" phase.

Not worth it when:
- The task is small enough that handover overhead (writing a spec precise enough to survive the handoff, re-establishing context, reconciling two models' conventions) costs more than just picking one capable model and doing the whole thing.
- The phases are tightly interleaved rather than cleanly sequential — constant back-and-forth between "planning" and "implementing" decisions doesn't tolerate a hard handover boundary.
- You don't have a way to verify the receiving model actually understood the handoff (see verification below) — an unverified handover is worse than no handover, because a subtly wrong plan implemented faithfully still produces a wrong result.

## The handover artifact

The single most important design decision: **the plan/spec that crosses the boundary must be a complete, self-contained written artifact, not a conversation.** The receiving model has zero access to the sending model's chat history, reasoning trace, or any context that wasn't written down. Treat it like briefing a contractor who will never talk to the architect — if it's not in the document, it doesn't exist for them.

A handover artifact should contain:
1. **Problem statement** — what and why, in 2-4 sentences, no ambiguity about scope.
2. **Constraints** — anything the receiving model must not violate (interfaces it can't change, files out of scope, performance/security requirements).
3. **The decision, not just the goal** — if the sending model chose an approach among alternatives, write down the approach chosen and enough of the "why" that the receiving model doesn't second-guess or silently substitute its own judgment. Vague goals invite scope drift; a decided plan doesn't.
4. **Acceptance criteria** — the specific, checkable condition that means the phase succeeded (a test to run, a behavior to verify, a diff shape to match).
5. **Explicit non-goals** — what's deliberately out of scope, so the receiving model doesn't "helpfully" expand it.

This is the same discipline as this repo's `dev-pipeline` skill's planner→builder handoff (`/plans/<ticket>.md`), just crossing a harder boundary with no shared session to fall back on if something's missing.

## A concrete three-phase example: plan → implement → review

1. **Plan (strong/expensive model).** Give it the problem statement, constraints, and codebase context it needs to make the hard calls. Output: one written plan document per the artifact structure above. The planning model does not touch code.
2. **Implement (mid-tier model).** Give it *only* the plan document and the exact file paths it names — not the planning conversation. It implements exactly what the plan says; if the plan turns out wrong once it's in the code, it should stop and flag rather than improvise past the plan's stated intent.
3. **Review (a different model/vendor from the implementer, ideally also different from the planner).** Give it the diff and the plan — not the implementer's reasoning trail. It checks the diff against the plan, not against its own idea of what "should" have been built. A cross-vendor reviewer is most valuable here specifically because it has no shared blind spots with whichever model wrote the plan or the code.

Each arrow above is a handover artifact boundary: plan.md, then diff + plan.md, then a verdict. Nothing else crosses.

## Practical mechanics for gluing this together

- **You are the orchestrator.** Unless your harness has native cross-model agent dispatch, you (the human, or the session coordinating this) manually copy the handover artifact from one model's session into the next model's session. Don't skip writing the artifact to disk just because you're about to paste it — a file in the repo (e.g. under `/plans/`) is recoverable if the handover needs to happen again or be audited later.
- **Normalize conventions before handoff.** If the two harnesses use different instructions-file names (CLAUDE.md vs. AGENTS.md) or different tool-call conventions, don't assume the receiving model infers your setup — say so explicitly in the handover artifact if it matters (e.g. "run tests with `pytest -q`, this repo has no `npm test`").
- **Verify the handover landed, don't assume it.** Before letting the receiving model start real work, have it restate its understanding of the task back in its own words, or check its first concrete action against the plan. Catching a misread spec before 500 lines of code get written is far cheaper than catching it in review.
- **Keep a handover log.** A short running note of what crossed each boundary and when (even just commit messages referencing the plan doc) makes it possible to trace a bad outcome back to which phase introduced it — same instinct as the `dev-pipeline` skill's rule that a reviewer only gets the diff, not the reasoning trail, so problems are traceable to a specific artifact rather than lost in a long unrecorded conversation.

## Common pitfalls

- **Treating a chat transcript as the handover artifact.** "I'll just paste our conversation" loses the decision under a pile of exploration and dead ends; the receiving model has to re-derive what actually got decided, and often gets it wrong.
- **Skipping acceptance criteria.** Without a checkable definition of done, the review phase becomes subjective vibes-checking instead of verification against the plan.
- **Using a weaker model for review than for implementation.** The review phase needs at least as much judgment as implementation, often more — it's catching mistakes the implementer didn't see, not just checking boxes.
- **Over-engineering the handover for a task that didn't need multiple models.** If you find yourself writing a longer spec than the task itself would have taken to just do with one capable model, that's a sign the split wasn't warranted (see "when it's worth it" above).
- **No fallback when a handover clearly failed.** Decide up front what happens if the implementer flags the plan as wrong, or the reviewer rejects the diff — does it go back to the planner, or does a human step in? An undefined failure path turns into an ad hoc scramble mid-task.

## Quick reference

| Question | Answer |
|---|---|
| What crosses each boundary? | A written artifact (plan, diff+plan, verdict) — never a raw conversation |
| Who verifies the handover landed? | The receiving model, before starting real work — restate understanding or check first action |
| How is this different from `dev-pipeline`? | Same phase-splitting logic, but crossing model/vendor boundaries with no shared session, so the artifact discipline matters even more |
| When to skip this entirely? | Task small enough that one capable model doing it end-to-end is cheaper than writing a handover-quality spec |
| Best use of a cross-vendor review phase? | Catching blind spots the planning/implementing model(s) share — not just double-checking mechanical correctness |
