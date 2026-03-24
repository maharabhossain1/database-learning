# Complete Database Design & Scalability Guide

**From Schema Design to Production-Scale Architecture**

> A comprehensive guide to designing scalable, performant, and maintainable database schemas with PostgreSQL, covering normalization, indexing strategies, query optimization, and the 7 key scaling strategies.

---

## Table of Contents

1. [Database Design Fundamentals](#1-database-design-fundamentals)
2. [Normalization Theory & Practice](#2-normalization-theory--practice)
3. [Schema Design Patterns](#3-schema-design-patterns)
4. [Designing for Query Performance](#4-designing-for-query-performance)
5. [Indexing Strategy & Design](#5-indexing-strategy--design)
6. [Denormalization (When & How)](#6-denormalization-when--how)
7. [The 7 Strategies to Scale Database](#7-the-7-strategies-to-scale-database)
8. [Real-World Design Examples](#8-real-world-design-examples)
9. [Design Anti-Patterns to Avoid](#9-design-anti-patterns-to-avoid)
10. [Production-Ready Design Checklist](#10-production-ready-design-checklist)

---

## 1. Database Design Fundamentals

### 1.1 Design Philosophy

**The Golden Rules of Database Design:**

1. **Understand the Domain First** - Before writing SQL, understand the business logic
2. **Design for Queries** - Your schema should optimize for the queries you'll run most
3. **Plan for Scale** - Consider how data will grow over time
4. **Keep It Simple** - Complexity is the enemy of maintainability
5. **Document Everything** - Future you (and your team) will thank you

### 1.2 The Design Process

```
Step 1: Requirements Gathering
  └─> What data needs to be stored?
  └─> What questions will be asked of the data?
  └─> What are the performance requirements?
  └─> What is the expected data volume?

Step 2: Conceptual Design (ER Diagram)
  └─> Identify entities (users, products, orders)
  └─> Identify relationships (one-to-many, many-to-many)
  └─> Identify attributes (what describes each entity)

Step 3: Logical Design (Normalization)
  └─> Apply normalization rules
  └─> Eliminate redundancy
  └─> Ensure data integrity

Step 4: Physical Design (Implementation)
  └─> Choose data types
  └─> Design indexes
  └─> Add constraints
  └─> Consider partitioning

Step 5: Optimize & Iterate
  └─> Analyze query patterns
  └─> Add selective denormalization
  └─> Refine indexes
  └─> Monitor and adjust
```

### 1.3 Entity-Relationship (ER) Modeling

**Example: E-commerce System**

```
Entities:
- Customer (id, name, email, phone, address)
- Product (id, name, description, price, category_id)
- Order (id, customer_id, order_date, total_amount, status)
- OrderItem (id, order_id, product_id, quantity, price)
- Category (id, name, description)

Relationships:
- Customer → Order (One-to-Many)
- Order → OrderItem (One-to-Many)
- Product → OrderItem (One-to-Many)
- Category → Product (One-to-Many)
```

**Think About:**

- What are the primary entities in your system?
- How do they relate to each other?
- What attributes does each entity have?
- Are there any junction tables needed for many-to-many relationships?

---

## 2. Normalization Theory & Practice

### 2.1 What is Normalization?

**Normalization** is the process of organizing data to:

1. **Eliminate redundancy** - Store each piece of data once
2. **Ensure data integrity** - Updates happen in one place
3. **Optimize for consistency** - Related data stays consistent

### 2.2 Normal Forms Explained

#### **First Normal Form (1NF)** - Atomic Values

**Rule:** Each column must contain atomic (indivisible) values, and each column must contain values of a single type.

❌ **Bad - Violates 1NF:**

```sql
CREATE TABLE orders_bad (
    id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100),
    products TEXT,  -- "iPhone, iPad, MacBook"
    prices TEXT     -- "999, 599, 1299"
);
```

✅ **Good - Follows 1NF:**

```sql
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100),
    order_date DATE
);

CREATE TABLE order_items (
    id INTEGER PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product_name VARCHAR(100),
    price DECIMAL(10, 2)
);
```

**Why?** Now you can query individual products, sum prices correctly, and update product names without parsing strings.

---

#### **Second Normal Form (2NF)** - No Partial Dependencies

**Rule:** Must be in 1NF AND all non-key columns must depend on the entire primary key (not just part of it).

❌ **Bad - Violates 2NF:**

```sql
CREATE TABLE order_items_bad (
    order_id INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    product_name VARCHAR(100),      -- Depends only on product_id
    product_category VARCHAR(50),   -- Depends only on product_id
    PRIMARY KEY (order_id, product_id)
);
```

✅ **Good - Follows 2NF:**

```sql
CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    category VARCHAR(50)
);

CREATE TABLE order_items (
    order_id INTEGER,
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER,
    PRIMARY KEY (order_id, product_id)
);
```

**Why?** Product details are stored once. If product name changes, update one row in `products`, not hundreds in `order_items`.

---

#### **Third Normal Form (3NF)** - No Transitive Dependencies

**Rule:** Must be in 2NF AND no non-key column depends on another non-key column.

❌ **Bad - Violates 3NF:**

```sql
CREATE TABLE employees_bad (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    department_id INTEGER,
    department_name VARCHAR(100),    -- Depends on department_id (non-key)
    department_location VARCHAR(100) -- Depends on department_id (non-key)
);
```

✅ **Good - Follows 3NF:**

```sql
CREATE TABLE departments (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    location VARCHAR(100)
);

CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    department_id INTEGER REFERENCES departments(id)
);
```

**Why?** Department details stored once. Moving a department to a new location? Update one row.

---

#### **Boyce-Codd Normal Form (BCNF)** - Stronger 3NF

**Rule:** For any dependency A → B, A must be a superkey.

❌ **Bad - Violates BCNF:**

```sql
CREATE TABLE course_instructor_bad (
    course_id INTEGER,
    instructor_id INTEGER,
    instructor_name VARCHAR(100),
    -- Problem: instructor_id → instructor_name
    -- but instructor_id is not a superkey
    PRIMARY KEY (course_id, instructor_id)
);
```

✅ **Good - Follows BCNF:**

```sql
CREATE TABLE instructors (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE course_assignments (
    course_id INTEGER,
    instructor_id INTEGER REFERENCES instructors(id),
    PRIMARY KEY (course_id, instructor_id)
);
```

---

#### **Fourth Normal Form (4NF)** - No Multi-Valued Dependencies

**Rule:** Must be in BCNF AND have no multi-valued dependencies.

❌ **Bad - Violates 4NF:**

```sql
CREATE TABLE student_courses_bad (
    student_id INTEGER,
    course VARCHAR(100),
    hobby VARCHAR(100),
    -- Problem: courses and hobbies are independent
    PRIMARY KEY (student_id, course, hobby)
);

-- Leads to redundancy:
-- (1, 'Math', 'Swimming')
-- (1, 'Math', 'Reading')
-- (1, 'Physics', 'Swimming')
-- (1, 'Physics', 'Reading')
```

✅ **Good - Follows 4NF:**

```sql
CREATE TABLE student_courses (
    student_id INTEGER,
    course VARCHAR(100),
    PRIMARY KEY (student_id, course)
);

CREATE TABLE student_hobbies (
    student_id INTEGER,
    hobby VARCHAR(100),
    PRIMARY KEY (student_id, hobby)
);
```

---

#### **Fifth Normal Form (5NF)** - No Join Dependencies

**Rule:** Must be in 4NF AND every join dependency is implied by candidate keys.

❌ **Bad - Violates 5NF:**

```sql
CREATE TABLE supplier_product_customer_bad (
    supplier_id INTEGER,
    product_id INTEGER,
    customer_id INTEGER,
    -- Problem: Can lead to redundant combinations
    PRIMARY KEY (supplier_id, product_id, customer_id)
);
```

✅ **Good - Follows 5NF:**

```sql
CREATE TABLE supplier_products (
    supplier_id INTEGER,
    product_id INTEGER,
    PRIMARY KEY (supplier_id, product_id)
);

CREATE TABLE product_customers (
    product_id INTEGER,
    customer_id INTEGER,
    PRIMARY KEY (product_id, customer_id)
);

CREATE TABLE supplier_customers (
    supplier_id INTEGER,
    customer_id INTEGER,
    PRIMARY KEY (supplier_id, customer_id)
);
```

---

### 2.3 Practical Normalization Example

**Scenario:** Design a database for an Islamic learning platform (relevant to GTAF).

**Initial Denormalized Design:**

```sql
CREATE TABLE courses_bad (
    id INTEGER PRIMARY KEY,
    title VARCHAR(200),
    description TEXT,
    instructor_name VARCHAR(100),
    instructor_email VARCHAR(100),
    instructor_bio TEXT,
    category_name VARCHAR(100),
    category_description TEXT,
    students TEXT, -- Comma-separated student IDs
    prices TEXT    -- Different prices for different countries
);
```

**Problems:**

1. Violates 1NF (students and prices are lists)
2. Violates 2NF/3NF (instructor details repeated for every course)
3. Hard to query (can't easily find all courses by an instructor)
4. Update anomalies (changing instructor email requires updating many rows)

**Normalized Design (3NF):**

```sql
-- Instructors (separate entity)
CREATE TABLE instructors (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    bio TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Categories (separate entity)
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    parent_category_id INTEGER REFERENCES categories(id)
);

-- Courses (main entity with foreign keys)
CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    instructor_id INTEGER NOT NULL REFERENCES instructors(id),
    category_id INTEGER NOT NULL REFERENCES categories(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Many-to-many: Students enrolled in courses
CREATE TABLE enrollments (
    id SERIAL PRIMARY KEY,
    course_id INTEGER NOT NULL REFERENCES courses(id),
    student_id INTEGER NOT NULL REFERENCES users(id),
    enrolled_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    progress INTEGER DEFAULT 0,
    UNIQUE(course_id, student_id)
);

-- Prices by country (one-to-many)
CREATE TABLE course_pricing (
    id SERIAL PRIMARY KEY,
    course_id INTEGER NOT NULL REFERENCES courses(id),
    country_code VARCHAR(2) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    UNIQUE(course_id, country_code)
);

-- Indexes for common queries
CREATE INDEX idx_courses_instructor ON courses(instructor_id);
CREATE INDEX idx_courses_category ON courses(category_id);
CREATE INDEX idx_enrollments_student ON enrollments(student_id);
CREATE INDEX idx_enrollments_course ON enrollments(course_id);
CREATE INDEX idx_pricing_country ON course_pricing(country_code);
```

**Benefits:**

- ✅ Each entity is independent
- ✅ Easy to query (find all courses by instructor, all students in a course)
- ✅ Update in one place (change instructor email once)
- ✅ No redundancy
- ✅ Scalable (can add more attributes to each entity)

---

### 2.4 When to Stop Normalizing

**General Rule:** Normalize to 3NF by default, BCNF if possible.

**Stop normalizing when:**

1. You're joining 5+ tables for common queries
2. Performance testing shows it's too slow
3. Read-heavy workloads suffer (e.g., analytics dashboards)
4. You're creating unnecessary complexity

**Example: Over-Normalized**

```sql
-- Too much normalization for simple data
CREATE TABLE address_countries (id SERIAL PRIMARY KEY, name VARCHAR(100));
CREATE TABLE address_states (id SERIAL PRIMARY KEY, name VARCHAR(100), country_id INTEGER);
CREATE TABLE address_cities (id SERIAL PRIMARY KEY, name VARCHAR(100), state_id INTEGER);
CREATE TABLE address_streets (id SERIAL PRIMARY KEY, name VARCHAR(100), city_id INTEGER);

-- Unless you need location analytics, this is overkill
-- Better: Store address as TEXT or JSONB
```

---

## 3. Schema Design Patterns

### 3.1 Naming Conventions

**Tables:**

- Use **plural nouns**: `users`, `products`, `orders`
- Lowercase with underscores: `order_items`, `user_sessions`
- Be descriptive: `payment_transactions` not `trans`

**Columns:**

- Use **singular nouns**: `user_id`, `created_at`
- Boolean: prefix with `is_` or `has_`: `is_active`, `has_shipped`
- Timestamps: use `_at` suffix: `created_at`, `updated_at`, `deleted_at`
- Foreign keys: `<table>_id`: `user_id`, `product_id`

**Indexes:**

- Prefix with `idx_`: `idx_users_email`, `idx_orders_customer_date`
- Include table and columns: `idx_order_items_order_product`

**Constraints:**

- Primary key: `pk_<table>`: `pk_users`
- Foreign key: `fk_<table>_<column>`: `fk_orders_user_id`
- Unique: `uq_<table>_<column>`: `uq_users_email`
- Check: `ck_<table>_<condition>`: `ck_orders_total_positive`

### 3.2 Standard Column Patterns

**Every Table Should Have:**

```sql
CREATE TABLE example (
    -- Primary key (always first)
    id BIGSERIAL PRIMARY KEY,

    -- Business logic columns
    name VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'active',

    -- Audit columns (always last)
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMPTZ  -- For soft deletes
);

-- Auto-update trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_example_updated_at
    BEFORE UPDATE ON example
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### 3.3 Data Type Selection Guide

**Choosing the Right Data Type:**

| Use Case      | Bad Choice     | Good Choice           | Why                      |
| ------------- | -------------- | --------------------- | ------------------------ |
| Primary Key   | `INTEGER`      | `BIGSERIAL`           | Won't run out (8B vs 2B) |
| Money         | `FLOAT`        | `DECIMAL(10,2)`       | Exact precision needed   |
| Status/Enum   | `VARCHAR(50)`  | `VARCHAR(20)` + CHECK | Memory efficient         |
| Boolean       | `CHAR(1)`      | `BOOLEAN`             | Native type is faster    |
| Timestamps    | `INTEGER`      | `TIMESTAMPTZ`         | Timezone aware           |
| JSON Data     | `TEXT`         | `JSONB`               | Indexable and faster     |
| Small Numbers | `INTEGER`      | `SMALLINT`            | 2 bytes vs 4 bytes       |
| Text          | `VARCHAR(MAX)` | `TEXT`                | No arbitrary limit       |

**Example: Status Column**

```sql
-- Bad: Any string allowed
status VARCHAR(50)

-- Good: Limited to valid values
status VARCHAR(20) CHECK (status IN ('pending', 'processing', 'completed', 'cancelled'))

-- Better: Use ENUM type (PostgreSQL)
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'completed', 'cancelled');
status order_status DEFAULT 'pending'
```

### 3.4 Relationship Patterns

#### **One-to-One**

Used when you want to split a table for organization or performance.

```sql
-- Example: User basic info vs sensitive info
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    full_name VARCHAR(100),
    bio TEXT,
    avatar_url TEXT,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Query: Always join these tables
SELECT u.username, p.full_name, p.bio
FROM users u
LEFT JOIN user_profiles p ON u.id = p.user_id
WHERE u.id = 123;
```

#### **One-to-Many** (Most Common)

The "one" side has a primary key, the "many" side has a foreign key.

```sql
-- One customer, many orders
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    order_date DATE DEFAULT CURRENT_DATE,
    total DECIMAL(10,2)
);

CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Query: Get customer with all orders
SELECT
    c.name,
    c.email,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.id = 123
GROUP BY c.id, c.name, c.email;
```

#### **Many-to-Many**

Requires a junction table.

```sql
-- Students can enroll in many courses
-- Courses can have many students

CREATE TABLE students (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE
);

CREATE TABLE courses (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(200),
    code VARCHAR(20) UNIQUE
);

-- Junction table
CREATE TABLE enrollments (
    id BIGSERIAL PRIMARY KEY,
    student_id BIGINT NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    course_id BIGINT NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    enrolled_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    grade VARCHAR(2),
    UNIQUE(student_id, course_id)
);

CREATE INDEX idx_enrollments_student ON enrollments(student_id);
CREATE INDEX idx_enrollments_course ON enrollments(course_id);

-- Query: Get all courses for a student
SELECT
    s.name as student_name,
    c.title as course_title,
    c.code,
    e.grade
FROM students s
JOIN enrollments e ON s.id = e.student_id
JOIN courses c ON e.course_id = c.id
WHERE s.id = 123;
```

### 3.5 Inheritance Patterns

#### **Single Table Inheritance (STI)**

All types in one table with a type discriminator.

```sql
CREATE TABLE vehicles (
    id BIGSERIAL PRIMARY KEY,
    type VARCHAR(20) NOT NULL CHECK (type IN ('car', 'truck', 'motorcycle')),
    make VARCHAR(50),
    model VARCHAR(50),
    year INTEGER,
    -- Car-specific
    num_doors INTEGER,
    -- Truck-specific
    bed_length DECIMAL(5,2),
    payload_capacity INTEGER,
    -- Motorcycle-specific
    engine_displacement INTEGER,
    has_sidecar BOOLEAN
);

-- Pros: Simple queries, one table
-- Cons: Many NULL values, hard to enforce constraints
```

#### **Class Table Inheritance (CTI)**

One table per type, shared attributes in parent table.

```sql
-- Parent table
CREATE TABLE vehicles (
    id BIGSERIAL PRIMARY KEY,
    make VARCHAR(50),
    model VARCHAR(50),
    year INTEGER
);

-- Child tables
CREATE TABLE cars (
    vehicle_id BIGINT PRIMARY KEY REFERENCES vehicles(id),
    num_doors INTEGER NOT NULL,
    trunk_capacity DECIMAL(5,2)
);

CREATE TABLE trucks (
    vehicle_id BIGINT PRIMARY KEY REFERENCES vehicles(id),
    bed_length DECIMAL(5,2) NOT NULL,
    payload_capacity INTEGER NOT NULL
);

-- Pros: No NULL columns, proper constraints
-- Cons: Requires joins for queries
```

#### **Concrete Table Inheritance**

Completely separate tables.

```sql
CREATE TABLE cars (
    id BIGSERIAL PRIMARY KEY,
    make VARCHAR(50),
    model VARCHAR(50),
    year INTEGER,
    num_doors INTEGER
);

CREATE TABLE trucks (
    id BIGSERIAL PRIMARY KEY,
    make VARCHAR(50),
    model VARCHAR(50),
    year INTEGER,
    bed_length DECIMAL(5,2)
);

-- Pros: No joins needed, simple
-- Cons: Code duplication, hard to query all vehicles
```

---

## 4. Designing for Query Performance

### 4.1 Query-Driven Design Philosophy

**Design Rule:** Your schema should optimize for your most frequent queries.

**Process:**

1. List your top 10 most frequent queries
2. List your top 5 most critical (slow/important) queries
3. Design schema to make these queries efficient
4. Add indexes specifically for these queries

**Example: E-commerce Platform**

**Top Queries:**

1. Get product details by ID
2. Search products by category
3. Get user's order history
4. Get order items with product details
5. Calculate user's total spending

**Schema Design:**

```sql
-- Optimized for Query #1: Get product details
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    category_id BIGINT REFERENCES categories(id),
    stock_quantity INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Index for Query #2: Search by category
CREATE INDEX idx_products_category_active ON products(category_id)
WHERE is_active = true;

-- Optimized for Query #3: User's order history
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Index for Query #3: Fast lookup by user + date
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Optimized for Query #4: Order items with product details
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    -- Denormalized for performance
    product_name VARCHAR(200),  -- Snapshot of name at purchase time
    UNIQUE(order_id, product_id)
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);

-- For Query #5: Use materialized view or aggregation table
CREATE MATERIALIZED VIEW user_spending AS
SELECT
    user_id,
    COUNT(DISTINCT id) as total_orders,
    SUM(total_amount) as total_spent,
    AVG(total_amount) as avg_order_value
FROM orders
WHERE status = 'completed'
GROUP BY user_id;

CREATE UNIQUE INDEX idx_user_spending_user ON user_spending(user_id);
```

### 4.2 Avoiding N+1 Query Problems

**Problem:** Loading a list of items, then making one query per item for related data.

❌ **Bad Pattern:**

```python
# Get all orders (1 query)
orders = Order.objects.all()

# For each order, get items (N queries)
for order in orders:
    items = OrderItem.objects.filter(order_id=order.id)  # N queries!
```

✅ **Solution 1: Design with proper foreign keys and use joins**

```sql
-- One query gets everything
SELECT
    o.id,
    o.total_amount,
    o.created_at,
    json_agg(
        json_build_object(
            'product_id', oi.product_id,
            'product_name', p.name,
            'quantity', oi.quantity,
            'price', oi.price
        )
    ) as items
FROM orders o
LEFT JOIN order_items oi ON o.id = oi.order_id
LEFT JOIN products p ON oi.product_id = p.id
GROUP BY o.id, o.total_amount, o.created_at;
```

✅ **Solution 2: Selective denormalization**

```sql
-- Add commonly needed data to avoid joins
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    user_name VARCHAR(100),  -- Denormalized
    user_email VARCHAR(100), -- Denormalized
    item_count INTEGER,      -- Denormalized
    total_amount DECIMAL(10,2),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Update with trigger
CREATE OR REPLACE FUNCTION update_order_denormalized_data()
RETURNS TRIGGER AS $$
BEGIN
    -- Update item count
    UPDATE orders
    SET item_count = (
        SELECT COUNT(*) FROM order_items WHERE order_id = NEW.order_id
    )
    WHERE id = NEW.order_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_order_item_count
    AFTER INSERT OR UPDATE OR DELETE ON order_items
    FOR EACH ROW
    EXECUTE FUNCTION update_order_denormalized_data();
```

### 4.3 Pagination Design

**Wrong Way - OFFSET (slow for large datasets):**

```sql
-- Gets slower as page number increases
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 100000;  -- Very slow!
```

**Right Way - Cursor-based (keyset pagination):**

```sql
-- Fast regardless of page position
-- First page
SELECT * FROM products
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page (use last created_at and id from previous page)
SELECT * FROM products
WHERE (created_at, id) < ('2024-01-15 10:30:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Need index for this
CREATE INDEX idx_products_created_id ON products(created_at DESC, id DESC);
```

### 4.4 Aggregation-Heavy Queries

**Scenario:** Dashboard showing statistics across millions of records.

❌ **Bad: Query on-demand**

```sql
-- Runs every time dashboard loads (slow!)
SELECT
    DATE_TRUNC('day', created_at) as date,
    COUNT(*) as order_count,
    SUM(total_amount) as revenue,
    AVG(total_amount) as avg_order_value
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY DATE_TRUNC('day', created_at)
ORDER BY date;
```

✅ **Good: Pre-aggregate with materialized view**

```sql
CREATE MATERIALIZED VIEW daily_order_stats AS
SELECT
    DATE_TRUNC('day', created_at) as date,
    COUNT(*) as order_count,
    SUM(total_amount) as revenue,
    AVG(total_amount) as avg_order_value,
    COUNT(DISTINCT user_id) as unique_customers
FROM orders
GROUP BY DATE_TRUNC('day', created_at);

CREATE UNIQUE INDEX idx_daily_stats_date ON daily_order_stats(date);

-- Refresh periodically (e.g., every hour)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_order_stats;

-- Query is now instant
SELECT * FROM daily_order_stats
WHERE date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY date DESC;
```

✅ **Better: Incremental aggregation table**

```sql
CREATE TABLE daily_order_statistics (
    date DATE PRIMARY KEY,
    order_count INTEGER DEFAULT 0,
    revenue DECIMAL(12,2) DEFAULT 0,
    avg_order_value DECIMAL(10,2) DEFAULT 0,
    unique_customers INTEGER DEFAULT 0,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Update via trigger or scheduled job
CREATE OR REPLACE FUNCTION update_daily_stats()
RETURNS TRIGGER AS $$
DECLARE
    order_date DATE;
BEGIN
    order_date := DATE_TRUNC('day', NEW.created_at);

    INSERT INTO daily_order_statistics (date, order_count, revenue)
    VALUES (order_date, 1, NEW.total_amount)
    ON CONFLICT (date) DO UPDATE
    SET order_count = daily_order_statistics.order_count + 1,
        revenue = daily_order_statistics.revenue + NEW.total_amount,
        avg_order_value = (daily_order_statistics.revenue + NEW.total_amount) / (daily_order_statistics.order_count + 1),
        updated_at = CURRENT_TIMESTAMP;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_stats_on_order
    AFTER INSERT ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_daily_stats();
```

---

## 5. Indexing Strategy & Design

### 5.1 Index Fundamentals

**What is an Index?**
An index is a data structure that improves query speed by creating a sorted reference to table data.

**Think of it like:**

- Book index: Instead of reading every page to find "PostgreSQL", check the index
- Phone book: Sorted by name for fast lookup

**The Trade-off:**

- ✅ **Pros:** Faster SELECT queries
- ❌ **Cons:** Slower INSERT/UPDATE/DELETE, uses disk space

### 5.2 When to Create an Index

**Always index:**

1. Primary keys (automatic)
2. Foreign keys (very important!)
3. Columns used in WHERE clauses frequently
4. Columns used in JOIN conditions
5. Columns used in ORDER BY
6. Columns with high cardinality (many unique values)

**Don't index:**

1. Small tables (<1000 rows)
2. Columns with low cardinality (e.g., boolean with only 2 values)
3. Columns that are rarely queried
4. Tables with high write:read ratio

### 5.3 Index Types in PostgreSQL

#### **B-tree Index (Default, Most Common)**

Best for: Equality and range queries

```sql
-- Single column
CREATE INDEX idx_users_email ON users(email);

-- Multi-column (order matters!)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Can answer these queries efficiently:
SELECT * FROM orders WHERE user_id = 123;
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01';
SELECT * FROM orders WHERE user_id = 123 ORDER BY created_at;

-- But NOT this (user_id must be first):
SELECT * FROM orders WHERE created_at > '2024-01-01';  -- Won't use index!
```

**Multi-Column Index Rules:**

1. **Most selective column first** (usually)
2. Columns used in WHERE before ORDER BY
3. Index can be used for queries on prefix columns

```sql
-- Index on (user_id, status, created_at)
CREATE INDEX idx_orders_composite ON orders(user_id, status, created_at);

-- Can use index:
WHERE user_id = 123
WHERE user_id = 123 AND status = 'completed'
WHERE user_id = 123 AND status = 'completed' AND created_at > '2024-01-01'
WHERE user_id = 123 ORDER BY status, created_at

-- Cannot use index (missing first column):
WHERE status = 'completed'
WHERE created_at > '2024-01-01'
```

#### **Partial Index**

Index only rows that match a condition (smaller, faster).

```sql
-- Only index active products
CREATE INDEX idx_products_active ON products(category_id)
WHERE is_active = true;

-- Perfect for this query:
SELECT * FROM products
WHERE category_id = 5 AND is_active = true;

-- Much smaller than indexing all products
```

**Common Use Cases:**

```sql
-- Index only unprocessed records
CREATE INDEX idx_jobs_pending ON background_jobs(created_at)
WHERE status = 'pending';

-- Index only recent records
CREATE INDEX idx_orders_recent ON orders(user_id, created_at)
WHERE created_at > CURRENT_DATE - INTERVAL '90 days';

-- Index only non-deleted records
CREATE INDEX idx_users_active ON users(email)
WHERE deleted_at IS NULL;
```

#### **Expression Index**

Index on function result.

```sql
-- Case-insensitive email lookup
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Now this query uses the index:
SELECT * FROM users WHERE LOWER(email) = 'test@example.com';

-- Date-based queries
CREATE INDEX idx_orders_year_month ON orders(
    EXTRACT(YEAR FROM created_at),
    EXTRACT(MONTH FROM created_at)
);

-- Efficient query:
SELECT * FROM orders
WHERE EXTRACT(YEAR FROM created_at) = 2024
AND EXTRACT(MONTH FROM created_at) = 6;
```

#### **GIN Index (For Arrays, JSONB, Full-Text)**

```sql
-- Array column
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200),
    tags TEXT[]
);

CREATE INDEX idx_products_tags ON products USING GIN(tags);

-- Fast array queries:
SELECT * FROM products WHERE tags @> ARRAY['electronics'];
SELECT * FROM products WHERE 'bluetooth' = ANY(tags);

-- JSONB column
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    preferences JSONB
);

CREATE INDEX idx_users_preferences ON users USING GIN(preferences);

-- Fast JSONB queries:
SELECT * FROM users WHERE preferences @> '{"theme": "dark"}';
SELECT * FROM users WHERE preferences ? 'notifications';

-- Full-text search
CREATE INDEX idx_articles_search ON articles USING GIN(to_tsvector('english', content));

SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'postgresql');
```

#### **GiST Index (For Ranges, Geometry)**

```sql
-- Range types
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200),
    time_range TSTZRANGE
);

CREATE INDEX idx_events_time ON events USING GIST(time_range);

-- Fast range queries:
SELECT * FROM events
WHERE time_range @> '2024-06-15 14:00:00'::TIMESTAMPTZ;

-- Overlapping ranges:
SELECT * FROM events
WHERE time_range && '[2024-06-15, 2024-06-16)'::TSTZRANGE;
```

#### **BRIN Index (For Large Sequential Data)**

Extremely small, good for time-series data.

```sql
-- Time-series table
CREATE TABLE sensor_readings (
    id BIGSERIAL,
    sensor_id INTEGER,
    value DECIMAL(10,2),
    recorded_at TIMESTAMPTZ
);

-- BRIN index (tiny compared to B-tree)
CREATE INDEX idx_readings_time ON sensor_readings USING BRIN(recorded_at);

-- Good for range scans on sequential data:
SELECT * FROM sensor_readings
WHERE recorded_at BETWEEN '2024-01-01' AND '2024-01-31';
```

### 5.4 Index Maintenance

**Monitor Index Usage:**

```sql
-- Find unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find duplicate indexes
SELECT
    pg_size_pretty(SUM(pg_relation_size(idx))::BIGINT) as size,
    (array_agg(idx))[1] as idx1,
    (array_agg(idx))[2] as idx2,
    (array_agg(idx))[3] as idx3,
    (array_agg(idx))[4] as idx4
FROM (
    SELECT
        indexrelid::regclass as idx,
        (indrelid::text ||E'\n'|| indclass::text ||E'\n'|| indkey::text ||E'\n'||
         COALESCE(indexprs::text,'')||E'\n' || COALESCE(indpred::text,'')) as key
    FROM pg_index
) sub
GROUP BY key
HAVING COUNT(*) > 1
ORDER BY SUM(pg_relation_size(idx)) DESC;
```

**Reindex:**

```sql
-- Reindex single index
REINDEX INDEX idx_users_email;

-- Reindex table
REINDEX TABLE users;

-- Reindex concurrently (doesn't lock table)
REINDEX INDEX CONCURRENTLY idx_users_email;
```

### 5.5 Index Design Checklist

**Before creating an index:**

- [ ] Is this query run frequently?
- [ ] Does the table have enough rows (>1000)?
- [ ] Will this index help the query?
- [ ] Check EXPLAIN plan to verify
- [ ] Consider partial index for filtered queries
- [ ] Use multi-column index for multiple conditions
- [ ] Monitor index usage after creation

**Index Design Example:**

```sql
-- Query to optimize:
SELECT
    p.name,
    p.price,
    c.name as category
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.is_active = true
AND p.price < 100
AND p.category_id IN (1, 2, 3)
ORDER BY p.created_at DESC
LIMIT 20;

-- Optimal indexes:

-- 1. Partial index for active products with price and category
CREATE INDEX idx_products_active_category_price ON products(category_id, price, created_at DESC)
WHERE is_active = true;

-- 2. Primary key on categories (automatic)
-- 3. Check EXPLAIN to verify
EXPLAIN ANALYZE <query>;
```

---

## 6. Denormalization (When & How)

### 6.1 Understanding Denormalization

**Denormalization** = Intentionally introducing redundancy for performance.

**When to Denormalize:**

1. Read-heavy workload (10:1 read:write ratio or higher)
2. Expensive joins on every query
3. Aggregations calculated repeatedly
4. Historical snapshots needed
5. Reporting/analytics queries

**When NOT to Denormalize:**

1. Write-heavy workload
2. Data changes frequently
3. You're normalizing (start with 3NF, denormalize only if needed)
4. Small tables

### 6.2 Common Denormalization Patterns

#### **Pattern 1: Precompute Aggregations**

❌ **Normalized (Slow):**

```sql
-- Calculate on every query
SELECT
    u.id,
    u.username,
    COUNT(o.id) as order_count,
    SUM(o.total_amount) as total_spent,
    AVG(o.total_amount) as avg_order
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username;
```

✅ **Denormalized (Fast):**

```sql
-- Add columns to users table
ALTER TABLE users ADD COLUMN order_count INTEGER DEFAULT 0;
ALTER TABLE users ADD COLUMN total_spent DECIMAL(12,2) DEFAULT 0;
ALTER TABLE users ADD COLUMN avg_order DECIMAL(10,2) DEFAULT 0;

-- Update with trigger
CREATE OR REPLACE FUNCTION update_user_stats()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE users
        SET order_count = order_count + 1,
            total_spent = total_spent + NEW.total_amount,
            avg_order = (total_spent + NEW.total_amount) / (order_count + 1)
        WHERE id = NEW.user_id;
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE users
        SET total_spent = total_spent - OLD.total_amount + NEW.total_amount,
            avg_order = total_spent / order_count
        WHERE id = NEW.user_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE users
        SET order_count = order_count - 1,
            total_spent = total_spent - OLD.total_amount,
            avg_order = CASE WHEN order_count - 1 = 0 THEN 0 ELSE total_spent / (order_count - 1) END
        WHERE id = OLD.user_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_user_order_stats
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_user_stats();

-- Now this query is instant:
SELECT id, username, order_count, total_spent, avg_order
FROM users
WHERE id = 123;
```

#### **Pattern 2: Snapshot Historical Data**

**Scenario:** E-commerce order should preserve product details at time of purchase.

✅ **Denormalized:**

```sql
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT REFERENCES orders(id),
    product_id BIGINT REFERENCES products(id),
    quantity INTEGER NOT NULL,

    -- Snapshot of product at purchase time
    product_name VARCHAR(200) NOT NULL,
    product_price DECIMAL(10,2) NOT NULL,
    product_category VARCHAR(100),

    -- Actual paid price (might be different due to discounts)
    unit_price DECIMAL(10,2) NOT NULL,
    line_total DECIMAL(10,2) NOT NULL
);

-- When inserting order item, copy current product data:
INSERT INTO order_items (
    order_id,
    product_id,
    quantity,
    product_name,
    product_price,
    unit_price,
    line_total
)
SELECT
    NEW.order_id,
    p.id,
    NEW.quantity,
    p.name,              -- Snapshot
    p.price,             -- Snapshot
    NEW.unit_price,      -- Actual price paid
    NEW.quantity * NEW.unit_price
FROM products p
WHERE p.id = NEW.product_id;
```

**Why?** Even if product name/price changes later, order history shows what was actually purchased.

#### **Pattern 3: Duplicate Related Data**

**Scenario:** Every query needs user's name and email along with their orders.

✅ **Denormalized:**

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),

    -- Denormalized user data
    user_name VARCHAR(100),
    user_email VARCHAR(100),

    total_amount DECIMAL(10,2),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Update when user changes
CREATE OR REPLACE FUNCTION update_denormalized_user_data()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.name IS DISTINCT FROM OLD.name OR NEW.email IS DISTINCT FROM OLD.email THEN
        UPDATE orders
        SET user_name = NEW.name,
            user_email = NEW.email
        WHERE user_id = NEW.id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_orders_user_data
    AFTER UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_denormalized_user_data();

-- Now no join needed:
SELECT id, user_name, user_email, total_amount
FROM orders
WHERE user_id = 123;
```

#### **Pattern 4: Flatten Hierarchy**

**Scenario:** Categories have parent-child relationships, but queries always need full path.

❌ **Normalized (Requires recursive query):**

```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100),
    parent_id BIGINT REFERENCES categories(id)
);

-- Complex query to get full path:
WITH RECURSIVE category_path AS (
    SELECT id, name, parent_id, name as path
    FROM categories
    WHERE id = 10
    UNION ALL
    SELECT c.id, c.name, c.parent_id, c.name || ' > ' || cp.path
    FROM categories c
    JOIN category_path cp ON c.id = cp.parent_id
)
SELECT path FROM category_path WHERE parent_id IS NULL;
```

✅ **Denormalized (Fast):**

```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100),
    parent_id BIGINT REFERENCES categories(id),

    -- Denormalized: Full path
    path TEXT,  -- "Electronics > Computers > Laptops"
    level INTEGER,
    root_id BIGINT
);

-- Update path when category moves
CREATE OR REPLACE FUNCTION update_category_path()
RETURNS TRIGGER AS $$
DECLARE
    parent_path TEXT;
    parent_level INTEGER;
BEGIN
    IF NEW.parent_id IS NULL THEN
        NEW.path := NEW.name;
        NEW.level := 0;
        NEW.root_id := NEW.id;
    ELSE
        SELECT path, level, root_id
        INTO parent_path, parent_level, NEW.root_id
        FROM categories
        WHERE id = NEW.parent_id;

        NEW.path := parent_path || ' > ' || NEW.name;
        NEW.level := parent_level + 1;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_category_path
    BEFORE INSERT OR UPDATE ON categories
    FOR EACH ROW
    EXECUTE FUNCTION update_category_path();
```

### 6.3 Denormalization Maintenance Strategies

**Strategy 1: Triggers (Immediate Consistency)**

```sql
-- Pros: Always consistent, automatic
-- Cons: Adds overhead to writes

CREATE TRIGGER maintain_denormalized_data
    AFTER INSERT OR UPDATE OR DELETE ON source_table
    FOR EACH ROW
    EXECUTE FUNCTION sync_denormalized_data();
```

**Strategy 2: Application Layer (Eventual Consistency)**

```python
# Pros: Decoupled, can batch updates
# Cons: Temporary inconsistency, more complex code

def update_product(product_id, new_price):
    # Update product
    Product.objects.filter(id=product_id).update(price=new_price)

    # Async job to update denormalized data
    update_denormalized_prices.delay(product_id)
```

**Strategy 3: Scheduled Rebuild (Batch Processing)**

```sql
-- Pros: No overhead on writes, can optimize batch
-- Cons: Stale data between runs

-- Run every hour via cron
CREATE OR REPLACE FUNCTION rebuild_user_stats()
RETURNS VOID AS $$
BEGIN
    UPDATE users u
    SET order_count = stats.order_count,
        total_spent = stats.total_spent,
        avg_order = stats.avg_order
    FROM (
        SELECT
            user_id,
            COUNT(*) as order_count,
            SUM(total_amount) as total_spent,
            AVG(total_amount) as avg_order
        FROM orders
        GROUP BY user_id
    ) stats
    WHERE u.id = stats.user_id;
END;
$$ LANGUAGE plpgsql;
```

**Strategy 4: Materialized Views (PostgreSQL Built-in)**

```sql
-- Pros: Built-in, easy to maintain
-- Cons: Refresh can lock table, not real-time

CREATE MATERIALIZED VIEW user_order_summary AS
SELECT
    u.id,
    u.username,
    COUNT(o.id) as order_count,
    COALESCE(SUM(o.total_amount), 0) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username;

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_summary;
```

### 6.4 Denormalization Checklist

Before denormalizing:

- [ ] Measure actual query performance (use EXPLAIN ANALYZE)
- [ ] Calculate read:write ratio (denormalize if reads > 10x writes)
- [ ] Consider index first (often solves the problem)
- [ ] Plan maintenance strategy (triggers, jobs, materialized views)
- [ ] Document why you're denormalizing
- [ ] Monitor data consistency
- [ ] Have rollback plan

---

## 7. The 7 Strategies to Scale Database

### 7.1 Strategy #1: Indexing

**What:** Create data structures that speed up data retrieval.

**When to Use:**

- Read-heavy workloads
- Large tables (>10K rows)
- Frequent WHERE/JOIN/ORDER BY clauses

**Implementation:**

```sql
-- Identify slow queries
EXPLAIN ANALYZE
SELECT * FROM products WHERE category_id = 5 AND price < 100;

-- Create appropriate index
CREATE INDEX idx_products_category_price ON products(category_id, price);

-- Verify improvement
EXPLAIN ANALYZE
SELECT * FROM products WHERE category_id = 5 AND price < 100;
```

**Best Practices:**

- Index foreign keys
- Use partial indexes for filtered queries
- Don't over-index (each index slows writes)
- Monitor index usage and remove unused indexes

---

### 7.2 Strategy #2: Denormalization

**What:** Add redundant data to reduce joins and improve read performance.

**When to Use:**

- Expensive joins on every query
- Read-heavy workload (>10:1 read:write)
- Need for query optimization outweighs data consistency concerns

**Implementation:**

```sql
-- Normalized (slow)
SELECT u.name, o.total, o.created_at
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.id = 123;

-- Denormalized (fast)
ALTER TABLE orders ADD COLUMN user_name VARCHAR(100);

-- Maintain with trigger
CREATE TRIGGER sync_user_name
    AFTER UPDATE ON users
    FOR EACH ROW
    WHEN (NEW.name IS DISTINCT FROM OLD.name)
    EXECUTE FUNCTION update_order_user_names();

-- Now query without join
SELECT user_name, total, created_at
FROM orders
WHERE id = 123;
```

**Best Practices:**

- Only denormalize after measuring performance
- Use triggers or jobs to maintain consistency
- Document denormalized fields
- Consider materialized views as alternative

---

### 7.3 Strategy #3: Database Caching

**What:** Store frequently accessed data in fast storage layer (Redis, Memcached).

**When to Use:**

- Same queries run repeatedly
- Read-heavy workload
- Expensive computations
- Session/user data

**Cache Layers:**

```
Application → Cache → Database

1. Cache Hit: Return immediately
2. Cache Miss: Query DB → Store in cache → Return
```

**Implementation (Django + Redis):**

```python
import redis
from django.core.cache import cache

# Cache product details (expensive query with joins)
def get_product_details(product_id):
    cache_key = f'product:{product_id}'

    # Try cache first
    cached = cache.get(cache_key)
    if cached:
        return cached

    # Cache miss: query database
    product = Product.objects.select_related('category').get(id=product_id)

    # Store in cache for 1 hour
    cache.set(cache_key, product, timeout=3600)
    return product

# Invalidate cache when product changes
def update_product(product_id, **kwargs):
    Product.objects.filter(id=product_id).update(**kwargs)
    cache.delete(f'product:{product_id}')
```

**PostgreSQL Built-in Caching:**

```sql
-- Shared buffers (in postgresql.conf)
shared_buffers = 4GB  -- 25% of RAM

-- Check cache hit ratio
SELECT
    SUM(heap_blks_read) as heap_read,
    SUM(heap_blks_hit) as heap_hit,
    SUM(heap_blks_hit) / (SUM(heap_blks_hit) + SUM(heap_blks_read)) as hit_ratio
FROM pg_statio_user_tables;
-- Aim for >0.99 (99% cache hit rate)
```

**Best Practices:**

- Set appropriate TTL (Time To Live)
- Cache stable data (product details, not stock levels)
- Invalidate cache on updates
- Use cache for expensive aggregations
- Monitor cache hit rate

---

### 7.4 Strategy #4: Materialized Views

**What:** Precompute and store query results for fast access.

**When to Use:**

- Complex aggregations
- Multi-table joins
- Reporting dashboards
- Data doesn't need to be real-time

**Implementation:**

```sql
-- Complex dashboard query
CREATE MATERIALIZED VIEW sales_dashboard AS
SELECT
    DATE_TRUNC('day', o.created_at) as date,
    c.name as category,
    COUNT(DISTINCT o.id) as order_count,
    COUNT(DISTINCT o.user_id) as unique_customers,
    SUM(o.total_amount) as revenue,
    AVG(o.total_amount) as avg_order_value
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
JOIN categories c ON p.category_id = c.id
WHERE o.status = 'completed'
GROUP BY DATE_TRUNC('day', o.created_at), c.name;

-- Create index on materialized view
CREATE INDEX idx_sales_dashboard_date ON sales_dashboard(date);

-- Query is now instant
SELECT * FROM sales_dashboard
WHERE date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY date DESC, revenue DESC;

-- Refresh periodically (e.g., every hour)
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_dashboard;
```

**Refresh Strategies:**

```sql
-- Full refresh (locks view)
REFRESH MATERIALIZED VIEW sales_dashboard;

-- Concurrent refresh (no lock, requires unique index)
CREATE UNIQUE INDEX idx_sales_dashboard_unique ON sales_dashboard(date, category);
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_dashboard;

-- Scheduled refresh (cron job)
0 * * * * psql -d mydb -c "REFRESH MATERIALIZED VIEW CONCURRENTLY sales_dashboard;"
```

**Best Practices:**

- Use for reporting/analytics, not transactional data
- Create unique index for concurrent refresh
- Balance refresh frequency vs staleness tolerance
- Monitor view size and query performance

---

### 7.5 Strategy #5: Vertical Scaling

**What:** Increase server resources (CPU, RAM, storage).

**When to Use:**

- Single server can handle load
- Before implementing complex distributed solutions
- When hardware upgrade is cheaper than engineering time

**Optimization Steps:**

**1. Increase RAM (Most Important)**

```bash
# postgresql.conf
shared_buffers = 8GB        # 25% of RAM
effective_cache_size = 24GB # 75% of RAM
work_mem = 50MB            # Per query operation
maintenance_work_mem = 2GB  # For VACUUM, CREATE INDEX
```

**2. Optimize Storage**

- Use SSD instead of HDD
- Separate OS, WAL, and data on different disks
- Use RAID 10 for redundancy and performance

**3. Connection Pooling**

```bash
# Use PgBouncer to manage connections
max_connections = 200       # PostgreSQL
default_pool_size = 25      # PgBouncer
```

**4. Tune PostgreSQL Parameters**

```bash
# postgresql.conf

# Memory
shared_buffers = 8GB
work_mem = 64MB
maintenance_work_mem = 2GB

# Checkpoint
checkpoint_completion_target = 0.9
wal_buffers = 16MB
max_wal_size = 4GB

# Query Planning
random_page_cost = 1.1  # For SSD (default 4.0 for HDD)
effective_io_concurrency = 200

# Logging
log_min_duration_statement = 1000  # Log queries >1s
```

**Monitoring:**

```sql
-- Check CPU and memory usage
SELECT
    pid,
    usename,
    application_name,
    state,
    query,
    backend_start,
    state_change
FROM pg_stat_activity
WHERE state = 'active';

-- Check bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

**Limits:**

- Single server capacity (usually 128GB RAM, 32+ cores)
- Cost increases exponentially
- Single point of failure

---

### 7.6 Strategy #6: Replication

**What:** Create copies of database on multiple servers for read scaling and redundancy.

**Types:**

**1. Primary-Replica (Master-Slave)**

```
Primary (Write) ──────> Replica 1 (Read)
                 ├────> Replica 2 (Read)
                 └────> Replica 3 (Read)
```

**When to Use:**

- Read-heavy workload (95%+ reads)
- Geographic distribution
- High availability
- Backup and disaster recovery

**Setup (PostgreSQL Streaming Replication):**

```sql
-- On Primary Server
-- postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB

-- Create replication user
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'password';

-- pg_hba.conf
host replication replicator replica-ip/32 md5

-- On Replica Server
-- Create base backup
pg_basebackup -h primary-ip -D /var/lib/postgresql/data -U replicator -P

-- standby.signal (create empty file)
touch /var/lib/postgresql/data/standby.signal

-- postgresql.conf
primary_conninfo = 'host=primary-ip port=5432 user=replicator password=password'
```

**Application Layer Routing:**

```python
# Django settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'primary-db-server',  # Writes go here
        # ... other config
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'replica-db-server',  # Reads go here
        # ... other config
    }
}

# Route reads to replica
from django.db import connections

# Write
User.objects.create(username='test')

# Read from replica
User.objects.using('replica').filter(is_active=True)
```

**2. Multi-Primary (Multi-Master)**

- All nodes accept writes
- Complex conflict resolution
- Higher availability
- Use PostgreSQL extensions like Citus or BDR

**Best Practices:**

- Monitor replication lag
- Handle failover scenarios
- Use async replication for performance
- Implement connection pooling per server

**Monitoring Replication:**

```sql
-- On Primary: Check replication status
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    sync_state
FROM pg_stat_replication;

-- On Replica: Check lag
SELECT
    now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

---

### 7.7 Strategy #7: Sharding

**What:** Split data across multiple databases horizontally.

**When to Use:**

- Single database can't handle data volume
- Need to scale beyond single server limits
- Geographic distribution of data
- Regulatory requirements (data locality)

**Sharding Strategies:**

**1. Range-Based Sharding**

```
Shard 1: user_id 1-1,000,000
Shard 2: user_id 1,000,001-2,000,000
Shard 3: user_id 2,000,001-3,000,000
```

```python
def get_shard(user_id):
    if user_id <= 1_000_000:
        return 'shard1'
    elif user_id <= 2_000_000:
        return 'shard2'
    else:
        return 'shard3'
```

**Pros:** Simple, range queries work
**Cons:** Uneven distribution, hotspots

**2. Hash-Based Sharding**

```python
def get_shard(user_id):
    shard_count = 4
    shard_number = hash(user_id) % shard_count
    return f'shard{shard_number}'
```

**Pros:** Even distribution
**Cons:** Range queries difficult, hard to rebalance

**3. Geographic Sharding**

```
Shard US: users.country = 'US'
Shard EU: users.country IN ('GB', 'FR', 'DE')
Shard ASIA: users.country IN ('JP', 'CN', 'IN')
```

**Pros:** Low latency, regulatory compliance
**Cons:** Uneven distribution

**4. Entity-Based Sharding**

```
Shard 1: tenant_id 1-100
Shard 2: tenant_id 101-200
```

**Pros:** Tenant data stays together
**Cons:** Large tenants can create hotspots

**Implementation Example:**

```python
# Django with multiple databases
DATABASES = {
    'shard_0': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'shard0.example.com',
        # ... config
    },
    'shard_1': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'shard1.example.com',
        # ... config
    },
    'shard_2': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'shard2.example.com',
        # ... config
    },
}

# Router
class ShardRouter:
    def db_for_read(self, model, **hints):
        if model._meta.app_label == 'users':
            user_id = hints.get('instance').id if hints.get('instance') else None
            return self.get_shard(user_id)
        return 'default'

    def get_shard(self, user_id):
        if not user_id:
            return 'shard_0'
        shard_count = 3
        shard_number = hash(user_id) % shard_count
        return f'shard_{shard_number}'
```

**Challenges:**

1. **Cross-shard queries** - Expensive or impossible
2. **Transactions** - Can't span multiple shards
3. **Rebalancing** - Moving data between shards is complex
4. **Schema changes** - Must apply to all shards

**Best Practices:**

- Choose shard key carefully (must be in every query)
- Keep related data on same shard
- Use connection pooling per shard
- Monitor shard sizes and rebalance proactively
- Plan for adding shards (consistent hashing)

**Alternative: PostgreSQL Citus Extension**

```sql
-- Automatic sharding with Citus
CREATE EXTENSION citus;

-- Distribute table across nodes
SELECT create_distributed_table('users', 'id');

-- Citus handles routing automatically
SELECT * FROM users WHERE id = 123;
```

---

## 8. Real-World Design Examples

### 8.1 E-commerce Platform (Like Amazon)

**Requirements:**

- Millions of products
- High read traffic (browsing)
- Complex search and filtering
- Order history and tracking
- Inventory management
- User reviews and ratings

**Schema Design:**

```sql
-- Users
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    is_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);

-- Categories (hierarchical)
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    parent_id BIGINT REFERENCES categories(id),
    path TEXT,  -- Denormalized: "Electronics > Computers > Laptops"
    level INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_categories_parent ON categories(parent_id);
CREATE INDEX idx_categories_path ON categories USING gin(to_tsvector('english', path));

-- Products
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    category_id BIGINT NOT NULL REFERENCES categories(id),
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    compare_at_price DECIMAL(10,2),  -- Original price for discounts
    cost DECIMAL(10,2),  -- Cost to business

    -- Stock
    stock_quantity INTEGER DEFAULT 0 CHECK (stock_quantity >= 0),
    low_stock_threshold INTEGER DEFAULT 10,

    -- Denormalized for performance
    category_path TEXT,  -- "Electronics > Computers > Laptops"
    review_count INTEGER DEFAULT 0,
    average_rating DECIMAL(3,2) DEFAULT 0,
    total_sales INTEGER DEFAULT 0,

    -- Metadata
    is_active BOOLEAN DEFAULT true,
    is_featured BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for common queries
CREATE INDEX idx_products_category ON products(category_id) WHERE is_active = true;
CREATE INDEX idx_products_price ON products(price) WHERE is_active = true;
CREATE INDEX idx_products_featured ON products(is_featured) WHERE is_active = true;
CREATE INDEX idx_products_name_search ON products USING gin(to_tsvector('english', name || ' ' || description));

-- Product variants (for sizes, colors, etc.)
CREATE TABLE product_variants (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,  -- "Large / Red"
    price_adjustment DECIMAL(10,2) DEFAULT 0,
    stock_quantity INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true
);

CREATE INDEX idx_variants_product ON product_variants(product_id);

-- Orders
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    user_id BIGINT NOT NULL REFERENCES users(id),

    -- Denormalized user info (snapshot at order time)
    user_email VARCHAR(100) NOT NULL,
    user_name VARCHAR(100),

    -- Pricing
    subtotal DECIMAL(10,2) NOT NULL,
    tax DECIMAL(10,2) DEFAULT 0,
    shipping_cost DECIMAL(10,2) DEFAULT 0,
    discount DECIMAL(10,2) DEFAULT 0,
    total_amount DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),

    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    CONSTRAINT ck_order_status CHECK (status IN ('pending', 'paid', 'processing', 'shipped', 'delivered', 'cancelled')),

    -- Addresses (denormalized)
    shipping_address JSONB NOT NULL,
    billing_address JSONB NOT NULL,

    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    paid_at TIMESTAMPTZ,
    shipped_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    cancelled_at TIMESTAMPTZ
);

CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at DESC);

-- Order Items
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT REFERENCES products(id),  -- Can be NULL if product deleted
    variant_id BIGINT REFERENCES product_variants(id),

    -- Snapshot at purchase time
    product_sku VARCHAR(50) NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    product_variant VARCHAR(100),

    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL,
    line_total DECIMAL(10,2) NOT NULL,

    UNIQUE(order_id, product_id, variant_id)
);

CREATE INDEX idx_order_items_order ON order_items(order_id);

-- Reviews
CREATE TABLE product_reviews (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL REFERENCES users(id),
    order_item_id BIGINT REFERENCES order_items(id),  -- Verified purchase

    rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title VARCHAR(200),
    content TEXT,

    helpful_count INTEGER DEFAULT 0,
    is_verified_purchase BOOLEAN DEFAULT false,
    is_approved BOOLEAN DEFAULT false,

    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, product_id)  -- One review per user per product
);

CREATE INDEX idx_reviews_product ON product_reviews(product_id) WHERE is_approved = true;
CREATE INDEX idx_reviews_user ON product_reviews(user_id);

-- Update product stats on review
CREATE OR REPLACE FUNCTION update_product_review_stats()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE products
    SET review_count = (
            SELECT COUNT(*) FROM product_reviews
            WHERE product_id = NEW.product_id AND is_approved = true
        ),
        average_rating = (
            SELECT AVG(rating)::DECIMAL(3,2) FROM product_reviews
            WHERE product_id = NEW.product_id AND is_approved = true
        )
    WHERE id = NEW.product_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_review_stats
    AFTER INSERT OR UPDATE ON product_reviews
    FOR EACH ROW
    WHEN (NEW.is_approved = true)
    EXECUTE FUNCTION update_product_review_stats();

-- Shopping Cart
CREATE TABLE cart_items (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    session_id VARCHAR(100),  -- For anonymous users
    product_id BIGINT NOT NULL REFERENCES products(id),
    variant_id BIGINT REFERENCES product_variants(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, product_id, variant_id),
    UNIQUE(session_id, product_id, variant_id)
);

CREATE INDEX idx_cart_user ON cart_items(user_id);
CREATE INDEX idx_cart_session ON cart_items(session_id);

-- Wishlist
CREATE TABLE wishlists (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, product_id)
);

CREATE INDEX idx_wishlist_user ON wishlists(user_id);
```

**Performance Optimizations:**

1. **Search Index:**

```sql
-- Full-text search
CREATE INDEX idx_products_fulltext ON products
USING gin(to_tsvector('english', name || ' ' || coalesce(description, '')));

-- Search query
SELECT * FROM products
WHERE to_tsvector('english', name || ' ' || description) @@ to_tsquery('english', 'laptop & gaming')
AND is_active = true
ORDER BY average_rating DESC, total_sales DESC
LIMIT 20;
```

2. **Materialized View for Popular Products:**

```sql
CREATE MATERIALIZED VIEW popular_products AS
SELECT
    p.id,
    p.name,
    p.price,
    p.average_rating,
    p.review_count,
    p.total_sales,
    c.path as category_path
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.is_active = true
AND p.total_sales > 100
ORDER BY p.total_sales DESC
LIMIT 1000;

CREATE UNIQUE INDEX idx_popular_products_id ON popular_products(id);

-- Refresh every hour
REFRESH MATERIALIZED VIEW CONCURRENTLY popular_products;
```

3. **Partitioning Orders by Date:**

```sql
-- Partition orders table by year-month
CREATE TABLE orders (
    -- ... columns
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Create automatically via cron
```

---

### 8.2 Social Media Platform (Like Twitter/X)

**Requirements:**

- Posts/tweets with likes, comments, retweets
- Follow/follower relationships
- Feed generation (home timeline)
- Notifications
- Direct messages
- Trending topics

**Schema Design:**

```sql
-- Users
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    bio TEXT,
    avatar_url TEXT,

    -- Denormalized counts
    follower_count INTEGER DEFAULT 0,
    following_count INTEGER DEFAULT 0,
    post_count INTEGER DEFAULT 0,

    is_verified BOOLEAN DEFAULT false,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_username ON users(username) WHERE is_active = true;

-- Posts
CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    content TEXT NOT NULL CHECK (char_length(content) <= 280),

    -- Denormalized user data (for faster timeline queries)
    user_username VARCHAR(50) NOT NULL,
    user_display_name VARCHAR(100),
    user_avatar_url TEXT,

    -- Reply/Retweet
    parent_post_id BIGINT REFERENCES posts(id),  -- For replies
    repost_of_id BIGINT REFERENCES posts(id),    -- For retweets

    -- Counts (denormalized)
    like_count INTEGER DEFAULT 0,
    repost_count INTEGER DEFAULT 0,
    reply_count INTEGER DEFAULT 0,
    view_count INTEGER DEFAULT 0,

    -- Metadata
    is_deleted BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_posts_user ON posts(user_id, created_at DESC) WHERE is_deleted = false;
CREATE INDEX idx_posts_parent ON posts(parent_post_id) WHERE parent_post_id IS NOT NULL;
CREATE INDEX idx_posts_created ON posts(created_at DESC) WHERE is_deleted = false;

-- Follows (optimized for fan-out queries)
CREATE TABLE follows (
    id BIGSERIAL PRIMARY KEY,
    follower_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    following_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(follower_id, following_id),
    CHECK (follower_id != following_id)
);

CREATE INDEX idx_follows_follower ON follows(follower_id);
CREATE INDEX idx_follows_following ON follows(following_id);

-- Likes
CREATE TABLE post_likes (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, post_id)
);

CREATE INDEX idx_likes_post ON post_likes(post_id);
CREATE INDEX idx_likes_user ON post_likes(user_id, created_at DESC);

-- Update post like count
CREATE OR REPLACE FUNCTION update_post_like_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE posts SET like_count = like_count + 1 WHERE id = NEW.post_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE posts SET like_count = like_count - 1 WHERE id = OLD.post_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_like_count
    AFTER INSERT OR DELETE ON post_likes
    FOR EACH ROW
    EXECUTE FUNCTION update_post_like_count();

-- Feed Cache (fanout on write)
-- When user posts, insert into all followers' feeds
CREATE TABLE feed_items (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL,

    UNIQUE(user_id, post_id)
);

CREATE INDEX idx_feed_user_time ON feed_items(user_id, created_at DESC);

-- Insert into feed on new post
CREATE OR REPLACE FUNCTION fanout_post_to_followers()
RETURNS TRIGGER AS $$
BEGIN
    -- Insert into followers' feeds
    INSERT INTO feed_items (user_id, post_id, created_at)
    SELECT follower_id, NEW.id, NEW.created_at
    FROM follows
    WHERE following_id = NEW.user_id;

    -- Also insert into author's feed
    INSERT INTO feed_items (user_id, post_id, created_at)
    VALUES (NEW.user_id, NEW.id, NEW.created_at);

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER fanout_to_feed
    AFTER INSERT ON posts
    FOR EACH ROW
    WHEN (NEW.is_deleted = false)
    EXECUTE FUNCTION fanout_post_to_followers();

-- Get user timeline (fast!)
SELECT
    p.id,
    p.content,
    p.user_username,
    p.like_count,
    p.repost_count,
    p.created_at
FROM feed_items f
JOIN posts p ON f.post_id = p.id
WHERE f.user_id = 123
AND p.is_deleted = false
ORDER BY f.created_at DESC
LIMIT 20;

-- Notifications
CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL,  -- 'like', 'follow', 'reply', 'mention'
    actor_id BIGINT REFERENCES users(id),  -- Who triggered the notification
    post_id BIGINT REFERENCES posts(id),

    message TEXT,
    is_read BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_notifications_user_unread ON notifications(user_id, created_at DESC)
WHERE is_read = false;
```

**Performance Strategies:**

1. **Timeline Generation (Fanout on Write)**

   - When user posts, write to all followers' feeds
   - Reading timeline is fast (pre-computed)
   - Trade-off: High-follower accounts are expensive to write

2. **Hybrid Approach for Celebrities**

```sql
-- Don't fanout for users with >100k followers
-- Instead, compute their timeline on read

CREATE OR REPLACE FUNCTION should_fanout(p_user_id BIGINT)
RETURNS BOOLEAN AS $$
DECLARE
    follower_count INTEGER;
BEGIN
    SELECT u.follower_count INTO follower_count
    FROM users u
    WHERE id = p_user_id;

    RETURN follower_count < 100000;
END;
$$ LANGUAGE plpgsql;
```

3. **Sharding Strategy**
   - Shard users by user_id
   - Shard posts by post_id
   - Shard feed_items by user_id (collocate with user)

---

### 8.3 Islamic Learning Platform (GTAF Context)

**Requirements:**

- Quran with multiple translations and tafsir
- Prayer times by location
- Courses and lessons
- User progress tracking
- Community features (Q&A, discussions)
- Multi-language support

**Schema Design:**

```sql
-- Quran Structure
CREATE TABLE surahs (
    id INTEGER PRIMARY KEY,  -- 1-114
    name_arabic VARCHAR(100) NOT NULL,
    name_transliteration VARCHAR(100) NOT NULL,
    name_translation VARCHAR(200) NOT NULL,
    revelation_place VARCHAR(20) CHECK (revelation_place IN ('Makkah', 'Madinah')),
    verse_count INTEGER NOT NULL,
    order_of_revelation INTEGER
);

CREATE TABLE verses (
    id BIGSERIAL PRIMARY KEY,
    surah_number INTEGER NOT NULL REFERENCES surahs(id),
    verse_number INTEGER NOT NULL,
    arabic_text TEXT NOT NULL,

    -- Simple text for matching/search
    arabic_simple TEXT NOT NULL,  -- Without diacritics

    -- Audio
    audio_url TEXT,

    UNIQUE(surah_number, verse_number)
);

CREATE INDEX idx_verses_surah ON verses(surah_number, verse_number);
CREATE INDEX idx_verses_search ON verses USING gin(to_tsvector('arabic', arabic_simple));

-- Translations
CREATE TABLE translations (
    id SERIAL PRIMARY KEY,
    language_code VARCHAR(10) NOT NULL,
    translator_name VARCHAR(100) NOT NULL,
    translator_info TEXT,
    is_active BOOLEAN DEFAULT true,
    UNIQUE(language_code, translator_name)
);

CREATE TABLE verse_translations (
    id BIGSERIAL PRIMARY KEY,
    verse_id BIGINT NOT NULL REFERENCES verses(id) ON DELETE CASCADE,
    translation_id INTEGER NOT NULL REFERENCES translations(id),
    text TEXT NOT NULL,
    footnotes TEXT,

    UNIQUE(verse_id, translation_id)
);

CREATE INDEX idx_verse_translations_verse ON verse_translations(verse_id);
CREATE INDEX idx_verse_translations_translation ON verse_translations(translation_id);
CREATE INDEX idx_verse_translations_search ON verse_translations USING gin(to_tsvector('english', text));

-- Tafsir (Quranic commentary)
CREATE TABLE tafsir_sources (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    author VARCHAR(100),
    language_code VARCHAR(10) NOT NULL,
    description TEXT
);

CREATE TABLE verse_tafsir (
    id BIGSERIAL PRIMARY KEY,
    verse_id BIGINT NOT NULL REFERENCES verses(id) ON DELETE CASCADE,
    tafsir_source_id INTEGER NOT NULL REFERENCES tafsir_sources(id),
    text TEXT NOT NULL,

    UNIQUE(verse_id, tafsir_source_id)
);

CREATE INDEX idx_tafsir_verse ON verse_tafsir(verse_id);

-- User Reading Progress
CREATE TABLE user_reading_progress (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    surah_number INTEGER NOT NULL REFERENCES surahs(id),
    verse_number INTEGER NOT NULL,
    last_read_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, surah_number)
);

-- User Bookmarks
CREATE TABLE user_bookmarks (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    verse_id BIGINT NOT NULL REFERENCES verses(id) ON DELETE CASCADE,
    note TEXT,
    tags TEXT[],
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, verse_id)
);

CREATE INDEX idx_bookmarks_user ON user_bookmarks(user_id, created_at DESC);

-- Prayer Times
CREATE TABLE cities (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    name_arabic VARCHAR(100),
    country_code VARCHAR(2) NOT NULL,
    latitude DECIMAL(10, 6) NOT NULL,
    longitude DECIMAL(10, 6) NOT NULL,
    timezone VARCHAR(50) NOT NULL,

    -- For search
    search_vector tsvector
);

CREATE INDEX idx_cities_country ON cities(country_code);
CREATE INDEX idx_cities_search ON cities USING gin(search_vector);
CREATE INDEX idx_cities_location ON cities(latitude, longitude);

-- Prayer times (pre-calculated for performance)
CREATE TABLE prayer_times (
    id BIGSERIAL PRIMARY KEY,
    city_id INTEGER NOT NULL REFERENCES cities(id),
    date DATE NOT NULL,

    fajr TIME NOT NULL,
    sunrise TIME NOT NULL,
    dhuhr TIME NOT NULL,
    asr TIME NOT NULL,
    maghrib TIME NOT NULL,
    isha TIME NOT NULL,

    calculation_method VARCHAR(50) DEFAULT 'MWL',

    UNIQUE(city_id, date)
);

-- Partition by year
CREATE TABLE prayer_times (
    -- ... columns
) PARTITION BY RANGE (date);

CREATE TABLE prayer_times_2024 PARTITION OF prayer_times
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE INDEX idx_prayer_times_city_date ON prayer_times(city_id, date);

-- Courses
CREATE TABLE courses (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,
    description TEXT,
    instructor_id BIGINT REFERENCES users(id),
    category VARCHAR(50),

    -- Denormalized
    instructor_name VARCHAR(100),
    lesson_count INTEGER DEFAULT 0,
    enrolled_count INTEGER DEFAULT 0,
    average_rating DECIMAL(3,2) DEFAULT 0,

    is_published BOOLEAN DEFAULT false,
    is_free BOOLEAN DEFAULT true,
    price DECIMAL(10,2),

    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_courses_category ON courses(category) WHERE is_published = true;
CREATE INDEX idx_courses_instructor ON courses(instructor_id);

-- Lessons
CREATE TABLE lessons (
    id BIGSERIAL PRIMARY KEY,
    course_id BIGINT NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    content TEXT,
    video_url TEXT,
    duration_minutes INTEGER,
    order_position INTEGER NOT NULL,

    is_published BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_lessons_course ON lessons(course_id, order_position);

-- Enrollments
CREATE TABLE course_enrollments (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    course_id BIGINT NOT NULL REFERENCES courses(id) ON DELETE CASCADE,

    progress_percentage INTEGER DEFAULT 0 CHECK (progress_percentage BETWEEN 0 AND 100),
    completed_lesson_count INTEGER DEFAULT 0,

    enrolled_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    last_accessed_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,

    UNIQUE(user_id, course_id)
);

CREATE INDEX idx_enrollments_user ON course_enrollments(user_id);
CREATE INDEX idx_enrollments_course ON course_enrollments(course_id);

-- Lesson Progress
CREATE TABLE lesson_progress (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    lesson_id BIGINT NOT NULL REFERENCES lessons(id) ON DELETE CASCADE,

    is_completed BOOLEAN DEFAULT false,
    progress_percentage INTEGER DEFAULT 0,
    last_position_seconds INTEGER,

    completed_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,

    UNIQUE(user_id, lesson_id)
);

CREATE INDEX idx_lesson_progress_user ON lesson_progress(user_id);

-- Daily Adhkar (Remembrance/Supplications)
CREATE TABLE adhkar_categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    name_arabic VARCHAR(100),
    description TEXT,
    icon VARCHAR(50)
);

CREATE TABLE adhkar (
    id SERIAL PRIMARY KEY,
    category_id INTEGER NOT NULL REFERENCES adhkar_categories(id),
    arabic_text TEXT NOT NULL,
    transliteration TEXT,
    translation TEXT NOT NULL,
    reference TEXT,  -- Hadith reference
    repetition_count INTEGER DEFAULT 1,

    order_position INTEGER
);

CREATE INDEX idx_adhkar_category ON adhkar(category_id, order_position);

-- User Adhkar Tracking
CREATE TABLE user_adhkar_completions (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    adhkar_id INTEGER NOT NULL REFERENCES adhkar(id),
    completed_count INTEGER DEFAULT 0,
    completion_date DATE NOT NULL DEFAULT CURRENT_DATE,

    UNIQUE(user_id, adhkar_id, completion_date)
);

CREATE INDEX idx_adhkar_completions_user_date ON user_adhkar_completions(user_id, completion_date DESC);
```

**Performance Optimizations:**

1. **Verse Lookup Optimization:**

```sql
-- Materialized view for popular verses
CREATE MATERIALIZED VIEW popular_verses AS
SELECT
    v.id,
    v.surah_number,
    v.verse_number,
    v.arabic_text,
    s.name_transliteration,
    json_agg(
        json_build_object(
            'language', t.language_code,
            'translator', tr.translator_name,
            'text', vt.text
        )
    ) as translations
FROM verses v
JOIN surahs s ON v.surah_number = s.id
LEFT JOIN verse_translations vt ON v.id = vt.verse_id
LEFT JOIN translations tr ON vt.translation_id = tr.id
WHERE v.id IN (
    -- Most bookmarked verses
    SELECT verse_id FROM user_bookmarks
    GROUP BY verse_id
    ORDER BY COUNT(*) DESC
    LIMIT 100
)
GROUP BY v.id, v.surah_number, v.verse_number, v.arabic_text, s.name_transliteration;
```

2. **Prayer Times Caching:**

```sql
-- Pre-generate prayer times for next 2 years
-- Run monthly via cron job

CREATE OR REPLACE FUNCTION generate_prayer_times(
    p_city_id INTEGER,
    p_start_date DATE,
    p_end_date DATE
)
RETURNS VOID AS $$
DECLARE
    current_date DATE;
BEGIN
    current_date := p_start_date;
    WHILE current_date <= p_end_date LOOP
        -- Calculate and insert (use external library like praytimes.org API)
        -- This is pseudocode
        INSERT INTO prayer_times (city_id, date, fajr, sunrise, dhuhr, asr, maghrib, isha)
        VALUES (p_city_id, current_date, ...calculated times...)
        ON CONFLICT (city_id, date) DO NOTHING;

        current_date := current_date + INTERVAL '1 day';
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

3. **Multi-language Search:**

```sql
-- Combined search across Arabic and translations
CREATE OR REPLACE FUNCTION search_quran(
    p_query TEXT,
    p_language VARCHAR DEFAULT 'en'
)
RETURNS TABLE (
    surah_number INTEGER,
    verse_number INTEGER,
    arabic_text TEXT,
    translation TEXT,
    rank REAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        v.surah_number,
        v.verse_number,
        v.arabic_text,
        vt.text as translation,
        ts_rank(
            to_tsvector(p_language, vt.text),
            to_tsquery(p_language, p_query)
        ) as rank
    FROM verses v
    JOIN verse_translations vt ON v.id = vt.verse_id
    JOIN translations t ON vt.translation_id = t.id
    WHERE t.language_code = p_language
    AND to_tsvector(p_language, vt.text) @@ to_tsquery(p_language, p_query)
    ORDER BY rank DESC
    LIMIT 50;
END;
$$ LANGUAGE plpgsql;
```

---

## 9. Design Anti-Patterns to Avoid

### 9.1 The "God Table" Anti-Pattern

❌ **Bad:** One massive table with everything.

```sql
CREATE TABLE everything (
    id BIGSERIAL PRIMARY KEY,
    user_name VARCHAR(100),
    user_email VARCHAR(100),
    product_name VARCHAR(200),
    product_price DECIMAL(10,2),
    order_date DATE,
    order_total DECIMAL(10,2),
    review_text TEXT,
    review_rating INTEGER,
    -- ... 50 more columns
);
```

✅ **Good:** Properly normalized separate tables.

```sql
CREATE TABLE users (...);
CREATE TABLE products (...);
CREATE TABLE orders (...);
CREATE TABLE reviews (...);
```

### 9.2 The "Entity-Attribute-Value" (EAV) Anti-Pattern

❌ **Bad:** Trying to make a "flexible" schema.

```sql
CREATE TABLE entity_attributes (
    entity_id BIGINT,
    attribute_name VARCHAR(50),
    attribute_value TEXT
);

-- Nightmare to query:
SELECT
    e1.attribute_value as name,
    e2.attribute_value as email,
    e3.attribute_value as age
FROM entity_attributes e1
JOIN entity_attributes e2 ON e1.entity_id = e2.entity_id AND e2.attribute_name = 'email'
JOIN entity_attributes e3 ON e1.entity_id = e3.entity_id AND e3.attribute_name = 'age'
WHERE e1.attribute_name = 'name';
```

✅ **Good:** Use JSONB for flexible attributes.

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200),
    price DECIMAL(10,2),
    attributes JSONB  -- Flexible attributes
);

-- Easy to query:
SELECT * FROM products
WHERE attributes->>'color' = 'red';
```

### 9.3 The "Lack of Constraints" Anti-Pattern

❌ **Bad:** No constraints, relying on application logic.

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT,  -- No foreign key
    total DECIMAL(10,2)  -- Could be negative
);
```

✅ **Good:** Let the database enforce rules.

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    total DECIMAL(10,2) NOT NULL CHECK (total >= 0)
);
```

### 9.4 The "Premature Optimization" Anti-Pattern

❌ **Bad:** Denormalizing before measuring performance.

```sql
-- Adding redundant data without proof it's needed
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    order_count INTEGER DEFAULT 0,  -- Maybe not needed?
    total_spent DECIMAL(12,2) DEFAULT 0,  -- Maybe not needed?
    avg_order DECIMAL(10,2) DEFAULT 0  -- Maybe not needed?
);
```

✅ **Good:** Start normalized, optimize based on data.

```sql
-- Start simple
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100)
);

-- Measure: Is this query slow?
EXPLAIN ANALYZE
SELECT user_id, COUNT(*), SUM(total) FROM orders GROUP BY user_id;

-- If slow AND run frequently, then denormalize
```

### 9.5 The "Magic Numbers" Anti-Pattern

❌ **Bad:** Using magic values instead of proper types or tables.

```sql
-- What does status = 3 mean?
SELECT * FROM orders WHERE status = 3;
```

✅ **Good:** Use enums, lookup tables, or constraints.

```sql
-- Option 1: Enum
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'completed', 'cancelled');
ALTER TABLE orders ADD COLUMN status order_status DEFAULT 'pending';

-- Option 2: Check constraint
ALTER TABLE orders ADD COLUMN status VARCHAR(20)
CHECK (status IN ('pending', 'processing', 'completed', 'cancelled'));

-- Option 3: Lookup table (for dynamic values)
CREATE TABLE order_statuses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(20) UNIQUE NOT NULL,
    description TEXT
);
```

---

## 10. Production-Ready Design Checklist

### 10.1 Schema Design Checklist

**Before Deploying:**

- [ ] **Primary Keys**

  - [ ] Every table has a primary key
  - [ ] Use BIGSERIAL (not SERIAL) for future-proofing
  - [ ] Avoid using business logic fields as primary keys

- [ ] **Foreign Keys**

  - [ ] All relationships have foreign key constraints
  - [ ] Specify ON DELETE behavior (CASCADE, SET NULL, RESTRICT)
  - [ ] Index all foreign key columns

- [ ] **Indexes**

  - [ ] Index all foreign keys
  - [ ] Index columns used in WHERE clauses
  - [ ] Index columns used in ORDER BY
  - [ ] Index columns used in JOINs
  - [ ] Don't over-index (each index slows writes)

- [ ] **Constraints**

  - [ ] NOT NULL where appropriate
  - [ ] CHECK constraints for valid values
  - [ ] UNIQUE constraints for unique data
  - [ ] Default values where sensible

- [ ] **Data Types**

  - [ ] Appropriate types for data (not everything is VARCHAR)
  - [ ] DECIMAL for money (not FLOAT)
  - [ ] TIMESTAMPTZ for timestamps (not TIMESTAMP)
  - [ ] TEXT for long strings (not VARCHAR(MAX))

- [ ] **Audit Columns**

  - [ ] created_at on all tables
  - [ ] updated_at with trigger on all tables
  - [ ] deleted_at for soft deletes where needed

- [ ] **Naming Conventions**
  - [ ] Consistent table names (plural, lowercase)
  - [ ] Consistent column names (snake_case)
  - [ ] Descriptive names (not abbreviations)
  - [ ] Standard suffixes (\_id, \_at, \_count)

### 10.2 Performance Checklist

- [ ] **Query Performance**

  - [ ] Identified top 10 queries
  - [ ] Optimized indexes for these queries
  - [ ] Used EXPLAIN ANALYZE to verify
  - [ ] Eliminated N+1 query problems

- [ ] **Scaling Strategy**

  - [ ] Determined read:write ratio
  - [ ] Planned for data growth (partitioning?)
  - [ ] Considered replication for reads
  - [ ] Identified sharding key if needed

- [ ] **Caching**

  - [ ] Identified cacheable queries
  - [ ] Implemented caching layer
  - [ ] Cache invalidation strategy in place

- [ ] **Monitoring**
  - [ ] Slow query logging enabled
  - [ ] pg_stat_statements extension installed
  - [ ] Monitoring dashboard set up
  - [ ] Alerts for slow queries

### 10.3 Security Checklist

- [ ] **Access Control**

  - [ ] Separate read-only and read-write users
  - [ ] Principle of least privilege
  - [ ] Row-level security where needed
  - [ ] SSL/TLS for connections

- [ ] **Data Protection**

  - [ ] Passwords hashed (bcrypt/argon2)
  - [ ] Sensitive data encrypted
  - [ ] PII data clearly marked
  - [ ] GDPR compliance considered

- [ ] **Backups**
  - [ ] Automated daily backups
  - [ ] Backup tested and verified
  - [ ] Point-in-time recovery enabled
  - [ ] Backup retention policy defined

### 10.4 Documentation Checklist

- [ ] **Schema Documentation**

  - [ ] ER diagram created
  - [ ] Table purposes documented
  - [ ] Relationships explained
  - [ ] Denormalized fields justified

- [ ] **Query Documentation**

  - [ ] Complex queries commented
  - [ ] Index rationale documented
  - [ ] Performance benchmarks recorded

- [ ] **Maintenance Documentation**
  - [ ] Migration strategy documented
  - [ ] Rollback procedures documented
  - [ ] Monitoring runbook created
  - [ ] Common issues and solutions documented

---

## Conclusion

You now have a comprehensive guide to database design and scalability! Here's your path forward:

### Immediate Actions (This Week)

1. ✅ Review your current GTAF schemas
2. ✅ Run EXPLAIN ANALYZE on top queries
3. ✅ Add missing indexes (especially foreign keys)
4. ✅ Document your schemas with ER diagrams

### Short-term (This Month)

1. ✅ Normalize to 3NF
2. ✅ Identify denormalization opportunities
3. ✅ Implement monitoring
4. ✅ Set up replication for reads

### Medium-term (Next 3 Months)

1. ✅ Implement caching layer
2. ✅ Create materialized views for analytics
3. ✅ Optimize slow queries
4. ✅ Plan for sharding if needed

### Long-term (Next 6-12 Months)

1. ✅ Master advanced PostgreSQL features
2. ✅ Contribute to database optimization at GTAF
3. ✅ Mentor others on database design
4. ✅ Achieve Staff/Principal Engineer level expertise

**Remember:**

- Start simple, optimize when needed
- Measure before optimizing
- Document your decisions
- Design for maintainability

Good luck with your journey to database mastery! 🚀

---

**Created for Maharab - Backend Engineer at Greentech Apps Foundation**
_From Schema Design to Production-Scale Architecture_
