# Methodology and calculation reference

## 1. Project scope and dates

This is a retail budget-versus-actual and revenue forecasting portfolio case study using fictional Tableau Sample Superstore data. The source copy contains 9,994 sales lines, 5,009 distinct orders and 37,873 units, with order dates from 3 January 2018 to 30 December 2021.

Calendar 2020 is the budget baseline and calendar 2021 is the reporting year. The simulated budget uses a notional 31 December 2020 planning cutoff. The forecast uses a notional 30 June 2021 information cutoff. Both were constructed retrospectively as portfolio exercises in October 2026.

All money is USD. Source `Sales` is treated as revenue and source `Profit` as reported profit. There is no complete expense ledger, balance sheet or cash-flow statement in this dataset.

## 2. Data quality decisions

- Order IDs repeat across sales lines; a sales line is not a distinct order.
- Loss-making lines are retained: the source includes 1,871 lines with negative reported profit.
- Eleven missing postal codes are retained rather than fabricated.
- The Returns sheet contains 800 records but only 296 distinct order IDs. Return status is identified through membership in that order-ID set, avoiding duplication of sales lines. The flag does not imply that refunds or accounting reversals have been applied.
- The monthly reporting grain is Month x Region x Category. Four regions, three categories and twelve months produce 144 observations per fact table.
- Source and Excel data retain their original precision. Formatting rounds values for display; calculations use unrounded values.

## 3. Budget construction

For each Region-Category segment `s`, define:

```text
BaselineRevenue_s = total 2020 revenue for segment s
BaselineMargin_s = total 2020 reported profit_s / BaselineRevenue_s
AnnualBudgetRevenue_s = BaselineRevenue_s * 1.08
TargetMargin_s = BaselineMargin_s + 0.01
MonthlyWeight_m,s = 2020 revenue_m,s / BaselineRevenue_s
BudgetRevenue_m,s = AnnualBudgetRevenue_s * MonthlyWeight_m,s
BudgetProfit_m,s = BudgetRevenue_m,s * TargetMargin_s
```

The 8% revenue assumption applies to each segment. The margin uplift is an absolute **one percentage point**, implemented as `+0.01` on the margin ratio. It is not a 1% relative increase in margin.

`Budget_build` contains the construction calculations. `Budget` stores the frozen `Budget_v1` comparison baseline. Changes to a live forecast scenario must not overwrite this budget baseline.

## 4. Actual-versus-budget metrics

```text
RevenueVariance = ActualRevenue - BudgetRevenue
RevenueVariancePct = RevenueVariance / BudgetRevenue
ProfitVariance = ActualReportedProfit - BudgetProfit
ActualMargin = ActualReportedProfit / ActualRevenue
BudgetMargin = BudgetProfit / BudgetRevenue
MarginGapPP = 100 * (ActualMargin - BudgetMargin)
```

Aggregated margins are ratios of aggregated profit to aggregated revenue, not averages of individual row margins. DAX `DIVIDE` returns blank for a zero denominator unless an alternate result is supplied.

The displayed 12.74% actual margin and 14.43% budget margin are rounded. Their underlying difference is approximately -1.68276 percentage points, displayed as **-1.68 pp**.

The workbook also decomposes the profit variance at Month x Region x Category level into revenue and profitability effects. The reviewed totals are:

| Component | Amount |
|---|---:|
| Revenue effect | +$14,740.27 |
| Profitability effect | -$16,219.21 |
| Reconciled profit variance | -$1,478.94 |

These are sums of cell-level effects. Applying one aggregated margin to the aggregated revenue gap will not generally reproduce the cell-level decomposition.

## 5. Forecast construction

The model uses January-June 2021 actuals as known information and forecasts July-December 2021. For each Region-Category segment `s` and future month `m`:

```text
NaiveForecast_m,s = revenue for corresponding month m and segment s in 2020
H1GrowthFactor_s = H1 2021 revenue_s / H1 2020 revenue_s
BaseForecast_m,s = NaiveForecast_m,s * H1GrowthFactor_s
DownsideForecast_m,s = BaseForecast_m,s * 0.95
UpsideForecast_m,s = BaseForecast_m,s * 1.05
ActiveForecast_m,s = forecast for the selected Excel sensitivity case
FullYearLatestEstimate = H1 2021 actual revenue + H2 active forecast revenue
```

Historical forecast and error fields are blank for January-June, rather than zero-valued forecasts. The `LatestEstimate` field uses actual revenue in historical months and the active forecast in future months.

July-December actual revenue is used as evaluation truth only. It is excluded from the forecast-driver calculations. Changing the sensitivity case affects Active Forecast and Latest Estimate; the Naive and Base error cards continue to compare those two named methods.

### Base-case reconciliation

| Item | Revenue |
|---|---:|
| H1 2021 actual | $256,909.23 |
| H2 seasonal-naive forecast | $379,886.12 |
| H2 Base forecast | $467,964.62 |
| Base full-year latest estimate | $724,873.85 |
| Full-year simulated budget | $657,942.05 |
| Base latest estimate minus budget | +$66,931.81 |
| H2 2021 actual, used for evaluation | $476,306.02 |

