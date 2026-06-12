# Day 5 — DAX Foundations: Core Measures

> Project: **BankSphere 360** — Power BI Banking Analytics
> Stage in lifecycle: requirements → source analysis → ETL → data model → **measures (you are here)** → visuals → security → deploy → support
> Goal of the day: turn the clean star schema into actual KPIs using DAX — and understand *filter context*, the concept everything else rests on.

---

## 1. The core concept — filter context

A **measure** does not compute over the whole table. It computes over only the rows that survive the **filter context** — the combination of:
- slicer selections,
- the rows/columns of the visual it sits in,
- and any filters the measure itself adds (via `CALCULATE`).

Mental model: *filters narrow the rows → the measure aggregates only the survivors.* Picking "Region = West" on a slicer and "Category = Deposits" inside the measure means the SUM only sees West-region deposit accounts.

---

## 2. Measure vs Calculated Column

| | Calculated Column | Measure |
|---|---|---|
| Computed | once, at refresh | on the fly, at query time |
| Stored | yes, per row | no |
| Context | **row context** (knows "this row") | **filter context** (knows "these filters") |
| Use for | slicing, relationships, row-level logic | KPIs, aggregations that respond to filters |

Rule of thumb: if you want to *slice by it* or *join on it* → column. If you want a *number that reacts to filters* → measure. Almost every KPI is a measure.

---

## 3. The `_Measures` table

Created an empty table via **Home → Enter data**, named **`_Measures`** (underscore prefix sorts it to the top of the Fields pane). All measures live here in one findable place rather than scattered across fact tables. The leftover empty `Column1` can be hidden/deleted — the table becomes a pure measure container.

---

## 4. Formatting at the measure level

Formatting is set on the **measure** (Measure tools → Format), not visual-by-visual. This means the measure displays consistently — same currency symbol, decimals, percentage — everywhere it appears: cards, tables, tooltips. Conventions used:
- Money measures → **Currency (₹), 0 decimals** (e.g. ₹794M)
- Ratios → **Percentage, 2 decimals** (e.g. 11.66%)
- Counts → **Whole Number, thousands separator** (e.g. 1,534)

Naming convention chosen: **spaces, human-readable** (`Total Deposits`, not `total_dep`).

---

## 5. The seven core measures built

### 1. Total Deposits — `CALCULATE` + filter
```dax
Total Deposits =
CALCULATE (
    SUM ( FactAccounts[Balance] ),
    DimProduct[ProductCategory] = "Deposits"
)
```
Sum of balances, restricted to deposit products. The filter references DimProduct but applies to FactAccounts via the Day-4 relationship. **Result: ₹794M.**
Validation: summing *all* balances (no filter) gave **−₹1.70bn** — negative because loans/cards are stored negative. This proved the CALCULATE filter correctly isolates deposits.

### 2. Total Loan Book — plain `SUM`
```dax
Total Loan Book = SUM ( FactLoans[OutstandingAmount] )
```
No CALCULATE needed — FactLoans contains only loans, nothing to filter out. **Result: ₹4bn.**
Key distinction: **OutstandingAmount = what's still owed today** (after EMI repayments), not the original sanctioned amount. The loan book is a snapshot of current exposure, and the correct denominator for NPA%.

### 3. Total Sanctioned — plain `SUM`
```dax
Total Sanctioned = SUM ( FactLoans[SanctionAmount] )
```
**Result: ₹8bn.** Sanctioned (₹8bn) > Loan Book (₹4bn) confirms customers have repaid ~half — a logical-integrity check that passed.

### 4. NPA Amount — `CALCULATE` + flag filter
```dax
NPA Amount =
CALCULATE (
    SUM ( FactLoans[OutstandingAmount] ),
    FactLoans[NPAFlag] = 1
)
```
Outstanding sitting in non-performing (90+ DPD) loans. **Result: ₹516M.**

### 5. NPA % — `DIVIDE` of measures
```dax
NPA % = DIVIDE ( [NPA Amount], [Total Loan Book] )
```
**Result: 11.66%.** Uses `DIVIDE` (safe division — returns blank, not error, on zero denominator) and **reuses existing measures** rather than rewriting their logic (DRY principle). Intentionally high (~12%) in this teaching dataset; real banks run 2–4%.

