---
name: prompt-eval
description: Use whenever the user wants to test/evaluate a system prompt, subagent definition, or SKILL.md dispatch description against example inputs before shipping it — "does this actually work", "test this skill's dispatch", "will this agent stay in scope", or before merging any prompt-engineering artifact that will be reused or shared. Verification companion to agent-builder and skill-builder: those cover how to write the artifact, this covers how to check it works before trusting it in production.
---

# Prompt Eval

A lightweight discipline for checking that a system prompt, subagent
definition, or SKILL.md description actually does what it's supposed to,
*before* it ships. This is not a testing framework — it's "write 5-8
examples by hand and actually run them," which catches most real problems
for a few minutes of work.

## What's being evaluated, and why the check differs

| Artifact | What can go wrong | Check needed |
|---|---|---|
| SKILL.md `description` | Fires on the wrong requests, or silently never fires on the right ones | **Dispatch eval** — a classification problem: given a request, does Claude invoke this skill or not? |
| Subagent system prompt | Wanders outside scope, misuses a granted tool, returns output in a format nothing downstream can parse, guesses instead of stopping on ambiguity | **Behavior eval** — run it, watch what it actually does against a rubric |
| General system prompt (a full assistant persona/config) | Both of the above — it has to trigger on/recognize the right situations *and* behave correctly once engaged | **Both** — dispatch-style check for "does it correctly handle vs. correctly decline," plus a behavior check for what it does once engaged |

Dispatch failures and behavior failures look completely different in
practice. A skill with a perfect description but a broken procedure will
still "pass" if you only check whether it fires. A subagent with a great
prompt but a vague description will never get dispatched to in the first
place. Check the one that matches the artifact — don't substitute one for
the other.

## The eval-set approach

Before shipping, write 5-8 example inputs by hand:

- **3-4 clear positive cases** — requests that obviously should trigger the
  skill / requests the subagent should handle normally / situations the
  system prompt is meant to cover.
- **2-3 clear negative cases** — adjacent requests that should NOT trigger
  it, or that the subagent/prompt should refuse or stay out of scope on.
  Pick things that sound similar on the surface but are actually a
  different job (this is where most real dispatch bugs live).
- **1-2 deliberately ambiguous edge cases** — a request that's genuinely
  unclear, missing information, or straddles two responsibilities. The
  correct behavior for these is usually "ask" or "stop and report," not
  "guess" — use the case to check that.

This is cheap on purpose. A formal eval harness is overkill for most of
what gets shipped in a personal skills/agents repo; a handwritten list
that you actually run is what catches problems, not a more elaborate
list that never gets run.

## Running a SKILL.md dispatch eval

Guessing whether a description "sounds specific enough" is not a
dispatch eval — dispatch is decided by a fresh session of the model/harness
you're evaluating, matching the request against the description, so that's
what you have to actually observe.

Method:
1. For each example in your eval set, open a **fresh** session of
   whichever model/harness you're evaluating (or use your harness's
   subagent/delegation tool, if it has one, with a general-purpose agent
   that has no prior context — otherwise just open a new session) and
   give it *only* the example request — nothing that hints you're testing
   a skill, no mention of the skill's name.
2. Watch whether it invokes the skill (visible as a `Skill` tool call, or
   ask it afterward "did you use a skill for that, and which one").
3. Record trigger / no-trigger against what you expected.

Do this for every positive and negative case. If a positive case doesn't
trigger, the description is missing a term/phrasing the real request
used — add it. If a negative case *does* trigger, the description is too
broad or overlaps another skill's territory — narrow it or add an
explicit "not for X" line.

## Running a subagent behavior eval

Method:
1. Pick (or construct) a real or realistic sandboxed task from the
   subagent's actual target domain — not a toy example, something with
   the same shape as what it'll really be asked to do.
2. Run it via your harness's subagent/delegation tool (e.g. Claude Code's
   Agent tool) with that subagent type — or, if your harness has no such
   mechanism, a fresh session given the same system prompt.
3. Score the transcript against this rubric:
   - Did it stay within its **granted tools** (no attempt to do something
     that would've required a tool it wasn't given)?
   - Did it stay within its **declared scope** (didn't drift into
     adjacent work the prompt didn't ask for)?
   - Did it produce the **expected output format** (verdict vocabulary,
     file location, report structure — whatever the prompt promised)?
   - On the ambiguous/edge case: did it **stop and report/escalate**
     rather than guess and proceed?

Any "no" is a finding — fix the prompt (tighten scope language, add an
explicit stop condition, fix the tool list) and rerun that same case.

## Regression checking on edits

When you edit an existing prompt/skill/agent, don't just check that your
fix works — rerun the **entire existing eval set**, before and after the
edit, and diff the results. A fix aimed at one failing case commonly
breaks a previously-passing one (tightening a description to stop a false
positive is the classic way to accidentally kill a true positive). This
before/after rerun is the whole point of keeping the eval set written
down instead of testing ad hoc each time.

## Scoring and reporting

Keep this to pass/fail plus one line — anything more elaborate won't
survive contact with "I'll just ship it, I'm sure it's fine."

```
| ID | Input                                   | Expected            | Actual            | Pass? |
|----|------------------------------------------|----------------------|--------------------|-------|
| P1 | "make a skill for X"                     | dispatch             | dispatched         | ✅    |
| P2 | "turn this into a skill"                 | dispatch             | dispatched         | ✅    |
| N1 | "review this skill for quality"          | no dispatch (or diff)| dispatched (wrong) | ❌    |
| E1 | "help me with skills stuff"               | ambiguous — ok either way, but shouldn't silently guess | asked for clarification | ✅ |
```

One line of "why" per failure is enough: *"N1 fired because 'skill'
appears in both descriptions and the review-request wording overlaps
skill-builder's — needs an explicit not-for-review line."*

## When NOT to bother

Skip a formal eval pass for a one-off, low-stakes, personal artifact you
alone will use once or twice — the overhead isn't worth it if nothing
regresses when it's wrong. Do a real eval pass whenever the artifact:

- will be **reused across sessions** (any skill or subagent that stays
  in the repo),
- will be **shared with others** (teammates, a public repo, anyone who
  didn't watch it get written),
- or touches something **consequential** — money, sending real
  communications, destructive/irreversible actions, or anything where a
  silent wrong-trigger or scope-creep is expensive to discover later.

## Quick reference

| Step | Do this |
|------|---------|
| Decide what kind of eval | Dispatch (SKILL.md), behavior (subagent), or both (general system prompt) |
| Build the eval set | 3-4 positive, 2-3 negative, 1-2 ambiguous — written by hand before shipping |
| Run a dispatch eval | Fresh session/Agent call per example, no hints, check trigger vs. no-trigger |
| Run a behavior eval | Real/realistic task, score against: tools / scope / output format / stop-on-ambiguity |
| After any edit | Rerun the full existing eval set, diff before vs. after |
| Report results | One row per example: input, expected, actual, pass/fail, one-line why on failures |
| Skip it | One-off, low-stakes, personal, single-use artifact only |
