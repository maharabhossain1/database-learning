# PostgreSQL Transactions & Concurrency Control

**From ACID Internals to Production-Safe Concurrency Patterns**

> The difference between a junior and a senior database engineer is knowing not just what isolation level to use, but why — and what breaks without it.

---

## Table of Contents

1. [ACID — What It Actually Means in PostgreSQL](#1-acid--what-it-actually-means-in-postgresql)
2. [MVCC — How PostgreSQL Handles Concurrency Internally](#2-mvcc--how-postgresql-handles-concurrency-internally)
3. [Isolation Levels — Real Behavior, Real Anomalies](#3-isolation-levels--real-behavior-real-anomalies)
4. [Locking — Row Locks, Table Locks, Advisory Locks](#4-locking--row-locks-table-locks-advisory-locks)
5. [Deadlocks — Detection, Diagnosis, Prevention](#5-deadlocks--detection-diagnosis-prevention)
6. [Real-World Race Conditions and How to Solve Them](#6-real-world-race-conditions-and-how-to-solve-them)
7. [Savepoints — Nested Rollback Control](#7-savepoints--nested-rollback-control)
8. [Transaction Anti-Patterns That Kill Production](#8-transaction-anti-patterns-that-kill-production)
9. [pg_stat_activity and pg_locks — Production Diagnosis](#9-pg_stat_activity-and-pg_locks--production-diagnosis)
10. [Decision Framework — Choosing the Right Strategy](#10-decision-framework--choosing-the-right-strategy)

---

## 1. ACID — What It Actually Means in PostgreSQL

ACID is not a checklist. It is a guarantee PostgreSQL makes, and understanding what happens when each property is *violated* (or not enforced) is what makes the concepts real.

### 1.1 Atomicity — All or Nothing

**Textbook:** A transaction either fully commits or fully rolls back.

**What it actually means:** PostgreSQL writes a WAL (Write-Ahead Log) record for every change. If the process crashes mid-transaction, the WAL is replayed on restart and any uncommitted changes are rolled back. There is no "partial write" visible to any other session.

**What violation looks like:**

Imagine transferring $500 from account A to account B with two separate UPDATE statements — without wrapping them in a transaction:

```sql
-- WRONG: No transaction wrapper
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
-- Server crashes here
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
```

Account A is debited. Account B is never credited. $500 vanishes. This is atomicity violation.

**The correct pattern:**

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;
```

If anything fails between BEGIN and COMMIT, both changes are rolled back atomically.

### 1.2 Consistency — The Database Never Enters an Invalid State

**What it actually means:** Every transaction takes the database from one valid state to another. Constraints (NOT NULL, UNIQUE, CHECK, FOREIGN KEY) are the enforcement mechanism. PostgreSQL checks them at statement time by default, or at COMMIT time for deferrable constraints.

**What violation looks like:**

```sql
-- Without a constraint:
INSERT INTO order_items (order_id, product_id, quantity) VALUES (9999, 1, 5);
-- order_id 9999 does not exist — this leaves an orphaned row
```

**The correct enforcement:**

```sql
ALTER TABLE order_items
  ADD CONSTRAINT fk_order
  FOREIGN KEY (order_id) REFERENCES orders(id)
  ON DELETE CASCADE;
```

**Deferrable constraints** — useful for circular references (user has a default_address, address belongs to user):

```sql
ALTER TABLE users
  ADD CONSTRAINT fk_default_address
  FOREIGN KEY (default_address_id) REFERENCES addresses(id)
  DEFERRABLE INITIALLY DEFERRED;

BEGIN;
  INSERT INTO users (id, name, default_address_id) VALUES (1, 'Alice', 100);
  INSERT INTO addresses (id, user_id, street) VALUES (100, 1, '123 Main St');
COMMIT;
-- FK is only checked at COMMIT, not per-statement
```

### 1.3 Isolation — Transactions Don't Step on Each Other

**What it actually means:** Concurrent transactions behave as if they ran serially. PostgreSQL provides *degrees* of isolation — stronger isolation costs more concurrency. The default (Read Committed) is intentionally weaker for performance.

This is covered in depth in Section 3. The key point here: isolation is a spectrum, not a switch.

### 1.4 Durability — Committed Data Survives Crashes

**What it actually means:** Once PostgreSQL returns "COMMIT", the data is written to the WAL and flushed to disk (fsync). Even if the server crashes the instant after COMMIT returns, the data is recoverable.

**The hidden danger — `synchronous_commit = off`:**

```sql
SHOW synchronous_commit;
-- on (default) — WAL is flushed before COMMIT returns to the client
-- off — COMMIT returns immediately; WAL flush happens asynchronously
```

With `synchronous_commit = off`, you can lose the last ~200ms of commits on a crash. This is a valid trade-off for high-throughput, low-criticality writes (logging, metrics), but never for financial data.

```sql
-- Per-transaction override — disable durability guarantee for this transaction only
SET LOCAL synchronous_commit = off;
INSERT INTO event_log (event_type, payload) VALUES ('page_view', '{}');
COMMIT;
```

> **Decision:** Use `synchronous_commit = off` for append-only, lossy-tolerant data (logs, analytics events). Never use it for anything you can't re-derive or re-process.

---

## 2. MVCC — How PostgreSQL Handles Concurrency Internally

This is the most important section for understanding why PostgreSQL behaves the way it does. MVCC is the reason reads don't block writes and writes don't block reads.

### 2.1 The Core Idea

Instead of locking a row when it's updated, PostgreSQL keeps **multiple versions** of the same row. Each transaction sees the version that was current when it started. Old versions are cleaned up by VACUUM.

### 2.2 xmin and xmax — The Row Visibility System

Every row in every table has two hidden system columns:

- `xmin` — the transaction ID that **inserted** this row version
- `xmax` — the transaction ID that **deleted or updated** this row version (0 if the row is current)

```sql
-- See the hidden columns
SELECT xmin, xmax, id, balance FROM accounts;

--  xmin  | xmax | id | balance
-- -------+------+----+---------
--  1042  |    0 |  1 |  10000
--  1043  | 1045 |  2 |   5000   -- this version was superseded by txn 1045
--  1045  |    0 |  2 |   4500   -- current version of row 2
```

When you UPDATE a row:
1. PostgreSQL marks the old row's `xmax` with your transaction ID.
2. It inserts a **new row version** with `xmin` = your transaction ID.
3. Both versions exist physically in the heap until VACUUM removes the old one.

When you DELETE a row:
1. PostgreSQL marks the row's `xmax` with your transaction ID.
2. The row is not physically removed — it becomes a "dead tuple" visible only to snapshots that started before the delete committed.

### 2.3 Transaction Snapshots — What Your Transaction Can See

When a transaction starts, PostgreSQL takes a **snapshot** of the current state. The snapshot records:

- `xmin` — the oldest active transaction ID at snapshot time
- `xmax` — the next unassigned transaction ID at snapshot time
- A list of in-progress transaction IDs at snapshot time

**Visibility rule:** A row version is visible to a transaction if:
- Its `xmin` committed before the snapshot was taken (and is not in the in-progress list)
- Its `xmax` is either 0, or did not commit before the snapshot was taken

```sql
-- Inspect your current snapshot
SELECT pg_current_snapshot();
-- Result: 1050:1052:1051
-- Means: txns < 1050 are all committed, txns >= 1052 are all invisible,
--        txn 1051 was in-progress when snapshot was taken (invisible)
```

### 2.4 Why Reads Don't Block Writes

Session A reads a row. Session B updates the same row. In a traditional lock-based system, B would wait for A to release its read lock.

In PostgreSQL:
- A is reading an old version of the row (its snapshot)
- B creates a new version of the row
- Both proceed simultaneously
- A never sees B's changes (because B's `xmin` > A's snapshot `xmax`)

```sql
-- Session A (txn 1050) — starts first
BEGIN;
SELECT balance FROM accounts WHERE id = 1;
-- Returns 10000, reads the version with xmin=1040, xmax=0

-- Session B (txn 1051) — runs concurrently
BEGIN;
UPDATE accounts SET balance = 9000 WHERE id = 1;
-- Inserts new row version: xmin=1051, xmax=0
-- Marks old version: xmax=1051
COMMIT;

-- Session A continues
SELECT balance FROM accounts WHERE id = 1;
-- At READ COMMITTED: returns 9000 (re-reads after B committed)
-- At REPEATABLE READ: returns 10000 (locked to its original snapshot)
COMMIT;
```

### 2.5 The MVCC Version Chain

For a heavily updated row, the heap file contains multiple versions:

```
Physical heap page for accounts row id=1:
┌─────────────────────────────────────────┐
│ xmin=1040, xmax=1043, balance=10500 │  dead tuple (cleaned by VACUUM)
│ xmin=1043, xmax=1048, balance=10000 │  dead tuple
│ xmin=1048, xmax=1051, balance=9500  │  visible to snapshots < 1051
│ xmin=1051, xmax=0,    balance=9000  │  current version
└─────────────────────────────────────────┘
```

### 2.6 MVCC and VACUUM — The Hidden Cost

Dead tuples accumulate. VACUUM is the process that reclaims them. **Long-running transactions prevent VACUUM from cleaning up**, because VACUUM cannot remove a row version that is still visible to any active transaction.

```sql
-- Find the oldest active transaction preventing VACUUM
SELECT pid, now() - pg_stat_activity.query_start AS duration,
       query, state, xact_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY xact_start;

-- Check table bloat from dead tuples
SELECT schemaname, tablename,
       n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

> **Key insight:** A transaction open for 30 minutes prevents VACUUM from reclaiming any row versions modified after it started. On a high-write table, this causes table bloat, index bloat, and eventually query slowdowns. This is why long-running transactions are dangerous even if they're just doing SELECTs.

---

## 3. Isolation Levels — Real Behavior, Real Anomalies

### 3.1 The Anomalies Reference

| Anomaly | Definition | Read Committed | Repeatable Read | Serializable |
|---|---|---|---|---|
| Dirty Read | Reading uncommitted data from another transaction | Prevented | Prevented | Prevented |
| Non-Repeatable Read | Same row returns different value within a transaction | Possible | Prevented | Prevented |
| Phantom Read | Same query returns different set of rows | Possible | Prevented* | Prevented |
| Serialization Anomaly | Transaction outcomes are inconsistent with any serial order | Possible | Possible | Prevented |

*PostgreSQL's Repeatable Read also prevents phantom reads, unlike the SQL standard which only requires prevention at Serializable.

### 3.2 Read Uncommitted — PostgreSQL's Silent Upgrade

The SQL standard defines four isolation levels. PostgreSQL implements only three distinct behaviors:

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- PostgreSQL silently upgrades this to READ COMMITTED.
-- There is no way to read uncommitted data in PostgreSQL.
-- Dirty reads are architecturally impossible under MVCC.
```

**Why:** MVCC never exposes uncommitted row versions to other transactions. An uncommitted UPDATE creates a new row version with `xmin` = the active transaction ID. That version is invisible to any snapshot until the transaction commits. There is no mechanism to "see" it.

**When you'd use it:** You wouldn't. It exists for SQL standard compliance. In PostgreSQL, READ UNCOMMITTED = READ COMMITTED.

### 3.3 Read Committed — The Default, and Its Dangers

```sql
-- This is the default. You don't need to set it explicitly.
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**Behavior:** Every statement in the transaction gets a fresh snapshot. This means two SELECTs in the same transaction can return different data if another transaction commits between them.

**Scenario — Non-repeatable read causing a logic bug:**

```sql
-- Scenario: Pricing engine that reads a product price twice in one transaction

-- Session A (transaction processing order)
BEGIN;
SELECT price FROM products WHERE id = 42;  -- Returns 100.00
-- ... some processing ...
-- Another session (B) changes price to 200.00 and commits here
SELECT price FROM products WHERE id = 42;  -- Returns 200.00 ← different!
INSERT INTO order_items (product_id, price) VALUES (42, 100.00);  -- Uses cached value
COMMIT;
-- Result: Order placed at old price, DB shows new price. Discrepancy.
```

**Scenario — Phantom read causing incorrect aggregation:**

```sql
-- Session A: computing a report
BEGIN;
SELECT COUNT(*) FROM orders WHERE status = 'pending';  -- Returns 50
-- Session B inserts 5 new pending orders and commits
SELECT SUM(total) FROM orders WHERE status = 'pending';  -- Sees 55 orders
COMMIT;
-- Report counts 50 orders but sums 55 orders. Data is internally inconsistent.
```

**When to use Read Committed:**
- Simple CRUD operations where each statement is self-contained
- High-concurrency workloads where occasional inconsistency is tolerable
- When you want maximum throughput and your application handles retries

### 3.4 Repeatable Read — Snapshot Isolation

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

**Behavior:** The transaction takes a single snapshot at START (not per-statement). All reads within the transaction see the same data, regardless of concurrent commits.

**What it prevents:** Non-repeatable reads and phantom reads.

**What it does NOT prevent:** Serialization anomalies (write skew).

**Scenario — Write skew (the anomaly Repeatable Read cannot prevent):**

```sql
-- Business rule: a doctor can take the day off only if at least 1 other doctor is on-call
-- doctors table: (id, name, on_call BOOLEAN)
-- Currently: Alice = on_call, Bob = on_call

-- Session A (Alice taking a day off)
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- Returns 2 (Alice + Bob)
-- "There are 2 on-call doctors, I can safely go off-call"
UPDATE doctors SET on_call = false WHERE name = 'Alice';

-- Session B (Bob taking a day off, concurrently)
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- Returns 2 (Alice + Bob)
-- "There are 2 on-call doctors, I can safely go off-call"
UPDATE doctors SET on_call = false WHERE name = 'Bob';

-- Both commit successfully.
-- Result: 0 doctors on call. Business rule violated.
-- This is write skew — each transaction read consistent data but the combined
-- writes created an invalid state.
```

**When to use Repeatable Read:**
- Financial reports that must be internally consistent
- Multi-statement business logic that reads the same rows multiple times
- Anywhere a non-repeatable read would cause a logic error

**Important:** Under Repeatable Read, if another transaction modifies a row you modified and commits first, PostgreSQL will **abort your transaction** with:
```
ERROR: could not serialize access due to concurrent update
```
Your application must catch this and retry.

### 3.5 Serializable — SSI (Serializable Snapshot Isolation)

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**Behavior:** Transactions behave as if they ran one at a time in some serial order. PostgreSQL implements this with SSI (Serializable Snapshot Isolation) — a highly concurrent algorithm that tracks read/write dependencies between transactions and aborts one when a cycle is detected.

**This is not two-phase locking.** Traditional serializable isolation uses S2PL which blocks aggressively. PostgreSQL's SSI detects conflicts without blocking and aborts losers retroactively. It has much better performance.

**SSI fixes the write skew problem:**

```sql
-- Same doctor scenario with SERIALIZABLE:
-- Session A and Session B both run under SERIALIZABLE
-- PostgreSQL tracks that both read "on_call doctors count" and both wrote to doctors
-- It detects the rw-anti-dependency cycle
-- One of the transactions gets aborted:
-- ERROR: could not serialize access due to read/write dependencies among transactions
```

**Scenario where SERIALIZABLE is essential:**

```sql
-- Coupon system: a coupon can be used once. Race condition under Read Committed:
-- Two sessions both check "is coupon used?" → both see "no" → both mark it used
-- Under SERIALIZABLE, PostgreSQL detects the conflict and aborts one.

BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT used FROM coupons WHERE code = 'SAVE20';
-- Returns false
UPDATE coupons SET used = true, used_by = 42 WHERE code = 'SAVE20' AND used = false;
COMMIT;
```

**Cost of SERIALIZABLE:**
- More transaction aborts (must handle retries)
- Additional memory for tracking predicate locks
- Slightly higher CPU overhead

**When to use Serializable:**
- Financial systems where write skew could cause inconsistency
- Any scenario where "check then act" logic spans multiple rows
- Complex business rule enforcement that constraints cannot capture

> **Decision:** Most applications should use Read Committed for throughput and Serializable for critical business logic. Repeatable Read is a useful middle ground but write skew is a footgun — if you need Repeatable Read, double-check whether you actually need Serializable.

---

## 4. Locking — Row Locks, Table Locks, Advisory Locks

### 4.1 Row-Level Locks

PostgreSQL row locks are NOT stored in a lock table for individual rows (unlike some databases). Instead, they are stored in the row itself (in the tuple header — the `xmax` field and associated flag bits). This means row locking scales to millions of locked rows without memory pressure.

**The four row lock modes:**

```sql
-- 1. FOR KEY SHARE
-- Weakest. Prevents deletion and changes to key columns.
-- Used by: foreign key checks on the referencing side.
SELECT * FROM orders WHERE id = 1 FOR KEY SHARE;

-- 2. FOR SHARE
-- Prevents UPDATE and DELETE. Allows other FOR SHARE lockers.
-- Use case: read a row and prevent others from changing it while you process it.
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- 3. FOR NO KEY UPDATE
-- Prevents other FOR SHARE, FOR UPDATE. Allows FOR KEY SHARE.
-- Used by: UPDATE statements that don't change key columns.
SELECT * FROM accounts WHERE id = 1 FOR NO KEY UPDATE;

-- 4. FOR UPDATE
-- Strongest. Exclusive. Prevents all other row locks.
-- Use case: "select then update" pattern — claim the row.
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
```

**Row lock compatibility matrix:**

| | FOR KEY SHARE | FOR SHARE | FOR NO KEY UPDATE | FOR UPDATE |
|---|---|---|---|---|
| **FOR KEY SHARE** | OK | OK | OK | BLOCK |
| **FOR SHARE** | OK | OK | BLOCK | BLOCK |
| **FOR NO KEY UPDATE** | OK | BLOCK | BLOCK | BLOCK |
| **FOR UPDATE** | BLOCK | BLOCK | BLOCK | BLOCK |

**The key insight:** Use `FOR NO KEY UPDATE` instead of `FOR UPDATE` when you're updating non-key columns. It allows FK checks to proceed on child tables concurrently, reducing contention in parent-child table patterns.

### 4.2 NOWAIT and SKIP LOCKED

**NOWAIT — fail immediately instead of waiting:**

```sql
-- Without NOWAIT: blocks until lock is available (could be forever)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- With NOWAIT: immediately raises an error if the row is locked
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "accounts"

-- Use case: prevent request pile-ups. If you can't get the lock immediately,
-- tell the user "try again" rather than having 1000 requests queue up.
BEGIN;
SELECT balance FROM accounts WHERE id = $1 FOR UPDATE NOWAIT;
-- If this raises, catch the error and return HTTP 409 Conflict
```

**SKIP LOCKED — skip locked rows, process the rest:**

```sql
-- The job queue pattern — multiple workers, no double processing:
CREATE TABLE jobs (
    id BIGSERIAL PRIMARY KEY,
    status TEXT DEFAULT 'pending',
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- Worker process (run this in multiple concurrent sessions)
BEGIN;
SELECT id, payload
FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- If a row is returned, process it
-- If nothing is returned, no pending jobs available
UPDATE jobs SET status = 'processing' WHERE id = $returned_id;
COMMIT;
```

With SKIP LOCKED, each worker atomically claims a different job. No worker ever sees a job another worker has locked. No polling, no explicit queue management, no double-processing.

### 4.3 Table-Level Locks

Table locks are heavier and held for the duration of the transaction. They show up in `pg_locks` with `relation` as the lock target.

```sql
-- Explicitly lock an entire table
LOCK TABLE accounts IN SHARE MODE;
LOCK TABLE accounts IN EXCLUSIVE MODE;
LOCK TABLE accounts IN ACCESS EXCLUSIVE MODE;
```

**Table lock modes (from lightest to heaviest):**

| Lock Mode | Conflicts With | Typical Trigger |
|---|---|---|
| ACCESS SHARE | ACCESS EXCLUSIVE | SELECT |
| ROW SHARE | EXCLUSIVE, ACCESS EXCLUSIVE | SELECT FOR UPDATE/SHARE |
| ROW EXCLUSIVE | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | INSERT, UPDATE, DELETE |
| SHARE UPDATE EXCLUSIVE | SHARE UPDATE EXCLUSIVE, SHARE, EXCLUSIVE, ACCESS EXCLUSIVE | VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY |
| SHARE | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | CREATE INDEX (non-concurrent) |
| SHARE ROW EXCLUSIVE | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | TRIGGER creation |
| EXCLUSIVE | All except ACCESS SHARE | Rare manual use |
| ACCESS EXCLUSIVE | All | ALTER TABLE, DROP TABLE, TRUNCATE, VACUUM FULL |

**The production danger — DDL operations:**

```sql
-- ALTER TABLE acquires ACCESS EXCLUSIVE lock.
-- This blocks ALL reads and writes while waiting.
-- If there's a long-running transaction, ALTER TABLE queues behind it.
-- Everything that needs even ACCESS SHARE then queues behind ALTER TABLE.
-- Result: a traffic jam that takes down your application.

-- Always set a lock timeout before schema changes in production:
SET lock_timeout = '2s';
ALTER TABLE users ADD COLUMN preferences JSONB;
-- If it can't get the lock in 2s, it fails rather than blocking the world.
```

### 4.4 Advisory Locks

Advisory locks are application-managed locks that PostgreSQL stores and coordinates, but has no semantic meaning to PostgreSQL itself. Your application defines what they mean.

```sql
-- Session-level advisory lock (held until session ends or explicitly released)
SELECT pg_advisory_lock(12345);           -- blocks until available
SELECT pg_try_advisory_lock(12345);       -- returns true/false immediately

-- Transaction-level advisory lock (released at end of transaction)
SELECT pg_advisory_xact_lock(12345);
SELECT pg_try_advisory_xact_lock(12345);

-- Release session-level locks
SELECT pg_advisory_unlock(12345);
SELECT pg_advisory_unlock_all();
```

**Real use case — distributed mutual exclusion:**

```sql
-- Ensure only one process runs a scheduled job at a time (across multiple app servers)
-- Use the job_id as the lock key

DO $$
DECLARE
    lock_acquired BOOLEAN;
BEGIN
    SELECT pg_try_advisory_xact_lock(hashtext('nightly_report_job')) INTO lock_acquired;

    IF NOT lock_acquired THEN
        RAISE NOTICE 'Another process is already running this job';
        RETURN;
    END IF;

    -- Run the job — lock is automatically released at transaction end
    PERFORM generate_nightly_report();
END;
$$;
```

**Advisory locks for rate limiting:**

```sql
-- Prevent concurrent processing of the same user's requests
-- Lock key = user_id cast to bigint
SELECT pg_try_advisory_xact_lock($user_id);
-- If false: another request for this user is in-flight, return 429 Too Many Requests
```

### 4.5 Implicit Locks from Django and Application Code

Django ORM triggers implicit locks without you writing explicit lock SQL:

```python
# Django ORM: select_for_update() → SELECT ... FOR UPDATE
with transaction.atomic():
    account = Account.objects.select_for_update().get(id=account_id)
    account.balance -= 500
    account.save()

# Django ORM: select_for_update(nowait=True) → SELECT ... FOR UPDATE NOWAIT
# Django ORM: select_for_update(skip_locked=True) → SELECT ... FOR UPDATE SKIP LOCKED
# Django ORM: select_for_update(of=('self',)) → only locks the primary table in joins
# Django ORM: select_for_update(no_key=True) → SELECT ... FOR NO KEY UPDATE
```

What Django generates implicitly:
- `Model.objects.filter(...).update(...)` → ROW EXCLUSIVE lock on each updated row (via UPDATE statement)
- `Model.objects.create(...)` → ROW EXCLUSIVE lock (INSERT)
- Any `save()` on an existing object → UPDATE → ROW EXCLUSIVE

---

## 5. Deadlocks — Detection, Diagnosis, Prevention

### 5.1 How Deadlocks Happen — Step by Step

A deadlock occurs when two transactions each hold a lock the other needs.

```sql
-- Schema
CREATE TABLE accounts (id INT PRIMARY KEY, balance NUMERIC);
INSERT INTO accounts VALUES (1, 10000), (2, 5000);

-- Transaction A: transfer from account 1 to account 2
-- Transaction B: transfer from account 2 to account 1 (at the same time)

-- TIME T1: Transaction A starts
BEGIN;  -- A
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
-- A now holds a lock on row id=1

-- TIME T2: Transaction B starts
BEGIN;  -- B
UPDATE accounts SET balance = balance - 300 WHERE id = 2;
-- B now holds a lock on row id=2

-- TIME T3: Transaction A tries to lock row id=2
UPDATE accounts SET balance = balance + 500 WHERE id = 2;  -- A BLOCKS, waiting for B

-- TIME T4: Transaction B tries to lock row id=1
UPDATE accounts SET balance = balance + 300 WHERE id = 1;  -- B BLOCKS, waiting for A

-- DEADLOCK: A waits for B, B waits for A. Neither can proceed.
-- PostgreSQL detects this after deadlock_timeout (default: 1 second)
-- PostgreSQL picks one as the victim and aborts it:
-- ERROR:  deadlock detected
-- DETAIL:  Process 1234 waits for ShareLock on transaction 5678; blocked by process 5678.
--          Process 5678 waits for ShareLock on transaction 1234; blocked by process 1234.
-- HINT:   See server log for query details.
```

### 5.2 The deadlock_timeout Setting

```sql
SHOW deadlock_timeout;
-- 1s (default)

-- PostgreSQL checks for deadlocks only when a lock wait exceeds deadlock_timeout.
-- This avoids the overhead of deadlock detection on every lock acquisition.
-- A lock_timeout fires first if set; deadlock detection only runs if the wait
-- exceeds deadlock_timeout.

-- Setting in postgresql.conf or per-session:
SET deadlock_timeout = '500ms';  -- more responsive detection
```

### 5.3 Reading pg_locks to Find Blocking Queries

```sql
-- Find all blocking/blocked lock pairs
SELECT
    blocked.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocked_activity.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocking_activity.query AS blocking_query,
    now() - blocked_activity.query_start AS blocked_duration
FROM pg_locks AS blocked
JOIN pg_stat_activity AS blocked_activity ON blocked.pid = blocked_activity.pid
JOIN pg_locks AS blocking
    ON blocking.locktype = blocked.locktype
    AND blocking.relation IS NOT DISTINCT FROM blocked.relation
    AND blocking.page IS NOT DISTINCT FROM blocked.page
    AND blocking.tuple IS NOT DISTINCT FROM blocked.tuple
    AND blocking.transactionid IS NOT DISTINCT FROM blocked.transactionid
    AND blocking.pid != blocked.pid
JOIN pg_stat_activity AS blocking_activity ON blocking.pid = blocking_activity.pid
WHERE NOT blocked.granted
ORDER BY blocked_duration DESC;
```

```sql
-- Detailed view of all current locks with table names
SELECT
    l.pid,
    l.locktype,
    l.relation::regclass AS table_name,
    l.mode,
    l.granted,
    a.query,
    a.state,
    now() - a.state_change AS lock_age
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation IS NOT NULL
ORDER BY lock_age DESC NULLS LAST;
```

### 5.4 Prevention Strategies

**Strategy 1: Consistent lock ordering**

The root cause of the deadlock above is that A locks row 1 then row 2, while B locks row 2 then row 1. If all transactions always lock rows in the same order, deadlocks cannot occur.

```sql
-- WRONG: locks rows in arbitrary order based on input
CREATE OR REPLACE FUNCTION transfer(from_id INT, to_id INT, amount NUMERIC)
RETURNS VOID AS $$
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
END;
$$ LANGUAGE plpgsql;

-- RIGHT: always lock the lower id first
CREATE OR REPLACE FUNCTION transfer(from_id INT, to_id INT, amount NUMERIC)
RETURNS VOID AS $$
DECLARE
    first_id INT  := LEAST(from_id, to_id);
    second_id INT := GREATEST(from_id, to_id);
BEGIN
    -- Lock both rows in a consistent order, regardless of direction
    PERFORM id FROM accounts WHERE id = first_id FOR UPDATE;
    PERFORM id FROM accounts WHERE id = second_id FOR UPDATE;

    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
END;
$$ LANGUAGE plpgsql;
```

**Strategy 2: Lock everything upfront**

```sql
-- Lock all rows you'll need at the start, before doing any work
BEGIN;
SELECT id FROM accounts
WHERE id IN (1, 2)
ORDER BY id  -- consistent ordering!
FOR UPDATE;

-- Now do the work — no additional lock waits possible
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;
```

**Strategy 3: Lock timeouts to fail fast**

```sql
-- Fail rather than wait indefinitely — surface deadlocks as errors quickly
SET lock_timeout = '5s';
SET deadlock_timeout = '500ms';

BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
-- If this blocks for >5s: ERROR: canceling statement due to lock timeout
COMMIT;
```

**Strategy 4: Retry on deadlock in application code**

Since PostgreSQL aborts one party in a deadlock, the correct response is to retry the entire transaction:

```python
import psycopg2
from psycopg2 import errors
import time

def transfer_with_retry(from_id, to_id, amount, max_retries=3):
    for attempt in range(max_retries):
        try:
            with connection.cursor() as cur:
                cur.execute("BEGIN")
                cur.execute(
                    "SELECT id FROM accounts WHERE id = ANY(%s) ORDER BY id FOR UPDATE",
                    ([sorted([from_id, to_id])],)
                )
                cur.execute(
                    "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                    (amount, from_id)
                )
                cur.execute(
                    "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                    (amount, to_id)
                )
                cur.execute("COMMIT")
                return
        except errors.DeadlockDetected:
            cur.execute("ROLLBACK")
            if attempt < max_retries - 1:
                time.sleep(0.1 * (2 ** attempt))  # exponential backoff
            else:
                raise
```

---

## 6. Real-World Race Conditions and How to Solve Them

### 6.1 The Double-Spend Problem (Bank Transfer)

**Scenario:** Two concurrent withdrawals from the same account, both checking the balance first.

**The wrong way:**

```sql
-- Session A: withdraw $800 (balance check → $1000, proceed)
-- Session B: withdraw $800 (balance check → $1000, proceed)

-- Session A:
SELECT balance FROM accounts WHERE id = 1;  -- Returns 1000
-- Session B:
SELECT balance FROM accounts WHERE id = 1;  -- Returns 1000
-- Both see $1000, both think they can withdraw $800
UPDATE accounts SET balance = balance - 800 WHERE id = 1;  -- A: balance = 200
UPDATE accounts SET balance = balance - 800 WHERE id = 1;  -- B: balance = -600 ← WRONG
```

**The right way — check-and-act atomically in the UPDATE:**

```sql
BEGIN;
UPDATE accounts
SET balance = balance - 800
WHERE id = 1 AND balance >= 800;  -- The check is in the WHERE clause

GET DIAGNOSTICS rows_affected = ROW_COUNT;
IF rows_affected = 0 THEN
    ROLLBACK;
    RAISE EXCEPTION 'Insufficient funds';
END IF;
COMMIT;
```

One of the concurrent sessions will update first. When the second session runs, `balance >= 800` is false (balance is now 200), so it updates 0 rows and fails gracefully. No need for an explicit SELECT.

**With explicit locking for more complex logic:**

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- Now I have an exclusive lock. The other session blocks here.
-- When I release the lock (COMMIT), it reads the updated balance.

IF balance < 800 THEN
    ROLLBACK;
    RAISE EXCEPTION 'Insufficient funds';
END IF;

UPDATE accounts SET balance = balance - 800 WHERE id = 1;
INSERT INTO transactions (account_id, amount, type) VALUES (1, 800, 'debit');
COMMIT;
```

### 6.2 Inventory Overselling (E-Commerce)

**Scenario:** 1 item left. 100 concurrent requests try to buy it.

**The wrong way:**

```sql
-- All 100 sessions do:
SELECT stock_count FROM products WHERE id = 99;  -- All see stock_count = 1
-- All decide to proceed
UPDATE products SET stock_count = stock_count - 1 WHERE id = 99;
-- stock_count is now -99. 100 orders placed for 1 item.
```

**The right way — optimistic locking with conditional UPDATE:**

```sql
BEGIN;
UPDATE products
SET stock_count = stock_count - 1
WHERE id = 99 AND stock_count > 0;  -- Only succeeds if stock available

-- Check if we actually got the stock
-- (using RETURNING is cleaner)
UPDATE products
SET stock_count = stock_count - 1
WHERE id = 99 AND stock_count > 0
RETURNING stock_count;
-- If no row returned: out of stock
-- If row returned with stock_count >= 0: success
COMMIT;
```

**For bulk reservation (reserve N items):**

```sql
CREATE OR REPLACE FUNCTION reserve_stock(p_product_id INT, p_quantity INT)
RETURNS BOOLEAN AS $$
DECLARE
    updated_count INT;
BEGIN
    UPDATE products
    SET stock_count = stock_count - p_quantity
    WHERE id = p_product_id
      AND stock_count >= p_quantity;

    GET DIAGNOSTICS updated_count = ROW_COUNT;
    RETURN updated_count > 0;
END;
$$ LANGUAGE plpgsql;
```

### 6.3 Unique Username Registration (Concurrent Signups)

**Scenario:** Two users try to register "alice" at the same millisecond.

**The wrong way:**

```sql
-- Both sessions:
SELECT id FROM users WHERE username = 'alice';  -- Both see 0 rows
-- Both decide username is available
INSERT INTO users (username, email) VALUES ('alice', '...');
-- One succeeds, one... also succeeds if there's no UNIQUE constraint
```

**Why a UNIQUE constraint is the correct solution:**

```sql
-- The UNIQUE constraint is enforced at INSERT/COMMIT time by PostgreSQL
-- regardless of isolation level. It is not subject to MVCC visibility rules.
-- One transaction will succeed; the other gets:
-- ERROR: duplicate key value violates unique constraint "users_username_key"

ALTER TABLE users ADD CONSTRAINT users_username_key UNIQUE (username);

-- Application code:
try:
    User.objects.create(username=username, email=email)
except IntegrityError:
    return Response({"error": "Username taken"}, status=400)
```

**For case-insensitive uniqueness:**

```sql
-- A regular UNIQUE constraint is case-sensitive: 'Alice' != 'alice'
-- Use a unique index on the lowercased value:
CREATE UNIQUE INDEX users_username_lower_idx ON users (lower(username));

-- Or use citext extension:
CREATE EXTENSION IF NOT EXISTS citext;
ALTER TABLE users ALTER COLUMN username TYPE citext;
-- Now 'Alice' and 'alice' are treated as duplicates automatically
```

### 6.4 Coupon Code Single-Use Enforcement

**Scenario:** A coupon should be redeemable exactly once. Concurrent requests try to use it.

**The wrong way (application-level check):**

```sql
-- Session A: SELECT used FROM coupons WHERE code = 'SAVE20' → false
-- Session B: SELECT used FROM coupons WHERE code = 'SAVE20' → false
-- Both proceed to mark as used and apply discount
-- Session A: UPDATE coupons SET used = true WHERE code = 'SAVE20'
-- Session B: UPDATE coupons SET used = true WHERE code = 'SAVE20'
-- Coupon used twice
```

**The right way — atomic conditional UPDATE:**

```sql
BEGIN;
UPDATE coupons
SET used = true,
    used_by_user_id = $user_id,
    used_at = now()
WHERE code = $coupon_code
  AND used = false;     -- Only succeeds if not already used

-- Check if update happened
-- If 0 rows updated: coupon already used or doesn't exist
COMMIT;
```

**With FOR UPDATE for multi-step redemption logic:**

```sql
BEGIN;
SELECT id, discount_percent, used
FROM coupons
WHERE code = $coupon_code
FOR UPDATE;  -- Lock the coupon row

IF NOT FOUND THEN
    ROLLBACK;
    RAISE EXCEPTION 'Coupon not found';
END IF;

IF used THEN
    ROLLBACK;
    RAISE EXCEPTION 'Coupon already used';
END IF;

-- Apply the discount
UPDATE orders SET total = total * (1 - discount_percent / 100.0)
WHERE id = $order_id;

-- Mark coupon as used
UPDATE coupons SET used = true, used_by_user_id = $user_id
WHERE code = $coupon_code;

COMMIT;
-- The FOR UPDATE ensures no other session can read-then-use this coupon concurrently
```

### 6.5 Job Queue with Multiple Workers (SKIP LOCKED Pattern)

**Scenario:** 10 worker processes pulling jobs from a queue. Each job must be processed exactly once.

**The wrong way — polling without locking:**

```sql
-- Multiple workers all run:
SELECT id FROM jobs WHERE status = 'pending' LIMIT 1;
-- All workers see the same job
UPDATE jobs SET status = 'processing' WHERE id = $id;
-- Multiple workers claim the same job
```

**The right way — atomic claim with SKIP LOCKED:**

```sql
-- Each worker runs this atomically:
BEGIN;
WITH claimed AS (
    SELECT id, payload
    FROM jobs
    WHERE status = 'pending'
    ORDER BY priority DESC, created_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED   -- Skip rows locked by other workers
)
UPDATE jobs
SET status = 'processing',
    started_at = now(),
    worker_id = $worker_id
FROM claimed
WHERE jobs.id = claimed.id
RETURNING jobs.id, jobs.payload;

-- If RETURNING returns a row: process it, then:
UPDATE jobs SET status = 'completed', completed_at = now() WHERE id = $job_id;
COMMIT;

-- If RETURNING returns nothing: no available jobs, worker can sleep/poll
```

**Complete job queue schema:**

```sql
CREATE TABLE jobs (
    id          BIGSERIAL PRIMARY KEY,
    status      TEXT    NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    priority    INT     NOT NULL DEFAULT 0,
    payload     JSONB   NOT NULL,
    worker_id   TEXT,
    error       TEXT,
    attempts    INT     NOT NULL DEFAULT 0,
    max_attempts INT    NOT NULL DEFAULT 3,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at  TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    scheduled_for TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX jobs_pending_idx ON jobs (priority DESC, created_at ASC)
    WHERE status = 'pending' AND scheduled_for <= now();

-- Reclaim stuck jobs (processing for > 30 minutes, eligible for retry)
UPDATE jobs
SET status = 'pending',
    worker_id = NULL,
    attempts = attempts + 1,
    started_at = NULL
WHERE status = 'processing'
  AND started_at < now() - INTERVAL '30 minutes'
  AND attempts < max_attempts;
```

---

## 7. Savepoints — Nested Rollback Control

### 7.1 What Savepoints Are

A savepoint is a named marker within a transaction. You can roll back to a savepoint without aborting the entire transaction, then continue.

```sql
BEGIN;

INSERT INTO orders (customer_id, total) VALUES (1, 100.00);

SAVEPOINT after_order;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 99, 2);

-- Something fails here (e.g., product 99 doesn't exist)
-- Roll back only to the savepoint, not the entire transaction
ROLLBACK TO SAVEPOINT after_order;

-- The order row still exists; only the order_item insert was rolled back
-- Try with a valid product
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 50, 2);

COMMIT;
-- The order and the order_item (for product 50) are committed
```

### 7.2 Real Use Case — Bulk Insert with Error Tolerance

**Scenario:** Import 10,000 rows. Some rows may violate constraints. Skip the bad rows, commit the good ones.

```sql
BEGIN;

DO $$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN SELECT * FROM staging_import LOOP
        SAVEPOINT row_import;
        BEGIN
            INSERT INTO users (email, username, created_at)
            VALUES (rec.email, rec.username, rec.created_at);
        EXCEPTION WHEN unique_violation OR not_null_violation THEN
            ROLLBACK TO SAVEPOINT row_import;
            -- Log the failure
            INSERT INTO import_errors (email, reason, created_at)
            VALUES (rec.email, SQLERRM, now());
        END;
        RELEASE SAVEPOINT row_import;
    END LOOP;
END;
$$;

COMMIT;
-- All valid rows are committed; invalid rows are logged and skipped
```

### 7.3 How Django Uses Savepoints

Django's `atomic()` block uses savepoints when nested:

```python
with transaction.atomic():  # BEGIN
    Order.objects.create(...)

    with transaction.atomic():  # SAVEPOINT sp1
        try:
            OrderItem.objects.create(...)
        except IntegrityError:
            pass  # ROLLBACK TO SAVEPOINT sp1
    # Outer transaction continues

# COMMIT
```

PostgreSQL's `EXCEPTION` block in PL/pgSQL also implicitly uses savepoints:

```sql
BEGIN
    -- code
EXCEPTION
    WHEN unique_violation THEN
        -- PostgreSQL internally created a savepoint here
        -- The exception block can handle the error and the function continues
END;
```

> **When to use savepoints:** Error isolation in batch operations. Partial rollback in complex stored procedures. Do not use them to replace proper transaction design — each SAVEPOINT adds overhead.

---

## 8. Transaction Anti-Patterns That Kill Production

### 8.1 Long-Running Transactions

This is the most dangerous anti-pattern.

**What happens:**
1. A transaction starts (xmin is assigned)
2. VACUUM cannot remove any dead tuple with `xmax` >= your transaction's `xmin`
3. Tables bloat with dead tuples
4. Index scans slow down (more pages to read)
5. `pg_stat_activity` shows your query; other sessions queue up

```sql
-- Find long-running transactions
SELECT
    pid,
    now() - xact_start AS transaction_age,
    now() - query_start AS query_age,
    state,
    wait_event_type,
    wait_event,
    left(query, 100) AS query_snippet
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND now() - xact_start > INTERVAL '5 minutes'
ORDER BY transaction_age DESC;
```

**The culprit: idle-in-transaction connections**

```sql
-- An idle-in-transaction connection has started a transaction but issued no query.
-- This often means application code opened a transaction and got stuck waiting
-- for something (HTTP call, lock, application logic).

SELECT pid, state, now() - state_change AS idle_duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_duration DESC;

-- Enforce a timeout on idle-in-transaction sessions:
-- In postgresql.conf:
-- idle_in_transaction_session_timeout = '5min'

-- Per-session:
SET idle_in_transaction_session_timeout = '5min';
```

### 8.2 Transactions in Loops

**The wrong way — one transaction per loop iteration:**

```python
# This creates 10,000 individual transactions
# Each BEGIN/COMMIT is a round-trip + WAL flush
for item in items:
    with transaction.atomic():
        Product.objects.filter(id=item.id).update(price=item.price)
```

**The right way — batch in a single transaction:**

```python
with transaction.atomic():
    for item in items:
        Product.objects.filter(id=item.id).update(price=item.price)
# One transaction, one WAL flush — 100x faster
```

**But don't make the transaction too large either:**

```python
# For very large batches, chunk it: one transaction per 1000 rows
BATCH_SIZE = 1000
for i in range(0, len(items), BATCH_SIZE):
    batch = items[i:i + BATCH_SIZE]
    with transaction.atomic():
        for item in batch:
            Product.objects.filter(id=item.id).update(price=item.price)
```

**Even better — use `bulk_update`:**

```python
with transaction.atomic():
    Product.objects.bulk_update(items, ['price'], batch_size=1000)
```

**Or a single UPDATE with a VALUES list:**

```sql
UPDATE products AS p
SET price = v.price
FROM (VALUES
    (1, 29.99),
    (2, 49.99),
    (3, 9.99)
) AS v(id, price)
WHERE p.id = v.id;
```

### 8.3 HTTP Calls Inside Transactions

```python
# CATASTROPHICALLY WRONG
with transaction.atomic():
    order = Order.objects.create(...)

    # This HTTP call could take 0ms or 30 seconds
    # Your transaction is OPEN the entire time
    # Locks are held, MVCC horizon is frozen
    response = requests.post('https://payment-api.com/charge', ...)

    payment = Payment.objects.create(order=order, ...)
```

If the payment API is slow:
- Your transaction is open for seconds (or forever if it times out)
- Locks on the `orders` table are held
- VACUUM cannot proceed on affected rows
- Connection pool is exhausted

**The right pattern — separate the I/O from the transaction:**

```python
# Step 1: Create order record (short transaction)
with transaction.atomic():
    order = Order.objects.create(status='pending', ...)

# Step 2: External call (outside any transaction)
try:
    response = requests.post('https://payment-api.com/charge', ...)
    charge_id = response.json()['charge_id']
except Exception as e:
    # Handle failure, mark order as failed
    Order.objects.filter(id=order.id).update(status='payment_failed')
    raise

# Step 3: Update record (short transaction)
with transaction.atomic():
    Payment.objects.create(order=order, charge_id=charge_id, ...)
    Order.objects.filter(id=order.id).update(status='confirmed')
```

### 8.4 Auto-Commit Misconceptions

In PostgreSQL's `psql` client, every statement is auto-committed by default. In Python with psycopg2, `autocommit` is **off** by default — every operation is in an implicit transaction.

```python
import psycopg2

conn = psycopg2.connect(...)
cur = conn.cursor()

# This is NOT auto-committed — it's in an implicit transaction
cur.execute("UPDATE accounts SET balance = 1000 WHERE id = 1")
# The update is not visible to other sessions yet

conn.commit()  # Now it's committed

# For read-only workloads, set autocommit=True to avoid transaction overhead:
conn.autocommit = True
cur.execute("SELECT * FROM users WHERE id = 1")
# No transaction overhead, no idle-in-transaction risk
```

**Django auto-commit behavior:**

```python
# Django wraps each ORM call in a transaction by default (autocommit mode)
# Each User.objects.create(), .update(), .filter() is its own transaction

# ATOMIC_REQUESTS = True in settings:
# Every HTTP request is wrapped in a single transaction
# Great for consistency, dangerous for slow requests that hold locks

# Explicit transaction management overrides this:
@transaction.non_atomic_requests
def my_view(request):
    # This view is NOT wrapped in a transaction
    pass
```

### 8.5 Missing Error Handling in Aborted Transactions

In PostgreSQL, once a transaction encounters an error, it is **aborted**. Any further commands until ROLLBACK return:
```
ERROR: current transaction is aborted, commands ignored until end of transaction block
```

```python
# WRONG: ignoring errors in a transaction
conn.autocommit = False
try:
    cur.execute("INSERT INTO users (email) VALUES ('duplicate@email.com')")
    # This raises IntegrityError (duplicate)
except psycopg2.IntegrityError:
    pass  # Error swallowed — but transaction is now ABORTED

cur.execute("SELECT 1")
# ERROR: current transaction is aborted

# RIGHT: always rollback on error
try:
    cur.execute("INSERT INTO users (email) VALUES ('duplicate@email.com')")
    conn.commit()
except psycopg2.IntegrityError:
    conn.rollback()  # Reset transaction state
    # Handle the duplicate case
```

---

## 9. pg_stat_activity and pg_locks — Production Diagnosis

### 9.1 The Essential Monitoring Queries

**Overview of all active connections:**

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - state_change AS state_duration,
    now() - xact_start AS txn_duration,
    left(query, 120) AS current_query
FROM pg_stat_activity
WHERE pid != pg_backend_pid()
ORDER BY txn_duration DESC NULLS LAST;
```

**Find all blocked queries and what's blocking them:**

```sql
SELECT
    a.pid AS blocked_pid,
    a.usename AS blocked_user,
    a.application_name AS blocked_app,
    a.query AS blocked_query,
    a.state AS blocked_state,
    now() - a.query_start AS blocked_for,
    b.pid AS blocking_pid,
    b.usename AS blocking_user,
    b.query AS blocking_query,
    b.state AS blocking_state,
    now() - b.state_change AS blocking_state_duration
FROM pg_stat_activity a
JOIN pg_stat_activity b ON b.pid = ANY(pg_blocking_pids(a.pid))
WHERE cardinality(pg_blocking_pids(a.pid)) > 0
ORDER BY blocked_for DESC;
```

**Find idle-in-transaction connections older than N minutes:**

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    now() - state_change AS idle_duration,
    left(query, 120) AS last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - state_change > INTERVAL '5 minutes'
ORDER BY idle_duration DESC;
```

**See all locks currently held:**

```sql
SELECT
    l.pid,
    l.locktype,
    CASE l.locktype
        WHEN 'relation' THEN l.relation::regclass::text
        WHEN 'transactionid' THEN l.transactionid::text
        ELSE l.locktype
    END AS lock_target,
    l.mode,
    l.granted,
    a.query,
    a.state
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE a.pid != pg_backend_pid()
ORDER BY l.granted, l.pid;
```

**Lock wait chains (who is waiting for whom, recursively):**

```sql
WITH RECURSIVE lock_chain AS (
    -- Seed: directly blocked pids
    SELECT
        blocked.pid AS blocked_pid,
        blocking.pid AS blocking_pid,
        1 AS depth,
        ARRAY[blocked.pid] AS path
    FROM pg_stat_activity blocked
    JOIN pg_stat_activity blocking
        ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))

    UNION ALL

    -- Recurse: find what's blocking the blockers
    SELECT
        lc.blocking_pid AS blocked_pid,
        blocking.pid AS blocking_pid,
        lc.depth + 1,
        lc.path || lc.blocking_pid
    FROM lock_chain lc
    JOIN pg_stat_activity blocking
        ON blocking.pid = ANY(pg_blocking_pids(lc.blocking_pid))
    WHERE lc.blocking_pid != ALL(lc.path)  -- prevent infinite loops
      AND lc.depth < 10
)
SELECT DISTINCT ON (blocked_pid)
    blocked_pid,
    blocking_pid,
    depth,
    path
FROM lock_chain
ORDER BY blocked_pid, depth DESC;
```

### 9.2 How to Kill a Blocking Query Safely

```sql
-- Cancel a query (sends SIGINT — graceful, the transaction is rolled back)
SELECT pg_cancel_backend(pid);
-- The session remains connected; the query is cancelled; transaction rolls back

-- Terminate a connection (sends SIGTERM — harder kill)
SELECT pg_terminate_backend(pid);
-- The session is disconnected; transaction rolls back; connection must reconnect

-- Cancel all idle-in-transaction queries older than 10 minutes
SELECT pg_cancel_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - state_change > INTERVAL '10 minutes';

-- IMPORTANT: only superusers can terminate other superuser sessions.
-- pg_cancel_backend and pg_terminate_backend return false if you lack permission.
-- Check return value!
SELECT pid, pg_cancel_backend(pid) AS cancelled
FROM pg_stat_activity
WHERE state = 'idle in transaction';
```

### 9.3 Monitoring Lock Contention Over Time

```sql
-- pg_stat_activity.wait_event and wait_event_type tell you what a process is waiting for
-- Lock waits appear as: wait_event_type = 'Lock', wait_event = 'relation' or 'tuple'

-- Snapshot of current lock wait events
SELECT
    wait_event_type,
    wait_event,
    COUNT(*) AS waiting_sessions
FROM pg_stat_activity
WHERE wait_event_type IS NOT NULL
GROUP BY wait_event_type, wait_event
ORDER BY waiting_sessions DESC;
```

**Setting proactive timeouts to prevent production incidents:**

```sql
-- In postgresql.conf (global defaults):
lock_timeout = '30s'                        -- fail if can't get lock in 30s
statement_timeout = '30s'                   -- fail any statement taking > 30s
idle_in_transaction_session_timeout = '5min' -- kill idle-in-transaction after 5 min

-- Per-session (inside application connection setup):
SET lock_timeout = '10s';
SET statement_timeout = '20s';
SET idle_in_transaction_session_timeout = '2min';

-- Per-transaction (for specific risky operations):
BEGIN;
SET LOCAL lock_timeout = '5s';
ALTER TABLE users ADD COLUMN bio TEXT;
COMMIT;
```

### 9.4 Diagnosing Vacuum Bloat

```sql
-- Tables with the most dead tuples (VACUUM backlog)
SELECT
    schemaname || '.' || relname AS table,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Find the transaction preventing vacuum from advancing
SELECT
    pid,
    usename,
    now() - xact_start AS txn_age,
    state,
    left(query, 100) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start ASC
LIMIT 5;
-- The oldest transaction is your vacuum bloat culprit

-- Check the oldest transaction ID horizon
SELECT
    datname,
    age(datfrozenxid) AS txn_age_since_freeze,
    2000000000 - age(datfrozenxid) AS txns_until_wraparound
FROM pg_database
ORDER BY txn_age_since_freeze DESC;
-- If txns_until_wraparound < 50,000,000: run VACUUM FREEZE immediately
```

---

## 10. Decision Framework — Choosing the Right Strategy

### 10.1 The Core Decision Tree

```
START: I need to perform a database operation

1. Is the operation a single statement?
   YES → autocommit is fine; the statement is atomic.
   NO  → wrap in a transaction. Which isolation level?

2. Does my transaction read data that it then uses to decide what to write?
   NO  → READ COMMITTED is safe. Use it.
   YES → continue...

3. Does it matter if the data I read changes before I commit?
   NO  (e.g., logging, insert-only) → READ COMMITTED.
   YES → continue...

4. Does my logic depend on aggregate conditions across multiple rows
   (e.g., "ensure at least 1 doctor is on-call", "count doesn't exceed limit")?
   YES → SERIALIZABLE. Write skew can corrupt these invariants.
   NO  → continue...

5. Do I read the same row multiple times and need consistency?
   YES → REPEATABLE READ.
   NO  → READ COMMITTED with SELECT FOR UPDATE on the specific rows you need.
```

### 10.2 Isolation Level Selection Guide

| Scenario | Recommended Level | Reason |
|---|---|---|
| Simple CRUD (insert user, update profile) | READ COMMITTED | No cross-row logic, low contention |
| Read-only report that must be internally consistent | REPEATABLE READ | Prevents phantom/non-repeatable reads within the report |
| Financial transfer (debit account A, credit account B) | READ COMMITTED + SELECT FOR UPDATE | Explicit locking gives you control without full serialization overhead |
| "Check then act" on a single row (claim coupon, book seat) | READ COMMITTED + conditional UPDATE or FOR UPDATE | Atomic conditional update is sufficient |
| Multi-row invariant enforcement (on-call doctors, budget caps) | SERIALIZABLE | Only level that prevents write skew |
| Job queue (multiple workers) | READ COMMITTED + SKIP LOCKED | Explicit concurrent queue semantics |
| Bulk import with error tolerance | READ COMMITTED + savepoints | Partial rollback without aborting entire batch |
| Scheduled job (run once across multiple servers) | READ COMMITTED + advisory locks | Distributed mutex without table locking |

### 10.3 Locking Strategy Selection Guide

| You need to... | Use... |
|---|---|
| Claim a row exclusively before updating it | `SELECT ... FOR UPDATE` |
| Allow multiple readers but block writers | `SELECT ... FOR SHARE` |
| Update non-key columns (allow FK checks) | `SELECT ... FOR NO KEY UPDATE` |
| Fail immediately if row is locked | `... FOR UPDATE NOWAIT` |
| Skip locked rows, process available ones | `... FOR UPDATE SKIP LOCKED` |
| Prevent concurrent runs of a function | `pg_advisory_xact_lock()` |
| Prevent table-level DDL from conflicting | `LOCK TABLE ... IN SHARE MODE` |
| Ensure consistent lock order | Always sort row IDs before locking |

### 10.4 Anomaly Prevention Summary

| Problem | Wrong Approach | Right Approach |
|---|---|---|
| Double spend | SELECT then UPDATE in separate statements | Conditional UPDATE with balance check in WHERE, or SELECT FOR UPDATE |
| Overselling inventory | Application-level stock check | `UPDATE ... WHERE stock > 0` with RETURNING |
| Duplicate registration | App-level SELECT → INSERT | UNIQUE constraint (enforced by PostgreSQL, always) |
| Write skew on multi-row invariant | REPEATABLE READ | SERIALIZABLE |
| Job queue double-processing | SELECT → UPDATE in separate transactions | `SELECT FOR UPDATE SKIP LOCKED` inside a transaction |
| Long transaction blocking vacuum | Unbounded transaction scope | Statement timeouts, idle_in_transaction_session_timeout |
| Deadlock | Random lock acquisition order | Consistent lock ordering (sort by ID), or lock all rows upfront |

### 10.5 Production Configuration Baseline

```sql
-- Recommended postgresql.conf settings for a production OLTP workload:

-- Prevent runaway queries
statement_timeout = '30s'

-- Prevent DDL from hanging the world
lock_timeout = '10s'

-- Kill forgotten transactions
idle_in_transaction_session_timeout = '5min'

-- Detect deadlocks quickly
deadlock_timeout = '500ms'

-- Ensure durability (only disable for explicitly lossy workloads)
synchronous_commit = on

-- Per application connection setup (defense in depth):
SET statement_timeout = '30s';
SET lock_timeout = '10s';
SET idle_in_transaction_session_timeout = '3min';
```

### 10.6 The Mental Model in One Paragraph

PostgreSQL's concurrency control is built on MVCC: every write creates a new row version, every read sees a consistent snapshot, and reads never block writes. The isolation level controls which snapshot you see. `READ COMMITTED` gives you maximum concurrency but exposes you to non-repeatable reads and write skew — safe only when each statement is self-contained or when you use explicit locking (`FOR UPDATE`). `REPEATABLE READ` gives you a stable snapshot for the transaction's duration but still allows write skew across rows. `SERIALIZABLE` (SSI) is the only level that eliminates all anomalies — it does so by detecting read/write dependency cycles and aborting one party, not by blocking. The practical rule: use `READ COMMITTED` by default, use explicit `FOR UPDATE` locking for "claim then modify" patterns, and escalate to `SERIALIZABLE` only when your business invariant spans multiple rows and cannot be enforced by a constraint.

---

*Last updated: March 2026. Covers PostgreSQL 14+.*
