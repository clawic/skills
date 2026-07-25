# Debugging — Symptom to Cause

Work symptom-first. Each chain is ordered by probability, and every step is a check you can run, not a guess. Wrong results outrank slow results: a fast query returning the wrong rows never raises an error.

Contents: First Three · Wrong Row Count · Wrong Totals · Query Slow · Was Fast Yesterday · Index Ignored · Hangs · Deadlock · Connection Errors · Constraint Violations · Disk Full · Encoding · Works In One Client · Migration Failed · Replica Disagrees

## The Universal First Three

1. **Read the actual statement, not the intent.** For ORM-generated SQL, turn on query logging and copy the emitted text; for dynamic SQL, log the final string with parameters bound.
2. **Count before you optimize.** `SELECT COUNT(*)` on each base table with only its own filters. If a base count already surprises you, the bug is upstream of the join.
3. **`EXPLAIN (ANALYZE, BUFFERS)`** the real statement with real parameters. A plan for `user_id = 1` can differ from `user_id = 99` when one value is a most-common-value in the stats.

## Wrong Row Count (too many, too few, duplicates)

1. Strip the query to one table plus its `WHERE`. Add joins back one at a time and re-count after each — the join that changes the count names the bug.
2. Count grew → 1:N fan-out. Check the join key's uniqueness: `SELECT key, COUNT(*) FROM t GROUP BY key HAVING COUNT(*) > 1`. Fix by pre-aggregating, not by `DISTINCT` (SKILL.md Traps).
3. Count shrank → a `LEFT JOIN` with a predicate in `WHERE` (now an inner join, with no error), or a `NOT IN` against a subquery containing NULL (SKILL.md rule 6).
4. Count differs between two "equivalent" queries → collation or case sensitivity: MySQL's `_ci` collations match `'A' = 'a'`, PostgreSQL does not.
5. Trailing-space equality: `CHAR(n)` pads, and some engines ignore trailing spaces in comparison. `'a ' = 'a'` is true on MySQL `CHAR`, false on PostgreSQL `TEXT`.
6. Count changes between runs of the same query → concurrent writes, or a non-deterministic `LIMIT` without `ORDER BY`.

## Wrong Totals (sums, averages, percentages)

- Sum too high → fan-out (above). Verify by comparing `SUM(x)` against `SUM(x) / COUNT(DISTINCT join_key)`-shaped intuition, then rewrite with a pre-aggregated CTE.
- Average is off → `AVG` skips NULLs, so the denominator is not the row count. `SUM(x) / COUNT(*)` and `AVG(x)` differ whenever `x` is nullable.
- Money off by cents → `FLOAT`/`REAL` storage. `0.1 + 0.2 != 0.3` accumulates over aggregation; the schema needs `NUMERIC`/`DECIMAL` (SKILL.md rule 7).
- Integer division: `SELECT 1/2` is `0` in PostgreSQL, MySQL, and SQL Server for integer operands. Cast one side: `x * 100.0 / y`.
- Percentages don't sum to 100 → rounding each row instead of the total; round once at the end.
- Totals differ from the dashboard → different timezone boundary for "today" or a soft-delete filter one query applies and the other doesn't.

## Query Slow

1. Is this query even the problem? Rank by total cost first (SKILL.md rule 9).
2. `EXPLAIN (ANALYZE, BUFFERS)`: find the node with the largest actual time, not the largest estimate.
3. Estimated vs actual rows off by more than 10× → stale statistics; `ANALYZE` the table and re-plan.
4. Sequential scan with a selective filter → the index is missing or disabled (→ Index Ignored).
5. Correct index, still slow → the query returns too many rows to matter. A million rows through the network is slow no matter the plan; paginate or aggregate server-side.
6. Fast standalone, slow in the app → parameter sniffing, a different search_path/session setting, or lock waiting rather than work (→ Hangs).
7. Everything is slow, not one query → server-level: cache hit ratio, connection saturation, or another workload.

## "It Was Fast Yesterday"

