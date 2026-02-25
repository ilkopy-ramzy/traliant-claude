# Traliant Audit [WIP]

> Source: https://www.notion.so/valiotti-analytics/Traliant-Audit-WIP-30de9a24f9f880d8a59bd923959c3a47

---

## Overview of the Current Data Platform

The current system synchronizes data from Salesforce into the Traliant DWH using Microsoft Fabric. The process runs daily at **12:00** in 3 stages:

1. **Environment Preparation:**
   - `DropSFTables` — clears existing Salesforce-related tables in the warehouse
   - `CreateSFTablesSchema` — initializes the fresh schema to receive new data
2. **The Copy Job:**
   - Managed within Microsoft Fabric using native Salesforce connectors
3. **Data Transmission:**
   - Data is pulled from 21 Salesforce tables via parallel sessions
   - Transfer completes with a "Data Fully Copied" status before committing to final tables

---

## Component #1: Traliant Data Warehouse

**Link:** https://app.powerbi.com/groups/f132b662-3e1e-443d-b02b-39287f26caf4/warehouses/7e4d97c7-3f71-4910-ab98-37d9f1595b56?experience=power-bi

Centralized repository for data from external systems. All primary data assets are in the **`dbo`** schema.

### 1. Data Integration & Tables

- **Source:** Salesforce database
- **Scale:** 21 tables synchronized from Salesforce
- **Examples:** `Account`, `Account_Health__c`, `Account_Value_History__c`, `Campaign`, `CampaignMember`, `Case`, etc.

### 2. Procedures

Two key stored procedures manage the data lifecycle:

- `CreateSFTables` — automates generation of table structures for Salesforce data
- `DropSFTables` — environment cleanup / schema reset

---

## Component #2: ETL Pipeline

**Link:** https://app.powerbi.com/groups/f132b662-3e1e-443d-b02b-39287f26caf4/pipelines/f1f1202b-a79b-4cf6-b237-6c73f2f49038?experience=power-bi

**Full load pattern** — completely refreshes the target environment on every execution cycle.

Three-stage sequential process:

1. **`DropSFTables`** — purges existing Salesforce tables from the `dbo` schema
2. **`CreateSFTablesSchemas`** — recreates necessary table structures and schemas
3. **`CopySFTablesData`** (Copy Job) — transfers raw data from Salesforce into the prepared `dbo` tables

---

## Component #3: Data Ingestion Infrastructure (Salesforce → DWH)

**Link:** https://app.powerbi.com/groups/f132b662-3e1e-443d-b02b-39287f26caf4/copyjobs/0adeb6a5-3386-431f-9d6b-5b3d6e964ad8?experience=power-bi

Primary bridge between Salesforce and the internal DWH via a dedicated Copy Job.

### 1. Source Configuration (Salesforce Connector)

- Establishes a secure link to the live Salesforce database
- **21 individual sessions** run in parallel, each fetching data from one specific object

### 2. Transformation & Transfer (Full Copy Logic)

- **Full Copy** mechanism — destination is an exact replica of source at execution time
- Schema aligned directly from Salesforce objects into `dbo` schema tables

### 3. Destination Integration (DWH Connector)

- Loads ingested data into **TrilliantDataWarehouse**
- Ensures all 21 tables are populated correctly within the `dbo` schema

---

## Evaluation of Requirements for the New Data Platform

Traliant requires a fully automated, operationalized Data Warehouse within their existing Microsoft Fabric instance. The goal is to replace the current ingestion setup with a more robust, scalable, and cost-effective architecture, executed in two primary phases.

### TL;DR

| Category | Key Requirements & Specifications |
|---|---|
| **Objective** | Replace existing ingestion with a robust, automated, and cost-effective Fabric Data Warehouse. |
| **Phase 1: Salesforce** | Ingest 20+ core objects (Accounts, Contacts, Opportunities, etc.) using API names. |
| **Phase 2: LMS** | Consolidate data from LMS Prod, Remote, and eComm dataflows. |
| **Architecture** | Mirror source API names. Logical schema separation for Salesforce vs. LMS. |
| **Pipeline Logic** | Nightly automated refreshes. Incremental loading (Upsert). Auto-detection of Schema Drift. |

