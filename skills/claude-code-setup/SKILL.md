---
name: claude-code-setup
description: Use whenever the user wants to audit or improve their own local Claude Code installation/configuration — settings.json (user and project), permissions allow/deny lists, hooks, CLAUDE.md quality, installed plugins/skills hygiene, model/statusline/keybinding config. Distinct from workflow-audit (analyzes usage patterns from session logs) and mcp-tools-setup/plugin-creator (deep dives on those specific subsystems) — this is the config-surface checklist that ties them together.
---

# Claude Code Setup Audit

Audits the Claude Code harness configuration itself — not the codebase it's working in,
and not usage patterns over time (that's `workflow-audit`). This is a point-in-time check
of the config files that shape every session: what's allowed, what's loaded into context,
what's installed but unused.

## Where config lives (real hierarchy, highest precedence first)

1. **Managed** — `/etc/claude-code/managed-settings.json` (or platform equivalent) — org-deployed, cannot be overridden except specific exceptions
2. **Command-line args** — temporary, session-only overrides
3. **Local project** — `<repo>/.claude/settings.local.json` — personal, gitignored, not shared with the team
4. **Project** — `<repo>/.claude/settings.json` — committed, shared with all collaborators
5. **User** — `~/.claude/settings.json` — applies to you across all projects, lowest precedence

**Exception:** `permissions` rules *merge* across all scopes rather than override — a
restrictive rule anywhere in the chain wins.

Other config surfaces, not part of the settings.json precedence chain but part of the same
audit:

| Surface | Location |
|---|---|
| User CLAUDE.md | `~/.claude/CLAUDE.md` — loaded every session, every project |
| Project CLAUDE.md | `<repo>/CLAUDE.md` or `<repo>/.claude/CLAUDE.md` — committed |
| Local project memory | `<repo>/CLAUDE.local.md` — personal, gitignored |
| Installed plugins | `~/.claude/plugins/` (also referenced under the `plugins` key in settings.json) |
| Personal skills | `~/.claude/skills/*/SKILL.md` |
| Project skills | `<repo>/.claude/skills/*/SKILL.md` |
| Custom slash commands | `<repo>/.claude/commands/` |
| MCP servers (user) | `~/.claude.json` |
| MCP servers (project) | `<repo>/.mcp.json` |
| Auto-memory | `~/.claude/projects/<project>/memory/` (path configurable via `autoMemoryDirectory`) |

## Step-by-step audit procedure

1. **Read current settings.json** — `~/.claude/settings.json` and, if inside a repo,
   `<repo>/.claude/settings.json` and `<repo>/.claude/settings.local.json`. Note which
   scope each `permissions`/`hooks`/`env` entry actually lives in — a rule in the wrong
   scope (e.g. a personal allowlist entry accidentally committed to the shared project
   file) is itself a finding.
2. **Read CLAUDE.md files** for staleness and bloat — see checklist below.
3. **List installed plugins and skills** and check for ones that look unused or
   duplicative:
   ```bash
   ls ~/.claude/plugins/
   ls ~/.claude/skills/*/SKILL.md
   ls <repo>/.claude/skills/*/SKILL.md 2>/dev/null
   ```
4. **Check permission allow/deny lists against actual usage.** If `workflow-audit` has
   been run recently (or run it now), cross-reference its "same Bash command approved
   repeatedly" findings against what's already allowlisted — gaps are candidates to add;
   allowlist entries that never get exercised are candidates to remove.
5. **Produce a short prioritized fix list** — same format as `workflow-audit`: one row
   per finding, concrete file + change, ordered by impact. Confirm before touching
   `settings.json` permissions or deleting anything (destructive/security-relevant).

## Permissions audit

Read the `permissions.allow`, `permissions.deny`, and `permissions.ask` arrays in each
settings.json in scope. Rule syntax is `ToolName(pattern)`, e.g. `Bash(npm run *)`,
`Read(./src/**)`, `Write(.env*)`.

What to look for:
- **Overly broad allow entries** — `Bash(*)` or similarly unscoped wildcards defeat the
  purpose of the allowlist; tighten to the actual command family.
- **Overly narrow / stale entries** — an allow rule for a command or path that no longer
  exists in the project (renamed script, removed directory).
- **Missing deny entries** for obviously sensitive paths — `.env`, `.env.*`, credentials
  files, secrets directories — if the project handles any of these and they're not denied.
- **The recurring-approval pattern**: the single highest-value finding is "same Bash
  command approved repeatedly across sessions" — that's a workflow signal, not a
  settings.json signal, so pull it from `workflow-audit`'s tool-call-frequency grep (or
  the `fewer-permission-prompts` skill, which automates turning that into an allowlist
  diff) rather than re-deriving it here.

## CLAUDE.md quality check

CLAUDE.md is loaded into every session's context, so bloat has a recurring token cost —
audit for signal density, not just correctness.

- **Concise and load-bearing?** Every line should be something a session actually needs
  on every task (git workflow, PR format, key architecture facts) — not project trivia
  that only matters occasionally. Detail that's only needed sometimes belongs in a linked
  doc under `docs/`, referenced by path, not inlined.
- **Stale?** Check referenced commands, scripts, and file paths still exist — a
  `npm run audit:marketing` or similar line pointing at a script that was deleted or
  renamed is a direct find-and-fix.
