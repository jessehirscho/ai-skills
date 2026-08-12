---
name: changelog-writer
description: Use whenever the user wants to turn a range of commits or merged PRs into a user-facing changelog entry. Distinct from a commit message (for other developers, describes the diff) and from a session-handoff doc (internal, for whoever continues the work next) — a changelog entry is for end users or external stakeholders and describes impact/behavior change, not implementation.
---

# Changelog Writer

Translate a range of commits/PRs into an entry a user or stakeholder would actually want to read. The audience never sees the code — they see the product. Every line should answer "what changes for me," not "what did the engineer do."

## The core skill: translating implementation → impact

A commit message or PR title describes the diff. A changelog line describes what a user notices. Translate, don't copy.

| Implementation-focused (commit/PR title) | User-facing (changelog line) |
|---|---|
| `refactor auth middleware` | (usually nothing — no behavior change, leave it out) |
| `fix null check in booking form submit handler` | Fixed an issue where the booking form could fail to submit with certain phone number formats |
| `add retry logic to Resend email call` | Fixed an issue where booking confirmation emails could occasionally fail to send |
| `bump react-router 6→7, migrate routes` | (leave out unless it changed visible behavior, e.g. broken deep links) |
| `add FAQPage JSON-LD to suburb pages` | Improved how suburb pages appear in Google search results |
| `debounce suburb search input` | Improved responsiveness of the suburb search |
| `merge PR #64: consolidate duplicate CTA components` | (internal cleanup — leave out) |

If you can't state the user-visible effect in one plain sentence, it probably doesn't belong in the changelog — either it's genuinely internal, or you need to go look at what actually changed on-screen/in-behavior before writing the line.

## Categories (Keep a Changelog)

Follow the [Keep a Changelog](https://keepachangelog.com/) convention — it's a widely-used standard, not a house invention. Use only the categories that have entries; don't print empty headers.

- **Added** — new user-facing capability that didn't exist before
- **Changed** — existing behavior that works differently now
- **Fixed** — a bug the user could have noticed, now corrected
- **Removed** — a feature or capability taken away
- **Security** — a vulnerability closed (even without a CVE, flag security-relevant fixes here, not under Fixed)
- **Deprecated** — still works today, but scheduled for removal; tell users what to migrate to

**Discipline: not every commit gets a line.** A changelog is curated, not a mirror of `git log`. If ten commits fixed one user-perceptible bug (e.g. flaky email delivery), that's one Fixed line, not ten. If a commit has zero user-visible effect, it doesn't appear at all — no "misc internal improvements" catch-all line either; if it's not visible, it's not in the changelog.

## Gathering source material without re-reading every diff

Work from titles/summaries first; only open a diff when the title is ambiguous about user impact.

```bash
# Commits since the last release tag
git log <last-tag>..HEAD --oneline

# Or since a date, if there's no tag scheme
git log --since="2026-07-01" --oneline

# Merged PRs in a date range (titles + bodies carry more intent than raw commits)
gh pr list --state merged --search "merged:>=2026-07-01" --json number,title,body,mergedAt

# One PR's summary, if the title alone isn't enough to judge user impact
gh pr view <number> --json title,body
```

Filter before drafting, not after:
- Drop pure dependency bumps with no behavior change (`bump vite 5→6`).
- Drop refactors/renames that don't change what a user sees or can do.
- Drop test-only and CI/tooling changes (`add e2e test`, `fix lint config`, `speed up build`).
- Drop anything that's really a handoff-doc concern (internal state, what to re-check next session) — that belongs in the project's `docs/handoff-*.md`, per `workflow-conventions`, not in a user-facing changelog.
- Keep anything a user could plausibly notice or would want to know changed, even if the underlying fix was small.

## Tone and voice

- Plain language, no jargon a non-engineer would have to look up ("database" not "Supabase row", "search results" not "SERP").
- Active voice, past tense for what shipped: "Fixed", "Added", "Improved" — not "This PR fixes."
- One line per change where possible. Lead with the verb.
- Group small related fixes into a single line if a user would perceive them as one improvement (e.g. three separate mobile-layout patches → "Fixed several layout issues on mobile screens").
- No PR numbers, file names, or internal component names in the user-facing line. If traceability matters, put the PR/commit ref in parentheses at the end for internal readers only, and confirm that's wanted before including it.

## Versioning tie-in (brief)

If the project uses semver, the categories map naturally to bump decisions:
- **Added** → minor bump
- **Fixed** / **Security** → patch bump
- **Removed** or any breaking change → major bump (call out breaking changes explicitly, regardless of category)
- **Changed** → minor or major depending on whether it breaks existing usage

This project (soar-solutions) is a marketing site with no package versioning scheme, so this section is informational only — skip version numbers here unless the user asks for them.

## Minimal entry template

```markdown
## [Unreleased]

### Added
- Short description of the new capability, from the user's point of view.

### Changed
- What now behaves differently, and what a user should expect.

### Fixed
- The problem a user could have hit, stated plainly, now resolved.

### Removed
- What's no longer available, and where to go instead if relevant.
```

Only include the categories that have content that week/release — an empty `### Removed` header is noise.

## What to leave out entirely

- Internal refactors, renames, restructuring with no behavior change.
- Dependency/toolchain bumps that don't change what ships.
- CI, lint, test, or build-tooling changes.
- Anything that belongs in a session-handoff doc instead — open questions, what to re-check, next steps for whoever continues the work. That's written for the next developer/AI, not the end user; see `workflow-conventions` for that format.
- Anything you can't state as a one-sentence, plain-language user impact. If you can't translate it, don't include it — go look at the diff first, or drop it.