---

### Requirement #1: Data Sources & Scope

**Phase 1: Salesforce (CRM)**

Targeted objects:
1. Account
2. Account Value History
3. Account History
4. Campaign
5. Campaign Member
6. Contact
7. Contact History
8. Event
9. Lead
10. Lead History
11. Opportunity
12. Opportunity Contact Role
13. Opportunity Field History
14. Opportunity Line Item
15. Period
16. Product
17. Task
18. User
19. Case
20. Course Rollout
21. AskNicely Response
22. Record Type
23. Group

**Phase 2: LMS (Product Usage Data)**

All data tables currently utilized in the LMS Prod, Remote, and eComm dataflows.

### Requirement #2: Data Warehouse Architecture & Design

- **Naming Conventions:** Table names must strictly mirror source systems (API names)
- **Logical Segmentation:** Distinct schemas for Salesforce vs. LMS tables

### Requirement #3: ETL/ELT Pipeline Specifications

- **Execution Frequency:** Automated nightly refreshes for both Salesforce and LMS
- **Incremental Loading (Upsert Logic):** Upsert records, update existing modified records — no costly "drop and replace". Critical for LMS data (millions of rows).
- **Schema Drift:** Pipelines must automatically detect and ingest newly added fields/columns

### Requirement #4: Open Items for Cost-Benefit Analysis

1. **Salesforce Storage Strategy:**
   - Option A: Full backup of the Salesforce database
   - Option B: Strictly pull the 23 baseline Phase 1 tables
2. **LMS Storage Strategy:**
   - Full LMS replication vs. maintaining current scoped dataflow lists

---

## Proposed Solutions

### Solution 1: Data Platform with Medallion Architecture

> **Intentional:** simplified variant with minimal operational overhead. Does **not** include incremental load, schema drift handling, or error-handling mechanics.

**Flow:**
- Salesforce → Fabric Pipelines (Copy activity) → **Bronze** (Raw Data)
- Bronze → Fabric Pipelines (Spark / dbt/SQL) → **Silver** (Curated Data)
- Silver → Fabric Pipelines (dbt) → **Gold** (Data Marts) → Power BI

**Tech Stack:**
- **Orchestration** — Fabric Data Pipelines
- **Ingestion** — Native Salesforce connector (Copy activity)
- **Transform (default)** — dbt-fabric / SQL Endpoint (covers 90%+ of typical transformations)
- **Transform (exception)** — Spark Notebooks (complex JSON, ML enrichment, heavy custom logic)
- **Modeling (Silver → Gold)** — dbt-fabric (star schemas, aggregations)
- **Storage** — Fabric Lakehouse (Delta tables)
- **BI** — Power BI DirectLake mode

---

### Solution 2: Data Platform with Quarantine Support

**Flow:**
- **Ingestion** — Watermark-based incremental extract (`SystemModstamp > watermark - safety_window`)
- Hard delete handling: parallel `getDeleted()` reconciliation
  ```sql
  UPDATE table
  SET is_deleted = true,
      deleted_at = <sf_deleted_time>
  WHERE sf_id IN (<deleted_ids>)
  ```
- **Schema Drift:**
  - Yes → Quarantine Layer → Alert data owner → Fix/approve → Reprocess Job
  - No → Schema matches registry → Bronze (append-only) → Silver → Gold
- **Silver** — Idempotent MERGE (key: `Id`, version: `SystemModstamp`)
- **Gold** — Data Marts → Power BI (DirectLake)

