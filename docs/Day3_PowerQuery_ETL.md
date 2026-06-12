# Day 3 — Power Query: ETL, Cleansing & Data Quality

> Project: **BankSphere 360** — Power BI Banking Analytics
> Stage in lifecycle: requirements → source analysis → **ETL (you are here)** → data model → measures → visuals → security → deploy → support
> Goal of the day: take 7 raw CSV sources and turn them into a clean, correctly-typed, governed layer ready for modeling.

---

## 1. What we set out to do

Every table needed, at minimum, the **three-step spine**:

1. **Promote Headers** — lift the real column names out of row 1.
2. **Set Data Types** — explicitly type every column (keys as whole number, money as decimal, dates as date, categories as text).
3. **Table-specific cleansing** — only where the column profiler showed it was needed.

The guiding principle: **let the data drive the cleansing, not the roadmap.** We profiled every column and only added steps the data actually justified.

---

## 2. Pre-work — profiler setup

Before transforming, we turned the Power Query profiler into a real QA tool:

- **View tab** → enabled **Column quality**, **Column distribution**, **Column profile**.
- **Bottom status bar** → switched **"Column profiling based on top 1000 rows"** to **"based on entire data set"**.

**Why it matters:** by default the profiler only inspects the first 1,000 rows. On a 60,000-row table that means a bad value at row 45,000 would never be seen. Profiling on the entire dataset makes every quality check honest, not sampled.

> ⚠️ Note: this setting can silently revert when you reopen the editor. Re-confirm it before trusting quality reads on large tables.

---

## 3. Encoding — the first data-quality control

On every CSV import, the **File Origin** was set to **65001: Unicode (UTF-8)**, never the legacy default **1252 (Western European)**.

| Encoding | What it is | Limit |
|---|---|---|
| **1252** | Legacy single-byte Windows ("ANSI") | 256 characters, Western only |
| **65001 (UTF-8)** | Modern variable-width universal standard | every language, every character |

**The bug avoided:** a character like `é` is stored as two bytes in UTF-8. Read with 1252 (one byte = one char), it shows as garbled `Ã©` ("mojibake"). This only surfaces on non-English characters — which is why `DimDate` looked fine under 1252 but `DimCustomer` (accented names) would have corrupted. In an Indian bank, customer/branch names routinely contain regional characters, so encoding is the first data-quality control in the pipeline.

---

## 4. What we did to each table

