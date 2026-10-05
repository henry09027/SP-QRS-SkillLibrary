---
name: sec-off-balance-sheet-liabilities-hyper5
description: "Extract ALL off-balance-sheet commitments and contingencies (leases not yet commenced, purchase commitments, guarantees/backstops, contingent liabilities) for the Hyper 5 (Alphabet, Meta, Microsoft, Amazon, Oracle) from their most recent periodic filings, then produce a markdown report, raw-data CSV, summary-stats file, and a cross-sectional visualization. Primary source is the Pronto MCP SEC Filings corpus, with a MANDATORY cross-check on the official SEC website (EDGAR) for any missing/stale figure. Use when: analyzing off-balance-sheet liabilities, data-center/AI lease build-out, uncommenced leases, purchase commitments, RVG/backstop guarantees, or contingent liabilities for the megacap tech names. Triggers: off-balance-sheet, off balance sheet, leases not yet commenced, uncommenced leases, purchase commitments, residual value guarantee, RVG, lease backstop, contingent liabilities, Hyper 5, HYPER 5 extraction."
---

# SEC FILINGS: HYPER 5 OFF-BALANCE-SHEET LIABILITIES EXTRACTION (Enhanced)

## OBJECTIVE

Extract ALL off-balance-sheet commitments and contingencies from the Hyper 5's most recent periodic filings. Focus on four categories:
1. Leases Not Yet Commenced
2. Purchase Commitments
3. Guarantees & Lease Backstops
4. Contingent Liabilities

Generate a markdown report with detailed tables for each category and each company, a raw-data CSV, a summary-stats block, AND a cross-sectional visualization of the Hyper 5. Every figure carries a source marker AND a source-basis note (which filing, filed when, corpus vs. SEC/EDGAR).

**DISPLAY IN-SESSION ONLY — do NOT save to the workspace.** Every deliverable (report, CSV, summary stats, and the chart) is rendered directly in the chat session for the user to read/copy. Do NOT write deliverables to `/workspace/`, do NOT emit `<asset>` tags, and do NOT tell the user a file was saved. The markdown report and its tables are printed inline; the CSV and summary-stats are printed as fenced code blocks the user can copy; the chart is shown in-session (see Part 3). The sandbox `/tmp` may still be used purely as scratch space to render the chart image, but nothing is persisted to the workspace.

**This skill stores NO figures.** All values are pulled live from primary filings at extraction time because they roll forward every quarter.

---

## DATA SOURCE — READ FIRST (MANDATORY)

Values are extracted from primary filings via two sources, in this order:

1. **Pronto MCP SEC Filings corpus** — the primary retrieval path. Set `corpus:["SEC Filings"]` on every Pronto MCP call. Every figure must come from a document returned by the Pronto MCP tools and must carry the inline citation marker the tool returns (`[MARKER](url)` / `$SENTID_...` / `$DOCID_...`). Cite the Pronto MCP website (`https://spglobal.prontonlp.com/...`) as the source of record for corpus-sourced figures.
2. **The official SEC website / EDGAR** (`sec.gov`, `data.sec.gov`) — the authoritative cross-check and the fallback whenever the corpus is stale or a figure is missing.

**MANDATORY SEC-website cross-check.** Do NOT rely on the corpus alone. For every company, verify the extracted figures against the official SEC filing before finalizing, and specifically search the SEC website whenever:
- a figure is missing / would otherwise be marked `n/d`, or
- the corpus's latest document predates the company's most recent fiscal period-end (a newer filing may exist).

