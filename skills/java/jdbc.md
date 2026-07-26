# JDBC — Connections, Pools, Batching, Transactions

The query itself belongs to the `sql` skill; this is what the Java side does wrong around it. ORM-specific behavior (lazy loading, N+1, `@Transactional`) is in `spring.md`.

## Connection Pool Sizing

- A connection is expensive to create and cheap to borrow: always a pool (HikariCP is the default choice and Spring Boot's).
- **Small pools beat large pools.** The database is the constrained resource; queueing in the client is cheaper than thrashing the server. A common starting point is `cores × 2 + effective_spindles` on the DATABASE host — for an 8-core database, on the order of 20 connections, shared across all application instances. Measure and adjust; a pool of 200 is almost always a mistake made in a hurry.
- Total connections across every instance, plus migration tools, plus admin sessions, must stay under the server's `max_connections`. Instances × pool size is the number that matters, and it grows every time someone scales out.
- `connectionTimeout` is how long a caller waits for a free connection before failing — keep it well below the caller's own timeout so the failure is a clear pool-exhaustion error, not a mysterious hang (`async.md`).
- Set `maxLifetime` shorter than any network idle timeout in front of the database (firewall, proxy, or the server's own `wait_timeout`), or the pool hands out connections the network already dropped: "connection reset by peer" on the first query, intermittently.
- Symptom of exhaustion: threads all parked in `getConnection` in a thread dump (`debug.md`). Causes, in order: a leaked connection, a long transaction holding one across an HTTP call, or nested `REQUIRES_NEW` transactions (`spring.md`).

## Leaks and Closing

- `Connection`, `Statement`, and `ResultSet` are all `AutoCloseable`; nest them in one try-with-resources (SKILL.md rule 4). Closing the connection returns it to the pool — it does not close the socket.
- Closing a `Connection` closes its statements and result sets, but only when the connection actually closes; in a pool, an unclosed `Statement` accumulates on a live connection until the driver's cursor limit is hit ("maximum open cursors exceeded").
- HikariCP's `leakDetectionThreshold` (e.g. 30000 ms) logs a stack trace for any connection held longer than that — enable it the moment you suspect a leak, and leave it on in non-production.
- A connection returned with an open transaction poisons the next borrower. Commit or roll back on every path, or let the pool's `autoCommit`/rollback-on-return policy handle it.

## Statements and Parameters

- `PreparedStatement` with `?` placeholders for every value: it is the injection fix and the performance fix (server-side plan reuse) at once (`security.md`).
- Setting a null: `setObject(i, value, Types.VARCHAR)` — `setObject(i, null)` fails on several drivers because the type cannot be inferred.
- Time and date: pass `Instant`/`LocalDate` via `setObject` (JDBC 4.2+) instead of the `java.sql.*` wrappers; the legacy types apply the JVM's default zone (`datetime.md`).
- `getInt()` returns 0 for SQL NULL — check `rs.wasNull()` immediately after, or read into an object type. Silent zeros in aggregates come from here.
- Reading by column label is refactor-safe; reading by index is faster and unreadable. Prefer labels outside hot loops.
- `LIKE` with user input needs escaping of `%` and `_`, or a search box becomes a full-table scan.

## Batching and Fetching

```java
try (var ps = conn.prepareStatement(sql)) {
    for (var row : rows) {
        bind(ps, row);
        ps.addBatch();
        if (++n % 1000 == 0) ps.executeBatch();     // flush in chunks; a single giant batch is a memory spike
    }
    ps.executeBatch();
}
```

- Batching turns N round trips into one. The win is proportional to network latency, so it is largest exactly where it matters (a database on another host).
- Driver flags decide whether batching is real: MySQL needs `rewriteBatchedStatements=true` to combine inserts; without it the batch is still N statements on the wire.
- `executeBatch` returns per-statement counts and throws `BatchUpdateException` on failure — the exception tells you how many succeeded, which matters because the batch may be partially applied.
- Streaming a large result set is not the default: the driver usually buffers the whole result in memory. PostgreSQL requires `setFetchSize(n)` **and** autoCommit off; MySQL requires `setFetchSize(Integer.MIN_VALUE)` with a forward-only result set. Getting this wrong is a textbook OOM on a "simple query" (`memory.md`).
- Never `SELECT *` into a `List` of unknown size. The row count is the user's, not yours.

## Transactions at the JDBC Level

- `setAutoCommit(false)` starts the unit of work; `commit()`/`rollback()` end it; restore autoCommit before returning the connection to the pool.
- Isolation levels are a database decision with Java consequences: `READ_COMMITTED` is the usual default, and code that reads-then-writes without `SELECT ... FOR UPDATE` or an optimistic version column has a lost-update race regardless of the language.
- Keep transactions short and free of network calls. A transaction open across an HTTP request holds a connection and a row lock for the remote service's latency.
- Retryable failures are database-specific: serialization failures and deadlock victims (SQLState `40001`, `40P01`) are safe to retry with backoff; constraint violations (`23xxx`) are not (`async.md`).
- Inspect `SQLException` properly: `getSQLState()` (portable-ish) and `getErrorCode()` (vendor) beat string-matching the message, and `getNextException()` holds the real cause for chained driver errors.

## Drivers and Diagnostics

- The driver jar is `runtimeOnly`/`runtime` scope: your code should never import a vendor class (`build.md`).
- `Class.forName("...Driver")` has been unnecessary since JDBC 4 (SPI auto-registration) — and it fails after shading strips `META-INF/services`, which is why "No suitable driver found" shows up only in the packaged jar (`debug.md`).
- A JDBC URL contains credentials in many formats: it appears in logs, exception messages, and `jcmd VM.system_properties`. Keep it out of both (`security.md`).
- Slow-query visibility: enable the driver's own logging (`loggerLevel`, `profileSQL`) or wrap the `DataSource` with a proxy that times statements. Guessing which query is slow from application timings wastes a day.
- Validate the pool's health, not the database's: a `/health` check that borrows a connection and runs `SELECT 1` reports pool exhaustion, which is what actually breaks.
