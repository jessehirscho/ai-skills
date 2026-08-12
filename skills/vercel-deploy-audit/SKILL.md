---
name: vercel-deploy-audit
description: Fast pre-deploy audit checklist to run before promoting a Vercel deployment to production. Use whenever the user is about to deploy/promote to prod and wants a sanity check that catches "worked in preview, broke in prod" failures — env var drift, silent routing changes, accidental Edge runtime, and missing rollback plan. Not a deployment-mechanics guide.
---

# Vercel Pre-Deploy Audit

This is a **checklist/audit skill**, not a deployment mechanics guide. For the
how-to on any individual step (linking projects, `vercel deploy` flags, env
var CRUD, CI config), use the built-in `vercel:deployments-cicd` and
`vercel:env-vars` skills instead. This skill only tells you *what to check,
in what order,* right before promoting to production.

## When to use this

Right before running `vercel --prod` / `vercel promote`, or right before
merging a PR that will trigger a production deploy.

## 5-minute pre-promote checklist

Run through this literally, top to bottom, before hitting promote:

- [ ] `vercel env ls production` and `vercel env ls preview` diffed — no var
      present in preview but missing/stale in production
- [ ] `vercel build` (or the project's real build command) succeeds locally
      on the exact commit being promoted — not "it built last time"
- [ ] Opened the actual preview URL for this commit and clicked through the
      pages/flows that changed — not a cached memory of an older preview
- [ ] Diffed `vercel.json` (and any `next.config.js` / middleware) against
      the last production commit — rewrites/redirects/headers unchanged or
      changes are understood
- [ ] Any new route added this change is confirmed reachable on the preview
      URL (not just present in local dev)
- [ ] Grepped changed files for `runtime = 'edge'` / `export const runtime`
      — none added unintentionally
- [ ] Function region/timeout config in `vercel.json` unchanged or the
      change is intentional
- [ ] Current production deployment ID/URL noted down *before* promoting, so
      a rollback target is already in hand

If all boxes are checked, promote. If any box fails, stop and fix before
promoting — don't promote "to see."

## 1. Env var parity: preview vs. production

The single most common "worked in preview, broke in prod" cause is a var
that exists in one environment and not the other.

```bash
# List each environment separately (there is no built-in `env diff` command)
vercel env ls production
vercel env ls preview

# Pull each to a local file and diff the variable NAMES (not values —
# avoid printing secrets to your terminal history / a shared file)
vercel env pull .env.production.local --environment=production
vercel env pull .env.preview.local --environment=preview
diff <(cut -d= -f1 .env.production.local | sort) \
     <(cut -d= -f1 .env.preview.local | sort)
```

Anything only on one side of that diff is a candidate bug. Delete the pulled
files afterward — don't leave production secrets sitting in the repo dir.

## 2. Build output sanity

Confirm the production build actually succeeds locally, on the commit about
to ship — don't trust a green check from a build run days ago.

```bash
git status              # confirm you're on the exact commit going to prod
vercel build            # runs the platform build locally, same as prod
# or, if you don't need Vercel's build pipeline specifically:
npm run build
```

Look at the output for:
- New warnings that weren't there before (unused deps, large chunks)
- A sudden jump in bundle/output size vs. the last known-good build
- Any step that silently skipped (cached step reused when it shouldn't be)

## 3. Preview-vs-prod diffing

Never assume "it worked last time" covers the commit you're about to
promote. Get the actual preview URL for *this* commit and manually verify:

```bash
vercel ls                      # find the preview deployment for this commit
vercel inspect <deployment-url>  # confirm commit SHA / branch matches
```

Open that exact preview URL and walk through whatever changed — new routes,
edited forms, altered layout — before promoting it.

## 4. Domain / routing sanity

Silent `vercel.json` changes (rewrites, redirects, headers) are easy to miss
in a diff full of unrelated code changes.

```bash
git diff <last-prod-commit>..HEAD -- vercel.json
```

For each changed rule, confirm on the preview URL:
- Old routes still resolve where expected (no accidental catch-all shadowing)
- New routes are reachable, not just defined
- Headers (especially CSP, cache-control) didn't change unintentionally

## 5. Rollback readiness

Know your rollback target *before* you need it, not while a prod incident is
live.

```bash
# Note the current production deployment before promoting
vercel ls --prod       # or: vercel inspect <current-prod-url>
```

If the new deploy goes bad:

```bash
vercel rollback [deployment-id-or-url]   # roll back to a previous deployment
vercel rollback status                   # check rollback progress
vercel rollback status --timeout 30s     # with a custom poll timeout

# To undo a rollback (re-promote):
vercel promote [deployment-id-or-url]
```

Note: on the Hobby plan you can only roll back to the immediately previous
production deployment — passing an older deployment ID errors out. Pro plans
can target any prior deployment.

## 6. Function / runtime sanity

Edge runtime is generally not the recommended default anymore (Fluid
Compute / Node.js is preferred for most workloads) — check nothing picked it
up by accident, e.g. from a copy-pasted example or a framework default.

```bash
# Search changed files for edge runtime declarations
git diff <last-prod-commit>..HEAD | grep -n "runtime.*edge"
grep -rn "runtime = ['\"]edge['\"]" api/ app/ src/ 2>/dev/null
```

Also re-check, in `vercel.json` or per-function config, that region and
`maxDuration`/timeout settings match what you expect — these silently affect
cost and cold-start behavior and are easy to leave stale after copy-pasting
config.

## Related skills

- `vercel:deployments-cicd` — deployment mechanics, promote/rollback details, CI config
- `vercel:env-vars` — env var CRUD, `.env` file handling, OIDC tokens
- `vercel:vercel-functions` — runtime/region/timeout configuration in depth
- `vercel:cdn-caching` — cache-header and revalidation debugging
