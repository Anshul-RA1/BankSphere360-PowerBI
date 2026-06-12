# NOTES.md — BankSphere 360: Power BI Banking Analytics Project

> Your complete 10-day learning + interview-prep companion.
> Project codename: **BankSphere 360 — Retail & SME Banking Performance Analytics**
> Target role: **Microsoft Power BI Developer / Analyst (6+ yrs)**

---

## 0. How to use this file

Every Power BI skill in the job description maps to a day below. Each day has:
**Concept → Why it matters in banking → Hands-on steps → Interview hooks.**
Do the work in Power BI Desktop yourself; reading is not learning. Build the `.pbix`, break it, fix it.

The job description skills, mapped:

| JD Skill | Covered on Day |
|---|---|
| Power BI dashboards & reports | 4, 7, 8 |
| Business requirements → technical solutions | 1, 2 |
| Data models, KPIs, scorecards | 3, 5 |
| DAX calculations & measures | 5, 6 |
| Power Query transformations | 3 |
| Multi-source integration (SQL, Excel, API, cloud, ERP) | 2 |
| Data validation, cleansing, quality | 3 |
| Performance optimization & scalability | 6, 9 |
| Stakeholder collaboration | 1, 10 |
| Row-Level Security (RLS) & governance | 7 |
| Production support, enhancements, deployment | 9 |

---

## Day 1 — Domain, requirements, and the BI lifecycle

**Concept.** BI delivery follows: requirements → source analysis → data model → ETL → measures → visuals → security → deploy → support. Power BI is one tool inside that lifecycle.

**Why banking.** A retail bank earns on Net Interest Margin (interest earned on loans minus interest paid on deposits) and fees, and loses on bad loans (NPAs — Non-Performing Assets, where the borrower has not paid for 90+ days). Leadership wants three things: *Are deposits & loans growing? Is the loan book healthy? Are customers active across channels?*

**Hands-on.**
1. Read the **Project Summary** and **SSRD** documents. Note the 5 personas (CXO, Branch Head, Risk Officer, Relationship Manager, Compliance).
2. Write down the 8 KPIs you must deliver: Total Deposits, Total Loan Book (Outstanding), CASA Ratio, NPA %, Net Interest Margin proxy, Active Customer %, Channel Mix, Fraud Flagged Value.
3. For each KPI, write the *business question* it answers in one sentence.

**Interview hooks.**
- "Walk me through your BI development lifecycle." → use the chain above.
- "How do you gather requirements?" → personas, KPI definitions signed off in writing, mock-up first, avoid scope creep.

---

## Day 2 — Connecting to multiple sources

**Concept.** Power BI connects via **connectors**. Each source has a *connector* + a *connectivity mode* (Import vs DirectQuery). Import = data copied into Power BI's in-memory engine (VertiPaq), fast, refreshed on schedule. DirectQuery = queries left in the source, used for huge or real-time data.

**Why banking.** Core banking on **SQL Server / Oracle**, customer master in an **ERP**, KYC exceptions in **Excel**, exchange rates from a **REST API**, statements in **cloud (ADLS / SharePoint)**. A real bank report stitches all of these.

**Hands-on.**
1. We ship CSVs (a stand-in for a SQL warehouse export). In Power BI: **Get Data → Text/CSV** for each of the 8 files in `/data`.
2. Simulate the multi-source story: load `DimDate`, `DimBranch`, `DimProduct`, `DimCustomer` as your "ERP/master" feeds and the three Fact files as your "core banking" feeds.
3. (Optional) **Get Data → Web** and point at any public exchange-rate JSON to prove API ingestion. Note: in production you would parameterize the URL and key.

**Interview hooks.**
- "Import vs DirectQuery — when do you pick which?" → volume, latency, source load, model size, feature support (some DAX/Power Query limited in DirectQuery).
- "How do you connect to SQL Server securely?" → On-premises Data Gateway + stored credentials + least-privilege service account.

---

## Day 3 — Power Query: cleansing, shaping, the M language

**Concept.** **Power Query** (engine = "M") is the ETL layer. Every action is a *step* in `Applied Steps`. **Query Folding** = Power Query pushes transformations back to the source as SQL; preserve it for performance by doing source-foldable steps first.

