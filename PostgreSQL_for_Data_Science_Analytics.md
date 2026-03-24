# PostgreSQL for Data Science & Analytics
**From Raw Data to Actionable Insights**

> A comprehensive guide covering analytical SQL, window functions, statistical analysis, time series, data cleaning, and Python integration — everything a data scientist or data analyst needs to master PostgreSQL.

---

## Table of Contents

1. [Why PostgreSQL for Data Science?](#1-why-postgresql-for-data-science)
2. [Analytical SQL Foundations](#2-analytical-sql-foundations)
3. [Window Functions — The Data Analyst's Superpower](#3-window-functions--the-data-analysts-superpower)
4. [Statistical & Mathematical Functions](#4-statistical--mathematical-functions)
5. [Time Series Analysis](#5-time-series-analysis)
6. [Data Cleaning & Transformation](#6-data-cleaning--transformation)
7. [Pivoting & Crosstab Reports](#7-pivoting--crosstab-reports)
8. [CTEs & Recursive Queries for Complex Analysis](#8-ctes--recursive-queries-for-complex-analysis)
9. [Working with JSON & Semi-Structured Data](#9-working-with-json--semi-structured-data)
10. [Python Integration (pandas, SQLAlchemy, psycopg2)](#10-python-integration)
11. [Performance for Analytical Queries](#11-performance-for-analytical-queries)
12. [Real-World Analytics Use Cases](#12-real-world-analytics-use-cases)
13. [Analytics Cheat Sheet & Quick Reference](#13-analytics-cheat-sheet--quick-reference)

---

## 1. Why PostgreSQL for Data Science?

### 1.1 PostgreSQL vs Other Tools

| Feature | PostgreSQL | MySQL | SQLite | BigQuery |
|---|---|---|---|---|
| Window Functions | ✅ Full support | ✅ (v8+) | ✅ (v3.25+) | ✅ |
| Statistical Functions | ✅ Built-in | ❌ Limited | ❌ | ✅ |
| JSON/JSONB | ✅ Advanced | ✅ Basic | ❌ | ✅ |
| Full Text Search | ✅ | ✅ | ❌ | ✅ |
| CROSSTAB / Pivot | ✅ tablefunc | ❌ | ❌ | ✅ |
| Python UDFs | ✅ PL/Python | ❌ | ❌ | ✅ |
| Cost | Free | Free | Free | Pay-per-query |

### 1.2 Sample Dataset Setup

We'll use an e-commerce analytics dataset throughout this guide.

```sql
-- Create schema
CREATE SCHEMA analytics;

-- Customers
CREATE TABLE customers (
    customer_id   SERIAL PRIMARY KEY,
    name          VARCHAR(100),
    email         VARCHAR(100) UNIQUE,
    city          VARCHAR(50),
    country       VARCHAR(50),
    age           INTEGER,
    gender        CHAR(1),
    registered_at TIMESTAMPTZ DEFAULT NOW()
);

-- Products
CREATE TABLE products (
    product_id   SERIAL PRIMARY KEY,
    name         VARCHAR(150),
    category     VARCHAR(50),
    price        NUMERIC(10,2),
    cost         NUMERIC(10,2)
);

-- Orders
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date  TIMESTAMPTZ DEFAULT NOW(),
    status      VARCHAR(20) CHECK (status IN ('pending','completed','cancelled','refunded')),
    total       NUMERIC(10,2)
);

-- Order Items
CREATE TABLE order_items (
    item_id    SERIAL PRIMARY KEY,
    order_id   INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity   INTEGER,
    unit_price NUMERIC(10,2)
);

-- Page Events (clickstream)
CREATE TABLE page_events (
    event_id   BIGSERIAL PRIMARY KEY,
    customer_id INTEGER,
    event_type  VARCHAR(30),   -- 'view', 'add_to_cart', 'purchase'
    page        VARCHAR(100),
    occurred_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 2. Analytical SQL Foundations

### 2.1 GROUP BY & Aggregations

```sql
-- Revenue by category
SELECT
    p.category,
    COUNT(DISTINCT o.order_id)        AS total_orders,
    SUM(oi.quantity)                  AS units_sold,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS revenue
FROM order_items oi
JOIN products p  ON p.product_id = oi.product_id
JOIN orders o    ON o.order_id   = oi.order_id
WHERE o.status = 'completed'
GROUP BY p.category
ORDER BY revenue DESC;
```

### 2.2 HAVING — Filter After Aggregation

```sql
-- Only categories with revenue > 50,000
SELECT
    p.category,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o   ON o.order_id   = oi.order_id
WHERE o.status = 'completed'
GROUP BY p.category
HAVING SUM(oi.quantity * oi.unit_price) > 50000
ORDER BY revenue DESC;
```

### 2.3 Grouping Sets — Multiple GROUP BY Levels at Once

```sql
-- Revenue totals at different granularities in one query
SELECT
    p.category,
    DATE_TRUNC('month', o.order_date) AS month,
    SUM(oi.quantity * oi.unit_price)  AS revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o   ON o.order_id   = oi.order_id
GROUP BY GROUPING SETS (
    (p.category, DATE_TRUNC('month', o.order_date)),  -- per category per month
    (p.category),                                      -- per category total
    (DATE_TRUNC('month', o.order_date)),               -- per month total
    ()                                                 -- grand total
);
```

### 2.4 ROLLUP & CUBE

```sql
-- ROLLUP: hierarchical subtotals (country → city → NULL grand total)
SELECT country, city, COUNT(*) AS customers
FROM customers
GROUP BY ROLLUP (country, city)
ORDER BY country NULLS LAST, city NULLS LAST;

-- CUBE: all combinations
SELECT country, gender, COUNT(*) AS customers
FROM customers
GROUP BY CUBE (country, gender);
```

---

## 3. Window Functions — The Data Analyst's Superpower

Window functions compute across a "window" of related rows **without collapsing them** like GROUP BY does.

```
Syntax:
  function_name(...) OVER (
      PARTITION BY col1, col2   -- define groups (optional)
      ORDER BY col3             -- define sort order within group
      ROWS/RANGE BETWEEN ...    -- define the frame (optional)
  )
```

### 3.1 Ranking Functions

```sql
-- ROW_NUMBER: unique sequential rank (no ties)
-- RANK: same rank for ties, gaps after
-- DENSE_RANK: same rank for ties, no gaps

SELECT
    customer_id,
    total_spent,
    ROW_NUMBER() OVER (ORDER BY total_spent DESC) AS row_num,
    RANK()       OVER (ORDER BY total_spent DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY total_spent DESC) AS dense_rank
FROM (
    SELECT customer_id, SUM(total) AS total_spent
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
) t;
```

```sql
-- Top 3 products per category by revenue
WITH ranked AS (
    SELECT
        p.category,
        p.name,
        SUM(oi.quantity * oi.unit_price) AS revenue,
        DENSE_RANK() OVER (
            PARTITION BY p.category
            ORDER BY SUM(oi.quantity * oi.unit_price) DESC
        ) AS rnk
    FROM order_items oi
    JOIN products p ON p.product_id = oi.product_id
    GROUP BY p.category, p.name
)
SELECT category, name, revenue
FROM ranked
WHERE rnk <= 3;
```

### 3.2 Offset Functions — LAG & LEAD

```sql
-- Month-over-month revenue comparison
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(total) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY 1
)
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month)  AS prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS mom_change,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
        / NULLIF(LAG(revenue) OVER (ORDER BY month), 0),
        2
    ) AS mom_pct_change
FROM monthly
ORDER BY month;
```

### 3.3 Cumulative & Running Totals

```sql
-- Cumulative revenue over time
SELECT
    DATE_TRUNC('month', order_date) AS month,
    SUM(total)                       AS monthly_revenue,
    SUM(SUM(total)) OVER (ORDER BY DATE_TRUNC('month', order_date)) AS cumulative_revenue
FROM orders
WHERE status = 'completed'
GROUP BY 1
ORDER BY 1;
```

### 3.4 Rolling Averages (Moving Average)

```sql
-- 3-month rolling average revenue
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(total) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY 1
)
SELECT
    month,
    revenue,
    ROUND(AVG(revenue) OVER (
        ORDER BY month
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2) AS rolling_3mo_avg
FROM monthly
ORDER BY month;
```

### 3.5 NTILE — Percentile Buckets

```sql
-- Divide customers into 4 spending quartiles (Q1 = lowest, Q4 = highest)
SELECT
    customer_id,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent) AS spending_quartile,
    CASE NTILE(4) OVER (ORDER BY total_spent)
        WHEN 1 THEN 'Low'
        WHEN 2 THEN 'Mid-Low'
        WHEN 3 THEN 'Mid-High'
        WHEN 4 THEN 'High'
    END AS segment
FROM (
    SELECT customer_id, SUM(total) AS total_spent
    FROM orders WHERE status = 'completed'
    GROUP BY customer_id
) t;
```

### 3.6 PERCENT_RANK & CUME_DIST

```sql
-- What percentile is each customer in?
SELECT
    customer_id,
    total_spent,
    ROUND(PERCENT_RANK() OVER (ORDER BY total_spent) * 100, 1) AS percentile,
    ROUND(CUME_DIST()    OVER (ORDER BY total_spent) * 100, 1) AS cumulative_pct
FROM (
    SELECT customer_id, SUM(total) AS total_spent
    FROM orders WHERE status = 'completed'
    GROUP BY customer_id
) t
ORDER BY total_spent DESC;
```

### 3.7 FIRST_VALUE / LAST_VALUE / NTH_VALUE

```sql
-- Compare each order to the customer's first and latest order amount
SELECT
    customer_id,
    order_date,
    total,
    FIRST_VALUE(total) OVER (
        PARTITION BY customer_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS first_order_amount,
    LAST_VALUE(total) OVER (
        PARTITION BY customer_id ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_order_amount
FROM orders
WHERE status = 'completed';
```

---

## 4. Statistical & Mathematical Functions

### 4.1 Descriptive Statistics

```sql
-- Full statistical summary per product category
SELECT
    p.category,
    COUNT(*)                                          AS n,
    ROUND(AVG(oi.unit_price), 2)                     AS mean_price,
    ROUND(STDDEV(oi.unit_price), 2)                  AS stddev_price,
    ROUND(VARIANCE(oi.unit_price), 2)                AS variance_price,
    MIN(oi.unit_price)                               AS min_price,
    MAX(oi.unit_price)                               AS max_price,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY oi.unit_price) AS p25,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY oi.unit_price) AS median,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY oi.unit_price) AS p75,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY oi.unit_price) AS p90
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
GROUP BY p.category;
```

### 4.2 PERCENTILE_CONT vs PERCENTILE_DISC

```sql
-- PERCENTILE_CONT: interpolates (returns decimal between two values)
-- PERCENTILE_DISC: returns actual value that exists in the data

SELECT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total) AS median_cont,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY total) AS median_disc
FROM orders
WHERE status = 'completed';
```

### 4.3 Correlation & Regression

```sql
-- Correlation between customer age and total spending
SELECT
    ROUND(CORR(c.age, customer_totals.total_spent)::NUMERIC, 4) AS age_spend_correlation
FROM customers c
JOIN (
    SELECT customer_id, SUM(total) AS total_spent
    FROM orders WHERE status = 'completed'
    GROUP BY customer_id
) customer_totals ON customer_totals.customer_id = c.customer_id;

-- Linear regression: predict spend from age
-- y = slope * x + intercept
SELECT
    REGR_SLOPE(total_spent, age)     AS slope,
    REGR_INTERCEPT(total_spent, age) AS intercept,
    REGR_R2(total_spent, age)        AS r_squared,
    REGR_COUNT(total_spent, age)     AS n
FROM customers c
JOIN (
    SELECT customer_id, SUM(total) AS total_spent
    FROM orders WHERE status = 'completed'
    GROUP BY customer_id
) t ON t.customer_id = c.customer_id;
```

### 4.4 Frequency Distribution (Histogram Buckets)

```sql
-- Revenue distribution in $100 buckets
SELECT
    FLOOR(total / 100) * 100 AS bucket_start,
    FLOOR(total / 100) * 100 + 100 AS bucket_end,
    COUNT(*) AS order_count
FROM orders
WHERE status = 'completed'
GROUP BY FLOOR(total / 100)
ORDER BY bucket_start;

-- Using WIDTH_BUCKET for cleaner binning
SELECT
    WIDTH_BUCKET(total, 0, 1000, 10) AS bucket,   -- 10 equal buckets between 0-1000
    COUNT(*) AS count
FROM orders
WHERE status = 'completed'
GROUP BY bucket
ORDER BY bucket;
```

---

## 5. Time Series Analysis

### 5.1 Date Truncation & Extraction

```sql
-- Key date functions
SELECT
    order_date,
    DATE_TRUNC('hour',  order_date) AS hour,
    DATE_TRUNC('day',   order_date) AS day,
    DATE_TRUNC('week',  order_date) AS week,
    DATE_TRUNC('month', order_date) AS month,
    DATE_TRUNC('quarter', order_date) AS quarter,
    DATE_TRUNC('year',  order_date) AS year,
    EXTRACT(DOW FROM order_date)    AS day_of_week,   -- 0=Sun, 6=Sat
    EXTRACT(HOUR FROM order_date)   AS hour_of_day,
    TO_CHAR(order_date, 'YYYY-MM') AS year_month
FROM orders
LIMIT 5;
```

### 5.2 Filling Gaps in Time Series

```sql
-- Generate all months even if no orders (gap filling)
WITH date_range AS (
    SELECT GENERATE_SERIES(
        DATE_TRUNC('month', MIN(order_date)),
        DATE_TRUNC('month', MAX(order_date)),
        INTERVAL '1 month'
    ) AS month
    FROM orders
),
monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(total) AS revenue
    FROM orders WHERE status = 'completed'
    GROUP BY 1
)
SELECT
    dr.month,
    COALESCE(mr.revenue, 0) AS revenue    -- 0 instead of NULL for empty months
FROM date_range dr
LEFT JOIN monthly_revenue mr ON mr.month = dr.month
ORDER BY dr.month;
```

### 5.3 Day-over-Day & Week-over-Week Analysis

```sql
-- Daily orders with DoD change
WITH daily AS (
    SELECT
        DATE_TRUNC('day', order_date) AS day,
        COUNT(*)   AS orders,
        SUM(total) AS revenue
    FROM orders WHERE status = 'completed'
    GROUP BY 1
)
SELECT
    day,
    orders,
    revenue,
    orders - LAG(orders) OVER (ORDER BY day) AS orders_dod,
    -- Same day last week
    LAG(orders, 7)  OVER (ORDER BY day) AS orders_last_week,
    orders - LAG(orders, 7) OVER (ORDER BY day) AS wow_change
FROM daily
ORDER BY day;
```

### 5.4 Cohort Analysis (Retention)

```sql
-- Monthly cohort retention
WITH cohorts AS (
    -- First order month = cohort
    SELECT
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),
orders_with_cohort AS (
    SELECT
        o.customer_id,
        c.cohort_month,
        DATE_TRUNC('month', o.order_date) AS order_month,
        -- Months since first purchase
        EXTRACT(YEAR FROM AGE(
            DATE_TRUNC('month', o.order_date),
            c.cohort_month
        )) * 12 +
        EXTRACT(MONTH FROM AGE(
            DATE_TRUNC('month', o.order_date),
            c.cohort_month
        )) AS period_number
    FROM orders o
    JOIN cohorts c ON c.customer_id = o.customer_id
    WHERE o.status = 'completed'
)
SELECT
    cohort_month,
    period_number,
    COUNT(DISTINCT customer_id) AS customers
FROM orders_with_cohort
GROUP BY cohort_month, period_number
ORDER BY cohort_month, period_number;
```

### 5.5 Session Analysis with Gaps

```sql
-- Identify user sessions (gap > 30 minutes = new session)
WITH events_with_prev AS (
    SELECT
        customer_id,
        occurred_at,
        LAG(occurred_at) OVER (
            PARTITION BY customer_id ORDER BY occurred_at
        ) AS prev_event_at
    FROM page_events
),
session_starts AS (
    SELECT
        customer_id,
        occurred_at,
        CASE
            WHEN prev_event_at IS NULL
              OR occurred_at - prev_event_at > INTERVAL '30 minutes'
            THEN 1 ELSE 0
        END AS is_new_session
    FROM events_with_prev
),
sessions AS (
    SELECT
        customer_id,
        occurred_at,
        SUM(is_new_session) OVER (
            PARTITION BY customer_id ORDER BY occurred_at
        ) AS session_id
    FROM session_starts
)
SELECT
    customer_id,
    session_id,
    MIN(occurred_at) AS session_start,
    MAX(occurred_at) AS session_end,
    COUNT(*)          AS events_in_session,
    MAX(occurred_at) - MIN(occurred_at) AS session_duration
FROM sessions
GROUP BY customer_id, session_id
ORDER BY customer_id, session_id;
```

---

## 6. Data Cleaning & Transformation

### 6.1 Finding Duplicates

```sql
-- Find duplicate emails
SELECT email, COUNT(*) AS count
FROM customers
GROUP BY email
HAVING COUNT(*) > 1;

-- Find full duplicate rows
SELECT *, COUNT(*) OVER (PARTITION BY email) AS dup_count
FROM customers;

-- Delete duplicates, keep lowest id
DELETE FROM customers
WHERE customer_id NOT IN (
    SELECT MIN(customer_id)
    FROM customers
    GROUP BY email
);
```

### 6.2 Handling NULLs

```sql
-- Replace NULLs
SELECT
    name,
    COALESCE(age, 0)                       AS age,           -- replace with 0
    COALESCE(city, 'Unknown')              AS city,          -- replace with string
    NULLIF(city, '')                       AS city_clean,    -- empty string → NULL
    age IS NULL                            AS is_age_missing
FROM customers;

-- NULL-safe comparison
-- = fails on NULL, use IS NOT DISTINCT FROM
SELECT * FROM customers
WHERE age IS NOT DISTINCT FROM NULL;
```

### 6.3 String Cleaning

```sql
-- Common string transformations
SELECT
    TRIM(name)                             AS name_trimmed,
    LOWER(TRIM(email))                     AS email_clean,
    REGEXP_REPLACE(name, '\s+', ' ', 'g') AS name_single_space,
    SPLIT_PART(email, '@', 2)             AS email_domain,
    LEFT(name, 1)                          AS first_initial,
    LENGTH(name)                           AS name_length,
    name ILIKE '%john%'                    AS is_john          -- case-insensitive match
FROM customers;
```

### 6.4 Type Casting & Conversion

```sql
-- Safe casting (returns NULL instead of error)
SELECT
    CAST('2024-01-15' AS DATE)             AS date_val,
    '42'::INTEGER                          AS int_val,
    '3.14'::NUMERIC                        AS num_val,
    TO_DATE('15/01/2024', 'DD/MM/YYYY')   AS parsed_date,
    TO_TIMESTAMP('2024-01-15 10:30', 'YYYY-MM-DD HH24:MI') AS ts_val,
    -- Safe cast with TRY (PostgreSQL 14+)
    -- Use a custom approach:
    CASE WHEN value ~ '^\d+$' THEN value::INTEGER ELSE NULL END AS safe_int
FROM some_table;
```

### 6.5 Outlier Detection

```sql
-- IQR method for outlier detection in order amounts
WITH stats AS (
    SELECT
        PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY total) AS q1,
        PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY total) AS q3
    FROM orders WHERE status = 'completed'
),
iqr_bounds AS (
    SELECT
        q1,
        q3,
        q3 - q1                AS iqr,
        q1 - 1.5 * (q3 - q1)  AS lower_fence,
        q3 + 1.5 * (q3 - q1)  AS upper_fence
    FROM stats
)
SELECT
    o.order_id,
    o.total,
    b.lower_fence,
    b.upper_fence,
    o.total < b.lower_fence OR o.total > b.upper_fence AS is_outlier
FROM orders o
CROSS JOIN iqr_bounds b
WHERE o.status = 'completed'
  AND (o.total < b.lower_fence OR o.total > b.upper_fence)
ORDER BY o.total DESC;
```

---

## 7. Pivoting & Crosstab Reports

### 7.1 Manual CASE-based Pivot

```sql
-- Monthly revenue by status (pivot)
SELECT
    DATE_TRUNC('month', order_date)                          AS month,
    SUM(CASE WHEN status = 'completed'  THEN total ELSE 0 END) AS completed,
    SUM(CASE WHEN status = 'cancelled'  THEN total ELSE 0 END) AS cancelled,
    SUM(CASE WHEN status = 'refunded'   THEN total ELSE 0 END) AS refunded,
    COUNT(CASE WHEN status = 'pending'  THEN 1      END)       AS pending_count
FROM orders
GROUP BY 1
ORDER BY 1;
```

### 7.2 CROSSTAB with tablefunc Extension

```sql
-- Enable extension first (one time)
CREATE EXTENSION IF NOT EXISTS tablefunc;

-- Revenue by category and month using CROSSTAB
SELECT *
FROM CROSSTAB(
    $$
    SELECT
        p.category,
        TO_CHAR(o.order_date, 'YYYY-MM') AS month,
        ROUND(SUM(oi.quantity * oi.unit_price), 2) AS revenue
    FROM order_items oi
    JOIN products p ON p.product_id = oi.product_id
    JOIN orders o   ON o.order_id   = oi.order_id
    WHERE o.status = 'completed'
      AND o.order_date >= '2024-01-01'
    GROUP BY p.category, TO_CHAR(o.order_date, 'YYYY-MM')
    ORDER BY 1, 2
    $$,
    $$SELECT DISTINCT TO_CHAR(order_date, 'YYYY-MM')
      FROM orders WHERE order_date >= '2024-01-01'
      ORDER BY 1$$
) AS ct (
    category VARCHAR,
    "2024-01" NUMERIC,
    "2024-02" NUMERIC,
    "2024-03" NUMERIC
    -- add more months as needed
);
```

---

## 8. CTEs & Recursive Queries for Complex Analysis

### 8.1 Multi-Step CTEs

```sql
-- Funnel analysis: view → cart → purchase
WITH views AS (
    SELECT customer_id
    FROM page_events WHERE event_type = 'view'
    GROUP BY customer_id
),
cart AS (
    SELECT customer_id
    FROM page_events WHERE event_type = 'add_to_cart'
    GROUP BY customer_id
),
purchased AS (
    SELECT DISTINCT customer_id
    FROM orders WHERE status = 'completed'
)
SELECT
    (SELECT COUNT(*) FROM views)      AS total_viewers,
    (SELECT COUNT(*) FROM cart)       AS added_to_cart,
    (SELECT COUNT(*) FROM purchased)  AS purchased,
    ROUND(100.0 * (SELECT COUNT(*) FROM cart)
                / NULLIF((SELECT COUNT(*) FROM views), 0), 1) AS view_to_cart_pct,
    ROUND(100.0 * (SELECT COUNT(*) FROM purchased)
                / NULLIF((SELECT COUNT(*) FROM cart), 0), 1) AS cart_to_purchase_pct;
```

### 8.2 Recursive CTE — Organizational Hierarchy

```sql
CREATE TABLE employees (
    employee_id INTEGER PRIMARY KEY,
    name        VARCHAR(100),
    manager_id  INTEGER REFERENCES employees(employee_id),
    department  VARCHAR(50)
);

-- Traverse the org chart from any employee upward
WITH RECURSIVE org_chart AS (
    -- Base case: start employee
    SELECT employee_id, name, manager_id, 0 AS depth, name::TEXT AS path
    FROM employees
    WHERE manager_id IS NULL  -- top-level / CEO

    UNION ALL

    -- Recursive case: go down the hierarchy
    SELECT
        e.employee_id,
        e.name,
        e.manager_id,
        oc.depth + 1,
        oc.path || ' → ' || e.name
    FROM employees e
    JOIN org_chart oc ON oc.employee_id = e.manager_id
)
SELECT employee_id, name, depth, path
FROM org_chart
ORDER BY path;
```

### 8.3 RFM Analysis (Recency, Frequency, Monetary)

```sql
-- Classic RFM scoring for customer segmentation
WITH rfm_base AS (
    SELECT
        customer_id,
        MAX(order_date)  AS last_order_date,
        COUNT(*)         AS frequency,
        SUM(total)       AS monetary
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
),
rfm_scores AS (
    SELECT
        customer_id,
        CURRENT_DATE - last_order_date::DATE  AS recency_days,
        frequency,
        monetary,
        NTILE(5) OVER (ORDER BY CURRENT_DATE - last_order_date::DATE ASC)  AS r_score, -- lower recency = better
        NTILE(5) OVER (ORDER BY frequency   DESC) AS f_score,
        NTILE(5) OVER (ORDER BY monetary    DESC) AS m_score
    FROM rfm_base
)
SELECT
    customer_id,
    recency_days,
    frequency,
    ROUND(monetary, 2) AS monetary,
    r_score,
    f_score,
    m_score,
    r_score + f_score + m_score AS rfm_total,
    CASE
        WHEN r_score >= 4 AND f_score >= 4 THEN 'Champions'
        WHEN r_score >= 3 AND f_score >= 3 THEN 'Loyal Customers'
        WHEN r_score >= 4 AND f_score <= 2 THEN 'New Customers'
        WHEN r_score <= 2 AND f_score >= 3 THEN 'At Risk'
        WHEN r_score <= 2 AND f_score <= 2 THEN 'Lost'
        ELSE 'Potential Loyalists'
    END AS segment
FROM rfm_scores
ORDER BY rfm_total DESC;
```

---

## 9. Working with JSON & Semi-Structured Data

### 9.1 JSON vs JSONB

| | JSON | JSONB |
|---|---|---|
| Storage | Text (preserves whitespace) | Binary (compressed) |
| Query Speed | Slow | Fast (indexable) |
| Write Speed | Fast | Slightly slower |
| Key order | Preserved | Not preserved |
| Use case | Audit logs, exact storage | Querying, indexing |

### 9.2 Querying JSONB

```sql
-- Suppose orders has a metadata JSONB column
ALTER TABLE orders ADD COLUMN metadata JSONB DEFAULT '{}';

-- Insert JSON data
UPDATE orders SET metadata = '{"channel": "mobile", "promo_code": "SAVE10", "tags": ["vip","repeat"]}'
WHERE order_id = 1;

-- Access JSON fields
SELECT
    order_id,
    metadata->>'channel'           AS channel,        -- returns TEXT
    metadata->'promo_code'         AS promo_json,      -- returns JSON
    metadata->>'promo_code'        AS promo_text,      -- returns TEXT
    metadata->'tags'->0            AS first_tag,       -- array index
    jsonb_array_length(metadata->'tags') AS tag_count
FROM orders;

-- Filter on JSON field
SELECT * FROM orders
WHERE metadata->>'channel' = 'mobile';

-- Check key exists
SELECT * FROM orders
WHERE metadata ? 'promo_code';

-- Nested access
SELECT metadata #>> '{address,city}' AS city FROM orders;
```

### 9.3 JSONB Aggregation

```sql
-- Build JSON from query results
SELECT
    customer_id,
    jsonb_agg(
        jsonb_build_object(
            'order_id', order_id,
            'total', total,
            'date', order_date
        ) ORDER BY order_date DESC
    ) AS orders_json
FROM orders
GROUP BY customer_id;

-- JSON to rows (JSONB_TO_RECORDSET)
SELECT *
FROM jsonb_to_recordset('[
    {"name": "Alice", "score": 95},
    {"name": "Bob",   "score": 87}
]'::jsonb) AS t(name TEXT, score INT);
```

---

## 10. Python Integration

### 10.1 psycopg2 — Raw Connection

```python
import psycopg2
import pandas as pd

# Connect
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    dbname="analytics",
    user="postgres",
    password="your_password"
)

# Execute query and fetch into DataFrame
query = """
    SELECT
        p.category,
        DATE_TRUNC('month', o.order_date) AS month,
        SUM(oi.quantity * oi.unit_price)  AS revenue
    FROM order_items oi
    JOIN products p ON p.product_id = oi.product_id
    JOIN orders o   ON o.order_id   = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY 1, 2
    ORDER BY 1, 2
"""

df = pd.read_sql(query, conn)
conn.close()

print(df.head())
print(df.dtypes)
```

### 10.2 SQLAlchemy — Recommended for Data Science

```python
from sqlalchemy import create_engine, text
import pandas as pd

# Create engine
engine = create_engine(
    "postgresql+psycopg2://postgres:password@localhost:5432/analytics"
)

# Read into DataFrame
with engine.connect() as conn:
    df = pd.read_sql(text("""
        SELECT * FROM orders
        WHERE status = 'completed'
        AND order_date >= :start_date
    """), conn, params={"start_date": "2024-01-01"})

# Write DataFrame back to PostgreSQL
df_processed = df.copy()
df_processed['revenue_bucket'] = pd.cut(df_processed['total'],
    bins=[0, 100, 500, 1000, float('inf')],
    labels=['small', 'medium', 'large', 'enterprise'])

df_processed.to_sql(
    'orders_enriched',
    engine,
    if_exists='replace',  # or 'append'
    index=False,
    dtype={}               # optionally specify pg types
)
```

### 10.3 pandas + PostgreSQL Workflow

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql+psycopg2://postgres:password@localhost/analytics")

# --- Load ---
df = pd.read_sql("SELECT * FROM orders WHERE status = 'completed'", engine)
df['order_date'] = pd.to_datetime(df['order_date'])

# --- Analyze in pandas ---
# Monthly revenue
monthly = (df.set_index('order_date')
             .resample('ME')['total']
             .sum()
             .reset_index()
             .rename(columns={'order_date': 'month', 'total': 'revenue'}))

# 3-month rolling average
monthly['rolling_avg'] = monthly['revenue'].rolling(3).mean()

# Customer cohorts
df['cohort'] = df.groupby('customer_id')['order_date'].transform('min').dt.to_period('M')
df['order_period'] = df['order_date'].dt.to_period('M')
df['period_number'] = (df['order_period'] - df['cohort']).apply(lambda x: x.n)

cohort_data = (df.groupby(['cohort', 'period_number'])['customer_id']
                 .nunique()
                 .reset_index())

# --- Save results back to PostgreSQL ---
monthly.to_sql('monthly_revenue_summary', engine, if_exists='replace', index=False)
cohort_data.to_sql('cohort_retention', engine, if_exists='replace', index=False)

print("Analysis complete. Results saved to PostgreSQL.")
```

### 10.4 Using %sql / ipython-sql in Jupyter

```python
# Install: pip install ipython-sql sqlalchemy psycopg2-binary

# In Jupyter cell:
%load_ext sql
%sql postgresql://postgres:password@localhost/analytics

# Run SQL directly
%%sql
SELECT
    p.category,
    COUNT(*) AS orders,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o   ON o.order_id   = oi.order_id
WHERE o.status = 'completed'
GROUP BY p.category
ORDER BY revenue DESC;
```

---

## 11. Performance for Analytical Queries

### 11.1 EXPLAIN ANALYZE — Read the Query Plan

```sql
-- Always use EXPLAIN (ANALYZE, BUFFERS) for real stats
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT
    p.category,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
GROUP BY p.category;

-- Key things to look for:
-- Seq Scan → consider adding an index
-- Hash Join  → efficient for large tables
-- Nested Loop → can be slow, check for missing indexes
-- rows= estimate vs actual → big gap = stale statistics
-- Buffers: hit vs read → 'read' means disk I/O (slow)
```

### 11.2 Indexes for Analytics

```sql
-- Index for time range queries (very common in analytics)
CREATE INDEX idx_orders_date ON orders (order_date DESC);

-- Partial index (only completed orders — skip cancelled/pending)
CREATE INDEX idx_orders_completed ON orders (order_date, customer_id)
WHERE status = 'completed';

-- Covering index (includes all columns needed — avoids heap fetch)
CREATE INDEX idx_orders_covering ON orders (customer_id, order_date)
INCLUDE (total, status);

-- BRIN index for huge sequential tables (timestamps, IDs)
-- Much smaller than B-tree, good for time-series
CREATE INDEX idx_events_brin ON page_events USING BRIN (occurred_at);

-- Expression index
CREATE INDEX idx_orders_month ON orders (DATE_TRUNC('month', order_date));
```

### 11.3 Materialized Views for Expensive Aggregations

```sql
-- Pre-compute expensive aggregations
CREATE MATERIALIZED VIEW mv_monthly_category_revenue AS
SELECT
    DATE_TRUNC('month', o.order_date) AS month,
    p.category,
    COUNT(DISTINCT o.order_id)        AS total_orders,
    SUM(oi.quantity)                  AS units_sold,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o   ON o.order_id   = oi.order_id
WHERE o.status = 'completed'
GROUP BY 1, 2
WITH DATA;

-- Index the materialized view
CREATE INDEX ON mv_monthly_category_revenue (month, category);

-- Refresh the view (run via cron or after data loads)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_category_revenue;
-- CONCURRENTLY allows reads during refresh (requires unique index)
CREATE UNIQUE INDEX ON mv_monthly_category_revenue (month, category);
```

### 11.4 Partitioning for Large Fact Tables

```sql
-- Range partition orders by year (common for analytics)
CREATE TABLE orders_partitioned (
    order_id    SERIAL,
    customer_id INTEGER,
    order_date  TIMESTAMPTZ NOT NULL,
    status      VARCHAR(20),
    total       NUMERIC(10,2)
) PARTITION BY RANGE (order_date);

-- Create yearly partitions
CREATE TABLE orders_2023 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders_partitioned
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- PostgreSQL automatically routes queries to correct partition
-- Queries with WHERE order_date = '2024-06-01' only scan 2024 partition
```

---

## 12. Real-World Analytics Use Cases

### 12.1 Customer Lifetime Value (CLV)

```sql
WITH customer_stats AS (
    SELECT
        customer_id,
        MIN(order_date) AS first_order,
        MAX(order_date) AS last_order,
        COUNT(*)         AS total_orders,
        SUM(total)       AS total_revenue,
        AVG(total)       AS avg_order_value,
        -- Days active
        EXTRACT(DAY FROM MAX(order_date) - MIN(order_date)) AS days_active
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
)
SELECT
    customer_id,
    total_orders,
    ROUND(total_revenue, 2)    AS clv,
    ROUND(avg_order_value, 2)  AS aov,
    days_active,
    -- Predicted future value (simple model: avg * expected orders)
    ROUND(avg_order_value * (365.0 / NULLIF(days_active, 0) * total_orders), 2) AS predicted_annual_clv
FROM customer_stats
ORDER BY clv DESC;
```

### 12.2 Churn Detection

```sql
-- Flag customers who haven't ordered in 90 days (churned)
WITH last_orders AS (
    SELECT
        customer_id,
        MAX(order_date) AS last_order_date
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
)
SELECT
    c.customer_id,
    c.name,
    c.email,
    lo.last_order_date,
    CURRENT_DATE - lo.last_order_date::DATE AS days_since_last_order,
    CASE
        WHEN CURRENT_DATE - lo.last_order_date::DATE > 90  THEN 'Churned'
        WHEN CURRENT_DATE - lo.last_order_date::DATE > 60  THEN 'At Risk'
        WHEN CURRENT_DATE - lo.last_order_date::DATE > 30  THEN 'Cooling'
        ELSE 'Active'
    END AS churn_status
FROM customers c
JOIN last_orders lo ON lo.customer_id = c.customer_id
ORDER BY days_since_last_order DESC;
```

### 12.3 A/B Test Analysis

```sql
-- Analyze A/B test results with statistical significance proxy
WITH test_groups AS (
    SELECT
        customer_id,
        CASE WHEN customer_id % 2 = 0 THEN 'control' ELSE 'treatment' END AS group_name
    FROM customers
),
group_metrics AS (
    SELECT
        tg.group_name,
        COUNT(DISTINCT o.customer_id)              AS customers,
        COUNT(o.order_id)                          AS total_orders,
        ROUND(SUM(o.total), 2)                     AS total_revenue,
        ROUND(AVG(o.total), 2)                     AS avg_order_value,
        ROUND(AVG(o.total), 2) AS mean,
        ROUND(STDDEV(o.total), 2)                  AS stddev
    FROM test_groups tg
    LEFT JOIN orders o ON o.customer_id = tg.customer_id
        AND o.status = 'completed'
    GROUP BY tg.group_name
)
SELECT
    *,
    ROUND(100.0 * total_orders / customers, 2) AS conversion_rate
FROM group_metrics;
```

### 12.4 Product Affinity / Market Basket

```sql
-- Which products are frequently bought together?
SELECT
    a.product_id AS product_a,
    b.product_id AS product_b,
    p1.name      AS name_a,
    p2.name      AS name_b,
    COUNT(*)     AS co_occurrence
FROM order_items a
JOIN order_items b ON b.order_id = a.order_id
    AND b.product_id > a.product_id      -- avoid duplicates
JOIN products p1 ON p1.product_id = a.product_id
JOIN products p2 ON p2.product_id = b.product_id
GROUP BY a.product_id, b.product_id, p1.name, p2.name
HAVING COUNT(*) >= 5
ORDER BY co_occurrence DESC
LIMIT 20;
```

### 12.5 Funnel Drop-Off Analysis

```sql
-- Conversion funnel with drop-off rates
WITH funnel AS (
    SELECT
        'Step 1: View'         AS step, 1 AS step_order,
        COUNT(DISTINCT customer_id) AS users
    FROM page_events WHERE event_type = 'view'

    UNION ALL

    SELECT 'Step 2: Add to Cart', 2,
        COUNT(DISTINCT customer_id)
    FROM page_events WHERE event_type = 'add_to_cart'

    UNION ALL

    SELECT 'Step 3: Purchase', 3,
        COUNT(DISTINCT customer_id)
    FROM orders WHERE status = 'completed'
)
SELECT
    step,
    users,
    FIRST_VALUE(users) OVER (ORDER BY step_order) AS top_of_funnel,
    ROUND(100.0 * users / FIRST_VALUE(users) OVER (ORDER BY step_order), 1) AS pct_of_top,
    LAG(users) OVER (ORDER BY step_order) AS prev_step_users,
    ROUND(100.0 * users / NULLIF(LAG(users) OVER (ORDER BY step_order), 0), 1) AS step_conversion_pct
FROM funnel
ORDER BY step_order;
```

---

## 13. Analytics Cheat Sheet & Quick Reference

### Window Functions Quick Reference

```sql
-- Ranking
ROW_NUMBER() OVER (PARTITION BY cat ORDER BY val DESC)
RANK()        OVER (PARTITION BY cat ORDER BY val DESC)
DENSE_RANK()  OVER (PARTITION BY cat ORDER BY val DESC)
NTILE(4)      OVER (ORDER BY val)
PERCENT_RANK() OVER (ORDER BY val)   -- 0.0 to 1.0
CUME_DIST()    OVER (ORDER BY val)   -- 0.0 to 1.0

-- Offsets
LAG(col, 1, default)  OVER (PARTITION BY cat ORDER BY date)
LEAD(col, 1, default) OVER (PARTITION BY cat ORDER BY date)
FIRST_VALUE(col)      OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
LAST_VALUE(col)       OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)

-- Aggregates as windows
SUM(col)   OVER (PARTITION BY cat ORDER BY date)                          -- running total
AVG(col)   OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) -- 7-row rolling avg
COUNT(col) OVER (PARTITION BY cat)                                        -- group count without collapsing
```

### Statistical Functions Quick Reference

```sql
AVG(col)                                      -- mean
STDDEV(col)   / STDDEV_POP(col)               -- sample / population std dev
VARIANCE(col) / VAR_POP(col)                  -- sample / population variance
CORR(x, y)                                    -- pearson correlation [-1, 1]
REGR_SLOPE(y, x)                              -- linear regression slope
REGR_INTERCEPT(y, x)                          -- linear regression intercept
REGR_R2(y, x)                                 -- R-squared
PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY col) -- median (interpolated)
PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY col) -- median (actual value)
MODE() WITHIN GROUP (ORDER BY col)            -- most frequent value
```

### Date Functions Quick Reference

```sql
DATE_TRUNC('month', ts)                       -- truncate to month start
EXTRACT(DOW FROM ts)                          -- day of week (0=Sun)
DATE_PART('hour', ts)                         -- extract part as float
NOW(), CURRENT_TIMESTAMP                      -- current timestamp with tz
CURRENT_DATE                                  -- today's date
ts + INTERVAL '7 days'                        -- date arithmetic
AGE(ts1, ts2)                                 -- interval between timestamps
GENERATE_SERIES(start, end, interval)         -- generate time series
TO_CHAR(ts, 'YYYY-MM')                        -- format timestamp as string
TO_DATE('2024-01', 'YYYY-MM')                 -- parse string to date
```

### Common Analytics Patterns

```sql
-- Month-over-month % change
100.0 * (current - previous) / NULLIF(previous, 0)

-- Running total
SUM(col) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING)

-- Percentage of total
100.0 * col / SUM(col) OVER ()

-- Rank within group
DENSE_RANK() OVER (PARTITION BY group ORDER BY metric DESC)

-- Gap fill with COALESCE + LEFT JOIN to generated series
LEFT JOIN GENERATE_SERIES(...) gs ON gs.date = t.date
SELECT COALESCE(t.value, 0)

-- Deduplication (keep latest)
ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) = 1
```

---

> **Next Steps:**
> - Practice window functions on [LeetCode SQL problems](https://leetcode.com/problemset/database/)
> - Connect this guide to `SQL_PostgreSQL_Mastery_Guide_Clean.md` for deeper query optimization
> - Explore `Database_Design_Scalability_Guide.md` for how to design schemas that make analytics fast
