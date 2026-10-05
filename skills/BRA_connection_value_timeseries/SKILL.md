---
name: connection-value-timeseries
description: >
  Plot and interpret the connection-value time series between a supplier company
  and a customer company. Use when the user asks for the connection value, supply
  relationship trend, or business-relationship value over time between two companies
  and provides (or the prompt supplies) a supplierCompanyId and customerCompanyId.
---

# Connection Value Time Series

## When to use
Use this skill whenever the request is about the **connection value over time** between a
**supplier** company and a **customer** company, and you have both a `supplierCompanyId`
and a `customerCompanyId`. These IDs are provided in the prompt.

## Inputs
- `supplierCompanyId` — Capital IQ company ID of the supplier.
- `customerCompanyId` — Capital IQ company ID of the customer.

Both are given to you in the prompt. Do not ask the user for them.

## Steps

### Step 1 — Query the data
Call the stored Snowflake function, substituting the two IDs from the prompt:

```sql
SELECT *
FROM TABLE(
    QRSLLM_POC_DB.HENRY_SCHEMA.GET_CONNECTION_VALUE_TIMESERIES(
        <supplierCompanyId>, <customerCompanyId>
    )
)
ORDER BY NETWORK_AS_OF;
```

The result columns are:
`NETWORK_AS_OF`, `CUSTOMER_COMPANY_ID`, `SUPPLIER_COMPANY_ID`, `CUSTOMER_NAME`,
`SUPPLIER_NAME`, `STATUS_TYPE`, `CONNECTION_VALUE`.

If the query returns zero rows, tell the user there is no recorded connection between
that supplier and customer in this direction, and suggest they verify the ID roles
(supplier vs. customer may be swapped).

### Step 2 — Render the line chart
Produce a single-view Vega-Lite line chart from the query result. Field names must match
the SQL result columns exactly (uppercase). Use this specification:

```json
{
  "title": "Connection Value Over Time (USD millions)",
  "mark": {"type": "line"},
  "encoding": {
    "x": {"field": "NETWORK_AS_OF", "type": "temporal", "axis": {"title": "Network As Of"}},
    "y": {"field": "CONNECTION_VALUE", "type": "quantitative", "axis": {"title": "Connection Value (USD millions)"}}
  }
}
```

- If both `ACTUAL` and `ESTIMATE` rows are present, add a color encoding on `STATUS_TYPE`
  so the two are visually distinct.
- Title the chart with the actual supplier and customer names from the result
  (e.g., "KLA (customer) — Lumentum (supplier) Connection Value Over Time").

### Step 3 — Interpret the results
After the chart, write a concise interpretation covering:
- **Coverage**: date range (first to last `NETWORK_AS_OF`) and number of observations.
- **Direction & status**: which company is supplier vs. customer, and whether values
  are `ACTUAL`, `ESTIMATE`, or a mix.
- **Level & trend**: starting value, peak (with its date), trough (with its date), and
  ending value; describe the overall trajectory (rising, falling, step changes, plateaus).
- **Notable movements**: sharp jumps or drops and roughly when they occurred.
- Keep values in USD millions and round sensibly for readability.

## Output format
1. One short context sentence naming the two companies and the relationship direction.
2. The line chart.
3. The bulleted interpretation.
4. Optionally, the full time-series table for reference.
