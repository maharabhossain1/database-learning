# Complete SQL & PostgreSQL Mastery Guide
**From Zero to Advanced Backend Engineer**

> A comprehensive guide covering SQL fundamentals to PostgreSQL advanced features with 100+ problems, LeetCode challenges, and real-world backend engineering scenarios.

---

## Table of Contents

1. [SQL Fundamentals](#1-sql-fundamentals)
2. [PostgreSQL Setup & Basics](#2-postgresql-setup--basics)
3. [Intermediate SQL](#3-intermediate-sql)
4. [Advanced SQL Queries](#4-advanced-sql-queries)
5. [PostgreSQL Advanced Features](#5-postgresql-advanced-features)
6. [Complex Joins & Relationships](#6-complex-joins--relationships)
7. [Query Optimization & Performance](#7-query-optimization--performance)
8. [Real-World Backend Problems](#8-real-world-backend-problems)
9. [LeetCode SQL Problems](#9-leetcode-sql-problems)
10. [Database Design Patterns](#10-database-design-patterns)
11. [Production Best Practices](#11-production-best-practices)

---

## 1. SQL Fundamentals

### 1.1 What is SQL?
SQL (Structured Query Language) is a standard language for managing relational databases. PostgreSQL is an advanced open-source RDBMS that extends SQL with powerful features.

### 1.2 Basic Database Concepts

**Database → Schema → Table → Row → Column**

```sql
-- Create a database
CREATE DATABASE my_app;

-- Connect to database
\c my_app

-- Create a table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    age INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 1.3 Data Types in PostgreSQL

```sql
-- Numeric Types
INTEGER, BIGINT, SERIAL, BIGSERIAL
DECIMAL(10,2), NUMERIC(10,2)
REAL, DOUBLE PRECISION

-- Character Types
CHAR(n), VARCHAR(n), TEXT

-- Date/Time Types
DATE, TIME, TIMESTAMP, TIMESTAMPTZ, INTERVAL

-- Boolean
BOOLEAN (TRUE/FALSE/NULL)

-- JSON Types (PostgreSQL specific)
JSON, JSONB

-- Array Types (PostgreSQL specific)
INTEGER[], TEXT[], VARCHAR(50)[]

-- UUID (PostgreSQL specific)
UUID

-- Special Types
CIDR, INET, MACADDR -- Network addresses
GEOMETRY, GEOGRAPHY -- PostGIS for spatial data
```

### 1.4 CRUD Operations

#### CREATE (INSERT)
```sql
-- Single insert
INSERT INTO users (username, email, age)
VALUES ('maharab', 'maharab@example.com', 25);

-- Multiple inserts
INSERT INTO users (username, email, age) VALUES
    ('john', 'john@example.com', 30),
    ('jane', 'jane@example.com', 28),
    ('bob', 'bob@example.com', 35);

-- Insert with RETURNING (PostgreSQL feature)
INSERT INTO users (username, email, age)
VALUES ('alice', 'alice@example.com', 27)
RETURNING id, username, created_at;
```

#### READ (SELECT)
```sql
-- Select all columns
SELECT * FROM users;

-- Select specific columns
SELECT username, email FROM users;

-- With WHERE clause
SELECT * FROM users WHERE age > 25;

-- Multiple conditions
SELECT * FROM users 
WHERE age > 25 AND username LIKE 'j%';

-- Using IN operator
SELECT * FROM users 
WHERE username IN ('john', 'jane', 'alice');

-- Using BETWEEN
SELECT * FROM users 
WHERE age BETWEEN 25 AND 30;

-- NULL checks
SELECT * FROM users WHERE age IS NULL;
SELECT * FROM users WHERE age IS NOT NULL;
```

#### UPDATE
```sql
-- Update single row
UPDATE users 
SET age = 26 
WHERE username = 'maharab';

-- Update multiple columns
UPDATE users 
SET age = 31, email = 'john.new@example.com' 
WHERE username = 'john';

-- Update with calculation
UPDATE users 
SET age = age + 1 
WHERE id = 1;

-- Update with RETURNING
UPDATE users 
SET age = 28 
WHERE username = 'alice'
RETURNING *;
```

#### DELETE
```sql
-- Delete specific row
DELETE FROM users WHERE id = 1;

-- Delete multiple rows
DELETE FROM users WHERE age < 25;

-- Delete all (dangerous!)
DELETE FROM users;

-- Delete with RETURNING
DELETE FROM users 
WHERE username = 'bob'
RETURNING id, username;
```

---

### Practice Problems - Level 1 (Basics)

**Problem 1:** Create a `products` table with columns: id, name, price, stock_quantity, category, created_at

**Solution:**
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    stock_quantity INTEGER DEFAULT 0,
    category VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Problem 2:** Insert 5 products into the products table

**Solution:**
```sql
INSERT INTO products (name, price, stock_quantity, category) VALUES
    ('iPhone 15', 999.99, 50, 'Electronics'),
    ('Samsung TV', 1299.99, 30, 'Electronics'),
    ('Nike Shoes', 89.99, 100, 'Fashion'),
    ('Coffee Maker', 49.99, 75, 'Home'),
    ('Desk Lamp', 29.99, 120, 'Home');
```

**Problem 3:** Find all products with price less than $100

**Solution:**
```sql
SELECT * FROM products WHERE price < 100;
```

**Problem 4:** Update the stock quantity of 'iPhone 15' to 45

**Solution:**
```sql
UPDATE products 
SET stock_quantity = 45 
WHERE name = 'iPhone 15';
```

**Problem 5:** Delete all products in 'Fashion' category

**Solution:**
```sql
DELETE FROM products WHERE category = 'Fashion';
```

---

## 2. PostgreSQL Setup & Basics

### 2.1 Installation

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install postgresql postgresql-contrib

# macOS (using Homebrew)
brew install postgresql

# Start PostgreSQL
sudo service postgresql start  # Linux
brew services start postgresql  # macOS

# Connect to PostgreSQL
psql -U postgres
```

### 2.2 PostgreSQL Command Line (psql)

```sql
-- List all databases
\l

-- Connect to database
\c database_name

-- List all tables
\dt

-- Describe table structure
\d table_name

-- List all schemas
\dn

-- List all users
\du

-- Execute SQL file
\i /path/to/file.sql

-- Quit
\q
```

### 2.3 User & Permission Management

```sql
-- Create user
CREATE USER maharab WITH PASSWORD 'secure_password';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE my_app TO maharab;
GRANT SELECT, INSERT, UPDATE ON users TO maharab;

-- Create user with specific privileges
CREATE USER readonly_user WITH PASSWORD 'password';
GRANT CONNECT ON DATABASE my_app TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- Revoke privileges
REVOKE INSERT, UPDATE, DELETE ON users FROM readonly_user;

-- Change user password
ALTER USER maharab WITH PASSWORD 'new_password';
```

### 2.4 Constraints

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    order_number VARCHAR(50) NOT NULL UNIQUE,
    user_id INTEGER NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL CHECK (total_amount > 0),
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Table constraints
    CONSTRAINT valid_status CHECK (status IN ('pending', 'processing', 'completed', 'cancelled'))
);
```

Types of constraints:
- **PRIMARY KEY**: Unique identifier
- **FOREIGN KEY**: References another table
- **UNIQUE**: No duplicate values
- **NOT NULL**: Cannot be null
- **CHECK**: Custom validation
- **DEFAULT**: Default value

---

## 3. Intermediate SQL

### 3.1 Sorting and Limiting

```sql
-- ORDER BY
SELECT * FROM products ORDER BY price ASC;  -- Ascending
SELECT * FROM products ORDER BY price DESC; -- Descending

-- Multiple columns
SELECT * FROM products 
ORDER BY category ASC, price DESC;

-- LIMIT and OFFSET (Pagination)
SELECT * FROM products LIMIT 10;          -- First 10 rows
SELECT * FROM products LIMIT 10 OFFSET 20; -- Skip 20, get next 10

-- Equivalent using FETCH (SQL standard)
SELECT * FROM products 
OFFSET 20 ROWS 
FETCH NEXT 10 ROWS ONLY;
```

### 3.2 Aggregate Functions

```sql
-- COUNT
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT category) FROM products;

-- SUM
SELECT SUM(price) as total_value FROM products;

-- AVG
SELECT AVG(age) as average_age FROM users;

-- MIN and MAX
SELECT MIN(price), MAX(price) FROM products;

-- Multiple aggregates
SELECT 
    COUNT(*) as total_products,
    AVG(price) as avg_price,
    MIN(price) as min_price,
    MAX(price) as max_price,
    SUM(stock_quantity) as total_stock
FROM products;
```

### 3.3 GROUP BY & HAVING

```sql
-- GROUP BY
SELECT category, COUNT(*) as product_count
FROM products
GROUP BY category;

-- With aggregate
SELECT category, AVG(price) as avg_price
FROM products
GROUP BY category;

-- Multiple columns
SELECT category, 
       EXTRACT(YEAR FROM created_at) as year,
       COUNT(*) as count
FROM products
GROUP BY category, EXTRACT(YEAR FROM created_at);

-- HAVING (filter after grouping)
SELECT category, AVG(price) as avg_price
FROM products
GROUP BY category
HAVING AVG(price) > 100;

-- WHERE vs HAVING
SELECT category, COUNT(*) as count
FROM products
WHERE price > 50                    -- Filter rows before grouping
GROUP BY category
HAVING COUNT(*) > 5;                -- Filter groups after grouping
```

### 3.4 String Functions

```sql
-- Concatenation
SELECT 'Hello' || ' ' || 'World';
SELECT CONCAT('Hello', ' ', 'World');

-- UPPER and LOWER
SELECT UPPER(username), LOWER(email) FROM users;

-- SUBSTRING
SELECT SUBSTRING(username FROM 1 FOR 3) FROM users;

-- LENGTH
SELECT username, LENGTH(username) as name_length FROM users;

-- LIKE and ILIKE (case-insensitive)
SELECT * FROM users WHERE username LIKE 'j%';      -- Starts with 'j'
SELECT * FROM users WHERE email LIKE '%@gmail.com'; -- Ends with
SELECT * FROM users WHERE username ILIKE 'JOHN%';   -- Case-insensitive

-- TRIM
SELECT TRIM('  hello  ');
SELECT LTRIM('  hello  ');
SELECT RTRIM('  hello  ');

-- REPLACE
SELECT REPLACE(email, '@gmail.com', '@proton.me') FROM users;

-- POSITION
SELECT POSITION('@' IN email) FROM users;
```

### 3.5 Date/Time Functions

```sql
-- Current date and time
SELECT CURRENT_DATE;
SELECT CURRENT_TIME;
SELECT CURRENT_TIMESTAMP;
SELECT NOW();

-- Extract parts
SELECT 
    EXTRACT(YEAR FROM created_at) as year,
    EXTRACT(MONTH FROM created_at) as month,
    EXTRACT(DAY FROM created_at) as day,
    EXTRACT(HOUR FROM created_at) as hour
FROM users;

-- Date arithmetic
SELECT created_at + INTERVAL '7 days' FROM users;
SELECT created_at - INTERVAL '1 month' FROM users;

-- Age calculation
SELECT username, AGE(CURRENT_DATE, created_at) as account_age
FROM users;

-- Date formatting
SELECT TO_CHAR(created_at, 'YYYY-MM-DD') FROM users;
SELECT TO_CHAR(created_at, 'Month DD, YYYY') FROM users;

-- Date comparison
SELECT * FROM users 
WHERE created_at > CURRENT_DATE - INTERVAL '30 days';
```

---

### Practice Problems - Level 2 (Intermediate)

**Problem 6:** Find the average price of products by category

**Solution:**
```sql
SELECT category, AVG(price) as avg_price
FROM products
GROUP BY category
ORDER BY avg_price DESC;
```

**Problem 7:** Find categories that have more than 3 products

**Solution:**
```sql
SELECT category, COUNT(*) as product_count
FROM products
GROUP BY category
HAVING COUNT(*) > 3;
```

**Problem 8:** Find products created in the last 7 days

**Solution:**
```sql
SELECT * FROM products
WHERE created_at >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY created_at DESC;
```

**Problem 9:** Get the top 5 most expensive products

**Solution:**
```sql
SELECT name, price 
FROM products
ORDER BY price DESC
LIMIT 5;
```

**Problem 10:** Find all users whose email ends with '@gmail.com'

**Solution:**
```sql
SELECT * FROM users 
WHERE email LIKE '%@gmail.com';
-- OR
SELECT * FROM users 
WHERE email ~ '@gmail\.com$';  -- Using regex
```

---

## 4. Advanced SQL Queries

### 4.1 Subqueries

```sql
-- Subquery in WHERE
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- Subquery in SELECT
SELECT 
    name,
    price,
    (SELECT AVG(price) FROM products) as avg_price,
    price - (SELECT AVG(price) FROM products) as price_diff
FROM products;

-- Subquery in FROM (Derived table)
SELECT category, avg_price
FROM (
    SELECT category, AVG(price) as avg_price
    FROM products
    GROUP BY category
) as category_averages
WHERE avg_price > 100;

-- EXISTS
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
);

-- NOT EXISTS
SELECT * FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi
    WHERE oi.product_id = p.id
);

-- IN with subquery
SELECT * FROM products
WHERE category IN (
    SELECT DISTINCT category 
    FROM products 
    WHERE price > 100
);

-- ANY/SOME
SELECT * FROM products
WHERE price > ANY (
    SELECT AVG(price) FROM products GROUP BY category
);

-- ALL
SELECT * FROM products
WHERE price > ALL (
    SELECT AVG(price) FROM products GROUP BY category
);
```

### 4.2 Common Table Expressions (CTEs)

```sql
-- Simple CTE
WITH expensive_products AS (
    SELECT * FROM products WHERE price > 100
)
SELECT category, COUNT(*) as count
FROM expensive_products
GROUP BY category;

-- Multiple CTEs
WITH 
category_stats AS (
    SELECT 
        category,
        COUNT(*) as product_count,
        AVG(price) as avg_price
    FROM products
    GROUP BY category
),
high_value_categories AS (
    SELECT category FROM category_stats
    WHERE avg_price > 100
)
SELECT p.*
FROM products p
INNER JOIN high_value_categories hvc ON p.category = hvc.category;

-- Recursive CTE (organizational hierarchy)
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER REFERENCES employees(id)
);

WITH RECURSIVE org_chart AS (
    -- Base case: top-level managers
    SELECT id, name, manager_id, 0 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: employees under managers
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    INNER JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY level, name;
```

### 4.3 Window Functions

Window functions perform calculations across rows related to the current row without collapsing the result set.

```sql
-- ROW_NUMBER()
SELECT 
    name,
    category,
    price,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) as rank_in_category
FROM products;

-- RANK() and DENSE_RANK()
SELECT 
    name,
    price,
    RANK() OVER (ORDER BY price DESC) as rank,
    DENSE_RANK() OVER (ORDER BY price DESC) as dense_rank
FROM products;

-- Running total
SELECT 
    created_at::DATE as date,
    total_amount,
    SUM(total_amount) OVER (ORDER BY created_at) as running_total
FROM orders
ORDER BY created_at;

-- Moving average
SELECT 
    created_at::DATE as date,
    total_amount,
    AVG(total_amount) OVER (
        ORDER BY created_at 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7days
FROM orders;

-- LAG and LEAD (access previous/next row)
SELECT 
    created_at::DATE as date,
    total_amount,
    LAG(total_amount, 1) OVER (ORDER BY created_at) as previous_day,
    LEAD(total_amount, 1) OVER (ORDER BY created_at) as next_day,
    total_amount - LAG(total_amount, 1) OVER (ORDER BY created_at) as day_over_day_change
FROM orders;

-- FIRST_VALUE and LAST_VALUE
SELECT 
    name,
    category,
    price,
    FIRST_VALUE(price) OVER (PARTITION BY category ORDER BY price) as cheapest_in_category,
    LAST_VALUE(price) OVER (
        PARTITION BY category 
        ORDER BY price
        RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as most_expensive_in_category
FROM products;

-- NTILE (divide into buckets)
SELECT 
    name,
    price,
    NTILE(4) OVER (ORDER BY price) as price_quartile
FROM products;
```

### 4.4 CASE Expressions

```sql
-- Simple CASE
SELECT 
    name,
    price,
    CASE 
        WHEN price < 50 THEN 'Budget'
        WHEN price BETWEEN 50 AND 200 THEN 'Mid-range'
        WHEN price > 200 THEN 'Premium'
        ELSE 'Unknown'
    END as price_category
FROM products;

-- Searched CASE with multiple conditions
SELECT 
    username,
    age,
    CASE 
        WHEN age < 18 THEN 'Minor'
        WHEN age BETWEEN 18 AND 30 THEN 'Young Adult'
        WHEN age BETWEEN 31 AND 50 THEN 'Adult'
        ELSE 'Senior'
    END as age_group
FROM users;

-- CASE in aggregation
SELECT 
    category,
    COUNT(*) as total_products,
    COUNT(CASE WHEN price < 100 THEN 1 END) as budget_count,
    COUNT(CASE WHEN price >= 100 AND price < 500 THEN 1 END) as mid_count,
    COUNT(CASE WHEN price >= 500 THEN 1 END) as premium_count
FROM products
GROUP BY category;

-- CASE with UPDATE
UPDATE products
SET stock_status = CASE
    WHEN stock_quantity = 0 THEN 'Out of Stock'
    WHEN stock_quantity < 10 THEN 'Low Stock'
    WHEN stock_quantity < 50 THEN 'In Stock'
    ELSE 'High Stock'
END;
```

### 4.5 UNION, INTERSECT, EXCEPT

```sql
-- UNION (combine results, remove duplicates)
SELECT username FROM users WHERE age < 25
UNION
SELECT username FROM users WHERE email LIKE '%@gmail.com';

-- UNION ALL (keep duplicates)
SELECT username FROM users WHERE age < 25
UNION ALL
SELECT username FROM users WHERE email LIKE '%@gmail.com';

-- INTERSECT (common rows)
SELECT username FROM users WHERE age < 30
INTERSECT
SELECT username FROM users WHERE email LIKE '%@gmail.com';

-- EXCEPT (rows in first query but not in second)
SELECT username FROM users WHERE age < 30
EXCEPT
SELECT username FROM users WHERE email LIKE '%@gmail.com';
```

---

### Practice Problems - Level 3 (Advanced Queries)

**Problem 11:** Find the 2nd highest price for each category

**Solution:**
```sql
WITH ranked_products AS (
    SELECT 
        name,
        category,
        price,
        DENSE_RANK() OVER (PARTITION BY category ORDER BY price DESC) as rank
    FROM products
)
SELECT name, category, price
FROM ranked_products
WHERE rank = 2;
```

**Problem 12:** Calculate running total of sales by date

**Solution:**
```sql
SELECT 
    created_at::DATE as date,
    SUM(total_amount) as daily_total,
    SUM(SUM(total_amount)) OVER (ORDER BY created_at::DATE) as running_total
FROM orders
GROUP BY created_at::DATE
ORDER BY date;
```

**Problem 13:** Find products that have never been ordered

**Solution:**
```sql
SELECT p.*
FROM products p
WHERE NOT EXISTS (
    SELECT 1 
    FROM order_items oi 
    WHERE oi.product_id = p.id
);

-- Alternative using LEFT JOIN
SELECT p.*
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
WHERE oi.id IS NULL;
```

**Problem 14:** Get top 3 products by revenue in each category

**Solution:**
```sql
WITH product_revenue AS (
    SELECT 
        p.id,
        p.name,
        p.category,
        SUM(oi.quantity * oi.price) as revenue,
        ROW_NUMBER() OVER (PARTITION BY p.category ORDER BY SUM(oi.quantity * oi.price) DESC) as rank
    FROM products p
    INNER JOIN order_items oi ON p.id = oi.product_id
    GROUP BY p.id, p.name, p.category
)
SELECT id, name, category, revenue
FROM product_revenue
WHERE rank <= 3
ORDER BY category, rank;
```

**Problem 15:** Find users who made purchases in consecutive months

**Solution:**
```sql
WITH monthly_users AS (
    SELECT DISTINCT
        user_id,
        EXTRACT(YEAR FROM created_at) as year,
        EXTRACT(MONTH FROM created_at) as month
    FROM orders
),
consecutive_months AS (
    SELECT 
        user_id,
        year,
        month,
        LAG(month) OVER (PARTITION BY user_id ORDER BY year, month) as prev_month,
        LAG(year) OVER (PARTITION BY user_id ORDER BY year, month) as prev_year
    FROM monthly_users
)
SELECT DISTINCT user_id
FROM consecutive_months
WHERE (year = prev_year AND month = prev_month + 1)
   OR (year = prev_year + 1 AND month = 1 AND prev_month = 12);
```

---

## 5. PostgreSQL Advanced Features

### 5.1 JSON/JSONB Operations

```sql
-- Create table with JSONB
CREATE TABLE users_metadata (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    preferences JSONB,
    settings JSONB
);

-- Insert JSON data
INSERT INTO users_metadata (user_id, preferences) VALUES
    (1, '{"theme": "dark", "language": "en", "notifications": {"email": true, "sms": false}}'),
    (2, '{"theme": "light", "language": "bn", "notifications": {"email": false, "sms": true}}');

-- Query JSON fields
SELECT preferences->>'theme' as theme FROM users_metadata;
SELECT preferences->'notifications'->>'email' as email_notif FROM users_metadata;

-- JSON operators
-- -> returns JSON object
-- ->> returns text
-- #> access nested path (returns JSON)
-- #>> access nested path (returns text)

SELECT 
    preferences #>> '{notifications, email}' as email_notif
FROM users_metadata;

-- Query by JSON value
SELECT * FROM users_metadata 
WHERE preferences->>'theme' = 'dark';

SELECT * FROM users_metadata 
WHERE preferences->'notifications'->>'email' = 'true';

-- JSON contains
SELECT * FROM users_metadata 
WHERE preferences @> '{"theme": "dark"}';

-- JSON exists
SELECT * FROM users_metadata 
WHERE preferences ? 'theme';

-- JSON array operations
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    tags JSONB
);

INSERT INTO articles (title, tags) VALUES
    ('SQL Guide', '["database", "sql", "tutorial"]'),
    ('Python Tips', '["python", "programming", "tips"]');

SELECT * FROM articles WHERE tags @> '["sql"]';

-- JSON aggregation
SELECT jsonb_agg(username) as all_users FROM users;
SELECT jsonb_object_agg(id, username) as user_map FROM users;

-- Build JSON
SELECT jsonb_build_object(
    'id', id,
    'username', username,
    'email', email
) as user_json
FROM users;

-- Update JSON field
UPDATE users_metadata
SET preferences = jsonb_set(
    preferences,
    '{theme}',
    '"light"'
)
WHERE user_id = 1;
```

### 5.2 Array Operations

```sql
-- Create table with arrays
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    subjects TEXT[],
    scores INTEGER[]
);

INSERT INTO students (name, subjects, scores) VALUES
    ('Alice', ARRAY['Math', 'Physics', 'Chemistry'], ARRAY[95, 88, 92]),
    ('Bob', ARRAY['Math', 'Biology', 'English'], ARRAY[78, 85, 90]);

-- Array access
SELECT name, subjects[1] as first_subject FROM students;

-- Array contains
SELECT * FROM students WHERE 'Math' = ANY(subjects);
SELECT * FROM students WHERE subjects @> ARRAY['Math'];

-- Array length
SELECT name, array_length(subjects, 1) as num_subjects FROM students;

-- Unnest array
SELECT name, unnest(subjects) as subject FROM students;

SELECT name, unnest(subjects) as subject, unnest(scores) as score 
FROM students;

-- Array aggregation
SELECT array_agg(name) as all_students FROM students;

-- Array operations
SELECT ARRAY[1,2,3] || ARRAY[4,5,6];  -- Concatenation
SELECT ARRAY[1,2,3] || 4;              -- Append element
```

### 5.3 Full-Text Search

```sql
-- Create table for articles
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    search_vector tsvector
);

-- Basic full-text search
SELECT * FROM articles 
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'postgresql');

-- Create index for better performance
CREATE INDEX idx_article_search ON articles USING GIN(to_tsvector('english', content));

-- Combine title and content search
SELECT * FROM articles 
WHERE to_tsvector('english', title || ' ' || content) 
      @@ to_tsquery('english', 'postgresql & database');

-- Use generated column for search vector
ALTER TABLE articles 
ADD COLUMN search_vector tsvector 
GENERATED ALWAYS AS (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, ''))
) STORED;

CREATE INDEX idx_articles_search_vector ON articles USING GIN(search_vector);

-- Search with ranking
SELECT 
    title,
    ts_rank(search_vector, query) as rank
FROM articles, to_tsquery('english', 'postgresql') query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- Phrase search
SELECT * FROM articles 
WHERE to_tsvector('english', content) @@ phraseto_tsquery('english', 'open source database');

-- Search with highlighting
SELECT 
    title,
    ts_headline('english', content, to_tsquery('english', 'postgresql'), 
                'MaxWords=50, MinWords=25, MaxFragments=2') as snippet
FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql');
```

### 5.4 Generate Series and Test Data

```sql
-- Generate number series
SELECT * FROM generate_series(1, 10);

-- Generate date series
SELECT * FROM generate_series(
    '2024-01-01'::DATE,
    '2024-12-31'::DATE,
    '1 day'::INTERVAL
);

-- Create test data
INSERT INTO orders (user_id, total_amount, created_at)
SELECT 
    (random() * 100)::INTEGER + 1 as user_id,
    (random() * 1000)::DECIMAL(10,2) as total_amount,
    timestamp '2024-01-01' + random() * (timestamp '2024-12-31' - timestamp '2024-01-01') as created_at
FROM generate_series(1, 10000);

-- Random data functions
SELECT 
    random(),                           -- Random between 0 and 1
    (random() * 100)::INTEGER,         -- Random integer 0-100
    md5(random()::TEXT),               -- Random hash
    substring(md5(random()::TEXT) from 1 for 8) as random_string;
```

### 5.5 Transactions and Isolation Levels

```sql
-- Basic transaction
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Rollback on error
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    -- Some error happens
ROLLBACK;

-- Savepoints
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    SAVEPOINT my_savepoint;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
    -- Oops, rollback just the second update
    ROLLBACK TO SAVEPOINT my_savepoint;
    -- Try different update
    UPDATE accounts SET balance = balance + 100 WHERE id = 3;
COMMIT;

-- Isolation levels
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Lock rows
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- Exclusive lock
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- Share lock
SELECT * FROM accounts WHERE id = 1 FOR SHARE;  -- Others can read but not update
```

### 5.6 Views and Materialized Views

```sql
-- Create view
CREATE VIEW user_order_summary AS
SELECT 
    u.id,
    u.username,
    COUNT(o.id) as total_orders,
    SUM(o.total_amount) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username;

-- Use view
SELECT * FROM user_order_summary WHERE total_orders > 5;

-- Updatable views (simple views)
CREATE VIEW active_users AS
SELECT * FROM users WHERE is_active = true;

UPDATE active_users SET email = 'new@email.com' WHERE id = 1;

-- Materialized view (stores results)
CREATE MATERIALIZED VIEW daily_sales AS
SELECT 
    created_at::DATE as date,
    COUNT(*) as order_count,
    SUM(total_amount) as total_sales
FROM orders
GROUP BY created_at::DATE;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW daily_sales;

-- Concurrent refresh (non-blocking)
CREATE UNIQUE INDEX ON daily_sales (date);
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;
```

### 5.7 Triggers and Functions

```sql
-- Create function
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER update_user_timestamp
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();

-- Another example: audit log
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name TEXT,
    operation TEXT,
    old_data JSONB,
    new_data JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION log_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_data)
        VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD));
        RETURN OLD;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD), row_to_json(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, row_to_json(NEW));
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_users
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW
    EXECUTE FUNCTION log_changes();
