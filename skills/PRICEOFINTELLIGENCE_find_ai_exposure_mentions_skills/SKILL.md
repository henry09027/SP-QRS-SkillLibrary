---
name: find_ai_exposure_mentions_skills
description: "Produces a sourced research note on a company's Remaining Performance Obligation (RPO) backlog and its concentration in private AI labs (OpenAI, Anthropic, and peers) by cross-referencing SEC filings with earnings-call transcripts via the Pronto MCP. Use for RPO disclosure analysis, backlog concentration, and AI-lab counterparty exposure at hyperscalers and cloud vendors. Triggers: RPO exposure, RPO concentration, remaining performance obligation, OpenAI backlog, Anthropic backlog, AI lab exposure, backlog footnote analysis."
---

# RPO & Private AI-Lab Exposure Note

## Purpose

For a given company and fiscal quarter, pair the SEC-filed RPO balance with earnings-call commentary on how much of that backlog sits with a private AI lab, then reconstruct the lab's dollar share.

**Framing to state in every note:** RPO is a *leading indicator* of contracted backlog, not recognized revenue. Concentration figures are frequently analyst-derived from disclosed contract values divided by total RPO — never present a derived figure as a company disclosure.

## Output Style (mandatory)

- Do not write an opening framing paragraph or a closing summary paragraph. No "Summary." lead-in, no "Read-through" narrative wrap-up. Keep the output concise and professional.
- The comparability caveat and any coverage gaps are conveyed via the chart footnote and a short bulleted `Notes` block — not a prose essay.
- Allowed prose: section headers, table/column headers, terse (<1 line) bullets in `Notes`, and STOP checkpoints. Nothing else.

## Inputs

| Input | Description | Example |
|---|---|---|
| Company | Name or ticker | "Microsoft", "MSFT", "Oracle" |
| Quarter | Fiscal quarter + year | "Q4 FY2026", "Q2 2026" |

If either input is missing, request it once, then proceed. No other inputs are required.

## Prerequisites

- Pronto MCP tools: `getCompanies`, `searchSentences` (required); `getDocuments`, `getSentenceContext` (optional).
- All figures and quotations must originate from Pronto results. Fabricated RPO balances, percentages, or contract values are a hard failure.

---

## Citation rule (mandatory)

Every statement derived from a filing or transcript must be immediately followed by a visible, clickable inline source link, reproduced verbatim from the Pronto result field.

**Required format:** literal Markdown link syntax — `[MARKER](URL)` — with a non-empty marker and no space between `]` and `(`.

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
| `<cite>[#1170](url)</cite>` | Tag suppresses rendering |

**Placement and integrity:**

- One link per source, positioned at the end of the sentence it supports (before the terminal period is acceptable).
- Where multiple sources support one claim, attach all of them. Never collapse several claims under a single trailing citation.
- Copy markers and URLs verbatim. Never invent, reword, shorten, renumber, or construct a URL.
- Never substitute a document-title link for a sentence-level link, or the reverse.
- Verification-table cells must contain the same clickable `[MARKER](url)`, not a bare marker or a description.

**Reference pattern:**

> Total RPO was **$638 billion** as of May 31, 2026, up from $138 billion a year earlier [#1170](https://spglobal.prontonlp.com/#/ref/$SENTID_SEC000262329667-1170), of which "approximately 12%" is expected to convert within twelve months [#1396](https://spglobal.prontonlp.com/#/ref/$SENTID_SEC000262329667-1396).

---

## Workflow

### Step 1 — Resolve the company

Call `getCompanies` with `companyNameOrTicker` set to the company input.

- Single match: retain `companyId` and continue.
- Multiple matches: present ticker, exchange, sector, and market cap, then **stop** and ask which entity is intended.

### Step 2 — Derive the date window

Map the quarter to its period-end and set `dateRange` to bracket both the filing and the call:

- `gte` ≈ two quarters before the period-end
- `lte` = current date, or one month after the expected report date

Account for non-December fiscal year-ends (Microsoft, Oracle, NVIDIA, and others). When the exact period-end is uncertain, widen the window rather than risk missing the disclosure.

### Step 3 — Retrieve the filed RPO balance

Call `searchSentences`:

| Parameter | Value |
|---|---|
| `companiesIds` | `[companyId]` |
| `corpus` | `["SEC Filings"]` |
| `dateRange` | from Step 2 |
| `size` | 30 |
| `topicSearchQuery` | "remaining performance obligation commercial contracted revenue not yet recognized" |

Extract for the target period-end, retaining each result's inline source link:

- Total company RPO, and the commercial RPO portion where disclosed
- Weighted-average duration
- Percentage expected to be recognized within twelve months
- The ASC 606 definition sentence (useful framing quotation)

**Fallback ladder** if the target period-end is absent from the corpus:

1. Fall back to the closest prior filing and label it as such.
2. State the coverage gap explicitly in the note.
3. Flag that the target-quarter figure may be available only from the call (Step 4).

### Step 4 — Retrieve RPO and AI-lab concentration commentary

Call `searchSentences`:

| Parameter | Value |
|---|---|
| `companiesIds` | `[companyId]` |
| `corpus` | `["S&P Transcripts"]` |
| `dateRange` | from Step 2 |
| `size` | 40 |
| `topicSearchQuery` | "OpenAI Anthropic share of commercial remaining performance obligation RPO backlog" |

Capture, retaining each result's inline source link:

- **Direct disclosure** of a lab's share of RPO or backlog (e.g., "approximately 45% of our commercial RPO").
- **Growth-rate disclosure** permitting derivation of the share when the percentage is not restated (e.g., "RPO increased X% excluding [lab]").
- Contract-value mentions, contract duration, and any volatility or lumpiness caveats.
- Any qualitative language on private AI-lab counterparties.

Where a second counterparty is material (e.g., Amazon → OpenAI and Anthropic), rerun the query naming that lab.

### Step 5 — Assemble and self-check

Draft the note per the **Output** template, then complete both checks before presenting:

1. **Citation check.** Scan the rendered draft line by line. Every filing- or transcript-derived sentence must carry an adjacent, non-empty, clickable `[MARKER](url)`. Remediate any missing, empty-bracketed, un-URL'd, backtick-wrapped, or bare-marker citation.
2. **Provenance check.** Confirm each fact is correctly classified as filed, stated-on-call, or analyst-derived, and that no derived figure is presented as a disclosure.

A note containing an uncited or non-rendering sourced fact is incomplete and must not be delivered.

---

## Output template

```
## 1. Filed RPO balance (SEC filings)
- Total and commercial RPO, weighted-average duration, 12-month recognition
  percentage, ASC 606 definition — each with its inline source link.
- Coverage note where the target-quarter filing is absent from the corpus.

## 2. Management commentary (earnings-call transcripts)
- Direct percentage disclosure, or the growth-rate disclosure used to derive it —
  each with its inline source link.
- Backlog-concentration calculation: disclosed % × commercial RPO ≈ $ share.
  Show the arithmetic and label the result as derived.

## 3. Verification status
| Claim | Pronto source | Status |
|---|---|---|
| <claim> | [#NNNN](url) | ✅ Filed / ◑ Call-sourced / ◑ Analyst-derived |

**Coverage note.** <Anything absent from the corpus, and its effect on the analysis.>
```

## Stopping points

- **Step 1** — halt if the company is ambiguous.
- **Step 5** — halt for citation and provenance self-check before presenting.
- **On delivery** — present the note only. Do not chain into additional companies or quarters unless asked.

## Reusability

The template applies across hyperscalers and cloud vendors. The private-lab counterparty is inferred per company from transcript hits rather than hard-coded.
