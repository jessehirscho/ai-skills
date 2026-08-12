---
name: seo-audit
description: Use whenever the user wants to audit or improve a website's traditional search engine SEO — crawlability, indexation, on-page signals, technical SEO, structured data, Core Web Vitals as a ranking factor, and backlinks for Google/Bing organic ranking. Covers React/Vite and other client-side SPAs explicitly, since they commonly ship uncrawlable pages by default.
---

# SEO Audit

A practical checklist-and-procedure skill for classic search engine optimization: making pages
crawlable, indexable, and well-signaled for Google/Bing organic ranking. This is about ranking
mechanics, not AI-answer-engine citability — for that, use the companion `geo-audit` skill.

## When NOT to bother

Skip the full procedure if the change is copy-only with no template/markup impact (e.g. fixing a
typo in body text) — spot-check the one page instead of running the whole audit.

If a ranking or traffic claim is being made ("this will help us rank", "this fixes our SEO"), it
needs evidence from this procedure — a `curl`, a validator output, a real check — not intuition.

## The SPA trap (check this first, always)

Client-side-rendered SPAs (React/Vite, CRA, plain SPA) often ship an empty shell to crawlers even
though a human sees full content, because JS execution isn't guaranteed at crawl/index time.
Before anything else, verify what a crawler actually receives:

```bash
curl -s https://example.com/some-page | less
# Look for: real <title>, real <h1>/body text, real <link rel="canonical">, JSON-LD blocks
# Red flag: an almost-empty <div id="root"></div> or <div id="app"></div> with no content,
# meaning the page depends on client-side JS to render anything meaningful.
```

If the raw HTML is empty, the site needs prerendering (static HTML generated per route at build
time, e.g. a Vite postbuild script), SSR, or dynamic rendering. Confirm the fix by re-running the
same `curl` against the built/deployed output and checking the title/H1/JSON-LD are present
verbatim, not just present after JS runs. Also check that this generated HTML is genuinely what
gets served in production (not just present in a local `dist/` folder) — `curl` the live URL, not
localhost, for the final confirmation.

## Procedure

### 1. Crawlability

- **robots.txt** — `curl https://example.com/robots.txt`. Confirm it's not blocking paths that
  should be indexed (a stray `Disallow: /` is the most common catastrophic mistake). Confirm it
  references the sitemap (`Sitemap: https://example.com/sitemap.xml`).
- **XML sitemap** — `curl https://example.com/sitemap.xml`. Confirm every canonical page type is
  present (core pages, content/article pages, category/location pages, etc.), URLs are absolute
  and return 200 (spot-check a sample with `curl -o /dev/null -s -w "%{http_code}\n" <url>`), and
  `<lastmod>` dates are plausible/fresh, not hardcoded or stale. If the sitemap is generated at
  build time, confirm it's regenerated automatically when content is added — not a file someone
  has to remember to hand-edit.
- **Canonical tags** — spot-check a handful of pages for a single, correct, self-referencing (or
  intentionally cross-referencing) `<link rel="canonical">`. Watch for canonicals that
  accidentally point to a dev/staging domain, a paginated root, or all point to the homepage.
- **Accidental noindex** — grep the codebase and crawl a sample of live pages for
  `noindex`/`nofollow` meta or `X-Robots-Tag` headers:
  ```bash
  grep -rn "noindex" src/ public/
  curl -sI https://example.com/some-page | grep -i x-robots-tag
  ```
  Confirm nothing important is unintentionally excluded, and that anything that *should* be
  excluded (thank-you pages, internal search results, staging routes) actually is.

### 2. Indexation status

- `site:example.com` search (manual, in a browser) gives a rough count of indexed pages — compare
  against the sitemap's page count. A large gap (sitemap has 90 pages, `site:` shows 20) signals
  an indexation problem worth investigating, not just a technical-audit pass.
- If Google Search Console access is available, check Coverage/Indexing report for excluded pages
  and the reason given (crawled-not-indexed, duplicate content, soft 404, etc.) — this is ground
  truth and should be preferred over inference from `site:` searches.

### 3. On-page audit (per template/page-type, not every single page)

