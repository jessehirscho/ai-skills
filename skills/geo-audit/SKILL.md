---
name: geo-audit
description: Use whenever the user wants to audit or improve how citable/retrievable their content is for AI answer engines (ChatGPT Search, Google AI Overviews/AI Mode, Perplexity, Claude web search, Bing Copilot) — distinct from ranking in a traditional blue-links SERP. Companion to the seo-audit skill, which covers traditional Google/Bing ranking; this one does not duplicate that.
---

# GEO Audit (Generative Engine Optimization)

GEO is the practice of making content easy for AI answer engines to retrieve, quote, and cite.
The term comes from a 2023 research paper ("GEO: Generative Engine Optimization," Aggarwal et
al.) and has since become a fast-moving practical discipline. **Re-verify bot names, engine
behaviors, and tooling claims periodically** — this file can go stale within months. Treat
anything below not marked "verified/mechanical" as current-best-understanding, not settled fact.

For classic ranking-factor / crawlability / backlink work, use the `seo-audit` skill instead.

## Why this is mechanically different from SEO

Traditional search ranks whole documents against a query and shows a link. Most AI answer
engines instead run some form of retrieval-augmented generation: they (or a search index they
call out to) fetch a set of candidate pages, chunk them, retrieve the passages most relevant to
the question, and have a model synthesize an answer — sometimes citing 1-3 sources, sometimes
paraphrasing without attribution at all. Consequences:

- The unit of competition is a **passage/chunk**, not a page. A page can rank #1 in Google and
  still never get quoted if no single self-contained chunk answers the question cleanly.
- A chunk needs to make sense **out of context** — pulled from mid-page, without your nav,
  headings above it, or brand framing, and dropped into someone else's answer.
- There is no guaranteed backlink or click. Being cited can mean zero referral traffic even when
  it's working — the value is brand/answer presence, not necessarily a visit.
- Being "selected" depends on retrieval matching + extractability + trust signals, not on
  backlink-driven authority scores alone (though authority still correlates with being crawled
  and trusted).

## 1. Confirm AI crawler access

