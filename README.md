# Brazilian E-Commerce Analytics — Olist Dataset

**Tools:** BigQuery · PopSQL · Power BI  
**Data:** [Olist Public E-Commerce Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — Kaggle  
**Status:** Complete

---

## Project Overview

End-to-end analysis of 99,000+ real Brazilian e-commerce orders from Olist, covering revenue trends, product performance, delivery operations, and customer geography.

The goal was to answer 6 business questions using production-quality SQL, then surface the findings in a 3-page interactive Power BI dashboard built for a non-technical audience.

---

## Business Questions Answered

| # | Question | Query |
|---|---|---|
| Q1 | What does monthly revenue look like over time, and how is it growing month-over-month? | `Q1_monthly_revenue_mom.sql` |
| Q2 | Which product categories drive the most revenue? | `Q2_top_categories_by_revenue.sql` |
| Q3 | What is the breakdown of order statuses across the platform? | `Q3_order_status_breakdown.sql` |
| Q4 | How does actual delivery time compare to estimated delivery time? | `Q4_delivery_delay_vs_estimated.sql` |
| Q5 | Which Brazilian states generate the most revenue? | `Q5_revenue_by_customer_state.sql` |
| Q6 | Which product categories have the highest average customer review scores? | `Q6_avg_review_score_per_category.sql` |

---

## Key Findings

**Revenue & Growth**
- Total revenue across the dataset: **$15.42M** across **99K+ orders**
- Revenue grew steadily from early 2017, peaking in November 2017, and stabilised through 2018
- Average order value: **$155.09**

**Product Performance**
- **Health & Beauty** is the #1 revenue category at ~$1.5M — but ranks 5th in customer satisfaction (4.1/5), suggesting a quality gap worth investigating
- **Bed, Bath & Table** leads in total order volume despite ranking 3rd in revenue — indicating lower average order value
- **Toys** ranks #1 in customer satisfaction (4.5/5) despite moderate order volume — strong product-market fit in an underpenetrated category

**Delivery Operations**
- **93.2% on-time delivery rate** across all delivered orders
- Orders delivered late averaged **10.62 days** beyond the estimated date
- Only 6.77% of orders were early — suggesting estimated dates are set conservatively

**Geography**
- **São Paulo (SP)** dominates revenue at ~$6M — nearly 3× the next state (RJ at ~$2M)
- Revenue is heavily concentrated in Brazil's southeast — top 3 states (SP, RJ, MG) account for the majority of platform revenue

---

## SQL Techniques Used

- **CTEs** (Common Table Expressions) for multi-step query logic
- **Window functions:** `LAG()` for month-over-month growth, `SUM() OVER()` for running totals and percentage calculations
- **`APPROX_QUANTILES()`** for median and P90 delivery delay analysis (BigQuery-native)
- **Proportional revenue allocation** across multi-item orders using `SAFE_DIVIDE()`
- **`COALESCE()`** for NULL handling across joined translation tables
- **`HAVING`** clause to filter categories with fewer than 30 reviews (statistical significance filter)
- **`DATE_DIFF()`** and `DATE_TRUNC()` for date-based calculations and monthly aggregation
- **`CASE WHEN`** for business logic bucketing (order status grouping, delivery timing classification)

---

## Power BI Dashboard

3-page interactive dashboard connected to BigQuery:

**Page 1 — Executive Overview**
KPI cards (Total Revenue, Total Orders, Avg Order Value, On-Time Rate) + Monthly Revenue Trend line chart (2017–2018)

**Page 2 — Products & Categories**
Top 10 categories by revenue, avg review score per category, total orders per category — with an interactive category slicer filtering all visuals simultaneously

**Page 3 — Delivery & Geography**
Delivery timing donut chart, avg delay KPI card, Brazil filled map (revenue by state), revenue by state bar chart

---

## Repository Structure

```
olist-ecommerce-analysis/
│
├── sql/
│   ├── Q1_monthly_revenue_mom.sql
│   ├── Q2_top_categories_by_revenue.sql
│   ├── Q3_order_status_breakdown.sql
│   ├── Q4_delivery_delay_vs_estimated.sql
│   ├── Q5_revenue_by_customer_state.sql
│   └── Q6_avg_review_score_per_category.sql
│
├── dashboard/
│   └── olist_ecommerce_dashboard.pdf
│
└── README.md
```

---

## Data Source

**Brazilian E-Commerce Public Dataset by Olist**  
Published on Kaggle under CC BY-NC-SA 4.0 license.  
Contains anonymised commercial data from orders placed on the Olist platform between 2016 and 2018.

Tables used: `orders`, `order_items`, `payments`, `customers`, `products`, `reviews`, `sellers`, `category_translation`

---

## Author

**Shahenda Hassan**  
Data & Business Analyst  
[LinkedIn](www.linkedin.com/in/shahenda-hassan-20013y)· shahendahassan099@gmail.com
