# Auto Loan Securitisation Risk Analytics
## Power BI / Analysis Services Data Model Design + DAX Measures Library
### RBI (SMA/NPA) and IFRS 9 (ECL) Dual Framework

---

## 0. Grounding: what's actually in your data

This design is built directly against your four files — not a generic template.

| File | Grain | Rows | Key fields |
|---|---|---|---|
| `auto_loan_securitisation_data.csv` | 1 row per loan (as of cut-off) | 500 | `LoanID`, `PoolID`, `IFRS9_Stage` (1/2/3), `PD_Estimate`, `LGD_Estimate`, `EAD`, `ECL_Provision`, `CIBIL_Score_*`, `LTV_*`, `DTI_Ratio` |
| `dpd_snapshot_history.csv` | 1 row per loan per month | 6,000 | `SnapshotDate`, `DPD_Bucket`, `RBI_SMA_Class` (Standard/SMA-0/SMA-1/SMA-2/NPA), `RollFlag` (Same/Worse/Better), `TransitionType` (Static/Roll/Cure), `WriteOffFlag` |
| `dynamic_loss_monthly.csv` | 1 row per pool per reporting month | 12 | `NewDefaults_*`, `GrossLoss`, `NetLoss`, `SMM`, `CPR_Annualised`, `CollectionEfficiency`, `ExcessSpread_Monthly` |
| `static_pool_vintage_data.csv` | 1 row per vintage per month-on-book | 375 | `VintageID` (e.g. `2021-Q1`), `MonthsOnBook`, `CumulativeNetLossRate`, `MarginalLossRate`, `PoolFactor` |

Single pool today (`ZAAUTO2024-1`), 3 servicers, 5 regions — the model below is built to scale to multiple pools/vintages without redesign, since that's the normal evolution of a securitisation shelf.

**Important framing note:** your loan table already carries an `IFRS9_Stage`/`PD`/`LGD`/`EAD`/`ECL_Provision` column — that's the *accounting* (Ind AS 109 / IFRS 9) view. `dpd_snapshot_history` carries `RBI_SMA_Class` — that's the *regulatory* (RBI IRACP / Master Direction) view. These two frameworks classify the **same loan** differently and on different triggers (DPD-only for RBI vs SICR/forward-looking for IFRS9), which is exactly why a side-by-side reconciliation view has genuine analytical value — it's the classic "provisioning gap" investors and auditors ask about.

---

## 1. Star Schema / Tabular Model Design

### 1.1 Table roles

```
FACT TABLES (Import mode; DPD history + dynamic loss can be Incremental Refresh in production)
├─ FactLoanSnapshot        ← auto_loan_securitisation_data.csv (latest cut-off, 1 row/loan)
├─ FactDPDHistory          ← dpd_snapshot_history.csv (1 row/loan/month) — DELINQUENCY MIGRATION ENGINE
├─ FactDynamicLossMonthly  ← dynamic_loss_monthly.csv (1 row/pool/month) — POOL-LEVEL PERFORMANCE
└─ FactStaticPoolVintage   ← static_pool_vintage_data.csv (1 row/vintage/MoB) — COHORT CURVES

DIMENSION TABLES
├─ DimDate                 (standard date table, marked as Date Table, one per each date role via USERELATIONSHIP)
├─ DimLoan                 (LoanID, VehicleMake/Model/Type, EmploymentType, LoanPurpose — degenerate attrs split off FactLoanSnapshot)
├─ DimPool                 (PoolID, ServicerID → ServicerName, OriginationChannel)
├─ DimGeography            (Region, State)
├─ DimServicer             (ServicerID, ServicerName)
├─ DimRBI_SMA              (RBI_SMA_Class, SortOrder, DPD_Lower, DPD_Upper, ProvisioningRate_RBI)  -- bridge/reference
├─ DimIFRS9_Stage          (Stage 1/2/3, StageLabel, SICR_Trigger_Desc)                              -- bridge/reference
├─ DimVintage              (VintageID, VintageStartDate, Quarter, Year)
└─ CalcGroup: Reporting Framework   ("RBI (IRACP)" vs "IFRS 9 (Ind AS 109)")  -- see §3
```

### 1.2 Relationships (cardinality, direction)

```
DimDate[Date]  1 ──── * FactDPDHistory[SnapshotDate]        (active)
DimDate[Date]  1 ──── * FactDynamicLossMonthly[ReportingDate] (inactive → USERELATIONSHIP)
DimDate[Date]  1 ──── * FactLoanSnapshot[CutoffDate]          (inactive)
DimDate[Date]  1 ──── * FactLoanSnapshot[OriginationDate]     (inactive, role-playing "Vintage Date")
DimDate[Date]  1 ──── * FactLoanSnapshot[MaturityDate]        (inactive)
DimLoan[LoanID] 1 ──── * FactDPDHistory[LoanID]               (active, single direction)
DimLoan[LoanID] 1 ──── 1 FactLoanSnapshot[LoanID]             (active — snapshot is loan grain)
DimPool[PoolID] 1 ──── * FactLoanSnapshot[PoolID]
DimPool[PoolID] 1 ──── * FactDynamicLossMonthly[PoolID]  (note: your dynamic file has no PoolID column today —
                                                            add one before load, or hard-tag "ZAAUTO2024-1")
DimVintage[VintageID] 1 ──── * FactStaticPoolVintage[VintageID]
DimGeography[Region] 1 ──── * FactLoanSnapshot[Region]  (via bridging on DimLoan or direct if flattened)
```

