# Query Patterns — SQL

Contents: Pagination · Deduplication · Conditional Aggregation · Gaps · Window Functions · Pivot/Unpivot · Hierarchies · Temporal · Sampling · Locking & Queues · Bulk Operations

## Pagination

OFFSET reads and discards every skipped row: cost grows linearly with page number. Fine for the first few pages, wrong for "jump to page 500" or infinite scroll — switch to keyset.

```sql
-- Offset (acceptable only for shallow pages)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 100;

-- Keyset: constant cost at any depth. Requires a deterministic order —
-- always tie-break on a unique column (id), or rows with equal
-- created_at get skipped or duplicated across pages.
SELECT * FROM posts ORDER BY created_at DESC, id DESC LIMIT 20;

SELECT * FROM posts
WHERE (created_at, id) < ('2026-01-15 10:00:00', 12345)   -- last row of previous page
ORDER BY created_at DESC, id DESC LIMIT 20;
```

Row-value comparison `(a, b) < (x, y)` is index-friendly in PostgreSQL and SQLite. MySQL accepts the syntax but often won't use the index for it — expand to `a < x OR (a = x AND b < y)`.

Keyset can't show "page 7 of 93". If the UI demands numbered pages, keep OFFSET but cap depth.

## Deduplication

```sql
-- Keep one row per key (PostgreSQL): DISTINCT ON columns must be the
-- leading ORDER BY columns; the rest of ORDER BY picks WHICH row survives
SELECT DISTINCT ON (user_id) *
FROM orders ORDER BY user_id, created_at DESC;   -- latest order per user

-- Delete duplicates, keep highest id (PostgreSQL USING; adapt elsewhere)
DELETE FROM users a
USING users b
WHERE a.id < b.id AND a.email = b.email;

-- Find duplicates first — always run this before the DELETE
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;
```

After deduplicating, add the unique constraint in the same migration — otherwise duplicates return.

## Conditional Aggregation

One pass over the table beats N filtered queries.

```sql
-- FILTER: PostgreSQL, SQLite >=3.30
SELECT
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE status = 'paid') AS paid,
    SUM(total) FILTER (WHERE status = 'paid') AS revenue
FROM orders;

-- CASE: portable everywhere (MySQL, SQL Server)
SELECT
    COUNT(*) AS total,
    SUM(CASE WHEN status = 'paid' THEN 1 ELSE 0 END) AS paid,
    SUM(CASE WHEN status = 'paid' THEN total END) AS revenue
FROM orders;
```

## Gap Analysis (Missing Values)

```sql
-- PostgreSQL (generate_series is PostgreSQL-only)
WITH all_ids AS (
    SELECT generate_series(1, (SELECT MAX(id) FROM products)) AS id
)
SELECT a.id FROM all_ids a
LEFT JOIN products p ON a.id = p.id
WHERE p.id IS NULL;

-- Portable: recursive CTE as series generator (MySQL >=8.0, SQLite, SQL Server)
WITH RECURSIVE seq(n) AS (
    SELECT 1 UNION ALL SELECT n + 1 FROM seq WHERE n < 1000
)
SELECT n FROM seq LEFT JOIN products p ON p.id = seq.n WHERE p.id IS NULL;
```

Gaps in sequences are normal (rollbacks consume ids) — investigate only when a gap means lost data, not to "fix" the numbering.

## Window Functions

```sql
-- Running total
SELECT date, amount,
       SUM(amount) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING) AS running_total
FROM transactions;

-- Day-over-day change
SELECT date, amount,
       amount - LAG(amount) OVER (ORDER BY date) AS change
FROM daily_metrics;

-- Percentage of total
SELECT category, amount,
       ROUND(amount * 100.0 / SUM(amount) OVER (), 2) AS pct
FROM category_totals;
```

Frame trap: the default frame for `SUM() OVER (ORDER BY date)` is `RANGE`, which treats tied dates as peers — all rows with the same date get the same "running" total. Use `ROWS UNBOUNDED PRECEDING` (or a unique ORDER BY) for a true row-by-row total.

`WHERE` cannot reference a window function (it runs earlier) — wrap in a subquery/CTE to filter on one.

## Pivoting Data

Default to portable CASE; `crosstab` needs an extension, breaks when a category appears that isn't in the column list, and buys little.

```sql
SELECT month,
       SUM(CASE WHEN category = 'electronics' THEN total END) AS electronics,
       SUM(CASE WHEN category = 'clothing' THEN total END) AS clothing,
       SUM(CASE WHEN category = 'food' THEN total END) AS food
FROM sales
GROUP BY month;
```

Unknown category set → don't pivot in SQL; return `(month, category, total)` rows and pivot in the application.

## Unpivoting Data

