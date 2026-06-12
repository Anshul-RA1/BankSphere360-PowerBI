# Day 9 — Deployment, Refresh, Gateways & Production Support

> Project: **BankSphere 360** — Power BI Banking Analytics
> Stage in lifecycle: requirements → source analysis → ETL → data model → measures → visuals → scorecards → **deploy + support (you are here)** → governance
> Goal of the day: understand everything that happens after you click Publish — publishing, workspaces, scheduled refresh, gateways, deployment pipelines, RLS member assignment, incremental refresh, and the production support runbook.

---

## 1. The big picture — what "deployment" means in Power BI

Development ends when the dashboard looks good in Desktop. **Deployment** is what makes it available to real users, keeps the data fresh automatically, and ensures it stays working in production. A developer who can only build but not deploy is only half-useful in a real project.

The deployment lifecycle:
```
Desktop (build) → Publish → Service (workspace) → Configure refresh
→ Assign RLS members → Deploy through pipeline → Production support
```

---

## 2. Publishing to the Power BI Service

### What publishing creates

When you click **Home → Publish** in Power BI Desktop, two artifacts are created in the Service:

**Semantic Model** (formerly called Dataset):
- Your data (VertiPaq in-memory compressed copy)
- All relationships
- All DAX measures
- RLS role definitions
- This is the engine — it answers all data queries

**Report:**
- Your five pages of visuals
- All formatting, bookmarks, drillthrough settings
- This is the interface — it connects to the semantic model

**Why they're separated:** multiple reports can connect to one semantic model. A Risk Officer's report and a CXO's report can share the same BankSphere 360 model without duplicating data or measures. One update to a measure in the semantic model propagates to all connected reports instantly.

### Publishing steps

1. **Sign in to Desktop:** top right → Sign in → organizational email → authenticate
2. **Click Publish:** Home ribbon → Publish → select destination workspace → Select
3. **Progress dialog:** "Preparing report" → "Sending to Power BI" → "Success!"
4. **In the Service:** My Workspace → two items appear: Report + Semantic Model
5. Click the Report → dashboard loads in browser at `https://app.powerbi.com`

### Interview answer

*"Publishing from Desktop creates two separate artifacts in the Service — the semantic model (data, relationships, measures, RLS) and the report (visual layer). I separate them deliberately so multiple reports can share one certified model. After publishing I configure scheduled refresh on the semantic model and assign RLS role members — those steps happen in the Service, not Desktop."*

---

## 3. Workspaces

### My Workspace vs App Workspace

**My Workspace:** personal, only you can see it. Use for development and testing only. Never use for production content others need to access.

**App Workspace:** shared, governed. You invite colleagues, assign roles, and it becomes the home for production content.

### Workspace roles

| Role | Permissions |
|---|---|
| Admin | Everything — manage members, delete workspace, publish, edit |
| Member | Publish, edit, share content |
| Contributor | Publish and edit, cannot share |
| Viewer | Read-only — see reports, cannot edit or publish |

**BankSphere 360 production assignment:**
- Developer (you) → Member or Admin
- Branch Heads → Viewer (see reports filtered by RLS)
- Risk Officers → Viewer
- Compliance → Viewer (read-only by design)

### Creating a workspace

Service → left sidebar → Workspaces → + New workspace → name: `BankSphere360-Prod` → Advanced settings → License mode: Pro → Save.

---

## 4. Scheduled Refresh

### Why it's needed

Import mode copies data into the model at publish time. Without scheduled refresh, the report shows the same stale snapshot forever. Scheduled refresh re-runs all Power Query steps and updates the snapshot automatically on a schedule.

### The gateway requirement — the most important concept

This is what most candidates miss in interviews.

