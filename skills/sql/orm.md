# ORM-Generated SQL

The ORM is not the problem; the invisibility is. Every fix below starts with reading the SQL the ORM actually emitted, because the generated statement rarely resembles the method chain that produced it.

Contents: See the SQL · N+1 · Over-Fetching · Pagination · Bulk Operations · Transaction Boundaries · Lazy Loading Outside the Session · Type and Parameter Mismatches · Pool Configuration · Migration Autogeneration · When to Drop to SQL · Traps

## First: See the SQL

Nothing here is diagnosable without the emitted statements and their count per request.

- Turn on the ORM's query log in development and print the statement plus its duration and the call site.
- Turn on the database's own logging for a bounded window when the ORM's log is not enough: PostgreSQL `log_min_duration_statement = 0` on a session or role, MySQL's general log, SQL Server Extended Events.
- Count queries per request, not just their duration. A page issuing 340 fast queries looks healthy in every per-query metric and is the most common performance failure in ORM applications.
- Add an assertion on the query count for the endpoints that matter — it is the only detector that survives refactoring.

## N+1 Queries

The shape: one query for a list, then one query per element for a relation.

- Fix by eager loading the relation in the same round trip. Two mechanisms exist in most ORMs and they are not interchangeable:
  - **Join-based** (a single query with a `JOIN`) — one round trip, but a 1:N join multiplies the parent rows, so hydrating 100 parents with 50 children each transfers 5,000 rows and the framework deduplicates in memory. Fine for 1:1 and small collections.
  - **Batched select** (a second query with `WHERE parent_id IN (...)`) — two round trips, no row multiplication. Better for large collections, and the right default for nested relations.
- Nested eager loading of several 1:N relations in one join is a cartesian product: two collections of 50 each produce 2,500 rows per parent. Split into separate batched selects.
- The `IN (...)` list has a limit: PostgreSQL's protocol caps bind parameters at 65,535 per statement, and huge lists plan badly regardless. Most ORMs chunk automatically — verify yours does, and at what size.
- A query count that scales with result size is the signature. If 10 rows issue 11 queries and 100 rows issue 101, no amount of index tuning will help.

## Over-Fetching

- `SELECT *` is the ORM default because it must hydrate a full entity. It blocks index-only scans and drags large columns (`TEXT`, JSON blobs) across the wire on every read (SKILL.md Traps).
- Use the ORM's projection facility (select specific columns, or a DTO/tuple query) on hot read paths and on any table with a large column.
- Defer or exclude big columns at the mapping level so they load only when touched.
- Loading an entity to update one field reads the whole row, hydrates an object, and writes every column back. For a targeted change, issue the `UPDATE` directly — it is also atomic, where read-modify-write is not.
- Counting by loading a collection and taking its length transfers every row to count them. Use the ORM's `count` method, which emits `COUNT(*)`.

## Pagination

- ORM `.offset(n).limit(m)` maps to SQL `OFFSET`, which reads and discards every skipped row. Deep pages degrade linearly.
- Keyset pagination usually requires dropping to a raw or expression predicate, because most ORMs cannot express a row-value comparison. The trade is worth it for infinite scroll and any API that iterates a full table.
- Ordering must include a unique tiebreaker or rows are skipped and duplicated across pages — a bug that surfaces as "the export is missing records" long after release.
- Counting the total for a page number costs a second full aggregate. Offer "next page" instead of "page 7 of 93" where the product allows it.

## Bulk Operations

- Saving N entities in a loop issues N statements and N round trips. Use the ORM's bulk insert, which emits a multi-row `INSERT` — typically an order of magnitude faster for the same rows.
- ORM-level `update_all`/`delete_all`-style methods bypass callbacks, validations, and the identity map. That is the point (they are one statement) and the risk (audit hooks and cache invalidation do not run). Decide explicitly, per call.
- Bulk statements over very large sets need chunking for the same reason hand-written ones do: one transaction holding millions of rows bloats WAL/undo.
- After a large bulk load through the ORM, `ANALYZE` still matters — the planner does not know about the ORM.

## Transaction Boundaries

