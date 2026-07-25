# Testing SQL and Migrations

Most database test suites verify that the ORM works. The failures that reach production are constraint violations under concurrency, migrations that lock, and queries whose plan collapses at real data volumes — none of which a green suite on 20 seeded rows detects.

Contents: What To Test · Test Database · Isolation Strategies · Fixtures · Determinism · Constraint Tests · Migration Tests · Performance Tests · Data Quality Checks · CI · Traps

## What Is Worth Testing

| Target | Why | Shape of the test |
|---|---|---|
| Constraints and defaults | They are the last line of defense and get lost in migrations unnoticed | Insert violating rows, assert the error |
| Complex queries (aggregates, window functions, recursive CTEs) | Off-by-one and fan-out bugs are invisible in review | Fixed input rows, exact expected output |
| Migrations | The only code that runs once and cannot be retried | Apply, assert schema, apply to a production-shaped copy |
| Uniqueness under concurrency | Check-then-insert races pass every serial test | Two connections, one assertion |
| Query plans on hot paths | Regression from an added join or dropped index | Assert the plan does not contain a sequential scan on a large table |
| The dialect the code actually targets | Test-engine leniency masks target-engine failures | Run against the target engine |

Not worth testing: that `SELECT * FROM users` returns rows, that the ORM maps a column, or the engine's own behavior.

## The Test Database

- Test against the **same engine and major version** as production. SQLite standing in for PostgreSQL accepts statements PostgreSQL rejects, enforces less, and has different NULL ordering and case sensitivity — the substitution hides exactly the bugs a database test exists to catch.
- Create the schema by **running the migrations**, not by loading a snapshot. If the suite loads a dump, nothing ever tests the migration path, and the two drift until a deploy fails.
- Keep one canonical schema dump under version control anyway, regenerated from the migrations, and diff it in CI. A migration that produces a different schema than the checked-in dump is a review finding.
- One database per parallel worker (or one schema per worker) — shared state across parallel tests produces flakes that look like race conditions in the application.
- Never point a test suite at a database that also holds real data. A truncation helper with the wrong connection string is the classic way to lose production.

## Isolation Between Tests

| Strategy | Speed | Limitation |
|---|---|---|
| Transaction per test, rolled back at the end | Fastest | Cannot test anything that commits, and nested transaction behavior in the code under test conflicts with the wrapper |
| `TRUNCATE` the touched tables between tests | Fast enough, always correct | Must reset sequences (`TRUNCATE ... RESTART IDENTITY`) or ids leak across tests |
| Recreate the schema per test | Correct, very slow | Only for migration tests |
| Template database / snapshot restore | Fast and complete | Setup complexity; PostgreSQL `CREATE DATABASE ... TEMPLATE` is the common form |

Default: transaction-rollback for the bulk of the suite, truncation for tests that must commit (concurrency, triggers on commit, anything checking what another connection sees).

## Fixtures

- Build entities through factories that fill required fields with valid defaults and let each test override only the field it is about. A test that constructs 15 fields to assert on one is unreadable and breaks on every schema change.
- Insert the minimum: two rows to prove a filter, three to prove ordering. Large fixture files make every test slower and none clearer.
- Never share mutable fixture rows across tests, and never depend on fixture ids being specific numbers.
- Test data must be obviously fake (`user-1@example.test`) so it is recognizable if it ever escapes into a real system.
- Production data as fixtures is a data-protection incident waiting to happen; if realistic distributions are required, generate them or mask a copy inside the restore job.

## Determinism

Flaky database tests almost always come from one of these:

- `SELECT` without `ORDER BY`: row order is undefined and changes after any write or vacuum. Order every assertion's query by a unique column (SKILL.md Traps).
- `NOW()`/`CURRENT_DATE` in fixtures or assertions: freeze time in the application, or assert on ranges, not equality. Tests that only fail near midnight or on the last day of a month are this.
- Depending on generated ids: sequences do not reset with a rollback, so ids differ between runs.
- Locale or timezone of the test machine differing from CI: pin both explicitly.
- Floating-point equality: compare with a tolerance, or use exact decimal types.

## Testing Constraints and Concurrency

