# Brazilian E-Commerce Analytics — Olist Dataset

**Tools Used:** `BigQuery` · `PopSQL` · `Power BI`  
**Data Source:** Olist Public E-Commerce Dataset (Kaggle)  
**Project Status:** Complete  

---

## 📋 Project Overview
An end-to-end analysis of **99,000+** real Brazilian e-commerce orders from Olist, covering revenue trends, product performance, delivery operations, and customer geography.

The goal was to answer 6 critical business questions using production-quality SQL in BigQuery, then surface the findings in a 3-page interactive Power BI dashboard built for a non-technical corporate audience.

---

## 💡 Key Findings

### 💰 Revenue & Growth
- **Total Revenue:** $15.42M across 99K+ orders (2017–2018)
- **Growth Trend:** Revenue grew steadily from early 2017, peaking in November 2017 (Black Friday), and stabilized throughout 2018
- **Average Order Value (AOV):** $155.09

### 📦 Product Performance
- **Health & Beauty** leads revenue at ~$1.5M — but ranks 5th in customer satisfaction (4.1/5), suggesting a potential quality or logistics gap worth investigating
- **Bed, Bath & Table** leads in total order volume despite ranking 3rd in revenue, indicating a lower average order value
- **Toys** ranks #1 in customer satisfaction (4.5/5) despite moderate volume, showing strong product-market fit in an underpenetrated category

### 🚚 Delivery Operations
- **On-Time Delivery Rate:** 93.2% across all delivered orders
- **Fulfillment Delays:** Orders delivered late averaged **10.62 days** beyond the estimated delivery date
- **Early Shipments:** Only 6.77% of orders arrived early, suggesting estimated dates are set conservatively by logistics teams

### 🗺️ Geography
- **São Paulo (SP)** dominates revenue at ~$6M — nearly **3×** the next highest state (RJ at ~$2M)
- Top 3 states (SP, RJ, MG) account for the majority of platform revenue

---

## 🔍 SQL Queries

> All queries written in BigQuery SQL. Replace `your-project.your_dataset` with your own project and dataset path.

---

### 1. Monthly Revenue & Month-over-Month Growth

```sql
WITH order_revenue AS (
  SELECT order_id, SUM(payment_value) AS revenue
  FROM `your-project.your_dataset.payments`
  GROUP BY 1
),
monthly AS (
  SELECT
    DATE_TRUNC(DATE(o.order_purchase_timestamp), MONTH) AS month_date,
    SUM(orv.revenue) AS monthly_revenue
  FROM `your-project.your_dataset.orders` o
  JOIN order_revenue orv ON o.order_id = orv.order_id
  WHERE o.order_status = 'delivered'
  GROUP BY 1
)
SELECT
  month_date,
  FORMAT_DATE('%Y-%m', month_date) AS month,
  ROUND(monthly_revenue, 2) AS monthly_revenue,
  ROUND(
    100 * (monthly_revenue - LAG(monthly_revenue) OVER (ORDER BY month_date))
    / NULLIF(LAG(monthly_revenue) OVER (ORDER BY month_date), 0),
    2
  ) AS mom_growth_pct
FROM monthly
ORDER BY month_date;
```

---

### 2. Top Product Categories by Realized Revenue

```sql
-- Step 1: Create clean English translation view
CREATE OR REPLACE VIEW `your-project.your_dataset.category_translation_clean` AS
SELECT
  string_field_0 AS product_category_name,
  string_field_1 AS product_category_name_english
FROM `your-project.your_dataset.category_translation`;

-- Step 2: Allocate payment revenue proportionally across multi-item orders
WITH order_payments AS (
  SELECT order_id, SUM(payment_value) AS order_revenue
  FROM `your-project.your_dataset.payments`
  GROUP BY order_id
),
order_item_totals AS (
  SELECT order_id, SUM(price + freight_value) AS order_items_total
  FROM `your-project.your_dataset.order_items`
  GROUP BY order_id
),
item_allocated_revenue AS (
  SELECT
    oi.order_id,
    p.product_category_name,
    op.order_revenue * SAFE_DIVIDE((oi.price + oi.freight_value), oit.order_items_total) AS allocated_revenue
  FROM `your-project.your_dataset.order_items` oi
  JOIN `your-project.your_dataset.products` p ON p.product_id = oi.product_id
  JOIN order_payments op ON op.order_id = oi.order_id
  JOIN order_item_totals oit ON oit.order_id = oi.order_id
  JOIN `your-project.your_dataset.orders` o ON o.order_id = oi.order_id
  WHERE o.order_status = 'delivered'
)
SELECT
  COALESCE(ct.product_category_name_english, iar.product_category_name, 'unknown') AS category,
  ROUND(SUM(iar.allocated_revenue), 2) AS revenue,
  COUNT(DISTINCT iar.order_id) AS orders
FROM item_allocated_revenue iar
LEFT JOIN `your-project.your_dataset.category_translation_clean` ct
  ON ct.product_category_name = iar.product_category_name
GROUP BY 1
ORDER BY revenue DESC
LIMIT 10;
```

