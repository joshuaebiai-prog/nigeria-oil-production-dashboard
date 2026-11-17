# Project Notes: Nigeria Crude Oil and Condensate Production Dashboard

Detailed method notes for the Power BI dashboard in `dashboard/`. The README gives the summary; this file records how the data was extracted, cleaned, validated and modelled, so the work can be checked or reproduced.

## Contents

1. [Data source and coverage](#1-data-source-and-coverage)
2. [Source layout](#2-source-layout)
3. [Extraction and cleaning](#3-extraction-and-cleaning)
4. [Unit conversion for 2026](#4-unit-conversion-for-2026)
5. [Stream name standardisation](#5-stream-name-standardisation)
6. [Stream types](#6-stream-types)
7. [Validation](#7-validation)
8. [Data model and DAX measures](#8-data-model-and-dax-measures)
9. [Dashboard design decisions](#9-dashboard-design-decisions)
10. [Caveats and limitations](#10-caveats-and-limitations)
11. [Possible next steps](#11-possible-next-steps)

---

## 1. Data source and coverage

- **Source:** Nigerian Upstream Petroleum Regulatory Commission (NUPRC), monthly Crude Oil and Condensate Production reports, published as PDF files. [Link in README.]
- **Period:** January 2024 to September 2026.
- **Granularity:** monthly production by terminal or stream, split by liquid type (crude oil and condensate).
- **Status:** 2024 and 2025 are full-year reports. The 2026 report is labelled **provisional** and contains data for January to September only. NUPRC may revise it.

## 2. Source layout

Each yearly PDF has two table pages, and the 2026 file follows the same two-page structure.

| Page | Content | Features that affect cleaning |
|---|---|---|
| Page 1 | Terminals reported with crude, condensate and a **Blend Total** row (Bonny, Brass, Qua Iboe, Forcados, Escravos, Odudu, Tulja-Okwuibome, Aje) plus several crude-only streams | Merged terminal-name cells; Blend Total subtotal rows; watermark text |
| Page 2 | Other crude streams and condensate-only streams, followed by summary blocks (total liquid and daily average) | Summary rows below the stream table; watermark text |

Differences between years:

| | 2024 and 2025 | 2026 |
|---|---|---|
| Unit | Barrels per month | **Thousand barrels per day** |
| Decimals | None | Two decimal places |
| Months with data | 12 | 9 (January to September); October to December are dashes |
| Status | Published | **Provisional** |
| Total liquid summary rows | Present (barrels) | Daily average block only (thousand bpd) |

## 3. Extraction and cleaning

Tables were extracted with Power Query's PDF connector. Each year has two queries (page 1 and page 2), appended into one yearly query. The three yearly queries are then appended into `Production_All`. Only `Production_All` loads into the report; the building-block queries have **Enable load** switched off to avoid double counting.

### 3.1 Steps for 2024 and 2025

**Page 1 query**

1. Promote headers and rename the first two columns to `TERMINAL/STREAM` and `Liquid Type`.
2. **Fill down** the terminal name, because merged cells extract as blanks below the first row.
3. Filter out `Blend Total` rows. They are subtotals; keeping them would count every barrel twice.
4. Remove blank rows.
5. Remove the watermark text (see 3.3).
6. Replace `-` with `0`.
7. Change month columns to numeric types.
8. Unpivot the month columns into `Month` and `Barrels`.
9. Add `Year` and a `Date` column (first day of the month).
10. Add `Stream Type` (see section 6).

**Page 2 query**

The same steps, except no fill down and no Blend Total filter. The summary blocks at the bottom of the page are removed by keeping only rows where `Liquid Type` is `Crude Oil` or `Condensate`.

**Row counts after cleaning**

| Query | Rows |
|---|---|
| 2024 page 1 | 276 |
| 2024 page 2 | 252 |
| 2024 combined | 528 |
| 2025 combined | 528 |
| 2026 page 1 | 207 |
| 2026 page 2 | 180 |
| 2026 combined | 387 |
| `Production_All` | 1,443 |

### 3.2 Steps for 2026

The 2026 queries were built separately instead of duplicating the 2025 queries, because the 2025 steps convert to whole numbers and would have rounded away the decimals in the thousand-bpd values. Differences:

1. The empty `OCTOBER`, `NOVEMBER` and `DECEMBER` columns are **removed** before unpivoting. They are dashes meaning "not yet reported", so they must not become zero-production months.
2. Dashes in the nine reported months are treated as zero, because those months are reported.
3. Month columns are converted to **decimal** numbers, not whole numbers.
4. After unpivoting, thousand bpd is converted to monthly barrels (section 4).

### 3.3 Watermark handling

The PDFs carry a diagonal "NUPRC" watermark. Where it overlapped a cell, Power Query extracted the cell as text such as `149,116` + line break + `NUPRC`, which prevents numeric conversion.

The fix, applied **before** the data type change:

1. Replace `NUPRC` with nothing in all month columns.
2. Apply **Clean** (removes the line break).
3. Apply **Trim**.

Clean and Trim were also applied to the `TERMINAL/STREAM` and `Liquid Type` columns. The order matters: in one query the type conversion initially failed because the clean-up steps sat after it.

### 3.4 Handling of dashes

In the source tables, `-` means no reported production. Within a reported month this is treated as **zero**. For unreported future months (October to December 2026) it is treated as **missing** and removed.

## 4. Unit conversion for 2026

2024 and 2025 report barrels per month. 2026 reports thousand barrels per day. To stack the years consistently, 2026 values are converted to monthly barrels:

```
Barrels = Thousand BPD x 1,000 x days in the month
```

Worked example, total crude oil for January 2026:

```
1,459.24 x 1,000 x 31 days = 45,236,440 barrels
```

Daily averages in the dashboard are then recalculated from barrels and days, so all three years are on the same basis.

**Rounding:** the 2026 source values have two decimals, so converted monthly barrels are accurate to roughly a few hundred barrels per stream. This is negligible at dashboard level.

## 5. Stream name standardisation

The same stream is labelled differently across years. Names were standardised in `Production_All`, and the original label is kept in a column called `Stream (As Reported)` for traceability.

The replacements used **Match entire cell contents** so partial matches were not altered. 

| Name as reported | Standardised name |
|---|---|
| `ESCRAVOS (Oil Terminal)` | `ESCRAVOS` |
| `OTAKPIPO (Ex ImaTerminal)` | `OTAKPIPO` |
| `OYO / OBODO` | `OYO` |
| `OKORO (Ex Ima Terminal)` | `OKORO` |
| `ASARAMATORU (Ex ImaTerminal)` | `ASARAMATORU` |
| `ODUDU (AMENAM BLEND)` | `ODUDU` |

Notes:

- The distinct name list fell from **38 to 35** after the merges.
- Some labels read `Ex ImaTerminal` with no space because the Clean step removed a line break inside the PDF cell.
- `IMA` (a condensate stream) is a separate stream from the "Ex Ima Terminal" label that appears on Okoro, Otakpipo and Asaramatoru. It was **not** renamed or merged.
- After standardisation, `ESCRAVOS` covers both the terminal's blended rows and the separate unblended condensate row that appears in the 2026 file. They are kept apart by `Stream Type`.

A duplicate check (group by stream, liquid type, stream type and date, then count rows) returned a count of 1 for every row.

## 6. Stream types

`Stream Type` separates how each row is reported in the source:

| Stream Type | Meaning | Where it comes from |
|---|---|---|
| Crude Oil | Crude oil production | Both pages |
| Blended Condensate | Condensate reported with a terminal's crude blend | Page 1 |
| Unblended Condensate | Condensate reported as its own stream | Page 2 |

The source summary blocks use the same blended and unblended split, which is why it was kept.

## 7. Validation

The cleaned data was reconciled against totals published in the NUPRC reports.

**2024 (barrels per month)**

| Check | Result |
|---|---|
| Crude oil total, January 2024 | 44,223,789, equal to the report |
| Blended condensate total, January 2024 | 1,803,495, equal to the report |
| Unblended condensate total, January 2024 | 4,926,520, equal to the report |
| Crude oil total, December 2024 | 46,022,147, equal to the report |

Crude and blended condensate totals matched for every month checked; unblended condensate was checked for January and December. 

**2025:** crude, blended condensate and unblended condensate group-by totals matched the report.

**2026:** crude and condensate totals match the report's daily averages to within **0.01 to 0.02 thousand bpd**. The difference arises because NUPRC rounds each stream to two decimals before summing. For example, February crude is 1,313.68 in the model against 1,313.69 published.

**Daily average cross-check:** the dashboard's average daily production figures match the published daily averages, for example 1,643,671 bpd for January 2024 and 1,667,560 bpd for December 2024.

**Other checks**

- Row counts matched expected values (stream count x months).
- Column quality showed 0% errors on numeric columns after cleaning.
- No duplicate rows for the same stream, liquid type, stream type and date.

## 8. Data model and DAX measures

### Model

- `Production_All`: fact table, one row per stream, stream type and month.
- `Calendar`: monthly date table, one row per month from January 2024 to the latest month in the data.
- Relationship: `Calendar[Date]` to `Production_All[Date]`, many-to-one, single direction.

Columns in `Production_All`: `TERMINAL/STREAM`, `Stream (As Reported)`, `Liquid Type`, `Stream Type`, `Month`, `Year`, `Date`, `Barrels`, and `Data Status` (Published for 2024 and 2025, Provisional for 2026). **[CONFIRM column list]**

### Calendar table

```DAX
Calendar =
ADDCOLUMNS(
    FILTER(
        CALENDAR(DATE(2024,1,1), EOMONTH(MAX(Production_All[Date]),0)),
        DAY([Date]) = 1
    ),
    "Year", YEAR([Date]),
    "Month Number", MONTH([Date]),
    "Month", FORMAT([Date], "MMM")
)
```

`Month` is sorted by `Month Number` so months appear in calendar order.

### Measures

| Measure | Plain-English meaning |
|---|---|
| `Total Barrels` | Sum of barrels in the current filter context |
| `Crude Barrels` | Barrels where stream type is Crude Oil |
| `Condensate Barrels` | Barrels where stream type is not Crude Oil |
| `Days With Data` | Days in the months that have data, so partial years are not divided by 365 |
| `Avg Daily Production (bpd)` | Total barrels divided by days with data |
| `Avg Daily Crude (bpd)` | Crude barrels divided by days with data |
| `Cutoff Month` | Latest month with data (9 for September) |
| `Avg Daily bpd (Jan to Cutoff)` | Average daily production from January to the cutoff month for the selected year |
| `Avg Daily Crude bpd (Jan to Cutoff)` | The same, crude only |
| `Prior Year bpd (Jan to Cutoff)` | The same measure for the previous year |
| `YoY Change %` | Percentage change versus the same months last year |
| `Share of Production %` | A stream's share of total production in the current selection |
| `MoM Change %` | Percentage change versus the previous month, used for matrix colours |

```DAX
Total Barrels = SUM(Production_All[Barrels])

Crude Barrels =
CALCULATE([Total Barrels], Production_All[Stream Type] = "Crude Oil")

Condensate Barrels =
CALCULATE([Total Barrels], Production_All[Stream Type] <> "Crude Oil")

Days With Data =
SUMX(
    VALUES(Production_All[Date]),
    DAY(EOMONTH(Production_All[Date], 0))
)

Avg Daily Production (bpd) = DIVIDE([Total Barrels], [Days With Data])

Avg Daily Crude (bpd) = DIVIDE([Crude Barrels], [Days With Data])

Cutoff Month =
MONTH(CALCULATE(MAX(Production_All[Date]), ALL(Production_All), ALL(Calendar)))

Avg Daily bpd (Jan to Cutoff) =
VAR cutoff = [Cutoff Month]
RETURN
CALCULATE([Avg Daily Production (bpd)], Calendar[Month Number] <= cutoff)

Avg Daily Crude bpd (Jan to Cutoff) =
VAR cutoff = [Cutoff Month]
RETURN
CALCULATE([Avg Daily Crude (bpd)], Calendar[Month Number] <= cutoff)

Prior Year bpd (Jan to Cutoff) =
VAR y = MAX(Calendar[Year])
RETURN
CALCULATE([Avg Daily bpd (Jan to Cutoff)], Calendar[Year] = y - 1)

YoY Change % =
DIVIDE(
    [Avg Daily bpd (Jan to Cutoff)] - [Prior Year bpd (Jan to Cutoff)],
    [Prior Year bpd (Jan to Cutoff)]
)

Share of Production % =
DIVIDE(
    [Total Barrels],
    CALCULATE(
        [Total Barrels],
        ALLSELECTED(Production_All[TERMINAL/STREAM], Production_All[Stream Type])
    )
)

MoM Change % =
VAR cur = [Avg Daily Production (bpd)]
VAR prev =
    CALCULATE(
        [Avg Daily Production (bpd)],
        FILTER(ALL(Calendar), Calendar[Date] = EDATE(MAX(Calendar[Date]), -1))
    )
RETURN
DIVIDE(cur - prev, prev)
```

### Why year-on-year compares January to September

2026 has only nine months. Comparing a partial year with full years would make 2026 look artificially low. All year-on-year figures therefore compare **January to the latest reported month** in each year. Test results from the model: January to September average daily production was 1,520,816 bpd in 2024, 1,664,032 bpd in 2025 (+9.4%) and 1,648,131 bpd in 2026 (-1.0%, provisional).

## 9. Dashboard design decisions

- **Year slicers** use single select with 2026 as the default, so cards never show blended multi-year figures.
- **Trend line chart** is not filtered by the year slicer, so all three years can be compared.
- **Terminals matrix** shows the top 15 streams by volume, sorted by size, with zero-only streams removed.
- **Matrix colours** show month-on-month change, not size. The colour scale is fixed at -20% (red), 0 (white) and +20% (green), so one extreme change cannot wash out the rest. Cells with no previous month, such as January 2024, are uncoloured.
- **Average daily production** is used instead of raw monthly barrels in comparisons, so months of different lengths are comparable.
- **The Checks page** (validation tables) is kept in the file but hidden.

## 10. Caveats and limitations

1. **2026 is provisional** and covers January to September. Figures may be revised by NUPRC.
2. **Bonga, February 2026:** reported output of 2.02 thousand bpd, against 119.40 in January and 91.99 in March. It is shown as published and has not been adjusted. The report does not state a cause, so the dashboard does not claim one.
3. **Rounding:** 2026 source values are rounded to two decimals, so totals can differ from NUPRC's published totals by about 0.01 to 0.02 thousand bpd.
4. **Zero-production streams** (for example Aje and Ima) report zero throughout and were kept in the data but filtered out of the matrix view.
5. **Small streams show large percentage swings** from small absolute changes, so matrix colours for low-volume streams should be read with care.
6. **Terminals, not wells:** NUPRC reports by terminal or stream. Each stream aggregates output from many wells and fields, so the dashboard makes no well-level claims.
7. **Source dependency:** the queries read the downloaded PDFs from a local folder. If the folder moves, the data source paths must be updated in Power Query.
8. **Later data:** NUPRC has published figures for months after September 2026. They are not included in this version.

## 11. Possible next steps

- Add October 2026 and later months as they are published (append a new monthly query; the cutoff measure updates automatically).
- Add a view of production against the OPEC crude quota (1.5 million bpd), which NUPRC's reports reference.
- Add field or operator level detail if a source with that breakdown is available.

## Reproducing the project

1. Download the NUPRC production PDFs into one folder.
2. Open the `.pbix` in Power BI Desktop (Windows).
3. In Power Query, go to **Data source settings** and point each PDF query to your local folder.
4. Refresh. Check row counts against section 3.1 and totals against section 7.