**Tech Stack:**
- **Orchestration** — Fabric Data Pipelines
- **Ingestion** — Native Salesforce connector + SOQL incremental pulls
- **Schema Registry** — Purview or custom Delta table storing expected schemas per object
- **Schema Drift Detection** — Salesforce Metadata API (`describeSObject`) before ingestion; Spark as fallback
- **Quarantine Storage** — Separate Lakehouse table/folder, flagged for review
- **Data Quality** — Great Expectations or dbt tests
- **Alerting** — Azure Monitor / Logic Apps / Teams webhook
- **Heavy transforms** — Spark Notebooks (PySpark)
- **Light transforms** — dbt-fabric or SQL Endpoint
- **Storage** — Fabric Lakehouse (Delta tables)
- **BI** — Power BI DirectLake mode
- **Catalogue** — Microsoft Purview
- **CI/CD** — Azure DevOps / GitHub Actions + Fabric Git integration

---

### Solution 3: Data Platform without Quarantine Support

**Flow:**
- **Ingestion** — Watermark-based incremental extract (`SystemModstamp > watermark - safety_window`)
- **Bronze** — Append-only raw data in Delta Lakehouse
- **Silver** — Idempotent MERGE (key: `Id`, version: `SystemModstamp`)
- **Gold** — Data Marts → Power BI (DirectLake)
- **Governance** — Data Catalogue tracks lineage/freshness

**Tech Stack:**
- **Orchestration** — Fabric Data Pipelines
- **Ingestion** — Native Salesforce connector + SOQL incremental pulls
- **Heavy transforms** — Spark Notebooks (PySpark)
- **Light transforms** — dbt-fabric or SQL Endpoint
- **Storage** — Fabric Lakehouse (Delta tables)
- **BI** — Power BI DirectLake mode
- **Catalogue** — Microsoft Purview
- **Data quality** — dbt tests or Great Expectations
- **CI/CD** — Azure DevOps / GitHub Actions + Fabric Git integration
- **Monitoring** — Fabric Monitor + Azure Monitor

---

## Solutions Comparison

### Structured Comparison

| | Solution 1 (Medallion) | Solution 2 (with Quarantine) | Solution 3 (without Quarantine) |
|---|---|---|---|
| **CU Consumption** | High — full reload every run | Medium — incremental, but Spark drift detection adds overhead | Low — incremental MERGE only, dbt/SQL for light transforms |
| **Implementation Complexity** | Low — copy activity + dbt, no upsert/drift logic | High — schema registry, quarantine layer, alerting, approval workflow | Medium — watermark logic + idempotent MERGE, no quarantine overhead |
| **Maintenance Complexity** | Low — simple pipeline, easy to debug | High — quarantine requires active ownership and manual approval | Medium — dbt tests surface issues automatically |
| **Data Quality Guarantees** | Weak — no validation layer, schema changes silently break models | Strong — drift caught before Silver, bad batches never pollute production | Good — dbt tests catch issues at Silver, but drift lands in Bronze first |
| **Best Fit** | PoC / quick win | High-frequency schema changes, multiple unstable sources | ✅ Traliant current scope |

### Assumptions & Methodology

- Microsoft Fabric pricing: F4–F8 capacity SKUs (~$0.18–$0.36/CU-hour)
- Data volume: ~21 Salesforce objects (CRM); LMS data in millions of rows (Phase 2)
- Pipeline frequency: nightly automated runs (Phase 1 baseline)
- Monthly costs assume steady-state post-go-live (no burst scenario)

### Side-by-Side Cost Comparison

> Prices are approximate and meant for comparison. Precise calculation can be provided at a later stage.

