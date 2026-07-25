# Query Patterns

Contents: Pagination · Deduplication · Top-N Per Group · Gaps and Islands · Conditional Aggregation · Missing Values · Window Functions · Pivot/Unpivot · Hierarchies · Graph Traversal · Temporal · Set Operations · Anti-Joins and Semi-Joins · Optional Filters · String Aggregation · Diffing Two Tables · Sampling · Locking and Queues · Bulk Operations

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

Keyset can't show "page 7 of 93". If the UI demands numbered pages, keep OFFSET but cap depth. For a stable cursor across concurrent inserts, encode the sort values (not the page number) in an opaque cursor token so the client cannot craft one.

## Deduplication

```sql
-- Keep one row per key (PostgreSQL): DISTINCT ON columns must be the
-- leading ORDER BY columns; the rest of ORDER BY picks WHICH row survives
SELECT DISTINCT ON (user_id) *
FROM orders ORDER BY user_id, created_at DESC;   -- latest order per user

-- Portable equivalent
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) t WHERE rn = 1;

-- Find duplicates first — always run this before any DELETE
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;

-- Delete duplicates, keep highest id (PostgreSQL USING; adapt elsewhere)
DELETE FROM users a USING users b
WHERE a.id < b.id AND a.email = b.email;

-- Portable delete via window function
DELETE FROM users WHERE id IN (
    SELECT id FROM (
        SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id DESC) AS rn FROM users
    ) t WHERE rn > 1
);
```

After deduplicating, add the unique constraint in the same migration — otherwise duplicates return. On a large table, delete in chunks (→ Bulk Operations) rather than in one statement.

## Top-N Per Group

```sql
-- N rows per group, portable
SELECT * FROM (
    SELECT o.*, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total DESC) AS rn
    FROM orders o
) t WHERE rn <= 3;

-- LATERAL: stops after N rows per group, so it beats the window form
-- when there are many groups and an index on (user_id, total DESC)
SELECT u.id, o.*
FROM users u
CROSS JOIN LATERAL (
    SELECT * FROM orders WHERE user_id = u.id ORDER BY total DESC LIMIT 3
) o;
```

`ROW_NUMBER` sorts every row in every partition; `LATERAL` (or `CROSS APPLY` in SQL Server) with a matching index reads only 3 rows per group. Use `LEFT JOIN LATERAL ... ON true` when groups with no rows must still appear.

`RANK` vs `DENSE_RANK` vs `ROW_NUMBER`: `ROW_NUMBER` always yields exactly N rows and breaks ties arbitrarily; `RANK` returns all tied rows (so "top 3" may return 5) and skips the following numbers; `DENSE_RANK` returns all ties without skipping. Choose by what a tie should mean, and add a tiebreaker column when it should not happen at all.

## Gaps and Islands

Consecutive runs — streaks of active days, contiguous id ranges, uninterrupted sessions. The trick is that `value − ROW_NUMBER()` is constant within a run.

```sql
-- Longest streak of consecutive active days per user
WITH marked AS (
    SELECT user_id, activity_date,
           activity_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY activity_date))::int
               AS grp
    FROM daily_activity
)
SELECT user_id, MIN(activity_date) AS streak_start, MAX(activity_date) AS streak_end,
       COUNT(*) AS streak_days
FROM marked GROUP BY user_id, grp
ORDER BY streak_days DESC;
```

For runs defined by a changing status rather than a date sequence, mark the boundary with `LAG` and take a running `SUM` of the boundary flag as the group id (the same shape as sessionization).

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

`COUNT(CASE WHEN cond THEN 1 END)` counts matches because `COUNT` skips NULLs; `COUNT(CASE WHEN cond THEN 1 ELSE 0 END)` counts every row and is the standard off-by-everything bug (SKILL.md rule 6).

## Missing Values (Gap Analysis)

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

Gaps in generated id sequences are normal — rollbacks consume values. Investigate only when a gap means lost data, never to "fix" the numbering. For missing dates in a report, the same shape is the date spine.

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

-- Percentage of total, and of the partition
SELECT category, amount,
       ROUND(amount * 100.0 / SUM(amount) OVER (), 2) AS pct_of_all,
       ROUND(amount * 100.0 / SUM(amount) OVER (PARTITION BY region), 2) AS pct_of_region
FROM category_totals;

-- First and last value in a partition; the frame matters for LAST_VALUE
SELECT user_id, event,
       FIRST_VALUE(event) OVER w AS first_event,
       LAST_VALUE(event)  OVER w AS last_event
FROM events
WINDOW w AS (PARTITION BY user_id ORDER BY occurred_at
             ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING);
