# Migrations — Changing Shape Under Live Traffic

MongoDB has no `ALTER TABLE`, which is not the same as having no migration problem. The schema lives in the documents and in the code that reads them, so a change is a rollout, not a statement.

## Expand → Dual-Write → Backfill → Contract

The only sequence that is safe with a running application:

1. **Expand** — add the new field. Deploy readers that tolerate BOTH shapes (new field if present, old field otherwise). Nothing writes the new field yet.
2. **Dual-write** — deploy writers that populate old AND new. Now every new document is correct and the old ones are not.
3. **Backfill** — batch-update the historical documents (loop below). Idempotent, resumable, paced.
4. **Verify** — `countDocuments({newField: {$exists: false}})` must be 0, and stay 0 for a full cycle including any batch job that writes on its own schedule.
5. **Contract** — deploy readers that only use the new field, then stop writing the old one, then `$unset` it in a final batched pass.

Each step is a separate deploy. Collapsing steps 1 and 2 leaves old readers looking at documents they do not understand for the length of a rolling deploy — which is exactly the window in which nobody is watching.

## The Backfill Loop

```javascript
let last = null;
const BATCH = 1000;                       // config: backfill_batch
while (true) {
  const filter = {legacyField: {$exists: true}, newField: {$exists: false}};
  if (last) filter._id = {$gt: last};
  const docs = db.coll.find(filter).sort({_id: 1}).limit(BATCH).toArray();
  if (docs.length === 0) break;
  const ops = docs.map(d => ({updateOne: {
    filter: {_id: d._id},
    update: {$set: {newField: transform(d.legacyField)}}
  }}));
  db.coll.bulkWrite(ops, {ordered: false});
  last = docs[docs.length - 1]._id;
  sleep(50);                              // pacing: let replication and eviction breathe
}
```

- **Range by `_id`, never `skip`.** `skip` re-walks the collection on every batch and is quadratic (SKILL.md Query Semantics).
- **Resumable by construction**: the filter excludes already-migrated documents, so a crashed run restarts by re-running.
- **Paced**: a backfill with no `sleep` becomes a replication-lag incident and a cache stall (→ `incidents.md`). Watch lag while it runs and raise the sleep until lag stays flat.
- **`ordered: false`** so one bad document does not stop the batch; read `writeErrors[]` afterward and quarantine those ids rather than aborting.
- Never wrap the whole backfill in a transaction. Batches are the unit of atomicity here; the whole job is not.

## Lazy Migration (often better than a backfill)

Migrate on read, with `schema_version` on every document:

- Reader sees `schemaVersion: 1`, transforms in memory, writes back the v2 shape with `schemaVersion: 2`.
- Hot documents migrate themselves in hours under real traffic; a background backfill sweeps the cold tail whenever convenient.
- Cost: the dual-shape read path lives in the code until the last document converts, so keep the counter query in a dashboard or it lives forever.
- This is the default for large collections where a full pass is measured in hours (→ `schema.md`, schema versioning).

## Operations by Kind of Change

| Change | How |
|---|---|
| Add a field with a default | Do not backfill at all if readers can treat "missing" as the default — the cheapest migration is the one you skip |
| Rename a field | Expand/contract; `$rename` in a batched `updateMany` for the backfill step, never in one statement over millions |
| Change a field's type | New field, never in place. Two types in one field breaks every range query while the backfill runs (SKILL.md Query Semantics) |
| Split one collection into two | Dual-write both, backfill, cut readers over, then stop writing the original. `$merge` from an aggregation is the backfill (→ `aggregation.md`) |
| Merge two collections into one | `$merge` with `whenMatched: "merge"` on the target key; verify counts before cutting readers |
| Add a required field to the validator | Backfill first, then `collMod` the validator. The reverse order rejects every update to a legacy document (→ `schema.md`) |
| Add a unique index | Find duplicates with a `$group` count FIRST; a unique build fails at the first duplicate after scanning everything (→ `indexes.md`) |
| Drop a field | Last step of contract, batched `$unset`. Storage returns to WiredTiger, not to the filesystem (→ `incidents.md`) |

## Bulk Loading

- `bulkWrite` with `ordered: false` parallelizes across the batch and does not stop at the first error; `ordered: true` (the default) is sequential and stops. Choose `false` unless the order between operations is semantically required.
- Batch size: start at 1,000 documents and keep each batch's BSON well under 16MB — that limit applies to the whole command, not to one document. Larger batches stop helping quickly and start hurting replication.
- Build indexes AFTER the initial load, not before: loading into an indexed collection pays a B-tree write per index per document. The exception is the unique index that is enforcing correctness during the load.
- `mongoimport --numInsertionWorkers` for file loads; for collection-to-collection, an aggregation with `$out`/`$merge` never leaves the server and beats any client-side copy.
- During a large load, drop the write concern to `w: 1` only if the data is reproducible from source. If a re-run is expensive, keep `majority` and accept the slower load.

## Migration Discipline

- Every migration is a script in version control with a version number, run once, recorded in a `migrations` collection with its timestamp and outcome. A migration that lives in someone's shell history is not a migration.
- Write the reverse before you run the forward. If the reverse is impossible (a lossy type change), that is a fact to state out loud before running, not to discover afterward.
- Test on a restored copy of production, timed. A backfill that takes 20 minutes on a laptop and 9 hours on production is the normal ratio, not a surprise (→ `backups.md`).
- Never run a migration and a deploy in the same change window unless the sequence above requires it — when something breaks you want one variable, not two.