```

---

## 6. Complex Joins & Relationships

### 6.1 Types of Joins

```sql
-- Sample tables
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    order_date DATE,
    total_amount DECIMAL(10,2)
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER,
    price DECIMAL(10,2)
);

-- INNER JOIN (only matching rows)
SELECT 
    c.name,
    o.order_date,
    o.total_amount
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;

-- LEFT JOIN (all from left table)
SELECT 
    c.name,
    COUNT(o.id) as order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

-- RIGHT JOIN (all from right table)
SELECT 
    o.order_date,
    c.name
FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.id;

-- FULL OUTER JOIN (all from both)
SELECT 
    c.name,
    o.order_date
FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id;

-- CROSS JOIN (Cartesian product)
SELECT 
    c.name,
    p.name
FROM customers c
CROSS JOIN products p;

-- SELF JOIN (join table to itself)
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER REFERENCES employees(id)
);

SELECT 
    e.name as employee,
    m.name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

### 6.2 Complex Multi-Table Queries

```sql
-- 3-table join
SELECT 
    c.name as customer_name,
    o.order_date,
    p.name as product_name,
    oi.quantity,
    oi.price
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id;

-- Find customers with their total spending and product categories
SELECT 
    c.id,
    c.name,
    COUNT(DISTINCT o.id) as total_orders,
    COUNT(DISTINCT p.category) as categories_purchased,
    SUM(oi.quantity * oi.price) as total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
LEFT JOIN order_items oi ON o.id = oi.order_id
LEFT JOIN products p ON oi.product_id = p.id
GROUP BY c.id, c.name
HAVING SUM(oi.quantity * oi.price) > 1000;

-- Find customers who bought products from all categories
SELECT c.id, c.name
FROM customers c
WHERE (
    SELECT COUNT(DISTINCT p.category)
    FROM orders o
    JOIN order_items oi ON o.id = oi.order_id
    JOIN products p ON oi.product_id = p.id
    WHERE o.customer_id = c.id
) = (
    SELECT COUNT(DISTINCT category) FROM products
);
```

