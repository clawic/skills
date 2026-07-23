---
name: sql
slug: sql
version: 1.0.3
description: >-
  Designs schemas, writes queries, tunes performance, and runs migrations on PostgreSQL, MySQL, SQLite, SQL Server.
  Use when writing SQL, fixing slow queries, designing tables and indexes, or operating a database.
homepage: https://clawic.com/skills/sql
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
metadata:
  clawdbot:
    emoji: 🗄️
    requires:
      anyBins:
      - sqlite3
      - psql
      - mysql
      - sqlcmd
    os:
    - linux
    - darwin
    - win32
    displayName: SQL
---

# SQL

Relational databases across SQLite, PostgreSQL, MySQL, and SQL Server: schema design, query patterns, performance, migrations, operations.

## When To Use

- Writing or reviewing SQL, or debugging why a query is slow
- Designing tables, indexes, and constraints for a new feature
- Planning schema migrations that must not take production down
- Operating a database: backups, monitoring, pooling, replication
- Not for ORM-level modeling — jump to `prisma` for Node.js ORM workflows

## Quick Reference

| Situation | Play |
|-----------|------|
| Query slow | `EXPLAIN (ANALYZE, BUFFERS)`, fix the worst node first (→ EXPLAIN) |
| Paginating past the first few thousand rows | Keyset, never OFFSET (`patterns.md`) |
| Read-modify-write race | `SELECT ... FOR UPDATE`; job queues add `SKIP LOCKED` (`patterns.md`) |
| Totals inflated after adding a JOIN | 1:N fan-out — aggregate before joining (→ Traps) |
| Schema change on a live table | Expand → migrate → contract, `lock_timeout` first (`operations.md`) |
| Tenant isolation | Shared tables + `tenant_id`, RLS to enforce (`schemas.md`) |
| Choosing an engine | SQLite embedded/local · PostgreSQL default for servers · MySQL when the platform dictates it · SQL Server in .NET/Windows shops |
| Anything else | Schema-shaped → `schemas.md` · query-shaped → `patterns.md` · ops-shaped → `operations.md` |

## Core Rules