- "Transaction per request" middleware makes template rendering, serialization, and any outbound HTTP call part of the transaction — locks held for the entire request (SKILL.md rule 5).
- Nested `transaction do ... end` blocks usually map to savepoints, not real nested transactions. An inner rollback may leave the outer transaction alive and the object graph inconsistent with the database.
- After-commit hooks are the correct place for side effects. A hook that fires inside the transaction sends the email for a transaction that then rolls back.
- Connection-per-transaction matters under a transaction-mode pooler: session state set through the ORM (`SET`, advisory locks, temp tables) does not survive between statements.
- The ORM's optimistic-locking column is checked by row count; ignoring the "0 rows affected" result discards a user's edit with no error.

## Lazy Loading Outside the Session

- Accessing a relation after the session, unit of work, or request has closed raises an error in strict ORMs and issues a surprise query in permissive ones — including inside a template or a serializer, where nobody is looking.
- Fix by loading everything the response needs before the boundary, and by configuring the ORM to raise on lazy loads in test and development so the failure appears at authoring time.
- Serializers are the usual culprit: a serializer that walks associations turns one endpoint into an unbounded query count that varies by payload.

## Type and Parameter Mismatches

- A driver that binds a string where the column is an integer forces an implicit cast on the column and disables its index (SKILL.md Traps). It looks like a missing index and no index will fix it.
- Enum mapping: an ORM enum stored as an integer sorts and filters by the declaration order, not by label. Renumbering the enum reinterprets every stored row, with nothing to warn you.
- `NULL` versus an empty string differ in the database but are often conflated by web frameworks binding empty form fields.
- Timestamps: the ORM may send local time while the column expects UTC. Set the connection timezone explicitly and store UTC.
- Decimal columns bound as floats lose precision before the database ever sees the value; map money columns to the language's decimal type.

## Pool Configuration

- Pool size is per process. Total connections = instances × workers per instance × pool size — that product is what must fit the server's limit, and it is routinely 10× what anyone intended.
- Sizing follows the same rule as any pool: throughput peaks near `cores × 2` active connections at the database, not at the application.
- Set `max_lifetime` below any infrastructure idle timeout (load balancer, NAT gateway, managed-database idle cutoff), or connections die mid-query.
- Set an acquisition timeout so pool exhaustion fails fast and visibly instead of piling up waiters that look like database slowness.
- Instrument pool wait time. Rising wait time with flat query time means the pool is the bottleneck, and adding database capacity will do nothing.

## Migration Autogeneration

- Autogenerated migrations are a draft. Read every one before committing: they miss data backfills, generate destructive operations for renames (drop plus add loses the data), and reorder operations in ways that break dependencies.
- They also generate DDL with no `lock_timeout` and no online variant, which is what turns a routine deploy into an outage on a large table.
- A model rename produces a drop-and-create. The correct sequence is expand-migrate-contract, written by hand (SKILL.md rule 8).
- Index definitions in ORM models frequently drift from the database. Diff the checked-in schema dump against the migrated schema in CI.

## When to Drop to SQL

Use a raw or hand-written query when the ORM cannot express it, and keep the boundary explicit:

- Window functions, recursive CTEs, `LATERAL` joins, `DISTINCT ON`, set operations with ordering — usually inexpressible or unreadable in the query builder.
- Reporting aggregations, upserts with conditional updates, `SKIP LOCKED` queue pulls.
- Anything where the emitted plan matters and you need control over the exact statement.

Rules for raw SQL through an ORM: always parameterized, never string-interpolated; returned as a plain row/DTO rather than a partially-hydrated entity; and kept in one place (a repository or query object) rather than scattered through controllers.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Judging performance by per-query timing | Every query is fast; the count is the problem | Count queries per request and assert on it |
| Eager loading everything, everywhere | Turns N+1 into one enormous cartesian join | Load the relations the response uses, batched for collections |
| Loading an entity to change one field | Reads the whole row, races other writers | Targeted `UPDATE` statement |
| Trusting an autogenerated migration | Renames become drop-and-create; no lock protection | Read and rewrite before committing |
| Raising pool size to fix timeouts | Multiplies connections against a server limit | Measure pool wait time; size from database cores |
| `update_all` on a table with audit callbacks | Bypasses callbacks and validations, with no error | Decide per call; document which hooks are skipped |
| Serializing associations lazily | Query count varies with payload shape and is invisible in tests | Preload before the serializer; raise on lazy loads in tests |
| `.raw()` with string interpolation | Injection, straight through the ORM's protections | Parameter binding, always |
