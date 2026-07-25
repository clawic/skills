# Performance — Plans, Indexes, and What To Change

Tuning order that avoids wasted work: rank queries by total cost (SKILL.md rule 9) → read the plan → fix the schema or the query → re-measure the same plan. Steps skipped in that order produce indexes nobody uses.

Contents: Plan Nodes · Join Algorithms · Statistics · Index Types · Composite Design · Sargability · Rewrites That Win · Aggregation · Sorting · Memory & Spills · Prepared Statements · Measuring · When Not To Tune

## Plan Nodes: What Each One Tells You

| Node | Means | Change it by |
|---|---|---|
| Seq Scan / table scan | Whole table read | Adding a usable index — or accepting it if selectivity is above roughly 5-10% (SKILL.md rule 3) |
| Index Scan | Index walked, heap fetched per row | Making it index-only with `INCLUDE`/covering columns |
| Index Only Scan | No heap access | Watch `Heap Fetches`; high means the visibility map is stale — vacuum |
| Bitmap Heap Scan | Many index matches, heap read in page order | Normal for medium selectivity; if `Rows Removed by Filter` is high, widen the index |
| Nested Loop | Inner side executed per outer row | Cheap when the outer side is tiny AND the inner is indexed; catastrophic when the outer estimate was wrong |
| Hash Join | Build a hash of the smaller side | Fine for large unsorted joins; watch for `Batches > 1` (spilled to disk) |
| Merge Join | Both sides sorted | Good when indexes already provide the order; expensive when it has to sort |
| Sort | Ordering rows | Provide the order via an index, or reduce rows first |
| Materialize / Memoize | Caching an inner result | Usually the planner compensating for repetition; not a bug |
| Aggregate vs HashAggregate | Grouping strategy | HashAggregate spilling means `work_mem`-class pressure (`pg` for tuning it) |

Read the plan bottom-up and inside-out: the deepest node runs first. In PostgreSQL, `actual time` on a node is **per loop** — multiply by `loops` for total. A node showing `actual time=0.05..0.08 rows=1 loops=200000` costs 16 seconds, not 0.08 ms; missing this is the single most common plan misreading.

## Join Algorithms: Choosing By Shape

- Nested loop wins when the outer row count is small (roughly hundreds) and the inner side has an index on the join key. It is also the failure mode of a bad estimate: the planner expected 10 outer rows, got 100,000, and now runs 100,000 index lookups.
- Hash join wins when one side fits in memory and neither is usefully sorted. Build side = the smaller estimated side; a wrong estimate hashes the wrong table.
- Merge join wins when both inputs arrive sorted from indexes; forcing it by adding an explicit sort rarely pays.
- Join order matters more than join algorithm on many-table queries. PostgreSQL exhaustively searches up to `join_collapse_limit` (default 8) tables, then stops optimizing and uses the written order — a 12-table report is partly hand-ordered whether you meant it or not.

## Statistics: Why The Planner Guessed Wrong

- Row estimates drive every decision. An estimate off by more than 10× invalidates the plan, not the index.
- After a bulk load, mass delete, or restore, statistics describe the previous shape. `ANALYZE` is the first move, before any index.
- Skewed columns: default histograms track a limited number of most-common values. A `status` column where 99% is `'done'` needs more granularity — PostgreSQL `ALTER TABLE ... ALTER COLUMN ... SET STATISTICS 1000` (default 100), MySQL `ANALYZE TABLE ... UPDATE HISTOGRAM ON col`.
- Correlated columns break the independence assumption: `WHERE city = 'Paris' AND country = 'France'` is estimated as `sel(city) × sel(country)`, thousands of times too low. Declare it: PostgreSQL `CREATE STATISTICS (dependencies) ON city, country FROM addresses` (>=10).
- Expression predicates have no statistics unless a matching expression index exists — creating the index improves the estimate as well as the access path.

## Index Types Beyond B-tree

| Type | Use for | Cost |
|---|---|---|
| B-tree | Equality, ranges, sorting, prefix `LIKE 'x%'` | The default; nothing else beats it for ordered data |
| Hash (PostgreSQL) | Equality only | No range, no sort; B-tree is nearly always fine instead |
| GIN | Array containment, JSONB `@>`, full-text | Slow to update; batch writes or accept write amplification |
| GiST | Ranges, geometry, exclusion constraints | Supports overlap operators B-tree cannot |
| BRIN | Huge tables physically ordered by the indexed column (append-only time series) | Tiny index, but useless if physical order does not match the column |
| Full-text / FTS5 | Word and phrase search | Word-based, never substring |
| Trigram (`pg_trgm`) | Substring, fuzzy, `LIKE '%x%'`, typo tolerance | Large index; worth it exactly when B-tree cannot help |

Filtered/partial and covering variants apply to most of these — see SKILL.md Index Strategy for the syntax.

## Composite Index Design

Order columns: **equality first, then the one range/sort column, then included payload columns.** Only the first range column can use the index for ordering; everything after it is filtered, not sought.

Worked example. Query: `WHERE tenant_id = ? AND status = ? AND created_at > ? ORDER BY created_at DESC LIMIT 20`.

- `(tenant_id, status, created_at)` — correct. Both equalities seek, `created_at` provides the range and the sort, so the `LIMIT` stops after 20 rows.
- `(created_at, tenant_id, status)` — wrong. The index is ordered by time across all tenants; the engine scans time descending and discards other tenants' rows until it collects 20.
- `(tenant_id, created_at, status)` — half right. Seeks on tenant and time, but `status` is filtered per row: with 5% matching, the engine reads roughly 400 rows to return 20.