### DimBranch (10 rows) — the simplest
- Promote Headers → `BranchID, BranchName, City, Region, State`
- `BranchID` → **Whole Number** (it's the join key)
- The four descriptive columns → **Text**
- Result: Count 10, 0 errors, 0 empties. The phantom header row dropped the count from 11 → 10.

### DimProduct (10 rows)
- Same spine. `ProductID` → Whole Number; `ProductName, ProductCategory, Segment` → Text.
- `ProductCategory` has 4 distinct (Deposits, Loans, Cards, Wealth); `Segment` has 3 (Retail, SME, HNI).
- This small table is the backbone of half the KPIs — Total Deposits and Total Loan Book filter on `ProductCategory`.

### DimCustomer (1,000 rows) — most-transformed dimension
- Promote Headers.
- `CustomerID` → Whole Number; `DOB` and `OnboardDate` → **Date** (ISO source parsed cleanly, 0% error); rest → Text.
- **Removed Duplicates** keyed on `CustomerID` — defensive guard. Profiler confirmed Distinct = Unique = 1000 (no dupes), but the step protects every future refresh.
- **City cleanup**: Trim → Clean → Capitalize Each Word (`Text.Trim`, `Text.Clean`, `Text.Proper`). City stayed 10 distinct — proof the cleanup didn't fragment or merge anything.

### DimDate (731 rows)
- Spine only. `DateKey, Day, Month, Quarter, Year, IsWeekend` → Whole Number; `Date` → Date; `MonthName, WeekdayName` → Text.
- 731 = 2 years (730 days) + 1 leap day (2024). Continuous, no gaps — essential because DimDate becomes the marked Date table for time intelligence.

### FactAccounts (3,500 rows)
- Was auto-promoted/typed on load; verified rather than rebuilt.
- Keys → Whole Number; `Balance` and `InterestRate` → **Decimal** (rupee values, with negatives for loans/cards); `Status` → Text (3 distinct).
- **Did NOT add a "replace null InterestRate" step** — the profiler showed 0% empty, so the contingency step from the roadmap was unnecessary. *(Key lesson: don't add steps the data doesn't justify.)*
- **Creative addition:** `AbsBalance` custom column = `Number.Abs([Balance])`, typed Decimal — gives balance magnitude regardless of sign.

### FactLoans (1,000 rows) — the flagship
- Keys → Whole Number; `SanctionAmount, OutstandingAmount, EMIAmount` → Decimal; `DPD, NPAFlag` → Whole Number; `Status` → Text.
- Verified the banking rule holds in data: DPD ≥ 90 ⟶ NPAFlag = 1 ⟶ Status = "NPA".
- **Creative centerpiece — `DPD Bucket` conditional column** using RBI-style asset-classification language:

| DPD | Label |
|---|---|
| 0 | Current (0 DPD) |
| 1–30 | Watch (1-30 DPD) |
| 31–60 | Substandard (31-60 DPD) |
| 61–89 | Doubtful (61-90 DPD) |
| 90+ | Loss / NPA (90+ DPD) |

  Built via Add Column → Conditional Column (cascading `if/else if`), typed Text, 5 distinct values, 0 errors.
  *(Pending: a numeric sort-order helper so buckets display in risk order, not alphabetically — to be added in the modeling stage.)*

### FactTransactions (60,000 rows) — the big one
- Keys → Whole Number; `Channel, TxnType` → Text; `Amount` → Decimal; `IsFraudFlag` → Whole Number.
- **Reconciliation (UAT TC-01):** Count = 60,000 = source `wc -l` count. Pipeline integrity confirmed.
- `Channel` 6 distinct, `TxnType` 7 distinct — as designed.
- `IsFraudFlag` heavily imbalanced (~0.4% fraud) — expected rare-event distribution, not an error.

---

## 5. Data quality outcome

Across all 7 tables: **Error 0%, Empty 0%.** Keys verified unique. Row counts reconciled to source. Types explicit on every column. Two creative derived columns (AbsBalance, DPD Bucket) added with purpose.

---

## 6. Closing the stage

- **Close & Apply** runs every transformation step and loads shaped data into the model.
- Steps are a *saved recipe* — they re-run on every future refresh, so the pipeline stays clean automatically.

---

# INTERVIEW QUESTIONS — Day 3 (Power Query / ETL / Data Quality)
> Added: 2026-06-04
> Format: keyword trigger → spoken answer → senior signal

---

### Q: What is Power Query and where does it sit in the Power BI workflow?
**Keywords:** ETL layer · M language · applied steps · before the model

**Spoken answer:**
"Power Query is the ETL layer in Power BI. It's where I connect to sources
and clean, shape and type the data before it enters the model. Every action
becomes a named step in a saved recipe, written in the M language under the
hood, and that recipe re-runs on every refresh. The rule I follow is: shape
the data in Power Query, not in the visuals — fix it once at the source layer
rather than patching logic across dozens of visuals."

**Senior signal:** framing steps as a repeatable recipe that re-runs on refresh,
not one-time manual cleaning.

---

### Q: What is query folding and why do you care?
**Keywords:** push to source · native query · faster refresh · less memory

**Spoken answer:**
"Query folding is when Power Query translates my steps into a native query —
SQL, for example — and pushes the work back to the source, so the source does
the heavy lifting instead of my machine. That means faster refresh and lower
memory. I preserve it by doing foldable steps first, and I verify it by
right-clicking a step and choosing View Native Query — if that's available,
the step folded."

**Senior signal:** mentioning View Native Query as the verification method.

---

### Q: How do you handle data quality in Power Query?
**Keywords:** typing first · column profiling · dedup · null handling · reconciliation

**Spoken answer:**
"Several controls. First I set explicit data types as the earliest step,
because wrong types silently break relationships and DAX. I turn on column
profiling — quality, distribution and profile — and crucially set it to the
entire dataset, not the default 1,000-row sample, so my checks aren't lying to
me on big tables. I check keys for uniqueness, handle nulls where the profiler
shows them, normalize text with trim and clean, and finally reconcile row
counts against the source as an integrity check. On BankSphere 360 the 60,000
transaction rows reconciled exactly to the source count."

**Senior signal:** the entire-dataset profiling point and source reconciliation
are details juniors skip.

---

### Q: Why set data types explicitly instead of trusting auto-detect?
**Spoken answer:**
"Auto-detect guesses from a sample and can get it wrong — typing a money column
as whole number would chop off the decimals, or a key as text would silently
fail to match across tables. Explicit typing protects every downstream
relationship and DAX measure, and it's the foundation step before any other
cleansing. I'd rather control it than inherit a guess."

---

### Q: What's the difference between a data type and a display format?
**Keywords:** type = how it's stored · format = how it's shown · separate layers

**Spoken answer:**
"They're separate concerns. The data type is how the value is stored — a Date
is a true date value. The display format is just how it's shown — 9/11/1962 vs
1962-09-11 are the same value in different costumes. In Power Query I care about
the type being correct; presentation formatting belongs in the model or the
visual. The raw format in the file doesn't dictate the display."

**Senior signal:** knowing formatting is a downstream (model/visual) concern,
not a Power Query one.

---

### Q: How does Remove Duplicates work, and what's its limitation?
**Keywords:** Table.Distinct · keeps first occurrence · sort first · or dedup at source

**Spoken answer:**
"Remove Duplicates adds a Table.Distinct step keyed on the column I select, and
it re-runs on every refresh so the pipeline stays clean. The limitation is that
it keeps the *first* occurrence — it doesn't know which row is correct or
newest. So if duplicates are the same ID with different details, I'd sort
newest-first before deduping to guarantee I keep the latest version. Better
still, in production I'd dedup at the source with a ROW_NUMBER view so the model
never even ingests duplicates."

**Senior signal:** the "keeps first occurrence → sort first → or fix at source"
chain shows you understand the mechanism, its risk, and the better architecture.

---

### Q: Why did you clean the City column when it already looked clean?
**Keywords:** defensive · trailing spaces fragment groupings · trim/clean/proper

**Spoken answer:**
"Defensively. Text from source systems often has invisible junk — trailing
spaces, control characters, inconsistent casing. If 'Mumbai' and 'Mumbai ' with
a trailing space both exist, Power BI treats them as two different cities and
my groupings and slicers fragment. Trim removes spaces, Clean removes
non-printing characters, and Proper-case normalizes casing. The data looked
clean, but the guard means a messy future extract can't break my groupings."

---

### Q: You expected null interest rates but didn't add a replace-null step. Why?
**Keywords:** profiler showed 0% empty · don't add unjustified steps

**Spoken answer:**
"Because the column profiler showed zero empties. The roadmap flagged null
interest rates as a *possible* issue, but the data didn't have any — the
zero-rate accounts were stored as actual 0, not null. Adding a replace-null
step on a column with no nulls is clutter, and a reviewer would rightly ask why
it's there. I let the data drive the cleansing decision rather than applying
steps blindly."

**Senior signal:** restraint — not over-engineering — is a maturity signal.

---

### Q: What is a Conditional Column and how did you use it?
**Keywords:** Table.AddColumn · cascading if/else · DPD risk buckets

**Spoken answer:**
"A Conditional Column builds an if/else cascade without writing code — it
generates the M for me. On FactLoans I used it to turn raw Days-Past-Due
numbers into risk buckets labeled with RBI asset-classification terms —
Current, Watch, Substandard, Doubtful, Loss — each carrying the DPD range. The
cascade works top-down: I catch 0 first, then 'less than or equal to 30' means
1-30 because the prior clause already removed the lower values, and so on. That
let the Risk page speak the regulator's language instead of showing raw
numbers."

**Senior signal:** the cascade logic explanation + using regulator terminology
shows both technical and domain depth.

---

### Q: Why type a derived/conditional column explicitly?
**Spoken answer:**
"A conditional or custom column comes in as the 'any' type because Power Query
doesn't infer it. Leaving a column untyped is a loose end — it can cause
ambiguous sorting, relationship, or DAX behavior and looks unfinished. I set
the DPD Bucket to Text and AbsBalance to Decimal so every column, including
derived ones, has a committed type."

---

### Q: How do you reconcile that no data was lost during import?
**Keywords:** row count vs source · wc -l / source query count · TC-01

**Spoken answer:**
"I compare the loaded row count to the source. For the 60,000-row transaction
table I knew the source count from the file, and Power Query's profiler
confirmed Count and Distinct both at 60,000 with the ID perfectly unique. That
reconciliation is a formal UAT test — it's the integrity proof that lets
stakeholders trust everything built on top."

---

### Q: Import vs DirectQuery — and which did this project use? *(carryover, often paired)*
**Spoken answer:**
"Import copies data into the in-memory VertiPaq engine — fastest, full feature
support, refreshed on schedule — and that's what BankSphere 360 uses because
the volumes are small. DirectQuery leaves queries in the source for very large
or real-time data, at the cost of speed and some feature limits. Composite
blends them with aggregations for big datasets."

---

## Quick-reference: M functions used today

| Function | What it does | Where used |
|---|---|---|
| `Table.PromoteHeaders` | First row becomes column names | every table |
| `Table.TransformColumnTypes` | Sets data types | every table |
| `Table.Distinct` | Removes duplicate rows by key | DimCustomer |
| `Text.Trim` | Strips leading/trailing spaces | DimCustomer.City |
| `Text.Clean` | Removes non-printing characters | DimCustomer.City |
| `Text.Proper` | Capitalizes each word | DimCustomer.City |
| `Table.AddColumn` + `Number.Abs` | Adds magnitude column | FactAccounts.AbsBalance |
| `Table.AddColumn` + `if/else if` | Conditional bucket column | FactLoans.DPD Bucket |
