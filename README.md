# Sales Analytics

A **Power BI** sales-monitoring solution built for a multi-brand FMCG/consumer-goods
distribution business — daily and monthly sales performance across time, brand,
product, customer, store, and sales-rep dimensions.

This repository documents the architecture and DAX behind the solution's **Sales
Metrics** module. It's a portfolio write-up, not the live report: all entity names in
the underlying model are generic sample labels (`Client 001`, `Store 001`, `Product
001`, `Brand 1`…), and no real business figures, screenshots, or connection details
are included here.

## Business context

The business distributes multiple brands across roughly four divisions through three sales channels — Modern Trade, Traditional Trade, and E-commerce — via a network of stores grouped into regional delivery routes, each served by a sales rep, an area manager, and (where applicable) a field merchandiser. This solution provides a centralized view of commercial performance: how is this month tracking against plan and last year, which brands and products are driving or dragging performance, which customers need attention, and how is each rep's territory doing.

## What this covers

- **Daily & monthly sales monitoring** — value, volume, margin, invoice count, and
  numeric distribution, sliced by month, week, brand, product, customer, store, and
  sales rep.
- **Plan vs. actual tracking** — a sales plan derived automatically from each brand's
  growth target, tracked to achievement rate and variance.
- **Rankings & sales share** — top customers, top products, and share-of-total views
  by brand and by channel.
- **A reusable DAX time-intelligence engine** — 23 purpose-built functions that every
  KPI composes, instead of one-off measures per comparison.
- **A fully dynamic, multi-language UI** — every label in the report resolves live
  from one culture measure; English is the default, with a second language (Albanian)
  supported as a live, switchable culture — no duplicated pages.

## Architecture

```mermaid
flowchart LR
    ERP[("On-prem ERP\n(SQL Server)")] -->|Import, Power Query M| SM[Semantic Model\nTMDL / PBIP]
    SM --> FN["23 DAX functions\nperiod-window · period-average\ncomparison"]
    FN --> DAX["Sales Metrics KPIs\ncomposed from the function library"]
    SM --> RLS["Row-Level Security\n(role + culture per user)"]
    DAX --> RPT[Report — 7 pages]
    RLS --> RPT
    RPT --> EN[English UI]
    RPT --> Alt[Second language UI\nswitched live]
```

## Tech stack

- **Power BI** — authored in [PBIP](https://learn.microsoft.com/power-bi/developer/projects/projects-overview) (TMDL) format for source control
- **DAX user-defined functions** — a 23-function reusable time-intelligence and comparison library (see [DAX Function Library](docs/dax-functions.md))
- **DAX measures** — Sales Metrics KPIs composed from that library (see [DAX Measure Catalog](docs/dax-measures.md))
- **Power Query (M)** — SQL Server import and shaping
- **Row-Level Security** — role- and culture-based, resolved from a single access-control table
- **Dynamic localization** — a live culture toggle driving every label in the report from one measure

## Documentation

| Doc | Contents |
|---|---|
| [Data Model](docs/data-model.md) | Star schema, ER diagram, table catalog, relationships, calculation groups, RLS, and the dynamic localization architecture |
| [DAX Function Library](docs/dax-functions.md) | All 23 reusable functions — period-window, period-average/extreme, and comparison families — with full DAX bodies |
| [DAX Measure Catalog](docs/dax-measures.md) | Sales, plan, margin, invoicing/distribution, payments, and ranking measures, built on the function library |
| [Report Tour](docs/report-pages.md) | Page-by-page walkthrough of all 7 Sales Metrics pages |

## Highlights worth a closer look

- **A function library, not copy-paste DAX.** 23 DAX user-defined functions —
  period-window (`YearToDate`, `PriorYear`, `Trailing12Months`, …), period-average
  (`WeeklyAverage`, `MonthlyAverage`, …), and comparison (`GrowthIndex`,
  `AchievementRatio`, `ValueDifference`, …) — mean every KPI gets a full
  current/prior/YTD/weekly-average/growth-index family for free. Adding a new metric
  is composition, not new date logic.
- **Guarded comparison logic.** The comparison functions share consistent blank /
  future-date / past-date / negative-base handling, so a KPI card never shows a
  misleading "+100%" for a product that didn't exist yet or a blank for one that sold
  out.
- **Zero-duplication dynamic UI.** One `[User Culture]` measure, resolved from
  row-level security, drives every page title, KPI label, and filter caption through
  a single label dictionary — a genuinely live language switch, not two copies of
  the report.

## License

[MIT](LICENSE) — this repository (documentation, DAX/architecture write-ups) may be
reused freely with attribution. It does not include the underlying `.pbix`/PBIP
project files, which remain private.

## Author

**Fuad Gashi** — Power BI / Business Intelligence developer.
[GitHub](https://github.com/fuadgashi)
