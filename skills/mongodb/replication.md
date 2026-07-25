# Replication — Replica Sets, Concerns, Oplog, Upgrades

## Write Concern: the Durability Ladder

- `w: 0` fire-and-forget · `w: 1` primary-ack, rolls back if the primary dies before replication · `w: "majority"` survives failover.
- The implicit default became `w: "majority"` in 5.0 — EXCEPT topologies with arbiters, which stay at `w: 1`. Never rely on it: put `w=majority&retryWrites=true` in the URI (SKILL.md rule 4).
- `j: true` forces a journal flush before ack; otherwise the journal group-commits on a 100ms interval. `w: "majority"` already implies journaling on the majority by default.
- `wtimeout` on a majority write is a trap without a plan: on timeout the write is NOT rolled back, it is merely unacknowledged. Code that treats a `wtimeout` as a failure and retries produces duplicates unless the operation is idempotent.
- Assign a concern per data class: analytics/telemetry `w: 1` · user data `w: "majority"` · irreversible actions (payment, deletion) `w: "majority"` and verify the ack in code.

## Read Concern

| Level | Guarantee | Cost / catch |
|---|---|---|
| `local` | Whatever this node has | Can return writes that later roll back |
| `available` | Same, minus orphan filtering on sharded clusters | Can return the same document twice — never for correctness |
| `majority` | Only majority-committed data | Pins WiredTiger history; expensive under a degraded PSA topology |
| `linearizable` | Real-time linearized, single-document reads on the primary only | Waits for a majority write to confirm; use with `maxTimeMS` or it can hang |
| `snapshot` | Consistent point-in-time across the operation | Transactions and some aggregations only (→ `transactions.md`) |

## The PSA Trap

- Primary-Secondary-Arbiter costs one data node less and fails differently: lose ONE data node and `w: "majority"` hangs, while majority read concern pins WiredTiger history on the survivor → cache pressure on the exact node you need healthy.
- Mitigation when degraded: reconfigure to strip the dead member's vote. Prevention: three data-bearing nodes. Arbiters belong only where you truly cannot afford the third data copy — and then you accept `w: 1` semantics during any outage.

## Replication Health

- Oplog window = oplog size ÷ write churn per hour. It must exceed your longest tolerable member outage (maintenance + resync headroom) AND your longest tolerable change-stream consumer outage (→ `change-streams.md`). Default size: 5% of free disk, clamped to 990MB–50GB; resize live with `replSetResizeOplog`.
- The oplog is idempotent by construction: an `$inc` is recorded as the resulting `$set`. This is why replaying it is safe, and why oplog entries can be much larger than the update that produced them (an update touching one field of a large array can log the whole array).
- Flow control (MongoDB >=4.2) throttles the primary when majority-commit lag exceeds 10s (`flowControlTargetLagSeconds`) — a sudden primary write-latency spike with a lagging secondary is usually flow control working as designed. Fix the secondary, not the primary.
- Elections: `electionTimeoutMillis` defaults to 10s; retryable writes hide most failovers from clients — with the multi-statement exception in SKILL.md (Consistency Model).
- Bound secondary staleness with `maxStalenessSeconds` (minimum 90) instead of hoping (→ `connections.md`).
- Priority and votes shape who can win: `priority: 0` for a member that must never be primary (analytics node, distant region), `hidden: true` so clients never route reads to it, `votes: 0` for members beyond the seven-voter limit.

## Failover: What Clients See

1. Primary becomes unreachable or steps down. In-flight writes to it fail.
2. Remaining members hold an election; with the default settings a new primary is typically serving within seconds, not minutes.
3. Drivers discover the new topology through heartbeats and retry eligible operations once (retryable writes and reads).
4. Writes acknowledged only at `w: 1` on the old primary, and not yet replicated, become a rollback file on that node when it rejoins. Nothing in the application is told.

Test this deliberately: `rs.stepDown()` in staging, with the application running. An application that needs a restart after a stepdown has a connection-string bug (→ `connections.md`).

## Rolling Maintenance

The pattern for any change requiring a restart — configuration, OS patch, storage move, minor version:

1. Apply to each SECONDARY in turn: stop, change, start, wait until `rs.status()` shows `SECONDARY` and lag has returned to normal.
2. `rs.stepDown()` on the primary; wait for a new primary.
3. Apply to the old primary, now a secondary.

Never do two members at once in a three-member set: that is a minority and the set goes read-only (→ `incidents.md`).

## Upgrades

- One major version at a time. Skipping a version is unsupported and the failure surfaces as data-format errors, not as a clean refusal.
- Sequence: upgrade the binaries with a rolling restart (above) → run at the new binary version with the OLD `featureCompatibilityVersion` for a burn-in period → `setFeatureCompatibilityVersion` to the new value only when you are confident.
- FCV is the downgrade switch: until you raise it, the on-disk format stays compatible and you can roll back by reversing the rolling restart. After you raise it, you cannot.
- Upgrade drivers BEFORE the server, not after. Driver versions declare which server versions they support; the reverse order strands you.
- On a sharded cluster the order is config servers → shards → mongos routers (→ `sharding.md`).
- Read the release notes for behavior changes, not just features: implicit write concern (5.0) and `allowDiskUseByDefault` (6.0) both changed semantics for code that never mentioned them.

## Adding and Recovering Members

- A new member does an initial sync: full copy plus oplog catch-up, running at disk and network speed for hours on a large set. During it, your effective redundancy is one member lower.
- Faster alternative when you have a consistent snapshot: seed the new member's data directory from a filesystem snapshot of a healthy member, then let it catch up from the oplog — only valid if the snapshot is newer than the oldest oplog entry (→ `backups.md`).
- A member whose last applied timestamp falls outside the oplog window cannot catch up at all and needs a fresh initial sync (→ `incidents.md`).
- Remove a member with `rs.remove()` before decommissioning the host, not after. A member that vanishes still counts in the configuration's arithmetic for elections.
