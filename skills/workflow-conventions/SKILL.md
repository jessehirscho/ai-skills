---
name: workflow-conventions
description: Use at the start of any coding session in a git repo, and whenever committing, opening a PR, or wrapping up work — even if not explicitly asked. Enforces never committing/pushing/merging to main, feature-branch-only workflow, a required PR test-plan checklist format, and a required session-handoff doc before ending a substantial work session. Applies across all projects, not just one repo.
---

# Workflow Conventions

Personal defaults for how to work in any git repo, independent of what that repo's own CLAUDE.md says (if the repo's CLAUDE.md is stricter or more specific, follow it — these are the floor, not a ceiling).

## Git workflow

- Never commit or push directly to `main`.
- Never merge a PR/MR under any circumstances — not even if asked. Only the repo owner merges to `main`.
- All work happens on a feature branch.
- Always push the branch and present the PR link — don't just say the work is "done."
- Before any command that could discard uncommitted work (`git checkout`/`restore`/`reset`/`clean`, `rm -rf` in a repo), run `git status` first and stash or commit anything found.
- When staging broadly (`git add -A` or similar), review what actually got staged before committing — check for secrets or files that don't belong.

## Pull request format

Every PR description must include a **Test plan** section: a specific checklist the human can follow to manually verify the change. Tailor it to exactly what changed — name the actual pages, viewports, and interactions to check, not generic boilerplate like "test the feature works."

```
## Summary
- Bullet points describing what changed and why

## Test plan
- [ ] Specific thing to check on desktop
- [ ] Specific thing to check on mobile (375px) if UI changed
- [ ] Any other interaction, page, or state to verify
```

## Session handoff docs

Before ending a working session — the user signals they're wrapping up, or a substantial chunk of work (a fix, a feature, a multi-PR effort) is complete — write a handoff doc.

- Location: follow the target project's own convention if it has one (check its CLAUDE.md); otherwise default to `docs/handoff-<topic>-<YYYY-MM-DD>.md`.
- Always on its own branch with its own PR — never bundled into a feature/content PR, never pushed straight to `main`.
- If a handoff doc already exists for the same topic/effort from earlier the same day, extend it rather than creating a duplicate.
- Write it so a **different AI tool or a different person** can pick the work up cold — no assumed shared context, no assistant-specific branding or phrasing.

Required sections, regardless of project:

- **What was done this session** — concrete and verifiable: what changed and why, what was tested and how (exact commands where applicable), what was explicitly decided against or ruled out and why.
- **What to re-check, not take on faith** — claims from this session that were only checked once, informally, or partially, and should be independently re-verified rather than assumed correct.
- **What could be done next in the immediate next session** — concrete, actionable next steps.
- **What could be done in future sessions** — longer-horizon or lower-priority follow-ups, known gaps, deferred items, things explicitly out of scope this time.

## Why this exists

These rules exist because unauthorized merges and untested claims are expensive to undo, and undocumented context loss between sessions wastes real time re-deriving what already happened. Treat them as defaults to apply proactively, not just when explicitly asked.
