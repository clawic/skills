# Dialects — Where Engines Actually Differ

Portable SQL is a smaller language than most people write. This file is the list of places a statement that runs on one engine does something else on another without complaining, plus how to choose an engine in the first place.

Contents: Choosing an Engine · Identifiers and Quoting · Strings and Collation · NULL Ordering · Types · Auto-Increment · Upsert · Limit and Top · Returning · Grouping Rules · Window and CTE Support · DDL Transactionality · Booleans · Concatenation and Math · Error Behavior · Feature Floors · Portability Strategy

## Choosing an Engine

| Engine | Choose when | Real cost |
|---|---|---|
| SQLite | Embedded, single-machine, local-first apps, tests, CLI tools, read-heavy sites on one box | One writer at a time; no network access; type affinity instead of enforcement |
| PostgreSQL | Default for anything server-side: richest types, strictest correctness, extensions | Connection-per-process (needs pooling); more knobs to get wrong |
| MySQL / MariaDB | The platform or host dictates it, or the team's operational muscle is there | No transactional DDL; historically lenient defaults; MariaDB and MySQL have diverged |
| SQL Server | .NET/Windows shops, existing licensing, strong tooling requirements | Licensing; default lock-based isolation until `READ_COMMITTED_SNAPSHOT` is enabled |

Do not switch engines for a performance problem that is really a missing index or a bad plan. Do switch when the workload shape is wrong: heavy analytical scans belong in a columnar store (`duckdb`, `clickhouse`, a warehouse), not in a tuned OLTP database.

## Identifiers and Quoting

| Engine | Quote char | Unquoted case | Consequence |
|---|---|---|---|
| PostgreSQL | `"col"` | Folded to **lower**case | `CREATE TABLE "Users"` must be quoted forever after |
| MySQL | `` `col` `` | Preserved; table-name case sensitivity depends on the filesystem | A schema built on macOS breaks on Linux |
| SQLite | `"col"`, `` `col` ``, `[col]` | Preserved, compared case-insensitively | Very permissive; hides problems until you migrate |
| SQL Server | `[col]` | Preserved, compared by database collation | Usually case-insensitive |

Use lowercase snake_case unquoted everywhere and the problem never appears. MySQL's `lower_case_table_names` is set at initialization and cannot be safely changed afterwards — decide before the first deploy.

## Strings and Collation

- Default comparison: PostgreSQL and SQLite are case-**sensitive**; MySQL (`utf8mb4_0900_ai_ci`) and SQL Server are case-**insensitive** by default. The same `WHERE email = ?` finds different rows across engines.
- PostgreSQL: use `citext` or compare `lower(col)` with a matching expression index. MySQL: choose a `_bin` or `_cs` collation on the column when you need sensitivity.
- MySQL `utf8` is 3-byte and cannot store emoji or some CJK characters. `utf8mb4` is the only correct choice, and it must be set on the column, the table, and the connection (SKILL.md rule 7).
- Mixing collations across a join disables the index and can raise "illegal mix of collations" — align collations when creating tables, not later.
- `CHAR(n)` pads with spaces and some engines ignore trailing spaces in comparison; `VARCHAR`/`TEXT` do not. Avoid `CHAR` except for genuinely fixed-width codes.
- Empty string versus NULL: Oracle treats `''` as NULL; every engine here treats them as different values. Anything importing from Oracle inherits that ambiguity.
- Concatenating anything with NULL yields NULL in PostgreSQL, SQLite, and MySQL; SQL Server's `+` does too unless `CONCAT_NULL_YIELDS_NULL` is off, while `CONCAT()` ignores NULLs everywhere.

## NULL Ordering

| Engine | `ORDER BY col ASC` | Override |
|---|---|---|
| PostgreSQL | NULLs **last** | `NULLS FIRST` / `NULLS LAST` |
| SQLite | NULLs first | `NULLS LAST` (>=3.30) |
| MySQL | NULLs first | No clause — sort by `col IS NULL, col` |
| SQL Server | NULLs first | No clause — sort by `CASE WHEN col IS NULL THEN 1 ELSE 0 END, col` |

Adding an explicit `NULLS LAST` can disable an index that does not carry that ordering, so prefer making the column `NOT NULL` when the ordering matters.

## Types

