# Loading and Moving Data

Imports fail on the data, not the SQL: encodings, quoting, NULL conventions, and types the source never enforced. The workflow that survives is always the same — land raw text, validate, then transform into typed tables.

Contents: The Staging Pattern · CSV Quirks · Bulk Load by Engine · Load Performance · Idempotent Loads · Validation · Upsert at Scale · Exporting · Dump and Restore · Cross-Engine Migration · Type Mapping · Verification · Traps

## The Staging Pattern

1. Create a staging table with **every column as text** and no constraints. A load that fails on row 400,000 because of one bad date has wasted the whole run.
2. Load the file into staging with the fastest bulk mechanism available.
3. Run validation queries against staging: row count, null counts, distinct value sets, parse failures.
4. `INSERT ... SELECT` with explicit casts into the real table, routing rejects to a separate table with the reason.
5. Keep the staging table until the load is confirmed; it is the only copy of what actually arrived.

This is slower to write and much faster to finish than a load that aborts halfway.

## CSV Quirks That Cost Hours

| Symptom | Cause | Handling |
|---|---|---|
| Row count off by one | Header row loaded as data, or missing `HEADER` option | Declare the header explicitly |
| Fields shifted after a specific row | Unescaped delimiter inside a quoted field, or a newline inside a quoted field | Use the engine's real CSV parser (`COPY ... CSV`), never a hand-split |
| Empty strings became NULL (or vice versa) | Engines differ: PostgreSQL `COPY CSV` treats unquoted empty as NULL, quoted empty as `''` | Set `NULL ''` explicitly and decide which one you mean |
| Leading zeros gone | Loaded straight into a numeric column | Load as text into staging; cast only what is truly numeric |
| Dates off by a month | `MM/DD/YYYY` parsed as `DD/MM/YYYY` | Set `DateStyle`/`--date-format` explicitly, or parse in the cast step |
| "invalid byte sequence" | File is Latin-1/Windows-1252, declared UTF-8 | Convert with `iconv` before loading; do not guess row by row |
| First column name has odd prefix characters | UTF-8 BOM | Strip the BOM |
| Line endings appear in the last column | CRLF file loaded as LF | Set the newline convention, or strip `\r` in the cast |
| Numbers with thousands separators or currency symbols | Locale-formatted export | Clean in the cast step: strip, then cast |
| `\N`, `NULL`, `-`, `n/a` all present | Multiple export tools in the source pipeline | Normalize in the cast step, not with a single NULL marker |

## Bulk Load by Engine

```bash
# PostgreSQL: server-side COPY (needs server file access); \copy runs client-side
psql -c "\copy staging_users FROM 'users.csv' WITH (FORMAT csv, HEADER, NULL '')"

# MySQL: LOCAL reads the client's filesystem; the server may require local_infile=1
mysql -e "LOAD DATA LOCAL INFILE 'users.csv' INTO TABLE staging_users
          FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '\"'
          LINES TERMINATED BY '\n' IGNORE 1 LINES"

# SQLite
sqlite3 mydb.sqlite -cmd ".mode csv" -cmd ".import --skip 1 users.csv staging_users"

# SQL Server
sqlcmd -Q "BULK INSERT staging_users FROM 'users.csv'
           WITH (FORMAT='CSV', FIRSTROW=2, FIELDTERMINATOR=',')"
```

`COPY`/`LOAD DATA` is typically an order of magnitude faster than the equivalent `INSERT` batches for the same rows; single-row inserts in a loop are two orders slower and dominated by round-trip latency.

## Load Performance

- Drop or disable secondary indexes before loading into an **empty** table, rebuild after. On a table that already holds data and stays queryable, keep them — a missing index during the load makes concurrent reads worse than the load itself.
- Load inside one transaction where the engine allows it, so failure leaves nothing behind — but a single enormous transaction bloats WAL/undo. Chunk at `batch_size` rows per commit (default 5,000) for very large loads.
- Defer foreign key checks during the load and validate afterwards: PostgreSQL `SET CONSTRAINTS ALL DEFERRED` (constraints must be declared `DEFERRABLE`), MySQL `SET FOREIGN_KEY_CHECKS = 0` (which does **not** validate retroactively — you must check for orphans yourself).
- `ANALYZE` immediately after the load. Every plan until then is based on the pre-load statistics, and this is the top cause of "the import worked but now everything is slow".
- SQLite specifically: `PRAGMA journal_mode = WAL` and one transaction around the inserts turn a minutes-long load into seconds.

