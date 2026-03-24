# Database Security and Access Control
## Roles, Privileges, RLS, Encryption, and Auditing in PostgreSQL

> "The most common database breach isn't a sophisticated attack — it's an app role with superuser privileges and no row-level controls. Defense starts at the schema."

---

## Table of Contents

1. [The Threat Model](#1-the-threat-model)
2. [PostgreSQL Role System](#2-postgresql-role-system)
3. [Privileges — The Complete Picture](#3-privileges--the-complete-picture)
4. [Row-Level Security (RLS)](#4-row-level-security-rls)
5. [pg_hba.conf — Authentication Configuration](#5-pg_hbaconf--authentication-configuration)
6. [SSL/TLS — Encrypting Connections](#6-ssltls--encrypting-connections)
7. [Encryption at Rest](#7-encryption-at-rest)
8. [SQL Injection — Database-Level Defenses](#8-sql-injection--database-level-defenses)
9. [Auditing with pgaudit](#9-auditing-with-pgaudit)
10. [Secrets Management](#10-secrets-management)
11. [Security Hardening Checklist and Audit Queries](#11-security-hardening-checklist-and-audit-queries)

---

## 1. The Threat Model

Before configuring anything, understand what you're actually defending against.

### 1.1 Compromised Application Credentials (Most Common)

Your Django app connects to PostgreSQL with a username and password. If that password leaks (environment variable exposed, config file committed, memory dump), an attacker has everything the app has.

**What stops the damage**: The app role should only have `SELECT`, `INSERT`, `UPDATE`, `DELETE` on specific tables — never `DROP`, `TRUNCATE`, `CREATE`, or superuser. Least privilege limits the blast radius.

### 1.2 SQL Injection

An attacker crafts input that becomes part of a SQL query. Even with parameterized queries in your app, defense-in-depth at the DB layer helps.

**What stops the damage**: A role with only `SELECT` on specific tables can't drop tables or read other schemas even if injection succeeds. Row-Level Security limits which rows can be read.

### 1.3 Insider Threats

A developer or DBA with too much access accidentally or maliciously reads sensitive data (PII, payments, health records).

**What stops the damage**: Role separation — no one person has access to everything. Audit logging via pgaudit shows who accessed what and when.

### 1.4 Backup Theft / Data Exfiltration

A database backup copied off the server contains everything unless encrypted.

**What stops the damage**: Encryption at rest (filesystem or pgcrypto for specific columns). Encrypted backups. Access controls on backup storage.

### 1.5 Privilege Escalation

An attacker with limited DB access finds a way to gain superuser or higher-privilege role access.

**What stops the damage**: Never use `SECURITY DEFINER` functions carelessly. Lock down `search_path`. Audit all superuser accounts.

---

## 2. PostgreSQL Role System

### 2.1 Roles Are Users — They're the Same Thing

```sql
-- These are equivalent:
CREATE USER app_user WITH PASSWORD 'secret';
CREATE ROLE app_user WITH LOGIN PASSWORD 'secret';

-- A ROLE without LOGIN cannot connect to the database
CREATE ROLE readonly_role;  -- no LOGIN — this is a group role

-- Grant group role membership to a user:
GRANT readonly_role TO app_user;
```

### 2.2 Key Role Attributes

```sql
CREATE ROLE app_user WITH
    LOGIN                    -- can connect to the database
    PASSWORD 'strong_pass'
    NOSUPERUSER              -- cannot bypass all permission checks
    NOCREATEDB               -- cannot create databases
    NOCREATEROLE             -- cannot create other roles
    NOINHERIT                -- does NOT inherit permissions from parent roles automatically
    CONNECTION LIMIT 20;     -- max simultaneous connections (protects against connection exhaustion)
```

### 2.3 Role Hierarchy for a Production Application

Design your roles like layers. No single role should have everything.

```sql
-- ── LAYER 1: Group roles (no LOGIN, define permission sets) ──────────

-- Read-only: for analytics, reporting, read replicas
CREATE ROLE readonly_role NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT;

-- Application: for your Django/backend app (most restricted)
CREATE ROLE app_role NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT;

-- Migration: for running schema changes (more permissions, never in app)
CREATE ROLE migration_role NOSUPERUSER NOCREATEDB NOCREATEROLE NOINHERIT;

-- ── LAYER 2: Login users (inherit from group roles) ──────────────────

-- The actual app user (used in DATABASE_URL)
CREATE USER app_user WITH PASSWORD 'app_strong_pass' CONNECTION LIMIT 50;
GRANT app_role TO app_user;

-- Read-only analytics user
CREATE USER analytics_user WITH PASSWORD 'analytics_pass' CONNECTION LIMIT 10;
GRANT readonly_role TO analytics_user;

-- Migration user (used only during deployments, never in running app)
CREATE USER migration_user WITH PASSWORD 'migration_pass' CONNECTION LIMIT 3;
GRANT migration_role TO migration_user;

-- DBA user (used only for maintenance, never in app connection strings)
CREATE USER dba_user WITH PASSWORD 'dba_pass' CREATEDB CREATEROLE CONNECTION LIMIT 5;
```

### 2.4 SUPERUSER — The Nuclear Option

```sql
-- Never do this for an application role:
ALTER USER app_user SUPERUSER;
-- Superuser bypasses ALL permission checks. If this account is compromised,
-- the attacker can read everything, drop everything, create backdoors.

-- Audit who has superuser right now:
SELECT usename, usesuper, usecreatedb, usecreaterole
FROM pg_user
WHERE usesuper = true;
```

---

## 3. Privileges — The Complete Picture

### 3.1 Object-Level Privileges

```sql
-- Grant to app_role on specific tables:
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE users TO app_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE orders TO app_role;
GRANT USAGE, SELECT ON SEQUENCE users_id_seq TO app_role;
GRANT USAGE, SELECT ON SEQUENCE orders_id_seq TO app_role;

-- Read-only role gets only SELECT:
GRANT SELECT ON TABLE users TO readonly_role;
GRANT SELECT ON TABLE orders TO readonly_role;

-- Migration role gets DDL-level access:
GRANT ALL ON ALL TABLES IN SCHEMA public TO migration_role;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO migration_role;
GRANT CREATE ON SCHEMA public TO migration_role;
```

### 3.2 Default Privileges — Critical for New Tables

Without this, every new table you create won't be accessible to existing roles:

```sql
-- Whenever the 'migration_user' creates a new table,
-- automatically grant SELECT,INSERT,UPDATE,DELETE to app_role:
ALTER DEFAULT PRIVILEGES FOR ROLE migration_user IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_role;

ALTER DEFAULT PRIVILEGES FOR ROLE migration_user IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly_role;

ALTER DEFAULT PRIVILEGES FOR ROLE migration_user IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO app_role;

-- Without this, you'd have to GRANT on every new table manually after migrations
```

### 3.3 The Public Schema Security Risk

By default, PostgreSQL grants `CREATE` on the `public` schema to every user. This means any authenticated user can create tables, functions, or operators in `public` — a potential privilege escalation vector.

```sql
-- Fix it: revoke CREATE from PUBLIC on the public schema
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- Also revoke CONNECT privilege from PUBLIC on the database
-- (force explicit grants instead):
REVOKE CONNECT ON DATABASE mydb FROM PUBLIC;

-- Then explicitly grant what's needed:
GRANT CONNECT ON DATABASE mydb TO app_role;
GRANT CONNECT ON DATABASE mydb TO readonly_role;
GRANT CONNECT ON DATABASE mydb TO migration_role;
GRANT USAGE ON SCHEMA public TO app_role;
GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT CREATE ON SCHEMA public TO migration_role;
```

### 3.4 Column-Level Privileges

For sensitive columns (SSN, credit card, health data):

```sql
-- Revoke access to the entire table first:
REVOKE SELECT ON TABLE users FROM readonly_role;

-- Grant only non-sensitive columns:
GRANT SELECT (id, email, created_at, username) ON TABLE users TO readonly_role;
-- readonly_role can NOT read ssn, date_of_birth, phone_number
```

### 3.5 Schema Privileges

```sql
-- USAGE: allows referencing objects in the schema (needed to SELECT from tables)
-- CREATE: allows creating new objects in the schema

GRANT USAGE ON SCHEMA public TO app_role;
GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT USAGE, CREATE ON SCHEMA public TO migration_role;

-- Multi-schema setup (separate schemas for isolation):
CREATE SCHEMA app_data;
CREATE SCHEMA analytics;

GRANT USAGE ON SCHEMA app_data TO app_role;
GRANT USAGE ON SCHEMA analytics TO readonly_role;
-- app_role cannot see analytics schema at all
```

---

## 4. Row-Level Security (RLS)

RLS is the most powerful and most underused PostgreSQL security feature. It makes security part of the database, not just the application layer.

### 4.1 How RLS Works

Without RLS, a role with `SELECT` on a table can read **all rows**. With RLS, the database itself filters rows based on policies before returning results — even if the app's query has no WHERE clause.

```sql
-- Enable RLS on a table:
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- By default, with RLS enabled and no policies, NO rows are visible
-- (except to table owner and superuser)

-- Create a policy that allows users to see only their own orders:
CREATE POLICY user_orders_policy ON orders
    FOR ALL  -- applies to SELECT, INSERT, UPDATE, DELETE
    TO app_role
    USING (user_id = current_setting('app.current_user_id')::integer);
```

### 4.2 Setting the Application Context

The policy uses `current_setting()` — your app must set this at the start of each request:

```python
# Django: set the user context before any DB operation in the request
# In middleware or at the start of each view:

from django.db import connection

def set_rls_context(user_id):
    with connection.cursor() as cursor:
        cursor.execute("SET LOCAL app.current_user_id = %s", [user_id])
        # SET LOCAL means it's reset at the end of the transaction
```

```sql
-- Verify it works:
SET app.current_user_id = '42';

SELECT * FROM orders;
-- Returns ONLY rows where user_id = 42, regardless of what the query says

SELECT * FROM orders WHERE user_id = 99;
-- Returns 0 rows — RLS filters FIRST, then the WHERE clause applies
```

### 4.3 Multi-Tenant SaaS — The Killer Use Case

Every query automatically scoped to the correct tenant — no chance of data leakage even if a developer forgets a WHERE clause:

```sql
-- Add tenant_id to every table:
ALTER TABLE users ADD COLUMN tenant_id INTEGER NOT NULL;
ALTER TABLE orders ADD COLUMN tenant_id INTEGER NOT NULL;
ALTER TABLE products ADD COLUMN tenant_id INTEGER NOT NULL;

-- Enable RLS on all tenant tables:
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- Create policies using a tenant context variable:
CREATE POLICY tenant_isolation ON users
    FOR ALL TO app_role
    USING (tenant_id = current_setting('app.tenant_id')::integer)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::integer);

CREATE POLICY tenant_isolation ON orders
    FOR ALL TO app_role
    USING (tenant_id = current_setting('app.tenant_id')::integer)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::integer);
```

```python
# Django middleware to set tenant context on every request:
class TenantRLSMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.user.is_authenticated:
            tenant_id = request.user.tenant_id
            with connection.cursor() as cursor:
                cursor.execute(
                    "SELECT set_config('app.tenant_id', %s, FALSE)",
                    [str(tenant_id)]
                )
                # FALSE = not local, persists for the connection
                # Use TRUE (SET LOCAL) if using connection pooling in transaction mode
        return self.get_response(request)
```

### 4.4 USING vs WITH CHECK

```sql
-- USING: filters rows for SELECT, UPDATE, DELETE
--   "which rows can this role see/affect?"

-- WITH CHECK: validates rows for INSERT, UPDATE
--   "which rows can this role create/modify?"

-- Read-only policy (can see own rows, can't modify):
CREATE POLICY read_own_data ON orders
    FOR SELECT TO app_role
    USING (user_id = current_setting('app.current_user_id')::integer);

-- Write policy (can only insert/update with own user_id):
CREATE POLICY write_own_data ON orders
    FOR INSERT TO app_role
    WITH CHECK (user_id = current_setting('app.current_user_id')::integer);

CREATE POLICY update_own_data ON orders
    FOR UPDATE TO app_role
    USING (user_id = current_setting('app.current_user_id')::integer)
    WITH CHECK (user_id = current_setting('app.current_user_id')::integer);
```

### 4.5 Permissive vs Restrictive Policies

```sql
-- Default (PERMISSIVE): row is visible if ANY policy allows it (OR logic)
CREATE POLICY p1 ON t AS PERMISSIVE FOR SELECT USING (owner = current_user);

-- RESTRICTIVE: row is visible only if ALL restrictive policies allow it (AND logic)
-- Use for mandatory access controls that must ALWAYS apply:
CREATE POLICY always_active ON users AS RESTRICTIVE
    USING (is_deleted = false);
-- Even if another policy would show deleted users, this blocks them
```

### 4.6 BYPASSRLS

```sql
-- Table owners and superusers bypass RLS by default
-- For a DBA role that needs to see all rows without bypassing superuser:
ALTER ROLE dba_user BYPASSRLS;

-- The migration role also typically needs to bypass RLS:
ALTER ROLE migration_role BYPASSRLS;
```

### 4.7 Performance: Index Your RLS Columns

```sql
-- Your policy:
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::integer);

-- Without this index, every query does a full table scan filtered by tenant:
CREATE INDEX idx_orders_tenant_id ON orders (tenant_id);

-- For composite queries:
CREATE INDEX idx_orders_tenant_status ON orders (tenant_id, status);
-- Supports: WHERE tenant_id = X AND status = Y
```

---

## 5. pg_hba.conf — Authentication Configuration

### 5.1 What pg_hba.conf Controls

`pg_hba.conf` controls **who can connect** and **how they authenticate**. It does NOT control what they can do after connecting (that's privileges and RLS).

Format: `TYPE  DATABASE  USER  ADDRESS  METHOD`

### 5.2 Authentication Methods

| Method | Security | When to Use |
|--------|----------|-------------|
| `trust` | None — anyone can connect | Never in production |
| `md5` | Weak — sends MD5 hash | Legacy, avoid |
| `scram-sha-256` | Strong — challenge-response | Always use this |
| `peer` | OS user = DB user | Local connections on same server |
| `cert` | Certificate-based | Highest security, complex setup |
| `reject` | Deny connection | Block specific hosts |

### 5.3 Production pg_hba.conf

```
# TYPE   DATABASE    USER              ADDRESS           METHOD

# Local superuser access via peer (no password, OS-authenticated):
local    all         postgres                            peer

# Local connections for app (requires password):
local    mydb        app_user                            scram-sha-256
local    mydb        migration_user                      scram-sha-256

# Remote connections — only from known app server IPs:
hostssl  mydb        app_user          10.0.1.0/24       scram-sha-256
hostssl  mydb        analytics_user    10.0.2.0/24       scram-sha-256

# Replication connections (for streaming replication):
hostssl  replication replication_user  10.0.0.0/24       scram-sha-256

# Deny everything else:
host     all         all               0.0.0.0/0         reject
```

Key points:
- Use `hostssl` (requires SSL) not `host` for remote connections
- Restrict by IP range — don't use `0.0.0.0/0` except for reject rules
- Use `scram-sha-256` not `md5`
- Reload after changes: `SELECT pg_reload_conf();`

---

## 6. SSL/TLS — Encrypting Connections

### 6.1 Enable SSL in postgresql.conf

```
# postgresql.conf:
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'          # for client certificate verification

# Minimum TLS version:
ssl_min_protocol_version = 'TLSv1.2'
```

### 6.2 Connection String SSL Modes

| sslmode | What It Does | When to Use |
|---------|-------------|-------------|
| `disable` | No SSL | Never in production |
| `allow` | Use SSL if server supports it | Never |
| `prefer` | Use SSL if possible, fallback to no SSL | Never |
| `require` | Require SSL, don't verify cert | Encrypts but vulnerable to MITM |
| `verify-ca` | Require SSL + verify server cert from CA | Good |
| `verify-full` | Require SSL + verify cert + hostname | Best |

```python
# Django settings.py — production database config with SSL:
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'app_user',
        'PASSWORD': os.environ['DB_PASSWORD'],
        'HOST': 'db.example.com',
        'PORT': '5432',
        'OPTIONS': {
            'sslmode': 'verify-full',
            'sslrootcert': '/etc/ssl/certs/db-ca.crt',
        },
    }
}

# Or via DATABASE_URL (with dj-database-url):
# DATABASE_URL=postgresql://app_user:pass@db.example.com/mydb?sslmode=verify-full
```

---

## 7. Encryption at Rest

### 7.1 What Encryption at Rest Protects

- Physical theft of disk/storage
- Unauthorized access to backup files
- Cloud provider access to raw storage

**What it does NOT protect**: Compromised application credentials. If an attacker has valid DB credentials, encryption at rest doesn't help — they're reading decrypted data through normal connections.

### 7.2 Filesystem-Level Encryption

The simplest approach — encrypt at the OS/storage level:
- AWS RDS: enable storage encryption (AES-256) — one checkbox
- Self-hosted: LUKS on Linux for encrypted volumes
- PostgreSQL has no built-in full-database encryption

### 7.3 Column-Level Encryption with pgcrypto

For specific sensitive columns (SSN, payment data, health records):

```sql
-- Enable the extension:
CREATE EXTENSION pgcrypto;

-- Encrypt on insert:
INSERT INTO users (email, ssn_encrypted)
VALUES (
    'user@example.com',
    pgp_sym_encrypt('123-45-6789', current_setting('app.encryption_key'))
);

-- Decrypt on read:
SELECT
    email,
    pgp_sym_decrypt(ssn_encrypted::bytea, current_setting('app.encryption_key')) AS ssn
FROM users
WHERE id = 42;
```

```python
# Django: set the encryption key at connection time
# (key comes from environment/vault, not hardcoded)
with connection.cursor() as cursor:
    cursor.execute(
        "SELECT set_config('app.encryption_key', %s, FALSE)",
        [settings.DB_ENCRYPTION_KEY]
    )
```

### 7.4 Password Hashing with pgcrypto

```sql
-- Hash a password (never store plaintext):
INSERT INTO users (email, password_hash)
VALUES (
    'user@example.com',
    crypt('user_password', gen_salt('bf', 12))  -- bcrypt, cost factor 12
);

-- Verify password:
SELECT email
FROM users
WHERE email = 'user@example.com'
  AND password_hash = crypt('provided_password', password_hash);
-- Returns row if password matches, empty if not
```

> **In practice**: Use your application's password hashing (Django's `make_password` / `check_password` with bcrypt). Only use pgcrypto if you need DB-side verification without going through the app.

---

## 8. SQL Injection — Database-Level Defenses

### 8.1 Why Parameterized Queries Are Not Enough Alone

Parameterized queries prevent injection at the application layer. But defense in depth means the database itself should also limit damage.

```python
# Good — parameterized (safe from injection):
cursor.execute("SELECT * FROM users WHERE email = %s", [user_input])

# Bad — string concatenation (injectable):
cursor.execute(f"SELECT * FROM users WHERE email = '{user_input}'")
# Input: ' OR '1'='1 → SELECT * FROM users WHERE email = '' OR '1'='1'
```

### 8.2 Privilege Isolation Limits Injection Impact

Even if injection succeeds, the damage is bounded by the role's privileges:

```sql
-- Scenario: injection in an app running as app_role (SELECT, INSERT, UPDATE, DELETE only)

-- Attacker tries to drop a table:
-- DROP TABLE users; -- ERROR: permission denied for table users
-- app_role has no DROP privilege → query fails

-- Attacker tries to read system passwords:
-- SELECT * FROM pg_shadow; -- ERROR: permission denied for view pg_shadow
-- Only superusers can read pg_shadow

-- Attacker tries to read another tenant's data:
-- SELECT * FROM orders; -- Returns ONLY current tenant's orders (RLS!)
```

### 8.3 SECURITY DEFINER — A Privilege Escalation Risk

```sql
-- SECURITY DEFINER: function runs with the OWNER's privileges, not the caller's
-- Dangerous if the owner is a high-privilege role

CREATE OR REPLACE FUNCTION get_all_users()
RETURNS TABLE(id INT, email TEXT) AS $$
    SELECT id, email FROM users;  -- runs as function owner, not caller
$$ LANGUAGE SQL SECURITY DEFINER;
-- If function owner is a superuser, any caller effectively has superuser access to 'users'

-- Safer: always set search_path in SECURITY DEFINER functions:
CREATE OR REPLACE FUNCTION safe_function()
RETURNS void AS $$
BEGIN
    -- do something
END;
$$ LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp;
-- Prevents search_path manipulation attacks
```

### 8.4 search_path Manipulation

```sql
-- Attack: create a table in a schema that shadows a trusted function
CREATE SCHEMA attacker;
CREATE FUNCTION attacker.pg_has_role(...)  -- shadows the real function
-- Then if search_path includes 'attacker' first, calls to pg_has_role use the fake one

-- Defense: always explicitly schema-qualify in SECURITY DEFINER functions
-- and lock down search_path:
ALTER ROLE app_user SET search_path = myschema, public;
```

---

## 9. Auditing with pgaudit

### 9.1 What pgaudit Does

PostgreSQL's default logging captures connection events and slow queries. `pgaudit` adds **statement-level audit logging**: who ran what SQL, when, on which objects.

```sql
-- Install:
CREATE EXTENSION pgaudit;
```

### 9.2 Configuration

```
# postgresql.conf:

# Load the extension:
shared_preload_libraries = 'pgaudit'

# What to log:
pgaudit.log = 'read, write, ddl, role'
# read  = SELECT, COPY FROM
# write = INSERT, UPDATE, DELETE, TRUNCATE, COPY TO
# ddl   = CREATE, ALTER, DROP
# role  = GRANT, REVOKE, CREATE/ALTER/DROP ROLE
# misc  = FETCH, CHECKPOINT, etc.
# all   = everything

# Log catalog access (system tables):
pgaudit.log_catalog = on

# Log the parameters passed to parameterized queries:
pgaudit.log_parameter = on

# Log relation names in statement:
pgaudit.log_relation = on
```

### 9.3 Object-Level Auditing

Log access to specific sensitive tables only:

```sql
-- Enable object-level auditing for a specific role:
ALTER ROLE analytics_user SET pgaudit.log = 'read';

-- Now create an audit rule for a specific table:
-- (pgaudit object auditing requires pgaudit.log_level = 'log' and
--  pgaudit.role set to an audit role)

CREATE ROLE auditor;
GRANT SELECT ON TABLE users TO auditor;
GRANT SELECT ON TABLE payments TO auditor;

-- In postgresql.conf:
-- pgaudit.role = 'auditor'
-- Any access to tables the 'auditor' role has access to will be logged
```

### 9.4 Sample Audit Log Output

```
AUDIT: SESSION,1,1,READ,SELECT,TABLE,public.users,
"SELECT id, email, ssn_encrypted FROM users WHERE id = 42",<not logged>
```

### 9.5 Compliance Use Case

```
# For PCI-DSS / HIPAA compliance:
pgaudit.log = 'read, write, ddl, role'
pgaudit.log_parameter = on
pgaudit.log_relation = on
log_connections = on
log_disconnections = on
log_duration = on
```

---

## 10. Secrets Management

### 10.1 Never Hardcode Credentials

```python
# BAD — committed to git, readable by anyone with repo access:
DATABASES = {
    'default': {
        'PASSWORD': 'mydbpassword123',
    }
}

# BETTER — environment variable:
DATABASES = {
    'default': {
        'PASSWORD': os.environ['DB_PASSWORD'],
    }
}
# Still risky: env vars can leak in logs, error messages, /proc

# BEST — secrets manager:
import boto3
def get_db_password():
    client = boto3.client('secretsmanager', region_name='us-east-1')
    response = client.get_secret_value(SecretId='prod/myapp/db')
    return json.loads(response['SecretString'])['password']
```

### 10.2 Password Rotation Without Downtime

```sql
-- Create new password without dropping old connection:
ALTER USER app_user PASSWORD 'new_strong_password';
-- Existing connections using old password stay alive
-- New connections use new password
-- Update your secrets manager, then rotate app config with rolling restart
```

### 10.3 Django: Secure Database Configuration

```python
# settings.py — production pattern:
import os

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ['DB_NAME'],
        'USER': os.environ['DB_USER'],
        'PASSWORD': os.environ['DB_PASSWORD'],
        'HOST': os.environ['DB_HOST'],
        'PORT': os.environ.get('DB_PORT', '5432'),
        'CONN_MAX_AGE': 60,  # persistent connections
        'OPTIONS': {
            'sslmode': 'verify-full',
            'sslrootcert': os.environ.get('DB_CA_CERT', '/etc/ssl/certs/db-ca.crt'),
            'connect_timeout': 10,
            'options': '-c statement_timeout=30000'  # 30s query timeout
        },
        'CONN_HEALTH_CHECKS': True,  # Django 4.1+
    }
}
```

---

## 11. Security Hardening Checklist and Audit Queries

### 11.1 Hardening Checklist

```sql
-- 1. Revoke CREATE on public schema from PUBLIC:
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- 2. Revoke CONNECT from PUBLIC:
REVOKE CONNECT ON DATABASE mydb FROM PUBLIC;

-- 3. Verify no application roles have SUPERUSER:
SELECT rolname, rolsuper, rolcreatedb, rolcreaterole
FROM pg_roles
WHERE rolsuper = true;

-- 4. Ensure app_user is not the table owner:
-- Table owner can always ALTER/DROP regardless of grants
-- Use migration_user as owner, grant to app_role separately

-- 5. No trust authentication in pg_hba.conf:
-- Grep pg_hba.conf for 'trust' and remove/replace with scram-sha-256
```

### 11.2 Audit Queries — Know Who Has What

```sql
-- Who has access to which tables:
SELECT
    grantee,
    table_schema,
    table_name,
    string_agg(privilege_type, ', ' ORDER BY privilege_type) AS privileges
FROM information_schema.role_table_grants
WHERE table_schema = 'public'
GROUP BY grantee, table_schema, table_name
ORDER BY table_name, grantee;

-- Which roles have superuser, createdb, createrole:
SELECT
    rolname,
    rolsuper,
    rolcreatedb,
    rolcreaterole,
    rolcanlogin,
    rolconnlimit,
    rolvaliduntil
FROM pg_roles
ORDER BY rolsuper DESC, rolname;

-- Role memberships (who is in which group):
SELECT
    r.rolname AS role,
    m.rolname AS member_of
FROM pg_auth_members am
JOIN pg_roles r ON r.oid = am.roleid
JOIN pg_roles m ON m.oid = am.member
ORDER BY m.rolname;

-- Active connections and their roles:
SELECT
    usename,
    application_name,
    client_addr,
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY usename, application_name, client_addr, state
ORDER BY connections DESC;

-- Tables with RLS enabled:
SELECT schemaname, tablename, rowsecurity, forcerowsecurity
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;

-- All RLS policies:
SELECT
    schemaname,
    tablename,
    policyname,
    permissive,
    roles,
    cmd,
    qual,
    with_check
FROM pg_policies
ORDER BY tablename, policyname;

-- Find tables with no RLS (that maybe should have it):
SELECT t.tablename
FROM pg_tables t
LEFT JOIN pg_class c ON c.relname = t.tablename
WHERE t.schemaname = 'public'
  AND c.relrowsecurity = false
ORDER BY t.tablename;
```

### 11.3 postgresql.conf Security Settings

```
# Logging for security events:
log_connections = on
log_disconnections = on
log_failed_connections = on
log_duration = on
log_min_duration_statement = 1000   # log queries > 1 second

# Prevent privilege escalation via search_path:
# Per-role setting (apply to all app roles):
# ALTER ROLE app_user SET search_path = public;

# SSL:
ssl = on
ssl_min_protocol_version = 'TLSv1.2'

# Connection limits:
max_connections = 200    # set appropriately, use PgBouncer for pooling

# Timeout to prevent hung connections:
idle_in_transaction_session_timeout = '10min'
statement_timeout = '30s'     # set per-role for app users
lock_timeout = '5s'           # set per-session before DDL
```

### 11.4 Quick Security Verification

```sql
-- Run this to get a security overview:
SELECT 'Superusers' AS check_name,
       string_agg(rolname, ', ') AS details
FROM pg_roles WHERE rolsuper = true
UNION ALL
SELECT 'Roles with no password expiry',
       string_agg(rolname, ', ')
FROM pg_roles WHERE rolcanlogin AND rolvaliduntil IS NULL
UNION ALL
SELECT 'Tables without RLS',
       string_agg(tablename, ', ')
FROM pg_tables t
JOIN pg_class c ON c.relname = t.tablename
WHERE t.schemaname = 'public' AND NOT c.relrowsecurity;
```
