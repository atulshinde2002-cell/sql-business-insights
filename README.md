# 📊 SQL Business Insights — E-commerce Analytics Project

A structured SQL analytics project built on a simulated e-commerce database.  
Covers **15 real-world business queries** across customer analytics, sales trends, product performance, and operations — using window functions, CTEs, self-joins, and cohort analysis.

> ✅ Compatible with **MySQL 8+** and **PostgreSQL 13+**

---

## 🗂️ Database Schema — ShopMetrics

5 normalized tables simulating a retail e-commerce platform:

```
customers       → customer_id, name, email, city, signup_date, segment
orders          → order_id, customer_id, order_date, status, shipping_city, discount_pct
order_items     → item_id, order_id, product_id, quantity, unit_price
products        → product_id, category_id, product_name, cost_price, stock_qty
categories      → category_id, category_name, parent_category
```

---

## 🔍 Queries Overview

| # | Query | Category | Concepts Used |
|---|-------|----------|---------------|
| 1 | Top 10 customers by lifetime value | Customer | JOIN, GROUP BY, ORDER BY |
| 2 | Churn detection — no order in 90 days | Customer | HAVING, DATEDIFF, INTERVAL |
| 3 | New vs returning customer revenue split | Customer | CTE, CASE WHEN |
| 4 | Monthly revenue trend with MoM growth | Sales | CTE, LAG(), Window Function |
| 5 | Revenue by city with share % | Sales | Window Function, SUM OVER |
| 6 | Weekly sales heatmap (day of week) | Sales | DAYOFWEEK, CASE WHEN |
| 7 | Discount impact on revenue | Sales | CASE WHEN, bucketing |
| 8 | Top 10 products by revenue & margin | Product | JOIN, margin calculation |
| 9 | Category YoY revenue growth | Product | CTE, self-join, YEAR() |
| 10 | Slow-moving inventory alert | Product | LEFT JOIN, COALESCE, NULLIF |
| 11 | Product cross-sell pairs | Product | Self-join, basket analysis |
| 12 | Order fulfillment status breakdown | Operations | Window Function, SUM OVER |
| 13 | Customer cohort retention matrix | Operations | CTE, PERIOD_DIFF, CASE WHEN |
| 14 | Revenue per customer segment | Operations | GROUP BY, ratio calculation |
| 15 | Cumulative YTD revenue | Operations | ROWS BETWEEN, running total |

---

## 📁 Queries

### Q1 — Top 10 Customers by Lifetime Value

```sql
-- Top 10 customers by total lifetime spend
SELECT
    c.customer_id,
    c.name,
    c.city,
    COUNT(DISTINCT o.order_id)        AS total_orders,
    SUM(oi.quantity * oi.unit_price)  AS lifetime_value,
    ROUND(AVG(oi.quantity * oi.unit_price), 2) AS avg_order_value
FROM customers c
JOIN orders      o  ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id    = oi.order_id
WHERE o.status != 'cancelled'
GROUP BY c.customer_id, c.name, c.city
ORDER BY lifetime_value DESC
LIMIT 10;
```

---

### Q2 — Customer Churn (No Order in 90 Days)

```sql
-- Customers who haven't ordered in the last 90 days
SELECT
    c.customer_id,
    c.name,
    c.email,
    MAX(o.order_date)                         AS last_order_date,
    DATEDIFF(CURRENT_DATE, MAX(o.order_date)) AS days_since_order
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name, c.email
HAVING MAX(o.order_date) < CURRENT_DATE - INTERVAL 90 DAY
ORDER BY days_since_order DESC;

-- PostgreSQL: use CURRENT_DATE - INTERVAL '90 days'
```

---

### Q3 — New vs Returning Customer Revenue

```sql
WITH first_orders AS (
    SELECT customer_id, MIN(order_date) AS first_order_date
    FROM   orders
    GROUP BY customer_id
)
SELECT
    DATE_FORMAT(o.order_date, '%Y-%m') AS month,
    -- TO_CHAR(o.order_date,'YYYY-MM') AS month,  -- PostgreSQL
    SUM(CASE WHEN o.order_date = fo.first_order_date
              THEN oi.quantity * oi.unit_price ELSE 0 END) AS new_revenue,
    SUM(CASE WHEN o.order_date > fo.first_order_date
              THEN oi.quantity * oi.unit_price ELSE 0 END) AS returning_revenue
FROM orders o
JOIN first_orders fo ON o.customer_id = fo.customer_id
JOIN order_items oi  ON o.order_id   = oi.order_id
GROUP BY month
ORDER BY month;
```

---

### Q4 — Monthly Revenue Trend with MoM Growth

