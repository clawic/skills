# Change Streams — Reacting to Writes

A change stream is a tailing cursor over the oplog with a resumable position. It replaces polling for changes, database triggers, and most homemade CDC. It requires a replica set — a standalone `mongod` cannot serve one (→ `transactions.md` for the local single-node setup).

## The Shape That Survives a Restart

```javascript
const pipeline = [{$match: {"fullDocument.status": "paid", operationType: {$in: ["insert", "update"]}}}];
const opts = {fullDocument: "updateLookup"};
if (savedToken) opts.startAfter = savedToken;      // startAfter, not resumeAfter — see below

const stream = coll.watch(pipeline, opts);
for await (const change of stream) {
  await handle(change);                            // must be idempotent
  await saveToken(change._id);                     // persist AFTER the work, never before
}
```

- **Persist the resume token after the side effect, not before.** Token-then-work loses events on a crash; work-then-token repeats them. Repeats are survivable if the handler is idempotent; losses are not.
- `startAfter` resumes past an invalidate event (a dropped or renamed collection); `resumeAfter` fails on one. Use `startAfter` unless you specifically want the stream to die on invalidation.
- Store the token as the opaque document it is. Do not parse it, compare it, or reconstruct it — its internals are not a contract.

## The Resume Window Is the Oplog Window

- A consumer offline longer than the oplog retains history cannot resume: its token points at an entry that no longer exists, and the only recovery is a full re-read of current state plus a fresh stream.
- This makes consumer downtime an oplog-sizing question, not an application question. Size the oplog against your longest tolerable consumer outage, the same number that governs secondary resync (→ `replication.md`).
- Alarm on consumer lag (now minus the token's cluster time) at a fraction of the oplog window, not at a fixed number of minutes.

## fullDocument and Its Trap

- By default an `update` event carries only the delta (`updateDescription`), not the document.
- `fullDocument: "updateLookup"` performs a fresh read of the document AT LOOKUP TIME, which may already reflect LATER changes. It is not the state at the moment of the change. For an audit log, that difference is a correctness bug.
- `fullDocument: "whenAvailable"` / `"required"` plus `fullDocumentBeforeChange` (MongoDB >=6.0) give true before and after images — but only if the collection has pre- and post-images enabled with `changeStreamPreAndPostImages: {enabled: true}`, which costs storage per change.
- `delete` events never carry the document, in any configuration. If your consumer needs to know what was deleted, soft-delete with a flag and let a TTL index do the real removal later (→ `indexes.md`).

## Scope and Filtering

- Watch a collection (`coll.watch()`), a database (`db.watch()`), or the whole deployment (`client.watch()`). Broader scope means more events to filter and a bigger blast radius when a consumer falls behind.
- The pipeline runs on the SERVER: filtering there costs nothing on the wire, filtering in the consumer costs the whole stream. Only `$match`, `$project`, `$addFields`, `$replaceRoot`, `$redact` and a few others are allowed.
- Matching on `fullDocument.*` requires `fullDocument` to be requested — otherwise the filter silently matches nothing on updates.
- `operationType` values worth handling explicitly: `insert`, `update`, `replace`, `delete`, `drop`, `rename`, `invalidate`. Treating `replace` as `update` is the usual oversight, and `replaceOne` is common in ODM-generated code (→ `connections.md`).

## Delivery Semantics

- At-least-once, in order per document, from the position the token names. There is no exactly-once; build the handler around that.
- Idempotency key: `documentKey._id` plus `clusterTime`, or a version field in the document itself. A handler that reprocesses the last event after a restart must produce the same result.
- Transaction writes arrive together after commit, sharing a `txnNumber` and `lsid`, and never partially (→ `transactions.md`).
- On a sharded cluster the stream merges shards through mongos and orders by cluster time — correct, but one slow shard slows the whole stream.
- One consumer per stream position. Two processes with the same token both do the work; scaling out means partitioning by a document attribute in the `$match`, not running duplicates.

## What It Is Good For

| Use | Shape |
|---|---|
| Cache invalidation | Watch the collection, evict by `documentKey._id`. Simplest correct use |
| Search index sync | Feed the external engine; this is exactly how Atlas Search stays current (→ `search.md`) |
| Outbox / event publishing | Write the business change and its event in ONE document write, stream it out. Removes the dual-write problem without a transaction |
| Materialized read models | Update the projection incrementally instead of rebuilding it on a schedule (→ `aggregation.md`) |
| Audit trail | Needs pre/post images enabled; `updateLookup` alone produces an audit log that is subtly wrong |
| Cross-service notification | Publish to a real broker from the consumer. Do not let ten services watch the same collection |

## What It Is Not

- Not a job queue. A queue needs claim, retry, visibility timeout and dead-lettering; a change stream has none of those. Use `findOneAndUpdate` on a jobs collection (→ `transactions.md`) or a real broker.
- Not a replacement for replication or backups.
- Not free: each open stream is a cursor and a share of the primary's attention. Dozens of watchers on a hot collection is a load pattern, not a design.
- Not available on collections you have not thought about: time-series collections emit at bucket granularity, not per measurement (→ `time-series.md`).

## Operating a Consumer

- Run it as a supervised long-lived process. Reconnection is the normal case: elections, network blips, and rolling restarts all break the cursor, and the retry path is "reopen with `startAfter`".
- Set `maxAwaitTimeMS` deliberately; the default keeps the cursor waiting for new data, which is what you want for a tail and not what you want for a batch job that should exit.
- Instrument three numbers: events processed, handler failures, and lag behind the oplog head. The third is the one that predicts the unrecoverable failure.
- On Atlas, triggers are managed change-stream consumers with the same semantics and the same resume-window limits — the constraints here still apply (→ `atlas.md`).
