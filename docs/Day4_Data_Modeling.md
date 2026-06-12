# Day 4 — Data Modeling: Building the Star Schema

> Project: **BankSphere 360** — Power BI Banking Analytics
> Stage in lifecycle: requirements → source analysis → ETL → **data model (you are here)** → measures → visuals → security → deploy → support
> Goal of the day: turn 7 cleansed tables into a clean, unambiguous **star schema**, and mark a proper date table so time-intelligence works.

---

## 1. The concept — what a star schema is

A **star schema** has descriptive **dimension** tables surrounding central **fact** tables:

- **Dimensions** (the "who / what / when / where"): DimDate, DimCustomer, DimProduct, DimBranch — small, descriptive, used for slicing.
- **Facts** (the "events / measurements"): FactAccounts, FactTransactions, FactLoans — large, numeric, the things you aggregate.

Relationships are **one-to-many** (`1` → `*`): one dimension row relates to many fact rows. Filters flow **from the one-side to the many-side** by default. Picking a year in DimDate filters the facts; picking a transaction does not filter the date.

**Why star, not snowflake:** star keeps dimensions denormalized (flat), which means fewer joins, better VertiPaq compression, simpler DAX, and cleaner relationships. Snowflake normalizes dimensions into sub-tables — more joins, more complexity, rarely worth it in Power BI.

---

## 2. The first move — distrust auto-detect

When data loads, Power BI **auto-detects** relationships by matching column names. This is a convenience that frequently makes a mess, because it links any columns that *happen* to share a name, regardless of whether the link is meaningful.

**First action taken:** turned OFF *File → Options → Current File → Data Load → "Autodetect new relationships after data is loaded"* so it stops interfering, and committed to building/auditing relationships manually.

---

## 3. The audit — every auto-detected relationship reviewed

Opened **Home → Manage relationships** to see the exact list. Classified each:

| # | Relationship | Verdict | Reason |
|---|---|---|---|
| 1 | DimCustomer(City) → DimBranch(City) | ❌ DELETE | Coincidental name match. A customer's city has no business link to a branch's city. False filter path. |
| 2 | FactAccounts(BranchID) → DimBranch | ✅ KEEP | Proper star: one branch, many accounts. |
| 3 | FactAccounts(CustomerID) → DimCustomer | ⚠️ was Inactive → became Active | Correct link; activated automatically once the bad link was removed. |
| 4 | FactAccounts(ProductID) → DimProduct | ✅ KEEP | One product, many accounts. |
| 5 | FactLoans(AccountID) → FactAccounts | ❌ DELETE | 1:1 **fact-to-fact** link with bidirectional arrow — creates ambiguity. Facts should join through dimensions, not each other. |
| 6 | FactLoans(CustomerID) → DimCustomer | ✅ KEEP | One customer, many loans. |
| 7 | FactLoans(ProductID) → DimProduct | ✅ KEEP | One product, many loans. |
| 8 | FactTransactions(AccountID) → FactAccounts | ⚠️ KEEP (intentional) | The one defensible fact-to-fact hop: transactions only carry AccountID, so they must reach customer/product/branch *through* accounts. |
| 9 | FactTransactions(DateKey) → DimDate | ✅ KEEP | The only existing DimDate link — and the active primary time path. |

**Result of deletes:** removing #1 and #5 cleaned the model of false/ambiguous links, and deleting #5 freed the path so #3 auto-activated.

---

## 4. The missing pieces — two DimDate relationships

After cleanup, DimDate connected only to FactTransactions. Accounts couldn't be analyzed by open date, loans couldn't be analyzed by sanction date. Two relationships needed adding:

- DimDate(DateKey) → FactAccounts(OpenDateKey)
- DimDate(DateKey) → FactLoans(SanctionDateKey)

### The ambiguous-path problem (the key lesson)

Trying to add **DimDate → FactAccounts active** triggered a red error:

> *"There are ambiguous paths between 'FactTransactions' and 'DimDate'."*

**Why:** FactTransactions could now reach DimDate two ways — directly (FactTransactions → DimDate) **and** indirectly (FactTransactions → FactAccounts → DimDate). Power BI can't decide whether a transaction should be filtered by its own date or by the open-date of its account. So it blocks the relationship to protect against silently-wrong numbers.

