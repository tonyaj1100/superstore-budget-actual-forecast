# DAX measure reference

These are the 23 measure definitions from the reviewed Power BI model. Each code block is a separate measure definition for **New measure**, not a complete DAX query or a single script to paste at once.

Currency measures use USD. WAPE and margin ratios use percentage formatting. `Margin Gap pp` uses ordinary numeric formatting with two decimal places. The formulas retain full precision.

## 1. Actual Revenue

Home table: `FactActualMonthly`.

```dax
Actual Revenue =
SUM(FactActualMonthly[Revenue])
```

## 2. Budget Revenue

Home table: `FactActualMonthly`.

```dax
Budget Revenue =
SUM(FactBudgetMonthly[BudgetRevenue])
```

## 3. Revenue Variance

Home table: `FactActualMonthly`.

```dax
Revenue Variance =
[Actual Revenue] - [Budget Revenue]
```

## 4. Revenue Variance %

Home table: `FactActualMonthly`.

```dax
Revenue Variance % =
DIVIDE([Revenue Variance], [Budget Revenue])
```

## 5. Actual Profit

Home table: `FactActualMonthly`.

```dax
Actual Profit =
SUM(FactActualMonthly[ReportedProfit])
```

## 6. Budget Profit

Home table: `FactActualMonthly`.

```dax
Budget Profit =
SUM(FactBudgetMonthly[BudgetProfit])
```

## 7. Profit Variance

Home table: `FactActualMonthly`.

```dax
Profit Variance =
[Actual Profit] - [Budget Profit]
```

## 8. Actual Margin %

Home table: `FactActualMonthly`.

```dax
Actual Margin % =
DIVIDE([Actual Profit], [Actual Revenue])
```

## 9. Budget Margin %

Home table: `FactActualMonthly`.

```dax
Budget Margin % =
DIVIDE([Budget Profit], [Budget Revenue])
```

## 10. Margin Gap pp

Home table: `FactActualMonthly`.

Returns a number in percentage points, not a percentage-formatted ratio. Calculate before rounding.

```dax
Margin Gap pp =
IF(
    ISBLANK([Actual Margin %])
        || ISBLANK([Budget Margin %]),
    BLANK(),
    100 * ([Actual Margin %] - [Budget Margin %])
)
```

## 11. Naive Forecast Revenue

Home table: `FactForecastMonthly`.

```dax
Naive Forecast Revenue =
SUM(FactForecastMonthly[NaiveForecast])
```

## 12. Base Forecast Revenue

Home table: `FactForecastMonthly`.

```dax
Base Forecast Revenue =
SUM(FactForecastMonthly[BaseForecast])
```

## 13. Active Forecast Revenue

Home table: `FactForecastMonthly`.

```dax
Active Forecast Revenue =
SUM(FactForecastMonthly[ActiveForecast])
```

## 14. Latest Estimate Revenue

Home table: `FactForecastMonthly`.

Sums known first-half actuals and the selected second-half forecast. The Base full-year result is $724,873.85.

```dax
Latest Estimate Revenue =
SUM(FactForecastMonthly[LatestEstimate])
```

## 15. Latest Estimate vs Budget

Home table: `FactForecastMonthly`.

```dax
Latest Estimate vs Budget =
[Latest Estimate Revenue] - [Budget Revenue]
```

## 16. Forecast Evaluation Cells

Home table: `FactForecastMonthly`.

Counts nonblank Base absolute-error observations; the unfiltered July-December comparison has 72 cells.

```dax
Forecast Evaluation Cells =
COUNT(FactForecastMonthly[AbsErrorBase])
```

## 17. Forecast Scenario

Home table: `FactForecastMonthly`.

Displays the scenario imported from the saved Excel workbook; it does not select a scenario.

```dax
Forecast Scenario =
SELECTEDVALUE(FactForecastMonthly[ScenarioName])
```

## 18. Forecast Cutoff

Home table: `FactForecastMonthly`.

The forecast information cutoff is 30 June 2021. This measure returns a date/time value.

```dax
Forecast Cutoff =
MAX(FactForecastMonthly[ForecastAsOf])
```

## 19. Profit Variance Color

Home table: `FactActualMonthly`.

Use this measure for field-value conditional formatting: red for a shortfall, green for a surplus, grey for zero.

```dax
Profit Variance Color =
VAR Gap = [Profit Variance]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(Gap), BLANK(),
        Gap < 0, "#B91C1C",
        Gap > 0, "#15803D",
        "#64748B"
    )
```

## 20. Naive Segment WAPE

Home table: `FactForecastMonthly`.

Absolute errors are calculated at Month x Region x Category level before aggregation. Unfiltered result: 48.67%.

```dax
Naive Segment WAPE =
DIVIDE(
    SUM(FactForecastMonthly[AbsErrorNaive]),
    SUM(FactForecastMonthly[EvaluationActual])
)
```

## 21. Base Segment WAPE

Home table: `FactForecastMonthly`.

Absolute errors are calculated at Month x Region x Category level before aggregation. Unfiltered result: 66.34%.

```dax
Base Segment WAPE =
DIVIDE(
    SUM(FactForecastMonthly[AbsErrorBase]),
    SUM(FactForecastMonthly[EvaluationActual])
)
```

## 22. Naive Monthly WAPE

Home table: `FactForecastMonthly`.

Aggregates selected segments within each evaluation month before taking the absolute error. Unfiltered result: 25.77%.

```dax
Naive Monthly WAPE =
VAR FutureMonths =
    FILTER(
        VALUES(DimMonth[MonthStart]),
        CALCULATE(
            COUNT(FactForecastMonthly[EvaluationActual])
        ) > 0
    )
RETURN
    DIVIDE(
        SUMX(
            FutureMonths,
            ABS(
                CALCULATE([Actual Revenue])
                    - CALCULATE([Naive Forecast Revenue])
            )
        ),
        SUMX(
            FutureMonths,
            CALCULATE([Actual Revenue])
        )
    )
```

## 23. Base Monthly WAPE

Home table: `FactForecastMonthly`.

Aggregates selected segments within each evaluation month before taking the absolute error. Unfiltered result: 21.25%.

```dax
Base Monthly WAPE =
VAR FutureMonths =
    FILTER(
        VALUES(DimMonth[MonthStart]),
        CALCULATE(
            COUNT(FactForecastMonthly[EvaluationActual])
        ) > 0
    )
RETURN
    DIVIDE(
        SUMX(
            FutureMonths,
            ABS(
                CALCULATE([Actual Revenue])
                    - CALCULATE([Base Forecast Revenue])
            )
        ),
        SUMX(
            FutureMonths,
            CALCULATE([Actual Revenue])
        )
    )
```
