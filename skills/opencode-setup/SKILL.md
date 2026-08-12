---
name: opencode-setup
description: Use whenever the user wants to audit or improve their opencode setup — the open-source terminal coding agent (opencode.ai, github.com/sst/opencode). Covers opencode.json config, provider/model setup, AGENTS.md, MCP servers, and TUI keybindings.
---

# opencode Setup

This skill targets **opencode** — the open-source terminal coding agent at
opencode.ai, built by sst (github.com/sst/opencode). "opencode" is a fairly
generic name; before applying anything here, confirm the project in question
is this one (CLI binary `opencode`, config file `opencode.json`, docs at
opencode.ai/docs) and not an unrelated project that happens to share the name.

## Confidence note — read before trusting details below

Everything below was pulled from opencode.ai/docs during this skill's
research pass (config, rules/AGENTS.md, MCP servers, keybinds pages). It is a
fast-moving pre-1.0-feeling OSS project, so:

- **Verify against the live docs before acting on anything specific** —
  field names, defaults, and precedence order are the kind of thing that
  changes across releases. Treat this file as a starting map, not ground
  truth.
- The docs site appears to have two doc trees in the wild during research
  (`opencode.ai/docs/...` and `opencode.ai/v2/docs/...`), which suggests a
  docs version transition was in progress — double-check you're reading the
  current one.
- Default keybindings and exact TUI field names were captured from one pass
  over `opencode.ai/docs/keybinds/` and were not cross-checked against the
  running app; confirm with `opencode` itself (there is usually a
  command/help list in the TUI) rather than assuming the list below is
  exhaustive or current.

## Config file: location, format, precedence

Format: JSON or JSONC (JSON with comments). Config files reference a schema:
`"$schema": "https://opencode.ai/config.json"`.

Config sources are **merged**, not replaced — later sources override earlier
ones only for keys that actually conflict. As documented, load order is:

1. Remote config (`.well-known/opencode` endpoint)
2. Global config — `~/.config/opencode/opencode.json`
3. Custom path — `OPENCODE_CONFIG` env var
4. Project config — `opencode.json` in the project root (opencode walks up
   to the nearest `.git` directory to find it; safe to commit to the repo)
5. `.opencode` directories
6. Inline config — `OPENCODE_CONFIG_CONTENT` env var
7. Managed config files (system directories, for org-wide policy)
8. macOS managed preferences (MDM) — highest priority

TUI-specific settings (keybinds, appearance) live in a separate file:
`~/.config/opencode/tui.json`, schema `https://opencode.ai/tui.json`.

Variable substitution inside config values:
- `"{env:VAR_NAME}"` — environment variable (empty string if unset)
- `"{file:path/to/file}"` — file contents, relative to the config dir or
  absolute with `/` or `~` — handy for keeping API keys or large instruction
  blocks out of the committed config

## Provider / model configuration

opencode integrates the Vercel AI SDK plus the Models.dev registry, and is
documented as supporting 75+ providers this way — this is the project's
signature feature, worth checking first in any audit.

```json
{
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "{env:ANTHROPIC_API_KEY}",
        "timeout": 600000,
        "chunkTimeout": 30000
      }
    }
  },
  "model": "anthropic/claude-sonnet-4-5",
  "small_model": "anthropic/claude-haiku-4-5"
}
```

- Model IDs use `provider/model-id` format.
- `model` sets the default; `small_model` lets you point cheaper/faster
  tasks (e.g. planning) at a different model.
- Per-provider `options` cover auth (`apiKey`, usually via `{env:...}`),
  request `timeout` (ms, default reportedly 300000; `false` disables it),
  `chunkTimeout` for streaming, and provider-specific extras (e.g. an AWS
  region for Bedrock-backed providers).
- `enabled_providers` / `disabled_providers` allowlist or blocklist which
  providers load at all — useful to keep unused providers from being probed
  or from cluttering the model picker.
- Custom/self-hosted or overridden models can be added under a provider's
  config alongside the built-in registry entries.
- In the TUI, `ctrl+a` opens the provider/model list and `ctrl+f` toggles a
  model as favorite (per the keybinds doc — reconfirm live).

## Instructions file (AGENTS.md)

opencode's equivalent of `CLAUDE.md` is **`AGENTS.md`**. Documented search/
precedence order:

1. Project-level `AGENTS.md` in the repo root (applies to that directory and
   subdirectories)
2. Global `~/.config/opencode/AGENTS.md` (applies across all sessions)
3. Falls back to reading Claude Code's `CLAUDE.md` (project or
   `~/.claude/CLAUDE.md`) if present — so a repo with only a `CLAUDE.md` and
   no `AGENTS.md` is not necessarily un-instructed

Generate or refresh one with the `/init` slash command inside opencode — it
scans the repo, may ask a few targeted questions, and writes build commands,
architecture notes, and conventions into `AGENTS.md`.

