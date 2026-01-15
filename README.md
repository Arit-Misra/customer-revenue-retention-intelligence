# Customer Revenue & Retention Intelligence Platform

**One-line summary**: End-to-end customer analytics platform: Python does the data engineering, cohort & RFM analysis; Power BI provides an interactive dashboard for revenue, retention, segmentation and operations.

---

## Table of contents
- Overview
- Repository structure
- CSV outputs (schema)
- Python pipeline (what it does & how to run)
- Power BI: data model & relationships
- Core DAX measures (copy-paste ready)
- Dashboard pages: visuals & fields (page-by-page)
- Formatting & presentation checklist
- Troubleshooting & common fixes
- Interview / presentation script (2–3 minutes)
- Extensions & next steps

---

## Overview
This project turns raw transaction data into actionable analytics. It delivers:
- A repeatable Python pipeline that outputs curated CSVs for BI consumption.
- A Power BI `.pbix` dashboard with four pages: Executive Overview, Retention & Cohorts, Customer Segmentation, and Operations.

The goal is to surface revenue trends, quantify customer stickiness, identify high-value customers, and monitor operational KPIs.

---

## Repository structure
```
/ (repo root)
│   readme.md
│
├───Code/
│       customer_analytics.ipynb   # Python analytics: cohort, RFM, exports
│
├───data_processed/
│       cohort_retention.csv
│       master_orders.csv
│       monthly_revenue.csv
│       ops_metrics.csv
│       rfm_customers.csv
│
├───Olist Dataset/                # Raw source data (excluded from git)
│       olist_customers_dataset.csv
│       olist_geolocation_dataset.csv
│       olist_orders_dataset.csv
│       olist_order_items_dataset.csv
│       olist_order_payments_dataset.csv
│       olist_order_reviews_dataset.csv
│       olist_products_dataset.csv
│       olist_sellers_dataset.csv
│       product_category_name_translation.csv
│
└───Power BI Dashboard/
        dashboard.pbix
```

/ (repo root)
├─ Code/
│  ├─ data_ingest.py         # load raw files / validation
│  ├─ pipeline_etl.py       # cleaning, feature engineering, export CSVs
│  ├─ rfm.py                 # RFM scoring & segmentation
│  ├─ cohorts.py             # cohort computation & retention table
│  ├─ requirements.txt
│  └─ outputs/
│     ├─ master_orders.csv
│     ├─ monthly_revenue.csv
│     ├─ cohort_retention.csv
│     ├─ rfm_customers.csv
│     └─ ops_metrics.csv

├─ Power BI Dashboard/
│  └─ Customer_Revenue_Retention_Intelligence.pbix

├─ README.md
└─ LICENSE

```
/ (repo root)
├─ python/
│  ├─ data_ingest.py         # load raw files / validation
│  ├─ pipeline_etl.py       # cleaning, feature engineering, export CSVs
│  ├─ rfm.py                 # RFM scoring & segmentation
│  ├─ cohorts.py             # cohort computation & retention table
│  ├─ requirements.txt
│  └─ outputs/
│     ├─ master_orders.csv
│     ├─ monthly_revenue.csv
│     ├─ cohort_retention.csv
│     ├─ rfm_customers.csv
│     └─ ops_metrics.csv

├─ powerbi/
│  └─ Customer_Revenue_Retention_Intelligence.pbix

├─ README.md
└─ LICENSE
```

---

## CSV outputs (schema)
**master_orders** (order-level)
- order_id
- customer_unique_id
- order_month (YYYY-MM)
- order_value
- payment_type
- delivery_days
- review_score
- customer_state
- is_repeat_customer (0/1)

**monthly_revenue** (monthly aggregate)
- order_month
- revenue
- orders
- customers
- aov
- mom_growth_pct

**cohort_retention** (cohort × month)
- cohort_month (YYYY-MM)
- order_month (YYYY-MM)
- active_customers
- cohort_size
- retention_rate (0–1)

**rfm_customers** (customer-level)
- customer_unique_id
- recency
- frequency
- monetary
- RFM_score
- segment (Low Value / Mid Value / High Value)

**ops_metrics** (monthly ops)
- order_month
- avg_delivery_days
- avg_review_score
- late_delivery_rate

---

## Python pipeline (what it does)
**Objectives**
- Clean raw transaction data
- Generate monthly and cohort aggregates
- Compute RFM features and deterministic segments
- Export stable CSVs for Power BI

**How to run (example)**
```bash
cd python
python -m venv .venv
source .venv/bin/activate   # or .\.venv\Scripts\activate on Windows
pip install -r requirements.txt
python pipeline_etl.py --input-dir ../raw --output-dir ./outputs
```

**Key notes**
- Ensure timestamps are parsed and normalized to month-grain (`YYYY-MM`).
- Cohort computation should define `cohort_month` as customer's first order month.
- When computing retention, use cohort size as denominator and ensure missing future months are left blank (not zero) where appropriate.

---

## Power BI — Data model & relationships
**Tables to load**: `master_orders`, `monthly_revenue`, `cohort_retention`, `rfm_customers`, `ops_metrics`

**Relationships (Model view)**
- `master_orders[customer_unique_id]` → `rfm_customers[customer_unique_id]` (Many-to-One, Single)
- `master_orders[order_month]` → `monthly_revenue[order_month]` (Many-to-One, Single)
- `master_orders[order_month]` → `ops_metrics[order_month]` (Many-to-One, Single)

