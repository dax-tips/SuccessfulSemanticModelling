# DAX snippets, master copy

Paste these into the shared login spreadsheet so attendees can copy them without typing.
**This file is the master.** If a cell in the spreadsheet gets edited or mangled during the day,
re-copy from here rather than trying to repair it live.

Written against the models `0-create-lab-models` builds, so table and column names are the real ones:
`Sales`, `Product`, `Customer`, `Territory`, `Date`, and the measures `Total Sales`, `Total Cost`,
`Total Quantity`, `Margin`, `Order Count`. `Date` is already marked as a date table.

---

## Module 4 - scaling techniques

Three levers, three presenter models. **You can run every query below in your own DAX Perf
Optimizer notebook**, but tracing needs admin on the model, and you are Viewer on the presenter's
workspace. So levers 1 and 2 are watch-along, and the last query in lever 3 runs on your own model.

Read the **Trace details** grid, not the clock.

### Lever 1: aggregations

Model: `04 Scaling - Aggregation`. `Sales_agg` is grouped on `OrderDateKey` x `ProductKey` and has
**no CustomerKey**. Watch the `Rewrite Attempted` row: `matchFound` versus `attemptFailed`.

**1a. The hit.** Grouping by date, which the agg holds. Expect `matchFound`, and the scan reads
`Sales_agg` rather than the 3M-row fact.

```dax
EVALUATE
SUMMARIZECOLUMNS ( 'date'[MonthYear], "Total Sales", [Total Sales] )
```

**1b. The hit again, at product grain.** `Brand` lives on the product dimension, which `Sales_agg`
relates to, so the rewrite still lands. It is the **grain** that matters, not the literal column.

```dax
EVALUATE
SUMMARIZECOLUMNS ( 'product'[Brand], "Total Sales", [Total Sales] )
```

**1c. The miss, and the headline of the whole lever.** Same measure, grouped by customer.
`Sales_agg` has no customer column, so the rewrite is attempted and rejected and the query falls
through to the DirectQuery fact. Expect `attemptFailed`, `failureReasons` naming
`customer[Segment]`, and an SQL row appearing.

```dax
EVALUATE
SUMMARIZECOLUMNS ( 'customer'[Segment], "Total Sales", [Total Sales] )
```

Nothing is broken. Anyone can demo a win; showing an aggregation that **cannot** help is what stops
people building one nobody's queries can use.

**1d. Optional second miss.** Only if someone asks whether it was something specific about `Segment`.

```dax
EVALUATE
SUMMARIZECOLUMNS ( 'customer'[City], "Total Sales", [Total Sales] )
```

### Lever 2: hybrid tables

Model: `04 Scaling - Hybrid`. `sales-hot` is **Import** (`OrderDateKey >= 20260101`), `sales-cold` is
**DirectQuery** (older), and the cold partition carries
`dataCoverageDefinition: RELATED('date'[DateKey]) < 20260101`.

The coverage definition sits on the **date dimension**, because a range predicate has to sit on a
dual table. So the query has to constrain `'date'` too.

Each query below is complete, copy and paste it whole. **Only the filter line differs between them**,
which is the entire lesson: partition elimination depends on *how* you write the filter, not on having
partitions. Watch whether **SQL rows appear** in the trace.

**2a. Hot slice only.** Expect **no SQL rows**.

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'date'[MonthYear],
    'product'[Brand],
    FILTER ( VALUES ( 'date'[DateKey] ), 'date'[DateKey] >= 20260101 ),
    "Total Sales", [Total Sales]
)
```

**2b. The same filter, wrong table.** Run straight after 2a. Logically identical, returns exactly the
same numbers, but it constrains the **fact**. The coverage definition is written against
`'date'[DateKey]`, so the engine has nothing to match and reads the DirectQuery partition anyway.
Expect **SQL rows**. The filter looks perfectly reasonable, which is exactly why this catches people.

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'date'[MonthYear],
    'product'[Brand],
    FILTER ( VALUES ( 'sales'[OrderDateKey] ), 'sales'[OrderDateKey] >= 20260101 ),
    "Total Sales", [Total Sales]
)
```

