# PostgreSQL Internals and Architecture
**How PostgreSQL Actually Works Under the Hood**

> Understanding PostgreSQL at the process, memory, and storage level is what separates engineers who tune databases from engineers who guess at them. This guide builds that understanding — no abstractions, no hand-waving.

---

## Table of Contents

1. [Process Architecture](#1-process-architecture)
2. [Memory Architecture](#2-memory-architecture)
3. [WAL — Write-Ahead Log](#3-wal--write-ahead-log)
4. [MVCC Deep Dive](#4-mvcc-deep-dive)
5. [Vacuum and Autovacuum](#5-vacuum-and-autovacuum)
6. [Query Planner and EXPLAIN](#6-query-planner-and-explain)
7. [Table Storage Internals](#7-table-storage-internals)
8. [PostgreSQL Configuration](#8-postgresql-configuration)
9. [Monitoring PostgreSQL Internals](#9-monitoring-postgresql-internals)
10. [The Complete Mental Model](#10-the-complete-mental-model)

---

## 1. Process Architecture

### 1.1 The Postmaster: PostgreSQL's Parent Process

When you start PostgreSQL, a single process called the **postmaster** (or `postgres` on modern systems) starts first. It owns the shared memory segment, the listening socket, and the WAL writer. Everything else is its child.

```bash
# See the process tree yourself
ps aux | grep postgres

# Typical output:
# postgres  1234  ... postgres -D /var/lib/postgresql/14/main
# postgres  1235  ... postgres: checkpointer
# postgres  1236  ... postgres: background writer
# postgres  1237  ... postgres: walwriter
# postgres  1238  ... postgres: autovacuum launcher
# postgres  1239  ... postgres: stats collector
# postgres  1240  ... postgres: logical replication launcher
# postgres  1350  ... postgres: app_user mydb 10.0.0.5(52341) idle
# postgres  1351  ... postgres: app_user mydb 10.0.0.5(52342) SELECT
```

The postmaster's jobs:
- Listen on the TCP port (default 5432) and Unix socket
- Authenticate incoming connections
- Fork a new backend process for each accepted connection
- Restart crashed background workers
- Handle signals (SIGHUP for config reload, SIGTERM for smart shutdown, SIGINT for fast shutdown, SIGQUIT for immediate shutdown)

If the postmaster dies, all backends die with it. If a backend crashes, the postmaster detects it and initiates crash recovery — it restarts all remaining backends to ensure they see a consistent state.

### 1.2 Backend Processes: One Per Connection

Every client connection gets its own OS process — not a thread, a full process. This is the fork-per-connection model.

```
Client ──TCP──► Postmaster ──fork()──► Backend Process
                                            │
                                    Parses query
                                    Plans query
                                    Executes query
                                    Returns results
                                    Dies on disconnect
```

**Why this matters for you:**
- Each backend process uses ~5–10 MB of memory just for its own structures (catalog caches, planner structures, etc.)
- 500 connections = ~2.5–5 GB of RAM consumed just in per-backend overhead, before any work_mem is counted
- Each backend needs a slot in the shared memory lock table
- The OS itself pays a cost: context switches between hundreds of processes are expensive

```sql
-- See all current backends
SELECT pid, usename, application_name, client_addr, state, query_start, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;
```

### 1.3 Background Workers

These processes run continuously regardless of client connections:

**Checkpointer**
Writes dirty shared buffer pages to disk at checkpoint intervals. Checkpoints ensure that after a crash, PostgreSQL only needs to replay WAL from the last checkpoint forward. Without the checkpointer, recovery would require replaying all WAL from the beginning of time.

**Background Writer (bgwriter)**
Proactively writes dirty buffers to disk *between* checkpoints. This spreads the I/O load so checkpoints don't cause sudden disk spikes. It uses the `bgwriter_lru_maxpages` and `bgwriter_delay` settings to pace itself.

**WAL Writer**
Flushes the WAL buffer to the WAL segment files on disk. It runs on a tight loop (`wal_writer_delay`, default 200ms) and also flushes when a transaction commits. WAL is always written before the data pages that reference it — this is the guarantee.

**Autovacuum Launcher**
Monitors tables and launches autovacuum worker processes when tables accumulate enough dead tuples. It does not do vacuum work itself — it manages a pool of workers.

**Stats Collector**
Receives statistics from backends and writes them to shared memory and files. Every time you query `pg_stat_user_tables` or `pg_stat_activity`, you are reading what the stats collector has gathered. It communicates over UDP to avoid blocking the backends that send it data.

**Logical Replication Launcher**
Manages logical replication workers if you have logical publications configured.

### 1.4 Why Connection Count Matters

```
connections = 10:
  per-backend memory: ~100 MB
  lock table pressure: low
  scheduler overhead: negligible

connections = 200:
  per-backend memory: ~1-2 GB
  lock table pressure: moderate
  scheduler overhead: noticeable

connections = 1000:
  per-backend memory: ~5-10 GB
  lock table pressure: significant
  scheduler overhead: severe
  max_connections default (100) already exceeded — you need pgBouncer
```

The `max_connections` parameter reserves space in shared memory for that many backends (lock slots, proc array entries). Setting it to 1000 when you normally have 50 connections wastes shared memory permanently.

```sql
-- Check current connection usage
SELECT count(*) AS total_connections,
       count(*) FILTER (WHERE state = 'idle') AS idle,
       count(*) FILTER (WHERE state = 'active') AS active,
       count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_txn
FROM pg_stat_activity
WHERE pid != pg_backend_pid();

-- Check connection limit
SHOW max_connections;
SELECT count(*) FROM pg_stat_activity;
```

### 1.5 PgBouncer: Why It Exists

PgBouncer sits between your application and PostgreSQL and maintains a pool of real PostgreSQL connections, multiplexing many application connections onto fewer real connections.

```
App Servers (500 threads)
       │
       ▼
   PgBouncer
  (pool of 20 real connections)
       │
       ▼
  PostgreSQL
  (20 backends instead of 500)
```

PgBouncer modes:
- **Session mode**: Application gets a server connection for its entire session. Not much help.
- **Transaction mode**: Application gets a server connection only for the duration of a transaction. This is the useful mode — 500 app threads sharing 20 backend processes. Incompatible with `SET`, advisory locks, and prepared statements.
- **Statement mode**: Connection returned after every statement. Very restrictive — breaks almost all real applications.

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
reserve_pool_size = 5
reserve_pool_timeout = 3
server_idle_timeout = 600
```

> **Production decision**: Use PgBouncer in transaction mode as soon as your connection count exceeds ~100. This is not optional for web applications at any real scale. Supabase, RDS, and every managed PostgreSQL service puts a pooler in front for this reason.

---

## 2. Memory Architecture

### 2.1 Shared Buffers: The Database Page Cache

`shared_buffers` is PostgreSQL's own page cache — a region of shared memory that all backend processes access simultaneously. When PostgreSQL needs to read a data page, it first checks shared buffers. Cache hit means no disk I/O. Cache miss means reading from the OS page cache or disk.

```
Query needs page 7 of table "orders"
         │
         ▼
Is page in shared_buffers? ──YES──► Use it directly (microseconds)
         │
        NO
         │
         ▼
Is page in OS page cache? ──YES──► Copy into shared_buffers, use it (~milliseconds)
         │
        NO
         │
         ▼
Read from disk ──────────────────► Copy into shared_buffers, use it (~10s of milliseconds)
```

**Sizing shared_buffers:**

The canonical starting point is 25% of RAM. But the reasoning matters more than the rule.

PostgreSQL relies on the OS page cache as a second-level cache. The OS cache sits between shared_buffers and disk. Setting shared_buffers to 75% of RAM starves the OS cache and can actually hurt performance.

```
Server RAM: 64 GB
  shared_buffers = 16 GB    (25%)
  OS page cache: ~40 GB available
  work_mem + other: ~8 GB
```

If your working set (the pages your queries actually touch frequently) is smaller than 16 GB on a 64 GB server, even 16 GB of shared_buffers is plenty. The OS page cache will hold the rest.

```sql
-- Install pg_buffercache to inspect what's actually in shared_buffers
CREATE EXTENSION pg_buffercache;

-- What tables are consuming your shared_buffers?
SELECT c.relname,
       count(*) AS buffers,
       count(*) * 8 / 1024 AS size_mb,
       round(100.0 * count(*) / (SELECT count(*) FROM pg_buffercache), 1) AS pct
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
JOIN pg_database d ON b.reldatabase = d.oid
WHERE d.datname = current_database()
  AND b.usagecount > 0
GROUP BY c.relname
ORDER BY buffers DESC
LIMIT 20;

-- Overall buffer utilization
SELECT count(*) AS total_buffers,
       count(*) FILTER (WHERE usagecount > 0) AS used_buffers,
       count(*) FILTER (WHERE usagecount = 0) AS free_buffers,
       round(100.0 * count(*) FILTER (WHERE usagecount > 0) / count(*), 1) AS utilization_pct
FROM pg_buffercache;
```

If utilization is near 100%, your working set is larger than shared_buffers. Either increase shared_buffers or accept more I/O.

**Buffer replacement policy**: PostgreSQL uses a clock-sweep algorithm (an approximation of LRU). Each buffer has a usage count (0–5). On access, usage count increments. When a new page needs a slot, the algorithm sweeps and decrements usage counts until it finds a buffer with count 0. This is why recently-used pages survive brief scans across the whole table.

### 2.2 work_mem: Per-Operation Sort and Hash Memory

`work_mem` is not per-connection — it is per **operation**. A single query can use multiple sort and hash operations simultaneously.

```sql
-- This query could use work_mem 3 times:
SELECT a.x, b.y, c.z
FROM a
JOIN b ON a.id = b.a_id    -- hash join: 1 work_mem
JOIN c ON b.id = c.b_id    -- hash join: 1 work_mem
ORDER BY a.x               -- sort: 1 work_mem
```

If you have 200 connections and each runs a query with 3 operations:
```
worst case memory = 200 connections × 3 operations × work_mem
```

With `work_mem = 64MB`: 200 × 3 × 64 = **38 GB**. On a 32 GB server, this kills you.

**What happens when work_mem is exceeded?**

PostgreSQL spills to temporary files on disk. You can see this:

```sql
-- After running a sort-heavy query
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table ORDER BY some_column;

-- If you see this in EXPLAIN output:
-- Sort Method: external merge  Disk: 1024kB
-- That's a spill to disk — work_mem was too small for this sort
```

**Practical approach:**
- Set `work_mem` conservatively in `postgresql.conf` (4–16 MB)
- Increase it for specific sessions or queries that need it:

```sql
-- For this session only
SET work_mem = '256MB';

-- Run your big sort/aggregation
SELECT ...;

-- Reset
RESET work_mem;
```

### 2.3 maintenance_work_mem: For Heavy Maintenance Operations

Used by: `VACUUM`, `ANALYZE`, `CREATE INDEX`, `ALTER TABLE ... ADD FOREIGN KEY`, `CLUSTER`.

This memory is not shared across operations the same way work_mem is — you rarely have many maintenance operations running in parallel. So setting it higher is safer.

```ini
# A good default for most servers
maintenance_work_mem = 512MB

# For servers with lots of RAM doing large index builds
maintenance_work_mem = 2GB
```

Effect on `CREATE INDEX`: PostgreSQL sorts the index entries before writing the B-tree. The sort happens in maintenance_work_mem. Larger means fewer sort passes, faster index builds.

```sql
-- Temporarily increase for a one-time large index build
SET maintenance_work_mem = '2GB';
CREATE INDEX CONCURRENTLY idx_orders_customer_date
    ON orders (customer_id, created_at DESC);
RESET maintenance_work_mem;
```

### 2.4 effective_cache_size: A Hint to the Planner

This is not a memory allocation. PostgreSQL allocates nothing based on this setting. It is a **hint** to the query planner about how much memory is available for caching data across all levels (shared_buffers + OS page cache combined).

The planner uses it to estimate the cost of index scans. A larger `effective_cache_size` makes index scans look cheaper relative to sequential scans.

```ini
# Rule: set to total RAM minus what you'd expect the OS and other processes to use
# On a dedicated 64GB database server:
effective_cache_size = 48GB   # shared_buffers (16GB) + OS cache (~32GB)
```

Getting this wrong in the downward direction (e.g., leaving it at the default 4GB on a 64GB server) causes the planner to underestimate cache efficiency and prefer sequential scans over index scans.

### 2.5 WAL Buffers

`wal_buffers` is the in-memory buffer for WAL data before it is flushed to disk. The default (`-1`) auto-sizes it to 1/32 of shared_buffers, capped at 64 MB.

For write-heavy workloads, 64 MB is usually fine. More than 64 MB rarely helps because the WAL writer flushes it so frequently.

```ini
wal_buffers = 64MB   # explicit is better than relying on auto-sizing
```

### 2.6 Memory Sizing Reference

```
Server RAM:    4 GB       16 GB      64 GB
─────────────────────────────────────────────────────────
shared_buffers    1 GB       4 GB      16 GB
work_mem          4 MB       8 MB      16 MB
maintenance_wm   128 MB     512 MB     2 GB
effective_cache   3 GB      12 GB      48 GB
wal_buffers      16 MB      32 MB      64 MB
```

---

## 3. WAL — Write-Ahead Log

### 3.1 What WAL Is and Why It Exists

WAL (Write-Ahead Log) is the mechanism that gives PostgreSQL its **durability guarantee**: once a transaction commits, that data is safe even if the server crashes a millisecond later.

The rule: **WAL records describing a change must be written to disk before the data pages reflecting that change are written to disk.**

Why? Because writing a full 8 KB page for every small change would be catastrophically expensive. Instead:
1. Write a compact WAL record (often < 100 bytes) describing the change
2. Change the page in shared_buffers (memory only)
3. Eventually flush the changed page to disk (in the background)

On crash, PostgreSQL replays WAL records against the on-disk data pages to reconstruct the committed state. The data pages may be stale; WAL has everything needed to bring them up to date.

### 3.2 WAL Segments and LSN

WAL is stored as a sequence of **segment files**, each 16 MB by default (tunable via `--wal-segsize` at initdb time).

```bash
ls -la $PGDATA/pg_wal/
# 000000010000000000000001
# 000000010000000000000002
# 000000010000000000000003
# ...
```

The segment file name encodes the timeline ID and segment number. Within each segment, records are written sequentially with no gaps.

The **Log Sequence Number (LSN)** is a byte offset within the WAL stream — a 64-bit integer. It tells you exactly where in the WAL sequence a particular record lives.

```sql
-- Current WAL insert position (on primary)
SELECT pg_current_wal_lsn();
-- Returns something like: 0/3A7B4F8

-- On a replica, current replay position
SELECT pg_last_wal_replay_lsn();

-- How far is the replica behind?
-- On primary:
SELECT pg_current_wal_lsn() - sent_lsn AS send_lag,
       sent_lsn - write_lsn AS write_lag,
       write_lsn - flush_lsn AS flush_lag,
       flush_lsn - replay_lsn AS replay_lag,
       client_addr
FROM pg_stat_replication;
```

### 3.3 How a Write Actually Works

```
Application: UPDATE orders SET status = 'shipped' WHERE id = 42;

1. Backend acquires row lock on the target row

2. Backend constructs a WAL record describing the change:
   - Table OID, page number, offset within page
   - Old tuple values (for ROLLBACK and replication)
   - New tuple values

3. Backend writes WAL record to WAL buffer (memory, not disk yet)

4. Backend modifies the data page in shared_buffers (memory)
   - The old row version gets xmax set to current transaction ID
   - A new row version is inserted with xmin = current transaction ID

5. Application issues COMMIT

6. Backend calls WAL writer to flush WAL buffer to disk (fsync)
   - This is the moment the transaction becomes durable
   - Client receives success response AFTER this fsync completes

7. Background writer or checkpointer eventually flushes the
   modified data page to disk — this happens asynchronously
```

The critical insight: step 6 (WAL fsync) must complete before step 7 (data page flush). WAL must always be ahead of the data pages on disk.

### 3.4 fsync: Do Not Turn It Off

```ini
fsync = on   # Default. Never change this in production.
```

`fsync = off` tells PostgreSQL not to call `fsync()` after writing WAL. The OS may buffer the write in kernel memory. If the server crashes before the OS flushes it, **your WAL data is gone** — and your database is corrupt.

The correct response to "fsync is slowing us down" is not to disable it. It is to:
1. Use faster storage (NVMe, SSDs with power-loss protection)
2. Use a battery-backed write cache (BBWC) on the storage controller
3. Accept that the latency is the cost of correctness

```ini
# What you can tune instead of disabling fsync:
synchronous_commit = off   # Allows transaction commit before WAL flush
                           # Risk: up to wal_writer_delay (200ms) of data loss on crash
                           # Zero risk of corruption — only data loss within the window
```

`synchronous_commit = off` is a legitimate tradeoff for workloads where losing the last 200ms of writes is acceptable (analytics inserts, logging, etc.) — because the database itself will never be corrupt, just missing the very latest committed transactions.

### 3.5 wal_level: Controlling WAL Verbosity

```ini
wal_level = minimal   # Minimum WAL needed for crash recovery only.
                      # Cannot do replication or PITR from this.

wal_level = replica   # Default. Adds WAL needed for physical replication and PITR.
                      # Use this for all production systems.

wal_level = logical   # Adds WAL needed for logical replication (row-level change decoding).
                      # Required for logical replication, pglogical, Debezium, etc.
                      # More verbose = slightly more WAL generated.
```

```sql
-- Check current wal_level
SHOW wal_level;

-- You cannot change wal_level without a restart
-- (requires modifying postgresql.conf and restarting PostgreSQL)
```

### 3.6 Checkpoints: Bounding Recovery Time

A **checkpoint** is the moment when all dirty pages in shared_buffers that were modified up to a certain WAL position are flushed to disk. After a checkpoint completes, PostgreSQL can begin crash recovery from that point — it does not need to replay WAL before the checkpoint.

```
WAL stream:  ──────[CP1]────────────[CP2]────────────[CP3]──► (crash here)
                                                       │
                                           Recovery starts here
                                           Replays WAL from CP3
```

The checkpoint interval is controlled by:
```ini
checkpoint_timeout = 5min           # Maximum time between checkpoints
max_wal_size = 1GB                  # Also triggers a checkpoint if WAL grows this large
checkpoint_completion_target = 0.9  # Spread checkpoint I/O over 90% of the interval
```

**Why frequent checkpoints are bad:**
- Each checkpoint writes all dirty pages — high I/O spike
- More checkpoints = more total I/O (the same page written multiple times before it's flushed)
- `checkpoint_completion_target = 0.9` tells PostgreSQL to spread the I/O over 90% of the checkpoint interval, smoothing the spike

**Why infrequent checkpoints are also bad:**
- Longer recovery time after crash
- More WAL accumulation on disk

```sql
-- Monitor checkpoint frequency and I/O
SELECT checkpoints_timed,
       checkpoints_req,           -- Checkpoints triggered by WAL size (bad — means checkpoints too frequent)
       checkpoint_write_time,
       checkpoint_sync_time,
       buffers_checkpoint,
       buffers_clean,             -- bgwriter activity
       maxwritten_clean,          -- bgwriter hitting its rate limit
       buffers_backend            -- Backends forced to write dirty pages themselves (bad)
FROM pg_stat_bgwriter;
```

If `checkpoints_req` is much higher than `checkpoints_timed`, your WAL is filling up too fast. Increase `max_wal_size`.

If `buffers_backend` is non-zero and growing, backends are writing pages directly because shared_buffers is full and bgwriter cannot keep up. Increase `shared_buffers` or tune `bgwriter_lru_maxpages`.

### 3.7 WAL Archiving for Point-In-Time Recovery (PITR)

WAL archiving copies WAL segment files to a safe location as they are completed. Combined with a base backup, this lets you restore the database to any point in time — not just the last backup.

```ini
# postgresql.conf
archive_mode = on
archive_command = 'cp %p /mnt/wal_archive/%f'
# Or use a tool like pgBackRest, Barman, or WAL-G:
archive_command = 'wal-g wal-push %p'
```

PITR restore procedure (conceptual):
1. Restore base backup to `PGDATA`
2. Create `recovery.signal` file
3. Set `restore_command` to copy archived WAL files back
4. Start PostgreSQL — it replays WAL until it reaches the target time or LSN
5. Issue `SELECT pg_wal_replay_resume()` to open the database for writes

```ini
# recovery.conf (PostgreSQL < 12) or postgresql.conf (PostgreSQL >= 12)
restore_command = 'wal-g wal-fetch %f %p'
recovery_target_time = '2026-03-20 14:30:00'
recovery_target_action = 'promote'
```

### 3.8 How Replication Uses WAL

Physical (streaming) replication works by shipping WAL from the primary to replicas in real time.

```
Primary:
  Backend writes → WAL buffer → WAL writer → WAL segment files
                                    │
                                    └──► WAL sender process ──► Replica's WAL receiver
                                                                      │
                                                                      ▼
                                                              Replica startup process
                                                              replays WAL continuously
```

The replica is essentially doing crash recovery — forever. It applies WAL records as they arrive, keeping its data files in sync with the primary's committed state.

```sql
-- On primary: replication status
SELECT client_addr,
       state,                -- startup, catchup, streaming
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       write_lag,
       flush_lag,
       replay_lag
FROM pg_stat_replication;
```

---

## 4. MVCC Deep Dive

### 4.1 The Problem MVCC Solves

Without MVCC, a reader and a writer accessing the same row must take turns — the reader blocks the writer or the writer blocks the reader. PostgreSQL's solution: **readers never block writers, writers never block readers**. This is achieved by keeping multiple versions of the same row simultaneously.

### 4.2 System Columns: The Hidden Row Metadata

Every row in every PostgreSQL table has system columns invisible to `SELECT *`:

```sql
-- Make them visible
SELECT xmin, xmax, ctid, tableoid, *, cmin, cmax
FROM orders
WHERE id = 42;

-- xmin: transaction ID that created this row version (INSERT or UPDATE)
-- xmax: transaction ID that deleted/replaced this row version (DELETE or UPDATE)
--       0 if the row is still current (not deleted)
-- ctid: physical location of this row version — (page_number, tuple_offset)
--       (0,1) means page 0, first tuple on that page
-- tableoid: OID of the table (useful in queries against partitioned tables)
-- cmin: command ID within the creating transaction (for same-transaction visibility)
-- cmax: command ID within the deleting transaction
```

### 4.3 What Happens on INSERT, UPDATE, DELETE

**INSERT:**
```sql
BEGIN;
-- xact_id = 1001
INSERT INTO orders (customer_id, total) VALUES (5, 99.99);
COMMIT;

-- Created row version:
-- xmin=1001, xmax=0, ctid=(0,1), customer_id=5, total=99.99
```

**UPDATE (creates a new row version, marks old one dead):**
```sql
BEGIN;
-- xact_id = 1002
UPDATE orders SET total = 149.99 WHERE id = 42;
COMMIT;

-- Old row version (now dead):
-- xmin=1001, xmax=1002, ctid=(0,1), total=99.99

-- New row version (current):
-- xmin=1002, xmax=0, ctid=(0,2), total=149.99
```

The old row version is NOT immediately removed. It stays on the page until VACUUM reclaims it.

**DELETE (marks existing row version dead):**
```sql
BEGIN;
-- xact_id = 1003
DELETE FROM orders WHERE id = 42;
COMMIT;

-- Row version (now dead):
-- xmin=1002, xmax=1003, ctid=(0,2), total=149.99
-- The tuple is still physically on disk — just invisible to new transactions
```

### 4.4 Transaction Snapshots: What a Backend "Sees"

When a backend starts a transaction (or, in REPEATABLE READ/SERIALIZABLE, when it first accesses data), PostgreSQL captures a **snapshot**:

```
Snapshot = {
  xmin:    lowest active transaction ID at snapshot time
           (transactions with ID < xmin are guaranteed committed or rolled back)
  xmax:    next transaction ID to be assigned
           (transactions with ID >= xmax haven't started yet)
  xip:     list of transaction IDs that are active (in-progress) at snapshot time
           (between xmin and xmax but not committed)
}
```

**Visibility rule for a row version:**

A row version (xmin, xmax) is visible to a snapshot (snap_xmin, snap_xmax, xip) if:
1. `xmin` is committed AND `xmin < snap_xmax` AND `xmin` is not in `xip`
   (the creating transaction committed before our snapshot)
2. AND either `xmax = 0` OR `xmax` is not committed OR `xmax >= snap_xmax` OR `xmax` is in `xip`
   (the deleting transaction has not committed yet from our snapshot's perspective)

```sql
-- See your current transaction snapshot
SELECT * FROM txid_current_snapshot();
-- Returns: xmin:xmax:xip_list
-- Example: 1001:1005:1002,1004
-- Meaning: txns < 1001 are all done, txns >= 1005 haven't started,
--          1002 and 1004 are currently in-progress

-- Current transaction ID
SELECT txid_current();
```

### 4.5 Isolation Levels and MVCC

```sql
-- READ COMMITTED (default):
-- Snapshot taken at the start of EACH STATEMENT
-- You can see data committed between your statements
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- REPEATABLE READ:
-- Snapshot taken once at the start of the TRANSACTION
-- You see a consistent view throughout — no phantom reads for rows
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- SERIALIZABLE:
-- Like REPEATABLE READ but also detects serialization anomalies
-- PostgreSQL uses SSI (Serializable Snapshot Isolation)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

### 4.6 Dead Tuples: The Cost of MVCC

Every UPDATE and DELETE leaves a dead tuple behind. These dead tuples:
- Waste space on data pages
- Make index scans slower (indexes still point to dead row versions that must be checked)
- Are invisible but physically present — bloating the table

```sql
-- See dead tuples accumulating
SELECT schemaname,
       relname,
       n_live_tup,
       n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_vacuum,
       last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;
```

### 4.7 Seeing MVCC Live

```sql
-- Set up a demo
CREATE TABLE mvcc_demo (id int, val text);
INSERT INTO mvcc_demo VALUES (1, 'original');

-- See the initial state
SELECT xmin, xmax, ctid, id, val FROM mvcc_demo;
-- xmin=some_txid, xmax=0, ctid=(0,1), id=1, val='original'

-- Terminal 1: Start a long-running transaction
BEGIN;
SELECT txid_current();  -- note this ID

-- Terminal 2: Update the row
BEGIN;
UPDATE mvcc_demo SET val = 'updated' WHERE id = 1;
COMMIT;

-- Terminal 2: See both versions exist
SELECT xmin, xmax, ctid, id, val FROM mvcc_demo;
-- (0,1): xmin=old, xmax=new_txid  ← dead tuple (old version)
-- (0,2): xmin=new_txid, xmax=0    ← live tuple (new version)

-- Terminal 1 still sees 'original' (its snapshot predates the update)
SELECT val FROM mvcc_demo WHERE id = 1;
-- Returns: original  ← MVCC at work

-- Terminal 1: COMMIT (releases snapshot)
COMMIT;
```

---

## 5. Vacuum and Autovacuum

### 5.1 What VACUUM Does

VACUUM is not optional. It is a core maintenance operation that PostgreSQL requires to function correctly over time.

**What VACUUM does:**
1. Scans the table, identifies dead tuples (where `xmax` is a committed transaction)
2. Marks those slots as reusable in the **Free Space Map (FSM)**
3. Updates the **Visibility Map (VM)** — marks pages where all tuples are visible to all transactions (enables index-only scans)
4. Updates `pg_stat_user_tables` vacuum stats
5. Advances the `relfrozenxid` for the table (critical for XID wraparound prevention)

**What VACUUM does NOT do:**
- Does not return space to the OS (the table file does not shrink)
- Does not lock the table for writes (regular VACUUM is non-blocking)
- Does not fix index bloat (it does clean up index entries pointing to dead tuples)

```sql
-- Run vacuum manually (usually autovacuum handles this)
VACUUM orders;

-- Vacuum with detailed output
VACUUM VERBOSE orders;

-- Sample output:
-- INFO:  vacuuming "public.orders"
-- INFO:  scanned index "orders_pkey" to remove 1503 row versions
-- INFO:  "orders": removed 1503 row versions in 89 pages
-- INFO:  "orders": found 1503 removable, 48291 nonremovable row versions
-- INFO:  index "orders_pkey" now contains 48291 row versions in 334 pages
-- INFO:  "orders": 89 pages from table (1.04%) had 1503 dead item identifiers removed
```

### 5.2 VACUUM FULL: The Dangerous One

`VACUUM FULL` is fundamentally different from regular VACUUM:
- Rewrites the entire table to a new file (like `CLUSTER` or `pg_repack`)
- Returns space to the OS
- **Acquires an exclusive lock on the table for the entire duration**
- Blocks ALL reads and writes while running

```sql
-- DO NOT run this on large tables in production without a maintenance window
VACUUM FULL large_orders_table;
-- This table is now locked for potentially hours
```

**Better alternatives for reclaiming space in production:**
- `pg_repack` extension: rewrites the table without holding a long exclusive lock
- Accept the bloat and let regular VACUUM keep dead tuples at manageable levels

```bash
# pg_repack: online table repack (no long locks)
pg_repack -d mydb -t large_orders_table
```

### 5.3 ANALYZE: Updating Planner Statistics

ANALYZE collects statistics about column value distributions and stores them in `pg_statistic`. The query planner reads these to estimate how many rows a filter will return.

```sql
-- Run analyze manually
ANALYZE orders;
ANALYZE orders (customer_id, created_at);  -- specific columns

-- See what statistics exist
SELECT attname,
       n_distinct,
       correlation,
       most_common_vals,
       most_common_freqs,
       histogram_bounds
FROM pg_stats
WHERE tablename = 'orders'
  AND attname = 'status';
```

`n_distinct`: number of distinct values (negative means it's a fraction of total rows)
`correlation`: how well the physical row order matches the sorted order of this column. Values near 1 or -1 mean the data is well-ordered, making index scans efficient. Values near 0 mean random distribution — index scans may require many random I/Os.

```ini
# How many distinct values does ANALYZE sample?
# default_statistics_target = 100 (default)
# Each column: ~300 * statistics_target rows sampled
# Increase for columns with skewed distributions or many distinct values

ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 500;
ANALYZE orders (customer_id);
```

### 5.4 Autovacuum Configuration

Autovacuum triggers VACUUM on a table when:
```
dead_tuples > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * table_rows
```

Default:
```ini
autovacuum_vacuum_threshold = 50         # minimum dead tuples
autovacuum_vacuum_scale_factor = 0.2     # 20% of table rows
```

For a 10-million-row table: `50 + 0.2 * 10,000,000 = 2,000,050` dead tuples before autovacuum triggers. That is a lot of dead tuples. For large tables, reduce the scale factor:

```sql
-- Per-table autovacuum settings (override global config)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,    -- trigger at 1% dead tuples
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.005,  -- analyze at 0.5% changes
    autovacuum_vacuum_cost_delay = 2          -- less aggressive throttling for this table
);
```

```ini
# Key autovacuum settings in postgresql.conf
autovacuum = on                        # Never turn this off
autovacuum_max_workers = 3             # Concurrent autovacuum workers
autovacuum_naptime = 1min              # How often autovacuum checks for work
autovacuum_vacuum_cost_delay = 2ms     # Throttling: pause after writing this much data
autovacuum_vacuum_cost_limit = 200     # How much "cost" before throttling pause
                                       # Increase if autovacuum is too slow
```

### 5.5 Transaction ID Wraparound: The Catastrophic Failure Mode

This is the most dangerous failure mode in PostgreSQL and the least understood.

Transaction IDs (XIDs) are 32-bit integers. PostgreSQL can only distinguish ~2 billion active transactions at a time (it uses modular arithmetic). When XIDs approach wraparound, PostgreSQL marks the database as unsafe and **refuses all write transactions** to prevent data corruption.

```sql
-- Monitor distance to wraparound (do this regularly)
SELECT datname,
       age(datfrozenxid) AS xid_age,
       2000000000 - age(datfrozenxid) AS xids_remaining,
       round(100.0 * age(datfrozenxid) / 2000000000, 2) AS pct_used
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Per-table wraparound monitoring
SELECT schemaname,
       relname,
       age(relfrozenxid) AS table_xid_age,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE relkind = 'r'
  AND age(relfrozenxid) > 100000000  -- getting concerning above 100M
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

VACUUM's role in preventing wraparound: VACUUM **freezes** old row versions — it replaces their `xmin` with a special `FrozenTransactionId` value that is considered visible to all transactions forever. This resets the age of the tuple's XID.

```ini
# When autovacuum kicks in specifically for wraparound prevention (regardless of dead tuples):
autovacuum_freeze_max_age = 200000000   # 200M transactions — force-freeze before this
vacuum_freeze_min_age = 50000000        # Don't freeze XIDs younger than this

# Emergency: if age(datfrozenxid) approaches 2B, PostgreSQL shuts down writes
# The warning appears in logs around 40M transactions from the limit
```

> **Production alarm**: Set a monitoring alert when `age(datfrozenxid)` exceeds 500M for any database. You have time to react at 500M; at 1.8B you are in emergency territory.

### 5.6 Table Bloat and Index Bloat

**Table bloat**: Pages that are more than X% empty because dead tuples were vacuumed but the space was never reused.

**Index bloat**: B-tree indexes accumulate dead index entries pointing to vacuumed dead tuples. Regular VACUUM cleans these, but if vacuum is too infrequent, indexes grow beyond their optimal size.

```sql
-- Estimate table bloat (approximate — uses statistics, not exact counts)
SELECT schemaname,
       tablename,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
       pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) AS table_size,
       pg_size_pretty(
           pg_total_relation_size(schemaname || '.' || tablename) -
           pg_relation_size(schemaname || '.' || tablename)
       ) AS index_size,
       n_dead_tup,
       n_live_tup,
       round(100 * n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 20;

-- For precise bloat measurement, use the pgstattuple extension
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstattuple('orders');
-- Returns: table_len, tuple_count, tuple_len, tuple_percent,
--          dead_tuple_count, dead_tuple_len, dead_tuple_percent, free_space, free_percent
```

---

## 6. Query Planner and EXPLAIN

### 6.1 How the Planner Works

When PostgreSQL receives a query, it goes through these stages:
1. **Parse**: Syntax check → parse tree
2. **Analyze/Rewrite**: Semantic check, view expansion → query tree
3. **Plan**: Generate candidate plans, cost each one, pick cheapest → plan tree
4. **Execute**: Walk the plan tree, produce rows

The planner's cost model assigns abstract costs to operations (sequential page reads, random page reads, CPU tuple processing, etc.) and sums them to compare plans:

```ini
# Cost constants (rarely need changing, but understand what they mean)
seq_page_cost = 1.0          # Cost of reading one page sequentially (baseline)
random_page_cost = 4.0       # Cost of a random page read (set to 1.1 for SSDs)
cpu_tuple_cost = 0.01        # Cost of processing one row
cpu_index_tuple_cost = 0.005
cpu_operator_cost = 0.0025
```

For SSDs, reducing `random_page_cost` to 1.1–1.5 makes the planner correctly prefer index scans over sequential scans more often.

```sql
-- Check current settings
SHOW random_page_cost;
SHOW seq_page_cost;
```

### 6.2 EXPLAIN: Three Levels of Detail

```sql
-- Level 1: Estimated plan only (no query executed)
EXPLAIN
SELECT o.id, c.name, o.total
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days';

-- Level 2: Execute the query, show actual vs estimated rows and timing
EXPLAIN ANALYZE
SELECT o.id, c.name, o.total
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days';

-- Level 3: Full detail — actual rows, timing, AND buffer hit/miss counts
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, c.name, o.total
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days';
```

### 6.3 Reading EXPLAIN Output

```
Hash Join  (cost=284.50..1847.20 rows=1250 width=48)
           (actual time=12.340..89.231 rows=1187 loops=1)
  Buffers: shared hit=892 read=143 dirtied=0 written=0
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..1402.80 rows=1250 width=32)
                             (actual time=0.018..67.412 rows=1187 loops=1)
        Filter: ((status = 'pending') AND (created_at > (now() - '7 days'::interval)))
        Rows Removed by Filter: 48813
        Buffers: shared hit=780 read=140
  ->  Hash  (cost=159.00..159.00 rows=10000 width=24)
             (actual time=11.890..11.890 rows=10000 loops=1)
        Buckets: 16384  Batches: 1  Memory Usage: 634kB
        Buffers: shared hit=112 read=3
        ->  Seq Scan on customers c  (cost=0.00..159.00 rows=10000 width=24)
                                      (actual time=0.012..5.891 rows=10000 loops=1)
            Buffers: shared hit=112 read=3
Planning Time: 1.230 ms
Execution Time: 89.456 ms
```

**Anatomy:**
- `cost=284.50..1847.20`: startup cost..total cost (planner's estimate in abstract units)
- `rows=1250`: planner estimates 1,250 rows output
- `width=48`: average output row width in bytes
- `actual time=12.340..89.231`: actual startup time..total time in milliseconds
- `actual rows=1187`: actual rows returned
- `loops=1`: this node executed 1 time (nested loop nodes may execute many times)
- `shared hit=892`: 892 pages served from shared_buffers (fast)
- `shared read=143`: 143 pages read from disk or OS cache (slower)

**Red flags in EXPLAIN output:**
- `rows=1 actual rows=50000`: massive row estimate error → stale statistics, run ANALYZE
- `Rows Removed by Filter: 48813` on a Seq Scan: an index might eliminate this scan
- High `shared read` values: your working set exceeds shared_buffers or OS cache
- `Sort Method: external merge Disk: 2048kB`: sort spilled to disk, increase work_mem
- `Hash Batches: 4`: hash join spilled to disk (batches > 1), increase work_mem
- Long `loops` count on Nested Loop inner side: a missing index on the inner table

### 6.4 Key Plan Nodes

**Sequential Scan (Seq Scan)**
Reads every page of the table. Good for: large fraction of the table, small tables. Bad for: fetching a few rows from a large table.

**Index Scan**
Follows index to find matching TIDs (tuple IDs), then fetches each row from the heap page. Good for: small result sets with good index correlation. Bad for: large result sets with poor correlation (many random I/Os).

**Index Only Scan**
Satisfies the query entirely from the index without touching heap pages. Requires the visibility map to show the page is all-visible (VACUUM must have run recently). Shows as `Heap Fetches: 0` when fully index-only.

```sql
-- Create an index that enables index-only scans for a covering query
CREATE INDEX idx_orders_covering
ON orders (customer_id, created_at DESC)
INCLUDE (total, status);

-- This query can be satisfied entirely from the index:
SELECT customer_id, created_at, total, status
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC;
```

**Bitmap Heap Scan**
Used when multiple index conditions apply, or when an index scan would generate too many random I/Os. Two phases:
1. **Bitmap Index Scan**: scans index, builds a bitmap of matching page locations
2. **Bitmap Heap Scan**: sorts the page locations, reads pages in order (sequential-ish)

This is why Bitmap Heap Scans are more efficient than Index Scans for medium-selectivity queries.

**Hash Join**
Builds a hash table from the smaller relation, probes it with each row from the larger relation. Best for large joins where neither side is very small. Requires work_mem for the hash table.

**Merge Join**
Joins two pre-sorted inputs by walking them in parallel. Requires both inputs sorted on the join key. Excellent when both sides have matching sorted indexes.

**Nested Loop**
For each row in the outer relation, probe the inner relation. Excellent when outer is small and inner has an index. Catastrophic when outer is large.

### 6.5 When the Planner Makes Bad Choices

**Cause 1: Stale statistics**
```sql
-- The planner estimates 1 row, but 50,000 rows exist for this status value
-- Because statistics show 'pending' as rare, but the data has changed
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';
-- rows=1 vs actual rows=50000

-- Fix: run ANALYZE
ANALYZE orders;
-- Or increase statistics target for this column:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
```

**Cause 2: High column correlation / skew**
The planner assumes values are uniformly distributed. If 90% of orders have `status = 'shipped'` and 0.01% have `status = 'pending'`, a query for `'pending'` should use an index — but if the planner thinks it's 10%, it might choose a Seq Scan.

**Cause 3: Correlated predicates**
The planner estimates `WHERE city = 'NYC' AND state = 'NY'` as if city and state are independent. They're not — if city is NYC, state is always NY. Extended statistics can help:

```sql
-- Create statistics on correlated columns
CREATE STATISTICS orders_city_state ON city, state FROM orders;
ANALYZE orders;
```

**Cause 4: Function wrapping a column**
```sql
-- Bad: function prevents index use
WHERE date_trunc('day', created_at) = '2026-03-20'

-- Good: expose the column for the index
WHERE created_at >= '2026-03-20' AND created_at < '2026-03-21'
-- Or use a functional index:
CREATE INDEX idx_orders_date ON orders (date_trunc('day', created_at));
```

### 6.6 Plan-Affecting Settings

```sql
-- Force or disable specific join/scan strategies (for debugging only)
SET enable_seqscan = off;        -- Force index use (test if an index would help)
SET enable_hashjoin = off;       -- Force merge/nested loop
SET enable_nestloop = off;       -- Avoid nested loop when it's pathologically slow
SET enable_indexscan = off;      -- Force seq scan

-- IMPORTANT: These settings are session-level debugging tools.
-- Do not set them globally in postgresql.conf — you will disable valid strategies.
-- The correct fix is always: better statistics, better indexes, or better queries.
```

### 6.7 pg_stat_statements: Production Query Analysis

`pg_stat_statements` tracks aggregated execution statistics for every distinct query across all sessions. It is the most valuable tool for identifying slow queries in production.

```sql
-- Install (requires postgresql.conf change + restart)
-- shared_preload_libraries = 'pg_stat_statements'

CREATE EXTENSION pg_stat_statements;

-- Top 10 queries by total execution time
SELECT round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       round(stddev_exec_time::numeric, 2) AS stddev_ms,
       calls,
       round(rows / calls::numeric, 1) AS avg_rows,
       shared_blks_hit,
       shared_blks_read,
       round(100.0 * shared_blks_hit /
           NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS hit_pct,
       left(query, 120) AS query
FROM pg_stat_statements
WHERE calls > 10
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries with poor cache hit rates (lots of physical I/O)
SELECT round(mean_exec_time::numeric, 2) AS avg_ms,
       calls,
       round(100.0 * shared_blks_hit /
           NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS hit_pct,
       left(query, 120) AS query
FROM pg_stat_statements
WHERE shared_blks_hit + shared_blks_read > 1000
  AND calls > 10
ORDER BY hit_pct ASC
LIMIT 10;

-- Reset stats (do after a schema/index change to get fresh baseline)
SELECT pg_stat_statements_reset();
```

---

## 7. Table Storage Internals

### 7.1 Pages: The Fundamental Storage Unit

All data in PostgreSQL is stored in **pages** (also called blocks), each exactly 8 KB. Every table, index, sequence, and visibility map is a sequence of these 8 KB pages.

```
┌─────────────────────────────────────────────────────┐
│                    Page (8192 bytes)                 │
├──────────────────┬──────────────────────────────────┤
│  Page Header     │  24 bytes                        │
│  (pd_lsn, flags, │  - pd_lsn: LSN of last WAL       │
│   free space     │    record that changed this page  │
│   pointers)      │  - pd_lower: start of free space  │
│                  │  - pd_upper: end of free space    │
├──────────────────┤  - pd_special: start of special   │
│  Item Pointers   │    area (indexes use this)        │
│  (line pointer   │                                   │
│   array, 4 bytes │                                   │
│   each)          │                                   │
├──────────────────┤                                   │
│                  │                                   │
│   Free Space     │                                   │
│                  │                                   │
├──────────────────┤                                   │
│  Tuple Data      │  Tuples packed from the bottom   │
│  (rows stored    │  up; item pointers from top down │
│   from bottom)   │                                   │
├──────────────────┤                                   │
│  Special Area    │  B-tree indexes store right-link  │
│  (index-specific)│  here; heap tables have none     │
└─────────────────────────────────────────────────────┘
```

Each tuple (row) has its own header: `t_xmin`, `t_xmax`, `t_ctid`, `t_infomask`, and then the actual column data. The tuple header is 23 bytes before any column data.

```sql
-- Physical size of a row (header + data)
SELECT pg_column_size(t.*) AS row_bytes
FROM some_table t
LIMIT 5;

-- What is the actual page count for a table?
SELECT relpages, reltuples,
       round(reltuples / NULLIF(relpages, 0)) AS tuples_per_page
FROM pg_class
WHERE relname = 'orders';
```

### 7.2 TOAST: Storing Large Values

PostgreSQL's page size is 8 KB. Column values larger than ~2 KB (the TOAST threshold is 2,048 bytes before compression, or 8,192 bytes / 4 = 2,040 bytes more precisely) are handled by **TOAST** (The Oversized-Attribute Storage Technique).

When a row would exceed the page size, PostgreSQL:
1. **Compresses** the large column value (using pglz or lz4)
2. If still large, **slices** it into chunks and stores them in a separate **TOAST table**
3. Replaces the column value in the main table with a pointer to the TOAST table

```sql
-- Every table with TOAST-able columns has a corresponding TOAST table
SELECT c.relname AS table_name,
       t.relname AS toast_table,
       pg_size_pretty(pg_total_relation_size(t.oid)) AS toast_size
FROM pg_class c
JOIN pg_class t ON t.oid = c.reltoastrelid
WHERE c.relname = 'orders';

-- Column storage strategies
-- extended: try to compress, then TOAST out-of-line (default for TEXT, JSON, BYTEA)
-- main: try to compress in-line, avoid TOAST
-- external: TOAST out-of-line without compression
-- plain: never TOAST (for fixed-width types like INT, FLOAT)
SELECT attname, attstorage
FROM pg_attribute
WHERE attrelid = 'orders'::regclass
  AND attnum > 0;

-- Change storage strategy (rarely needed)
ALTER TABLE orders ALTER COLUMN notes SET STORAGE EXTERNAL;  -- no compression, TOAST directly
```

**TOAST performance implications:**
- Queries that don't select the large column pay no TOAST access cost
- `SELECT id, status FROM orders` will not read the TOAST table even if `notes` is huge
- `SELECT *` on a table with large text columns will fetch from TOAST — much slower

### 7.3 Fill Factor: Tuning for UPDATE-Heavy Tables

The **fill factor** controls what percentage of each page PostgreSQL fills when writing new rows. The default is 100% (pack pages completely).

For tables with frequent UPDATEs, this is a problem: an UPDATE creates a new row version, which needs space on the same page (for the "HOT" optimization — Heap-Only Tuple). If the page is full, the new version goes to a different page, and a full index update is required.

```sql
-- Set fill factor to 70% for a heavily-updated table
-- 30% of each page reserved for row version updates
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id BIGINT,
    last_activity TIMESTAMPTZ,
    data JSONB
) WITH (fillfactor = 70);

-- Or alter an existing table (takes effect on future writes and VACUUM)
ALTER TABLE sessions SET (fillfactor = 70);

-- After setting fill factor, VACUUM reclaims and restructures the space:
VACUUM sessions;
```

**HOT updates (Heap-Only Tuple)**: When an UPDATE can find space on the same page, and the updated columns are not part of any index, PostgreSQL can do a HOT update — no index update required, just a chain of row versions on the same page. This is dramatically faster than a full update. Fill factor below 100% creates the space headroom that enables HOT updates.

```sql
-- Monitor HOT update effectiveness
SELECT relname,
       n_tup_upd AS total_updates,
       n_tup_hot_upd AS hot_updates,
       round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 1) AS hot_pct
FROM pg_stat_user_tables
WHERE n_tup_upd > 0
ORDER BY n_tup_upd DESC;
```

---

## 8. PostgreSQL Configuration

### 8.1 Key Configuration Categories

**postgresql.conf** is the primary configuration file. Changes take effect on reload (`SELECT pg_reload_conf()`) unless they require a restart (indicated by the `context` column in `pg_settings`).

```sql
-- What context does a setting require?
SELECT name, setting, unit, context, short_desc
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'max_connections',
               'wal_level', 'synchronous_commit', 'checkpoint_timeout');

-- context values:
-- internal: cannot be changed (compiled in)
-- postmaster: requires restart
-- sighup: reload sufficient (pg_reload_conf())
-- superuser: superuser can set in session
-- user: any user can set in session
```

### 8.2 Realistic Configurations by Server Size

**Small Server — 4 GB RAM (development or small production)**
```ini
# Connection Settings
max_connections = 100

# Memory
shared_buffers = 1GB
work_mem = 4MB
maintenance_work_mem = 128MB
effective_cache_size = 3GB

# WAL
wal_level = replica
wal_buffers = 16MB
checkpoint_timeout = 5min
max_wal_size = 1GB
checkpoint_completion_target = 0.9
synchronous_commit = on

# Query Planner
random_page_cost = 1.1             # Assume SSD
effective_io_concurrency = 200     # SSD-appropriate

# Logging
log_min_duration_statement = 1000  # Log queries > 1 second
log_checkpoints = on
log_lock_waits = on
log_temp_files = 0                 # Log all temp file usage

# Autovacuum
autovacuum = on
autovacuum_max_workers = 2
autovacuum_vacuum_cost_delay = 10ms

# Extensions
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 5000
pg_stat_statements.track = all
```

**Medium Server — 16 GB RAM (standard production)**
```ini
# Connection Settings
max_connections = 200              # Use PgBouncer in front

# Memory
shared_buffers = 4GB
work_mem = 8MB                     # 200 conn × 3 ops × 8MB = 4.8GB worst case
maintenance_work_mem = 512MB
effective_cache_size = 12GB

# WAL
wal_level = replica
wal_buffers = 64MB
checkpoint_timeout = 10min
max_wal_size = 4GB
checkpoint_completion_target = 0.9
synchronous_commit = on

# Query Planner
random_page_cost = 1.1
effective_io_concurrency = 200

# Background Writer
bgwriter_lru_maxpages = 100
bgwriter_delay = 200ms

# Autovacuum
autovacuum = on
autovacuum_max_workers = 3
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 400

# Logging
log_min_duration_statement = 500
log_checkpoints = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 250ms  # Log autovacuum runs > 250ms

shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
```

**Large Server — 64 GB RAM (high-traffic production)**
```ini
# Connection Settings
max_connections = 400              # Still use PgBouncer — real connection count should be lower

# Memory
shared_buffers = 16GB
work_mem = 16MB                    # Watch actual parallel query work_mem multiplication
maintenance_work_mem = 2GB
effective_cache_size = 48GB

# WAL
wal_level = replica
wal_buffers = 64MB                 # 64MB cap is effective regardless
checkpoint_timeout = 15min
max_wal_size = 16GB
min_wal_size = 2GB
checkpoint_completion_target = 0.9
synchronous_commit = on

# Query Planner
random_page_cost = 1.1
effective_io_concurrency = 300
max_parallel_workers_per_gather = 4    # Enable parallel query
max_parallel_workers = 8
max_worker_processes = 16

# Background Writer
bgwriter_lru_maxpages = 200
bgwriter_delay = 50ms              # More aggressive on large servers

# Autovacuum
autovacuum = on
autovacuum_max_workers = 6
autovacuum_vacuum_cost_delay = 1ms
autovacuum_vacuum_cost_limit = 800

# Logging
log_min_duration_statement = 200
log_checkpoints = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 100ms

shared_preload_libraries = 'pg_stat_statements, pg_buffercache'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
```

### 8.3 pg_hba.conf: Authentication Configuration

```ini
# pg_hba.conf
# TYPE   DATABASE   USER        ADDRESS         METHOD

# Local socket connections (OS-level auth)
local    all        postgres                    peer      # OS user 'postgres' → DB user 'postgres'
local    all        all                         peer

# Loopback TCP (application on same host)
host     all        all         127.0.0.1/32    scram-sha-256
host     all        all         ::1/128         scram-sha-256

# Application servers (specific subnet)
host     mydb       app_user    10.0.1.0/24     scram-sha-256

# Replica servers
host     replication repl_user  10.0.2.0/24     scram-sha-256

# Reject everything else (explicit deny — optional, as reject is the default)
host     all         all         0.0.0.0/0      reject
```

Auth methods:
- `scram-sha-256`: Password authentication with strong hashing. Use this everywhere.
- `md5`: Legacy password auth. Avoid — MD5 is weak.
- `peer`: Authenticates by matching the OS user to the database user. Only for local connections.
- `trust`: No authentication. Never use in production outside `localhost` for specific needs.
- `cert`: Client certificate authentication. Use for service-to-service.
- `reject`: Explicitly deny the connection.

### 8.4 Applying Configuration Changes

```sql
-- Check if a setting requires restart
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart = true;

-- Apply sighup-context changes without restart
SELECT pg_reload_conf();

-- Verify the new value took effect
SHOW work_mem;
SELECT name, setting, unit FROM pg_settings WHERE name = 'work_mem';

-- For settings that require a restart:
-- 1. Edit postgresql.conf
-- 2. pg_ctl restart  (or via systemd: systemctl restart postgresql)
-- Note: pending_restart = true will appear until the restart happens
```

---

## 9. Monitoring PostgreSQL Internals

### 9.1 pg_stat_activity: What Every Backend Is Doing

```sql
-- Full view: current backend state
SELECT pid,
       usename,
       application_name,
       client_addr,
       backend_start,
       xact_start,                    -- when current transaction started (NULL if no txn)
       query_start,
       state_change,
       state,                         -- idle, active, idle in transaction, idle in transaction (aborted)
       wait_event_type,               -- Lock, LWLock, IO, Client, etc.
       wait_event,                    -- specific event name
       backend_type,                  -- client backend, autovacuum worker, etc.
       left(query, 200) AS query
FROM pg_stat_activity
ORDER BY query_start NULLS LAST;

-- Find long-running queries
SELECT pid,
       now() - query_start AS duration,
       state,
       wait_event_type,
       wait_event,
       left(query, 200) AS query
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < NOW() - INTERVAL '30 seconds'
ORDER BY duration DESC;

-- Find idle-in-transaction connections (danger: holding locks)
SELECT pid,
       now() - xact_start AS txn_duration,
       left(query, 200) AS last_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND xact_start < NOW() - INTERVAL '5 minutes'
ORDER BY txn_duration DESC;

-- Kill a stuck query (replace PID)
SELECT pg_cancel_backend(12345);    -- sends SIGINT (graceful)
SELECT pg_terminate_backend(12345); -- sends SIGTERM (forceful)
```

The `wait_event_type` column tells you what a backend is waiting for. Key types:
- `Lock`: waiting on a heavyweight lock (table, row, advisory)
- `LWLock`: waiting on a lightweight lock (internal, usually transient)
- `IO`: waiting for disk I/O
- `Client`: waiting for the client to send data (idle connections often show this)
- `IPC`: waiting for another process

### 9.2 pg_stat_user_tables: Vacuum and Access Health

```sql
SELECT schemaname,
       relname,
       seq_scan,                      -- number of sequential scans (high = missing index?)
       seq_tup_read,                  -- rows read by seq scans
       idx_scan,                      -- number of index scans
       idx_tup_fetch,                 -- rows fetched via index
       n_tup_ins,                     -- inserts
       n_tup_upd,                     -- updates
       n_tup_del,                     -- deletes
       n_tup_hot_upd,                 -- HOT updates (good — no index churn)
       n_live_tup,
       n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_vacuum,
       last_autovacuum,
       last_analyze,
       last_autoanalyze,
       vacuum_count,
       autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Tables with high seq_scan counts relative to idx_scan (possible missing indexes)
SELECT schemaname,
       relname,
       seq_scan,
       idx_scan,
       CASE WHEN idx_scan = 0 THEN 'no index scans'
            ELSE round(seq_scan::numeric / idx_scan, 1)::text
       END AS seq_to_idx_ratio
FROM pg_stat_user_tables
WHERE seq_scan > 1000
  AND schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY seq_scan DESC
LIMIT 20;
```

### 9.3 pg_stat_user_indexes: Are Your Indexes Being Used?

```sql
-- Find unused indexes (candidates for removal)
SELECT schemaname,
       relname AS table_name,
       indexrelname AS index_name,
       idx_scan,                      -- 0 = never used since last stats reset
       idx_tup_read,
       idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(indexrelid) DESC;

-- Note: Always check that the index has existed long enough to accumulate stats.
-- pg_stat_reset() resets all stats. Check pg_stat_bgwriter.stats_reset for timing.
SELECT stats_reset FROM pg_stat_bgwriter;
```

### 9.4 pg_statio_user_tables: Buffer Hit Rates

```sql
-- Cache hit rate per table (should be > 99% for hot tables)
SELECT schemaname,
       relname,
       heap_blks_read,               -- pages read from disk/OS cache
       heap_blks_hit,                -- pages served from shared_buffers
       round(100.0 * heap_blks_hit /
           NULLIF(heap_blks_read + heap_blks_hit, 0), 2) AS heap_hit_pct,
       idx_blks_read,
       idx_blks_hit,
       round(100.0 * idx_blks_hit /
           NULLIF(idx_blks_read + idx_blks_hit, 0), 2) AS idx_hit_pct
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY heap_blks_read DESC
LIMIT 20;

-- Overall database cache hit rate
SELECT round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0), 2)
           AS cache_hit_pct
FROM pg_stat_database
WHERE datname = current_database();
```

If cache hit rate is below 95% for tables that should be in cache, your `shared_buffers` is too small or your working set is too large.

### 9.5 pg_locks: What Is Blocking What

```sql
-- Find blocking queries (the lock chain)
WITH lock_info AS (
    SELECT
        a.pid,
        a.usename,
        a.application_name,
        l.granted,
        l.mode,
        l.locktype,
        l.relation::regclass AS table_name,
        left(a.query, 100) AS query,
        now() - a.query_start AS query_duration
    FROM pg_locks l
    JOIN pg_stat_activity a ON a.pid = l.pid
    WHERE l.locktype = 'relation'
)
SELECT
    w.pid AS waiting_pid,
    w.query AS waiting_query,
    w.query_duration AS waiting_duration,
    b.pid AS blocking_pid,
    b.query AS blocking_query,
    b.query_duration AS blocking_duration,
    b.table_name
FROM lock_info w
JOIN lock_info b ON w.table_name = b.table_name
                AND b.granted = true
                AND w.granted = false
WHERE w.mode IN ('ExclusiveLock', 'RowExclusiveLock', 'ShareLock',
                 'ShareUpdateExclusiveLock', 'AccessExclusiveLock');

-- Full lock dependency tree (recursive)
WITH RECURSIVE lock_tree AS (
    SELECT pid, usename, pg_blocking_pids(pid) AS blocked_by, query, state
    FROM pg_stat_activity
    WHERE cardinality(pg_blocking_pids(pid)) > 0
    UNION ALL
    SELECT a.pid, a.usename, pg_blocking_pids(a.pid), a.query, a.state
    FROM pg_stat_activity a
    JOIN lock_tree lt ON a.pid = ANY(lt.blocked_by)
)
SELECT * FROM lock_tree;
```

### 9.6 The DBA Monitoring Dashboard

Run these queries regularly (or pipe them into your monitoring system):

```sql
-- Query 1: Connection summary
SELECT state,
       count(*) AS count,
       max(now() - query_start) AS max_duration
FROM pg_stat_activity
WHERE pid != pg_backend_pid()
GROUP BY state
ORDER BY count DESC;

-- Query 2: Long-running transactions and queries
SELECT pid,
       usename,
       round(extract(epoch FROM now() - xact_start)::numeric, 1) AS txn_age_seconds,
       round(extract(epoch FROM now() - query_start)::numeric, 1) AS query_age_seconds,
       state,
       wait_event_type,
       wait_event,
       left(query, 150) AS query
FROM pg_stat_activity
WHERE (xact_start < NOW() - INTERVAL '60 seconds'
   OR query_start < NOW() - INTERVAL '30 seconds')
  AND state != 'idle'
  AND pid != pg_backend_pid()
ORDER BY COALESCE(xact_start, query_start);

-- Query 3: Tables needing vacuum attention
SELECT relname,
       n_dead_tup,
       n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_autovacuum,
       age(relfrozenxid) AS xid_age
FROM pg_stat_user_tables
JOIN pg_class ON relname = pg_class.relname AND relkind = 'r'
WHERE n_dead_tup > 10000
   OR age(relfrozenxid) > 100000000
ORDER BY dead_pct DESC NULLS LAST;

-- Query 4: XID wraparound risk
SELECT datname,
       age(datfrozenxid) AS xid_age,
       round(100.0 * age(datfrozenxid) / 2000000000, 2) AS pct_to_wraparound,
       pg_size_pretty(pg_database_size(datname)) AS db_size
FROM pg_database
WHERE datallowconn
ORDER BY age(datfrozenxid) DESC;

-- Query 5: Slowest queries from pg_stat_statements
SELECT round(mean_exec_time::numeric, 1) AS avg_ms,
       calls,
       round(total_exec_time::numeric / 1000, 1) AS total_seconds,
       round(rows::numeric / calls, 0) AS avg_rows,
       left(query, 150) AS query
FROM pg_stat_statements
WHERE calls > 50
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Query 6: Cache and checkpoint health
SELECT round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0), 2)
           AS db_cache_hit_pct,
       sum(blks_read) AS total_physical_reads,
       sum(blks_hit) AS total_cache_hits
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
UNION ALL
SELECT NULL, checkpoints_req::numeric, checkpoints_timed::numeric
FROM pg_stat_bgwriter;
```

---

## 10. The Complete Mental Model

### 10.1 From Query Arrival to Result Return

This is what actually happens when your application issues `SELECT * FROM orders WHERE customer_id = 42`:

```
1. APPLICATION
   └─► TCP packet arrives at PostgreSQL port 5432

2. POSTMASTER
   └─► Reads the incoming connection request
   └─► Authenticates via pg_hba.conf rules
   └─► If connection limit not reached: fork() a new backend process
   └─► If at max_connections: return "FATAL: sorry, too many clients already"

3. BACKEND PROCESS (freshly forked)
   └─► Receives the query string over the socket
   └─► Parser: lexes and parses SQL → parse tree
       └─► Syntax error? Return error immediately.
   └─► Analyzer: resolves names → query tree
       └─► Does table 'orders' exist? Does column 'customer_id' exist?
       └─► Checks permissions (has this user SELECT on orders?)
   └─► Rewriter: expands views, applies row-level security policies

4. PLANNER
   └─► Reads statistics from pg_statistic (pg_stats view)
       └─► How many rows in orders? What's the distribution of customer_id?
   └─► Generates candidate plans:
       Option A: Seq Scan on orders, filter customer_id = 42
       Option B: Index Scan on idx_orders_customer_id
   └─► Costs each plan using cost constants and statistics
   └─► Selects cheapest plan → plan tree

5. EXECUTOR
   └─► Walks the plan tree node by node
   └─► For Index Scan node:
       a. Looks up customer_id = 42 in the B-tree index
       b. Gets list of CTIDs (physical row locations): [(3,7), (3,9), (8,2)]
       c. For each CTID:
          i.  Compute page number (e.g., page 3)
          ii. Check if page 3 is in shared_buffers
              ─ HIT: use it directly (microseconds)
              ─ MISS: request I/O to load page 3 from disk into shared_buffers
         iii. Locate tuple at offset 7 on page 3
          iv. Check visibility: is xmin committed? Is xmax 0 or uncommitted?
              ─ VISIBLE: include in results
              ─ NOT VISIBLE: skip (dead tuple or not-yet-committed)
          v.  Add row to result set

6. MEMORY INVOLVEMENT
   └─► Index pages needed → check shared_buffers → load if absent
   └─► Heap pages needed → check shared_buffers → load if absent
   └─► If result needs sorting: allocate up to work_mem in backend memory
   └─► If work_mem exceeded: spill sort to temporary file in $PGDATA/base/pgsql_tmp/

7. RESULT RETURN
   └─► Rows streamed back to client over TCP as they are produced
   └─► Protocol: row description message, then data row messages, then command complete
   └─► Client receives rows and acknowledges

8. WRITE PATH (if this were an UPDATE instead)
   a. Acquire row lock (RowExclusiveLock)
   b. Create new row version in shared_buffers (xmin = current txid, xmax = 0)
   c. Mark old row version dead (xmax = current txid)
   d. Write WAL record to WAL buffer describing the change
   e. On COMMIT:
      - WAL writer calls fsync() on WAL buffer → WAL on disk
      - Return success to client
      - Modified data pages remain in shared_buffers (dirty, not yet on disk)
   f. Eventually:
      - Checkpointer or bgwriter flushes dirty pages to disk
      - WAL can be recycled up to the checkpoint LSN

9. CLEANUP (after the connection closes)
   └─► Backend process exits
   └─► Postmaster cleans up the slot in shared memory
   └─► MVCC: dead tuple left by UPDATE sits until VACUUM reclaims it
   └─► Autovacuum: eventually fires, removes dead tuples, updates FSM
```

### 10.2 The Layered View

```
Application Layer
    │  SQL query strings
    ▼
Protocol Layer (libpq / wire protocol)
    │  Messages: Parse, Bind, Execute, Sync
    ▼
Backend Process
    │  Parse → Analyze → Rewrite → Plan → Execute
    ▼
Shared Memory
    │  Shared buffers (8KB pages), WAL buffer, lock table, proc array
    ▼
WAL Layer
    │  WAL records → WAL segments (pg_wal/)
    │  Checkpoint → data pages flushed
    ▼
Heap & Index Files
    │  base/[db_oid]/[table_filenode]
    │  base/[db_oid]/[toast_filenode]
    ▼
OS Page Cache
    │  Second-level cache (transparent to PostgreSQL)
    ▼
Storage (SSD / NVMe / HDD / EBS)
```

### 10.3 The Key Decisions This Understanding Drives

**Connection count**: Every connection is a process. Use PgBouncer. Set `max_connections` to what you actually need plus 10% headroom, not an arbitrary large number.

**shared_buffers**: Your first-tier cache. 25% of RAM is the start. Use `pg_buffercache` to verify your working set fits.

**work_mem**: Per-operation, not per-connection. Set conservatively globally. Raise per-session for specific heavy queries.

**WAL settings**: `wal_level = replica` always. `fsync = on` always. `synchronous_commit` can be `off` for workloads that tolerate 200ms of potential data loss. `checkpoint_timeout` and `max_wal_size` control recovery time vs write I/O.

**Autovacuum**: Never disable. Tune scale factors down for large tables. Monitor XID age obsessively. Dead tuples are normal; excessive dead tuples are a configuration problem.

**Indexes**: Use EXPLAIN ANALYZE to verify index use. Check `pg_stat_user_indexes` for unused indexes (they cost write overhead with no read benefit). Check `pg_stat_user_tables` for tables with high seq_scan counts.

**random_page_cost**: Set to 1.1 on SSDs. This single change often dramatically improves plan quality by making index scans look more attractive.

**Statistics**: Run ANALYZE after bulk loads. Increase statistics targets for high-cardinality or skewed columns. Use `pg_stat_statements` to find the queries that matter.

> **The fundamental insight**: PostgreSQL is not a black box. Every slow query, every lock contention, every vacuum failure, every configuration tradeoff has a mechanical explanation. The system columns, the catalog views, the WAL, the buffer pool — they are all inspectable. Engineers who understand the internals do not guess at performance problems. They look at what the system is actually doing and fix the actual cause.

---

*Written for backend engineers who need to understand PostgreSQL deeply enough to make real decisions in production — not just repeat configuration recommendations without knowing why.*
