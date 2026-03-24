# Database Replication and High Availability

## How PostgreSQL Keeps Running When Everything Tries to Break It

> Replication is not a backup strategy. Failover is not a fire drill. High availability is not a feature you add later. This guide is about understanding the real mechanics — so that when a primary dies at 3 AM, you already know exactly what to do.

---

## Table of Contents

1. [Why Replication Exists](#1-why-replication-exists)
   - 1.1 [Single Point of Failure](#11-single-point-of-failure)
   - 1.2 [Read Scalability](#12-read-scalability)
   - 1.3 [Disaster Recovery](#13-disaster-recovery)
   - 1.4 [Zero-Downtime Deployments](#14-zero-downtime-deployments)
   - 1.5 [The CAP Theorem Applied to PostgreSQL](#15-the-cap-theorem-applied-to-postgresql)
2. [How PostgreSQL Streaming Replication Works](#2-how-postgresql-streaming-replication-works)
   - 2.1 [WAL-Based Replication: The Core Mechanism](#21-wal-based-replication-the-core-mechanism)
   - 2.2 [WAL Sender and WAL Receiver Processes](#22-wal-sender-and-wal-receiver-processes)
   - 2.3 [Replication Lag: What It Is and What Causes It](#23-replication-lag-what-it-is-and-what-causes-it)
   - 2.4 [Synchronous vs Asynchronous Replication](#24-synchronous-vs-asynchronous-replication)
   - 2.5 [synchronous_commit Settings in Depth](#25-synchronous_commit-settings-in-depth)
   - 2.6 [Replication Slots: Power and Danger](#26-replication-slots-power-and-danger)
3. [Setting Up Streaming Replication](#3-setting-up-streaming-replication)
   - 3.1 [Primary: postgresql.conf Settings](#31-primary-postgresqlconf-settings)
   - 3.2 [Primary: pg_hba.conf for Replication](#32-primary-pg_hbaconf-for-replication)
   - 3.3 [Creating the Replication User](#33-creating-the-replication-user)
   - 3.4 [Initializing the Replica with pg_basebackup](#34-initializing-the-replica-with-pg_basebackup)
   - 3.5 [Configuring the Replica (PostgreSQL 12+)](#35-configuring-the-replica-postgresql-12)
   - 3.6 [Verifying Replication is Working](#36-verifying-replication-is-working)
4. [Replication Lag — Monitoring and Reducing It](#4-replication-lag--monitoring-and-reducing-it)
   - 4.1 [Measuring Lag with pg_stat_replication](#41-measuring-lag-with-pg_stat_replication)
   - 4.2 [What Causes Lag](#42-what-causes-lag)
   - 4.3 [Intentional Lag: recovery_min_apply_delay](#43-intentional-lag-recovery_min_apply_delay)
   - 4.4 [When to Act on Lag vs When to Accept It](#44-when-to-act-on-lag-vs-when-to-accept-it)
5. [Logical Replication — When and Why](#5-logical-replication--when-and-why)
   - 5.1 [Streaming vs Logical: The Key Difference](#51-streaming-vs-logical-the-key-difference)
   - 5.2 [Publications and Subscriptions](#52-publications-and-subscriptions)
   - 5.3 [Use Cases](#53-use-cases)
   - 5.4 [Limitations You Must Know](#54-limitations-you-must-know)
   - 5.5 [Complete Logical Replication Setup](#55-complete-logical-replication-setup)
6. [Read Replicas — Scaling Reads](#6-read-replicas--scaling-reads)
   - 6.1 [Routing Reads to Replicas](#61-routing-reads-to-replicas)
   - 6.2 [The Stale Read Problem](#62-the-stale-read-problem)
   - 6.3 [Hot Standby and Query Conflicts](#63-hot-standby-and-query-conflicts)
   - 6.4 [hot_standby_feedback](#64-hot_standby_feedback)
7. [Connection Pooling with PgBouncer](#7-connection-pooling-with-pgbouncer)
   - 7.1 [Why PostgreSQL's Process Model Breaks at Scale](#71-why-postgresqls-process-model-breaks-at-scale)
   - 7.2 [PgBouncer Modes: Session, Transaction, Statement](#72-pgbouncer-modes-session-transaction-statement)
   - 7.3 [Complete pgbouncer.ini Configuration](#73-complete-pgbouncerini-configuration)
   - 7.4 [Pool Sizing: The Math](#74-pool-sizing-the-math)
   - 7.5 [What PgBouncer Does Not Solve](#75-what-pgbouncer-does-not-solve)
   - 7.6 [PgBouncer + Django: Full Setup](#76-pgbouncer--django-full-setup)
8. [Failover — When the Primary Dies](#8-failover--when-the-primary-dies)
   - 8.1 [Manual Failover: Promoting a Replica](#81-manual-failover-promoting-a-replica)
   - 8.2 [Automatic Failover: Patroni](#82-automatic-failover-patroni)
   - 8.3 [The Split-Brain Problem](#83-the-split-brain-problem)
   - 8.4 [Fencing: How Patroni Prevents Split-Brain](#84-fencing-how-patroni-prevents-split-brain)
   - 8.5 [Patroni Failover: Step by Step](#85-patroni-failover-step-by-step)
9. [High Availability Architecture Patterns](#9-high-availability-architecture-patterns)
   - 9.1 [Single Primary + 1 Sync + 1 Async Replica](#91-single-primary--1-sync--1-async-replica)
   - 9.2 [Multi-Region Setup](#92-multi-region-setup)
   - 9.3 [RTO vs RPO: Defining Your Requirements](#93-rto-vs-rpo-defining-your-requirements)
   - 9.4 [Backup + Replication: Different Problems](#94-backup--replication-different-problems)
10. [Monitoring Replication Health](#10-monitoring-replication-health)
    - 10.1 [pg_stat_replication: Every Column Explained](#101-pg_stat_replication-every-column-explained)
    - 10.2 [pg_stat_wal_receiver: The Replica Side](#102-pg_stat_wal_receiver-the-replica-side)
    - 10.3 [Complete Monitoring Query Set](#103-complete-monitoring-query-set)
    - 10.4 [What to Alert On](#104-what-to-alert-on)
11. [Common Replication Problems and Fixes](#11-common-replication-problems-and-fixes)
    - 11.1 [Replica Falling Too Far Behind](#111-replica-falling-too-far-behind)
    - 11.2 [Replication Slot Bloating WAL Disk](#112-replication-slot-bloating-wal-disk)
    - 11.3 [Promoting the Wrong Replica](#113-promoting-the-wrong-replica)
    - 11.4 [Application Not Handling Failover](#114-application-not-handling-failover)
    - 11.5 [Schema Changes Breaking Replication](#115-schema-changes-breaking-replication)

---

## 1. Why Replication Exists

Replication is not one feature solving one problem. It solves several distinct problems simultaneously — and each reason for running it implies different configuration choices.

### 1.1 Single Point of Failure

A single PostgreSQL instance means that when the machine running it fails — hardware fault, kernel panic, storage corruption, accidental `rm -rf` — your application is down until you restore from backup. That restoration process, even with a perfect backup, takes time: minutes to hours depending on database size and infrastructure.

A replica changes this: you already have a warm (or hot) copy of the data, continuously updated, ready to accept connections. The question is no longer "how fast can we restore?" but "how fast can we redirect traffic?"

The failure modes a replica protects against:

- Physical server failure (disk, CPU, power, NIC)
- OS-level failure requiring a reboot
- PostgreSQL process crash requiring restart
- Datacenter/availability zone outage (with cross-AZ replicas)
- Accidental or malicious data destruction (with delayed replicas — see Section 4.3)

The failure modes a replica does **not** protect against:

- Logical data corruption (a bad UPDATE runs on the primary and replicates to all replicas)
- Human error that replicates before you catch it
- These are backup problems, not replication problems

### 1.2 Read Scalability

PostgreSQL is CPU-bound on complex queries and I/O-bound on sequential scans. A single primary can become a read bottleneck when many clients issue concurrent `SELECT` queries.

Replicas accept read-only queries. You can route reporting workloads, analytics queries, and background data exports to replicas, leaving the primary free for writes and latency-sensitive reads.

This is not free: you now have to think about read consistency (Section 6.2). A replica might be seconds behind the primary. A user who just saved a record and immediately queries a replica might not see it yet.

### 1.3 Disaster Recovery

A replica in the same datacenter protects against server failure but not datacenter failure. A cross-region replica protects against datacenter loss. This is the disaster recovery (DR) use case.

Cross-region replication is almost always asynchronous — the network latency between regions (50–150 ms round trip) makes synchronous replication impractical because every write would block waiting for the remote acknowledgment.

With async cross-region replication, you accept RPO > 0: if the primary's region disappears, some recent writes may be lost.

### 1.4 Zero-Downtime Deployments

Replicas enable upgrade patterns that would otherwise require maintenance windows:

**Blue-green database deployments:** bring up a new database version alongside the old one, migrate data, cut over.

**PostgreSQL major version upgrades with logical replication:** run PostgreSQL 15 and PostgreSQL 16 in parallel with logical replication from old to new. Once the new version is caught up, redirect the application. Downtime is measured in seconds for the DNS/load-balancer flip, not in hours for `pg_upgrade`.

**Schema migrations:** run a migration on a replica first, verify it, then apply to primary. Or use tools like `pg_repack` that leverage logical replication to rebuild tables without locking.

### 1.5 The CAP Theorem Applied to PostgreSQL

The CAP theorem says a distributed system can guarantee at most two of: Consistency, Availability, Partition Tolerance.

PostgreSQL replication forces you to choose explicitly:

**Synchronous replication (CP):** Every commit waits for at least one replica to acknowledge. If the replica is unreachable, the primary blocks writes (or errors, depending on `synchronous_commit` setting). You get consistency — the replica is never behind — but availability suffers when the replica is gone.

**Asynchronous replication (AP):** Every commit completes when the primary has written WAL locally. The replica catches up when it can. If the primary fails, you promote the replica and accept that some recent writes may be lost (RPO > 0). You get availability — writes never block — but consistency suffers.

There is no free lunch. The choice is determined by your application's tolerance for data loss vs tolerance for write latency.

> **Production decision:** For most OLTP workloads, run one synchronous replica for durability and one asynchronous replica for DR. The sync replica gives you RPO=0 on local failure. The async replica gives you availability if the sync replica goes down. This is the "1 sync + 1 async" pattern covered in Section 9.

---

## 2. How PostgreSQL Streaming Replication Works

### 2.1 WAL-Based Replication: The Core Mechanism

Everything PostgreSQL replication does is built on the WAL (Write-Ahead Log). Before any data page is modified, the change is written to WAL. This ensures durability: even if PostgreSQL crashes mid-write, the WAL is replayed on restart.

Replication repurposes this same WAL stream. Instead of only writing WAL locally for crash recovery, the primary also ships that WAL to replicas. Replicas replay the exact same WAL records, producing identical data files.

The flow:

```
Client COMMIT
      │
      ▼
WAL Record Written to pg_wal/
      │
      ├──► WAL Flushed to Disk (fsync) — local durability
      │
      ├──► WAL Sender Process ships WAL to Replica
      │
      ▼
Client gets COMMIT response (async) or waits for replica ACK (sync)


On Replica:
WAL Receiver receives WAL
      │
      ▼
WAL written to replica's pg_wal/
      │
      ▼
Startup Process replays WAL records
      │
      ▼
Data files on replica match primary
```

This is called **physical replication** — it replicates the exact binary changes to data files. The replica is a byte-for-byte copy of the primary at some point in time. It cannot have a different schema, different encoding, or different PostgreSQL major version.

### 2.2 WAL Sender and WAL Receiver Processes

On the **primary**, a **WAL sender** process is spawned for each replica connection. You can see them:

```bash
ps aux | grep "wal sender"
# postgres: walsender replicator 10.0.0.102(54321) streaming 0/5A3BC10
```

The WAL sender:

1. Reads WAL from `pg_wal/` starting from the replica's current LSN (Log Sequence Number)
2. Streams WAL records over the replication connection
3. Receives confirmation (feedback) from the replica: which LSN has been written, flushed, and applied
4. Reports this to `pg_stat_replication`

On the **replica**, a **WAL receiver** process handles the other side:

```bash
ps aux | grep "wal receiver"
# postgres: walreceiver   streaming 0/5A3BC10
```

The WAL receiver:

1. Connects to the primary using `primary_conninfo`
2. Sends its current LSN (so the primary knows where to start streaming)
3. Receives WAL records and writes them to the replica's `pg_wal/`
4. Sends feedback messages back to the primary: write_lsn, flush_lsn, replay_lsn
5. Reports status to `pg_stat_wal_receiver`

A separate **startup process** on the replica reads the WAL from `pg_wal/` and applies it to the data files. This is the actual replay step. The WAL receiver and startup process are separate: WAL can arrive faster than it can be replayed, creating a replay queue.

### 2.3 Replication Lag: What It Is and What Causes It

Lag is the gap between the primary's current write position and where the replica is. It has three distinct phases, each with its own cause:

```
Primary writes WAL at LSN X
         │
         │  Network transit (write_lag)
         ▼
Replica receives WAL, writes to pg_wal/
         │
         │  fsync to disk (flush_lag)
         ▼
Replica flushed WAL to disk
         │
         │  Replay process applies changes (replay_lag)
         ▼
Replica's data files reflect LSN X
```

`pg_stat_replication` shows each phase as a `INTERVAL`:

- `write_lag`: time between primary flush and replica write to disk
- `flush_lag`: time between primary flush and replica fsync
- `replay_lag`: time between primary flush and replica applying to data files

The most visible lag is `replay_lag`, because this is what determines how stale a query on the replica is.

LSN-based lag (in bytes) is also important. A replica 500ms behind in time might be 10 MB or 10 GB of WAL behind, depending on write rate. A replica that is 10 GB of WAL behind cannot catch up quickly if writes continue at full speed.

### 2.4 Synchronous vs Asynchronous Replication

**Asynchronous (default):**

```
Primary writes WAL → Primary acks COMMIT to client → Replica catches up later
```

The client gets a fast response. If the primary fails immediately after the commit, the replica may not have that commit yet. This is RPO > 0 (potential data loss on failover).

**Synchronous:**

```
Primary writes WAL → Primary sends WAL to replica → Replica ACKs → Primary acks COMMIT to client
```

The client waits. The commit is not confirmed until at least one replica has acknowledged. If that replica is unreachable, the primary blocks (or times out, depending on configuration). RPO = 0 for the failure scenarios where the primary can communicate with the replica before failing.

To enable synchronous replication, on the primary:

```ini
# postgresql.conf
synchronous_standby_names = 'replica1'
# or for any one replica out of a list:
synchronous_standby_names = 'ANY 1 (replica1, replica2)'
# or first-listed replica must confirm (FIRST mode):
synchronous_standby_names = 'FIRST 1 (replica1, replica2)'
```

The replica advertises its name via:

```ini
# postgresql.conf on replica
application_name = 'replica1'
```

> **Production decision:** Do not run synchronous replication with a single replica and `synchronous_commit = on` unless you have a plan for what happens when the replica goes down. If the replica disconnects and `synchronous_standby_names` is set, writes on the primary will hang indefinitely. Use `ANY 1 (replica1, replica2)` to require one of two replicas to respond, so a single replica failure doesn't block writes.

### 2.5 synchronous_commit Settings in Depth

`synchronous_commit` is the most nuanced knob in replication configuration. It controls what the primary waits for before confirming a commit to the client.

| Value          | Meaning                                     | Durability                                       | Performance |
| -------------- | ------------------------------------------- | ------------------------------------------------ | ----------- |
| `off`          | Don't wait for local WAL flush              | No guarantee — data can be lost on primary crash | Fastest     |
| `local`        | Wait for local WAL flush only               | Primary durability, no replica guarantee         | Fast        |
| `remote_write` | Wait for replica to write WAL to OS buffer  | Replica OS buffer (lost if OS crashes)           | Medium      |
| `remote_apply` | Wait for replica to apply WAL to data files | Replica can serve this data immediately          | Slower      |
| `on` (default) | Wait for replica to flush WAL to disk       | Full synchronous durability                      | Slowest     |

The default `on` means a commit waits for the replica to flush WAL (fsync) — not just write to the OS buffer. This is the strongest guarantee.

`remote_apply` is useful when you want to read your own writes from the replica: you know that after a commit, the replica has applied it. Without `remote_apply`, you could commit on primary and immediately query the replica and not see your data.

`off` is often misunderstood. It disables the wait for even the **local** fsync. This means a crash of the primary right after a commit could lose that commit. The database remains consistent (no partial writes, no corruption), but committed transactions can vanish. Use this only for bulk loads where you're going to re-run on failure anyway, never for OLTP.

You can set `synchronous_commit` per transaction:

```sql
-- Per-session override for a bulk import
SET synchronous_commit = off;
-- Do your bulk work
RESET synchronous_commit;

-- Or per-transaction
BEGIN;
SET LOCAL synchronous_commit = off;
INSERT INTO bulk_table ...;
COMMIT;
```

### 2.6 Replication Slots: Power and Danger

A replication slot is a named cursor that tracks how far a consumer (a replica or a logical replication subscriber) has consumed the WAL. PostgreSQL guarantees it will not delete WAL that the slot has not yet consumed.

```sql
-- Create a physical replication slot
SELECT pg_create_physical_replication_slot('replica1_slot');

-- View all slots
SELECT slot_name, slot_type, active, active_pid,
       restart_lsn, confirmed_flush_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag_size
FROM pg_replication_slots;
```

**Why slots exist:** Without a slot, if a replica disconnects for several hours, the primary might delete WAL segments (based on `wal_keep_size`) before the replica returns. The replica can no longer catch up from streaming — it needs a full `pg_basebackup` reinitializaiton. Slots prevent this.

**The danger of slots:** If a replica using a slot goes offline and never reconnects, the slot remains active. The primary cannot delete any WAL past that slot's `restart_lsn`. Your `pg_wal/` directory will grow without bound until the disk fills. This has taken down production databases.

```sql
-- DANGER: A slot that hasn't moved in hours means WAL is accumulating
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;

-- Drop a slot that belongs to a dead replica
SELECT pg_drop_replication_slot('dead_replica_slot');
```

> **Production rule:** Always monitor slot lag. Alert when `pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)` exceeds 10 GB. Consider dropping slots for replicas that have been offline for more than a few hours, and reinitialize those replicas from `pg_basebackup` when they return.

Patroni (Section 8) manages slot lifecycle automatically: it creates and drops slots as replicas come and go, preventing this class of problem.

---

## 3. Setting Up Streaming Replication

This is the complete walkthrough. Every command. Every file. No gaps.

**Environment:**

- Primary: `10.0.0.10`, PostgreSQL 16
- Replica: `10.0.0.11`, PostgreSQL 16 (same version required for physical replication)
- PostgreSQL data directory: `/var/lib/postgresql/16/main`

### 3.1 Primary: postgresql.conf Settings

```ini
# /etc/postgresql/16/main/postgresql.conf on PRIMARY

# WAL level must be 'replica' or 'logical' for replication
# 'minimal' does not produce enough WAL for replication
wal_level = replica

# Number of concurrent WAL sender processes
# Set to number of replicas + 2 (for tools like pg_basebackup)
max_wal_senders = 10

# How much WAL to retain in pg_wal/ for replicas that fall behind
# This is the fallback when NOT using replication slots
# 1GB is often enough for brief network interruptions
wal_keep_size = 1024   # in MB

# If using synchronous replication, name your replicas here
# Leave empty for async-only setup
synchronous_standby_names = ''

# Required for replicas to run queries (hot standby)
# This is the default in PostgreSQL 10+
hot_standby = on

# Archive settings (needed for PITR, optional for basic replication)
# archive_mode = on
# archive_command = 'cp %p /mnt/wal_archive/%f'

# Listen on all interfaces so the replica can connect
listen_addresses = '*'
```

Reload configuration (no restart needed for most of these):

```bash
# On primary
sudo -u postgres psql -c "SELECT pg_reload_conf();"
# wal_level change requires restart:
sudo systemctl restart postgresql@16-main
```

### 3.2 Primary: pg_hba.conf for Replication

```ini
# /etc/postgresql/16/main/pg_hba.conf on PRIMARY
# Add a line to allow the replication user to connect

# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    replication     replicator      10.0.0.11/32            scram-sha-256

# If you have multiple replicas in a subnet:
# host    replication     replicator      10.0.0.0/24             scram-sha-256
```

Reload:

```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

### 3.3 Creating the Replication User

```sql
-- On PRIMARY
CREATE USER replicator WITH REPLICATION LOGIN PASSWORD 'strong_password_here';

-- Optional: if you're using pg_basebackup, the user also needs login permission
-- REPLICATION attribute already implies LOGIN, but be explicit:
-- CREATE USER replicator WITH REPLICATION LOGIN PASSWORD 'strong_password_here';

-- Verify
SELECT usename, userepl FROM pg_user WHERE usename = 'replicator';
--  usename   | userepl
-- -----------+---------
--  replicator| t
```

### 3.4 Initializing the Replica with pg_basebackup

`pg_basebackup` creates a consistent copy of the primary's data directory. It uses the replication protocol, so it captures a point-in-time snapshot with the WAL needed to make it consistent.

```bash
# On REPLICA machine
# Stop PostgreSQL if it's running (we're about to overwrite the data directory)
sudo systemctl stop postgresql@16-main

# Remove existing data directory content
sudo -u postgres rm -rf /var/lib/postgresql/16/main/*

# Run pg_basebackup
# -h: primary host
# -U: replication user
# -D: target directory on replica
# -P: progress reporting
# -Xs: stream WAL during backup (ensures we have WAL from start to end of backup)
# -R: write a standby.signal file and primary_conninfo to postgresql.auto.conf
sudo -u postgres pg_basebackup \
    -h 10.0.0.10 \
    -U replicator \
    -D /var/lib/postgresql/16/main \
    -P \
    -Xs \
    -R

# You will be prompted for the password
# Output looks like:
# 25159/25159 kB (100%), 1/1 tablespace

# Verify the standby.signal file was created
ls -la /var/lib/postgresql/16/main/standby.signal
```

The `-R` flag is critical: it writes the connection info into `postgresql.auto.conf` and creates `standby.signal`, which tells PostgreSQL to start in standby (replica) mode instead of normal mode.

### 3.5 Configuring the Replica (PostgreSQL 12+)

In PostgreSQL 12, `recovery.conf` was eliminated. Standby configuration now lives in `postgresql.conf` (or `postgresql.auto.conf`), and the presence of `standby.signal` in the data directory tells PostgreSQL to start as a standby.

If you used `pg_basebackup -R`, this was done automatically. If not, or if you need to adjust:

```ini
# /var/lib/postgresql/16/main/postgresql.auto.conf on REPLICA
# (this file is auto-generated and takes precedence over postgresql.conf)

primary_conninfo = 'host=10.0.0.10 port=5432 user=replicator password=strong_password_here application_name=replica1'

# If using a replication slot (created on primary first):
# primary_slot_name = 'replica1_slot'

# Intentional replay delay (for disaster recovery replicas):
# recovery_min_apply_delay = '1h'
```

```bash
# Ensure standby.signal exists
touch /var/lib/postgresql/16/main/standby.signal
chown postgres:postgres /var/lib/postgresql/16/main/standby.signal

# Start the replica
sudo systemctl start postgresql@16-main
```

**Pre-12 (PostgreSQL 11 and earlier):** you wrote a `recovery.conf` file in the data directory with `standby_mode = 'on'`, `primary_conninfo`, etc. This file is no longer used in 12+.

### 3.6 Verifying Replication is Working

**On the primary:**

```sql
-- pg_stat_replication shows connected standbys
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    sync_state
FROM pg_stat_replication;

-- Expected output (replica is healthy, minimal lag):
--  pid  | usename   | application_name | client_addr | state     | sent_lsn  | write_lsn | flush_lsn | replay_lsn | write_lag | flush_lag | replay_lag | sync_state
-- ------+-----------+------------------+-------------+-----------+-----------+-----------+-----------+------------+-----------+-----------+------------+------------
--  3421 | replicator| replica1         | 10.0.0.11   | streaming | 0/5A3BC10 | 0/5A3BC10 | 0/5A3BC10 | 0/5A3BC10  | 00:00:00  | 00:00:00  | 00:00:00   | async
```

If `state` is `streaming`, replication is live. If it shows `startup` or `catchup`, the replica is still applying the base backup WAL.

**On the replica:**

```sql
-- pg_stat_wal_receiver shows connection to primary
SELECT
    status,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time,
    slot_name,
    conninfo
FROM pg_stat_wal_receiver;
```

**Quick sanity test:**

```sql
-- On PRIMARY: write something
CREATE TABLE replication_test (id SERIAL, msg TEXT, ts TIMESTAMPTZ DEFAULT now());
INSERT INTO replication_test (msg) VALUES ('hello from primary');

-- On REPLICA: read it back (within seconds)
SELECT * FROM replication_test;
-- Should see the row

-- Confirm you're on the replica (read-only)
SELECT pg_is_in_recovery();
-- Returns: t (true = standby/replica)

-- On PRIMARY:
SELECT pg_is_in_recovery();
-- Returns: f (false = primary)
```

---

## 4. Replication Lag — Monitoring and Reducing It

### 4.1 Measuring Lag with pg_stat_replication

The authoritative source for lag is `pg_stat_replication` on the primary. Understanding every column is essential.

```sql
SELECT
    application_name,
    -- Where we've sent WAL up to
    sent_lsn,
    -- Where the replica has written it to disk (OS buffer)
    write_lsn,
    -- Where the replica has flushed to disk (fsync'd)
    flush_lsn,
    -- Where the replica has applied it to data files
    replay_lsn,
    -- Time-based lag measurements (calculated from primary's perspective)
    write_lag,
    flush_lag,
    replay_lag,
    -- Byte-based lag (how much WAL the replica still needs to process)
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS total_bytes_behind,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag_pretty,
    sync_state  -- async, sync, potential, quorum
FROM pg_stat_replication;
```

LSN (Log Sequence Number) is a monotonically increasing position in the WAL stream, expressed as `segment/offset` in hex (e.g., `0/5A3BC10`). `pg_wal_lsn_diff(a, b)` returns the difference in bytes.

**The pipeline of lag:**

```
Primary flushes WAL at LSN X
         │
         ├── sent_lsn: we've sent up to here to the replica
         │
         │   [network transit]
         │
         ├── write_lsn: replica wrote to OS buffer
         │
         │   [replica fsync]
         │
         ├── flush_lsn: replica flushed to disk
         │
         │   [startup process applies]
         │
         └── replay_lsn: replica applied to data files
```

For read consistency, `replay_lsn` is the relevant number: this is what queries on the replica actually see.

You can also check lag from the **replica side**:

```sql
-- On the replica
SELECT
    now() - pg_last_xact_replay_timestamp() AS replication_delay,
    pg_is_in_recovery() AS is_standby,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn,
    pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS receive_replay_gap_bytes;
```

`pg_last_xact_replay_timestamp()` returns the timestamp of the last transaction applied. `now() - pg_last_xact_replay_timestamp()` gives you human-readable lag.

**Caveat:** If the primary has no write activity, `pg_last_xact_replay_timestamp()` doesn't advance even though replication is healthy. A 5-minute lag on a quiet primary is not a lag problem. This is why byte-based lag from `pg_stat_replication` is often more reliable than time-based lag on idle systems.

### 4.2 What Causes Lag

**Write bursts on the primary.** A bulk `INSERT`, a large `UPDATE`, or a `VACUUM` that modifies many pages generates a lot of WAL rapidly. The replica's startup process may not apply WAL as fast as it arrives. The WAL receiver buffers it, but if the burst is large enough, the replica falls behind.

**Slow replica disk.** WAL must be written and flushed on the replica before it can be replayed. If the replica's disk has slower random write performance than the primary's, flush_lag grows. Fix: give replicas the same disk tier as the primary, or use NVMe.

**Network latency and bandwidth.** Cross-region replicas are fundamentally limited by network RTT. A 100 Mbps link saturated by other traffic will cause WAL to queue up on the primary. Fix: dedicated replication network, WAL compression.

**Expensive queries on the replica.** If hot_standby is enabled and long queries run on the replica, they can interfere with WAL replay. The startup process must wait for conflicting queries to finish (or cancel them). See Section 6.3.

**Checkpoints.** During a checkpoint on the primary, a burst of dirty pages is written. This generates I/O pressure that can momentarily slow WAL writing, but more often the issue is that the checkpoint generates a large number of WAL records for the full-page writes (FPW) of pages modified for the first time since the last checkpoint.

### 4.3 Intentional Lag: recovery_min_apply_delay

You can configure a replica to deliberately stay behind the primary by a fixed time interval. This creates a **delayed replica** — a rolling window into the past.

```ini
# postgresql.auto.conf on the DELAYED REPLICA
recovery_min_apply_delay = '2h'
```

With a 2-hour delay, the replica is always showing data as it was 2 hours ago. If you realize at 10:00 AM that a developer ran `DELETE FROM orders WHERE status = 'pending'` at 9:30 AM and it replicated to all sync/async replicas, your delayed replica still has the data. You can:

1. Stop WAL replay on the delayed replica (`SELECT pg_wal_replay_pause();`)
2. Export the affected data
3. Restore it to the primary

```sql
-- On the delayed replica: pause at a specific point before the incident
SELECT pg_wal_replay_pause();

-- Check what time the replica is currently at
SELECT now() - pg_last_xact_replay_timestamp() AS delay,
       pg_last_xact_replay_timestamp() AS replica_time;

-- Query the data as of that point in time
SELECT * FROM orders WHERE status = 'pending';

-- Export it
COPY (SELECT * FROM orders WHERE status = 'pending') TO '/tmp/rescued_orders.csv' CSV HEADER;

-- Resume replay when done
SELECT pg_wal_replay_resume();
```

> **Production decision:** A 1-2 hour delayed replica is cheap insurance against human error. The hardware cost is one extra server. The benefit is a recovery option that doesn't require restoring a multi-terabyte backup from S3.

### 4.4 When to Act on Lag vs When to Accept It

**Act immediately if:**

- Lag is growing continuously (the replica is falling further behind with no sign of catching up)
- Byte-lag exceeds 10 GB (the replica might need a full reinitialize if the primary's WAL rotates faster than `wal_keep_size`)
- A replication slot has inactive=true and its `restart_lsn` is not advancing (disk bloat imminent)

**Accept it if:**

- Lag spikes during a known write burst (bulk import, end-of-month report run) but recovers afterward
- Cross-region async replica lags by 200-500ms consistently — this is the physics of the network
- A delayed replica shows 2 hours of lag — that's by design

**Never accept:**

- Lag on a synchronous replica (by definition it cannot lag while synchronous_commit is active; if it appears to lag in monitoring, something is wrong with your monitoring)
- Growing byte-lag on any replica used for failover — this is your RPO materializing

---

## 5. Logical Replication — When and Why

### 5.1 Streaming vs Logical: The Key Difference

Physical (streaming) replication copies the raw WAL binary stream. The replica applies the exact same byte-level changes to its data files. The result is a perfect physical copy of the entire cluster.

Logical replication decodes the WAL into logical operations — `INSERT`, `UPDATE`, `DELETE` at the row level — and applies those to a subscriber. The subscriber can be a different PostgreSQL version, a different schema, or even a non-PostgreSQL database.

| Property             | Physical/Streaming            | Logical                          |
| -------------------- | ----------------------------- | -------------------------------- |
| Granularity          | Entire cluster                | Per-table, per-publication       |
| Cross-version        | No (must match major version) | Yes (source and dest can differ) |
| Subscriber can write | No (read-only standby)        | Yes                              |
| DDL replication      | Yes (everything)              | No (DDL not replicated)          |
| Sequence replication | Yes                           | No                               |
| Overhead             | Lower                         | Higher (decoding cost)           |
| Primary use case     | HA, failover                  | Selective sync, upgrades, ETL    |

### 5.2 Publications and Subscriptions

Logical replication uses a publisher-subscriber model.

**Publisher (source database):** A publication defines what tables to replicate and what operations (INSERT, UPDATE, DELETE, TRUNCATE).

**Subscriber (destination database):** A subscription connects to a publisher, creates a copy of the published tables, and continuously applies changes.

### 5.3 Use Cases

**Major version upgrades (this is the main one):** You can run PostgreSQL 15 and 16 side by side. Create a publication on 15, subscribe from 16. Once 16 is caught up, redirect the application with seconds of downtime for the DNS flip.

**Selective table replication:** Replicate only the `orders` table to a separate analytics database. The analytics team gets their own PostgreSQL instance with write access for aggregations, receiving real-time `orders` data without touching the production primary.

**Data integration:** Replicate from PostgreSQL to another PostgreSQL (possibly in a different organization's infrastructure) or use logical decoding plugins (Debezium, pglogical) to stream changes to Kafka, Elasticsearch, or data warehouses.

**Tenant data isolation:** Shard tenants across databases; replicate a subset of tables to a consolidated reporting database.

### 5.4 Limitations You Must Know

**No DDL replication.** If you `ALTER TABLE orders ADD COLUMN discount NUMERIC` on the source, that change does not replicate. The subscription will break because the replication tries to insert rows into a table with a different schema on the destination. You must apply DDL on the destination first, then on the source.

**No sequence replication.** Sequences (`SERIAL`, `BIGSERIAL`, `GENERATED ALWAYS AS IDENTITY`) do not replicate. If you fail over to a logical replica, your sequences will be behind. You must manually advance them or use a different ID strategy (UUIDs, distributed ID generators).

**Initial copy.** When a subscription is created, PostgreSQL copies all existing data (table sync phase) before switching to streaming. This initial copy takes time for large tables and generates significant I/O.

**Row-level filters (14+).** PostgreSQL 14 added row filtering on publications: `CREATE PUBLICATION pub FOR TABLE orders WHERE (status = 'active')`. Only rows matching the filter are replicated.

**Column-level filtering (15+).** PostgreSQL 15 added column lists: `CREATE PUBLICATION pub FOR TABLE orders (id, amount, status)`. Useful to avoid replicating sensitive columns.

### 5.5 Complete Logical Replication Setup

**Scenario:** Replicate the `orders` and `products` tables from a production database to an analytics database.

**On the source (production primary):**

```ini
# postgresql.conf
wal_level = logical   # Must be 'logical', not just 'replica'
max_replication_slots = 10
max_wal_senders = 10
```

```sql
-- Create a user for logical replication
CREATE USER logical_replicator WITH REPLICATION LOGIN PASSWORD 'strong_pass';

-- Grant SELECT on the tables being published
GRANT SELECT ON orders, products TO logical_replicator;

-- Create the publication
CREATE PUBLICATION analytics_pub
FOR TABLE orders, products
WITH (publish = 'insert, update, delete');

-- Or replicate all tables in the schema:
-- CREATE PUBLICATION analytics_pub FOR ALL TABLES;

-- Verify
SELECT pubname, puballtables, pubinsert, pubupdate, pubdelete
FROM pg_publication;
```

```ini
# pg_hba.conf on source
host    replication     logical_replicator   10.0.0.20/32   scram-sha-256
host    mydb            logical_replicator   10.0.0.20/32   scram-sha-256
# (logical replication needs both replication and database access)
```

**On the destination (analytics database):**

```sql
-- Tables must exist on the destination with compatible schemas
-- Create them (DDL is not replicated, you must do this manually):
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    user_id     BIGINT,
    amount      NUMERIC(10,2),
    status      TEXT,
    created_at  TIMESTAMPTZ
);

CREATE TABLE products (
    id    BIGINT PRIMARY KEY,
    name  TEXT,
    price NUMERIC(10,2)
);

-- Create the subscription
CREATE SUBSCRIPTION analytics_sub
CONNECTION 'host=10.0.0.10 port=5432 dbname=mydb user=logical_replicator password=strong_pass'
PUBLICATION analytics_pub;

-- Monitor the initial sync
SELECT subname, pid, relid::regclass, received_lsn, last_msg_send_time, last_msg_receipt_time
FROM pg_stat_subscription;

-- Check sync status per table
SELECT * FROM pg_subscription_rel;
-- srsubstate = 'r' means ready (initial sync complete, streaming live changes)
-- srsubstate = 's' means syncing (initial copy in progress)
-- srsubstate = 'i' means initializing

-- Verify data is flowing
SELECT count(*) FROM orders;
```

**Handling DDL changes during logical replication:**

```sql
-- Wrong: Apply on source first — breaks replication
-- ALTER TABLE orders ADD COLUMN discount NUMERIC;  -- DO NOT DO THIS FIRST

-- Correct sequence:
-- 1. Apply DDL on destination first
ALTER TABLE orders ADD COLUMN discount NUMERIC;   -- On destination

-- 2. Then apply DDL on source
ALTER TABLE orders ADD COLUMN discount NUMERIC;   -- On source
-- Replication continues without interruption
```

---

## 6. Read Replicas — Scaling Reads

### 6.1 Routing Reads to Replicas

There are three layers where you can route reads:

**Application layer:** The application maintains two connection strings — one for writes (primary), one for reads (replica). This is the simplest approach and gives you the most control.

```python
# Django settings.py — multi-database setup
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'primary.db.internal',
        'NAME': 'mydb',
        # ...
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'replica.db.internal',
        'NAME': 'mydb',
        # ...
    }
}

DATABASE_ROUTERS = ['myapp.routers.PrimaryReplicaRouter']
```

```python
# myapp/routers.py
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        # Route all reads to replica
        return 'replica'

    def db_for_write(self, model, **hints):
        return 'default'

    def allow_relation(self, obj1, obj2, **hints):
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'default'
```

**PgBouncer:** PgBouncer can route read vs write traffic to different PostgreSQL instances. You configure two pools: one pointing to the primary, one pointing to replicas. The application connects to two PgBouncer listeners, one for reads, one for writes. This is the most common production setup.

**HAProxy:** HAProxy can inspect SQL at the application layer (using the MySQL/PostgreSQL protocol) to route `SELECT` statements to read pools and writes to the primary. This is more complex and adds overhead but gives you transparent routing — the application sees a single connection endpoint.

```
# HAProxy-based routing (concept, not production config)
frontend db_frontend
    bind *:5432
    default_backend db_primary

backend db_primary
    server primary 10.0.0.10:5432 check

backend db_replica
    balance roundrobin
    server replica1 10.0.0.11:5432 check
    server replica2 10.0.0.12:5432 check
```

### 6.2 The Stale Read Problem

The most common bug with read replicas is the "read your own write" problem. Here is the failure scenario:

1. User submits a form → write goes to primary → `INSERT INTO orders` succeeds
2. Application immediately redirects user to "view your order" page
3. View page reads from the replica
4. Replica hasn't applied the INSERT yet (30ms of lag)
5. User sees "order not found" — the write they just made is invisible

```
Client ──POST──► App ──INSERT──► Primary
Client ──GET───► App ──SELECT──► Replica (not yet applied!)
Result: 404 / empty page
```

Solutions:

**Read after write from primary:** For one request after a write, read from primary. Simple, correct, but adds load to primary.

**Sticky session to primary after write:** Set a flag in the session (or cookie) that says "for the next N seconds or N requests, read from primary." After that window, switch back to replica.

**`synchronous_commit = remote_apply`:** The write does not return to the application until the replica has applied it. After commit, reading from the replica is safe. Trade-off: higher commit latency.

**LSN-based routing:** The application records the LSN returned by the primary after a write (via `pg_current_wal_lsn()`), then passes it to the replica query. The replica waits until it has replayed at least that LSN before executing the query:

```sql
-- On primary after write, capture the LSN:
SELECT pg_current_wal_lsn();
-- Returns: 0/5A3BC10

-- On replica, wait until that LSN is replayed:
SELECT pg_wal_replay_wait('0/5A3BC10', timeout_ms := 1000);
-- Blocks until replica reaches that LSN or timeout

-- Then query safely
SELECT * FROM orders WHERE id = 12345;
```

`pg_wal_replay_wait()` was added in PostgreSQL 16. Before that, you had to poll `pg_last_wal_replay_lsn()` manually.

### 6.3 Hot Standby and Query Conflicts

PostgreSQL allows queries on replicas while they replay WAL (`hot_standby = on`). But there is a fundamental conflict: what happens when WAL replay needs to modify or delete rows that a running query is currently reading?

PostgreSQL resolves conflicts in favor of WAL replay. It will cancel the conflicting query on the replica with:

```
ERROR: canceling statement due to conflict with recovery
DETAIL: User query might have needed to see row versions that must be removed.
```

The conflicts arise from:

- **Vacuum:** Primary vacuums dead rows; replica must remove them. Running queries on replica reading those rows conflict.
- **Lock conflicts:** DDL on the primary acquires exclusive locks; this is replicated as a WAL record that the replica must apply.
- **Dropped tablespace:** Queries using a tablespace that is being dropped.

Configuration parameters that control conflict behavior:

```ini
# How long to wait before canceling a conflicting query on the replica
# Default: 30 seconds. After this, the query is killed.
max_standby_streaming_delay = 30s

# Same for WAL being applied from archive (during recovery)
max_standby_archive_delay = 30s
```

If you set `max_standby_streaming_delay = -1`, conflicting queries are never canceled — WAL replay waits indefinitely. This can cause the replica to fall arbitrarily far behind primary if long queries are common. Do not do this unless you understand the trade-off.

### 6.4 hot_standby_feedback

```ini
# postgresql.conf on REPLICA
hot_standby_feedback = on
```

When `hot_standby_feedback = on`, the replica sends its oldest running transaction ID back to the primary in heartbeat messages. The primary uses this to avoid vacuuming rows that the replica's queries might still need. This prevents the "canceling statement due to conflict with recovery" errors.

The cost: the primary's MVCC horizon is held back by the replica's oldest transaction. If a long query runs on the replica, the primary cannot vacuum old versions. This causes table bloat on the primary.

> **Production decision:** Enable `hot_standby_feedback` only on replicas that run short, latency-sensitive queries. For replicas running analytics queries (which may run for minutes), leave `hot_standby_feedback = off` and instead set `max_standby_streaming_delay` high enough for typical query duration. If your analytics queries run for 10 minutes, set `max_standby_streaming_delay = 600s`.

---

## 7. Connection Pooling with PgBouncer

### 7.1 Why PostgreSQL's Process Model Breaks at Scale

Every PostgreSQL connection is a full OS process. This is reliable, isolated, and simple — and it does not scale beyond a few hundred connections.

The costs:

- Each idle connection consumes ~5–10 MB of RAM (backend process overhead)
- Each active connection with `work_mem = 64MB` can consume 64+ MB under sort pressure
- 1000 connections × 10 MB = 10 GB of overhead before any actual work
- OS scheduler overhead: context switching between hundreds of processes adds latency
- Lock table: every backend needs a slot in the shared memory lock table (`max_locks_per_transaction` × `max_connections`)

A modern web application with 50 app server instances, each maintaining a connection pool of 20 connections, wants 1000 database connections — far beyond what PostgreSQL handles efficiently.

`max_connections` in PostgreSQL defaults to 100 and should rarely exceed 200–400. Beyond that, PgBouncer is not optional.

### 7.2 PgBouncer Modes: Session, Transaction, Statement

PgBouncer multiplexes many client connections onto fewer server connections. The pooling mode determines how long a server connection is held:

**Session pooling:** A server connection is assigned to a client for the entire session. When the client disconnects, the connection is returned to the pool. This is equivalent to a simple connection pool in an ORM. It reduces connection setup overhead but does not reduce the number of concurrent server connections.

**Transaction pooling:** A server connection is assigned only for the duration of a transaction. Between transactions, the connection is returned to the pool. This is the mode that actually multiplexes: 1000 idle clients between transactions share a pool of 50 server connections. This is the mode most production systems use.

**Statement pooling:** A server connection is held only for a single SQL statement. Most restrictive. Breaks anything that uses multi-statement sequences (explicit transactions, prepared statements, `SET` commands that persist state). Not practical for OLTP.

| Mode        | Server connections held | Multiplexing | Compatibility |
| ----------- | ----------------------- | ------------ | ------------- |
| Session     | Per client session      | None         | Full          |
| Transaction | Per transaction         | High         | Most apps     |
| Statement   | Per statement           | Maximum      | Very limited  |

### 7.3 Complete pgbouncer.ini Configuration

```ini
# /etc/pgbouncer/pgbouncer.ini

[databases]
# Database alias = connection string to actual PostgreSQL
mydb = host=10.0.0.10 port=5432 dbname=mydb

# Read replica pool (optional — point at replica for read-only connections)
mydb_replica = host=10.0.0.11 port=5432 dbname=mydb

[pgbouncer]
# Listen address and port for client connections
listen_addr = 0.0.0.0
listen_port = 6432

# Authentication
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Pool mode: session | transaction | statement
pool_mode = transaction

# Maximum number of server connections PgBouncer will open per (db, user) pair
default_pool_size = 50

# Minimum number of idle server connections to maintain
min_pool_size = 5

# Reserve pool: extra server connections available for high-traffic bursts
reserve_pool_size = 10
reserve_pool_timeout = 5

# Maximum number of client connections PgBouncer will accept
max_client_conn = 1000

# How long to wait for a server connection before giving up
query_wait_timeout = 15

# Close idle server connections after this many seconds
server_idle_timeout = 600

# Close client connections that have been idle
client_idle_timeout = 0

# Log settings
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid

# Admin interface
admin_users = pgbouncer_admin
stats_users = pgbouncer_stats

# Server reset query (sent when returning a connection to pool)
# In transaction mode, this runs between transactions
server_reset_query = DISCARD ALL

# Don't use server_reset_query for session mode (too expensive per query)
# server_reset_query_always = 0

# TLS (strongly recommended for production)
# server_tls_sslmode = require
# server_tls_ca_file = /etc/ssl/certs/ca-certificates.crt
```

```
# /etc/pgbouncer/userlist.txt
# Format: "username" "password-hash"
# Generate hash: SELECT concat('md5', md5(concat(password, username)));
# Or for scram: use pgbouncer's built-in tooling
"app_user" "SCRAM-SHA-256$4096:..."
"pgbouncer_admin" "SCRAM-SHA-256$4096:..."
```

Start and manage:

```bash
sudo systemctl start pgbouncer
sudo systemctl enable pgbouncer

# Connect to PgBouncer admin interface
psql -h 127.0.0.1 -p 6432 -U pgbouncer_admin pgbouncer

# Useful admin commands
SHOW POOLS;         -- Pool statistics per (database, user)
SHOW CLIENTS;       -- Active client connections
SHOW SERVERS;       -- Active server connections
SHOW STATS;         -- Request rates, wait times
RELOAD;             -- Reload config without restart
PAUSE mydb;         -- Pause new queries (for maintenance)
RESUME mydb;        -- Resume
```

### 7.4 Pool Sizing: The Math

This is where most engineers get it wrong.

```
PostgreSQL max_connections = X
PgBouncer pool_size = X - superuser_reserved - monitoring_connections

Rule of thumb:
max_connections = 2 * CPU cores + number of disks (for I/O bound)

For a 16-core server: max_connections = ~100-150
PgBouncer pool_size = 80 (leaving room for direct connections, monitoring)
PgBouncer max_client_conn = 1000+ (clients can be much larger than server connections)
```

The key insight: `max_client_conn` and `default_pool_size` are independent. You can have 1000 client connections sharing 50 server connections. Between transactions, 950 of those clients are waiting in PgBouncer's queue, not consuming a server connection.

```
Application: 20 servers × 50 connections each = 1000 client connections to PgBouncer
PgBouncer: 1000 clients → 50 server connections to PostgreSQL
PostgreSQL: sees 50 connections (well within max_connections)

Concurrently active: at any moment, only ~50-100 of those 1000 clients are mid-transaction
The rest are idle between transactions — they need no server connection
```

Sizing `pool_size`:

```
pool_size = max_connections × 0.8  -- leave headroom for DBA, monitoring, migrations
```

Do not set `pool_size` higher than PostgreSQL's `max_connections`. This defeats the purpose and causes queuing at the PostgreSQL level instead of at PgBouncer.

### 7.5 What PgBouncer Does Not Solve

**Prepared statements in transaction mode.** PostgreSQL prepared statements (`PREPARE`, `EXECUTE`) are per-connection. In transaction pooling mode, your client might use a different server connection for each transaction. A prepared statement created in transaction A on server connection S1 is not available in transaction B on server connection S2.

Solution: Use PgBouncer's `server_reset_query = DEALLOCATE ALL` (or `DISCARD ALL`), which clears prepared statements when the server connection is returned to the pool. But this means you lose the performance benefit of prepared statements.

Better solution: Use protocol-level prepared statements at the driver level (e.g., libpq's extended query protocol), but configure your application to re-prepare when it gets a new connection. Django's psycopg3 adapter handles this correctly with server-side binding when using transaction pooling.

**`SET` statements and session state.** In transaction pooling, `SET search_path = myschema` set in one transaction does not persist to the next, because you might get a different server connection. PgBouncer's `DISCARD ALL` in `server_reset_query` resets session state on return.

**Long-running transactions.** A client holding an open transaction holds a server connection for the duration. In transaction pooling, this removes one server connection from the pool for as long as the transaction runs. Optimize long transactions; do not open a transaction and leave it idle.

### 7.6 PgBouncer + Django: Full Setup

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'app_user',
        'PASSWORD': 'app_password',
        'HOST': '10.0.0.5',   # PgBouncer host
        'PORT': '6432',        # PgBouncer port
        'OPTIONS': {
            # Disable server-side cursors — incompatible with transaction pooling
            # because cursors are session-scoped
            'cursor_factory': None,
            # Disable persistent connections at the Django level
            # PgBouncer handles pooling, Django should not maintain its own pool
        },
        # Set CONN_MAX_AGE to 0 to let PgBouncer manage connections
        # Or set it to a reasonable value if using session pooling
        'CONN_MAX_AGE': 0,
    }
}
```

For Django with transaction pooling, disable server-side cursors globally. Django uses server-side cursors for large querysets (when using `iterator()`). These are session-scoped and break transaction pooling.

```python
# For large querysets, use chunked iteration without server-side cursors:
for batch in MyModel.objects.all().iterator(chunk_size=2000):
    process(batch)
# Django's iterator() uses server-side cursors by default when CONN_MAX_AGE > 0
# With CONN_MAX_AGE = 0 and PgBouncer, this is not an issue
```

---

## 8. Failover — When the Primary Dies

### 8.1 Manual Failover: Promoting a Replica

If the primary is confirmed dead and you need to manually promote a replica:

```bash
# Option 1: pg_ctl promote (OS command on replica server)
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main

# Option 2: touch the trigger file (legacy, pre-12)
# touch /tmp/postgresql.trigger.5432

# Option 3: SQL function (must be connected to the replica)
SELECT pg_promote();
-- Returns: t (true = promotion succeeded)
```

After promotion, the replica becomes a fully writable primary. The `standby.signal` file is removed from its data directory. It is no longer connected to the old primary.

**What to do after promotion:**

```bash
# 1. Update your DNS or load balancer to point to the new primary
# e.g., update your Route53 record for db.internal from old_primary IP to new_primary IP

# 2. If there are other replicas, repoint them to the new primary
# On each remaining replica, update primary_conninfo:
sudo -u postgres psql -c "ALTER SYSTEM SET primary_conninfo = 'host=NEW_PRIMARY_IP port=5432 user=replicator password=...';"
sudo -u postgres psql -c "SELECT pg_reload_conf();"
# Or restart the replica

# 3. Handle the old primary when it comes back
# The old primary has diverged (it may have accepted writes that the new primary didn't replay)
# It CANNOT simply rejoin as a replica without a full reinitialize
# Reinitialize it from the new primary:
sudo systemctl stop postgresql@16-main   # on old primary
sudo -u postgres rm -rf /var/lib/postgresql/16/main/*
sudo -u postgres pg_basebackup -h NEW_PRIMARY_IP -U replicator -D /var/lib/postgresql/16/main -P -Xs -R
sudo systemctl start postgresql@16-main
```

> **Production decision:** Manual failover is acceptable for planned maintenance. For unplanned failures (3 AM primary crash), manual failover means waking someone up, SSHing into servers, running commands under pressure. RTO with manual failover is 15–60 minutes in practice. For RTO < 60 seconds, you need Patroni.

### 8.2 Automatic Failover: Patroni

Patroni is the industry-standard solution for automatic PostgreSQL failover. It is a Python daemon that runs on each PostgreSQL server and manages the cluster as a whole.

**What Patroni does:**

- Manages PostgreSQL start/stop/promote
- Uses a distributed consensus store (DCS) — etcd, Consul, or ZooKeeper — as a single source of truth for "who is the primary"
- Automatically promotes a replica when the primary fails
- Prevents split-brain through DCS leader leases and fencing
- Manages replication slots, `recovery.conf`/`postgresql.auto.conf`, and switchovers
- Exposes a REST API and `patronictl` CLI for operators

**Architecture:**

```
┌─────────────────────────────────────────────────────┐
│                  etcd cluster                       │
│  (3 or 5 nodes for quorum — separate from Postgres) │
└───────────────┬─────────────────┬───────────────────┘
                │ DCS API         │ DCS API
                │                 │
┌───────────────▼──────┐  ┌───────▼─────────────────┐
│   Node 1 (PRIMARY)   │  │  Node 2 (REPLICA)        │
│                      │  │                          │
│  Patroni daemon      │  │  Patroni daemon          │
│       │              │  │       │                  │
│  PostgreSQL          │  │  PostgreSQL              │
│  (read/write)        │  │  (read-only standby)     │
└──────────────────────┘  └──────────────────────────┘
```

**Patroni configuration (patroni.yml):**

```yaml
scope: postgres-cluster # Cluster name — must match across all nodes
namespace: /db/ # Key prefix in DCS
name: node1 # This node's name

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.10:8008

etcd3:
  hosts:
    - 10.0.0.20:2379
    - 10.0.0.21:2379
    - 10.0.0.22:2379

bootstrap:
  dcs:
    ttl: 30 # Leader lease duration in seconds
    loop_wait: 10 # How often Patroni checks DCS
    retry_timeout: 10
    maximum_lag_on_failover: 1048576 # 1 MB — don't promote replica with > 1MB lag
    postgresql:
      use_pg_rewind: true # Use pg_rewind to rejoin old primary after failover
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: on
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: on # Required for pg_rewind

  initdb:
    - encoding: UTF8
    - data-checksums # Highly recommended

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.0.10:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: strong_password
    superuser:
      username: postgres
      password: postgres_password

tags:
  nofailover: false # Set true to prevent this node from being promoted
  noloadbalance: false # Set true to exclude from load balancing
  clonefrom: false # Set true to prefer this node as clone source
```

### 8.3 The Split-Brain Problem

Split-brain occurs when two nodes both believe they are the primary and accept writes independently. This is the most dangerous failure mode in a distributed database.

**How it happens:**

```
Normal state:
  Primary (10.0.0.10) — writes flowing
  Replica (10.0.0.11) — replicating

Network partition:
  10.0.0.10 ←── network ──✗──► 10.0.0.11
  (can't see each other)

Without coordination:
  10.0.0.10: "I'm still the primary, accept writes"
  10.0.0.11: "Primary is dead, I'll promote myself"

Now both nodes accept writes.
When the network heals, their WALs have diverged.
There is no automatic way to merge this.
Data loss is certain.
```

### 8.4 Fencing: How Patroni Prevents Split-Brain

Patroni uses the DCS as a distributed lock (leader lease). The primary must continuously renew its lease in the DCS. If it fails to renew within `ttl` seconds (default 30s), it loses the lease.

**The key rule:** A node will only accept writes if it holds the DCS leader lease. Before issuing writes to a newly promoted replica, the node verifies it holds the lease.

```
Old Primary loses network to DCS (can't renew lease):
  → Lease expires after 30 seconds
  → Patroni on old primary calls pg_ctl stop immediately (STONITH)
  → Old primary stops accepting connections

New Primary wins DCS election:
  → Acquires leader lease
  → Promotes PostgreSQL
  → Starts accepting connections

Result: Only one primary ever holds the lease.
```

This is **STONITH** (Shoot The Other Node In The Head) by software: the old primary shuts itself down when it loses the lease, rather than waiting to be fenced by external hardware.

**What `wal_log_hints` and `pg_rewind` do:** When the old primary comes back, its WAL has diverged from the new primary (it may have written WAL locally that was never replicated). `pg_rewind` can fast-forward or backward-rewind the old primary's data files to the point of divergence, without a full `pg_basebackup`. This dramatically reduces the time to rejoin a failed primary as a replica.

### 8.5 Patroni Failover: Step by Step

Here is the exact sequence of events during an automatic Patroni failover:

```
T=0:  Primary (node1) crashes or loses DCS connectivity.
      PostgreSQL process on node1 stops responding.

T=0–10s: Patroni on node1 cannot reach DCS. It logs a warning.
         Patroni on node2 and node3 detect that node1 is not renewing its lease.

T=10–30s: node1's lease in DCS expires (ttl=30s).
          Patroni on node2/node3 sees leader key is gone.
          They enter election: each tries to acquire the leader key in DCS.
          DCS provides atomic compare-and-set — only one node wins.

T=30s: node2 wins the election, acquires the leader lease.
       Patroni on node2:
         1. Checks if node2's replica is a valid candidate:
            - Is it reachable? Yes.
            - Is it within maximum_lag_on_failover? Check WAL lag.
         2. Calls pg_promote() on node2's PostgreSQL.
         3. PostgreSQL on node2 exits recovery mode.
         4. Writes standby.signal removal, updates pg_control.
         5. node2 is now a read/write primary.

T=30–35s: Patroni on node3:
          - Lost the DCS election.
          - Updates its primary_conninfo to point to node2.
          - Reloads PostgreSQL configuration.
          - PostgreSQL on node3 reconnects to node2 as a replica.

T=35s: HAProxy/load balancer health checks detect:
       - node1 (port 8008 /primary): HTTP 503 (not primary)
       - node2 (port 8008 /primary): HTTP 200 (is primary)
       - HAProxy routes traffic to node2.

T=40s: Application connections to the old primary fail.
       New connections succeed on node2.
       Application must handle connection failure and reconnect.

Total RTO: ~30–45 seconds from crash to new primary accepting connections.
```

**Health check endpoints Patroni provides:**

```bash
# On any node — returns 200 if this node is the primary
curl -s http://10.0.0.10:8008/primary

# Returns 200 if this node is a healthy replica
curl -s http://10.0.0.10:8008/replica

# Returns full cluster state
curl -s http://10.0.0.10:8008/ | python3 -m json.tool
```

**`patronictl` commands:**

```bash
# View cluster state
patronictl -c /etc/patroni/patroni.yml list

# Manually trigger a switchover (planned, graceful)
patronictl -c /etc/patroni/patroni.yml switchover --master node1 --candidate node2

# Manually trigger a failover (unplanned — skips safety checks)
patronictl -c /etc/patroni/patroni.yml failover postgres-cluster --master node1

# Pause automatic failover (for maintenance)
patronictl -c /etc/patroni/patroni.yml pause
patronictl -c /etc/patroni/patroni.yml resume

# Reinitialize a replica from the primary
patronictl -c /etc/patroni/patroni.yml reinit postgres-cluster node3
```

---

## 9. High Availability Architecture Patterns

### 9.1 Single Primary + 1 Sync + 1 Async Replica

This is the most common production HA pattern. It balances durability, availability, and cost.

```
                    ┌──────────────────────────────────────────────┐
                    │              Application Layer               │
                    │   (HAProxy / PgBouncer / Application)        │
                    └────────────┬──────────────┬──────────────────┘
                                 │ Writes        │ Reads
                                 │               │
                    ┌────────────▼──────────────────────────────┐
                    │         Primary (Node 1)                   │
                    │   10.0.0.10:5432                           │
                    │   wal_level = replica                      │
                    │   synchronous_standby_names = 'node2'     │
                    └───────────────────┬────────────────────────┘
                                        │
              ┌─────────── WAL ─────────┼──────────────────┐
              │  synchronous            │                   │ asynchronous
              │  (waits for ACK)        │                   │ (fire and forget)
              ▼                         ▼                   ▼
┌─────────────────────────┐   ┌─────────────────────────────────┐
│   Sync Replica (Node 2) │   │   Async Replica (Node 3)        │
│   10.0.0.11:5432        │   │   10.0.0.12:5432                │
│   Same DC / AZ          │   │   Different region / AZ         │
│   Failover candidate    │   │   DR / delayed replica          │
└─────────────────────────┘   └─────────────────────────────────┘
```

**Properties:**

- RPO = 0 for single-node failure (sync replica has all committed data)
- RPO > 0 for region failure (async replica may lag)
- RTO = 30–60 seconds with Patroni automatic failover
- Write performance: slightly higher latency (sync replica must ACK)
- Read scalability: reads can go to node2 or node3

**`synchronous_standby_names` with fallback:**

```ini
# On primary:
# Require confirmation from node2 (sync), fall back to any 1 of {node2, node3}
# if node2 is unavailable
synchronous_standby_names = 'ANY 1 (node2, node3)'
```

This prevents writes from hanging if node2 (the designated sync replica) goes down — node3 takes over as the synchronous replica automatically.

### 9.2 Multi-Region Setup

```
Region US-EAST                          Region EU-WEST
┌──────────────────────────────┐       ┌──────────────────────────────┐
│  Primary          Node 1     │       │  Replica          Node 3     │
│  10.0.0.10:5432              │──────►│  10.1.0.10:5432              │
│  (all writes go here)        │ async │  (async, ~80ms lag)          │
│                              │  WAL  │  (DR site — promote on       │
│  Sync Replica    Node 2      │       │   regional failure)          │
│  10.0.0.11:5432              │       └──────────────────────────────┘
│  (same AZ as primary)        │
└──────────────────────────────┘

DNS / Global Load Balancer:
  db.global → us-east primary (normal operation)
  db.global → eu-west replica promoted (after regional failover)
```

**Cross-region replication is always asynchronous** due to network latency. Acknowledge this in your SLA: cross-region failover has RPO > 0. How much data loss depends on your write rate and the network latency. For 100 writes/second with 100ms of lag, you may lose ~10 transactions on a regional failover.

**Patroni in multi-region:** Patroni requires a DCS cluster. Running etcd across regions adds latency to leader lease operations. Options:

- Run etcd in the primary region only; use a witness node approach for the remote region
- Use Consul with WAN federation
- Accept that cross-region automatic failover is slow or requires human confirmation

Most teams choose: automatic failover within a region (sub-60s), manual/semi-automatic failover across regions (minutes, with human confirmation).

### 9.3 RTO vs RPO: Defining Your Requirements

Before choosing an HA architecture, define these numbers explicitly. Everything else follows.

**RPO (Recovery Point Objective):** How much data loss is acceptable? Measured in time or transactions.

- RPO = 0: Zero data loss. Requires synchronous replication. Commits are slower.
- RPO = 1 minute: Accept losing up to 1 minute of writes. Allows async replication.
- RPO = 24 hours: Daily backups are sufficient. No replication needed for DR.

**RTO (Recovery Time Objective):** How long can the service be unavailable?

- RTO = 30 seconds: Requires Patroni or equivalent automatic failover.
- RTO = 5 minutes: Manual failover with a prepared runbook is feasible.
- RTO = 1 hour: Restoring from backup is acceptable.

```
RPO = 0, RTO < 60s:
  → Synchronous replica in same DC, Patroni automatic failover
  → This is the most expensive configuration

RPO < 5 minutes, RTO < 5 minutes:
  → Asynchronous replica, Patroni automatic failover
  → Most OLTP applications land here

RPO < 24 hours, RTO < 4 hours:
  → Daily PITR backups to S3, no replica required
  → Appropriate for non-critical systems
```

### 9.4 Backup + Replication: Different Problems

These are frequently confused. They solve different problems and are not substitutes for each other.

|                        | Replication                                | Backup                                   |
| ---------------------- | ------------------------------------------ | ---------------------------------------- |
| Protects against       | Hardware failure, server crash, AZ failure | Data corruption, human error, ransomware |
| Latency                | Real-time                                  | Point-in-time (daily/hourly)             |
| Recovery target        | Current state of data                      | Any past point in time                   |
| Does it copy bad data? | Yes — a DELETE replicates immediately      | Depends on RPO: PITR can go back         |
| Disk usage             | ~1× data size per replica                  | Varies (incremental backups are smaller) |

A scenario that replication alone cannot solve: a developer runs `UPDATE orders SET amount = 0` without a WHERE clause. This replicates to all replicas in milliseconds. Every replica now has zeroed-out amounts. Your replication is working perfectly — perfectly replicating destruction.

The solution is PITR (Point-in-Time Recovery) from backups, or a delayed replica (Section 4.3).

Run both. Use `pgBackRest` or `WAL-G` for PITR backups to object storage, alongside streaming replication.

```bash
# WAL-G example: continuous WAL archiving to S3
# postgresql.conf
archive_mode = on
archive_command = 'wal-g wal-push %p'
archive_timeout = 60  # Archive at least every 60 seconds

# Restore from PITR:
# wal-g backup-fetch /var/lib/postgresql/16/main LATEST
# Create recovery.signal and set:
# restore_command = 'wal-g wal-fetch %f %p'
# recovery_target_time = '2026-03-24 09:30:00'
```

---

## 10. Monitoring Replication Health

### 10.1 pg_stat_replication: Every Column Explained

This view exists on the **primary** and has one row per connected standby.

```sql
SELECT * FROM pg_stat_replication;
```

| Column             | Type        | Meaning                                                                |
| ------------------ | ----------- | ---------------------------------------------------------------------- |
| `pid`              | integer     | PID of the WAL sender process on the primary                           |
| `usesysid`         | oid         | OID of the replication user                                            |
| `usename`          | name        | Name of the replication user                                           |
| `application_name` | text        | `application_name` set on the replica's `primary_conninfo`             |
| `client_addr`      | inet        | IP address of the replica                                              |
| `client_hostname`  | text        | Hostname if reverse DNS resolves                                       |
| `client_port`      | integer     | Ephemeral port on the replica                                          |
| `backend_start`    | timestamptz | When this WAL sender started (i.e., when the replica connected)        |
| `backend_xmin`     | xid         | Oldest transaction ID needed on primary due to this standby's feedback |
| `state`            | text        | `startup`, `catchup`, `streaming`, `backup`, `stopping`                |
| `sent_lsn`         | pg_lsn      | Last WAL position sent to this standby                                 |
| `write_lsn`        | pg_lsn      | Last WAL position written to disk on standby                           |
| `flush_lsn`        | pg_lsn      | Last WAL position flushed (fsync'd) on standby                         |
| `replay_lsn`       | pg_lsn      | Last WAL position applied to standby's data files                      |
| `write_lag`        | interval    | Time elapsed between primary flush and standby write                   |
| `flush_lag`        | interval    | Time elapsed between primary flush and standby flush                   |
| `replay_lag`       | interval    | Time elapsed between primary flush and standby replay                  |
| `sync_priority`    | integer     | Priority of this standby for synchronous replication (0 = async)       |
| `sync_state`       | text        | `async`, `sync`, `potential`, `quorum`                                 |
| `reply_time`       | timestamptz | Last time this standby sent a status reply to the primary              |

`sync_state` values:

- `async`: not in `synchronous_standby_names`, never waited on
- `potential`: listed in `synchronous_standby_names` but not currently the active sync standby
- `sync`: the active synchronous standby — commits wait for this one
- `quorum`: participating in quorum-based synchronous replication

### 10.2 pg_stat_wal_receiver: The Replica Side

This view exists on the **replica** and has at most one row.

```sql
SELECT
    pid,
    status,               -- 'streaming' or 'replicating'
    receive_start_lsn,    -- LSN where this WAL receiver started
    receive_start_tli,    -- Timeline when started
    written_lsn,          -- Last LSN written to pg_wal/ on this replica
    flushed_lsn,          -- Last LSN flushed on this replica
    received_tli,         -- Current timeline
    last_msg_send_time,   -- Last time the receiver sent a message to primary
    last_msg_receipt_time,-- Last time the receiver got a message from primary
    latest_end_lsn,       -- Latest LSN reported to primary
    latest_end_time,       -- Time of latest_end_lsn
    slot_name,            -- Replication slot being used (if any)
    sender_host,          -- Primary's hostname
    sender_port,          -- Primary's port
    conninfo              -- Full connection string (passwords masked)
FROM pg_stat_wal_receiver;
```

If this view is empty, the replica is not connected to a primary. Check `primary_conninfo` and PostgreSQL logs.

### 10.3 Complete Monitoring Query Set

```sql
-- ============================================================
-- 1. Replication status overview (run on PRIMARY)
-- ============================================================
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn))   AS total_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag,
    -- When did we last hear from this replica?
    now() - reply_time                                         AS time_since_reply
FROM pg_stat_replication
ORDER BY pg_wal_lsn_diff(sent_lsn, replay_lsn) DESC;


-- ============================================================
-- 2. Replication slot health (run on PRIMARY) — CRITICAL
-- ============================================================
SELECT
    slot_name,
    slot_type,
    active,
    active_pid,
    restart_lsn,
    confirmed_flush_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS wal_retained,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS wal_retained_bytes
FROM pg_replication_slots
ORDER BY wal_retained_bytes DESC;


-- ============================================================
-- 3. WAL directory size (run on PRIMARY)
-- ============================================================
SELECT
    pg_size_pretty(sum(size)) AS wal_dir_size
FROM pg_ls_waldir();


-- ============================================================
-- 4. Lag from replica's perspective (run on REPLICA)
-- ============================================================
SELECT
    now() - pg_last_xact_replay_timestamp()   AS replication_delay,
    pg_last_wal_receive_lsn()                 AS received_lsn,
    pg_last_wal_replay_lsn()                  AS replayed_lsn,
    pg_wal_lsn_diff(
        pg_last_wal_receive_lsn(),
        pg_last_wal_replay_lsn()
    )                                          AS receive_replay_gap_bytes,
    pg_is_in_recovery()                        AS is_standby;


-- ============================================================
-- 5. Is this replica connected to a primary? (run on REPLICA)
-- ============================================================
SELECT
    status,
    sender_host,
    sender_port,
    last_msg_receipt_time,
    now() - last_msg_receipt_time AS time_since_last_message
FROM pg_stat_wal_receiver;
-- Empty result = not connected


-- ============================================================
-- 6. Timeline history (run on REPLICA after failover)
-- ============================================================
SELECT * FROM pg_control_checkpoint();
-- Shows current timeline (tli) — should increment after each failover


-- ============================================================
-- 7. Logical replication subscription lag (run on SUBSCRIBER)
-- ============================================================
SELECT
    subname,
    pid,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    now() - last_msg_receipt_time AS lag
FROM pg_stat_subscription;


-- ============================================================
-- 8. Combined health report for alerting (run on PRIMARY)
-- ============================================================
WITH replica_status AS (
    SELECT
        application_name,
        state,
        sync_state,
        replay_lag,
        now() - reply_time AS silence_duration,
        pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
    FROM pg_stat_replication
),
slot_status AS (
    SELECT
        slot_name,
        active,
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_bytes
    FROM pg_replication_slots
)
SELECT
    'replica' AS type,
    application_name AS name,
    CASE
        WHEN state != 'streaming' THEN 'CRITICAL: not streaming'
        WHEN silence_duration > INTERVAL '1 minute' THEN 'WARNING: no reply for ' || silence_duration
        WHEN lag_bytes > 1073741824 THEN 'WARNING: > 1GB lag'
        ELSE 'OK'
    END AS status,
    replay_lag,
    lag_bytes
FROM replica_status
UNION ALL
SELECT
    'slot' AS type,
    slot_name AS name,
    CASE
        WHEN NOT active AND retained_bytes > 10737418240 THEN 'CRITICAL: inactive slot retaining > 10GB'
        WHEN NOT active THEN 'WARNING: inactive slot'
        ELSE 'OK'
    END AS status,
    NULL,
    retained_bytes
FROM slot_status
ORDER BY status DESC, type, name;
```

### 10.4 What to Alert On

These are the conditions that require immediate action:

| Condition                                                | Severity | Query                                          |
| -------------------------------------------------------- | -------- | ---------------------------------------------- |
| No replicas connected                                    | CRITICAL | `SELECT count(*) FROM pg_stat_replication` = 0 |
| Sync replica not `streaming`                             | CRITICAL | `state != 'streaming' AND sync_state = 'sync'` |
| Replication slot inactive with >10 GB retained           | CRITICAL | `NOT active AND retained_bytes > 10GB`         |
| Replica not sending replies for >2 minutes               | WARNING  | `now() - reply_time > INTERVAL '2 minutes'`    |
| Replay lag > 30 seconds (for latency-sensitive replicas) | WARNING  | `replay_lag > INTERVAL '30 seconds'`           |
| Byte lag >1 GB and growing                               | WARNING  | Trend in `lag_bytes` over time                 |
| WAL directory > 80% of disk                              | CRITICAL | `pg_ls_waldir()` total vs disk capacity        |

For alerting, write a monitoring script that runs the health report query and sends to PagerDuty / OpsGenie / Slack when any row has status != 'OK'. Run it every 30 seconds.

---

## 11. Common Replication Problems and Fixes

### 11.1 Replica Falling Too Far Behind

**Symptoms:** `pg_wal_lsn_diff(sent_lsn, replay_lsn)` growing continuously. `replay_lag` > minutes. The replica is not catching up even during quiet periods.

**Diagnosis:**

```sql
-- On primary: is lag growing or stable?
SELECT
    application_name,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag,
    replay_lag
FROM pg_stat_replication;

-- Watch it over time:
-- If lag grows during writes but recovers during quiet periods → burst problem
-- If lag grows continuously → the replica cannot keep up with write rate

-- On replica: is WAL being received but not applied?
SELECT
    pg_size_pretty(pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn())) AS receive_replay_gap,
    now() - pg_last_xact_replay_timestamp() AS time_lag;
```

**Root causes and fixes:**

_Slow replica disk:_ WAL replay is I/O bound. The startup process applies WAL records faster than the disk can handle the random writes.

```bash
# Check disk I/O on replica
iostat -x 1 10
# If %util on the data disk is consistently > 80%, the disk is the bottleneck
# Fix: upgrade to faster storage (NVMe), or use a separate WAL disk
```

_Conflicting queries on hot standby:_ Long queries on the replica block WAL replay.

```sql
-- On replica: check for blocked recovery
SELECT pid, query_start, query, wait_event_type, wait_event
FROM pg_stat_activity
WHERE backend_type = 'walsender' OR wait_event = 'RecoveryConflictSnapshot';

-- Check for long-running queries on the replica that might be causing conflicts
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

Fix: reduce `max_standby_streaming_delay` or cancel long queries manually, or switch analytics workload to a dedicated replica with `hot_standby_feedback = on`.

_The replica is too far behind to stream (needs reinitialize):_

```bash
# If wal_keep_size on primary has rotated past the replica's restart_lsn,
# the primary will refuse streaming and the replica falls into 'startup' state

# Check: on primary
SELECT state FROM pg_stat_replication;
-- 'startup' instead of 'streaming' = replica needs reinitialize

# Fix: full reinitialize
sudo systemctl stop postgresql@16-main    # on replica
sudo -u postgres rm -rf /var/lib/postgresql/16/main/*
sudo -u postgres pg_basebackup -h PRIMARY_IP -U replicator -D /var/lib/postgresql/16/main -P -Xs -R
sudo systemctl start postgresql@16-main
```

### 11.2 Replication Slot Bloating WAL Disk

**Symptoms:** `/var/lib/postgresql/16/main/pg_wal/` is growing. `df -h` shows data disk filling up. `pg_replication_slots` shows an inactive slot with a large `wal_retained` value.

**Immediate action:**

```sql
-- Identify the culprit
SELECT
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;

-- If the slot belongs to a dead/disconnected replica you don't need:
SELECT pg_drop_replication_slot('dead_slot_name');
-- PostgreSQL will now delete the retained WAL on the next checkpoint
```

**If the disk is already critically full:**

```bash
# PostgreSQL may be unable to write WAL and will crash if disk is full
# Emergency: drop the slot NOW (you will lose the ability to use that replica
# for streaming — it will need a full reinitialize)

sudo -u postgres psql -c "SELECT pg_drop_replication_slot('bloated_slot');"

# Force a checkpoint to allow WAL cleanup
sudo -u postgres psql -c "CHECKPOINT;"

# Verify WAL dir shrinks
sudo du -sh /var/lib/postgresql/16/main/pg_wal/
```

**Prevention:**

```ini
# postgresql.conf: set maximum WAL retained by any slot
# PostgreSQL 13+: slots exceeding this are invalidated automatically
max_slot_wal_keep_size = 10GB   # Slots won't retain more than this
```

With `max_slot_wal_keep_size`, if a slot falls too far behind, it is invalidated (not auto-dropped, but marked unusable). The replica using it will need a full reinitialize, but at least your disk is safe.

```sql
-- Check for invalidated slots
SELECT slot_name, invalidation_reason
FROM pg_replication_slots
WHERE invalidation_reason IS NOT NULL;
-- If you see rows here, those replicas need reinitializing
```

### 11.3 Promoting the Wrong Replica

In a cluster with multiple replicas at different lag values, promoting the replica that is furthest behind means losing the data that the more-current replica had.

**Before promoting, always check lag on all replicas:**

```sql
-- On each replica, check how far behind it is:
SELECT
    pg_last_wal_replay_lsn() AS replayed_lsn,
    now() - pg_last_xact_replay_timestamp() AS lag;
```

```bash
# With Patroni, this is automatic — patronictl shows lag for all nodes:
patronictl -c /etc/patroni/patroni.yml list
#  Member  | Host        | Role    | State   | TL | Lag in MB
# ---------+-------------+---------+---------+----+----------
#  node1   | 10.0.0.10   | Leader  | running |  1 |          (primary — no lag)
#  node2   | 10.0.0.11   | Replica | running |  1 | 0        (in sync)
#  node3   | 10.0.0.12   | Replica | running |  1 | 4        (4 MB behind)
```

Patroni's `maximum_lag_on_failover` setting rejects replicas that are too far behind. If node3 is 4 MB behind and `maximum_lag_on_failover = 1048576` (1 MB), Patroni will not promote node3. This prevents promoting a stale replica.

**After promoting the wrong replica:** You have data loss. If the correct replica (the one further ahead) is still running as a primary (split-brain avoided by Patroni/fencing), you need to:

1. Dump the data that was on the further-ahead replica but not on the incorrectly-promoted one
2. Import it into the now-promoted primary
3. This is a manual data reconciliation. There is no automatic fix.

**This is why you must fence before promoting.** Patroni's fencing guarantees the old primary is stopped before the new one is promoted, preventing confusion about who has the latest data.

### 11.4 Application Not Handling Failover

The database failed over successfully in 30 seconds. Your application is down for 10 minutes because it didn't notice.

Common failures:

**Connection pool holds stale connections.** PgBouncer or the ORM connection pool holds connections to the old primary. After failover, those connections are to a dead server. New connections succeed, but pooled connections fail.

Fix: PgBouncer detects broken server connections on next use and replaces them. Set aggressive health check intervals:

```ini
# pgbouncer.ini
server_check_delay = 10    # Check server health every 10 seconds
server_check_query = SELECT 1
server_lifetime = 3600     # Recycle connections every hour
```

**Application doesn't retry on connection errors.** Code that opens a DB connection at startup and never reconnects will fail after failover.

```python
# Fragile — no retry logic
def get_data():
    conn = psycopg2.connect(DATABASE_URL)   # Fails after failover
    cursor = conn.cursor()
    cursor.execute("SELECT ...")

# Robust — with retry
from psycopg2 import OperationalError
import time

def get_data():
    for attempt in range(3):
        try:
            conn = psycopg2.connect(DATABASE_URL)
            cursor = conn.cursor()
            cursor.execute("SELECT ...")
            return cursor.fetchall()
        except OperationalError as e:
            if attempt == 2:
                raise
            time.sleep(2 ** attempt)  # Exponential backoff: 1s, 2s
```

**DNS TTL too long.** You update your DNS record to point to the new primary, but applications cache DNS for 5 minutes. They keep trying the old IP.

Fix: Set DNS TTL to 30–60 seconds for database DNS records. Accept the increased DNS lookup overhead as the cost of fast failover.

**Connection string hardcodes an IP.** `host=10.0.0.10` in the connection string doesn't update when the primary IP changes.

Fix: Use a DNS name (`host=db.primary.internal`). After failover, update the DNS record. The application will reconnect to the new primary via the same hostname.

**Better fix:** Use Patroni's built-in HAProxy/DNS integration, or a virtual IP (VIP) that moves with the primary (keepalived-style, or cloud load balancer).

### 11.5 Schema Changes Breaking Replication

**In physical/streaming replication:** DDL replicates automatically via WAL. `ALTER TABLE` on the primary replicates to all replicas with no special handling. This is safe but has implications for migrations.

**The table lock problem:** `ALTER TABLE ... ADD COLUMN` without a default value is fast (metadata only). `ALTER TABLE ... ADD COLUMN DEFAULT value` on a large table (before PostgreSQL 11) rewrote the entire table and held an exclusive lock the whole time. This lock replicated, blocking queries on replicas. Since PostgreSQL 11, adding a non-volatile default is also metadata-only.

**In logical replication:** DDL does not replicate. Every schema change must be applied to subscribers manually, in the correct order (subscriber first, then publisher).

**Zero-downtime migration strategy for active systems:**

```sql
-- 1. Add column as nullable (no default, no rewrite, no lock)
ALTER TABLE orders ADD COLUMN discount_pct NUMERIC(5,2);

-- 2. Backfill in batches to avoid locking
UPDATE orders SET discount_pct = 0 WHERE id BETWEEN 1 AND 100000 AND discount_pct IS NULL;
UPDATE orders SET discount_pct = 0 WHERE id BETWEEN 100001 AND 200000 AND discount_pct IS NULL;
-- ... repeat

-- 3. Add NOT NULL constraint once all rows are populated
-- PostgreSQL 12+: NOT NULL constraint without full table scan if no nulls exist:
ALTER TABLE orders ADD CONSTRAINT orders_discount_pct_nn CHECK (discount_pct IS NOT NULL) NOT VALID;
-- Validate it (scans the table but doesn't lock writes):
ALTER TABLE orders VALIDATE CONSTRAINT orders_discount_pct_nn;
-- Drop the CHECK constraint and add the actual NOT NULL:
ALTER TABLE orders ALTER COLUMN discount_pct SET NOT NULL;
-- This is a metadata-only operation once VALIDATE CONSTRAINT has passed
```

**Adding an index without blocking:**

```sql
-- CONCURRENT index creation does not hold a table lock
-- It can run alongside inserts, updates, deletes (but takes longer)
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
-- Note: CONCURRENT indexes cannot be created inside a transaction block
```

**Index creation on replicas:** When you run `CREATE INDEX CONCURRENTLY` on the primary, it replicates to replicas as a standard blocking `CREATE INDEX` (the non-concurrent WAL record). To avoid blocking replicas during large index builds on hot systems, some teams use `pg_repack` or accept a brief replication delay on replicas during the build.

---

## The Mental Model: Putting It Together

Here is how every concept in this guide connects:

```
WAL is the source of truth for everything:
  ├── Physical replication: ship WAL bytes to replicas
  ├── Logical replication: decode WAL into row-level changes
  ├── PITR backups: archive WAL to object storage
  └── Crash recovery: replay WAL after a crash

Synchronous_commit controls the durability/latency trade-off:
  ├── off → fastest, can lose committed data on crash
  ├── local → safe on primary crash, not on primary+replica crash
  ├── remote_write → safe unless both primary OS and replica crash simultaneously
  ├── remote_apply → read your writes from replica guaranteed
  └── on → strongest durability, highest commit latency

Replication slots prevent WAL deletion — use with care:
  ├── Essential for replicas that may disconnect temporarily
  └── Fatal if the replica disappears permanently without slot cleanup

Patroni provides the control plane:
  ├── DCS (etcd) = single source of truth for "who is primary"
  ├── Leader lease = prevents split-brain
  ├── automatic promotion = sub-60s RTO
  └── pg_rewind = fast rejoining of failed primary as replica

PgBouncer provides the connection layer:
  ├── Transaction pooling = 1000 app connections → 50 DB connections
  ├── session_reset_query = clean state between transactions
  └── health checks = automatic removal of broken server connections

Monitoring is your early warning system:
  ├── pg_stat_replication = lag, sync state, slot health
  ├── pg_stat_wal_receiver = replica-side connectivity
  └── Alert on: slot bloat, disconnected replicas, growing lag
```

> The engineer who understands WAL flow, synchronous_commit trade-offs, and slot lifecycle does not panic when a primary dies. They already know what the system will do, what they need to verify, and what to watch as the cluster heals. That knowledge is what this guide was built to give you.