**2c. History.** Same column as 2a, opposite direction. The query really does need the cold half.
Expect **SQL rows**. Correct behaviour, and worth saying out loud before someone reads it as a fault.

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'date'[MonthYear],
    'product'[Brand],
    FILTER ( VALUES ( 'date'[DateKey] ), 'date'[DateKey] < 20260101 ),
    "Total Sales", [Total Sales]
)
```

**2d. The DATE() trap.** Dwell here. Logically identical to 2a, on the right column, and it returns
the **right answer**. But `DATE()` yields a DATETIME, held as a DOUBLE (46023.0), so comparing it to
an int64 key promotes the whole comparison to floating point. `>= 46023` is true for every yyyymmdd
key, so it filters nothing **and** the coverage definition can no longer be matched. Expect **SQL
rows**, and look for the `.000000` tail in the xmSQL. Compare integer to integer.

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'date'[MonthYear],
    'product'[Brand],
    FILTER ( VALUES ( 'date'[DateKey] ), 'date'[DateKey] >= DATE ( 2026, 1, 1 ) ),
    "Total Sales", [Total Sales]
)
```

**2e. The good one, and the point of the lever.** No filter on the fact, and none on `DateKey`
either. `'date'` is a **dual** table, so the engine resolves `Year = 2026` in memory, turns it into a
range of DateKeys, and concludes the cold partition cannot contribute. Expect **no SQL rows**.

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'date'[MonthYear],
    'product'[Brand],
    TREATAS ( { 2026 }, 'date'[Year] ),
    "Total Sales", [Total Sales]
)
```

Put 2e next to 2b: one filters the fact and fails, one never mentions `DateKey` and succeeds. It is
about which table the engine can reason over, not which column you happened to type.

### Lever 3: column splitting

Model: `04 Scaling - Split Columns`. Three tables, the same 3,000,000 rows, the same answer.

**3a. Prove the split is lossless.** `Difference` must be **exactly 0**. A blank means the Import
tables are empty.

```dax
EVALUATE
ROW (
    "Fat",         [Net Sales (fat)],
    "Fat no hier", [Net Sales (fat, no hierarchy)],
    "Split",       [Net Sales (split)],
    "Difference",  [Difference]
)
```

Then open VertiPaq Analyzer and sort by Total Size. Walk **Cardinality, then Hier Size, then
Dictionary, then Total**, in that order, so the mechanism lands before the headline number. Hier Size
falls 8,964,792 to 46,640 (192x) and cardinality falls 2,241,197 to 11,657 (also 192x). Hier Size
*is* the sorted index, so it tracks cardinality almost exactly.

Be honest about where the win comes from. Most of the fat column is that hierarchy, and it can be
removed for nothing with `isAvailableInMDX = false`. Turning the hierarchy off is ~1.7x and costs you
nothing; splitting the column is 3.45x and costs you a measure and a refresh. Data barely moves
(12.0 MB to 6.0 MB), because you still store one value per row whatever the cardinality. **Low
cardinality buys you the index, not the data.**

**3b. This one runs on YOUR model.** `01 Star Schema (fixed)`, or the `My Star Schema` you built in
Lab 1b, no presenter access needed, because your `Sales` table carries the same three columns.
`Difference` should be zero **to the cent**. A residue around `0.0000002` is expected and is not data
loss: `Fat` sums three million doubles, so the answer depends on summation order, while `Split` is
integer maths and only rounds once at the end.

```dax
EVALUATE
ADDCOLUMNS (
    ROW (
        "Fat",   SUM ( Sales[NetAmount] ),
        "Split", SUM ( Sales[NetAmount_Whole] ) + DIVIDE ( SUM ( Sales[NetAmount_Frac] ), 10000 )
    ),
    "Difference", [Fat] - [Split]
)
```

Then point VertiPaq Analyzer at your own model and compare the size of `NetAmount` (~2.2M distinct)
against `NetAmount_Whole` + `NetAmount_Frac` (~1.6k + 10k).

---

## Module 6 - the five patterns

Model: `06 DAX + Calendar`. Add these as measures via web modelling or the TMDL editor.

### 1. Period comparison

```dax
Sales LY = CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )
```

```dax
Sales YoY % =
VAR Curr = [Total Sales]
VAR Prior = [Sales LY]
RETURN DIVIDE ( Curr - Prior, Prior )
```

Watch: `SAMEPERIODLASTYEAR` needs a **marked date table**. It already is in this model, so if someone
builds their own star in Module 1 and skips that step, this is where it bites them.

### 1b. Role-playing dates - using the inactive relationship

You built this relationship in Module 1: `Sales[ShipDateKey]` to `'Date'[DateKey]`, **inactive**.
This is how you use it.

```dax
Sales by Ship Date =
CALCULATE (
    [Total Sales],
    USERELATIONSHIP ( Sales[ShipDateKey], 'Date'[DateKey] )
)
```

Ordered against shipped, side by side, off one `Date` table:

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'Date'[MonthYearSort],
    'Date'[MonthYear],
    "Ordered", [Total Sales],
    "Shipped", CALCULATE ( [Total Sales],
                           USERELATIONSHIP ( Sales[ShipDateKey], 'Date'[DateKey] ) )
)
ORDER BY 'Date'[MonthYearSort]
```