---

### 3. Order Status Breakdown

```sql
WITH base AS (
  SELECT
    order_id,
    order_status,
    CASE
      WHEN order_status = 'delivered' THEN 'delivered'
      WHEN order_status IN ('shipped', 'invoiced', 'approved', 'processing') THEN 'in_progress'
      WHEN order_status IN ('canceled', 'unavailable') THEN 'problem'
      ELSE 'other'
    END AS status_bucket,
    CASE
      WHEN order_status = 'delivered' THEN 1
      WHEN order_status IN ('shipped', 'invoiced', 'approved', 'processing') THEN 2
      WHEN order_status IN ('canceled', 'unavailable') THEN 3
      ELSE 4
    END AS status_bucket_sort
  FROM `your-project.your_dataset.orders`
)
SELECT
  status_bucket_sort,
  status_bucket,
  order_status,
  COUNT(*) AS orders,
  SUM(COUNT(*)) OVER () AS total_orders,
  ROUND(100 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS pct_of_total
FROM base
GROUP BY status_bucket_sort, status_bucket, order_status
ORDER BY status_bucket_sort, orders DESC;
```

---

### 4. Delivery Performance — Actual vs Estimated

```sql
WITH delivered AS (
  SELECT
    order_id,
    DATE(order_purchase_timestamp)      AS purchase_date,
    DATE(order_delivered_customer_date) AS delivered_date,
    DATE(order_estimated_delivery_date) AS estimated_date
  FROM `your-project.your_dataset.orders`
  WHERE order_status = 'delivered'
    AND order_delivered_customer_date IS NOT NULL
    AND order_estimated_delivery_date IS NOT NULL
),
calc AS (
  SELECT
    order_id,
    purchase_date,
    delivered_date,
    estimated_date,
    DATE_DIFF(delivered_date, estimated_date, DAY) AS delay_days,
    CASE
      WHEN DATE_DIFF(delivered_date, estimated_date, DAY) > 0 THEN 'late'
      WHEN DATE_DIFF(delivered_date, estimated_date, DAY) < 0 THEN 'early'
      ELSE 'on_time'
    END AS delivery_timing
  FROM delivered
)
SELECT
  delivery_timing,
  COUNT(*) AS orders,
  ROUND(100 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS pct_of_delivered_orders,
  ROUND(AVG(delay_days), 2)                              AS avg_delay_days,
  APPROX_QUANTILES(delay_days, 100)[OFFSET(50)]          AS median_delay_days,
  APPROX_QUANTILES(delay_days, 100)[OFFSET(90)]          AS p90_delay_days,
  MIN(delay_days)                                        AS min_delay_days,
  MAX(delay_days)                                        AS max_delay_days
FROM calc
GROUP BY delivery_timing
ORDER BY
  CASE delivery_timing
    WHEN 'late'    THEN 1
    WHEN 'on_time' THEN 2
    WHEN 'early'   THEN 3
  END;
```

---

### 5. Revenue by Customer State