**Why banking.** Raw core-banking extracts have nulls, inconsistent date keys, negative balances for loans, mixed casing in city names, and duplicate customer rows from system merges. Clean once, in Power Query, not in 40 visuals.

**Hands-on (do each as a named step).**
1. **Promote headers** & set data types explicitly (DateKey as whole number, Amount as decimal, Date as date). Wrong types silently break relationships and DAX.
2. **Trim/Clean/Proper-case** `City` in `DimCustomer`.
3. **Replace errors / nulls** in `InterestRate` with 0.
4. **Remove duplicates** on `CustomerID` in `DimCustomer`.
5. Add a **conditional column** `LoanFlag` = 1 if `ProductCategory = "Loans"`.
6. Add **custom column** `AbsBalance = Number.Abs([Balance])` for loan magnitude.
7. Filter `FactTransactions` to keep only `Status` rows you need; **disable load** on staging queries.
8. Inspect **Column quality / distribution** (View tab) to validate — this is your *data quality check*.

**Interview hooks.**
- "What is query folding and why care?" → fold = source does the work = faster refresh & less memory; check via *View Native Query*.
- "How do you handle data quality?" → typing, dedup, null handling, column profiling, reconciliation totals vs source.

---

## Day 4 — Data modeling: the star schema & relationships

**Concept.** A **star schema** = central *fact* tables (events/measures) surrounded by *dimension* tables (descriptive). Relationships are 1-to-many (one dimension row → many fact rows). Filters flow **from the one-side to the many-side** by default. Avoid snowflaking and many-to-many unless necessary.

**Why banking.** `FactTransactions`, `FactAccounts`, `FactLoans` are facts. `DimDate`, `DimCustomer`, `DimProduct`, `DimBranch` are shared dimensions (conformed). One `DimDate` feeds all facts — that's how you slice every metric by time consistently.

**Hands-on.**
1. In **Model view**, create relationships:
   - `DimCustomer[CustomerID]` → `FactAccounts[CustomerID]`
   - `DimProduct[ProductID]` → `FactAccounts[ProductID]` and → `FactLoans[ProductID]`
   - `DimBranch[BranchID]` → `FactAccounts[BranchID]`
   - `DimDate[DateKey]` → `FactTransactions[DateKey]`, `FactAccounts[OpenDateKey]`, `FactLoans[SanctionDateKey]`
   - `FactAccounts[AccountID]` → `FactTransactions[AccountID]` and → `FactLoans[AccountID]`
2. Mark `DimDate` as a **Date table** (Table tools → Mark as date table).
3. Only one **active** relationship is allowed between two tables; extras become inactive (used later with `USERELATIONSHIP`).
4. Hide foreign-key columns on the fact side from report view (clean field list).

**Interview hooks.**
- "Star vs snowflake?" → star = denormalized dims, fewer joins, faster, simpler DAX; snowflake = normalized, more joins.
- "Why a dedicated Date table?" → continuous dates, time-intelligence functions need it, mark-as-date-table enables `DATEADD` etc.
- "Single vs bidirectional cross-filter?" → default single; bidirectional risks ambiguity & perf, use sparingly (e.g., dimension-to-dimension via bridge).

---

## Day 5 — DAX foundations: measures, calculated columns, context

**Concept.** **DAX** = Data Analysis Expressions. Two contexts: **row context** (current row, used in calculated columns / iterators like `SUMX`) and **filter context** (the slicers/visual filters in effect). `CALCULATE` is the only function that *modifies* filter context — the most important function in DAX.

**Measure vs calculated column:** measure = computed at query time, aggregated, no storage cost, use for KPIs. Calculated column = computed at refresh, stored per-row, use for slicing/relationships.

**Why banking.** Every KPI on the dashboard is a measure. Getting `CALCULATE` and context right is the difference between a correct NPA% and a wrong one.

