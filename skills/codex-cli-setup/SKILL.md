---
name: codex-cli-setup
description: Use whenever the user wants to audit or improve their OpenAI Codex CLI setup — config.toml, AGENTS.md, approval/sandbox modes, MCP servers, or profiles.
---

# Codex CLI Setup

Practical checklist for auditing or improving a Codex CLI (OpenAI's terminal coding agent) configuration. Codex CLI is a *different* harness from Claude Code — do not assume Claude Code conventions (settings.json, CLAUDE.md, permission modes) carry over directly, even though the concepts rhyme.

> **Confidence note:** This was compiled from OpenAI's current docs (`developers.openai.com/codex`, which redirect to `learn.chatgpt.com/docs/...`) plus the `openai/codex` GitHub repo. Codex CLI's config surface has changed multiple times (e.g. `--profile` used to read `[profiles.name]` from `config.toml`; newer versions read a separate `~/.codex/<name>.config.toml` file instead). Treat anything marked **[verify]** below as something to double-check against the live docs before relying on it — don't take it as settled fact.

## Config file location and format

- Primary user config: `~/.codex/config.toml` (TOML format).
- Project-scoped override: `.codex/config.toml` in the repo root — only honored for **trusted** projects, and it cannot override provider/auth/notification/profile/telemetry keys **[verify]**.
- System-wide (Unix): `/etc/codex/config.toml`.
- Precedence, highest to lowest: CLI flags/`--config` overrides → project `.codex/config.toml` → profile file → user `~/.codex/config.toml` → system config → built-in defaults. **[verify — precedence order specifically]**

### Key top-level settings

```toml
model = "gpt-5.5"                       # model id — check current names, these change often
model_provider = "openai"               # references [model_providers.<id>] if using a custom/proxy endpoint
model_reasoning_effort = "medium"       # minimal | low | medium | high | xhigh
approval_policy = "on-request"          # see Approval modes below
sandbox_mode = "workspace-write"        # see Sandbox modes below
```

## The `AGENTS.md` convention

Codex CLI popularized `AGENTS.md` as a project-instructions file — the direct analog of Claude Code's `CLAUDE.md`. It's now a semi-open convention other tools have started reading too.

- **Global**: `~/.codex/AGENTS.md` — instructions applied to every project.
- **Repo root**: `AGENTS.md` at the project root — project-specific, overrides/adds to global.
- **Nested**: subdirectory `AGENTS.md` files scoped to that directory tree are supported **[verify — the exact merge/override semantics are still being refined upstream; see openai/codex issue #12115 on nested-file loading clarity]**.
- There is an open feature request (`--agents <name>` flag, issue #10067) to switch between named `AGENTS.<name>.md` variants — not yet shipped as of this research, don't assume it exists.

Quality guidance (same principles as CLAUDE.md):
- Keep it concise and load-bearing — every line should change what the agent actually does, not restate obvious project facts.
- Put commands (build/test/lint), architectural gotchas, and house conventions here, not general programming advice.
- Re-read it periodically for staleness: renamed scripts, removed directories, or workflow changes that happened after the file was written are the most common source of drift.
- Don't duplicate the README — AGENTS.md is agent operating instructions, not user-facing docs.

## Approval and sandbox modes

Two separate axes control autonomy. Don't confuse them.

**`approval_policy`** — when Codex pauses to ask before acting:
- `untrusted` — approval required for anything risky.
- `on-request` — Codex asks before running commands it generates (a middle ground).
- `never` — runs without prompting (org-level policy may still restrict this).
- Granular form: `approval_policy = { granular = { sandbox_approval = ..., rules = ..., mcp_elicitations = ..., request_permissions = ..., skill_approval = ... } }` for per-category control. **[verify — this granular form looked like a newer/more advanced feature; confirm it exists in your installed version before relying on it]**

Note: the *very old* Codex CLI terminology (`suggest` / `auto-edit` / `full-auto`) from the original 2023 release has been superseded by the above — if you see that terminology in older blog posts or your own memory, it's stale.

**`sandbox_mode`** — what Codex is allowed to touch while executing, enforced at the OS level, not just by the agent:
- `read-only` — no filesystem writes, no network.
- `workspace-write` — can write inside the project workspace; network access and extra writable roots are configured separately via `sandbox_workspace_write` (`network_access`, `writable_roots`, `exclude_slash_tmp`, `exclude_tmpdir_env_var`).
- `danger-full-access` — sandboxing fully disabled; only use this if the surrounding environment (e.g. a disposable VM/container) already isolates the process.

Tradeoff: `read-only` + `untrusted` is safest but interrupts flow constantly; `workspace-write` + `on-request` is the common default for real work; `danger-full-access` + `never` should be reserved for already-isolated/ephemeral environments (CI, throwaway containers), never a developer's main machine.

There's also a `permissions.<name>` custom-profile system with `:read-only` / `:workspace` / `:danger-full-access` built-ins and an `extends` mechanism for finer filesystem/network rules. **[verify — this looked like a more advanced/newer layer on top of sandbox_mode; confirm current syntax before writing configs against it]**

## MCP server support

Codex CLI supports MCP servers, configured directly in `config.toml`.

Stdio server:
```toml
[mcp_servers.my-server]
command = "npx"
args = ["-y", "@some-org/some-mcp-server"]
env = { MY_ENV_VAR = "value" }
```

Streamable HTTP server:
```toml
[mcp_servers.figma]
url = "https://mcp.figma.com/mcp"
bearer_token_env_var = "FIGMA_OAUTH_TOKEN"
```

Useful optional keys: `enabled` (toggle without deleting), `required` (fail startup if unreachable), `startup_timeout_sec` / `tool_timeout_sec`, `enabled_tools` / `disabled_tools` (allow/block lists), `default_tools_approval_mode`. **[verify exact key names against your Codex version — this is a fast-moving part of the config surface]**

## Profiles

Codex supports named profiles to switch between setups (e.g. a fast/cheap profile vs. a deep-reasoning profile).

- Newer versions: create a separate file `~/.codex/<profile-name>.config.toml` with plain top-level keys (`model`, `model_reasoning_effort`, `approval_policy`, etc.), then invoke with `codex --profile <name>` or `codex exec --profile <name> "..."`.
- Older versions read `[profiles.<name>]` tables inline inside `config.toml` — **this is deprecated as of Codex 0.134.0**; `--profile` stopped reading it. If you find `[profiles.*]` tables in an existing `config.toml`, flag it as likely stale and migrate to the separate-file form. **[verify against the user's installed Codex version — `codex --version`]**

## Audit procedure

1. **Locate config**: check for `~/.codex/config.toml`, any project-level `.codex/config.toml`, and any `~/.codex/*.config.toml` profile files. Note the Codex version (`codex --version`) since config surface has shifted across versions.
2. **Check AGENTS.md**:
   - Does one exist at the repo root? At `~/.codex/AGENTS.md`?
   - Read it for staleness — referenced commands/paths that no longer exist, decisions the project has since reversed.
   - Check length/density — bloated or vague files get skimmed less effectively than tight, concrete ones.
3. **Check approval_policy and sandbox_mode** against the work being done:
   - Solo dev on their own machine doing normal feature work → `workspace-write` + `on-request` is typically reasonable.
   - Anything with `danger-full-access` or `never` outside an isolated/ephemeral environment is a flag — ask why.
   - If a `[profiles.*]` table exists inline in `config.toml`, flag as possibly-stale (see Profiles section).
4. **Check MCP servers**: list `[mcp_servers.*]` entries, confirm each is still needed (`enabled = true` but unused = clutter/attack surface), check secrets are referenced via `env`/`bearer_token_env_var` rather than hardcoded inline.
5. **Check profiles**: confirm any `~/.codex/*.config.toml` profile files are current and actually used; look for drift between profiles (e.g. one profile still pointing at a retired model id).
6. **Produce a prioritized fix list**: security-relevant issues first (overly permissive sandbox/approval settings, hardcoded secrets), then staleness (dead AGENTS.md content, deprecated `[profiles.*]` syntax, retired model ids), then polish (verbose AGENTS.md, unused MCP servers).

## What to explicitly re-verify

This document was written from current OpenAI docs but Codex CLI's configuration surface is young and changes across releases. Before acting on this skill's specifics, re-check against `https://developers.openai.com/codex` (redirects to `learn.chatgpt.com/docs/...`) or `codex --help` / `codex config --help` if available:
- The granular `approval_policy` object form and the `permissions.<name>` profile system — these looked like newer/more advanced layers and may not be present in older Codex versions.
- Exact current model ids (they change frequently; don't hardcode one from this doc into a user's config without checking).
- Nested `AGENTS.md` merge/override semantics — still being refined upstream as of this research.
- Whether `--profile` in the user's installed version reads inline `[profiles.*]` tables or requires separate files (version-dependent, see Profiles section).