```sql
WITH monthly AS (
    SELECT
        DATE_FORMAT(o.order_date, '%Y-%m') AS month,
        COUNT(DISTINCT o.order_id)          AS orders,
        SUM(oi.quantity * oi.unit_price)    AS revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status != 'cancelled'
    GROUP BY month
)
SELECT
    month,
    orders,
    revenue,
    ROUND(revenue / orders, 2) AS avg_order_value,
    LAG(revenue) OVER (ORDER BY month) AS prev_month_rev,
    ROUND(
      (revenue - LAG(revenue) OVER(ORDER BY month))
      / LAG(revenue) OVER(ORDER BY month) * 100, 1
    ) AS mom_growth_pct
FROM monthly
ORDER BY month;
```

---

### Q5 — Revenue by City with Share %

```sql
SELECT
    o.shipping_city,
    COUNT(DISTINCT o.order_id)       AS total_orders,
    COUNT(DISTINCT o.customer_id)    AS unique_customers,
    SUM(oi.quantity * oi.unit_price) AS total_revenue,
    ROUND(
      SUM(oi.quantity * oi.unit_price) * 100.0
      / SUM(SUM(oi.quantity * oi.unit_price)) OVER(), 1
    ) AS revenue_share_pct
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status != 'cancelled'
GROUP BY o.shipping_city
ORDER BY total_revenue DESC;
```

---

### Q6 — Weekly Sales Heatmap

```sql
SELECT
    DAYOFWEEK(o.order_date) - 1 AS weekday_num,
    CASE DAYOFWEEK(o.order_date)
        WHEN 1 THEN 'Sunday'   WHEN 2 THEN 'Monday'
        WHEN 3 THEN 'Tuesday'  WHEN 4 THEN 'Wednesday'
        WHEN 5 THEN 'Thursday' WHEN 6 THEN 'Friday'
        ELSE 'Saturday'
    END                          AS weekday_name,
    COUNT(DISTINCT o.order_id)   AS orders,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY weekday_num, weekday_name
ORDER BY weekday_num;
```

---

### Q7 — Discount Impact on Revenue

```sql
SELECT
    CASE
        WHEN o.discount_pct = 0      THEN 'No discount'
        WHEN o.discount_pct <= 0.10  THEN '1–10%'
        WHEN o.discount_pct <= 0.20  THEN '11–20%'
        ELSE '21%+'
    END                                       AS discount_band,
    COUNT(DISTINCT o.order_id)                AS orders,
    ROUND(AVG(oi.quantity * oi.unit_price),2) AS avg_order_value,
    SUM(oi.quantity * oi.unit_price)          AS gross_revenue,
    SUM(oi.quantity * oi.unit_price
        * (1 - o.discount_pct))               AS net_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY discount_band
ORDER BY AVG(o.discount_pct);
```

---

### Q8 — Top 10 Products by Revenue & Margin

```sql
SELECT
    p.product_id,
    p.product_name,
    cat.category_name,
    SUM(oi.quantity)                                  AS units_sold,
    SUM(oi.quantity * oi.unit_price)                  AS revenue,
    SUM(oi.quantity * (oi.unit_price - p.cost_price)) AS gross_profit,
    ROUND(
      SUM(oi.quantity * (oi.unit_price - p.cost_price))
      / SUM(oi.quantity * oi.unit_price) * 100, 1
    )                                                 AS margin_pct
FROM order_items oi
JOIN products   p   ON oi.product_id = p.product_id
JOIN categories cat ON p.category_id = cat.category_id
GROUP BY p.product_id, p.product_name, cat.category_name
ORDER BY revenue DESC
LIMIT 10;
```

---

### Q9 — Category YoY Revenue Growth

```sql
WITH cat_rev AS (
    SELECT
        cat.category_name,
        YEAR(o.order_date)            AS yr,
        SUM(oi.quantity * oi.unit_price) AS revenue
    FROM order_items oi
    JOIN orders     o   ON oi.order_id   = o.order_id
    JOIN products   p   ON oi.product_id = p.product_id
    JOIN categories cat ON p.category_id = cat.category_id
    GROUP BY cat.category_name, yr
)
SELECT
    cur.category_name,
    cur.revenue                   AS current_year,
    prv.revenue                   AS prior_year,
    ROUND((cur.revenue - prv.revenue)
          / prv.revenue * 100, 1) AS yoy_growth_pct
FROM  cat_rev cur
JOIN  cat_rev prv
  ON  cur.category_name = prv.category_name
  AND cur.yr = prv.yr + 1
ORDER BY yoy_growth_pct DESC;
```

---

### Q10 — Slow-Moving Inventory Alert

```sql
SELECT
    p.product_id,
    p.product_name,
    p.stock_qty,
    COALESCE(recent.units_sold, 0) AS units_sold_60d,
    ROUND(p.stock_qty / NULLIF(COALESCE(recent.units_sold, 0), 0), 1)
                                   AS weeks_of_stock
FROM products p
LEFT JOIN (
    SELECT oi.product_id, SUM(oi.quantity) AS units_sold
    FROM   order_items oi
    JOIN   orders o ON oi.order_id = o.order_id
    WHERE  o.order_date >= CURRENT_DATE - INTERVAL 60 DAY
    GROUP BY oi.product_id
) recent ON p.product_id = recent.product_id
WHERE p.stock_qty > 50
  AND COALESCE(recent.units_sold, 0) < 5
ORDER BY p.stock_qty DESC;
```

