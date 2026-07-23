---
name: pg
slug: pg
version: 1.0.2
description: >-
  Tunes and designs PostgreSQL: indexing, EXPLAIN-driven query optimization, schema types, safe migrations, vacuum, connection pooling.
  Use when writing Postgres SQL, designing schemas, running production DDL, or debugging slow queries.
homepage: https://clawic.com/skills/pg
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
metadata:
  clawdbot:
    emoji: 🐘
    requires:
      anyBins:
      - psql
      - pgcli
    os:
    - linux
    - darwin
    - win32
    displayName: PostgreSQL
---

## When To Use

- A Postgres query is slow, or an EXPLAIN plan needs interpretation
- Designing a schema: types, constraints, indexes, keys
- Running DDL or a migration against a live production table
- Operating Postgres: connections, timeouts, vacuum, bloat, locks
- Building queue/upsert/search patterns that Postgres solves natively
- Not for cross-engine SQL portability or ORM-level modeling (see Related Skills)

## Quick Reference

| Situation | Play |
|---|---|
| Slow query, cause unknown | Slow-Query Triage below, in order — never tune from query text alone |
| WHERE on a function of a column | Expression index matching the query text exactly (→ Indexing) |
| Status/flag column, minority rows queried | Partial index `WHERE status = 'active'` |
| Substring/fuzzy search (`LIKE '%x%'`, ILIKE) | pg_trgm GIN index; B-tree cannot serve a leading wildcard |
| Word/phrase search | Stored generated tsvector column + GIN; query via `websearch_to_tsquery` |
| Job queue in Postgres | `SELECT ... FOR UPDATE SKIP LOCKED` — no external broker needed |
| Insert-or-update | `INSERT ... ON CONFLICT ... DO UPDATE` + `RETURNING` |
| First/top-N rows per group | `DISTINCT ON` for first-row; `LATERAL` + matching index for top-N at scale |
| Deep pagination | Keyset (`WHERE (created_at, id) < (?, ?)`), never large OFFSET |
| DDL on a busy table | `SET lock_timeout` first, then the online variant (→ Safe DDL) |
| Table bloated / stats stale | Vacuum & Long Transactions section |
| Anything else surprising | `EXPLAIN (ANALYZE, BUFFERS)` before and after every change |

## Slow-Query Triage

Run in this order; skipping to step 4 wastes the fix on the wrong query.

1. `pg_stat_statements` ordered by `total_exec_time DESC` (needs `shared_preload_libraries`). High `mean_exec_time` = slow query; high `calls` = hot path — a 5ms query called 10k/min beats a 2s report as target.
2. `EXPLAIN (ANALYZE, BUFFERS)` on the worst offender. Plain EXPLAIN shows estimates only — estimates are the thing that lies.
3. Compare estimated vs actual rows per node. Off by >10x → stale or insufficient stats: run `ANALYZE`; still off → `ALTER TABLE ... ALTER COLUMN ... SET STATISTICS 1000` (default 100), or `CREATE STATISTICS` for correlated columns (city+country style).
4. Read Buffers: `read` dominant → I/O-bound (missing index, cold cache); `hit` dominant → plan/CPU-bound (wrong join order, overwide scan).
5. Seq scan is not automatically the bug: it beats an index once a query touches roughly 5-15% of the table (correlation-dependent). On SSD, set `random_page_cost = 1.1` (default 4.0 assumes spinning disk) or the planner avoids good indexes.
6. After the fix, re-run step 2 and compare buffers and actual time — a prettier plan shape with the same buffer count fixed nothing.

## Indexing

- Foreign key columns are NOT auto-indexed. Every JOIN on the FK and every parent `DELETE`/`ON DELETE CASCADE` scans the child table without one.
- Composite order: equality columns first, then the range/sort column. `(a, b)` serves `WHERE a = ?` and `WHERE a = ? AND b > ?`, never `WHERE b = ?` alone.
- Partial index size is proportional to matching rows: `WHERE active` over a 5%-active table is ~95% smaller and stays hot in cache.
- Expression index must match query text exactly: `ON lower(email)` serves `WHERE lower(email) = ?`, not `WHERE email ILIKE ?`.
- Covering index `INCLUDE (name)` enables index-only scans — verify "Heap Fetches" near 0 in EXPLAIN; a stale visibility map (vacuum lag) silently degrades them back to heap fetches.
- Updating any indexed column defeats HOT updates and writes to every index on the table. Keep frequently-updated columns (counters, timestamps) out of indexes; set `fillfactor = 90` on update-heavy tables so HOT has page room.
- Drop unused: `pg_stat_user_indexes` where `idx_scan = 0` — but check across a full business cycle (month-end reports) and on every replica; stats are per-node.
- Don't index a low-cardinality column alone (boolean, small enum): planner ignores it. Combine into a composite or partial index instead.