### 6.3 Advanced Join Patterns

```sql
-- LATERAL JOIN (correlated subquery in FROM)
SELECT 
    c.name,
    latest_orders.order_date,
    latest_orders.total_amount
FROM customers c
LEFT JOIN LATERAL (
    SELECT order_date, total_amount
    FROM orders
    WHERE customer_id = c.id
    ORDER BY order_date DESC
    LIMIT 3
) latest_orders ON true;

-- Multiple conditions in JOIN
SELECT 
    c.name,
    o.order_date
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id 
    AND o.order_date >= '2024-01-01'
    AND o.total_amount > 100;

-- Anti-join (find customers with no orders)
SELECT c.*
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.id IS NULL;

-- Semi-join (find customers who have ordered)
SELECT DISTINCT c.*
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;
```

---

### Practice Problems - Level 4 (Joins & Complex Queries)

**Problem 16:** Find customers who have made purchases in every month of 2024

**Solution:**
```sql
SELECT c.id, c.name
FROM customers c
WHERE (
    SELECT COUNT(DISTINCT EXTRACT(MONTH FROM o.order_date))
    FROM orders o
    WHERE o.customer_id = c.id
    AND EXTRACT(YEAR FROM o.order_date) = 2024
) = 12;
```

**Problem 17:** Find pairs of customers who bought the same products