```sql
WITH order_revenue AS (
  SELECT
    order_id,
    SUM(payment_value) AS revenue
  FROM `your-project.your_dataset.payments`
  GROUP BY order_id
)
SELECT
  c.customer_state                                          AS state,
  ROUND(SUM(orv.revenue), 2)                               AS total_revenue,
  COUNT(DISTINCT o.order_id)                               AS total_orders,
  ROUND(SUM(orv.revenue) / COUNT(DISTINCT o.order_id), 2) AS avg_revenue_per_order
FROM `your-project.your_dataset.orders` o
JOIN order_revenue orv ON o.order_id = orv.order_id
JOIN `your-project.your_dataset.customers` c ON c.customer_id = o.customer_id
WHERE o.order_status = 'delivered'
GROUP BY state
ORDER BY total_revenue DESC;
```

---

### 6. Customer Satisfaction by Product Category

```sql
WITH delivered_reviews AS (
  SELECT
    o.order_id,
    r.review_score
  FROM `your-project.your_dataset.orders` o
  JOIN `your-project.your_dataset.reviews` r ON r.order_id = o.order_id
  WHERE o.order_status = 'delivered'
    AND r.review_score IS NOT NULL
),
order_category AS (
  SELECT DISTINCT
    oi.order_id,
    p.product_category_name
  FROM `your-project.your_dataset.order_items` oi
  JOIN `your-project.your_dataset.products` p ON p.product_id = oi.product_id
  WHERE p.product_category_name IS NOT NULL
)
SELECT
  COALESCE(ct.product_category_name_english, oc.product_category_name, 'unknown') AS category,
  ROUND(AVG(dr.review_score), 2)   AS avg_review_score,
  COUNT(DISTINCT dr.order_id)      AS orders_with_reviews,
  COUNT(*)                         AS order_category_pairs
FROM delivered_reviews dr
JOIN order_category oc ON oc.order_id = dr.order_id
LEFT JOIN `your-project.your_dataset.category_translation_clean` ct
  ON ct.product_category_name = oc.product_category_name
GROUP BY category
HAVING orders_with_reviews >= 30
ORDER BY avg_review_score DESC, orders_with_reviews DESC;
```

---

## 🛠️ SQL Techniques Deployed

- **CTEs** for multi-step query logic and readability
- **Window Functions:** `LAG()` for month-over-month growth · `SUM() OVER()` for percentage calculations
- **Advanced Aggregations:** BigQuery-native `APPROX_QUANTILES()` for median and P90 delivery delay insights
- **Proportional Revenue Allocation** across multi-item orders using `SAFE_DIVIDE()`
- **Data Cleansing:** `COALESCE()` for NULL handling during English mapping joins
- **Statistical Filtering:** `HAVING` clause restricting categories with fewer than 30 reviews for statistical validity
- **Date Intelligence:** `DATE_DIFF()` and `DATE_TRUNC()` for delivery tracking and monthly aggregation
- **Conditional Logic:** `CASE WHEN` bucketing across order status and delivery timing

---

## 📊 Power BI Dashboard

3-page interactive dashboard connected to BigQuery:

- **Page 1 — Executive Overview:** KPI cards (Total Revenue, Total Orders, Avg Order Value, On-Time Rate) + Monthly Revenue Trend line chart
- **Page 2 — Products & Categories:** Top 10 categories by revenue · Avg review score · Total orders — with interactive category slicer filtering all visuals simultaneously
- **Page 3 — Delivery & Geography:** Delivery timing donut chart · Avg delay KPI card · Brazil filled map · Revenue by state bar chart

---

## 📁 Repository Structure

```
olist-ecommerce-analysis/
│
├── sql/
│   ├── monthly_revenue_mom.sql
│   ├── top_categories_by_revenue.sql
│   ├── order_status_breakdown.sql
│   ├── delivery_performance.sql
│   ├── revenue_by_state.sql
│   └── customer_satisfaction_by_category.sql
│
├── dashboard/
│   └── olist_ecommerce_dashboard.pdf
│
└── README.md
```

---

## 🗃️ Data Source

**Brazilian E-Commerce Public Dataset by Olist** — Kaggle (CC BY-NC-SA 4.0)  
Tables used: `orders` · `order_items` · `payments` · `customers` · `products` · `reviews` · `sellers` · `category_translation`

---

## 👤 Author

**Shahenda Hassan**  
*Data & Business Analyst*  
📩 shahendahassan099@gmail.com · [LinkedIn](www.linkedin.com/in/shahenda-hassan-20013y) ·
