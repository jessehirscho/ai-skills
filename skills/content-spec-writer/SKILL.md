---
name: content-spec-writer
description: Use when the user wants to write a well-scoped ticket or spec for a content or feature change BEFORE it goes into a planning/build pipeline. This is the upstream, pre-pipeline step — distinct from dev-pipeline, which assumes a ticket list already exists. Produces the CONTENT_SPEC.md-style entries dev-pipeline's Phase 0/1 consumes directly.
---

# Content Spec Writer: turning a goal into a buildable ticket

The step before `dev-pipeline`. `dev-pipeline` Phase 0 assumes a ticket list already exists with clear scope, goals, and file-overlap flags. This skill is how you produce that list — one tight ticket at a time, or a batch/round of them.

## Why ticket quality matters upstream

A vague ticket — "improve the markets section," "make the blog better" — forces the `planner` subagent in `dev-pipeline` to guess scope. It resolves that guess one of two ways, both bad:

- **Over-broad plan**: the planner invents scope you didn't ask for, touching files you didn't intend, because "improve X" has no boundary.
- **Critique-loop grind**: the critic keeps flagging "unclear what this ticket actually wants," burning both revision rounds without ever converging, and the ticket escalates to the user having wasted a full plan→critique cycle.

A tight ticket avoids both: the planner has exactly enough to bound a plan, and the critic can check the plan against a real acceptance signal instead of guessing at intent.

## The ticket template

Every ticket gets these six fields. Slug and non-goals are the two fields people skip and regret skipping.

```
### `<ticket-id>` — <short human title>
**Goal:** one sentence. What outcome, not what steps.
**Current state:** what exists today, and *specifically* why it's insufficient —
  not "make it better," but the actual gap (missing data, wrong structure,
  broken interaction, stale content, etc.)
**Change:** the specific proposed change. Specific enough to bound scope;
  not so specific it pre-empts the planner's job of deciding *how*.
**Non-goals:** explicit list of what NOT to touch. This is what stops scope creep.
**Files:** exact files/areas this ticket is expected to touch.
**Acceptance:** how anyone — including a reviewer with no other context —
  will know this ticket is done.
```

Field notes:

- **Ticket ID**: short, kebab-case, stable once written. It becomes the branch name (`feature/<ticket-id>`) and the plan filename (`/plans/<ticket-id>.md`) downstream — don't rename it mid-flight.
- **Goal**: one sentence, outcome-oriented. If you can't write it in one sentence, the ticket is probably two tickets.
- **Current state**: this is the field that most often gets skipped or written as filler ("the section could be improved"). Force yourself to name the actual gap — a missing field, a stale number, a UX dead end, a metric that doesn't exist yet.
- **Change**: specific enough that a planner can't wander into unrelated files, loose enough that the planner still gets to decide implementation details (which component, which helper, exact copy).
- **Non-goals**: the highest-leverage field for keeping tickets small. "Do not touch the sidebar." "Do not add new suburb pages." "Do not change the booking API."
- **Files**: name the exact files/areas expected to change. This is the raw material `dev-pipeline` Phase 0 uses to detect file-overlap between tickets before parallel building starts — if you can't name the files, the ticket isn't scoped yet.
- **Acceptance**: a concrete check, not a feeling. "The Markets section shows a 1-line takeaway per data point" is checkable; "the Markets section is better" is not.

## Sizing guidance

**Right-sized** (one planner→builder→reviewer cycle):
- Touches a bounded, nameable set of files (you can list them, not "probably some components").
- Has exactly one acceptance signal, not a bundle of them.
- Doesn't require a design decision so large it needs its own mini-spec first (e.g. "redesign the nav" is not a ticket — decide the nav structure first, then ticket the implementation).

**Too big — split it:**
- Goal needs "and" to state ("add investor commentary AND redesign the layout AND add a new data source") — each "and" is a candidate for its own ticket.
- Files list sprawls across unrelated areas of the codebase.
- Acceptance can't be checked in one pass; it's really a checklist of sub-acceptances.

**Too small — merge or just do it directly:**
- The change is a one-line fix, a typo, a config value — the pipeline's plan→critique→build→test→review overhead costs more than the change itself. Just make the edit.
- The change only makes sense bundled with an adjacent ticket (e.g. don't ticket "update the alt text" separately from "replace the hero image" if they're the same edit).

## Batch / round planning

When writing several tickets at once (a "content round" or "sprint"):

1. Write each ticket independently using the template above.
2. Cross-check every ticket's **Files** field against every other ticket's. Any file named in two or more tickets is an overlap.
3. For each overlap, decide and record the build order: which ticket must merge first, and which must rebase onto it. Tickets with no overlap are safe to plan and build in parallel.
4. Record the whole round — tickets plus overlap/order notes — in one file (e.g. `CONTENT_SPEC.md` or `/plans/tickets.md`, whatever the target repo's `dev-pipeline` setup expects for Phase 0).

This overlap map is exactly what `dev-pipeline` Phase 0 needs — write it once here so Phase 0 in the pipeline is a read, not a re-derivation.

## Worked example

```
### `pricing-page-annual-toggle` — let visitors compare monthly vs. annual pricing
**Goal:** visitors can see annual pricing and savings without leaving the pricing page.
**Current state:** the pricing page (`src/pages/Pricing.jsx`) only shows monthly
  prices. Support gets recurring questions about annual cost/savings that the
  page doesn't answer, and there's no toggle or secondary price anywhere on
  the page today.
**Change:** add a monthly/annual toggle above the pricing cards. Toggling
  updates each card's displayed price and shows a "save X%" badge on annual.
  Default state on page load is monthly.
**Non-goals:** do not change the pricing tiers, features list, or CTA copy.
  Do not add annual pricing to the booking flow or emails — display only.
**Files:** `src/pages/Pricing.jsx`, `src/components/PricingCard.jsx`
  (new: `src/components/PricingToggle.jsx`)
**Acceptance:** on `/pricing`, a toggle is visible above the cards; switching
  it to "annual" updates all card prices and shows a savings badge; switching
  back to "monthly" restores the original prices. No other page content changes.
```

## Handoff

Once a ticket (or a round of tickets with overlap notes) is written to the template above, it is ready to feed into `dev-pipeline` Phase 0/1 as-is. No further translation step — the planner subagent reads the ticket's Goal/Current state/Change/Non-goals/Files/Acceptance fields directly as its brief.