### 6. Active Customers — `DISTINCTCOUNT`
```dax
Active Customers =
CALCULATE (
    DISTINCTCOUNT ( FactAccounts[CustomerID] ),
    FactAccounts[Status] = "Active"
)
```
**Result: 1,534.** `DISTINCTCOUNT` counts each customer once even if they hold multiple accounts — a plain COUNT would inflate the number by counting accounts.

### 7. CASA Ratio — `VAR` + `IN` + `DIVIDE`
```dax
CASA Ratio =
VAR CASA =
    CALCULATE (
        [Total Deposits],
        DimProduct[ProductName] IN { "Savings Account", "Current Account" }
    )
RETURN
    DIVIDE ( CASA, [Total Deposits] )
```
**Result: 67.28%.** Share of deposits held in cheap current+savings accounts (vs expensive fixed deposits). `VAR` computes the CASA numerator once and names it; `RETURN` gives the final output; `IN { }` tests list membership. A healthy CASA (banks target 40%+) means strong low-cost funding.

---

## 6. The business story the numbers tell

Cheap funding (CASA 67%) but a risky loan book (NPA 11.66%) — strong deposit profile, distressed lending. A good dashboard surfaces exactly this kind of tension.

---

## 7. DAX vocabulary covered today

`SUM` · `CALCULATE` (filtered aggregation) · `DISTINCTCOUNT` · `DIVIDE` (safe division) · `VAR … RETURN` · `IN { }` list operator · measure references `[Measure]` vs column references `Table[Column]`.

---

# INTERVIEW QUESTIONS — Day 5 (DAX Foundations)
> Added: 2026-06-04
> Format: keyword trigger → spoken answer → senior signal

---

### Q: Explain filter context vs row context.
**Keywords:** filter context = slicers/visual/CALCULATE · row context = current row · measures vs columns

**Spoken answer:**
"Row context is 'the current row' — it exists in calculated columns and inside
iterators like SUMX, where the engine walks row by row. Filter context is the
set of filters in effect — slicers, the rows and columns of a visual, and
anything CALCULATE adds. A measure evaluates inside filter context: the filters
narrow the rows, then the measure aggregates the survivors. So the same Total
Deposits measure returns different numbers in a card with no slicer versus a
card filtered to one region — same formula, different context."

**Senior signal:** the "same formula, different context" framing shows you
understand context is dynamic, not baked into the measure.

---

### Q: Measure vs calculated column — when each?
**Keywords:** measure = query-time, no storage, KPIs · column = refresh-time, stored, slicing

**Spoken answer:**
"A calculated column computes once at refresh and is stored per row — it has row
context and I use it for slicing, relationships, or row-level logic, like the
DPD bucket I built. A measure computes on the fly at query time, stores nothing,
and responds to filter context — I use it for KPIs. Rule of thumb: if I want to
slice by it or join on it, it's a column; if I want a number that reacts to
filters, it's a measure. Over-using calculated columns bloats the model, so I
default to measures for anything aggregated."

**Senior signal:** the model-bloat point — calculated columns cost memory,
measures don't.

---

### Q: What does CALCULATE do, and when do you NOT use it?
**Keywords:** only function that changes filter context · don't use when no modification needed

**Spoken answer:**
"CALCULATE evaluates an expression inside a filter context I modify with its
filter arguments — it's the only function that can change filter context.
I used it for Total Deposits to restrict the sum to deposit products. But I
deliberately did NOT use it for Total Loan Book — that's just SUM, because
FactLoans already contains only loans, so there's nothing to filter. Reaching
for CALCULATE out of habit is a smell; I use the simplest function that does
the job."

**Senior signal:** knowing when *not* to use CALCULATE is rarer than knowing how
to use it.

---

### Q: Why DIVIDE instead of the / operator?
**Keywords:** safe division · returns blank not error · zero denominator

**Spoken answer:**
"DIVIDE handles divide-by-zero gracefully — it returns a blank instead of
throwing an error, whereas the slash operator errors out. On a dashboard that
filters down to, say, a branch with no loans, the denominator can become zero,
and DIVIDE keeps the visual clean instead of showing an error. I use it for
every ratio — NPA%, CASA, anything with a denominator that could be filtered to
nothing."

