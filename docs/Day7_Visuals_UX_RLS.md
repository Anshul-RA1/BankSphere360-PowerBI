# Day 7 — Visuals, UX, Drillthrough & Row-Level Security

> Project: **BankSphere 360** — Power BI Banking Analytics
> Stage in lifecycle: requirements → source analysis → ETL → data model → measures → **visuals + security (you are here)** → deploy → support
> Goal of the day: build three report pages, a drillthrough page, implement dynamic RLS, and understand the UX principles behind a professional banking dashboard.

---

## 1. Dashboard design principles

Before building, three rules that govern every decision:

**Z-pattern read:** users scan a dashboard top-left → top-right → bottom-left → bottom-right. Place the most important information (headline KPIs) top-left. Detail and supporting charts go bottom. Every page in BankSphere 360 follows this: title top-right, cards across, charts below.

**Consistency over decoration:** one color scheme, one card style, one font. A dashboard that looks "designed by committee" (four different card styles, random colors) signals a junior. A uniform aesthetic signals governance. Every formatting choice was applied consistently across all four pages.

**Formatting at the visual level vs measure level:** currency formatting (₹, 0 decimals) is set on the measure — so it's consistent everywhere the measure appears. Visual-level formatting (card size, font, background) is set per visual. These are separate concerns.

---

## 2. Setup steps (apply before first visual)

**Canvas ratio:** kept at **16:9** — the standard for Power BI. Most screens, monitors, projectors, and the Power BI Service viewer are 16:9. A different ratio causes scroll bars or clipping in the Service.

**Theme:** applied a built-in theme (View → Themes) for consistent colors and fonts across all visuals. A theme governs centrally — you don't style visuals one by one.

**Page naming:** right-click tab → Rename before building. Naming pages from the start prevents bookmark and navigation issues later.

**Auto-detect relationships OFF:** File → Options → Current File → Data Load → untick "Autodetect new relationships after data is loaded." This was set on Day 4 and stays off throughout.

---

## 3. Page 1 — Executive Overview

**Persona:** CXO / Branch Head (filtered by RLS)
**Purpose:** bank-wide KPIs, growth trends, branch performance at a glance

### Visuals built

**Five KPI cards (top row):**
- Total Deposits (₹794M)
- Total Loan Book (₹4bn)
- NPA % (11.66%)
- Active Customers (1,534)
- CASA Ratio (67.28%)

