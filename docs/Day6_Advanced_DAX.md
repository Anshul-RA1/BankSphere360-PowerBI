# Day 6 — Advanced DAX: Time Intelligence, RANKX & Performance

> Project: **BankSphere 360** — Power BI Banking Analytics
> Stage in lifecycle: requirements → source analysis → ETL → data model → **advanced measures (you are here)** → visuals → security → deploy → support
> Goal of the day: build time-based KPIs (YTD, YoY, MoM), ranking, fraud analysis, resolve a circular dependency, and understand Performance Analyzer.

---

## 1. The DPD sort-order problem — and the circular dependency lesson

### The problem

`DPD Bucket` (text) sorts **alphabetically** by default in visuals:
Current → Doubtful → Loss → Substandard → Watch

That's nonsense as a risk progression. The correct order is:
Current → Watch → Substandard → Doubtful → Loss (1 → 2 → 3 → 4 → 5)

### First attempt — DAX calculated column

Created `DPD Bucket Order` as a DAX calculated column using `SWITCH`:

```dax
DPD Bucket Order =
SWITCH (
    FactLoans[DPD Bucket],
    "Current (0 DPD)",          1,
    "Watch (1-30 DPD)",         2,
    "Substandard (31-60 DPD)",  3,
    "Doubtful (61-90 DPD)",     4,
    "Loss / NPA (90+ DPD)",     5,
    99
)
```

Then tried to set DPD Bucket → Sort by Column → DPD Bucket Order.

**Result: circular dependency error:**

> _"A circular dependency was detected: FactLoans[DPD Bucket], FactLoans[DPD Bucket Order], FactLoans[DPD Bucket]"_

### Why the error occurs

Power BI's sort-by creates a dependency: DPD Bucket's order depends on DPD Bucket Order, which is a DAX column that references DPD Bucket. The engine sees: DPD Bucket → depends on → DPD Bucket Order → references → DPD Bucket. A loop. Power BI blocks it.

**Known limitation:** you cannot sort a column by a DAX calculated column that references it in the same table.

### The fix — Power Query column

Deleted the DAX column. Recreated `DPD Bucket Order` in **Power Query** (Add Column → Conditional Column) as a numeric output (1–5, 99 default), typed as Whole Number. Applied via Close & Apply.

Power Query columns are evaluated at **load time**, independently of the DAX layer. No circular dependency possible.

Then: Column tools → **Sort by column** → DPD Bucket Order. Applied cleanly.

**Interview answer:** _"You can't sort a column by a DAX calculated column that references it in the same table — it creates a circular dependency. The fix is to build the sort-key in Power Query, where it's evaluated at load time independently of DAX. Text is for display, the number is for ordering — the same principle as sorting month names by their month number."_

---

## 2. The foundational concept — how time intelligence works

Time-intelligence functions take whatever date filter is **currently active** and **shift it** to a different period. They require a **marked date table** with continuous, unbroken dates (which is why DimDate was marked on Day 4).

**Mental model:**

```
User selects "March 2024" on a slicer
↓
Current filter context = March 1–31, 2024
↓
SAMEPERIODLASTYEAR shifts it → March 1–31, 2023
↓
Base measure recalculates over that shifted period
↓
Result = March 2023's value → used to compute YoY growth
```

The three functions used today:

| Function             | What it does                           | Shifts to                  |
| -------------------- | -------------------------------------- | -------------------------- |
| `TOTALYTD`           | Accumulates from Jan 1 to current date | Jan 1 → current date       |
| `SAMEPERIODLASTYEAR` | Same period, 12 months earlier         | Current window − 12 months |
| `DATEADD`            | Custom shift by any interval and unit  | Configurable               |

---

## 3. Base measures — built first, wrapped by time intelligence

### Total Txn Value

```dax
Total Txn Value = SUM ( FactTransactions[Amount] )
```

Sum of all transaction amounts. No CALCULATE needed — no filter modification required. **Result: ₹15bn.** The base measure that all three time-intelligence measures wrap.

### Total Transactions

```dax
Total Transactions = COUNTROWS ( FactTransactions )
```

Count of transaction rows. **Result: 60,000** — matching the source count exactly (reconciliation confirmation). Whole Number, thousands separator.

---

## 4. The three time-intelligence measures — deep dive

### Measure A: Deposits YTD using `TOTALYTD`

```dax
Deposits YTD =
TOTALYTD ( [Total Deposits], DimDate[Date] )
```

**What TOTALYTD does:**
It builds a date filter from **January 1st of the current year up to the last date in the current filter context** and evaluates the base measure over that window.