Check `robots.txt` and server logs. Bot names change and multiply — verify current ones before
relying on this list, but as of 2026 the major ones (verify, don't assume) commonly cited:

| Purpose | Example user-agents (verify currency) |
|---|---|
| Training-data crawlers | `GPTBot` (OpenAI), `ClaudeBot` / `anthropic-ai` (Anthropic), `CCBot` (Common Crawl, feeds many models), `Google-Extended` (Gemini training/grounding), `Meta-ExternalAgent` |
| Search-index / retrieval bots | `OAI-SearchBot` (ChatGPT Search), `PerplexityBot`, `Bingbot` (feeds Copilot/Bing AI), `Google-Extended` also covers AI Overviews signals |
| Live user-triggered fetchers | `ChatGPT-User`, `Perplexity-User`, `Claude-User` — fired when a user pastes/asks about a specific URL |

**The tradeoff**: blocking training bots (`GPTBot`, `CCBot`, etc.) keeps your content out of
future model weights but does not necessarily stop you being cited live — retrieval/search bots
are often separate user-agents from training bots, so a site can block training and still allow
citation-relevant crawling, or vice versa. Per this project's existing decision, Soar Solutions
intentionally allows search, AI-answer, user-triggered, and model-training crawlers — that stance
is a defensible GEO-forward choice, not a mistake; don't second-guess it without a new owner
decision. When auditing a *different* site, treat "allow vs. block" as a business decision to
surface, not something to silently change — blocking trades citability for content control.

Verification is mechanical: `curl -A "GPTBot" https://example.com/robots.txt`, or just read the
file and check each bot name against `Disallow` rules, including any per-path exceptions.

## 2. Audit content structure for extractability

For each priority page/question, check:

- **Direct-answer-first**: does the first sentence or two of the relevant section actually answer
  the implied question, plainly? Avoid throat-clearing ("At Soar Solutions, we believe that...")
  before the fact. Lead with the fact, then elaborate/hedge/brand after.
- **Self-contained factual statements**: can a single sentence be lifted out and still make sense?
  ("Mobile physiotherapy in Bondi typically costs $X and includes Y" reads fine alone; "This
  service includes it too" does not.)
- **Definitional clarity**: state plainly what a thing *is* early in the page ("A mobile physio
  visit is..."), not just what it does for the reader.
- **Explicit Q&A / FAQ formatting**: literal question-as-heading followed by a direct-answer
  paragraph. This maps cleanly onto how answer engines pattern-match retrievable chunks.
- **Scannable lists**: numbered/bulleted facts (prices, steps, eligibility, symptoms) extract more
  cleanly than the same facts embedded in prose.
- **One idea per chunk**: avoid a single paragraph doing definition + caveat + CTA + unrelated
  fact — that dilutes what any one retrieved chunk can confidently answer.

## 3. Structured data — what's verified vs. still emerging

- **Verified/mechanical**: FAQPage, HowTo, Article, and LocalBusiness JSON-LD are machine-readable
  restatements of the same on-page content, and are an established, low-cost way to expose
  structured facts (Q, A, steps, entity attributes) unambiguously.
- **Honestly unconfirmed**: no AI answer engine vendor has published documentation stating
  schema.org markup directly increases citation likelihood the way it earns rich-result
  eligibility in classic Google Search. Practitioner consensus (blog/vendor content, not vendor
  documentation) treats it as *plausibly helpful, not proven* — a low-cost bet, not a lever with a
  confirmed multiplier. Don't oversell it to stakeholders as a guaranteed GEO win; frame it as
  "cheap, consistent with content, no known downside."
- If a page already has FAQPage schema for classic rich results, that work does double duty here
  — no separate GEO-only markup is usually needed, just keep the JSON-LD and visible copy in sync.

## 4. Authority and trust signals AI engines reportedly weight

Not independently verifiable per-engine (no public ranking-factor doc equivalent to Google's),
but consistently reported across GEO practitioner research:

- **Third-party citations**: being referenced by other sites/publications the engine already
  trusts appears to matter more than raw backlink count.
- **Author/expertise signals**: named author bios, credentials, and about/expertise pages — an
  E-E-A-T-style signal — reportedly help, especially for medical/health/financial content where
  engines apply more caution (this site is a physiotherapy business — relevant).
- **Cross-web factual consistency**: if your NAP (name/address/phone), pricing, hours, or service
  claims differ across your site, Google Business Profile, directories, and review platforms,
  that inconsistency is reported to get down-weighted as a trust signal. Consistency check is
  cheap and worth doing before anything fancier.
- **Recency/freshness cues**: visible last-updated dates and current-year references reportedly
  help on time-sensitive topics; low-stakes to add, uncertain magnitude of effect.

## 5. Measurement — there is no Search Console for this yet

- **Manual query testing (free, do this first)**: pick 8-15 real target questions a prospective
  customer would ask ("who does mobile physio in Bondi", "what does a mobile physiotherapy
  session cost"), run each through ChatGPT Search, Perplexity, and Google AI Overviews/AI Mode,
  and record: cited or not, which URL, what was quoted, and how accurately. Repeat periodically —
  answers are not stable over time.
- Perplexity reliably shows numbered source links, making it the easiest engine to audit citation
  behavior on. Google AI Overviews shows source chips/links. ChatGPT Search/Answers cites
  inconsistently and sometimes not at all even when it clearly used your content — absence of a
  visible citation isn't proof of non-use.
- **Server log / bot-hit monitoring (free, mechanical)**: grep access logs for the crawler
  user-agents from Section 1. Frequent `OAI-SearchBot` or `PerplexityBot` hits on a page indicate
  it's being actively indexed for retrieval, even without a confirmed citation yet.
- **Third-party citation-tracking tools**: a market of paid tools (e.g. Profound, Peec AI, Otterly,
  Scrunch, Siftly and similar) has emerged that automates the manual-query-testing loop at scale
  across engines. Useful once you have budget and need trend tracking across many queries; treat
  as a convenience layer over the same manual method, not a fundamentally different measurement —
  and this is a young, consolidating vendor category, so evaluate current offerings rather than
  trusting any specific name to still be the best option later.

## Audit procedure

1. **Crawler access** — read `robots.txt`; confirm intended AI crawlers/bots aren't blocked
   (or are deliberately blocked, per an explicit owner decision). Spot-check server logs for
   actual bot hits if available.
2. **Content structure** — for each priority page, check direct-answer-first opening, standalone
   factual sentences, explicit definitions, FAQ formatting, and scannable lists per Section 2.
   Flag pages that are well-optimized for SEO but bury the answer in paragraph 4.
3. **Structured data coverage** — confirm FAQPage/Article/LocalBusiness JSON-LD exists where
   relevant and matches the visible copy exactly (stale schema that contradicts the page is worse
   than no schema).
4. **Manual citation-check queries** — run 8-15 target questions across at least 2-3 of
   {ChatGPT Search, Perplexity, Google AI Overviews/AI Mode}; log cited/not, which page, quote
   accuracy.
5. **Prioritized fix list** — rank fixes by (a) how often the underlying question was asked in
   step 4 and not cited, (b) how cheap the fix is (rewriting an opening sentence vs. restructuring
   a page), (c) whether cross-web facts need reconciling first (Section 4) since that can suppress
   citation regardless of on-page quality.

## Field-currency warning

Bot user-agent names, which engines cite vs. paraphrase, whether structured data measurably
helps, and which third-party tracking tools are credible all change fast in this space. Before
relying on any specific name or claim from this file in a client-facing report, re-verify it with
a fresh search rather than assuming this file is still current.