**Hands-on — build these core measures (full DAX in `dax_measures.txt`):**
```
Total Deposits        = CALCULATE( SUM(FactAccounts[Balance]),
                                   DimProduct[ProductCategory] = "Deposits" )
Total Loan Book       = CALCULATE( SUM(FactLoans[OutstandingAmount]) )
Total Transactions    = COUNTROWS(FactTransactions)
Total Txn Value       = SUM(FactTransactions[Amount])
NPA Amount            = CALCULATE( SUM(FactLoans[OutstandingAmount]),
                                   FactLoans[NPAFlag] = 1 )
NPA %                 = DIVIDE( [NPA Amount], [Total Loan Book] )
Active Customers      = CALCULATE( DISTINCTCOUNT(FactAccounts[CustomerID]),
                                   FactAccounts[Status] = "Active" )
```
- Always wrap division in **`DIVIDE()`** (handles divide-by-zero).
- Build a **measures table** (empty table to hold all measures) for organization.

**Interview hooks.**
- "Difference between SUM and SUMX?" → SUM aggregates a column; SUMX iterates row-by-row evaluating an expression (e.g., `SUMX(Fact, qty*price)`).
- "Explain row vs filter context." → use a calculated column vs a measure example.
- "What does CALCULATE do?" → evaluates an expression in a modified filter context.

---

## Day 6 — Advanced DAX: time intelligence, RANKX, performance

**Concept.** Time-intelligence functions (`TOTALYTD`, `SAMEPERIODLASTYEAR`, `DATEADD`) need a marked date table. `RANKX` ranks. `VAR` stores intermediate results (readability + performance — evaluated once). Performance: avoid calculated columns where a measure works, minimize cardinality, avoid `FILTER` over whole tables when a boolean column filter suffices.

**Why banking.** Leadership wants **MoM and YoY growth** of deposits and loans, **top branches**, and **NPA trend**. These are all time-intelligence + ranking.

**Hands-on.**
```
Deposits YTD     = TOTALYTD( [Total Deposits], DimDate[Date] )
Txn Value LY     = CALCULATE( [Total Txn Value], SAMEPERIODLASTYEAR(DimDate[Date]) )
Txn YoY %        = DIVIDE( [Total Txn Value] - [Txn Value LY], [Txn Value LY] )
Txn MoM %        = VAR Cur = [Total Txn Value]
                   VAR Prev = CALCULATE([Total Txn Value], DATEADD(DimDate[Date],-1,MONTH))
                   RETURN DIVIDE(Cur - Prev, Prev)
Branch Rank      = RANKX( ALL(DimBranch[BranchName]), [Total Txn Value],, DESC )
CASA Ratio       = VAR CASA = CALCULATE([Total Deposits],
                       DimProduct[ProductName] IN {"Savings Account","Current Account"})
                   RETURN DIVIDE(CASA, [Total Deposits])
```
- Learn **Performance Analyzer** (View tab): record, read the DAX query duration per visual.
- Learn `ALL`, `ALLEXCEPT`, `REMOVEFILTERS`, `KEEPFILTERS`.

**Interview hooks.**
- "How do you compute YoY?" → SAMEPERIODLASTYEAR within CALCULATE, then DIVIDE.
- "Why use VAR?" → evaluated once, readable, avoids repeated context transitions.
- "A report is slow — how do you fix it?" → Performance Analyzer → reduce visuals/cardinality → measures over columns → aggregations → check folding → star schema.

---

## Day 7 — Visuals, UX, bookmarks, and Row-Level Security

**Concept.** Good dashboards follow a **Z-pattern**: KPI cards top, trend middle, detail bottom; consistent colors, minimal clutter, tooltips for depth. **RLS** restricts rows by user: define **roles** with a DAX filter on a dimension, map users to roles in the Service.

**Why banking.** A Mumbai Branch Head must see only Mumbai. A Risk Officer sees the whole loan book. Compliance is read-only. RLS is non-negotiable in a regulated industry.

**Hands-on.**
1. Build 3 report pages: **Executive Overview**, **Loan & Risk (NPA)**, **Customer & Channel**.
2. Page 1: KPI cards (Deposits, Loan Book, NPA %, Active Customers), a line chart (Txn value by month with YoY), a map by region.
3. Use **slicers** (Year, Region, Segment), **bookmarks** + buttons for a guided story, and a **drillthrough** page to branch detail.
4. **RLS:** Modeling → Manage roles → create `BranchHead`:
   `[Region] = USERPRINCIPALNAME()` pattern, or static `DimBranch[Region] = "West"`. For dynamic RLS add a `UserMapping` table relating email → region and filter with `USERPRINCIPALNAME()`.
5. Test with **View as roles**.

