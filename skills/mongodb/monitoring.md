# Monitoring — What To Watch and What To Alert On

Two different lists. Watching is for understanding; alerting is for waking someone. A metric that has never predicted an incident belongs on a dashboard, not in a pager rule.

## The Six That Predict Incidents

| Metric | Source | Alarm when |
|---|---|---|
| Replication lag | `rs.printSecondaryReplicationInfo()`, `serverStatus().repl` | Above the value your reads tolerate, and separately above 10% of the oplog window |
| Oplog window (hours of history retained) | `db.getReplicationInfo().timeDiff` | Below the longest maintenance you perform — that is the point where an outage becomes a full resync (→ `replication.md`) |
| WiredTiger cache dirty share | `serverStatus().wiredTiger.cache` | Sustained dirty above ~20% of the configured cache: application threads start doing eviction work (→ `incidents.md`) |
| Connections used vs available | `serverStatus().connections` | Above 80% of the cap; connection storms have no gradual phase (→ `connections.md`) |
| Concurrency tickets available | `serverStatus().wiredTiger.concurrentTransactions` | Available read or write tickets hitting zero — the queue forms there before it shows as query latency |
| Disk free on the data path | Host | Below 25%: WiredTiger needs headroom to checkpoint and compact, and freed documents do not return space |

## Query-Health Metrics

- **Examined:returned ratio**, per query shape, not cluster-wide. Atlas's Query Targeting alert defaults to 1000:1 (SKILL.md rule 1); self-hosted, compute it from the profiler over a window.
- **Scan-and-order count** — `serverStatus().metrics.operation.scanAndOrder` counts operations that sorted in memory because no index provided the order. A rising trend names a missing index before users notice.
- **Slow-op count over `slow_ms`**, split by collection. One collection dominating is a design conversation; every collection rising together is a resource conversation.
- **`opcounters`** — the shape of your traffic. A query:insert ratio that changes without a deploy usually means retries, not new users.
- **Queued operations** — `globalLock.currentQueue.readers/writers`. Nonzero for more than a moment means the server, not the query, is the bottleneck.

## Sources, and What Each Is Good For

- **`serverStatus()`** — the one-shot snapshot of everything above. Diff two samples 10 seconds apart; the absolute counters are since startup and mean little alone.
- **FTDC (`diagnostic.data` directory)** — per-second metrics the server records continuously, rotating over days. It is the only post-incident record with sub-minute resolution; capture it before it rotates (→ `incidents.md`).
- **The profiler** — per-operation detail, per database, expensive at level 2 (→ `slow-queries.md`).
- **`$currentOp` aggregation stage** — richer and filterable, unlike the `db.currentOp()` helper: `db.aggregate([{$currentOp: {allUsers: true, idleConnections: true}}, {$match: ...}])`.
- **`$indexStats`** — per-index usage counters, per node, reset on restart (→ `indexes.md`).
- **`collStats` / `dbStats`** — size, storage size, index sizes. `storageSize` far above `size` is reclaimable space held inside the files.
- **Atlas metrics and alerts** — the same underlying numbers with defaults already tuned; Performance Advisor turns the query-health metrics into index suggestions (→ `atlas.md`).

## Building the Alert Set

Rank by "would this page have been actionable at 3am":

1. No primary, or a member unreachable for over a minute — always page.
2. Replication lag past the tolerance you actually wrote down, sustained for two intervals.
3. Connections above 80% of cap.
4. Disk below 25%, and again at 10% as a separate, louder rule.
5. Oplog window below your maintenance duration — this one is a ticket, not a page, because it degrades over days.
6. Query targeting above the ratio for a shape that serves users — ticket.

Everything else (cache dirty share, tickets, scan-and-order, opcounters) is dashboard material you read while diagnosing an alert from the list above.

## Baselines Are the Whole Point

- Record the normal value of each metric on a quiet Tuesday. "Cache dirty at 12%" means nothing until you know your normal is 3%.
- Re-baseline after every capacity change, index change, or major version upgrade — all three move these numbers legitimately.
- Keep the numbers with the cluster, not in someone's memory: the `Thresholds` preference area exists for exactly this (SKILL.md Configuration), and observed values belong in `memory.md`, not in the skill body.

## What Not To Alert On

- CPU alone: MongoDB uses available CPU by design, and a cluster at 70% CPU serving fast queries is healthy.
- Absolute connection count without the cap: 2,000 connections is a crisis on one tier and idle on another.
- Individual slow queries: a report that takes 4 seconds at 3am is not an incident. Alert on the rate of slow queries, not on their existence.
- Cache "bytes in cache" near the configured maximum: a full cache is the goal. It is the DIRTY share that predicts trouble.