---

### Q11 — Product Cross-Sell Pairs

```sql
SELECT
    p1.product_name AS product_a,
    p2.product_name AS product_b,
    COUNT(*)        AS co_purchase_count
FROM   order_items oi1
JOIN   order_items oi2
    ON  oi1.order_id   = oi2.order_id
   AND  oi1.product_id < oi2.product_id
JOIN   products p1 ON oi1.product_id = p1.product_id
JOIN   products p2 ON oi2.product_id = p2.product_id
GROUP BY p1.product_name, p2.product_name
ORDER BY co_purchase_count DESC
LIMIT 20;
```

---

### Q12 — Order Fulfillment Status Breakdown

```sql
SELECT
    status,
    COUNT(*)                              AS order_count,
    ROUND(COUNT(*) * 100.0
          / SUM(COUNT(*)) OVER(), 1)      AS share_pct,
    SUM(SUM(oi.quantity * oi.unit_price)) OVER
        (PARTITION BY status)             AS revenue_at_risk
FROM   orders o
JOIN   order_items oi ON o.order_id = oi.order_id
GROUP BY status
ORDER BY order_count DESC;
```

---

### Q13 — Customer Cohort Retention Matrix

```sql
WITH cohorts AS (
    SELECT
        c.customer_id,
        DATE_FORMAT(c.signup_date, '%Y-%m') AS cohort_month,
        DATE_FORMAT(o.order_date,  '%Y-%m') AS order_month,
        PERIOD_DIFF(
            DATE_FORMAT(o.order_date,  '%Y%m'),
            DATE_FORMAT(c.signup_date, '%Y%m')
        )                                   AS month_number
    FROM customers c
    JOIN orders    o ON c.customer_id = o.customer_id
)
SELECT
    cohort_month,
    COUNT(DISTINCT customer_id)                                          AS cohort_size,
    COUNT(DISTINCT CASE WHEN month_number=0 THEN customer_id END) AS m0,
    COUNT(DISTINCT CASE WHEN month_number=1 THEN customer_id END) AS m1,
    COUNT(DISTINCT CASE WHEN month_number=2 THEN customer_id END) AS m2,
    COUNT(DISTINCT CASE WHEN month_number=3 THEN customer_id END) AS m3
FROM cohorts
GROUP BY cohort_month
ORDER BY cohort_month;
```

---

### Q14 — Revenue per Customer Segment

```sql
SELECT
    c.segment,
    COUNT(DISTINCT c.customer_id)         AS customers,
    COUNT(DISTINCT o.order_id)            AS total_orders,
    SUM(oi.quantity * oi.unit_price)      AS revenue,
    ROUND(SUM(oi.quantity * oi.unit_price)
          / COUNT(DISTINCT c.customer_id), 2) AS revenue_per_customer,
    ROUND(SUM(oi.quantity * oi.unit_price)
          / COUNT(DISTINCT o.order_id),  2)   AS avg_order_value
FROM customers    c
JOIN orders       o  ON c.customer_id = o.customer_id
JOIN order_items  oi ON o.order_id    = oi.order_id
WHERE o.status != 'cancelled'
GROUP BY c.segment
ORDER BY revenue DESC;
```

---

### Q15 — Cumulative YTD Revenue (Running Total)

```sql
WITH monthly_rev AS (
    SELECT
        DATE_FORMAT(o.order_date, '%Y-%m') AS month,
        SUM(oi.quantity * oi.unit_price)   AS monthly_revenue
    FROM   orders o
    JOIN   order_items oi ON o.order_id = oi.order_id
    WHERE  YEAR(o.order_date) = YEAR(CURRENT_DATE)
      AND  o.status != 'cancelled'
    GROUP BY month
)
SELECT
    month,
    monthly_revenue,
    SUM(monthly_revenue) OVER (
        ORDER BY month
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_ytd
FROM monthly_rev
ORDER BY month;
```

---

## 🛠️ Tools & Compatibility

| Tool | Version |
|------|---------|
| MySQL | 8.0+ |
| PostgreSQL | 13+ |
| DBeaver / MySQL Workbench | Any |

---

## 💡 Key SQL Concepts Covered

- `JOIN` (INNER, LEFT)
- `GROUP BY` + `HAVING`
- Common Table Expressions (`WITH`)
- Window Functions (`LAG`, `SUM OVER`, `ROWS BETWEEN`)
- `CASE WHEN` bucketing
- Self-joins for basket analysis
- `COALESCE` / `NULLIF` for null handling
- Cohort analysis with `PERIOD_DIFF`
- YoY & MoM growth calculations

---

## 👤 Author

**[Your Name]**  
Aspiring Data Analyst | SQL • MySQL • PostgreSQL  
[LinkedIn Profile URL]

---

⭐ If this helped you, consider starring the repo!