1. **Parameterize values; allowlist identifiers.** Placeholders (`?`, `$1`) stop injection for values, but table/column names cannot be bound — when those are dynamic, check them against a hardcoded allowlist, never interpolate user input.
2. **BIGINT (or UUIDv7) primary keys by default.** `INT` overflows at 2,147,483,647 — at a sustained 100 inserts/s that is ~8 months (2.1B ÷ 100/s ≈ 248 days), and the fix is an outage-grade type change. Random UUIDv4 keys fragment the B-tree; UUIDv7/ULID keep insert locality.
3. **Index for the query shape: equality columns first, then range/sort.** `(user_id, created_at)` serves `WHERE user_id = ? AND created_at > ?` and `WHERE user_id = ?` alone — never `created_at` alone. A seq scan on a filter matching more than roughly 5-10% of rows is the planner being right, not broken.
4. **Transactions stay short and never wait on the outside world.** No HTTP calls, no user input inside `BEGIN...COMMIT`: open transactions hold locks, and in PostgreSQL they also block vacuum, causing table bloat. Anything open past the >1 min monitoring threshold (`operations.md`) gets investigated.
5. **NULL is three-valued.** `NOT IN (subquery)` returns zero rows if the subquery yields a single NULL — use `NOT EXISTS`. `x = NULL` is never true — use `IS NULL`. `COUNT(col)` skips NULLs; `COUNT(*)` counts rows.
6. **Types that avoid the next migration.** Money → `NUMERIC` (float money loses cents in aggregation); timestamps → `TIMESTAMPTZ` stored as UTC; strings → `TEXT` in PostgreSQL (`varchar(255)` is a cargo-cult limit you will later raise); MySQL charset → `utf8mb4` (MySQL's `utf8` is 3-byte and rejects emoji).
7. **Migrations are additive first.** Rename/retype/drop happens over multiple deploys with both versions live in between (expand-migrate-contract, `operations.md`). A single-deploy column rename breaks every instance still running old code.
8. **Rank before you tune.** `pg_stat_statements` ordered by `total_exec_time` (or MySQL slow query log) tells you which query costs the most overall — usually not the one someone complained about. Optimizing an unranked query is guessing.

## Quick Start

```bash
# SQLite
sqlite3 mydb.sqlite                              # open (creates if missing)
sqlite3 -header -csv mydb.sqlite "SELECT ..." > out.csv
sqlite3 mydb.sqlite "PRAGMA journal_mode=WAL;"   # readers no longer block the writer

# PostgreSQL
psql -h localhost -U myuser -d mydb              # connect; \dt tables, \d+ t schema, \di+ indexes
psql -f migration.sql mydb

# MySQL
mysql -h localhost -u root -p mydb
mysql -e "SELECT NOW();" mydb

# SQL Server
sqlcmd -S localhost -U myuser -d mydb            # -E for Windows auth
sqlcmd -Q "SELECT GETDATE()"
```

## EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 5;  -- PostgreSQL
EXPLAIN QUERY PLAN SELECT * FROM orders WHERE user_id = 5;          -- SQLite
EXPLAIN ANALYZE SELECT ...;                                         -- MySQL >=8.0.18
```

Read actual behavior, not just the plan:

- `Seq Scan` on a large table with a selective filter → missing or unusable index (→ Traps for what disables one)
- `Rows Removed by Filter` high → the index found candidates but the filter did the work; extend the index to cover the filter
- Estimated vs actual rows off by >10x → stale stats: run `ANALYZE tablename;`; if still off, the planner assumes column independence — `CREATE STATISTICS` on the correlated columns (PostgreSQL >=10)
- `Buffers: read` large vs `hit` → data is coming from disk; recheck after warm cache before concluding
- Nested Loop over thousands of outer rows → often the >10x misestimate above feeding a bad join choice

## Index Strategy

```sql
-- Composite: equality columns first, range/sort last (rule 3)
CREATE INDEX idx_orders ON orders(user_id, status);

-- Covering: index-only scan, no heap fetch (PostgreSQL >=11 INCLUDE)
CREATE INDEX idx_orders ON orders(user_id) INCLUDE (total);

-- Partial: index only the rows you query
CREATE INDEX idx_pending ON orders(user_id) WHERE status = 'pending';

-- Expression: make a function sargable
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

- A plain B-tree on a low-cardinality column (`status` with 5 values) rarely helps; a partial index on the rare value you actually query does.
- PostgreSQL with a non-C locale ignores B-tree indexes for `LIKE 'term%'` — add an index with `text_pattern_ops` opclass for prefix search.
- Index-only scans still hit the heap for pages not marked all-visible; if `EXPLAIN` shows `Heap Fetches` high, the table needs a `VACUUM`.
- Every index taxes writes and doubles as disk: drop unused ones (`pg_stat_user_indexes` where `idx_scan = 0`, after enough uptime to trust it).

## Portability

| Feature | PostgreSQL | MySQL | SQLite | SQL Server |
|---------|------------|-------|--------|------------|
| Limit | LIMIT n | LIMIT n | LIMIT n | TOP n / OFFSET-FETCH |
| Upsert | ON CONFLICT | ON DUPLICATE KEY | ON CONFLICT | MERGE |
| Boolean | true/false | 1/0 | 1/0 | 1/0 |
| Concat | \|\| | CONCAT() | \|\| | + or CONCAT() |
| Auto-id | GENERATED / SERIAL | AUTO_INCREMENT | INTEGER PRIMARY KEY | IDENTITY |
| Returning rows from DML | RETURNING | — (MariaDB has it) | RETURNING (>=3.35) | OUTPUT |
| Aggregate FILTER | Yes | CASE only | Yes (>=3.30) | CASE only |

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| `WHERE YEAR(created_at) = 2024` | Function on column hides it from the index | Range: `created_at >= '2024-01-01' AND created_at < '2025-01-01'` |
| `WHERE varchar_col = 123` | Implicit cast applies to the column, disabling its index | Match types: quote the literal or fix the schema |
| `LEFT JOIN t ... WHERE t.col = x` | WHERE runs after the join, filters the NULL rows out → silent INNER JOIN | Move the predicate into `ON`, or test `t.id IS NULL` for anti-join |
| Join 1:N then `SUM`/`COUNT` | Fan-out duplicates left-side rows before aggregation | Aggregate in a subquery/CTE, then join the result |
| `LIKE '%term'` | Leading wildcard defeats B-tree ordering | Full-text search (`schemas.md`) or trigram index (`pg_trgm`) |
| Composite `(a, b)` for `WHERE b = ?` | B-tree is sorted by `a` first; `b` alone is unordered | Separate index on `b`, or reorder if `a` is always filtered |
| `ORDER BY random() LIMIT n` | Full scan + sort of the entire table | `TABLESAMPLE` or random-key probe (`patterns.md`) |
| `SELECT *` in application code | Blocks index-only scans, fetches columns you drop, breaks on schema change | Name the columns |
| DDL without `lock_timeout` | `ALTER TABLE` queues behind one long query and every new query queues behind it — brief lock, full outage | `SET lock_timeout = '2s'` then retry (`operations.md`) |

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/sql (install if the user confirms):
- `prisma` — Node.js ORM
- `sqlite` — SQLite-specific patterns
- `analytics` — data analysis queries

## Feedback

- If useful, star it: https://clawic.com/skills/sql
- Latest version: https://clawic.com/skills/sql

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/sql.
