# Operations — SQL

Contents: Migrations · Zero-Downtime Changes · Backup & Restore · Maintenance · Monitoring · Connection Pooling · Replication

## Migrations

```
migrations/
├── 001_create_users.sql
├── 002_create_orders.sql
├── 003_add_users_phone.sql
```

```sql
CREATE TABLE IF NOT EXISTS schema_migrations (
    version TEXT PRIMARY KEY,
    applied_at TIMESTAMPTZ DEFAULT NOW()
);
```

Use an established runner in production — golang-migrate, Flyway, sqitch, Alembic, Prisma Migrate, Liquibase. They handle what ad-hoc scripts miss: transactional application, concurrent-run locking, dirty-state detection. Two rules regardless of tool:

- Applied migrations are immutable — fix mistakes with a new migration, never by editing an applied file (checksums diverge across environments).
- One logical change per migration: a half-applied 400-line migration is unrecoverable by tooling.

## Zero-Downtime Changes

The failure mode isn't slow DDL — most DDL is instant. It's the lock queue: `ALTER TABLE` needs an exclusive lock, waits behind one long-running query, and every new query then queues behind the ALTER. A 100ms change becomes a full outage. Cap the wait first:

```sql
SET lock_timeout = '2s';   -- ALTER fails fast instead of stalling all traffic
ALTER TABLE users ADD COLUMN phone TEXT;
-- on failure: find/kill the blocker (→ Monitoring), retry
```

Per-change playbook (PostgreSQL specifics noted):

```sql
-- Add nullable column: metadata-only, instant
ALTER TABLE users ADD COLUMN phone TEXT;

-- Add column with default: instant in PostgreSQL >=11 (stored as metadata);
-- earlier versions rewrite the whole table
ALTER TABLE users ADD COLUMN status TEXT NOT NULL DEFAULT 'active';

-- NOT NULL on an existing big column, without a long table lock:
ALTER TABLE users ADD CONSTRAINT users_email_nn
    CHECK (email IS NOT NULL) NOT VALID;      -- instant, applies to new rows
-- backfill existing NULLs in bounded batches (UPDATE ... WHERE id BETWEEN :lo AND :hi
-- in a loop, keyset-advancing, committing each batch), then:
ALTER TABLE users VALIDATE CONSTRAINT users_email_nn;  -- scans without blocking writes

-- Index on a live table: CONCURRENTLY doesn't block writes.
-- Caveats: cannot run inside a transaction; a failed build leaves an INVALID
-- index that still taxes writes — check and drop:
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
```

Rename/retype/drop = expand-migrate-contract, because old code runs against the new schema during every deploy. Rename `email` → `email_address` in three deploys:

1. **Expand**: add `email_address`; dual-write both columns from app code; backfill old rows in batches.
2. **Migrate**: switch all reads to `email_address`; verify no traffic touches `email` (log or `pg_stat` checks).
3. **Contract**: drop the old column in a later deploy.

Same shape for type changes (new column of new type) and table renames. A one-deploy `ALTER TABLE ... RENAME` is only safe when you control the maintenance window.

## Backup & Restore

A backup is untested until you've restored it — schedule a periodic restore drill into a scratch database and run row-count/checksum spot checks. Every other property of the backup is secondary.

### PostgreSQL

```bash
pg_dump -Fc mydb > backup.dump                   # custom format: compressed, selective restore
pg_dump -Fc -t users -t orders mydb > partial.dump
pg_dump -Fc --schema-only mydb > schema.dump

pg_restore -d mydb --clean --if-exists backup.dump
pg_restore -d mydb -j 4 backup.dump              # parallel restore (custom/dir formats only)

pg_dump mydb > backup.sql                        # plain SQL: portable, no parallel restore
psql -c "\copy (SELECT * FROM users) TO 'users.csv' CSV HEADER"
```

`pg_dump` is a point-in-time snapshot: everything since the dump is lost on restore. When the tolerable loss window is minutes, not hours — or dump duration stops fitting the night — move to WAL archiving with pgBackRest or WAL-G (continuous backup, point-in-time recovery).

### SQLite

```bash
sqlite3 mydb.sqlite ".backup backup.sqlite"      # safe during writes; plain cp is not
sqlite3 mydb.sqlite .dump > backup.sql
sqlite3 newdb.sqlite < backup.sql
```

### MySQL

```bash
mysqldump --single-transaction mydb > backup.sql # consistent snapshot without locking (InnoDB)
mysqldump --single-transaction mydb users orders > partial.sql
mysql mydb < backup.sql
```

### SQL Server

```bash
sqlcmd -S localhost -U sa -Q "BACKUP DATABASE mydb TO DISK='backup.bak'"
sqlcmd -S localhost -U sa -Q "RESTORE DATABASE mydb FROM DISK='backup.bak'"
```

## Maintenance

### PostgreSQL

```sql
ANALYZE users;          -- refresh planner stats; first move after bulk load / bad estimates
VACUUM users;           -- reclaim dead tuples for reuse (does not shrink the file)
```

- `VACUUM FULL` shrinks the file but takes an exclusive lock for the whole rewrite — on a production table use pg_repack (online) instead.
- Autovacuum triggers when dead tuples exceed `autovacuum_vacuum_scale_factor` (default 0.2 = 20% of the table). On a 100M-row hot table that means 20M dead rows of bloat before it fires — lower it per table: `ALTER TABLE big_hot SET (autovacuum_vacuum_scale_factor = 0.01);`
- Long-open transactions (rule 4, SKILL.md) pin dead tuples: vacuum can't clean anything newer than the oldest open snapshot. Bloat despite aggressive autovacuum → look for idle-in-transaction sessions first.