**Why single-direction, mostly-inactive on DimDate:** a securitisation model has 4+ legitimate date roles per loan (origination/vintage, cut-off/snapshot, maturity, last payment). Power BI/SSAS Tabular only allows one *active* relationship to a shared date table; every time-intelligence measure that needs a different date role must explicitly call `USERELATIONSHIP()`. This is the single most common DAX bug in securitisation models — flag it now so it doesn't bite later.

### 1.3 Mermaid ERD (paste into any Mermaid renderer)

```mermaid
erDiagram
    DimDate ||--o{ FactDPDHistory : "SnapshotDate (active)"
    DimDate ||--o{ FactDynamicLossMonthly : "ReportingDate (inactive)"
    DimDate ||--o{ FactLoanSnapshot : "CutoffDate/OriginationDate/MaturityDate (inactive)"
    DimLoan ||--|| FactLoanSnapshot : "LoanID"
    DimLoan ||--o{ FactDPDHistory : "LoanID"
    DimPool ||--o{ FactLoanSnapshot : "PoolID"
    DimPool ||--o{ FactDynamicLossMonthly : "PoolID"
    DimVintage ||--o{ FactStaticPoolVintage : "VintageID"
    DimServicer ||--o{ DimPool : "ServicerID"
    DimRBI_SMA ||--o{ FactDPDHistory : "RBI_SMA_Class (bridge)"
    DimIFRS9_Stage ||--o{ FactLoanSnapshot : "IFRS9_Stage (bridge)"
```

### 1.4 Reference/bridge tables to build (disconnected, used only for labelling & sort order)

```DAX
DimRBI_SMA =
DATATABLE(
    "RBI_SMA_Class", STRING, "SortOrder", INTEGER, "DPD_Lower", INTEGER, "DPD_Upper", INTEGER, "IRACP_Provision_Pct", DOUBLE,
    {
        {"Standard", 1, 0, 0, 0.0025},   -- 0.25% standard asset provisioning (indicative, confirm current RBI rate)
        {"SMA-0",    2, 1, 30, 0.0025},
        {"SMA-1",    3, 31, 60, 0.0025},
        {"SMA-2",    4, 61, 90, 0.0025},
        {"NPA",      5, 91, 999, 0.15}   -- substandard indicative; real NPA provisioning is tiered (sub-standard/doubtful/loss)
    }
)

DimIFRS9_Stage =
DATATABLE(
    "IFRS9_Stage", INTEGER, "StageLabel", STRING, "SortOrder", INTEGER, "ECL_Horizon", STRING, "Trigger", STRING,
    {
        {1, "Stage 1 – Performing", 1, "12-month ECL", "No SICR since origination"},
        {2, "Stage 2 – Underperforming", 2, "Lifetime ECL", "Significant Increase in Credit Risk (SICR)"},
        {3, "Stage 3 – Credit-impaired", 3, "Lifetime ECL", "Objective evidence of default/impairment"}
    }
)
```

---

## 2. Naming & organisation convention used below

- Prefix `_` = base/helper measure not meant for direct visual use.
- Measures grouped into **Display Folders**: `Portfolio`, `Delinquency (RBI)`, `IFRS9 ECL`, `RBI vs IFRS9 Reconciliation`, `Vintage & Cohort`, `Dynamic Pool Performance`, `Waterfall & Structure`, `Stress Testing`, `Time Intelligence`.
- All monetary measures assume `CurrentBalance`, `EAD`, `ECL_Provision` etc. are in INR as loaded; add `FORMAT(..., "₹#,##0")` at the visual level, not baked into the measure, so the value stays a true number for further calculation.

---

## 3. Calculation Groups (Power BI Premium / PPU / SSAS 2019+, or Tabular Editor)

Two calculation groups make this a *single* model that serves both regulatory and accounting reporting rather than duplicating every measure.

### 3.1 Calculation Group: `Reporting Framework`

```DAX
-- Calculation Item 1: "RBI (IRACP)"
SELECTEDMEASURE()  -- pass-through; RBI measures are already DPD/SMA-driven natively

-- Calculation Item 2: "IFRS 9 (Ind AS 109)"
SELECTEDMEASURE()  -- pass-through; IFRS9 measures are already Stage/PD/LGD-driven natively
```
> In practice this calc group is most useful when paired with a **field parameter** (see §9.1) that swaps which classification column measures group by — the calc item mainly exists to give a clean slicer label "Framework: RBI | IFRS9" for the report page.

### 3.2 Calculation Group: `Scenario` (used by Stress Testing, §8)

```DAX
-- "Base Case"
SELECTEDMEASURE()

-- "Mild Stress"
SELECTEDMEASURE() * 1.5

-- "Severe Stress (Crisis)"
SELECTEDMEASURE() * 3.0

-- "Custom (What-If)"
SELECTEDMEASURE() * SELECTEDVALUE('WhatIf Stress Multiplier'[Value], 1)
```

