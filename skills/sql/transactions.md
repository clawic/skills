# Transactions, Isolation, and Locking

Concurrency bugs pass every single-user test. The defense is knowing which anomaly your isolation level still allows and closing that one hole explicitly.

Contents: Isolation Levels · Anomalies · Engine Defaults · Explicit Locks · Lock Ordering · Deadlocks · Retry Loops · Optimistic Locking · Idempotency · Advisory Locks · Long Transactions · Transaction Boundaries

## Isolation Levels and What They Still Allow

| Level | Dirty read | Non-repeatable read | Phantom | Write skew |
|---|---|---|---|---|
| READ UNCOMMITTED | Possible (not in PostgreSQL — it behaves as READ COMMITTED) | Possible | Possible | Possible |
| READ COMMITTED | No | Possible | Possible | Possible |
| REPEATABLE READ | No | No | No in PostgreSQL and InnoDB (MVCC snapshot); allowed by the standard | Possible |
| SERIALIZABLE | No | No | No | No |

Each statement in READ COMMITTED takes a fresh snapshot: a multi-statement report can see two different states of the database and produce totals that never existed. Reports that must be internally consistent run in REPEATABLE READ.

## The Anomalies, With Their Real Shapes

- **Lost update**: two sessions read `balance = 100`, both write `balance - 10`, the result is 90 instead of 80. Read-modify-write in application code always has this bug unless locked or made atomic.
- **Non-repeatable read**: the same `SELECT` in one transaction returns different values because another committed in between. Breaks "check then act".
- **Phantom**: a range query returns new rows on re-execution. Breaks "count then enforce a maximum".
- **Write skew**: two transactions each read a set, each verifies an invariant, each writes a *different* row, and together they break the invariant. The classic case: two doctors both cancel their on-call shift because each sees the other still on call. No level below SERIALIZABLE prevents it — and it is the anomaly people assume REPEATABLE READ covers.

## Engine Defaults (know yours before reasoning)

| Engine | Default level | Note |
|---|---|---|
| PostgreSQL | READ COMMITTED | SERIALIZABLE uses SSI: no extra locks, but transactions abort with `40001` |
| MySQL / InnoDB | REPEATABLE READ | Uses next-key (gap) locks, so it blocks phantoms by locking ranges — more deadlocks than PostgreSQL at the same level |
| MariaDB | REPEATABLE READ | As InnoDB |
| SQLite | SERIALIZABLE | One writer at a time; `PRAGMA journal_mode=WAL` lets readers proceed during a write |
| SQL Server | READ COMMITTED (lock-based) | Readers block writers unless `READ_COMMITTED_SNAPSHOT ON` — turning it on is usually the single biggest concurrency win on SQL Server |

## Explicit Row Locks

```sql
-- Lock the rows you are about to modify; other writers wait
SELECT * FROM inventory WHERE product_id = 5 FOR UPDATE;

-- Weaker: prevents deletion/key change, allows concurrent non-key updates
SELECT * FROM users WHERE id = 5 FOR SHARE;               -- FOR KEY SHARE in PostgreSQL

-- Fail immediately instead of queueing (interactive paths, health checks)
SELECT * FROM jobs WHERE id = 5 FOR UPDATE NOWAIT;

-- Skip contended rows (queues; each worker gets a disjoint set)
SELECT id FROM jobs WHERE status = 'pending'
ORDER BY created_at LIMIT 1 FOR UPDATE SKIP LOCKED;
```

- `FOR UPDATE` locks rows the query *returns*. In READ COMMITTED, a row modified after your snapshot is re-read at the new version — your `WHERE` may no longer be true for it. Re-check the condition after acquiring the lock.
- `FOR UPDATE` combined with `LEFT JOIN` locks rows from the outer table too unless you scope it (`FOR UPDATE OF t`).
- `SKIP LOCKED` changes semantics, not just performance: the query is no longer "the oldest pending job", it is "the oldest one nobody else holds". That is correct for queues and wrong for reports.
- Row locks are released only at COMMIT or ROLLBACK — never at the end of the statement.

## Lock Ordering (the deadlock prevention that actually works)

Two transactions locking `{1,2}` and `{2,1}` deadlock; both locking `{1,2}` queue. Impose a total order everywhere rows are locked together:

```sql
SELECT * FROM accounts WHERE id IN (:a, :b) ORDER BY id FOR UPDATE;
```

The order must be consistent across *code paths*, not just within one function — a transfer routine and a batch-settlement job locking the same two accounts in different orders is the standard production deadlock.

Additional ordering hazards:

- Parent/child: inserting a child takes a lock on the parent row for FK validation. A job that updates parents and another that inserts children touch the same rows in opposite order.
- Index maintenance: two inserts with no row in common can still contend on the same index page or, in InnoDB, the same gap.
- Upserts: `INSERT ... ON CONFLICT` may take locks in key order determined by the values, so batches should be sorted by key before insert.

