# Data dictionary

## Bus matrix

| Business process | Grain | DimDate | DimStore | DimProduct | DimCustomer |
|---|---|:-:|:-:|:-:|:-:|
| Retail sales | One row per sales order line | ✓ | ✓ | ✓ | ✓ |

## SCD attribute matrix

| Dimension | Attribute | Type | Why |
|---|---|---|---|
| DimStore | Region | **2** | Sales must stay reported under the region that owned the store at the time |
| DimStore | StoreType | **2** | A remodel (Express to Standard) changes store economics; history should reflect the format in operation |
| DimStore | CurrentRegion | **6** | Today's region on every version, for restated "as is" reporting |
| DimStore | StoreName, City, State, Country, OpenDate | 1 | Corrections; history has no analytical value |
| DimProduct | UnitCost | **2** | Historical margin depends on the cost at the time of sale |
| DimProduct | Category, Subcategory | **2** | Reclassification must not rewrite how past sales were categorised |
| DimProduct | ProductName, Brand, ListPrice | 1 | Corrections and reference values; the fact stores the actual selling price |
| DimCustomer | All attributes | 1 | The business only needs current customer details (matches the lab's design) |
| DimDate | All attributes | 0 | Static reference data |

## Measure additivity

| Measure | Additivity | How to aggregate |
|---|---|---|
| Quantity | Additive | `SUM` |
| UnitPrice | Non-additive | Derive as `SUM(SalesAmount) / SUM(Quantity)`; never `SUM` |
| SalesAmount | Additive | `SUM` |
| DiscountAmount | Additive | `SUM` |
| NetSalesAmount | Additive | `SUM` |
| CostAmount | Additive | `SUM` |
| Gross margin % | Non-additive | Ratio of sums: `(SUM(Net) - SUM(Cost)) / SUM(Net)` |

## Hierarchies

| Dimension | Hierarchy |
|---|---|
| DimDate | Year → Quarter → Month → Day; FiscalYear → FiscalQuarter → FiscalMonth |
| DimStore | Region → Country → State → City → Store |
| DimProduct | Category → Subcategory → Brand → Product |
| DimCustomer | Segment → LoyaltyTier → Customer |

## Tables

### fact.FactSales

| Column | Type | Description |
|---|---|---|
| DateKey | INT | FK to DimDate (YYYYMMDD of the order date) |
| StoreKey | INT | FK to the DimStore version valid on the order date; -1 if unresolved |
| ProductKey | INT | FK to the DimProduct version valid on the order date; -1 if unresolved |
| CustomerKey | INT | FK to DimCustomer; inferred member if not yet mastered; -1 if no customer |
| SalesOrderNumber | VARCHAR(20) | Degenerate dimension, part of the grain |
| SalesOrderLine | INT | Degenerate dimension, part of the grain |
| Quantity | INT | Units sold |
| UnitPrice | DECIMAL(10,2) | Actual selling price per unit |
| SalesAmount | DECIMAL(12,2) | Quantity × UnitPrice |
| DiscountAmount | DECIMAL(12,2) | Discount applied to the line |
| NetSalesAmount | DECIMAL(12,2) | SalesAmount − DiscountAmount |
| CostAmount | DECIMAL(12,2) | Quantity × UnitCost of the product version valid on the order date |
| LoadBatchId | INT | Batch that loaded the row (lineage to audit.LoadBatch) |
| LoadedAtUtc | DATETIME2(6) | Load timestamp |

### dim.DimStore

| Column | Type | Description |
|---|---|---|
| StoreKey | INT | Surrogate key, one per version; -1 = Unknown |
| StoreCode | VARCHAR(10) | Natural key from the source |
| StoreName, City, State, Country, OpenDate | | Type 1 attributes |
| StoreType, Region | VARCHAR | Type 2 attributes |
| CurrentRegion | VARCHAR(50) | Type 6: current region, repeated on every version |
| ValidFrom / ValidTo | DATE | Half-open validity window; current version ends 9999-12-31 |
| IsCurrent | BIT | 1 for the current version |
| InsertedBatchId / UpdatedBatchId / UpdatedAtUtc | | Row lineage |

### dim.DimProduct

| Column | Type | Description |
|---|---|---|
| ProductKey | INT | Surrogate key, one per version; -1 = Unknown |
| ProductCode | VARCHAR(10) | Natural key |
| ProductName, Brand, ListPrice | | Type 1 attributes |
| Category, Subcategory, UnitCost | | Type 2 attributes |
| ValidFrom / ValidTo / IsCurrent | | As DimStore |
| InsertedBatchId / UpdatedBatchId / UpdatedAtUtc | | Row lineage |

### dim.DimCustomer

| Column | Type | Description |
|---|---|---|
| CustomerKey | INT | Surrogate key; -1 = Unknown (anonymous sale) |
| CustomerCode | VARCHAR(10) | Natural key |
| CustomerName, Segment, City, State, Country, LoyaltyTier, JoinDate | | Type 1 attributes |
| IsInferred | BIT | 1 = placeholder created from a sale before the CRM record arrived |
| InsertedBatchId / UpdatedBatchId / UpdatedAtUtc | | Row lineage |

### dim.DimDate

| Column | Description |
|---|---|
| DateKey | YYYYMMDD integer |
| FullDate | Calendar date (mark as the date table in Power BI) |
| Year, Quarter, QuarterName, Month, MonthName, MonthShort, YearMonth, Day | Calendar attributes |
| DayOfWeekNumber, DayOfWeek | 1 = Monday, independent of server settings |
| FiscalYear, FiscalQuarter, FiscalMonth | Fiscal year starts 1 July, named by the year it ends |
| IsWeekday, IsHoliday, HolidayName | US federal holidays, including floating dates |

### audit schema

| Table | Purpose |
|---|---|
| audit.LoadBatch | One row per run: parameters, status, timings, error message |
| audit.LoadStepLog | Rows inserted, updated, deleted and rejected per step |
| audit.RejectedSalesLine | Quarantined source lines with the reason |
| audit.DataQualityResult | Every check result per batch |

### rpt views

| View | Purpose |
|---|---|
| rpt.vw_SalesDetail | Flat fact plus dimension attributes plus margin |
| rpt.vw_SalesByMonthCategory | Monthly sales and margin by category |
| rpt.vw_ProductMarginByMonth | Monthly margin per product across SCD versions |
| rpt.vw_RegionSales_AsWasVsAsIs | Store revenue labelled with historical and current region |
| rpt.vw_PipelineRunHistory | Operational history of every batch |
| rpt.vw_DataQualityLatest | DQ results for the most recent batch |