## Idempotent, Resumable Loads

- Give every source row a stable natural key or a file-plus-line identifier, and make the target write an upsert. Reruns then converge instead of duplicating.
- Record progress in the same transaction as the work: a `load_batches(file, chunk_no, rows, loaded_at)` row committed with its chunk means a restart resumes exactly where it stopped.
- Never rely on "the file has not changed" — hash the file and store the hash with the batch record.
- Loading the same file twice is normal in production. A load pipeline without a uniqueness constraint on the target will eventually double every number in the business.

## Validation Before Promotion

```sql
-- Row count against the source
SELECT COUNT(*) FROM staging_users;

-- Which columns actually failed to parse
SELECT COUNT(*) FILTER (WHERE amount !~ '^-?\d+(\.\d+)?$') AS bad_amount,
       COUNT(*) FILTER (WHERE email NOT LIKE '%@%')        AS bad_email,
       COUNT(*) FILTER (WHERE created_at IS NULL OR created_at = '') AS missing_date
FROM staging_users;

-- Duplicates on the intended key, before the unique constraint rejects the load
SELECT email, COUNT(*) FROM staging_users GROUP BY email HAVING COUNT(*) > 1;

-- Orphans against the parent table
SELECT COUNT(*) FROM staging_orders s
LEFT JOIN users u ON u.id = s.user_id::bigint WHERE u.id IS NULL;
```

Compare the loaded row count against the source count every time, and reconcile a control total (a sum of one numeric column) — count alone misses truncated fields.

## Upsert at Scale

```sql
-- PostgreSQL / SQLite: one statement, no race
INSERT INTO users (email, name, updated_at)
SELECT email, name, NOW() FROM staging_users
ON CONFLICT (email) DO UPDATE
SET name = EXCLUDED.name, updated_at = EXCLUDED.updated_at
WHERE users.name IS DISTINCT FROM EXCLUDED.name;   -- skip no-op writes

-- MySQL
INSERT INTO users (email, name) SELECT email, name FROM staging_users
ON DUPLICATE KEY UPDATE name = VALUES(name);
```

- The `WHERE ... IS DISTINCT FROM` clause matters at scale: without it, an unchanged row is still rewritten, producing dead tuples, WAL, and replication traffic for nothing.
- Deduplicate the source before the upsert. PostgreSQL raises `ON CONFLICT DO UPDATE command cannot affect row a second time` when one statement touches the same key twice.
- `MERGE` (SQL Server, PostgreSQL >=15) reads more naturally for multi-action loads but has more concurrency footguns; prefer `ON CONFLICT` when it expresses the operation.

## Exporting

```bash
# PostgreSQL
psql -c "\copy (SELECT * FROM users WHERE created_at >= '2026-01-01') TO 'users.csv' CSV HEADER"

# MySQL: SELECT ... INTO OUTFILE writes on the SERVER and needs secure_file_priv;
# from the client, pipe a batch-mode query instead
mysql --batch --raw -e "SELECT * FROM users" mydb > users.tsv

# SQLite
sqlite3 -header -csv mydb.sqlite "SELECT * FROM users" > users.csv

# SQL Server
sqlcmd -Q "SELECT * FROM users" -s"," -W -o users.csv
```

Export the query, not the table, whenever personal data is involved — select the columns you are allowed to share. Quote and escape via the engine's CSV writer; a hand-built `CONCAT` export breaks on the first embedded comma.

## Dump and Restore

```bash
pg_dump -Fc mydb > backup.dump            # custom format: compressed, selective, parallel restore
pg_restore -d mydb -j 4 backup.dump
mysqldump --single-transaction mydb > backup.sql   # consistent snapshot without locking (InnoDB)
sqlite3 mydb.sqlite ".backup backup.sqlite"        # safe during writes; plain cp is not
```

