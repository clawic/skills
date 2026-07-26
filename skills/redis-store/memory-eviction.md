# Memory — Limits, Eviction, Big Keys, Fragmentation

Redis dies of memory more often than of anything else. Three numbers explain almost every case: `used_memory`, `maxmemory`, and `used_memory_rss`.

## Reading INFO memory

| Field | Means | Acts on |
|---|---|---|
| `used_memory` | Bytes the allocator handed to Redis for data | Compared against `maxmemory` for eviction decisions |
| `used_memory_rss` | Bytes the OS says the process holds | Compared against `used_memory` for fragmentation |
| `used_memory_peak` | High-water mark since start | Sizing: a peak near the limit means the headroom is theoretical |
| `used_memory_lua` / `used_memory_scripts` | Script engine | A growing number means scripts are being loaded per request |
| `mem_fragmentation_ratio` | `used_memory_rss / used_memory` | 1.0-1.5 normal; >1.5 fragmentation; <1.0 means part of the process is swapped out — the worst state |
| `maxmemory_policy` | Active eviction policy | `noeviction` turns full into failed writes |
| `mem_not_counted_for_evict` | Replica buffers and AOF buffer | Explains why eviction starts "late" relative to RSS |

`INFO keyspace` complements it: `keys=` minus `expires=` is the TTL-less population.

## Sizing maxmemory

Canonical rule in `SKILL.md` Core Rule 2: with fork-based persistence enabled, `maxmemory` ≤ 55-60% of host RAM; a pure cache with persistence off can go to ~80%.

Why the gap: `BGSAVE` and AOF rewrite fork, and the child's copy-on-write pages grow with the write rate during the save. A write-heavy instance can approach a second copy of the dataset before the child exits. Additional consumers that are *not* the dataset:

- Client output buffers, especially replicas and Pub/Sub subscribers (`client-output-buffer-limit` defaults: replica `256mb 64mb 60`, pubsub `32mb 8mb 60`)
- The replication backlog (`repl-backlog-size`, default 1mb, sized for the expected reconnect window)
- The AOF buffer between fsyncs, and the rewrite buffer during a rewrite

Managed providers reserve this for you and expose it as a percentage (ElastiCache's `reserved-memory-percent` defaults to 25) — do not add your own reservation on top without checking.

## When Redis Is Full

The two-question triage:

1. `INFO stats` — is `evicted_keys` rising (policy is working, capacity or TTLs are wrong) or is it flat while writes fail with OOM (nothing is evictable)?
2. `CONFIG GET maxmemory-policy` — `noeviction` means the OOM is by design; a `volatile-*` policy with no TTL-carrying keys means the candidate pool is empty and the effect is identical.

Then, in order of how fast they help:

- Emergency headroom: raise `maxmemory` if the host has RAM to spare (`CONFIG SET maxmemory 8gb`, then `CONFIG REWRITE` or it disappears at the next restart).
- Find the leak before deleting anything: `redis-cli --memkeys` for the biggest keys, `--bigkeys` for the biggest per type, `INFO keyspace` for the TTL-less mass.
- Delete by prefix with `SCAN` + `UNLINK`, never `KEYS` and never `FLUSHDB`.
- Fix the class of key that grew: an untrimmed stream, a set that only ever gains members, a cache key built from unbounded user input.

## Choosing An Eviction Policy

| Workload | Policy | Why |
|---|---|---|
| Pure cache, everything regenerable | `allkeys-lru` | Any key may go; recency approximates value |
| Cache with a stable hot set and cold scans | `allkeys-lfu` | Frequency survives a nightly full scan that would poison LRU |
| Mixed: cache keys have TTLs, durable keys do not | `volatile-lru` **plus** an audit that every cache key really has a TTL | Protects the durable half — and fails with OOM the moment the TTL-carrying half is exhausted |
| Queue, lock, or system-of-record data | `noeviction` | Silent eviction of a lock or a queue entry is a correctness bug, not a capacity event |
| Anything else | `allkeys-lru` with `maxmemory-samples 10` | Safe default; revisit once the access pattern is measured |

- LRU is sampled: `maxmemory-samples` (default 5) candidates per eviction, 10 tracks true LRU closely at a small CPU cost.
- LFU counters are logarithmic (`lfu-log-factor`, default 10) and decay (`lfu-decay-time`, default 1 minute). Inspect a key's frequency with `OBJECT FREQ` (LFU only) and its idle time with `OBJECT IDLETIME` (LRU only).
- Eviction runs *before* the command that would exceed the limit, in a loop until it fits — a huge incoming value can evict a lot at once and shows up as a latency spike.

## Big Keys

A "big key" is any key large enough that a single operation on it stalls the server (Core Rule 4). Practical alarm line: **any collection above ~10k elements or any value above ~100 KB deserves a design review**; above 1M elements it is an incident waiting for the first `HGETALL`.

- Find them: `redis-cli --bigkeys` (per-type largest, SCAN-based, safe on a live server) and `--memkeys` (ranked by real bytes).
- Measure one: `MEMORY USAGE key SAMPLES 0`.
- Fixes, in order of preference: shard by id (`obj:{id mod 1000}`), keep only a window (`LTRIM`, `XADD MAXLEN ~`, `ZREMRANGEBYRANK`), move the payload out of Redis and keep a pointer, or split the hot fields into their own key.
- Deleting one is itself the hazard: `UNLINK`, and enable `lazyfree-lazy-expire`, `lazyfree-lazy-eviction`, `lazyfree-lazy-server-del` so expiry and eviction of big keys also happen off the main thread.

## Fragmentation

- Ratio 1.0-1.5 is normal — the allocator rounds allocations up to size classes. Above 1.5 the process holds memory it is not using; below 1.0 the OS has swapped part of it out, which turns microsecond operations into millisecond ones.
- The usual cause is a workload that wrote many large values and then deleted them, leaving holes jemalloc cannot return.
- Active defrag: `activedefrag yes`, with `active-defrag-ignore-bytes` (default 100mb) and `active-defrag-threshold-lower` (default 10%) deciding when it starts. It costs CPU and works gradually; it is the right answer when a restart is expensive.
- The reliable answer is still a restart or a failover to a fresh replica — schedule it rather than fighting the ratio during an incident.
- Never let Redis swap: set `vm.swappiness` low or disable swap for the instance, and enable `vm.overcommit_memory=1` so `BGSAVE`'s fork is not refused.

## Counting Before Committing

Estimating a design before writing it beats measuring after:

```
total ≈ Σ per key ( 60-100 B overhead + key name bytes + value bytes )
```

- 5M sessions × (80 B + 25 B key + 300 B hash, packed) ≈ 2.0 GB — plus replication and fork headroom, so a 4 GB instance is tight and an 8 GB one is honest.
- Verify the assumption instead of trusting the arithmetic: write 10k representative keys into a scratch instance, read `used_memory` before and after, divide. That number includes encoding effects the formula cannot know.
