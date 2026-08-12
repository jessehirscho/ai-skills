# claude-skills

Personal Claude Code skills, built from patterns actually used across my own projects rather than generic templates.

## Skills

- **[dev-pipeline](skills/dev-pipeline/SKILL.md)** — a strict Plan → Critique → Iterate → Build → Test → Review → Merge subagent pipeline for shipping several independent tickets in one repo, with token-efficient parallel throughput. Includes ready-to-copy subagent definitions (`planner`, `critic`, `builder`, `tester`, `reviewer`) in `skills/dev-pipeline/agents/`.
- **[workflow-conventions](skills/workflow-conventions/SKILL.md)** — baseline git/PR/handoff-doc discipline to apply in any repo: no direct commits or merges to `main`, feature-branch-only, a required PR test-plan checklist format, and a required session-handoff doc before wrapping up substantial work.

## Usage

Copy a skill's folder into `~/.claude/skills/<name>/` (personal, all projects) or `<repo>/.claude/skills/<name>/` (project-scoped). Claude Code discovers `SKILL.md` files there automatically.

For `dev-pipeline`, also copy `skills/dev-pipeline/agents/*.md` into the target repo's `.claude/agents/` — those are how the pipeline's five subagent roles get invoked via the `Agent` tool.
