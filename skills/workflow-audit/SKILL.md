---
name: workflow-audit
description: Use whenever the user wants to audit how they're actually working with Claude Code across sessions, and get concrete recommendations to improve it. Mines local session transcripts for real usage patterns (tool calls, skills, subagents, slash commands), finds friction/waste/repetition, and turns it into specific fixes — new skills, CLAUDE.md additions, settings.json permission changes, memory entries. Not a codebase audit. Periodic/on-demand, not every session.
---

# Workflow Audit

Audits how the user actually uses Claude Code, based on real session transcripts, not
impressions — and turns findings straight into concrete changes: a skill to write, a
CLAUDE.md line to add, a permission to allowlist, a memory entry to record.

## When to use this

Use when the user asks something like "how am I actually using Claude Code," "what should
I automate," "audit my workflow," or "what's eating my time across sessions." Don't use it
to audit a codebase, a PR, or a single session's work — this is about behavior across
sessions.

## Scope: Claude Code only

This skill's transcript-mining technique depends on Claude Code's specific local session
storage — the `~/.claude/projects/<escaped-cwd>/*.jsonl` format and location. It does not
work for other harnesses (Codex CLI, Copilot CLI, Pi, opencode, etc.). If the user wants an
equivalent audit for another harness, first check whether that harness persists local
session logs at all — many don't, or only keep them in-memory/ephemeral. If it does, the
same friction-pattern-mining methodology here (tool-call frequency, repeated clarifying
questions, permission-prompt patterns) still applies conceptually, but the extraction
commands below need to be rewritten for that harness's actual log format and location —
don't assume they translate as-is.

## Where the data lives

Every Claude Code session is logged as JSONL, one file per session, one JSON object per
line, under:

```
~/.claude/projects/<escaped-cwd>/*.jsonl
```

The directory name is the project's working-directory path with `/` replaced by `-`
(e.g. `/Users/jesse/claude/projects/soar-solutions` becomes
`-Users-jesse-claude-projects-soar-solutions`). List all projects with:

```bash
ls ~/.claude/projects/
```

This is a verified location — this exact technique was used earlier in this repo's
history to find the user's most-used skills. Treat it as ground truth over guesses about
usage patterns.

## Extract signal without reading full transcripts

Reading whole transcripts into context is wasteful and mostly noise. Grep for structured
fields instead. Run these per-project or across all projects (`~/.claude/projects/*/*.jsonl`):

**Tool-call frequency** — what's actually getting invoked, and how often:
```bash
grep -oh '"name":"[A-Za-z_]*"' ~/.claude/projects/*/*.jsonl | sort | uniq -c | sort -rn | head -30
```

**Skill invocation frequency** — which skills get used vs. sit unused:
```bash
grep -oh '"skill":"[^"]*"' ~/.claude/projects/*/*.jsonl | sort | uniq -c | sort -rn
```
Cross-reference against every skill actually installed (`ls ~/.claude/skills/*/SKILL.md`
and any project-local `.claude/skills/`) to see which ones never fire.

**Slash command usage** — look for `<command-name>` tags in message content:
```bash
grep -oh '<command-name>[^<]*</command-name>' ~/.claude/projects/*/*.jsonl | sort | uniq -c | sort -rn
```

**Subagent usage** — which agent types get launched, and how often:
```bash
grep -oh '"subagent_type":"[^"]*"' ~/.claude/projects/*/*.jsonl | sort | uniq -c | sort -rn
```

**Session count and rough length per project** — files per project dir, lines per file:
```bash
for d in ~/.claude/projects/*/; do
  n=$(ls "$d"*.jsonl 2>/dev/null | wc -l)
  echo "$n sessions: $d"
done | sort -rn
wc -l ~/.claude/projects/<project>/*.jsonl | sort -rn | head -10   # longest sessions
```

**Tool-call-to-turn ratio for a specific long session** (flags possible thrashing):
```bash
wc -l < session.jsonl                                   # total lines (~turns+events)
grep -c '"type":"tool_use"' session.jsonl                # tool calls in that session
```

**Repeated user asks across sessions** (for spotting a recurring clarifying question or
recurring manual instruction — scan just the human-turn text, not the whole line):
```bash
grep -oh '"role":"user".\{0,200\}' ~/.claude/projects/*/*.jsonl | sort | uniq -c | sort -rn | head -20
```
This one is the most privacy-sensitive extraction — see the note at the bottom before
quoting anything it surfaces.

