# Operations

Contents: Migrations · Zero-Downtime Changes · Large-Table DDL · Backup & Restore · RPO/RTO · Restore Drills · Maintenance · Monitoring · Alert Thresholds · Incident Runbook · Connection Pooling · Replication · High Availability · Major Version Upgrades · Capacity Planning

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

Use an established runner in production — golang-migrate, Flyway, sqitch, Alembic, Prisma Migrate, Liquibase. They handle what ad-hoc scripts miss: transactional application, concurrent-run locking, dirty-state detection. Rules regardless of tool:

- Applied migrations are immutable — fix mistakes with a new migration, never by editing an applied file (checksums diverge across environments).
- One logical change per migration: a half-applied 400-line migration is unrecoverable by tooling. On MySQL, one DDL statement per migration, because there is no rollback.
- Migrations run under a lock so two deploying instances do not apply the same file twice. Confirm your runner takes one; some do not.
- Data backfills belong in their own migration (or a job), separated from DDL, and written to be resumable — a backfill that times out mid-run must restart from where it stopped, not from the beginning.
- Test every migration against production-shaped data before it ships.

## Zero-Downtime Changes

The failure mode isn't slow DDL — most DDL is instant. It's the lock queue: `ALTER TABLE` needs an exclusive lock, waits behind one long-running query, and every new query then queues behind the ALTER. A 100 ms change becomes a full outage. Cap the wait first:

```sql
SET lock_timeout = '2s';   -- ALTER fails fast instead of stalling all traffic
ALTER TABLE users ADD COLUMN phone TEXT;
-- on failure: find/kill the blocker (→ Monitoring), retry
```

The `2s` is the `lock_timeout` default from SKILL.md Configuration; a house standard replaces it in every DDL block emitted.

Per-change playbook (PostgreSQL specifics noted):

```sql
-- Add nullable column: metadata-only, instant
ALTER TABLE users ADD COLUMN phone TEXT;

-- Add column with a constant default: instant in PostgreSQL >=11 (stored as metadata);
-- a volatile default (now(), gen_random_uuid()) still rewrites the table
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

Ordering rule for adding a column the application will write: deploy the schema change first, then the code. For dropping: deploy the code that stops using it first, then the schema change. Getting the order backwards is the most common deploy-time outage.

## Large-Table DDL

- PostgreSQL: the online variants above cover most cases; `pg_repack` rebuilds a bloated table without an exclusive lock.
- MySQL: `ALGORITHM=INPLACE, LOCK=NONE` works for many changes but falls back to a copy for others without saying so — state it explicitly so the statement errors instead of locking the table for an hour. For anything it cannot do online, use `gh-ost` or `pt-online-schema-change`, which build a shadow table and swap it.
- SQL Server: `WITH (ONLINE = ON)` for index rebuilds on supported editions.
- Any tool that copies the table needs free disk equal to the table plus its indexes; check before starting, not at 90%.
- Announce and schedule: even online tools double the write load for the duration, and the swap step takes a brief exclusive lock.

## Backup & Restore

A backup is untested until you've restored it. Every other property of the backup is secondary.

### PostgreSQL

```bash
pg_dump -Fc mydb > backup.dump                   # custom format: compressed, selective restore
pg_dump -Fc -t users -t orders mydb > partial.dump
pg_dump -Fc --schema-only mydb > schema.dump

pg_restore -d mydb --clean --if-exists backup.dump
pg_restore -d mydb -j 4 backup.dump              # parallel restore (custom/dir formats only)