**Settings**
- Disable Auto Date/Time (File → Options → Data load → Auto date/time = OFF)
- Treat `order_month`, `cohort_month` as **Text** with an additional numeric sort key `YYYYMM` for chronological sorting

---

## Core DAX measures (create table `KPI_Measures` and paste these)
```DAX
Total Revenue = SUM(master_orders[order_value])

Total Orders = DISTINCTCOUNT(master_orders[order_id])

Active Customers = DISTINCTCOUNT(master_orders[customer_unique_id])

AOV = DIVIDE([Total Revenue], [Total Orders])

Repeat Customers =
CALCULATE(
    DISTINCTCOUNT(master_orders[customer_unique_id]),
    master_orders[is_repeat_customer] = 1
)

Repeat Rate = DIVIDE([Repeat Customers], [Active Customers])

Avg Delivery Days = AVERAGE(master_orders[delivery_days])

Avg Review Score = AVERAGE(master_orders[review_score])
```
**Optional**
```DAX
High Value Customer % =
DIVIDE(
  CALCULATE(COUNT(rfm_customers[customer_unique_id]), rfm_customers[segment] = "High Value"),
  COUNT(rfm_customers[customer_unique_id])
)
```

---

## Dashboard pages (fields & layout)
**Page 1 — Executive Overview**
- KPI cards: `Total Revenue`, `Active Customers`, `Total Orders`, `AOV`, `Repeat Rate`
- Line chart: X = `master_orders[order_month]`, Values = `Total Revenue` (X-axis type: Categorical)
- Bar chart: Axis = `master_orders[customer_state]`, Values = `Total Revenue`
- Slicers: `master_orders[order_month]`, `master_orders[payment_type]` (Select all ON)

**Page 2 — Retention & Cohorts**
- Matrix: Rows = `cohort_retention[cohort_month_txt]`, Columns = `cohort_retention[order_month_txt]`, Values = `retention_rate` (Average) — format as % and remove totals
- Line chart: X = `master_orders[order_month]`, Values = `Repeat Rate`
- Optional: Bar chart: Cohort size (axis: `cohort_retention[cohort_month_txt]`, values: `cohort_size`)

**Page 3 — Customer Segmentation**
- Bar chart: Axis = `rfm_customers[segment_ordered]` (1. Low, 2. Mid, 3. High), Values = `COUNT(customer_unique_id)`
- Scatter: X = `frequency`, Y = `monetary`, Legend = `segment`, Tooltips include `customer_unique_id` (no size encoding)
- KPI card: `High Value Customer %` (optional)

**Page 4 — Operations**
- Line chart: X = `ops_metrics[order_month]`, Values = `Avg Delivery Days` (Average)
- Bar chart: Axis = `master_orders[payment_type]`, Values = `Total Orders` (DISTINCTCOUNT by order_id)
- KPI card: `Avg Review Score`
- Slicer: `master_orders[order_month]` or page-level

---

## Formatting & presentation checklist
- Disable date hierarchies (Auto date/time = OFF)
- Sort text months by a numeric `YYYYMM` column (Sort by Column)
- Matrix: remove row & column totals, show % with 2 decimals
- Scatter: remove size; use legend; reduce marker size; enable tooltips
- Bar charts: semantic ordering via sort column or prefixed labels (avoid circular sort dependencies)
- Use Card visuals for single-number KPIs and format units/decimals

---

## Troubleshooting (common issues & fixes)
- **Matrix shows full dates (01 Jan 2017)**: Convert `cohort_month` and `order_month` to text or create `FORMAT(date, "yyyy-MM")` columns and use those in the visual.
- **Circular dependency when sorting**: Avoid sorting a column by a derived column created from itself. Use prefixed labels ("1. Low Value") or a disconnected sort table.
- **Power BI sums numeric columns by default**: Use explicit measures for averages and rates. Replace raw column visuals with measures where applicable.
- **Scatter shows identical dots (Count used)**: Change Size aggregation to `Sum` or remove Size; use monetary on Y axis.

---

## Interview / presentation script (2–3 minutes)
Use this when demoing the dashboard:

1. One-liner: "This dashboard tracks revenue health, customer stickiness, segment-level value, and operations to prioritize retention vs acquisition decisions."
2. Page 1 (30s): "Executive view — revenue trend, AOV, and repeat rate for quick health checks."
3. Page 2 (45s): "Cohort analysis — shows retention decay. We see a ~3% repeat rate, indicating an acquisition-driven business and an opportunity in retention programs. Cohort sizes explain volatility."
4. Page 3 (30s): "RFM segmentation — a small High Value segment drives most value; scatter shows frequency correlates with spend. Prioritize these customers for retention and LTV initiatives."
5. Page 4 (15s): "Operations — delivery time and payment mix; delivery improves over time and aligns with high review scores."
6. Close: "Next steps: run targeted experiments to increase repeat rate and model LTV improvements."

---

## Extensions & next steps
- Add an LTV page with cohort-level LTV and payback period
- Build an automated refresh (Power BI dataflow or direct connection to data lake)
- Add segmentation stability and churn prediction (ML model)
- Provide Copilot / Power Automate instructions to push periodic reports

---

## License
MIT

---

If you need: a shorter GitHub description, a `CONTRIBUTING.md`, or the interview script expanded into talking points, say which and I will add it.