**The rule:**
- **Cloud sources** (SharePoint, OneDrive, Azure SQL, REST API) → Service reaches them directly → **no gateway needed**
- **On-premises sources** (local CSV files, SQL Server on a laptop, SQL Server in a bank's data centre) → Service is in Microsoft's cloud and cannot reach your local network → **gateway required**

Our CSVs are local files → on-premises → gateway required for scheduled refresh.

### Two gateway types

**Standard mode (enterprise gateway):**
- Installed on a dedicated server that's always running
- Shared across multiple users and reports
- Supports all data sources
- Used in production environments
- Managed by IT/data engineering team

**Personal mode:**
- Installed on a personal machine
- Only works for the installing user
- Import mode only (not DirectQuery)
- Machine must be running and signed in for refresh to work
- Used for personal/dev scenarios only

**For BankSphere 360 portfolio:**
The simplest solution is to move CSVs to **OneDrive** (upload the 8 files) → change the source path in Power Query from local `C:\...` to the OneDrive URL → Power BI Service reaches OneDrive directly → no gateway needed → scheduled refresh works.

### Setting up scheduled refresh

1. Service → Workspace → three dots next to Semantic Model → **Settings**
2. **Data source credentials** → enter credentials for each source
3. **Scheduled refresh** section → toggle **On**
4. Set refresh times:
   - Daily at **06:00 AM** (before business hours — data fresh when staff log in)
   - Optional second refresh at 12:00 PM for midday updates
5. **Refresh failure notifications** → enter email → alerts on failure
6. Click **Apply**

### What happens on refresh

At 6 AM, Service wakes up → connects to source via gateway (if needed) → runs all Power Query steps → loads fresh data into VertiPaq → report updated. For our dataset: 2-5 minutes. For 500M row production tables: hours (→ use incremental refresh).

### Interview answer

*"After publishing I configure scheduled refresh on the semantic model in the Service. For on-premises sources like SQL Server, I install the on-premises data gateway in standard mode on a server that's always running, register it in the Service, and store credentials securely. For cloud sources like OneDrive or Azure SQL, no gateway is needed — the Service reaches them directly. I set daily 6 AM refresh so data is current when staff start, and configure failure notification emails. For large fact tables I implement incremental refresh to avoid refreshing the full table daily."*

---

## 5. Deployment Pipelines (Dev → Test → Prod)

### The problem without pipelines

Without pipelines: Edit in Desktop → Publish → overwrites production → if something breaks, the CXO sees a broken dashboard. Unacceptable in a bank.

### The three-stage pipeline

```
Dev workspace          Test workspace         Prod workspace
(developer builds) →  (UAT testing)     →   (live users)
BANK_SHPERE_360-Dev   BANK_SHPERE_360-Test  BANK_SHPERE_360-Prod
```

**Rules:**
- Developers always publish to **Dev** — never directly to Prod
- Test is for UAT (run the UAT document's 16 test cases here)
- Prod is what live users access — only promoted from Test after sign-off
- Promotion = one click, no re-uploading

### Creating a pipeline

Service → Deployment pipelines → + New pipeline → name: `BankSphere360` → Create → assign workspaces:
- Stage 1 (Dev) → `BankSphere360-Dev`
- Stage 2 (Test) → `BankSphere360-Test`
- Stage 3 (Prod) → `BankSphere360-Prod`

Promote: Dev stage → **Deploy to Test** → Power BI copies content → Test stage → **Deploy to Prod** after UAT passes.

### Parameter rules (source switching)

Different environments point to different data sources:
- Dev → local CSV files / dev database
- Test → test database
- Prod → production database

Configure parameter rules in the pipeline: when promoting Dev → Prod, automatically swap the connection string. Set once; pipeline handles it on every promotion. No manual editing, no risk of pointing Prod at the Dev database.

### Interview answer

*"I use deployment pipelines to move content through Dev, Test, and Prod workspaces. Developers always publish to Dev — never directly to Prod. After internal testing, content is promoted to Test for UAT against the test cases from the UAT document, then to Prod after sign-off. Promotion is a single click — no re-uploading. I configure parameter rules so the data source connection string automatically switches between environments on promotion."*

---

## 6. RLS Member Assignment in the Service

### Why it's done in the Service, not Desktop

You define RLS roles in Desktop (which you did on Day 7). But role membership — which email addresses belong to which role — is assigned in the Service after publishing. This separation means you can change who has access without republishing the report.

### Steps

1. Service → Workspace → three dots next to Semantic Model → **Security**
2. Select the `BranchHead` role
3. Add members:
   - `west.head@bank.com` → Add
   - `north.head@bank.com` → Add
   - `south.head@bank.com` → Add
   - `east.head@bank.com` → Add
4. Save

Now when `west.head@bank.com` opens the report, USERPRINCIPALNAME() returns their email, LOOKUPVALUE finds "West" in UserRegionMap, [Region] = "West" filter applies to DimBranch, and they see only West data.

### The critical gotcha — dataset owners bypass RLS

**Workspace admins and dataset owners always see all data, regardless of role membership.** This catches every developer: you test RLS, see all data, think it's broken — but it's working correctly. You're bypassing it because you're the owner.

**Always test RLS using:**
- View-as-roles in Desktop (with a test email)
- A non-owner Service account assigned to the role

### Interview answer

*"RLS role definitions are created in Desktop; member assignment happens in the Service after publishing under Semantic Model → Security. I assign the actual user emails there. The critical gotcha is that workspace admins and dataset owners bypass RLS by default — if you test as the owner you always see all data and incorrectly think RLS isn't working. I always test using View-as-roles in Desktop with a non-owner email, or a separate non-owner Service account."*

---

## 7. Incremental Refresh

### The problem

For our 60,000-row FactTransactions, full refresh takes minutes. For a real banking table with 500 million rows, full refresh takes hours — unacceptable for a daily 6 AM schedule.

### The solution — partition by date

Incremental refresh splits the fact table into date partitions. On each refresh, only the recent partitions (last 1-3 days of new/changed data) are refreshed. Historical partitions are untouched.

### Setup steps

**Step 1 — Create two Power Query parameters:**
- Name: `RangeStart`, Type: Date/Time, default: `1/1/2023`
- Name: `RangeEnd`, Type: Date/Time, default: `1/1/2025`

**Step 2 — Filter the fact table in Power Query:**
```
Table.SelectRows(Source, each [Date] >= RangeStart and [Date] < RangeEnd)
```
This filter **must fold to the source** (generate native SQL). If it doesn't fold, partition pruning fails and the full table loads anyway.

**Step 3 — Configure incremental refresh in Desktop:**
Right-click FactTransactions → Incremental refresh → configure:
- Archive data starting: **2 years** before refresh date
- Refresh data starting: **2 days** before refresh date
- Apply

**Step 4 — Publish and let the Service partition:**
After publishing, the Service creates date partitions automatically. Each subsequent refresh only touches the last 2 days.

### Interview answer

*"For large fact tables I implement incremental refresh using RangeStart and RangeEnd parameters in Power Query that partition the table by date. I configure the model to archive 2 years but refresh only the last 2 days — so instead of reloading 500 million rows daily, only new and changed data is processed. The critical requirement is that the date filter must fold to the source — without folding, Power BI loads the full table regardless of the partition configuration, defeating the entire purpose."*

---

## 8. Production Support Runbook

A runbook documents exactly what to do when things go wrong. A senior developer always has one. Juniors don't — and it shows at 2 AM when a refresh fails before month-end reporting.

### Incident 1 — Scheduled refresh failed

**Trigger:** failure notification email at 6 AM.

**Response:**
1. Service → Semantic Model → Settings → **Refresh history** → read exact error
2. **Gateway offline?** → check gateway machine is running → restart gateway service → manual refresh
3. **Credentials expired?** → Settings → Data source credentials → re-enter → manual refresh
4. **Source schema changed?** (column renamed/dropped) → open Desktop → fix Power Query step → republish → manual refresh
5. **Source unavailable?** (SQL Server down) → notify source team → manual refresh when restored
6. **Notify stakeholders:** "Refresh failed at 6 AM due to [reason]. Data shows [last refresh date]. Expected resolution: [time]."
7. Document in change log.

**SLA:** notify stakeholders within 30 minutes of detection. Resolve within 2 hours for non-month-end, immediately for month-end.

### Incident 2 — Wrong number reported

**Trigger:** stakeholder says dashboard number doesn't match their Excel.

**Response:**
1. **Don't immediately agree it's wrong** — first understand their calculation method.
2. Reproduce their exact scenario: apply same filters they used.
3. Check the measure definition: is DAX filter context correct?
4. Reconcile to source: export underlying data, compare to source extract.
5. **If dashboard is wrong:** hotfix in Dev → UAT → promote to Prod → communicate fix.
6. **If stakeholder is wrong:** show them the SSRD KPI definition signed off before development. That document is your protection — "NPA% was defined as NPA Outstanding / Total Loan Book Outstanding, signed off by [name] on [date]."
7. Never argue without the SSRD — always anchor to the agreed definition.

### Incident 3 — New KPI request

**Trigger:** stakeholder wants a metric that doesn't exist on the dashboard.

**Response:**
1. **Log as a change request** — not a bug, not urgent by default.
2. Get the definition in writing: exactly what should this measure calculate?
3. Check if existing measures + a slicer already answer the question.
4. Estimate effort: measure only (1 hour) vs new page (1 day) vs new data source (1 week).
5. Get timeline approval.
6. Build in Dev → UAT → Prod.
7. Update SSRD with new KPI definition.

### Incident 4 — Data security breach (wrong data visible)

**Trigger:** a branch head can see another region's data.

**Response:**
1. **Treat as Critical** — data security incident, not a standard bug.
2. Immediately check RLS role definition in Desktop: is the DAX filter correct?
3. Check UserRegionMap: is the user's email mapped to the correct region?
4. Check workspace roles: was the user accidentally added as Admin (admins bypass RLS)?
5. Fix, test with View-as-roles, republish.
6. **Notify manager immediately** — data security incidents must be reported even if resolved.
7. Document timeline and resolution in the incident log.

---

## 9. Version Control with PBIP

### Why PBIP for Git

The `.pbix` format is a binary file — Git can accept it but cannot diff it. Every commit stores a complete new copy. You can't see what changed between versions, can't merge, and the repository grows fast.

The `.pbip` format (Power BI Project) saves as a folder of **plain text files** (JSON, TMDL) that Git can diff line by line. You can see exactly which measure changed, which relationship was added, which visual was modified.

### Repository structure

```
banksphere360/
├── README.md
├── .gitignore
├── data/
│   ├── DimDate.csv
│   ├── DimBranch.csv
│   ├── DimCustomer.csv
│   ├── DimProduct.csv
│   ├── FactAccounts.csv
│   ├── FactLoans.csv
│   └── FactTransactions.csv
├── docs/
│   ├── Day3_PowerQuery_ETL.md
│   ├── Day4_Data_Modeling.md
│   ├── Day5_DAX_Foundations.md
│   ├── Day6_Advanced_DAX.md
│   ├── Day7_Visuals_UX_RLS.md
│   ├── Day8_Scorecards_Targets_AI.md
│   ├── Day9_Deployment.md
│   ├── 01_Project_Summary.docx
│   ├── 02_SSRD.docx
│   ├── 03_LLD.docx
│   └── 04_UAT.docx
├── report/
│   ├── BANK_SHPERE_360.pbip
│   ├── BANK_SHPERE_360.Report/
│   └── BANK_SHPERE_360.SemanticModel/
└── outputs/
    ├── BankSphere360_Journey.html
    └── screenshots/
```

### .gitignore for Power BI

```
# Power BI cache files
.pbi/
cache.abf
*.abf

# Local settings
*.pbix

# Data files if sensitive (keep CSVs since they're synthetic)
# *.csv
```

---

## 10. Governance — Dataset Endorsement

### Promoted vs Certified

After publishing, you can endorse a dataset to signal its quality to other report authors in the organisation.

**Promoted:** the dataset owner self-endorses. Signals "this is ready for use."

**Certified:** requires a Power BI admin to certify. Signals "this has been formally reviewed and is the organisation's official source of truth." Appears with a gold badge in the data hub.

**For BankSphere 360:** after publishing, go to Semantic Model → Settings → Endorsement → select **Promoted** (you can do this yourself). For Certified you'd need an admin.

### Why endorsement matters

In an organisation with hundreds of datasets, a report author searching for "NPA" might find 15 different datasets. Endorsed/Certified datasets appear prominently and signal "use this one, not the others." It's the governance mechanism that prevents the proliferation of conflicting data sources.

---

# INTERVIEW QUESTIONS — Day 9 (Deployment & Production Support)
> Added: 2026-06-06
> Format: keyword trigger → spoken answer → senior signal

---

### Q: Walk me through publishing a Power BI report to production.
**Keywords:** Desktop → Publish → semantic model + report separated · workspace · refresh · pipeline · RLS members

**Spoken answer:**
"I publish from Desktop to a Dev workspace first — never directly to Prod.
Publishing creates two artifacts: the semantic model (data, relationships,
measures, RLS definitions) and the report (visual layer), separated so
multiple reports can share one certified model. After publishing to Dev I
configure scheduled refresh on the semantic model, run the UAT test cases,
then promote through the deployment pipeline to Test and then Prod. RLS role
members are assigned in the Service under Semantic Model → Security. The
whole process means production is never touched until Dev and Test have
passed — a change request in banking has to go through that gate."

---

### Q: What is an on-premises data gateway and when do you need one?
**Keywords:** cloud vs on-prem · secure tunnel · standard vs personal mode · always-on server

**Spoken answer:**
"The gateway creates a secure tunnel between the Power BI Service (in
Microsoft's cloud) and data sources inside a private network or on a local
machine. You need it whenever your source is on-premises — a SQL Server in
a bank's data centre, local CSV files, an on-premises Oracle database. Cloud
sources like Azure SQL, SharePoint, or OneDrive don't need a gateway because
the Service can reach them directly. Standard mode is installed on a dedicated
server that's always running and can be shared across many users and reports.
Personal mode is for individual use on a laptop — the machine must be on
and signed in for refresh to work, so it's only suitable for development."

**Senior signal:** the "machine must be on and signed in" limitation of
personal mode — candidates who've actually used it know this trap.

---

### Q: What are deployment pipelines and why use them?
**Keywords:** Dev-Test-Prod · one-click promotion · no re-upload · parameter rules · never publish to Prod directly

**Spoken answer:**
"Deployment pipelines provide a governed path from development to production.
Developers publish to a Dev workspace; after internal testing, content is
promoted to a Test workspace for UAT; after sign-off, it's promoted to Prod.
Promotion is one click — no re-uploading the file, no manual copying, no
risk of accidentally overwriting something. I configure parameter rules so
data source connection strings switch automatically between environments on
promotion — Dev points at the test database, Prod points at production, and
the pipeline handles the swap. In a bank you never publish directly to Prod
because a mistake would break the dashboard a CXO is looking at."

---

### Q: How do you handle a scheduled refresh failure?
**Keywords:** refresh history · gateway · credentials · schema change · notify stakeholders · runbook

**Spoken answer:**
"First I check the refresh history in the Service to read the exact error
message. The four most common causes are: gateway offline (restart the
gateway service), expired credentials (re-enter in data source settings),
a schema change in the source (a column was renamed — fix the Power Query
step and republish), or the source being unavailable (SQL Server down —
notify the source team and wait). I always notify stakeholders proactively
within 30 minutes of a failure rather than waiting for them to notice stale
data. For month-end reporting I treat refresh failures as P1 incidents with
immediate escalation."

---

### Q: What is incremental refresh and when would you implement it?
**Keywords:** RangeStart/RangeEnd · partition by date · refresh only recent data · folding required

**Spoken answer:**
"Incremental refresh partitions a large fact table by date so only recent
data is refreshed on each run. I create RangeStart and RangeEnd parameters
in Power Query and use them to filter the fact table's date column. Power BI
then creates date partitions and on each refresh only processes the
configured recent window — say the last 2 days — rather than reloading the
full table. The critical requirement is that the date filter must fold to the
source. Without folding, Power BI still loads the full table to apply the
filter locally, defeating the purpose entirely. I implement incremental
refresh for any fact table over 10 million rows where full refresh would
exceed the available refresh window."

---

### Q: Why do dataset owners bypass RLS and how do you test RLS properly?
**Keywords:** owners bypass · View-as-roles · non-owner test account · Service test

**Spoken answer:**
"Workspace admins and dataset owners always see all data regardless of RLS
role membership — this is by design in Power BI. It catches every developer:
you open the report as the owner, see all the data, and think RLS is broken
when it's actually working correctly for everyone else. I test RLS two ways:
in Desktop using View-as-roles with a specific test email to simulate
USERPRINCIPALNAME(), and in the Service using a non-owner account that's
been assigned to the role. Both tests must pass before I sign off RLS as
working. This gotcha is also why you never add the developer account to a
viewer-only role — you'd inadvertently restrict your own development access."

---

### Q: What is the difference between a Promoted and Certified dataset?
**Keywords:** self-endorsed vs admin-certified · gold badge · source of truth · data hub discoverability

**Spoken answer:**
"Promoted is self-endorsed by the dataset owner — it signals 'this is ready
for use and I stand behind it.' Certified requires a Power BI admin to
formally review and approve — it signals 'this is the organisation's official
source of truth' and appears with a gold badge in the data hub. In a large
organisation with hundreds of datasets, certification is what stops report
authors from building on the wrong dataset. For BankSphere 360 I would
self-endorse as Promoted immediately after publishing, and work with the BI
governance team to get Certified once the formal review process is complete."

---

### Q: How do you version-control a Power BI project?
**Keywords:** PBIP not PBIX · text files · Git diff · .gitignore · TMDL

**Spoken answer:**
"I save in PBIP format (Power BI Project) rather than PBIX. PBIX is a binary
file — Git accepts it but can't diff it, so every commit is a full copy and
you can never see what changed between versions. PBIP saves the model as
plain text files (JSON and TMDL — Tabular Model Definition Language) that
Git can diff line by line. I can see exactly which measure changed, which
relationship was added, which visual was modified in a pull request. I add
a .gitignore for cache files (.abf, .pbi/) so only the source files are
tracked, not the computed artifacts."

---

## Quick reference — deployment checklist

Before going live with any Power BI report in production:

- [ ] Published to Dev workspace first (never directly to Prod)
- [ ] Scheduled refresh configured (daily 6 AM, failure notifications enabled)
- [ ] Gateway installed and registered (if on-premises sources)
- [ ] Data source credentials entered in the Service
- [ ] RLS tested with View-as-roles in Desktop
- [ ] RLS members assigned in the Service
- [ ] All 16 UAT test cases passed in Test workspace
- [ ] Deployment pipeline configured (Dev → Test → Prod)
- [ ] Dataset endorsed (Promoted minimum)
- [ ] Runbook documented
- [ ] Stakeholders notified of go-live
- [ ] First manual refresh triggered and verified successful
- [ ] PBIP committed to Git with meaningful commit message
