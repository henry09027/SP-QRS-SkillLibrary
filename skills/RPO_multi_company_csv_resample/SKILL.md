---
name: Multi-Company RPO Extraction & Month-End Merge
description: Extract Remaining Performance Obligations (RPO) from SEC filings for a set of companies, writing one `<Company> RPO.csv` per company, then merge them into a single month-end resampled file keyed on `FILINGDATE`. **The only output is the CSV files** — no summary, tables, or commentary in the response.
---

## Output

Per company — `<Company> RPO.csv` (one row per 10-K/10-Q, newest first):

    CALENDARYEAR,CALENDARQUARTER,FISCALYEAR,FISCALQUARTER,FILINGDATE,PERIODENDDATE,COMPANYID,COMPANYNAME,FORMTYPE,RPO ($M),Excerpt

Merged — one file with columns:

    MONTHEND,<COMPANY_1>,<COMPANY_2>,...

- Dates: `DD/MM/YYYY`
- `RPO ($M)`: value in millions
- `Excerpt`: verbatim RPO sentence(s) from the filing, in double quotes
- Merged values in `$M`; one column per company, named in uppercase.

## Steps

### 1. Resolve each company
Use `getCompanies` (name/ticker) to get `companyId` for every company.

### 2. Get financial periods
Call `GET_FIN_PERIODS(companyId)` for the calendar/fiscal period mapping and period-end dates. If unavailable, derive from the fiscal calendar (e.g. Dec-31 fiscal = calendar; Microsoft Jun-30; Oracle May-31).

### 3. Extract RPO per filing
- `getDocuments` (corpus `SEC Filings`, types `10-K`/`10-Q`) to list filings with filing dates and transcript IDs.
- `searchSentences` scoped to each filing with queries like `remaining performance obligations` / `revenue backlog`.
- **Validate every figure ONLY against the official SEC EDGAR filing** (`https://www.sec.gov/edgar/`). Tool/summarizer output is a lead only; confirm the number verbatim on EDGAR. Do NOT use earnings calls, news, or aggregators as the source of truth.
- Align each filing's period-end date to the correct period from Step 2.
- **Note the metric differs by company** and must be the company's own RPO-equivalent disclosure (e.g. Oracle/Microsoft "remaining performance obligations"; Alphabet "revenue backlog"; Amazon "commitments not yet recognized for contracts with original terms exceeding one year"). Record the exact label in `Excerpt`.

### 4. Write per-company CSVs
- Convert reported billions to millions for `RPO ($M)`.
- Omit any period whose figure cannot be verified verbatim — never guess.
- Match the column order and formatting above exactly.

### 5. Merge (month-end resample by FILINGDATE)
- For each company, index RPO by `FILINGDATE`, resample to month-end (`ME`), taking the last filing in a month.
- Build a monthly month-end index from the earliest to latest filing date across all companies; reindex each company onto it.
- **Forward-fill** so each `MONTHEND` carries the most recently filed RPO. Leave months before a company's first filing blank.
- Write the merged `MONTHEND,<COMPANIES...>` CSV.

## Rules
- **Output = the CSV files only.** No prose, tables, or explanation in the reply.
- Never fabricate an RPO value; skip unverifiable periods.
- Companies have different RPO disclosure histories and start dates; blanks before first disclosure are expected.
