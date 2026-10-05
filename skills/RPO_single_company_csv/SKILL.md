---
name: Company RPO Extraction Skill
description: Replicate the `Oracle RPO.csv` format for any company by extracting Remaining Performance Obligations (RPO) from SEC filings. **The only output is the `<Company> RPO.csv` file** — no summary, tables, or commentary in the response.
---

## Output Format
CSV with these exact columns (one row per 10-K/10-Q, newest first):

```
CALENDARYEAR,CALENDARQUARTER,FISCALYEAR,FISCALQUARTER,FILINGDATE,PERIODENDDATE,COMPANYID,COMPANYNAME,FORMTYPE,RPO ($M),Excerpt
```

- Dates: `DD/MM/YYYY`
- `RPO ($M)`: value in millions
- `Excerpt`: verbatim RPO sentence(s) from the filing, wrapped in double quotes

## Steps

### 1. Resolve companyId
Use the Pronto MCP `getCompanies` function with the company name/ticker to get
`companyId`.

### 2. Get financial periods
Call `GET_FIN_PERIODS(companyId)` to obtain calendar/fiscal period mapping and
period-end dates. If the function is unavailable, derive periods from the
company's fiscal calendar (e.g., a Dec-31 fiscal year means fiscal = calendar).

### 3. Extract RPO from filings
- `getDocuments` (corpus: `SEC Filings`, types `10-K`/`10-Q`) to list filings
  with filing dates and transcript IDs.
- `searchSentences` / `getDocumentSummary` scoped to each filing with query
  `remaining performance obligations` / `revenue backlog`.
- **Validate every figure ONLY against the official SEC EDGAR filing**
  (`https://www.sec.gov/edgar/`) — the source 10-Q/10-K on EDGAR is the sole
  source of truth. Do NOT validate against earnings call transcripts, and do
  NOT use news, blogs, or third-party aggregators. Tool/summarizer output is a
  lead only; it can cross-contaminate figures across periods and must be
  confirmed on EDGAR.
- Align each filing's period-end date to the correct row from Step 2.

### 4. Consolidate & write CSV
- Convert reported billions to millions for `RPO ($M)`.
- Omit any period whose figure cannot be verified verbatim rather than guessing.
- Surface `<Company> RPO.csv` as an asset.

## Rules
- **Output = the CSV file only.** No prose, tables, or explanation in the reply.
- Never fabricate an RPO value; skip unverifiable periods.
- Match `Oracle RPO.csv` column order and formatting exactly.
