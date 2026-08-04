# DAX Function Library

Rather than writing separate year-to-date, prior-year, weekly-average, and
growth-index variants of every KPI by hand, the model defines a small library of
**reusable DAX functions** (Power BI's user-defined function feature) once, and every
measure in the [KPI catalog](dax-measures.md) composes them. This is what keeps a
model with 60+ base KPIs from turning into 1,000+ near-duplicate measures — write the
base calculation once, then wrap it in whichever function you need.

Functions fall into three families: **period-window** (evaluate a measure over a
different date range than the current filter context), **period-average /
period-extreme** (average or min/max a measure across a time grain), and
**comparison** (turn two values into a % index, a ratio, or an absolute difference,
with consistent blank/future-date/negative-base handling).

## Period-window functions

Each takes any measure expression and re-evaluates it over a shifted or widened date
window, while respecting (or in the "Constant" variants, deliberately ignoring) the
report's current filter context.

```dax
/// Year-to-date: evaluates the measure for the current calendar year, dates up to
/// and including today. KEEPFILTERS preserves any existing date filters.
function YearToDate =
    (Measure: anyref expr) =>
        CALCULATE(
            Measure,
            KEEPFILTERS(Calendar[Year] = YEAR(TODAY())),
            KEEPFILTERS(Calendar[Date] <= TODAY())
        )

/// Rest-of-year from today: evaluates the measure for the current calendar year,
/// dates from today onward. The forward-looking counterpart of YearToDate.
function FromToday =
    (Measure: anyref expr) =>
        CALCULATE(
            Measure,
            KEEPFILTERS(Calendar[Year] = YEAR(TODAY())),
            KEEPFILTERS(Calendar[Date] >= TODAY())
        )

/// Prior year: shifts the measure back one year using DATEADD and restricts to the
/// previous calendar year. Used for year-over-year comparisons.
function PriorYear =
    (Measure: anyref expr) =>
        CALCULATE(
            Measure,
            DATEADD(Calendar[Date], -1, YEAR),
            Calendar[Year] = YEAR(TODAY()) - 1
        )

/// Rolling 12 months: evaluates the measure for dates from 12 months before today
/// onward, respecting the current filter context.
function Trailing12Months =
    (Measure: anyref expr) =>
        VAR Last12Months = EDATE(TODAY(), -12)
        RETURN
            CALCULATE(Measure, KEEPFILTERS(Calendar[Date] >= Last12Months))

/// Rolling 12 months ignoring external filter context — the "constant" counterpart
/// of Trailing12Months, used where the window must not shrink/shift with slicers.
function Trailing12MonthsConstant =
    (Measure: anyref expr) =>
        VAR Last12Months = EDATE(TODAY(), -12)
        RETURN
            CALCULATE(
                Measure,
                KEEPFILTERS(Calendar[Date] >= Last12Months),
                REMOVEFILTERS()
            )

/// Rolling 6 months: evaluates the measure for dates from 6 months before today onward.
function Trailing6Months =
    (Measure: anyref expr) =>
        VAR Last6Months = EDATE(TODAY(), -6)
        RETURN
            CALCULATE(Measure, KEEPFILTERS(Calendar[Date] >= Last6Months))

/// Rolling 3 months: evaluates the measure for dates from 3 months before today onward.
function Trailing3Months =
    (Measure: anyref expr) =>
        VAR Last3Months = EDATE(TODAY(), -3)
        RETURN
            CALCULATE(Measure, KEEPFILTERS(Calendar[Date] >= Last3Months))
```

## Period-average & period-extreme functions

These collapse a measure across every week/month/day/year up to the current point (or
across all of them, for the min/max variants) — the engine behind every "weekly
average" card and "vs. best/worst month" comparison in the report.

```dax
/// Weekly average: averages the measure across all weeks up to the current week,
/// removing other Calendar filters. Blank-safe.
function WeeklyAverage =
    (Measure: anyref expr) =>
        IF(
            ISBLANK(Measure),
            BLANK(),
            CALCULATE(
                AVERAGEX(
                    FILTER(ALL(Calendar[Week]), Calendar[Week] <= MAX(Calendar[Week])),
                    Measure
                ),
                REMOVEFILTERS(Calendar)
            )
        )

/// Monthly average: averages the measure across all months up to the current month,
/// removing other Calendar filters. Blank-safe.
function MonthlyAverage =
    (Measure: anyref expr) =>
        IF(
            ISBLANK(Measure),
            BLANK(),
            CALCULATE(
                AVERAGEX(
                    FILTER(ALL(Calendar[Month]), Calendar[Month] <= MAX(Calendar[Month])),
                    Measure
                ),
                REMOVEFILTERS(Calendar)
            )
        )

/// Running daily average: averages the measure across all days up to the latest date
/// in context. Returns blank if the measure is blank.
function DailyAverage =
    (Measure: anyref expr) =>
        VAR LatestDate = MAX(Calendar[Date])
        RETURN
            IF(
                ISBLANK(Measure),
                BLANK(),
                CALCULATE(
                    AVERAGEX(FILTER(ALL(Calendar[Date]), Calendar[Date] <= LatestDate), Measure)
                )
            )

/// Constant daily average to today: averages the measure across all days up to
/// TODAY(), independent of the date selected in context (the constant counterpart
/// of DailyAverage).
function DailyAverageConstant =
    (Measure: anyref expr) =>
        VAR CurrentDate = TODAY()
        RETURN
            IF(
                ISBLANK(Measure),
                BLANK(),
                CALCULATE(
                    AVERAGEX(FILTER(ALL(Calendar[Date]), Calendar[Date] <= CurrentDate), Measure)
                )
            )

/// Distributes any monthly total proportionally across working days in the current
/// week. Only parameter: the monthly measure to prorate.
function DynamicWeeklyAverage =
    (MonthlyValue: DECIMAL EXPR) =>
        VAR CurrentWorkingDays = [Total Working Days]
        VAR RemoveWeekFilter   = ALL(Calendar[WeekOfMonth])
        VAR TotalWorkingDays   = CALCULATE([Total Working Days], RemoveWeekFilter)
        VAR MonthlyTotal       = CALCULATE(MonthlyValue, RemoveWeekFilter)
        VAR DailyRate          = DIVIDE(MonthlyTotal, TotalWorkingDays, 0)
        RETURN
            IF(
                OR(OR(CurrentWorkingDays = 0, ISBLANK(CurrentWorkingDays)), TotalWorkingDays = 0),
                0,
                DailyRate * CurrentWorkingDays
            )

/// Maximum monthly value: returns the highest value of the measure across all
/// months. Blank-safe.
function MaxMonth =
    (Measure: anyref expr) =>
        IF(
            ISBLANK(Measure),
            BLANK(),
            MAXX(ALL(Calendar[Month]), CALCULATE(Measure, ALLEXCEPT(Calendar, Calendar[Month])))
        )

/// Maximum weekly value: returns the highest value of the measure across all weeks
/// (ALLEXCEPT keeps the week grain). Blank-safe.
function MaxWeek =
    (Measure: anyref expr) =>
        IF(
            ISBLANK(Measure),
            BLANK(),
            MAXX(ALL(Calendar[Week]), CALCULATE(Measure, ALLEXCEPT(Calendar, Calendar[Week])))
        )

/// Maximum daily value: returns the highest value of the measure across all days.
/// Blank-safe.
function MaxDay =
    (Measure: anyref expr) =>
        IF(ISBLANK(Measure), BLANK(), MAXX(ALL(Calendar[Date]), CALCULATE(Measure)))

/// Maximum yearly value: returns the highest value of the measure across all years.
/// Blank-safe.
function MaxYear =
    (Measure: anyref expr) =>
        IF(
            ISBLANK(Measure),
            BLANK(),
            MAXX(ALL(Calendar[Year]), CALCULATE(Measure, ALLEXCEPT(Calendar, Calendar[Year])))
        )

/// Minimum monthly value: returns the lowest value of the measure across all months.
/// On negative-stored measures (e.g. returns) this returns the most-negative — i.e.
/// the worst month.
function MinMonth =
    (Measure: anyref expr) =>
        IF(
            ISBLANK(Measure),
            BLANK(),
            MINX(ALL(Calendar[Month]), CALCULATE(Measure, ALLEXCEPT(Calendar, Calendar[Month])))
        )

/// Minimum daily value: returns the lowest value of the measure across all days.
/// Blank-safe.
function MinDay =
    (Measure: anyref expr) =>
        IF(ISBLANK(Measure), BLANK(), MINX(ALL(Calendar[Date]), CALCULATE(Measure)))
```

## Comparison functions

Every "vs. prior year" or "vs. plan" pair in the report funnels through one of these
five functions instead of five different hand-written `DIVIDE`/subtraction patterns —
each shares the same blank / future-date / past-date guard logic, so a KPI card never
shows a misleading "+100%" for a product that simply didn't exist yet, or a blank for
one that sold out.

```dax
/// Percentage difference (CurrentValue - PriorValue) / PriorValue for normal
/// positive-base measures. Guards: blank when both blank or current-blank on a
/// future date; returns 1 when current exists but prior was blank on a past date, or
/// when prior is negative. For negative-base measures use GrowthIndexAbsoluteBase.
function GrowthIndex =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank   = ISBLANK(PriorValue)
        VAR _IsFutureDate   = MAX(Calendar[Date]) > TODAY()
        VAR _IsPastDate     = MAX(Calendar[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, 1,
                NOT (_IsCurrentBlank) && PriorValue < 0, 1,
                DIVIDE(CurrentValue - PriorValue, PriorValue)
            )

/// Absolute value difference (CurrentValue - PriorValue). Mirrors the comparison
/// family's blank / future / past guards. Unlike the index functions this returns a
/// value delta, so it does NOT special-case a negative prior with a literal 1.
function ValueDifference =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank   = ISBLANK(PriorValue)
        VAR _IsFutureDate   = MAX(Calendar[Date]) > TODAY()
        VAR _IsPastDate     = MAX(Calendar[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, CurrentValue,
                CurrentValue - PriorValue
            )

/// Achievement ratio CurrentValue / PriorValue (e.g. actual vs. plan) for
/// positive-base measures. Shares the comparison family's guards; returns 1 when
/// prior was blank on a past date or is negative.
function AchievementRatio =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank   = ISBLANK(PriorValue)
        VAR _IsFutureDate   = MAX(Calendar[Date]) > TODAY()
        VAR _IsPastDate     = MAX(Calendar[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, 1,
                NOT (_IsCurrentBlank) && PriorValue < 0, 1,
                DIVIDE(CurrentValue, PriorValue)
            )

/// Index (ratio) difference for measures whose base can be NEGATIVE (e.g. product
/// returns, stored as negative values). Uses ABS(PriorValue) in the denominator so
/// the percentage keeps its real direction when the base is below zero, and returns
/// -1 (i.e. -100%) where a value appears but the prior was blank on a past date — the
/// negative-measure counterpart of GrowthIndex's +1. Pair with negative-magnitude
/// measures only; use GrowthIndex for normal positive measures.
function GrowthIndexAbsoluteBase =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank   = ISBLANK(PriorValue)
        VAR _IsFutureDate   = MAX(Calendar[Date]) > TODAY()
        VAR _IsPastDate     = MAX(Calendar[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, -1,
                DIVIDE(CurrentValue - PriorValue, ABS(PriorValue))
            )

/// Achievement ratio CurrentValue / PriorValue for NEGATIVE-base measures (used
/// exclusively by the returns family). Unlike AchievementRatio it omits the
/// "prior < 0 -> 1" guard, because for returns the prior is always negative and that
/// guard would otherwise fire every time. The ratio of two same-sign negatives comes
/// out positive naturally, so no ABS() is needed in the body. Pair with
/// GrowthIndexAbsoluteBase (the difference counterpart for the same negative-base
/// family).
function AchievementRatioAbsoluteBase =
    (CurrentValue: decimal val, PriorValue: decimal val) =>
        VAR _IsCurrentBlank = ISBLANK(CurrentValue)
        VAR _IsPriorBlank   = ISBLANK(PriorValue)
        VAR _IsFutureDate   = MAX(Calendar[Date]) > TODAY()
        VAR _IsPastDate     = MAX(Calendar[Date]) < TODAY()
        RETURN
            SWITCH(
                TRUE(),
                _IsCurrentBlank && _IsPriorBlank, BLANK(),
                _IsCurrentBlank && _IsFutureDate && NOT (_IsPriorBlank), BLANK(),
                NOT (_IsCurrentBlank) && _IsPriorBlank && _IsPastDate, 1,
                DIVIDE(CurrentValue, PriorValue)
            )
```

## Why this matters

23 functions, each written once with its edge cases fully worked out, back every
time-comparison in the report. A new KPI — say, "average discount per invoice" —
gets a full current-year / prior-year / YTD / weekly-average / growth-index family
for free:

```dax
Average Discount per Invoice =
    DIVIDE([Total Discount Value], [Invoice Count])

Average Discount per Invoice, Current Year YTD =
    YearToDate([Average Discount per Invoice])

Average Discount per Invoice, Prior Year =
    PriorYear([Average Discount per Invoice])

Average Discount per Invoice, YoY Growth =
    GrowthIndex(
        [Average Discount per Invoice, Current Year YTD],
        PriorYear([Average Discount per Invoice, Current Year YTD])
    )
```

No new date logic, no new blank-handling — just compose the library.
