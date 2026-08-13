# BI Analytics Portfolio

**Sheetal Sonke**

Power BI and Microsoft Fabric work samples.

---

## Enterprise Sales Analytics

A five-page Power BI report on a synthetic enterprise sales dataset, built to demonstrate
semantic modelling, DAX, and report design.

**To open it:** download and double-click
[`Power-BI/Enterprise-Sales-Analytics/Enterprise_Sales_Analytics.pbix`](Power-BI/Enterprise-Sales-Analytics/Enterprise_Sales_Analytics.pbix).
No login, no database connection, no setup — the data is stored inside the file.

### What it is, in plain terms

A sales leadership dashboard for a fictional company with six regional hubs, 300 customers
and 100 products, covering January 2024 to July 2026.

It answers the questions a sales director actually asks: are we growing, are we hitting
target, which customers and products matter, and are we about to run out of stock.

It is a working report, not a picture. Click anything, change the year, and every number
recalculates.

| Page | What it shows |
|---|---|
| **Executive Performance** | Revenue, profit, target attainment, and a written summary that composes itself from the data |
| **Sales & Customer Analysis** | Who is driving growth: segments, salespeople, top customers |
| **Product & Inventory Performance** | Stock exposure — what's selling, what's overstocked, what's about to run out |
| **Technical Overview** | How the solution is built |
| **Model Validation** | Proof the numbers are right, reconciled against source |

Try the **Year** dropdown at the top right of the first three pages. Everything re-bases:
2026 shows seven months, 2025 a full year, and the prior-year comparison follows
automatically.

---

### Three things worth noticing

**The written summary on page 1 is calculated, not typed.** It works out the best and worst
performing location itself and writes the sentence. Refresh next month and it rewrites
itself. Most dashboards have a text box that goes stale.

**Row-level security works and can be demonstrated.** In Power BI Desktop:
*Modeling → View as → Regional Manager - West*. Total revenue drops from $56.8M to $12.8M
and the header label changes from "All locations" to "West Hub". Most portfolio reports
describe security; this one runs it.

**A real modelling problem was found and fixed.** Inventory is recorded as a monthly
snapshot. Added up naively across 31 months it reports **$369M of stock and 905 products
below reorder point** — against a catalogue of only 100 products, which is impossible. The
correct figures are **$12.3M and 22 products**.

Full write-up: [`docs/inventory-semi-additive-finding.md`](docs/inventory-semi-additive-finding.md)

That last one is the point of the sample. Anyone can build charts. The value is in noticing
when a number can't be true.

---

### An honest read of the data

Target attainment shows 144%. That is not straightforwardly good news — targets total
$39.4M against $56.8M actual, and by location the range runs 81% to 231%. A plan set that
far below reality is a planning problem as much as a performance win, and the report says
so rather than showing a clean green number.

Growth is also slowing: 2025 grew 12.4% on 2024, while 2026 is annualising close to flat.

---

## Repository layout

```
Power-BI/
  Enterprise-Sales-Analytics/
    Enterprise_Sales_Analytics.pbip           open this to work on it
    Enterprise_Sales_Analytics.Report/        PBIR — report definition, plain JSON
    Enterprise_Sales_Analytics.SemanticModel/ TMDL — model and all DAX, plain text
    Enterprise_Sales_Analytics.pbix           open this to just look at it
  Data/
    enterprise_sales_analytics_dataset/       nine source CSVs
    erp_sales_bronze_source/                  raw ERP-shaped extract
docs/
```

The model and report are stored as text, so the DAX is readable in the browser without
opening Power BI. Start here:
[`inventory_measures.tmdl`](Power-BI/Enterprise-Sales-Analytics/Enterprise_Sales_Analytics.SemanticModel/definition/tables/inventory_measures.tmdl)

---

## For a technical reader

<details>
<summary>Model, DAX, and architecture detail</summary>

**Semantic model.** Star schema. Five conformed dimensions plus a hidden security table,
and three facts at three deliberately different grains:

| Table | Grain | Additivity |
|---|---|---|
| `fact_sales` | order line, 33,497 rows | fully additive |
| `fact_sales_target` | month × location | no product or customer key by design |
| `fact_inventory` | month × product × location snapshot | **semi-additive** |

Ten relationships, all single-direction dimension → fact. No bidirectional filtering —
three facts sharing five dimensions is exactly the shape where it creates ambiguity.
`dim_date` is marked as a date table, keys are hidden, `MonthName` sorts by `MonthNumber`.

**DAX.** 32 measures across six domain measure tables — `sales_measures`,
`profitability_measures`, `target_measures`, `time_intelligence_measures`,
`inventory_measures`, `narrative_measures` — each documented with a description that
surfaces in the field-list tooltip. Plus a **Time Intelligence calculation group**
(Current / PY / YoY / YTD) so one set of items applies to every measure.

**Row-level security.** Two roles, both filtering `dim_location` rather than the facts, so
one predicate on a six-row dimension propagates to all three fact tables and target
attainment stays correct because numerator and denominator filter as a pair.
`Regional Manager - West` is static; `Regional Manager (Dynamic)` reads `dim_security` via
`USERPRINCIPALNAME()`.

**Storage mode.** This copy is **Import**, so it opens with no workspace access. The same
model also runs on Microsoft Fabric as a semantic model with the report as a thin report
over a live connection. Measure names and organisation are identical, so the report binds
to either.

**Source control.** PBIP format: the model is TMDL and the report is PBIR, both plain text,
diffable and mergeable in Git, with a path to CI/CD.

**To run from source:**

1. Turn off *File → Options → Data Load → Auto date/time for new files*
2. Open `Power-BI/Enterprise-Sales-Analytics/Enterprise_Sales_Analytics.pbip`
3. *Home → Transform data → Edit parameters* → set `DataFolder` to your local path to
   `Power-BI/Data/enterprise_sales_analytics_dataset` (no trailing backslash)
4. Refresh

**Reconciled figures**, all verified independently against source:

| | Unfiltered | Year = 2026 |
|---|---|---|
| Total Revenue | $56,791,116.63 | $13,485,614 |
| Gross Profit | $20,527,903.42 | $4,846,921 |
| Gross Margin % | 36.1% | 35.9% |
| Sales Target | $39,390,803.16 | $10,061,027 |
| Target Attainment % | 144.2% | 134.0% |
| Order Count | 18,155 | 4,364 |
| Units Sold | 113,894 | 27,120 |
| Inventory Value (corrected) | — | $12,338,824 |

</details>

---

## Data

Synthetic, generated to resemble an ERP sales extract. No real customer or commercial data.
Deliberately shaped to contain genuine modelling problems: a semi-additive snapshot fact, a
target fact at a coarser grain than sales, and targets set well below realised demand.
