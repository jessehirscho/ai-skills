---
name: subagents
description: Use whenever deciding whether and how to delegate work to subagents via the Agent tool during a session — judging if delegation is worth it, writing a good dispatch prompt, foreground vs background, parallel dispatch, and avoiding common subagent failure modes. Distinct from agent-builder (authoring new subagent definitions) and dev-pipeline (a specific plan/build/test/review pipeline).
---

# Subagents: when and how to delegate

The general judgment layer for using the Agent tool mid-session. Every dispatch
costs a cold start — a fresh agent re-derives context you already have. Only
pay that cost when it buys you something: parallelism, context isolation, or
an independent perspective.

## When TO delegate

- **Genuinely independent, well-scoped work.** You can state the goal, the
  relevant files/context, and what "done" looks like in a few sentences.
- **Tasks that would pollute main context.** Large greps, reading many files,
  broad codebase exploration — you want the *answer*, not the raw search
  output, sitting in your context.
- **Multiple independent tasks.** Fan them out in parallel rather than doing
  them one at a time yourself.
- **Fresh/unbiased perspective.** Reviewing your own work, a second opinion on
  a risky change, a code review — a subagent that hasn't seen your reasoning
  won't rubber-stamp it.

## When NOT to delegate

- **Trivial single-step tasks.** Reading one known file, making one small
  edit, running one command — just do it. The cold-start tax exceeds the
  savings.
- **Unclear scope.** If you can't yet state what "done" looks like, that's a
  planning problem, not a delegation problem. Plan first, dispatch second.
- **Anything needing back-and-forth with the user.** Subagents can't ask the
  user questions mid-task. If the task will likely need a clarification,
  handle it directly or get the answer before dispatching.
- **Work you're about to duplicate.** If you're delegating research to an
  agent, don't also go do that same research yourself while it runs. Decide
  who owns what before dispatching, not after.

## Writing a good dispatch prompt

A subagent has none of this conversation's context. Brief it like a capable
colleague who just walked in:

- **Goal + why.** Not just "fix X" — what's this in service of, so the agent
  can make reasonable judgment calls when it hits something you didn't
  anticipate.
- **What's already been tried/ruled out.** Save it from repeating your dead
  ends.
- **Enough surrounding context to exercise judgment.** File paths, relevant
  function names, prior decisions — the specifics that prove you understood
  the problem before handing it off.
- **Explicit constraints.** Which files it may/may not touch, whether it
  should write code vs. just research, response length if you need it kept
  short (raw findings can be expensive to relay back).
- **Lookup vs. investigation — hand over different things:**
  - *Lookup* (you know the answer exists in a specific place): give the exact
    command or path. "Run `git log --oneline -20` and summarize."
  - *Investigation* (open-ended, premise might be wrong): give the
    **question**, not a prescribed sequence of steps. Prescribed steps become
    dead weight — or actively misleading — the moment the premise doesn't
    hold. Let the agent figure out its own path.

Terse command-style prompts ("fix the bug in auth.js") produce shallow,
generic work. Never write "based on your findings, implement the fix" — that
pushes synthesis onto the agent. Do the synthesis yourself; hand the agent a
concrete, understood task.

## Foreground vs background

- **Foreground** when you need the result before you can take your next step
  — research that determines what you do next, a lookup blocking your current
  task. Pass `run_in_background: false`.
- **Background** (default) when you can keep working and get notified later.
  Prefer this whenever the task doesn't block your very next action.
- **Never fabricate or predict a background agent's results.** Not as prose,
  not as a guess, not to "keep things moving." The result arrives later as a
  notification. If the user asks before it lands, say it's still running —
  give status, not a guess.

## Parallel dispatch

- When tasks are **truly independent**, launch all of them as multiple Agent
  tool calls in a **single message/turn** — not one at a time across separate
  turns.
- Anti-pattern: calling two tasks "parallel" when task B actually consumes
  task A's output. That's a sequential dependency wearing a parallel costume
  — run A, wait for it, then dispatch B with A's result in the prompt.
- Before fanning out, briefly sanity-check that the tasks don't share mutable
  state (same file, same branch) — parallel agents editing the same file will
  clobber each other.

## Avoiding duplicated work

Decide ownership before dispatching. If a subagent is doing the research,
don't simultaneously grep the same codebase yourself "just in case" — you'll
pay for the work twice and may end up reconciling two different answers.
If you do want a second opinion, make that explicit and intentional (see
"fresh perspective" above), not an accident of impatience.

## Trust but verify

A subagent's final report describes what it **intended** to do, not
necessarily what it **did**. Before relaying "done" to the user:

- For code changes: read the actual diff or the changed file yourself. Don't
  just relay the agent's self-report as fact.
- For research: spot-check a claim or two if it's load-bearing for a decision
  you're about to make.

This isn't distrust of the agent's honesty — it's that summaries compress and
can drift from what actually landed on disk.

## Quick-reference

| Scenario | Delegate? | Foreground / Background |
|---|---|---|
| Read one known file, small edit | No — just do it | n/a |
| Broad codebase search, unclear where target lives | Yes | Foreground if it blocks your next step |
| Research that informs your immediate next decision | Yes | Foreground |
| Independent cleanup/refactor task while you keep working | Yes | Background |
| 3 unrelated files each need a similar fix | Yes, one agent per file, single turn | Background (or foreground if you're blocked on all three) |
| Second opinion / review of your own diff | Yes | Background unless you're about to ship |
| Task depends on output of another in-flight agent | Not in parallel — sequence it | Foreground for the first, then dispatch the second |
| Scope is fuzzy, "done" undefined | No — plan first | n/a |
| Task needs mid-task clarification from the user | No — handle directly | n/a