**Concrete example on a monthly line chart:**

| Month    | Total Deposits (monthly) | Deposits YTD |
| -------- | ------------------------ | ------------ |
| Jan 2024 | ₹62M                     | ₹62M         |
| Feb 2024 | ₹58M                     | ₹120M        |
| Mar 2024 | ₹71M                     | ₹191M        |
| Apr 2024 | ₹65M                     | ₹256M        |
| Dec 2024 | ₹68M                     | ₹794M        |

Each month adds to the prior cumulative. Resets to zero on January 1st.

**Key arguments:**

- `[Total Deposits]` — the measure to accumulate (required)
- `DimDate[Date]` — the Date column from the marked table (required)
- Optional third argument: custom year-end. For Indian financial year ending March 31: `TOTALYTD([Total Deposits], DimDate[Date], "03-31")`

**On a card with no slicer:** shows ₹794M — same as Total Deposits — because "year to date" covering the full dataset accumulates to the total. Not a bug; it shows its value in time-series visuals.

**Result: ₹794M (full dataset, no slicer)**
**Format: Currency (₹), 0 decimals**

---

### Measure B: Txn Value LY + Txn YoY% using `SAMEPERIODLASTYEAR`

#### Step 1 — the building block

```dax
Txn Value LY =
CALCULATE (
    [Total Txn Value],
    SAMEPERIODLASTYEAR ( DimDate[Date] )
)
```

**What SAMEPERIODLASTYEAR does:**
It is a **table function** that returns the set of dates exactly 12 months before the current filter context. CALCULATE applies that date set as the new filter, and the base measure recalculates over it.

**Concrete example on a monthly line chart:**

| Month on visual | Current filter | Shifted to   | Txn Value LY |
| --------------- | -------------- | ------------ | ------------ |
| Jan 2024        | Jan 2024       | Jan 2023     | ₹620M        |
| Feb 2024        | Feb 2024       | Feb 2023     | ₹580M        |
| Q1 2024         | Jan–Mar 2024   | Jan–Mar 2023 | ₹1.91bn      |
| Full Year 2024  | Jan–Dec 2024   | Jan–Dec 2023 | ₹8bn         |

Works for any window shape — a day, a month, a quarter, a year. Always shifts by exactly 12 months.

**Important:** `SAMEPERIODLASTYEAR` is a table function, not a scalar. It returns a set of dates, not a number. CALCULATE receives that set as a filter argument. This is why it must sit inside CALCULATE.

**On a card with no slicer:** returns 2023's full transaction value (~₹8bn). With no filter, "last year" = the year prior to the last date in the dataset.

**Result: ₹8bn (full dataset, no slicer)**

#### Step 2 — the actual KPI

```dax
Txn YoY % =
DIVIDE (
    [Total Txn Value] - [Txn Value LY],
    [Txn Value LY]
)
```

- Numerator: Current − Last Year = the change
- Denominator: Last Year = the base
- Result: growth rate

**Concrete example:**

> March 2024 Txn Value = ₹710M
> March 2023 Txn Value = ₹590M
> YoY% = (710 − 590) / 590 = **+20.3%**

**On a card with no slicer:** 99.68% — comparing full 2024 vs full 2023. On a line chart this shows each month's growth vs the same month last year.

**Result: 99.68% (full dataset, no slicer)**
**Format: Percentage, 2 decimals**

---

### Measure C: Txn MoM% using `DATEADD` + `VAR`

```dax
Txn MoM % =
VAR CurrentVal = [Total Txn Value]
VAR PrevVal =
    CALCULATE (
        [Total Txn Value],
        DATEADD ( DimDate[Date], -1, MONTH )
    )
RETURN
    DIVIDE ( CurrentVal - PrevVal, PrevVal )
```

**What DATEADD does:**
DATEADD is the flexible time shifter. It takes three arguments:

1. The date column: `DimDate[Date]`
2. The interval (signed integer): `-1` means "go back 1 unit"; use `-3` for three units back, `+1` to look forward
3. The unit: `DAY`, `MONTH`, `QUARTER`, or `YEAR`

So `DATEADD(DimDate[Date], -1, MONTH)` shifts the current date filter back 1 month.

**Why VAR here:**
Without VAR, you'd need to nest the CALCULATE inside DIVIDE, making the formula unreadable. VAR evaluates each value once and names it:

- `CurrentVal` = this month's transaction value (captured in filter context)
- `PrevVal` = prior month's transaction value (shifted with DATEADD)
- `RETURN` = the final output: (current − prior) / prior

