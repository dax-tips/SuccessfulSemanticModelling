# DAX snippets, master copy

Paste these into the shared login spreadsheet so attendees can copy them without typing.
**This file is the master.** If a cell in the spreadsheet gets edited or mangled during the day,
re-copy from here rather than trying to repair it live.

Written against the models `0-create-lab-models` builds, so table and column names are the real ones:
`Sales`, `Product`, `Customer`, `Territory`, `Date`, and the measures `Total Sales`, `Total Cost`,
`Total Quantity`, `Margin`, `Order Count`. `Date` is already marked as a date table.

---

## Module 4 - column splitting (attendees run this)

Runs against **their own** `01 Star Schema (fixed)`, so no access to the presenter model is needed.
Both numbers must be identical. That is the point.

```dax
EVALUATE
ROW (
    "Fat",   SUM ( Sales[NetAmount] ),
    "Split", SUM ( Sales[NetAmount_Whole] ) + DIVIDE ( SUM ( Sales[NetAmount_Frac] ), 10000 )
)
```

Then, to see why it matters, point VertiPaq Analyzer at the model and compare the size of
`NetAmount` (~2.2M distinct) against `NetAmount_Whole` + `NetAmount_Frac` (~1.6k + 10k).

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

`Sales[OrderNumber]` has **5,000,000 distinct values** over 100M rows here, so the difference is
measurable rather than theoretical.

---

## Module 6 - the Calendar feature

### The calendars are already in the model

`0-create-lab-models` builds both of them on `'Date'` via TOM, so there is nothing to paste. Open
`06 DAX + Calendar`, **Open data model**, then **TMDL view** and read them. The full definition, the
reasoning behind every column choice, and the DMV check are in
[module6-calendar.md](module6-calendar.md).

Verified against TOM and real TMDL on 23 Aug:

- A calendar is `calendar <Name>`, nested under `table`. A table can hold several.
- Each group is `calendarColumnGroup = <unit>` with a `primaryColumn` and optional
  `associatedColumn` lines. In TOM the group class is `TimeUnitColumnAssociation`, but the TMDL
  serialisation shows no such distinction.
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

### Checking the mapping is right, rather than hoping

`'Date'[PriorYearDateKey]` is precomputed by the generator for the row-based TI demo, so it is a
known-good answer to compare the calendar against. These two columns must agree:

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'Date'[Year],
    "Calendar LY",  [Sales LY],
    "RowBased LY",  CALCULATE ( [Sales],
                        TREATAS ( VALUES ( 'Date'[PriorYearDateKey] ), 'Date'[DateKey] ) )
)
```

### The DAX side - CLOSED 23 Aug, verified on the tenant

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

### The demo query: three notions of "last year", one measure shape

```dax
EVALUATE
SUMMARIZECOLUMNS (
    'Date'[Year],
    "Sales",      [Total Sales],
    "LY classic", CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( 'Date'[Date] ) ),
    "LY Greg",    CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( Gregorian ) ),
    "LY Fiscal",  CALCULATE ( [Total Sales], SAMEPERIODLASTYEAR ( Fiscal ) )
)
ORDER BY 'Date'[Year]
```

### Classic and calendar TI agree, with one documented exception

Measured 23 Aug on this model: `SAMEPERIODLASTYEAR ( 'Date'[Date] )` and
`SAMEPERIODLASTYEAR ( 'Gregorian' )` returned **identical values to the cent** on every row. Nothing
existing breaks.

The documented divergence is at date grain across a leap boundary. Shifting **29 Feb 2008** back one
year gives **1 Mar 2007** under calendar TI, because it is the 60th day of the year, but
**28 Feb 2007** under classic. For a 13-month calendar the documented workaround is
`DATEADD ( Calendar, -13, month )`.

This is why the primary column mattered. The docs state calendar input returns "all primary tagged
columns and all time related columns", so the primary is the column being shifted.

### The trap: the calendar must match the column you group by

Group by `'Date'[Year]` but ask for `SAMEPERIODLASTYEAR ( 'Fiscal' )` and you get roughly **double**
the expected value. No error, no warning. Measured on this model, whose fiscal year starts 1 July:

| Gregorian year | fiscal years spanned | shifted back one | months with data | measured |
|---|---|---|---|---|
| 2021 | FY21 + FY22 | FY20 + FY21 | 6 | 47.42M |
| 2022 | FY22 + FY23 | FY21 + FY22 | 18 | 143.49M |
| 2023+ | FY23 + FY24 | FY22 + FY23 | 24 | 192.0M |

A Gregorian year straddles two fiscal years, so shifting each back one fiscal year produces a
**24-month window**. The result is arithmetically correct and semantically meaningless.

This is the failure mode the feature will cause in the wild: a report author picks a date field from
a slicer, a measure author picked a calendar months earlier, and nothing connects the two.

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