## Query Patterns

- `FOR UPDATE SKIP LOCKED`: concurrent workers each grab unclaimed rows; the canonical Postgres job queue.
- `pg_advisory_lock(key)`: application mutex with no table; pair with explicit unlock — session locks survive commit.
- `IS NOT DISTINCT FROM`: NULL-safe equality, replaces `(a = b OR (a IS NULL AND b IS NULL))`.
- `count(*) > 0` for existence scans every match; `EXISTS (SELECT 1 ...)` stops at the first.
- `x NOT IN (subquery)` returns zero rows if the subquery yields any NULL — use `NOT EXISTS`.
- Aggregates with `FILTER (WHERE ...)` replace CASE-inside-SUM pivots, readably.
- `now()` is frozen at transaction start; rows stamped in one long transaction share it. `clock_timestamp()` gives wall time.
- CTEs: PostgreSQL >=12 inlines them like subqueries; on older versions every CTE is an optimization fence that blocks index pushdown. `MATERIALIZED` keyword restores the fence deliberately.
- Full-text: precompute tsvector as a stored generated column with GIN — computing it per-query rescans every row. Feed user input through `websearch_to_tsquery` (PostgreSQL >=11: handles quotes, `-`, OR); raw `to_tsquery` throws syntax errors on user text. `'english'` config stems words, `'simple'` matches exact tokens. FTS is word-based — substring matching still needs pg_trgm.

## Data Types & Constraints

- `GENERATED ALWAYS AS IDENTITY` over `SERIAL` (PostgreSQL >=10): SERIAL leaves sequence ownership and permission quirks.
- `TIMESTAMPTZ` by default — it stores the UTC instant, not a zone. If original wall-clock zone matters (calendar events), store the zone name in a second column.
- Money: `NUMERIC(12,2)` or integer cents. Never float (0.1 + 0.2 ≠ 0.3), never the `money` type (locale-dependent formatting).
- `TEXT` over `VARCHAR(n)`: identical performance in Postgres; add length as a CHECK only when it is a real business rule (easier to alter later than a retype).
- JSONB for genuinely variable attributes only; any key you regularly WHERE/JOIN on belongs in a real column — jsonb keys have no per-key statistics, so the planner guesses. Index containment queries with GIN `jsonb_path_ops` (smaller and faster than the default opclass; supports only `@>`).
- Soft delete + uniqueness: partial unique index `UNIQUE ... WHERE deleted_at IS NULL` — a plain unique constraint blocks re-creating a deleted record.
- Unique over nullable columns treats NULLs as distinct; `UNIQUE NULLS NOT DISTINCT` (PostgreSQL >=15) closes that hole.

## Safe DDL on Busy Tables

Every step here starts with `SET lock_timeout = '2s'` (and retry logic): a blocked `ALTER TABLE` waiting on ACCESS EXCLUSIVE queues ALL new queries behind it — the outage comes from the wait, not the lock.

- `ADD COLUMN ... DEFAULT <constant>` is metadata-only (PostgreSQL >=11); a volatile default (`now()`, `gen_random_uuid()`) rewrites the whole table. Add nullable, backfill in batches, then set the default.
- New constraint on a big table: `ADD CONSTRAINT ... NOT VALID` (instant), then `VALIDATE CONSTRAINT` (weak lock, full scan without blocking writes).
- `SET NOT NULL`: PostgreSQL >=12 skips the full-table scan if a validated `CHECK (col IS NOT NULL)` already proves it — add the check NOT VALID, validate, then SET NOT NULL, then drop the check.
- `CREATE INDEX CONCURRENTLY`: no write lock, but cannot run inside a transaction, and a failure leaves an INVALID index behind — check `pg_index.indisvalid`, drop, retry. Same for `REINDEX CONCURRENTLY` (PostgreSQL >=12).
- Type changes: widening `varchar(n)` or `varchar → text` is metadata-only; `int → bigint` rewrites the table. For the latter on a hot table: new column, dual-write, backfill, swap.

## Connections & Memory