## 6. Forecast evaluation at two levels

The general WAPE definition is:

```text
WAPE = SUM(ABS(Actual - Forecast)) / SUM(ABS(Actual))
```

Revenues in this model are positive, so summing evaluation revenue is equivalent to summing its absolute value. Results are formatted as percentages.

**Segment WAPE:** calculate an absolute error for every Month-Region-Category observation, then sum those errors across the six evaluation months. The unfiltered denominator covers 72 observations: 6 months x 4 regions x 3 categories.

**Monthly WAPE:** aggregate revenue and forecast across the selected segments within each evaluation month, take that month's absolute error, then sum the six monthly errors. Aggregation occurs before the absolute-value operation.

| Level | Seasonal naive | Base |
|---|---:|---:|
| Segment WAPE | 48.67% | 66.34% |
| Company monthly WAPE | 25.77% | 21.25% |

The Base method has lower company-month error but higher segment error. Opposite-signed segment errors can offset during aggregation. Method comparisons therefore use the same July-December window and the same level of aggregation.

WAPE is not labelled accuracy, and values above 100% are possible for some filtered subsets. A single six-month holdout is a limited evaluation; these results are not evidence of performance on later periods. WAPE also has limitations when comparing changing windows in trending or seasonal series. A future extension could add rolling-origin backtesting and scaled error metrics.

## 7. Power BI model and field dictionary

| Table | Grain / size | Main fields |
|---|---|---|
| `FactActualMonthly` | Month x Region x Category; 144 rows | `MonthStart`, `Region`, `Category`, `Revenue`, `ReportedProfit` |
| `FactBudgetMonthly` | Month x Region x Category; 144 rows | `MonthStart`, `Region`, `Category`, `BudgetRevenue`, `BudgetProfit`, `TargetMargin`, `BudgetVersion` |
| `FactForecastMonthly` | Month x Region x Category; 144 rows | `MonthStart`, `Region`, `Category`, `NaiveForecast`, `BaseForecast`, `ActiveForecast`, `LatestEstimate`, `AbsErrorNaive`, `AbsErrorBase`, `EvaluationActual`, `ForecastAsOf`, `ScenarioName` |
| `DimMonth` | One row per reporting month; 12 rows | `MonthStart`, `Year`, `MonthNo`, `MonthName`, `YearMonth` |
| `DimRegion` | One row per region; 4 rows | `Region` |
| `DimCategory` | One row per category; 3 rows | `Category` |

Each fact connects to each dimension through an active many-to-one relationship, giving nine relationships in total. Filters flow from dimensions to facts. `DimMonth` is a monthly reporting dimension, not a daily calendar used for classic time-intelligence functions.

The Region and Category dropdowns use dimension fields. The report has three presentation pages and a hidden Validation page. The PDF and PNG previews are static; slicers and matrix expansion require Power BI.

## 8. Scenario selection and refreshing

1. Open `excel/Retail_Budget_Actual_2021.xlsx`.
2. Change `Controls!M2`: 1 = Downside, 2 = Base, 3 = Upside. `M3` displays the case name and `M4` its adjustment.
3. Recalculate and save Excel so Power BI can read the saved results.
4. In Power Query, update the existing `pWorkbook` parameter to the workbook's full local path.
5. Apply the query changes and refresh the PBIX.

The report's scenario card displays the saved Excel case. It does not provide live scenario switching in Power BI. The uploaded PDF records the Base scenario and shows the cutoff timestamp as `6/30/2021 12:00:00 AM`.

## 9. Validation evidence and limits

- The reviewed Excel workbook had 25 passing checks and no cached formula errors.
- The reviewed Power BI data had 144 rows in each fact table. Every row matched Excel on Month, Region and Category, with zero unmatched keys and zero numeric discrepancies within the comparison tolerance.
- The 23 DAX measure definitions and nine active relationship definitions were inspected in the reviewed PBIX.
- Forecast errors were independently recalculated at both aggregation levels.
- The current three-page PDF was rendered and visually inspected; its visible totals and error cards match those reviewed results.

Stored-data and definition checks are distinct from native Power BI interaction testing. The PDF review does not execute DAX or test interactive filtering.

## 10. Interpretation and next analysis

Revenue growth did not translate into meeting the reported-profit budget. Central, South and Furniture are priorities for follow-up. Product mix, discounting, customer concentration and individual loss-making transactions are reasonable areas to investigate; this report does not establish any of them as causal explanations.

The source does not provide the statements needed for liquidity ratios, leverage ratios, a cash-flow model or a full company valuation. This project's scope is budgeting, variance analysis and revenue forecasting.

## References

- [Tableau Superstore context](https://help.tableau.com/current/guides/get-started-tutorial/en-us/get-started-tutorial-connect.htm)
- [Microsoft Power Query parameters](https://learn.microsoft.com/en-us/power-query/power-query-query-parameters)
- [Rob J Hyndman on WAPE](https://robjhyndman.com/hyndsight/wape.html)