---

## 4. Portfolio & Exposure Measures

```DAX
Total Loan Count =
DISTINCTCOUNT(FactLoanSnapshot[LoanID])

Total Original Pool Balance =
SUM(FactLoanSnapshot[OriginalLoanAmount])

Total Current Balance (EAD basis) =
SUM(FactLoanSnapshot[CurrentBalance])

Pool Factor =
DIVIDE([Total Current Balance (EAD basis)], [Total Original Pool Balance])

Weighted Avg Interest Rate =
DIVIDE(
    SUMX(FactLoanSnapshot, FactLoanSnapshot[InterestRate] * FactLoanSnapshot[CurrentBalance]),
    [Total Current Balance (EAD basis)]
)

Weighted Avg LTV (Current) =
DIVIDE(
    SUMX(FactLoanSnapshot, FactLoanSnapshot[LTV_Current] * FactLoanSnapshot[CurrentBalance]),
    [Total Current Balance (EAD basis)]
)

Weighted Avg CIBIL (Current) =
DIVIDE(
    SUMX(FactLoanSnapshot, FactLoanSnapshot[CIBIL_Score_Current] * FactLoanSnapshot[CurrentBalance]),
    [Total Current Balance (EAD basis)]
)

Weighted Avg Remaining Term (Months) =
DIVIDE(
    SUMX(FactLoanSnapshot, FactLoanSnapshot[RemainingTerm] * FactLoanSnapshot[CurrentBalance]),
    [Total Current Balance (EAD basis)]
)

CIBIL Migration (Orig vs Current) =
[Weighted Avg CIBIL (Current)] -
    DIVIDE(
        SUMX(FactLoanSnapshot, FactLoanSnapshot[CIBIL_Score_Origination] * FactLoanSnapshot[CurrentBalance]),
        [Total Current Balance (EAD basis)]
    )
-- Negative = book-wide credit score deterioration since origination; a leading indicator worth its own KPI card.

Loan Count % of Original =
DIVIDE([Total Loan Count], CALCULATE([Total Loan Count], ALL(FactLoanSnapshot)))

Concentration – Top Servicer Share =
VAR TopServicerBal =
    MAXX(
        VALUES(FactLoanSnapshot[ServicerName]),
        CALCULATE([Total Current Balance (EAD basis)])
    )
RETURN DIVIDE(TopServicerBal, [Total Current Balance (EAD basis)])

HHI – Servicer Concentration =
-- Herfindahl-Hirschman Index on servicer balance share, standard concentration-risk stat for investor decks
SUMX(
    VALUES(FactLoanSnapshot[ServicerName]),
    DIVIDE(CALCULATE([Total Current Balance (EAD basis)]), [Total Current Balance (EAD basis)]) ^ 2
)
```

---

## 5. Delinquency & RBI SMA/NPA Measures

Built on `FactDPDHistory`, which is the only table with monthly granularity + `RollFlag`/`TransitionType` — this is your **roll-rate engine**.

