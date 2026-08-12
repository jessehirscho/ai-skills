---
name: dev-pipeline
description: Use when shipping multiple independent features/tickets in a repo and you want token-efficient, parallel subagent throughput instead of one long single-thread session. Strict pipeline — Plan → Critique → Iterate → Build → Test → Review → Merge — with a fixed subagent per phase. Best for repos with several well-scoped, mostly-independent tickets (content/config changes, small features, bug fixes). Not for single tiny changes or deeply interdependent rewrites.
---

# Dev Pipeline: Plan → Critique → Iterate → Build → Test → Review → Merge

A strict, repeatable pipeline for shipping several tickets via subagents, built for token efficiency and parallel throughput. Adapted from a pipeline proven in production use (`ai-market-digest`'s `WORKFLOW.md`).

## When to use this

- You have 2+ independent, well-scoped tickets to ship in one repo.
- You want fresh-eyes critique and review, not the same instance grading its own work.
- You're willing to spend more tokens for higher throughput and less babysitting (subagent-heavy workflows run roughly 5-7x the token cost of one single-thread session — this trades tokens for parallelism and quality gates, not for cheapness).

Don't use it for a single one-line fix, or for a change so tangled across files that it can't be scoped into an independent ticket.

## One-time setup

1. Copy `agents/*.md` from this skill into `.claude/agents/` at the target repo's root. Claude Code auto-discovers them there and invokes them through the `Agent` tool.
2. Adjust each agent's test commands (Phase 5) to match the repo's actual toolchain — the shipped agents assume `pytest`/`python -m py_compile` as placeholders; swap for the repo's real lint/build/test commands.
3. Add this pointer block to the repo's `CLAUDE.md`:
   ```
   ## Development workflow
   Follow the dev-pipeline skill for any multi-ticket change: plan → critique →
   iterate → branch → build → test → review → merge. Subagent definitions live
   in .claude/agents/. Do not skip the critique phase, and do not merge a
   ticket whose plan wasn't APPROVED.
   ```

## Phase 0 — Scope

Read only the repo's spec/README, not the whole tree. Confirm the ticket list. Flag any file-overlap between tickets (two tickets touching the same file must build sequentially, not in parallel — rebase the second onto the first's merged result). Write the ticket list and overlap flags to `/plans/tickets.md`.

## Phase 1 — Plan (parallel, `planner` subagent)

Launch one `planner` subagent per ticket, in parallel. Pass each one only its ticket name, scope, and the exact relevant file paths — a subagent's context starts empty, so anything it needs must be in the prompt string, never assumed. Each planner writes exactly one file: `/plans/<ticket>.md`. No source code is touched in this phase.

## Phase 2 — Critique (parallel, `critic` subagent, fresh instance)

For each plan, launch a `critic` subagent — never the same subagent instance that wrote it. Fixed rubric (see `agents/critic.md`). Output: `/plans/<ticket>.critique.md` with **APPROVE** or **REVISE** plus specific, actionable fixes.

## Phase 3 — Iterate

On REVISE, re-launch a fresh `planner` instance with the original plan and the critique in its prompt. **Cap at 2 revision rounds per ticket.** If still not approved after 2 rounds, stop and escalate to the user instead of looping.

## Phase 4 — Build (branch per ticket, `builder` subagent)

Once APPROVED:
```bash
git checkout main && git pull
git checkout -b feature/<ticket>
```
Launch one `builder` subagent, given only the approved plan and the files it names. It implements exactly what the plan says — no scope creep. Small, scoped commits, each referencing the ticket.

## Phase 5 — Test (per branch, `tester` subagent)

Every branch must pass before Phase 6: the repo's build/compile check, its test suite, plus the ticket-specific check the plan defined in its "Test plan" section. FAIL sends the branch back to Phase 4 (build), not back to planning — unless the failure reveals the plan itself was flawed, in which case it goes back to Phase 1.

## Phase 6 — Review (`reviewer` subagent, diff-only context)

Reviewer gets `git diff main...feature/<ticket>` plus the approved plan — never the builder's reasoning trail or commit-by-commit history. **APPROVE** or **REQUEST CHANGES** (back to Phase 4).

## Phase 7 — Merge

- No-overlap tickets merge in any order.
- Overlap-flagged tickets merge in build order (per Phase 0), rebasing before each.
- After every merge to `main`: pull, then run the full test suite (regression check, not just the merged branch).
- Squash-merge, delete the branch.
- After all tickets land, do one final end-to-end smoke check before considering the round done.

## Token-efficiency rules

- Never let a subagent read the whole repo — pass exact file paths in its prompt, always.
- Cap critique/revision loops at 2 rounds.
- Use a general-purpose `Explore` subagent for open-ended "where is X handled" questions instead of the main session grepping around manually.
- Batch independent reads before writes; don't re-open a file already in context this turn.
- Keep the repo's `CLAUDE.md` short — point to a workflow doc (or this skill) for pipeline detail rather than duplicating it inline.
- Prefer targeted edits over full-file rewrites once a file exists.

## Subagent definitions

Ship these five files into `.claude/agents/` (see `agents/` in this skill directory for the source):

- **planner** (`Read, Grep, Glob, Write`, model: sonnet) — writes one scoped plan per ticket; read-only against source.
- **critic** (`Read, Write`, model: sonnet) — reviews a plan against a fixed rubric and writes the critique file; never the instance that wrote the plan.
- **builder** (`Read, Write, Edit, Bash`, model: sonnet) — implements exactly one approved plan on its own branch.
- **tester** (`Bash, Read`, model: haiku) — runs the test suite plus the ticket-specific check; reports PASS/FAIL, never fixes code.
- **reviewer** (`Bash, Read`, model: sonnet) — reviews the branch diff against the plan; diff-scoped context only.

Each agent's rubric/checklist should be adapted per-repo (see `agents/critic.md`'s "staleness" check as an example of a project-specific failure mode worth encoding once you've been burned by it).