**Concrete example:**

| Month    | Total Txn Value | PrevVal (prior month) | MoM %  |
| -------- | --------------- | --------------------- | ------ |
| Jan 2024 | ₹620M           | Dec 2023: ₹590M       | +5.1%  |
| Feb 2024 | ₹580M           | Jan 2024: ₹620M       | −6.5%  |
| Mar 2024 | ₹710M           | Feb 2024: ₹580M       | +22.4% |

**DATEADD vs SAMEPERIODLASTYEAR:**

- SAMEPERIODLASTYEAR = exactly 12 months back, no configuration
- DATEADD = any direction, any interval, any unit (months, quarters, days, years)
- Use SAMEPERIODLASTYEAR for YoY (simpler, more readable)
- Use DATEADD for everything else (MoM, QoQ, rolling periods)

**On a card with no slicer:** 0.04% — comparing the last two months in the dataset (Dec vs Nov 2024), which happen to be nearly equal. Not blank because DATEADD found a prior month. On a chart this shows monthly momentum.

**Result: 0.04% (full dataset, no slicer — compares last two months)**
**Format: Percentage, 2 decimals**

---

## 5. Branch Rank using `RANKX`

```dax
Branch Rank =
RANKX (
    ALL ( DimBranch[BranchName] ),
    [Total Txn Value],
    ,
    DESC
)
```

**Line-by-line:**

- `RANKX (` — iterates over a table and assigns rank positions.
- `ALL ( DimBranch[BranchName] )` — the table to rank over. `ALL` removes filters on BranchName so _every_ branch is included in the ranking comparison, even if a visual filter has narrowed the context. Without ALL, every branch would rank #1 (comparing only against itself).
- `[Total Txn Value]` — the expression to rank by: compute this for each branch.
- `,` — third argument (value for the current row) left blank — RANKX reuses the expression. Standard shorthand.
- `, DESC` — highest value = Rank 1. The branch with the most transaction value gets Rank 1.

**Meaningful on:** a table or matrix visual with BranchName on rows — each branch shows its rank number. On a card it returns 1 (no filter = all branches compete = first alphabetically gets 1). **Format: Whole Number.**

**Interview hook:** _"The key to RANKX is the ALL — without it, every filtered row compares only against itself and ranks #1. ALL forces the comparison against the full dimension so ranks are meaningful."_

---

## 6. Fraud Flagged Value

```dax
Fraud Flagged Value =
CALCULATE (
    [Total Txn Value],
    FactTransactions[IsFraudFlag] = 1
)
```

CALCULATE + flag filter — the same pattern as NPA Amount. Reuses Total Txn Value base measure and filters to fraud-flagged rows only (~0.4% of transactions by design, high-value digital channel bias).

**Result: ₹31M** — ₹31M of ₹15bn total = 0.2% by value. A small but significant number feeding the Compliance persona's page. **Format: Currency (₹), 0 decimals.**

---

## 7. Performance Analyzer — results and interpretation

**Results from today's run:**
Every Card visual rendered in **195–213 milliseconds**. All sub-200ms.

**Interpretation:**

- Sub-200ms per visual = excellent for Import mode
- 500ms+ = investigate
- 2,000ms+ = problem (likely: FILTER over whole table, high cardinality, non-star model, or too many visuals)

**What the fast results confirm:**

- Star schema is clean (no circular paths, no many-to-many)
- Integer keys compress well in VertiPaq
- Measures are lean (no FILTER(table,...) patterns, DIVIDE not slash, measures not calculated columns)
- Import mode VertiPaq engine serving from memory efficiently

**The optimization workflow:**

```
Performance Analyzer → identify slow visual
→ read its DAX query duration
→ fix the MODEL first (star schema, cardinality, hide unused columns)
→ then fix the MEASURE (VAR, boolean filters, DIVIDE, remove FILTER(table))
→ then reduce visual count per page
→ consider Aggregations / Composite for very large data
```

---

## 8. The complete measure library — Day 6 final state