## Reading a Deadlock Report

- PostgreSQL logs both statements and the lock each was waiting on. The victim is chosen by cost; it is not necessarily the guilty one.
- MySQL: `SHOW ENGINE INNODB STATUS`, section `LATEST DETECTED DEADLOCK` — it holds only the most recent one, so capture it immediately.
- The two statements shown are where the transactions *waited*, not where they took their first lock. Read the whole transaction, not the reported line.
- Deadlock frequency rising with load and no shared rows → gap locks (InnoDB REPEATABLE READ) or missing indexes forcing wide range locks. An index that narrows the scan narrows the locks.

## The Retry Loop (non-optional at SERIALIZABLE)

Serialization failures and deadlocks are expected outcomes, not exceptions to log and drop.

```
attempt = 0
while attempt < 3:
    begin
    try:
        do work; commit; break
    except sqlstate in ('40001', '40P01'):   # serialization failure, deadlock
        rollback
        attempt += 1
        sleep(random jitter, e.g. 50ms × 2^attempt ± 25%)
    except other:
        rollback; raise
```

- Retry only the whole transaction. Retrying one statement inside a rolled-back transaction executes against nothing.
- The work must be idempotent or wholly inside the transaction — a retry that re-sends an email is worse than the original failure.
- Cap attempts (3 is a reasonable default; a house retry cap is a Thresholds preference) and jitter the backoff; synchronized retries recreate the same collision.
- Without this loop, SERIALIZABLE is strictly worse than REPEATABLE READ: same cost, plus failures you do not handle.

## Optimistic Locking (no database lock at all)

```sql
UPDATE documents
SET body = :body, version = version + 1
WHERE id = :id AND version = :expected_version;
-- 0 rows affected = someone else won; re-read and re-apply or surface a conflict
```

- Correct choice when conflicts are rare and the "transaction" spans a user thinking (an edit form open for minutes). Never hold a database lock across user time (SKILL.md rule 5).
- The application must check the affected-row count. Ignoring it discards the user's edit with no error — the most common implementation bug.
- A `updated_at` timestamp works as the version only if its resolution exceeds the update rate; an integer counter has no such failure.

## Idempotency and Exactly-Once Writes

- Give externally triggered writes a client-supplied idempotency key with a unique constraint: the second attempt violates the constraint, and the handler returns the first result.
- `INSERT ... ON CONFLICT DO NOTHING RETURNING id` returns no row when the insert was a no-op — fetch the existing row explicitly rather than treating the empty result as failure.
- Database transactions do not extend to external systems. Commit the database first and drive the external effect from a durable outbox row, or accept a window where one succeeded and the other did not.
- Retries of a batch must be safe at any prefix: chunk the work and record progress in the same transaction as the work.

## Advisory / Application Locks

```sql
SELECT pg_advisory_xact_lock(hashtext('nightly-rollup'));  -- released at commit
SELECT GET_LOCK('nightly-rollup', 10);                     -- MySQL, with timeout
```

- Use for "only one worker runs this job", cron singletons, and serializing a migration across instances — cases where no row represents the resource.
- Prefer the transaction-scoped variant (`pg_advisory_xact_lock`). Session-scoped locks survive commit and leak when a connection is returned to a pool still holding one.
- Advisory locks live in a flat namespace of integers: hash a descriptive string and keep the list of names in one place, or two features end up sharing a lock.
- Under a transaction-mode pooler, session-level advisory locks are unusable.

## Long Transactions Are a Systemic Cost

- They hold row locks, block DDL, and in PostgreSQL pin dead tuples so nothing can be vacuumed database-wide — one forgotten `BEGIN` bloats every table.
- `idle in transaction` is worse than a long-running query: it holds everything while doing nothing. Set `idle_in_transaction_session_timeout` on the application role.
- The monitoring threshold is the canonical one: investigate any transaction open past 1 minute.
- A transaction that spans an HTTP call inherits that service's worst-case latency as its lock duration.

## Where Transaction Boundaries Belong

- One transaction per unit of business meaning. Framework "transaction per request" middleware makes every slow template render part of the transaction.
- Do not open a transaction to run a single statement — it is already atomic.
- Read-only work does not need a transaction unless it needs a consistent multi-statement snapshot; then declare it (`SET TRANSACTION READ ONLY` also lets the engine skip work).
- Nested transactions do not exist in most engines: an inner `BEGIN` is ignored or errors. Partial rollback needs savepoints, and a savepoint per loop iteration has real cost in PostgreSQL (each consumes a subtransaction slot; past 64 open subtransactions per session, other backends pay a lookup penalty).
- DDL inside a transaction is safe in PostgreSQL, SQLite, and SQL Server; MySQL commits implicitly (SKILL.md Traps).