| Cause | Check |
|---|---|
| Data crossed a size threshold and the plan flipped | Compare current plan against the shape you remember; look for a switch from index scan to seq scan |
| Statistics went stale after a bulk load | `ANALYZE`, re-plan; PostgreSQL: `last_analyze` in `pg_stat_user_tables` |
| An index was dropped or left INVALID by a failed concurrent build | PostgreSQL: `SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid` |
| Bloat from a long-running transaction | Oldest `xact_start` in `pg_stat_activity` |
| A new index made writes slow, not reads | Compare index count against write latency; every index is written on every insert |
| Cache no longer holds the working set | Cache hit ratio dropped below the 99% OLTP threshold |
| Someone added a `LEFT JOIN` for one column | Read the diff of the query, not just the plan |
| Version upgrade changed the planner | Compare plans across the versions before blaming data |

## Index Exists But Is Not Used

Check in this order; the first four cover most cases.

1. **Function or cast on the column.** `WHERE lower(email) = ?`, `WHERE created_at::date = ?`, or an implicit cast from a type mismatch all hide the column. Fix the predicate or add an expression index that matches the query text exactly.
2. **Wrong column order.** A composite `(a, b)` cannot serve `WHERE b = ?` (SKILL.md rule 3).
3. **Low selectivity.** Matching more than roughly 5-10% of rows, a scan is genuinely cheaper. Confirm with `SELECT COUNT(*) FILTER (WHERE <predicate>) * 100.0 / COUNT(*) FROM t`.
4. **Stale stats** making the planner think the predicate is unselective. `ANALYZE`, retry.
5. **`OR` across columns**, leading wildcard `LIKE '%x'`, or a non-C locale with `LIKE 'x%'` in PostgreSQL (SKILL.md Index Strategy).
6. **Different collation** between the index and the query's comparison (MySQL joins across `utf8mb4_general_ci` and `utf8mb4_0900_ai_ci` cannot use the index).
7. **The index is invalid or being built.** PostgreSQL `indisvalid = false`; MySQL `SHOW INDEX` `Comment` column.
8. Prove it before rewriting: force the choice temporarily (`SET enable_seqscan = off` in PostgreSQL, `FORCE INDEX` in MySQL) and compare actual times. If forcing the index is slower, the planner was right — never ship the hint as the fix.

## Query Hangs (no result, no error)

1. Is it running or waiting? PostgreSQL `pg_stat_activity.wait_event_type = 'Lock'` means waiting; MySQL `SHOW PROCESSLIST` state `Waiting for table metadata lock`.
2. Waiting → find the blocker (PostgreSQL `pg_blocking_pids(pid)`, MySQL `sys.innodb_lock_waits`) and decide: cancel it, or wait if it is nearly done.
3. Running with no output → it may be working correctly on too much data; `EXPLAIN` without `ANALYZE` returns instantly and shows the plan it chose.
4. Client shows nothing while the server is idle → the result is buffered by the driver, or the app never fetched; check the driver's cursor/fetch mode.
5. `ALTER TABLE` hangs → the lock queue, not the DDL. Every new query is now queued behind it: cancel, set `lock_timeout`, retry.
6. An open transaction in a REPL or notebook is the most common self-inflicted hang: an uncommitted `BEGIN` in another window holds the lock.

## Deadlock Detected

- The engine already picked a victim and rolled it back; the error is the report, not the failure. Read the log: PostgreSQL logs both statements, MySQL exposes the last one in `SHOW ENGINE INNODB STATUS`.
- Root cause is almost always inconsistent lock order across two code paths. Fix by ordering (`ORDER BY id FOR UPDATE`), not by retrying blindly.
- Deadlocks that appear only under load with no obvious two-row pattern → gap/next-key locks in MySQL REPEATABLE READ, or foreign-key locks taken on the parent row.
- Every application that writes concurrently needs a bounded retry on deadlock and serialization errors regardless.

## Connection Errors

| Error | Real cause | Move |
|---|---|---|
| "too many connections" / "remaining slots reserved" | App pools × instances exceed the server limit | Size pools from cores, add PgBouncer instead of raising `max_connections` |
| "connection refused" | Server not listening on that address, or wrong port | Check bind address and port before credentials |
| "no pg_hba.conf entry" / "Host is not allowed" | Host-based auth rules, not a bad password | Fix the auth rule for the client's network |
| "password authentication failed" for one app only | Different role than you tested with, or a rotated secret | Compare the role, not the password |
| "SSL required" / cert errors | Managed providers force TLS | Set the driver's SSL mode; do not disable verification to move on |
| Connections work then die after minutes | Idle timeout in a proxy/load balancer below the pool's `max_lifetime` | Set pool `max_lifetime` under the infrastructure timeout |
| Intermittent failures under load only | Pool exhaustion — waiters timing out, not the database refusing | Instrument pool wait time before touching the server |