**Solution:**
```sql
SELECT DISTINCT 
    c1.name as customer1,
    c2.name as customer2,
    p.name as common_product
FROM customers c1
JOIN orders o1 ON c1.id = o1.customer_id
JOIN order_items oi1 ON o1.id = oi1.order_id
JOIN products p ON oi1.product_id = p.id
JOIN order_items oi2 ON p.id = oi2.product_id
JOIN orders o2 ON oi2.order_id = o2.id
JOIN customers c2 ON o2.customer_id = c2.id
WHERE c1.id < c2.id
ORDER BY customer1, customer2, common_product;
```

**Problem 18:** Get each customer's favorite product (most ordered)

**Solution:**
```sql
WITH customer_product_counts AS (
    SELECT 
        c.id as customer_id,
        c.name as customer_name,
        p.id as product_id,
        p.name as product_name,
        SUM(oi.quantity) as total_quantity,
        ROW_NUMBER() OVER (PARTITION BY c.id ORDER BY SUM(oi.quantity) DESC) as rank
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    JOIN order_items oi ON o.id = oi.order_id
    JOIN products p ON oi.product_id = p.id
    GROUP BY c.id, c.name, p.id, p.name
)
SELECT customer_name, product_name, total_quantity
FROM customer_product_counts
WHERE rank = 1;
```

**Problem 19:** Find products that are frequently bought together

**Solution:**
```sql
SELECT 
    p1.name as product1,
    p2.name as product2,
    COUNT(*) as times_bought_together
FROM order_items oi1
JOIN order_items oi2 ON oi1.order_id = oi2.order_id AND oi1.product_id < oi2.product_id
JOIN products p1 ON oi1.product_id = p1.id
JOIN products p2 ON oi2.product_id = p2.id
GROUP BY p1.id, p1.name, p2.id, p2.name
HAVING COUNT(*) >= 5
ORDER BY times_bought_together DESC;
```

**Problem 20:** Calculate customer lifetime value with cohort analysis

**Solution:**
```sql
WITH customer_cohorts AS (
    SELECT 
        c.id,
        c.name,
        DATE_TRUNC('month', MIN(o.order_date)) as cohort_month
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    GROUP BY c.id, c.name
),
customer_orders AS (
    SELECT 
        c.id,
        DATE_TRUNC('month', o.order_date) as order_month,
        SUM(o.total_amount) as monthly_revenue
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    GROUP BY c.id, DATE_TRUNC('month', o.order_date)
)
SELECT 
    cc.cohort_month,
    EXTRACT(YEAR FROM AGE(co.order_month, cc.cohort_month)) * 12 +
    EXTRACT(MONTH FROM AGE(co.order_month, cc.cohort_month)) as months_since_first_order,
    COUNT(DISTINCT co.id) as customers,
    SUM(co.monthly_revenue) as total_revenue,
    AVG(co.monthly_revenue) as avg_revenue_per_customer
FROM customer_cohorts cc
JOIN customer_orders co ON cc.id = co.id
GROUP BY cc.cohort_month, months_since_first_order
ORDER BY cc.cohort_month, months_since_first_order;
```

---

## 7. Query Optimization & Performance

### 7.1 EXPLAIN and ANALYZE

```sql
-- EXPLAIN shows query plan
EXPLAIN SELECT * FROM orders WHERE customer_id = 100;

-- EXPLAIN ANALYZE actually runs query and shows real timing
EXPLAIN ANALYZE 
SELECT * FROM orders WHERE customer_id = 100;

-- EXPLAIN with more details
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM orders WHERE customer_id = 100;
```

**Reading EXPLAIN output:**
- Seq Scan - Full table scan (slow for large tables)
- Index Scan - Uses index (fast)
- Index Only Scan - Gets data from index without accessing table (fastest)
- Bitmap Index Scan - Scans index and creates bitmap
- Bitmap Heap Scan - Uses bitmap to fetch rows
- Nested Loop - Joins small tables
- Hash Join - Joins medium tables
- Merge Join - Joins large sorted tables

### 7.2 Indexes

```sql
-- B-tree index (default, most common)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- Multi-column index
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Partial index (smaller, faster)
CREATE INDEX idx_orders_pending ON orders(status) 
WHERE status = 'pending';

-- Expression index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- Unique index
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- GIN index (for arrays, JSONB, full-text)
CREATE INDEX idx_products_tags ON products USING GIN(tags);
CREATE INDEX idx_users_preferences ON users_metadata USING GIN(preferences);

-- GiST index (for spatial data, ranges)
CREATE INDEX idx_events_time_range ON events USING GIST(time_range);

-- Hash index (equality only, rarely used)
CREATE INDEX idx_users_username ON users USING HASH(username);

-- List indexes
\di

-- Get index usage stats
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as index_scans,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- Find unused indexes
SELECT 
    schemaname,
    tablename,
    indexname
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexname NOT LIKE '%_pkey';

-- Drop index
DROP INDEX idx_orders_customer_date;
```

### 7.3 Query Optimization Techniques