```DAX
_Latest Snapshot Date =
CALCULATE(MAX(FactDPDHistory[SnapshotDate]), ALL(FactDPDHistory[SnapshotDate]))

Current Balance (Latest DPD Snapshot) =
CALCULATE(
    SUM(FactDPDHistory[CurrentBalance]),
    FactDPDHistory[SnapshotDate] = [_Latest Snapshot Date]
)

DPD 30+ Balance % =
VAR NonCurrentBal =
    CALCULATE(
        SUM(FactDPDHistory[CurrentBalance]),
        FactDPDHistory[SnapshotDate] = [_Latest Snapshot Date],
        FactDPDHistory[RBI_SMA_Class] <> "Standard"
    )
RETURN DIVIDE(NonCurrentBal, [Current Balance (Latest DPD Snapshot)])

DPD 90+ / NPA Balance % =
VAR NPABal =
    CALCULATE(
        SUM(FactDPDHistory[CurrentBalance]),
        FactDPDHistory[SnapshotDate] = [_Latest Snapshot Date],
        FactDPDHistory[RBI_SMA_Class] = "NPA"
    )
RETURN DIVIDE(NPABal, [Current Balance (Latest DPD Snapshot)])

Gross NPA Ratio (RBI) =
[DPD 90+ / NPA Balance %]     -- alias for investor-facing terminology

SMA-2 Early Warning Balance % =
VAR SMA2Bal =
    CALCULATE(
        SUM(FactDPDHistory[CurrentBalance]),
        FactDPDHistory[SnapshotDate] = [_Latest Snapshot Date],
        FactDPDHistory[RBI_SMA_Class] = "SMA-2"
    )
RETURN DIVIDE(SMA2Bal, [Current Balance (Latest DPD Snapshot)])
-- SMA-2 (61-90 DPD) is the last stop before NPA — a rising SMA-2 % is the single best forward indicator for RBI reporting.

_Loan Count in Bucket, Prior Month =
CALCULATE(
    DISTINCTCOUNT(FactDPDHistory[LoanID]),
    DATEADD(FactDPDHistory[SnapshotDate], -1, MONTH)
)

Roll Rate 30→60 =
VAR PriorMonth = EDATE([_Latest Snapshot Date], -1)
VAR Base =
    CALCULATE(
        DISTINCTCOUNT(FactDPDHistory[LoanID]),
        FactDPDHistory[SnapshotDate] = PriorMonth,
        FactDPDHistory[DPD_Bucket] = "30-59 DPD"
    )
VAR RolledForward =
    CALCULATE(
        DISTINCTCOUNT(FactDPDHistory[LoanID]),
        FactDPDHistory[SnapshotDate] = [_Latest Snapshot Date],
        FactDPDHistory[DPD_Bucket] = "60-89 DPD",
        FactDPDHistory[TransitionType] = "Roll"
    )
RETURN DIVIDE(RolledForward, Base)
-- Duplicate this pattern for 60→90, 90→120, 1-29→30-59 by changing the two bucket literals — 
-- consider a parameter table (§9) instead of 6 near-identical measures if you need all transitions on one matrix visual.

Cure Rate (any delinquent bucket) =
VAR PriorMonth = EDATE([_Latest Snapshot Date], -1)
VAR DelinquentPrior =
    CALCULATE(
        DISTINCTCOUNT(FactDPDHistory[LoanID]),
        FactDPDHistory[SnapshotDate] = PriorMonth,
        FactDPDHistory[DPD_Bucket] <> "Current"
    )
VAR CuredThisMonth =
    CALCULATE(
        DISTINCTCOUNT(FactDPDHistory[LoanID]),
        FactDPDHistory[SnapshotDate] = [_Latest Snapshot Date],
        FactDPDHistory[TransitionType] = "Cure"
    )
RETURN DIVIDE(CuredThisMonth, DelinquentPrior)

Re-Default Rate (Modified Loans) =
VAR ModifiedLoans =
    CALCULATETABLE(VALUES(FactLoanSnapshot[LoanID]), FactLoanSnapshot[IsModified] = TRUE)
VAR ModifiedAndDefaulted =
    CALCULATE(
        DISTINCTCOUNT(FactLoanSnapshot[LoanID]),
        FactLoanSnapshot[IsModified] = TRUE,
        FactLoanSnapshot[IsDefaulted] = TRUE
    )
RETURN DIVIDE(ModifiedAndDefaulted, COUNTROWS(ModifiedLoans))
-- Re-default on Rate Reduction / Payment Deferral / Term Extension modifications is an RBI regulatory
-- disclosure item (restructured book asset quality) as well as an IFRS9 SICR trigger.

WriteOff Balance (Cumulative) =
CALCULATE(
    SUM(FactDPDHistory[CurrentBalance]),
    FactDPDHistory[WriteOffFlag] = TRUE
)

Repossession Rate =
DIVIDE(
    CALCULATE(DISTINCTCOUNT(FactDPDHistory[LoanID]), FactDPDHistory[RepossessionFlag] = TRUE, FactDPDHistory[SnapshotDate] = [_Latest Snapshot Date]),
    [Total Loan Count]
)

Consecutive Delinquency – Avg Months (Delinquent loans only) =
CALCULATE(
    AVERAGE(FactDPDHistory[ConsecutiveMonthsDelinquent]),
    FactDPDHistory[SnapshotDate] = [_Latest Snapshot Date],
    FactDPDHistory[DPD_Bucket] <> "Current"
)
```

---

## 6. IFRS 9 / Ind AS 109 ECL Measures

Built on `FactLoanSnapshot`, which already carries model outputs (`PD_Estimate`, `LGD_Estimate`, `EAD`, `ECL_Provision`) — these measures aggregate and validate that output; §8 shows how to *re-derive* ECL under stress instead of just reading it.

```DAX
Total EAD =
SUM(FactLoanSnapshot[EAD])

Total ECL Provision =
SUM(FactLoanSnapshot[ECL_Provision])

ECL Coverage Ratio =
DIVIDE([Total ECL Provision], [Total EAD])

Weighted Avg PD =
DIVIDE(
    SUMX(FactLoanSnapshot, FactLoanSnapshot[PD_Estimate] * FactLoanSnapshot[EAD]),
    [Total EAD]
)

Weighted Avg LGD =
DIVIDE(
    SUMX(FactLoanSnapshot, FactLoanSnapshot[LGD_Estimate] * FactLoanSnapshot[EAD]),
    [Total EAD]
)

EAD by Stage % =
DIVIDE(SUM(FactLoanSnapshot[EAD]), CALCULATE([Total EAD], ALL(FactLoanSnapshot[IFRS9_Stage])))
-- Place on a matrix with IFRS9_Stage on rows for the classic "Stage 1/2/3 exposure mix" investor chart.

Stage 2 & 3 Exposure % (Underperforming + Impaired) =
VAR S23EAD =
    CALCULATE([Total EAD], FactLoanSnapshot[IFRS9_Stage] IN {2, 3})
RETURN DIVIDE(S23EAD, [Total EAD])

Stage 3 Coverage Ratio =
VAR S3ECL = CALCULATE([Total ECL Provision], FactLoanSnapshot[IFRS9_Stage] = 3)
VAR S3EAD = CALCULATE([Total EAD], FactLoanSnapshot[IFRS9_Stage] = 3)
RETURN DIVIDE(S3ECL, S3EAD)

_ECL Recompute (Independent Cross-Check) =
-- Re-derives ECL from PD x LGD x EAD independently of the stored ECL_Provision column —
-- run this next to "Total ECL Provision" as a control/reconciliation check; a material gap
-- signals the stored provision uses a different (e.g. lifetime vs 12m, or discounted) basis.
SUMX(
    FactLoanSnapshot,
    FactLoanSnapshot[PD_Estimate] * FactLoanSnapshot[LGD_Estimate] * FactLoanSnapshot[EAD]
)

ECL Model Variance (Stored vs Recomputed) =
[Total ECL Provision] - [_ECL Recompute (Independent Cross-Check)]

Stage Migration Count (Loan-level, period over period) =
-- Requires a second FactLoanSnapshot-like table at a prior cut-off (or store IFRS9_Stage
-- history inside FactDPDHistory going forward — currently only RBI_SMA_Class is tracked monthly,
-- IFRS9_Stage only exists at the single latest cut-off in your data). Flagging this as a
-- DATA GAP: to build a genuine Stage 1→2→3 migration matrix you need monthly IFRS9_Stage
-- history, mirroring what you already do for RBI_SMA_Class in FactDPDHistory.
BLANK()

Coverage Gap vs RBI Provisioning (see §7) =
[Total ECL Provision] - [_RBI Provisioning Estimate]
```