pg_dump mydb > backup.sql                        # plain SQL: portable, no parallel restore
```

`pg_dump` is a point-in-time snapshot: everything since the dump is lost on restore, and it does not include roles or other databases (`pg_dumpall --globals-only` does). When the tolerable loss window is minutes, not hours — or dump duration stops fitting the night — move to WAL archiving with pgBackRest or WAL-G (continuous backup, point-in-time recovery).

### MySQL

```bash
mysqldump --single-transaction mydb > backup.sql # consistent snapshot without locking (InnoDB)
mysqldump --single-transaction --routines --triggers --events mydb > full.sql
mysql mydb < backup.sql
```

`--single-transaction` is consistent for InnoDB only; a MyISAM table in the same database is dumped inconsistently and nothing warns you. Physical backups (Percona XtraBackup) restore far faster than a logical dump at any real size.

### SQLite

```bash
sqlite3 mydb.sqlite ".backup backup.sqlite"      # safe during writes; plain cp is not
sqlite3 mydb.sqlite .dump > backup.sql
sqlite3 newdb.sqlite < backup.sql
```

### SQL Server

```bash
sqlcmd -S localhost -U sa -Q "BACKUP DATABASE mydb TO DISK='backup.bak'"
sqlcmd -S localhost -U sa -Q "RESTORE DATABASE mydb FROM DISK='backup.bak'"
```

## RPO and RTO: Decide the Numbers First

- **RPO** (recovery point objective) = how much data you can lose. A nightly dump means an RPO of up to 24 hours, whatever anyone assumed.
- **RTO** (recovery time objective) = how long recovery may take. Measure it on a real restore of a real-sized database; logical restores of large databases run for hours, and index rebuilds dominate.
- Write both numbers down and check the backup strategy against them. Nightly dumps satisfy an RPO of 24 hours and nothing tighter; continuous WAL archiving gets to minutes; synchronous replication gets to near-zero at a latency cost.
- Keep multiple generations and at least one copy in a different failure domain. A backup on the same host protects against nothing that actually happens.
- Backups contain everything the database contains: encrypt them and restrict access.

## Restore Drills

Schedule a restore into a scratch environment on the user's stated Cadence (default: monthly) and verify, not just that it completed, but that it is right:

1. Restore to an isolated environment (masked or network-isolated — a drill is a common way production data leaks).
2. Row counts on the largest tables against the source.
3. A control total: `SUM` of a monetary column, `MIN`/`MAX` of the main timestamp column.
4. Run the application's smoke tests against the restored copy.
5. Record how long the whole thing took — that number is your real RTO.

Automate the drill. A documented but unpracticed restore procedure fails on details (missing roles, extensions, sequences) at the worst moment.

## Maintenance

### PostgreSQL

```sql
ANALYZE users;          -- refresh planner stats; first move after bulk load / bad estimates
VACUUM users;           -- reclaim dead tuples for reuse (does not shrink the file)
```

- `VACUUM FULL` shrinks the file but takes an exclusive lock for the whole rewrite — on a production table use pg_repack (online) instead.
- Autovacuum runs on its own schedule, so a large hot table can carry dead tuples long past the point where scans slow down. The observable symptom is on-disk size growing while the row count does not.
- Long-open transactions (SKILL.md rule 5) pin dead tuples: vacuum can't clean anything newer than the oldest open snapshot. Bloat despite aggressive autovacuum → look for idle-in-transaction sessions and inactive replication slots first.
- Every PostgreSQL knob behind this — autovacuum thresholds, freeze/wraparound, `work_mem`, `shared_buffers` — belongs to `pg`, not here.

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
PRAGMA journal_mode=WAL;   -- readers no longer block the writer
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

-- Which queries cost the most overall (SKILL.md rule 9); needs
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
SELECT * FROM sys.schema_unused_indexes; -- performance_schema view
```

## Alert Thresholds

Alert on symptoms a human can act on, not on raw counters. These levels are defaults; a shop with its own alerting standard overrides them under the Thresholds preference area.

| Signal | Page when | Why this level |
|---|---|---|
| Connections in use / max | > 80% sustained | Above this the next traffic spike fails to connect |
| Longest transaction age | > 1 min investigate, > 15 min page | Blocks vacuum and DDL; the canonical threshold above |
| Cache hit ratio (OLTP) | Sustained < 99% | Working set no longer fits memory |
| Replication lag | > the RPO you committed to | Ties the alert to a promise, not a guess |
| Disk free on the data volume | < 20% warn, < 10% page | A full data volume can wedge the server entirely |
| Deadlocks per minute | Any sustained non-zero rate | Occasional is normal; a rate means a lock-order bug |
| Failed connections / auth errors | Any spike | Credential rotation gone wrong, or an attack |
| Inactive replication slots (PostgreSQL) | Any, immediately | Retains WAL forever and fills the disk with no warning |
| Backup job age | > 1 scheduled interval + margin | A failed backup job is otherwise discovered during the incident |

Alerting on CPU alone produces noise: databases are supposed to use CPU. Alert on the queue behind the CPU (waiting connections, transaction age) instead.

## Incident Runbook

When the database is the suspect, in this order:

1. **Scope it.** All queries slow, or one? All clients, or one service? Started when, and what deployed then?
2. **Look at activity**, not at averages: the running-query list above, sorted by duration. One blocker at the top explains most incidents.
3. **Check the four resources**: connections in use, disk free, cache hit ratio, replication lag.
4. **Decide before killing.** Cancel first (`pg_cancel_backend`); terminate only if cancel does not work. Killing a long transaction rolls it back, and the rollback can take as long as the work did.
5. **Stop the bleeding before finding the cause**: cancel the runaway report, disable the feature flag, throttle the batch job. Root cause after service is restored.
6. **Do not restart the database** as a first move. It rolls back every open transaction, empties the cache, and produces a slow recovery period that looks like a second incident.
7. **Capture evidence while it is happening** — the activity list, the plan, the lock graph. It is unavailable afterwards.
8. **Write down the trigger and the fix.** Recurring incidents with no recorded cause are the same incident.

## Connection Pooling

