# ai-skills

Personal coding-agent skills, built from patterns actually used across my own projects rather than generic templates.

Where the guidance is genuinely portable, these are written to be useful in any coding-agent harness — Claude Code, Codex CLI, GitHub Copilot CLI, Pi, opencode. Where something is tied to one harness's mechanics, that's stated in the skill rather than papered over. See [Usage](#usage) for what "portable" does and doesn't mean in practice.

## Skills

### Workflow & process

- **[dev-pipeline](skills/dev-pipeline/SKILL.md)** — a strict Plan → Critique → Iterate → Build → Test → Review → Merge subagent pipeline for shipping several independent tickets in one repo. Includes ready-to-copy subagent definitions (`planner`, `critic`, `builder`, `tester`, `reviewer`) in `skills/dev-pipeline/agents/`, plus a sequential-session fallback for harnesses with no parallel-subagent primitive.
- **[content-spec-writer](skills/content-spec-writer/SKILL.md)** — writing well-scoped tickets/specs *before* they enter `dev-pipeline`: template, sizing guidance, and file-scope declarations that feed the pipeline's overlap-flagging.
- **[workflow-conventions](skills/workflow-conventions/SKILL.md)** — baseline git/PR/handoff-doc discipline for any repo: no direct commits or merges to `main`, feature-branch-only, a required PR test-plan checklist, and a session-handoff doc before wrapping up substantial work.
- **[pr-workflow](skills/pr-workflow/SKILL.md)** — `gh` CLI mechanics for the PR lifecycle: creating PRs, reading and replying to review comments, checking CI status, handling merge conflicts. Companion to `workflow-conventions` (policy/format, not commands).
- **[ci-pipeline](skills/ci-pipeline/SKILL.md)** — setting up, modifying, and debugging CI pipelines (primarily GitHub Actions): workflow scaffolding, caching, matrix builds, secrets, deploy gating, triaging failed runs.
- **[changelog-writer](skills/changelog-writer/SKILL.md)** — turning a commit/PR range into a user-facing changelog entry: implementation-to-impact translation, Keep a Changelog categories, what to leave out.
- **[incident-writeup](skills/incident-writeup/SKILL.md)** — a blameless postmortem template: quantified impact, real timeline, root cause past the first proximate cause, prioritized action items.

### Agents, skills & prompts

- **[skill-builder](skills/skill-builder/SKILL.md)** — the meta-skill for creating new skills: description-first authoring, file layout conventions, content quality bar, pre-publish checklist.
- **[agent-builder](skills/agent-builder/SKILL.md)** — authoring `.claude/agents/*.md` subagent definitions: frontmatter format, least-privilege tool scoping, model selection, writing a scoped system prompt.
- **[subagents](skills/subagents/SKILL.md)** — judgment for delegating mid-session: when it's actually worth the cold start, how to write a dispatch prompt, foreground vs. background, parallel dispatch, avoiding duplicated work. Applies to any harness with a delegation primitive.
- **[agent-handover](skills/agent-handover/SKILL.md)** — splitting one task across different LLM vendors or product lines, with a written handover artifact at each boundary instead of a shared conversation. Distinct from `subagents`/`dev-pipeline`, which cover delegation within one vendor's lineup.
- **[prompt-eval](skills/prompt-eval/SKILL.md)** — lightweight verification for a system prompt, subagent definition, or skill description before shipping it: dispatch evals, behavior evals, before/after regression checks. Companion to `agent-builder`/`skill-builder`.
- **[plugin-creator](skills/plugin-creator/SKILL.md)** — packaging skills/agents/commands/MCP servers into a distributable Claude Code plugin with a marketplace manifest, vs. the plain-folder approach this repo itself uses.
- **[token-minimiser](skills/token-minimiser/SKILL.md)** — reducing tokens burned per unit of useful work: targeted reads/edits, scoped delegation, avoiding redundant re-derivation within a session.
- **[workflow-audit](skills/workflow-audit/SKILL.md)** — mining local session transcripts for real usage patterns (tool/skill/subagent frequency, repeated friction) and turning them into concrete fixes. **Claude Code only** — the technique depends on its specific local transcript format.

### MCP

- **[mcp-builder](skills/mcp-builder/SKILL.md)** — scaffolding a new MCP server (TypeScript or Python), defining tools with proper schemas, choosing a transport, and registering it with your harness.
- **[mcp-tools-setup](skills/mcp-tools-setup/SKILL.md)** — discovering, installing, scoping, and troubleshooting MCP servers that already exist (as opposed to `mcp-builder`, which builds a new one).

### Harness setup

Each of these is deliberately specific to one harness — they audit that tool's own config surface.

- **[claude-code-setup](skills/claude-code-setup/SKILL.md)** — `settings.json` precedence, CLAUDE.md quality, hooks, permissions, plugin/skill hygiene.
- **[codex-cli-setup](skills/codex-cli-setup/SKILL.md)** — `config.toml`, `AGENTS.md`, approval/sandbox modes, profiles.
- **[copilot-cli-setup](skills/copilot-cli-setup/SKILL.md)** — custom instructions files, config, and known-unverified areas flagged explicitly.
- **[opencode-setup](skills/opencode-setup/SKILL.md)** — config precedence, multi-provider/model config, MCP, keybinds.
- **[pi-harness-setup](skills/pi-harness-setup/SKILL.md)** — settings, `AGENTS.md`/`CLAUDE.md` concatenation, project trust model, extensions.

### Audits

- **[performance-audit](skills/performance-audit/SKILL.md)** — comparing a current implementation against a proposed one: baseline measurement, controlled comparison, results-reporting template.
- **[seo-audit](skills/seo-audit/SKILL.md)** — traditional search SEO: crawlability, on-page technical checks, structured data validation, Core Web Vitals as a ranking factor, local SEO.
- **[geo-audit](skills/geo-audit/SKILL.md)** — Generative Engine Optimization: content citability for AI answer engines (ChatGPT Search, AI Overviews, Perplexity) — crawler access, direct-answer-first structure, manual citation checks. Companion to `seo-audit`.
- **[vercel-deploy-audit](skills/vercel-deploy-audit/SKILL.md)** — a fast pre-promote checklist for Vercel deploys: env var parity, build sanity, preview-vs-prod diffing, rollback readiness.

### Content & documents

- **[data-insights](skills/data-insights/SKILL.md)** — turning messy real-world data into a defensible business insight: profiling, cleaning, question-first analysis, sanity-checking findings, presenting them in business terms.
- **[html-comparison-docs](skills/html-comparison-docs/SKILL.md)** — self-contained HTML documents that visually compare results (before/after, option A vs. B): side-by-side, slider, and tabbed layouts.
- **[pdf](skills/pdf/SKILL.md)** — reading, extracting, merging, splitting, creating, watermarking, encrypting, and OCR'ing PDF files.

## Usage

### Claude Code

Copy a skill's folder into `~/.claude/skills/<name>/` (personal, all projects) or `<repo>/.claude/skills/<name>/` (project-scoped). `SKILL.md` files are discovered there automatically and auto-invoked when a request matches the skill's `description`.

For `dev-pipeline`, also copy `skills/dev-pipeline/agents/*.md` into the target repo's `.claude/agents/` — that's how the five subagent roles get invoked via the `Agent` tool.

### Other harnesses

The `SKILL.md` format and its auto-dispatch behavior are a Claude Code convention. Pi supports the same Agent Skills format; Codex CLI, Copilot CLI, and opencode do not — so on those, **these files work as reference docs rather than auto-invoked skills**.

The practical approach there: point the agent at the relevant file from that harness's own instructions file (`AGENTS.md` for Codex and opencode, `.github/copilot-instructions.md` for Copilot), or paste the file's body in when you need it. The guidance inside is written to survive that, but the automatic "notice this applies and load it" step is what you give up.

Two specific caveats worth knowing before porting:

- **`workflow-audit` is Claude Code only.** It mines `~/.claude/projects/**/*.jsonl`; other harnesses don't persist sessions the same way, and many don't persist them at all.
- **`dev-pipeline` assumes parallel subagents.** Codex CLI and Copilot CLI don't expose an equivalent, so the skill documents running its phases as sequential sessions instead — same handoff-artifact discipline, less throughput.

## Conventions

Skills in this repo aim to be concrete over theoretical: real commands, runnable snippets, a quick-reference table, and one clearly-scoped job per skill. Where a skill depends on fast-moving third-party details, it says so inline rather than presenting them as settled. `skill-builder` documents the bar; `prompt-eval` covers checking a skill actually dispatches and behaves as intended before you rely on it.
