# Resales Analytics Data Warehouse - Technical Documentation

## Project Overview

**Project Name:** `Retail Sales Analytics`  
**Business Domain:** Retail Electronics Sales Analytics  
**Objective:** Automate sales reporting, inventory management, and customer analytics using a modern data warehouse

**Business Problem:**  
Manual Excel-based reporting was causing errors and slow decision-making. This project builds an automated data warehouse to track daily sales, inventory health, and customer loyalty across 5 retail stores.

---

## Architecture: Medallion Pattern

We implemented a **3-layer medallion architecture** to ensure data quality and separation of concerns:

```
Azure Blob Storage (Source)
        ↓
   BRONZE LAYER
   (Raw Data - No Transformation)
        ↓
   SILVER LAYER  
   (Cleaned & Standardized)
        ↓
   GOLD LAYER
   (Star Schema - Business Ready)
        ↓
   BI Dashboards & Reports
```

### Why This Approach?

When we tried building reports directly from raw data, every source data quality issue broke our dashboards. The medallion pattern solved this:

* **Bronze** - Accept everything as-is from the source. No transformations.
* **Silver** - Apply data quality rules. Fix dates, standardize IDs, handle nulls.
* **Gold** - Build dimensional model with business logic. Ready for analytics.

---

## Data Catalog Structure

**Unity Catalog:** `de_mini_project`

### Schemas

| Schema | Purpose | Table Count |
|--------|---------|-------------|
| `azure_blob_storage` | Bronze layer - Raw data from source | 6 tables |
| `silver` | Silver layer - Cleaned data | 6 tables |
| `gold` | Gold layer - Dimensional model | 5 tables |

**Total:** 17 managed Delta tables

---

## Bronze Layer: Raw Data (`azure_blob_storage` schema)

**Purpose:** Landing zone for raw data from Azure Blob Storage, synced via Fivetran

**Tables:**

| Table | Description | Key Source Columns |
|-------|-------------|--------------------|
| `customers` | Customer master data | `_cust_ref_id_`, `full_name`, `gender_code`, `joined_date_` |
| `sales` | Transaction records | `trxn_id_`, `_date_ref_`, `prod_code_id`, `store_loc_id`, `qty_sold`, `_unit_price_` |
| `products` | Product catalog | Product codes, names, pricing, categories |
| `inventory` | Stock levels | Product quantities by store |
| `stores` | Store locations | `st_id_`, `location_name_address`, `city_region` |
| `transactions` | Return transactions | `orgnl_trxn_id_`, `rtn_id_no`, `rtn_date_string`, `rsn_code_` |

**Data Ingestion:**
* Source: Azure Blob Storage (CSV files)
* Tool: Fivetran (nightly sync at 2 AM)
* Metadata: All tables include `_file`, `_line`, `_modified`, `_fivetran_synced` for lineage tracking

**Important:** Column names in Bronze match the source system exactly, including underscores and special characters. No renaming happens here.

---

## Silver Layer: Cleaned Data (`silver` schema)

**Purpose:** Data quality and standardization layer

**Tables:**

| Table | Transformations Applied |
|-------|-------------------------|
| `customer` | Parse join dates, validate gender codes (M/F/Unknown), standardize `customer_id` |
| `sales` | Convert date strings to DATE type, rename columns (`trxn_id_` → `transaction_id`), cast numeric types, validate foreign keys |
| `product` | Standardize categories (mapped 15 variants to 5 standard values), clean pricing |
| `inventory` | Validate stock quantities, flag negatives for review |
| `store` | Extract city and region from address text using regex |
| `returns` | Link to original transactions via `original_transaction_id` |

**Key Transformations:**

* **Date Parsing:** Source dates like "20260115" → proper DATE type
* **ID Standardization:** All IDs converted to STRING for consistency
* **Null Handling:** Missing customer IDs → `'UNKNOWN_CUSTOMER'`
* **Type Casting:** Prices → DECIMAL(10,2), Quantities → INT
* **Remove Metadata:** Strip Fivetran audit columns (`_file`, `_line`, etc.)

---

## Gold Layer: Star Schema (`gold` schema)

**Purpose:** Business-ready dimensional model for analytics and reporting

### Dimensional Model

We implemented a **star schema** with 3 dimension tables and 2 fact tables.

#### Dimension Tables

**1. `dim_product` - Product Dimension**
```
product_id (PK)
item_name
category
cost_price
selling_price
profit_per_unit (calculated: selling_price - cost_price)
```

**2. `dim_customer` - Customer Dimension**
```
customer_id (PK)
name
gender
```

**3. `dim_store` - Store Dimension**
```
store_id (PK)
store_name
address
city
region
manager_contact
```

