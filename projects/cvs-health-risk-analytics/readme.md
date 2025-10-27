# CVS Health – Claims Quality, Denials & Cost Analytics

**📊 Senior Data Analyst (Hyderabad offshore delivery to U.S. stakeholders) — reduced denials by 14%, increased first-pass claim rate by 9%, and improved audit transparency across medical & pharmacy claims.**  
🧾 Claims Quality • 🚫 Denial Reduction • 💊 Pharmacy Cost • 🩺 HL7 FHIR • 🧮 SQL/ETL (dbt + Snowflake) • 📊 Tableau

**Tags:** `SQL` `Snowflake` `dbt` `Tableau` `Python (Pandas)` `HL7 FHIR` `ICD-10` `CPT/HCPCS` `Claims Analytics`

---

### 📌 Role Context
Delivered this as part of an enterprise analytics program for CVS Health via the Hyderabad delivery team, collaborating remotely with U.S. claims operations, provider relations, and pharmacy benefit management (PBM) teams.

---

### 🎯 Overview
I led analytics for claims quality, denial reduction, and pharmacy cost visibility across medical and Rx claims. The objective was to (1) reduce preventable denials and rework, (2) improve first-pass adjudication rate, and (3) provide transparent audit trails for compliance. I designed claim KPIs, standardized coding logic (ICD-10, CPT/HCPCS), built a dbt/Snowflake analytics layer, and delivered interactive Tableau dashboards used by operations and finance.

---

### 👤 Role & Scope
- **Role:** Senior Data Analyst — Claims & PBM Analytics  
- **Ownership:** KPI framework → data modeling (FHIR-aware) → dbt ELT → SQL marts → Tableau dashboards → enablement  
- **Collaboration:** Claims Ops, Provider Relations, PBM Pricing, Finance, Compliance (HIPAA-aligned workflows)  
- **Focus Areas:** Denials, first-pass rate, billed vs paid variance, provider accuracy, high-cost NDC tracking

---

### 🔢 KPI Summary
| KPI | Definition |
|---|---|
| **Denial Rate** | Denied claims / total submitted |
| **First-Pass Rate (FPR)** | % claims paid without corrections or pends |
| **Appeal Win Rate** | % overturned denials |
| **Billed vs Paid Variance** | Billed minus paid (allowed & paid trend) |
| **Top Denial Reasons** | Grouped by ANSI reason codes |
| **High-Cost Drug Watchlist** | NDC cost & utilization flags |
| **Provider Edit Hit-Rate** | % claims triggering pre-adjudication edits |

---

### 🧩 Business Problem
Claims teams faced high denial volumes and rework due to coding variance (ICD-10, CPT/HCPCS), missing documentation, and provider submission errors. Pharmacy benefit teams lacked a unified view of high-cost NDC utilization vs formulary alternatives. Leadership needed a single source of truth to reduce preventable denials, trim cost leakage, and improve provider performance.

---

### ✅ Solution Approach
1. **KPI Design & Standardization** — codified denial taxonomy (ANSI codes), cost buckets, and FPR thresholds.  
2. **FHIR-aware Data Modeling** — unified **Member**, **Provider**, **Claim**, **Line**, **Diagnosis (ICD-10)**, **Procedure (CPT/HCPCS)** with FHIR mappings for extensibility.  
3. **dbt + Snowflake ELT** — source staging → conformed models → **marts_claims**, **marts_pharmacy** with tests (unique, not null, referential).  
4. **Rule-Ready SQL Views** — denial reason rollups, billed vs paid waterfalls, provider edit hit-rate, NDC cost monitoring.  
5. **Tableau Dashboards** — Ops overview, denial drilldowns, provider scorecards, PBM cost view with alternative drug flags.  
6. **Governance & HIPAA** — PHI masking at view level, row-level security by region/provider, audit columns (created_by/changed_by job IDs).

---

