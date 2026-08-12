---
name: pi-harness-setup
description: Use whenever the user wants to audit or improve their Pi coding-agent harness setup — Pi is the minimal, extensible terminal coding agent from earendil-works (github.com/badlogic/pi-mono, pi.dev), distinct from Raspberry Pi and Inflection's Pi chatbot. Covers settings.json, AGENTS.md, trust model, extensions, skills, and CLI flags.
---

# Pi Harness Setup

Audits and improves a user's configuration of **Pi**, the terminal coding-agent
harness published as `@earendil-works/pi-coding-agent` (repo:
`badlogic/pi-mono`, package dir `packages/coding-agent`; site: pi.dev). Pi
bills itself as a minimal harness you reshape around your own workflow rather
than forking — no built-in MCP, a project-trust model instead of per-action
permission prompts, and everything extensible via TypeScript extensions and
Agent-Skills-standard skills.

Note: "Pi" is a common/ambiguous name. Don't confuse this with Raspberry Pi,
Inflection AI's "Pi" chatbot, or any other unrelated product — this skill is
specifically about the coding-agent CLI harness above.

## Confidence note — read before trusting details below

Pi is a small, actively-developed OSS project with less documentation surface
than the bigger harnesses. File locations, flag names (`--tools`,
`--exclude-tools`, `--approve`/`--no-approve`, `--mode json`/`--mode rpc`),
and the exact built-in tool list below were compiled from the project's docs
and repo at one point in time and have **not** been cross-checked against a
running `pi --help`. Treat this file as a starting map, not ground truth —
confirm specifics against `pi --help` or the current pi.dev docs before
relying on an exact flag or file path in a script.

## Config surface

| Purpose | Location |
|---|---|
| Global settings | `~/.pi/agent/settings.json` |
| Project settings (overrides global) | `.pi/settings.json` |
| Trust decisions (persisted) | `~/.pi/agent/trust.json` |
| Global instructions/context | `~/.pi/agent/AGENTS.md` (or `CLAUDE.md`) |
| Project instructions/context | `AGENTS.md` (or `CLAUDE.md`) in cwd and parent dirs — all matching files are concatenated |
| Override instructions for a dir | `AGENTS.override.md` in that directory (takes precedence over `AGENTS.md`/`CLAUDE.md` there) |
| Replace default system prompt | `.pi/SYSTEM.md` (project) or `~/.pi/agent/SYSTEM.md` (global) |
| Append to system prompt | corresponding `APPEND_SYSTEM.md` |
| Global skills | `~/.pi/agent/skills/` |
| Project skills | `.pi/skills/` or `.agents/skills/` |
| Extensions | loaded via `-e`/`--extension`, or referenced in settings |

Config files are plain JSON (`settings.json`) and Markdown (`AGENTS.md`,
`SYSTEM.md`). Both a global and a project layer exist, and project overrides
global — audit both layers, don't assume one is authoritative.

## Audit checklist

1. **Locate and read both settings layers.** Check `~/.pi/agent/settings.json`
   and `.pi/settings.json` in the project. Note conflicts — project wins.
2. **Locate and read the instructions stack.** Pi concatenates every
   `AGENTS.md`/`CLAUDE.md` it finds from `~/.pi/agent/`, each parent
   directory, down to the cwd, plus any `AGENTS.override.md` in a given
   directory (which suppresses the normal file there). Read the resulting
   effective instructions, not just one file in isolation — duplication or
   contradiction across layers is a common issue.
3. **Check the trust model, not permission prompts.** Pi doesn't do
   per-action approval popups by default; instead it asks once whether to
   trust a project folder, and saves that to `~/.pi/agent/trust.json`. For
   non-interactive use (`-p`, `--mode json`, `--mode rpc`), confirm
   `defaultProjectTrust` (`ask` / `always` / `never`) is set deliberately
   rather than left to prompt in a script, and check whether `--approve` /
   `--no-approve` flags are used appropriately in any automation.
4. **Check tool allow/deny lists.** Built-in tools are `read`, `bash`,
   `edit`, `write`, `grep`, `find`, `ls`. Verify `--tools` / `--exclude-tools`
   (or the equivalent settings.json fields) match intent — e.g. a read-only
   audit context should exclude `bash`/`edit`/`write`.
5. **Check extensions, not MCP.** Pi has no built-in MCP support by design;
   its documented alternative is either a CLI tool with a README (discoverable
   as a skill) or a hand-built extension registering tools via
   `pi.registerTool()`. If the user wants "MCP-like" integrations, look for or
   help build an extension rather than assuming MCP config exists.
6. **Check skills placement.** Confirm skills live in one of the recognized
   dirs (`~/.pi/agent/skills/`, `.pi/skills/`, `.agents/skills/`) and follow
   the Agent Skills standard (a `SKILL.md` per skill). Skills are invoked
   explicitly via `/skill:name` or picked up automatically by the model —
   check the user's expectation matches which mode they're relying on.
7. **Confirm the operating mode matches the use case.** Interactive TUI is
   the default; `-p`/`--print` for one-shot scripted use; `--mode json` for
   event-stream integration; `--mode rpc` for embedding via stdin/stdout
   JSONL. Automation scripts should use `-p` or `--mode json`/`rpc`, not the
   interactive default.
8. **Global vs. project split.** For each setting found, confirm it's at the
   layer the user intends — personal defaults belong in `~/.pi/agent/`,
   team/repo-shared conventions belong in the project's `.pi/` (and get
   committed), same as the global/project split in other harnesses.

## What to flag as a likely misconfiguration

- Instructions duplicated near-verbatim across the global `AGENTS.md` and a
  project `AGENTS.md` (redundant, wastes context — Pi is explicitly tuned to
  be token-efficient via a minimal system prompt).
- `defaultProjectTrust: "always"` (or scripts always passing `--approve`) in
  a repo that runs untrusted or third-party code — defeats the trust model.
- Broad tool access (no `--exclude-tools`) for a harness instance meant to be
  read-only/advisory.
- Assuming an MCP server config exists somewhere — it doesn't; redirect to
  extensions or skills instead.
