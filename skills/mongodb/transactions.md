# Transactions — Sessions, Retry Loops, and When Not To

Start from SKILL.md rule 6: single-document atomicity is the primitive. A transaction is what you reach for when an invariant genuinely spans documents you could not co-locate. Most transactions in real codebases are joins someone was uncomfortable doing without one.

## Prerequisites That Surprise People

- Transactions require a replica set (MongoDB >=4.0) or a sharded cluster (>=4.2). A standalone `mongod` rejects them — which is why they work in production and fail on a developer's laptop.
- Local fix: run the local instance as a single-node replica set (`--replSet rs0`, then `rs.initiate()` once). This is the correct local setup for any project that uses transactions or change streams, and it costs nothing.
- Every operation inside the transaction must be passed the session. An operation that forgets it runs OUTSIDE the transaction, commits immediately, and is invisible in review — the most common transaction bug there is.

## The Only Correct Shape

```javascript
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    await accounts.updateOne({_id: from, balance: {$gte: amount}}, {$inc: {balance: -amount}}, {session});
    await accounts.updateOne({_id: to}, {$inc: {balance: amount}}, {session});
    await ledger.insertOne({from, to, amount, at: new Date()}, {session});
  }, {readConcern: {level: "snapshot"}, writeConcern: {w: "majority"}});
} finally {
  await session.endSession();
}
```

- `withTransaction` is not a convenience wrapper — it implements BOTH required retry loops. Hand-rolled `startTransaction`/`commitTransaction` code that lacks them is broken under exactly the conditions transactions exist for.
- The callback must be idempotent, because it WILL be re-executed. Anything with a side effect outside MongoDB (sending an email, charging a card, publishing to a queue) must not live inside it.
- The guard belongs in the filter (`balance: {$gte: amount}`), not in a read-then-write in application code. Read-check-write inside a transaction is still a race you solved the expensive way.

## The Two Retry Loops

| Label on the error | What it means | What to retry |
|---|---|---|
| `TransientTransactionError` | The whole transaction failed atomically (write conflict, election, lock timeout). Nothing was committed | The ENTIRE transaction, from the first statement |
| `UnknownTransactionCommitResult` | The commit was sent; the outcome is unknown (network blip, primary stepdown mid-commit) | The COMMIT only — commits are idempotent by transaction number |

- Match on the LABEL, never on a code list. Codes change between versions; the labels are the contract (→ `errors.md`).
- Add a retry ceiling and a backoff. An unbounded retry loop on a genuinely conflicting hot document turns a slow endpoint into an outage.
- A `TransientTransactionError` at high frequency is a schema signal: two writers keep colliding on the same document. Reduce contention (shard the counter, queue the work) rather than tuning the retry.

## Limits That Decide the Design

- Default lifetime is 60 seconds (`transactionLifetimeLimitSeconds`). Past it the transaction aborts and its next operation returns error 251. Raising the parameter is almost always the wrong fix: a transaction holding snapshot state for a minute is already pinning WiredTiger history and pressuring the cache (→ `tuning.md`).
- Keep modified documents under 1,000 (SKILL.md rule 6). Large transactions replicate as large oplog entries and land on every secondary at once.
- Do not iterate a large cursor inside a transaction. The snapshot is held for the whole iteration, and the 60s clock is running the entire time.
- No DDL inside a transaction: creating a collection or index is not transactional (implicit collection creation inside a transaction is allowed from 4.4, but relying on it is fragile).
- Transactions on a sharded cluster cost a coordinator round trip per participating shard. Two shards is fine; a transaction spanning every shard is a distributed commit on your critical path.

## Alternatives That Are Usually Better

| Instead of a transaction | Use |
|---|---|
| Update parent and child together | Embed the child so it is one document (→ `schema.md`) |
| Claim a job / hold a seat / take a lock | `findOneAndUpdate` filtering on the current state — the filter IS the lock |
| Keep a counter consistent with rows | `$inc` on the parent, atomic per document, plus a periodic rebuild as a drift backstop |
| Write to the database and publish an event | Outbox: write the event into the SAME document or collection in one write, then a change stream publishes it (→ `change-streams.md`) |
| Update many denormalized copies | `updateMany` and accept eventual consistency, unless the staleness is user-visible (→ `schema.md`) |
| Multi-step business process across services | A saga with compensating actions; a database transaction cannot span services anyway |

## Reading Inside Transactions

- `readConcern: "snapshot"` gives a consistent view of all reads in the transaction, at the cost of pinning history on the server for its duration.
- Reads inside a transaction do not see other transactions' uncommitted writes, and do see their own.
- A transaction's writes are invisible to change streams until it commits, and then arrive together — consumers see the whole transaction or none of it (→ `change-streams.md`).
- Read preference inside a transaction must be `primary`. There is no such thing as a transaction served from a secondary.

## Reviewing Transaction Code

- Does every operation carry `{session}`?
- Is the callback free of external side effects and safe to run twice?
- Is `withTransaction` (or an equivalent implementing both loops) used, rather than a bare commit?
- Is the write concern `majority`? A transaction committed at `w: 1` can still roll back, which defeats the point of using one.
- Is there a bound on retries and a metric counting them? Silent retries hide a contention problem until it is a latency problem.
- Could this be one document instead? Ask last, answer honestly.
