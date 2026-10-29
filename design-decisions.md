# Design decisions

Short records of the main choices, what was considered, and why.

---

### 1. Layered schemas instead of `dbo`

**Decision:** separate `src`, `stg`, `dim`, `fact`, `audit` and `rpt` schemas.
**Why:** each layer has a different owner and lifecycle. Staging is disposable; dimensions and facts are the contract; `rpt` is what analysts and Power BI see, so it can be secured and changed without touching the base tables. It also mirrors the bronze, silver and gold separation teams already use in Fabric.

### 2. Constraints declared but enforced by the pipeline

**Decision:** declare primary, unique and foreign keys as `NOT ENFORCED`, and verify integrity with `audit.usp_RunDataQualityChecks` after every batch.
**Why:** Fabric Warehouse does not enforce constraints. Declaring them still documents the model and lets Power BI detect relationships. Because nothing in the engine prevents an orphan or a duplicate, the guarantee has to come from the load design and be proven by checks.

### 3. Surrogate keys from `MAX + ROW_NUMBER()`

**Decision:** assign keys in the load procedures.
**Why:** a portable, predictable pattern that produces dense integers and works on any T-SQL engine. It is safe because the orchestrator serialises batches. **Trade-off:** it would not be safe for concurrent writers; in that scenario I would use an `IDENTITY` column where available or a key-allocation table.

### 4. Half-open validity windows

**Decision:** a version is valid where `ValidFrom <= date < ValidTo`; the expired version's `ValidTo` equals the next version's `ValidFrom`.
**Why:** no off-by-one gaps, no "minus one day" arithmetic, and the lookup predicate is identical for every version. The DQ suite checks for overlaps and gaps.

### 5. First version starts on 1900-01-01

**Decision:** a member's first version is open from 1900-01-01, not from its first load date.
**Why:** a sale dated before the dimension was first loaded (historical backfill, late-arriving data) still resolves to a real version instead of falling to Unknown. Later versions start on the batch's effective date.

### 6. Same-day correction rule

**Decision:** if the current version started on the batch's effective date and changes again, update it in place.
**Why:** otherwise a correction made on the same day creates a zero-length version (`ValidFrom = ValidTo`) that no fact can ever bind to.

### 7. Type 6 `CurrentRegion` on DimStore

**Decision:** keep `Region` as Type 2 and add `CurrentRegion` as a Type 1 column copied onto every version.
**Why:** the business needs both "as was" and "as is" region reporting. Type 6 gives both from one join, so a Power BI user just picks the column. The alternative, a second join to the current row on the natural key, is easy to get wrong in self-service reports.

### 8. Cost captured on the fact

**Decision:** compute `CostAmount` during the fact load from the product version valid on the order date.
**Why:** margin becomes a simple additive measure, fast in Direct Lake, and still historically correct. Recomputing it at query time would need a date-range join to the product history in every report.

### 9. Unknown members versus inferred members

**Decision:** use Unknown (`-1`) for references that will never resolve (anonymous sale, invalid product code). Use inferred members for references that will (a customer the CRM has not sent yet).
**Why:** both keep fact totals complete. The inferred member keeps the sale tied to the real customer from day one; when the record arrives, the Type 1 load completes the row with the same key, so no facts are reloaded. Both are reported as DQ warnings so they are visible, not hidden.

### 10. Quarantine instead of failing the batch

**Decision:** invalid lines go to `audit.RejectedSalesLine` with a reason; valid lines load.
**Why:** one bad line should not block a day's trading data. The rejected rows stay queryable for the source team to fix, and when a corrected line arrives the idempotent load replaces it.

### 11. Idempotent fact load by degenerate keys

**Decision:** delete fact rows for every staged order line, then insert.
**Why:** replaying a window (after a failure, or to pick up corrections) produces the same result instead of duplicates. The rerun test in `05_run/04` proves it. **Alternative considered:** `MERGE`, which is a good extension; delete-and-insert is simpler to reason about and performs well at this scale.

### 12. Incremental extract on CreatedDate, not OrderDate

**Decision:** the sales window filters on when a record reached the source.
**Why:** filtering on the business date would miss late-arriving lines forever. Extracting by arrival date and looking up dimensions by business date gives both completeness and historical accuracy.

### 13. One transaction per batch

**Decision:** dimension and fact loads commit together; the batch row is written outside the transaction.
**Why:** a failure never leaves new dimension versions without their facts (or the reverse), and the failed run stays visible with its error message.

### 14. Effective-date ordering guard rail

**Decision:** `audit.usp_RunPipeline` refuses an effective date earlier than the last successful batch.
**Why:** applying an older change after a newer one would expire the wrong version and corrupt Type 2 history. Making the procedure refuse is safer than documenting "please don't".

### 15. Deterministic synthetic data

**Decision:** generate data from integer arithmetic on a tally table, not `NEWID()` or `RAND()`.
**Why:** every run produces identical data, so the expected results in the runbook are exact and anyone can reproduce the screenshots.

### 16. Data quality after commit

**Decision:** run checks after the load commits and record the outcome in the batch status (`Data quality failed`).
**Why:** results stay persisted for investigation. **Trade-off:** a failing batch is visible but not rolled back. A stricter design would gate the commit on Error checks; I chose observability for this project and note gating as an extension.
