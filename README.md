# Retail Pricing & Margin Dashboard

A four-page Power BI report over a UK retail dataset, built to answer where gross margin
is leaking — and to flag the transactions that shouldn't have happened at all.

**Stack:** Power BI Desktop · Power Query (M) · DAX · star-schema modelling
**Data:** Synthetic UK retail dataset, January – April 2026

---

## The question

A retailer running eight stores has revenue landing 8.9% below budget. Revenue shortfall
alone doesn't explain much — it could be volume, price, discounting, or margin erosion,
and each has a different fix. The report separates those causes, and adds a control layer
for the transactions that fall outside policy.

> **Note on the data:** this is a synthetic dataset, generated for the project. The
> modelling, DAX and report design are the work; the numbers are not a real trading
> record.

## Headline figures

| | |
| --- | ---: |
| Revenue | £1,000,959 |
| Gross profit | £580,954 |
| Gross margin | 58.04% |
| Units sold | 34,903 |
| Revenue vs budget | **−8.90%** |
| Markdown cost | £111,320 |

## Data model

A star schema — two fact tables sharing conformed dimensions:

```
                    ┌──────────────┐
                    │  DateTable   │
                    └──────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼────────┐  ┌──────▼──────┐   ┌───────▼──────┐
│ Product Master │◄─┤    Sales    ├──►│ Store Master │
│   30 products  │  │Transactions │   │   8 stores   │
│  6 categories  │  │  2,000 rows │   │   5 regions  │
└───────┬────────┘  └─────────────┘   └───────┬──────┘
        │                                     │
        │           ┌─────────────┐           │
        ├──────────►│   Budget    │◄──────────┘
        │           │  192 rows   │
        │           └─────────────┘
        │
┌───────▼────────┐
│ Price Changes  │
│    10 rows     │
└────────────────┘
```

**6 tables · 16 measures · 4 report pages**

Every relationship is many-to-one with single-direction cross-filtering. Filters flow from
the dimensions down into the facts and nowhere else — no bidirectional filtering, which
keeps the filter propagation predictable and the model debuggable.

`DateTable` is a dedicated calendar marked as the model's date table, so both
`Sales Transactions[Date]` and `Budget[Month]` are filtered from a single source of truth.
Budget sits at monthly grain while sales sit at daily grain, which is worth knowing when
reading a part-month: budget attributes to the first of the month.

### Tables

| Table | Grain | Role |
| --- | --- | --- |
| Sales Transactions | One line per transaction | Fact |
| Budget | Store × category × month | Fact |
| Product Master | One row per product | Dimension |
| Store Master | One row per store | Dimension |
| DateTable | One row per day | Dimension |
| Price Changes | One row per price amendment | Audit log |

## Report pages

**1 · Executive Summary** — revenue, gross profit, GM% and variance to budget as KPI
cards; actual against budget by month on a single shared axis; margin by category; top
products by revenue.

**2 · Pricing and Margin Analysis** — markdown cost by category, achieved selling price
against standard price by product, low-margin transaction counts, and the price-change
log with reason codes.

**3 · Store & Category Performance** — performance by store and region, category mix,
and variance to target by location.

**4 · Audit & Control Exceptions** — the control layer. Every transaction carries a flag,
and this page surfaces the ones that need review:

| Flag | Count | What it means |
| --- | ---: | --- |
| `BELOW_COST` | 25 | Sold beneath cost price — a direct loss |
| `HIGH_DISCOUNT` | 40 | Discount beyond policy threshold |
| `PRICE_MISMATCH` | 20 | Transaction price ≠ current standard price |
| `VARIANCE_OUTLIER` | 15 | Statistically anomalous against budget |

## Selected measures

```dax
GM % = DIVIDE([Gross Profit], [Total Revenue], 0)

Revenue Variance   = [Total Revenue] - [Budget Revenue]
Revenue Variance % = DIVIDE([Revenue Variance], [Budget Revenue], 0)

Below Cost Count =
COUNTROWS(
    FILTER('Sales Transactions', 'Sales Transactions'[Flag] = "BELOW_COST")
)

Avg Discount % =
AVERAGEX(
    FILTER('Sales Transactions', 'Sales Transactions'[Discount %] > 0),
    'Sales Transactions'[Discount %]
)
```

`DIVIDE` throughout rather than `/`, so a filter selection that empties the denominator
returns a clean zero instead of an error spreading across the visual.

## Design decisions

**One shared axis for actual against budget.** A secondary y-axis makes the gap between
two lines a drawing artefact rather than the variance. Both series sit on the same scale;
budget is dashed, following the convention that plan is dashed and actual is solid, so the
two are distinguishable without relying on colour.

**Colour carries meaning or stays out of the way.** Category bars are a single hue —
categories are already labelled on the axis, and a rainbow there encodes nothing. Colour
is spent where it does work: actual against budget, and negative variance in red.

**Explicit format strings.** Currency and percentage formats are set on the measures
rather than per visual, so a figure reads identically wherever it appears.

## Opening it

Requires **Power BI Desktop** (free).

```
├── BI Project.pbix
├── Retail_Dashboard_Dataset.xlsx
└── README.md
```

The queries currently point at a local path. After cloning, open
**Transform data → Data source settings** and repoint them at your copy of the `.xlsx`.

## Limitations

- **Synthetic data**, so the commercial findings are illustrative. The model and measure
  logic are the substance.
- **Four months of history** — not enough for seasonality, year-on-year comparison, or
  any meaningful trend work.
- **`Low Margin Transactions` uses a fixed 30% threshold**, while `Product Master` carries
  a per-product `Target Margin %` that the measure does not yet read. Driving the
  threshold from the dimension is the correct approach and is the next change.
- **Row-level security is not implemented.** A real deployment would restrict store
  managers to their own store.

## Next

- Point the low-margin test at `Product Master[Target Margin %]` rather than a fixed 30%
- Parameterise the source path in Power Query so the file refreshes on any machine
- Row-level security by store and region
- Time intelligence — prior period, year-to-date, rolling averages — once there is enough
  history for it to mean anything
