# Power BI semantic model

## Create the model

1. In the `ContosoDW` warehouse, open the **Reporting** tab and select **New semantic model**.
2. Name it `Contoso Retail Sales` and select:
   `dim.DimDate`, `dim.DimStore`, `dim.DimProduct`, `dim.DimCustomer`, `fact.FactSales`.
3. Confirm. The model is created in **Direct Lake** mode.

## Relationships

Open the model and check the relationships created from the warehouse foreign keys. Add any that are missing.

| From (many) | To (one) | Cardinality | Cross-filter |
|---|---|---|---|
| FactSales[DateKey] | DimDate[DateKey] | Many to one | Single |
| FactSales[StoreKey] | DimStore[StoreKey] | Many to one | Single |
| FactSales[ProductKey] | DimProduct[ProductKey] | Many to one | Single |
| FactSales[CustomerKey] | DimCustomer[CustomerKey] | Many to one | Single |

Relationships join on **surrogate keys**, so every fact row automatically picks up the dimension version that was valid when the sale happened.

## Model settings

- **Mark as date table:** DimDate, using `FullDate`.
- **Sort by column:** `MonthName` by `Month`, `MonthShort` by `Month`, `DayOfWeek` by `DayOfWeekNumber`.
- **Hide:** all key columns, `ValidFrom`, `ValidTo`, `IsCurrent`, `InsertedBatchId`, `UpdatedBatchId`, `UpdatedAtUtc`, `LoadBatchId`, `LoadedAtUtc`, and the raw measure columns on FactSales (use measures instead).
- **Hierarchies:** Calendar (Year → QuarterName → MonthName → FullDate), Fiscal (FiscalYear → FiscalQuarter → FiscalMonth), Geography (Region → State → City → StoreName), Product (Category → Subcategory → ProductName).
- **Display folders** for measures: Sales, Margin, Time Intelligence, Data Quality.
- **Formats:** currency for sales and cost, percentage with one decimal for rates.

## Measures

Create a measure table (for example `_Measures`) and add the measures in [`powerbi/measures.dax`](../powerbi/measures.dax).

## Suggested report (one or two pages)

**Page 1: Sales performance**
- KPI cards: Net Sales, Gross Margin %, Orders, Avg Order Value
- Line chart: Net Sales and Gross Margin % by month
- Bar chart: Net Sales by Category
- Matrix: Region (rows) × MonthShort (columns), Net Sales

**Page 2: Historical accuracy**
- Line chart: Gross Margin % by month for Mountain Bike Pro (filter ProductName). The step down in March is the SCD Type 2 cost change.
- Two clustered bar charts side by side: Net Sales by `DimStore[Region]` (as was) and by `DimStore[CurrentRegion]` (as is). The Central and South bars differ by ST-005's January and February revenue.
- Card: Unknown Member Share

Take `14-power-bi-report.png` of the page that tells the story best (page 2 is the more distinctive one).

## Note on views

The `rpt` views are for SQL users and ad hoc analysis. If you add them to a Direct Lake model, queries against views fall back to DirectQuery, so the star schema tables plus DAX measures are the recommended model.