| Cost Category | Solution 1 – Medallion | Solution 2 – With Quarantine | Solution 3 – Without Quarantine |
|---|---|---|---|
| Orchestration (Fabric Pipelines) | ~$20–$40/mo | ~$30–$60/mo | ~$25–$50/mo |
| Compute – dbt / SQL Endpoint | ~$30–$60/mo | ~$50–$100/mo | ~$40–$80/mo |
| Compute – Spark Notebooks | $0 (not default) | ~$80–$150/mo | ~$60–$120/mo |
| Storage – Delta Lakehouse | ~$15–$30/mo | ~$25–$50/mo | ~$20–$40/mo |
| Schema Registry / Purview | N/A | ~$30–$80/mo | ~$30–$80/mo |
| Alerting (Azure Monitor / Logic Apps) | N/A | ~$10–$25/mo | ~$10–$25/mo |
| CI/CD (DevOps / GitHub Actions) | N/A (optional) | ~$10–$20/mo | ~$10–$20/mo |
| Monitoring (Fabric + Azure Monitor) | N/A (basic) | ~$15–$30/mo | ~$15–$30/mo |
| **Estimated Total Monthly** | **~$65–$130/mo** | **~$250–$515/mo** | **~$200–$415/mo** |
| **Estimated Annual Cost (infra only)** | **~$800–$1,600/yr** | **~$3,000–$6,200/yr** | **~$2,400–$5,000/yr** |
| **Overall Value Rating** | Low – cheap but risky | High – best long-term ROI | Good – strong balance |

---

## Open Cost Items (Requirement #4)

### Salesforce Storage Strategy

**Option A – Full Salesforce Database Backup**
- Higher Fabric storage cost (~2–3× increase)
- Simpler to implement
- Safest for long-term completeness and recoverability
- **Estimated additional cost:** +$20–$60/month vs. scoped extraction

**Option B – Scoped Extraction (Phase 1 tables only — 23 objects)**
- Minimal storage footprint
- Lower CU consumption
- Directly aligned with current business requirements
- More cost-efficient for early phases

**Recommendation:** Start with **Option B (scoped extraction)**. Revisit full backup only if audit or compliance requirements later mandate full historical coverage.

### LMS Storage Strategy

**Full LMS Replication**
- Potentially millions of additional rows
- **Estimated impact:** +$100–$300/month in storage and compute
- Requires disciplined Delta table compaction strategy
- Maximises analytical flexibility but increases operational overhead

**Scoped Dataflow Lists (Current Approach)**
- Lower cost, faster pipeline execution, simpler maintenance
- Risk of missing tables required by future analytics use cases

**Recommendation:** Maintain **scoped dataflow lists** for Phase 2 initial deployment.

---

## FAQ

### When is Quarantine worth it?

Quarantine adds meaningful operational overhead (schema registry, drift-detection notebook, alerting pipeline, manual approval step). It is justified when:

- Source schema changes frequently and unpredictably
- Multiple heterogeneous sources with weak data quality guarantees
- Downstream Gold/BI models are business-critical and a bad schema change could silently corrupt reports

For Traliant's current scope — 23 well-defined Salesforce objects with a versioned API — schema drift is an infrequent, low-blast-radius event. Salesforce's API versioning provides a natural buffer. **Recommendation:** Start with Solution 3. Revisit Quarantine if LMS Phase 2 sources prove unstable or frequent custom field additions break downstream models.

### What are the risks of using dbt-fabric for LMS data?

dbt-fabric works well for the 23 Salesforce objects in Phase 1. LMS data introduces challenges:

- **Volume:** LMS tables contain millions of rows; SQL Endpoint can spike in execution time and risk timeouts
- **Complex transformations:** Heavy deduplication, nested JSON parsing, multi-source consolidation — better handled by Spark
- **Incremental MERGE at scale:** MERGE on millions of rows can be expensive on Fabric's SQL Endpoint without careful partitioning
- **SQL Endpoint limitations:** Concurrency and resource governance constraints can cause contention with parallel dbt models

**Recommendation:** Use dbt-fabric for LMS where transformations are straightforward SQL. Offload heavy deduplication and row-level processing to Spark Notebooks. Establish a threshold (e.g. tables > 5M rows or complex logic) as the trigger to escalate from dbt to Spark.

| | Solution 2 (with Quarantine) | Solution 3 (without Quarantine) |
|---|---|---|
| Schema drift frequency | High / unpredictable | Low / controlled |
| Number of sources | Many, heterogeneous | One or two, versioned |
| Downtime criticality | High | Medium |
| Operational overhead | High | Moderate |
| **Current scope** | Overkill | ✅ Recommended |