The teaching points:

- `USERELATIONSHIP` swaps which relationship is active **for that one `CALCULATE` only**. It changes
  nothing about the model, so the two columns above coexist in the same query.
- Both columns read the **same** `Date` table. The alternative is a second date dimension
  (`ShipDate`), which costs memory, has to be kept in step, and forces every measure and every slicer
  to pick a side. One conformed dimension with a spare inactive relationship is nearly always better.
- Ship date is order date plus 0 to 7 days, so `Shipped` lags `Ordered` and the gap shows at month
  boundaries. **Grand totals are identical**: same rows, different date attribution.
- The generator clamps ship dates to the last day of the calendar, so the final month is slightly
  inflated. That is an artefact of the data, not a lesson.

Watch: `USERELATIONSHIP` needs the relationship to **exist and be inactive**. If Module 1 step 3 was
skipped, or the relationship was left active, this errors rather than quietly giving you the wrong
number, which is the kinder of the two failure modes.

### 2. Running total

```dax
Sales YTD = CALCULATE ( [Total Sales], DATESYTD ( 'Date'[Date] ) )
```

```dax
Sales Running Total =
CALCULATE (
    [Total Sales],
    'Date'[Date] <= MAX ( 'Date'[Date] ),
    ALL ( 'Date' )
)
```

The second is the general form, no date-table magic. Worth showing both: one is a helper, one is the
mechanism underneath it.

### 3. Child to parent ratio

```dax
Share of Category =
DIVIDE (
    [Total Sales],
    CALCULATE ( [Total Sales], ALLEXCEPT ( Product, Product[Category] ) )
)
```

Watch: `ALLEXCEPT` keeps `Category` and drops every other filter on `Product`. Swap it for `ALL` and
the denominator becomes the whole model, which is the mistake people actually make.

### 4. Ranking

```dax
Product Rank =
RANKX ( ALL ( Product[ProductName] ), [Total Sales],, DESC, DENSE )
```

```dax
Rank in Category =
RANKX (
    CALCULATETABLE ( VALUES ( Product[ProductName] ), ALLEXCEPT ( Product, Product[Category] ) ),
    [Total Sales],, DESC, DENSE
)
```

Watch: `RANKX` over `ALL(Product)` ranks against every column combination, not just the name. Rank
the **column** you are displaying.

### 5. Distinct count - leave this one last

```dax
Orders (distinct) = DISTINCTCOUNT ( Sales[OrderNumber] )
```

