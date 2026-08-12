---
name: html-comparison-docs
description: Use whenever the user wants a standalone HTML document to visually show and compare the results of something they're building. This includes before/after screenshots, two implementation approaches side by side, A/B design comparisons, benchmark/metric comparisons, code diffs, or "here's what changed" summaries. Produces self-contained single-file HTML (inline CSS/JS, no CDN) that works opened directly as a file or published as an artifact. Not for general dashboards or charts (use dataviz) or generic Artifact styling questions (use artifact-design) — this is specifically for the side-by-side / before-after / toggle comparison pattern.
---

# HTML Comparison Docs

Build a single self-contained HTML file that lets someone *see* a comparison rather than read about it. Use this when the deliverable is "show me A vs B," not when a sentence or a normal chart would do.

## When to use this vs. just describing results in text

Use a comparison doc when:
- There are two or more concrete things to hold side by side (screenshots, code, rendered UI, metrics) and layout/visual alignment itself carries information.
- The user will look at it more than once, share it, or want to toggle/scrub between states.
- "Before vs after" or "option A vs option B" is the actual shape of the result.

Just write text (or use `dataviz` for a single chart) when there's one clear winner to state, only one or two numbers change, or a table in chat is enough. Don't build a comparison doc to avoid writing a one-line answer.

## Structure of every comparison doc

1. A short header stating what's being compared and why (1-2 sentences, not a wall of prose).
2. The comparison itself, using one of the patterns below.
3. Optional: a metrics/summary table underneath the visual comparison.
4. Self-contained: no CDN `<script src>` / `<link href>` to external domains, no build step. Inline everything. Works via `open file.html` and as a Claude Artifact.

## Pattern 1 — Side-by-side columns (default choice)

