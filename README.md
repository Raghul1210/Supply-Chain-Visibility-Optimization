# Supply-Chain-Visibility-Optimization

## Objective
Build a clean, scalable **star/snowflake schema** in Power BI from the raw DataCo Smart Supply Chain transactional dataset, enabling efficient reporting and analysis of sales, shipping performance, product categories, customer segments, and delivery risk — without duplicating data across a single flat table.

## Dataset Source
**DataCo Smart Supply Chain for Big Data Analysis**
Kaggle: https://www.kaggle.com/datasets/shashwatwork/dataco-smart-supply-chain-for-big-data-analysis

The raw dataset is a single flat CSV containing order, customer, product, shipping, and location details combined (e.g. `Customer Id`, `Order Item Cardprod Id`, `Category Id`, `Department Id`, `Market`, `Order City`, `Days for shipping (real)`, `Delivery Status`, `order date (DateOrders)`, `shipping date (DateOrders)`, `Sales`, etc.).

## Tools Used
- **Power BI Desktop** — data modeling, DAX, relationships
- **Power Query Editor (PQE)** — data cleaning, transformation, table splitting
- **DAX** — calculated columns, calculated date table, `USERELATIONSHIP()`

## Data Cleaning and Transformation Steps

### 1. Load & Initial Cleanup
1. Import the CSV into Power BI Desktop.
2. Open **Transform Data** → Power Query Editor.
3. Rename the main query to `Fact_table`.
4. Remove sensitive/unneeded columns: `Customer Email`, `Customer Password`, `Order Zipcode`.

### 2. Build Dimension Tables (via duplication of Fact_table)

| Dimension | Columns Selected | Deduplicated On | Extra Steps |
|---|---|---|---|
| **Dim_Customer** | Customer Id, Customer City, Customer Country, Customer Fname, Customer Lname, Customer Segment, Customer Street, Customer Zipcode, Customer State | Customer Id | — |
| **Dim_Product** | Product Card Id, Product Category Id, Product Name, Product Price, Product Status, Category Id, Department Id | Product Card Id | — |
| **Dim_Category** | Category Id | Category Id | — |
| **Dim_Department** | Department Id | Department Id | — |
| **Dim_Shipping** | Days for shipping (real), Days for shipment (scheduled), Delivery Status, Late_delivery_risk, shipping date (DateOrders), Shipping Mode | All columns (composite) | Added Index column (from 1) → renamed `Shipping_id` |
| **Dim_Location** | Market, Order City, Order Country, Order Region, Order State | All columns (composite) | Added Index column (from 1) → renamed `Location_id` |

For each dimension: **Duplicate `Fact_table`** → **Choose Columns** (Home → Choose Columns) → **Remove Duplicates** (Home → Remove Rows → Remove Duplicates).

For `Dim_Shipping` and `Dim_Location`, since no natural single-column key exists, a **surrogate key** was generated using an Index column starting from 1.

### 3. Reintroduce Surrogate Keys into Fact_table
For both `Dim_Shipping` and `Dim_Location`:
1. In `Fact_table`, go to **Home → Merge Queries**.
2. Top table: `Fact_table`; bottom table: `Dim_Shipping` (or `Dim_Location`).
3. Select all matching common columns (held with Ctrl) as the join key.
4. Join kind: **Left Outer**.
5. Expand the resulting merged column and keep only the surrogate key (`Shipping_id` / `Location_id`) — dropping the rest to avoid duplicating attribute columns already living in the dimension.

### 4. Calculated Date Dimension (DAX)
Created in **Modeling → New Table** (not in Power Query, so it dynamically spans the full order/shipping date range):

```DAX
Dim_Date =
CALENDAR (
    MIN ( Fact_table[order date (DateOrders)] ),
    MAX ( Fact_table[shipping date (DateOrders)] )
)
```

Calculated columns added on `Dim_Date`:

```DAX
Year         = YEAR ( Dim_Date[Date] )
Month Number = MONTH ( Dim_Date[Date] )
Month        = FORMAT ( Dim_Date[Date], "MMMM" )
Quarter      = "Q" & QUARTER ( Dim_Date[Date] )
Week         = WEEKNUM ( Dim_Date[Date] )
Day          = DAY ( Dim_Date[Date] )
Day Name     = FORMAT ( Dim_Date[Date], "dddd" )
```

### 5. Apply & Save
- **Close & Apply** in Power Query Editor to load all transformed tables into the model.
- Save the `.pbix` file frequently throughout the build.

## Data Model Overview

The model follows a **star/snowflake schema** with `Fact_table` at the center, surrounded by conformed dimensions:

- **Fact_table** — order-line grain: sales, product price, product status, sales per customer, shipping date, shipping mode, and foreign keys to all dimensions.
- **Dim_Customer** — one row per customer (name, location, segment).
- **Dim_Product** — one row per product card (price, status, links to Category/Department).
- **Dim_Category** — one row per product category.
- **Dim_Department** — one row per department.
- **Dim_Shipping** — one row per unique shipping profile (scheduled vs. real shipping days, delivery status, late delivery risk), keyed by a surrogate `Shipping_id`.
- **Dim_Location** — one row per unique market/order geography combination, keyed by a surrogate `Location_id`.
- **Dim_Date** — a continuous calendar table spanning order and shipping dates, used for time intelligence.

### Relationships
| From | To | Cardinality | Status |
|---|---|---|---|
| Dim_Customer[Customer Id] | Fact_table[Order Customer Id] | 1 → * | Active |
| Dim_Product[Product Card Id] | Fact_table[Order Item Cardprod Id] | 1 → * | Active |
| Dim_Category[Category Id] | Dim_Product[Category Id] | 1 → * | Active |
| Dim_Department[Department Id] | Dim_Product[Department Id] | 1 → * | Active |
| Dim_Shipping[Shipping_id] | Fact_table[Shipping_id] | 1 → * | Active |
| Dim_Location[Location_id] | Fact_table[Location_id] | 1 → * | Active |
| Dim_Date[Date] | Fact_table[order date (DateOrders)] | 1 → * | **Active** |
| Dim_Date[Date] | Fact_table[shipping date (DateOrders)] | 1 → * | **Inactive** |

Because `Fact_table` has two date columns (order date and shipping date) that both relate to `Dim_Date`, only one relationship can be active at a time. The **order date** relationship is kept active by default; the **shipping date** relationship is kept inactive and activated in specific measures using:

```DAX
Shipping Amount by Ship Date =
CALCULATE (
    SUM ( Fact_table[Sales] ),
    USERELATIONSHIP ( Dim_Date[Date], Fact_table[shipping date (DateOrders)] )
)
```

This snowflaked design (`Dim_Category` and `Dim_Department` linking through `Dim_Product` rather than directly to `Fact_table`) keeps the model normalized, avoids redundant attribute storage, and supports flexible time-based analysis via the dual date relationships.