The sandbox may be blocked from reaching `data.sec.gov` directly; use `Web_Search` with `allowed_domains:["sec.gov"]` to retrieve the filing text (e.g. the company's `*-YYYYMMDD.htm` 10-Q/10-K on EDGAR). Reputable third-party readers (Bloomberg, WSJ, Reuters) may only CONFIRM a primary-source number, never originate one. Never substitute a web estimate for a number that is genuinely absent from disclosure — mark it `n/d` (never `0`, never estimated). Record the source basis for every figure.

**AMAZON — MUST have a leases-not-yet-commenced value.** Amazon DOES disclose an aggregate uncommenced-lease figure in its 10-Q, even if a first-pass corpus search suggests otherwise. Do not finalize Amazon as `n/d` for Category 1: cross-check the official Amazon 10-Q on the SEC website (`amzn-YYYYMMDD.htm`, CIK 0001018724) and capture the disclosed aggregate. Only if the primary filing genuinely omits it may it remain `n/d`.

### Pronto MCP tools

| Purpose | Tool | Key arguments |
|---------|------|---------------|
| Resolve each company to an ID | `getCompanies` | `companyNameOrTicker`, `corpus:["SEC Filings"]` |
| Confirm valid document types/sections | `getFilterOptions` | `filterNames:["documentTypes","sections"]` |
| Find the most recent 10-Q/10-K | `getDocuments` | `companiesIds`, `corpus:["SEC Filings"]`, `documentTypes:["10-Q","10-K"]`, `sortOrder:"desc"`, `excludeFutureDocuments:true`, `size:5` |
| Pull the disclosure sentences (primary) | `searchSentences` | `topicSearchQuery`, `transcriptsIds`, `corpus:["SEC Filings"]`, `size:~40`, `similarityThreshold:"Low"` |
| Summarize a category within a filing | `getDocumentSummary` | `transcriptsIds`, `focus` |
| Read a full section verbatim | `readDocuments` | `transcriptsIds`, `sections` |
| Widen context around a hit | `getSentenceContext` | `sentenceIds`, `beforeSentence`/`afterSentence` |
| SEC/EDGAR cross-check & fallback | `Web_Search` | `allowed_domains:["sec.gov"]` |

### Standard workflow (per company)

1. **Resolve company** -> `getCompanies(companyNameOrTicker, corpus:["SEC Filings"])`. If several matches, pick the primary issuer (largest market cap / matching CIK below). Capture the `companyId`.
2. **Find the latest filing** -> `getFilterOptions(["documentTypes"])` once, then `getDocuments(companiesIds:[id], corpus:["SEC Filings"], documentTypes:["10-Q","10-K"], sortOrder:"desc", excludeFutureDocuments:true)`. Take the single most recent 10-Q, OR the 10-K if it is newer. Capture `transcriptId`, document type, period/as-of date, and filing date. **Flag any company whose latest corpus doc predates its most recent fiscal period-end — that company needs a SEC-website check for a newer filing.**
3. **Extract each category** with `searchSentences` scoped to that `transcriptId` (see per-category `topicSearchQuery` phrases). Raise `size` (~40) and lower `similarityThreshold` to "Low" for a thorough single-document read. Use `getDocumentSummary` for a narrative pass and `readDocuments`/`getSentenceContext` when a number needs context or a verbatim excerpt.
4. **Record the citation marker + source basis** next to every figure.
5. **Cross-check on the SEC website (EDGAR)** — mandatorily for every `n/d`, for Amazon's Category 1, and for any company whose corpus doc may be stale. Replace stale corpus numbers with the newest EDGAR figure; note the source basis.
6. Repeat for all five companies, then compile report / CSV / summary / visualization.

**Verify recency:** confirm each figure is tied to the filing's period/as-of date. If the most recent filing predates a known subsequent event, capture the subsequent-event addition separately (Category 1, Field 5).

---

## TARGET COMPANIES & FILINGS

Resolve each via `getCompanies`; CIKs disambiguate the correct issuer.

| Company | CIK | Filing |
|---------|-----|--------|
| **Alphabet Inc.** | 1652044 | Most recent 10-Q |
| **Meta Platforms, Inc.** | 1326801 | Most recent 10-Q |
| **Microsoft Corporation** | 789019 | Most recent 10-Q OR 10-K — whichever is newer (check for a just-filed fiscal-year-end 10-K) |
| **Amazon.com, Inc.** | 1018724 | Most recent 10-Q (MUST capture uncommenced-lease aggregate; SEC-website cross-check) |
| **Oracle Corporation** | 1341439 | Most recent 10-Q OR 10-K — whichever is newer |

---

## EXTRACTION CATEGORY 1: LEASES NOT YET COMMENCED (ASC 842-20-50-3)

### How to find it in the corpus
Run `searchSentences` scoped to the company's latest `transcriptId` (`corpus:["SEC Filings"]`). Also try `getDocumentSummary(focus:"leases not yet commenced")`. Suggested `topicSearchQuery` values (run several short ones, not one long string):
- "leases not yet commenced"
- "future lease payments not yet recorded"
- "operating and finance leases not commenced"
- "data center leases signed not commenced"

If a Commitments/Leases section exists, confirm section names via `getFilterOptions(["sections"])` and use `readDocuments(sections:[...])`. **If the aggregate is not surfaced by the corpus (notably for Amazon), search the SEC website: `Web_Search(allowed_domains:["sec.gov"], query:"<company> <accession/htm> leases not yet commenced future payments")`.**

### Field 1: Total Uncommenced Lease Amount
Capture: exact dollar amount as disclosed; currency (if not USD); discounted vs. undiscounted; ALL amounts if several are given. Format:
```
Total Uncommenced Leases: $[EXACT AMOUNT] [as of Filing Date]
Discount Status: [Undiscounted / Discounted to PV / Not specified]
Source basis: [10-Q / 10-K, as-of date, filed date; corpus marker or SEC/EDGAR]
```

### Field 2: Asset Type / Facility Description
Primary asset class (data centers, offices, infrastructure, warehouses); geographic concentration; strategic purpose; breakdown by asset type if provided.

### Field 3: Commencement Window
Fiscal/calendar year range (verbatim); exact dates if specified; phased schedule; latest commencement date.

### Field 4: Lease Terms
Duration in years; range if terms vary; renewal options.

### Field 5: Recent Amendments or Additions (Post-Balance Sheet)
Leases signed AFTER the balance-sheet date but BEFORE filing (Subsequent Events / Recent Commitments). Capture amount + commencement date. Search `topicSearchQuery:"subsequent events leases"`.

### Field 6: Verbatim Footnote Excerpt
The official footnote text (exact quote, no paraphrase) with its citation marker; include footnote/section reference. Use `getSentenceContext` to widen.

---

## EXTRACTION CATEGORY 2: PURCHASE COMMITMENTS (CAPITAL & OPERATING)

### How to find it
`searchSentences` on the latest `transcriptId` with `topicSearchQuery` such as: "unconditional purchase obligations", "purchase commitments", "non-cancelable contractual commitments", "cloud capacity commitments" / "supply and energy commitments". Also `getDocumentSummary(focus:"purchase commitments")`. Capture amounts and timing bands as disclosed. **Distinguish what the company OWES (purchase obligations) from remaining performance obligations / backlog that customers owe the company — do not conflate.** Cross-check any missing total on the SEC website.

### Keywords (as they appear in filings)
"unconditional purchase obligations", "capital commitments", "purchase agreements", "supply commitments", "minimum purchase requirements".

### Purchase Commitments Table
| Commitment Type | Amount ($B) | Timing | Notes | Source |
|-----------------|-------------|--------|-------|--------|
| [Type] | $[B] | [When due] | [Description] | [MARKER / SEC] |

Look for: semiconductor/chip commitments; cloud infrastructure/server purchases; real-estate development agreements; equipment financing; power/energy contracts. Compute `Total Purchase Commitments: $[SUM in B]`.

---

## EXTRACTION CATEGORY 3: GUARANTEES & LEASE BACKSTOPS

### How to find it
`searchSentences` with `topicSearchQuery` such as: "residual value guarantee", "lease backstop" / "credit support", "maximum exposure to loss" (VIE/JV disclosures), "indemnification guarantee". Also `getDocumentSummary(focus:"residual value guarantee maximum exposure")` — these guarantees often sit in the non-marketable equity / VIE note rather than the commitments note.

### Keywords
"guarantee", "residual value guarantee", "lease backstop", "indemnification", "credit support", "liquidity guarantee", "fair value guarantee".

### Guarantee Table
| Guarantor | Guarantee Type | Max Amount ($B) | Trigger/Term | Status | Source |
|-----------|----------------|-----------------|--------------|--------|--------|
| [Name/Entity] | [Type] | $[B] | [When triggered] | [Active/Contingent] | [MARKER / SEC] |

Special attention: RVG structures (e.g. data-center venture residual value guarantees); third-party lease / JV / SPV guarantees; supplier financial guarantees or backstops; credit-derivative data-center backstops. **Attribute every guarantee/RVG to the issuer whose filing discloses it — never move a figure across companies.**

---

## EXTRACTION CATEGORY 4: CONTINGENT LIABILITIES

### How to find it
`searchSentences` with `topicSearchQuery` such as: "legal proceedings" / "litigation contingency", "loss contingency reasonably possible", "uncertain tax positions" / "income tax contingencies", "environmental remediation". Also `getDocumentSummary(focus:"contingencies and legal proceedings")`.

### What to Capture
Litigation with potential material obligation (>$100M); environmental remediation; tax contingencies (>$100M range); warranty/recall; regulatory/compliance contingencies. Format:
```
Contingency Type: [Type]
Estimated Range: $[X-Y million]
Probability: [Remote / Reasonably possible / Probable]
Status: [Active dispute / Settlement pending / Other]
Source: [MARKER / SEC]
```

---

## EXTRACTION CATEGORY 5: SUMMARY METRICS

Per company, calculate/extract:
- Total Off-Balance-Sheet Liabilities = Leases + Purchase Commitments + Guarantees (+ Contingent where quantified)
- Percentage breakdown of each category
- Leases as % of total off-balance-sheet
- Leases as % of reported balance-sheet debt (balance-sheet debt also from the filing)

---

## PART 1 OUTPUT: MARKDOWN REPORT WITH TABLES

### Report Structure Template
```markdown
# Hyper 5 Off-Balance-Sheet Liabilities Analysis
**Source: most recent 10-Q/10-K per company — Pronto SEC Filings corpus, cross-checked on SEC/EDGAR**

## Executive Summary: Total Off-Balance-Sheet Liabilities

| Company | Leases ($B) | Source basis (leases) | Purchase Commitments ($B) | Guarantees ($B) | Contingent ($B) | Total ($B) |
|---------|-------------|-----------------------|--------------------------|-----------------|-----------------|-----------|
| Alphabet | $[X] | [filing] | $[X] | $[X] | $[X] | $[X] |
| Meta | $[X] | [filing] | $[X] | $[X] | $[X] | $[X] |
| Microsoft | $[X] | [filing] | $[X] | $[X] | $[X] | $[X] |
| Amazon | $[X] | [filing] | $[X] | $[X] | $[X] | $[X] |
| Oracle | $[X] | [filing] | $[X] | $[X] | $[X] | $[X] |
| **TOTAL** | **$[X]** | | **$[X]** | **$[X]** | **$[X]** | **$[X]** |

*n/d = not disclosed / not aggregated as a single figure in the filing. Totals are NOT apples-to-apples where companies disclose fewer discrete aggregates.*

## Section 1: Leases Not Yet Commenced
### COMPANY: [Name]
**Filing Details:** Document [10-Q/10-K] · Date [filed] · Fiscal Period [as-of] · Source [Pronto doc link + MARKER, and/or SEC/EDGAR htm]
**Total Uncommenced Lease Obligation:** $[X]B ([undiscounted]); Primary Asset [..]; Status [Non-cancelable binding commitments]
**Commencement Schedule:** Window [..]; Earliest [..]; Latest [..]; Lease Terms [X-Y yrs]
**Recent Additions (Subsequent Events):** [Amount + date, or "None disclosed"]
**Disclosure (Verbatim):** "[excerpt]" [MARKER / SEC]
[Repeat for all 5]

## Section 2: Purchase Commitments   [per-company table + verbatim excerpt]
## Section 3: Guarantees & Lease Backstops   [per-company table + significant-guarantee detail]
## Section 4: Contingent Liabilities   [table]
## Section 5: Lease Focus — Detailed Breakdown
  - Total Hyper 5 uncommenced leases; est. PV; time horizon; heaviest commencement window; primary asset class
  - Lease commencement timeline table
  - Company lease ranking by size (with % of total and source basis)
## Section 6: Key Findings & Observations
## Source Documentation   [per-company Pronto $DOCID link + SEC/EDGAR htm where cross-checked]
```
Populate strictly from freshly-extracted figures — no illustrative/carried-over values.

---

## PART 2 OUTPUT: RAW DATA TABLE (CSV)

Print the CSV **in-session as a fenced ```csv code block** so the user can copy it — do NOT write it to the workspace. One row per company. Include a `Source_Basis` column and computed `Total_OBS_Disclosed_B`. Column layout (example header only, not data):
```csv
Company,Leases_Undiscounted_B,Source_Basis,Lease_Asset_Type,Commencement_Start,Commencement_End,Lease_Terms_Years,Subsequent_Lease_Addition_B,Purchase_Commitments_B,Guarantees_B,Contingent_B,Total_OBS_Disclosed_B
```

---

## PART 3 OUTPUT: CROSS-SECTIONAL VISUALIZATION (MANDATORY)

After the report and CSV are built, produce a single cross-sectional comparison figure using the extracted figures. Use `matplotlib` in the sandbox, rendering the PNG to `/tmp` scratch space only.

**Display the chart in-session — do NOT save it to the workspace and do NOT emit an `<asset>` tag.** Render the image to `/tmp/hyper5/hyper5_cross_section.png` purely as scratch, then surface it directly in the chat session so the user sees it inline. Nothing is persisted to `/workspace/`.

**Rules:**
- Feed ONLY freshly-extracted values. Pass `None` for any figure not disclosed / not aggregated — the function renders these as "n/d" (not disclosed), never `0` or an estimate.
- Source note on the figure must name the per-company filing basis (e.g. "Microsoft = FY2026 10-K; others = Q2 2026 / Q1 FY2027 10-Q") and credit the SEC filings / Pronto SEC corpus.

### Reusable function (inline — render to `/tmp` scratch, then display in-session)
```python
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import numpy as np


def plot_hyper5_cross_section(leases, purchase, guarant, conting, addition,
                              out_path="/tmp/hyper5/hyper5_cross_section.png",
                              companies=("Microsoft", "Oracle", "Meta", "Amazon", "Alphabet")):
    """Render a 4-panel cross-sectional comparison of Hyper 5 off-balance-sheet liabilities.

    Each argument is a dict keyed by company name -> value in $B (float), or None
    when the figure is NOT disclosed / not aggregated in that company's filing.
    None is drawn as "n/d" (never 0, never estimated).
      leases   : leases not yet commenced (undiscounted, $B)
      purchase : purchase commitments ($B)
      guarant  : guarantees / lease backstops ($B)
      conting  : contingent liabilities accrued/exposure ($B)
      addition : subsequent-event lease additions ($B), 0 if none
    Returns the output path.
    """
    plt.rcParams.update({"font.size": 10, "axes.titlesize": 12, "axes.titleweight": "bold",
                         "figure.facecolor": "white", "axes.facecolor": "#fbfbfd"})
    companies = list(companies)
    colors = {"Oracle": "#C74634", "Meta": "#0866FF", "Microsoft": "#00A4EF",
              "Alphabet": "#EA4335", "Amazon": "#FF9900"}
    ccol = [colors.get(c, "#555555") for c in companies]
    x = np.arange(len(companies))

    fig = plt.figure(figsize=(15, 10))
    gs = fig.add_gridspec(2, 2, hspace=0.38, wspace=0.22)

    def barlabels(ax, xs, vals, fmt="${:,.0f}B", na="n/d"):
        for xi, v in zip(xs, vals):
            if v is None or (isinstance(v, float) and np.isnan(v)):
                ax.text(xi, 2, na, ha="center", va="bottom", fontsize=8, color="#888")
            else:
                ax.text(xi, v, fmt.format(v), ha="center", va="bottom", fontsize=8.5, fontweight="bold")

    # Panel A: leases not yet commenced (+ subsequent-event additions stacked/hatched)
    axA = fig.add_subplot(gs[0, 0])
    base = [leases[c] or 0 for c in companies]
    add = [addition.get(c, 0) or 0 for c in companies]
    axA.bar(x, base, color=ccol, edgecolor="white")
    axA.bar(x, add, bottom=base, color=ccol, alpha=0.4, hatch="//", edgecolor="white",
            label="Subsequent-event addition")
    for xi, c in zip(x, companies):
        if leases[c] is None:
            axA.text(xi, 4, "n/d in filing", ha="center", va="bottom", fontsize=8, color="#888")
        else:
            tot = (leases[c] or 0) + (addition.get(c, 0) or 0)
            lbl = f"${leases[c]:,.0f}B" + (f"\n(+${addition[c]:,.0f}B)" if addition.get(c) else "")
            axA.text(xi, tot, lbl, ha="center", va="bottom", fontsize=8.5, fontweight="bold")
    axA.set_title("A. Leases Not Yet Commenced (undiscounted, $B)")
    axA.set_xticks(x); axA.set_xticklabels(companies)
    axA.set_ylim(0, max([b + a for b, a in zip(base, add)] + [1]) * 1.25)
    axA.legend(loc="upper right", fontsize=8, frameon=False)
    axA.spines[["top", "right"]].set_visible(False)

    # Panel B: purchase commitments
    axB = fig.add_subplot(gs[0, 1])
    pv = [purchase[c] for c in companies]
    axB.bar(x, [v or 0 for v in pv], color=ccol, edgecolor="white")
    barlabels(axB, x, pv, na="not agg. in filing")
    axB.set_title("B. Purchase Commitments ($B)")
    axB.set_xticks(x); axB.set_xticklabels(companies)
    axB.set_ylim(0, max([v or 0 for v in pv] + [1]) * 1.2)
    axB.spines[["top", "right"]].set_visible(False)

    # Panel C: guarantees & contingent (grouped)
    axC = fig.add_subplot(gs[1, 0])
    w = 0.38
    gv = [guarant[c] for c in companies]; cv = [conting[c] for c in companies]
    axC.bar(x - w / 2, [v or 0 for v in gv], w, color="#6C5CE7", edgecolor="white", label="Guarantees / backstops")
    axC.bar(x + w / 2, [v or 0 for v in cv], w, color="#00B894", edgecolor="white", label="Contingent (accrued/exposure)")
    for xi, v in zip(x - w / 2, gv):
        axC.text(xi, (v or 0), "" if v is None else f"${v:,.1f}", ha="center", va="bottom", fontsize=7.5, fontweight="bold")
    for xi, v in zip(x + w / 2, cv):
        axC.text(xi, (v or 0), "" if v is None else f"${v:,.1f}", ha="center", va="bottom", fontsize=7.5, fontweight="bold")
    axC.set_title("C. Guarantees & Contingent Liabilities ($B)")
    axC.set_xticks(x); axC.set_xticklabels(companies)
    axC.set_ylim(0, max([v or 0 for v in gv + cv] + [1]) * 1.25)
    axC.legend(loc="upper right", fontsize=8, frameon=False)
    axC.spines[["top", "right"]].set_visible(False)

    # Panel D: total disclosed off-balance-sheet stack
    axD = fig.add_subplot(gs[1, 1])
    cats = [("Leases", leases, "#0984E3"), ("+Subseq. lease", addition, "#74B9FF"),
            ("Purchase", purchase, "#E17055"), ("Guarantees", guarant, "#6C5CE7"),
            ("Contingent", conting, "#00B894")]
    bottoms = np.zeros(len(companies))
    for name, d, col in cats:
        vals = np.array([(d.get(c) or 0) for c in companies], dtype=float)
        axD.bar(x, vals, bottom=bottoms, color=col, edgecolor="white", label=name)
        bottoms += vals
    for xi, tot in zip(x, bottoms):
        axD.text(xi, tot, f"${tot:,.0f}B", ha="center", va="bottom", fontsize=8.5, fontweight="bold")
    axD.set_title("D. Total Disclosed Off-Balance-Sheet Stack ($B)")
    axD.set_xticks(x); axD.set_xticklabels(companies)
    axD.set_ylim(0, max(bottoms.tolist() + [1]) * 1.2)
    axD.legend(loc="upper right", fontsize=7.5, frameon=False)
    axD.spines[["top", "right"]].set_visible(False)

    fig.suptitle("Hyper 5 Off-Balance-Sheet Liabilities — Cross-Sectional Comparison",
                 fontsize=16, fontweight="bold", y=0.975)
    fig.text(0.5, 0.005,
             "Source: most recent primary SEC filing per company (Pronto SEC corpus, cross-checked on SEC/EDGAR). "
             "n/d = not disclosed / not aggregated as a single figure in the filing.",
             ha="center", fontsize=8, color="#666")
    fig.savefig(out_path, dpi=150, bbox_inches="tight")
    return out_path


if __name__ == "__main__":
    # Replace ALL values with figures freshly extracted from the filings (Pronto + SEC/EDGAR).
    # Use None wherever the filing does not disclose / aggregate the figure. Do NOT hardcode.
    leases   = {"Microsoft": None, "Oracle": None, "Meta": None, "Amazon": None, "Alphabet": None}
    purchase = {"Microsoft": None, "Oracle": None, "Meta": None, "Amazon": None, "Alphabet": None}
    guarant  = {"Microsoft": None, "Oracle": None, "Meta": None, "Amazon": None, "Alphabet": None}
    conting  = {"Microsoft": None, "Oracle": None, "Meta": None, "Amazon": None, "Alphabet": None}
    addition = {"Microsoft": 0, "Oracle": 0, "Meta": 0, "Amazon": 0, "Alphabet": 0}
    print(plot_hyper5_cross_section(leases, purchase, guarant, conting, addition))
```

**Panels produced:** A. leases not yet commenced (subsequent-event additions as a hatched cap); B. purchase commitments; C. guarantees/backstops vs. contingent (grouped); D. total disclosed off-balance-sheet stack (composition per company).

**Interpretation note to include in the writeup:** `n/d` bars are disclosure gaps, not zeros, so Panel D totals are NOT apples-to-apples across companies that disclose fewer discrete aggregates. State this caveat whenever the stacked totals are compared.

---

## KNOWN-GOOD REFERENCE DATASET (as of 2026-09-22)

This is the validated dataset from the reference run. Use it to sanity-check a fresh extraction; DO NOT paste these as current without re-verifying — they roll forward each quarter. Each figure lists its source basis. See `references/hyper5_reference_data.md` for the full detail, verbatim excerpts, markers, and the two corrections that were applied.

### Category 1 — Leases Not Yet Commenced (undiscounted), ranked
| Rank | Company | Amount | Source basis | Commence | Terms |
|------|---------|--------|--------------|----------|-------|
| 1 | Microsoft | $329.1B | FY2026 10-K (as of 2026-06-30, filed 2026-07-30) | FY2027-FY2033 | 1-20 yr |
| 2 | Oracle | $288.0B | Q1 FY2027 10-Q (as of 2026-08-31) | Q2 FY2027-FY2029 | 15-19 yr |
| 3 | Meta | $278.99B (+$68B July subseq.) | Q2 2026 10-Q (as of 2026-06-30) | 2026-2036 | >1-30 yr |
| 4 | Amazon | $137.2B | Q2 2026 10-Q (as of 2026-06-30) | n/d | n/d |
| 5 | Alphabet | $85.2B (+$5.8B June subseq.) | Q2 2026 10-Q (as of 2026-06-30) | 2026-2031 | 1-26 yr |
| — | **Total** | **$1,118.49B (~$1.12T)** | mixed | — | — |
Total incl. subsequent additions: ~$1.19T (consistent with ~$1.09T independent press tallies).

### Category 2 — Purchase Commitments
| Company | Amount | Notes | Source basis |
|---------|--------|-------|--------------|
| Alphabet | $811.0B | material purchase commitments & other obligations ($200.7B short-term; $707.0B fixed/guaranteed) | Q2 2026 10-Q |
| Amazon | ~$496.0B | long-term (>1 yr) contractual commitments not yet recognized; open POs largely cancellable | Q2 2026 10-Q |
| Meta | $349.31B | non-cancelable commitments (+$14.72B contingent cloud-capacity over 5 yr) | Q2 2026 10-Q |
| Oracle | $34.15B | unconditional purchase & other obligations >=1 yr | Q1 FY2027 10-Q |
| Microsoft | n/d | not separately aggregated; refer to 10-K commitments note | FY2026 10-K |

### Category 3 — Guarantees & Backstops
| Company | Amount | Notes | Source basis |
|---------|--------|-------|--------------|
| Alphabet | $51.4B | $7.6B financial guarantees + $43.8B credit derivatives (data-center backstops) | Q2 2026 10-Q |
| Meta | ~$41.0B | ~$28B RVG aggregate threshold (decreasing) + ~$13B El Paso venture RVG | Q2 2026 10-Q |
| Oracle / Amazon / Microsoft | n/d | not quantified (Amazon A-to-z Guarantee unquantified) | — |

### Category 4 — Contingent Liabilities (quantified)
| Company | Amount | Notes | Source basis |
|---------|--------|-------|--------------|
| Meta | $7.14B | uncertain tax positions; + EUR 1.2B IDPC GDPR fine (under appeal) | Q2 2026 10-Q |
| Amazon | $7.1B | income tax contingencies (excluded from commitments table) | Q2 2026 10-Q |
| Alphabet / Oracle / Microsoft | n/d amt | legal/tax contingencies referenced, not separately quantified | — |

### Grand totals (disclosed only; NOT apples-to-apples due to n/d gaps)
- Leases not yet commenced (5 co.): $1,118.49B (+$73.8B subseq.)
- Purchase commitments (4 co.): ~$1,690.46B
- Guarantees/backstops (2 co.): $92.4B
- Contingent (2 co. quantified): $14.24B
- **TOTAL DISCLOSED OFF-BALANCE-SHEET: ~$2.99 TRILLION**

---

## OUTPUT DELIVERABLES

All four deliverables are **displayed in-session** — nothing is written to the workspace, and no `<asset>` tag is emitted.

1. **Markdown report** — full detailed report with all tables, a source-basis column, verbatim excerpts + markers, and caveats. Printed inline in the chat.
2. **Raw data table (CSV)** — key metrics per company incl. `Source_Basis` and computed `Total_OBS_Disclosed_B`. Printed in-session as a fenced ```csv code block the user can copy.
3. **Summary stats** — executive totals, ranked leases, per-company filing basis, any revisions applied during cross-check, key observations. Printed in-session as a fenced code block.
4. **Cross-sectional visualization** — rendered to `/tmp` scratch and shown in-session (Part 3). Not saved to the workspace.

---

## PITFALLS TO AVOID (process learnings — do NOT hardcode figures)

1. **Stale interim filing vs. newer year-end filing.** An interim 10-Q's uncommenced-lease number can be superseded by a much larger figure in a just-filed fiscal-year-end 10-K. Check the SEC website for the newest filing near fiscal year-ends; the corpus may lag. Use the newest disclosed figure.
2. **Wrongly marking a disclosed figure `n/d` (esp. Amazon).** Verify any `n/d` against the actual filing text on the SEC website before concluding a number is absent. Amazon in particular DOES disclose an aggregate uncommenced-lease figure.
3. **Cross-company misattribution.** Guarantees/RVGs and venture-specific commitments belong to the issuer whose filing discloses them. Never move a figure across companies.
4. **Conflating commitments with performance obligations.** Remaining performance obligations / backlog that customers owe the company are NOT the company's purchase commitments. Keep distinct.

---

## VERIFICATION CHECKLIST

- [ ] Every figure pulled live from a primary filing (Pronto SEC corpus and/or SEC/EDGAR) — no stored/estimated numbers
- [ ] Each figure carries its source marker AND source-basis (filing, as-of/filed date, corpus vs. SEC)
- [ ] Each company cross-checked on the SEC website; every `n/d` verified against filing text
- [ ] Checked for a newer 10-K/10-Q than the corpus's latest doc (esp. near fiscal year-ends)
- [ ] All 5 companies have a leases-not-yet-commenced figure — **Amazon included** (not left as n/d unless the primary filing truly omits it)
- [ ] Each figure tied to a specific filing date and fiscal period (not estimated)
- [ ] Commencement window taken verbatim; asset type identified; lease terms captured
- [ ] Verbatim footnote excerpt (with marker) per company
- [ ] Post-balance-sheet additions captured separately
- [ ] Purchase-commitments table complete; commitments not conflated with performance obligations
- [ ] Guarantees identified with max exposure and correct issuer attribution
- [ ] Contingent liabilities >$100M flagged
- [ ] Summary totals correct; all figures in $B
- [ ] Any figure not found marked "n/d" (never guessed or web-estimated)
- [ ] Cross-sectional visualization generated with `None` for undisclosed figures (rendered "n/d", never 0), rendered to `/tmp` scratch, and displayed in-session (NOT saved to the workspace, NO `<asset>` tag)
- [ ] All deliverables (report, CSV, summary, chart) shown in-session; nothing written to `/workspace/`
