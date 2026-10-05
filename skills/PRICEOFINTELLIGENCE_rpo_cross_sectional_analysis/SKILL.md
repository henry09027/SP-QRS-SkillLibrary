---
name: rpo-cross-sectional-analysis
description: "Produce a sourced, cross-sectional comparison of Remaining Performance Obligation (RPO) / revenue-backlog balances across a peer set of cloud vendors and AI hyperscalers, pairing SEC-filed balances with earnings-call commentary on private AI-lab concentration, and render the results as a comparative chart. Use for: RPO peer comparison, backlog benchmarking, AI-lab exposure across hyperscalers, cross-company backlog concentration. Triggers: cross sectional RPO, RPO comparison, hyperscaler backlog, compare RPO, backlog benchmarking, AI lab exposure comparison."
---

# Cross-Sectional RPO / Backlog & AI-Lab Exposure

## Purpose

For a peer set of cloud vendors / AI hyperscalers, retrieve each company's latest filed backlog balance (RPO, "revenue backlog", or "commitments not yet recognized"), pair it with earnings-call commentary on private AI-lab concentration (OpenAI, Anthropic, xAI, and peers), and present the comparison as a chart plus tables.

## Output Style (mandatory)

- **Tables and charts carry the analysis, not prose.** Do not write an opening framing paragraph or a closing summary paragraph. No "Summary." lead-in, no "Read-through" narrative wrap-up. Keep the output concise and professional.
- Every quantitative fact lives in a table cell or on the chart — never restated in sentences.
- The comparability caveat and any coverage gaps are conveyed via the chart footnote and a short bulleted `Notes` block — not a prose essay.
- Allowed prose: section headers, table/column headers, terse (<1 line) bullets in `Notes`, and STOP checkpoints. Nothing else.

## Inputs

| Input | Description | Example |
|---|---|---|
| Peer set | List of companies (names or tickers) | "Oracle, Microsoft, Alphabet, Amazon, Meta" |
| Period (optional) | Reporting window of interest | "latest", "Q2 2026" |

If the peer set is missing, request it once, then proceed. Default to each company's **latest filed** balance when no period is specified.

## Prerequisites

- Pronto MCP tools: `getCompanies`, `searchSentences` (required); `getDocuments`, `getSentenceContext` (optional).
- A code sandbox with `matplotlib` for the comparative chart.
- All figures and quotations must originate from Pronto results. Fabricated balances, percentages, or contract values are a hard failure.

---

## Citation rule (mandatory)

Every filing- or transcript-derived value must carry a visible, clickable inline source link, reproduced verbatim from the Pronto result field, placed in the table's `Source` column.

**Required format:** literal Markdown link syntax — `[MARKER](URL)` — non-empty marker, no space between `]` and `(`.

```
Valid:   [#1170](https://spglobal.prontonlp.com/#/ref/$SENTID_SEC000262329667-1170)
```

**Invalid forms — all render as dead or invisible text:**

| Invalid | Reason |
|---|---|
| `[](url)` | Empty marker |
| `[#1170]` | No URL |
| Bare URL | Not a link in the client |
| `[1]` footnote pointer | Not resolvable inline |
| `` `[#1170](url)` `` | Backticks suppress rendering |

**Integrity:**

- One link per source; where multiple sources back one cell, include all of them in that cell.
- Copy markers and URLs verbatim. Never invent, reword, shorten, renumber, or construct a URL.
- A `Source` cell must contain the clickable `[MARKER](url)`, never a bare marker or a description.

---

## Workflow

### Step 1 — Resolve every company

Call `getCompanies` once per name/ticker in the peer set.

- Single match: retain `companyId` and continue.
- Multiple matches (e.g. "Oracle" → Oracle Corp, Oracle Japan, Oracle Power): select the primary listing by market cap / expected exchange, or present the candidates and **STOP** if genuinely ambiguous.

### Step 2 — Derive the date window

Map each company to its fiscal calendar; RPO is disclosed at period-end and these differ (e.g. Oracle FY ends May, Microsoft June, most others December). Use a window wide enough to capture the target period's 10-K/10-Q **and** the matching call:

- `gte` ≈ two quarters before the target period-end (or ~6 months back for "latest")
- `lte` = current date

Widen the window rather than risk missing a disclosure.

### Step 3 — Retrieve the filed backlog balance (per company)

Call `searchSentences` with `corpus: ["SEC Filings"]`, `companiesIds: [companyId]`, the Step-2 `dateRange`, `size: 20`, and a `topicSearchQuery` adapted to the issuer's vocabulary — the label is **not** uniform:

| Issuer style | Disclosure term | Suggested query |
|---|---|---|
| Oracle, Microsoft | "remaining performance obligations" | `remaining performance obligations total RPO balance` |
| Alphabet | "revenue backlog" | `revenue backlog remaining performance obligations cloud` |
| Amazon | "commitments not yet recognized" (primarily AWS) | `commitments not yet recognized AWS backlog remaining performance obligation` |

