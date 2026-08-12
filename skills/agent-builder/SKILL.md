---
name: agent-builder
description: Use whenever the user wants to create a new Claude Code subagent definition (a .claude/agents/<name>.md file invocable via the Agent tool). Distinct from mcp-builder (exposing tools via MCP) and skill-builder (SKILL.md files) — this is specifically about the YAML-frontmatter-plus-system-prompt subagent format.
---

# Agent Builder

A Claude Code subagent is a single file: `.claude/agents/<name>.md`. YAML
frontmatter declares how it's dispatched and what it's allowed to touch; the
body is its system prompt. This skill is a procedure and reference for
writing one well.

## File format

```markdown
---
name: <kebab-case-name>
description: <when the main session should dispatch to this agent>
tools: <comma-separated allowlist, e.g. Read, Grep, Glob>
model: <haiku | sonnet | opus>
---

<system prompt body>
```

- **name** — kebab-case, matches the filename (minus `.md`). This is what
  `Agent(subagent_type: "<name>")` references.
- **description** — the dispatch trigger. The main session (or a human)
  reads this to decide *whether this agent is the right tool for the task at
  hand*. It is matched against task descriptions, not read as documentation
  for later. See "Writing the description" below.
- **tools** — an explicit allowlist. Omit tools the agent has no reason to
  use. If omitted entirely, some harnesses default to "all tools" — always
  set it explicitly for anything other than a general-purpose agent.
- **model** — pick per the cost/judgment tradeoff (see below). Omitting it
  usually inherits the caller's model, which is rarely what you want for a
  narrow mechanical agent.

## Principle of least tool access

Grant only what the single responsibility requires. Tool access is the
actual safety boundary — a reviewer that can't call `Edit` or `Write`
*cannot* silently patch the code it was supposed to only critique, no matter
what its prompt says.

Worked examples from a real pipeline (`dev-pipeline/agents/*.md`):

| Agent | Responsibility | Tools | Why not more |
|---|---|---|---|
| `planner` | Write an implementation plan | `Read, Grep, Glob, Write` | No `Edit`/`Bash` — it must never touch source or run code, only read it and write one plan file |
| `critic` | Score a plan against a rubric | `Read` | Read-only by design — a critique agent that can write would be tempted to "just fix it" instead of reporting |
| `builder` | Implement an approved plan | `Read, Write, Edit, Bash` | Needs full read/write/execute to actually build; scope is enforced by the prompt body, not tools |
| `tester` | Run tests, report PASS/FAIL | `Bash, Read` | No `Edit`/`Write` — it must report failures, never silently patch code to make tests pass |
| `reviewer` | Review a diff pre-merge | `Bash, Read` | `Bash` only for `git diff`; no edit tools — same rationale as critic |

Rule of thumb: if you find yourself granting `Edit` "just in case," that's a
sign the agent's responsibility is underspecified, not that it needs the
tool.

## Model selection

- **haiku** — mechanical, low-judgment work with a clear pass/fail: running
  a test suite, linting, checking a build compiles, extracting data from
  well-structured text. Cheap and fast; use it whenever the task doesn't
  require weighing tradeoffs.
- **sonnet** — anything requiring judgment: planning, code review, critique
  against a rubric, implementation that has to make small design decisions.
  Default choice when unsure.
- **opus** — reserve for genuinely hard reasoning: architecture decisions
  with many interacting constraints, ambiguous specs needing inference, or
  when a sonnet-authored plan has already failed critique twice. Expensive;
  don't default to it.

## Writing the description (the dispatch trigger)

The description is matched against "what does the current task need," so it
must be specific about **when to use it**, not just what it does in the
abstract.

Weak (vague, competes with everything): `description: Reviews code.`

Strong (specific trigger + scope + when *not* to use it):
```
description: Reviews a branch's diff against its approved plan before
merge. Diff-scoped context only — never given the builder's reasoning
trail. Use for Phase 6 of the dev-pipeline skill.
```