- **Duplicates what's derivable from reading the code?** If a CLAUDE.md paragraph just
  restates a directory listing or an obvious `package.json` script table with no added
  interpretation, it's not earning its permanent context cost — trim it or replace with a
  pointer.
- Check both the user-level (`~/.claude/CLAUDE.md`) and project-level file — they compound
  every session, so redundancy between the two (the same instruction stated in both) is
  also worth flagging.

## Hooks

Hooks live under the `hooks` key in settings.json, keyed by event name (`SessionStart`,
`SessionEnd`, `PreToolUse`, `PostToolUse`, `ConfigChange`, etc.), each with a `bash`,
`http`, or `script` action.

Sanity checks, not a deep dive:
- Does a `PreToolUse` hook ever block a legitimate, expected action (check its exit code
  logic / matcher pattern) — a hook that's too aggressive silently blocks work with no
  clear error surfaced to the user.
- Does a hook swallow errors — e.g. a `bash` hook command piped to `/dev/null` or missing
  `set -e`, so a real failure in the hook doesn't surface.
- Is an `http` hook pointed at a URL still under `allowedHttpHookUrls` (if that key is
  set) — a hook silently no-oping because its target got denied is easy to miss.
- If the environment has a built-in skill for editing hook config (e.g. an
  `update-config`-style skill), prefer it over hand-editing settings.json for hook
  changes — check what's available rather than assuming a specific one exists.

## Plugin/skill hygiene

- **Stale/unused plugins** — installed in `~/.claude/plugins/` but never referenced;
  confirm with the user before removing (plugins can be silently load-bearing for MCP
  tools that don't obviously map to the plugin name).
- **Skills with vague descriptions that never auto-invoke** — a skill whose
  `description` frontmatter doesn't clearly state trigger conditions won't get selected
  even when relevant. Candidate for a rewrite; see the `skill-builder`-style skill if the
  environment has one, otherwise apply the same principle directly: description should
  name concrete trigger phrases and scope, not just a vague topic.
- **Duplicate skills covering the same ground** — two skills answering overlapping
  questions cause ambiguous selection. Merge or narrow scope so each has a distinct,
  non-overlapping trigger.
- Cross-reference against actual invocation frequency if `workflow-audit` data is
  available — it directly reports which installed skills get zero hits.
- For anything involving building/scaffolding a new plugin or MCP tool integration, defer
  to `plugin-creator` and `mcp-tools-setup` respectively — this skill only flags hygiene
  issues in what's already installed, it doesn't cover authoring new ones.

## Model / statusline / keybindings

Low-risk, personal-preference config — worth a one-line mention in the audit output if
something looks obviously broken (e.g. `model` pinned to a deprecated ID), but not worth
deep scrutiny:
- Model: `model` / `fallbackModel` / `effortLevel` keys in settings.json, or `/model`
  mid-session.
- Statusline: configured via `statusLine` key in settings.json (script path) or the
  `statusline-setup` agent.
- Keybindings: `~/.claude/keybindings.json` — see the `keybindings-help` skill for the
  mechanics of editing it.

## Quick-reference table

| Config surface | File path | What to check |
|---|---|---|
| User settings | `~/.claude/settings.json` | permissions, hooks, model, env, plugins key |
| Project settings (shared) | `<repo>/.claude/settings.json` | committed permissions/hooks appropriate for the whole team |
| Project settings (personal) | `<repo>/.claude/settings.local.json` | gitignored; personal allowlist entries live here, not in the shared file |
| Managed settings | `/etc/claude-code/managed-settings.json` (or org platform equivalent) | org-wide overrides; usually not user-editable |
| User memory | `~/.claude/CLAUDE.md` | staleness, bloat, cross-project applicability |
| Project memory | `<repo>/CLAUDE.md` / `<repo>/.claude/CLAUDE.md` | staleness, bloat, duplication with code |
| Local project memory | `<repo>/CLAUDE.local.md` | personal notes not meant for the team |
| Plugins | `~/.claude/plugins/` | unused/stale installs |
| Personal skills | `~/.claude/skills/*/SKILL.md` | vague descriptions, zero-invocation skills, duplicates |
| Project skills | `<repo>/.claude/skills/*/SKILL.md` | same, scoped to this repo |
| MCP servers | `~/.claude.json` (user), `<repo>/.mcp.json` (project) | servers referenced but not configured, or vice versa |
| Auto-memory | `~/.claude/projects/<project>/memory/` | see `workflow-audit` for mining this for patterns |
| Keybindings | `~/.claude/keybindings.json` | low-risk; see `keybindings-help` |

## Related skills

- **`workflow-audit`** — mines session transcripts for usage *patterns* (what commands
  get approved repeatedly, which skills never fire, thrash-y sessions). Run it first if
  the permissions or skill-hygiene sections above need real frequency data rather than a
  static read of the config.
- **`fewer-permission-prompts`** — automates turning repeated approvals into a
  `settings.json` allowlist diff.
- **`update-config`** — the mechanics of editing settings.json for hooks/permissions/env,
  if hand-editing isn't preferred.
- **`mcp-tools-setup`** — deep dive on configuring MCP servers specifically.
- **`plugin-creator`** — authoring a new plugin, as opposed to auditing installed ones.
- **`keybindings-help`** — mechanics of editing `~/.claude/keybindings.json`.