```sql
-- The constraint exists and bites: expect a unique violation
INSERT INTO users (email) VALUES ('a@example.test');
INSERT INTO users (email) VALUES ('a@example.test');   -- assert SQLSTATE 23505

-- The FK actually blocks the orphan: expect 23503
INSERT INTO orders (user_id) VALUES (999999);

-- Soft-delete uniqueness is scoped to live rows only
-- (insert, soft-delete, re-insert the same email → must succeed)
```

Concurrency tests need two real connections; a single connection cannot produce a lock conflict with itself. The minimal shape: connection A opens a transaction and takes the lock, connection B attempts the conflicting write with a short `lock_timeout` and asserts either the timeout or the expected serialization failure. Keep these few and targeted — they are slow and easy to make flaky.

## Testing Migrations

Every migration gets four checks before it is allowed near production:

1. **Applies cleanly** to a database at the previous revision.
2. **Reverses cleanly** if the project keeps down migrations — and if it does not, that is a stated decision, not an omission.
3. **Applies to production-shaped data**: a restored copy (masked) or a generated dataset at production row counts. A migration tested on 100 rows tells you nothing about lock duration on 100 million.
4. **Is compatible with the previous application version**, because during a deploy both run at once. This is what expand-migrate-contract exists for (SKILL.md rule 8), and the test is simply running the previous version's test suite against the new schema.

Additional checks worth automating: the migration does not exceed a stated duration on the production-shaped copy; it sets `lock_timeout`; and for MySQL, it contains a single DDL statement, since a multi-statement migration cannot roll back there.

## Performance Regression Tests

- Assert on the **plan**, not on wall time. Wall time on shared CI hardware is noise; "the plan for this query contains no sequential scan on `orders`" is stable.
- Generate a dataset at a realistic order of magnitude for the handful of queries that matter, and `ANALYZE` it so the planner has honest statistics.
- Track index usage over time in production instead of trying to test it: a new index whose `idx_scan` stays at zero after a week is dead weight.
- Catch N+1s structurally: assert the number of queries a request issues, which is the only reliable detector because each individual query is fast.

## Data Quality Checks in Production

Tests cover code; assertions cover data. Run these as scheduled queries on the user's stated Cadence (default: hourly for money, daily for the rest) and alert on non-zero results:

```sql
-- Orphans that a missing or disabled FK allowed in
SELECT COUNT(*) FROM orders o LEFT JOIN users u ON u.id = o.user_id WHERE u.id IS NULL;

-- Duplicates on a key that should be unique but is not constrained yet
SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;

-- Invariants the schema cannot express
SELECT COUNT(*) FROM accounts WHERE balance < 0;
SELECT COUNT(*) FROM bookings WHERE end_at <= start_at;
```

Each check that fires twice earns a real constraint. The check is the interim measure; the constraint is the fix.

## In CI

- Run the database as a service container pinned to the production major version; a floating `latest` tag turns an upstream release into a mystery failure.
- Order: apply migrations → assert the schema matches the checked-in dump → run the suite → run the data-quality queries against the seeded database.
- Fail the build on a migration that is not accompanied by its schema-dump update.
- Keep the whole suite fast enough to run on every commit; the moment it is not, people stop running it and the database tests are the first to be skipped.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| SQLite in tests, PostgreSQL in production | Different type enforcement, case sensitivity, NULL ordering, and constraint behavior | Same engine and major version as production |
| Loading a schema dump instead of running migrations | The migration path is never exercised until deploy day | Migrate up from empty; diff against the dump |
| Transaction-rollback isolation for everything | Skips anything that must commit, with no failure, including trigger and concurrency tests | Truncation strategy for those tests |
| Asserting on ids | Sequences do not roll back; ids vary between runs | Assert on business keys |
| `SELECT` in an assertion without `ORDER BY` | Passes until row order changes | Deterministic ordering with a unique tiebreaker |
| Migration tested only on an empty database | Says nothing about lock time or backfill duration | Production-shaped copy |
| Production data copied into test fixtures | Uncontrolled personal data outside production | Generated or masked data |
| Timing assertions on CI hardware | Noise, then a disabled test | Assert on plans and query counts |