```sql
-- PostgreSQL
SELECT id, key, value
FROM metrics,
LATERAL (VALUES
    ('metric_a', metric_a),
    ('metric_b', metric_b),
    ('metric_c', metric_c)
) AS t(key, value);

-- SQL Server
SELECT id, metric_name, metric_value
FROM metrics
UNPIVOT (metric_value FOR metric_name IN (metric_a, metric_b, metric_c)) AS u;
```

## Hierarchical Data

Adjacency list is the default: simplest writes, and recursive CTEs make reads workable everywhere (MySQL >=8.0, SQLite >=3.8.3, PostgreSQL, SQL Server). Switch to materialized path only when tree reads dominate and depth queries are hot.

```sql
CREATE TABLE categories (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    parent_id BIGINT REFERENCES categories(id)
);

-- Ancestors of node 5
WITH RECURSIVE ancestors AS (
    SELECT *, 1 AS depth FROM categories WHERE id = 5
    UNION ALL
    SELECT c.*, a.depth + 1 FROM categories c
    JOIN ancestors a ON c.id = a.parent_id
    WHERE a.depth < 50            -- guard: a cycle in the data loops forever without this
)
SELECT * FROM ancestors;
```

Cycle protection: the depth guard above works everywhere; PostgreSQL >=14 has a native `CYCLE id SET is_cycle USING path` clause. `UNION` (not `UNION ALL`) also stops exact-duplicate rows but hides the cycle instead of surfacing it.

```sql
-- Materialized path: fast subtree reads, but every move rewrites descendants
CREATE TABLE categories (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    path TEXT NOT NULL            -- '1/3/7/15'
);
SELECT * FROM categories WHERE path LIKE '1/3/%';   -- all descendants of 3
```

## Temporal Queries

```sql
-- Overlapping ranges (PostgreSQL): && handles all four overlap cases;
-- hand-rolled start/end comparisons routinely miss one
SELECT * FROM bookings
WHERE daterange(start_date, end_date, '[]') && daterange('2026-01-01', '2026-01-31', '[]');

-- Portable overlap test: (a.start <= b.end) AND (a.end >= b.start)

-- Fill missing dates so charts don't silently skip empty days
SELECT d.date, COALESCE(s.revenue, 0) AS revenue
FROM generate_series('2026-01-01'::date, '2026-01-31'::date, '1 day') AS d(date)
LEFT JOIN daily_sales s ON s.date = d.date;
```

Exclusion constraint replaces the check-then-insert race for bookings: `EXCLUDE USING gist (room_id WITH =, daterange(start_date, end_date) WITH &&)`.

## Sampling

```sql
-- PostgreSQL block sampling: fast; SYSTEM picks whole pages (clustered bias —
-- rows inserted together are sampled together), BERNOULLI picks rows uniformly but scans more
SELECT * FROM large_table TABLESAMPLE SYSTEM (1);      -- ~1% of pages
SELECT * FROM large_table TABLESAMPLE BERNOULLI (1);   -- ~1% of rows, unbiased

-- ORDER BY random()/RAND() LIMIT n = full scan + sort; only for small tables
```

## Locking & Queues

```sql
-- Read-modify-write without lost updates
SELECT * FROM inventory WHERE product_id = 5 FOR UPDATE;

-- Job queue: SKIP LOCKED lets N workers pull disjoint jobs with no coordinator
UPDATE jobs SET status = 'running', started_at = NOW()
WHERE id = (
    SELECT id FROM jobs WHERE status = 'pending'
    ORDER BY created_at LIMIT 1
    FOR UPDATE SKIP LOCKED
)
RETURNING *;
```

Deadlock prevention: when a transaction locks multiple rows, lock them in a consistent order (`ORDER BY id FOR UPDATE`); two transactions locking {1,2} and {2,1} deadlock, {1,2} and {1,2} queue.

## Bulk Operations

```sql
-- Multi-row insert. PostgreSQL wire protocol caps bind parameters at 65,535:
-- max rows per statement = 65535 / column_count (4 columns → 16k rows/batch)
INSERT INTO users (email, name) VALUES
    ('a@example.com', 'Alice'),
    ('b@example.com', 'Bob');

-- Bulk upsert (PostgreSQL/SQLite)
INSERT INTO users (email, name) VALUES ('a@example.com', 'Alice')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;

-- Big loads: COPY (psql \copy) / LOAD DATA INFILE — typically an order of
-- magnitude faster than INSERT batches; drop indexes first on empty-table loads
\copy users FROM 'users.csv' CSV HEADER
```

Large DELETE/UPDATE: one statement holds locks for the whole run and bloats WAL/undo. Chunk it — delete with `LIMIT` (or a keyed range) in batches of roughly 1k-10k rows, commit between batches, loop until 0 rows affected:

```sql
DELETE FROM events WHERE id IN (
    SELECT id FROM events WHERE created_at < NOW() - INTERVAL '90 days' LIMIT 5000
);
-- repeat until DELETE 0
```