Full backup strategy, retention, and restore drills route from SKILL.md Quick Reference. For loading purposes the relevant points are that a dump restores a *schema plus data* pair — restoring into a database whose schema has moved on fails on the first mismatch — and that `--schema-only`/`--data-only` splits let you restore data into an already-migrated schema.

## Cross-Engine Migration

Order of operations that avoids rework:

1. **Inventory the incompatibilities first**: auto-increment style, boolean representation, `ENUM`s, unsigned integers, zero dates, character sets, and every stored procedure, trigger, and view.
2. **Translate the schema by hand or with a tool, then review it.** Automated converters map types conservatively — everything becomes `TEXT` and `DOUBLE` unless corrected.
3. **Move data through a neutral format** (CSV per table) or a purpose-built tool (`pgloader` for MySQL→PostgreSQL). Do not use one engine's SQL dump as another's input.
4. **Sequences and identity columns must be reset** after the load, or the first insert collides: `SELECT setval('users_id_seq', (SELECT MAX(id) FROM users))`.
5. **Recreate indexes and constraints after loading**, not before.
6. **Run both systems in parallel** with dual writes and a reconciliation query before cutting over, unless downtime is acceptable.

## Type Mapping Landmines

| Source | Target | Problem |
|---|---|---|
| MySQL `TINYINT(1)` | boolean | Holds any value 0-255; audit for values outside 0/1 before casting |
| MySQL `DATETIME '0000-00-00'` | `TIMESTAMPTZ` | Not a valid date anywhere else; decide NULL or a sentinel date before the load |
| MySQL unsigned `BIGINT` | PostgreSQL `BIGINT` | Values above 2^63-1 do not fit; check `MAX()` first |
| MySQL `ENUM` | `TEXT` + CHECK | Ordering was by declaration order, not alphabetical — any `ORDER BY` on it changes meaning |
| PostgreSQL `TEXT[]` | MySQL | No array type; junction table or JSON |
| PostgreSQL `TIMESTAMPTZ` | MySQL `TIMESTAMP` | MySQL's range stops in 2038 and converts using the session timezone; use `DATETIME` in UTC |
| Any `FLOAT` money | `NUMERIC` | Values are already imprecise; the migration cannot restore lost cents |
| SQLite anything | Typed engine | SQLite has type affinity, not enforcement: a `INTEGER` column can hold text. Profile actual values, not the declared type |

## Verification After Any Move

- Row counts per table, both sides.
- A control total per table: `SUM` of a numeric column and `COUNT` of NULLs per important column.
- Min/max of every date column — this catches timezone shifts and epoch defaults instantly.
- Checksum a sample: order by primary key, hash the concatenated columns, compare.
- Run the application's ten most important queries against both and diff the results.
- Confirm sequences, defaults, constraints, indexes, and grants exist on the target; data-only migrations routinely lose all five.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Loading straight into the typed target table | One bad row aborts the whole load, and you learn about it at minute 40 | Text staging table first |
| Splitting CSV lines on the delimiter | Quoted fields containing delimiters or newlines shift every subsequent column | The engine's CSV parser |
| `SET FOREIGN_KEY_CHECKS = 0` and forgetting to verify | MySQL does not re-validate when you turn it back on | Explicit orphan query after the load |
| Skipping `ANALYZE` after a large load | Every plan uses the pre-load statistics | `ANALYZE` before declaring the load done |
| One transaction for tens of millions of rows | Bloats WAL/undo, blocks vacuum, and an error loses everything | Chunk and commit; record progress |
| Re-running a load "to be safe" | Duplicates the data unless the target has a unique key and an upsert | Idempotent upsert keyed on a stable source identifier |
| Dumping production to a laptop for a "quick look" | An uncontrolled copy of every personal record you hold | Query the columns you need, on a masked environment |
| Trusting a declared type in a SQLite source | Affinity, not enforcement — the column may hold anything | Profile the real values before mapping types |
