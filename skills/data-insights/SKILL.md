---
name: data-insights
description: Use whenever the user has messy, real-world data (CSV/Excel/JSON dumps, exported reports, scraped data, log files) and wants business insights out of it. Covers profiling and cleaning messy data, doing the actual analysis, and presenting findings as defensible, business-framed recommendations rather than raw statistics.
---

# Data Insights

Methodology for going from a messy raw data dump to a defensible business
insight. The failure mode this skill guards against: cleaning data blindly,
running whatever aggregation seems obvious, and presenting a number without
context, confidence, or business framing. Follow the steps in order — do not
skip profiling to jump to analysis, and do not skip the sanity-check to jump
to presentation.

## Step 0: Profile before touching anything

Never clean data you haven't looked at. Run this block first, every time,
before writing a single `.fillna()` or `.drop_duplicates()`:

```python
import pandas as pd

df = pd.read_csv("data.csv")  # or read_excel, read_json

df.shape
df.head(10)
df.info()                      # dtypes, non-null counts — mixed types show up as `object`
df.describe(include="all")     # numeric ranges + categorical top/freq in one call
df.isna().sum().sort_values(ascending=False)
df.duplicated().sum()          # exact full-row duplicates
for col in df.select_dtypes("object").columns:
    print(col, df[col].nunique(), df[col].unique()[:10])  # cardinality + sample values
```

What you're looking for:
- **dtype surprises**: a numeric-looking column typed `object` almost always means embedded currency symbols, commas, or stray text.
- **cardinality**: a categorical column with far more unique values than expected usually means inconsistent casing/whitespace/typos, not real distinct categories (e.g. "NSW", "nsw", "New South Wales " all meaning one thing).
- **missingness pattern**: is it random, or concentrated in specific columns/rows/time periods? `df[df['col'].isna()]` — look at what else is true about those rows before deciding how to handle them.
- **date column dtype**: if a date column isn't `datetime64`, don't assume `pd.to_datetime` will parse it cleanly — check for mixed formats first (see below).

Only after this do you know what's actually wrong — and cleaning steps you
didn't find a symptom for are usually unnecessary work.

## Common problems and their fixes

| Symptom | Fix |
|---|---|
| Inconsistent casing/whitespace in strings | `df['col'] = df['col'].str.strip().str.lower()` (or `.title()` if display matters) |
| Mixed date formats in one column | `pd.to_datetime(df['col'], format='mixed', dayfirst=True, errors='coerce')` then check `.isna()` for rows that still failed |
| Numbers stored as strings with `$`, `,`, `%` | `df['col'] = pd.to_numeric(df['col'].replace(r'[\$,%,]', '', regex=True), errors='coerce')` |
| Exact duplicate rows | `df = df.drop_duplicates()` |
| Duplicate business keys with differing other columns (e.g. same order ID, different timestamps) | Decide which record wins: `df.sort_values('updated_at').drop_duplicates('order_id', keep='last')` — never silently `drop_duplicates()` on the whole row, it won't catch these |
| Missing values | See decision guide below — don't default to dropping |
| Outliers | IQR or z-score flagging — see below; decide keep/exclude per business context, not automatically |
| Inconsistent category spelling ("NSW" vs "N.S.W." vs "New South Wales") | Build an explicit mapping dict and `.map()`/`.replace()` it; don't try to regex-guess your way through this |
| Free-text fields that should be categorical | `.value_counts()` first to see the real vocabulary before deciding how to bucket it |
| Encoding errors / mojibake | Re-read with explicit `encoding='utf-8'` or `'latin1'`; `df['col'].str.encode('latin1').str.decode('utf-8')` as a repair attempt |
| Leading/trailing whitespace hiding as "different" values | `df.columns = df.columns.str.strip()` too — column *names* get this bug as often as values |

### Missing values: drop vs impute vs flag

Don't default to dropping. Ask three questions in order:

1. **How much is missing, and is it random?** Under ~5% and no pattern → dropping rows is usually safe: `df.dropna(subset=['col'])`.
2. **Does "missing" itself mean something?** (e.g. a `cancelled_at` column being null means "not cancelled" — that's not missing data, that's a signal.) Don't impute over it; keep it as-is or convert to a boolean flag: `df['is_cancelled'] = df['cancelled_at'].notna()`.
3. **Is the column needed for the analysis and not random?** Impute with a defensible value and say so in the write-up — median for skewed numerics (`df['col'].fillna(df['col'].median())`), mode for categoricals, or group-wise imputation (`df['col'].fillna(df.groupby('segment')['col'].transform('median'))`) when the missingness correlates with another column. Never impute silently — flag it: `df['col_was_imputed'] = df['col'].isna()` before you fill it, so downstream analysis can exclude or weight those rows if needed.

### Outliers: detect, then decide — don't auto-remove

```python
# IQR method (robust, good default for skewed business data)
q1, q3 = df['col'].quantile([0.25, 0.75])
iqr = q3 - q1
lower, upper = q1 - 1.5 * iqr, q3 + 1.5 * iqr
outliers = df[(df['col'] < lower) | (df['col'] > upper)]

# z-score method (better for roughly-normal distributions)
from scipy import stats
df['zscore'] = stats.zscore(df['col'].dropna())
outliers = df[df['zscore'].abs() > 3]
```

