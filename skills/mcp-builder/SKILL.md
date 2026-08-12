---
name: mcp-builder
description: Use whenever the user wants to build a new MCP (Model Context Protocol) server to expose tools/resources/prompts to Claude or any MCP client. Covers scaffolding a server in TypeScript or Python, defining tools with proper schemas, choosing stdio vs HTTP/SSE transport, and registering/testing it locally with Claude Code.
---

# Building an MCP Server

MCP (Model Context Protocol) is the open standard for connecting LLM clients (Claude Code, Claude Desktop, etc.) to external tools and data. A server exposes **tools** (actions), **resources** (readable data), and **prompts** (reusable templates) over a JSON-RPC transport.

As of August 2026: the published stable TypeScript SDK is `@modelcontextprotocol/sdk` (npm, currently 1.30.x). A v2 SDK (`@modelcontextprotocol/server`) targeting the 2026-07-28 spec exists on the `main` branch but is not yet a stable release — default to the 1.x SDK below unless the user specifically wants the v2 beta. The Python SDK is `mcp` (installed via `pip install "mcp[cli]"`); most people build on top of its `FastMCP` class, or the standalone `fastmcp` package (currently 3.x, with a 4.0 beta) which offers a superset of features. Verify versions with `npm view @modelcontextprotocol/sdk version` / `pip index versions mcp` before pinning, since this moves fast.

## Quick reference

