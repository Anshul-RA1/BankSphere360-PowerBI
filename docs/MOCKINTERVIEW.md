Q1. Walk me through the data model you built for BankSphere 360. What's the structure and why did you design it that way?

_"I built a star schema with four dimension tables — DimDate, DimBranch, DimCustomer, DimProduct — and three fact tables — FactAccounts, FactLoans, FactTransactions. All relationships are one-to-many from dimension to fact, single direction, because the dimension's key is unique while the fact table has many rows per key — for example, one customer in DimCustomer can have multiple accounts in FactAccounts.
The reason this matters isn't just convention — it's filter propagation. When a user selects 'West' in a region slicer on DimBranch, that filter flows through the relationship to FactAccounts and then to FactTransactions, so every visual connected to those tables updates automatically. That's the entire mechanism behind cross-filtering.
I also didn't accept the auto-detected relationships blindly. On audit I found two that were wrong — one linked DimCustomer to DimBranch by matching City names, which was coincidental, not a real relationship. Another was a 1-to-1 link between FactLoans and FactAccounts which created ambiguity. I removed both, which actually let a correct relationship — FactAccounts to DimCustomer — activate on its own."_

Q2. What is the difference between a measure and a calculated column, and how do you decide which to use?

Feature Measure Calculated Column
Definition A dynamic calculation evaluated when a report is queried. A new column calculated and stored in the data model.
Calculated When At query/runtime when a visual is rendered. During data refresh/load.
Storage Formula only is stored; results are not stored. Values are physically stored for every row.
Context Used Filter Context. Row Context.
Memory Usage Low memory consumption. Higher memory consumption.
Performance Generally more efficient for aggregations and KPIs. Can increase model size and affect performance.
Recalculation Recalculates dynamically based on filters, slicers, and user selections. Recalculates only when the dataset is refreshed.
Can Be Used in Slicers? ❌ No ✅ Yes
Can Be Used in Relationships? ❌ No ✅ Yes
Can Be Used as Axis/Legend? ❌ No ✅ Yes
Best Use Cases KPIs, aggregations, percentages, YTD, MTD, dynamic calculations. Categorization, flags, age groups, customer segments, calculated attributes.
Example DAX Total Sales = SUM(Sales[Amount]) Profit = Sales[SalesAmount] - Sales[Cost]
Banking Example Total Deposits = SUM(FactAccounts[Balance]) Loan Category = IF(FactLoans[OutstandingAmount] > 1000000,"Large Loan","Retail Loan")

Q3. Explain CALCULATE — what does it do, and give me an example from your project where you used it and one where you deliberately did NOT use it.

_"CALCULATE is the only function in DAX that can modify filter context — it takes an expression and one or more filter arguments, and evaluates that expression under the new, narrower context.
In my project, NPA Amount uses it: CALCULATE(SUM(FactLoans[OutstandingAmount]), FactLoans[NPAFlag]=1). Without CALCULATE, SUM would add up all loan outstanding amounts; the filter argument restricts it to only loans flagged as non-performing.
But I didn't use CALCULATE everywhere. Total Loan Book is just SUM(FactLoans[OutstandingAmount]) — no CALCULATE — because FactLoans only contains loans, so there's nothing to filter out. Reaching for CALCULATE when a plain aggregation already does the job is a habit I avoid — it adds unnecessary complexity and can hide what the measure is actually doing."_