```

- Frame trap: the default frame for `SUM() OVER (ORDER BY date)` is `RANGE`, which treats tied dates as peers — all rows with the same date get the same "running" total. Use `ROWS UNBOUNDED PRECEDING` (or a unique ORDER BY) for a true row-by-row total.
- `LAST_VALUE` with the default frame returns the current row, because the frame ends at the current row. It needs the explicit `UNBOUNDED FOLLOWING` above — the most common window-function bug after the RANGE default.
- `WHERE` cannot reference a window function (it runs earlier) — wrap in a subquery/CTE to filter on one. `QUALIFY` exists in some warehouses but not in these four engines.
- Naming the window with a `WINDOW` clause avoids repeating the specification and guarantees the frames match.

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

Unknown category set → don't pivot in SQL; return `(month, category, total)` rows and pivot in the application. SQL cannot produce a dynamic column list without building the statement as text, which is an injection surface.

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

-- Portable: UNION ALL, one branch per column
SELECT id, 'metric_a' AS key, metric_a AS value FROM metrics
UNION ALL SELECT id, 'metric_b', metric_b FROM metrics;

-- SQL Server
SELECT id, metric_name, metric_value
FROM metrics
UNPIVOT (metric_value FOR metric_name IN (metric_a, metric_b, metric_c)) AS u;
```

All unpivoted values must share one type; mixing a numeric and a text column forces a cast to text and loses ordering.

## Hierarchical Data

Adjacency list is the default: simplest writes, and recursive CTEs make reads workable everywhere (MySQL >=8.0, SQLite >=3.8.3, PostgreSQL, SQL Server).

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

Cycle protection: the depth guard above works everywhere; PostgreSQL >=14 has a native `CYCLE id SET is_cycle USING path` clause. `UNION` (not `UNION ALL`) also stops exact-duplicate rows but hides the cycle instead of surfacing it — and it forces a deduplication of the whole intermediate result on every iteration.

```sql
-- Materialized path: fast subtree reads, but every move rewrites descendants
CREATE TABLE categories (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    path TEXT NOT NULL            -- '1/3/7/15'
);
SELECT * FROM categories WHERE path LIKE '1/3/%';   -- all descendants of 3
```

Choose by read/write ratio: adjacency list when the tree changes often, materialized path when subtree reads dominate and moves are rare, closure table (one row per ancestor-descendant pair) when both are hot and you can afford the write amplification.

## Graph Traversal

The same recursive CTE handles arbitrary graphs, but a graph needs visited-set protection rather than a depth guard, because cycles are normal rather than corruption.

```sql
-- Shortest-hop reachability from node 1, tracking the path to avoid revisits
WITH RECURSIVE reach AS (
    SELECT to_id, 1 AS hops, ARRAY[from_id, to_id] AS path
    FROM edges WHERE from_id = 1
    UNION ALL
    SELECT e.to_id, r.hops + 1, r.path || e.to_id
    FROM edges e JOIN reach r ON e.from_id = r.to_id
    WHERE NOT e.to_id = ANY(r.path) AND r.hops < 6
)
SELECT to_id, MIN(hops) FROM reach GROUP BY to_id;
```

Both guards are required: the path check stops cycles, the hop limit stops combinatorial explosion in dense graphs. A relational database handles a few hops well; queries needing many hops or weighted shortest paths belong in a graph database (`neo4j`).

## Temporal Queries

```sql
-- Overlapping ranges (PostgreSQL): && handles all four overlap cases;
-- hand-rolled start/end comparisons routinely miss one
SELECT * FROM bookings
WHERE tstzrange(start_at, end_at, '[)') && tstzrange('2026-01-01', '2026-02-01', '[)');

-- Portable overlap test between half-open ranges
-- WHERE a_start < b_end AND a_end > b_start
```

Exclusion constraint replaces the check-then-insert race for bookings: `EXCLUDE USING gist (room_id WITH =, tstzrange(start_at, end_at, '[)') WITH &&)`.

## Set Operations

```sql
SELECT id FROM a UNION     SELECT id FROM b;   -- distinct: sorts/hashes the whole result
SELECT id FROM a UNION ALL SELECT id FROM b;   -- no deduplication, much cheaper
SELECT id FROM a INTERSECT SELECT id FROM b;   -- in both
SELECT id FROM a EXCEPT    SELECT id FROM b;   -- in a, not in b (MINUS in Oracle)
```

- Default to `UNION ALL` and add `UNION` only when duplicates are possible and unwanted.
- `ORDER BY` applies to the whole result and must come last; ordering a branch requires wrapping it in a subquery.
- Branches match by position, not by column name — a reordered column list pairs the wrong columns with no error if the types happen to be compatible.
- MySQL gained `INTERSECT`/`EXCEPT` in 8.0.31; SQLite has both; before that, emulate with `EXISTS`/`NOT EXISTS`.

## Anti-Joins and Semi-Joins

