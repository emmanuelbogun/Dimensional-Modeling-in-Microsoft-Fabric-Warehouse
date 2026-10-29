# Interview guide

How to talk about this project in a data engineering or BI interview. The answers are written to be said out loud, in your own voice, so adapt the wording until it sounds natural.

---

## The 60-second version

> "I took a Microsoft Learn lab on dimensional modelling in Fabric and rebuilt it as a proper pipeline. The lab creates a star schema and simulates slowly changing dimensions with a few hand-written updates. I turned that into a layered warehouse with source, staging, dimension, fact and audit schemas, driven by one orchestrator stored procedure that I schedule from a Fabric Data Pipeline.
>
> The interesting part is the edge cases. Product cost is tracked as SCD Type 2, so margin stays correct after a price rise. Store region is Type 2 with a Type 6 current-region column, so the business can report as it was or as it is. Late-arriving sales bind to the dimension version that was valid on the order date. Sales for customers the CRM hasn't sent yet get an inferred member that's completed later without reloading facts. Bad rows go to quarantine, and every batch finishes with twenty data quality checks, because Fabric doesn't enforce constraints. I also proved the load is idempotent by replaying a batch and showing nothing changes."

## The STAR version

- **Situation:** a retail business needed sales analysis by store, product, customer and time, and the numbers had to stay correct when master data changed.
- **Task:** design the dimensional model and a load process that handles change, late and bad data, and can be rerun safely.
- **Action:** layered schemas; hybrid SCD logic (Types 1, 2 and 6); point-in-time surrogate key lookups; inferred and Unknown members; quarantine; one transaction per batch; a twenty-check DQ suite; an idempotency test; a Data Pipeline schedule; a Direct Lake semantic model with DAX measures.
- **Result:** 2,468 fact rows loaded across four batches with zero integrity errors; three invalid lines quarantined instead of failing the load; the Mountain Bike Pro margin drop after the March cost increase visible without any rework of history; a rerun produced identical totals.

---

## Likely questions

**Why a star schema rather than one wide table?**
> "Dimensions are shared across facts and change independently. If store region lived on every sales row, a region change would mean rewriting history or living with inconsistency. With a star schema, I control that per attribute, and it's the shape Power BI's engine is optimised for."

**Explain SCD Type 1, 2 and 6 using your project.**
> "Type 1 overwrites: when Alex Chen's name was corrected to Alexandra, every past sale shows the new name, because the old one was simply wrong. Type 2 adds a version: when Mountain Bike Pro's cost went from 1,200 to 1,350, I expired the old row and inserted a new one, so February sales still carry 1,200. Type 6 combines them: the store row keeps the historical region, and a current-region column is overwritten on every version, so one join gives both views."

**How do you choose which attributes are Type 2?**
> "I ask whether a historical report would be wrong if the value changed. Cost and region pass that test: margin and regional targets depend on them. A product name typo doesn't, so that's Type 1. I documented the decision per attribute in the data dictionary, because it's a business decision as much as a technical one."

**What's a surrogate key and why not use the product code?**
> "The product code identifies the product; the surrogate key identifies a version of it. Once you have Type 2 history, MB-PRO exists twice, so the code can't be the primary key. Surrogate keys also insulate the warehouse from source system key changes and keep joins on small integers."

**How do you handle late-arriving facts?**
> "I extract by the date the record reached the source, so nothing is missed, but I look up dimensions by the business date. A February sale that arrived in March still binds to February's cost and region. I included four of those in the test data and verified them."

**And early-arriving facts, where the dimension doesn't exist yet?**
> "That's the inferred member pattern. If a sale references customer 13 before the CRM sends them, I create a placeholder row with the real customer code and a flag. When the record arrives, the Type 1 load fills it in, keeping the same key, so the facts never need reloading. Unknown, minus one, is different: it's for references that will never resolve, like an anonymous walk-in sale."

**Fabric doesn't enforce foreign keys. How do you know the data is right?**
> "The constraints are there for documentation and so Power BI detects relationships, but enforcement is my job. Every batch runs twenty checks: orphaned keys, duplicate grain, more than one current version, overlapping or gapped validity windows, facts bound to a version that wasn't valid on the order date, measure arithmetic, and a reconciliation between staged and loaded rows. Errors and warnings are separated, and the outcome sets the batch status."

**What happens if the load fails halfway?**
> "Dimension and fact changes run in one transaction, so it rolls back as a unit. The batch record is written before the transaction, so the failure is still logged with its error message. Because the load is idempotent, the fix is simply to rerun the same window."

**How did you prove it was idempotent?**
> "I captured row counts and totals for every table, replayed the batch, and compared. Fact rows, net sales, cost and every dimension count matched. The fact load deletes and reinserts by order number and line, and the dimension loads only act when a compared attribute actually differs."

**What would break your design?**
> "Concurrent loads, because I assign surrogate keys with MAX plus ROW_NUMBER and rely on the orchestrator to run one batch at a time. I'd move to an identity column or a key table for that. Running batches out of effective-date order would also corrupt history, so I made the orchestrator refuse it instead of relying on people."

**Why store cost on the fact instead of looking it up?**
> "It's captured once at load time from the right version, so margin becomes a simple additive sum. That's faster in Direct Lake and impossible to get wrong in a self-service report."

**What would you do next?**
> "Land the source as Delta tables in a Lakehouse bronze layer, move deployments to Fabric Git integration with dev and prod workspaces, gate the commit on error-level data quality checks, and add an accumulating snapshot fact for order fulfilment."

---

## Numbers worth remembering

| Fact | Value |
|---|---|
| Source lines generated | 2,471 |
| Fact rows after all batches | 2,468 |
| Lines quarantined | 3 |
| Data quality checks per batch | 20 |
| Batches in the demo | 4 (initial, incremental, rerun, inferred-member resolution) |
| SCD Type 2 changes exercised | 4 (two product costs, a store region, a store format) |
| Date dimension | 1,095 days, fiscal year from July, US holidays |

## One sentence for a CV

> Designed and built a Microsoft Fabric Warehouse star schema with an idempotent T-SQL load pipeline covering SCD Types 1, 2 and 6, late-arriving and inferred members, data quarantine and a 20-check data quality framework, orchestrated by a Fabric Data Pipeline and served through a Direct Lake Power BI model.
