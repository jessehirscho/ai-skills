---
name: performance-audit
description: Use whenever comparing the performance of a current implementation against a new/proposed solution — web frontend (Core Web Vitals, bundle size), backend/API (latency, throughput), database (query time, N+1s), or algorithms (Big-O, wall-clock). Establishes a reproducible baseline, measures the candidate the same way, and reports before/after numbers instead of vibes.
---

# Performance Audit

A methodology for proving a change is actually faster, not just assuming it. Applies to any
domain: frontend, backend, database, or plain code.

## When NOT to bother

Skip the full procedure if:
- The change is trivial and touches no hot path (e.g. renaming a variable, fixing a typo).
- No user-facing or system-facing metric is plausibly affected.
- The "improvement" is purely stylistic/readability with no claimed perf benefit — say so and
  move on instead of manufacturing a benchmark.

If any perf claim is being made in a PR description or handoff doc ("this is faster", "this
reduces load time"), the claim needs numbers from this procedure, not intuition.

## Procedure

### 1. Define the metric(s) that matter

Pick metrics tied to what the user/system actually experiences — not whatever is easiest to
measure. Examples:
- Frontend page: LCP, INP, CLS, Time to Interactive, total transferred bytes, JS bundle size.
- API endpoint: p50/p95/p99 latency, requests/sec at fixed concurrency, error rate under load.
- Database query: wall-clock execution time, rows scanned, index usage.
- Algorithm/function: wall-clock time across input sizes, memory allocated, Big-O if it changes.

Write the metric down before measuring anything. Changing which metric you report mid-audit is
how comparisons get gamed (accidentally or not).

### 2. Baseline the CURRENT solution — before touching anything

Measure the existing code/system as it runs today, in the environment closest to production.
Do this **before** writing any candidate code, on a clean checkout.

- Run multiple trials (minimum 5, ideally 10+), discard the first 1-2 as warm-up.
- Report the median (not mean — outliers from GC pauses, disk cache misses, etc. skew the mean).
- Record the environment: hardware, network conditions, build mode (prod vs dev), concurrent
  load on the machine, dataset size/shape.

### 3. Implement/identify the candidate

Change exactly one variable. If you change the algorithm AND the data structure AND the caching
layer in one shot, you cannot attribute the delta to any one of them. If multiple changes are
bundled for practical reasons, say so explicitly in the report and note the comparison is not
isolating a single cause.

### 4. Measure the candidate identically

Same machine, same network conditions, same build mode, same dataset, same tool, same number of
trials, same warm-up discard policy as step 2. If anything about the measurement setup differs
between baseline and candidate, the comparison is invalid — fix the setup, don't rationalize the
difference away.

### 5. Report with real numbers

Use the reporting template below. Include trial counts and variance, not just a single before/after
pair. State the percentage change and whether it's likely to matter at the metric's scale (e.g.
"140ms -> 90ms LCP" matters; "140.0ms -> 139.7ms p50" is noise).

---

## Domain-specific measurement techniques

### Web frontend

**Lighthouse CLI** (Core Web Vitals, on a built/production bundle, not `npm run dev`):
```bash
npm run build && npm run preview &   # serve the production build
npx lighthouse http://localhost:4173/ \
  --output=json --output-path=./baseline.json \
  --only-categories=performance \
  --chrome-flags="--headless" \
  --throttling-method=simulate
# repeat 5x, take median LCP/TBT/CLS from the JSON reports
```
Run it 5 times per version (baseline, then candidate) — Lighthouse scores vary run to run even
with no code changes. Use `--throttling-method=simulate` consistently so runs are comparable.

**web-vitals in the field/real browser** (captures actual RUM-style numbers):
```js
import { onLCP, onINP, onCLS } from 'web-vitals';
onLCP(console.log);
onINP(console.log);
onCLS(console.log);
```

**Bundle size**:
```bash
npx vite-bundle-visualizer         # Vite projects — opens a treemap of dist/
# or for webpack:
npx webpack-bundle-analyzer stats.json
```
Compare `dist/` total size and the size of the specific chunk that changed:
```bash
du -sh dist/assets/*.js | sort -h
```

**Chrome DevTools via claude-in-chrome** (for interaction-level profiling): load the
`claude-in-chrome` skill, navigate to the page, use `read_network_requests` for waterfall/transfer
sizes and the Performance panel for main-thread work, before and after the change, same steps
each time.

### Backend / API

**hyperfine** (CLI/process-level timing, great for scripts or CLI tools):
```bash
hyperfine --warmup 3 --min-runs 20 'node old-script.js' 'node new-script.js'
```
Gives you mean, stddev, median, and a relative comparison automatically.

**autocannon** (HTTP load testing, Node-native):
```bash
npx autocannon -c 50 -d 30 -w 4 http://localhost:3000/api/endpoint
# -c: concurrent connections, -d: duration in seconds, -w: worker threads
```
Report p50/p97.5/p99 latency and req/sec from the output. Run baseline and candidate back to back
against the same load profile, ideally with nothing else contending for the machine's CPU.

**wrk** (alternative HTTP load tool, useful for higher throughput scenarios):
```bash
wrk -t4 -c100 -d30s --latency http://localhost:3000/api/endpoint
```

**Python timeit** (function-level microbenchmark):
```bash
python -m timeit -s "from mod import old_fn" "old_fn(input_data)"
python -m timeit -s "from mod import new_fn" "new_fn(input_data)"
```

**Node --prof** (find where time actually goes before assuming a fix will help):
```bash
node --prof server.js
# after load, process the log:
node --prof-process isolate-*.log > profile.txt
```

### Database

**EXPLAIN ANALYZE** (Postgres/MySQL — shows real execution time and plan, not just estimate):
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```
Look for: sequential scans on large tables where an index scan is expected, actual row counts vs
estimated (large mismatches mean stale stats), nested loop joins over large row counts (often an
N+1 symptom surfacing in a single query plan).

**Raw query timing** (wrap in the app's own query layer so overhead matches production):
```sql
\timing on   -- psql
SELECT ...;
```
Run 5-10x, discard first run (cold cache), report median. Watch for N+1s by counting actual
queries issued per request (log query count, not just total time) — a "fast" per-query time can
still be an N+1 problem if 200 queries fire per page load.

### General code / algorithms

- Never compare a cold run of one version to a warm run of another — JIT warm-up (V8, JVM),
  disk cache, and OS page cache all bias early runs slower.
- Report median of >=5 trials, not a single run.
- For complexity claims ("O(n) instead of O(n^2)"), also benchmark at multiple input sizes
  (e.g. n=100, 1000, 10000) to show the curve, not just one data point — a lower-order algorithm
  can still lose at small n due to constant factors.

---

## Common pitfalls that invalidate a comparison

- **Different hardware/load** — baseline measured on a quiet laptop, candidate measured with
  Slack and a Docker build running in the background.
- **No warm-up runs** — first invocation pays JIT/cache costs that never repeat in steady state.
- **Single-run "before/after"** — one run of each proves nothing; report medians across trials.
- **Dev build vs prod build** — comparing an unminified dev bundle's load time to a minified
  prod bundle's load time is not a fair test; always benchmark frontend on `npm run build` output.
- **Changing more than one variable** — new algorithm + new caching layer + new data format in a
  single "candidate" makes the delta unattributable.
- **Different dataset/input size** — baseline tested against 100 rows, candidate against a
  seeded 10,000-row fixture.
- **Network variability for remote calls** — third-party API latency swings can dwarf the code
  change being measured; mock/stub external calls or run enough trials to average it out.
- **Ignoring variance** — reporting "140ms vs 90ms" when stddev is +/-40ms means the difference
  may not be real; report spread, not just the point estimate.

---

## Results-reporting template

Drop this straight into a PR description or handoff doc:

```markdown
## Performance comparison: <what changed>

**Metric(s):** <e.g. p95 API latency, LCP, bundle size>
**Environment:** <hardware, build mode, dataset, tool + version>
**Trials:** <N runs each, warm-up discarded>

| Metric            | Baseline (median) | Candidate (median) | Delta      | % change |
|--------------------|-------------------|---------------------|------------|----------|
| LCP                | 2.4s              | 1.6s                | -0.8s      | -33%     |
| Bundle size (gz)   | 312 KB            | 248 KB              | -64 KB     | -21%     |
| API p95 latency    | 180ms             | 145ms               | -35ms      | -19%     |

**Variance:** <e.g. baseline stddev ±15ms, candidate stddev ±12ms>
**Isolated variable:** <yes — only X changed / no — X and Y both changed, see note>
**Verdict:** <candidate is a clear win / marginal, not worth the added complexity / regression, do not ship>
```
