# Runbook: build and run the project in Microsoft Fabric

Total time: about 45 minutes on a Fabric trial capacity. Each step lists what to run, what you should see, and which screenshot to take (filenames match [screenshot-guide.md](screenshot-guide.md)).

---

## 1. Create the workspace and warehouse

1. Go to `https://app.fabric.microsoft.com` and sign in.
2. **Workspaces → + New workspace**. Name it `Contoso Retail DW`. Under *Advanced*, choose **Fabric trial** as the licence mode.
3. In the workspace, select **+ New item → Warehouse** and name it `ContosoDW`.

📸 `01-workspace.png`: the workspace showing the `ContosoDW` warehouse.

## 2. Deploy the database objects

In the warehouse select **New SQL query**. For each file below, paste the contents into the editor and select **Run**. Use a fresh query tab per file (or clear the editor) because each file is a single batch.

| Order | Folder | Files |
|---|---|---|
| 1 | `sql/01_setup` | `01` to `07` |
| 2 | `sql/02_seed` | `01`, `02` |
| 3 | `sql/03_procedures` | `01` to `09` |
| 4 | `sql/04_reporting` | `01` to `06` |

**Checkpoints**

- `01_seed_source_reference_data.sql` returns 6 stores, 7 products, 12 customers.
- `02_seed_source_sales.sql` returns `2400 | 786 | 1614 | 2026-01-01 | 2026-06-30`.

Select **Refresh** in the Explorer and expand **Schemas**. You should see `audit`, `dim`, `fact`, `rpt`, `src` and `stg`.

📸 `02-warehouse-schemas.png`: Explorer expanded to show the schemas, tables, views and stored procedures.

> **Command-line alternative:** `python scripts/deploy.py --server <sql-endpoint> --database ContosoDW` runs all of the above. Find the SQL endpoint under the warehouse's settings (gear icon) → *SQL endpoint*.

## 3. Check the model view

Open the **Model layouts** tab at the bottom of the warehouse. Add `dim.DimDate`, `dim.DimStore`, `dim.DimProduct`, `dim.DimCustomer` and `fact.FactSales`, then arrange them as a star with the fact table in the middle. The relationship lines come from the `NOT ENFORCED` foreign keys.

📸 `03-model-view.png`: the star schema layout. This is the hero image at the top of the README.

## 4. Batch 1: initial load

Run `sql/05_run/01_run_batch1_initial_load.sql`.

| Check | Expected |
|---|---|
| `dim.DimDate` | 1,095 rows, 2025-01-01 to 2027-12-31 |
| Batch 1 status | `Succeeded` |
| `fact.FactSales` step | 786 inserted, 0 deleted, 0 rejected |
| Dimension rows (incl. Unknown) | DimStore 7, DimProduct 8, DimCustomer 13 |
| Data quality | every check `Pass` |

📸 `04-batch1-run-summary.png`: the run summary and step log result sets.
📸 `05-batch1-dq-pass.png`: the data quality result set, all `Pass`.

## 5. Simulate source changes

Run `sql/05_run/02_simulate_source_changes.sql`. It applies the March price, region and name changes to the source and adds the edge-case transactions.

**Checkpoint:** 1,685 lines in the batch 2 window and 2,471 source lines in total.

## 6. Batch 2: incremental load with SCD changes

Run `sql/05_run/03_run_batch2_incremental_load.sql`.

| Check | Expected |
|---|---|
| Batch 2 status | `Succeeded with warnings` |
| `stg.SalesOrderLine window` | 1,685 |
| `fact.FactSales` | 1,682 inserted, 3 rejected (2,468 fact rows in total) |
| `dim.DimCustomer (Type 1)` | 0 inserted, 2 updated |
| `dim.DimCustomer inferred members` | 1 inserted (CUST-013) |
| `dim.DimStore` | Type 1: 1 · expire: 2 · new versions: 2 · Type 6 refresh: 1 |
| `dim.DimProduct` | Type 1: 1 · expire: 2 · new versions: 3 |
| DQ warnings | quarantined 3 · Unknown-member rows 2 · inferred awaiting 1 |
| DQ errors | none |

The rejected lines result set shows the three reasons: zero quantity, negative price, and a discount larger than the line value.

📸 `10-data-quality-results.png`: batch 2 DQ results with the three warnings visible.

## 7. Validate the SCD behaviour

Run the queries in `sql/06_analysis/02_scd_validation_queries.sql` one at a time (highlight a query, then **Run**).

