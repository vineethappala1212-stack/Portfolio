# Apollo Hospitals – Healthcare Operations & Forecasting Analytics

**📊 Senior Data Analyst (Hyderabad, India) — built an operations analytics suite for patient flow, capacity, and pharmacy inventory. Improved bed utilization by 8 pts, reduced OPD wait time by 17%, and cut pharmacy stockouts by 12%.**  
🏥 Hospital Ops • ⏱ Patient Flow • 🛏 Bed Utilization • 💊 Pharmacy DOH • 🔮 Forecasting • 🧮 SQL • 🐍 Python • 📊 Tableau

**Tags:** `SQL` `Tableau` `Python (Prophet)` `Hospital Operations` `OPD/IPD` `ALOS` `Readmission` `Inventory`

---

### 📌 Role Context
Delivered this as a provider-side analytics project with cross-functional partners in Operations, Nursing Administration, Pharmacy, and Finance. Worked with data from EMR/registration, ADT (admit–discharge–transfer), and pharmacy inventory systems to create a standardized KPI layer and self-service dashboards for daily huddles and monthly performance reviews.

---

### 🎯 Overview
I led analytics delivery for **patient flow**, **capacity planning**, and **pharmacy inventory**. The goals were to:  
1) improve **bed utilization** and **ALOS** (average length of stay),  
2) reduce **OPD wait times** and **readmission risk indicators**, and  
3) ensure **supply continuity** via Days-on-Hand (DOH) and stockout prevention.  
I designed the KPI framework, built SQL models, introduced a Python forecasting workflow for patient volumes, and delivered Tableau dashboards used by hospital leadership and unit managers.

---

### 👤 Role & Scope
- **Role:** Senior Data Analyst — Hospital Operations Analytics  
- **Ownership:** KPI definition → data modeling → SQL marts → forecasting (Python) → Tableau dashboards → training  
- **Stakeholders:** COO’s office, Nursing Admin, Pharmacy, OPD/ED managers, Finance  
- **Focus Areas:** OPD/IPD volumes, ALOS, BOR (Bed Occupancy Rate), readmission (30-day), OR utilization, pharmacy DOH/stockouts

---

### 🔢 KPI Summary
| KPI | Definition |
|---|---|
| **OPD Wait Time (median)** | Check-in → Physician start |
| **ALOS** | Average inpatient length of stay (discharge_date − admit_date) |
| **Bed Occupancy Rate (BOR)** | Inpatient bed days used / bed days available |
| **30-Day Readmission %** | % discharges readmitted within 30 days |
| **OR Utilization %** | Actual OR time / scheduled OR time |
| **Pharmacy DOH** | On-hand quantity / avg daily usage |
| **Stockout Rate** | % item-days with zero stock |

---

### 🧩 Business Problem
Leadership had siloed reports for OPD, IPD, and pharmacy, making it difficult to spot bottlenecks and plan capacity. Queue times were inconsistent across departments; bed occupancy and ALOS weren’t monitored with a single definition; and pharmacy teams lacked forward visibility on high-use items, causing stockouts and expedited purchases.

---

### ✅ Solution Approach
1. **KPI Governance:** Standardized definitions for ALOS/BOR, OPD wait time, readmission, and DOH; aligned with unit managers.  
2. **Data Modeling:** Unified **Registration/OPD**, **ADT (IPD)**, **Pharmacy** into conformed dimensions (Date, Department, Ward, Item).  
3. **SQL Marts:** Built curated views for **patient flow**, **capacity**, and **inventory** with daily granularity and rolling windows.  
4. **Forecasting (Python/Prophet):** Predicted OPD volumes per department to plan staffing and room allocations.  
5. **Dashboards (Tableau):** Executive overview + drilldowns for OPD, IPD, OR, and Pharmacy; daily huddle tiles and monthly trend views.  
6. **Adoption:** Trained unit heads; embedded a “next actions” panel (e.g., high-risk wards, stockout watchlist).

---

### 🏗️ Architecture
```text
Source Systems (Registration/OPD, ADT/IPD, OR, Pharmacy)
          │
          ▼
Staging Tables (opd_visits, ipd_stays, or_schedules, pharm_txn, pharm_onhand)
          │
          ▼
SQL Models / Marts
   ├─ mart_patient_flow (OPD wait, arrivals, dept trends)
   ├─ mart_capacity (ALOS, BOR, readmission-30d, ward KPI)
   └─ mart_pharmacy (DOH, stockouts, usage velocity)
          │
          ▼
Forecast Layer (Python → forecast_opd_by_dept)
          │
          ▼
Tableau Dashboards (Ops Overview, OPD, IPD/Capacity, OR, Pharmacy)
```