### 🏗️ Architecture
```text
Source Feeds (Medical Claims, PBM/Rx, Provider, Member, Code Sets)
           │
           ▼
Snowflake Staging (raw_* tables)  ← ingestion (secure file / connectors)
           │
           ▼
dbt Models (stg_* → int_* → dim_*/fact_* → marts_*)
    ├─ dim_member / dim_provider / dim_diagnosis(ICD10) / dim_procedure(CPT_HCPCS) / dim_ndc
    ├─ fact_claim_header / fact_claim_line
    ├─ marts_claims (denial_rate, fpr, reason_rollups, billed_vs_paid)
    └─ marts_pharmacy (ndc_cost, utilization, formulary_flags)
           │
           ▼
Curated SQL Views (row-level security, PHI masking, audit columns)
           │
           ▼
Tableau Dashboards (Ops Overview, Denials, Provider Scorecards, PBM Cost)
```
---

### 🧪 Advanced SQL — Denial Rate, First-Pass, and Top Reasons (Snowflake)

**Goal:** Produce dashboard-ready metrics by month and provider with reason code rollups.
```sql
-- 1) Normalize reasons at line level
WITH line_norm AS (
  SELECT
    clm.claim_id,
    clm.provider_id,
    clm.member_id,
    clm.service_date,
    clm.paid_date,
    clm.billed_amt,
    clm.paid_amt,
    clm.denied_flag,
    COALESCE(NULLIF(TRIM(clm.denial_reason_code), ''), 'UNK') AS denial_reason_code
  FROM fact_claim_line clm
  WHERE clm.service_date >= DATEADD(month, -12, CURRENT_DATE())
),

-- 2) Roll up reason codes (map ANSI to business buckets)
reason_rollup AS (
  SELECT
    denial_reason_code,
    CASE
      WHEN denial_reason_code IN ('CO-50','PR-50') THEN 'Medical Necessity'
      WHEN denial_reason_code IN ('CO-97','PR-97') THEN 'Bundled/Global'
      WHEN denial_reason_code IN ('CO-16','PR-16') THEN 'Missing/Invalid Info'
      WHEN denial_reason_code IN ('CO-109','PR-109') THEN 'Claim Not Covered'
      WHEN denial_reason_code = 'UNK' THEN 'Uncategorized'
      ELSE 'Other'
    END AS denial_bucket
  FROM (SELECT DISTINCT denial_reason_code FROM line_norm)
),

-- 3) Monthly provider metrics
monthly AS (
  SELECT
    DATE_TRUNC('month', service_date) AS month_start,
    l.provider_id,
    COUNT(DISTINCT l.claim_id)              AS claims_submitted,
    COUNT(DISTINCT CASE WHEN l.denied_flag = 1 THEN l.claim_id END) AS claims_denied,
    COUNT(DISTINCT CASE WHEN l.denied_flag = 0 AND l.paid_amt > 0 THEN l.claim_id END) AS claims_paid_first_pass,
    SUM(l.billed_amt) AS billed_amt,
    SUM(l.paid_amt)   AS paid_amt
  FROM line_norm l
  GROUP BY 1,2
),

-- 4) Denial reasons by month/provider
monthly_reasons AS (
  SELECT
    DATE_TRUNC('month', l.service_date) AS month_start,
    l.provider_id,
    r.denial_bucket,
    COUNT(DISTINCT l.claim_id) AS claims_in_bucket
  FROM line_norm l
  JOIN reason_rollup r USING (denial_reason_code)
  WHERE l.denied_flag = 1
  GROUP BY 1,2,3
)

SELECT
  m.month_start,
  m.provider_id,
  m.claims_submitted,
  m.claims_denied,
  m.claims_paid_first_pass,
  ROUND(100.0 * m.claims_denied / NULLIF(m.claims_submitted,0), 2) AS denial_rate_pct,
  ROUND(100.0 * m.claims_paid_first_pass / NULLIF(m.claims_submitted,0), 2) AS first_pass_rate_pct,
  m.billed_amt,
  m.paid_amt,
  (m.billed_amt - m.paid_amt) AS billed_vs_paid_variance,
  mr.denial_bucket,
  mr.claims_in_bucket
FROM monthly m
LEFT JOIN monthly_reasons mr
  ON mr.month_start = m.month_start AND mr.provider_id = m.provider_id
ORDER BY m.month_start, m.provider_id, mr.denial_bucket;
```

---

How I used this: Powered Tableau tiles (Denial Rate %, First-Pass %, Billed vs Paid) and a reason waterfall with provider drill-downs and month selector.

