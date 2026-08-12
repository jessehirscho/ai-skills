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
- **[workflow-audit](skills/workflow-audit/SKILL.md)** — mines local session transcripts for real usage patterns (tool/skill/subagent frequency, repeated friction) and turns them into concrete fixes: new skills, CLAUDE.md changes, permission allowlists.
- **[token-minimiser](skills/token-minimiser/SKILL.md)** — a checklist for reducing tokens burned per unit of useful work: targeted reads/edits, scoped subagent delegation, avoiding redundant re-derivation within a session.
- **[plugin-creator](skills/plugin-creator/SKILL.md)** — packaging skills/agents/commands/MCP servers into a distributable Claude Code plugin with a marketplace manifest, vs. the plain-folder approach this repo itself uses.
- **[mcp-tools-setup](skills/mcp-tools-setup/SKILL.md)** — discovering, installing, scoping, and troubleshooting existing MCP servers (as opposed to `mcp-builder`, which builds a new one).
- **[claude-code-setup](skills/claude-code-setup/SKILL.md)** — auditing/improving your own Claude Code installation: settings.json, CLAUDE.md quality, hooks, permissions, plugin/skill hygiene.
- **[copilot-cli-setup](skills/copilot-cli-setup/SKILL.md)** — auditing/improving a GitHub Copilot CLI setup: custom instructions files, config, and known unverified areas flagged explicitly.
- **[codex-cli-setup](skills/codex-cli-setup/SKILL.md)** — auditing/improving an OpenAI Codex CLI setup: `config.toml`, `AGENTS.md`, approval/sandbox modes, profiles.
- **[opencode-setup](skills/opencode-setup/SKILL.md)** — auditing/improving an opencode setup: config precedence, multi-provider/model config, MCP, keybinds.
- **[pi-harness-setup](skills/pi-harness-setup/SKILL.md)** — auditing/improving a Pi coding-agent setup (`pi.dev` / `badlogic/pi-mono`): settings, `AGENTS.md`/`CLAUDE.md` concatenation, project trust model, extensions.
- **[agent-handover](skills/agent-handover/SKILL.md)** — routing phases of a task across different LLMs/model tiers (e.g. a strong model plans, a mid-tier model implements, a different model reviews), with a written handover artifact at each boundary instead of a shared conversation. Distinct from `subagents`/`dev-pipeline`, which cover same-model delegation.

## Usage

Copy a skill's folder into `~/.claude/skills/<name>/` (personal, all projects) or `<repo>/.claude/skills/<name>/` (project-scoped). Claude Code discovers `SKILL.md` files there automatically.

For `dev-pipeline`, also copy `skills/dev-pipeline/agents/*.md` into the target repo's `.claude/agents/` — those are how the pipeline's five subagent roles get invoked via the `Agent` tool.