```sql
-- Table and index sizes: what's actually eating disk
SELECT relname, pg_size_pretty(pg_total_relation_size(relid))
FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC;

-- Unused indexes: pure write tax; verify uptime covers a full workload cycle before dropping
SELECT indexrelname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0;
```

### SQLite

```sql
VACUUM;                    -- rewrites the file; needs free disk ≈ db size
PRAGMA integrity_check;
PRAGMA optimize;           -- run before closing long-lived connections
```

### MySQL

```sql
ANALYZE TABLE users;       -- statistics
OPTIMIZE TABLE users;      -- rebuilds table + indexes, reclaims space (locks: online-ish in InnoDB)
SELECT table_name, ROUND(data_length/1024/1024, 2) AS size_mb
FROM information_schema.tables WHERE table_schema = 'mydb' ORDER BY data_length DESC;
```

## Monitoring

Canonical thresholds (referenced from SKILL.md): investigate any transaction or query older than **1 minute**; OLTP cache hit ratio should sit **above 99%** — sustained drops mean the working set outgrew memory.

### PostgreSQL

```sql
-- What is running right now
SELECT pid, NOW() - query_start AS duration, state, query
FROM pg_stat_activity
WHERE state != 'idle' AND query_start < NOW() - INTERVAL '1 minute'
ORDER BY duration DESC;

-- Idle-in-transaction: worse than active — holds locks and blocks vacuum while doing nothing
SELECT pid, NOW() - xact_start AS open_for, query
FROM pg_stat_activity WHERE state = 'idle in transaction';

-- Who blocks whom (then decide, then kill)
SELECT blocked.pid AS blocked_pid, blocking.pid AS blocking_pid,
       blocked_a.query AS blocked_query, blocking_a.query AS blocking_query
FROM pg_locks blocked
JOIN pg_stat_activity blocked_a ON blocked_a.pid = blocked.pid
JOIN pg_locks blocking ON blocking.locktype = blocked.locktype
    AND blocking.database IS NOT DISTINCT FROM blocked.database
    AND blocking.relation IS NOT DISTINCT FROM blocked.relation
    AND blocking.pid != blocked.pid
JOIN pg_stat_activity blocking_a ON blocking_a.pid = blocking.pid
WHERE NOT blocked.granted;

SELECT pg_cancel_backend(pid);     -- cancel query, keep connection (try first)
SELECT pg_terminate_backend(pid);  -- kill connection

-- Cache hit ratio
SELECT sum(blks_hit)*100/sum(blks_hit+blks_read) AS cache_hit_ratio
FROM pg_stat_database;

-- Which queries cost the most overall (rule 8, SKILL.md); needs
-- shared_preload_libraries = 'pg_stat_statements'
SELECT round(total_exec_time) AS total_ms, calls, round(mean_exec_time, 1) AS mean_ms, query
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

### MySQL

```sql
SHOW PROCESSLIST;                        -- KILL <id> to terminate
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;          -- seconds; then rank by total time with pt-query-digest
SHOW ENGINE INNODB STATUS;               -- deadlock section is the useful part
```

## Connection Pooling

Size pools from CPU, not from "more is faster": throughput peaks near `cores × 2` active connections (HikariCP guidance) and degrades beyond — hundreds of connections mostly buy context-switching and lock contention. PostgreSQL spawns a process per connection, so many app instances × generous pools exhausts `max_connections` fast; put PgBouncer in front instead of raising it.

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction        # server connection released at COMMIT — highest reuse
max_client_conn = 1000
default_pool_size = 20
```

Transaction mode breaks anything that assumes a stable session: session-level `SET`, `LISTEN/NOTIFY`, advisory locks, temp tables — and protocol-level prepared statements before PgBouncer 1.21 (which added support). Audit the app for these before switching from `session` mode.

Application pool knobs that matter: `max_connections` (per-instance ceiling — multiply by instance count when sizing the server), `max_lifetime` (recycle before any infra idle-timeout kills connections mid-query), `connection_timeout` (fail fast instead of piling up waiters during incidents).

## Replication (PostgreSQL)

```bash
# Primary: postgresql.conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB

# Primary: pg_hba.conf
host replication replicator replica_ip/32 md5

# Replica (bootstraps a copy and configures streaming)
pg_basebackup -h primary_ip -U replicator -D /var/lib/postgresql/data -P -R
```

```sql
-- On primary: per-replica state
SELECT client_addr, state, replay_lsn FROM pg_stat_replication;

-- On replica: lag as time. Gotcha: with zero write traffic this grows
-- unbounded — alert on lag only while the primary is receiving writes
SELECT NOW() - pg_last_xact_replay_timestamp() AS replication_lag;
```

Two facts to design around:

- Replication is async by default: an acknowledged commit can be lost on failover, and read-after-write against a replica can miss the write. Route read-your-own-writes traffic to the primary (or use synchronous replication and pay the latency).
- Long replica queries conflict with WAL replay and get canceled. `hot_standby_feedback = on` stops the cancellations but lets replica queries hold back vacuum on the primary — pick per workload: analytics replica → on, and accept primary bloat risk.

Replication is availability, not backup: a `DROP TABLE` replicates in milliseconds. You still need backups (→ Backup & Restore).