**Interview hooks.**
- "Static vs dynamic RLS?" → static = one role per filter value (doesn't scale); dynamic = mapping table + USERPRINCIPALNAME(), scales to thousands of users.
- "Where is RLS enforced?" → model roles defined in Desktop, members assigned in Service; works in Import & DirectQuery; does NOT apply to admins/owners by default.
- "Object-level vs row-level security?" → RLS hides rows; OLS hides tables/columns.

---

## Day 8 — Scorecards, KPI visuals, storytelling

**Concept.** A **scorecard** presents KPIs against targets with status (good/warning/bad). In Power BI use the **KPI visual**, **cards with conditional formatting**, or **Goals/Metrics** in the Service.

**Hands-on.**
1. Add target measures, e.g. `NPA Target = 0.04` (4%). Conditional-format the NPA card red when `[NPA %] > [NPA Target]`.
2. Build a small scorecard matrix: rows = KPI, columns = Actual / Target / Variance / Status (use a measure returning ▲/▼ or color via field formatting).
3. Add **what-if parameter** (Modeling → New parameter) for "Interest Rate Scenario" to demo sensitivity.

**Interview hooks.**
- "How do you show actual vs target?" → KPI visual / conditional formatting / variance measure.
- "What is a what-if parameter?" → generated table + measure to model scenarios interactively.

---

## Day 9 — Deployment, refresh, gateways, production support

**Concept.** Publish `.pbix` to a **Workspace** in the Power BI Service. A **Dataset (semantic model)** + **Reports** live there. Schedule **refresh** (needs a **gateway** for on-prem sources). Promote through **Dev → Test → Prod** with **Deployment Pipelines**. Production support = monitoring refresh failures, fixing broken visuals, handling enhancement requests via change control.

**Hands-on / talking points.**
1. Publish to a workspace; set **scheduled refresh** (e.g., 6 AM daily) and gateway credentials.
2. Use **Deployment Pipelines** to move from Dev to Prod without re-uploading.
3. Set up **failure notifications**, document a runbook (see NOTES "Runbook" below).
4. Version the `.pbix` in Git (or OneDrive) and keep a change log.

**Runbook (production support).**
- Refresh failed → check gateway online → check source credentials → check source schema change → re-run → notify stakeholders.
- Wrong number reported → reproduce → check measure filter context → check source reconciliation → hotfix in Dev → promote.
- New KPI request → log as change request → estimate → build in Dev → UAT → deploy.

**Interview hooks.**
- "How do you schedule refresh for on-prem SQL?" → On-premises data gateway (standard mode) + scheduled refresh + stored creds.
- "How do you manage environments?" → Deployment Pipelines (Dev/Test/Prod) with parameter rules for source switching.
- "Incremental refresh?" → partition by date, refresh only recent partitions; needs RangeStart/RangeEnd parameters & date filter that folds.

---

## Day 10 — Governance, mock interview, and the close

**Concept.** Governance = workspaces, sensitivity labels, certified datasets, naming standards, documentation, and a single source of truth. Endorsement: **Promoted / Certified** datasets.

**Hands-on.**
1. Apply a naming convention, add descriptions to measures, mark a **certified** dataset (admin).
2. Do a full **mock interview** using the question bank below — out loud, timed.
3. Polish the portfolio: clean pages, a one-line story per page, export a PDF, and push the `.pbix` + docs to GitHub.

---

# INTERVIEW QUESTION BANK

## A. Technical Round (Power BI / DAX / Modeling / SQL)

1. **Import vs DirectQuery vs Composite** — definition, trade-offs, when each.
2. **What is VertiPaq?** Columnar in-memory engine; compression via dictionary & RLE; why low cardinality compresses better.
3. **Star vs snowflake schema** and why star is preferred in Power BI.
4. **Row context vs filter context**; context transition (when `CALCULATE` turns row into filter context).
5. **CALCULATE** — what it does; modifiers (`ALL`, `KEEPFILTERS`, `USERELATIONSHIP`).
6. **SUM vs SUMX** and other X-iterators.
7. **Measure vs calculated column vs calculated table** — storage, timing, use cases.
8. **DIVIDE vs `/`** — divide-by-zero handling.
9. **Time intelligence**: `TOTALYTD`, `SAMEPERIODLASTYEAR`, `DATEADD`, `DATESYTD`; why a marked date table is required.
10. **RANKX / TOPN** patterns.
11. **ALL vs ALLEXCEPT vs REMOVEFILTERS vs ALLSELECTED.**
12. **Active vs inactive relationships**; `USERELATIONSHIP`.
13. **Bidirectional filtering** risks and bridge tables.
14. **Query folding** — what it is, how to verify (View Native Query), what breaks it.
15. **Power Query M vs DAX** — when work belongs in each.
16. **Incremental refresh** — RangeStart/RangeEnd, partitions, folding requirement.
17. **RLS** static vs dynamic; `USERPRINCIPALNAME()`; testing with "View as".
18. **Performance tuning** — Performance Analyzer, cardinality, aggregations, fewer visuals, star schema, measure optimization.
19. **Aggregations / composite models** for big data.
20. **Deployment pipelines**, dataflows, shared/certified datasets.
21. **SQL**: write a query for top 5 branches by transaction value; JOINs; GROUP BY/HAVING; window functions (`ROW_NUMBER`, `RANK`, running totals); difference between WHERE and HAVING.
22. **Gateway types** (standard vs personal) and when each is used.
23. **Calculated KPI**: derive CASA ratio / NPA% and explain the filter context.
24. **Handling slowly changing dimensions** at a conceptual level.

## B. Managerial / Project Round

1. Walk me through a Power BI project end-to-end (use BankSphere 360).
2. How do you gather and freeze requirements? How do you handle scope creep?
3. A stakeholder disputes a number on the dashboard — how do you respond?
4. How do you prioritize when three teams want changes the same week?
5. How do you ensure data quality and build trust in the report?
6. Describe a performance problem you solved and the measurable result.
7. How do you handle a tight deadline (like this 10-day build)?
8. How do you document and hand over a solution?
9. How do you collaborate with data engineers vs business users differently?
10. How do you manage a production incident (refresh failure during month-end)?

## C. HR Round

1. Tell me about yourself. (90-second pitch: years in BI, banking domain, signature project, the value you delivered.)
2. Why are you leaving / looking? (Growth into modern data/AI + BI, not away-from negativity.)
3. Strengths & weaknesses (pick a real, improving weakness).
4. Where do you see yourself in 3–5 years?
5. Tell me about a conflict and how you resolved it.
6. Why this company / role?
7. Salary expectations. (Give a researched range, anchored to market + your seniority.)
8. A time you failed and what you learned.
9. How do you keep your skills current?
10. Do you have questions for us? (Always yes: team structure, data maturity, success metrics for this role in 6 months.)

---

## Model answers — three you should memorize

**"Walk me through a Power BI project."**
> "On BankSphere 360 I started with stakeholder workshops to define five personas and eight KPIs, signed off in an SSRD. I profiled the sources — a SQL core-banking warehouse, an ERP customer master, Excel KYC exceptions — and built the ETL in Power Query, preserving query folding and handling nulls, dedup and typing. I designed a star schema with conformed Date, Customer, Product and Branch dimensions feeding three fact tables. I wrote the KPIs in DAX — deposits, loan book, NPA%, CASA, YoY and MoM with time intelligence — organized in a measures table. I built three report pages with drillthrough and bookmarks, then implemented dynamic RLS so a branch head sees only their region. I published with a gateway and scheduled refresh, promoted through deployment pipelines, and documented a support runbook. The result was a single source of truth that cut manual reporting effort significantly."

**"Explain CALCULATE."**
> "CALCULATE evaluates an expression in a filter context you modify with its filter arguments. It's the only function that changes filter context, and it also triggers context transition — turning the current row context into an equivalent filter context — which is why measures inside an iterator behave the way they do. For example, NPA% is `DIVIDE(CALCULATE(SUM(Outstanding), NPAFlag=1), [Total Loan Book])`."

**"How do you fix a slow report?"**
> "I use Performance Analyzer to find the slow visual and read its DAX query duration. Usual culprits: too many visuals on a page, high-cardinality columns, calculated columns that should be measures, FILTER over whole tables, broken query folding, or a non-star model. I fix the model first (star schema, hide unused columns, reduce cardinality), then the DAX (VAR, boolean filters, DIVIDE), then consider aggregations or composite models for large data."