> **Modelling note on the Stage Migration gap above:** this is the one real structural asymmetry in your four files — RBI classification is tracked monthly (good, supports roll-rate/vintage analysis) but IFRS9 staging is only a point-in-time snapshot. If the investor report needs a Stage migration matrix (a standard IFRS9 disclosure), the fix is upstream: extend `dpd_snapshot_history` (or a new `FactIFRS9StageHistory`) to carry `IFRS9_Stage` per loan per month, mirroring `RBI_SMA_Class`. I've left the measure stubbed rather than faking history from a single snapshot.

---

## 7. RBI vs IFRS9 Reconciliation (the side-by-side view you asked for)

This is the section that answers "how much more/less would we provision under RBI norms vs IFRS9" — the standard question from both regulators and rating agencies on Indian ABS/MBS shelves.

```DAX
_RBI Provisioning Estimate =
-- Applies indicative IRACP rates from DimRBI_SMA to latest-snapshot balances by SMA class.
-- Confirm current RBI rates (Standard 0.25-0.40% by asset class, NPA sub-standard/doubtful tiers)
-- before using in an actual filing — the DimRBI_SMA table rates are placeholders to be updated.
SUMX(
    VALUES(FactDPDHistory[RBI_SMA_Class]),
    VAR ClassBal =
        CALCULATE(
            SUM(FactDPDHistory[CurrentBalance]),
            FactDPDHistory[SnapshotDate] = [_Latest Snapshot Date]
        )
    VAR ClassRate =
        LOOKUPVALUE(DimRBI_SMA[IRACP_Provision_Pct], DimRBI_SMA[RBI_SMA_Class], FactDPDHistory[RBI_SMA_Class])
    RETURN ClassBal * ClassRate
)

Provisioning Gap (IFRS9 − RBI) =
[Total ECL Provision] - [_RBI Provisioning Estimate]
-- Positive = IFRS9 is more conservative than regulatory minimum (typical, since IFRS9 is forward-looking
-- and SICR-based rather than pure DPD-based) — useful "extra cushion" disclosure for investors.

Provisioning Gap % of EAD =
DIVIDE([Provisioning Gap (IFRS9 − RBI)], [Total EAD])

Classification Mismatch Count =
-- Loans where RBI says "Standard/SMA" but IFRS9 already flags Stage 2/3 (forward-looking SICR
-- catching risk before DPD does) — or the reverse (rare, would indicate a modelling issue).
VAR JoinedTable =
    NATURALINNERJOIN(
        SELECTCOLUMNS(FactLoanSnapshot, "LoanID", FactLoanSnapshot[LoanID], "IFRS9_Stage", FactLoanSnapshot[IFRS9_Stage]),
        CALCULATETABLE(
            SELECTCOLUMNS(FactDPDHistory, "LoanID", FactDPDHistory[LoanID], "RBI_SMA_Class", FactDPDHistory[RBI_SMA_Class]),
            FactDPDHistory[SnapshotDate] = [_Latest Snapshot Date]
        )
    )
RETURN
COUNTROWS(
    FILTER(
        JoinedTable,
        ([IFRS9_Stage] >= 2 && [RBI_SMA_Class] = "Standard") ||
        ([IFRS9_Stage] = 1 && [RBI_SMA_Class] = "NPA")
    )
)

Framework Divergence Rate % =
DIVIDE([Classification Mismatch Count], [Total Loan Count])
```

**Recommended visual:** a 3×5 matrix (IFRS9 Stage 1/2/3 as rows × RBI Standard/SMA-0/1/2/NPA as columns) with `[Total Current Balance]` in the cells — this single matrix is usually the centrepiece of an investor-facing RBI/IFRS9 reconciliation page.

---

## 8. Waterfall & Structural Measures

