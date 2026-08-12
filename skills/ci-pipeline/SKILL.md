---
name: ci-pipeline
description: Use whenever the user wants to set up, modify, or debug a CI pipeline — primarily GitHub Actions plus `gh` CLI, but generalizes to other CI providers and Vercel's own deploy checks. Covers scaffolding workflows, lint/test/build/deploy jobs, caching, matrix builds, secrets, branch protection, and debugging failing runs.
---

# CI Pipeline

Practical reference for setting up and debugging CI. Defaults to GitHub Actions
since it pairs directly with the `gh` CLI, but the patterns (cache deps, gate
deploy on tests, fail fast on matrix) generalize to any CI provider.

## Quick reference

| Task | Where |
|---|---|
| New workflow from scratch | Minimal workflow |
| Speed up repeat runs | Caching dependencies |
| Test multiple Node/OS versions | Matrix builds |
| Use API keys / tokens in CI | Secrets handling |
| Require CI to pass before merge | Branch protection |
| Auto-deploy only from main | Deploy job pattern |
| Pipeline is red, don't know why | Debugging a failing pipeline |
| Project deploys via Vercel | Vercel note |

## Minimal workflow

`.github/workflows/ci.yml` — runs on every PR and on push to main. This uses
Node via `actions/setup-node`; for other languages swap the setup step
(`actions/setup-python`, `actions/setup-go`, etc.) and keep the same shape:
checkout → setup toolchain → install → lint → test → build.

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

Use `npm ci`, not `npm install`, in CI — it installs exactly what's in the
lockfile and fails if the lockfile is out of sync, instead of silently
rewriting it.

## Caching dependencies

`actions/setup-node`'s `cache: npm` input (above) is the simplest correct
option — it hashes `package-lock.json` and caches `~/.npm` for you. Reach for
`actions/cache` directly when you need to cache something setup-node doesn't
cover (build output, a Python venv, Playwright browsers):

```yaml
- uses: actions/cache@v4
  with:
    path: |
      node_modules/.cache
      ~/.cache/ms-playwright
    key: ${{ runner.os }}-tools-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-tools-
```

`restore-keys` lets a partial/stale cache hit be reused as a base even when
the exact key misses (e.g. lockfile changed slightly) — faster than a cold
install.

## Matrix builds

Run the same job across multiple Node versions and OSes:

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
        node: [18, 20, 22]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm
      - run: npm ci
      - run: npm test
```

`fail-fast: false` keeps every matrix leg running even if one fails — useful
while diagnosing which OS/version combo actually broke, instead of GitHub
cancelling the rest at the first red leg.

## Secrets handling

- Reference secrets only inside `${{ secrets.NAME }}`, never print them:
  `run: echo ${{ secrets.API_KEY }}` will leak it into the log — pass it via
  `env:` instead and let the tool read the env var.
- Repo secrets (Settings → Secrets and variables → Actions) are available to
  every workflow in the repo. Environment secrets (Settings → Environments →
  `production`, etc.) are scoped to jobs that declare
  `environment: production`, and can require manual approval before the job
  runs — use these for deploy credentials, repo secrets for things like a
  shared lint API token.

```yaml
jobs:
  deploy:
    environment: production
    steps:
      - run: ./deploy.sh
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

GitHub also auto-redacts any exact secret value it sees in logs, but that
only works for literal matches — don't rely on it if you base64-encode or
otherwise transform the secret before printing.

## Branch protection

Requiring CI to pass before merge is a **GitHub repo setting**, not something
you configure in the workflow YAML:

Settings → Branches → branch protection rule for `main` → "Require status
checks to pass before merging" → select the job name(s) from your workflow
(e.g. `ci`). Also enable "Require branches to be up to date before merging"
if you want PRs re-tested against the latest main before merge.

Via `gh`:
```bash
gh api repos/{owner}/{repo}/branches/main/protection \
  --method PUT \
  -f required_status_checks[strict]=true \
  -f 'required_status_checks[contexts][]=ci'
```

## Deploy job pattern

Gate deploy on the earlier jobs succeeding, and only run it on `main`:

```yaml
jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build

  deploy:
    needs: ci
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

`needs: ci` blocks `deploy` from starting until `ci` succeeds. The `if` guard
stops it running on PRs or on pushes to other branches — without it, `needs`
alone would still let deploy fire on every push to main, but also on manual
reruns of PR branches if you're not careful, so keep the ref check explicit.

## Debugging a failing pipeline

```bash
# List recent runs
gh run list --limit 10

# View a run, jump straight to the failed step's logs
gh run view <run-id> --log-failed

# Re-run only the jobs that failed (not the whole matrix)
gh run rerun <run-id> --failed

# Watch a run live
gh run watch <run-id>
```

To get verbose step-by-step debug output, add a repo secret named
`ACTIONS_STEP_DEBUG` set to `true` (and `ACTIONS_RUNNER_DEBUG` for
runner-level diagnostics). Remove it afterward — it's noisy and shouldn't
stay on permanently.

**Common "works locally but fails in CI" causes:**

- **Node/toolchain version mismatch** — pin `node-version` in the workflow to
  match `.nvmrc`/`engines` in `package.json` instead of trusting the runner
  default.
- **Env vars only set locally** — check `.env` isn't gitignored-and-assumed;
  CI has none of it unless you add it via `env:` or secrets.
- **Case-sensitive filesystem** — GitHub's `ubuntu-latest` runners are
  case-sensitive; an import like `./Utils` that resolves on a Mac's default
  case-insensitive filesystem will 404 in CI. Fix the import casing, don't
  work around it.
- **Missing system deps / browsers** — e.g. Playwright/Puppeteer need
  `npx playwright install --with-deps` as an explicit CI step; a locally
  cached browser binary isn't there on a fresh runner.
- **Non-deterministic lockfile drift** — `npm install` locally can silently
  update the lockfile; CI's `npm ci` then fails on a mismatch. Commit the
  lockfile change or fix the version that drifted.
- **Timezone/locale differences** — runners default to UTC; date-formatting
  tests that assume a local timezone can fail intermittently.

## Vercel note

If this project deploys via Vercel, Vercel's own preview/production deploy
checks already run automatically on every push — you generally don't need a
separate GitHub Actions `deploy` job unless you want custom pre-deploy steps
(e.g. running a script before Vercel builds, or gating deploy on an external
check). For Vercel-specific CI/CD detail (`vercel deploy`, `--prebuilt`,
promoting/rolling back), use the Vercel deployment skills already available
in this environment rather than duplicating that here.
