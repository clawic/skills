# Incidents — When Production Is Already Broken

Order of operations under pressure: observe before you change, and never restart a mongod to "clear" something — a restart drops the plan cache, empties the WiredTiger cache, and can turn a slow cluster into a cold one.

## First Ninety Seconds

1. `rs.status()` — is there a primary, and how far behind is each secondary? No primary means nothing else matters.
2. `db.currentOp({"secs_running": {$gt: 5}})` — long-running operations, with their `opid`, namespace, and what they wait on.
3. `db.serverStatus().connections` — `current` vs `available`. A cluster that "went down" is often a cluster that ran out of connections.
4. `db.serverStatus().wiredTiger.cache` — dirty bytes vs bytes in cache; the invisible stall (→ `tuning.md`).
5. Disk free on the data path. WiredTiger needs free space to check point and compact; a full volume is a different incident from a slow one.

## No Primary / Cluster Read-Only

- Symptom: writes fail with 10107 or the driver reports no writable server; reads on secondaries still work.
- Cause: the set lost majority. Three-member set with two members down, or a PSA topology where the arbiter cannot make quorum alone.
- Do not force a reconfiguration reflexively: `rs.reconfig(cfg, {force: true})` on a minority side can create a second primary and diverge history. Bring a data node back first if that is possible at all.
- If a member is truly gone and will not return, the recovery is to reconfigure the set to exclude it so the survivors form a majority — done from the surviving node, once, with the intent to never let the old node rejoin without a resync.
- Prevention lives in `replication.md`: three data-bearing nodes, and a documented answer to "which node do we sacrifice".

## Connection Storm

- Symptom: `connections.current` at the ceiling, new clients get server selection timeouts, existing queries are fine. Latency graphs look like a wall, not a slope.
- Usual trigger: a deploy or pod restart where every replica opens `maxPoolSize` connections at once, or a client created per request (SKILL.md rule 8).
- Immediate relief: scale the app down, not the database. Each connection costs up to roughly 1MB of mongod RAM, so the storm is also memory pressure.
- Identify the source: `db.currentOp(true)` includes idle connections with their `client` and `appName` — set `appName` in every connection string so this step takes seconds instead of guesswork (→ `connections.md`).
- Structural fix: `maxPoolSize` sized from concurrent operations per process, and a client cached outside serverless handlers.

## Runaway Operation

- Find it: `db.currentOp({"secs_running": {$gt: 60}, "op": {$in: ["query", "command", "getmore"]}})`.
- Kill it: `db.killOp(<opid>)`. Kills are cooperative — the operation stops at its next interrupt point, so a long `$group` may take seconds to die and an index build responds differently (below).
- Killing a write does not roll back what it already wrote unless it was in a transaction. A half-finished `updateMany` leaves a half-updated collection: know what the update did before you kill it, and make the retry idempotent.
- Prevention: `maxTimeMS` on user-facing queries so this class of incident cancels itself (→ `slow-queries.md`).

## Disk Full

- mongod may refuse writes or shut down. Deleting documents does NOT return space to the filesystem: WiredTiger reuses freed pages inside its files.
- Fastest safe space, in order: rotate and remove old log files; remove old `diagnostic.data` only if you have already captured what you need; drop an unused index (immediate, file-level); drop a whole unused collection (immediate, file-level).
- `compact` returns space but takes a full lock on the collection for the duration on most versions — run it on a secondary, then step over. Never as the first move on a primary mid-incident.
- Do not shrink the oplog to buy space during an incident: you shorten the window every recovery path depends on (→ `replication.md`).
- Real fix is capacity plus a growth alarm at 75%, because the last 10% disappears faster than the first 90% did.

## Cache Stall (the invisible one)

- Symptom: no slow query in the profiler, no lock contention, yet every operation is slow at once. CPU is high in eviction, not in your queries.
- Check: `wiredTiger.cache` "tracked dirty bytes" as a share of "maximum bytes configured". Eviction starts working the cache down at 80% full and gets aggressive near 95%; dirty content above ~20% drafts application threads into doing eviction work themselves — that hand-off is the latency cliff.
- Common causes: documents far larger than the working set warrants (SKILL.md rule 2), a bulk write with no pacing, an unbounded aggregation scanning history, or a cache sized against the wrong instance memory (→ `tuning.md`).
- Relief: pause the bulk writer. Raising the cache mid-incident starves the filesystem cache underneath and usually makes it worse.

## Blown Oplog Window

- Symptom: a secondary reports `RECOVERING` and cannot catch up; its last applied timestamp is older than the oldest oplog entry on the primary.
- There is no partial catch-up: the member needs an initial sync, or a file copy from a healthy member's snapshot. Initial sync of a multi-TB set runs for hours at disk and network speed.
- While one member resyncs, your effective redundancy is reduced — a three-member set doing an initial sync is a two-member set for majority purposes.
- Prevent by sizing the oplog against the longest outage you tolerate, not against defaults (→ `replication.md`).

## Index Build Gone Wrong

- Builds are non-blocking (MongoDB >=4.2) but not free: they consume IO, cache, and build memory, and they replicate to every member.
- Abort a running build with `db.adminCommand({dropIndexes: "<coll>", index: "<name>"})` against the build's name — this is the supported abort path, and it aborts the build across the replica set.
- A build that fails on a secondary can hold the whole set at that build; check `rs.status()` and the logs of every member, not just the primary.
- A unique index build fails at the first duplicate, after scanning everything: find the duplicates first with a `$group` count over the key (→ `indexes.md`).

## After Any Incident

- Capture `diagnostic.data` (FTDC) from the affected window before it rotates — it is the only per-second record of what happened.
- Write the timeline against the metric that moved first, not the alert that fired first; the alert is usually downstream (→ `monitoring.md`).
- Every incident here has a prevention line in another file. Land the prevention in the same week, or the runbook is just a record of repeating.
