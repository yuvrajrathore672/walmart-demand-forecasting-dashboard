# Retail Demand Forecasting & Inventory Optimization Dashboard

A SQL → Python → Power BI pipeline that forecasts weekly product demand and flags stockout/overstock risk, with a business-impact dashboard quantifying revenue exposure.

## Problem Statement

Retailers routinely lose revenue in two directions: **stockouts** (lost sales when demand exceeds available inventory) and **overstock** (tied-up capital and markdown losses when inventory sits unsold). This project builds an end-to-end pipeline that predicts near-term demand per store/department, compares it against current stock levels, and surfaces which departments are at risk — before the stockout actually happens.

## Data Source

- **Walmart Recruiting - Store Sales Forecasting** (Kaggle), covering weekly sales by Store and Department from Feb 2010 to late 2012 (~421,570 rows), plus store metadata and economic/holiday features (temperature, fuel price, CPI, unemployment, markdown promotions).
- **Note on inventory data:** the source dataset contains no real stock/inventory information. An `inventory` table was engineered using standard supply-chain formulas (reorder point, safety stock) derived from each department's actual historical demand and demand variability, with `Current_Stock` randomized around each department's calculated reorder point to produce realistic risk variation. This is documented transparently rather than presented as real inventory data.

## Methodology

### 1. SQL (MySQL)
- Loaded raw CSVs via `LOAD DATA INFILE`, resolving data-quality issues along the way: an incomplete composite primary key that was silently dropping ~98% of sales rows, inconsistent boolean encoding for holiday flags across tables, and negative average demand values caused by returns-heavy departments.
- Built a `dept_demand_stats` table (average and standard deviation of weekly sales per Store-Department), with a fallback volatility estimate for departments with under 10 weeks of history.
- Engineered an `inventory` table applying industry-standard formulas:
  - `Safety_Stock = Z × StdDev(demand) × √(Lead_Time_Weeks)` (95% service level, Z = 1.65)
  - `Reorder_Point = (Avg_Weekly_Demand × Lead_Time_Weeks) + Safety_Stock`
- Used a window function (`AVG() OVER (PARTITION BY ... ROWS BETWEEN 3 PRECEDING AND CURRENT ROW)`) to compute a rolling 4-week sales average per department.
- Joined sales, rolling averages, and inventory into a single `sales_summary_view` combining aggregations, a window function, and CASE-based business logic (stockout risk flags).

### 2. Python (pandas, Prophet)
- Connected to MySQL via SQLAlchemy and pulled `sales_summary_view` into pandas.
- Selected the top 10 Store-Department combinations by total historical sales as forecasting targets.
- Ran exploratory analysis (sales trend, holiday vs. non-holiday averages) to sanity-check patterns before modeling.
- Fit a Prophet model per department (yearly seasonality enabled, weekly seasonality disabled since data is already weekly-aggregated) and forecasted 12 weeks ahead with confidence intervals.
- Validated all 10 forecasts visually against actual history before exporting.
- Exported a department-level dataset (forecast + inventory risk fields) and a company-level aggregated dataset to CSV for Power BI.

### 3. Power BI
- **Company Overview page:** actual vs. forecasted sales trend, with KPI cards for total forecasted revenue, departments at risk, and total revenue at risk.
- **Department Drill-Down page:** a Store-Department slicer, a per-department forecast chart, and a stock-vs-reorder-point table with conditional-formatting risk flags (red/green).
- DAX measures built to correctly aggregate values across a date-grained table without double-counting repeated per-department fields (caught and fixed a 155x overcounting bug in an early version of the revenue-at-risk measure).

## Key Findings (top 10 departments by revenue)

- **Total forecasted revenue, next 12 weeks:** ₹19.86M
- **Departments currently at risk of stockout:** 2 of 10
- **Total revenue at risk from under-stocked departments:** ₹565.64K
- Holiday weeks show a measurably higher average sales figure than non-holiday weeks, consistent with expected retail seasonality.
- The highest-revenue departments (Dept 92, Dept 95) were consistent across multiple stores, suggesting certain product categories drive disproportionate revenue company-wide rather than being store-specific outliers.

## Tools & Skills

- **SQL:** MySQL — joins, aggregations, window functions, data-quality debugging, views
- **Python:** pandas, SQLAlchemy, Prophet (time-series forecasting), matplotlib
- **BI:** Power BI Desktop — DAX measures, conditional formatting, interactive slicers, multi-page dashboard design
- **Domain:** inventory management formulas (reorder point, safety stock), retail demand forecasting

## Dashboard Screenshots

