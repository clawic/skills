# Tuning — WiredTiger Cache, Storage, and Host Settings

Order of leverage, highest first: schema shape → indexes → host settings → server parameters. Teams reliably do this backwards. Nothing in this file recovers a design that makes MongoDB read 100× more data than the query needs (→ `schema.md`, `indexes.md`).

## The Working Set Is the Only Sizing Question

- The working set is the data plus indexes that queries actually touch in a typical window — not the collection size.
- Rule of thumb: RAM should hold the working set plus all frequently-used indexes. When it does, reads are memory speed; when it does not, every miss is a disk read, and the transition is a cliff, not a slope.
- Measure rather than estimate: `db.coll.stats().totalIndexSize` summed across hot collections is the index half; the data half comes from how much of each collection a day's queries touch.
- Symptom of an undersized working set: `serverStatus().wiredTiger` cache read-into-cache rate climbing while query patterns are unchanged.

## WiredTiger Cache

- Default size: `max(50% × (RAM − 1GB), 256MB)`. The rest of RAM is the filesystem cache holding COMPRESSED blocks — do NOT raise the cache to 90% of RAM; you would starve the layer underneath and end up with less effective caching, not more.
- Eviction works the cache down starting around 80% full and becomes aggressive near 95%. Dirty content above roughly 20% of the cache drafts application threads into doing eviction work — that hand-off is the latency cliff described in `incidents.md`, and it appears as universal slowness with no slow query.
- Watch `serverStatus().wiredTiger.cache`: "bytes currently in the cache" versus "maximum bytes configured" (fullness) and "tracked dirty bytes" (the predictive one). A full cache is healthy; a dirty cache is not.
- Raise `cacheSizeGB` only when the host runs nothing else and the filesystem cache has more than it needs — which is rare outside dedicated database hosts.
- In a container, set `cacheSizeGB` explicitly. Older server versions size the cache from HOST memory rather than the cgroup limit, which produces a container that OOMs at startup on a large machine.

## Storage and Compression

- Collections use snappy compression by default: fast, modest ratio. `zstd` (MongoDB >=4.2) compresses substantially better at more CPU — the right trade for cold, large, write-once data such as archives and logs.
- Indexes use prefix compression and are NOT affected by the collection compressor. Index size is driven by key size and count, which is a design decision (→ `indexes.md`).
- Compression is set per collection at creation. Changing it means creating a new collection and copying (→ `migrations.md`).
- Checkpoints run about every 60 seconds; the journal group-commits about every 100ms. Both produce periodic IO spikes, and a disk that cannot absorb a checkpoint shows up as periodic latency waves rather than constant slowness.
- Deleting documents does not shrink files; space is reused internally. `compact` returns it at the cost of a collection-level lock on most versions — run it on a secondary and step over (→ `incidents.md`).

## Host Settings That Matter

| Setting | Value | Why |
|---|---|---|
| Transparent Huge Pages | disabled (`never`) | THP causes allocation stalls and memory bloat under WiredTiger's access pattern; MongoDB warns about it in the log at startup for a reason |
| Open file limit (`nofile`) | 64000 | Each connection and each WiredTiger file consumes handles; the default 1024 fails as mysterious connection errors |
| Process/thread limit (`nproc`) | 64000 | Same class of failure under connection load |
| Filesystem | XFS on Linux | Recommended for WiredTiger; ext4 works but has known performance edges under heavy checkpoint IO |
| `vm.swappiness` | 1 | Not 0: allow emergency swap rather than inviting the OOM killer, but never swap by preference |
| Readahead | small (8-32KB) | MongoDB reads scattered pages; large readahead loads pages nobody wants and evicts pages someone does |
| NUMA | interleaved | Running mongod bound to one NUMA node produces erratic latency; `numactl --interleave=all` is the documented remedy |
| `atime` | disabled (`noatime`) | Removes a metadata write on every read |
| Clock sync | NTP everywhere | Elections, cluster time, and TLS validity all depend on it (→ `replication.md`) |

On Atlas these are already set correctly and are not adjustable — the reason `deployment: atlas` suppresses this section entirely (→ `atlas.md`).

## Concurrency Tickets

- WiredTiger admits a bounded number of concurrent read and write operations; the rest queue. Historically the default was 128 of each; MongoDB >=7.0 sizes it dynamically from observed throughput.
- `serverStatus().wiredTiger.concurrentTransactions` reports available tickets. Available hitting zero is the earliest honest signal that the server, not the query, is the limit (→ `monitoring.md`).
- Raising the ticket count almost never helps: the queue exists because the storage engine is saturated, and admitting more work makes each operation slower. Fix the IO, the working set, or the query.

## Connection-Level Cost

- Each connection costs up to roughly 1MB of mongod RAM. Pool math and the fleet arithmetic live in `connections.md`; the tuning-side consequence is that connections compete with the cache for the same memory.
- A host sized for its data and then given 4,000 connections is a host that was silently resized.

## Verifying a Tuning Change

1. Baseline the six metrics in `monitoring.md` over a normal hour.
2. Change ONE thing.
3. Re-measure over a comparable hour — same time of day, same traffic shape. Comparing Monday morning against Saturday night proves nothing.
4. Keep the number, the date, and the reason with the cluster. Tuning knowledge that lives in one engineer's memory gets reverted by the next capacity change.