- Each connection is a full backend process (~5-10MB base, plus work memory). Pool before scaling `max_connections` (default 100): PgBouncer in transaction mode once you exceed roughly 50-100 real connections.
- Transaction pooling breaks session state: `SET`, advisory locks, LISTEN/NOTIFY, temp tables; protocol-level prepared statements need PgBouncer >=1.21.
- `work_mem` is per sort/hash NODE, not per query. Worst case = connections × nodes × work_mem: 100 conns × 4 sorts × 64MB = 25.6GB — that is the classic OOM. Raise it per-session for known big queries instead of globally.
- `shared_buffers` ~25% of RAM (official rule of thumb); Postgres also relies on the OS page cache, so more is not linearly better.
- Set per role, not globally forgotten: `statement_timeout = '30s'` (runaway queries) and `idle_in_transaction_session_timeout = '5min'` (abandoned transactions holding locks and pinning vacuum).

## Vacuum & Long Transactions

- Vacuum cannot remove tuples newer than the oldest open transaction or replication slot — one forgotten `BEGIN` (or dead slot) bloats every table in the database. Check `pg_stat_activity` for old `xact_start` and `pg_replication_slots` first when bloat grows.
- `autovacuum_vacuum_scale_factor` default 0.2 means a 100M-row table waits for 20M dead tuples before vacuuming. Per-table fix: `ALTER TABLE big SET (autovacuum_vacuum_scale_factor = 0.01)` → triggers at 1M.
- `VACUUM FULL` takes ACCESS EXCLUSIVE (full outage on that table); `pg_repack` rebuilds online. Plain VACUUM frees space for reuse but rarely shrinks the file.
- After bulk load or mass delete: `VACUUM ANALYZE` — the planner still holds pre-load statistics until then.
- Wraparound: xids are 32-bit; autovacuum freezes tuples from `autovacuum_freeze_max_age` (default 200M). Monitor `age(datfrozenxid)`; near ~2 billion the database force-stops. If autovacuum can't keep up on a huge table, schedule manual `VACUUM FREEZE` off-peak.
- Isolation: default READ COMMITTED takes a new snapshot per statement — multi-statement reports can see inconsistent totals; use REPEATABLE READ for them. SERIALIZABLE requires a retry loop on error 40001; without the loop it is strictly worse than REPEATABLE READ.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `NOT IN` with a nullable subquery | One NULL makes the whole predicate unknown → 0 rows, silently | `NOT EXISTS` |
| `OFFSET 100000` pagination | Postgres computes and discards all skipped rows; page 1000 costs 1000 pages | Keyset pagination on an indexed sort key |
| CTE performance assumed on old Postgres | <12 always materializes: no index pushdown into the CTE | Upgrade, inline as subquery, or accept the fence knowingly |
| `CREATE INDEX` (no CONCURRENTLY) on live table | SHARE lock blocks all writes for the whole build | CONCURRENTLY + indisvalid check |
| DDL without `lock_timeout` | The waiting ALTER queues every query behind it | `SET lock_timeout` + retry (→ Safe DDL) |
| Dropping an "unused" index seen on primary only | Index stats are per-node; replicas may serve reads from it | Check `pg_stat_user_indexes` on all nodes, full cycle |
| Trusting plain `EXPLAIN` | Costs are estimates; the misestimate IS the usual bug | `EXPLAIN (ANALYZE, BUFFERS)` |
| ENUM for fast-changing categories | Values can't be dropped or reordered without type surgery | Lookup table + FK when the set churns |

## Where Experts Disagree

- **bigint vs UUID primary keys.** Random UUIDv4 fragments the B-tree and bloats secondary indexes; bigint leaks row counts and needs coordination across writers. Boundary: external-facing or multi-writer IDs → UUID (v7/time-ordered if available); internal high-write tables → `bigint IDENTITY`.
- **Normalize vs denormalize.** Postgres joins are cheaper than most engineers assume; denormalize only after EXPLAIN shows the join as the measured bottleneck, and prefer a materialized view over duplicated columns (it has a single refresh point).
- **App-side pool vs PgBouncer.** One or two services → app pool suffices; many services or serverless (connection churn) → PgBouncer, accepting the session-state limits above.
- **Scale up vs scale out.** Read replicas add lag-handling complexity to the app; a bigger box is defensible far longer than fashionable. Reach for replicas when reads are heavy AND tolerant of staleness, not at the first slow query.

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):

- `sql` — cross-engine SQL and portability; jump there when the target isn't Postgres-specific
- `database-indexing` — deeper index theory: write-cost budgets, bitmap/hash structures
- `prisma` — ORM-level schema modeling and query pitfalls on top of Postgres
- `timescaledb` — time-series workloads on Postgres: hypertables, compression, continuous aggregates
- `db` — general database operations, reliability, and scaling patterns

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/pg.
