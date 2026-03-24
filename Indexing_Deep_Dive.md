# PostgreSQL Indexing Deep Dive
## How Indexes Actually Work — From B-Tree Internals to Production Decisions

> "An index is a bet. You're betting that the cost of maintaining it will be paid back by the queries that use it. Most engineers place too many bets without doing the math."

---

## Table of Contents

1. [How B-Tree Indexes Actually Work](#1-how-b-tree-indexes-actually-work)
   - 1.1 [The Tree Structure](#11-the-tree-structure)
   - 1.2 [What Lives in a Leaf Page](#12-what-lives-in-a-leaf-page)
   - 1.3 [How a Point Lookup Works](#13-how-a-point-lookup-works)
   - 1.4 [How a Range Scan Works](#14-how-a-range-scan-works)
   - 1.5 [Page Splits and Write Performance](#15-page-splits-and-write-performance)
   - 1.6 [Why Index Order Matters](#16-why-index-order-matters)
2. [Index Scan Types](#2-index-scan-types)
   - 2.1 [Sequential Scan](#21-sequential-scan)
   - 2.2 [Index Scan](#22-index-scan)
   - 2.3 [Index Only Scan](#23-index-only-scan)
   - 2.4 [Bitmap Index Scan + Bitmap Heap Scan](#24-bitmap-index-scan--bitmap-heap-scan)
   - 2.5 [The Random I/O Problem](#25-the-random-io-problem)
   - 2.6 [Forcing vs Allowing Planner Choice](#26-forcing-vs-allowing-planner-choice)
3. [B-Tree Index in Depth](#3-b-tree-index-in-depth)
   - 3.1 [Supported and Unsupported Operators](#31-supported-and-unsupported-operators)
   - 3.2 [Multicolumn Indexes and Column Order](#32-multicolumn-indexes-and-column-order)
   - 3.3 [The Leading Column Rule](#33-the-leading-column-rule)
   - 3.4 [Sort Optimization](#34-sort-optimization)
   - 3.5 [Unique Indexes](#35-unique-indexes)
4. [Partial Indexes](#4-partial-indexes)
   - 4.1 [What They Are and How They Work](#41-what-they-are-and-how-they-work)
   - 4.2 [Real Use Cases](#42-real-use-cases)
   - 4.3 [Size Advantage](#43-size-advantage)
   - 4.4 [The Catch](#44-the-catch)
5. [Covering Indexes (INCLUDE)](#5-covering-indexes-include)
   - 5.1 [What Index-Only Scans Need](#51-what-index-only-scans-need)
   - 5.2 [The INCLUDE Clause](#52-the-include-clause)
   - 5.3 [Before and After EXPLAIN](#53-before-and-after-explain)
6. [Expression Indexes](#6-expression-indexes)
   - 6.1 [Case-Insensitive Lookups](#61-case-insensitive-lookups)
   - 6.2 [Time Bucket Queries](#62-time-bucket-queries)
   - 6.3 [Computed Column Indexes](#63-computed-column-indexes)
   - 6.4 [Requirements and Cost](#64-requirements-and-cost)
7. [GIN Index — JSONB, Arrays, Full-Text](#7-gin-index--jsonb-arrays-full-text)
   - 7.1 [How GIN Works](#71-how-gin-works)
   - 7.2 [JSONB Operators](#72-jsonb-operators)
   - 7.3 [Array Operators](#73-array-operators)
   - 7.4 [Full-Text Search](#74-full-text-search)
   - 7.5 [GIN vs GiST](#75-gin-vs-gist)
   - 7.6 [fastupdate Parameter](#76-fastupdate-parameter)
8. [BRIN Index — Massive Sequential Tables](#8-brin-index--massive-sequential-tables)
   - 8.1 [How BRIN Works](#81-how-brin-works)
   - 8.2 [When to Use It](#82-when-to-use-it)
   - 8.3 [Size Comparison](#83-size-comparison)
   - 8.4 [pages_per_range](#84-pages_per_range)
9. [GiST Index — Geometric and Range Types](#9-gist-index--geometric-and-range-types)
   - 9.1 [Use Cases](#91-use-cases)
   - 9.2 [Range Type Queries](#92-range-type-queries)
10. [Hash Index](#10-hash-index)
    - 10.1 [When to Choose Hash over B-Tree](#101-when-to-choose-hash-over-b-tree)
11. [Index Bloat — The Silent Killer](#11-index-bloat--the-silent-killer)
    - 11.1 [Why Indexes Bloat](#111-why-indexes-bloat)
    - 11.2 [Detecting Bloat](#112-detecting-bloat)
    - 11.3 [REINDEX vs REINDEX CONCURRENTLY](#113-reindex-vs-reindex-concurrently)
    - 11.4 [fillfactor](#114-fillfactor)
12. [Multi-Column Index Strategy](#12-multi-column-index-strategy)
    - 12.1 [The Core Rule: Equality First, Range Last](#121-the-core-rule-equality-first-range-last)
    - 12.2 [Composite vs Separate Indexes](#122-composite-vs-separate-indexes)
    - 12.3 [Five Query Patterns and Their Index Designs](#123-five-query-patterns-and-their-index-designs)
13. [When NOT to Index](#13-when-not-to-index)
    - 13.1 [The Real Cost of Every Index](#131-the-real-cost-of-every-index)
    - 13.2 [Conditions That Rule Out an Index](#132-conditions-that-rule-out-an-index)
14. [Index Maintenance Queries](#14-index-maintenance-queries)
    - 14.1 [Find Unused Indexes](#141-find-unused-indexes)
    - 14.2 [Find Missing Indexes](#142-find-missing-indexes)
    - 14.3 [Find Duplicate Indexes](#143-find-duplicate-indexes)
    - 14.4 [Find Index Bloat](#144-find-index-bloat)
    - 14.5 [Index Size Comparison](#145-index-size-comparison)
    - 14.6 [Full Index Health Check](#146-full-index-health-check)
15. [Real-World Indexing Decisions](#15-real-world-indexing-decisions)
    - 15.1 [E-Commerce Product Search](#151-e-commerce-product-search)
    - 15.2 [User Authentication](#152-user-authentication)
    - 15.3 [Order History](#153-order-history)
    - 15.4 [Job Queue with SKIP LOCKED](#154-job-queue-with-skip-locked)
    - 15.5 [Multi-Tenant SaaS](#155-multi-tenant-saas)

---

## 1. How B-Tree Indexes Actually Work

### 1.1 The Tree Structure

A PostgreSQL B-Tree index is a balanced tree of fixed-size pages (8KB by default). Every page in the tree is one of three types:

```
Root Page
    │
    ├── Branch Page  (key ranges + pointers to child pages)
    │       ├── Branch Page
    │       │       ├── Leaf Page  (key + TID pairs, linked list)
    │       │       └── Leaf Page
    │       └── Branch Page
    │               ├── Leaf Page
    │               └── Leaf Page
    └── Branch Page
            ├── Leaf Page
            └── Leaf Page
```

**Root page**: The single entry point. Always at a fixed location in the index file. Contains high-level key separators and pointers to branch pages.

**Branch pages**: Internal navigation nodes. Each entry holds a key value and a pointer to a child page that contains entries with values less than or equal to that key. Branch pages never contain heap pointers — they exist solely for navigation.

**Leaf pages**: Where the actual data lives. Each entry is a (key_value, TID) pair. All leaf pages are linked in a doubly-linked list, ordered by key. This is what makes range scans efficient — once you find the start, you walk the linked list.

The tree stays balanced because PostgreSQL maintains the B-Tree invariant: every leaf is at the same depth. The height of the tree grows logarithmically with the number of entries. A table with 100 million rows with an 8-byte integer key typically produces a B-Tree with height 4-5 — meaning any lookup touches at most 5 pages.

### 1.2 What Lives in a Leaf Page

Each leaf page contains an array of index tuples. Every index tuple has:

- **Key value(s)**: The actual indexed column value(s), stored in the data type's on-disk format
- **TID (Tuple Identifier)**: A physical pointer to the heap row, encoded as `(block_number, item_offset)` — 6 bytes total

```
Leaf Page Layout:
┌─────────────────────────────────────────────┐
│ Page Header (24 bytes)                      │
│   - prev_page, next_page (linked list ptrs) │
│   - flags, free space info                  │
├─────────────────────────────────────────────┤
│ Item Pointers (array of offsets)            │
├─────────────────────────────────────────────┤
│ Free Space                                  │
├─────────────────────────────────────────────┤
│ Index Tuples (bottom up)                    │
│   [key=100, TID=(block=42, offset=7)]       │
│   [key=101, TID=(block=42, offset=8)]       │
│   [key=103, TID=(block=99, offset=2)]       │
│   [key=103, TID=(block=99, offset=4)]  ← duplicate key, different TID │
│   ...                                       │
└─────────────────────────────────────────────┘
```

Key insight: the TID is not a logical row ID — it is a physical location on disk. When VACUUM moves rows (VACUUM FULL), TIDs change, and all indexes must be updated. This is why VACUUM FULL is expensive and locks the table.

Duplicate key values are allowed in non-unique indexes. PostgreSQL stores them as separate entries, sorted first by key then by TID.

### 1.3 How a Point Lookup Works

For a query like `SELECT * FROM orders WHERE order_id = 12345`:

1. PostgreSQL reads the root page from disk (or buffer cache)
2. Binary search within the root page finds the branch page whose range contains `12345`
3. Read that branch page, binary search again to find the next-level branch or leaf
4. Repeat until a leaf page is reached
5. Binary search within the leaf page finds the exact (key, TID) entry
6. Use the TID to fetch the heap page at `(block_number, item_offset)`
7. Read the full row from the heap page
8. Apply any remaining filter conditions (e.g., WHERE clauses not covered by the index)

Total pages read: `tree_height + 1` (the `+1` is the heap page fetch). For a 100M row table, that's roughly 5-6 page reads to find any single row, regardless of table size. That is the O(log n) guarantee.

### 1.4 How a Range Scan Works

For a query like `SELECT * FROM orders WHERE order_id BETWEEN 10000 AND 10500`:

1. Traverse the tree top-down to find the leaf page containing `10000`
2. Scan linearly through the leaf linked list, collecting TIDs for all keys `<= 10500`
3. For each TID collected, fetch the corresponding heap page

The leaf linked list is what makes range scans efficient. Once the starting position is found (O(log n)), the range traversal is O(k) where k is the number of matching entries. There is no need to re-traverse the tree for each entry in the range.

This linked list traversal happens in sorted key order, which is why ORDER BY on an indexed column can be satisfied without a sort step — the index hands back TIDs in the right order already.

### 1.5 Page Splits and Write Performance

When a leaf page is full and a new entry must be inserted, PostgreSQL must split the page:

1. A new page is allocated
2. Half the entries from the full page are moved to the new page
3. A new separator key is added to the parent branch page pointing to the new leaf
4. If the parent branch page is also full, it splits too, propagating upward
5. In the worst case, the root splits, increasing the tree height by 1

**What this means for write performance:**

Every `INSERT` that adds to an index potentially triggers a page split. Page splits require:
- Writing two leaf pages (original + new)
- Writing the modified parent branch page
- Updating the linked list pointers

Splits are expensive: they generate write-ahead log (WAL) records for all modified pages. A heavily inserted index will generate significant WAL traffic purely from splits.

For sequential inserts (auto-incrementing IDs), splits only ever happen at the rightmost leaf — there is always one "hot" page at the far right. This is optimal; only one leaf is split at a time, and the tree grows cleanly rightward.

For random inserts (UUIDs, random values), splits happen anywhere in the tree. Over time, pages fill unevenly — some are densely packed, some are nearly empty from deletions. This fragmentation is called **index bloat**.

> **Insight**: This is a strong argument for using BIGSERIAL or GENERATED ALWAYS AS IDENTITY over UUID v4 as a primary key when you have high insert volume. The sequential nature of BIGSERIAL keeps the rightmost-leaf hot pattern, reducing split frequency and WAL pressure.

### 1.6 Why Index Order Matters for Range Queries

The B-Tree stores entries in sorted order. This sorted order is the source of its power for range queries. Consider two columns: `customer_id` and `order_date`.

Index on `(customer_id, order_date)`:

```
Leaf pages (logical view):
[(cust=1, date=2024-01-01), (cust=1, date=2024-02-15), (cust=1, date=2024-11-30),
 (cust=2, date=2024-03-10), (cust=2, date=2024-07-22), ...]
```

A query for `customer_id = 5 AND order_date BETWEEN '2024-01-01' AND '2024-12-31'` can seek directly to the first `(5, 2024-01-01)` entry and scan linearly — all matching rows are physically adjacent in the index.

Index on `(order_date, customer_id)` with the same query:

The planner must find all entries in the date range first, then filter on `customer_id`. Matching rows for `customer_id = 5` are scattered across all leaf pages covering the date range. This is far less efficient — more pages read, more heap fetches.

The rule: **put the equality-filtered columns first, then the range-filtered columns**. The index can seek precisely on equality, then scan linearly on the range.

---

## 2. Index Scan Types

Understanding which scan type appears in EXPLAIN output — and why — is the single most important skill for debugging query performance.

### 2.1 Sequential Scan

PostgreSQL reads every page of the heap table in order, from block 0 to block N.

```sql
EXPLAIN SELECT * FROM orders WHERE status = 'completed';
```

```
Seq Scan on orders  (cost=0.00..18540.00 rows=45000 width=87)
  Filter: ((status)::text = 'completed'::text)
```

**PostgreSQL chooses a sequential scan when:**

1. **The selectivity is too low**: If the query matches >10-20% of rows (the threshold varies by cost settings), reading the entire heap sequentially is cheaper than random-jumping through the index. PostgreSQL's cost model compares `seq_page_cost * heap_pages` against `random_page_cost * index_pages + random_page_cost * matching_rows`.

2. **The table is small**: A table that fits in a few pages costs almost nothing to scan. The planner defaults to seq scan for very small tables even if an index exists.

3. **Statistics are stale**: If `pg_statistic` has outdated row counts or value distributions, the planner may overestimate result size and choose a seq scan. Run `ANALYZE` after large data changes.

4. **The cost constants are misconfigured**: `random_page_cost` defaults to 4.0, assuming spinning disks. On SSDs or NVMe, set it to 1.1-1.5. Overestimating random I/O cost biases the planner toward seq scan.

```sql
-- For SSD/NVMe storage
SET random_page_cost = 1.1;
SET effective_cache_size = '75GB'; -- available to OS page cache
```

### 2.2 Index Scan

PostgreSQL traverses the B-Tree to find matching TIDs, then fetches each corresponding heap row one by one.

```sql
EXPLAIN SELECT * FROM orders WHERE order_id = 12345;
```

```
Index Scan using orders_pkey on orders  (cost=0.43..8.45 rows=1 width=87)
  Index Cond: (order_id = 12345)
```

**Characteristics:**
- Accesses heap pages in random order (one per matching row)
- Each heap page access is a random I/O
- Optimal for high-selectivity queries (few matching rows)
- Performance degrades when matching rows are scattered across many heap pages

The heap fetch is the expensive part. If 1,000 rows match and they live on 1,000 different heap pages, you have 1,000 random I/O operations. This is why low-selectivity queries prefer seq scan — sequential I/O is fundamentally cheaper on HDDs, and the difference matters less but still exists on SSDs.

### 2.3 Index Only Scan

PostgreSQL traverses the index and returns data directly from the index pages, without touching the heap at all.

```sql
EXPLAIN SELECT order_id, created_at FROM orders WHERE customer_id = 42;
-- With index on (customer_id, order_id, created_at)
```

```
Index Only Scan using idx_orders_customer_date on orders
  (cost=0.43..12.20 rows=23 width=16)
  Index Cond: (customer_id = 42)
  Heap Fetches: 0
```

**Why it sometimes still touches the heap: the visibility map**

PostgreSQL's MVCC model means index entries can point to dead or not-yet-committed heap rows. Before returning a row from an index-only scan, PostgreSQL must verify the row is visible to the current transaction. It does this via the **visibility map** — a one-bit-per-page structure that marks heap pages as "all-visible" (every row on this page is visible to all transactions).

- If the visibility map says a heap page is all-visible: the row from the index is returned directly
- If the heap page is not marked all-visible: PostgreSQL fetches the heap row to check visibility

`Heap Fetches: 0` means every page referenced was all-visible. After a table is freshly vacuumed, this is common. On a table with heavy update/delete activity, heap fetches will be non-zero even on an "index only scan."

```sql
-- Check visibility map status
SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'orders';

-- Force visibility map to be updated
VACUUM orders;
```

### 2.4 Bitmap Index Scan + Bitmap Heap Scan

For queries that match many rows but still benefit from the index, PostgreSQL uses a two-phase approach.

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
-- When customer_id = 42 matches thousands of rows
```

```
Bitmap Heap Scan on orders  (cost=245.32..4821.44 rows=6800 width=87)
  Recheck Cond: (customer_id = 42)
  ->  Bitmap Index Scan on idx_orders_customer_id
        (cost=0.00..243.62 rows=6800 width=0)
        Index Cond: (customer_id = 42)
```

**Phase 1 — Bitmap Index Scan:**
- Traverses the index and collects all matching TIDs
- Builds an in-memory bitmap where each bit represents a heap page
- Sets the bit for every heap page containing at least one matching row
- Never touches the heap

**Phase 2 — Bitmap Heap Scan:**
- Reads the bitmap, identifies which heap pages need to be read
- Reads each required heap page exactly once (never twice), in physical order
- Re-checks the index condition against each row on the page (because the bitmap is page-granular, not row-granular)

**Why this is better than a plain Index Scan for medium selectivity:**
- Heap pages are read in physical order = sequential-ish I/O
- Each heap page is read at most once, even if multiple matching rows are on it
- Eliminates the random I/O thrashing of fetching one row at a time from scattered pages

**When Bitmap scans combine multiple indexes:**

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42 OR product_id = 99;
```

```
Bitmap Heap Scan on orders
  ->  BitmapOr
        ->  Bitmap Index Scan on idx_orders_customer_id
              Index Cond: (customer_id = 42)
        ->  Bitmap Index Scan on idx_orders_product_id
              Index Cond: (product_id = 99)
```

PostgreSQL builds two bitmaps and ORs them together before doing a single heap pass. This is the primary reason to sometimes favor separate single-column indexes over a single composite index — the planner can combine them with BitmapOr/BitmapAnd.

**Lossy bitmap mode:** When the bitmap would exceed `work_mem`, PostgreSQL switches to lossy mode — it stores only page numbers, not row locations. This is why you see `Recheck Cond` in the plan. Every row on every matched page must be rechecked against the original condition. Increase `work_mem` to avoid this if you see it frequently.

```sql
SET work_mem = '64MB'; -- per-session, not global
```

### 2.5 The Random I/O Problem

This is the core tension in index design. Consider a heap table with 1 million rows spread across 10,000 heap pages (8KB each).

**Scenario A: Query matches 100 rows, scattered across 100 pages**
- Index Scan: 100 random page reads for the index traversal path (~5 pages) + 100 heap page reads = ~105 random reads
- Sequential Scan: 10,000 sequential reads
- Winner: Index Scan (105 << 10,000)

**Scenario B: Query matches 200,000 rows**
- Index Scan: ~200,000 random heap reads (rows are scattered)
- Sequential Scan: 10,000 sequential reads
- Winner: Sequential Scan (10,000 sequential reads cost roughly the same as 2,500 random reads at `random_page_cost = 4.0`)

The crossover point depends on `random_page_cost`, `seq_page_cost`, and how scattered the matching rows are. PostgreSQL's cost model estimates this using the table's correlation statistic (`pg_stats.correlation`) — how correlated the physical order of rows is with the index order.

```sql
-- Check correlation for a column
SELECT tablename, attname, correlation
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY abs(correlation) DESC;
```

A correlation of 1.0 means rows are in perfect physical order relative to the index (sequential inserts by ID). A correlation near 0 means rows are randomly scattered. High correlation = Index Scan is cheaper. Low correlation = Bitmap or Seq Scan preferred.

### 2.6 Forcing vs Allowing Planner Choice

**Never use these in production. Use them for debugging only.**

```sql
-- Force the planner to avoid certain scan types
SET enable_seqscan = OFF;     -- force index use (disables seq scan)
SET enable_indexscan = OFF;   -- force bitmap or seq scan
SET enable_bitmapscan = OFF;  -- force index scan or seq scan
SET enable_indexonlyscan = OFF;

-- Then run your query and check the plan
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42;

-- Re-enable after debugging
RESET enable_seqscan;
```

```sql
-- The correct production tool: check why planner chose seq scan
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM orders WHERE customer_id = 42;
```

In the output, look for:
- `rows=X` (estimated) vs `actual rows=Y` — large divergence means stale statistics
- `Buffers: shared hit=X read=Y` — `read` means disk I/O, `hit` means buffer cache
- `actual time=X..Y` — wall clock time

If the planner is wrong, the fix is almost always one of: run ANALYZE, adjust `random_page_cost`/`effective_cache_size`, or rewrite the query to be more selective.

---

## 3. B-Tree Index in Depth

### 3.1 Supported and Unsupported Operators

B-Tree indexes support operators that correspond to a total ordering: things that can be sorted.

**Supported:**
```sql
-- Equality
WHERE id = 42
WHERE status = 'active'

-- Comparison
WHERE price > 100
WHERE price < 500
WHERE price >= 100 AND price <= 500
WHERE price BETWEEN 100 AND 500

-- NULL checks
WHERE deleted_at IS NULL
WHERE deleted_at IS NOT NULL

-- IN list (becomes OR of equality checks)
WHERE status IN ('pending', 'active')

-- Prefix LIKE (key insight: only prefix, not suffix)
WHERE email LIKE 'john%'     -- uses index
WHERE email LIKE '%gmail.com' -- does NOT use index

-- Prefix pattern matching
WHERE code LIKE 'PRD-%'      -- uses index
```

**Not supported by default B-Tree:**
```sql
WHERE LOWER(email) = 'john@example.com'  -- function call breaks index use
WHERE email ILIKE 'john%'                -- case-insensitive, no index
WHERE email LIKE '%gmail.com'            -- suffix pattern, no ordering basis
WHERE description @@ to_tsquery('foo')  -- full-text, needs GIN
WHERE tags @> ARRAY['postgres']         -- array containment, needs GIN
WHERE metadata @> '{"role": "admin"}'   -- JSONB containment, needs GIN
```

For LIKE prefix to use an index, the column must have `text_pattern_ops` or `varchar_pattern_ops` operator class if your database uses a locale other than C:

```sql
-- Safe cross-locale LIKE prefix index
CREATE INDEX idx_users_email_pattern ON users (email text_pattern_ops);

-- Or just use C locale from the start
CREATE DATABASE mydb LC_COLLATE='C' TEMPLATE=template0;
```

### 3.2 Multicolumn Indexes and Column Order

A multicolumn B-Tree index concatenates the column values to form a composite sort key. The index sorts by the first column, then by the second within equal first values, then by the third within equal (first, second) values, and so on.

```sql
CREATE INDEX idx_orders_composite ON orders (customer_id, status, created_at);
```

Logical sort order in the leaf pages:
```
(customer_id=1, status='active',    created_at=2024-01-05)
(customer_id=1, status='active',    created_at=2024-06-20)
(customer_id=1, status='completed', created_at=2024-01-01)
(customer_id=1, status='completed', created_at=2024-11-30)
(customer_id=2, status='active',    created_at=2024-03-15)
(customer_id=2, status='cancelled', created_at=2024-02-01)
...
```

**The column order determines which queries can use this index efficiently.**

### 3.3 The Leading Column Rule

For a composite index `(a, b, c)`, the index can be used for queries that filter on:

| Query Filter | Uses Index? | Notes |
|---|---|---|
| `WHERE a = ?` | Yes | Leading column, full index seek |
| `WHERE a = ? AND b = ?` | Yes | Two leading columns |
| `WHERE a = ? AND b = ? AND c = ?` | Yes | All columns |
| `WHERE a = ? AND c = ?` | Partial | Uses index on `a`, then filters `c` |
| `WHERE b = ?` | No | Not the leading column |
| `WHERE b = ? AND c = ?` | No | Not the leading column |
| `WHERE c = ?` | No | Not the leading column |
| `WHERE a > ? AND b = ?` | Partial | Range on `a` stops prefix matching |

The leading column rule is strict: the planner can only use the index from the leftmost column continuously. A gap in the equality chain stops efficient index use.

```sql
-- For index on (customer_id, status, created_at):

-- Uses full index (seek to exact prefix, then scan):
SELECT * FROM orders WHERE customer_id = 42 AND status = 'active';

-- Uses index for customer_id, filters status and created_at manually:
SELECT * FROM orders WHERE customer_id = 42 AND created_at > '2024-01-01';
-- (skips status — can still use index on customer_id alone)

-- Does NOT use this index at all:
SELECT * FROM orders WHERE status = 'active';
```

```sql
-- Verify with EXPLAIN
EXPLAIN SELECT * FROM orders WHERE status = 'active';
-- Will show Seq Scan (no index on leading status column)

EXPLAIN SELECT * FROM orders WHERE customer_id = 42 AND status = 'active';
-- Will show Index Scan using idx_orders_composite
```

### 3.4 Sort Optimization

When a query needs sorted output, the planner checks whether an index already provides that order.

```sql
CREATE INDEX idx_orders_customer_date ON orders (customer_id, created_at);

-- This query needs no sort step:
SELECT * FROM orders
WHERE customer_id = 42
ORDER BY created_at;
```

```
Index Scan using idx_orders_customer_date on orders
  Index Cond: (customer_id = 42)
  -- No "Sort" node above this — the index provided sorted output
```

Compare against a query without index alignment:

```sql
SELECT * FROM orders
WHERE customer_id = 42
ORDER BY total_amount;  -- not in the index
```

```
Sort  (cost=... rows=23 width=87)
  Sort Key: total_amount
  ->  Index Scan using idx_orders_customer_date on orders
        Index Cond: (customer_id = 42)
```

The sort node adds significant cost for large result sets. This is why indexes designed for queries that both filter and sort on related columns are powerful — one index eliminates both the random-access cost and the sort cost.

**Descending order and backward scans:**

B-Tree indexes can be scanned both forward and backward. An index on `created_at ASC` can satisfy `ORDER BY created_at DESC` via a backward scan, though it's slightly less efficient. For mixed directions:

```sql
-- This requires a specific index direction:
SELECT * FROM orders ORDER BY customer_id ASC, created_at DESC;

-- Create an index that matches the exact sort direction:
CREATE INDEX idx_orders_cust_date_desc ON orders (customer_id ASC, created_at DESC);
```

### 3.5 Unique Indexes

A UNIQUE constraint is implemented as a unique index. PostgreSQL checks the index before every INSERT or UPDATE to enforce the constraint.

```sql
-- These two are equivalent:
ALTER TABLE users ADD CONSTRAINT users_email_unique UNIQUE (email);
CREATE UNIQUE INDEX users_email_unique ON users (email);

-- Partial unique index — uniqueness only within the partial condition:
CREATE UNIQUE INDEX idx_active_user_email ON users (email)
  WHERE deleted_at IS NULL;
-- This allows multiple rows with the same email as long as only one is not deleted
```

The uniqueness check uses the index to do an equality lookup before the insert. For high-concurrency inserts, this creates a potential bottleneck on popular index pages (latch contention). This is the "index hot spot" problem common in payment systems inserting into narrow ID ranges.

---

## 4. Partial Indexes

### 4.1 What They Are and How They Work

A partial index is a B-Tree (or other type) index that only contains entries for rows satisfying a WHERE predicate.

```sql
CREATE INDEX idx_orders_pending ON orders (created_at)
  WHERE status = 'pending';
```

This index stores only TIDs and `created_at` values for rows where `status = 'pending'`. Rows with other status values are completely absent from the index. The index is smaller, faster to scan, and faster to maintain because fewer rows require index entries.

Internally, the index stores a predicate. When the planner considers using this index, it checks whether the query's WHERE clause implies the index predicate. If yes, the index is a candidate.

### 4.2 Real Use Cases

**Pattern 1: Index only active/non-deleted records**

```sql
-- Soft-delete pattern: most queries never touch deleted rows
CREATE INDEX idx_users_email_active ON users (email)
  WHERE deleted_at IS NULL;

-- This query uses the index:
SELECT id FROM users WHERE email = 'alice@example.com' AND deleted_at IS NULL;

-- This query does NOT use the index (doesn't match the predicate):
SELECT id FROM users WHERE email = 'alice@example.com'; -- no deleted_at filter

-- The query must include the exact predicate condition
```

**Pattern 2: Job queue — only index actionable rows**

```sql
CREATE INDEX idx_jobs_pending ON jobs (priority DESC, created_at ASC)
  WHERE status = 'pending';

-- Worker polling query — hits tiny index instead of full table:
SELECT id FROM jobs
WHERE status = 'pending'
ORDER BY priority DESC, created_at ASC
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

The power here: a jobs table might have 10 million completed jobs and 5,000 pending ones. A full B-Tree index on `(status, priority, created_at)` has 10 million entries. The partial index has 5,000 entries — 2,000x smaller. The index fits in a few pages. Worker polling is constant-time regardless of historical job volume.

**Pattern 3: Only recent data**

```sql
-- Analytics queries almost always look at recent data
CREATE INDEX idx_events_recent ON events (user_id, event_type, created_at)
  WHERE created_at >= '2024-01-01';

-- Useful if you partition old data or archive it regularly
-- Rebuild the index periodically as your date threshold changes
```

**Pattern 4: Non-null selective columns**

```sql
-- Only index rows where the column has a value (skip the NULLs)
CREATE INDEX idx_orders_coupon ON orders (coupon_code)
  WHERE coupon_code IS NOT NULL;

-- If 90% of orders have no coupon, this index is 10x smaller
```

### 4.3 Size Advantage

```sql
-- Check the size difference between full and partial indexes
CREATE INDEX idx_orders_status_full ON orders (created_at);
CREATE INDEX idx_orders_status_partial ON orders (created_at)
  WHERE status = 'pending';

SELECT
  indexname,
  pg_size_pretty(pg_relation_size(indexname::regclass)) AS index_size,
  pg_size_pretty(pg_relation_size('orders')) AS table_size
FROM pg_indexes
WHERE tablename = 'orders'
  AND indexname IN ('idx_orders_status_full', 'idx_orders_status_partial');
```

Typical output for a 10M row orders table where 0.1% are pending:

```
indexname                   | index_size | table_size
----------------------------+------------+-----------
idx_orders_status_full      | 214 MB     | 1872 MB
idx_orders_status_partial   | 224 kB     | 1872 MB
```

A 1,000x size reduction is common. Smaller index = more likely to fit in `shared_buffers` = faster queries every time.

### 4.4 The Catch

**The query's WHERE clause must imply the index predicate.** PostgreSQL's predicate checking is not always sophisticated enough to recognize indirect implications.

```sql
CREATE INDEX idx_jobs_pending ON jobs (created_at)
  WHERE status = 'pending';

-- Uses the index:
SELECT * FROM jobs WHERE status = 'pending' AND created_at < NOW();

-- Does NOT use the index (status compared to variable, not literal):
SELECT * FROM jobs WHERE status = $1 AND created_at < NOW();
-- Even if $1 = 'pending', the planner can't guarantee it at plan time

-- Does NOT use the index (predicate not present):
SELECT * FROM jobs WHERE created_at < NOW(); -- no status filter

-- Fix for parameterized queries: add the literal to the query
SELECT * FROM jobs WHERE status = 'pending' AND created_at < $1;
-- Now 'pending' is a literal and matches the index predicate
```

This is the most common partial index gotcha. If your application uses parameterized queries with the status value as a parameter, the partial index won't be used even if the value is always 'pending'. You need the literal in the SQL text.

---

## 5. Covering Indexes (INCLUDE)

### 5.1 What Index-Only Scans Need

An index-only scan works when all columns referenced by the query are present in the index. The index contains the values; no heap fetch is needed. The constraint is: only the indexed key columns are stored in the index pages.

Before PostgreSQL 11, the only way to make an index-only scan work was to add the extra columns as key columns in the composite index. This worked but was wasteful — non-key columns inflate every level of the B-Tree, including branch pages, and they force the index to maintain sort order on those columns even when you don't need it.

### 5.2 The INCLUDE Clause

PostgreSQL 11 introduced `INCLUDE` to add non-key columns to index leaf pages only. These columns are not part of the sort key and don't appear in branch pages.

```sql
-- Old approach: add email as a key column
CREATE INDEX idx_users_email_old ON users (username, email);
-- Problem: index is sorted by (username, email) — email affects the sort order
-- and inflates branch pages

-- New approach: INCLUDE email in leaf pages only
CREATE INDEX idx_users_covering ON users (username) INCLUDE (email, created_at);
-- Index is sorted by username only
-- email and created_at are stored only in leaf pages for index-only scan support
-- Branch pages are smaller (only contain username)
```

**When to use INCLUDE:**
- High-frequency queries that filter/sort on some columns and select a few extra columns
- The extra columns are wide (long strings, JSONB) — keeping them out of branch pages saves space
- You don't need to filter or sort by the included columns

```sql
-- Concrete example: user authentication lookup
-- Query: SELECT id, password_hash, role FROM users WHERE username = 'alice';

CREATE INDEX idx_users_auth ON users (username)
  INCLUDE (id, password_hash, role);

-- Now the entire query can be satisfied from the index alone
```

### 5.3 Before and After EXPLAIN

```sql
-- Table: users(id, username, email, password_hash, role, created_at, metadata)
-- Query: SELECT id, email FROM users WHERE username = 'alice';

-- Without covering index:
CREATE INDEX idx_users_username ON users (username);
EXPLAIN (ANALYZE, BUFFERS) SELECT id, email FROM users WHERE username = 'alice';
```

```
Index Scan using idx_users_username on users
  (cost=0.43..8.45 rows=1 width=52) (actual time=0.082..0.084 rows=1 loops=1)
  Index Cond: ((username)::text = 'alice'::text)
  Buffers: shared hit=4
```

The index found the TID, then fetched the heap page (that's the `hit=4` — root + branch + leaf + heap page).

```sql
-- With covering index:
CREATE INDEX idx_users_username_cover ON users (username) INCLUDE (id, email);
EXPLAIN (ANALYZE, BUFFERS) SELECT id, email FROM users WHERE username = 'alice';
```

```
Index Only Scan using idx_users_username_cover on users
  (cost=0.43..4.45 rows=1 width=52) (actual time=0.041..0.042 rows=1 loops=1)
  Index Cond: ((username)::text = 'alice'::text)
  Heap Fetches: 0
  Buffers: shared hit=3
```

One fewer buffer hit (no heap page), and `Heap Fetches: 0` confirms no heap access. On a table under concurrent write load, this also means no MVCC visibility check overhead.

**Trade-off:** The index with INCLUDE is larger because leaf pages now store extra data. For the auth example, this is acceptable — the index is still small and the query is on the critical path. For a wide INCLUDE on a large table, measure the size impact before committing.

---

## 6. Expression Indexes

### 6.1 Case-Insensitive Lookups

```sql
-- Problem: users type emails in mixed case
SELECT * FROM users WHERE email = 'Alice@Example.COM'; -- won't match 'alice@example.com'

-- Application-level: always store lowercase, always query lowercase
-- Database-level: expression index

CREATE INDEX idx_users_email_lower ON users (LOWER(email));

-- Query MUST use the exact same expression:
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com'; -- uses index
SELECT * FROM users WHERE email = 'alice@example.com';         -- does NOT use index
SELECT * FROM users WHERE email ILIKE 'alice@example.com';     -- does NOT use index

-- Check it works:
EXPLAIN SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
```

```
Index Scan using idx_users_email_lower on users
  Index Cond: (lower((email)::text) = 'alice@example.com'::text)
```

### 6.2 Time Bucket Queries

```sql
-- Problem: monthly aggregation queries are slow on a 100M row events table
SELECT DATE_TRUNC('month', created_at) AS month, COUNT(*)
FROM events
WHERE DATE_TRUNC('month', created_at) = '2024-06-01'
GROUP BY 1;

-- Expression index on the function:
CREATE INDEX idx_events_month ON events (DATE_TRUNC('month', created_at));

-- Query uses index when the expression matches exactly:
SELECT * FROM events WHERE DATE_TRUNC('month', created_at) = '2024-06-01'::timestamptz;

-- Alternative: use a range that the planner can optimize better
SELECT * FROM events
WHERE created_at >= '2024-06-01' AND created_at < '2024-07-01';
-- This uses a plain B-Tree index on created_at (preferred for range queries)
```

The range query approach is usually better than the expression index approach for time ranges, because the planner can use the B-Tree's range capability directly on the raw column.

### 6.3 Computed Column Indexes

```sql
-- E-commerce: find products where (price * quantity) > threshold
CREATE INDEX idx_order_items_total ON order_items ((price * quantity));

-- Query must use exact expression:
SELECT * FROM order_items WHERE (price * quantity) > 10000;

-- Or generate a stored computed column (PostgreSQL 12+):
ALTER TABLE order_items ADD COLUMN total_price NUMERIC
  GENERATED ALWAYS AS (price * quantity) STORED;
CREATE INDEX idx_order_items_total_v2 ON order_items (total_price);

-- Stored generated column is often cleaner — no expression matching requirement
```

```sql
-- Array length index for filtering by array size:
CREATE INDEX idx_posts_tag_count ON posts ((array_length(tags, 1)));
SELECT * FROM posts WHERE array_length(tags, 1) > 5;
```

### 6.4 Requirements and Cost

**Requirements:**
1. The query must use the exact same expression as the index definition — no simplification, no rewrite
2. The expression must be immutable (no NOW(), no random(), no CURRENT_USER) — PostgreSQL enforces this
3. Functions must be marked IMMUTABLE or STABLE for the planner to consider using the index

**Cost:**
- Every INSERT, UPDATE, or DELETE must evaluate the expression to maintain the index
- Complex expressions add measurable overhead to write operations
- For `LOWER(email)`: trivial cost, worth it
- For heavy JSON transformations or expensive functions: profile before indexing

```sql
-- Check if your function is immutable (required for expression indexes)
SELECT proname, provolatile
FROM pg_proc
WHERE proname = 'your_function';
-- provolatile = 'i' means IMMUTABLE (can be indexed)
-- provolatile = 's' means STABLE (can sometimes be used, but not ideal)
-- provolatile = 'v' means VOLATILE (cannot be indexed)
```

---

## 7. GIN Index — JSONB, Arrays, Full-Text Search

### 7.1 How GIN Works

GIN (Generalized Inverted Index) is an inverted index — the same structure used by search engines. Instead of mapping `row → attributes`, GIN maps `element → set of rows containing that element`.

For a JSONB column, GIN decomposes each JSON value into its constituent elements:

```json
{"role": "admin", "region": "us-east", "permissions": ["read", "write"]}
```

Gets decomposed into elements stored in the GIN:
```
"role"          → {row_5, row_18, row_201}
"admin"         → {row_5, row_99}
"region"        → {row_5, row_12, row_18}
"us-east"       → {row_5, row_12}
"permissions"   → {row_5, row_18}
"read"          → {row_5, row_18, row_99, row_201}
"write"         → {row_5, row_18}
```

To answer `WHERE metadata @> '{"role": "admin"}'`, GIN looks up "role" and "admin", intersects the result sets, and returns row 5 and row 99 — all without touching the heap for filtering.

This structure is extremely efficient for containment queries on multi-value columns, but it is expensive to build and update because every write must update multiple GIN postings lists (one per element).

### 7.2 JSONB Operators

```sql
-- Create table with JSONB column
CREATE TABLE events (
  id          BIGSERIAL PRIMARY KEY,
  user_id     BIGINT NOT NULL,
  event_type  TEXT NOT NULL,
  metadata    JSONB,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Create GIN index on the JSONB column
CREATE INDEX idx_events_metadata ON events USING GIN (metadata);

-- Operators that USE the GIN index:
SELECT * FROM events WHERE metadata @> '{"source": "mobile"}';
-- @> containment: rows where metadata contains the given JSON

SELECT * FROM events WHERE metadata ? 'campaign_id';
-- ? key existence: rows where metadata has the key 'campaign_id'

SELECT * FROM events WHERE metadata ?| ARRAY['campaign_id', 'experiment_id'];
-- ?| any key: rows where metadata has any of the listed keys

SELECT * FROM events WHERE metadata ?& ARRAY['campaign_id', 'user_segment'];
-- ?& all keys: rows where metadata has all of the listed keys

-- Operators that do NOT use the default GIN index:
SELECT * FROM events WHERE metadata->>'source' = 'mobile';
-- -> and ->> extract operators: NOT supported by default GIN
-- Fix: use @> instead, or create a B-Tree expression index on the extracted value

-- For frequent access to a specific JSON key:
CREATE INDEX idx_events_source ON events ((metadata->>'source'));
-- This is a B-Tree expression index, not GIN
```

**GIN with jsonb_path_ops (faster, smaller, limited):**

```sql
CREATE INDEX idx_events_metadata_path ON events
  USING GIN (metadata jsonb_path_ops);
-- Only supports @> operator, but 30-50% smaller than default GIN
-- Use when you only need containment queries, not key existence
```

### 7.3 Array Operators

```sql
CREATE TABLE posts (
  id    BIGSERIAL PRIMARY KEY,
  title TEXT,
  tags  TEXT[]
);

CREATE INDEX idx_posts_tags ON posts USING GIN (tags);

-- Operators that use GIN:
SELECT * FROM posts WHERE tags @> ARRAY['postgresql', 'indexing'];
-- @> containment: posts that have ALL of the specified tags

SELECT * FROM posts WHERE tags && ARRAY['postgresql', 'database'];
-- && overlap: posts that have ANY of the specified tags

-- Does NOT use GIN:
SELECT * FROM posts WHERE tags[1] = 'postgresql'; -- array subscript access
SELECT * FROM posts WHERE 'postgresql' = ANY(tags); -- ANY() construct — actually DOES work
```

```sql
-- Verify ANY() uses the index:
EXPLAIN SELECT * FROM posts WHERE 'postgresql' = ANY(tags);
-- Should show Bitmap Index Scan using idx_posts_tags
```

### 7.4 Full-Text Search

```sql
CREATE TABLE articles (
  id        BIGSERIAL PRIMARY KEY,
  title     TEXT NOT NULL,
  body      TEXT NOT NULL,
  ts_vector TSVECTOR GENERATED ALWAYS AS (
    setweight(to_tsvector('english', title), 'A') ||
    setweight(to_tsvector('english', body), 'B')
  ) STORED
);

-- GIN index on the stored tsvector:
CREATE INDEX idx_articles_fts ON articles USING GIN (ts_vector);

-- Full-text search queries:
SELECT id, title
FROM articles
WHERE ts_vector @@ to_tsquery('english', 'postgresql & indexing')
ORDER BY ts_rank(ts_vector, to_tsquery('english', 'postgresql & indexing')) DESC;

-- Phrase search:
SELECT id, title FROM articles
WHERE ts_vector @@ phraseto_tsquery('english', 'index scan');

-- Prefix search:
SELECT id, title FROM articles
WHERE ts_vector @@ to_tsquery('english', 'index:*');
```

### 7.5 GIN vs GiST

For full-text search specifically:

| Aspect | GIN | GiST |
|---|---|---|
| Index build time | Slower | Faster |
| Index size | Larger | Smaller |
| Query speed | Faster | Slower |
| Update cost | Higher | Lower |
| Use when | Read-heavy, search-heavy | Write-heavy, approximate queries |

```sql
-- GiST alternative for full-text:
CREATE INDEX idx_articles_fts_gist ON articles USING GIST (ts_vector);

-- GiST also supports nearest-neighbor (similarity) search:
-- Useful for fuzzy matching (requires pg_trgm extension)
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_articles_title_trgm ON articles USING GIN (title gin_trgm_ops);
-- or
CREATE INDEX idx_articles_title_trgm ON articles USING GIST (title gist_trgm_ops);

-- Trigram similarity search (finds close matches even with typos):
SELECT title FROM articles
WHERE title % 'postresql'  -- note the typo
ORDER BY similarity(title, 'postresql') DESC
LIMIT 10;
```

### 7.6 fastupdate Parameter

GIN index updates are expensive — each write must update the postings list for every element. To mitigate this, PostgreSQL has a `fastupdate` mechanism: instead of updating postings lists immediately, new entries are collected in a "pending list" on a separate page and flushed in bulk during vacuuming or when the pending list gets too large.

```sql
CREATE INDEX idx_events_metadata ON events USING GIN (metadata)
  WITH (fastupdate = true);  -- default

-- When fastupdate causes problems:
-- 1. The pending list flush happens at unpredictable times, causing query latency spikes
-- 2. Queries must check both the main GIN tree AND the pending list — slower reads

-- When to disable fastupdate:
-- - You need consistent query latency (no periodic slowdowns)
-- - Your write rate is moderate and you can absorb per-write update cost

CREATE INDEX idx_events_metadata_nofastupdate ON events USING GIN (metadata)
  WITH (fastupdate = false);
```

For production systems with latency SLAs, disable fastupdate or manage the pending list manually:

```sql
-- Check pending list size:
SELECT pg_size_pretty(pg_relation_size('idx_events_metadata'));

-- Manually flush the pending list:
SELECT gin_clean_pending_list('idx_events_metadata'::regclass);
```

---

## 8. BRIN Index — Massive Sequential Tables

### 8.1 How BRIN Works

BRIN (Block Range INdex) does not index individual rows. Instead, it divides the table's heap pages into ranges (groups of pages) and stores the minimum and maximum value of the indexed column within each range.

```
Heap pages:   [0-127]     [128-255]    [256-383]    [384-511]
Min/Max:     min=2024-01  min=2024-04  min=2024-07  min=2024-10
              max=2024-03  max=2024-06  max=2024-09  max=2024-12
BRIN entry:  one per range of 128 pages (pages_per_range = 128 default)
```

A query for `WHERE created_at = '2024-08-15'` consults the BRIN index, eliminates ranges where the min/max range doesn't include August 15, and performs a sequential scan only on the surviving ranges.

**The critical requirement:** BRIN only works well when the data is physically ordered on the indexed column. If rows are inserted in timestamp order, pages 0-127 contain January data, pages 128-255 contain February data, and so on. If rows are inserted randomly (or updated frequently), BRIN becomes useless because every page range overlaps every other in its min/max values.

### 8.2 When to Use It

```sql
-- Perfect use case: time-series event log with sequential inserts
CREATE TABLE iot_readings (
  id          BIGSERIAL PRIMARY KEY,
  sensor_id   INTEGER NOT NULL,
  reading_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  value       NUMERIC
);

-- New rows are always appended with increasing reading_at
CREATE INDEX idx_iot_readings_brin ON iot_readings USING BRIN (reading_at)
  WITH (pages_per_range = 128);

-- Query: last week's readings
SELECT * FROM iot_readings
WHERE reading_at >= NOW() - INTERVAL '7 days';
-- BRIN eliminates all page ranges that end before last week
-- Only the last few hundred pages need to be scanned
```

Other good candidates:
- Audit log tables (insert-only, timestamp ordered)
- Append-only ledger tables
- Sequential serial IDs where rows are never reordered
- Partitioned tables where each partition contains a known date range

**When BRIN is wrong:**
- Tables with frequent random updates that move rows logically forward/backward in time
- Any table where insert order does not correlate with column value order
- Small tables (B-Tree will be faster)

```sql
-- Check physical correlation before choosing BRIN:
SELECT tablename, attname, correlation
FROM pg_stats
WHERE tablename = 'iot_readings' AND attname = 'reading_at';
-- correlation close to 1.0: BRIN is appropriate
-- correlation near 0: do NOT use BRIN for this column
```

### 8.3 Size Comparison

```sql
-- Create both types and compare sizes
CREATE INDEX idx_events_created_btree ON events USING BTREE (created_at);
CREATE INDEX idx_events_created_brin  ON events USING BRIN  (created_at);

SELECT
  indexname,
  pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes
WHERE tablename = 'events'
  AND indexname IN ('idx_events_created_btree', 'idx_events_created_brin');
```

Typical output for a 100M row table:

```
indexname                  | size
---------------------------+---------
idx_events_created_btree   | 2142 MB
idx_events_created_brin    | 48 kB
```

The BRIN index is roughly 44,000x smaller. It fits in one or two pages. For time-series tables that never need indexed access by a non-sequential column, BRIN gives you range scan acceleration at essentially zero storage cost.

### 8.4 pages_per_range

```sql
-- Smaller pages_per_range = more granular ranges = more precise filtering
-- but larger index (still tiny compared to B-Tree)
CREATE INDEX idx_events_brin_fine ON events USING BRIN (created_at)
  WITH (pages_per_range = 32);  -- 4x more granular than default 128

-- Larger pages_per_range = coarser filtering = scans more pages
-- but tiny index size
CREATE INDEX idx_events_brin_coarse ON events USING BRIN (created_at)
  WITH (pages_per_range = 512);

-- Rule: smaller pages_per_range when you need precise filtering on narrow ranges
-- Larger pages_per_range when you query large time windows anyway
```

After large bulk inserts, update the BRIN index manually (it's not automatically updated as aggressively as B-Tree):

```sql
SELECT brin_summarize_new_values('idx_events_brin_fine');
-- Summarizes any new page ranges that were added since last update
```

---

## 9. GiST Index — Geometric and Range Types

### 9.1 Use Cases

GiST (Generalized Search Tree) is a framework for building custom index types. PostgreSQL ships with GiST support for:

- **PostGIS geometry**: spatial queries, nearest-neighbor
- **Range types**: daterange, tsrange, int4range, numrange
- **Geometric types**: point, circle, box, polygon
- **Full-text**: tsvector (but GIN is usually preferred)
- **pg_trgm**: trigram similarity (GIST or GIN)

```sql
-- PostGIS spatial index
CREATE EXTENSION postgis;
CREATE TABLE locations (
  id       BIGSERIAL PRIMARY KEY,
  name     TEXT,
  geom     GEOMETRY(Point, 4326)
);
CREATE INDEX idx_locations_geom ON locations USING GIST (geom);

-- Find nearest restaurants to a point (KNN query):
SELECT name, geom <-> 'SRID=4326;POINT(-73.986 40.748)'::geometry AS distance
FROM locations
ORDER BY geom <-> 'SRID=4326;POINT(-73.986 40.748)'::geometry
LIMIT 10;
-- GiST enables this nearest-neighbor scan efficiently (no full table scan)
```

### 9.2 Range Type Queries

```sql
-- Hotel booking overlap detection
CREATE TABLE reservations (
  id          BIGSERIAL PRIMARY KEY,
  room_id     INTEGER NOT NULL,
  guest_id    INTEGER NOT NULL,
  period      DATERANGE NOT NULL,
  EXCLUDE USING GIST (room_id WITH =, period WITH &&)
  -- Exclusion constraint: no two rows can have the same room_id AND overlapping periods
);

CREATE INDEX idx_reservations_period ON reservations USING GIST (period);

-- Find all reservations overlapping a given period:
SELECT * FROM reservations
WHERE period && '[2024-12-20, 2024-12-27)'::daterange;

-- Find all reservations containing a specific date:
SELECT * FROM reservations
WHERE period @> '2024-12-25'::date;

-- Find all reservations contained within a range:
SELECT * FROM reservations
WHERE period <@ '[2024-12-01, 2024-12-31)'::daterange;
```

GiST on range types supports: `&&` (overlap), `@>` (contains), `<@` (is contained by), `=` (equals), `<<` (strictly left of), `>>` (strictly right of), `-|-` (adjacent to).

---

## 10. Hash Index

### 10.1 When to Choose Hash over B-Tree

A Hash index stores a hash of each key value and a pointer to the heap row. It supports only `=` (equality). It cannot support range queries, ordering, or null checks.

```sql
CREATE INDEX idx_sessions_token ON sessions USING HASH (session_token);

-- Uses the hash index:
SELECT * FROM sessions WHERE session_token = 'abc123xyz';

-- Does NOT use the hash index:
SELECT * FROM sessions WHERE session_token LIKE 'abc%';
SELECT * FROM sessions WHERE session_token IS NULL;
SELECT * FROM sessions ORDER BY session_token;
```

**Before PostgreSQL 10:** Hash indexes were not WAL-logged, meaning they were not crash-safe and had to be rebuilt after a crash. Avoid them in PostgreSQL < 10.

**PostgreSQL 10+:** Hash indexes are WAL-logged and crash-safe.

**When Hash beats B-Tree:**

1. **Very long key values**: B-Tree stores the actual key value in every index entry. For a 200-byte string key, each B-Tree leaf entry is 200+ bytes. A hash entry is always 4 bytes (the hash). For equality-only lookups on wide keys, Hash indexes are dramatically smaller.

2. **Pure equality workloads with no ordering needs**: The hash lookup (one page read to find the hash bucket) can be marginally faster than a B-Tree traversal for very large trees. In practice, for most workloads, the difference is negligible.

3. **When you don't need the index for sorting**: B-Tree's ability to provide sorted output is wasted if you never use it. A Hash index is more space-efficient for pure equality use.

> **Practical advice**: Use B-Tree unless you've measured that Hash provides a meaningful advantage for your specific workload. The additional versatility of B-Tree (range queries, sorting, partial matching) almost always justifies using it. The main valid use case for Hash today is equality-only lookups on very wide (>50 byte) key columns where index size is a concern.

---

## 11. Index Bloat — The Silent Killer

### 11.1 Why Indexes Bloat

PostgreSQL uses MVCC for concurrency. When a row is updated or deleted, the old version is not immediately removed — it's left in place (a "dead tuple") until VACUUM reclaims it. Dead tuples in the heap are eventually reclaimed by VACUUM, but dead index entries are a different story.

An index entry points to a specific heap TID. When that heap row is deleted or updated (creating a new version), the old index entry becomes dead — it points to a dead heap tuple. VACUUM cleans heap dead tuples and marks the corresponding index entries as dead. However, the index page space is not immediately compacted — the dead entries remain, leaving gaps.

Over time, with heavy UPDATE/DELETE workloads:
1. Index pages accumulate dead entries (gaps)
2. New insertions fill some gaps, but many pages end up partially full
3. The index grows larger than necessary
4. More pages must be read to satisfy index scans
5. The index consumes more memory in shared_buffers

**The worst case:** A table where most rows are updated frequently (e.g., a status field transitioning through many states). The index can become 3-10x larger than it should be.

### 11.2 Detecting Bloat

```sql
-- Install pgstattuple for detailed index analysis
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- Check a specific index
SELECT * FROM pgstatindex('idx_orders_customer_id');
```

```
version            | 2
tree_level         | 3
index_size         | 224395264
root_block_no      | 3
internal_pages     | 1091
leaf_pages         | 26308
empty_pages        | 0
deleted_pages      | 124        -- pages with only dead tuples
avg_leaf_density   | 57.16      -- leaf pages are only 57% full (was 90% originally)
leaf_fragmentation | 18.22      -- fragmentation percentage
```

An `avg_leaf_density` below 70% and significant `deleted_pages` indicate serious bloat.

```sql
-- Quick bloat estimate without pgstattuple (uses statistics):
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### 11.3 REINDEX vs REINDEX CONCURRENTLY

**REINDEX** rebuilds the index from scratch, starting with the current live heap tuples. Dead entries are not copied. The result is a compact, fully-packed index.

```sql
-- Locks the table for the duration — not safe for production
REINDEX INDEX idx_orders_customer_id;
REINDEX TABLE orders;  -- reindexes all indexes on the table
```

**REINDEX CONCURRENTLY** (PostgreSQL 12+) rebuilds the index without locking:

```sql
-- Builds a new index in the background while the old one remains active
-- Swaps them atomically when complete
-- Only requires brief locks at the start and end
REINDEX INDEX CONCURRENTLY idx_orders_customer_id;
REINDEX TABLE CONCURRENTLY orders;
```

REINDEX CONCURRENTLY takes 2-3x longer than regular REINDEX and uses more disk space during the rebuild (old + new index simultaneously), but it's the only safe option for production tables.

### 11.4 fillfactor

`fillfactor` tells PostgreSQL what percentage of each page to fill during initial builds and page splits. The default is 90 for B-Tree indexes (leaves some space for future inserts without immediate splits).

```sql
-- For tables with heavy UPDATE workloads on indexed columns:
-- Lower fillfactor leaves room for HOT updates (heap-only tuples)
-- This reduces index bloat from updates
CREATE INDEX idx_orders_status ON orders (status)
  WITH (fillfactor = 70);

-- For insert-only tables with sequential keys:
-- Higher fillfactor wastes less space
CREATE INDEX idx_events_id ON events (id)
  WITH (fillfactor = 100);
```

HOT (Heap-Only Tuple) updates: If an updated row fits on the same heap page as the original, and the update doesn't change any indexed columns, PostgreSQL creates a HOT update — no index entry needs to change. Lower `fillfactor` on the heap table (not the index) enables more HOT updates by keeping free space on heap pages.

```sql
-- Set fillfactor on the table itself to enable more HOT updates:
ALTER TABLE orders SET (fillfactor = 70);
-- Now 30% of each heap page is kept free for in-place HOT updates
```

---

## 12. Multi-Column Index Strategy

### 12.1 The Core Rule: Equality First, Range Last

The B-Tree can efficiently seek to a point in the key space and scan linearly from there. Equality conditions let the tree seek to an exact prefix. A range condition starts the linear scan. Any columns after a range condition in the index cannot be used for filtering — the index is already scanning linearly through a range and can't seek within it.

```sql
-- Index: (status, created_at, amount)
-- Query: WHERE status = 'active' AND created_at BETWEEN '2024-01-01' AND '2024-12-31'

-- Tree traversal:
-- 1. Seek to (status='active', created_at='2024-01-01')
-- 2. Scan linearly to (status='active', created_at='2024-12-31')
-- 3. 'amount' is IRRELEVANT for index filtering — once we're scanning the range,
--    we can't skip within it based on amount

-- If we also filter by amount > 1000:
-- The index returns all rows in the date range for 'active' status
-- Then PostgreSQL filters amount > 1000 as a heap-side filter (not index cond)
```

**Correct order:**
```sql
-- For: WHERE a = ? AND b = ? AND c BETWEEN ? AND ?
CREATE INDEX idx ON t (a, b, c);  -- equality columns first, range last

-- For: WHERE a = ? AND b BETWEEN ? AND ? AND c = ?
CREATE INDEX idx ON t (a, b, c);
-- But: c cannot help with filtering after b's range
-- Better: accept that c is a post-index filter
-- OR: if c is very selective, consider (a, c, b) and check if that serves other queries
```

### 12.2 Composite vs Separate Indexes

Two separate indexes `(a)` and `(b)` can be combined via Bitmap AND/OR. One composite index `(a, b)` is always preferred when:
- Queries almost always filter on both columns together
- The combined selectivity of (a, b) is much higher than either alone
- You care about sort order (composite index can serve ORDER BY)

Two separate indexes are preferable when:
- Queries sometimes filter on only `a`, sometimes on only `b`, sometimes on both
- The planner can BitmapAnd them when needed
- Maintaining two narrow indexes is cheaper than one wide composite

```sql
-- Scenario: orders table queried as:
--   WHERE customer_id = ?          (frequent)
--   WHERE product_id = ?           (frequent)
--   WHERE customer_id = ? AND product_id = ?  (occasional)

-- Option A: Two separate indexes
CREATE INDEX idx_orders_customer ON orders (customer_id);
CREATE INDEX idx_orders_product  ON orders (product_id);
-- Planner uses each alone for single-column queries
-- Planner BitmapAnds them for dual-column queries

-- Option B: Composite index
CREATE INDEX idx_orders_cust_prod ON orders (customer_id, product_id);
-- Great for: WHERE customer_id = ? AND product_id = ?
-- Great for: WHERE customer_id = ?  (leading column)
-- Useless for: WHERE product_id = ?  (not leading column)

-- Decision: if product-only queries are frequent, keep two separate indexes
-- If they're rare, the composite plus a separate product index is optimal:
CREATE INDEX idx_orders_cust_prod ON orders (customer_id, product_id);
CREATE INDEX idx_orders_product   ON orders (product_id);
```

### 12.3 Five Query Patterns and Their Index Designs

**Pattern 1: Point lookup then sort**

```sql
-- Query: SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 20;
-- Wrong index:
CREATE INDEX wrong ON orders (customer_id);
-- Forces a sort after the index lookup

-- Right index:
CREATE INDEX right ON orders (customer_id, created_at DESC);
-- Index provides rows in ORDER BY order — no sort step needed
```

**Pattern 2: Equality + range + projection**

```sql
-- Query: SELECT id, status FROM orders
--        WHERE customer_id = 42 AND created_at >= '2024-01-01';
-- Right index (covering):
CREATE INDEX right ON orders (customer_id, created_at) INCLUDE (id, status);
-- Equality first, range second, include projected columns for index-only scan
```

**Pattern 3: OR conditions across two columns**

```sql
-- Query: SELECT * FROM events WHERE user_id = 42 OR session_id = 'xyz';
-- Right approach: two separate indexes + BitmapOr
CREATE INDEX idx_events_user    ON events (user_id);
CREATE INDEX idx_events_session ON events (session_id);
-- Planner BitmapOrs both
-- A composite (user_id, session_id) would NOT help this query
```

**Pattern 4: Multi-value filter with counts**

```sql
-- Query: SELECT status, COUNT(*) FROM orders
--        WHERE customer_id = 42 GROUP BY status;
-- Right index:
CREATE INDEX right ON orders (customer_id, status);
-- Or even better for index-only scan:
CREATE INDEX right ON orders (customer_id) INCLUDE (status);
-- With INCLUDE, no heap access needed; both customer_id and status are in the index
```

**Pattern 5: Conditional aggregation across a large range**

```sql
-- Query: SELECT DATE_TRUNC('day', created_at), SUM(amount)
--        FROM orders
--        WHERE created_at >= '2024-01-01' AND status = 'completed'
--        GROUP BY 1 ORDER BY 1;
-- Here, status is equality and created_at is range:
CREATE INDEX right ON orders (status, created_at) INCLUDE (amount);
-- Equality (status) first, range (created_at) second, amount included for index-only scan
```

---

## 13. When NOT to Index

### 13.1 The Real Cost of Every Index

Every index you create imposes ongoing costs:

1. **INSERT cost**: Every new row requires an entry in each index. For 10 indexes on a table, one INSERT updates 11 structures (heap + 10 indexes). The index update requires a WAL write and potentially a page split.

2. **UPDATE cost**: If the update touches an indexed column, the old index entry must be removed (marked dead) and a new entry inserted. Non-HOT updates touch the index. Even with HOT updates, indexes with the changed column must be updated.

3. **DELETE cost**: The index entry is marked dead. VACUUM must later clean it.

4. **Storage cost**: Index storage competes with your table and WAL for disk I/O bandwidth and OS page cache.

5. **VACUUM cost**: More indexes means VACUUM must clean more index pages. Autovacuum rounds become slower, potentially falling behind on a write-heavy table.

6. **Query planner cost**: The planner must consider all indexes when building a plan. More indexes = longer planning time (usually negligible, but measurable in extreme cases with 50+ indexes).

7. **Lock cost**: DDL operations on indexes (CREATE INDEX, DROP INDEX, REINDEX) require various lock levels that can block concurrent queries.

### 13.2 Conditions That Rule Out an Index

**Low cardinality columns:**

```sql
-- Table: 10 million orders, status ∈ {pending, active, completed, cancelled}
-- completed = 95% of all rows
CREATE INDEX idx_orders_status ON orders (status); -- almost always wrong

-- Query: WHERE status = 'completed'
-- Matches 9.5 million rows — seq scan will win
-- Query: WHERE status = 'pending'
-- If pending is 0.1%, this might help — but use a partial index instead:
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';
```

**Small tables:**

```sql
-- Tables with fewer than ~1,000 rows rarely benefit from indexes
-- PostgreSQL may seq scan even with an index because the whole table fits in 1-2 pages
-- Adding indexes on a small lookup table (e.g., a currencies or countries table)
-- wastes resources and adds write overhead
```

**Write-heavy tables with low read selectivity:**

```sql
-- Log table: 50,000 inserts/second, rarely queried
-- Adding an index on a non-PK column slows every insert
-- If you rarely query by that column, or queries can tolerate seq scan latency, skip it
-- Use BRIN instead if you need basic range filtering
```

**Columns never used in WHERE, JOIN ON, or ORDER BY:**

```sql
-- If a column only appears in SELECT output, never in WHERE/JOIN/ORDER BY,
-- it should be in INCLUDE (if needed for index-only scans) not as a key column
-- A standalone index on a SELECT-only column provides zero query benefit
```

**Columns used with functions that break index use:**

```sql
-- Indexing a column that's always wrapped in functions in queries:
CREATE INDEX idx_orders_year ON orders (created_at); -- indexed

-- But every query uses:
SELECT * FROM orders WHERE EXTRACT(YEAR FROM created_at) = 2024;
-- This does NOT use idx_orders_year
-- The index is wasted — either create an expression index on the function,
-- or rewrite queries to use range predicates on created_at
```

> **The decision framework**: Before creating an index, ask:
> 1. Which specific queries will use this index?
> 2. What is the selectivity of those queries? (< 10% of rows to be useful)
> 3. How frequently are those queries run vs how frequently is the table written to?
> 4. Is there an existing index that could serve this query with a small adjustment?
> 5. Have you run EXPLAIN ANALYZE to confirm the planner actually uses the index?

---

## 14. Index Maintenance Queries

### 14.1 Find Unused Indexes

PostgreSQL tracks index usage since the last statistics reset. An index with `idx_scan = 0` over a long period is a candidate for removal.

```sql
SELECT
  s.schemaname,
  s.relname                                               AS table_name,
  s.indexrelname                                          AS index_name,
  s.idx_scan                                              AS times_used,
  pg_size_pretty(pg_relation_size(s.indexrelid))          AS index_size,
  pg_size_pretty(pg_relation_size(s.relid))               AS table_size,
  CASE WHEN i.indisunique THEN 'UNIQUE' ELSE '' END       AS is_unique,
  CASE WHEN i.indisprimary THEN 'PRIMARY' ELSE '' END     AS is_primary
FROM pg_stat_user_indexes s
JOIN pg_index i ON s.indexrelid = i.indexrelid
WHERE s.schemaname = 'public'
  AND s.idx_scan = 0             -- never used since last reset
  AND NOT i.indisprimary         -- exclude primary keys
  AND NOT i.indisunique          -- exclude unique constraints (they enforce correctness)
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

> **Important**: `pg_stat_user_indexes` resets when the database (or `pg_stat_reset()`) is called. A newly deployed service may show 0 scans on all indexes. Only trust these statistics after the database has been running under normal production load for at least 1-2 weeks.

```sql
-- Check when statistics were last reset:
SELECT stats_reset FROM pg_stat_bgwriter;
```

### 14.2 Find Missing Indexes

Tables with high sequential scan counts and large row estimates are candidates for new indexes.

```sql
SELECT
  schemaname,
  relname                           AS table_name,
  seq_scan,
  seq_tup_read,
  idx_scan,
  idx_tup_fetch,
  n_live_tup,
  ROUND(seq_scan::NUMERIC / NULLIF(idx_scan + seq_scan, 0) * 100, 1)
                                    AS seq_scan_pct,
  pg_size_pretty(pg_relation_size(relid)) AS table_size
FROM pg_stat_user_tables
WHERE schemaname = 'public'
  AND seq_scan > 100               -- at least 100 seq scans
  AND n_live_tup > 10000           -- non-trivial table size
ORDER BY seq_scan DESC
LIMIT 20;
```

High `seq_scan_pct` with large `n_live_tup` means queries are full-scanning a big table frequently. Check the query workload for that table and identify which columns appear in WHERE clauses without index coverage.

### 14.3 Find Duplicate Indexes

Duplicate indexes waste storage and slow down writes with no benefit.

```sql
SELECT
  a.indrelid::regclass            AS table_name,
  a.indexrelid::regclass          AS index_a,
  b.indexrelid::regclass          AS index_b,
  pg_size_pretty(pg_relation_size(a.indexrelid)) AS size_a,
  pg_size_pretty(pg_relation_size(b.indexrelid)) AS size_b
FROM pg_index a
JOIN pg_index b ON (
  a.indrelid = b.indrelid
  AND a.indexrelid < b.indexrelid
  AND a.indkey::int[] @> b.indkey::int[]  -- a's columns contain all of b's columns
  AND a.indkey::int[] <@ b.indkey::int[]  -- b's columns contain all of a's columns
  -- Both conditions together = same column set (but possibly different order)
)
ORDER BY pg_relation_size(a.indexrelid) DESC;
```

Also check for prefix duplicates (one index is a prefix of another):

```sql
-- Find indexes where one is a strict prefix of another:
SELECT
  a.indrelid::regclass            AS table_name,
  a.indexrelid::regclass          AS longer_index,
  b.indexrelid::regclass          AS shorter_prefix_index,
  pg_size_pretty(pg_relation_size(b.indexrelid)) AS wasted_size
FROM pg_index a
JOIN pg_index b ON (
  a.indrelid = b.indrelid
  AND a.indexrelid != b.indexrelid
  -- b's columns are a prefix of a's columns
  AND (SELECT array_agg(x ORDER BY n)
       FROM unnest(b.indkey::int[]) WITH ORDINALITY AS t(x,n)
       WHERE x != 0)
    =
      (SELECT array_agg(x ORDER BY n)
       FROM (SELECT x, n FROM unnest(a.indkey::int[]) WITH ORDINALITY AS t(x,n)
             WHERE x != 0 LIMIT array_length(b.indkey, 1)) sub)
)
WHERE array_length(a.indkey, 1) > array_length(b.indkey, 1);
```

### 14.4 Find Index Bloat

```sql
-- Requires pgstattuple extension
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- Check bloat on all indexes for a table:
SELECT
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid))    AS index_size,
  (pgstatindex(indexrelid)).avg_leaf_density       AS leaf_density_pct,
  (pgstatindex(indexrelid)).deleted_pages          AS deleted_pages,
  (pgstatindex(indexrelid)).leaf_fragmentation     AS fragmentation_pct
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
  AND relname = 'orders'
ORDER BY (pgstatindex(indexrelid)).leaf_fragmentation DESC;
```

Rule of thumb:
- `avg_leaf_density` < 70%: significant bloat, consider REINDEX CONCURRENTLY
- `deleted_pages` > 5% of total pages: autovacuum may not be keeping up
- `leaf_fragmentation` > 20%: range scan performance is degraded

### 14.5 Index Size Comparison

```sql
-- All indexes on a table with their sizes and usage:
SELECT
  i.relname                                      AS index_name,
  ix.indisunique                                 AS is_unique,
  ix.indisprimary                                AS is_primary,
  array_agg(a.attname ORDER BY array_position(ix.indkey, a.attnum))
                                                 AS indexed_columns,
  pg_size_pretty(pg_relation_size(i.oid))        AS index_size,
  s.idx_scan                                     AS scans,
  s.idx_tup_read                                 AS tuples_read
FROM pg_index ix
JOIN pg_class i  ON i.oid = ix.indexrelid
JOIN pg_class t  ON t.oid = ix.indrelid
JOIN pg_attribute a ON a.attrelid = t.oid AND a.attnum = ANY(ix.indkey)
JOIN pg_stat_user_indexes s ON s.indexrelid = ix.indexrelid
WHERE t.relname = 'orders'
GROUP BY i.relname, ix.indisunique, ix.indisprimary, i.oid, s.idx_scan, s.idx_tup_read
ORDER BY pg_relation_size(i.oid) DESC;
```

### 14.6 Full Index Health Check

Run this suite regularly in production (weekly, or after schema changes):

```sql
-- 1. Total index overhead vs table size
SELECT
  relname                                        AS table_name,
  pg_size_pretty(pg_relation_size(oid))          AS table_size,
  pg_size_pretty(pg_indexes_size(oid))           AS indexes_size,
  ROUND(pg_indexes_size(oid)::NUMERIC /
        NULLIF(pg_relation_size(oid), 0) * 100, 1)
                                                 AS index_overhead_pct
FROM pg_class
WHERE relkind = 'r'
  AND relnamespace = 'public'::regnamespace
ORDER BY pg_indexes_size(oid) DESC
LIMIT 20;

-- 2. Unused indexes (candidates for removal)
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS wasted
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
  AND idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- 3. High seq scan tables (potential missing indexes)
SELECT relname, seq_scan, n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan > 1000 AND n_live_tup > 50000
ORDER BY seq_scan DESC;

-- 4. Index efficiency (rows fetched vs rows read)
SELECT
  indexrelname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch,
  ROUND(idx_tup_fetch::NUMERIC / NULLIF(idx_tup_read, 0) * 100, 1)
                                                 AS fetch_ratio_pct
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
  AND idx_scan > 0
ORDER BY idx_scan DESC;
-- Low fetch_ratio_pct means the index is reading many entries but fetching few rows
-- This can indicate a visibility map problem (lots of dead tuples)
-- or an index being used for filtering that returns many candidates
```

---

## 15. Real-World Indexing Decisions

### 15.1 E-Commerce Product Search

**The table:**

```sql
CREATE TABLE products (
  id           BIGSERIAL PRIMARY KEY,
  category_id  INTEGER NOT NULL,
  brand_id     INTEGER NOT NULL,
  name         TEXT NOT NULL,
  price        NUMERIC(10,2) NOT NULL,
  in_stock     BOOLEAN NOT NULL DEFAULT TRUE,
  rating       NUMERIC(3,2),
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  metadata     JSONB
);
```

**Common queries:**

```sql
-- Q1: Category browse with price filter (most common)
SELECT * FROM products
WHERE category_id = 12 AND in_stock = TRUE AND price BETWEEN 50 AND 200
ORDER BY rating DESC
LIMIT 20;

-- Q2: Brand + category
SELECT * FROM products
WHERE brand_id = 5 AND category_id = 12 AND in_stock = TRUE
ORDER BY price ASC
LIMIT 20;

-- Q3: Full-text search
SELECT * FROM products WHERE name ILIKE '%wireless headphones%';

-- Q4: JSONB attribute filter (e.g., color, size)
SELECT * FROM products WHERE metadata @> '{"color": "red"}';
```

**Wrong index:**

```sql
-- Indexing on low-cardinality in_stock (2 values) is wasteful
CREATE INDEX idx_products_in_stock ON products (in_stock);
-- Useless: 50% or more of rows are in_stock, planner will prefer seq scan

-- Separate indexes on category_id, price, rating
CREATE INDEX idx_products_category ON products (category_id);
CREATE INDEX idx_products_price ON products (price);
CREATE INDEX idx_products_rating ON products (rating);
-- These may combine via BitmapAnd but the planner may not always choose optimally
-- And none of them eliminate the sort step for rating DESC
```

**Right indexes:**

```sql
-- Q1 and Q2: composite partial index (in_stock = TRUE is equality, low cardinality,
-- but using it as a partial index predicate removes out-of-stock rows entirely)
CREATE INDEX idx_products_category_price_rating ON products (category_id, price, rating DESC)
  WHERE in_stock = TRUE;
-- Seeks to category_id, filters price range, index already sorted by rating DESC

-- For Q2: brand + category
CREATE INDEX idx_products_brand_category ON products (brand_id, category_id)
  INCLUDE (price, rating)
  WHERE in_stock = TRUE;

-- Q3: expression index for case-insensitive search, or better — full-text GIN
ALTER TABLE products ADD COLUMN name_tsv TSVECTOR
  GENERATED ALWAYS AS (to_tsvector('english', name)) STORED;
CREATE INDEX idx_products_name_fts ON products USING GIN (name_tsv);
-- Query rewritten as:
SELECT * FROM products WHERE name_tsv @@ plainto_tsquery('english', 'wireless headphones');

-- Q4: GIN on JSONB
CREATE INDEX idx_products_metadata ON products USING GIN (metadata jsonb_path_ops);
-- jsonb_path_ops: only supports @>, but 30-50% smaller
```

**Why**: The partial index `WHERE in_stock = TRUE` immediately halves or more the index size (depending on your out-of-stock ratio). Category + price + rating DESC satisfies the most common query pattern in one index scan with no sort. The GIN indexes address the search and attribute filter use cases that B-Tree fundamentally cannot handle.

### 15.2 User Authentication

**The table:**

```sql
CREATE TABLE users (
  id             BIGSERIAL PRIMARY KEY,
  email          TEXT NOT NULL,
  password_hash  TEXT NOT NULL,
  role           TEXT NOT NULL DEFAULT 'user',
  deleted_at     TIMESTAMPTZ,
  created_at     TIMESTAMPTZ DEFAULT NOW()
);
```

**Common queries:**

```sql
-- Q1: Login (most critical path — must be fast and deterministic)
SELECT id, password_hash, role FROM users
WHERE LOWER(email) = LOWER('Alice@Example.COM') AND deleted_at IS NULL;

-- Q2: Admin check during requests
SELECT role FROM users WHERE id = $1;
```

**Wrong index:**

```sql
-- Simple B-Tree on email — won't help if users type mixed case
CREATE INDEX idx_users_email ON users (email);
-- Query: WHERE LOWER(email) = 'alice@example.com' — DOES NOT use this index
-- The function call on the column breaks the index
```

**Right indexes:**

```sql
-- Expression index on LOWER(email), partial for non-deleted users:
CREATE UNIQUE INDEX idx_users_email_lower_active
  ON users (LOWER(email))
  WHERE deleted_at IS NULL;

-- Now the query MUST use the exact expression:
SELECT id, password_hash, role FROM users
WHERE LOWER(email) = LOWER('Alice@Example.COM') AND deleted_at IS NULL;

-- And add INCLUDE so the auth lookup is an index-only scan:
CREATE UNIQUE INDEX idx_users_email_auth
  ON users (LOWER(email))
  INCLUDE (id, password_hash, role)
  WHERE deleted_at IS NULL;

-- Q2: Primary key lookup — already covered by the implicit PK index
```

**Verify it works:**

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT id, password_hash, role FROM users
WHERE LOWER(email) = 'alice@example.com' AND deleted_at IS NULL;
```

```
Index Only Scan using idx_users_email_auth on users
  Index Cond: (lower(email) = 'alice@example.com'::text)
  Heap Fetches: 0
  Buffers: shared hit=3
```

Three buffer hits (root + branch + leaf), zero heap fetches. Authentication lookup costs three page reads regardless of table size.

### 15.3 Order History

**The table:**

```sql
CREATE TABLE orders (
  id          BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  status      TEXT NOT NULL,  -- pending, processing, shipped, delivered, cancelled
  total       NUMERIC(12,2) NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ
);
```

**Common queries:**

```sql
-- Q1: Customer order history, recent first
SELECT id, status, total, created_at FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 20;

-- Q2: Customer order history filtered by status
SELECT id, status, total, created_at FROM orders
WHERE customer_id = 42 AND status = 'delivered'
ORDER BY created_at DESC
LIMIT 20;

-- Q3: Date range query for analytics
SELECT customer_id, SUM(total) FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
  AND status = 'delivered'
GROUP BY customer_id;
```

**Wrong indexes:**

```sql
-- Wrong: putting status before customer_id
CREATE INDEX wrong1 ON orders (status, customer_id, created_at);
-- Q1 and Q2 start with customer_id — this index can't help without a status filter

-- Wrong: putting created_at before customer_id (for date-range queries)
CREATE INDEX wrong2 ON orders (created_at, customer_id, status);
-- Q1 and Q2 must specify created_at range first — useless for customer lookups
```

**Right indexes:**

```sql
-- Q1 and Q2: customer_id (equality) first, then created_at (range/sort)
-- status after created_at can't be used for range, but covers Q2's sort order
CREATE INDEX idx_orders_customer_date ON orders (customer_id, created_at DESC)
  INCLUDE (id, status, total);
-- Q1: seeks to customer_id=42, scans created_at DESC, index-only scan
-- Q2: seeks to customer_id=42, scans created_at DESC, filters status as heap filter

-- For Q2 where status filter is highly selective:
CREATE INDEX idx_orders_customer_status_date ON orders (customer_id, status, created_at DESC)
  INCLUDE (id, total);
-- Seeks to (customer_id=42, status='delivered'), then scans created_at DESC

-- Q3: Analytics — equality on status (frequent, but low-cardinality), range on date
-- status first only if the delivered fraction is small enough
CREATE INDEX idx_orders_status_date ON orders (status, created_at)
  INCLUDE (customer_id, total)
  WHERE status = 'delivered';  -- partial: only delivered orders
-- Partial index for analytics on delivered orders only
```

**Decision**: Use `idx_orders_customer_date` for the customer-facing API (most common). Add `idx_orders_customer_status_date` if profiling shows Q2 is slow and the status filter is highly selective. Use the partial index on the analytics query if delivered orders represent a small fraction of total orders and analytics is a significant workload.

### 15.4 Job Queue with SKIP LOCKED

**The table:**

```sql
CREATE TABLE jobs (
  id           BIGSERIAL PRIMARY KEY,
  queue_name   TEXT NOT NULL DEFAULT 'default',
  status       TEXT NOT NULL DEFAULT 'pending',  -- pending, processing, done, failed
  priority     INTEGER NOT NULL DEFAULT 0,
  payload      JSONB NOT NULL,
  scheduled_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  attempts     INTEGER NOT NULL DEFAULT 0,
  max_attempts INTEGER NOT NULL DEFAULT 3
);
```

**Common queries:**

```sql
-- Worker polling: fetch next available job atomically
SELECT id, payload FROM jobs
WHERE status = 'pending'
  AND queue_name = 'email'
  AND scheduled_at <= NOW()
  AND attempts < max_attempts
ORDER BY priority DESC, scheduled_at ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- Retry failed jobs (scheduled externally, but still needs efficient lookup)
SELECT id FROM jobs
WHERE status = 'failed' AND attempts < max_attempts AND queue_name = $1;
```

**Why standard indexing fails:**

A jobs table might have 50 million historical completed and failed jobs, and 500 pending jobs. A B-Tree index on `(status, queue_name, scheduled_at)` has 50 million entries. The worker polls it continuously. The index is huge, the matching rows are rare, and every insert/update to completed jobs still updates this index.

**Right index:**

```sql
-- Partial index: only pending and failed jobs are actionable
CREATE INDEX idx_jobs_worker ON jobs (queue_name, priority DESC, scheduled_at ASC)
  WHERE status IN ('pending', 'failed') AND attempts < max_attempts;

-- Wait — partial index with dynamic conditions (attempts < max_attempts) is tricky
-- The predicate is stored as-is; PostgreSQL checks it using column statistics
-- 'attempts < max_attempts' references TWO columns — PostgreSQL does support this,
-- but only in PostgreSQL 14+ with better cross-column statistics

-- Safer approach: partial index on status only, let max_attempts be a query filter
CREATE INDEX idx_jobs_actionable ON jobs (queue_name, priority DESC, scheduled_at)
  WHERE status = 'pending';

-- Separate index for failed jobs
CREATE INDEX idx_jobs_retryable ON jobs (queue_name, scheduled_at)
  WHERE status = 'failed';

-- Worker query hits a tiny index (hundreds of rows instead of millions)
SELECT id, payload FROM jobs
WHERE status = 'pending'
  AND queue_name = 'email'
  AND scheduled_at <= NOW()
  AND attempts < max_attempts   -- filter after index access
ORDER BY priority DESC, scheduled_at ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

**The SKIP LOCKED interaction**: SKIP LOCKED skips rows that are locked by another transaction. It does not interact with the index structure — it's applied during the heap fetch phase. The index narrows the candidate set; SKIP LOCKED then skips locked rows among the candidates. With a partial index returning only a few hundred pending rows, SKIP LOCKED contention is minimal.

**Critical**: Rebuild or create a new partial index periodically as the definition of "actionable" changes (e.g., if you change status values). The index predicate is fixed at creation time.

### 15.5 Multi-Tenant SaaS

**The table:**

```sql
CREATE TABLE documents (
  id          BIGSERIAL PRIMARY KEY,
  tenant_id   UUID NOT NULL,
  owner_id    BIGINT NOT NULL,
  title       TEXT NOT NULL,
  content     TEXT,
  status      TEXT NOT NULL DEFAULT 'draft',
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ
);
```

**The constraint**: Every single query in a multi-tenant system is scoped by `tenant_id`. No query should ever touch another tenant's data. This isn't a hint — it's a hard requirement.

**Common queries:**

```sql
-- Q1: Tenant's document list
SELECT id, title, status, created_at FROM documents
WHERE tenant_id = $1
ORDER BY created_at DESC
LIMIT 50;

-- Q2: Specific owner's documents within a tenant
SELECT id, title FROM documents
WHERE tenant_id = $1 AND owner_id = $2
ORDER BY updated_at DESC;

-- Q3: Tenant's documents by status
SELECT id, title, updated_at FROM documents
WHERE tenant_id = $1 AND status = 'published'
ORDER BY updated_at DESC
LIMIT 20;

-- Q4: Full-text search within a tenant
SELECT id, title FROM documents
WHERE tenant_id = $1
  AND to_tsvector('english', title || ' ' || content) @@ plainto_tsquery('english', $2);
```

**Wrong indexes:**

```sql
-- Forgetting tenant_id as the leading column
CREATE INDEX wrong1 ON documents (owner_id, created_at);
-- Every query filtering by owner_id alone touches all tenants' data in the index

-- Single-column indexes on everything
CREATE INDEX wrong2 ON documents (status);
CREATE INDEX wrong3 ON documents (owner_id);
CREATE INDEX wrong4 ON documents (created_at);
-- None of these start with tenant_id — all cross tenant boundaries in the index
```

**Right indexes:**

```sql
-- RULE: tenant_id MUST be the leading column on every index
-- This ensures index seeks are always scoped to a single tenant

-- Q1: tenant list view
CREATE INDEX idx_documents_tenant_date ON documents
  (tenant_id, created_at DESC)
  INCLUDE (id, title, status);

-- Q2: owner within tenant
CREATE INDEX idx_documents_tenant_owner ON documents
  (tenant_id, owner_id, updated_at DESC)
  INCLUDE (id, title);

-- Q3: status within tenant
CREATE INDEX idx_documents_tenant_status ON documents
  (tenant_id, status, updated_at DESC)
  INCLUDE (id, title);

-- Q4: Full-text within tenant
-- Option A: Composite GIN (not directly supported — GIN can't be easily composited with tenant_id)
-- Option B: Use row-level security + GIN on ts_vector, accept that GIN doesn't scope by tenant
ALTER TABLE documents ADD COLUMN ts_content TSVECTOR
  GENERATED ALWAYS AS (to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(content, ''))) STORED;
CREATE INDEX idx_documents_fts ON documents USING GIN (ts_content);
-- The WHERE tenant_id = $1 filter is applied post-GIN using a Bitmap Heap Scan
-- The GIN returns candidate rows; tenant_id filter runs on the heap
-- For small-to-medium tenant sizes this is acceptable
-- For large tenants with millions of documents, consider per-tenant partitioning

-- Row-level security to enforce tenant isolation at the database level:
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

**The partitioning option for large tenants:**

If a single tenant has tens of millions of documents, per-tenant table partitioning becomes attractive:

```sql
CREATE TABLE documents (
  id        BIGSERIAL,
  tenant_id UUID NOT NULL,
  ...
) PARTITION BY HASH (tenant_id);

CREATE TABLE documents_p0 PARTITION OF documents FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE documents_p1 PARTITION OF documents FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE documents_p2 PARTITION OF documents FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE documents_p3 PARTITION OF documents FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Indexes on each partition automatically scope to partition's tenant subset
-- Planner uses partition pruning to eliminate partitions for queries with tenant_id = $1
```

With partitioning, a query for a specific tenant touches only one partition (1/4 of all data). Combined with a local index on `(owner_id, created_at)` within each partition, you get fast access without tenant_id needing to be the leading column — partition pruning handles tenant isolation at the partition level.

---

## Final Decision Framework

Before creating any index, run through this checklist:

**1. Identify the exact queries.** Get the SQL. Run `EXPLAIN (ANALYZE, BUFFERS)`. Know the current plan and its cost.

**2. Check if an existing index could serve the query with a small change.** Adding an INCLUDE column or making a partial predicate more specific is cheaper than a new index.

**3. Determine the selectivity.** Does the query return < 10% of rows? If not, index benefit is questionable.

**4. Choose the right index type for the data pattern:**
- Equality/range on scalar values → B-Tree
- JSONB containment, arrays, full-text → GIN
- Sequential large tables → BRIN
- Spatial, ranges, nearest-neighbor → GiST
- Pure equality on wide keys → Hash

**5. Design the column order for composite indexes:**
- Equality columns first (highest cardinality among them first)
- Range/sort columns last
- INCLUDE any projected columns needed for index-only scans

**6. Consider a partial index** if only a subset of rows is ever queried (active records, pending jobs, non-null values).

**7. Measure the size and write impact.** Check index size after creation. Monitor INSERT/UPDATE throughput before and after.

**8. Verify the planner uses it.** EXPLAIN ANALYZE with the real query and real data. If it doesn't use the index, understand why before forcing it.

**9. Monitor usage in production.** Check `pg_stat_user_indexes` weekly. Unused indexes are pure overhead. Remove them.

**10. Watch for bloat.** High-churn tables need periodic REINDEX CONCURRENTLY. Bloated indexes silently degrade performance over months.

---

*Every index is a trade-off between read performance and write overhead. Make the trade consciously, measure the outcome, and revisit your decisions as your data distribution and query patterns change.*
