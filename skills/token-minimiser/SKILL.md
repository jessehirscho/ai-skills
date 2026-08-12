---
name: token-minimiser
description: Use whenever context/token usage needs to be minimized — long sessions approaching context limits, expensive multi-subagent workflows, large repos where naive full-file reads burn context, or the user explicitly asks to work more efficiently/cheaply. A practical checklist for reducing tokens spent per unit of useful work, across a whole session's lifecycle.
---

# Token Minimiser

Token cost isn't one lever, it's death by a thousand cuts: re-reading files, exploring blindly, rewriting whole files for one-line changes, spinning up subagents for things a grep would answer, and narrating instead of answering. This skill is a checklist of where that waste actually happens and the concrete habit that fixes each one.

## Reading files

- If you already know roughly where the relevant code is (from a prior grep, an error stack trace, a file you just edited), read that line range with `offset`/`limit` — don't read the whole file "to be safe."
- If you don't know where something is, don't Read-around to find out — `grep`/targeted search for the symbol or string first, then read only the hit's surrounding lines.
- Never re-read a file already in context this turn or earlier in the session. If you're unsure whether it changed, check with a cheap `git diff`/`git status` on that path instead of a full re-read.
- Large generated files (lockfiles, build output, minified bundles) are almost never worth reading — grep for the specific key/symbol instead.

## Editing

- Once a file exists, prefer `Edit` (targeted diff) over `Write` (full-file rewrite). A `Read`-then-rewrite-the-whole-file pattern costs the full file twice for a change that touches a handful of lines.
- Only use `Write` for genuinely new files or when the change touches the majority of the file's content.
- Batch multiple independent edits to the same file into as few tool calls as reasonable rather than round-tripping Read→Edit→Read→Edit.

## Search and exploration

- For open-ended "where is X handled" / "which files reference Y" questions, delegate to a purpose-built, read-only search agent if your harness has an equivalent to Claude Code's `Explore` subagent — instead of the main session manually grepping, following dead ends, and accumulating all of that noise in its own context. If your harness has no such delegate, do targeted greps yourself rather than reading whole files to explore.
- Give the search agent a specific breadth ("quick" vs "very thorough") matching how confident you are about where the answer lives — don't default to maximal search depth for a lookup you're 90% sure is in one directory.
- Manual exploration in the main thread is fine for a single targeted grep; delegate as soon as it turns into more than 2-3 speculative searches.

## Subagent delegation

- Subagent-heavy workflows run roughly 5-7x the token cost of a single-thread session (see `dev-pipeline` skill) — that's a real multiplier, not a rounding error. Delegate because the task is genuinely independent/parallelizable or needs fresh eyes, not by default and not just because a task "feels big."
- Pass subagents the exact file paths and scope they need in the prompt itself. A subagent's context starts empty — if it has to explore to find what you already know, you paid for the exploration twice (once implicitly when you scoped the task, once when the subagent re-discovers it).
- Cap iteration/critique loops explicitly (e.g. 2 rounds) rather than letting a revise-and-resubmit cycle run unbounded. If it's not converged after the cap, stop and escalate rather than looping again.
- Prefer one well-scoped subagent over several loosely-scoped ones covering overlapping ground.

## Context compaction awareness

- Conversation history gets auto-summarized as it grows — trust that summary. Don't manually re-paste or re-summarize facts, decisions, or file contents already established earlier in the same session "just in case."
- Don't re-derive a conclusion or re-run a lookup (a grep, a build, a test suite) a second time in one session unless something material changed since the first run.
- Don't re-litigate a decision that was already made and acted on earlier in the session without new information forcing it.

## Prompt caching

- Batch related tool calls together and avoid unnecessary delay between them — gaps between calls in the same logical step risk cache expiry, turning a cheap cached prefix back into a full-price read.
- If a session involves scheduled wake-ups or polling a background task, prefer a small number of longer idle gaps over frequent short polling intervals — each wake is a fresh prompt-cache risk, and polling loops multiply that risk for no benefit over one well-timed check.

## Output discipline

- Match response length to the question. A yes/no or single-fact question gets a direct answer, not headers, bullet lists, or a "Summary" section.
- Don't narrate internal deliberation ("Let me think about this... First I'll check... Now I'll..."). Do the check, then state the result.
- Don't restate context the user already has (their own request, files they just showed you, decisions already made this session).

## Prioritized checklist — adopt in this order

1. **Stop Read-then-Write-whole-file.** Use `Edit` for existing files. This is usually the single biggest per-change waste in a session.
2. **Delegate search, not everything.** Use `Explore`/a search agent for open-ended lookups; keep single targeted greps in the main thread.
3. **Scope subagents tightly.** Exact file paths and task in the prompt, no "go explore and figure it out" unless that's genuinely the task.
4. **Cap loops.** 2-round limit on any critique/revise/retry cycle; escalate past that instead of grinding.
5. **Trust compaction.** Don't re-paste or re-derive what's already established in the session.
6. **Answer at the right altitude.** Short question, short answer — no headers or narration for simple asks.
