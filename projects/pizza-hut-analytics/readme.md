# Sales Performance Analytics Dashboard – Pizza Hut
**Tags:** `SQL` `Tableau` `ETL` `Redshift` `Data Modeling` `Analytics Engineering` `Business Intelligence`

### 🎯 Overview
I designed and delivered an executive sales & customer insights dashboard used by regional leadership to monitor revenue, order mix, coupon performance, and customer repeat behavior across U.S. stores. Working fully remote in the U.S., I partnered directly with marketing, operations, and finance leads to define the KPI framework and build a self-service analytics experience that accelerated weekly decision cycles.

---

### 👤 Role & Scope
- **Role:** Senior Data Analyst (end-to-end ownership)
- **Delivery:** Remote (U.S.) • Cross-functional with Marketing, Ops, Finance, BI
- **Key contributions:** KPI design, data modeling, SQL development, Tableau dashboarding, refresh automation, stakeholder enablement.

---

### 🧩 Business Problem
Leaders lacked a unified, trustworthy view of regional performance, coupon effectiveness, and customer repeat trends. Reports were fragmented, slow to update, and inconsistent across POS sources, which delayed decisions on promotions, assortment, and regional plans.

---

### 🔧 Tech Stack
**Tableau, SQL Server, Amazon Redshift, SSIS, Tableau Prep** (data prep & standardization), **Agile delivery**.

---

### 🔄 Solution Approach
1. **Aligned KPIs with stakeholders** (marketing, ops, finance): revenue, orders, AOV, coupon redemption, new vs. repeat customers, regional growth.
2. **Built a star schema** (FactSales + DimDate, DimStore, DimProduct, DimCustomer) and standardized coupon/promo keys in staging.
3. **Developed SQL views** for the dashboard layer (granular daily facts + curated KPI views).
4. **Published an interactive Tableau workbook** with drilldowns by region/store/product, time sliders, coupon filters, and repeat-customer segmentation.
5. **Automated refresh** with SSIS → Redshift nightly loads; set data quality checks for promo/code consistency.

---

### 📈 KPIs & Insights (examples)
- Revenue, Orders, **AOV**, Discount Impact, Coupon Redemption Rate  
- **New vs. Repeat Customer Mix** (30/60/90-day windows)  
- Region & Store rankings, Product/category contribution  
- Campaign performance vs. baseline  

---

### 🚀 Business Impact
- **+25% faster decision speed** for regional reviews (centralized KPI view).  
- **+12% increase in repeat purchases** after segmentation-led coupon targeting.  
- Executive adoption across multiple regions (weekly & monthly operating rhythms).

---

### 🏗️ Architecture
```text
POS / Source Feeds
      │
      ▼
[SSIS ETL - clean & standardize promo/coupon codes]
      │
      ▼
Amazon Redshift (staging → star schema)
      ├─ FactSales
      ├─ DimDate
      ├─ DimStore
      ├─ DimProduct
      └─ DimCustomer
      │
      ▼
Curated SQL Views (KPI + retention logic)
      │
      ▼
Tableau Dashboards
```
---

### 🧪 Advanced SQL (Analytical) — Repeat Rate / Retention by Month & Region

> **Goal:** Identify repeat customers within a rolling window and compute repeat rate by month/region.  
> **Notes:** Works in SQL Server / Redshift style; adjust date functions as needed.

```sql
WITH base_orders AS (
    SELECT
        fs.InvoiceID,
        fs.OrderDate,
        CAST(fs.OrderDate AS DATE) AS OrderDateOnly,
        fs.StoreID,
        fs.Region,
        fs.CustomerID,
        fs.SalesAmount
    FROM FactSales fs
    WHERE fs.OrderDate >= DATEADD(year, -2, GETDATE())
),
orders_with_prev AS (
    SELECT
        b.*,
        LAG(b.OrderDateOnly) OVER (PARTITION BY b.CustomerID ORDER BY b.OrderDateOnly) AS PrevOrderDate
    FROM base_orders b
),
repeat_flag AS (
    SELECT
        o.*,
        CASE 
            WHEN o.PrevOrderDate IS NOT NULL AND DATEDIFF(day, o.PrevOrderDate, o.OrderDateOnly) <= 90
            THEN 1 ELSE 0
        END AS IsRepeat90
    FROM orders_with_prev o
),
monthly_region AS (
    SELECT
        DATEFROMPARTS(YEAR(OrderDateOnly), MONTH(OrderDateOnly), 1) AS MonthStart,
        Region,
        COUNT(DISTINCT CustomerID) AS CustomersMonthly,
        COUNT(DISTINCT CASE WHEN IsRepeat90 = 1 THEN CustomerID END) AS RepeatCustomers90
    FROM repeat_flag
    GROUP BY DATEFROMPARTS(YEAR(OrderDateOnly), MONTH(OrderDateOnly), 1), Region
)
SELECT
    MonthStart,
    Region,
    CustomersMonthly,
    RepeatCustomers90,
    (RepeatCustomers90 * 1.0 / CustomersMonthly) AS RepeatRate90
FROM monthly_region
ORDER BY MonthStart, Region;
```
---

**How I used this:**  
I used the `RepeatRate90` metric to build a retention analytics view in Tableau by Region and Month. It helped business leaders identify high-value regions, improve customer loyalty campaigns, and measure coupon impact with data instead of assumptions.

---

### 🧠 Challenges & How I Solved Them
- **Inconsistent coupon codes from multiple POS systems** → Built a master promotion mapping table + cleaned codes using TRIM/UPPER logic in ETL  
- **No customer key across source systems** → Implemented hashed identity stitching logic (email/phone) with confidence scoring  
- **Slow dashboard refresh during month-end** → Added incremental ETL loads, optimized SQL views with indexing, and used materialized views for performance

---

### 🔗 Artifacts & Links
| Item | Link/Path |
|------|-----------|
| Tableau Dashboard |[ Coming soon (public link)](https://public.tableau.com/app/profile/vineeth.appala1100/viz/Book1_17617600957950/Dashboard2?publish=yes) |
| SQL Scripts | `/sql/repeat-rate-analysis.sql` |
| ETL Workflow | `/etl/sssis-pipeline-notes.md` |
| Data Model Diagram | `/docs/star-schema.png` |

---

### 📊 Key Insights
🟢 Sales rose by **18% in Q3**, driven primarily by online delivery orders.  
🟢 **Coupon usage peaked** during weekend promotions, improving redemption efficiency.  
🟢 **Texas and California contributed 42%** of total U.S. sales.  
🟢 **Classic pizzas** remain the top-selling category across regions.  
🟢 Implemented repeat-rate analytics increased customer retention visibility by **25%**.

---

### ✅ What I’d Improve Next
- Add **30/60/90 retention parameter** in Tableau using dynamic parameters  
- Build **customer churn signals** and alert logic  
- Integrate **Python retention forecasting model** to improve targeting  

---