| Concept | PostgreSQL | MySQL | SQLite | SQL Server |
|---|---|---|---|---|
| Unbounded text | `TEXT` | `TEXT`/`LONGTEXT` (cannot be fully indexed; needs a prefix length) | `TEXT` | `NVARCHAR(MAX)` |
| Exact decimal | `NUMERIC(p,s)` | `DECIMAL(p,s)` | `NUMERIC` (affinity only) | `DECIMAL(p,s)` |
| Boolean | `BOOLEAN` | `TINYINT(1)` | `INTEGER` 0/1 | `BIT` |
| UUID | `UUID` | `BINARY(16)` or `CHAR(36)` | `TEXT`/`BLOB` | `UNIQUEIDENTIFIER` |
| Array | `TYPE[]` | none (JSON) | none (JSON) | none (JSON) |
| JSON | `JSONB` | `JSON` | JSON functions over `TEXT` | `NVARCHAR` + JSON functions |
| Enum | native `ENUM` type | inline `ENUM(...)` | none | none |
| Unsigned integers | none | yes | none | none |
| IP / network | `INET`, `CIDR` | none | none | none |

SQLite's type affinity means declared types are advisory: an `INTEGER` column accepts `'abc'` unless the table is declared `STRICT` (>=3.37). Any migration out of SQLite must profile actual values, not declared types.

MySQL cannot index a `TEXT` column without a prefix length (`INDEX (col(191))`), and the 191 convention comes from the old 767-byte index limit under `utf8mb4` — modern InnoDB with `DYNAMIC` row format allows 3072 bytes, so the limit is often no longer needed.

## Auto-Increment Behavior

| Engine | Syntax | Gaps | Reset after load |
|---|---|---|---|
| PostgreSQL | `GENERATED ALWAYS AS IDENTITY` (prefer over `SERIAL`) | Sequences are non-transactional: rollbacks consume values | `SELECT setval('t_id_seq', (SELECT MAX(id) FROM t))` |
| MySQL | `AUTO_INCREMENT` | Gaps on rollback; InnoDB may reset the counter on restart in older versions | `ALTER TABLE t AUTO_INCREMENT = n` |
| SQLite | `INTEGER PRIMARY KEY` (rowid alias) | Reuses deleted maximum values unless `AUTOINCREMENT` is declared | Managed via `sqlite_sequence` |
| SQL Server | `IDENTITY(1,1)` | Large jumps possible after restart (identity cache) | `DBCC CHECKIDENT` |

Gaps in generated ids are normal in every engine. Treat them as meaningless: any business logic that counts on contiguous ids (invoice numbering, "records processed") is already broken and needs its own sequence table.

## Upsert

```sql
-- PostgreSQL / SQLite
INSERT INTO t (k, v) VALUES (?, ?)
ON CONFLICT (k) DO UPDATE SET v = EXCLUDED.v;

-- MySQL
INSERT INTO t (k, v) VALUES (?, ?)
ON DUPLICATE KEY UPDATE v = VALUES(v);          -- MySQL >=8.0.20: v = new.v

-- SQL Server (and PostgreSQL >=15)
MERGE INTO t AS tgt USING (VALUES (?, ?)) AS src(k, v) ON tgt.k = src.k
WHEN MATCHED THEN UPDATE SET v = src.v
WHEN NOT MATCHED THEN INSERT (k, v) VALUES (src.k, src.v);
```

`ON DUPLICATE KEY` fires on **any** unique constraint, not just the one you had in mind — a row can update through a different key than expected. `ON CONFLICT (k)` names the constraint explicitly, which is why it is the safer construct.

## Limit, Offset, Top

```sql
SELECT ... ORDER BY id LIMIT 20 OFFSET 40;                          -- PostgreSQL, MySQL, SQLite
SELECT TOP 20 ... ORDER BY id;                                      -- SQL Server, no offset
SELECT ... ORDER BY id OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY;      -- SQL Server 2012+, standard
```

SQL Server requires `ORDER BY` for `OFFSET ... FETCH`. Every engine returns undefined order without `ORDER BY`, and a unique tiebreaker is required for stable pagination.

## Returning Affected Rows

- PostgreSQL and SQLite (>=3.35): `INSERT/UPDATE/DELETE ... RETURNING *`.
- SQL Server: `OUTPUT INSERTED.*` / `OUTPUT DELETED.*`, which can also write into a table.
- MySQL: none. Use `LAST_INSERT_ID()` for a single insert, or re-select. MariaDB has `RETURNING`.

Multi-row inserts on MySQL give you the first generated id from `LAST_INSERT_ID()`; the rest are consecutive only when `innodb_autoinc_lock_mode` guarantees it. Batch inserts needing ids back should insert client-generated keys instead.

## GROUP BY Strictness

