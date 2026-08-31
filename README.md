# SSIS ETL Project — Union Bank Data Warehouse

An end-to-end SSIS (SQL Server Integration Services) ETL solution that migrates transactional banking data from a normalized OLTP database (**Union_Bank**) into a dimensional Data Warehouse (**Banking_DWH**) built on a **Star Schema**, including full **Slowly Changing Dimension (SCD)** handling and **incremental fact loading**.

## Overview

This project simulates a real-world banking ETL scenario. It covers the core SSIS concepts of connection management, data flow transformations, SCD Type 0/1/2 patterns, Lookup-based surrogate key resolution, and watermark-based incremental loading.

| | |
|---|---|
| **Source** | `Union_Bank` — 3NF normalized OLTP SQL Server database |
| **Target** | `Banking_DWH` — Star Schema data warehouse |
| **Technology** | SQL Server Integration Services (SSIS), Visual Studio (SSDT), SQL Server |
| **Deployment Model** | SSIS Project Deployment Model |

## Data Warehouse Schema

**Fact table:** `FactTransaction` (incremental load), surrounded by four dimensions:

| Dimension | SCD Type | Notes |
|---|---|---|
| `Dim accountCustomer` | Mixed (0, 1, 2) | Gender/DOB fixed, balance/card fields overwritten, state/account status tracked historically |
| `Dim AtmBranch` | Type 1 | ATM/branch changes overwrite the existing record |
| `Dim transaction` | Type 2 | Transaction type renames preserve full history |
| `DimDate` | Type 0 (static) | Pre-generated once, covers 2000–2030 |

## SSIS Packages

| Package | Purpose |
|---|---|
| `Master.dtsx` | Orchestrates all child packages in dependency order via Execute Package Tasks |
| `Load_DimDate.dtsx` | Populates `DimDate` once (skipped on subsequent runs if already populated) |
| `Load_DimAccountCustomer.dtsx` | Loads Customer + Account + Card data with mixed SCD Type 0/1/2 handling |
| `Load_DimAtmBranch.dtsx` | Loads ATM/Branch data with SCD Type 1 overwrite logic |
| `Load_DimTransaction.dtsx` | Loads transaction types with SCD Type 2 history tracking |
| `Load_FactTransaction.dtsx` | Incrementally loads new ATM transactions using a watermark (`meta_Control_Fact`), resolving all dimension surrogate keys via Lookup transformations |

**Execution order enforced by `Master.dtsx`:**
`Load_DimDate` → `Load_DimAccountCustomer` → `Load_DimAtmBranch` → `Load_DimTransaction` → `Load_FactTransaction`

### Project-level connection managers & parameters

| Connection Manager | Points To |
|---|---|
| `CM_OLTP` | `Union_Bank` (source) |
| `CM_DWH` | `Banking_DWH` (target) |

| Parameter | Default | Usage |
|---|---|---|
| `OLTP_Server` | `.` (localhost) | Source server name |
| `DWH_Server` | `.` (localhost) | Target server name |
| `BatchSize` | `10000` | Project-level parameter defined for future batch loading configuration |

## Key Implementation Details

- **SCD Type 2** for `Dim accountCustomer` (state, account status) and `Dim transaction` (transaction type) is implemented **manually** (Lookup on business key → Conditional Split on changed values → OLE DB Command to expire the old row → OLE DB Destination to insert the new one), rather than via the built-in SSIS SCD Wizard, for better performance on larger tables.
- **Incremental fact load** uses a watermark stored in `meta_Control_Fact`, updated after each successful run so only new transactions are extracted.
- **Lookup error handling**: unmatched rows in `Load_FactTransaction` are redirected to a flat-file error log (tagged with a rejection reason) instead of failing the package.
- **DateSK** is generated as an integer in `YYYYMMDD` format for a direct integer join between `FactTransaction` and `DimDate`.
- Dimension tables are never truncated between runs, preserving surrogate key integrity against the fact table.

## Repository Structure

```
├── Banking_ETL_Project/              # SSIS Visual Studio solution
│   ├── Banking_ETL_Project.sln
│   ├── Banking_ETL_Project.dtproj
│   ├── Project.params
│   ├── Master.dtsx
│   ├── Load_DimDate.dtsx
│   ├── Load_DimAccountCustomer.dtsx
│   ├── Load_DimAtmBranch.dtsx
│   ├── Load_DimTransaction.dtsx
│   └── Load_FactTransaction.dtsx
├── Screenshots/                     # Package designs & test evidence
│   ├── Load_DimAccountCustomer_Package/
│   ├── Load_DimAtmBranch_Package/
│   ├── Load_DimDate_Package/
│   ├── Load_DimTransaction_Package/
│   ├── Load_FactTransaction_Package/
│   ├── Master_Package/
│   └── Running and Testing/         # Before/after SCD & incremental-load proof
├── meta_data.sql                    # Seeds meta_Control_Fact watermark row
├── Update for Testing.sql           # Sample updates to trigger SCD changes
├── Delete for Test.txt              # Resets DWH tables for a clean re-run
└── SSIS_Project_UseCase.pdf         # Full technical specification
```

## Getting Started

1. **Prerequisites**: SQL Server, Visual Studio with the SSIS (SSDT) extension installed.
2. Restore/create the `Union_Bank` (source) and `Banking_DWH` (target) databases.
3. Run `meta_data.sql` against `Banking_DWH` to seed the initial watermark row.
4. Open `Banking_ETL_Project/Banking_ETL_Project.sln` in Visual Studio.
5. Verify the `OLTP_Server` and `DWH_Server` project parameters point to your SQL Server instance.
6. Run `Master.dtsx` to execute the full pipeline in order.
7. To verify incremental loading and SCD behavior, apply `Update for Testing.sql` to the source data and re-run `Master.dtsx` — dimension history rows and only the newly changed data should appear.
8. Use `Delete for Test.txt` to reset the warehouse tables for a clean re-test.

## Testing Evidence

The `Screenshots/Running and Testing/` folder documents:
- Row counts before/after each load
- SCD behavior verified after source updates:
  - Customer state changes create Type 2 history records
  - Transaction type changes create Type 2 history records
  - Branch name changes overwrite the existing row (Type 1)
- Incremental load idempotency (a second run with no new source data loads 0 new fact rows)

---
*Built as part of an SSIS ETL training project (banking domain use case).*