| Task | Where to look |
|---|---|
| Minimal TS server | [TypeScript quick start](#typescript-quick-start) |
| Minimal Python server | [Python quick start](#python-quick-start) |
| Writing a good tool description | [Defining a tool](#defining-a-tool) |
| stdio vs HTTP/SSE | [Transports](#transports) |
| Wire it into your harness | [Registering with your harness](#registering-with-your-harness) |
| Error handling | [Error handling](#error-handling) |
| Things that go wrong | [Common pitfalls](#common-pitfalls) |

## TypeScript quick start

```bash
mkdir my-mcp-server && cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node
```

`src/index.ts`:

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "weather-server",
  version: "1.0.0",
});

server.tool(
  "get_forecast",
  "Get the current weather forecast for a US city. Only call this when the " +
    "user asks about weather, temperature, or conditions for a specific " +
    "named location. Do not call for general climate questions.",
  {
    city: z.string().describe("City name, e.g. 'Austin' or 'Austin, TX'"),
    units: z.enum(["metric", "imperial"]).default("imperial")
      .describe("Temperature unit system"),
  },
  async ({ city, units }) => {
    const data = await fetchForecast(city, units); // your implementation
    return {
      content: [{ type: "text", text: JSON.stringify(data, null, 2) }],
    };
  }
);

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch((err) => {
  console.error("Fatal error:", err);
  process.exit(1);
});
```

Run it directly to sanity-check it starts without crashing:

```bash
npx tsx src/index.ts
# or after building: node dist/index.js
```

Note: `server.tool(name, description, zodRawShape, handler)` takes a **raw Zod shape object** (plain `{ key: z.type() }`), not `z.object({...})`. The SDK wraps it for you.

## Python quick start

```bash
pip install "mcp[cli]"
```

`server.py`:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather-server")

@mcp.tool()
def get_forecast(city: str, units: str = "imperial") -> str:
    """Get the current weather forecast for a US city.

    Only call this when the user asks about weather, temperature, or
    conditions for a specific named location. Do not call for general
    climate questions.

    Args:
        city: City name, e.g. "Austin" or "Austin, TX"
        units: "metric" or "imperial"
    """
    data = fetch_forecast(city, units)  # your implementation
    return json.dumps(data)

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

Run it:

```bash
python server.py
# or, for interactive testing:
mcp dev server.py
```

FastMCP infers the input schema from Python type hints and uses the function's docstring as the tool description — so write the docstring the same way you'd write a SKILL.md description: specific enough that the model knows exactly when (and when not) to call it.

## Defining a tool

The **description** is the single highest-leverage thing you write. The model decides whether and how to call the tool almost entirely from the name + description + parameter descriptions — it never sees your implementation. Treat it like writing a SKILL.md trigger:

- State exactly when to call it and when *not* to ("only call when X", "do not use for Y").
- Mention required context/preconditions (e.g. "requires an authenticated session").
- Keep parameter descriptions concrete with example values, not just types.
- Avoid vague, all-purpose descriptions like "Fetches data" — that's how a tool gets invoked for the wrong thing or invoked when unnecessary.

Schema validation:
- **TypeScript**: Zod raw shape passed to `server.tool(...)`, or a full `z.object()` schema plus `.describe()` on each field.
- **Python**: Pydantic-style type hints on the function signature (FastMCP), or explicit `inputSchema` JSON Schema if using the lower-level `mcp.server.Server` API instead of `FastMCP`.

Always validate untrusted input inside the handler even though the schema does basic shape checking — schemas don't catch things like path traversal, SQL injection, or out-of-range values you need domain checks for.

## Transports

| Transport | When to use |
|---|---|
| **stdio** | Local use with Claude Code / Claude Desktop. The client spawns your process and talks over stdin/stdout. No auth needed — the process boundary *is* the trust boundary. This is the default and simplest choice. |
| **Streamable HTTP** | Remote/hosted servers, or local servers you want multiple clients to share. Supports OAuth. This replaced the older HTTP+SSE transport in the spec; use it for anything new that needs to run as a network service. |
| **SSE (legacy)** | Older remote transport, still supported by many clients for backward compatibility. Prefer Streamable HTTP for new servers unless a client you must support only speaks SSE. |
| **WebSocket** | Rare — only when the server needs to push unprompted events to the client over a persistent connection. No OAuth support. |

TypeScript HTTP transport sketch:

```typescript
import express from "express";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";

const app = express();
app.use(express.json());

app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined, // stateless mode
  });
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.listen(3000);
```

Python (FastMCP) equivalent: `mcp.run(transport="streamable-http")` (add `mount_path`/`port` as needed), or `mcp.run(transport="sse")` for the legacy transport.

## Registering with your harness

Every MCP-compatible harness has its own way of registering a server — the commands below are Claude Code's, shown as a concrete worked example. Codex CLI registers servers under `[mcp_servers.<name>]` in `config.toml`; opencode uses the `mcp` block in `opencode.json`; other harnesses (Copilot CLI, Pi, Claude Desktop, etc.) each have their own config surface. Check your harness's own docs for the exact syntax rather than assuming the Claude Code commands below apply directly.

```bash
# stdio — local process, most common
claude mcp add --transport stdio my-server -- node dist/index.js
claude mcp add --transport stdio my-server -- python server.py
claude mcp add --env API_KEY=xxx --transport stdio my-server -- npx -y my-mcp-package

# HTTP — remote server
claude mcp add --transport http my-server https://api.example.com/mcp
claude mcp add --transport http my-server https://api.example.com/mcp \
  --header "Authorization: Bearer TOKEN"

# SSE — legacy remote
claude mcp add --transport sse my-server https://api.example.com/sse
```

Notes:
- `--` separates Claude's own flags from the server's launch command — required whenever your command has its own flags (e.g. `python server.py --port 8080`).
- Default scope is `local` (this project only, in `~/.claude.json`). Use `--scope project` to write a shareable `.mcp.json` checked into the repo, or `--scope user` to make it available across all your projects.
- After adding, run `claude mcp list` to confirm it shows `✔ Connected` (not `✘ Failed to connect`), or run `/mcp` inside a session to inspect status and re-authenticate if needed.
- `claude mcp add-json <name> '{...}'` lets you paste a raw config block (useful for `type: "ws"` or copying a config verbatim from a server's own docs).

## Error handling

Return errors as a normal tool result with `isError: true` — don't just `throw`. Throwing an unhandled exception from a stdio handler can crash the process or produce an opaque JSON-RPC error the model can't act on; returning a structured error result lets the model see what went wrong and retry or explain it to the user.

TypeScript:

```typescript
async ({ city }) => {
  try {
    const data = await fetchForecast(city);
    return { content: [{ type: "text", text: JSON.stringify(data) }] };
  } catch (err) {
    return {
      content: [{ type: "text", text: `Failed to fetch forecast: ${err.message}` }],
      isError: true,
    };
  }
}
```

Python:

```python
@mcp.tool()
def get_forecast(city: str) -> str:
    try:
        return json.dumps(fetch_forecast(city))
    except Exception as e:
        raise ToolError(f"Failed to fetch forecast: {e}")  # FastMCP surfaces this as isError
```

Reserve actually raising/rejecting for protocol-level failures (bad transport, malformed request) — that's handled by the SDK, not something you need to implement yourself.

## Common pitfalls

- **Overly broad tool descriptions.** "Search the database" invites the model to call it for anything remotely search-shaped. Name the domain, the inputs, and explicit non-goals.
- **Too many tools, thin descriptions.** A server with 40 loosely-described tools degrades tool selection for every client that connects it, not just yours. Prefer fewer, well-scoped tools; split rarely-used ones into a separate optional server.
- **Missing input validation.** Schema types alone won't stop injection, path traversal, or unreasonable values (e.g. `limit: 999999999`). Validate inside the handler.
- **Blocking the event loop.** In TS, don't do synchronous heavy computation or blocking I/O in a tool handler — await everything. In Python with `FastMCP`, prefer `async def` tool functions for any I/O-bound work so the server stays responsive to other requests.
- **Not handling client disconnects.** Long-running tools (polling, streaming) should check for cancellation/abort signals and clean up subprocesses, sockets, or timers instead of leaking them on disconnect.
- **Silent stdout pollution (stdio transport).** Anything your process prints to stdout that isn't a JSON-RPC message corrupts the protocol stream. Log to stderr, never `console.log`/`print` to stdout for debugging.
- **Forgetting version pinning.** Both SDKs are evolving quickly (see the v2 TS SDK and FastMCP 4.0 beta noted above). Pin exact versions in production and re-verify against current docs before upgrading.
- **No timeout on outbound calls.** A tool handler that hangs on an upstream API hangs the whole exchange from the client's perspective — set explicit timeouts on any network call inside a handler.
