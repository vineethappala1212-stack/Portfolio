# 7-Eleven – Inventory Optimization & Promotion Profitability Analytics

**Tags:** `SQL` `Tableau` `ETL (SSIS)` `SQL Server` `Snowflake` `Python (Prophet)` `Forecasting` `Retail Analytics`

### 🎯 Overview
I led an inventory optimization initiative for a U.S. retail client from the 7-Eleven Hyderabad team, partnering with U.S. merchandising, supply chain, and finance stakeholders. The goal was to reduce stockouts and excess inventory while improving promotion profitability. I built KPIs, data models, and dashboards in Tableau, and introduced a Python forecasting workflow to improve demand planning at the category-store level.

---

### 👤 Role & Scope
- **Role:** Senior Data Analyst 
- **Location:** Hyderabad, India (Remote collaboration with U.S. stakeholders)
- **Core contributions:** KPI design, SQL data modeling, ETL coordination (SSIS), Tableau dashboards, Python forecasting (Prophet), decision enablement for category managers.

---

### 🧩 Business Problem
Store teams lacked a consistent view of **stockouts, excess inventory, and promo uplift vs. margin**, leading to lost sales and high carrying costs. Replenishment decisions were reactive, and promotion planning didn’t consider demand volatility by region or store cluster.

---

### 🔧 Tech Stack
**SQL Server, SSIS (ETL), Snowflake (DWH), Tableau (dashboards), Python (Prophet), Excel (business inputs)**

---

### 🔄 Solution Approach
1. **Defined KPIs** with stakeholders: Stockout Rate, Inventory Days on Hand (DOH), Sell-Through %, Promo Uplift %, Promo Margin %, Forecast Accuracy (MAPE).
2. **Built a star schema** (FactInventory, FactSales, FactPromo + DimStore, DimProduct, DimDate) and curated SQL views for Tableau.
3. **Designed Tableau dashboards** for store/category insights, highlighting stock risk, excess flags, and promo profitability by cluster.
4. **Implemented Python forecasting (Prophet)** for weekly demand by category-store; exposed forecasts to Tableau for guided replenishment.
5. **Automated data pipelines** via SSIS → Snowflake; scheduled refresh with data quality checks and exception logging.

---

### 📈 KPIs & Insights Delivered
- **Stockout Rate**, **Excess Inventory %**, **DOH**, **Sell-Through %**
- **Promo Uplift %** and **Promo Margin %** (uplift vs. baseline)
- **Forecast Accuracy (MAPE)** by category-store
- **At-risk items** (low on-hand + high forecast) and **over-stocked items** (high on-hand + low forecast)

---

### 🚀 Business Impact
- **−18% stockouts** across pilot store clusters (8 weeks)  
- **−11% excess inventory** (DOH reduction) through targeted pull-downs  
- **+7% promo margin** by pruning low-margin promotions and optimizing discount depth  
- Faster weekly replanning cycles with forecast visibility in Tableau

---

### 🏗️ Architecture (high level)
```text
POS / Inventory / Promo Feeds
           │
           ▼
[SSIS ETL: standardize + validate]
           │
           ▼
Snowflake (staging → star schema)
   ├─ FactSales / FactInventory / FactPromo
   └─ DimStore / DimProduct / DimDate
           │
           ├─ Curated SQL Views (KPI, DOH, Uplift, Margin)
           └─ Forecast Table (Python outputs)
           │
           ▼
Tableau Dashboards (Inventory & Profitability Insights)
```

---

### 🧪 Advanced SQL — DOH, Stockout & Promo Margin (Store-Category)

**Goal:** Produce dashboard-ready KPIs for Days on Hand, Stockout Flag, and Promo Margin % at Store × Category level (last 8 weeks).

