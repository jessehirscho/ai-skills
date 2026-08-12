---
name: mcp-tools-setup
description: Use whenever the user wants to discover, install, configure, or troubleshoot MCP servers/tools that already exist (an official server, a community server, a company's remote MCP endpoint). Covers scopes, the `claude mcp` CLI, `.mcp.json`, auth (env vars, headers, OAuth), and debugging a broken connection. Not for building a brand-new MCP server from scratch — see mcp-builder for that.
---

# MCP tools setup

Wiring existing MCP servers into Claude Code: finding one worth adding, installing it at the
right scope, handling auth, and debugging it when it won't connect.

## Scopes

| Scope | Flag | Stored in | Shared with team | Use for |
|---|---|---|---|---|
| Local | `--scope local` (default) | `~/.claude.json`, under this project's path | No | Personal/experimental servers, or ones with credentials you don't want in git |
| Project | `--scope project` | `.mcp.json` in project root | Yes, via version control | Servers the whole team should get automatically |
| User | `--scope user` | `~/.claude.json` | No | Personal utility servers you want in every project (e.g. a GitHub or filesystem server) |

Short flag is `-s`. If a server is defined at more than one scope, Claude Code uses the
highest-precedence one whole (no field merging): local > project > user > plugin-provided >
claude.ai connectors.

Project-scoped servers from a cloned repo's `.mcp.json` sit at "Pending approval" until you run
`claude` interactively and accept them (workspace trust). This is deliberate — don't be surprised
when a freshly cloned repo's `.mcp.json` servers aren't live yet.

## Adding a server

**Remote HTTP (preferred for cloud services):**

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp

# with a bearer token header
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"
```

**Remote SSE (deprecated, use only if a server has no HTTP endpoint):**

```bash
claude mcp add --transport sse asana https://mcp.asana.com/sse
```

**Local stdio (runs as a child process on your machine):**

```bash
# note the `--` separating Claude's own flags from the server's command/args
claude mcp add --env AIRTABLE_API_KEY=YOUR_KEY --transport stdio airtable \
  -- npx -y airtable-mcp-server
```

Everything after `--` is passed to the server untouched — without it, Claude Code tries to parse
the server's own flags (e.g. `--port`) as its own.

**Remote WebSocket** (persistent connection, server-pushed events) — no `--transport` shortcut,
must use JSON:

```bash
claude mcp add-json events-server \
  '{"type":"ws","url":"wss://mcp.example.com/socket","headers":{"Authorization":"Bearer YOUR_TOKEN"}}'
```

**From raw JSON** (when a vendor gives you a config blob):

```bash
claude mcp add-json weather-api \
  '{"type":"http","url":"https://api.weather.com/mcp","headers":{"Authorization":"Bearer token"}}'
```

**Import from Claude Desktop** (macOS/WSL only):

```bash
claude mcp add-from-claude-desktop
```

## Managing servers

```bash
claude mcp list              # all configured servers + live connection status
claude mcp get <name>        # details for one server, including Issue: on failure
claude mcp remove <name>     # delete the config entry
claude mcp login <name>      # run OAuth flow from the shell (no need to open /mcp)
claude mcp logout <name>     # clear stored OAuth credentials
```

Inside a session, `/mcp` shows live status per server (`✔ Connected`, `! Needs authentication`,
`✘ Failed to connect`, `⏸ Pending approval`), lets you authenticate, re-authenticate, or toggle a
server off without removing its config.

## Auth patterns

- **Env vars for stdio servers**: `--env KEY=value` (repeatable). Place at least one other flag
  between `--env` and the server name or the CLI misparses the name as another `KEY=value` pair.
- **Header-based auth for HTTP/SSE servers**: `--header "Authorization: Bearer <token>"` (short
  form `-H`).
- **OAuth 2.0**: for servers that support it, just add the server, then run `/mcp` (or
  `claude mcp login <name>`) and complete the browser sign-in. Tokens are stored securely
  (system keychain on macOS) and refreshed automatically.
  - If the server needs a specific redirect URI, pin it with `--callback-port 8080`.
  - If the server needs pre-registered OAuth app credentials (no dynamic client registration),
    pass `--client-id your-client-id --client-secret` (the secret prompts with masked input, or
    set it via `MCP_CLIENT_SECRET` in CI).
  - Restrict requested scopes with the `oauth.scopes` field in `.mcp.json` if the upstream server
    advertises more than you want to grant.
- **Non-OAuth custom auth** (Kerberos, short-lived tokens, internal SSO): use `headersHelper` in
  `.mcp.json` — a shell command that prints a JSON object of headers to stdout, re-run fresh on
  every connection/reconnect.

## `.mcp.json` (project scope)

```json
{
  "mcpServers": {
    "shared-server": {
      "type": "http",
      "url": "https://example.com/mcp"
    }
  }
}
```

- Commit this file when the whole team should get the server automatically.
- Never hardcode secrets in it — reference env vars instead:
  `${VAR}` expands to the env var's value, `${VAR:-default}` falls back if unset. Supported in
  `command`, `args`, `env`, `url`, and `headers`.
- A missing referenced var doesn't hard-fail the config; `claude mcp list` shows a warning and the
  literal `${VAR}` text is used, which typically makes the server fail to connect.
- Keep machine-specific or personal-credential servers out of `.mcp.json` — use local or user
  scope (`~/.claude.json`) for those instead.

## Discovering servers worth installing

- Browse the **Anthropic Directory** (claude.ai/directory) for reviewed connectors — anything
  listed there installs with the same `claude mcp add` flow.
- Well-known community/official servers people commonly add: GitHub (code review, issues, PRs),
  Slack, filesystem, Postgres/DBHub, Sentry (error monitoring), Notion, Asana, HubSpot.
- This list goes stale fast — treat it as illustrative, not authoritative. Check the current
  Directory and the server's own docs before adding, and verify you trust a server before
  connecting it (a malicious or compromised server can prompt-inject through tool output).

## Debugging a broken connection

1. Run `/mcp` (or `claude mcp list` / `claude mcp get <name>`) and read the status line —
   `claude mcp get` shows the specific `Issue:` (HTTP status code + server error text) for a
   failed connection.
2. **Common causes, roughly in order of likelihood:**
   - Missing/wrong env var for a stdio server (`--env` typo, or the var isn't actually exported
     in your shell/CI).
   - Wrong `--transport`/`type` — an entry with a `url` but no `type` is silently treated as
     stdio and skipped; `.mcp.json` copy-pasted from vendor docs sometimes omits `type`.
   - Stdio server process crashing on startup — test the raw command outside Claude Code first
     (e.g. `npx -y airtable-mcp-server` directly) to see the actual error.
   - Auth: expired OAuth token (re-run `/mcp` → Re-authenticate, or `claude mcp login <name>`),
     bad/placeholder header token (`claude mcp add` doesn't validate credentials at add time —
     a bad token only surfaces as a 401 on first connect).
   - Network/firewall blocking an HTTP/SSE endpoint, or a corporate proxy stripping headers.
   - Project-scoped server stuck at "Pending approval" because the workspace hasn't been trusted
     yet — run `claude` interactively in the repo and accept the trust dialog.
   - Leading/trailing whitespace in a pasted token — `claude mcp list` flags this explicitly
     (common when copy-pasting a token that has a trailing newline).
3. For HTTP/SSE servers, Claude Code auto-reconnects with backoff (up to 5 attempts) if the
   connection drops mid-session — a transient blip usually self-heals; a hard 4xx/auth failure
   won't.
4. Stdio servers are **not** auto-reconnected — if the process dies, remove/re-add or restart the
   session.

## Quick reference

| Task | Command |
|---|---|
| Add remote HTTP server | `claude mcp add --transport http <name> <url>` |
| Add remote HTTP server with auth header | `claude mcp add --transport http <name> <url> --header "Authorization: Bearer <token>"` |
| Add local stdio server | `claude mcp add --env KEY=val --transport stdio <name> -- <command> [args...]` |
| Add server from JSON | `claude mcp add-json <name> '<json>'` |
| Add at project scope (shared) | `claude mcp add ... --scope project` |
| Add at user scope (all projects) | `claude mcp add ... --scope user` |
| List servers + status | `claude mcp list` |
| Inspect one server | `claude mcp get <name>` |
| Remove a server | `claude mcp remove <name>` |
| Check live status in-session | `/mcp` |
| OAuth login from shell | `claude mcp login <name>` |
| OAuth logout | `claude mcp logout <name>` |
| Import from Claude Desktop | `claude mcp add-from-claude-desktop` |
