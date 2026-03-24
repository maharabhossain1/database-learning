# Database Migrations Guide
## Zero-Downtime Schema Changes and Production Safety

> "A migration that works in staging and destroys production is not a bug — it's a failure to understand locks, table sizes, and deployment order."

---

## Table of Contents

1. [Why Migrations Are Dangerous](#1-why-migrations-are-dangerous)
2. [PostgreSQL Lock Levels for DDL](#2-postgresql-lock-levels-for-ddl)
3. [Safe vs Unsafe Schema Changes](#3-safe-vs-unsafe-schema-changes)
4. [Zero-Downtime Migration Patterns](#4-zero-downtime-migration-patterns)
5. [Backfilling Large Tables](#5-backfilling-large-tables)
6. [Django Migrations Deep Dive](#6-django-migrations-deep-dive)
7. [Migration Anti-Patterns That Cause Outages](#7-migration-anti-patterns-that-cause-outages)
8. [Production Migration Workflow](#8-production-migration-workflow)
9. [Rollback Strategies](#9-rollback-strategies)

---

## 1. Why Migrations Are Dangerous

### 1.1 The Lock Problem

Every DDL statement in PostgreSQL acquires a lock. Some locks are harmless. Some block every read and write on the table for the entire duration of the operation.

On a table with 10 million rows, `ALTER TABLE orders ADD COLUMN notes TEXT DEFAULT ''` can lock the table for **minutes**. During those minutes, every `INSERT`, `UPDATE`, `SELECT` on that table queues up — and your application returns errors or times out.

```sql
-- This seems innocent. On 50M rows it takes 3+ minutes and holds ACCESS EXCLUSIVE lock.
ALTER TABLE orders ADD COLUMN notes TEXT DEFAULT '';

-- Meanwhile, every query hitting orders is blocked:
-- SELECT * FROM orders WHERE user_id = 123;  ← BLOCKED
-- INSERT INTO orders (...) VALUES (...);     ← BLOCKED
-- UPDATE orders SET status = 'shipped' ...;  ← BLOCKED
```

### 1.2 The Lock Queue Cascade

The lock doesn't just block queries that arrive after the DDL. It causes a queue. Here's why it's worse than you think:

1. Long-running `SELECT` holds `ACCESS SHARE` lock on orders
2. Your `ALTER TABLE` arrives — needs `ACCESS EXCLUSIVE` — waits for the SELECT to finish
3. While the ALTER waits, **every new query** that wants any lock on orders also waits (behind the ALTER)
4. Within seconds, your connection pool is exhausted

```sql
-- See this happening in real time:
SELECT pid, query, state, wait_event_type, wait_event, query_start
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
ORDER BY query_start;
```

### 1.3 Table Rewrites

Some operations don't just lock — they physically rewrite the entire table:

| Operation | Rewrites Table? | Lock Level |
|-----------|----------------|------------|
| `ADD COLUMN` with volatile default | Yes (pre-PG11) | ACCESS EXCLUSIVE |
| `ADD COLUMN NOT NULL` with default (constant) | No (PG11+) | ACCESS EXCLUSIVE |
| `ALTER COLUMN TYPE` (incompatible) | Yes | ACCESS EXCLUSIVE |
| `ALTER COLUMN TYPE` varchar(10)→varchar(20) | No | ACCESS EXCLUSIVE |
| `ADD CONSTRAINT CHECK NOT VALID` | No | SHARE UPDATE EXCLUSIVE |
| `VALIDATE CONSTRAINT` | No rewrite, full scan | SHARE UPDATE EXCLUSIVE |
| `ADD PRIMARY KEY` | Yes (if no existing index) | ACCESS EXCLUSIVE |
| `CLUSTER` | Yes | ACCESS EXCLUSIVE |

### 1.4 The Deployment Order Problem

Your app code and database schema must be compatible during the deployment window. If you have rolling deploys (some instances on old code, some on new):

- **Adding a column**: safe to add DB-side first, old code ignores new column
- **Removing a column**: you MUST remove from code first, then drop from DB
- **Renaming a column**: the hardest case — old and new code expect different names simultaneously

---

## 2. PostgreSQL Lock Levels for DDL

### 2.1 The Lock Hierarchy

PostgreSQL has 8 lock levels (weakest → strongest):

```
ACCESS SHARE           ← SELECT holds this
ROW SHARE              ← SELECT FOR UPDATE holds this
ROW EXCLUSIVE          ← INSERT, UPDATE, DELETE hold this
SHARE UPDATE EXCLUSIVE ← VACUUM, CREATE INDEX CONCURRENTLY
SHARE                  ← CREATE INDEX (non-concurrent)
SHARE ROW EXCLUSIVE    ← CREATE TRIGGER
EXCLUSIVE              ← rarely used directly
ACCESS EXCLUSIVE       ← ALTER TABLE, DROP TABLE, TRUNCATE
```

**Key rule**: `ACCESS EXCLUSIVE` conflicts with **everything**. One `ALTER TABLE` blocks all readers and writers.

### 2.2 Which DDL Takes Which Lock

```sql
-- ACCESS EXCLUSIVE (most dangerous — blocks everything):
ALTER TABLE t ADD COLUMN c TEXT;
ALTER TABLE t DROP COLUMN c;
ALTER TABLE t ALTER COLUMN c TYPE bigint;
ALTER TABLE t ADD CONSTRAINT c PRIMARY KEY (id);
DROP TABLE t;
TRUNCATE t;
LOCK TABLE t;

-- SHARE UPDATE EXCLUSIVE (safer — only blocks other schema changes):
CREATE INDEX CONCURRENTLY ON t (col);
VACUUM t;
ALTER TABLE t VALIDATE CONSTRAINT c;
ALTER TABLE t SET (autovacuum_enabled = false);

-- SHARE (blocks writes, allows reads):
CREATE INDEX ON t (col);  -- non-concurrent!
```

### 2.3 lock_timeout — Your Safety Net

Always set `lock_timeout` before running DDL in production. If the lock can't be acquired within N milliseconds, the statement fails with an error instead of queueing indefinitely.

```sql
-- Set for this session only (always do this before production DDL):
SET lock_timeout = '2s';
SET statement_timeout = '30s';

-- Now run your DDL. If it can't get the lock in 2s, it errors out
-- instead of blocking your entire connection pool.
ALTER TABLE orders ADD COLUMN notes TEXT;

-- Reset after:
RESET lock_timeout;
RESET statement_timeout;
```

> **Rule**: Never run DDL in production without `lock_timeout`. The worst outcome is an ALTER that waits 10 minutes while your app melts down. A failed migration is recoverable. A hung migration that takes down production is not.

---

## 3. Safe vs Unsafe Schema Changes

### 3.1 Adding Columns

```sql
-- SAFE: nullable column, no default — instant, no rewrite
ALTER TABLE users ADD COLUMN bio TEXT;

-- SAFE on PostgreSQL 11+: NOT NULL with constant default — instant, no rewrite
-- PostgreSQL stores the default in catalog, doesn't rewrite rows
ALTER TABLE users ADD COLUMN is_active BOOLEAN NOT NULL DEFAULT true;

-- DANGEROUS: volatile default — rewrites entire table (even on PG11+)
ALTER TABLE users ADD COLUMN created_at TIMESTAMPTZ DEFAULT NOW();
--  ^ NOW() is volatile — PG must call it per-row, forces full rewrite

-- RIGHT WAY for volatile defaults:
ALTER TABLE users ADD COLUMN created_at TIMESTAMPTZ;  -- add nullable first
UPDATE users SET created_at = NOW() WHERE created_at IS NULL;  -- backfill (in batches!)
ALTER TABLE users ALTER COLUMN created_at SET NOT NULL;  -- add constraint after
ALTER TABLE users ALTER COLUMN created_at SET DEFAULT NOW();  -- add default after
```

### 3.2 Removing Columns

Never drop a column while code still reads/writes it.

```sql
-- Phase 1: Remove column from all application code, deploy.
-- (At this point, DB still has the column — old deploys won't break)

-- Phase 2: After all instances are on new code, drop from DB:
ALTER TABLE users DROP COLUMN legacy_field;

-- If you want to be extra safe, first mark it unused:
-- In PostgreSQL you can't "hide" a column, but you can check access first:
SELECT count(*) FROM pg_stats
WHERE tablename = 'users' AND attname = 'legacy_field';
-- Then drop only after confirming no app queries reference it
```

### 3.3 Adding Indexes

```sql
-- DANGEROUS: blocks all writes for the entire index build duration
CREATE INDEX ON orders (user_id);

-- SAFE: CONCURRENTLY — builds index without blocking writes
-- Takes longer, but app stays live
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id);

-- If CONCURRENTLY fails partway, it leaves an INVALID index:
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'orders';
-- If you see an index marked INVALID:

SELECT indexrelid::regclass, indisvalid
FROM pg_index
WHERE indrelid = 'orders'::regclass AND NOT indisvalid;

-- Clean up and retry:
DROP INDEX CONCURRENTLY idx_orders_user_id;
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id);
```

> **Critical**: `CREATE INDEX CONCURRENTLY` cannot run inside a transaction block. Django's `atomic=False` is required. See Section 6.

### 3.4 Adding Constraints

#### Foreign Keys — the NOT VALID pattern

```sql
-- DANGEROUS: scans entire table with lock to verify constraint
ALTER TABLE orders ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES users(id);

-- SAFE: two-step pattern
-- Step 1: Add constraint without validating existing rows (fast, minimal lock)
ALTER TABLE orders ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES users(id)
    NOT VALID;

-- Step 2: Validate in background (SHARE UPDATE EXCLUSIVE — allows reads/writes)
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_user;
```

#### CHECK Constraints — same pattern

```sql
-- DANGEROUS: full table scan with lock
ALTER TABLE products ADD CONSTRAINT chk_price_positive CHECK (price > 0);

-- SAFE: NOT VALID + VALIDATE
ALTER TABLE products ADD CONSTRAINT chk_price_positive CHECK (price > 0) NOT VALID;
ALTER TABLE products VALIDATE CONSTRAINT chk_price_positive;
```

#### NOT NULL — the tricky one

```sql
-- On PostgreSQL 12+: use a CHECK constraint as proxy for large tables
-- Step 1: Add CHECK NOT VALID (no lock, no scan)
ALTER TABLE users ADD CONSTRAINT chk_email_not_null CHECK (email IS NOT NULL) NOT VALID;

-- Step 2: Validate (SHARE UPDATE EXCLUSIVE, allows reads/writes)
ALTER TABLE users VALIDATE CONSTRAINT chk_email_not_null;

-- Step 3: Now add the actual NOT NULL (PostgreSQL is smart enough to use the constraint)
-- On PG12+, this is fast because the constraint already guarantees it
ALTER TABLE users ALTER COLUMN email SET NOT NULL;

-- Step 4: Drop the now-redundant CHECK
ALTER TABLE users DROP CONSTRAINT chk_email_not_null;
```

### 3.5 Renaming Columns

A direct rename locks the table and breaks all code that uses the old name simultaneously.

```sql
-- WRONG: direct rename breaks running app instances
ALTER TABLE users RENAME COLUMN username TO handle;

-- RIGHT: two-phase zero-downtime rename
-- Phase 1: Add new column
ALTER TABLE users ADD COLUMN handle TEXT;

-- Phase 2: Keep in sync with a trigger
CREATE OR REPLACE FUNCTION sync_username_handle()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' OR TG_OP = 'UPDATE' THEN
        NEW.handle = NEW.username;
        NEW.username = NEW.handle;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_username_handle
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION sync_username_handle();

-- Phase 3: Backfill (in batches — see Section 5)
UPDATE users SET handle = username WHERE handle IS NULL;

-- Phase 4: Deploy new code using `handle` column

-- Phase 5: Drop trigger and old column
DROP TRIGGER trg_sync_username_handle ON users;
DROP FUNCTION sync_username_handle();
ALTER TABLE users DROP COLUMN username;
```

### 3.6 Changing Column Types

Almost always a full table rewrite. Safe exceptions:

```sql
-- SAFE: increasing varchar length (no rewrite)
ALTER TABLE users ALTER COLUMN username TYPE VARCHAR(255);

-- SAFE: int → bigint (no rewrite in PostgreSQL 14+)
-- UNSAFE: bigint → int (lossy, rewrite)

-- UNSAFE: text → integer (requires cast — full rewrite)
ALTER TABLE events ALTER COLUMN count TYPE BIGINT USING count::bigint;
--  ^ USING clause = full table rewrite with ACCESS EXCLUSIVE lock

-- SAFE pattern for type change on large table:
-- 1. Add new column with correct type
ALTER TABLE events ADD COLUMN count_new BIGINT;
-- 2. Dual-write trigger (same as rename pattern)
-- 3. Backfill
UPDATE events SET count_new = count::bigint WHERE count_new IS NULL;
-- 4. Deploy code using new column
-- 5. Drop old column
ALTER TABLE events DROP COLUMN count;
ALTER TABLE events RENAME COLUMN count_new TO count;
```

---

## 4. Zero-Downtime Migration Patterns

### 4.1 The Expand/Contract Pattern

The fundamental pattern for all zero-downtime schema changes:

```
Phase 1 — EXPAND (database change, backwards compatible):
  - Add new structure (column, table, index)
  - Old app code still works fine

Phase 2 — MIGRATE (data + code change):
  - Backfill data into new structure
  - Deploy new app code that uses new structure
  - Both old and new code must work simultaneously during rolling deploy

Phase 3 — CONTRACT (cleanup):
  - Remove old structure (after all app instances are on new code)
  - Drop old column, drop trigger, etc.
```

### 4.2 Applied Example: Splitting a Column

**Scenario**: Split `full_name TEXT` into `first_name TEXT` and `last_name TEXT`.

```sql
-- EXPAND: add new columns
ALTER TABLE users ADD COLUMN first_name TEXT;
ALTER TABLE users ADD COLUMN last_name TEXT;

-- Keep in sync with trigger during transition:
CREATE OR REPLACE FUNCTION sync_name_split()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.full_name IS NOT NULL AND NEW.first_name IS NULL THEN
        NEW.first_name = split_part(NEW.full_name, ' ', 1);
        NEW.last_name = substring(NEW.full_name from position(' ' in NEW.full_name) + 1);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_name
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION sync_name_split();
```

```sql
-- MIGRATE: backfill existing rows (in batches — see Section 5)
UPDATE users
SET
    first_name = split_part(full_name, ' ', 1),
    last_name = substring(full_name from position(' ' in full_name) + 1)
WHERE first_name IS NULL AND full_name IS NOT NULL;

-- Deploy new app code that writes first_name/last_name
-- and reads from first_name/last_name
```

```sql
-- CONTRACT: after all app instances on new code
DROP TRIGGER trg_sync_name ON users;
DROP FUNCTION sync_name_split();
ALTER TABLE users DROP COLUMN full_name;
```

### 4.3 Backwards Compatibility Rules

| Change Type | Deploy Order |
|-------------|-------------|
| Add column | DB first, then code |
| Drop column | Code first (stop using it), then DB |
| Add table | DB first, then code |
| Drop table | Code first (stop referencing it), then DB |
| Add index | DB anytime (doesn't affect app behavior) |
| Rename column | Use expand/contract — never direct rename in production |
| Change type | Use expand/contract — never direct type change on large table |

---

## 5. Backfilling Large Tables

### 5.1 Why Bulk UPDATE Kills Production

```sql
-- NEVER do this on a large table in production:
UPDATE users SET handle = username;
-- On 10M rows: holds row locks for minutes, floods WAL,
-- spikes replication lag, can exhaust memory
```

### 5.2 Batched Backfill Pattern

```sql
-- Safe batched backfill using id ranges
DO $$
DECLARE
    batch_size INT := 5000;
    min_id BIGINT;
    max_id BIGINT;
    current_id BIGINT;
    rows_updated INT;
BEGIN
    SELECT MIN(id), MAX(id) INTO min_id, max_id FROM users;
    current_id := min_id;

    WHILE current_id <= max_id LOOP
        UPDATE users
        SET handle = username
        WHERE id >= current_id
          AND id < current_id + batch_size
          AND handle IS NULL;

        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        RAISE NOTICE 'Updated % rows, current_id = %', rows_updated, current_id;

        current_id := current_id + batch_size;

        -- Pause to let replication catch up and reduce WAL pressure
        PERFORM pg_sleep(0.1);
    END LOOP;

    RAISE NOTICE 'Backfill complete';
END $$;
```

### 5.3 Monitoring Backfill Progress

```sql
-- Check how many rows still need backfilling:
SELECT
    COUNT(*) FILTER (WHERE handle IS NULL) AS remaining,
    COUNT(*) FILTER (WHERE handle IS NOT NULL) AS done,
    COUNT(*) AS total,
    ROUND(100.0 * COUNT(*) FILTER (WHERE handle IS NOT NULL) / COUNT(*), 1) AS pct_done
FROM users;

-- Monitor replication lag during backfill:
SELECT client_addr,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

> **Tune the batch size**: Start with 1000, watch replication lag and query latency. If lag grows, reduce batch size or increase sleep. If lag is fine, increase batch size to finish faster.

---

## 6. Django Migrations Deep Dive

### 6.1 How Django Migrations Work

Django tracks applied migrations in the `django_migrations` table. Each migration file has dependencies, and Django builds a DAG to determine order.

```python
# django_migrations table:
# id | app    | name                    | applied
# 1  | users  | 0001_initial            | 2024-01-01
# 2  | users  | 0002_add_handle_column  | 2024-01-15
```

```bash
# Generate a migration:
python manage.py makemigrations

# Apply migrations:
python manage.py migrate

# Show migration plan:
python manage.py migrate --plan

# Show all migrations and status:
python manage.py showmigrations
```

### 6.2 RunSQL for Custom Operations

```python
from django.db import migrations

class Migration(migrations.Migration):
    dependencies = [('users', '0002_add_handle_column')]

    operations = [
        # Raw SQL for things Django can't express:
        migrations.RunSQL(
            sql="""
                ALTER TABLE users
                ADD CONSTRAINT chk_handle_not_empty
                CHECK (handle <> '') NOT VALID;
            """,
            reverse_sql="ALTER TABLE users DROP CONSTRAINT chk_handle_not_empty;"
        ),
        migrations.RunSQL(
            sql="ALTER TABLE users VALIDATE CONSTRAINT chk_handle_not_empty;",
            reverse_sql=migrations.RunSQL.noop,
        ),
    ]
```

### 6.3 atomic=False for CONCURRENTLY

`CREATE INDEX CONCURRENTLY` cannot run inside a transaction. Django wraps each migration in a transaction by default. You must disable this:

```python
from django.db import migrations

class Migration(migrations.Migration):
    # CRITICAL: disables the wrapping transaction for this migration
    atomic = False

    dependencies = [('orders', '0005_add_status_column')]

    operations = [
        migrations.RunSQL(
            sql="""
                CREATE INDEX CONCURRENTLY IF NOT EXISTS
                idx_orders_user_id_status
                ON orders (user_id, status)
                WHERE status != 'completed';
            """,
            reverse_sql="DROP INDEX CONCURRENTLY IF EXISTS idx_orders_user_id_status;"
        ),
    ]
```

> **Warning**: With `atomic=False`, if the migration fails partway, it won't be rolled back. Check for `INVALID` indexes after a failed run and clean them up manually.

### 6.4 SeparateDatabaseAndState

Use when you want the database schema and Django's understanding of it to be decoupled. Critical for zero-downtime renames.

**Scenario**: Rename `username` to `handle` at the DB level while Django still thinks it's `username` during the transition period.

```python
# Migration 1: Add new column + trigger (DB-only, Django doesn't know about handle yet)
class Migration(migrations.Migration):
    atomic = False
    operations = [
        migrations.SeparateDatabaseAndState(
            database_operations=[
                migrations.RunSQL(
                    "ALTER TABLE users ADD COLUMN handle TEXT;",
                    reverse_sql="ALTER TABLE users DROP COLUMN handle;"
                ),
                migrations.RunSQL(
                    """
                    CREATE OR REPLACE FUNCTION sync_handle() RETURNS TRIGGER AS $$
                    BEGIN NEW.handle = NEW.username; RETURN NEW; END;
                    $$ LANGUAGE plpgsql;

                    CREATE TRIGGER trg_sync_handle
                        BEFORE INSERT OR UPDATE ON users
                        FOR EACH ROW EXECUTE FUNCTION sync_handle();
                    """,
                    reverse_sql="""
                    DROP TRIGGER trg_sync_handle ON users;
                    DROP FUNCTION sync_handle();
                    """
                ),
            ],
            state_operations=[],  # Django state unchanged — still sees 'username'
        ),
    ]
```

```python
# Migration 2: Tell Django about the rename (no DB change needed — column already exists)
class Migration(migrations.Migration):
    operations = [
        migrations.SeparateDatabaseAndState(
            database_operations=[],  # DB already has both columns
            state_operations=[
                migrations.RenameField('User', 'username', 'handle'),
            ],
        ),
    ]
```

### 6.5 Fake Migrations

Use when the DB already has the schema but Django doesn't know about it:

```bash
# Mark a migration as applied without running it:
python manage.py migrate users 0003_add_handle_column --fake

# Mark all migrations as applied (for a DB you created manually):
python manage.py migrate --fake-initial
```

---

## 7. Migration Anti-Patterns That Cause Outages

### 7.1 No lock_timeout

```python
# WRONG: This can queue indefinitely if a long transaction holds a lock
migrations.RunSQL("ALTER TABLE orders ADD COLUMN notes TEXT;")

# RIGHT:
migrations.RunSQL("""
    SET lock_timeout = '3s';
    ALTER TABLE orders ADD COLUMN notes TEXT;
    RESET lock_timeout;
""")
```

### 7.2 CREATE INDEX without CONCURRENTLY on a Live Table

```python
# WRONG: blocks all writes during index build (minutes on large table)
migrations.AddIndex(
    model_name='Order',
    index=models.Index(fields=['user_id'], name='idx_orders_user_id'),
)
# Django's AddIndex uses CREATE INDEX (not CONCURRENTLY)

# RIGHT: use atomic=False + RunSQL
migrations.RunSQL(
    "CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders (user_id);",
    # migration must have atomic = False
)
```

### 7.3 Data Migration in a Transaction on a Large Table

```python
# WRONG: locks rows for the entire duration of the update
class Migration(migrations.Migration):
    def populate_handle(apps, schema_editor):
        User = apps.get_model('users', 'User')
        User.objects.filter(handle='').update(handle=F('username'))
        # On 5M rows: holds locks for minutes inside a transaction

    operations = [migrations.RunPython(populate_handle)]
```

```python
# RIGHT: batched, outside of a tight transaction loop
class Migration(migrations.Migration):
    atomic = False  # no wrapping transaction

    def populate_handle(apps, schema_editor):
        from django.db import connection
        with connection.cursor() as cursor:
            while True:
                cursor.execute("""
                    UPDATE users SET handle = username
                    WHERE id IN (
                        SELECT id FROM users
                        WHERE handle IS NULL OR handle = ''
                        LIMIT 5000
                    )
                """)
                if cursor.rowcount == 0:
                    break

    operations = [migrations.RunPython(populate_handle, migrations.RunPython.noop)]
```

### 7.4 Dropping Column Before Removing from Code

```python
# WRONG order:
# Step 1: migration drops the column
# Step 2: deploy new code
# Problem: during deploy, old app instances try to SELECT/INSERT the dropped column → crash

# RIGHT order:
# Step 1: deploy new code that doesn't reference the column
# Step 2: migration drops the column (now safe)
```

### 7.5 Mixing Schema Change and Data Migration

```python
# WRONG: one migration does both — can't partial-rollback, hard to diagnose
operations = [
    migrations.AddColumn('Order', 'total_cents', ...),
    migrations.RunPython(backfill_total_cents),  # modifies millions of rows
    migrations.AddConstraint('Order', NOT NULL on total_cents),
]

# RIGHT: separate migrations
# Migration A: add column (fast, safe)
# Migration B: backfill data (separate deploy, can monitor progress)
# Migration C: add constraint (after backfill verified)
```

---

## 8. Production Migration Workflow

### 8.1 Pre-Migration Checklist

```sql
-- 1. Check table size before migrating:
SELECT
    relname AS table,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    n_live_tup AS approx_rows
FROM pg_stat_user_tables
WHERE relname = 'orders';

-- 2. Check for long-running transactions (can cause lock waits):
SELECT pid, now() - xact_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start IS NOT NULL
  AND now() - xact_start > interval '30 seconds'
ORDER BY duration DESC;

-- 3. Check existing locks on the table:
SELECT pid, mode, granted, query
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
WHERE relation = 'orders'::regclass;

-- 4. Set safety timeouts:
SET lock_timeout = '3s';
SET statement_timeout = '60s';
```

### 8.2 During Migration — Monitoring

```sql
-- Watch for blocking queries during your migration:
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    now() - blocked.query_start AS wait_duration
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
ORDER BY wait_duration DESC;

-- If your migration is hanging and blocking everything, kill it:
SELECT pg_cancel_backend(<your_migration_pid>);
-- If cancel doesn't work:
SELECT pg_terminate_backend(<your_migration_pid>);
```

### 8.3 Post-Migration Validation

```sql
-- Check for INVALID indexes:
SELECT schemaname, tablename, indexname, indexdef
FROM pg_indexes
WHERE tablename = 'orders'
  AND indexname IN (
      SELECT indexrelid::regclass::text
      FROM pg_index
      WHERE NOT indisvalid
  );

-- Verify constraint was created:
SELECT conname, contype, convalidated
FROM pg_constraint
WHERE conrelid = 'orders'::regclass;

-- Verify column exists and has correct type:
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_name = 'orders'
ORDER BY ordinal_position;
```

---

## 9. Rollback Strategies

### 9.1 What's Easy to Rollback

```sql
-- Add column → drop column (fast)
ALTER TABLE users DROP COLUMN handle;

-- Add index → drop index (fast with CONCURRENTLY)
DROP INDEX CONCURRENTLY idx_orders_user_id;

-- Add constraint NOT VALID → drop constraint (fast)
ALTER TABLE users DROP CONSTRAINT chk_handle_not_empty;
```

### 9.2 What's Hard to Rollback

| Operation | Rollback Difficulty | Reason |
|-----------|--------------------|-|
| Drop column | Hard | Data is gone |
| Type change with data transform | Hard | Original data may be lost |
| Large data migration | Hard | Partial state, reverse migration needed |
| Drop table | Very hard | Requires backup restore |

For hard-to-rollback migrations:

```bash
# Always take a backup immediately before:
pg_dump -h localhost -U postgres -Fc mydb > backup_before_migration_$(date +%Y%m%d_%H%M%S).dump

# Know your PITR (Point-In-Time Recovery) window
# If using WAL archiving, you can restore to any point in time
```

### 9.3 Django Rollback

```bash
# Roll back to a specific migration:
python manage.py migrate users 0003

# Roll back one migration:
python manage.py migrate users 0003  # if you're on 0004, this rolls back 0004

# Check what a rollback would do first:
python manage.py migrate users 0003 --plan
```

> **Note**: Only reversible if you've defined `reverse_sql` in `RunSQL` or a `reverse` function in `RunPython`. Always define reversal logic.

### 9.4 Point-in-Time Recovery as Ultimate Fallback

```bash
# If WAL archiving is set up, restore to 1 minute before migration:
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2024-03-24 14:30:00'  # 1 minute before migration started
recovery_target_action = 'promote'
```

> This is your nuclear option. It means losing any data written after the migration started. Know your RPO (Recovery Point Objective) before deciding to use it.

---

## Quick Reference: Migration Decision Tree

```
Is the table > 1M rows?
├── YES: Are you adding an index?
│   ├── YES → CREATE INDEX CONCURRENTLY (atomic=False in Django)
│   └── NO: Are you changing column type or adding NOT NULL?
│       ├── YES → Use expand/contract pattern (multi-phase migration)
│       └── NO: Are you adding a FK/CHECK constraint?
│           ├── YES → Use NOT VALID + VALIDATE CONSTRAINT pattern
│           └── NO → Set lock_timeout, proceed with care
└── NO: Still set lock_timeout, but can proceed more directly
```

```
Are you removing something (column, table, constraint)?
├── YES → Remove from application code FIRST, deploy, then run DB migration
└── NO → DB migration can usually go first (if additive and backwards-compatible)
```