This is the intentional FactTransactions → FactAccounts hop *revealing its cost*: fact-to-fact links create ambiguity the moment a shared dimension is involved.

### The fix — create them INACTIVE

Both secondary date relationships were created as **Inactive** (the "Make this relationship active" box unticked):

- They exist in the model (shown as **dashed lines**) but don't auto-filter, so there's no ambiguity.
- When a specific analysis needs them (e.g. "new accounts opened per month", "loans sanctioned per month"), they're activated **on-demand inside a measure** using `USERELATIONSHIP`.

**Design outcome:** one unambiguous default date path (transactions), two secondary date paths parked as inactive for on-demand use.

---

## 5. Final model shape

**Active relationships (default filter paths):**
- FactAccounts → DimBranch, DimCustomer, DimProduct
- FactLoans → DimCustomer, DimProduct
- FactTransactions → FactAccounts (intentional hop), → DimDate

**Inactive relationships (USERELATIONSHIP on demand):**
- FactAccounts(OpenDateKey) → DimDate
- FactLoans(SanctionDateKey) → DimDate

All active relationships are `*` → `1`, single cross-filter direction.

---

## 6. Mark as Date Table — the mandatory final step

Time-intelligence functions (`TOTALYTD`, `SAMEPERIODLASTYEAR`, `DATEADD`) require a formally-declared date table.

**Steps taken:** selected DimDate → **Table tools → Mark as date table** → chose the **`Date`** column (the true date type, NOT `DateKey`) → validated with no error → Save.

**Why `Date` and not `DateKey`:** `DateKey` (20230102) is an integer — great for *relationships*, but not a real date. The `Date` column is an actual date type, which is what time-intelligence navigates. You join on the integer key, but you mark the real date.

**Why it validated cleanly:** the column had unique, contiguous values with no blanks — the 731 continuous days verified during Day 3 cleansing. Marking it also strips the auto-generated date hierarchies, since a proper date table now exists.

---

# INTERVIEW QUESTIONS — Day 4 (Data Modeling)
> Added: 2026-06-04
> Format: keyword trigger → spoken answer → senior signal

---

### Q: What is a star schema and why is it preferred in Power BI?
**Keywords:** dimensions around facts · one-to-many · denormalized · fewer joins · VertiPaq

**Spoken answer:**
"A star schema has descriptive dimension tables surrounding central fact
tables, joined one-to-many. The facts hold the events I aggregate —
transactions, accounts, loans — and the dimensions hold the attributes I slice
by, like date, customer, product, branch. It's preferred over snowflake because
the dimensions stay denormalized: fewer joins, better VertiPaq compression,
simpler DAX, and cleaner relationships. On BankSphere 360 I had four conformed
dimensions feeding three fact tables."

**Senior signal:** the word "conformed" (shared dimensions used by multiple
facts) and naming VertiPaq compression.

---

### Q: Do you trust Power BI's auto-detected relationships?
**Keywords:** match on name · coincidental links · always audit · often rebuild

**Spoken answer:**
"No. Auto-detect matches on column names, so it links any columns that happen
to share a name whether or not the link is meaningful. On this project it
created a DimCustomer-to-DimBranch link purely because both had a City column —
which is nonsense. I always audit every auto-detected relationship in Manage
Relationships, delete the false ones, and build the rest deliberately so I
understand and control each filter path. I also turn off auto-detect so it
stops interfering."

**Senior signal:** the concrete "City-to-City coincidence" example proves you've
actually done this, not just read about it.

---

### Q: How do filters flow in a relationship?
**Keywords:** one-side to many-side · single cross-filter · default direction

**Spoken answer:**
"By default, filters flow from the one-side to the many-side. So a dimension
filters a fact — picking a year in the date table filters the transactions —
but not the reverse. That's single cross-filter direction, which is the safe
default. Bidirectional filtering exists but I use it sparingly because it can
create ambiguity and performance issues."

---

### Q: Why avoid fact-to-fact relationships?
**Keywords:** ambiguity · join through dimensions · shared dimension creates conflict

