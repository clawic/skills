# Sharding — Shard Keys and the Balancer

Sharding is the last scaling tool, not the next one. It adds a config server replica set, a router tier, cross-shard query semantics, and a decision (the shard key) that used to be permanent and is now merely expensive to change.

## When To Shard

- Working set no longer fits in the largest instance's RAM, and adding RAM is no longer available or affordable.
- Write throughput exceeds what one primary can absorb — remembering that every secondary applies every write, so replicas add read capacity only.
- Operational time, not query time: initial sync or restore of a multi-TB replica set runs at disk and network speed for hours. When restore time exceeds your RTO, that is the signal, independent of performance (→ `backups.md`).
- Data residency: zone sharding pins ranges to specific regions, which is sometimes the only reason to shard at all.

Before sharding, exhaust: indexes (→ `indexes.md`), schema shape (→ `schema.md`), archiving cold data, and a bigger machine. Each is reversible; sharding is not, in practice.

## Judging a Shard Key

Three axes, all three required:

| Axis | Question | Failure if wrong |
|---|---|---|
| Cardinality | How many distinct values? | A key with 50 values caps you at 50 chunks and therefore 50 shards, forever |
| Frequency | Are values evenly used? | One tenant with 80% of the traffic makes one shard hot no matter how many exist |
| Monotonicity | Does the value always increase? | Every insert lands in the last chunk: one shard does all the writes while the others idle |

- Monotonic keys — `ObjectId`, timestamps, auto-incrementing ids — are the classic mistake (SKILL.md, ObjectId and _id). Fix by compounding: `{tenantId: 1, ts: 1}` distributes across tenants while keeping each tenant's data range-queryable.
- Hashed sharding distributes perfectly and destroys range queries: every range becomes a scatter-gather across all shards. Choose it only when queries are point lookups by that key.
- **The shard key must appear in your hot query filters, not merely spread writes.** A query without the key is broadcast to every shard and gets SLOWER as you add shards. This is the axis teams forget, and it is the one that makes sharding feel like a downgrade.
- The key's fields must be indexed (the index may be compound with the key as its prefix), and shard key fields are effectively required on every document.

## Targeted vs Scatter-Gather

- Targeted: the filter contains the shard key (or its prefix) → mongos routes to one shard. This is the whole point.
- Scatter-gather: no shard key in the filter → every shard runs the query and mongos merges. Latency becomes the slowest shard's latency, and throughput divides rather than multiplies.
- Sorts and `$group` without the shard key merge on the router: mongos becomes the bottleneck and the memory pressure point.
- Audit which is which: `explain()` on a sharded collection reports `shards` — one entry means targeted, N entries means scatter-gather. Run it on your top ten queries before declaring the key good.

## Fixing a Bad Shard Key

Mistakes are recoverable now, at a cost:

- `refineCollectionShardKey` (MongoDB >=4.4) appends suffix fields to the existing key. Cheap, but it cannot fix a bad PREFIX — a monotonic first field stays monotonic.
- `reshardCollection` (>=5.0) rewrites the collection under a completely new key while it stays online. Budget free disk of roughly the collection's size plus IO headroom, and expect it to take as long as a full copy.
- The old workaround (dump, drop, reshard, restore) is still the fallback on older versions, and it is downtime.

## The Balancer

- The balancer moves ranges between shards to equalize DATA SIZE, not document count or load. A shard hot with traffic but small in bytes will never be rebalanced away from.
- Default range size is 128MB (MongoDB >=6.0; 64MB before). Larger ranges mean fewer, bigger migrations; smaller means more churn.
- Migrations consume IO on both shards. Set a balancing window during off-peak hours on write-heavy clusters rather than letting migrations compete with traffic.
- Jumbo ranges: a range that cannot be split because every document in it shares one shard key value. It stops migrating and the imbalance is permanent until the key changes. This is the cardinality failure showing up months later.
- Orphaned documents — left behind by interrupted migrations — are filtered on read by the `SHARDING_FILTER` stage (→ `slow-queries.md`). Read concern `available` skips that filter and returns them; `estimatedDocumentCount` counts them.

## Zone Sharding

- Tag ranges of shard key values to tagged shards: `{country: "DE"}` ranges to EU-tagged shards. The mechanism for data residency and for tiered storage (recent data on fast shards, archives on cheap ones).
- Zones constrain the balancer; they do not move data instantly. Adding a zone to a populated cluster starts a long migration.
- A zone with no eligible shard silently blocks migrations for that range. Check `sh.status()` after every zone change.

## Operating a Sharded Cluster

- Config servers are a replica set (CSRS) and are the cluster's metadata. Back them up with the same seriousness as data, and never let them run out of disk (→ `backups.md`).
- mongos routers are stateless and cheap to run; put them close to the application (sidecar or per-node), because every scatter-gather pays the router-to-shard round trip.
- Unsharded collections in a sharded database all live on that database's primary shard. A database full of small unsharded collections concentrates on one shard and surprises people during capacity planning; `movePrimary` relocates it, with the collection unavailable during the move.
- `sh.status()` is the state of the world: shards, chunk distribution per collection, balancer state, zones. Read it before and after every change.
- Backups must be cluster-consistent: stop the balancer, coordinate snapshots across all shards and the config servers, restart the balancer. Independent per-shard dumps restore to an inconsistent cluster (→ `backups.md`).