#### Fact Tables

**1. `fact_sales` - Sales Transactions**

**Grain:** One row per transaction (including returned transactions)

```
transaction_id (PK)
transaction_date
product_id (FK → dim_product)
customer_id (FK → dim_customer)
store_id (FK → dim_store)
sold_qty
actual_sold_price
original_marked_price
total_revenue (= sold_qty × actual_sold_price)
discount_loss_amount (= sold_qty × (original_marked_price - actual_sold_price))
is_returned (0 = no, 1 = yes)
```

**Design Decision:** We keep returned transactions with a flag (`is_returned = 1`) instead of deleting them. This allows us to calculate return rates and analyze what gets returned.

**2. `fact_inventory` - Inventory Snapshots**

**Grain:** Product × Store × Audit Date

```
product_id (FK → dim_product)
store_id (FK → dim_store)
stocks_qty
last_audit_date
is_out_of_stock (flag: 1 when stocks_qty = 0)
is_slow_moving (flag: 1 when no sales in last 30 days)
last_sale_date
```

---

## Key Performance Indicators (KPIs)

The data warehouse tracks **10 core business metrics**:

### Sales Performance

**KPI 1: Total Sales**  
Total revenue collected from all products sold (excluding returns)
```sql
SUM(CASE WHEN is_returned = 0 THEN total_revenue ELSE 0 END)
```

**KPI 2: Profit Per Product**  
Profit margin calculated as selling price minus cost price
```sql
SUM(sold_qty * profit_per_unit) WHERE is_returned = 0
```

**KPI 3: Top Selling Category**  
Product category generating the most revenue
```sql
SELECT category, SUM(total_revenue) 
GROUP BY category ORDER BY SUM(total_revenue) DESC
```

**KPI 4: Basket Size**  
Average number of items purchased per transaction
```sql
total_units_sold / total_transactions
```

**KPI 5: Return Rate**  
Percentage of transactions that were returned
```sql
(return_count / total_transactions) × 100
```

### Inventory Management

**KPI 6: Out-of-Stock Count**  
Number of unique products with zero inventory
```sql
COUNT(DISTINCT product_id) WHERE stocks_qty = 0
```

**KPI 10: Slow-Moving Inventory**  
Products with no sales in the last 30 days
```sql
COUNT(DISTINCT product_id) WHERE DATEDIFF(CURRENT_DATE, last_sale_date) > 30
```

### Store & Product Performance

**KPI 7: Store Performance**  
Ranking of stores based on sales volume
```sql
SELECT store_id, SUM(total_revenue)
GROUP BY store_id ORDER BY SUM(total_revenue) DESC
```

**KPI 8: Discount Impact**  
Revenue lost due to discounts (marked price vs. actual sold price)
```sql
SUM(discount_loss_amount)
```

### Customer Analytics

**KPI 9: Repeat Customer Count**  
Number of customers with more than one purchase
```sql
SELECT COUNT(customer_id) 
FROM (SELECT customer_id, COUNT(*) as purchases 
      GROUP BY customer_id HAVING COUNT(*) > 1)
```

---

## ETL Pipeline Workflow

### Project Folder Structure

```
/Users/norizzatharva@gmail.com/resales_analytics/
├── 01_bronze/
│   └── 01_ingest_raw.ipynb           (1 notebook)
├── 02_silver/
│   ├── 01_silver_customer.ipynb
│   ├── 02_silver_inventory.ipynb
│   ├── 03_silver_sales.ipynb
│   ├── 04_silver_product.ipynb
│   ├── 05_silver_store.ipynb
│   └── 06_silver_returns.ipynb       (6 notebooks)
└── 03_gold/
    ├── 01_dim_product.ipynb
    ├── 02_dim_customer.ipynb
    ├── 03_dim_store.ipynb
    ├── 04_fact_sales.ipynb
    ├── 05_fact_inventory.ipynb
    ├── 06_kpi_delivery.ipynb        (8 notebooks)
```

**Total:** 15 ETL notebooks organized by layer

### Pipeline Execution Flow

**Phase 1: Silver Transformation**
* Notebooks: 6 notebooks (one per entity)
* Action: Clean, validate, and standardize data
* Dependencies: Each Silver notebook depends on its corresponding Bronze table

**Phase 2: Gold Dimensional Model**
* Notebooks: 5 notebooks (3 dimensions + 2 facts)
* Action: Build star schema with business logic
* Dependencies: Fact tables require all dimensions to exist first

**Phase 3: KPI Calculation**
* Notebooks: 3 notebooks for KPI reporting
* Action: Calculate business metrics for dashboards
  