Rule of thumb for how many indexes: a write-heavy OLTP table carrying more than about five indexes is paying more on every insert than most read plans save. Consolidate — one composite often replaces three single-column indexes.

## Sargability (Can The Predicate Use An Index)

Sargable: `col = ?`, `col > ?`, `col BETWEEN ? AND ?`, `col IN (...)`, `col LIKE 'prefix%'`, `col IS NULL`.

Not sargable, with the rewrite:

| Written | Rewrite |
|---|---|
| `WHERE YEAR(d) = 2024` | `WHERE d >= '2024-01-01' AND d < '2025-01-01'` |
| `WHERE d::date = CURRENT_DATE` | `WHERE d >= CURRENT_DATE AND d < CURRENT_DATE + 1` |
| `WHERE amount * 1.2 > 100` | `WHERE amount > 100 / 1.2` |
| `WHERE lower(email) = ?` | Expression index on `lower(email)`, or store the normalized column |
| `WHERE col LIKE '%x%'` | Trigram index, or full-text if word-level is enough |
| `WHERE a = ? OR b = ?` | `SELECT ... WHERE a = ? UNION ALL SELECT ... WHERE b = ? AND a <> ?` |
| `WHERE col <> 'done'` | Partial index on the values you actually query, or list them with `IN` |
| `WHERE COALESCE(col, 0) > 5` | `WHERE col > 5` plus an explicit `OR col IS NULL` branch if needed |

## Rewrites That Win

- `EXISTS` over `IN (subquery)` for large subqueries: `EXISTS` short-circuits at the first match; `IN` may materialize the whole set. `NOT EXISTS` is also NULL-safe (SKILL.md rule 6).
- `LIMIT` with a covering index turns an ordered scan into an early stop. Without the matching order, the engine sorts everything then discards.
- Window function instead of a correlated subquery per row: one pass with `ROW_NUMBER()` replaces N executions.
- `LATERAL` join for top-N-per-group at scale beats `ROW_NUMBER()` filtering when there are many groups and few rows per group, because it can stop per group.
- `UNION ALL` instead of `UNION` when duplicates are impossible: `UNION` sorts the whole result to deduplicate.
- Push filters into subqueries and CTEs rather than filtering the outer result — with an optimization-fence CTE (PostgreSQL <12, or `MATERIALIZED`), the outer predicate cannot reach inside.
- Replace `OFFSET` deep pagination with keyset; this is a plan improvement of a different order than any index.
- Batch N single-row statements into one multi-row statement: round-trip latency usually dominates execution time for small writes.

## Aggregation Performance

- Pre-aggregate before joining whenever a 1:N join feeds a `SUM`/`COUNT` — it also fixes the fan-out correctness bug (SKILL.md Traps).
- `COUNT(*)` on a large table is a scan in PostgreSQL (MVCC visibility must be checked per row) but O(1) in MyISAM and cheap in SQL Server. Use the planner estimate for "about how many".
- `GROUP BY` on a low-cardinality column can be served from an index; `GROUP BY` on a high-cardinality expression cannot.
- Materialized views trade freshness for latency; incremental rollup tables trade complexity for both.
- `DISTINCT` and `GROUP BY` on the same columns usually produce the same plan — the readability choice is free.

## Sorting

- An index provides order only if the `ORDER BY` matches its column order AND direction pattern. Mixed directions (`a ASC, b DESC`) need an index declared that way, which PostgreSQL and SQL Server support and SQLite supports from 3.3.
- Sorting the full result then applying `LIMIT` is a top-N heap sort in most engines — much cheaper than a full sort, but still touches every row. Only an index removes that.
- NULL sort position differs by engine; adding `NULLS LAST` to a query can disable an index that does not carry that ordering.

## Memory and Spills

- A sort or hash that exceeds the per-node work memory spills to disk. In PostgreSQL, `EXPLAIN ANALYZE` reports `Sort Method: external merge Disk: NkB` — that string is the direct signal.
- The budget is per node, not per query: a query with four sorts uses four allocations, and every connection can do this at once. Raising the global value multiplies by connections (`pg` for the sizing formula).
- Cheaper than more memory: fewer rows entering the sort (filter earlier), or an index that supplies the order.

## Prepared Statements and Plan Caching

- A prepared statement may switch to a generic plan after several executions. With skewed data, the generic plan is right on average and wrong for the parameter you care about.
- Symptom: identical SQL is fast in a console with literals, slow from the application.
- Fixes in order: keep parameters out of highly skewed predicates, split the query into per-branch statements, or force a custom plan per execution (`plan_cache_mode` in PostgreSQL >=12; `OPTION (RECOMPILE)` in SQL Server).
- Server-side prepared statements also interact with transaction-mode connection poolers.

## Measuring Honestly

1. Warm the cache: run the query twice and compare the second run against the second run of the alternative. First runs measure disk, not the change.
2. Compare buffers/reads, not just wall time — a shared machine's wall time is noise.
3. Change one thing at a time, and re-run `EXPLAIN ANALYZE` after each. Two changes at once means you keep an index you did not need.
4. Verify on production-shaped data volumes. A plan chosen over 1,000 rows tells you nothing about 10 million.
5. After deploying an index, confirm it is actually used (`pg_stat_user_indexes.idx_scan` climbing) and that write latency did not regress.

## When Not To Tune

- The query runs once a month and takes 40 seconds. Cost of tuning exceeds cost of waiting.
- The table has 5,000 rows. Full scans of small tables are free; adding indexes there adds write cost and clutter.
- The real fix is upstream: pagination the UI does not need, a report that should be a nightly rollup, or an N+1 the ORM emits.
- You are at the limit of one node rather than one query — that is a topology problem.