```sql
-- Semi-join: rows in a that have a match (stops at the first match)
SELECT * FROM users u WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- Anti-join: rows in a with no match. NULL-safe, unlike NOT IN
SELECT * FROM users u WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- Equivalent anti-join via LEFT JOIN; same plan in most engines
SELECT u.* FROM users u LEFT JOIN orders o ON o.user_id = u.id WHERE o.id IS NULL;
```

Prefer `EXISTS` over `IN (subquery)` for large subqueries and always over `NOT IN` when the subquery column is nullable (SKILL.md rule 6). Do not `SELECT *` inside an `EXISTS` — `SELECT 1` states the intent and avoids expanding the row.

## Optional Filters (Dynamic Search)

The convenient form disables indexes:

```sql
-- Convenient, and slow: the planner cannot know which branch applies
WHERE (:status IS NULL OR status = :status)
  AND (:min_total IS NULL OR total >= :min_total);
```

With literals unknown at plan time, the engine plans for the general case and typically scans. Three ways out, in order of preference:

1. **Build the statement from the filters that are actually present**, appending only the supplied predicates and binding their values. This is safe as long as the predicate fragments are hardcoded and only values are bound.
2. Force a fresh plan per execution (`OPTION (RECOMPILE)` in SQL Server, custom plan mode in PostgreSQL).
3. Write one query per common filter combination when there are only two or three.

## String Aggregation

```sql
SELECT user_id, STRING_AGG(tag, ',' ORDER BY tag)  FROM user_tags GROUP BY user_id;  -- PostgreSQL, SQL Server 2017+
SELECT user_id, GROUP_CONCAT(tag ORDER BY tag)     FROM user_tags GROUP BY user_id;  -- MySQL, SQLite
SELECT user_id, JSON_AGG(tag ORDER BY tag)         FROM user_tags GROUP BY user_id;  -- structured, PostgreSQL
```

MySQL's `GROUP_CONCAT` truncates without warning at `group_concat_max_len` (1024 bytes by default) — a report that looks fine in testing loses data in production. Prefer returning rows and joining in the application, or aggregate to JSON where the engine supports it.

## Diffing Two Tables

```sql
-- Full row-level diff: rows only in one side, tagged
SELECT 'only_in_new' AS side, * FROM (SELECT * FROM new_data EXCEPT SELECT * FROM old_data) a
UNION ALL
SELECT 'only_in_old', * FROM (SELECT * FROM old_data EXCEPT SELECT * FROM new_data) b;

-- Changed rows by key, with the differing column
SELECT n.id, o.status AS old_status, n.status AS new_status
FROM new_data n JOIN old_data o USING (id)
WHERE n.status IS DISTINCT FROM o.status;
```

`IS DISTINCT FROM` is the NULL-safe comparison: `o.status <> n.status` is NULL (not true) when either side is NULL, so rows where a value became NULL are omitted with no error. MySQL spells it `<=>` for the equality form. This is the standard verification step after any data migration.

## Sampling

```sql
-- PostgreSQL block sampling: fast; SYSTEM picks whole pages (clustered bias —
-- rows inserted together are sampled together), BERNOULLI picks rows uniformly but scans more
SELECT * FROM large_table TABLESAMPLE SYSTEM (1);      -- ~1% of pages
SELECT * FROM large_table TABLESAMPLE BERNOULLI (1);   -- ~1% of rows, unbiased

-- One pseudo-random row from a large table without a full sort:
-- probe by key around a random point, then fall back if the probe misses
SELECT * FROM large_table WHERE id >= (random() * (SELECT MAX(id) FROM large_table))::bigint
ORDER BY id LIMIT 1;

-- ORDER BY random()/RAND() LIMIT n = full scan + sort; only for small tables
```

The key-probe form is biased when ids have gaps (rows after a large gap are picked more often); accept it for "show me a random example" and never for statistics.

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

Deadlock prevention: when a transaction locks multiple rows, lock them in a consistent order (`ORDER BY id FOR UPDATE`); two transactions locking {1,2} and {2,1} deadlock, {1,2} and {1,2} queue. Isolation levels, retry loops, and the queue table's own schema route from SKILL.md Quick Reference.

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

-- Update many rows to different values in one statement
UPDATE products p SET price = v.price
FROM (VALUES (1, 9.99), (2, 14.50)) AS v(id, price)
WHERE p.id = v.id;
```

Large DELETE/UPDATE: one statement holds locks for the whole run and bloats WAL/undo. Chunk it — delete with `LIMIT` (or a keyed range) in batches of `batch_size` rows (default 5,000; the useful band is 1k-10k), commit between batches, loop until 0 rows affected:

```sql
DELETE FROM events WHERE id IN (
    SELECT id FROM events WHERE created_at < NOW() - INTERVAL '90 days' LIMIT 5000
);
-- repeat until DELETE 0
```

Deleting most of a very large table is faster as a rebuild: create a new table with the rows you keep, swap names, drop the old one. If the deletion is periodic and by time, partitioning turns it into an instant `DROP`.