| #   | Measure             | Pattern                                   | Result | Format |
| --- | ------------------- | ----------------------------------------- | ------ | ------ |
| 1   | Total Deposits      | CALCULATE(SUM, Category=Deposits)         | ₹794M  | ₹ 0dp  |
| 2   | Total Loan Book     | SUM(OutstandingAmount)                    | ₹4bn   | ₹ 0dp  |
| 3   | Total Sanctioned    | SUM(SanctionAmount)                       | ₹8bn   | ₹ 0dp  |
| 4   | NPA Amount          | CALCULATE(SUM, NPAFlag=1)                 | ₹516M  | ₹ 0dp  |
| 5   | NPA %               | DIVIDE([NPA Amount],[Loan Book])          | 11.66% | % 2dp  |
| 6   | Active Customers    | CALCULATE(DISTINCTCOUNT, Status=Active)   | 1,534  | #      |
| 7   | CASA Ratio          | VAR + IN list + DIVIDE                    | 67.28% | % 2dp  |
| 8   | Total Txn Value     | SUM(Amount)                               | ₹15bn  | ₹ 0dp  |
| 9   | Total Transactions  | COUNTROWS                                 | 60,000 | #      |
| 10  | Deposits YTD        | TOTALYTD([Total Deposits], Date)          | ₹794M  | ₹ 0dp  |
| 11  | TXN Value LY        | CALCULATE([TxnValue], SAMEPERIODLASTYEAR) | ₹8bn   | ₹ 0dp  |
| 12  | Txn YoY%            | DIVIDE(Current−LY, LY)                    | 99.68% | % 2dp  |
| 13  | Txn MoM %           | VAR + DATEADD(-1,MONTH) + DIVIDE          | 0.04%  | % 2dp  |
| 14  | Branch Rank         | RANKX(ALL(Branch), [TxnValue],, DESC)     | 1      | #      |
| 15  | Fraud Flagged Value | CALCULATE([TxnValue], FraudFlag=1)        | ₹31M   | ₹ 0dp  |

---

# INTERVIEW QUESTIONS — Day 6 (Advanced DAX)

> Added: 2026-06-05
> Format: keyword trigger → spoken answer → senior signal

---

### Q: What is time intelligence and what does it require?

**Keywords:** date filter shift · marked date table · continuous calendar

**Spoken answer:**
"Time intelligence is a set of DAX functions that shift the active date filter
to a different period — last year, year to date, prior month — and re-evaluate
a base measure over that shifted period. They all require a marked date table
with a continuous, unbroken calendar. Without it, the functions can't find year
boundaries or navigate correctly. On BankSphere 360 I marked DimDate on its
Date column (not the integer DateKey) after verifying 731 consecutive days with
no gaps."

**Senior signal:** distinguishing the Date column from DateKey, and mentioning
the continuous-calendar requirement.

---

### Q: TOTALYTD — how does it work and when would you use a custom year-end?

**Keywords:** accumulates Jan 1 to current · resets annually · fiscal year end parameter

**Spoken answer:**
"TOTALYTD evaluates a base measure over a date range from January 1st of the
current year to the last date in context. So in a monthly chart, each month
shows the running total from the start of the year. For an Indian bank with a
financial year ending March 31, I'd pass a third argument:
TOTALYTD([Deposits], DimDate[Date], '03-31'). On a card with no slicer it
returns the full total — its value appears in time-series visuals where the
accumulation is visible."

---

### Q: SAMEPERIODLASTYEAR — is it a scalar or a table function?

**Keywords:** table function · returns a set of dates · sits inside CALCULATE

**Spoken answer:**
"It's a table function — it returns a _set of dates_ representing the same
period twelve months earlier, not a single value. That's why it must sit inside
CALCULATE as a filter argument. CALCULATE applies that date set as the new
filter context, and the base measure recalculates over the prior-year window.
A common mistake is trying to use it outside CALCULATE — it won't work because
you need CALCULATE to apply the date set as a filter."

**Senior signal:** the table-function distinction is an advanced DAX concept
most candidates don't know.

---

### Q: SAMEPERIODLASTYEAR vs DATEADD — when do you use which?

**Keywords:** SAMEPERIODLASTYEAR = fixed 12 months · DATEADD = any interval/unit

**Spoken answer:**
"SAMEPERIODLASTYEAR is the simpler choice for year-over-year — it always shifts
exactly 12 months back, no configuration needed, readable. DATEADD is more
flexible: I specify the interval (any positive or negative integer) and the
unit (DAY, MONTH, QUARTER, or YEAR). So for MoM I use DATEADD(-1, MONTH),
for QoQ DATEADD(-1, QUARTER), for rolling 90 days DATEADD(-90, DAY). I use
SAMEPERIODLASTYEAR for YoY and DATEADD for everything else."

---

### Q: Walk through Txn MoM% — why VAR, and what does DATEADD(-1, MONTH) do?

**Keywords:** VAR evaluated once · DATEADD shifts -1 month · RETURN gives output