Pull counts across the whole `~/.claude/projects/` tree for the cross-project picture, and
per-project when the user cares about one repo specifically. Prefer running these directly
over spawning a subagent for it — the commands are cheap and the raw counts are what
matter, not narrative exploration.

## What counts as friction

- **Heavy repeated Bash/Grep for the same kind of lookup** — same tool run many times
  across sessions doing structurally the same thing (e.g. `grep -r TODO`, `git log --oneline
  -20`, a recurring build+deploy sequence). Candidate for a purpose-built skill or a script
  in `scripts/`.
- **The same clarifying question asked in multiple sessions** — surfaces as similar
  early-session user turns or assistant questions repeating across files. Signals a
  CLAUDE.md or memory gap: the answer should be written down once, not re-asked.
- **Long sessions with a high tool-call-to-output ratio** — many tool calls relative to
  what actually shipped. Possible sign of thrashing: unclear task scoping, a missing
  skill that would have shortcut the exploration, or a subagent that should have been
  used to keep the main thread token-light.
- **Skills that exist but never get invoked** — zero (or near-zero) hits in the skill
  invocation grep despite being installed. Either dead weight (delete it) or the
  `description` frontmatter doesn't match how the user actually phrases the trigger
  (rewrite the description).
- **Permission-prompt-heavy patterns** — the same read-only command class approved over
  and over. Candidate for a `settings.json` allowlist entry (see the `fewer-permission-prompts`
  and `update-config` skills for the mechanics of adding one).
- **Subagent types requested but never available**, or general-purpose agents spawned
  for tasks a specialized one would fit — signals a missing custom agent definition.

## Turn findings into fixes, not observations

Every row in the audit output must map to one of exactly four concrete actions. Don't stop
at "consider improving X" — name the file and the change.

| Friction pattern | Concrete fix |
|---|---|
| Repeated multi-step Bash sequence across sessions | New skill: write `SKILL.md` with the exact commands, or a script under `scripts/` the skill calls |
| Same fact/preference re-asked or re-explained | CLAUDE.md addition (project or global `~/.claude/CLAUDE.md`) stating it once |
| Same read-only command repeatedly permission-prompted | `settings.json` / `settings.local.json` allowlist entry — use the `update-config` skill |
| Durable fact about how the user works, not project-specific | Memory entry (`~/.claude/projects/<project>/memory/`) |
| Skill installed but never triggered | Rewrite its `description` frontmatter to match real phrasing, or delete it |
| Long thrash-y sessions on one task type | Recommend a subagent split or an existing skill (e.g. `dev-pipeline`, `subagents`) the user isn't using yet |

## Output format

Present findings as a single table, ordered by impact (highest-frequency / highest-cost
friction first), then a short list of the fixes actually applied or proposed:

```
| Pattern observed                          | Frequency         | Recommended fix                          |
|--------------------------------------------|--------------------|-------------------------------------------|
| `git log --oneline -30` + manual grep      | 14 sessions        | New skill: `recent-changes-summary`       |
| Asked "what's the deploy command" again    | 4 sessions         | Add to CLAUDE.md under Commands           |
| `npm run lint` approved every session      | 22 prompts         | Allowlist in settings.json                |
| `content-editor` skill installed, 0 hits   | 0 / 40 sessions    | Rewrite description or delete             |
```

Follow the table with what you're proposing to change and ask before writing anything
that touches `settings.json` permissions or deletes a skill — those are the two categories
worth a confirmation before acting, since they're either security-relevant or destructive.
CLAUDE.md additions and new skill drafts can be proposed inline and written once approved.

## Cadence

This is a periodic or on-demand audit, not a per-session habit. Re-running it against a
small amount of new data (a handful of sessions since the last audit) produces noise, not
signal — the frequency counts need enough sessions to be meaningful. A reasonable trigger
is "it's been a few weeks" or "a notable chunk of new sessions has accumulated," not "every
time I finish a task." If the user asks to run it again shortly after a previous run, say
so and confirm they still want it before repeating the full mining pass.

## Privacy and scope

This only reads the user's own local transcripts under `~/.claude/projects/` — nothing is
sent anywhere, and no other user's data is touched. Be conservative about quoting raw
prompt content back in the audit output: use it only to illustrate a pattern (e.g. a short
paraphrase of a recurring question), never dump full user turns verbatim into the report,
and never surface content unrelated to the friction pattern being reported.