---

### Q: SUM vs DISTINCTCOUNT vs COUNT?
**Keywords:** SUM = adds values · COUNT = counts rows · DISTINCTCOUNT = counts unique

**Spoken answer:**
"SUM adds up a numeric column. COUNT counts rows. DISTINCTCOUNT counts unique
values. For Active Customers I used DISTINCTCOUNT on CustomerID, because one
customer can hold several accounts — a plain count would count the accounts and
overstate the customer base. DISTINCTCOUNT collapses each customer to one."

**Senior signal:** the concrete "one customer, many accounts" reasoning proves
you've hit this in practice.

---

### Q: What is VAR and why use it?
**Keywords:** evaluated once · readability · performance · VAR…RETURN structure

**Spoken answer:**
"VAR declares a variable that's computed once, stored, and reused by name. It
makes formulas readable — I can name an intermediate result like 'CASA' instead
of nesting everything — and it can improve performance because the value is
evaluated a single time rather than repeatedly. Every measure using VAR needs a
RETURN that specifies the final output. I used it in CASA Ratio to compute the
current-plus-savings balance once, then divide."

**Senior signal:** mentioning that VAR is evaluated once (not re-evaluated per
reference) is the performance nuance interviewers look for.

---

### Q: How do you reference another measure vs a column in DAX?
**Keywords:** [Measure] brackets only · Table[Column] table-qualified

**Spoken answer:**
"A measure reference is just the measure name in square brackets — [NPA Amount].
A column reference is table-qualified — FactLoans[OutstandingAmount]. The
convention matters: always qualify columns with their table, never qualify
measures. It also makes intent readable — anyone scanning the code can tell a
measure reference from a column reference at a glance."

---

### Q: Why build NPA% from existing measures instead of rewriting the logic?
**Keywords:** DRY · single source of truth · maintainability · composition

**Spoken answer:**
"Composition. NPA% references the NPA Amount and Total Loan Book measures rather
than duplicating their SUM/CALCULATE logic. If I later change how NPA Amount is
defined, NPA% updates automatically — one source of truth. If I'd copy-pasted
the logic, I'd have to fix it in multiple places and risk them drifting apart.
Building measures compositionally keeps the model maintainable."

**Senior signal:** the word "composition" and the maintenance argument signal
you think about the model as a system, not a pile of formulas.

---

### Q: Where should you set number formatting?
**Keywords:** measure level not visual level · consistency everywhere

**Spoken answer:**
"On the measure, not the visual. Setting Currency or Percentage on the measure
itself means it displays consistently everywhere it appears — every card, table
and tooltip — instead of me reformatting it visual by visual and risking
inconsistency. Formatting is a property of what the number *means*, so it
belongs with the measure."

---

### Q: Outstanding vs sanctioned amount — what's the loan book?
**Keywords:** outstanding = still owed · sanctioned = originally lent · book = current exposure

**Spoken answer:**
"The loan book is the current outstanding principal — what customers still owe
today after their EMI repayments — not the cumulative sanctioned amount that was
originally disbursed. I use outstanding because the loan book is a snapshot of
present exposure, and it's the correct denominator for asset-quality ratios like
NPA%. In my data, sanctioned was ₹8bn but outstanding ₹4bn, because customers
had repaid about half."

**Senior signal:** this domain distinction separates candidates who understand
banking from those who just know Power BI.

---

## Quick-reference: the seven measures

| Measure | Pattern | Result | Format |
|---|---|---|---|
| Total Deposits | CALCULATE(SUM, Category=Deposits) | ₹794M | Currency 0dp |
| Total Loan Book | SUM(OutstandingAmount) | ₹4bn | Currency 0dp |
| Total Sanctioned | SUM(SanctionAmount) | ₹8bn | Currency 0dp |
| NPA Amount | CALCULATE(SUM, NPAFlag=1) | ₹516M | Currency 0dp |
| NPA % | DIVIDE([NPA Amount],[Loan Book]) | 11.66% | Percentage 2dp |
| Active Customers | CALCULATE(DISTINCTCOUNT(CustomerID), Status=Active) | 1,534 | Whole number |
| CASA Ratio | VAR + IN list + DIVIDE | 67.28% | Percentage 2dp |