```sql
WITH inv AS (
    SELECT
        i.StoreID,
        p.Category,
        CAST(i.SnapDate AS DATE) AS SnapDate,
        SUM(i.OnHandQty)       AS OnHandQty
    FROM FactInventory i
    JOIN DimProduct p  ON p.ProductID = i.ProductID
    WHERE i.SnapDate >= DATEADD(week, -8, CAST(GETDATE() AS DATE))
    GROUP BY i.StoreID, p.Category, CAST(i.SnapDate AS DATE)
),
sales AS (
    SELECT
        s.StoreID,
        p.Category,
        CAST(s.SaleDate AS DATE) AS SaleDate,
        SUM(s.Qty)               AS UnitsSold,
        SUM(s.NetSalesAmt)       AS NetSales
    FROM FactSales s
    JOIN DimProduct p ON p.ProductID = s.ProductID
    WHERE s.SaleDate >= DATEADD(week, -8, CAST(GETDATE() AS DATE))
    GROUP BY s.StoreID, p.Category, CAST(s.SaleDate AS DATE)
),
daily AS (
    SELECT
        COALESCE(i.StoreID, s.StoreID)    AS StoreID,
        COALESCE(i.Category, s.Category)  AS Category,
        COALESCE(i.SnapDate, s.SaleDate)  AS AsOfDate,
        COALESCE(i.OnHandQty, 0)          AS OnHandQty,
        COALESCE(s.UnitsSold, 0)          AS UnitsSold,
        COALESCE(s.NetSales, 0)           AS NetSales
    FROM inv i
    FULL OUTER JOIN sales s
      ON i.StoreID = s.StoreID
     AND i.Category = s.Category
     AND CAST(i.SnapDate AS DATE) = s.SaleDate
),
rolling_usage AS (
    SELECT
        StoreID,
        Category,
        AsOfDate,
        UnitsSold,
        AVG(UnitsSold * 1.0) OVER (
            PARTITION BY StoreID, Category
            ORDER BY AsOfDate
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS Avg7D_Units
    FROM daily
),
promo AS (
    SELECT
        pr.StoreID,
        p.Category,
        CAST(pr.PromoDate AS DATE) AS PromoDate,
        SUM(pr.UpliftUnits)        AS UpliftUnits,
        SUM(pr.PromoRevenue - pr.PromoCost) AS PromoMargin
    FROM FactPromo pr
    JOIN DimProduct p ON p.ProductID = pr.ProductID
    WHERE pr.PromoDate >= DATEADD(week, -8, CAST(GETDATE() AS DATE))
    GROUP BY pr.StoreID, p.Category, CAST(pr.PromoDate AS DATE)
),
final AS (
    SELECT
        r.StoreID,
        r.Category,
        r.AsOfDate,
        r.Avg7D_Units,
        d.OnHandQty,
        CASE WHEN d.OnHandQty <= 0 THEN 1 ELSE 0 END AS StockoutFlag,
        SUM(p.PromoMargin) OVER (
            PARTITION BY r.StoreID, r.Category
            ORDER BY r.AsOfDate
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS PromoMargin7D
    FROM rolling_usage r
    JOIN daily d
      ON d.StoreID = r.StoreID
     AND d.Category = r.Category
     AND d.AsOfDate = r.AsOfDate
    LEFT JOIN promo p
      ON p.StoreID = r.StoreID
     AND p.Category = r.Category
     AND p.PromoDate = r.AsOfDate
)
SELECT
    StoreID,
    Category,
    AsOfDate,
    OnHandQty,
    Avg7D_Units,
    CASE WHEN Avg7D_Units > 0
         THEN ROUND(OnHandQty / NULLIF(Avg7D_Units,0), 2)
         ELSE NULL
    END AS DOH,                       -- Days on Hand
    StockoutFlag,
    ROUND(COALESCE(PromoMargin7D, 0), 2) AS PromoMargin7D
FROM final
ORDER BY StoreID, Category, AsOfDate;
```
---

### How I used this:

DOH flagged items with low coverage; StockoutFlag highlighted zero on-hand days.

PromoMargin7D identified promotions with sales uplift but poor margin, guiding discount depth and item selection.

These metrics fed Tableau tiles and store/category drill-downs for weekly reviews.

### 🐍 Python (Prophet) — Weekly Demand Forecast (Category × Store)

**Goal:** Forecast next 8–12 weeks of demand to guide replenishment and safety stock.

```python
# requirements: prophet, pandas
import pandas as pd
from prophet import Prophet

# sales_df: columns = ['AsOfDate','StoreID','Category','UnitsSold'] (daily)
hist = (
    sales_df
    .groupby(['StoreID', 'Category', 'AsOfDate'], as_index=False)['UnitsSold']
    .sum()
)

def forecast_store_category(df, periods=12):
    df = df.rename(columns={'AsOfDate': 'ds', 'UnitsSold': 'y'}).sort_values('ds')
    model = Prophet(weekly_seasonality=True, yearly_seasonality=True, changepoint_prior_scale=0.5)
    model.fit(df[['ds', 'y']])
    future = model.make_future_dataframe(periods=periods, freq='W')
    fcst = model.predict(future)[['ds', 'yhat', 'yhat_lower', 'yhat_upper']]
    return fcst.tail(periods)

# Example: one store-category
sample = hist[(hist['StoreID'] == 101) & (hist['Category'] == 'Beverages')]
forecast_output = forecast_store_category(sample, periods=12)

# (Pseudo) Write forecast results to Snowflake table for Tableau use
# write_to_snowflake(forecast_output.assign(StoreID=101, Category='Beverages'))
```



### How I used this:
I published Prophet forecasts to a Snowflake Forecast table and blended them with DOH and stock levels in Tableau to:

✅ Identify at-risk items (high forecast ✔ low stock)

✅ Highlight overstocked SKUs (low forecast ✔ high stock)

✅ Improve replenishment decisions before stockouts happen

---

### 🧠 Challenges & How I Solved Them

- Noisy weekly seasonality & promo spikes: Smoothed with rolling windows; tuned Prophet seasonalities.

- Mixed data quality across stores: Standardized item codes; enforced DQ checks in SSIS; imputed gaps.

- Adoption with field teams: Added red/amber/green thresholds and store filters; ran enablement sessions.

---


### 🔗 Artifacts & Links
| Item                   | Link/Path                             |
| ---------------------- | ------------------------------------- |
| Tableau Dashboard      | Coming soon (public link)             |
| SQL Script             | `/sql/inventory_doh_promo_margin.sql` |
| Python Forecast Script | `/python/prophet_demand_forecast.py`  |
| ETL Notes (SSIS)       | `/etl/notes.md`                       |
| Data Model Diagram     | `/docs/inventory-star-schema.png`     |


---

### ✅ What I’d Improve Next

Add store clustering to tune forecasts by similar demand patterns

Introduce service-level based safety stock (target fill rate)

Expand to price-elasticity signals to support markdown optimization
