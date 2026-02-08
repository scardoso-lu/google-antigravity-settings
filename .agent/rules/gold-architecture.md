---
trigger: always_on
---

# Gold Layer Agent Rules

**Scope:** Business Aggregation, Dimensional Modeling, & KPI Agents
**Layer Objective:** High-Performance, Governance-Ready, "Boardroom Quality" Data
**Status:** Enforced
**Storage Format:** **Delta Lake** (Star Schema / Data Marts)

---

## 1. The Core Mandate: "One Version of the Truth"

Gold is not for data engineers; it is for the business.

**Rule GL-01: Dimensional Modeling (Kimball)**
We do not simply copy Silver tables to Gold. We restructure them into a **Star Schema** to make querying intuitive and fast.
* **Fact Tables:** Store measurable events (e.g., `fact_orders`, `fact_page_views`).
    * *Characteristics:* Long, narrow, immutable, contains Foreign Keys and Metrics.
* **Dimension Tables:** Store descriptive attributes (e.g., `dim_customers`, `dim_products`).
    * *Characteristics:* Short, wide, slowly changing (SCD), contains Attributes.



[Image of star schema diagram]


**Rule GL-02: Metric Standardization**
Business logic (e.g., "What is Churn?") must be calculated in the Gold Agent, **not** in the BI tool (Tableau/PowerBI).
* *Why:* If logic lives in Tableau, two analysts will calculate "Churn" differently. If it lives in Gold, everyone sees the same number.

---

## 2. Transformation Policies

### Rule GL-03: The "Referential Integrity" Gate
A Fact record must never point to a missing Dimension.
* **Constraint:** Before writing to `fact_orders`, the Agent must verify that the `customer_id` exists in `dim_customers`.
* **Fallback:** If a dimension is missing (Late Arriving Dimension), link the fact to a "Unknown" (-1) dimension key rather than dropping the revenue data.

### Rule GL-04: Aggregation & Pre-Computation
Gold tables should answer the "Big Questions" instantly.
* **Granularity:** Maintain one "Atomic Fact" table (lowest grain) and multiple "Aggregate Fact" tables (rolled up).
* **Example:**
    * `fact_sales_atomic` (Every single line item).
    * `fact_sales_monthly_kpi` (Sum of revenue by Region by Month).

---

## 3. Storage & Performance (Read-Heavy Optimization)

Gold is Read-Optimized (99% Reads, 1% Writes).

### Rule GL-05: Aggressive Z-Ordering
Since analysts filter by Time and Entity, Gold tables must be physically sorted.
* **Command:** `OPTIMIZE fact_orders ZORDER BY (order_date, region_id)`
* **Frequency:** Run immediately after every write job. This can speed up dashboard queries by 50x.

### Rule GL-06: Liquid Clustering (Advanced Delta)
For massive tables (>1TB), use **Liquid Clustering** instead of Partitioning.
* *Why:* Prevents the "Small File Problem" when users query across partition boundaries (e.g., "Show me sales for the last 5 years").

---

## 4. Governance & Access Control

**Rule GL-07: Data Mart Isolation**
Gold is divided into "Marts" to secure sensitive data.
* **Finance Mart:** Access restricted to Finance Team + CFO.
* **Marketing Mart:** Accessible to Marketing Analysts.
* **Public Mart:** High-level KPIs accessible to the whole company.

**Rule GL-08: Row-Level Security (RLS)**
If a table contains multi-tenant data (e.g., multiple regions), enforce security at the view level.
* *Policy:* A generic user querying `gold_sales` should only see rows where `region_id` matches their assigned region.

---

## 5. Technical Implementation Reference

### Python Agent Snippet (Building a Gold Fact Table)
This script reads from Silver, joins with Dimensions, Aggregates, and overwrites the Gold KPI table.

```python
from deltalake import DeltaTable, write_deltalake
import pandas as pd

def gold_build_sales_kpi(silver_orders_path: str, silver_items_path: str):
    """
    Reads Silver Delta tables, Joins, Aggregates, and writes Gold Fact.
    """
    # 1. Read Silver Data (Lazy read if using Spark, eager for Pandas)
    dt_orders = DeltaTable(silver_orders_path).to_pandas()
    dt_items = DeltaTable(silver_items_path).to_pandas()
    
    # 2. Join (Reconstructing the Transaction)
    # Silver is normalized, so we join to get the full picture
    full_sales = pd.merge(
        dt_orders, 
        dt_items, 
        on='order_id', 
        how='inner'
    )
    
    # 3. Apply Business Logic (Rule GL-02)
    # Example: Net Revenue = (Price * Qty) - Discount - Tax
    full_sales['net_revenue'] = (
        (full_sales['unit_price'] * full_sales['quantity']) 
        - full_sales['discount_amount']
    )
    
    # 4. Aggregate (Rule GL-04)
    # Monthly KPI Grain
    full_sales['month_key'] = full_sales['order_date'].dt.to_period('M').astype(str)
    
    gold_kpi = full_sales.groupby(['month_key', 'region_id']).agg({
        'order_id': 'nunique',      # Count of Orders
        'net_revenue': 'sum',       # Total Revenue
        'customer_id': 'nunique'    # Unique Customers
    }).reset_index()
    
    # Rename for Business Clarity
    gold_kpi.columns = ['month', 'region', 'total_orders', 'total_revenue', 'active_customers']
    
    # 5. Write to Gold (Overwrite Strategy)
    # KPIs are small and recalculated daily, so Overwrite is safe and simple.
    write_deltalake(
        "s3://antigravity-gold/sales/fact_monthly_kpi",
        gold_kpi,
        mode="overwrite",
        overwrite_schema=True
    )
    
    # 6. Optimize (Rule GL-05)
    # (Requires Spark or specific Delta client commands, simplified here)
    # spark.sql("OPTIMIZE delta.`s3://...` ZORDER BY (month, region)")

```

---

## 6. Operational Resilience

### Rule GL-09: Dependency Awareness

Gold jobs must **never** run until Silver jobs are confirmed successful.

* **Orchestration:** Use a DAG (Directed Acyclic Graph) in Airflow/Prefect.
* **Trigger:** `Silver_Sales_Completed` -> `Gold_Sales_KPI_Start`.

### Rule GL-10: Schema Locking

Unlike Bronze (which evolves), Gold schemas are a **Contract**.

* **Constraint:** Schema evolution is **disabled** by default.
* **Change Management:** Adding a column to a Gold table requires a Pull Request and a "Backfill" plan (updating history), as it breaks downstream dashboards.

```