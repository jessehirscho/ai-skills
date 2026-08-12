# claude-skills

Personal Claude Code skills, built from patterns actually used across my own projects rather than generic templates.

## Skills

- **[dev-pipeline](skills/dev-pipeline/SKILL.md)** — a strict Plan → Critique → Iterate → Build → Test → Review → Merge subagent pipeline for shipping several independent tickets in one repo, with token-efficient parallel throughput. Includes ready-to-copy subagent definitions (`planner`, `critic`, `builder`, `tester`, `reviewer`) in `skills/dev-pipeline/agents/`.
- **[workflow-conventions](skills/workflow-conventions/SKILL.md)** — baseline git/PR/handoff-doc discipline to apply in any repo: no direct commits or merges to `main`, feature-branch-only, a required PR test-plan checklist format, and a required session-handoff doc before wrapping up substantial work.
- **[workflow-audit](skills/workflow-audit/SKILL.md)** — mines local session transcripts for real usage patterns (tool/skill/subagent frequency, repeated friction) and turns them into concrete fixes: new skills, CLAUDE.md changes, permission allowlists.
- **[token-minimiser](skills/token-minimiser/SKILL.md)** — a checklist for reducing tokens burned per unit of useful work: targeted reads/edits, scoped subagent delegation, avoiding redundant re-derivation within a session.
- **[plugin-creator](skills/plugin-creator/SKILL.md)** — packaging skills/agents/commands/MCP servers into a distributable Claude Code plugin with a marketplace manifest, vs. the plain-folder approach this repo itself uses.
- **[mcp-tools-setup](skills/mcp-tools-setup/SKILL.md)** — discovering, installing, scoping, and troubleshooting existing MCP servers (as opposed to `mcp-builder`, which builds a new one).
- **[claude-code-setup](skills/claude-code-setup/SKILL.md)** — auditing/improving your own Claude Code installation: settings.json, CLAUDE.md quality, hooks, permissions, plugin/skill hygiene.
- **[copilot-cli-setup](skills/copilot-cli-setup/SKILL.md)** — auditing/improving a GitHub Copilot CLI setup: custom instructions files, config, and known unverified areas flagged explicitly.
- **[codex-cli-setup](skills/codex-cli-setup/SKILL.md)** — auditing/improving an OpenAI Codex CLI setup: `config.toml`, `AGENTS.md`, approval/sandbox modes, profiles.
- **[opencode-setup](skills/opencode-setup/SKILL.md)** — auditing/improving an opencode setup: config precedence, multi-provider/model config, MCP, keybinds.
- **[pi-harness-setup](skills/pi-harness-setup/SKILL.md)** — auditing/improving a Pi coding-agent setup (`pi.dev` / `badlogic/pi-mono`): settings, `AGENTS.md`/`CLAUDE.md` concatenation, project trust model, extensions.

## Usage

Copy a skill's folder into `~/.claude/skills/<name>/` (personal, all projects) or `<repo>/.claude/skills/<name>/` (project-scoped). Claude Code discovers `SKILL.md` files there automatically.

For `dev-pipeline`, also copy `skills/dev-pipeline/agents/*.md` into the target repo's `.claude/agents/` — those are how the pipeline's five subagent roles get invoked via the `Agent` tool.