---

### 🐍 Python (Pandas) — High-Cost NDC Watchlist & Outlier Flag

**Goal:** Surface outlier NDCs (drug codes) by cost & utilization; flag alternatives for PBM review.
# requirements: pandas
import pandas as pd

# ndc_df columns: ['claim_id','member_id','ndc','fill_date','qty','days_supply','billed_amt','paid_amt','therapeutic_class','formulary_flag']
ndc_df['month'] = pd.to_datetime(ndc_df['fill_date']).dt.to_period('M').dt.to_timestamp()

# monthly agg
agg = (ndc_df.groupby(['month','ndc','therapeutic_class','formulary_flag'], as_index=False)
              .agg(units=('qty','sum'),
                   fills=('claim_id','nunique'),
                   billed=('billed_amt','sum'),
                   paid=('paid_amt','sum')))

# cost per fill & z-score outlier flag within class
agg['cost_per_fill'] = (agg['paid'] / agg['fills']).replace([pd.NA, float('inf')], 0)

agg['class_mean'] = agg.groupby(['month','therapeutic_class'])['cost_per_fill'].transform('mean')
agg['class_std']  = agg.groupby(['month','therapeutic_class'])['cost_per_fill'].transform('std').fillna(0)
agg['z_cost']     = (agg['cost_per_fill'] - agg['class_mean']) / agg['class_std'].replace(0, 1)

# outlier rule: high cost & non-formulary
agg['watchlist_flag'] = (agg['z_cost'] >= 2.0) & (agg['formulary_flag'] == 0)
watchlist = agg.loc[agg['watchlist_flag'], ['month','ndc','therapeutic_class','cost_per_fill','fills','paid']].sort_values(['month','cost_per_fill'], ascending=[True, False])

---

How I used this: Published the watchlist to Snowflake and visualized:

✅ top non-formulary outliers by therapeutic class

✅ monthly trend of cost per fill

✅ drill-down to claims & provider detail for PBM team review

---

### 📈 Results & Business Impact

- −14% denial rate across targeted providers (reason-specific outreach + edit rules)

- +9% first-pass rate from documentation & coding fixes (ICD-10, CPT/HCPCS guidance)

- 20% fewer pended claims via pre-adjudication checks (missing info, eligibility)

- Faster audits with transparent reason rollups & billed-vs-paid waterfalls

  ---

  ### 🧠 Challenges & How I Solved Them
  | Challenge                               | Solution                                                                      |
| --------------------------------------- | ----------------------------------------------------------------------------- |
| Inconsistent diagnosis/procedure coding | Validated ICD-10/CPT references; reason mapping to standardized buckets       |
| PHI handling & access control           | PHI masking in curated views; row-level security; audited service roles       |
| Provider adoption                       | Delivered provider scorecards; quarterly review cadence & enablement          |
| Rx cost spikes                          | Outlier watchlist + formulary flags; PBM alternative suggestions in dashboard |

---

### 🔗 Artifacts & Links
| Item                          | Path/Link                                         |
| ----------------------------- | ------------------------------------------------- |
| Tableau – Claims Ops Overview | *Coming soon (public link)*                       |
| SQL – Denial/FPR mart         | `/sql/claims_denial_fpr.sql`                      |
| Python – NDC Watchlist        | `/python/ndc_watchlist.py`                        |
| dbt Models & Tests            | `/dbt/models/marts_claims/`                       |
| Data Model Diagram            | `/docs/claims_fhir_model.png`                     |
| Governance Notes (HIPAA)      | `/docs/governance_phi_controls.md`                |
| Dashboard Screenshot          | `/images/claims-ops-overview.png` *(placeholder)* |

---

### ✅ What I’d Improve Next

-Pre-adjudication rules simulator to estimate denial reduction before rollout

-Provider education tracker linked to denial buckets & improvement trend

-PBM alternative drug recommender (rules-based, then ML uplift testing)

-Expand to appeals analytics (win-rate by reason/provider)

---

✅ This project reflects real enterprise healthcare analytics work (claims & PBM), demonstrating domain logic, FHIR-aware modeling, secure ELT with dbt/Snowflake, and decision-driving dashboards.