**Spoken answer:**
"Facts should join through shared dimensions, not directly to each other,
because a direct fact-to-fact link creates ambiguous filter paths once a common
dimension is involved. On BankSphere 360, transactions connect to accounts as a
deliberate hop — transactions only carry an AccountID, so they reach customer,
product and branch through the account. But the moment I tried to connect the
date table to accounts as well, Power BI flagged an ambiguous path, because
transactions could now reach the date two ways. That's exactly the cost of
fact-to-fact links."

**Senior signal:** explaining the *cost* of the one fact-to-fact link you kept,
rather than pretending you avoided them entirely.

---

### Q: What is an ambiguous path and how did you resolve one?
**Keywords:** two routes to a table · Power BI blocks it · inactive + USERELATIONSHIP

**Spoken answer:**
"An ambiguous path is when a table can reach another table through more than one
route, so the engine can't decide which filter to apply. I hit this connecting
the date table to accounts while transactions already linked both to the date
and to accounts. My fix was to create the secondary date relationships as
*inactive* — they exist as dashed lines but don't auto-filter, so there's no
ambiguity — and then activate them on demand inside specific measures using
USERELATIONSHIP. One unambiguous default path, with secondary paths available
when I explicitly need them."

**Senior signal:** the inactive-relationship + USERELATIONSHIP pattern is an
intermediate-to-advanced modeling technique.

---

### Q: What's the difference between an active and an inactive relationship?
**Keywords:** active = auto-filters (solid) · inactive = dormant (dashed) · one active path

**Spoken answer:**
"Only one relationship between two tables can be active at a time — it's the
solid line and it filters automatically. Any others are inactive, shown dashed,
and they don't filter unless I explicitly invoke them in a measure with
USERELATIONSHIP. I used this for role-playing dates: transaction date is the
active default, while account open-date and loan sanction-date are inactive and
activated only in measures that specifically analyze those events."

**Senior signal:** the term "role-playing dimension" (one date table playing
multiple date roles).

---

### Q: Why do you need a dedicated Date table, and why mark it?
**Keywords:** continuous dates · time intelligence requires it · mark on real date column

**Spoken answer:**
"Time-intelligence functions like TOTALYTD and SAMEPERIODLASTYEAR need a table
flagged as the date table with a continuous, unbroken run of dates. Marking it
tells the engine 'this is a navigable calendar.' I marked DimDate on its true
Date column — not the integer DateKey — because time-intelligence navigates a
real date type. I join facts on the integer key for performance, but mark the
actual date. My table had 731 contiguous days with no gaps, so it validated
cleanly."

**Senior signal:** distinguishing the integer key (for joins) from the date
column (for marking) shows real understanding, not rote clicking.

---

### Q: Single vs bidirectional cross-filter direction?
**Spoken answer:**
"Single is the default and what I use almost always — the dimension filters the
fact, one direction. Bidirectional makes filters flow both ways, which is
occasionally needed for many-to-many scenarios through a bridge table, but it
risks ambiguity and slows performance, so I avoid it unless there's a specific
reason."

---

### Q: What is VertiPaq and why does the model design affect it?
**Keywords:** in-memory columnar engine · dictionary + RLE compression · low cardinality

**Spoken answer:**
"VertiPaq is Power BI's in-memory columnar engine. It compresses each column
using dictionary encoding and run-length encoding, so low-cardinality columns
compress far better than high-cardinality ones. A star schema helps because
denormalized dimensions and clean integer keys give the engine compact,
low-cardinality columns to work with — which is why integer keys beat text keys
and why I don't keep unnecessary high-cardinality columns in the model."

---

## Quick-reference: the model at a glance

| Table | Type | Active links to |
|---|---|---|
| DimDate | Dimension (marked date table) | FactTransactions (active); FactAccounts & FactLoans (inactive) |
| DimCustomer | Dimension | FactAccounts, FactLoans |
| DimProduct | Dimension | FactAccounts, FactLoans |
| DimBranch | Dimension | FactAccounts |
| FactAccounts | Fact | hub for FactTransactions |
| FactTransactions | Fact | → FactAccounts (intentional hop), → DimDate |
| FactLoans | Fact | → DimCustomer, DimProduct |

**Inactive (USERELATIONSHIP on demand):** FactAccounts(OpenDateKey)→DimDate, FactLoans(SanctionDateKey)→DimDate.