Best for: two screenshots, two design options, two code blocks, anything where you want both visible at once and comparable at a glance.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Before vs After</title>
<style>
  :root { color-scheme: light dark; }
  body { font-family: system-ui, sans-serif; margin: 0; padding: 2rem; background: #f7f7f8; color: #1a1a1a; }
  @media (prefers-color-scheme: dark) { body { background: #16171a; color: #e8e8ea; } }
  h1 { font-size: 1.25rem; margin: 0 0 1.5rem; }
  .compare { display: grid; grid-template-columns: 1fr 1fr; gap: 1.25rem; align-items: start; }
  .panel { background: #fff; border: 1px solid #e2e2e5; border-radius: 10px; overflow: hidden; }
  @media (prefers-color-scheme: dark) { .panel { background: #1f2023; border-color: #303136; } }
  .panel header { padding: .6rem 1rem; font-weight: 600; font-size: .85rem; letter-spacing: .02em; text-transform: uppercase; border-bottom: 1px solid inherit; }
  .panel.before header { color: #b3532b; background: #fdf1ea; }
  .panel.after header { color: #1b7a4a; background: #eafaf1; }
  @media (prefers-color-scheme: dark) { .panel.before header { background: #3a2419; } .panel.after header { background: #123424; } }
  .panel img, .panel iframe { display: block; width: 100%; border: 0; }
  .panel .content { padding: 1rem; }
  /* Stack on narrow screens instead of squeezing columns */
  @media (max-width: 720px) {
    .compare { grid-template-columns: 1fr; }
  }
</style>
</head>
<body>
  <h1>Checkout flow — before vs after redesign</h1>
  <div class="compare">
    <div class="panel before">
      <header>Before</header>
      <img src="before.png" alt="Before screenshot">
    </div>
    <div class="panel after">
      <header>After</header>
      <img src="after.png" alt="After screenshot">
    </div>
  </div>
</body>
</html>
```

For 3+ options, change `grid-template-columns: 1fr 1fr` to `repeat(auto-fit, minmax(260px, 1fr))` so it wraps automatically instead of squeezing every column at once.

## Pattern 2 — Before/after slider (single image area, draggable divider)

Best for: two screenshots of the *same* view (same crop/framing) where the interesting part is a pixel-level visual diff — a redesign, a fixed layout bug, a color change.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Before/After Slider</title>
<style>
  body { font-family: system-ui, sans-serif; margin: 0; padding: 2rem; background: #f7f7f8; }
  .slider-wrap { position: relative; max-width: 900px; margin: 0 auto; aspect-ratio: 16/10; overflow: hidden; border-radius: 10px; user-select: none; border: 1px solid #ddd; }
  .slider-wrap img { position: absolute; inset: 0; width: 100%; height: 100%; object-fit: cover; display: block; }
  .after-img { clip-path: inset(0 0 0 50%); }
  .handle { position: absolute; top: 0; bottom: 0; left: 50%; width: 3px; background: #fff; box-shadow: 0 0 0 1px rgba(0,0,0,.2); cursor: ew-resize; transform: translateX(-1.5px); }
  .handle::after { content: "⇔"; position: absolute; top: 50%; left: 50%; width: 34px; height: 34px; background: #fff; border-radius: 50%; display: flex; align-items: center; justify-content: center; transform: translate(-50%, -50%); box-shadow: 0 2px 6px rgba(0,0,0,.3); font-size: 14px; }
  .tag { position: absolute; top: 10px; padding: 3px 8px; font-size: .7rem; font-weight: 700; text-transform: uppercase; background: rgba(0,0,0,.55); color: #fff; border-radius: 4px; }
  .tag.left { left: 10px; } .tag.right { right: 10px; }
</style>
</head>
<body>
  <div class="slider-wrap" id="wrap">
    <img src="before.png" alt="Before">
    <img class="after-img" id="afterImg" src="after.png" alt="After">
    <div class="tag left">Before</div>
    <div class="tag right">After</div>
    <div class="handle" id="handle"></div>
  </div>
<script>
  const wrap = document.getElementById('wrap');
  const afterImg = document.getElementById('afterImg');
  const handle = document.getElementById('handle');
  let dragging = false;
  function setPos(clientX) {
    const rect = wrap.getBoundingClientRect();
    let pct = ((clientX - rect.left) / rect.width) * 100;
    pct = Math.max(0, Math.min(100, pct));
    afterImg.style.clipPath = `inset(0 0 0 ${pct}%)`;
    handle.style.left = pct + '%';
  }
  handle.addEventListener('pointerdown', () => dragging = true);
  window.addEventListener('pointerup', () => dragging = false);
  window.addEventListener('pointermove', e => { if (dragging) setPos(e.clientX); });
  wrap.addEventListener('click', e => setPos(e.clientX));
</script>
</body>
</html>
```

## Pattern 3 — Tabbed comparison (3+ options, or limited space)

Best for: more than two options, mobile-first docs, or when panels are tall (long code) and side-by-side would force too much scrolling.

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Approach Comparison</title>
<style>
  body { font-family: system-ui, sans-serif; margin: 0; padding: 2rem; max-width: 800px; margin-inline: auto; }
  .tabs { display: flex; gap: .25rem; border-bottom: 2px solid #e2e2e5; margin-bottom: 1rem; flex-wrap: wrap; }
  .tab-btn { padding: .5rem 1rem; border: 0; background: none; cursor: pointer; font-size: .9rem; font-weight: 600; color: #888; border-bottom: 2px solid transparent; margin-bottom: -2px; }
  .tab-btn.active { color: #0c8aa4; border-bottom-color: #0c8aa4; }
  .tab-panel { display: none; }
  .tab-panel.active { display: block; }
  pre { background: #1e1e1e; color: #d4d4d4; padding: 1rem; border-radius: 8px; overflow-x: auto; }
</style>
</head>
<body>
  <div class="tabs">
    <button class="tab-btn active" data-tab="a">Option A</button>
    <button class="tab-btn" data-tab="b">Option B</button>
    <button class="tab-btn" data-tab="c">Option C</button>
  </div>
  <div class="tab-panel active" id="a"><pre>// implementation A</pre></div>
  <div class="tab-panel" id="b"><pre>// implementation B</pre></div>
  <div class="tab-panel" id="c"><pre>// implementation C</pre></div>
<script>
  document.querySelectorAll('.tab-btn').forEach(btn => {
    btn.addEventListener('click', () => {
      document.querySelectorAll('.tab-btn, .tab-panel').forEach(el => el.classList.remove('active'));
      btn.classList.add('active');
      document.getElementById(btn.dataset.tab).classList.add('active');
    });
  });
</script>
</body>
</html>
```

## Comparing specific content types

**Screenshots/images** — use Pattern 1 (side-by-side) for different views, Pattern 2 (slider) only when both images share identical framing/crop, otherwise the slider misleads. Always add `alt` text and a visible label (`Before`/`After`, not just color coding — color-blind-safe).

**Code diffs** — don't hand-roll a diff algorithm. Either paste both full snippets side-by-side (Pattern 1, one `<pre>` per panel) with changed lines highlighted manually via a `<mark>`/`.changed` class, or for a real line-by-line diff, generate the diff text yourself (e.g. via `diff -u`) and render it with simple CSS: lines starting `+` green background, `-` red background, monospace font. Example line styling:

```css
.diff-line { font-family: ui-monospace, monospace; white-space: pre; padding: 0 .5rem; }
.diff-add { background: #e6ffed; color: #22863a; }
.diff-del { background: #ffeef0; color: #b31d28; }
@media (prefers-color-scheme: dark) {
  .diff-add { background: #113321; color: #85e0a3; }
  .diff-del { background: #3a1a1e; color: #ff9aa5; }
}
```

**Metrics/numbers** — use a table, not prose, and make the delta explicit (don't make the reader subtract):

```html
<table style="width:100%; border-collapse:collapse; font-size:.9rem;">
  <thead>
    <tr style="text-align:left; border-bottom:2px solid #ddd;">
      <th style="padding:.5rem;">Metric</th><th style="padding:.5rem;">Before</th><th style="padding:.5rem;">After</th><th style="padding:.5rem;">Δ</th>
    </tr>
  </thead>
  <tbody>
    <tr style="border-bottom:1px solid #eee;">
      <td style="padding:.5rem;">LCP</td><td style="padding:.5rem;">3.2s</td><td style="padding:.5rem;">1.4s</td>
      <td style="padding:.5rem; color:#1b7a4a; font-weight:600;">-56%</td>
    </tr>
  </tbody>
</table>
```

**Rendered live content (iframes)** — for comparing two running pages/prototypes, embed both with `<iframe>` inside Pattern 1 panels:

```html
<iframe src="version-a.html" style="width:100%; height:500px; border:0;" title="Version A"></iframe>
```
Only do this for local/self-contained HTML — pointing at an external live site will typically be blocked by CSP/`X-Frame-Options` inside an Artifact sandbox. If the target can't be iframed, fall back to a screenshot.

## Responsive/mobile

Every pattern above must collapse to a single column below ~700-750px — comparisons that keep two columns on a 375px phone become unreadable. The rule to always include:

```css
@media (max-width: 720px) {
  .compare { grid-template-columns: 1fr !important; }
  .tabs { overflow-x: auto; flex-wrap: nowrap; }
}
```
For the slider pattern, keep it single-column full-width on mobile too — it already works at any width since it's one stacked element, just make sure the handle hit-target stays ≥ 34px for touch.

## Light/dark mode

Always add `color-scheme: light dark` on `:root` and pair every hardcoded light color with a `prefers-color-scheme: dark` override (see Pattern 1's CSS). If publishing as a Claude Artifact, also support the viewer's explicit theme toggle by adding `:root[data-theme="dark"]` / `:root[data-theme="light"]` overrides alongside the media query, since the toggle stamps `data-theme` on the root and must win in both directions.

## Quick reference

| Task | Pattern |
|---|---|
| Two screenshots, different views | Pattern 1 — side-by-side columns |
| Two screenshots, same crop, want pixel-level diff | Pattern 2 — before/after slider |
| 3+ options to compare | Pattern 3 — tabs (or Pattern 1 with `auto-fit` grid) |
| Code before/after | Pattern 1 with `<pre>` panels, or diff-line CSS for true line diffs |
| Metrics/benchmark numbers | HTML table with an explicit Δ column, inside or below any pattern |
| Two live/running prototypes | Pattern 1 with `<iframe>` panels (same-origin/local only) |
| Long content, limited vertical space | Pattern 3 — tabs |
| Deliverable is one sentence of a winner | Skip this skill, just say it |
