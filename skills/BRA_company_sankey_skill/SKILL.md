# Skill: company-supply-chain-sankey

## Description
Build a company-level supply-chain **Sankey chart** (in the style of "In It Together: How
Company Performance Transmits Through Supply Chains", Exhibit 1) for a target company,
using the Business Relationship Analytics (BRA) feed. The chart shows how customers flow
into the company's revenue, how revenue splits into total costs and net income, and how
total costs flow out to suppliers.

## When to use
Trigger this skill when the user asks to:
- "Create / recreate the Exhibit 1 Sankey" for a company.
- Visualize a company's customers, suppliers, revenue, and cost flows as a Sankey.
- Show supply-chain revenue/cost transmission for a named company.

## Inputs
You will **always be given the company name and its companyId in the prompt** (e.g.
"Lumentum, companyId 272054403"). Use them directly — do not look up the ID.
- `company_name` — display label for the chart.
- `company_id` — Capital IQ company ID (integer).
- `as_of_date` — optional. If the user gives a date, use it; otherwise use the current date.
  The feed resolves to the latest snapshot on or before this date.

## Data source
A saved Snowflake table function does all the feed logic and point-in-time resolution:

```sql
SELECT *
FROM TABLE(QRSLLM_POC_DB.HENRY_SCHEMA.COMPANY_SANKEY_FEED(<company_id>, '<as_of_date>'));
```

Returned columns (one row per counterparty connection; monetary values in USD millions):

| Column | Meaning |
|---|---|
| `NETWORKASOF` | Resolved snapshot date actually used |
| `ROLE` | `CUSTOMER` (feeds revenue) or `SUPPLIER` (paid from cost) |
| `COUNTERPARTY_NAME` | Customer or supplier name |
| `COUNTERPARTY_ID` | Counterparty company ID |
| `STATUSTYPE` | `ACTUAL` or `ESTIMATE` |
| `CONNECTIONVALUE` | Flow size in USD millions |
| `SHARE` | Fraction of target revenue (customers) or cost (suppliers) |
| `IMPLIED_TOTAL_REVENUE` | Company total revenue implied by the feed (constant per row) |
| `IMPLIED_TOTAL_COST` | Company total operational cost implied by the feed (constant per row) |

## Derived quantities (compute after fetching)
- `Total Revenue`  = `IMPLIED_TOTAL_REVENUE`
- `Total Costs`    = `IMPLIED_TOTAL_COST`
- `Net Income`     = `IMPLIED_TOTAL_REVENUE - IMPLIED_TOTAL_COST`
- `Other Customers`= `Total Revenue - SUM(CONNECTIONVALUE where ROLE='CUSTOMER')`
- `Other Suppliers`= `Total Costs    - SUM(CONNECTIONVALUE where ROLE='SUPPLIER')`
- Show the top **N** (default 8) customers/suppliers by `CONNECTIONVALUE`; roll the rest
  into "Other Customers" / "Other Suppliers".

## Chart structure (4 columns, left → right)
1. **Customers** (top-N + "Other Customers") → flow into
2. **Total Revenue** (single full-height node) → splits into
3. **Total Costs** + **Net Income** → Total Costs flows into
4. **Suppliers** (top-N + "Other Suppliers")

## Procedure
1. Run the table function via `system_execute_sql` (through the BRA semantic model) with the
   given `company_id` and `as_of_date`.
2. Read the full result in the sandbox with `RESULT_SCAN('<query_id>')`; compute the derived
   quantities above.
3. Render the Sankey with the reusable matplotlib renderer below. Write the PNG to `/tmp`
   first, then `cp` it into `/workspace/` (FUSE cannot finalize some writers in place; PNG is
   fine but copying is the safe habit).
4. Surface the PNG with an `<asset>` tag and present a short table of the flow values.

## Caveats to state in the answer
- **Feed-only**: the middle waterfall is simplified to a single **Total Costs** node with
  **Net Income** as the residual. The feed carries one aggregate "operational cost" concept,
  not a separate COGS vs. operating-expense split, so it is not broken out like the paper.
- Most connections are `ESTIMATE`; disclosed suppliers are often few, making "Other Suppliers"
  a large plug. Mention the resolved `NETWORKASOF` date.

## Reusable renderer
```python
import matplotlib; matplotlib.use("Agg")
import matplotlib.pyplot as plt
from matplotlib.path import Path
import matplotlib.patches as patches

def build_company_sankey(company_name, as_of, out_png,
                         customers, suppliers,          # list[(name, value_usd_m)] top-N, largest first
                         total_rev, total_cost,
                         other_customers, other_suppliers):
    net_income = total_rev - total_cost
    customers = list(customers) + [("Other Customers", other_customers)]
    suppliers = list(suppliers) + [("Other Suppliers", other_suppliers)]
    GAP = total_rev * 0.02; NW = 0.045
    C_BLUE, C_GREEN, C_GREY, C_ORANGE, C_REVBAR = "#2E6FB7","#3E9E6E","#B7C0C9","#E08A2B","#1F4E79"
    fig, ax = plt.subplots(figsize=(15, 9))

    def node(x, y0, y1, color):
        ax.add_patch(patches.Rectangle((x, y0), NW, y1 - y0, color=color, zorder=3))
    def lbl(x, y, s, ha, size=11, bold=False):
        ax.text(x, y, s, ha=ha, va="center", fontsize=size,
                fontweight=("bold" if bold else "normal"), zorder=5)
    def ribbon(x0, ya0, yb0, x1, ya1, yb1, color, alpha=0.42):
        cx = (x0 + x1) / 2
        verts = [(x0,ya0),(cx,ya0),(cx,ya1),(x1,ya1),(x1,yb1),(cx,yb1),(cx,yb0),(x0,yb0),(x0,ya0)]
        codes = [Path.MOVETO,Path.CURVE4,Path.CURVE4,Path.CURVE4,Path.LINETO,
                 Path.CURVE4,Path.CURVE4,Path.CURVE4,Path.CLOSEPOLY]
        ax.add_patch(patches.PathPatch(Path(verts, codes), facecolor=color,
                                       edgecolor="none", alpha=alpha, zorder=1))

    X_CUST, X_REV, X_MID, X_SUPP = 0.05, 0.40, 0.62, 0.92
    node(X_REV, 0.0, total_rev, C_REVBAR)
    ax.text(X_REV + NW/2, total_rev + GAP*0.6, f"Total Revenue\n${total_rev:,.0f}M",
            ha="center", va="bottom", fontsize=12, fontweight="bold", zorder=5)

    y = total_rev; rev_cursor = total_rev
    for name, val in customers:
        node(X_CUST, y - val, y, C_BLUE)
        lbl(X_CUST - 0.012, y - val/2, f"{name}  ${val:,.0f}M", "right")
        ribbon(X_CUST + NW, y, y - val, X_REV, rev_cursor, rev_cursor - val, C_BLUE)
        rev_cursor -= val; y -= val + GAP

    cost_top, cost_bot = total_rev, total_rev - total_cost
    node(X_MID, cost_bot, cost_top, C_GREY)
    lbl(X_MID + NW + 0.012, (cost_top + cost_bot)/2, f"Total Costs\n${total_cost:,.0f}M", "left")
    ni_top, ni_bot = cost_bot - GAP, cost_bot - GAP - net_income
    node(X_MID, ni_bot, ni_top, C_GREEN)
    lbl(X_MID + NW + 0.012, (ni_top + ni_bot)/2, f"Net Income\n${net_income:,.0f}M", "left")
    ribbon(X_REV + NW, total_rev, cost_bot, X_MID, cost_top, cost_bot, C_GREY)
    ribbon(X_REV + NW, cost_bot, 0.0, X_MID, ni_top, ni_bot, C_GREEN)

    supp_cursor = cost_top; sy = cost_top
    for name, val in suppliers:
        col = C_ORANGE if name != "Other Suppliers" else C_GREY
        node(X_SUPP, sy - val, sy, col)
        lbl(X_SUPP + NW + 0.012, sy - val/2, f"${val:,.0f}M  {name}", "left")
        ribbon(X_MID + NW, supp_cursor, supp_cursor - val, X_SUPP, sy, sy - val, col)
        supp_cursor -= val; sy -= val + GAP

    ax.set_xlim(-0.16, 1.16); ax.set_ylim(min(0, sy) - GAP*2, total_rev + GAP*8); ax.axis("off")
    ax.set_title(f"{company_name} — Supply Chain Revenue & Cost Flow (Sankey)\n"
                 f"Feed-only reconstruction · network as of {as_of} · values in USD millions",
                 fontsize=14, fontweight="bold")
    plt.tight_layout(); plt.savefig(out_png, dpi=150, bbox_inches="tight", facecolor="white")
```

## Output
- One PNG Sankey surfaced via `<asset>`.
- A short flow table and a one-line note on the resolved date + feed-only caveat.
- Offer to schedule the chart on a recurring cadence (the feed refreshes over time).
