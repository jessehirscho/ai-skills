# claude-skills

Personal Claude Code skills, built from patterns actually used across my own projects rather than generic templates.

## Skills

- **[dev-pipeline](skills/dev-pipeline/SKILL.md)** — a strict Plan → Critique → Iterate → Build → Test → Review → Merge subagent pipeline for shipping several independent tickets in one repo, with token-efficient parallel throughput. Includes ready-to-copy subagent definitions (`planner`, `critic`, `builder`, `tester`, `reviewer`) in `skills/dev-pipeline/agents/`.
- **[workflow-conventions](skills/workflow-conventions/SKILL.md)** — baseline git/PR/handoff-doc discipline to apply in any repo: no direct commits or merges to `main`, feature-branch-only, a required PR test-plan checklist format, and a required session-handoff doc before wrapping up substantial work.
- **[pr-workflow](skills/pr-workflow/SKILL.md)** — `gh` CLI mechanics for the PR lifecycle: creating PRs, reading/responding to review comments, checking CI status, handling merge conflicts. Companion to `workflow-conventions` (which covers policy/format, not commands).
- **[ci-pipeline](skills/ci-pipeline/SKILL.md)** — setting up, modifying, and debugging CI pipelines (primarily GitHub Actions): workflow scaffolding, caching, matrix builds, secrets, deploy gating, and triaging failed runs.
- **[mcp-builder](skills/mcp-builder/SKILL.md)** — scaffolding a new MCP server (TypeScript or Python), defining tools with proper schemas, choosing a transport, and registering it with Claude Code.
- **[agent-builder](skills/agent-builder/SKILL.md)** — authoring new `.claude/agents/*.md` subagent definitions: frontmatter format, least-privilege tool scoping, model selection, and writing a scoped system prompt.
- **[subagents](skills/subagents/SKILL.md)** — judgment layer for using the `Agent` tool day-to-day: when delegation is actually worth it, how to write a good dispatch prompt, foreground vs. background, parallel dispatch, and avoiding duplicated work.
- **[skill-builder](skills/skill-builder/SKILL.md)** — the meta-skill for creating new Claude Code skills: description-first authoring, file layout conventions, content quality bar, and a pre-publish checklist.
- **[performance-audit](skills/performance-audit/SKILL.md)** — methodology and concrete tooling for comparing a current implementation against a proposed one: baseline measurement, controlled comparison, and a results-reporting template.
- **[html-comparison-docs](skills/html-comparison-docs/SKILL.md)** — building self-contained HTML documents that visually compare results (before/after, option A vs. B): side-by-side, slider, and tabbed layout patterns.
- **[data-insights](skills/data-insights/SKILL.md)** — turning messy real-world data into a defensible business insight: profiling, cleaning, question-first analysis, sanity-checking findings, and presenting them in business terms.
- **[pdf](skills/pdf/SKILL.md)** — reading, extracting, merging, splitting, creating, watermarking, encrypting, and OCR'ing PDF files.

## Usage

Copy a skill's folder into `~/.claude/skills/<name>/` (personal, all projects) or `<repo>/.claude/skills/<name>/` (project-scoped). Claude Code discovers `SKILL.md` files there automatically.

For `dev-pipeline`, also copy `skills/dev-pipeline/agents/*.md` into the target repo's `.claude/agents/` — those are how the pipeline's five subagent roles get invoked via the `Agent` tool.