```sql
-- Bad: SELECT *
SELECT * FROM orders;  -- Don't do this

-- Good: Select only needed columns
SELECT id, customer_id, total_amount FROM orders;

-- Bad: Functions on indexed columns prevent index usage
SELECT * FROM users WHERE UPPER(email) = 'TEST@EXAMPLE.COM';

-- Good: Use expression index or transform comparison
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- Bad: Leading wildcards prevent index usage
SELECT * FROM users WHERE email LIKE '%gmail.com';

-- Good: Trailing wildcards can use index
SELECT * FROM users WHERE email LIKE 'user@%';

-- Bad: OR conditions can be slow
SELECT * FROM orders WHERE customer_id = 1 OR customer_id = 2;

-- Good: Use IN
SELECT * FROM orders WHERE customer_id IN (1, 2);

-- Bad: Subquery in SELECT executed for each row
SELECT 
    c.name,
    (SELECT COUNT(*) FROM orders WHERE customer_id = c.id) as order_count
FROM customers c;

-- Good: Use JOIN
SELECT 
    c.name,
    COUNT(o.id) as order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

-- Bad: COUNT(*) on large tables
SELECT COUNT(*) FROM orders;  -- Very slow on huge tables

-- Good: Estimate from statistics
SELECT reltuples::BIGINT as estimate 
FROM pg_class 
WHERE relname = 'orders';

-- Optimize with LIMIT
SELECT * FROM orders ORDER BY created_at DESC LIMIT 100;

-- Use EXISTS instead of COUNT when checking existence
-- Bad
SELECT * FROM customers WHERE (SELECT COUNT(*) FROM orders WHERE customer_id = customers.id) > 0;

-- Good
SELECT * FROM customers WHERE EXISTS (SELECT 1 FROM orders WHERE customer_id = customers.id);
```

### 7.4 Connection Pooling and Configuration

```sql
-- Check current connections
SELECT COUNT(*) FROM pg_stat_activity;

-- Show connection details
SELECT 
    datname,
    usename,
    application_name,
    client_addr,
    state,
    query
FROM pg_stat_activity;

-- Kill idle connections
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
AND state_change < CURRENT_TIMESTAMP - INTERVAL '30 minutes';
```

**Important PostgreSQL configurations (postgresql.conf):**
- shared_buffers = 25% of RAM (up to 8-16GB)
- effective_cache_size = 50-75% of RAM
- work_mem = (Total RAM / max_connections) / 2
- maintenance_work_mem = RAM / 16
- max_connections = 100-200 (use connection pooler like PgBouncer)
- random_page_cost = 1.1 (for SSD, default 4 for HDD)
- effective_io_concurrency = 200 (for SSD)

### 7.5 Vacuum and Maintenance

```sql
-- Manual vacuum
VACUUM orders;

-- Vacuum with analyze
VACUUM ANALYZE orders;

-- Full vacuum (locks table, reclaims space)
VACUUM FULL orders;

-- Analyze only (update statistics)
ANALYZE orders;

-- Check table bloat
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    n_tup_ins as inserts,
    n_tup_upd as updates,
    n_tup_del as deletes,
    n_dead_tup as dead_tuples,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Reindex
REINDEX TABLE orders;
REINDEX INDEX idx_orders_customer_id;
```

**Auto-vacuum settings (postgresql.conf):**
- autovacuum = on
- autovacuum_max_workers = 3
- autovacuum_naptime = 1min

---

### Practice Problems - Level 5 (Performance)

**Problem 21:** Identify slow queries in your database

**Solution:**
```sql
-- Enable pg_stat_statements extension first
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find slowest queries
SELECT 
    substring(query, 1, 50) as short_query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time,
    stddev_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

**Problem 22:** Find tables that need indexing

**Solution:**
```sql
-- Tables with lots of sequential scans
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / seq_scan as avg_seq_read
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 20;
```

**Problem 23:** Optimize a query that's doing a sequential scan

**Solution:**
```sql
-- Before optimization
EXPLAIN ANALYZE
SELECT * FROM orders 
WHERE customer_id = 100 AND order_date > '2024-01-01';

-- Result shows Seq Scan
-- Seq Scan on orders (cost=0.00..1829.00 rows=100 width=40) (actual time=0.023..15.234 rows=100 loops=1)
-- Filter: ((customer_id = 100) AND (order_date > '2024-01-01'::date))

-- Create composite index
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- After optimization
EXPLAIN ANALYZE
SELECT * FROM orders 
WHERE customer_id = 100 AND order_date > '2024-01-01';

-- Now uses Index Scan
-- Index Scan using idx_orders_customer_date on orders (cost=0.29..8.31 rows=100 width=40) (actual time=0.012..0.045 rows=100 loops=1)
-- Index Cond: ((customer_id = 100) AND (order_date > '2024-01-01'::date))
```

**Problem 24:** Optimize aggregation query on large table

**Solution:**
```sql
-- Slow query
SELECT 
    customer_id,
    COUNT(*) as order_count,
    SUM(total_amount) as total_spent
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 10;

-- Solution 1: Use materialized view
CREATE MATERIALIZED VIEW customer_stats AS
SELECT 
    customer_id,
    COUNT(*) as order_count,
    SUM(total_amount) as total_spent,
    AVG(total_amount) as avg_order
FROM orders
GROUP BY customer_id;

CREATE INDEX idx_customer_stats_order_count ON customer_stats(order_count);

-- Query materialized view (much faster)
SELECT * FROM customer_stats WHERE order_count > 10;

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY customer_stats;

