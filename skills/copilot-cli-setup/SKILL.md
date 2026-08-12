---
name: copilot-cli-setup
description: Use whenever the user wants to audit or improve their GitHub Copilot CLI / Copilot coding-agent setup — instructions files, MCP servers, model/approval config, or custom agents.
---

# Copilot CLI Setup

Practical checklist for auditing or improving a GitHub Copilot CLI (`copilot`) setup — the
standalone terminal agent, not the VS Code extension (though instructions files are shared).

Researched against GitHub's official docs as of August 2026. Where noted as unverified, check
the live docs before treating it as fact — this surface (especially nested AGENTS.md discovery)
is actively changing.

## Where instructions live

- **`.github/copilot-instructions.md`** — repo-wide instructions, auto-included in every prompt
  in that repo. This is the closest equivalent to `CLAUDE.md`.
- **`AGENTS.md`** (repo root, or nested in subdirectories) — also supported; Copilot CLI reads
  the nearest `AGENTS.md` up the tree from the current working directory to the git root, and
  nested files take precedence for the parts of the project they sit in. **Unverified**: whether
  *all* intermediate AGENTS.md files get merged (with the closest one winning) or only the
  nearest one is used — GitHub's own issue tracker (github/copilot-cli#1655, #3051) shows this is
  still being refined, so don't assume VS Code's `chat.useNestedAgentsMdFiles` behavior carries
  over exactly.
- **`*.instructions.md`** files with an `applyTo` frontmatter glob — path-specific instructions,
  conceptually like a scoped CLAUDE.md for a subfolder or file type.
- **`~/.copilot/...`** — user-level instructions can also apply globally; when both user-level and
  repo-level files exist, Copilot CLI combines them (de-duping identical content) rather than
  having one strictly override the other.
- You can pull in another file from any of these with `@relative/path.md`.

Keeping them effective (same principles as CLAUDE.md):
- Keep `.github/copilot-instructions.md` short and project-specific — build/test commands, repo
  conventions, things a generic model would get wrong. Don't restate what's discoverable from
  reading the code.
- Push path-specific rules into `*.instructions.md` with `applyTo` rather than bloating the root
  file.
- Re-check periodically for staleness the same way you'd check a CLAUDE.md — stale commands and
  dead file references are the most common rot.

## Config / settings surface

Base directory: **`~/.copilot/`** (override with `$COPILOT_HOME` or `$XDG_CONFIG_HOME`).

| File | Purpose |
|---|---|
| `~/.copilot/config.json` | Core CLI settings |
| `~/.copilot/settings.json` | Model selection persisted across sessions (also settable live with the `/model` slash command in an interactive session) |
| `~/.copilot/mcp-config.json` | User-level MCP server definitions (persists across all sessions/workspaces) |
| `~/.copilot/permissions-config.json` | Saved tool approvals (what you approved gets written here) |
| `~/.copilot/agents/` | Personal custom agent definitions (`.agent.md` files), available in all sessions |
| `.github/mcp.json` | Repo-local MCP server config, auto-discovered per workspace |
| `.github/agents/` | Repo-committed custom agents — shared with the whole team once pushed |

Model precedence (highest to lowest): a model pinned inside a custom agent definition → `--model`
CLI flag → `model` key in `settings.json` → CLI default.

Approvals: `--allow-tool`/`--deny-tool` flags set session policy; deny always beats allow, even
over `--allow-all` or a saved permissions-config.json entry. Admins can set
`disableBypassPermissionsMode` to stop users from turning on a full-bypass mode. **Unverified**:
exact syntax/flag names may shift between CLI versions — confirm against `copilot --help` and
the current CLI reference before relying on a specific flag in a script.

## MCP / tool-extension support

Copilot CLI does support MCP servers, configured via `~/.copilot/mcp-config.json` (user-level)
and `.github/mcp.json` (repo-level, auto-discovered). Tool access is gated per-server or
per-tool with `--allow-tool='ServerName'` / `--deny-tool='ServerName(tool_name)'`.

Compared to Claude Code's MCP support: conceptually similar (JSON server config, per-tool
allow/deny), but Claude Code's permission model is richer (project vs. user vs. enterprise
settings layers, hook-based automation). Copilot CLI's equivalent is the
config/permissions-config.json split above plus enterprise-managed settings for orgs. If the
user is porting an existing Claude Code MCP server list over, the server definitions themselves
(`command`, `args`, `env`) are typically portable as-is since both follow the same MCP spec —
verify each server still starts correctly under `copilot` rather than assuming.

## Agent / subagent equivalent

Copilot CLI has **custom agents**: Markdown files with an `.agent.md` extension (frontmatter +
instructions), stored either personally (`~/.copilot/agents/`, all sessions) or per-repo
(`.github/agents/`, version-controlled and shared with the team). These define a persona/toolset
for a single agent invocation.

What it does **not** have (as far as this research found): a first-class multi-agent
orchestration/delegation primitive equivalent to Claude Code's `Agent` tool spawning parallel
subagents with independent context. Custom agents in Copilot CLI are selectable personas you run
*as*, not workers you dispatch *to* from within a session. Workflows in this repo that rely on
fan-out delegation — e.g. the `dev-pipeline` skill's pattern of spawning several subagents in
parallel — likely don't map directly onto Copilot CLI and would need to be redesigned as
sequential single-agent steps, or run outside the CLI (e.g. via separate `copilot` invocations
scripted from a shell loop). Treat this as a structural gap, not a config item to fix.

## Audit procedure

1. **Locate instructions files.** `find . -iname "AGENTS.md" -o -iname "copilot-instructions.md" -o -iname "*.instructions.md"` from repo root, plus check `~/.copilot/` for user-level ones.
2. **Check staleness/quality** of `.github/copilot-instructions.md` / `AGENTS.md`: do referenced
   commands still exist (`npm run <x>`)? Do referenced files/paths still exist? Is it duplicating
   things obvious from the code (bloat) or missing the non-obvious project quirks (gap)?
3. **Check config surface**: `cat ~/.copilot/settings.json ~/.copilot/config.json` (or the repo's
   `$COPILOT_HOME` equivalent) — confirm model choice is intentional, not left on a stale default.
4. **Check MCP setup**: inspect `~/.copilot/mcp-config.json` and `.github/mcp.json` for servers
   that are configured but dead (wrong command path, missing env var) vs. servers the user
   expects but hasn't configured yet.
5. **Check custom agents**: list `~/.copilot/agents/*.agent.md` and `.github/agents/*.agent.md` —
   confirm each still matches current project structure/commands.
6. **Check approvals**: skim `~/.copilot/permissions-config.json` for stale or overly broad
   blanket approvals (e.g. an entire MCP server allowed when only one tool is actually needed).
7. **Produce a prioritized fix list** ordered: (a) broken/stale instructions that will cause wrong
   behavior, (b) missing instructions for known project gotchas, (c) config drift (wrong model,
   overbroad permissions), (d) MCP/agent hygiene, (e) nice-to-haves.

## Confidence notes

High confidence (multiple corroborating official-docs hits): `.github/copilot-instructions.md`
location, `AGENTS.md` support, `~/.copilot/` base directory and its config/mcp-config/permissions
files, `--allow-tool`/`--deny-tool` existing, custom agents as `.agent.md` files in `agents/`
directories.

Lower confidence, verify before relying on: exact nested-AGENTS.md merge semantics, precise CLI
flag syntax (may have changed since this was written), whether any newer per-directory
instructions convention has shipped since. Re-check against
https://docs.github.com/en/copilot/how-tos/copilot-cli/ before treating any of this as gospel for
a specific CLI version.
