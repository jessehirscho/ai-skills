---
name: plugin-creator
description: Use whenever the user wants to build a distributable Claude Code plugin or plugin marketplace — a package bundling skills, subagents, slash commands, hooks, and/or MCP servers into an installable unit, versus a one-off skill or agent file. Covers plugin.json / marketplace.json format, local testing, and publishing to GitHub. For the content of individual skills or agents, see skill-builder and agent-builder — this skill only covers packaging and distribution.
---

# Plugin Creator

Builds and publishes Claude Code **plugins** — self-contained directories of
skills/agents/commands/hooks/MCP config — and **marketplaces**, the catalogs
that make plugins installable by others via `/plugin marketplace add` +
`/plugin install`.

This skill is a superset of `skill-builder` and `agent-builder`: use those to
write the *content* of an individual `SKILL.md` or agent definition, then use
this skill to package one or more of them into an installable, versioned
unit. Don't duplicate their authoring guidance here — reference them.

Verified against code.claude.com/docs (plugins, plugin-marketplaces,
plugins-reference) — re-check those pages if this drifts, since the format
has changed before.

## Plugin vs. plain skills folder — decide this first

| | Plain skills folder (this repo, `ai-skills`) | Plugin marketplace |
|---|---|---|
| Setup | Just files in `skills/<name>/SKILL.md` | `.claude-plugin/plugin.json` + optional `marketplace.json` |
| Install | Manual copy/symlink into `~/.claude/skills/` | `/plugin marketplace add` + `/plugin install` |
| Updates | Manual re-sync | `/plugin marketplace update`, versioned |
| Namespacing | None — flat names | Skills namespaced as `/plugin-name:skill-name` |
| Best for | Personal, single-machine, low ceremony | Sharing across machines/teams, versioned releases, mixing skills+agents+hooks+MCP |

This repo (`ai-skills`) deliberately stays a plain folder — simplicity over
install/update tooling. Reach for a plugin/marketplace only when the user
explicitly wants installable, versioned, or team-shared distribution, or
needs to bundle hooks/MCP servers/commands alongside skills (a plain folder
only carries skills).

## Plugin anatomy

A plugin is a directory. Only `plugin.json` lives inside `.claude-plugin/` —
every other directory sits at the plugin **root**, not nested inside
`.claude-plugin/`. This is the single most common structural mistake.

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json       # manifest: name, description, version, author...
├── skills/                # each subdir = one skill (see skill-builder)
│   └── code-review/
│       └── SKILL.md
├── agents/                 # subagent definitions (see agent-builder)
│   └── reviewer.md
├── commands/                # flat-file slash commands (legacy skill style)
├── hooks/
│   └── hooks.json          # event handlers, same shape as settings.json hooks
├── .mcp.json                # MCP server configs
├── .lsp.json                 # LSP server configs
└── settings.json               # default settings applied when plugin enabled
```

A plugin shipping exactly one skill can skip `skills/` and put `SKILL.md`
directly at the plugin root; use the `skills/` layout as soon as it may grow
past one.

`plugin.json` minimal example:

```json
{
  "name": "my-plugin",
  "description": "What this plugin does",
  "version": "1.0.0",
  "author": { "name": "Your Name" }
}
```

`name` is required and becomes the namespace prefix — a skill in
`skills/hello/` becomes `/my-plugin:hello`. `description` and `version` are
strongly recommended even though technically optional; without `version`,
update detection falls back to a content hash (see Versioning below).

Skills/agents inside a plugin follow exactly the same authoring rules as
standalone ones — see `skill-builder` and `agent-builder` for how to write
good frontmatter, descriptions, and body content. Nothing about being inside
a plugin changes that.

## Marketplace anatomy

A marketplace is a separate catalog file, `.claude-plugin/marketplace.json`,
that lists one or more plugins and where to fetch each from. It can live in
the same repo as the plugin(s) it lists, or a separate repo entirely.

```json
{
  "name": "my-plugins",
  "owner": { "name": "Your Name" },
  "plugins": [
    {
      "name": "quality-review-plugin",
      "source": "./plugins/quality-review-plugin",
      "description": "Adds a quality-review skill for quick code reviews"
    }
  ]
}
```

Required: `name` (kebab-case, public-facing — users type
`plugin@marketplace-name`), `owner`, `plugins` (array). Each plugin entry
needs at minimum `name` and `source`.

`source` can be:
- A relative path (`"./plugins/foo"`) — resolves against the marketplace
  root (the directory containing `.claude-plugin/`), for plugins living in
  the same repo. No `../` outside the marketplace root.
- `{ "source": "github", "repo": "owner/repo", "ref": "v2.0.0" }` — a
  separate GitHub repo, optionally pinned to a branch/tag/`sha`.
- `{ "source": "url", "url": "https://..." }` — any git remote.
- `{ "source": "git-subdir", "url": "...", "path": "tools/plugin" }` — a
  subdirectory of a monorepo, fetched via sparse clone.
- `{ "source": "archive", "url": "https://.../plugin.zip", "sha256": "..." }`
  — a zip archive over HTTPS, no git/npm required on the install side.

The marketplace `source` (what you pass to `/plugin marketplace add`) and
each plugin's `source` field are independent and pinned separately — a
marketplace repo can list plugins that live in entirely different repos.

## Minimal working example

One repo containing a marketplace that lists one plugin containing one skill:

```
my-marketplace/
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    └── quality-review-plugin/
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
            └── quality-review/
                └── SKILL.md