-- Solution 2: Incremental aggregation table
CREATE TABLE customer_aggregates (
    customer_id INTEGER PRIMARY KEY,
    order_count INTEGER DEFAULT 0,
    total_spent DECIMAL(10,2) DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Update with trigger
CREATE OR REPLACE FUNCTION update_customer_aggregates()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO customer_aggregates (customer_id, order_count, total_spent)
        VALUES (NEW.customer_id, 1, NEW.total_amount)
        ON CONFLICT (customer_id) DO UPDATE
        SET order_count = customer_aggregates.order_count + 1,
            total_spent = customer_aggregates.total_spent + NEW.total_amount,
            updated_at = CURRENT_TIMESTAMP;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE customer_aggregates
        SET order_count = order_count - 1,
            total_spent = total_spent - OLD.total_amount,
            updated_at = CURRENT_TIMESTAMP
        WHERE customer_id = OLD.customer_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_customer_agg_trigger
    AFTER INSERT OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_customer_aggregates();
```

**Problem 25:** Implement efficient pagination

**Solution:**
```sql
-- Bad: OFFSET gets slower as page number increases
SELECT * FROM orders 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 100000;  -- Very slow!

-- Good: Keyset pagination (cursor-based)
-- First page
SELECT * FROM orders 
ORDER BY created_at DESC, id DESC 
LIMIT 20;

-- Next page (use last created_at and id from previous page)
SELECT * FROM orders 
WHERE (created_at, id) < ('2024-01-15 10:30:00', 12345)
ORDER BY created_at DESC, id DESC 
LIMIT 20;

-- Create index for this query
CREATE INDEX idx_orders_created_id ON orders(created_at DESC, id DESC);
```

---

## 8. Real-World Backend Problems

These problems are based on common scenarios you'll face as a backend engineer at GTAF or similar companies.

### Problem 26: Multi-tenant Database Design

**Scenario:** You're building a SaaS application where each organization has its own data. Design a multi-tenant database.

**Solution:**
```sql
-- Approach 1: Shared schema with tenant_id (most common)
CREATE TABLE tenants (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    subdomain VARCHAR(50) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenants(id),
    email VARCHAR(100) NOT NULL,
    username VARCHAR(50) NOT NULL,
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

-- Row Level Security (RLS) for automatic filtering
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_policy ON users
    USING (tenant_id = current_setting('app.current_tenant')::INTEGER);

-- Set tenant context in application
SET app.current_tenant = '123';

-- Now all queries automatically filter by tenant
SELECT * FROM users;  -- Only sees users for tenant 123

-- Approach 2: Schema per tenant (better isolation)
CREATE SCHEMA tenant_123;
CREATE SCHEMA tenant_456;

CREATE TABLE tenant_123.users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(100),
    username VARCHAR(50)
);

-- Approach 3: Database per tenant (maximum isolation, harder to maintain)
CREATE DATABASE tenant_123_db;
```

### Problem 27: Soft Delete Implementation

**Scenario:** Implement soft delete for orders so deleted records can be recovered.

**Solution:**
```sql
ALTER TABLE orders ADD COLUMN deleted_at TIMESTAMP;

-- Create index for active records
CREATE INDEX idx_orders_active ON orders(id) WHERE deleted_at IS NULL;

-- View for active orders
CREATE VIEW active_orders AS
SELECT * FROM orders WHERE deleted_at IS NULL;

-- Soft delete function
CREATE OR REPLACE FUNCTION soft_delete_order(order_id INTEGER)
RETURNS VOID AS $$
BEGIN
    UPDATE orders 
    SET deleted_at = CURRENT_TIMESTAMP 
    WHERE id = order_id AND deleted_at IS NULL;
END;
$$ LANGUAGE plpgsql;

-- Restore function
CREATE OR REPLACE FUNCTION restore_order(order_id INTEGER)
RETURNS VOID AS $$
BEGIN
    UPDATE orders 
    SET deleted_at = NULL 
    WHERE id = order_id;
END;
$$ LANGUAGE plpgsql;

-- Query active orders
SELECT * FROM active_orders;

-- Query deleted orders
SELECT * FROM orders WHERE deleted_at IS NOT NULL;

-- Permanent delete after 30 days (run this in a cron job)
DELETE FROM orders 
WHERE deleted_at < CURRENT_DATE - INTERVAL '30 days';
```

### Problem 28: Rate Limiting with PostgreSQL

**Scenario:** Implement API rate limiting: 100 requests per hour per user.

**Solution:**
```sql
CREATE TABLE api_requests (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    endpoint VARCHAR(200),
    request_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address INET
);

-- Index for fast lookups
CREATE INDEX idx_api_requests_user_time ON api_requests(user_id, request_time DESC);

-- Function to check rate limit
CREATE OR REPLACE FUNCTION check_rate_limit(
    p_user_id INTEGER,
    p_limit INTEGER DEFAULT 100,
    p_window INTERVAL DEFAULT '1 hour'
)
RETURNS BOOLEAN AS $$
DECLARE
    request_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO request_count
    FROM api_requests
    WHERE user_id = p_user_id
    AND request_time > CURRENT_TIMESTAMP - p_window;
    
    RETURN request_count < p_limit;
END;
$$ LANGUAGE plpgsql;

-- Log request
CREATE OR REPLACE FUNCTION log_api_request(
    p_user_id INTEGER,
    p_endpoint VARCHAR,
    p_ip INET
)
RETURNS BOOLEAN AS $$
BEGIN
    IF check_rate_limit(p_user_id) THEN
        INSERT INTO api_requests (user_id, endpoint, ip_address)
        VALUES (p_user_id, p_endpoint, p_ip);
        RETURN TRUE;
    ELSE
        RETURN FALSE;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Usage in application
SELECT log_api_request(123, '/api/products', '192.168.1.1'::INET);

-- Cleanup old requests (run periodically)
DELETE FROM api_requests 
WHERE request_time < CURRENT_TIMESTAMP - INTERVAL '7 days';
```

### Problem 29: Implementing a Notification System

**Scenario:** Build a notification system that tracks read/unread status efficiently.

**Solution:**
```sql
CREATE TABLE notifications (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    type VARCHAR(50) NOT NULL,
    title VARCHAR(200) NOT NULL,
    message TEXT,
    data JSONB,
    is_read BOOLEAN DEFAULT FALSE,
    read_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_notifications_user_unread ON notifications(user_id, created_at DESC) 
WHERE is_read = FALSE;

CREATE INDEX idx_notifications_user_read ON notifications(user_id, created_at DESC) 
WHERE is_read = TRUE;

-- Get unread count (very fast with partial index)
CREATE OR REPLACE FUNCTION get_unread_count(p_user_id INTEGER)
RETURNS INTEGER AS $$
    SELECT COUNT(*)::INTEGER
    FROM notifications
    WHERE user_id = p_user_id AND is_read = FALSE;
$$ LANGUAGE SQL STABLE;

-- Mark as read
CREATE OR REPLACE FUNCTION mark_notification_read(p_notification_id INTEGER)
RETURNS VOID AS $$
BEGIN
    UPDATE notifications
    SET is_read = TRUE,
        read_at = CURRENT_TIMESTAMP
    WHERE id = p_notification_id AND is_read = FALSE;
END;
$$ LANGUAGE plpgsql;

-- Mark all as read
CREATE OR REPLACE FUNCTION mark_all_read(p_user_id INTEGER)
RETURNS INTEGER AS $$
DECLARE
    updated_count INTEGER;
BEGIN
    UPDATE notifications
    SET is_read = TRUE,
        read_at = CURRENT_TIMESTAMP
    WHERE user_id = p_user_id AND is_read = FALSE;
    
    GET DIAGNOSTICS updated_count = ROW_COUNT;
    RETURN updated_count;
END;
$$ LANGUAGE plpgsql;

-- Get paginated notifications
SELECT 
    id,
    type,
    title,
    message,
    is_read,
    created_at
FROM notifications
WHERE user_id = 123
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;

-- Auto-delete old read notifications (run daily)
DELETE FROM notifications
WHERE is_read = TRUE 
AND read_at < CURRENT_DATE - INTERVAL '30 days';
```

### Problem 30: Audit Trail for Sensitive Data

**Scenario:** Track all changes to sensitive user data (like email, password changes).

**Solution:**
```sql
CREATE TABLE user_audit_log (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    action VARCHAR(50) NOT NULL, -- 'email_changed', 'password_changed', etc.
    old_value TEXT,
    new_value TEXT,
    changed_by INTEGER, -- admin or user themselves
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_user_time ON user_audit_log(user_id, created_at DESC);

-- Trigger function for email changes
CREATE OR REPLACE FUNCTION log_email_change()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.email IS DISTINCT FROM NEW.email THEN
        INSERT INTO user_audit_log (
            user_id, 
            action, 
            old_value, 
            new_value, 
            changed_by
        )
        VALUES (
            NEW.id,
            'email_changed',
            OLD.email,
            NEW.email,
            NEW.id  -- In production, get this from application context
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER email_change_audit
    AFTER UPDATE ON users
    FOR EACH ROW
    WHEN (OLD.email IS DISTINCT FROM NEW.email)
    EXECUTE FUNCTION log_email_change();

-- Get user's change history
SELECT 
    action,
    old_value,
    new_value,
    created_at
FROM user_audit_log
WHERE user_id = 123
ORDER BY created_at DESC;

-- Encrypt sensitive data in audit log (using pgcrypto)
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Store encrypted values
INSERT INTO user_audit_log (user_id, action, old_value, new_value)
VALUES (
    123,
    'password_changed',
    crypt('old_password', gen_salt('bf')),
    crypt('new_password', gen_salt('bf'))
);
```

### Problem 31: Implementing Prayer Times (Islamic App Context)

**Scenario:** Store and query prayer times for different locations (relevant to your GTAF work).

**Solution:**
```sql
CREATE TABLE cities (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    country VARCHAR(100),
    latitude DECIMAL(10, 6) NOT NULL,
    longitude DECIMAL(10, 6) NOT NULL,
    timezone VARCHAR(50) NOT NULL
);

CREATE TABLE prayer_times (
    id SERIAL PRIMARY KEY,
    city_id INTEGER REFERENCES cities(id),
    date DATE NOT NULL,
    fajr TIME NOT NULL,
    sunrise TIME NOT NULL,
    dhuhr TIME NOT NULL,
    asr TIME NOT NULL,
    maghrib TIME NOT NULL,
    isha TIME NOT NULL,
    calculation_method VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(city_id, date)
);

CREATE INDEX idx_prayer_times_city_date ON prayer_times(city_id, date);

-- Get today's prayer times for a city
SELECT 
    c.name as city,
    pt.date,
    pt.fajr,
    pt.dhuhr,
    pt.asr,
    pt.maghrib,
    pt.isha
FROM prayer_times pt
JOIN cities c ON pt.city_id = c.id
WHERE c.id = 1
AND pt.date = CURRENT_DATE;

-- Get next prayer
CREATE OR REPLACE FUNCTION get_next_prayer(
    p_city_id INTEGER,
    p_current_time TIME DEFAULT CURRENT_TIME
)
RETURNS TABLE(
    prayer_name VARCHAR,
    prayer_time TIME
) AS $$
BEGIN
    RETURN QUERY
    SELECT * FROM (
        SELECT 'Fajr'::VARCHAR, fajr FROM prayer_times WHERE city_id = p_city_id AND date = CURRENT_DATE AND fajr > p_current_time
        UNION ALL
        SELECT 'Dhuhr'::VARCHAR, dhuhr FROM prayer_times WHERE city_id = p_city_id AND date = CURRENT_DATE AND dhuhr > p_current_time
        UNION ALL
        SELECT 'Asr'::VARCHAR, asr FROM prayer_times WHERE city_id = p_city_id AND date = CURRENT_DATE AND asr > p_current_time
        UNION ALL
        SELECT 'Maghrib'::VARCHAR, maghrib FROM prayer_times WHERE city_id = p_city_id AND date = CURRENT_DATE AND maghrib > p_current_time
        UNION ALL
        SELECT 'Isha'::VARCHAR, isha FROM prayer_times WHERE city_id = p_city_id AND date = CURRENT_DATE AND isha > p_current_time
    ) prayers
    ORDER BY prayer_time
    LIMIT 1;
END;
$$ LANGUAGE plpgsql;

-- Get prayer times for a date range
SELECT * FROM prayer_times
WHERE city_id = 1
AND date BETWEEN '2024-01-01' AND '2024-01-31'
ORDER BY date;

-- Find cities near a location (using PostGIS if available)
-- Without PostGIS (simple distance calculation)
CREATE OR REPLACE FUNCTION haversine_distance(
    lat1 DECIMAL, lon1 DECIMAL,
    lat2 DECIMAL, lon2 DECIMAL
) RETURNS DECIMAL AS $$
DECLARE
    r DECIMAL := 6371; -- Earth's radius in km
    dlat DECIMAL;
    dlon DECIMAL;
    a DECIMAL;
    c DECIMAL;
BEGIN
    dlat := radians(lat2 - lat1);
    dlon := radians(lon2 - lon1);
    a := sin(dlat/2) * sin(dlat/2) + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2) * sin(dlon/2);
    c := 2 * atan2(sqrt(a), sqrt(1-a));
    RETURN r * c;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Find nearest cities
SELECT 
    id,
    name,
    country,
    haversine_distance(23.8103, 90.4125, latitude, longitude) as distance_km
FROM cities
ORDER BY distance_km
LIMIT 10;
```

### Problem 32: Quran Search with Arabic Text

**Scenario:** Implement efficient search for Quran verses in Arabic and translations.

**Solution:**
```sql
CREATE TABLE quran_verses (
    id SERIAL PRIMARY KEY,
    surah_number INTEGER NOT NULL,
    verse_number INTEGER NOT NULL,
    arabic_text TEXT NOT NULL,
    arabic_search tsvector,
    UNIQUE(surah_number, verse_number)
);

CREATE TABLE quran_translations (
    id SERIAL PRIMARY KEY,
    verse_id INTEGER REFERENCES quran_verses(id),
    language VARCHAR(10) NOT NULL, -- 'en', 'bn', 'ur', etc.
    translator VARCHAR(100),
    text TEXT NOT NULL,
    search_vector tsvector
);

-- Create full-text search indexes
CREATE INDEX idx_quran_arabic_search ON quran_verses USING GIN(arabic_search);
CREATE INDEX idx_translations_search ON quran_translations USING GIN(search_vector);

-- Populate search vectors (using trigger)
CREATE OR REPLACE FUNCTION update_quran_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := to_tsvector('english', NEW.text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER quran_translation_search_update
    BEFORE INSERT OR UPDATE ON quran_translations
    FOR EACH ROW
    EXECUTE FUNCTION update_quran_search_vector();

-- Search in translations
SELECT 
    v.surah_number,
    v.verse_number,
    v.arabic_text,
    t.text as translation,
    t.language,
    ts_rank(t.search_vector, query) as rank
FROM quran_verses v
JOIN quran_translations t ON v.id = t.verse_id
CROSS JOIN to_tsquery('english', 'paradise | heaven') query
WHERE t.search_vector @@ query
ORDER BY rank DESC
LIMIT 20;

-- Search by Surah and Verse
SELECT 
    v.arabic_text,
    t.text as translation
FROM quran_verses v
LEFT JOIN quran_translations t ON v.id = t.verse_id AND t.language = 'en'
WHERE v.surah_number = 2 AND v.verse_number = 255;

-- Get verse with context (surrounding verses)
WITH target_verse AS (
    SELECT id, surah_number, verse_number
    FROM quran_verses
    WHERE surah_number = 2 AND verse_number = 255
)
SELECT 
    v.surah_number,
    v.verse_number,
    v.arabic_text,
    t.text as translation,
    CASE 
        WHEN v.id = tv.id THEN TRUE 
        ELSE FALSE 
    END as is_target
FROM quran_verses v
CROSS JOIN target_verse tv
LEFT JOIN quran_translations t ON v.id = t.verse_id AND t.language = 'en'
WHERE v.surah_number = tv.surah_number
AND v.verse_number BETWEEN tv.verse_number - 2 AND tv.verse_number + 2
ORDER BY v.verse_number;

-- Bookmarking verses
CREATE TABLE user_bookmarks (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL,
    verse_id INTEGER REFERENCES quran_verses(id),
    note TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, verse_id)
);

-- Reading progress tracking
CREATE TABLE user_reading_progress (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL UNIQUE,
    last_read_verse_id INTEGER REFERENCES quran_verses(id),
    last_read_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 9. LeetCode SQL Problems

Here are 20 LeetCode-style SQL problems with solutions:

### Problem 33: Second Highest Salary (LeetCode #176)

**Problem:** Write a SQL query to get the second highest salary from the Employee table. If there is no second highest salary, return null.

```sql
CREATE TABLE Employee (
    id INT PRIMARY KEY,
    salary INT
);
```

**Solution:**
```sql
-- Solution 1: Using OFFSET
SELECT 
    (SELECT DISTINCT salary 
     FROM Employee 
     ORDER BY salary DESC 
     LIMIT 1 OFFSET 1) AS SecondHighestSalary;

-- Solution 2: Using subquery
SELECT MAX(salary) AS SecondHighestSalary
FROM Employee
WHERE salary < (SELECT MAX(salary) FROM Employee);

-- Solution 3: Using DENSE_RANK
SELECT 
    CASE 
        WHEN COUNT(*) = 0 THEN NULL
        ELSE MAX(salary)
    END AS SecondHighestSalary
FROM (
    SELECT 
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) as rank
    FROM Employee
) ranked
WHERE rank = 2;
```

### Problem 34: Nth Highest Salary (LeetCode #177)

**Problem:** Write a SQL query to get the nth highest salary from the Employee table.

**Solution:**
```sql
CREATE OR REPLACE FUNCTION NthHighestSalary(N INTEGER) 
RETURNS TABLE (salary INTEGER) AS $$
BEGIN
    RETURN QUERY (
        SELECT DISTINCT e.salary
        FROM Employee e
        ORDER BY e.salary DESC
        LIMIT 1 OFFSET N-1
    );
END;
$$ LANGUAGE plpgsql;

-- Alternative using DENSE_RANK
CREATE OR REPLACE FUNCTION NthHighestSalary(N INTEGER) 
RETURNS TABLE (salary INTEGER) AS $$
BEGIN
    RETURN QUERY (
        SELECT DISTINCT e.salary
        FROM (
            SELECT 
                salary,
                DENSE_RANK() OVER (ORDER BY salary DESC) as rank
            FROM Employee
        ) e
        WHERE e.rank = N
    );
END;
$$ LANGUAGE plpgsql;
```

### Problem 35: Consecutive Numbers (LeetCode #180)

**Problem:** Find all numbers that appear at least three times consecutively.

```sql
CREATE TABLE Logs (
    id INT PRIMARY KEY,
    num INT
);
```

**Solution:**
```sql
-- Solution 1: Using LEAD/LAG
SELECT DISTINCT num AS ConsecutiveNums
FROM (
    SELECT 
        num,
        LEAD(num, 1) OVER (ORDER BY id) AS next1,
        LEAD(num, 2) OVER (ORDER BY id) AS next2
    FROM Logs
) t
WHERE num = next1 AND num = next2;

-- Solution 2: Self join
SELECT DISTINCT l1.num AS ConsecutiveNums
FROM Logs l1
JOIN Logs l2 ON l1.id = l2.id - 1 AND l1.num = l2.num
JOIN Logs l3 ON l2.id = l3.id - 1 AND l2.num = l3.num;
```

### Problem 36: Employees Earning More Than Managers (LeetCode #181)

**Problem:** Find employees who earn more than their managers.

```sql
CREATE TABLE Employee (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    salary INT,
    managerId INT
);
```

**Solution:**
```sql
SELECT e.name AS Employee
FROM Employee e
JOIN Employee m ON e.managerId = m.id
WHERE e.salary > m.salary;
```

### Problem 37: Delete Duplicate Emails (LeetCode #196)

**Problem:** Delete duplicate emails, keeping only the one with the smallest id.

```sql
CREATE TABLE Person (
    id INT PRIMARY KEY,
    email VARCHAR(100)
);
```

**Solution:**
```sql
DELETE FROM Person
WHERE id NOT IN (
    SELECT MIN(id)
    FROM Person
    GROUP BY email
);

-- Alternative using window function
DELETE FROM Person
WHERE id IN (
    SELECT id
    FROM (
        SELECT 
            id,
            ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) as rn
        FROM Person
    ) t
    WHERE rn > 1
);
```

### Problem 38: Rising Temperature (LeetCode #197)

**Problem:** Find dates with higher temperature compared to the previous day.

```sql
CREATE TABLE Weather (
    id INT PRIMARY KEY,
    recordDate DATE,
    temperature INT
);
```

**Solution:**
```sql
SELECT w1.id
FROM Weather w1
JOIN Weather w2 ON w1.recordDate = w2.recordDate + INTERVAL '1 day'
WHERE w1.temperature > w2.temperature;

-- Alternative using LAG
SELECT id
FROM (
    SELECT 
        id,
        temperature,
        LAG(temperature) OVER (ORDER BY recordDate) as prev_temp,
        LAG(recordDate) OVER (ORDER BY recordDate) as prev_date,
        recordDate
    FROM Weather
) t
WHERE temperature > prev_temp 
AND recordDate = prev_date + INTERVAL '1 day';
```

### Problem 39: Department Highest Salary (LeetCode #184)

**Problem:** Find employees who have the highest salary in their department.

```sql
CREATE TABLE Employee (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    salary INT,
    departmentId INT
);

CREATE TABLE Department (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);
```

**Solution:**
```sql
SELECT 
    d.name AS Department,
    e.name AS Employee,
    e.salary AS Salary
FROM Employee e
JOIN Department d ON e.departmentId = d.id
WHERE (e.departmentId, e.salary) IN (
    SELECT departmentId, MAX(salary)
    FROM Employee
    GROUP BY departmentId
);

-- Alternative using window function
SELECT 
    Department,
    Employee,
    Salary
FROM (
    SELECT 
        d.name AS Department,
        e.name AS Employee,
        e.salary AS Salary,
        RANK() OVER (PARTITION BY e.departmentId ORDER BY e.salary DESC) as rank
    FROM Employee e
    JOIN Department d ON e.departmentId = d.id
) t
WHERE rank = 1;
```

### Problem 40: Customers Who Never Order (LeetCode #183)

**Problem:** Find customers who never ordered anything.

```sql
CREATE TABLE Customers (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE Orders (
    id INT PRIMARY KEY,
    customerId INT
);
```

**Solution:**
```sql
-- Solution 1: Using NOT IN
SELECT name AS Customers
FROM Customers
WHERE id NOT IN (SELECT customerId FROM Orders);

-- Solution 2: Using LEFT JOIN
SELECT c.name AS Customers
FROM Customers c
LEFT JOIN Orders o ON c.id = o.customerId
WHERE o.id IS NULL;

-- Solution 3: Using NOT EXISTS
SELECT name AS Customers
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o WHERE o.customerId = c.id
);
```

---

## 10. Database Design Patterns

### 10.1 One-to-Many Relationship

```sql
-- Example: Users and Posts
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_posts_user ON posts(user_id);
```

### 10.2 Many-to-Many Relationship

```sql
-- Example: Students and Courses
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE
);

CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20) UNIQUE NOT NULL
);

-- Junction table
CREATE TABLE enrollments (
    id SERIAL PRIMARY KEY,
    student_id INTEGER NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    course_id INTEGER NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    enrolled_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    grade VARCHAR(2),
    UNIQUE(student_id, course_id)
);

CREATE INDEX idx_enrollments_student ON enrollments(student_id);
CREATE INDEX idx_enrollments_course ON enrollments(course_id);
```

### 10.3 Self-Referencing Relationship

```sql
-- Example: Employee hierarchy
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    manager_id INTEGER REFERENCES employees(id),
    department VARCHAR(50)
);

CREATE INDEX idx_employees_manager ON employees(manager_id);

-- Query hierarchy
WITH RECURSIVE employee_hierarchy AS (
    -- Base: top-level managers
    SELECT id, name, manager_id, 0 as level, name as path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive: subordinates
    SELECT 
        e.id, 
        e.name, 
        e.manager_id, 
        eh.level + 1,
        eh.path || ' > ' || e.name
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT * FROM employee_hierarchy ORDER BY path;
```

### 10.4 Polymorphic Association

```sql
-- Example: Comments on different types (posts, photos, videos)
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),
    commentable_type VARCHAR(50) NOT NULL, -- 'Post', 'Photo', 'Video'
    commentable_id INTEGER NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_comments_commentable ON comments(commentable_type, commentable_id);

-- Query comments for a specific post
SELECT * FROM comments
WHERE commentable_type = 'Post' AND commentable_id = 123;
```

### 10.5 Temporal Data (Slowly Changing Dimensions)

```sql
-- Track historical changes
CREATE TABLE employee_salary_history (
    id SERIAL PRIMARY KEY,
    employee_id INTEGER NOT NULL,
    salary DECIMAL(10, 2) NOT NULL,
    valid_from DATE NOT NULL,
    valid_to DATE,
    is_current BOOLEAN DEFAULT TRUE,
    CONSTRAINT valid_dates CHECK (valid_to IS NULL OR valid_from < valid_to)
);

CREATE INDEX idx_salary_history_employee ON employee_salary_history(employee_id);
CREATE INDEX idx_salary_history_current ON employee_salary_history(employee_id) 
WHERE is_current = TRUE;

-- Get current salary
SELECT salary 
FROM employee_salary_history
WHERE employee_id = 123 AND is_current = TRUE;

-- Get salary at a specific date
SELECT salary
FROM employee_salary_history
WHERE employee_id = 123
AND valid_from <= '2024-06-01'
AND (valid_to IS NULL OR valid_to > '2024-06-01');

-- Update salary (close old record, open new one)
CREATE OR REPLACE FUNCTION update_employee_salary(
    p_employee_id INTEGER,
    p_new_salary DECIMAL,
    p_effective_date DATE DEFAULT CURRENT_DATE
)
RETURNS VOID AS $$
BEGIN
    -- Close current record
    UPDATE employee_salary_history
    SET is_current = FALSE,
        valid_to = p_effective_date
    WHERE employee_id = p_employee_id AND is_current = TRUE;
    
    -- Insert new record
    INSERT INTO employee_salary_history (employee_id, salary, valid_from)
    VALUES (p_employee_id, p_new_salary, p_effective_date);
END;
$$ LANGUAGE plpgsql;
```

---

## 11. Production Best Practices

### 11.1 Connection Pooling

**Django Example:**
```python
# Django settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'my_database',
        'USER': 'db_user',
        'PASSWORD': 'secure_password',
        'HOST': 'localhost',
        'PORT': '5432',
        'OPTIONS': {
            'connect_timeout': 10,
        },
        'CONN_MAX_AGE': 600,  # Connection pooling
    }
}
```

**PgBouncer Configuration:**
```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
```

### 11.2 Backup and Recovery

```bash
# Backup
pg_dump -U postgres -Fc my_database > backup.dump

# Restore
pg_restore -U postgres -d my_database backup.dump

# Backup specific table
pg_dump -U postgres -t users my_database > users_backup.sql

# Continuous archiving (Point-in-Time Recovery)
# postgresql.conf
# archive_mode = on
# archive_command = 'cp %p /path/to/archive/%f'

# Create base backup
pg_basebackup -D /backup/base -Ft -z -P

# Restore to specific point in time
# recovery.conf
# restore_command = 'cp /path/to/archive/%f %p'
# recovery_target_time = '2024-01-15 12:00:00'
```

### 11.3 Security Best Practices

```sql
-- Use least privilege principle
CREATE USER readonly WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE my_app TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- Row Level Security
ALTER TABLE sensitive_data ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_data_policy ON sensitive_data
    FOR ALL
    TO app_user
    USING (user_id = current_setting('app.current_user_id')::INTEGER);

-- Encrypt sensitive columns
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE users_secure (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    password_hash TEXT,  -- Store hashed passwords
    ssn BYTEA  -- Encrypted SSN
);

-- Insert with encryption
INSERT INTO users_secure (username, password_hash, ssn)
VALUES (
    'john',
    crypt('plain_password', gen_salt('bf', 10)),
    pgp_sym_encrypt('123-45-6789', 'encryption_key')
);

-- Query with decryption
SELECT 
    username,
    pgp_sym_decrypt(ssn, 'encryption_key') as ssn_decrypted
FROM users_secure
WHERE username = 'john'
AND crypt('plain_password', password_hash) = password_hash;
```

### 11.4 Monitoring Queries

```sql
-- Enable pg_stat_statements
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find slow queries
SELECT 
    query,
    calls,
    total_exec_time / 1000 as total_seconds,
    mean_exec_time / 1000 as avg_seconds,
    max_exec_time / 1000 as max_seconds
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Active queries
SELECT 
    pid,
    now() - query_start as duration,
    usename,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;

-- Lock monitoring
SELECT 
    l.pid,
    l.mode,
    l.granted,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT l.granted;

-- Kill long-running query
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid = <pid_number>;
```

### 11.5 Database Migrations

**Django Migrations:**
```bash
python manage.py makemigrations
python manage.py migrate
```

**Manual Migration Template:**
```sql
-- Migration: 001_add_user_preferences.sql
BEGIN;

-- Add new column
ALTER TABLE users ADD COLUMN preferences JSONB DEFAULT '{}';

-- Create index
CREATE INDEX idx_users_preferences ON users USING GIN(preferences);

-- Backfill data
UPDATE users SET preferences = '{"theme": "light"}' WHERE preferences IS NULL;

COMMIT;

-- Rollback: 001_add_user_preferences_rollback.sql
BEGIN;

DROP INDEX IF EXISTS idx_users_preferences;
ALTER TABLE users DROP COLUMN IF EXISTS preferences;

COMMIT;
```

---

## Conclusion & Next Steps

You've now learned SQL and PostgreSQL from basics to advanced! Here's your learning path:

### Beginner (Weeks 1-2)
✅ Basic CRUD operations
✅ WHERE, ORDER BY, LIMIT
✅ Basic joins
✅ Aggregate functions

### Intermediate (Weeks 3-4)
✅ Complex joins
✅ Subqueries
✅ Window functions
✅ CTEs
✅ Indexes basics

### Advanced (Weeks 5-8)
✅ Query optimization
✅ PostgreSQL-specific features (JSONB, arrays, full-text search)
✅ Triggers and functions
✅ Advanced indexing strategies
✅ Performance tuning

### Expert (Weeks 9-12)
✅ Complex database design
✅ Partitioning and sharding
✅ Replication
✅ Real-world production patterns
✅ Security and monitoring

### Practice Resources
- **LeetCode Database:** 50+ SQL problems
- **HackerRank SQL:** 70+ challenges
- **SQLBolt:** Interactive tutorials
- **PostgreSQL Documentation:** Official docs
- **Use The Index, Luke:** Indexing guide

### Your Next Projects
1. Build a multi-tenant SaaS backend
2. Implement a real-time analytics dashboard
3. Create a complex e-commerce database
4. Develop an Islamic app backend (Quran, prayers, etc.)

Keep practicing with real projects at GTAF, and you'll master PostgreSQL! 🚀

---

**Created for Maharab - Backend Engineer at Greentech Apps Foundation**
*From Basic to Advanced PostgreSQL Mastery*
