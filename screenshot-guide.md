# Screenshot guide

Save each image in the `screenshots/` folder with exactly the filename below; the README already links to the starred ones.

| # | Filename | What to capture | Runbook step |
|---|---|---|---|
| 01 | `01-workspace.png` | Workspace on trial capacity with the `ContosoDW` warehouse | 1 |
| 02 | `02-warehouse-schemas.png` | Explorer expanded: the six schemas with tables, views and procedures | 2 |
| 03 ⭐ | `03-model-view.png` | Model layouts tab arranged as a star | 3 |
| 04 | `04-batch1-run-summary.png` | Batch 1 summary and step log | 4 |
| 05 | `05-batch1-dq-pass.png` | Batch 1 DQ results, all Pass | 4 |
| 06 ⭐ | `06-product-scd-history.png` | DimProduct history query | 7 |
| 07 | `07-store-scd-history.png` | DimStore history with Region and CurrentRegion | 7 |
| 08 ⭐ | `08-historical-cost-binding.png` | MB-PRO sales and the cost version each bound to | 7 |
| 09 ⭐ | `09-as-was-vs-as-is.png` | As-was versus as-is region totals | 8 |
| 10 ⭐ | `10-data-quality-results.png` | Batch 2 DQ results with warnings | 6 |
| 11 ⭐ | `11-idempotency-test.png` | Rerun test, all Pass | 9 |
| 12 | `12-inferred-member-resolved.png` | CUST-013 before and after | 10 |
| 13 | `13-data-pipeline.png` | Data Pipeline canvas with a successful run | 12 |
| 14 ⭐ | `14-power-bi-report.png` | Power BI report page | 13 |
| 15 | `15-pipeline-run-history.png` | `rpt.vw_PipelineRunHistory` across all batches | 13 |

## Tips for clean screenshots

- Use the browser at 100% zoom, full width; close the Copilot pane and notifications.
- Crop to the relevant area: the query and its results, not the whole browser.
- For result grids, widen columns so no values are truncated.
- Hide anything personal in the top bar (account name, tenant). The workspace name is fine.
- PNG at around 1600 px wide keeps text sharp on GitHub without large files.
- Optional: record a 30 to 60 second GIF or video of a batch run plus the DQ results for your LinkedIn post.