Additional instruction sources can be pulled in via the `instructions` array
in `opencode.json` — paths, glob patterns, or (per the docs) remote URLs —
and all discovered instruction files are combined into the model's context
rather than only the single nearest one winning.

## MCP servers

Configured under the `mcp` key in `opencode.json`/`opencode.jsonc`, keyed by
server name.

Local (spawns a subprocess):
```json
{
  "mcp": {
    "my-server": {
      "type": "local",
      "command": ["npx", "-y", "my-mcp-command"],
      "environment": { "VAR": "value" },
      "enabled": true
    }
  }
}
```

Remote (HTTP/SSE endpoint):
```json
{
  "mcp": {
    "my-remote": {
      "type": "remote",
      "url": "https://mcp-server.com",
      "headers": { "Authorization": "Bearer {env:MCP_TOKEN}" },
      "enabled": true
    }
  }
}
```

Notes:
- `enabled: false` turns a server off without deleting its config.
- `timeout` defaults to 5000ms per the docs.
- The `tools` config key can disable specific tools or name patterns
  globally (`"tools": { "server-name": false, "pattern*": false }`), and
  agents can re-enable specific ones in their own config even when globally
  off.
- Docs explicitly warn that every enabled MCP server adds to context, so
  audits should treat "MCP servers enabled" as a cost, not a free win.

## TUI / keybindings

Customized via `~/.config/opencode/tui.json` (schema
`https://opencode.ai/tui.json`). Formats:

```json
"session_new": "<leader>n"
"messages_copy": "ctrl+shift+c,<leader>y"
"messages_copy": ["<leader>y", "ctrl+shift+c"]
"input_paste": { "key": "ctrl+v", "preventDefault": false }
```

- Default leader key is `ctrl+x`; leader-bound actions are pressed as two
  strokes (leader, then key). `leader_timeout` (default ~2000ms) controls
  how long opencode waits for the second stroke.
- Disable a binding with `"none"` or `false`.
- Windows has forced platform differences (e.g. `terminal_suspend` forced to
  `"none"` since POSIX suspend isn't available there) — don't assume a
  keybind config is portable across OSes without checking.
- Reportedly-common defaults worth checking live rather than trusting
  blindly: `ctrl+p` (command list), `ctrl+a` (model/provider list), `ctrl+f`
  (toggle favorite model), `tab`/`shift+tab` (cycle agents), `escape`
  (interrupt session).

## Audit procedure

1. **Locate config.** Check, in order: `opencode.json`/`.jsonc` in the
   project root, `~/.config/opencode/opencode.json`, and whether
   `OPENCODE_CONFIG` or `OPENCODE_CONFIG_CONTENT` is set in the shell env
   (these silently override file-based config and are easy to miss). Note
   which file actually owns each setting you're about to change — remember
   these merge, so a fix belongs in project config if it should travel with
   the repo, global config if it's a personal preference.

2. **Check the instructions file.** Does `AGENTS.md` exist at the project
   root? If not, is there a `CLAUDE.md` opencode is falling back to, and is
   that acceptable, or should the user run `/init` to generate a proper
   `AGENTS.md`? If one exists, read it for staleness: build commands that no
   longer work, missing architecture notes for code added since it was
   written, and whether anything in the `instructions` array in
   `opencode.json` (extra files/globs) is dead or missing.

3. **Check provider/model config.** Confirm `model` (and `small_model` if
   set) point to models that still exist and match what the user actually
   wants to be spending on. Confirm API keys are referenced via `{env:...}`
   or `{file:...}` rather than pasted in plaintext into a config file that
   might get committed. If `enabled_providers`/`disabled_providers` are set,
   confirm they match intent (e.g. a provider the user pays for isn't
   accidentally disabled, or a stale/unused provider isn't left enabled and
   probed).

4. **Check MCP / tool setup.** List servers under `mcp`; for each, confirm
   it's still needed, `enabled` matches intent, and secrets (`headers`,
   `environment`) use `{env:...}` substitution rather than literal values.
   Cross-check against the `tools` block for any global disables that might
   be silently suppressing something the user expects to work. Flag servers
   that are enabled but clearly unused — they cost context budget for no
   benefit.

5. **Check TUI/keybind config** (only if the user mentioned friction here):
   look at `~/.config/opencode/tui.json` for conflicting or unintentionally
   disabled bindings, and check `leader_timeout` if leader-key chords feel
   unreliable.

6. **Produce a prioritized fix list**, ordered roughly:
   - Secrets in plaintext config (fix first — security)
   - Broken/missing `AGENTS.md` or stale instructions (biggest quality lever)
   - Wrong or unintentionally-disabled default model/provider
   - Unused or misconfigured MCP servers (cost/context bloat)
   - Keybind/TUI polish (lowest priority, cosmetic)

   For each item, name the exact file and key to change, and — since this
   project's config format may have moved since this skill was written —
   suggest the user spot-check the relevant `opencode.ai/docs/*` page before
   applying anything non-trivial.
