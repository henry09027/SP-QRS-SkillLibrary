---
name: Cosine Similarity Distribution Analysis Skill
description: Use this skill whenever the user asks about the distribution of year-over-year changes in S&P 500 10-K risk factor sections — including phrasings such as "changes distribution", "cosine distribution", "change distribution", "distribution of changes", "how much did 10-Ks change", "YoY change distribution", "distribution of edits/revisions", "spread of risk factor changes", "histogram of changes", "how similar/different are filings year-over-year", or any Lazy-Prices–style change analysis. In this domain, the magnitude of change between consecutive 10-K filings is operationalized as Loughran-McDonald (LM) cosine similarity over the risk factor section — high cosine similarity means a small change, low cosine similarity means a large change — so any query about the distribution, spread, histogram, or bucketing of changes between consecutive 10-K filings maps to this skill, even when the user does not say the word "similarity". The skill executes a deployed Snowflake stored table function that returns a deterministic, bucket-by-bucket aggregation filtered by fiscal year, and renders that result set as a table plus a distribution plot.
---

## Objective
Returns a flat, tabular result set (similarity buckets × fiscal year × filing counts) for the Lazy-Prices–style YoY risk-factor change analysis. All bucket boundaries, year filtering, projection, and ordering are encapsulated inside the deployed table function `QRSLLM_POC_DB.LAZY_PRICES_DECLARATIVE_SHARE_BETA.SP_LM_SIMILARITY_DISTRIBUTION`, so every invocation produces an identical schema and identical bucket logic regardless of caller.

---

## 🎯 EXECUTION PROCEDURE — FOLLOW EXACTLY

### Step 1 — Execute the stored table function
Execute exactly:

```sql
SELECT * FROM TABLE(QRSLLM_POC_DB.LAZY_PRICES_DECLARATIVE_SHARE_BETA.SP_LM_SIMILARITY_DISTRIBUTION({YEAR}));
```

Substitute `{YEAR}` with the four-digit fiscal year as an integer literal (e.g. `2024`). Do not wrap the call in a `SELECT`, do not add a `LIMIT`, and do not pass additional arguments — the table function signature accepts exactly one `INTEGER`.

### Step 2 — Display the data in a table
Render the returned result set using the column order and types in the *Output Table Form* below. The table function returns Snowflake-default UPPERCASE identifiers — preserve them. Do not rename, re-case, or post-process columns. Accompany the table with a distribution plot.

### Step 3 — Narrate in ≤ 4 sentences
State (i) total filing count for the year, (ii) the shape of the distribution (where mass concentrates), and (iii) any interesting observations across buckets. Narrate using bullet points, do not enumerate every bucket.

---

## Table Function Contract

- **Fully qualified name:** `QRSLLM_POC_DB.LAZY_PRICES_DECLARATIVE_SHARE_BETA.SP_LM_SIMILARITY_DISTRIBUTION`
- **Input:** `YEAR_INPUT INTEGER` — single fiscal year, e.g. `2024`.
- **Output:** six-column table, one row per non-`'other'` bucket present in the data for that year.

---

## Output Table Form

| Column              | Type    | Description                                            |
|---------------------|---------|--------------------------------------------------------|
| `SIMILARITY_BUCKET` | STRING  | Ordered bucket label, e.g. `'0.99-0.995'`              |
| `FISCAL_YEAR`       | INT     | Year extracted from `next_perioddate`                  |
| `FILING_COUNT`      | INT     | Filings in this (bucket × year) cell                   |
| `MIN_SIMILARITY`    | FLOAT   | Min cosine similarity in cell                          |
| `MAX_SIMILARITY`    | FLOAT   | Max cosine similarity in cell                          |
| `AVG_SIMILARITY`    | FLOAT   | Mean cosine similarity in cell                         |

---

## ❌ DOs and DON'Ts

1. **Do not re-create, alter, or replace the table function** from within this skill. The table function is managed and deployed out-of-band; the skill is a pure consumer.
2. **Do not inline the bucket-logic SQL** as a substitute. The point of the table  function is that buckets and filters are defined once, server-side — ad-hoc SQL defeats reproducibility.
3. **Do not append a `LIMIT`**. The table function already returns at most 6 rows per year.
4. **Do not lowercase or quote-wrap column names** in display. Preserve Snowflake-default UPPERCASE identifiers.
---
