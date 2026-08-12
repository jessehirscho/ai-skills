---
name: incident-writeup
description: Use whenever a production incident, outage, data issue, or significant bug needs a structured postmortem writeup — a blameless internal document covering what broke, why, how it was found and fixed, and what prevents recurrence. Distinct from a session-handoff doc (continuing in-progress work across sessions) and distinct from a changelog entry (brief, user-facing). Use for customer-facing outages, data loss/corruption, near-misses that would have been serious, and security incidents.
---

# Incident Writeup

A structured, blameless postmortem for something that broke in production. This is not a session-handoff doc (which hands off in-progress work to the next session) and not a changelog entry (which briefly tells users what changed). An incident writeup exists to make recurrence less likely, by understanding the system and process failure — not to assign fault.

## Blameless framing — first-class, not an afterthought

The writeup investigates the systems and processes that allowed the failure, never the individual who "caused" it. This isn't a courtesy — it's load-bearing for the document's actual purpose:

- Blame culture suppresses exactly the information that prevents recurrence. If naming the person who fat-fingered a config or shipped the bug results in consequences for them, people stop reporting near-misses, stop being candid about what they didn't know, and stop flagging the moment they noticed something looked wrong but didn't say anything. Those are the details that catch the *next* incident before it happens.
- Write actions, not actors: "the deploy script ran without a confirmation prompt" not "X deployed without checking." "The alert threshold was set too high to catch this class of failure" not "nobody was watching."
- Root cause analysis should terminate at a systemic factor (missing safeguard, absent test, unclear ownership, no alerting) — never at "a person made a mistake." People will always make mistakes; the question is what the system did to make that mistake catastrophic instead of caught.

## When to write one

Write a full incident writeup for:
- Any customer-facing outage or degradation, regardless of duration
- Data loss or data corruption, even if fully recovered
- A near-miss that would have been serious if not caught (these are often more valuable than actual incidents — the safeguard worked once, but why did it almost not work?)
- A security incident (unauthorized access, credential exposure, vulnerability exploited)

Skip it for a routine bug fix with no production/user impact — that's a normal commit message or changelog entry, not a postmortem. If unsure, the deciding question is: "did this affect a real user, real data, or real money — or could it have?" If yes, write it up.

## Template

```markdown
# Incident: <short descriptive title>

**One-line summary:** <what broke, in plain language, one sentence>

## Severity / Impact

- **Who/what was affected:** <users, systems, data — be specific>
- **Duration:** <start → end, with timezone>
- **Quantified impact:** <numbers, not vibes — e.g. "approximately 12% of
  booking form submissions failed for 40 minutes" not "some users were
  affected">
- **Detection method:** <alert, user report, manual discovery, etc.>

## Timeline

All times in <timezone>, pulled from logs/monitoring/deploy history —
not reconstructed from memory. Note any gaps or uncertainty explicitly
rather than smoothing them over.

- `HH:MM` — <event: deploy, config change, first error, etc.>
- `HH:MM` — <detection: alert fired / user reported / engineer noticed>
- `HH:MM` — <diagnosis milestone>
- `HH:MM` — <mitigation applied, e.g. rollback, feature flag off>
- `HH:MM` — <confirmed resolved>

If the gap between "problem started" and "problem detected" is large,
call that out as its own finding — it's often the highest-leverage thing
to fix, independent of what caused the original break.

## Root Cause

The actual mechanism — not "human error." Push past the first proximate
cause to the systemic one underneath it. The "5 whys" is a useful forcing
function here:

1. Why did the outage happen? <e.g. the API returned 500s>
2. Why? <e.g. a null value hit an unguarded field access>
3. Why? <e.g. an upstream schema change wasn't validated at the boundary>
4. Why? <e.g. there's no contract test between these two services>
5. Why? <e.g. the two services are owned by different teams with no
   shared review gate on schema changes>

Stop when you hit something a process or system change can actually fix.
"Human error" is never a root cause — it's where a shallow analysis
stops early.

## What Went Well

Credit what limited the damage or sped up recovery, not just what
failed. E.g. "the feature flag allowed instant rollback without a
redeploy," "the alert fired within 2 minutes of the error rate spike."

## What Went Poorly

The gaps, honestly stated, in system/process terms. E.g. "there was no
staging environment that would have caught this," "the runbook for this
alert was out of date."

## Action Items

Specific, owned, and prioritized — and explicitly distinguish two kinds,
since conflating them produces a vague list nobody executes:

- **Prevents this exact recurrence** (do first): <e.g. add null check +
  regression test for the specific field> — owner: <name>, tracked in
  <ticket link>
- **Improves general resilience** (do after, lower urgency): <e.g. add
  contract testing between these two services> — owner: <name>, tracked
  in <ticket link>

Every action item needs a real owner and a linked ticket. An action item
with no ticket doesn't happen.
```

## Distribution and follow-through

- Store the writeup in a durable, searchable location the whole team can find later — a docs folder, wiki, or issue tracker, not buried in a chat thread or a single person's notes. If the project already has a `docs/` convention, use it (e.g. `docs/incidents/<date>-<slug>.md`); don't invent a new location if one exists.
- Link the writeup from wherever the team tracks incidents generally, if such a place exists.
- Action items must become real tracked tickets (linked from the writeup), not just prose in the doc. A postmortem whose action items are never tracked is a one-time ritual with no lasting effect — the whole point is that the next person hits a system that's better than the one that failed, not just a document that says it should be.
- Revisit open action items periodically; an incident writeup with three "TODO, unowned" items six months later is a sign the process isn't working, not a document to quietly retire.