Your files describe the **collateral pool**, not the **note/tranche structure** — there's no tranche-level data (Class A/B/Equity, subordination %, coupon) in any of the four CSVs. To build genuine waterfall DAX I need either (a) a tranche/note structure table added to the model, or (b) explicit assumptions. Below assumes a simple illustrative structure so the DAX pattern is usable immediately; swap in your real tranche table when available.

```DAX
-- Assumed reference table (replace with real deal structure):
DimTranche =
DATATABLE(
    "TrancheID", STRING, "TrancheName", STRING, "SeniorityRank", INTEGER,
    "OriginalFaceValue", DOUBLE, "CouponRate", DOUBLE,
    {
        {"A", "Class A (Senior)", 1, 450000000, 0.085},
        {"B", "Class B (Mezzanine)", 2, 60000000, 0.11},
        {"E", "Equity / First Loss", 3, 32637916, 0.0}
    }
)

Available Collections for Waterfall =
CALCULATE(
    SUM(FactDynamicLossMonthly[CollectionsTotal]),
    FactDynamicLossMonthly[ReportingDate] = MAX(FactDynamicLossMonthly[ReportingDate])
)

Senior Interest Due (Class A) =
VAR ABal = LOOKUPVALUE(DimTranche[OriginalFaceValue], DimTranche[TrancheID], "A")
VAR ARate = LOOKUPVALUE(DimTranche[CouponRate], DimTranche[TrancheID], "A")
RETURN ABal * ARate / 12

_Waterfall Step 1 – Senior Fees & Expenses =
-- Servicing fee assumption: 1% p.a. of EOP balance (confirm actual servicing fee in the deal docs)
CALCULATE(SUM(FactDynamicLossMonthly[EOP_Balance]), FactDynamicLossMonthly[ReportingDate] = MAX(FactDynamicLossMonthly[ReportingDate])) * 0.01 / 12

_Waterfall Step 2 – Remaining After Senior Fees =
[Available Collections for Waterfall] - [_Waterfall Step 1 – Senior Fees & Expenses]

_Waterfall Step 3 – Remaining After Class A Interest =
[_Waterfall Step 2 – Remaining After Senior Fees] - [Senior Interest Due (Class A)]

_Waterfall Step 4 – Remaining After Class A Principal =
VAR SchedPrin = CALCULATE(SUM(FactDynamicLossMonthly[ScheduledAmort]), FactDynamicLossMonthly[ReportingDate] = MAX(FactDynamicLossMonthly[ReportingDate]))
RETURN [_Waterfall Step 3 – Remaining After Class A Interest] - SchedPrin

Excess Spread to Equity =
MAX([_Waterfall Step 4 – Remaining After Class A Principal], 0)

Overcollateralization (OC) % =
VAR TotalNoteFace = SUMX(DimTranche, DimTranche[OriginalFaceValue])
RETURN DIVIDE([Total Current Balance (EAD basis)] - TotalNoteFace, [Total Current Balance (EAD basis)])

OC Trigger Breach Flag =
-- Typical structural trigger: breach if OC falls below a floor (e.g. 6%)
VAR OCFloor = 0.06
RETURN IF([Overcollateralization (OC) %] < OCFloor, "BREACH", "PASS")

Cumulative Net Loss Trigger Test =
-- Breach if cumulative net loss ratio exceeds a step-down schedule threshold (illustrative flat 5%)
VAR CNLThreshold = 0.05
VAR CNL = DIVIDE(SUM(FactDynamicLossMonthly[NetLoss_ThisMonth]), [Total Original Pool Balance])
RETURN IF(CNL > CNLThreshold, "BREACH", "PASS")

Subordination % (Class A) =
VAR ABal = LOOKUPVALUE(DimTranche[OriginalFaceValue], DimTranche[TrancheID], "A")
VAR TotalNoteFace = SUMX(DimTranche, DimTranche[OriginalFaceValue])
RETURN DIVIDE(TotalNoteFace - ABal, TotalNoteFace)
```

---

## 9. Dynamic Pool Performance (from `FactDynamicLossMonthly`)