| Query | What to look for | Screenshot |
|---|---|---|
| 1. Product history | MB-PRO and RB-ELT have two versions split at 2026-03-01; WB-STD is renamed with one version; PM-STD exists | `06-product-scd-history.png` |
| 2. Store history | ST-005 has Region Central then South, and CurrentRegion South on both rows; ST-004 has Express then Standard | `07-store-scd-history.png` |
| 3. MB-PRO cost binding | Sales up to 28 February show cost 1,200 (including `SO-LATE-*` rows from batch 2); sales from 1 March show 1,350 | `08-historical-cost-binding.png` |
| 4. Late-arriving facts | `SO-LATE-0003` for ST-005 shows RegionAtSale Central, CurrentRegion South | |
| 5. Customers | CUST-002 renamed, CUST-004 tier Silver, CUST-013 `Pending - CUST-013` with IsInferred 1 | |

## 8. Business insights

Run `sql/06_analysis/03_business_insight_queries.sql`.

- Query 1 shows Mountain Bike Pro gross margin % falling from March, while Road Bike Elite improves.
- Query 2 shows Central and South totals differ between the as-was and as-is perspectives by exactly ST-005's January and February revenue.

📸 `09-as-was-vs-as-is.png`: the result of query 2.

## 9. Idempotency test

Run `sql/05_run/04_idempotency_rerun_test.sql`. It replays batch 2 and compares before and after.

**Expected:** every metric shows `Pass`, and the new batch reports 1,682 rows replaced and 1,682 inserted.

📸 `11-idempotency-test.png`: the Pass/Fail comparison.

## 10. Resolve the inferred member

Run `sql/05_run/05_resolve_inferred_member.sql`.

**Expected:** before, CUST-013 is `Pending - CUST-013` with IsInferred 1. After, it is `Robin Achebe`, Premium, Gold, IsInferred 0, with the **same CustomerKey**, and its two April sales still attached. The batch status is `Succeeded`.

📸 `12-inferred-member-resolved.png`: the before and after result sets.

## 11. Optional: guard-rail tests

Highlight and run each test in `sql/05_run/06_failure_and_guardrail_tests.sql` separately. Both should be refused with a clear error message (50204 and 50202), and no batch row is created.

## 12. Orchestrate with a Data Pipeline

1. In the workspace select **+ New item → Data pipeline**, name it `pl_contoso_daily_load`.
2. Add a **Stored procedure** activity named `Run warehouse batch`.
3. On **Settings**, choose the `ContosoDW` warehouse, select `audit.usp_RunPipeline`, then **Import** the parameters.
4. Set the parameter values with dynamic content (if the imported type is `Datetime`, keep it; the `yyyy-MM-dd` string converts to `DATE`):

   | Parameter | Value |
   |---|---|
   | `EffectiveDate` | `@formatDateTime(pipeline().TriggerTime, 'yyyy-MM-dd')` |
   | `ExtractFromDate` | `@formatDateTime(addDays(pipeline().TriggerTime, -1), 'yyyy-MM-dd')` |
   | `ExtractToDate` | `@formatDateTime(pipeline().TriggerTime, 'yyyy-MM-dd')` |

5. Optional: add a failure path (red arrow) to an **Office 365 Outlook** or **Teams** activity that sends the error message.
6. **Run** once, then add a daily **Schedule**.

The run appears in `rpt.vw_PipelineRunHistory` as a new batch with 0 new fact rows (there is no source data for today), which demonstrates the scheduled path end to end.

📸 `13-data-pipeline.png`: the pipeline canvas with a successful run in the Output pane.

## 13. Semantic model and report

Follow [semantic-model.md](semantic-model.md) to create the semantic model and a short report.

📸 `14-power-bi-report.png`: the report page.
📸 `15-pipeline-run-history.png` (optional): `SELECT * FROM rpt.vw_PipelineRunHistory` showing every batch.

## 14. Start again from scratch

`sql/99_maintenance/99_reset_environment.sql` drops every object created by the project. Then repeat from step 2.

## 15. Clean up

To stop using trial capacity, delete the workspace: **Workspace settings → General → Remove this workspace**. Take all screenshots first.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Invalid object name 'src.Store'` | Setup scripts run out of order | Run `01_setup` in file order, starting with `01_create_schemas.sql` |
| `CREATE/ALTER PROCEDURE must be the first statement in a query batch` | Two files pasted into one editor tab | Run each procedure file on its own |
| Tables do not appear in Explorer | Explorer not refreshed | Select **Refresh**, or reload the browser page |
| Error 50204 when running a batch | An older effective date after a newer one | Expected guard rail. To replay from scratch, run the reset script |
| Error 50203 | Date dimension empty | Run `EXEC dim.usp_BuildDimDate` (included in the batch 1 script) |
| Batch status `Data quality failed` | An Error-severity check found rows | `SELECT * FROM rpt.vw_DataQualityLatest WHERE Status = 'Fail'` |