Include, where relevant:
- The concrete trigger ("after a ticket's plan is marked APPROVE", "before
  merge")
- What it does NOT do, if there's a same-sounding agent it could be
  confused with (e.g. "critic reviews the *plan*, not the diff")
- Any phase/workflow name it's tied to, so it isn't picked out of context

## Writing the system prompt body

Scope it narrowly. An agent's tool allowlist stops it from doing things it
shouldn't be *able* to do; the prompt body is what stops it from doing
things it's *able* to do but shouldn't.

Include:
- **Exactly what to read and write.** Name file paths/patterns, not "the
  relevant files." A fresh agent has no context — vague pointers cause it
  to explore broadly and burn budget.
- **Explicit stop/escalate conditions.** "If the plan is wrong or
  incomplete once you're in the code, stop and report back — do not
  improvise a fix that goes beyond the plan." Without this, agents default
  to "be helpful" and quietly expand scope.
- **What NOT to do**, stated plainly, especially anything a reasonable
  agent might do by default (don't refactor unrelated code, don't fix
  tests by editing them, don't touch files outside declared scope).
- **The verdict/output format**, if the agent reports a judgment (e.g.
  `APPROVE`/`REVISE`, `PASS`/`FAIL`) — fix the vocabulary so callers can
  parse it reliably.

## Procedure

1. **Clarify the single responsibility.** One sentence. If it takes "and"
   to describe, it's two agents.
2. **Decide the minimal tool set.** List only what's needed to do exactly
   that responsibility — re-derive from scratch, don't copy another
   agent's tool list by default.
3. **Decide the model.** Mechanical/cheap → haiku. Judgment call → sonnet.
   Hard/ambiguous reasoning → opus.
4. **Write the description** so the main session (or a human) picks this
   agent for the right task and not a neighboring one. State the trigger,
   the scope, and what it's not for.
5. **Write the scoped system prompt.** File paths to read/write, explicit
   stop/escalate conditions, an explicit "do not" list, and an output
   format if it returns a verdict.
6. **Test it.** Invoke it via the Agent tool on a real task from the
   target repo and check: did it stay inside its declared file scope? Did
   it use only its granted tools? Did it stop/escalate correctly when the
   task went outside what the prompt covers? Tighten the prompt if not.

## Example agent definitions

**Narrow read-only reviewer** — flags issues, fixes nothing:

```markdown
---
name: a11y-auditor
description: Audits a set of changed frontend files for accessibility
issues (missing alt text, non-semantic interactive elements, contrast,
focus order). Use after UI changes, before a PR is opened. Read-only —
does not fix anything, only reports.
tools: Read, Grep, Glob
model: sonnet
---

You audit accessibility only. You will be given a list of changed files or
a diff. Read only those files.

Check for: missing/empty alt text, `<div onClick>` instead of a real
button, missing form labels, heading-level skips, focus traps, color
contrast below WCAG AA (flag, don't compute exact ratios from code alone).

Output a list of findings, each with file, line, and a one-sentence fix
suggestion. Do not edit any file. If a file has no issues, say so briefly
— don't pad the report.
```

**Scoped implementer** — builds one ticket, nothing else:

```markdown
---
name: migration-writer
description: Writes one database migration file matching an approved
schema-change ticket. Use only after the schema change has been reviewed
and approved; do not use for arbitrary schema exploration.
tools: Read, Write, Bash
model: sonnet
---

You write exactly one migration file for the ticket you're given, following
this repo's existing migration naming/format convention (read 2-3 existing
files in `migrations/` first to match style).

Do not modify application code. Do not run the migration against a real
database — only run the repo's local dry-run/lint command on it, if one
exists. If the ticket is ambiguous about column type, nullability, or
default value, stop and ask rather than guessing.
```

**Fast mechanical checker** — cheap model, no judgment calls:

```markdown
---
name: lint-runner
description: Runs the repo's lint and format checks on a given file list
and reports pass/fail with raw output. Use as a pre-commit or pre-PR
mechanical gate — not for style opinions beyond what the linter enforces.
tools: Bash, Read
model: haiku
---

Given a list of files (or "all staged files" via `git diff --cached
--name-only`), run this repo's lint command and format-check command on
them.

Report **PASS** or **FAIL** followed by raw command output. Do not
attempt to fix violations yourself — report back so a human or another
agent can decide the fix.
```

## Common pitfalls

- **Description too vague.** `description: Helps with testing.` never wins
  dispatch against a more specific agent, and invites use on tasks it
  wasn't scoped for. Name the concrete trigger.
- **Too many tools granted "to be safe."** Every extra tool is scope the
  prompt now has to defend against instead of the harness enforcing it.
  A reviewer with `Edit` access will eventually "helpfully" fix something
  instead of reporting it.
- **No stop/escalate condition.** Without one, an agent facing an
  incomplete or wrong plan will improvise past its scope rather than
  report back — the single most common way a "simple, scoped" agent turns
  into an uncontrolled refactor.
- **Vague file scope.** "Read the relevant files" makes a fresh-context
  agent explore the whole repo. Name exact paths or globs.
- **No output/verdict format for judgment agents.** If a caller (human or
  another agent) needs to branch on the result, an unstructured prose
  report is unreliable to parse — fix the vocabulary (`APPROVE`/`REVISE`,
  `PASS`/`FAIL`) up front.
