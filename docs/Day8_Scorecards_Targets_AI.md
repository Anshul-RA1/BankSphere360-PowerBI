# Day 8 — Scorecards, Targets, What-if & AI Visuals

> Project: **BankSphere 360** — Power BI Banking Analytics
> Stage in lifecycle: requirements → source analysis → ETL → data model → measures → visuals → **scorecards + AI (you are here)** → deploy → support
> Goal of the day: add target measures, conditional formatting, a scorecard matrix, a what-if parameter for scenario analysis, and AI-powered visuals.

---

## 1. The concept — scorecards and targets

A dashboard without targets is just a display. A dashboard with targets tells you whether performance is acceptable or not. The board approves KPI tolerances; the dashboard enforces them visually. This is what separates a reporting tool from a decision-support tool.

**Two types of targets used today:**
- **Hard threshold:** NPA% must stay below 4% (RBI-aligned tolerance). Binary — either within or breached.
- **Soft target:** CASA Ratio should be above 40% (industry benchmark). Directional — higher is better.

---

## 2. Target measures

### NPA Target
```dax
NPA Target = 0.04
```
A constant measure representing the board-approved NPA tolerance of 4%. Stored as 0.04 (decimal) because DAX percentage measures are stored as decimals internally — 11.66% is stored as 0.1166, so the comparison threshold must also be decimal.

**Format:** Percentage, 2 decimals → displays as 4.00%.

### CASA Target
```dax
CASA Target = 0.40
```
Industry benchmark for CASA ratio. Indian banks target 40%+ as a sign of healthy low-cost funding.

**Format:** Percentage, 2 decimals → displays as 40.00%.

### NPA Status
```dax
NPA Status =
IF ( [NPA %] > [NPA Target], "⚠️ At Risk", "✅ Within Target" )
```
Returns a text status string by comparing actual to target. Uses emoji for immediate visual recognition in the scorecard matrix. Reuses existing measures — no recalculation of NPA logic.

### CASA Status
```dax
CASA Status =
IF ( [CASA Ratio] >= [CASA Target], "✅ Good", "⚠️ Below Target" )
```
Same pattern as NPA Status. Note the `>=` (greater than or equal) for CASA vs `>` for NPA — CASA is "higher is better" so meeting the target exactly is acceptable.

### NIM Scenario
```dax
NIM Scenario =
DIVIDE (
    [Total Deposits] * [Interest Rate Scenario Value],
    100
)
```
**NIM = Net Interest Margin** — the difference between interest earned on loans and interest paid on deposits, expressed as a percentage. It is one of the most important profitability metrics in banking. A bank borrows cheaply (savings accounts at 3-4%) and lends expensively (home loans at 8-14%); the spread is where profit is made.

This measure is a simplified NIM proxy: deposit base × interest rate ÷ 100 = estimated interest income. The `Interest Rate Scenario Value` comes from the what-if parameter slider, so the result updates live as the user drags the slider.

- At 5% → ₹794M × 5 ÷ 100 = **₹40M**
- At 7% → ₹794M × 7 ÷ 100 = **₹56M**
- At 15% → ₹794M × 15 ÷ 100 = **₹119M**

**Format:** Currency (₹), 0 decimals.

---

## 3. Conditional formatting on NPA% card

**What it does:** the NPA% card font color changes automatically based on whether the value is above or below the 4% target. No manual intervention needed — the color responds to whatever data is in context (full bank, a region, a product, a segment).

