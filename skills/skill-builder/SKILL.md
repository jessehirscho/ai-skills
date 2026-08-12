---
name: skill-builder
description: Use whenever the user wants to create a new Claude Code skill (a SKILL.md file, optionally with supporting files/scripts), edit an existing one in this repo, or plan/build a batch of new skills. This is the meta-skill that governs how every skill in this repo gets authored — triggers on "make a skill for X", "add a skill", "write a SKILL.md", "turn this into a skill", or requests to review/fix an existing skill's quality.
---

# Skill Builder

Meta-skill for authoring Claude Code skills in this repo. A skill is a `SKILL.md` file that Claude Code discovers at session start and decides to invoke by matching the current task against its `description` field — that's the entire dispatch mechanism, so getting the description right matters more than anything else in the file.

## How dispatch actually works

Claude Code reads the YAML frontmatter (`name` + `description`) of every skill up front, but not the body. When a task comes in, Claude decides whether to invoke a skill purely by whether the description sounds relevant. That means:

- A vague description like `"Helps with PDFs"` will under-fire (Claude won't reliably connect "merge these two files" to it) and over-fire unpredictably.
- A good description lists the concrete triggering actions and nouns, e.g. the `pdf` skill's actual description: *"Use this skill whenever the user wants to do anything with PDF files. This includes reading or extracting text/tables from PDFs, combining or merging multiple PDFs into one, splitting PDFs apart, rotating pages, adding watermarks, creating new PDFs, filling PDF forms, encrypting/decrypting PDFs, extracting images, and OCR on scanned PDFs to make them searchable. If the user mentions a .pdf file or asks to produce one, use this skill."*
- Include both the "use when" triggers (verbs, file types, phrases the user might actually say) and, if there's likely confusion with something else, an explicit "not for X" boundary.

The body is read only after dispatch, so it doesn't need to sell the skill — it needs to be usable reference material once Claude is already committed to using it.

## File layout conventions

```
skills/<name>/SKILL.md          # entry point — always this exact path
skills/<name>/REFERENCE.md      # optional: deep detail moved out to save context
skills/<name>/FORMS.md          # optional: specialized sub-topic doc (pdf skill's pattern)
skills/<name>/agents/*.md       # optional: subagent definitions the skill ships (dev-pipeline's pattern)
skills/<name>/scripts/*         # optional: runnable helper scripts, templates
```

Rules:
- `SKILL.md` is the only file Claude Code loads automatically. Everything else is loaded on demand, so **link to it explicitly** from SKILL.md (e.g. "If you need to fill a PDF form, read FORMS.md and follow its instructions") — don't assume Claude will discover a sibling file unprompted.
- Keep SKILL.md itself under ~300 lines. If a sub-topic needs more room than that would allow while keeping the main file scannable, split it into `REFERENCE.md` or a named sub-doc rather than letting SKILL.md sprawl.
- Only ship `agents/*.md` if the skill's procedure genuinely requires distinct subagent roles with different tool grants (like dev-pipeline's planner/critic/builder/tester/reviewer) — don't add subagents to a skill that's really just a single-pass procedure.

## Content quality bar

- **Concrete over prose.** Runnable code blocks, exact commands, real file paths — not paragraphs describing what the code would do.
- **One clear job per skill.** If a draft is covering three unrelated things ("PDFs, and also image conversion, and also OCR pipelines for scanned mail"), split it into separate skills rather than letting one sprawl. A skill that tries to cover everything dispatches on nothing precisely.
- **A quick-reference table near the end.** Every example skill in this repo ends with one — task/tool/command triples the reader can scan without re-reading prose.
- **Cover the 80% case first**, with a pointer to deeper material (REFERENCE.md, FORMS.md) for the rest, rather than trying to be exhaustive inline.

## Authoring procedure

1. **Write the description first, and stress-test it before writing anything else.** Draft 3-4 example user requests: 2-3 that should dispatch this skill, 1-2 adjacent requests that should NOT. Read the description cold and ask: would Claude Code correctly fire on the positive examples and correctly stay silent on the negative ones? If not, the description is the bug — fix it before touching the body. (Example negative case for a hypothetical `docx` skill: "summarize this document" should probably not dispatch a docx-editing skill.)
2. **Decide file layout.** Single file for a self-contained procedure under ~300 lines; `+ REFERENCE.md` if there's a large reference section (API tables, exhaustive option lists) that would drown the common-case instructions; `+ agents/*.md` only if the procedure needs genuinely different subagent roles.
3. **Draft the body focused on the 80% common case**, with real, concrete examples — actual commands, actual code, actual file paths from a plausible run, not placeholders like `<your command here>`.
4. **Add the quick-reference table.**
5. **Self-review against the checklist below before calling it done.**

## Pre-publish self-review checklist

- [ ] Description names specific triggering actions/nouns, not a vague summary — and was stress-tested against positive and negative example requests (step 1 above).
- [ ] No placeholder or TODO content anywhere in the body.
- [ ] Every code/command example is actually correct and runnable, not approximated from memory.
- [ ] Doesn't duplicate an existing skill in this repo, or a Claude Code built-in — check against built-ins like `artifact-design`, `dataviz`, `frontend-design` before assuming a gap exists.
- [ ] SKILL.md is under ~300 lines, or the overflow has been split into `REFERENCE.md`/a named sub-doc and linked from the main file.
- [ ] Frontmatter is valid YAML with exactly `name` and `description` — no extra fields, unless a specific harness requires one.

## Building a batch of new skills

This repo bootstraps itself using its own `workflow-conventions` and `dev-pipeline` skills — use that same pattern when authoring several new skills at once rather than writing them serially in one session:

1. One feature branch for the whole batch (`workflow-conventions`: never commit to `main`).
2. One subagent per skill, launched in parallel — each is independent, so there's no reason to serialize them (`dev-pipeline` Phase 1/4 pattern: give each subagent only what it needs, never the whole repo).
3. Each subagent's prompt bundles: this skill-builder skill's guidance (either by reference if the subagent can read this file, or inlined if not) plus that specific skill's brief — the topic, the intended triggers, and any known supporting files it should produce.
4. After the subagents return, run a human or critic pass over every new `SKILL.md` against the self-review checklist above before merging — treat this the same as `dev-pipeline`'s Phase 2 critique gate, don't let a subagent's own self-assessment be the only check.
5. Open the PR with a test plan that's just "read each SKILL.md, confirm dispatch description passes the stress test, confirm no duplication with existing skills" per `workflow-conventions`' required PR format.

## Quick reference

| Task | Do this |
|------|---------|
| New single-topic skill | `skills/<name>/SKILL.md`, description-first, under 300 lines |
| Skill needs deep reference material | Add `skills/<name>/REFERENCE.md`, link to it from SKILL.md |
| Skill needs a specialized sub-flow | Add a named doc like `FORMS.md`, link to it conditionally ("if X, read Y.md") |
| Skill needs distinct subagent roles | Add `skills/<name>/agents/*.md`, document one-time setup steps in SKILL.md |
| Building 3+ skills at once | Branch per batch, one parallel subagent per skill, critic pass before merge |
| Unsure if description will dispatch correctly | Write 3-4 example requests (positive + negative) and check by hand before drafting the body |
| Before merging any new skill | Run the pre-publish self-review checklist above |