## Constraint Violations

- Unique violation on insert that "should not exist" → a soft-deleted row still occupies the value, or the unique index is on a different column set than assumed.
- Unique violation under concurrency despite a check-then-insert → the check-then-insert race; use `INSERT ... ON CONFLICT`/`ON DUPLICATE KEY` or catch the violation.
- Foreign key violation on delete → children exist; decide `RESTRICT` vs `CASCADE` deliberately, and confirm the child FK column is indexed (SKILL.md rule 4).
- Foreign key violation on insert with a valid-looking id → different tenant, or the parent row was inserted in another uncommitted transaction.
- NOT NULL violation only in production → a default exists in one environment's schema and not the other; diff the schemas, don't reason about them.
- CHECK violation after a data import → the import brought values the constraint never saw; the constraint is right.

## Disk Full / Database Won't Accept Writes

1. Locate the consumer before deleting anything: table and index sizes ranked descending.
2. Common consumers in order: an unpartitioned events/audit/log table, bloat from dead tuples pinned by a long transaction, WAL retained by an inactive replication slot, an abandoned temp/sort spill.
3. `DELETE` does not return space to the filesystem on PostgreSQL or MySQL/InnoDB — it creates dead rows the vacuum must clean. Dropping a partition does.
4. Never `DROP` to free space during an incident until the backup is verified restorable.
5. PostgreSQL specifically: an inactive replication slot retains WAL forever and is the classic unannounced disk killer — check `pg_replication_slots` for `active = false`.

## Encoding and Character Problems

- `?` or `????` in place of accents/emoji → MySQL `utf8` (3-byte) instead of `utf8mb4`; the column, the table default, AND the client connection charset must all be `utf8mb4`.
- Mojibake (`Ã©` for `é`) → UTF-8 bytes read as Latin-1 somewhere in the chain; find the single wrong layer rather than re-encoding twice.
- Import fails on "invalid byte sequence" → the file is not the encoding you declared; detect and convert before loading.
- Sorting looks wrong → collation, not encoding. `ORDER BY` follows the column's collation; changing it after the fact requires reindexing.
- Invisible characters (zero-width space, non-breaking space) pasted into data make equality fail with identical-looking strings; compare `length()` against the visible character count.

## Works In One Client, Fails In Another

| Difference | Check |
|---|---|
| Autocommit on/off | The GUI committed; your script left an open transaction |
| Session settings (`search_path`, `sql_mode`, timezone) | Compare `SHOW ALL` / `SHOW VARIABLES` between sessions |
| Different role or database | Same host, different privileges: diff the `GRANT`s and any RLS policy on the table |
| Statement/lock timeouts set per role | A short `statement_timeout` on the app role only |
| Prepared vs literal statements | The generic plan for a prepared statement can differ from the plan for literals |
| Driver-side type conversion | The driver casts a parameter to a type the index can't use |

## Migration Failed Halfway

1. Check the runner's state table first — a "dirty" flag means it knows it stopped mid-way; resolve that before rerunning anything.
2. MySQL has no transactional DDL: half the statements are already committed. Determine what actually applied by inspecting the schema, not the migration file.
3. Re-running a partially applied migration usually fails on "already exists". Write the repair as a NEW migration; never edit an applied one.
4. A migration that timed out may still be running server-side — kill the backend before retrying, or the retry deadlocks against it.
5. Prevention: run every migration against a restored copy of production-shaped data before it reaches production.

## Replica Returns Different Data

- Read-after-write on an async replica misses the write that just committed; route read-your-own-writes traffic to the primary.
- A long analytics query on a replica gets canceled by WAL replay conflicts — that is not a query bug.
- Replica lag rises with zero write traffic on the primary: the time-based lag metric grows unbounded when there is nothing to replay; alert only while writes are flowing.
- Counts differ permanently → the replica broke and is no longer replaying, or someone wrote to it. Compare replay position, not row counts.

## When You Are Truly Stuck

Reduce to the smallest reproduction: one table, minimal columns, the fewest rows that still show the behavior, on a scratch copy. Most "impossible" SQL bugs resolve at the moment the reproduction stops reproducing — the step you removed is the cause.