Audit one representative page per template (home, article/blog post, category/location page,
product page, etc.) — issues are template-level, so fixing one page's template fixes all pages
using it.

- **Title tag** — present, unique per page, ~50-60 characters (avoid truncation in SERPs),
  primary keyword near the front, no duplicate titles across pages of the same template.
- **Meta description** — present, ~120-158 characters, unique per page, written as a compelling
  summary (not keyword-stuffed) since it drives click-through even though it's not a direct
  ranking signal.
- **Heading hierarchy** — exactly one `<h1>` per page, matching the page's primary topic; `<h2>`s
  logically nested under it; no skipped levels (`h1` -> `h3` with no `h2`) used purely for
  styling.
- **Image alt text** — meaningful, non-empty `alt` on content-relevant images; decorative images
  use `alt=""` rather than omitting the attribute.
- **Internal linking** — new/important pages are linked from at least one other indexed page
  (orphan pages get crawled less and rank worse); anchor text is descriptive, not "click here."

### 4. Structured data (JSON-LD)

- Locate the JSON-LD block(s) per template and confirm the `@type` matches the page's actual
  content (`Article`/`BlogPosting` for posts, `LocalBusiness` for a business page, `FAQPage` for
  FAQ content, `Product` for product pages, etc.).
- Sanity-check the JSON is valid and non-empty before reaching for a browser tool:
  ```bash
  curl -s https://example.com/some-page \
    | grep -o '<script type="application/ld+json">.*</script>' \
    | sed -e 's/<script[^>]*>//' -e 's/<\/script>//' \
    | jq .
  ```
  A `jq` parse error means malformed JSON-LD, which most validators/crawlers will simply ignore.
- For a definitive check (schema completeness, warnings, rich-result eligibility), run the page
  through Google's Rich Results Test (search.google.com/test/rich-results) or the schema.org
  validator — the `jq` check above only proves the JSON parses, not that it satisfies a given
  schema's required properties.
- Confirm required properties for the type are present (e.g. `LocalBusiness` needs `name`,
  `address`, `telephone`; `FAQPage` needs `mainEntity` with `Question`/`Answer` pairs).

### 5. Core Web Vitals (as a ranking factor)

CWV is one ranking signal among many — don't over-index on it, but a page failing thresholds
badly is worth flagging. Current thresholds (75th percentile, field data):

| Metric | Good | Needs improvement | Poor |
|--------|------|--------------------|------|
| LCP (Largest Contentful Paint) | ≤2.5s | ≤4.0s | >4.0s |
| INP (Interaction to Next Paint) | ≤200ms | ≤500ms | >500ms |
| CLS (Cumulative Layout Shift) | ≤0.1 | ≤0.25 | >0.25 |

For actual measurement tooling and methodology (Lighthouse CLI, web-vitals RUM, before/after
comparisons), use this repo's `performance-audit` skill — don't duplicate that setup here. This
section exists only to flag CWV as an SEO input worth checking, not to re-teach how to measure it.

### 6. Backlinks / off-page (audit-level only)

This is not a substitute for a dedicated backlink tool (Ahrefs, Semrush, Moz) — those give
referring-domain counts and link quality signals a manual audit can't reproduce. At an audit
level, without such a tool, check what's available:

- Any owner-controlled profiles/directories already claimed (Google Business Profile, industry
  directories, partner sites) actually link back to the site with a working URL.
- Anchor text on those known backlinks isn't uniformly exact-match keyword (a red flag for
  historical over-optimization, and a diversity check worth a comment even without full data).
- If Search Console access exists, the Links report gives a real (if incomplete) view of top
  linking sites and anchor text — prefer that over guessing.
- Flag this section as incomplete in the report if no backlink tool or Search Console access is
  available — say "not evaluated, needs a backlink tool" rather than fabricating a confident
  verdict from nothing.

### 7. Local SEO (for local-business sites)

- **NAP consistency** — Name, Address, Phone must match exactly (formatting included) across
  every page of the site and against the Google Business Profile listing. Grep for the phone
  number and address across the codebase to catch stale/inconsistent copies:
  ```bash
  grep -rn "555-0100\|123 Example St" src/
  ```
