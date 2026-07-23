# Production Configuration

## Write Concern: the Durability Ladder

- `w: 0` fire-and-forget · `w: 1` primary-ack, rolls back if the primary dies before replication · `w: "majority"` survives failover.
- The implicit default became `w: "majority"` in 5.0 — EXCEPT topologies with arbiters, which stay at `w: 1`. Never rely on it: put `w=majority&retryWrites=true` in the URI (SKILL.md rule 4).
- `j: true` forces a journal flush before ack; otherwise the journal group-commits on a 100ms interval. `w: "majority"` already implies journaling on the majority by default.
- Assign a concern per data class: analytics/telemetry `w: 1` · user data `w: "majority"` · irreversible actions (payment, deletion) `w: "majority"` and verify the ack in code.

## The PSA Trap

- Primary-Secondary-Arbiter costs one data node less and fails differently: lose ONE data node and `w: "majority"` hangs, while majority read concern pins WiredTiger history on the survivor → cache pressure on the exact node you need healthy.
- Mitigation when degraded: reconfigure to strip the dead member's vote. Prevention: three data-bearing nodes. Arbiters belong only where you truly cannot afford the third data copy — and then you accept `w: 1` semantics during any outage.

## Replication Health

- Oplog window = oplog size ÷ write churn per hour. It must exceed your longest tolerable member outage (maintenance + resync headroom). Default size: 5% of free disk, clamped to 990MB–50GB; resize live with `replSetResizeOplog`.
- Flow control (4.2+) throttles the primary when majority-commit lag exceeds 10s (`flowControlTargetLagSeconds`) — a sudden primary write-latency spike with a lagging secondary is usually flow control working as designed. Fix the secondary, not the primary.
- Elections: `electionTimeoutMillis` defaults to 10s; retryable writes hide most failovers from clients — with the multi-statement exception in SKILL.md (Consistency Model).
- Bound secondary staleness with `maxStalenessSeconds` (minimum 90) instead of hoping.

## Memory, Cache, Connections

- WiredTiger cache default: max(50% × (RAM − 1GB), 256MB). The rest of RAM is the filesystem cache for compressed blocks — do NOT raise the cache to 90% of RAM; you'd starve the layer below it.
- Stall signature: dirty cache ≥ 20% drafts application threads into eviction — a latency cliff, not a gradual slowdown. Eviction starts at 80% full, panics at 95%. Watch `serverStatus().wiredTiger.cache`: "tracked dirty bytes" vs "bytes currently in the cache".
- Each connection costs up to ~1MB of mongod RAM. Pool math: 40 app pods × driver-default `maxPoolSize` 100 = 4,000 connections ≈ 4GB gone before any data. Size `maxPoolSize` from concurrent operations per pod, not the default.
- Always connect with the replica set URI (all hosts + `replicaSet=`), never a single node — that's what makes failover automatic.

## Triage Order for a Slow Cluster

1. `db.currentOp()` — long-running or stuck operations first.
2. Profiler and logs — `db.setProfilingLevel(1, {slowms: 100})`; level 1 logs slow ops, level 2 logs everything (never in production). Look for COLLSCAN and `usedDisk`.
3. Cache dirty percentage (above) — the invisible stall cause.
4. Replication lag and flow control.
5. Connection count vs limit — connection storms from pod restarts look like database slowness.

## Sharding

- Judge a shard key on three axes: cardinality, frequency, monotonicity. Monotonic keys (ObjectId, timestamps) funnel every insert to the last chunk → one hot shard doing all writes. Fix: compound `{tenant_id: 1, ts: 1}` or hashed — but hashed kills range queries (broadcast to all shards).
- The shard key must appear in hot query filters, not just spread writes: queries without it scatter-gather every shard and get slower as you add shards.
- Mistakes are recoverable now: `refineCollectionShardKey` (4.4) adds suffix fields; `reshardCollection` (5.0) rewrites live — budget significant free disk and IO for it.
- When to shard: before operational limits, not at them. Initial sync or restore of a multi-TB replica set runs at disk/network speed for hours — when restore time exceeds your RTO, that's the signal, independent of query performance.
- Since 6.0 the default range size is 128MB and the balancer moves data by size, not chunk count.

## Backups

- `mongodump`: logical, slow, and pollutes the cache on large data — fine for small datasets and pre-migration snapshots, not a DR strategy at scale.
- Filesystem/EBS snapshots: journal and data must share the volume, or `fsyncLock` first; restore is a crash-recovery replay.
- Point-in-time recovery requires continuous oplog capture (Atlas/Ops Manager territory). Sharded clusters need cluster-consistent backups: stopped balancer + coordinated snapshots — tooling, not hand-rolled dumps.
- A backup you have never restored is a hypothesis. Rehearse the restore path and time it — that number is your real RTO.

## Upgrade Discipline

- One major version at a time, and run `setFeatureCompatibilityVersion` only after burn-in: FCV gates the new on-disk formats, and until you raise it, downgrade remains possible.