```dax
Orders (countrows) = COUNTROWS ( VALUES ( Sales[OrderNumber] ) )
```

```dax
Orders (approx) = APPROXIMATEDISTINCTCOUNT ( Sales[OrderNumber] )
```

All three answer the same question on this data. The teaching points:

- `COUNTROWS ( VALUES ( ... ) )` is safe **when the column has no blanks**, and often cheaper. On a
  column with blanks it differs from `DISTINCTCOUNT`, which is why it is not a blanket substitute.
- `APPROXIMATEDISTINCTCOUNT` trades exactness for speed. Legitimate on a high-cardinality column feeding
  a headline tile that nobody reconciles to the penny. Not legitimate on anything a finance team signs.
- Note the function is `APPROXIMATEDISTINCTCOUNT`, not `APPROXDISTINCTCOUNT`.

`Sales[OrderNumber]` is drawn from 1,000,000 possible order numbers across 3M rows, so roughly **1M
distinct**, about three lines per order. Enough cardinality that the difference is measurable rather
than theoretical.

---

## Module 6 - the Calendar feature

### The calendars are already in the model

`0-create-lab-models` builds both of them on `'Date'`, so there is nothing to paste. Open
`06 DAX + Calendar`, **Open data model**, then **TMDL view**, and read them there. `Gregorian` has 7
column groups, `Fiscal` has 6.

Verified against TOM and real TMDL on 23 Aug:

- A calendar is `calendar <Name>`, nested under `table`. A table can hold several.
- Each group is `calendarColumnGroup = <unit>` with a `primaryColumn` and optional
  `associatedColumn` lines.
- **Time units serialise lowercase**, compound units camelCase: `year`, `month`, `date`,
  `monthOfYear`, `quarterOfYear`, `dayOfWeek`, `dayOfMonth`. The API examples say `Years` and the
  parameter docs say `Year`. Both are wrong.
- Requires compatibility level 1701 or higher.

### Why each column sits where it does

The primary column has to **identify one specific instance of the unit**. `Month` is 1-12 and
repeats every year, so it cannot be the primary for `month`; `MonthYearSort` (202101) can. `Month`
goes to `monthOfYear` instead. The same test puts `Quarter` and `FiscalQuarter` in `quarterOfYear`
rather than `quarter`, because there is no running-quarter column on this table.

The Fiscal calendar has **no `monthOfYear` group** on purpose. This fiscal year starts 1 July, so
January is not fiscal month 1, and there is no fiscal month column to point at. For the same reason
it omits `Year`, `Quarter` and `QuarterName`: all three assert Gregorian meanings that are false
under a July start.

### How to reference a calendar in DAX

**A calendar is referenced exactly like a table.** No table qualifier and no brackets. Single quotes
are optional for a simple name and required if it contains spaces, the same rule as `Sales` versus
`'Sales'`. Both forms below are valid. `'Date'.Gregorian` is not, and errors at the dot.

```dax
CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Gregorian' ) )
CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( Fiscal ) )
```

Worth keeping calendar names free of spaces so attendees never have to think about the quoting.

Existing measures are untouched. `SAMEPERIODLASTYEAR ( 'Date'[Date] )` still works exactly as before,
with two calendars defined, and returns classic results. Calendar time intelligence is **opt-in per
call**, so nobody with a hundred existing measures has to rewrite anything. That is the migration
story, and it is a good one.

### Queries to run

Open the **DAX Perf Optimizer** notebook, run both cells, pick `06 DAX + Calendar` in the picker,
then paste each query below. You get the result and the timings together.