- Standard, PostgreSQL, and SQL Server: every selected non-aggregated column must appear in `GROUP BY` (PostgreSQL relaxes this when you group by the primary key, since everything else is functionally dependent).
- MySQL historically allowed selecting arbitrary ungrouped columns, returning an unspecified row's value. `ONLY_FULL_GROUP_BY` is in the default `sql_mode` from 5.7 — legacy queries fail on upgrade, and the fix is usually `ANY_VALUE()` or a proper aggregate.
- SQLite permits bare columns and documents that with `MIN`/`MAX` the bare columns come from the matching row — convenient, non-portable.

## Window Functions and CTEs

| Feature | Floor |
|---|---|
| Window functions | PostgreSQL 8.4, MySQL 8.0, MariaDB 10.2, SQLite 3.25, SQL Server 2005 |
| Recursive CTE | PostgreSQL 8.4, MySQL 8.0, SQLite 3.8.3, SQL Server 2005 |
| `FILTER (WHERE ...)` on aggregates | PostgreSQL, SQLite 3.30; MySQL and SQL Server need `CASE` |
| `GROUPING SETS`/`CUBE`/`ROLLUP` | PostgreSQL 9.5, SQL Server, MySQL (`WITH ROLLUP` only), SQLite none |
| Generated columns | PostgreSQL 12 (stored only), MySQL 5.7, SQLite 3.31, SQL Server (computed) |
| Partitioned tables | PostgreSQL 10 declarative, MySQL native, SQL Server, SQLite none |
| Full-text | PostgreSQL tsvector, MySQL InnoDB FTS, SQLite FTS5, SQL Server FTS |

CTE materialization also differs: PostgreSQL inlined CTEs from version 12 (before that every CTE was an optimization fence), MySQL 8 treats them like derived tables, SQL Server always inlines. A query that relied on the fence for performance changes behavior on upgrade.

## DDL Transactionality

PostgreSQL, SQLite, and SQL Server run DDL inside transactions — a failed migration rolls back cleanly. MySQL and MariaDB commit implicitly before and after each DDL statement, so a multi-statement migration can end half-applied with no rollback available.

Consequences for MySQL specifically: one DDL statement per migration file, a verified backup before schema changes, and online schema-change tooling (`gh-ost`, `pt-online-schema-change`) for large tables.

## Booleans, Concatenation, Math

- Booleans: PostgreSQL has a real type; MySQL's `TRUE`/`FALSE` are literals for 1/0 in a `TINYINT`; SQL Server has `BIT` and no boolean expression type, so `SELECT (a > b)` is invalid there and needs `CASE`.
- Concatenation: `||` in PostgreSQL and SQLite; `CONCAT()` in MySQL (where `||` means logical OR unless `PIPES_AS_CONCAT` is set); `+` or `CONCAT()` in SQL Server.
- Integer division truncates in PostgreSQL, MySQL, and SQL Server; SQLite also truncates for integer operands. Cast one operand to get a real division.
- Modulo of negatives, rounding mode (half-up vs half-even), and `AVG` of integers all vary. Cast to a decimal type before any arithmetic whose result is reported to a user.

## Error Behavior on Bad Data

- Inserting a too-long string: PostgreSQL and SQL Server error; MySQL errors in strict mode (default since 5.7) and truncated without warning before that.
- Invalid dates: MySQL historically accepted `'0000-00-00'`; strict mode rejects it, but old data survives upgrades and breaks every migration out.
- Division by zero: PostgreSQL, SQL Server, and SQLite raise an error; MySQL returns NULL unless `ERROR_FOR_DIVISION_BY_ZERO` is in `sql_mode`.
- Out-of-range numerics: error in PostgreSQL, clamped or rejected in MySQL depending on mode.

When auditing a MySQL database, read `SELECT @@sql_mode` first — it determines which of these are errors and which are corruption with no error.

## Portability Strategy

- **One target engine.** The portable subset is small, and code written for "any database" is usually written well for none. Support a second engine only when a paying requirement says so.
- If you must be portable: no vendor types, no engine-specific functions in shared code, ANSI `CASE` instead of `IF`/`IIF`, explicit `CAST`, no `RETURNING`, no arrays, no upsert syntax — push the differences into a thin per-engine layer.
- Test against the real target engine, not SQLite-in-memory standing in for PostgreSQL: SQLite accepts statements the target rejects, and enforces less.
- Keep a written list of the version floors you rely on (`engine_version` in Configuration) so an upgrade is a checklist, not an archaeology project.