---

## How to Query the Data

### For Analysts

**Always start with the Gold layer.** The dimensional model is optimized for analysis.

**Example 1: Total Sales by Month**
```sql
SELECT 
    date_format(transaction_date, 'yyyy-MM') as month,
    SUM(CASE WHEN is_returned = 0 THEN total_revenue ELSE 0 END) as total_sales
FROM de_mini_project.gold.fact_sales
GROUP BY date_format(transaction_date, 'yyyy-MM')
ORDER BY month;
```

**Example 2: Top 10 Products by Profit**
```sql
SELECT 
    p.item_name,
    p.category,
    SUM(s.sold_qty * p.profit_per_unit) as total_profit
FROM de_mini_project.gold.fact_sales s
JOIN de_mini_project.gold.dim_product p ON s.product_id = p.product_id
WHERE s.is_returned = 0
GROUP BY p.item_name, p.category
ORDER BY total_profit DESC
LIMIT 10;
```

**Example 3: Store Performance Ranking**
```sql
SELECT 
    st.store_name,
    st.city,
    st.region,
    SUM(s.total_revenue) as total_revenue,
    COUNT(DISTINCT s.transaction_id) as transactions
FROM de_mini_project.gold.fact_sales s
JOIN de_mini_project.gold.dim_store st ON s.store_id = st.store_id
WHERE s.is_returned = 0
GROUP BY st.store_name, st.city, st.region
ORDER BY total_revenue DESC;
```

**Example 4: Inventory Health Check**
```sql
SELECT 
    p.category,
    COUNT(DISTINCT i.product_id) as total_products,
    SUM(CASE WHEN i.is_out_of_stock = 1 THEN 1 ELSE 0 END) as out_of_stock,
    SUM(CASE WHEN i.is_slow_moving = 1 THEN 1 ELSE 0 END) as slow_moving
FROM de_mini_project.gold.fact_inventory i
JOIN de_mini_project.gold.dim_product p ON i.product_id = p.product_id
GROUP BY p.category;
```

### For BI Tools

**Connection Details:**
* Connect to: `de_mini_project.gold` schema only
* Use the star schema for dimensional slicing
* Fact tables: `fact_sales`, `fact_inventory`
* Dimension tables: `dim_product`, `dim_customer`, `dim_store`

---

## Data Lineage

Full lineage tracking from source to reporting:

```
POS System (Source)
    ↓
Azure Blob Storage (CSV)
    ↓ [Fivetran Sync]
Bronze: azure_blob_storage.* (6 tables)
    ↓ [Data Cleaning]
Silver: silver.* (6 tables)
    ↓ [Dimensional Modeling]
Gold: gold.dim_* (3 tables) + gold.fact_* (2 tables)
    ↓ [KPI Calculation]
Dashboards & Reports
```

**Lineage Tools:**
* Unity Catalog lineage tracking enabled
* Fivetran metadata columns in Bronze for source traceability
* Delta Lake time travel for historical data recovery

---

## Known Issues & Workarounds

**Issue 1: Anonymous Customers**  
Some transactions have `customer_id = 'UNKNOWN_CUSTOMER'` because the POS allows purchases without customer login.  
**Workaround:** Exclude these from customer analytics but include in sales metrics.

**Issue 2: Missing Return Reasons**  
The returns table sometimes has null reason codes.  
**Workaround:** Default to `'UNSPECIFIED'` in Silver transformation.

**Issue 3: Irregular Inventory Audits**  
Not all stores perform daily inventory counts, creating gaps in `fact_inventory`.  
**Workaround:** We're working with operations to standardize the audit schedule. Current data shows actual audit dates.

---

## Maintenance & Monitoring

**Automated Monitoring:**
* Pipeline failures → #data-alerts Slack channel
* Row count validation (alert if drops > 20% day-over-day)
* Data quality checks at each layer

**Manual Tasks:**
* Weekly review of failed transactions
* Monthly audit of slow-moving inventory
* Quarterly review of KPI definitions with business stakeholders

**Support:**
* Email: norizzatharva@gmail.com
* Slack: #data-engineering channel

---

## Project Outcomes

**Before (Manual Excel):**
* Reports took 2-3 days to prepare
* Frequent calculation errors
* No historical trend analysis
* Limited to basic metrics

**After (Automated Data Warehouse):**
* Reports available daily by 8 AM
* Consistent, validated calculations
* 2 years of historical data for trends
* 10 comprehensive KPIs tracked
* Self-service analytics for business users

---

**Documentation Version:** 1.0  
**Last Updated:** March 27, 2026  
**Author:** norizzatharva@gmail.com
