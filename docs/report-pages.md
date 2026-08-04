# Report Tour — Sales Metrics

7 pages, all sharing the same core KPI set (net sales value, gross margin, invoice
count, products per invoice, numeric distribution, plan achievement) sliced across
six different dimensions — the day-to-day sales monitoring surface of the solution.

| Page | What it shows |
|---|---|
| **Monthly Performance** | Full KPI overview for the current month: weekly trend combo charts for sales value, gross margin, invoice count, payments received, and products-per-invoice, each benchmarked against prior year and the weekly plan pace. Includes a detailed date-by-date pivot table and "days elapsed / remaining this month" cards. |
| **Time Frame** | The same KPI set as a monthly trend across the full year, plus sales-vs.-plan variance by month — for spotting a multi-month trend rather than reacting to one bad week. |
| **Brand / Product** | Brand and product-level sales, gross margin, and plan-vs-actual breakdown; top-clients bar chart and sales-share donut by brand. |
| **Product Sales Analysis** | Product-level pivot tables (by value and by quantity) with sales share, YoY growth index, and net average selling price comparison; a "top customers for this product" ranking. |
| **Customer / Store** | Customer-level plan-vs-actual pivot, weekly payments-received trend, top-products-per-customer, sales share by channel, and account balance/currency. |
| **Store Performance** | Store-level plan-vs-actual pivot with sales share of plan and of total, plus a product breakdown per store. |
| **Sales Representative** | Rep-level pivot (sales, plan variance, returns, distribution, invoices/products) with a geographic map of sales by route/location and an area chart of rep-level plan achievement. |

Every page shares a consistent shell: a bookmarked left-hand navigator, a filter
panel (date range, brand/product, channel/customer, region/city, rep/manager), and a
description flyout explaining how to read that page — all localized through the
dynamic culture switch described in [Data Model](data-model.md#row-level-security--dynamic-localization).