Then look at the flagged rows individually before excluding anything:
- A $50,000 order in a table of $50 orders might be a data-entry error (extra
  zero) — or it might be your single biggest customer. Excluding it silently
  can erase the most important row in the dataset.
- If you exclude outliers, report how many and why in the final write-up —
  "3 rows excluded as likely data-entry errors (order values >$40k with no
  matching line items)" — not silently.

## From clean data to insight

**Start with a specific business question, not "let's explore."** "What's
driving the drop in repeat purchases in Q2?" produces a different (and
useful) analysis path; "let's look at the data" produces a pile of charts
nobody asked for. If the user hasn't stated the question, ask what decision
this analysis is meant to inform before picking an aggregation.

Once the question is specific, the pandas pattern usually falls out of it:

```python
# "Which segment/category is driving the change?"
df.groupby('segment')['revenue'].agg(['sum', 'mean', 'count']).sort_values('sum', ascending=False)

# "How does it break down two ways?" (e.g. segment x month)
pd.pivot_table(df, values='revenue', index='segment', columns='month', aggfunc='sum')

# "What's the trend over time?"
df.set_index('date')['revenue'].resample('W').sum()          # weekly
df.set_index('date')['revenue'].resample('MS').sum().pct_change()  # month-over-month % change

# "Is this a real change or noise?" — compare period-over-period with counts, not just totals
df.groupby(pd.Grouper(key='date', freq='ME')).agg(revenue=('revenue','sum'), n=('order_id','count'))
```

Keep the aggregation as close to the original question as possible — resist
adding three more cuts of the data "while you're at it." Extra unrequested
breakdowns dilute the one finding that actually answers the question.

## Sanity-check before presenting

Before a finding leaves this analysis, run it through these checks:

- **Does the magnitude make sense?** A claimed "300% increase" from 2 orders
  to 8 orders is technically true and practically meaningless. State the
  absolute numbers alongside any percentage.
- **Is the sample size big enough to matter?** A segment with n=12 driving a
  headline stat should be called out as low-confidence, not presented with
  the same authority as a segment with n=4,000.
- **Could this be explained by a confound?** Before saying "Channel A
  converts better than Channel B," check whether Channel A also skews toward
  a different price tier, season, or customer type that could explain the
  gap on its own. `df.groupby(['channel', 'confound'])['metric'].mean()` —
  if the pattern disappears when you control for the confound, the original
  finding was spurious.
- **Correlation vs causation** — if the finding implies "X causes Y," check:
  did X change before Y in time? Is there a plausible mechanism? Would a
  reasonable skeptic in the room accept this, or is "these two things moved
  together" being oversold as "one drove the other"? If it's genuinely just
  correlation, say so explicitly rather than implying causation through word
  choice ("associated with," not "caused by," "drove," or "led to").

If a finding fails any of these, either dig one level deeper before
presenting it, or present it with the caveat attached — don't quietly drop
the caveat to make the story cleaner.

## Presenting the insight in business terms

Structure every finding the same way:

1. **Lead with the "so what."** The first sentence is the decision-relevant
   takeaway, not the method. "Repeat-purchase rate dropped 18% in Q2,
   concentrated entirely in the email channel" — not "I grouped the data by
   channel and computed retention rates."
2. **Quantify impact**, in the business's own units where possible (revenue,
   customers, hours) rather than only in statistical units (percentages,
   p-values). "This represents roughly $42k in Q2 revenue at risk if the
   trend continues" lands harder than "the coefficient was -0.18."
3. **State confidence and caveats explicitly, in the same paragraph as the
   claim** — not buried in a methodology appendix. "Based on 340 orders
   across 6 weeks — directionally reliable but too short a window to rule
   out seasonality" is honest and still actionable. Hiding the caveat to
   make the finding sound more certain than it is will cost trust the first
   time it's wrong.
4. **Recommend, don't just report**, when the user's asking for a decision
   input. "This suggests X" is a fact; "given this, consider Y" is what
   business audiences usually actually need, if the data supports going that
   far — don't manufacture a confident recommendation from thin evidence.

For the actual chart or visualization to accompany the finding, use the
`dataviz` skill — it covers chart-type selection, color, and layout in
depth and shouldn't be duplicated here.

## Quick-reference checklist

- [ ] Profiled with `.info()`, `.describe()`, `.isna().sum()`,
      `.duplicated().sum()`, and cardinality checks before cleaning anything
- [ ] Identified whether missingness is random or meaningful, and chose
      drop/impute/flag deliberately, not by default
- [ ] Detected outliers with IQR or z-score, inspected them individually,
      and documented any exclusions with counts and reasons
- [ ] Started analysis from one specific business question, not open-ended
      exploration
- [ ] Checked magnitude, sample size, confounds, and correlation-vs-causation
      before trusting the headline finding
- [ ] Final write-up leads with the takeaway, quantifies impact in business
      units, and states confidence/caveats in the same breath as the claim —
      not hidden in a footnote