```

`my-marketplace/.claude-plugin/marketplace.json`:
```json
{
  "name": "my-plugins",
  "owner": { "name": "Your Name" },
  "plugins": [
    {
      "name": "quality-review-plugin",
      "source": "./plugins/quality-review-plugin",
      "description": "Adds a quality-review skill for quick code reviews"
    }
  ]
}
```

`my-marketplace/plugins/quality-review-plugin/.claude-plugin/plugin.json`:
```json
{
  "name": "quality-review-plugin",
  "description": "Adds a quality-review skill for quick code reviews",
  "version": "1.0.0",
  "author": { "name": "Your Name" }
}
```

`my-marketplace/plugins/quality-review-plugin/skills/quality-review/SKILL.md`:
```markdown
---
description: Reviews code for best practices and potential issues. Use when reviewing code, checking PRs, or analyzing code quality.
---

When reviewing code, check for:
1. Code organization and structure
2. Error handling
3. Security concerns
4. Test coverage
```

## Installing from a marketplace

```shell
/plugin marketplace add owner/repo          # or a local path, or a full git URL
/plugin install quality-review-plugin@my-plugins
```

(`claude plugin marketplace add ...` / `claude plugin install ...` work
equivalently from the shell, outside an interactive session.) If the install
summary says "Run /reload-plugins to activate," do that. Updates land with
`/plugin marketplace update` (refresh the catalog) — actual plugin upgrades
follow the versioning rules below.

## Versioning and updates

- Set `version` in `plugin.json` (or override per-entry in `marketplace.json`)
  and bump it on every release — users only receive an update when this
  string changes.
- If `version` is omitted everywhere, Claude Code falls back to a content
  hash (or, for archive sources, the zip's `sha256`) as the implicit version
  signal — functional but opaque; explicit versions are easier for users to
  reason about.
- Git-based plugin sources can additionally pin `ref` (branch/tag) and `sha`
  (exact commit) independently of the marketplace's own pin.

## Slash commands and hooks (pointer-level)

A plugin isn't limited to skills+agents:
- **Commands**: `commands/` holds flat Markdown files, the legacy
  slash-command style (prefer `skills/` for new work — same invocation
  surface, better structure).
- **Hooks**: `hooks/hooks.json` uses the same schema as the `hooks` key in
  `settings.json` (matcher + command per event). To migrate existing hooks
  from a project's `.claude/settings.json`, copy the `hooks` object in as-is.
- **MCP servers**: `.mcp.json` at the plugin root, same shape as a
  standalone MCP config.

Deep guidance on writing hooks or MCP configs belongs in their own skills if
this repo grows one — this skill only covers where they sit inside a plugin.

## Step-by-step procedure

1. **Decide scope.** One plugin only, or a marketplace listing several? A
   marketplace is worth it as soon as you want versioned install/update or
   plan to add a second plugin later; for a single one-off share, a bare
   plugin directory people `--plugin-dir` or clone is enough.
2. **Scaffold the manifest(s).** Create `.claude-plugin/plugin.json` per
   plugin, and `.claude-plugin/marketplace.json` at the repo root if
   distributing via marketplace. `claude plugin init <name>` will scaffold a
   single plugin with a starter skill under `~/.claude/skills/` if you want a
   quick local starting point instead of hand-writing it.
3. **Add skills/agents/commands/hooks.** Follow `skill-builder` for each
   `SKILL.md`, `agent-builder` for each agent definition. Keep them at the
   plugin root (`skills/`, `agents/`, `commands/`, `hooks/`), never nested
   inside `.claude-plugin/`.
4. **Test locally** before publishing:
   ```shell
   claude --plugin-dir ./my-plugin          # load one plugin directly, no install
   ```
   or, to test the full marketplace flow end to end:
   ```shell
   /plugin marketplace add ./my-marketplace
   /plugin install my-plugin@my-marketplace
   ```
   Run `claude plugin validate ./my-plugin` before sharing widely — it's the
   same check any community-marketplace submission pipeline runs.
5. **Publish to GitHub.** Push the repo. Nothing special is required beyond
   normal git hosting — GitHub, GitLab, self-hosted all work as marketplace
   or plugin sources.
6. **Document install instructions in the README**, e.g.:
   ```shell
   /plugin marketplace add <owner>/<repo>
   /plugin install <plugin-name>@<marketplace-name>
   ```

## Common pitfalls

- **Nesting `skills/`/`agents/`/`commands/`/`hooks/` inside `.claude-plugin/`.**
  Only `plugin.json` (and nothing else) goes inside `.claude-plugin/`; every
  component directory is a sibling of `.claude-plugin/`, at the plugin root.
- **Adding a plugin's files without adding it to `marketplace.json`.** A
  plugin directory that exists on disk but has no entry in the `plugins`
  array of `marketplace.json` is invisible to `/plugin install` — always
  update the catalog when you add a new plugin to the repo.
- **Path mismatches between `source` and the actual directory.** `source`
  paths in `marketplace.json` resolve relative to the marketplace root (the
  dir containing `.claude-plugin/`), not relative to the plugin itself, and
  `../` outside that root is disallowed — double check the relative path
  after any directory rename.
- **Forgetting `/reload-plugins`** after local edits during `--plugin-dir`
  testing — changes to skills/agents/hooks otherwise aren't picked up until
  restart.
- **Distributing via a raw URL to `marketplace.json` itself** rather than a
  git/GitHub/archive source — relative plugin paths won't resolve because
  Claude Code only downloads that one file. Use a GitHub repo, git URL, or
  archive source for anything with local (`./`) plugin paths.
- **No `version` set anywhere**, then wondering why users aren't picking up
  changes — without an explicit version bump, updates rely on an implicit
  content hash, which works but gives no human-readable signal of what
  changed.