- **Google Business Profile** — confirm the profile is active, category is correct, and the
  website URL on the profile points to the live production domain (not a staging/legacy domain).
  Do not invent or alter review counts/ratings — use the owner-supplied real numbers only.
- **Local structured data** — `LocalBusiness` (or a more specific subtype) JSON-LD present with
  matching `name`, `address`, `telephone`, and ideally `openingHours`/`geo` — validate the same
  way as step 4.
- **Location-page duplication** — if there are multiple location/suburb pages, confirm each has
  genuinely distinct content (not a templated find-replace of the city name alone), since
  near-duplicate local pages can suppress rather than help rankings.

### 8. Prioritized fix list

Order findings by likely ranking impact, roughly:

1. Crawlability/indexation blockers (accidental noindex, blocked robots.txt, empty SPA shell) —
   fix first, nothing else matters if pages can't be crawled/indexed at all.
2. Missing/broken structured data for the page's primary type.
3. Title/meta description issues on high-value pages (home, top service/category pages).
4. Heading hierarchy and internal linking gaps.
5. Core Web Vitals failures on high-traffic templates.
6. Backlink/off-page and local-listing consistency issues.

---

## Quick-reference table

| Check | How to verify | Common fix |
|-------|----------------|------------|
| robots.txt not blocking site | `curl .../robots.txt` | Remove stray `Disallow: /`, add `Sitemap:` line |
| Sitemap complete & fresh | `curl .../sitemap.xml`, spot-check URLs return 200 | Regenerate sitemap at build time from source data |
| SPA serves real HTML to crawlers | `curl` the live URL, look for content outside `<div id="root">` | Add prerendering/SSR/dynamic rendering |
| Canonical tags correct | View source / `curl`, check `<link rel="canonical">` | Fix to self-referencing absolute URL per page |
| No accidental noindex | `grep -rn noindex`, check `X-Robots-Tag` header | Remove stray noindex meta/header |
| Indexed page count matches sitemap | `site:example.com` search or GSC Coverage report | Investigate excluded-page reasons in GSC |
| Title tag unique, ~50-60 chars | View source per template | Rewrite per-template title logic |
| Meta description present, ~120-158 chars | View source per template | Add/rewrite per-template description |
| Single H1, logical heading order | View source / DOM inspection | Fix heading levels in template markup |
| Image alt text present | Grep `<img` tags, check `alt=` | Add descriptive alt text, `alt=""` for decorative |
| Internal links to new/important pages | Manual link-graph check or crawler tool | Add contextual internal links from related pages |
| JSON-LD valid and complete | `jq` parse + Rich Results Test | Fix malformed JSON, add required schema properties |
| Core Web Vitals within thresholds | Lighthouse CLI (see `performance-audit` skill) | Optimize LCP asset, reduce JS for INP, reserve layout space for CLS |
| NAP consistent across site & GBP | Grep phone/address, compare to GBP listing | Standardize NAP string, update stale copies |
| Backlinks present from owned profiles | Manual check of known directories/GBP, or backlink tool | Claim/fix broken profile links |

---

## Reporting template

```markdown
## SEO audit: <site/page(s)>

**Scope:** <full site / specific template(s) / specific page(s)>
**Date:** <date>

### Crawlability & indexation
- robots.txt: <pass/fail + detail>
- Sitemap: <pass/fail + detail>
- SPA HTML check: <pass/fail — what curl showed>
- Canonical tags: <pass/fail + detail>
- Accidental noindex: <none found / found on X>
- Indexed vs sitemap count: <N indexed vs M in sitemap>

### On-page (by template)
| Template | Title | Meta desc | H1/headings | Alt text | Internal links |
|----------|-------|-----------|-------------|----------|-----------------|

### Structured data
- <page type>: <valid/invalid, missing properties if any>

### Core Web Vitals
- See `performance-audit` skill output for numbers; flag any template failing thresholds here.

### Backlinks / local SEO
- <findings, or "not evaluated — no backlink tool/GSC access">

### Prioritized fix list
1. <highest impact>
2. ...
```