#### 1. Nothing existing breaks

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'Date'[Year],
    "Sales",        [Total Sales],
    "LY classic",   CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Date'[Date] ) ),
    "LY Gregorian", CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Gregorian' ) )
)
ORDER BY 'Date'[Year]
```

The two LY columns come back **identical**. Measured 23 Aug on this model: identical to the cent on
every row. Classic time intelligence keeps working exactly as it did, so defining a calendar costs no
rewrites and there is nothing to migrate.

#### 2. The year boundary moves - the one worth looking at

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'Date'[MonthYearSort],
    'Date'[MonthYear],
    "Sales",      [Total Sales],
    "YTD Greg",   TOTALYTD ( [Total Sales], 'Gregorian' ),
    "YTD Fiscal", TOTALYTD ( [Total Sales], 'Fiscal' )
)
ORDER BY 'Date'[MonthYearSort]
```

Two running totals that reset at different points: **Gregorian resets in January, Fiscal resets in
July**. Two sawtooths, six months out of phase.

#### 3. What the calendar actually replaces

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'Date'[MonthYearSort],
    'Date'[MonthYear],
    "YTD Fiscal old", TOTALYTD ( [Total Sales], 'Date'[Date], "6/30" ),
    "YTD Fiscal new", TOTALYTD ( [Total Sales], 'Fiscal' )
)
ORDER BY 'Date'[MonthYearSort]
```

These match. Fiscal YTD was always reachable, by passing a year-end string that had to be retyped in
every measure by whoever remembered it. The calendar puts that definition in the model, once, under a
name.

#### 4. The trap - the best demo in the module

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'Date'[Year],
    "Sales",     [Total Sales],
    "LY Greg",   CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Gregorian' ) ),
    "LY Fiscal", CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Fiscal' ) )
)
ORDER BY 'Date'[Year]
```

`LY Fiscal` comes back at roughly **double** `LY Greg`. No error, no warning, and the number looks
plausible. Measured on this model, whose fiscal year starts 1 July:

| Gregorian year | fiscal years spanned | shifted back one | months with data | measured |
|---|---|---|---|---|
| 2021 | FY21 + FY22 | FY20 + FY21 | 6 | 47.42M |
| 2022 | FY22 + FY23 | FY21 + FY22 | 18 | 143.49M |
| 2023+ | FY23 + FY24 | FY22 + FY23 | 24 | 192.0M |

You grouped by Gregorian year and asked for the previous **fiscal** year. A Gregorian year straddles
two July-start fiscal years, so shifting each back one produces a **24-month window**. The result is
arithmetically correct and semantically meaningless.

> **The calendar you name must match the calendar of the column you group by.**

This is the failure mode the feature will cause in the wild: a report author picks a date field from
a slicer, a measure author picked a calendar months earlier, and nothing connects the two.

### One documented difference to know about

Classic and calendar time intelligence disagree at date grain across a leap boundary. Shifting
**29 Feb 2008** back one year gives **1 Mar 2007** under calendar time intelligence, because it is
treated as the 60th day of the year, but **28 Feb 2007** under classic. For a calendar with a
non-standard number of months the documented workaround is `DATEADD ( Calendar, -13, month )`.

This is why the primary column mattered: the docs state calendar input returns "all primary tagged
columns and all time related columns", so the primary is the column being shifted.

### Verifying a calendar landed, with no tooling

Two DMVs, both confirmed working, so attendees can check their own work without Tabular Editor:

```
SELECT * FROM $SYSTEM.TMSCHEMA_CALENDARS
SELECT * FROM $SYSTEM.TMSCHEMA_CALENDAR_COLUMN_GROUPS
```

`TimeUnit` is an integer. Decoded on this model: **1** Year, **5** QuarterOfYear, **7** Month,
**8** MonthOfYear, **16** Date, **20** DayOfMonth, **21** DayOfWeek.

> Note the DMV `DESCRIPTION` strings in `$SYSTEM.MDSCHEMA_FUNCTIONS` still predate this feature.
> `SAMEPERIODLASTYEAR` reads as though it has no calendar overload. It does. Trust the docs.

### Rule the engine enforces

A column must carry the **same time unit in every calendar on the table**. Mapping `Month` as
`Month` in one calendar and `MonthOfYear` in another is rejected on commit with "a column has
inconsistent TimeUnit associations in multiple calendar objects owned by table 'Date'".