Extract for the target period-end, retaining each result's inline source link: total balance (and any commercial/cloud subtotal), weighted-average duration, % recognized within 12–24 months, and the definition sentence.

**No-disclosure case.** Some issuers (e.g. Meta) explicitly decline to disclose unsatisfied performance obligations and are net **buyers** of compute rather than hyperscale sellers. Record them as "not disclosed", capture the declining sentence, and exclude them from the ranked bars (show as an annotated gap).

### Step 4 — Retrieve backlog & AI-lab concentration commentary (per company)

Call `searchSentences` with `corpus: ["S&P Transcripts"]`, `companiesIds: [companyId]`, the Step-2 `dateRange`, `size: 15`, and `topicSearchQuery` such as `RPO backlog AI cloud contracts OpenAI Anthropic concentration`.

Capture, retaining each result's inline source link:
- **Direct concentration disclosure** — a lab's stated share of RPO/backlog.
- **Named-counterparty commitments** — dollar-sized lab contracts.
- Diversification framing (some issuers stress "a broad mix of customers").
- Volatility / lumpiness caveats management attaches to large lab contracts.

Where a company frames its backlog as diversified rather than lab-concentrated, record that framing rather than forcing a concentration figure. Rerun naming a second lab where material (e.g. Amazon → OpenAI and Anthropic).

### Step 5 — Build the comparative chart

In the sandbox, produce a single bar chart of latest filed balances:

- One bar per company, ranked descending by balance.
- Value labels on bars; a distinct greyed/annotated placeholder for any "not disclosed" company.
- X-axis labels include the **period-end date** so unequal fiscal calendars are visible; footnote any label mismatch (e.g. Amazon's "commitments not yet recognized").
- Chart footnote carries the comparability caveat (RPO ≠ recognized revenue; differing calendars/labels; non-disclosers).
- Write to `/tmp` then `cp` to `/workspace/`; surface with an `<asset>` tag.

Chart hygiene: title, axis units ("US$ billions"), source footnote, no chartjunk.

### Step 6 — Assemble output (tables + chart only) and self-check

Assemble per the **Output template** below. Before presenting, verify:

1. **No banned prose.** No opening framing paragraph, no closing summary/read-through paragraph. All values sit in tables or on the chart.
2. **Citation check.** Every quantitative table row has a non-empty, clickable `[MARKER](url)` in its `Source` column; no citations float in body text.
3. **Provenance check.** Each row's `Status` correctly marks it Filed / Call-sourced / Analyst-derived; no derived figure is labelled as a disclosure.

---

## Output template

```
# AI Hyperscalers — Cross-Sectional RPO / Backlog & AI-Lab Exposure

[comparative chart asset]

## 1. Filed backlog balances (latest SEC filing)

| Company | Backlog measure | Latest balance | Period-end | 12–24mo recognition | AI-lab counterparty | Source |
|---|---|---|---|---|---|---|
| ... | RPO / revenue backlog / commitments | $NNN B | YYYY-MM-DD | ~NN% | <lab or "diversified" or "n/a"> | [#NNNN](url) |
| <non-discloser> | Not disclosed | — | — | — | n/a | [#NNNN](url) |

## 2. AI-lab concentration

| Company | Disclosure type | Lab | Stated share | Derived $ (share × backlog) | Volatility caveat | Source |
|---|---|---|---|---|---|---|
| ... | Direct % / Named contract / Diversification | <lab> | ~NN% or n/d | $NNN B (DERIVED) or — | Y/N + terse note | [#NNNN](url) |

## 3. Verification status

| Claim | Source | Status |
|---|---|---|
| <claim> | [#NNNN](url) | Filed / Call-sourced / Analyst-derived |

**Notes.**
- RPO / revenue backlog is contracted backlog, not recognized revenue.
- Balances not directly comparable: fiscal calendars and disclosure labels differ; some issuers do not disclose.
- <period-end mismatches; any derived figures; non-disclosers>
```

## Stopping Points

- **Step 1** — STOP if a company is genuinely ambiguous.
- **Step 6** — STOP for the output-style, citation, and provenance self-check before presenting.
- **On delivery** — present the chart and tables only (plus the terse `Notes` bullets). Do not chain into additional peer sets or periods unless asked.

## Generalization notes

- The peer set is arbitrary — any cloud vendors or hyperscalers. The private-lab counterparty is **inferred per company** from filing/transcript hits, not hard-coded.
- The disclosure label is issuer-specific; always adapt the Step-3 `topicSearchQuery` rather than assuming the literal string "RPO".
- For a single-company deep dive (one issuer's RPO + lab share) use the companion single-name RPO exposure workflow; this skill is the multi-company comparison layer.
