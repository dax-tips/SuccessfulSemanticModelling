# Module 6 - the Calendar feature

Copy/paste material for Lab 6. Everything here is verified against the workshop model.

---

## 1. Add two calendars to the `Date` table

Open the `06 DAX + Calendar` model, **Open data model**, then the **TMDL view**.

Paste the block below at the **end of the `table Date` definition**, after the `partition` block.
Indentation matters: `calendar` sits at **one tab**, `calendarColumnGroup` at **two**, and the
column lines at **three**.

```tmdl
	calendar Gregorian

		calendarColumnGroup = year
			primaryColumn: Year

		calendarColumnGroup = quarterOfYear
			primaryColumn: Quarter
			associatedColumn: QuarterName

		calendarColumnGroup = month
			primaryColumn: MonthYearSort
			associatedColumn: MonthYear

		calendarColumnGroup = monthOfYear
			primaryColumn: Month
			associatedColumn: MonthName

		calendarColumnGroup = dayOfWeek
			primaryColumn: DayOfWeek
			associatedColumn: DayName

		calendarColumnGroup = dayOfMonth
			primaryColumn: Day

		calendarColumnGroup = date
			primaryColumn: Date
			associatedColumn: DateKey

	calendar Fiscal

		calendarColumnGroup = year
			primaryColumn: FiscalYear

		calendarColumnGroup = quarterOfYear
			primaryColumn: FiscalQuarter

		calendarColumnGroup = month
			primaryColumn: MonthYearSort
			associatedColumn: MonthYear

		calendarColumnGroup = dayOfWeek
			primaryColumn: DayOfWeek
			associatedColumn: DayName

		calendarColumnGroup = dayOfMonth
			primaryColumn: Day

		calendarColumnGroup = date
			primaryColumn: Date
			associatedColumn: DateKey
```

### Why these columns and not others

The primary column has to **identify one specific instance of the unit**.

- `MonthYearSort` is 202101, unique per month, so it is the `month` primary.
- `Month` is 1-12 and repeats every year, so it is `monthOfYear`, not `month`.
- Same test puts `Quarter` and `FiscalQuarter` in `quarterOfYear`, not `quarter`. There is no
  running-quarter column on this table.

The Fiscal calendar has **no `monthOfYear` group** on purpose. This fiscal year starts 1 July, so
January is not fiscal month 1, and there is no fiscal month column to point at.

### Two rules the engine enforces

1. A column must carry the **same time unit in every calendar** on the table. That is why `Month`
   cannot be `month` in one calendar and `monthOfYear` in another.
2. A column can be the primary of the **same** unit in several calendars. `Date`, `MonthYearSort`,
   `DayOfWeek` and `Day` are all shared above.

### Check it landed

Two DMV queries, no tooling needed:

```
SELECT * FROM $SYSTEM.TMSCHEMA_CALENDARS
SELECT * FROM $SYSTEM.TMSCHEMA_CALENDAR_COLUMN_GROUPS
```

`TimeUnit` is an integer: **1** year, **5** quarterOfYear, **7** month, **8** monthOfYear,
**16** date, **20** dayOfMonth, **21** dayOfWeek.

---

## 2. Reference a calendar in DAX

A calendar is referenced **like a table**. No table qualifier, no square brackets. Quotes are
optional unless the name contains a space.

```dax
SAMEPERIODLASTYEAR ( 'Gregorian' )
TOTALYTD ( [Total Sales], 'Fiscal' )
```

`'Date'.Gregorian` is **not** valid and errors at the dot.

---

## 3. Queries to run

Open the **DAX Perf Optimizer** notebook, run both cells, then pick `06 DAX + Calendar` in the
picker and paste each query below into it. You get the result and the timings together.

### 3a. Nothing existing breaks

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

The two LY columns should be **identical**. Classic time intelligence keeps working exactly as it
did, so defining a calendar costs you no rewrites.

### 3b. The year boundary moves - the one worth looking at

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

### 3c. What the calendar actually replaces

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

These should match. Fiscal YTD was always reachable, by passing a year-end string that had to be
retyped in every measure by whoever remembered it. The calendar puts that definition in the model,
once, under a name.

### 3d. The trap

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

`LY Fiscal` comes back at roughly **double** `LY Greg`. It is not a bug. You grouped by Gregorian
year and asked for the previous **fiscal** year, and a Gregorian year straddles two July-start fiscal
years, so shifting each back one produces a 24-month window.

There is no error and no warning, and the number looks plausible.

> **The calendar you name must match the calendar of the column you group by.**

---

## 4. One documented difference to know about

Classic and calendar time intelligence disagree at date grain across a leap boundary. Shifting
**29 Feb 2008** back one year gives **1 Mar 2007** under calendar time intelligence, because it is
treated as the 60th day of the year, but **28 Feb 2007** under classic. For a calendar with a
non-standard number of months the documented workaround is `DATEADD ( Calendar, -13, month )`.