---

### 🧪 Advanced SQL — ALOS & BOR (Ward × Day)

**Goal:** Produce ward-level ALOS and Bed Occupancy Rate by day for capacity monitoring.
-- ipd_stays: stay_id, patient_id, ward_id, admit_dt, discharge_dt, beds_available (by ward/day)
```sql
WITH daily_bed_days AS (
  SELECT
    ward_id,
    CAST(d AS DATE) AS day_date,
    COUNT(*) AS patient_bed_days
  FROM (
    -- expand each stay across calendar days
    SELECT ward_id, GENERATE_DATE_ARRAY(CAST(admit_dt AS DATE), CAST(discharge_dt AS DATE)) AS d_arr
    FROM ipd_stays
  ), LATERAL FLATTEN(input => d_arr) f,
  LATERAL (SELECT f.value::date AS d) v
  GROUP BY ward_id, day_date
),

bed_supply AS (
  SELECT ward_id, day_date, MAX(beds_available) AS beds_available
  FROM ward_bed_supply  -- one row per ward/day with available beds
  GROUP BY ward_id, day_date
),

alos_by_discharge AS (
  SELECT
    ward_id,
    CAST(discharge_dt AS DATE) AS day_date,
    AVG(DATEDIFF('day', CAST(admit_dt AS DATE), CAST(discharge_dt AS DATE))) AS alos_days
  FROM ipd_stays
  GROUP BY ward_id, day_date
)

SELECT
  s.ward_id,
  s.day_date,
  COALESCE(b.patient_bed_days, 0) AS patient_bed_days,
  s.beds_available,
  ROUND(100.0 * COALESCE(b.patient_bed_days, 0) / NULLIF(s.beds_available, 0), 1) AS bor_pct,
  COALESCE(a.alos_days, 0) AS alos_days
FROM bed_supply s
LEFT JOIN daily_bed_days b USING (ward_id, day_date)
LEFT JOIN alos_by_discharge a USING (ward_id, day_date)
ORDER BY s.day_date, s.ward_id;
```

---

**How I used this:** Drove Capacity tiles (BOR, ALOS) and a ward heatmap with alerts when BOR > 85% or ALOS

---

### 🧪 Advanced SQL — ALOS & BOR (Ward × Day)

**Goal:** Produce ward-level ALOS and Bed Occupancy Rate by day for capacity monitoring.
-- ipd_stays: stay_id, patient_id, ward_id, admit_dt, discharge_dt, beds_available (by ward/day)
```sql
WITH daily_bed_days AS (
  SELECT
    ward_id,
    CAST(d AS DATE) AS day_date,
    COUNT(*) AS patient_bed_days
  FROM (
    -- expand each stay across calendar days
    SELECT ward_id, GENERATE_DATE_ARRAY(CAST(admit_dt AS DATE), CAST(discharge_dt AS DATE)) AS d_arr
    FROM ipd_stays
  ), LATERAL FLATTEN(input => d_arr) f,
  LATERAL (SELECT f.value::date AS d) v
  GROUP BY ward_id, day_date
),

bed_supply AS (
  SELECT ward_id, day_date, MAX(beds_available) AS beds_available
  FROM ward_bed_supply  -- one row per ward/day with available beds
  GROUP BY ward_id, day_date
),

alos_by_discharge AS (
  SELECT
    ward_id,
    CAST(discharge_dt AS DATE) AS day_date,
    AVG(DATEDIFF('day', CAST(admit_dt AS DATE), CAST(discharge_dt AS DATE))) AS alos_days
  FROM ipd_stays
  GROUP BY ward_id, day_date
)

SELECT
  s.ward_id,
  s.day_date,
  COALESCE(b.patient_bed_days, 0) AS patient_bed_days,
  s.beds_available,
  ROUND(100.0 * COALESCE(b.patient_bed_days, 0) / NULLIF(s.beds_available, 0), 1) AS bor_pct,
  COALESCE(a.alos_days, 0) AS alos_days
FROM bed_supply s
LEFT JOIN daily_bed_days b USING (ward_id, day_date)
LEFT JOIN alos_by_discharge a USING (ward_id, day_date)
ORDER BY s.day_date, s.ward_id;
```

---

**How I used this:** Powered OPD panel with Median Wait, Visits, and a “busiest hours” line chart for staffing.

---

### 🧪 Advanced SQL — Pharmacy DOH & Stockout Flag

