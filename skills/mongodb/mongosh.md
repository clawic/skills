# mongosh — The Shell Toolkit

The commands that solve real problems, not the basics. `mongosh` is a Node REPL: JavaScript, `async`/`await`, and file loading all work, which is what makes it an operations tool rather than a query box.

## Connect Without Surprises

```javascript
mongosh "mongodb+srv://user@cluster.example.net/?appName=ops-shell"   // prompts for the password; never in shell history
mongosh --eval 'db.serverStatus().connections' "<uri>"                 // one-shot, scriptable
mongosh --quiet --file check.js "<uri>"                                // run a script, no banner
```

- Always set `appName` — it is how you find your own session in `db.currentOp()` during an incident (→ `incidents.md`).
- `--readPreference=secondaryPreferred` for read-only investigation keeps your exploration off the primary's cache.
- `db.getMongo().getURI()` confirms which cluster you are on before you run the destructive thing. Do this every time; the cost is one line.

## Triage One-Liners

```javascript
rs.status().members.map(m => ({name: m.name, state: m.stateStr, lag: m.optimeDate}))
db.currentOp({secs_running: {$gt: 5}}).inprog.map(o => ({op: o.opid, ns: o.ns, secs: o.secs_running, cmd: o.command}))
db.getSiblingDB("admin").aggregate([{$currentOp: {allUsers: true, idleConnections: true}}, {$group: {_id: "$appName", n: {$sum: 1}}}])
db.serverStatus().wiredTiger.cache["tracked dirty bytes in the cache"]
db.getReplicationInfo()          // oplog size and the window in hours
```

## Collection Forensics

```javascript
db.coll.stats()                                   // size, storageSize, index sizes, average doc size
db.coll.getIndexes()                              // what exists
db.coll.aggregate([{$indexStats: {}}])            // what is used — per node, resets on restart
db.coll.findOne()                                 // the actual shape, which is rarely the documented shape
db.coll.aggregate([{$sample: {size: 100}}, {$project: {size: {$bsonSize: "$$ROOT"}}}, {$group: {_id: null, max: {$max: "$size"}, avg: {$avg: "$size"}}}])
```

That last one is the fastest honest answer to "are our documents too big" (SKILL.md rule 2) — `$bsonSize` over a sample beats guessing from `avgObjSize`, which hides the tail.

## Finding the Shape Drift

```javascript
db.coll.aggregate([
  {$sample: {size: 1000}},
  {$project: {fields: {$objectToArray: "$$ROOT"}}},
  {$unwind: "$fields"},
  {$group: {_id: {k: "$fields.k", t: {$type: "$fields.v"}}, n: {$sum: 1}}},
  {$sort: {n: -1}}
])
```

One field appearing with two BSON types is the silent cause of range queries returning nothing (SKILL.md Query Semantics). Run this before designing any index on a collection you did not create.

## Safe Mutation Habits

- Preview before you mutate: run the filter as a `countDocuments` and a `findOne` first. `updateMany` with a wrong filter is not undoable.
- Batch by range rather than one giant statement — the loop pattern lives in `migrations.md`.
- `db.coll.deleteMany({})` and `db.dropDatabase()` have no confirmation. When `destructive_confirm` is on (SKILL.md Configuration), emit the command for review instead of running it.
- Verify a unique index will succeed before building it:

```javascript
db.coll.aggregate([{$group: {_id: "$email", n: {$sum: 1}}}, {$match: {n: {$gt: 1}}}, {$limit: 5}])
```

## Scripting the Shell

- `load("script.js")` or `--file`; `mongosh` exits with a non-zero status if the script throws, which makes it usable in CI.
- Cursor iteration in scripts: `while (cursor.hasNext())` on a huge result set idles the cursor out at 10 minutes (→ `connections.md`). Loop over ranges instead.
- `print()` and `printjson()` for output; the implicit REPL echo does not fire inside a script.
- `EJSON.stringify(doc, null, 2)` preserves types (`ObjectId`, `Date`, `NumberLong`) that plain `JSON.stringify` flattens into strings — the difference matters when the output feeds another script.
- `sleep(ms)` exists and is the pacing primitive for backfill loops.

## Types That Bite in the Shell

- `NumberLong("123")` vs `123`: the shell's plain numbers are doubles. A 64-bit id written from the shell as a plain number silently loses precision above 2^53 and will never match again.
- `new Date()` vs `ISODate()`: equivalent in `mongosh`, but a date written as a string is a string forever and sorts lexicographically.
- `ObjectId("...")` must be constructed; a 24-character hex string in a filter matches nothing and reports no error.
- Comparing a shell `Date` against a stored `Timestamp` (the internal oplog type) never matches — they are different BSON types.

## Compass, When It Beats the Shell

- Schema tab: samples the collection and charts field types and presence — the visual version of the drift query above.
- Explain plan tab: renders the plan tree, faster to read than nested JSON when hunting a `SORT` stage (→ `slow-queries.md`).
- Index tab shows usage counts alongside definitions, which is the audit that catches unused indexes.
- Aggregation builder previews stage by stage against real documents — the fastest way to develop a pipeline, and it exports to any driver language (→ `aggregation.md`).