Size pools from CPU, not from "more is faster": throughput peaks near `cores × 2` active connections (HikariCP guidance) and degrades beyond — hundreds of connections mostly buy context-switching and lock contention. PostgreSQL spawns a process per connection, so many app instances × generous pools exhausts `max_connections` fast; put PgBouncer in front instead of raising it.

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction        # server connection released at COMMIT — highest reuse
max_client_conn = 1000
default_pool_size = 20
```

Transaction mode breaks anything that assumes a stable session: session-level `SET`, `LISTEN/NOTIFY`, session advisory locks, temp tables — and protocol-level prepared statements before PgBouncer 1.21 (which added support). Audit the app for these before switching from `session` mode; `SET LOCAL` inside a transaction is safe.

Application pool knobs that matter: pool size (per-instance ceiling — multiply by instance count when sizing the server), `max_lifetime` (recycle before any infra idle-timeout kills connections mid-query), acquisition timeout (fail fast instead of piling up waiters during incidents).

## Replication

### PostgreSQL

```bash
# Primary: postgresql.conf
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB

# Primary: pg_hba.conf
host replication replicator replica_ip/32 scram-sha-256

# Replica (bootstraps a copy and configures streaming)
pg_basebackup -h primary_ip -U replicator -D /var/lib/postgresql/data -P -R
```

```sql
-- On primary: per-replica state
SELECT client_addr, state, replay_lsn FROM pg_stat_replication;

-- On replica: lag as time. Gotcha: with zero write traffic this grows
-- unbounded — alert on lag only while the primary is receiving writes
SELECT NOW() - pg_last_xact_replay_timestamp() AS replication_lag;

-- Replication slots retain WAL until consumed; an inactive slot fills the disk
SELECT slot_name, active, pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;
```

### MySQL

```sql
SHOW REPLICA STATUS\G     -- Seconds_Behind_Source, Last_Error, and the two thread states
```

MySQL's `Seconds_Behind_Source` measures the applier's position, not real staleness: it reads 0 while the I/O thread is stalled and the replica is minutes behind. Use a heartbeat table written on the primary and compared on the replica for a trustworthy number. Row-based binlog format is the correct default; statement-based replicates non-deterministic functions incorrectly.

Two facts to design around on any engine:

- Replication is async by default: an acknowledged commit can be lost on failover, and read-after-write against a replica can miss the write. Route read-your-own-writes traffic to the primary (or use synchronous replication and pay the latency).
- Long replica queries conflict with replay and get canceled. PostgreSQL's `hot_standby_feedback = on` stops the cancellations but lets replica queries hold back vacuum on the primary — pick per workload: analytics replica → on, and accept the primary bloat risk.

Replication is availability, not backup: a `DROP TABLE` replicates in milliseconds. You still need backups (→ Backup & Restore). Read routing and the point where replicas stop helping are topology questions — route from SKILL.md Quick Reference.

## High Availability and Failover

- Automatic failover needs three things: a health check that cannot be fooled by a slow query, a fencing mechanism that stops the old primary from accepting writes, and a way for applications to find the new primary.
- **Split brain** is the failure that loses data: two nodes both believing they are primary, both accepting writes. Any HA setup without fencing (STONITH, a lease, or a quorum) will eventually produce it.
- Application-side discovery: a virtual IP, a DNS record with a short TTL, a proxy (HAProxy, pgpool), or the driver's own multi-host connection string with `target_session_attrs=read-write`. Pick one and test that connections actually move.
- Managed services handle this for you; the useful question to ask them is the measured failover time and whether it is synchronous.
- Practice a failover on purpose, during business hours, before the first unplanned one. An untested failover is a rumor.

## Major Version Upgrades

1. Read the release notes for **breaking** changes, not features: removed functions, default changes (`sql_mode`, `ONLY_FULL_GROUP_BY`), collation changes, and planner behavior changes.
2. Restore a copy of production at the current version, upgrade the copy, and run the application's test suite plus the top queries from the ranked list against it. Compare plans, not only results.
3. Collation changes are the one nothing warns about: PostgreSQL text indexes built under a different glibc/ICU collation version can produce wrong results after an OS upgrade. Reindex text indexes when the collation version changes.
4. Plan the rollback before starting. In-place upgrades are usually one-way; a logical-replication upgrade (replicate old → new, then switch) allows cutting back.
5. Upgrade extensions and the client driver too, and re-run `ANALYZE` on the upgraded database before serving traffic — statistics are not always carried over.

## Capacity Planning

- Track four series over time: database size, largest table sizes, peak connections, and peak query rate. The trend tells you when, the absolute value tells you what.
- Project disk from growth rate plus retention policy, and add headroom for the maintenance operations that need it: a table rebuild needs free space equal to the table plus its indexes.
- The cliff to watch is the working set exceeding RAM; the cache hit ratio drops before latency does, which makes it a leading indicator.
- Growth is rarely linear: model against the business driver (tenants, orders per day), not against last month's bytes.
- Retention is a capacity decision. Deciding to keep events for 90 days instead of forever is cheaper than every other option on this page.