```DAX
Collection Efficiency (Latest Month) =
CALCULATE(AVERAGE(FactDynamicLossMonthly[CollectionEfficiency]), LASTDATE(FactDynamicLossMonthly[ReportingDate]))

CPR – 3M Rolling Average =
AVERAGEX(
    DATESINPERIOD(FactDynamicLossMonthly[ReportingDate], LASTDATE(FactDynamicLossMonthly[ReportingDate]), -3, MONTH),
    CALCULATE(AVERAGE(FactDynamicLossMonthly[CPR_Annualised]))
)

SMM – Latest Month =
CALCULATE(AVERAGE(FactDynamicLossMonthly[SMM]), LASTDATE(FactDynamicLossMonthly[ReportingDate]))

Net Loss Rate – Annualised (Latest 12M) =
VAR Sum12MNetLoss = CALCULATE(SUM(FactDynamicLossMonthly[NetLoss_ThisMonth]), DATESINPERIOD(FactDynamicLossMonthly[ReportingDate], LASTDATE(FactDynamicLossMonthly[ReportingDate]), -12, MONTH))
VAR AvgBOPBal = CALCULATE(AVERAGE(FactDynamicLossMonthly[BOP_Balance]), DATESINPERIOD(FactDynamicLossMonthly[ReportingDate], LASTDATE(FactDynamicLossMonthly[ReportingDate]), -12, MONTH))
RETURN DIVIDE(Sum12MNetLoss, AvgBOPBal)

Excess Spread – Annualised % =
VAR MonthlySpread = CALCULATE(AVERAGE(FactDynamicLossMonthly[ExcessSpread_Monthly]), LASTDATE(FactDynamicLossMonthly[ReportingDate]))
VAR AvgBal = CALCULATE(AVERAGE(FactDynamicLossMonthly[BOP_Balance]), LASTDATE(FactDynamicLossMonthly[ReportingDate]))
RETURN DIVIDE(MonthlySpread, AvgBal) * 12

Default Rate Trend (MoM Delta) =
VAR CurrentRate = CALCULATE(AVERAGE(FactDynamicLossMonthly[MonthlyDefaultRate]), LASTDATE(FactDynamicLossMonthly[ReportingDate]))
VAR PriorRate = CALCULATE(AVERAGE(FactDynamicLossMonthly[MonthlyDefaultRate]), DATEADD(LASTDATE(FactDynamicLossMonthly[ReportingDate]), -1, MONTH))
RETURN CurrentRate - PriorRate

Pool Runoff % (since deal close) =
1 - DIVIDE(
    CALCULATE(SUM(FactDynamicLossMonthly[EOP_Balance]), LASTDATE(FactDynamicLossMonthly[ReportingDate])),
    CALCULATE(SUM(FactDynamicLossMonthly[BOP_Balance]), FIRSTDATE(FactDynamicLossMonthly[ReportingDate]))
)
```

---

## 9.1 Static Pool Vintage / Cohort Curves

```DAX
Cumulative Net Loss Rate (by Vintage & MoB) =
AVERAGE(FactStaticPoolVintage[CumulativeNetLossRate])
-- Plot on a line chart: X = MonthsOnBook, Line = VintageID, Y = this measure.
-- The classic "vintage curve" chart — every ABS investor deck has this.

Marginal Loss Rate (by Vintage & MoB) =
AVERAGE(FactStaticPoolVintage[MarginalLossRate])

Vintage Curve vs Benchmark Vintage =
VAR BenchmarkVintage = "2021-Q1"  -- swap for your seasoned reference cohort
VAR BenchmarkValue =
    CALCULATE(
        AVERAGE(FactStaticPoolVintage[CumulativeNetLossRate]),
        FactStaticPoolVintage[VintageID] = BenchmarkVintage,
        VALUES(FactStaticPoolVintage[MonthsOnBook])
    )
RETURN [Cumulative Net Loss Rate (by Vintage & MoB)] - BenchmarkValue
-- Positive = current vintage is underperforming the benchmark cohort at the same MoB point —
-- this is how you catch underwriting deterioration early, well before it shows up pool-wide.

Static Pool Factor (by Vintage) =
AVERAGE(FactStaticPoolVintage[PoolFactor])

30+ DPD % (Static Pool basis) =
AVERAGE(FactStaticPoolVintage[CurrentDelinq30Plus])

Vintage Loss Curve – Projected Ultimate Loss =
-- Simple curve-fitting extrapolation: takes the last 3 marginal loss rate observations and
-- projects forward assuming a decaying tail (illustrative — replace with an actuarial curve
-- fit, e.g. logistic or Weibull, for production use)
VAR LastMoB = MAX(FactStaticPoolVintage[MonthsOnBook])
VAR RecentMarginal =
    CALCULATE(
        AVERAGE(FactStaticPoolVintage[MarginalLossRate]),
        FactStaticPoolVintage[MonthsOnBook] >= LastMoB - 2
    )
VAR RemainingMonths = 84 - LastMoB   -- 84 = typical max original term in your loan tape
RETURN [Cumulative Net Loss Rate (by Vintage & MoB)] + (RecentMarginal * RemainingMonths * 0.5)
-- The *0.5 tail-decay factor is a placeholder assumption — flag clearly in any investor-facing use.
```

---

## 10. Stress Testing Framework (What-If parameters)