**How it was set up:**
- Click NPA% card → Format → Visual → Callout value → Font color → Fx
- Format style: Rules
- Based on: NPA%
- Rule 1: If value > 0 and <= 0.04 → **Green** (#00B050)
- Rule 2: If value > 0.04 and < 1 → **Red** (#FF0000)

**Why 0.04 and not 4:** Power BI's rules engine compares against the stored decimal value (0.1166), not the display percentage (11.66%). The threshold must match the stored scale.

**Result:** 11.66% displays in **red** — immediately signaling a problem without any user interaction.

**Dynamic behavior:** filter to Personal Loan (10.11% NPA) → still red. If a segment ever achieves below 4%, the card turns green automatically. The color is a function of data, not a manual format choice.

---

## 4. KPI Scorecard matrix

**Visual:** Matrix (single row, six columns)
**Title:** `KPI Scorecard — Actual vs Target`

**Values added (left to right):**
- `NPA %` — actual
- `NPA Target` — board-approved tolerance
- `NPA Status` — computed status with emoji
- `CASA Ratio` — actual
- `CASA Target` — industry benchmark
- `CASA Status` — computed status with emoji

**Conditional formatting on NPA Status column:**
- Background color → Fx → Rules:
  - If value contains "At Risk" → background Red, font White
  - If value contains "Within Target" → background Green, font White

**Result:**

| NPA % | NPA Target | NPA Status | CASA Ratio | CASA Target | CASA Status |
|---|---|---|---|---|---|
| 12% | 4.00% | ⚠️ At Risk (red) | 67.28% | 40.00% | ✅ Good (green) |

**Why this visual matters:** a board pack always ends with a summary scorecard. This gives a CXO or Risk Officer a one-glance view of whether key metrics are within tolerance — no need to interpret charts. The red/green status is unambiguous.

---

## 5. What-if parameter — Interest Rate Scenario

**What a what-if parameter is:** a Power BI feature that creates an interactive numeric slider connected to a measure. The user drags the slider and every visual using that measure updates in real time. It's scenario analysis without any code — built into the modeling layer.

**Created via:** Modeling → New parameter → Numeric range

**Settings used:**
- Name: `Interest Rate Scenario`
- Data type: Decimal number
- Minimum: 5 (5% floor interest rate)
- Maximum: 20 (20% ceiling)
- Increment: 0.5 (steps of 0.5%)
- Default: 10 (10% starting point)
- Add slicer to this page: ✅

**What Power BI auto-creates:**
- A table called `Interest Rate Scenario` with a single column of values
- A measure called `Interest Rate Scenario Value` that returns the slider's current value
- A slicer on the canvas showing the slider control

**The NIM Scenario measure** uses `Interest Rate Scenario Value` as an input — so dragging the slider from 5 to 15 shows the NIM jumping from ₹40M to ₹119M in real time.

**Demo script:** *"If the RBI raises interest rates, how does our NIM change? Drag from 8% to 12% — NIM goes from ₹63M to ₹95M. That's a ₹32M improvement in estimated interest income. This lets the CFO model rate scenarios without needing a separate Excel model."*

---

## 6. AI Insights & Scenarios page (Page 5)

### What was built and why

| Element | Status | Reason |
|---|---|---|
| What-if parameter + NIM card | ✅ Built | Scenario analysis — fully functional |
| Smart Narrative | ✅ Built | AI text summary — works reliably |
| Decomposition Tree | ✅ Built | Interactive drill-down — works reliably |
| Q&A Visual | ❌ Excluded | Retiring December 2026 — Microsoft deprecating in favour of Copilot |
| Key Influencers | ❌ Excluded | Does not work with DAX ratio measures; requires row-level columns |

**Professional decision:** excluding broken or retiring features from a portfolio is the right call. A clean page with four working elements is stronger than five elements where two don't render correctly.

### Smart Narrative

Auto-generated AI text summary of the dashboard data. Reads measure values and dimension context to write a plain-English paragraph. Can be enhanced by inserting live measure values inline using the + Value button — turning it into a dynamic executive summary that updates when filters change.

### Decomposition Tree

Interactive AI visual that starts with a total value (Total Loan Book = ₹4bn) and lets users drill into any dimension to understand the composition:
- Click + → choose ProductCategory → Loans: ₹3.2bn, Cards: ₹0.8bn etc.
- Click Loans → click + → choose Region → see which region drives the loan book
- Click the ⚡ lightning bolt → Power BI chooses the next dimension automatically (AI split)

The lightning bolt (AI split) is the genuinely AI-powered part — the algorithm identifies which remaining dimension explains the most variance and selects it automatically.

### Key Influencers — why it didn't work

The Key Influencers visual requires:
1. The "Analyze" field to be a **column** (categorical or numeric at row level), not a DAX measure
2. Sufficient row counts in each category (minimum ~30 rows per group)
3. The column to have low-to-medium cardinality

NPA% is a DAX ratio measure — it doesn't exist at the row level, only at the aggregated level. The visual can't decompose it. The correct approach would be to use `FactLoans[NPAFlag]` (a 0/1 column) as the Analyze field instead.

**Interview answer:** *"Key Influencers requires row-level columns, not DAX measures — it can't decompose a ratio measure like NPA%. The fix is to use the underlying flag column (NPAFlag) as the Analyze field, which the AI can properly evaluate at row level. I excluded it from the final dashboard because it didn't render correctly with my data structure, and a broken visual is worse than no visual."*

---

## Banking terminology reference

All acronyms introduced in this project:

| Acronym | Full Form | Definition |
|---|---|---|
| NIM | Net Interest Margin | Profitability of lending vs deposit cost |
| NPA | Non-Performing Asset | Loan unpaid for 90+ days |
| DPD | Days Past Due | How many days a payment is overdue |
| CASA | Current Account Savings Account | Low-cost deposit ratio |
| RLS | Row-Level Security | Data access restriction by user |
| RBI | Reserve Bank of India | India's central bank / regulator |
| KYC | Know Your Customer | Customer identity verification |
| EMI | Equated Monthly Instalment | Fixed monthly loan repayment |
| SME | Small and Medium Enterprise | Mid-size business segment |
| HNI | High Net Worth Individual | Wealthy individual segment |
| CXO | Chief X Officer | Generic C-suite executive |
| YTD | Year to Date | Jan 1 to current date |
| YoY | Year over Year | Growth vs same period last year |
| MoM | Month over Month | Growth vs prior month |
| LTV | Loan to Value | Loan amount vs collateral value |

---

# INTERVIEW QUESTIONS — Day 8
> Added: 2026-06-06
> Format: keyword trigger → spoken answer → senior signal

---

### Q: What is conditional formatting in Power BI and how did you use it?
**Keywords:** Fx button · rules-based · responds to filter context · measure-driven

**Spoken answer:**
"Conditional formatting applies visual properties — font color, background,
icons — based on data values rather than fixed settings. I applied it on the
NPA% card so the number turns red automatically when it exceeds the 4% board
tolerance and green when within target. It's set up via the Fx button on the
font color property, using rules that compare against the stored decimal value
(0.04, not 4, because percentages are stored as decimals internally). The
formatting responds to filter context — filter to a region and the card color
updates based on that region's NPA ratio automatically."

**Senior signal:** knowing the decimal vs percentage distinction in the rules
engine — most candidates set 4 instead of 0.04 and wonder why it never fires.

---

### Q: What is a what-if parameter and when would you use it?
**Keywords:** numeric slider · measure input · scenario analysis · real-time update

**Spoken answer:**
"A what-if parameter creates an interactive slider connected to a DAX measure.
The user drags the slider and every visual using that measure updates in real
time — no recalculation needed. I used it for an interest rate scenario: the
slider runs from 5% to 20%, and a NIM Scenario measure multiplies the deposit
base by the selected rate. Dragging from 8% to 12% shows NIM jumping from
₹63M to ₹95M instantly. It lets the CFO model rate scenarios without a
separate Excel model. What-if parameters are ideal for any 'what if X changes'
question where you want interactive exploration rather than a fixed calculation."

---

### Q: Why did you exclude Key Influencers from the final dashboard?
**Spoken answer:**
"Key Influencers requires a row-level column as the Analyze field — it can't
decompose a DAX ratio measure like NPA%. The fix would be to use the NPAFlag
column (0/1 at row level) instead, which the AI can evaluate properly. I
excluded it because it didn't render correctly with the ratio measure, and a
broken visual in a portfolio damages credibility more than a missing one. I
replaced the AI analysis capability with a Decomposition Tree, which works
reliably with aggregated measures and gives similar drill-down exploration."

---

### Q: What is a scorecard in Power BI and how did you build one?
**Keywords:** actual vs target · status measure · conditional formatting · matrix visual

**Spoken answer:**
"A scorecard presents KPIs alongside their targets and a computed status
indicator. I built it as a matrix visual with no row grouping — six columns
showing NPA% actual, NPA target, NPA status, CASA Ratio actual, CASA target,
CASA status. The status columns use IF measures that return emoji-labelled
strings (At Risk / Good) and are conditionally formatted with red/green
backgrounds. The result is a one-glance board-pack summary: red means
breached, green means within tolerance, no interpretation needed."

---

### Q: What is NIM and why does it matter in banking?
**Spoken answer:**
"NIM stands for Net Interest Margin — the difference between the interest
income a bank earns on loans and the interest it pays on deposits, expressed
as a percentage of earning assets. It's the core profitability metric for a
lending institution. A bank borrows cheaply via savings accounts (3-4%) and
lends expensively via home loans (8-14%); the spread is profit. Indian banks
typically target NIM of 3-4%. I built a what-if NIM estimator that lets
executives model how interest rate changes affect estimated income from the
deposit base — dragging from 8% to 12% shows a ₹32M NIM improvement."

---

### Q: What is a Decomposition Tree and what makes it AI-powered?
**Keywords:** interactive drill-down · AI split · explains variance · lightning bolt

**Spoken answer:**
"A Decomposition Tree starts with a total value and lets users interactively
break it down by any dimension — click + to choose ProductCategory, then +
again to choose Region, building a branching exploration path. The AI-powered
element is the lightning bolt (AI split) option: instead of the user choosing
the next dimension, the algorithm automatically selects whichever remaining
dimension explains the most variance in the current branch. On BankSphere 360,
I used it to decompose the ₹4bn loan book — users can explore which product,
region, and segment combinations drive the largest exposures, with the AI
highlighting the most explanatory splits."

---

### Q: How do you present actual vs target in Power BI?
**Spoken answer:**
"Several ways depending on context. For a formal scorecard I use a matrix
visual with actual, target, and a computed status measure side by side — clean
and board-pack appropriate. For a trend visual I use a KPI visual which shows
the current value, the target line, and the trend sparkline in one compact
element. For a card I use conditional formatting so the color signals
good/bad without needing a separate target column. The choice depends on the
audience — executives want the card color, analysts want the matrix detail."

---

## Complete measure library — end of Day 8

| # | Measure | Result | Format |
|---|---|---|---|
| 1 | Total Deposits | ₹794M | ₹ 0dp |
| 2 | Total Loan Book | ₹4bn | ₹ 0dp |
| 3 | Total Sanctioned | ₹8bn | ₹ 0dp |
| 4 | NPA Amount | ₹516M | ₹ 0dp |
| 5 | NPA % | 11.66% | % 2dp |
| 6 | Active Customers | 1,534 | # |
| 7 | CASA Ratio | 67.28% | % 2dp |
| 8 | Total Txn Value | ₹15bn | ₹ 0dp |
| 9 | Total Transactions | 60,000 | # |
| 10 | Deposits YTD | ₹794M | ₹ 0dp |
| 11 | TXN Value LY | ₹8bn | ₹ 0dp |
| 12 | Txn YoY% | 99.68% | % 2dp |
| 13 | Txn MoM % | 0.04% | % 2dp |
| 14 | Branch Rank | 1 | # |
| 15 | Fraud Flagged Value | ₹31M | ₹ 0dp |
| 16 | Outstanding by DPD | ₹4bn | ₹ 0dp |
| 17 | Sanctions by Month | varies | ₹ 0dp |
| 18 | NPA Target | 4.00% | % 2dp |
| 19 | CASA Target | 40.00% | % 2dp |
| 20 | NPA Status | text | none |
| 21 | CASA Status | text | none |
| 22 | NIM Scenario | varies | ₹ 0dp |