**Spoken answer:**
"DATEADD(-1, MONTH) shifts the current date filter back by one month — so if
the visual shows March 2024, it returns February 2024's dates. CALCULATE
evaluates Total Txn Value over those February dates.

I wrapped everything in VAR for two reasons: readability and performance.
Without VAR, the formula is a nested mess. With VAR, I name the two values —
CurrentVal for this month and PrevVal for the prior month — then RETURN the
DIVIDE. VAR also evaluates each variable once, so PrevVal is computed a single
time rather than potentially twice inside the DIVIDE. The result is (current −
prior) / prior = the percentage change month over month."

**Senior signal:** mentioning the single-evaluation performance benefit of VAR.

---

### Q: How does RANKX work, and what happens without ALL?

**Keywords:** iterates over table · ALL removes filter · highest = Rank 1 · DESC

**Spoken answer:**
"RANKX iterates over every row in a table, evaluates an expression for each,
and assigns rank positions. The critical part is the first argument — the table
to rank over. I use ALL(DimBranch[BranchName]) so every branch is included in
the comparison regardless of what the visual has filtered. Without ALL, each
branch would only see itself in the ranking table and every branch would get
Rank 1 — completely useless. ALL forces the comparison against the full
dimension. DESC means the highest transaction value gets Rank 1."

---

### Q: How do you diagnose a slow Power BI report?

**Keywords:** Performance Analyzer · DAX query duration · fix model first

**Spoken answer:**
"I start with View → Performance Analyzer, record, interact with the report,
and read the DAX query duration per visual. Anything over 500ms per visual
gets investigated. Common causes: a FILTER over a whole fact table instead of
a boolean column filter; high-cardinality columns that should be hidden or
removed; too many visuals on one page; broken query folding in Power Query;
or a non-star model with complex join paths. I fix the model first — star
schema, cardinality — then the measure logic (replacing FILTER with simpler
filters, using VAR, DIVIDE), then reduce visual count. On BankSphere 360 my
cards all rendered under 200ms, confirming the star schema and integer keys
are compressing well in VertiPaq."

**Senior signal:** the "fix model first, then measure, then visual count"
priority order shows systematic thinking.

---

### Q: Why use COUNTROWS vs COUNT vs DISTINCTCOUNT?

**Keywords:** COUNTROWS = table rows · COUNT = non-blank column · DISTINCTCOUNT = unique values

**Spoken answer:**
"COUNTROWS counts the rows in a table — it's efficient because it operates at
the table grain. COUNT counts non-blank values in a specific column — slightly
different if there are blanks. DISTINCTCOUNT counts unique values — essential
when one entity (like a customer) appears on multiple rows (multiple accounts)
and you want to count the entity once. I used COUNTROWS for Total Transactions
(each row is one transaction) and DISTINCTCOUNT for Active Customers
(CustomerID repeats across accounts)."

---

### Q: Can you explain circular dependency in the context of Sort by Column?

**Keywords:** DAX column references source · sort creates dependency · fix in Power Query

**Spoken answer:**
"A circular dependency occurs when column A's sort order depends on column B,
and column B's DAX expression references column A. The engine traces: A needs B
to sort → B needs A to evaluate → back to A. It blocks the relationship. I hit
this trying to sort DPD Bucket (text) by a DAX calculated column that used
SWITCH on DPD Bucket. The fix was to move the sort-key column to Power Query,
where it's evaluated at load time independently of the DAX layer — no
dependency, no circular reference."

---

### Q: What is the difference between SAMEPERIODLASTYEAR and DATEADD(-1, YEAR)?

**Spoken answer:**
"They're similar but not identical. SAMEPERIODLASTYEAR returns the same set
of dates from the prior year — for Jan–Mar 2024, it returns Jan–Mar 2023
as a contiguous block. DATEADD(-1, YEAR) shifts each individual date in the
current filter back by one year and may behave slightly differently with
partial periods and leap years. For standard YoY calculations I use
SAMEPERIODLASTYEAR because it's more readable and behaves predictably with
month and quarter selections. DATEADD gives more control for custom scenarios."

---

### Q: What are the prerequisites for time-intelligence functions?

**Spoken answer:**
"Three things: first, a date table marked via Table tools → Mark as date table.
Second, the date column used in the function must be the marked column (the true
Date type, not an integer DateKey). Third, the date table must have continuous,
unbroken dates with no gaps — time functions navigate the calendar by walking
it; a gap means they can't find the correct boundary. If any of these is
missing, time-intelligence functions either error out or return wrong values."

**Senior signal:** listing all three prerequisites (marked, correct column, no
gaps) rather than just "you need a date table."