**Goal:** Compute Days on Hand and stockout flags by item/day.
-- pharm_onhand: item_id, snap_date, onhand_qty
-- pharm_txn: item_id, txn_date, qty_out (dispensed), qty_in (receipts)
```sql
WITH usage_7d AS (
  SELECT
    t.item_id,
    CAST(t.txn_date AS DATE) AS day_date,
    AVG(GREATEST(t.qty_out, 0)) OVER (
      PARTITION BY t.item_id
      ORDER BY CAST(t.txn_date AS DATE)
      ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS avg_daily_out_7d
  FROM pharm_txn t
),
onhand AS (
  SELECT item_id, CAST(snap_date AS DATE) AS day_date, MAX(onhand_qty) AS onhand_qty
  FROM pharm_onhand
  GROUP BY item_id, CAST(snap_date AS DATE)
)
SELECT
  o.item_id,
  o.day_date,
  o.onhand_qty,
  ROUND(o.onhand_qty / NULLIF(u.avg_daily_out_7d, 0), 1) AS doh,
  CASE WHEN o.onhand_qty <= 0 THEN 1 ELSE 0 END AS stockout_flag
FROM onhand o
LEFT JOIN usage_7d u USING (item_id, day_date)
ORDER BY o.day_date, o.item_id;
```

---

**How I used this:** Built Pharmacy watchlist tiles (DOH threshold < 5 days, stockout flag) + item drilldowns.

---

### 🐍 Python (Prophet) — OPD Volume Forecast (Dept)

**Goal:** Forecast next 8–12 weeks of OPD visits by department to plan rosters and rooms.
```python
# requirements: prophet, pandas
import pandas as pd
from prophet import Prophet

# visits_df columns: ['date','dept_id','visits'] (daily)
def dept_forecast(df, periods=12):
    df = df.rename(columns={'date':'ds', 'visits':'y'}).sort_values('ds')
    m = Prophet(weekly_seasonality=True, yearly_seasonality=True, changepoint_prior_scale=0.4)
    m.fit(df[['ds','y']])
    future = m.make_future_dataframe(periods=periods, freq='W')
    fcst = m.predict(future)[['ds','yhat','yhat_lower','yhat_upper']]
    return fcst.tail(periods)

# example for one department
cardio = visits_df[visits_df['dept_id'] == 'CARD']  # cardiology
cardio_fcst = dept_forecast(cardio, periods=12)

# persist to forecast table (pseudo)
# write_to_db(cardio_fcst.assign(dept_id='CARD'))
```

---

**How I used this:** Published forecasts and blended with current capacity → weekly over/under capacity indicator + staffing suggestions.

---

### 📈 Results & Business Impact

- +8 pts improvement in Bed Occupancy Rate (BOR) in high-demand wards

- −17% OPD wait time (median) via peak-hour staffing & triage improvements

− -12% pharmacy stockouts (DOH watchlist + reorder triggers)

- Faster morning huddles with one consolidated dashboard (OPD, IPD, Pharmacy)

---

### 🧠 Challenges & How I Solved Them
| Challenge                              | Solution                                                                             |
| -------------------------------------- | ------------------------------------------------------------------------------------ |
| Inconsistent timestamps across systems | Standardized to hospital local time; filled missing triage times with clinical rules |
| Varying ward bed counts over time      | Daily bed-supply table with effective-dated changes                                  |
| Inventory spikes (campaign days)       | 7-day rolling usage + manual overrides for known events                              |
| Adoption & actionability               | Added red/amber/green thresholds and “Next actions” notes on each panel              |

---

### 🔗 Artifacts & Links
| Item                    | Path/Link                        |
| ----------------------- | -------------------------------- |
| Tableau – Ops Overview  | *Coming soon (public link)*      |
| SQL – Capacity/ALOS/BOR | `/sql/capacity_alos_bor.sql`     |
| SQL – OPD Wait & Volume | `/sql/opd_wait_and_volume.sql`   |
| SQL – Pharmacy DOH      | `/sql/pharmacy_doh.sql`          |
| Python – OPD Forecast   | `/python/opd_volume_forecast.py` |
| Data Model Diagram      | `/docs/hospital_ops_model.png`   |
| Dashboard Screenshots   | `/images/ops-overview.png`       |

---

### ✅ What I’d Improve Next

- Add ED (Emergency) surge forecasting using hourly patterns

- Introduce OR block utilization analytics with turnover time

- Parameterize DOH thresholds by item criticality class (A/B/C)

- Add a readmission root-cause panel (diagnosis, LOS, discharge disposition)

---

✅ This project demonstrates provider-side operations analytics: clear KPI governance, robust SQL modeling, pragmatic forecasting, and dashboards that drive day-to-day decisions.




