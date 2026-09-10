# Power BI Date Layer

Goal: one slicer that lets HR pick a named period (Last 7 Days, Last Month, Last 3 Months...)
and have every visual, including a row-level detail table, follow it.

## Model

```
Leave Requests[Submission Date]  ──►  Dim Date[Date]      (active relationship)
SpecialDates                          (no relationship; disconnected on purpose)
```

## 1. Date table

```DAX
Dim Date = CALENDARAUTO()
```

`CALENDARAUTO()` scans the model's date columns and builds a complete range.
Using `CALENDAR(MIN(...), MAX(...))` with nested column references kept erroring,
and `CALENDARAUTO()` removed the problem entirely.

Supporting columns: Year, Month No, Month, Month Year, plus flag columns
(for example, Last 3 Months) for rolling-period visuals.

## 2. Period table (disconnected)

One row per (Period, Date) pair, built by stacking one block per period:

```DAX
SpecialDates =
VAR _Today = TODAY()
RETURN
UNION (
    SELECTCOLUMNS (
        FILTER ( 'Dim Date', 'Dim Date'[Date] > _Today - 7 && 'Dim Date'[Date] <= _Today ),
        "Period", "Last 7 Days", "Date", 'Dim Date'[Date]
    ),
    SELECTCOLUMNS (
        FILTER ( 'Dim Date', 'Dim Date'[Date] > _Today - 30 && 'Dim Date'[Date] <= _Today ),
        "Period", "Last 30 Days", "Date", 'Dim Date'[Date]
    ),
    SELECTCOLUMNS (
        FILTER ( 'Dim Date',
            'Dim Date'[Date] > EDATE ( _Today, -3 ) && 'Dim Date'[Date] <= _Today ),
        "Period", "Last 3 Months", "Date", 'Dim Date'[Date]
    )
    -- ...one block per period (11 in total)
)
```

`SELECTCOLUMNS` keeps every block's columns identically named and ordered, which
`UNION` requires.

## 3. Measures

Apply the selected period's dates to the date table at query time:

```DAX
Requests in Period =
CALCULATE (
    COUNTROWS ( 'Leave Requests' ),
    TREATAS ( VALUES ( SpecialDates[Date] ), 'Dim Date'[Date] )
)
```

Make the detail table follow the slicer:

```DAX
In Selected Period =
IF ( [Requests in Period] > 0, 1, 0 )
```

Add `In Selected Period` to the detail table's visual-level filters, set to **is 1**.

## The trap

Power BI may auto-detect a relationship between `SpecialDates[Date]` and
`Dim Date[Date]`. **Delete it.** With a live relationship, `TREATAS` is fighting a filter
that already exists, and the period slicer returns wrong or empty results.
