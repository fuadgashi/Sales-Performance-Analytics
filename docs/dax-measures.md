# DAX Measure Catalog

The measures below back the 7 pages of the Sales Metrics module (see
[Report Tour](report-pages.md)). Every current/prior/YTD/weekly-average/growth-index
variant is composed from the [function library](dax-functions.md) rather than
hand-written per measure — that composition is the pattern to notice throughout this
catalog, not any individual formula.

## 1. Core sales value & volume

The base measure everything else derives from:

```dax
Net Sales Value =
    SUMX('Sales Invoice Lines', 'Sales Invoice Lines'[Quantity] * 'Sales Invoice Lines'[NetUnitPrice])

Net Sales Value, Current Year YTD =
    YearToDate([Net Sales Value])

Net Sales Value, Prior Year =
    PriorYear([Net Sales Value])

Net Sales Value, Prior Year YTD =
    YearToDate([Net Sales Value, Prior Year])

YoY Growth Index =
    GrowthIndex([Net Sales Value, Current Year YTD], [Net Sales Value, Prior Year YTD])

YoY Value Difference =
    ValueDifference([Net Sales Value, Current Year YTD], [Net Sales Value, Prior Year YTD])

Weekly Average Sales Value, Current Year =
    WeeklyAverage([Net Sales Value, Current Year YTD])

Sales Quantity, Current Year YTD =
    YearToDate([Sales Quantity])

Sales Quantity, YoY Growth =
    GrowthIndex([Sales Quantity, Current Year YTD], PriorYear([Sales Quantity, Current Year YTD]))
```

## 2. Plan vs. actual (shown alongside sales value on Month/Brand/Store/Rep pages)

The sales plan is derived automatically from each brand's commercial growth target
applied to last year's actuals — not a manually entered number:

```dax
Sales Plan Value =
    SUMX(Brands, [Net Sales Value, Prior Year] * (1 + Brands[PlanGrowthIndex]))

Sales Plan Value, YTD =
    YearToDate([Sales Plan Value])

Plan Achievement Difference =
    ValueDifference([Net Sales Value, Current Year YTD], [Sales Plan Value])

Plan Achievement Index =
    GrowthIndex([Net Sales Value, Current Year YTD], [Sales Plan Value])

Achievement Rate vs. Plan =
    AchievementRatio([Net Sales Value, Current Year YTD], [Sales Plan Value, YTD])
```

## 3. Gross margin

```dax
Gross Profit =
    SUMX(
        'Sales Invoice Lines',
        'Sales Invoice Lines'[Quantity] * ('Sales Invoice Lines'[NetUnitPrice] - 'Sales Invoice Lines'[AverageCostWithVAT])
    )

Gross Margin % =
    DIVIDE([Gross Profit], [Net Sales Value])

Gross Margin %, Current Year YTD =
    YearToDate([Gross Margin %])

Gross Margin %, Prior Year =
    PriorYear([Gross Margin %])

Weekly Average Gross Margin, Current Year =
    WeeklyAverage([Gross Margin %, Current Year YTD])
```

## 4. Invoicing & numeric distribution

"Numeric distribution" is a standard FMCG metric: the % of active outlets that
bought at least one unit of a product/brand in the period. "Products per invoice"
and "invoice count" are the two other headline operational KPIs on the Month page.

```dax
Invoice Count =
    DISTINCTCOUNT('Sales Invoice Lines'[InvoiceId])

Invoice Count, Current Year YTD =
    YearToDate([Invoice Count])

Weekly Average Invoice Count, Current Year =
    WeeklyAverage([Invoice Count, Current Year YTD])

Products per Invoice =
    DIVIDE([Sales Quantity], [Invoice Count])

Products per Invoice, Current Year YTD =
    YearToDate([Products per Invoice])

Numeric Distribution =
    DIVIDE(
        DISTINCTCOUNT('Sales Invoice Lines'[OutletId]),
        CALCULATE(DISTINCTCOUNT('Customer Outlets'[Id]), REMOVEFILTERS(Products))
    )

Numeric Distribution, Current Year YTD =
    YearToDate([Numeric Distribution])

Numeric Distribution, Prior Year =
    PriorYear([Numeric Distribution])

Weekly Average Numeric Distribution, Current Year =
    WeeklyAverage([Numeric Distribution, Current Year YTD])

Max Monthly Numeric Distribution, Prior Year =
    MaxMonth([Numeric Distribution, Prior Year])
```

## 5. Customer payments

```dax
Payments Received =
    CALCULATE(SUM('Customer Payments'[Value]), 'Customer Payments'[DK] = "D")

Payments Received, Current Year YTD =
    YearToDate([Payments Received])

Payments Received, Prior Year =
    PriorYear([Payments Received])

Weekly Average Payments Received, Current Year =
    WeeklyAverage([Payments Received, Current Year YTD])

Latest Payment Received =
    CALCULATE(SUM('Customer Payments'[Value]), LASTDATE('Customer Payments'[Date]))
```

## 6. Rankings & sales share

Used by the "Top Clients" / "Top Products" visuals and the channel/brand sales-share
donuts across the Brand, Product, and Customer pages:

```dax
Product Rank by Sales =
    RANKX(ALL(Products), [Net Sales Value, Current Year YTD], , DESC)

Sales Share by Customer, Current Year =
    DIVIDE(
        [Net Sales Value, Current Year YTD],
        CALCULATE([Net Sales Value, Current Year YTD], REMOVEFILTERS('Customer Outlets'))
    )

Sales Share by Sales Channel, Current Year =
    DIVIDE(
        [Net Sales Value, Current Year YTD],
        CALCULATE([Net Sales Value, Current Year YTD], REMOVEFILTERS(Customers[SalesChannel]))
    )
```

## 7. Calendar helper measures

Small utility measures that drive the "days elapsed / remaining this month" cards on
the Month page and feed `DynamicWeeklyAverage` in the function library:

```dax
Total Working Days =
    CALCULATE(
        DISTINCTCOUNT(Calendar[WorkdayNumber]),
        Calendar[WorkdayNumber] > 0,
        KEEPFILTERS(YEAR(Calendar[Date]) = YEAR(TODAY()))
    )

Working Days Elapsed This Month =
    VAR FirstDayOfMonth = DATE(YEAR(TODAY()), MONTH(TODAY()), 1)
    RETURN
        COUNTROWS(
            FILTER(
                ALL(Calendar),
                Calendar[Date] >= FirstDayOfMonth && Calendar[Date] <= TODAY() && WEEKDAY(Calendar[Date], 2) <= 5
            )
        )

Working Days Remaining This Year =
    COUNTROWS(
        FILTER(
            Calendar,
            Calendar[Date] > TODAY() && Calendar[Date] <= MAX(Calendar[Date]) && WEEKDAY(Calendar[Date], 2) <= 5
        )
    )
```