Format applied: light background fill (#F4F6F8), no border, no underline accent bars, consistent font size (28–32pt for callout value). Formatting set consistently across all five.

**Monthly Transaction Value & YoY Growth (line and stacked column chart):**
- X-axis: `DimDate[MonthName]` (sorted by Month number via Sort-by-Column)
- Column Y-axis: `Total Txn Value`
- Line Y-axis: `Txn YoY%`
- Dual axis: left = ₹ value, right = % growth
- Title: `Monthly Transaction Value & YoY Growth`
- MonthName sorted correctly (January → December) by applying Sort-by-Column → Month in Table view

**Top Branches — Transaction Value (clustered bar chart):**
- Y-axis: `DimBranch[BranchName]`
- X-axis: `Total Txn Value`
- Tooltip: `Branch Rank`
- Sorted descending by Total Txn Value (Rank 1 = Bengaluru MG Road at top)
- Horizontal bars chosen so branch names are readable without rotation
- Title: `Top Branches — Transaction Value`

**Three slicers:**
- Segment (DimCustomer[Segment]) — tile style
- Year (DimDate[Year]) — tile style, blank filtered out
- Region (DimBranch[Region]) — tile style

### Slicer interaction test — West region

Selecting "West" filtered all visuals simultaneously:
- Total Deposits: ₹794M → ₹241M (West has 3 of 10 branches)
- Total Loan Book: stayed ₹4bn (FactLoans has no direct branch path — known model design)
- NPA%: stayed 11.66% (same reason)
- Active Customers: 1,534 → 732 (correct — DISTINCTCOUNT filtered via FactAccounts → DimBranch)
- CASA Ratio: 67.28% → 64.57% (West has a slightly lower CASA mix)
- Top Branches chart: 10 branches → 3 West branches only
- Line chart: bars dropped from ~₹1bn to ~₹0.4bn (3 branches worth of activity)

**Conclusion:** cross-filtering working correctly across all connected visuals. Star schema relationships propagating filters as designed. ✅

### Key formatting decisions

- **Sort-by-Column for MonthName:** set in Table view (Column tools → Sort by column → Month). Without this, months display alphabetically (April before January). Same pattern used for DPD Bucket → DPD Bucket Order.
- **No visual borders on charts:** Format → General → Effects → Visual border → Off for all visuals. Clean surfaces, no box clutter.
- **Title as floating text box:** Insert → Text box, no border, consistent font. Not a visual title property — a separate element so it can be positioned freely.

---

## 4. Page 2 — Loan & Risk

**Persona:** Risk Officer
**Purpose:** NPA exposure, DPD risk distribution, loan product quality

### Visuals built

**Three KPI cards:**
- NPA % (11.66%) — the headline risk ratio
- NPA Amount (₹516M) — absolute bad-loan exposure
- Total Loan Book (₹4bn) — the denominator context

**Loan Portfolio — DPD Risk Classification (clustered column chart):**
- X-axis: `FactLoans[DPD Bucket]`
- Y-axis: `Outstanding by DPD` (new measure: `SUM(FactLoans[OutstandingAmount])`)
- Sorted: DPD Bucket ascending (sorts by DPD Bucket Order via Sort-by-Column)
- Conditional bar colors:
  - Current (0 DPD) → Green (#00B050)
  - Watch (1-30 DPD) → Amber (#FFC000)
  - Substandard (31-60 DPD) → Orange (#FF6600)
  - Doubtful (61-90 DPD) → Dark Orange (#CC3300)
  - Loss / NPA (90+ DPD) → Red (#FF0000)
- Results: Current ₹3.2bn, Watch ₹0.3bn, Substandard ₹0.2bn, Doubtful ₹0.2bn, Loss ₹0.5bn

**Why a new measure (Outstanding by DPD) instead of NPA Amount:**
`NPA Amount` only returns values where NPAFlag=1, producing a single bar (Loss bucket only). The DPD chart needs outstanding across *all* buckets — so `Outstanding by DPD = SUM(FactLoans[OutstandingAmount])` was created. The X-axis filter context (DPD Bucket) automatically applies the per-bucket filter; no CALCULATE needed.

**Loan Book by Product (matrix):**
- Rows: `DimProduct[ProductName]`
- Values: `Total Loan Book`, `NPA Amount`, `NPA %`
- Display units: Millions (₹826M not ₹825,776,581)
- NPA% column: conditional coloring (green = low NPA, dark red = high NPA)
- Results: Home Loan highest NPA% (14%), Personal Loan lowest (10%)
- Total row reconciles to ₹4,427M loan book, ₹516M NPA, 12% — matching KPI cards ✅

**Total Txn by Month (line chart):**
- X-axis: `DimDate[MonthName]`
- Y-axis: `Total Txn Value`
- Shows genuine monthly variation (not flat) — gives the Risk page temporal context

**NPA Amount was rejected as a trend line** because FactLoans is a snapshot — plotting NPA Amount by month shows a flat line (same ₹516M every month). A flat line adds no insight. `Total Txn Value` by month was chosen instead as it shows real variation and gives context to the risk numbers.

**DPD Bucket slicer:** allows Risk Officer to filter all visuals to a specific risk band.

### DPD sort-order lesson (resolved from Day 6)

The Sort-by-Column approach (DPD Bucket sorted by DPD Bucket Order) avoids the circular dependency that arises when using a DAX calculated column for sorting. Power Query columns are evaluated at load time, independent of the DAX layer — no circular reference possible. The color-coded sort (Current → Loss, green → red) is the visual payoff of this work.

---

## 5. Page 3 — Customer & Channel

**Persona:** Relationship Manager / Compliance
**Purpose:** customer engagement, channel distribution, fraud exposure, KYC compliance

### Visuals built

**Two KPI cards:**
- Fraud Flagged Value (₹31M) — compliance headline
- Total Transactions (60K) — activity volume

**Active Customers by Segment (clustered bar chart):**
- Y-axis: `DimCustomer[Segment]`
- X-axis: `Active Customers`
- Results: Retail 1,090 | SME 299 | HNI 145 | Total 1,534 ✅ (reconciles)
- Color gradient: green (Retail) → pink (SME) → dark red (HNI)

**Transaction Value by Channel (donut chart):**
- Legend: `FactTransactions[Channel]`
- Values: `Total Txn Value`
- Six channels all ~16-17% each (Branch, NetBanking, POS, UPI, ATM, MobileApp)
- Story: healthy omnichannel distribution — no single channel dominates

**Customer KYC Status (matrix):**
- Rows: `DimCustomer[KYCStatus]`
- Values: `Active Customers`, `Total Deposits`
- Results: Verified 1,238 / ₹636M | Pending 190 / ₹104M | Re-KYC Due 106 / ₹54M
- Total: 1,534 customers / ₹794M — reconciles with Page 1 exactly ✅
- Key compliance insight: ₹158M in deposits held by customers with incomplete KYC

**Segment slicer:** tile style, consistent with Pages 1 and 2

---

## 6. Page 4 — Branch Detail (Drillthrough)

**Purpose:** hidden detail page, activated by right-clicking any branch anywhere in the report

### How drillthrough works

1. Created a new page named "Branch Detail."
2. In the Build pane → **Drillthrough** field well → dragged `DimBranch[BranchName]`.
3. Power BI automatically: added a Back button (←), set the page as a drillthrough target, configured all visuals to auto-filter by the drilled-through branch.
4. To use: right-click any branch name on any page → "Drill through → Branch Detail" → page opens filtered to that branch.
5. Back button returns to the originating page.

**No slicer needed** — the drillthrough filter acts as an automatic, invisible filter on every visual.

### Visuals built

**Branch Name card:**
- Field: `DimBranch[BranchName]` → Summarization: **First**
- Shows "Ahmedabad CG Road" by default (first alphabetically); updates dynamically to the drilled-through branch
- Styled prominently as the page identifier

**Three KPI cards:**
- `Total Txn Value` — this branch's transaction activity
- `Active Customers` — this branch's customer count
- `Total Deposits` — this branch's deposit balance

**Channel Mix (donut):**
- Same pattern as Page 3 but auto-filtered to one branch
- Shows whether this branch's customers prefer digital or physical channels

**Monthly Transaction Trend (line chart):**
- X-axis: `DimDate[MonthName]`
- Y-axis: `Total Txn Value`
- Auto-filtered to this branch — shows genuine branch-specific monthly patterns

### Drillthrough test result (Bengaluru MG Road)

Drilled from Page 1 Top Branches chart → Bengaluru MG Road (Rank 1):
- Branch Name card: "Bengaluru MG Road" ✅
- Total Txn Value: ₹2bn (13% of bank's ₹15bn — highest branch) ✅
- Active Customers: 312 ✅
- Total Deposits: ₹91M ✅
- Channel mix: all six channels roughly equal ✅
- Monthly trend: genuine variation (peak July, dip February) ✅

---

## 7. Row-Level Security (RLS)

### What RLS does

RLS restricts which *rows* of data a user can see based on their identity. A Branch Head for the West region should see only West data — not North, South, or East. Without RLS, every published user would see the full bank-wide dataset.

### Static vs Dynamic RLS

**Static RLS:** one role per filter value.
```
Role: West_BranchHead → filter: DimBranch[Region] = "West"
Role: North_BranchHead → filter: DimBranch[Region] = "North"
```
Simple to set up but doesn't scale — 100 branch heads = 100 roles.

**Dynamic RLS:** one role, user-specific filtering via a mapping table.
```
Role: BranchHead → filter: DimBranch[Region] =
    LOOKUPVALUE(UserRegionMap[Region], UserRegionMap[Email], USERPRINCIPALNAME())
```
One role serves all branch heads. Power BI reads the logged-in user's email, looks up their permitted region in a mapping table, and applies that as the filter. Scales to thousands of users.

### Implementation steps

**Step 1 — Create UserRegionMap table (Enter Data):**
| Email | Region |
|---|---|
| west.head@bank.com | West |
| north.head@bank.com | North |
| south.head@bank.com | South |
| east.head@bank.com | East |

**Step 2 — Create the RLS role (Modeling → Manage roles → New role):**
- Role name: `BranchHead`
- Table: `DimBranch`
- DAX filter:
```dax
[Region] = LOOKUPVALUE(
    UserRegionMap[Region],
    UserRegionMap[Email],
    USERPRINCIPALNAME()
)
```

**Step 3 — Test with View as roles:**
Modeling → View as → tick BranchHead → enter a test email → confirm only that region's data is visible across all pages.

**Step 4 — Publish and assign members:**
After publishing to Power BI Service → Dataset → Security → assign actual user emails to the BranchHead role.

### How filter propagation works

The RLS filter is applied on `DimBranch[Region]`. Because of the star schema relationships:
- DimBranch → FactAccounts → filters accounts by region ✅
- FactAccounts → FactTransactions → filters transactions by region ✅
- But FactLoans has no DimBranch path → loan data is NOT filtered by region (known limitation)

This is why NPA% and Total Loan Book didn't change during the West slicer test on Day 7 — the same model path limitation applies to RLS. For a production system, a BranchID on FactLoans would resolve this.

### Why RLS matters in banking

A Branch Head must never see another branch's customer data — this is both a data governance requirement and in many jurisdictions a regulatory requirement. Without RLS, publishing to the Service exposes the full customer master and transaction history to every viewer. RLS is non-negotiable in a production banking deployment.

---

# INTERVIEW QUESTIONS — Day 7 (Visuals, UX & RLS)
> Added: 2026-06-05
> Format: keyword trigger → spoken answer → senior signal

---

### Q: How do you approach dashboard layout and design?
**Keywords:** Z-pattern · hierarchy · consistency · formatting at measure level

**Spoken answer:**
"I follow a Z-pattern layout — headline KPIs top-left, supporting charts below,
title top-right. The most important number is always in the top-left because
that's where users look first. I apply one consistent style: same card format,
same color scheme, same font — consistency signals governance. Formatting goes
on the measure, not the visual, so a currency measure always displays as ₹
wherever it appears. I also choose the simplest visual type that tells the
story — a bar chart over a pie chart for comparisons, a line for trends, a
card for single KPIs."

---

### Q: What is drillthrough and how does it work?
**Keywords:** right-click → drill through → hidden page · drillthrough field well · auto-filtered · Back button

**Spoken answer:**
"Drillthrough is a navigation feature where a user right-clicks a data point
on any page and jumps to a detail page pre-filtered to that item. I set it up
by dragging the filter field — BranchName in my case — into the Drillthrough
well on the detail page. Power BI automatically adds a Back button and makes
every visual on that page inherit the drillthrough filter. On BankSphere 360,
right-clicking Bengaluru MG Road on the Top Branches chart opens the Branch
Detail page showing only Bengaluru's metrics — transaction value, customer
count, channel mix, monthly trend. No slicer needed; the filter is invisible
and automatic."

**Senior signal:** mentioning that no slicer is needed — the drillthrough
filter is implicit — shows you understand the mechanism, not just the clicks.

---

### Q: Static vs dynamic RLS — when do you use which?
**Keywords:** static = one role per value · dynamic = mapping table + USERPRINCIPALNAME · scales

**Spoken answer:**
"Static RLS creates one role per filter value — a West role, a North role etc.
It's simple but doesn't scale: 100 branch heads means 100 roles to maintain.
Dynamic RLS uses a user-to-attribute mapping table and USERPRINCIPALNAME() —
one role serves everyone. Power BI reads the logged-in user's email, looks up
their permitted region in the mapping table, and applies that as the row filter
automatically. For BankSphere 360 I used dynamic RLS so one BranchHead role
serves all regional heads without any code change as staff rotate."

**Senior signal:** mentioning staff rotation as the real-world reason dynamic
scales better — it shows you think about maintenance, not just setup.

---

### Q: Where is RLS enforced and who bypasses it?
**Keywords:** model layer · Service assignment · admins bypass · workspace owners bypass

**Spoken answer:**
"RLS is defined in the model (Desktop) and enforced in the Service after
publish. The roles are defined in Desktop with DAX filter expressions; members
are assigned to roles in the Service under Dataset → Security. Critically,
workspace admins and dataset owners bypass RLS by default — they always see
all data regardless of role membership. This is a common gotcha: if you test
RLS as the dataset owner, you'll always see everything and incorrectly conclude
it's working. You must use 'View as roles' in Desktop to test properly, or
assign a non-owner test account in the Service."

**Senior signal:** the "dataset owners bypass RLS" point is the one most
candidates miss — it's a genuine production trap.

---

### Q: How do you test RLS before publishing?
**Keywords:** View as roles · Desktop testing · enter email · confirm filter

**Spoken answer:**
"In Power BI Desktop: Modeling → View as → select the role → optionally enter
a test email to simulate USERPRINCIPALNAME(). The report then renders as that
user would see it — I confirm only their permitted data is visible across all
pages. After publishing, I test again in the Service using a non-owner account
assigned to the role, because owners bypass RLS and Desktop testing alone isn't
sufficient for sign-off."

---

### Q: Why sort a column by a different column, and what's the circular dependency risk?
**Keywords:** text sorts alphabetically · sort-by-column · DAX column = circular · Power Query column = safe

**Spoken answer:**
"Text columns sort alphabetically by default, which breaks any categorical
order that isn't alphabetical — month names, risk buckets, custom sequences.
Sort-by-Column lets you assign a numeric sort key so Power BI uses that key
instead of alphabetical order. The risk is circular dependency: if the sort
key is a DAX calculated column that references the column being sorted, Power
BI detects a loop and blocks it. The fix is to build the sort key in Power
Query, where it's evaluated at load time independently of DAX. I hit this
on the DPD Bucket sort and resolved it by moving the order column to Power
Query."

---

### Q: How do you choose which visual type for which insight?
**Keywords:** card = single KPI · bar = comparison · line = trend · donut = composition · matrix = multi-measure by category

**Spoken answer:**
"Card for a single headline number — NPA%, Total Deposits. Bar chart for
comparisons between categories — branches ranked by transaction value, segments
by customer count. Line chart for trends over time — monthly transaction value,
YoY growth. Donut for composition where parts sum to a whole — channel mix.
Matrix for multi-measure analysis by category — loan book, NPA amount, and NPA%
by product in one table. I avoid pie charts (hard to read precisely) and 3D
visuals (distort perception). The rule is: choose the simplest visual that
answers the business question."

---

### Q: What is cross-filtering and how does it work in your model?
**Keywords:** slicer filters → relationships propagate → all connected visuals update

**Spoken answer:**
"Cross-filtering means that selecting a value in one visual — a slicer, a bar,
a map region — filters every other visual on the page through the model's
relationships. In BankSphere 360, selecting 'West' in the Region slicer filters
DimBranch, which propagates through FactAccounts to FactTransactions, updating
the KPI cards, the line chart, and the top branches bar simultaneously. The
filter flows along active relationships in the star schema — single-direction
by default, from the one-side to the many-side. Visuals connected to tables
that have no path to DimBranch (like FactLoans) don't update, which is a known
model limitation."

---

### Q: How do you ensure a dashboard is production-ready before publishing?
**Keywords:** slicer test · RLS test · reconciliation · performance · naming · runbook

**Spoken answer:**
"Several checks. First, a slicer interaction test — click every slicer value
and confirm all visuals update consistently and the numbers make sense. Second,
a reconciliation check — totals on detail pages should match the headline KPI
cards. Third, RLS tested with View-as-roles for every role. Fourth, Performance
Analyzer run on every page — all visuals under 500ms. Fifth, naming standards:
all measures described, pages named clearly, no 'Page 1' or 'Measure 1' in
the final file. Then publish to Dev, test in Service, promote to Prod via
deployment pipelines."

---

## Quick reference — the four pages

| Page | Persona | Key visual | Interview story |
|---|---|---|---|
| Executive Overview | CXO / Branch Head | Monthly trend + YoY dual axis | Z-pattern, slicers, cross-filtering |
| Loan & Risk | Risk Officer | DPD bucket with conditional colors | RBI classification, Sort-by-Column |
| Customer & Channel | RM / Compliance | KYC matrix + channel donut | Reconciliation, compliance insight |
| Branch Detail | Any (drillthrough) | Dynamic branch name card | Drillthrough mechanism, auto-filter |
