---
name: ucode-harness-setup
description: Use whenever the user wants to audit or improve their ucode coding-agent harness setup — specifically the Databricks `ucode` launcher (github.com/databricks/ucode) that runs Claude Code, Codex, Gemini CLI, OpenCode, Copilot CLI, Pi, or Cursor Agent through Databricks AI Gateway.
---

# ucode harness setup

## What "ucode" is (verified)

`ucode` — Unity AI Gateway Coding CLI, from `github.com/databricks/ucode` — is a lightweight
**launcher**, not a coding-agent harness in its own right. It wraps existing agent CLIs (Claude
Code, Codex, Gemini CLI, OpenCode, GitHub Copilot CLI, Pi, Cursor Agent) and routes their model
traffic through Databricks AI Gateway using workspace credentials, so you don't manage separate
API keys per tool.

If the user means a different "ucode" (e.g. an unrelated coding bootcamp/platform, or some other
tool), this skill does not apply — confirm which one before using it.

Install: `uv tool install git+https://github.com/databricks/ucode` (requires Python 3.12+; `npm`
optional, used to auto-install the underlying agent CLIs).

Launch an agent through the gateway: `ucode claude`, `ucode codex`, `ucode gemini`,
`ucode opencode`, `ucode copilot`, `ucode pi`, or `ucode cursor`. First launch prompts for the
Databricks workspace URL and authenticates automatically.

## What ucode itself manages, not each agent's own settings

`ucode`'s job is to write/patch each underlying tool's *own* config so it points at the Databricks
gateway. It is not a second config surface competing with each tool's native one — auditing a
`ucode` setup means checking these files, in the format each underlying tool already expects:

| Tool | File ucode writes |
|---|---|
| Codex | `~/.codex/config.toml` |
| Claude Code | `~/.claude/settings.json` |
| Gemini CLI | `~/.gemini/.env` |
| OpenCode | `~/.config/opencode/opencode.json` |
| GitHub Copilot CLI | `~/.copilot/.env` |
| Pi | `~/.pi/agent/models.json` |
| Cursor Agent | `~/.cursor/mcp.json` (MCP servers only) |
| ucode itself (admin-authored, workspace-wide) | `~/.ucode/managed-state.json` |

`ucode` creates a backup before it overwrites any of these; `ucode revert` restores the
pre-`ucode` version if you want to unwind the gateway wiring.

## Audit checklist

1. **Confirm which agent(s) are wired through ucode.** Run `ucode <tool>` for the one(s) in use
   and check it launches without re-prompting for workspace auth — a repeat prompt usually means
   the config file above for that tool is missing or was reverted.
2. **Check `~/.ucode/managed-state.json`.** This is the workspace-wide config an org admin can
   author via `ucode setup` (including optional spend-based budget policies). If your org uses
   this, your personal per-tool configs are expected to be overridden by it on next launch —
   don't hand-edit the per-tool files if managed state is in play, edits will be clobbered.
3. **MCP servers**: run `ucode configure mcp` to register Databricks MCP servers. These are
   registered as local stdio servers that shell out to `ucode mcp-proxy`, which bridges to
   Databricks streamable-HTTP MCP endpoints and handles auth uniformly — you should not need to
   hand-write MCP server auth per tool once this is set up.
4. **Unity Catalog skills**: `ucode configure skills --location <schema>` downloads Unity Catalog
   skills to disk, or exposes them as MCP tools instead — decide which mode you want per project.
5. **Smart routing**: optional, enabled per agent with `--enable-smart-routing`; the setting
   persists per Databricks workspace once turned on.
6. **Multi-workspace / multi-profile**: if you work across more than one Databricks workspace, use
   `ucode configure --workspaces` or `--profiles` rather than hand-editing per-tool files — check
   which profile is active before assuming a config change applies to the workspace you think it
   does.
7. **Per-project vs global**: `ucode`'s own scope is the Databricks *workspace*, not the repo — it
   is not aware of per-project config the way, say, `.mcp.json` or a project's `CLAUDE.md` is.
   Each underlying tool's own project-vs-global split (e.g. Claude Code's project `.mcp.json` vs.
   user `~/.claude.json`) still applies independently and isn't touched by `ucode`.
8. **After any audit change**, re-launch the affected tool through `ucode` (not directly) so
   gateway auth stays wired, and re-check the specific per-tool file above rather than assuming
   the change stuck — `ucode` can silently re-patch files it manages on next launch if managed
   state or smart-routing flags are active.

## What this skill does not cover

The *content* of each underlying tool's instructions/context file (e.g. Claude Code's
`CLAUDE.md`/`SKILL.md`, Codex's `AGENTS.md`) is unaffected by `ucode` and out of scope here — use
that tool's own setup skill (e.g. `claude-code-setup`, `codex-cli-setup`) for that layer. This
skill is specifically for the Databricks-gateway wiring layer `ucode` owns.