```DAX
-- Step 1: create What-If parameter tables (Modeling > New Parameter, or manually):
WhatIf PD Stress Multiplier = GENERATESERIES(1.0, 3.0, 0.25)
WhatIf LGD Stress Multiplier = GENERATESERIES(1.0, 2.0, 0.1)
WhatIf CPR Stress Shift = GENERATESERIES(-0.10, 0.10, 0.01)   -- additive shift to CPR

_Selected PD Multiplier = SELECTEDVALUE('WhatIf PD Stress Multiplier'[Value], 1.0)
_Selected LGD Multiplier = SELECTEDVALUE('WhatIf LGD Stress Multiplier'[Value], 1.0)
_Selected CPR Shift = SELECTEDVALUE('WhatIf CPR Stress Shift'[Value], 0.0)

Stressed ECL =
SUMX(
    FactLoanSnapshot,
    MIN(FactLoanSnapshot[PD_Estimate] * [_Selected PD Multiplier], 1.0) *
    MIN(FactLoanSnapshot[LGD_Estimate] * [_Selected LGD Multiplier], 1.0) *
    FactLoanSnapshot[EAD]
)

Stressed ECL Delta vs Base =
[Stressed ECL] - [Total ECL Provision]

Stressed Coverage Ratio =
DIVIDE([Stressed ECL], [Total EAD])

Stressed CPR =
[CPR – 3M Rolling Average] + [_Selected CPR Shift]

Stressed Net Loss Rate (scenario-scaled) =
-- Applies the Scenario calculation group (§3.2) directly against the base net loss rate
[Net Loss Rate – Annualised (Latest 12M)]   -- wrap in the calc group at report level, not in DAX

Capital Adequacy Impact (Illustrative RWA sensitivity) =
-- Higher PD/LGD under stress increases risk-weighted assets under IRB approaches —
-- this is a simplified directional indicator, not a regulatory capital calculation.
[Stressed ECL Delta vs Base] * 12.5   -- 12.5 = 1/8% minimum capital ratio, illustrative multiplier only

Scenario Comparison Table (measure to drop on a matrix with Scenario calc group on columns) =
[Total ECL Provision]
```

**Recommended stress scenarios to pre-build as bookmarks:**
| Scenario | PD multiplier | LGD multiplier | CPR shift | Narrative |
|---|---|---|---|---|
| Base Case | 1.0x | 1.0x | 0% | Current model assumptions |
| Mild Stress | 1.5x | 1.2x | -3% | Moderate rate/employment shock |
| Severe Stress (2008/2020-style) | 3.0x | 1.75x | -8% | Systemic credit crisis, collateral value collapse |
| Idiosyncratic Servicer Failure | 1.0x | 1.5x | -5% | Single servicer operational failure raises LGD only |

---

## 11. Time Intelligence

```DAX
MTD Net Loss =
CALCULATE(SUM(FactDynamicLossMonthly[NetLoss_ThisMonth]), USERELATIONSHIP(DimDate[Date], FactDynamicLossMonthly[ReportingDate]))

YTD Net Loss =
CALCULATE(TOTALYTD(SUM(FactDynamicLossMonthly[NetLoss_ThisMonth]), DimDate[Date]))

Net Loss – Prior Month =
CALCULATE([MTD Net Loss], DATEADD(DimDate[Date], -1, MONTH))

Net Loss – MoM % Change =
DIVIDE([MTD Net Loss] - [Net Loss – Prior Month], [Net Loss – Prior Month])

Rolling 6M Average Default Rate =
CALCULATE(
    AVERAGE(FactDynamicLossMonthly[MonthlyDefaultRate]),
    DATESINPERIOD(DimDate[Date], LASTDATE(DimDate[Date]), -6, MONTH),
    USERELATIONSHIP(DimDate[Date], FactDynamicLossMonthly[ReportingDate])
)
```

---

## 12. Report Page Blueprint (maps measures → visuals)

| Page | Visuals | Key measures used |
|---|---|---|
| **1. Portfolio Overview** | KPI cards, geography map, servicer donut | §4 all |
| **2. RBI Delinquency & Roll Rates** | SMA bucket stacked area over time, roll-rate matrix | §5 |
| **3. IFRS9 ECL** | Stage 1/2/3 waterfall chart, PD/LGD/EAD scatter | §6 |
| **4. RBI vs IFRS9 Reconciliation** | 3×5 matrix (Stage × SMA class), Provisioning Gap KPI card, mismatch table | §7 |
| **5. Waterfall & Structural Triggers** | Waterfall chart (collections → fees → interest → principal → equity), OC/CNL trigger cards (Pass/Breach conditional formatting) | §8 |
| **6. Dynamic Pool Performance** | CPR/SMM trend, collection efficiency gauge, excess spread trend | §9 |
| **7. Vintage & Cohort Curves** | Multi-line vintage curve chart, vintage vs benchmark heatmap | §9.1 |
| **8. Stress Testing** | Field-parameter-driven scenario selector, tornado chart of ECL sensitivity to PD/LGD/CPR | §10 |

---

## 13. Data gaps to close before production use (be upfront about these)

1. **No PoolID on `dynamic_loss_monthly.csv`** — add it now; trivial with one pool, blocking once you have two.
2. **No monthly IFRS9_Stage history** — only RBI_SMA_Class is tracked monthly. A genuine Stage migration matrix needs this.
3. **No tranche/note structure table** — waterfall measures in §8 are illustrative; real deal documents (subordination %, coupon, trigger thresholds) must replace the `DimTranche` placeholder.
4. **RBI IRACP provisioning rates in `DimRBI_SMA` are indicative** — confirm current rates from the applicable RBI Master Direction before this feeds any regulatory filing.
5. **Vintage curve extrapolation (§9.1) uses a simplistic decay assumption** — fine for a dashboard sensitivity view, not a substitute for an actuarial loss-curve fit if this feeds ECL directly.

---

*This library is designed to paste directly into Power BI Desktop (Modeling → New Measure), Tabular Editor for calculation groups, or SSAS Tabular via SSDT. All measures reference table/column names exactly as they appear in your four uploaded CSVs.*
