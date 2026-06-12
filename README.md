# BankSphere 360 — Retail & SME Banking Analytics (Power BI)

A production-grade Power BI solution for a retail and SME bank, built as an end-to-end portfolio project demonstrating the full BI development lifecycle — from requirements gathering to deployment-ready governance.

---

## 📊 Project Overview

BankSphere 360 simulates a real banking analytics deployment for five personas — CXO, Branch Head, Risk Officer, Relationship Manager, and Compliance — covering deposits, loans, transactions, risk classification, and customer/channel analytics.

The project was built over a structured 10-day cycle, with each stage fully documented (see `/docs`).

---

## 🏗️ What's Inside

### Data Model

- **Star schema**: 4 dimension tables (DimDate, DimBranch, DimCustomer, DimProduct) + 3 fact tables (FactAccounts, FactLoans, FactTransactions)
- ~4,700 rows of synthetic but realistic banking data, including a deliberately seeded ~12% NPA ratio for risk storytelling
- Relationship audit — two incorrect auto-detected relationships identified and removed
- Marked date table (DimDate) enabling full time-intelligence support

### DAX

- **22 measures** — core KPIs, time intelligence (YTD, YoY, MoM), ranking (RANKX), scenario analysis
- Full measure descriptions for self-documentation
- Composable measure design (measures reference other measures — no duplicated logic)

### Reports — 5 Pages

| Page                    | Persona           | Highlights                                                                                                                       |
| ----------------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Executive Overview      | CXO / Branch Head | KPI cards, monthly trend + YoY dual-axis, top branches, region/segment/year slicers                                              |
| Loan & Risk             | Risk Officer      | NPA% with conditional formatting, RBI-style DPD risk classification chart, loan product matrix, KPI scorecard (actual vs target) |
| Customer & Channel      | RM / Compliance   | KYC status matrix, channel mix donut, segment analysis, fraud exposure                                                           |
| Branch Detail           | Drillthrough      | Auto-filtered branch-level deep dive                                                                                             |
| AI Insights & Scenarios | All               | What-if interest rate parameter (NIM estimator), Smart Narrative, Decomposition Tree                                             |

### Security

- **Dynamic Row-Level Security** — one role (`BranchHead`) serves all regional heads via a `UserRegionMap` table and `LOOKUPVALUE` + `USERPRINCIPALNAME()`
- Tested with View-as-roles — verified correct regional filtering across all pages

### Governance

- All measures documented with descriptions
- Internal/foreign-key columns hidden from report view
- Naming conventions enforced throughout
- PBIP format for Git-friendly version control (TMDL/JSON, diffable)

---

## 🏦 Banking Domain Coverage

| Term        | Meaning                                                                       |
| ----------- | ----------------------------------------------------------------------------- |
| NPA %       | Non-Performing Asset ratio — board tolerance 4%                               |
| CASA Ratio  | Current + Savings deposit share — industry benchmark 40%+                     |
| DPD Buckets | RBI-style classification: Current → Watch → Substandard → Doubtful → Loss/NPA |
| NIM         | Net Interest Margin — modeled via what-if scenario parameter                  |
| KYC Status  | Verified / Pending / Re-KYC Due — compliance tracking                         |

---

## 📁 Repository Structure

```
BankSphere360-PowerBI/
├── README.md
├── .gitignore
├── data/                          # Source CSVs (synthetic data)
├── docs/                          # Day-by-day documentation + formal docs
│   ├── Day3_PowerQuery_ETL.md
│   ├── Day4_Data_Modeling.md
│   ├── Day5_DAX_Foundations.md
│   ├── Day6_Advanced_DAX.md
│   ├── Day7_Visuals_UX_RLS.md
│   ├── Day8_Scorecards_Targets_AI.md
│   ├── Day9_Deployment_Production.md
│   ├── MockInterview_BankSphere360.md
│   ├── 01_Project_Summary.docx
│   ├── 02_SSRD.docx
│   ├── 03_LLD.docx
│   └── 04_UAT.docx
├── report/                        # Power BI Project (PBIP format)
│   ├── BankSphere360.pbip
│   ├── BankSphere360.Report/
│   └── BankSphere360.SemanticModel/
└── outputs/                       # Exports
    ├── BankSphere360_Journey.html
    └── BankSphere360_Dashboard.pdf
```

---

## 🛠️ Tech Stack

- **Power BI Desktop** (PBIP / TMDL format for version control)
- **DAX** — measures, time intelligence, RANKX, dynamic RLS
- **Power Query (M)** — ETL, conditional columns, query folding
- **Star Schema** dimensional modeling

---

## 📖 Documentation

Each development day is documented in `/docs` with two parts:

## 👤 Author

**Anshul Raghuvanshi**

#### [LinkedIn](https://www.linkedin.com/in/raghuvanshi-anshul/)

#### [GitHub](https://github.com/Anshul-RA1)
