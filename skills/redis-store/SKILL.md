---
name: redis-store
slug: redis-store
version: 1.0.1
description: >-
  Designs, tunes, and debugs Redis: data structures, memory limits, persistence, Streams and queues, locks, replication, and cluster.
  Use when writing Redis commands or Lua, choosing between a hash, a sorted set and a stream, setting expirations on cache keys,
  building a queue, distributed lock, rate limiter, leaderboard, session store or counter, or when Redis answers OOM, MISCONF,
  CROSSSLOT, MOVED, BUSY, LOADING or WRONGTYPE, latency spikes, memory keeps growing, keys vanish early or never expire,
  a replica lags or a failover loses writes, KEYS or a big DEL freezes the server, connections are refused, or a cluster reshard,
  a Valkey / ElastiCache / MemoryDB / Upstash move, or a persistence and backup plan is on the table. Covers redis-cli forensics,
  pipelining, eviction policies, keyspace notifications, ACLs and exposed-instance hardening,
  and the JSON, Search and TimeSeries modules. Not for store-agnostic cache hierarchy strategy (caching) or picking a
  rate-limit algorithm (rate-limiting).
homepage: https://clawic.com/skills/redis-store
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
metadata:
  clawdbot:
    emoji: 🔴
    requires:
      anyBins:
      - redis-cli
    os:
    - linux
    - darwin
    - win32
    displayName: Redis
    configPaths:
    - ~/Clawic/data/redis-store/
---

User preferences and memory live in `~/Clawic/data/redis-store/` (see `setup.md` on first use, `memory-template.md` for the file format). If you have data at an old location (`~/redis-store/` or `~/clawic/redis-store/`), move it to `~/Clawic/data/redis-store/`.

## When To Use

- Choosing how to model something in Redis: hash vs JSON string vs separate keys, sorted set vs stream, set vs bitmap vs HyperLogLog
- Writing Redis commands, pipelines, Lua scripts or Functions, and getting atomicity right
- Building the standard recipes: cache, session store, job queue, distributed lock, rate limiter, leaderboard, dedup/idempotency key, counter
- Operating a server: maxmemory and eviction, persistence and backups, ACLs, connection limits, replication, Sentinel, Cluster
- A production incident: OOM on writes, latency spike, unresponsive server, keyspace evicted or wiped, failover that lost writes, replica that never syncs
- Migrating: standalone to cluster, version upgrade, a managed provider (ElastiCache, MemoryDB, Redis Cloud, Upstash, Memorystore, Azure) or the Valkey fork
- Not for store-agnostic cache hierarchy design (caching), rate-limit algorithm selection (rate-limiting), or relational modeling (pg, sql)

## Quick Reference

| Situation | Play |
|---|---|
| Writes fail with `OOM command not allowed` | maxmemory reached under `noeviction` (the default policy) — Expiration And Eviction, then `memory-eviction.md` |
| Memory grows forever, hit rate fine | Keys written without a TTL, or a stream nobody trims (→ `memory-eviction.md`, `queues.md`) |
| Keys disappear before their TTL | Eviction, not expiry: check `evicted_keys` in `INFO stats` (→ `memory-eviction.md`) |
| Keys never expire | A plain `SET` on an existing key clears its TTL — use `SET ... KEEPTTL` (Redis >=6.0) or re-set it (→ `keys-ttl.md`) |
| Server froze for seconds | An O(N) command on a big collection, a fork, or swap — Latency Triage |
| Need to walk the keyspace | `SCAN` cursor loop with `COUNT 500`, never `KEYS` (→ `cli.md`) |
| Which structure for this data | Choosing The Data Structure |
| Job queue with retries and acks | Stream + consumer group; a List only for fire-and-forget (→ `queues.md`) |
| Subscribers miss messages sent while down | Pub/Sub is at-most-once and stores nothing — move to Streams (→ `pubsub.md`) |
| Two workers must not run at once | `SET lock <token> NX EX <ttl>` + Lua release comparing the token (→ `locks.md`) |
| Read-modify-write race | One atomic command or one Lua script; `MULTI` is not a rollback (→ `scripting.md`) |
| `CROSSSLOT keys ... don't hash to the same slot` | Hash-tag the related keys: `{user:1}:profile` (→ `cluster.md`) |
| `MOVED` / `ASK` reaching the app | Client is not cluster-aware, or its slot map is stale (→ `cluster.md`) |
| `MISCONF ... unable to persist to disk` | The last background save failed; writes are refused on purpose (→ `persistence.md`) |
| Writes acknowledged then lost after failover | Replication is asynchronous by default — `WAIT`, `min-replicas-to-write` (→ `high-availability.md`) |
| Latency spike with no CPU load | Fork for RDB/AOF rewrite, transparent huge pages, or swap (→ `performance.md`) |
| Throughput far below 10^5 ops/s | Round trips, not Redis: pipeline or use multi-key commands (→ `performance.md`) |
| Everything recomputes at once after a deploy or restart | Cache stampede: TTL jitter + single-flight recompute (→ `caching.md`) |
| Instance was reachable from the internet | Treat as compromised, then bind, ACL, TLS (→ `security.md`) |
| Provider rejects `CONFIG SET`, `DEBUG`, `SHUTDOWN` | Managed capability matrix (→ `managed-redis.md`) |
| Tests are flaky or slow around Redis | Real server in a container, unique prefix per test, no shared FLUSHDB (→ `testing.md`) |
| Anything else | `INFO`, `SLOWLOG GET 10`, `LATENCY DOCTOR`, `MEMORY DOCTOR` before changing anything; reproduce in `redis-cli` before blaming the client library |

Depth on demand, by phase:

- **Model** — `data-types.md` which structure wins and what it costs · `keys-ttl.md` key design, expiry semantics, SCAN, keyspace notifications · `patterns.md` sessions, leaderboards, counters, dedup, autocomplete, geo · `modules.md` JSON, Search, vectors, Bloom, TimeSeries
- **Build** — `caching.md` cache-aside, invalidation, stampede, client-side caching · `queues.md` Streams, consumer groups, retries, delayed jobs · `pubsub.md` fan-out, sharded Pub/Sub, notifications · `locks.md` distributed locks and fencing · `scripting.md` atomicity, MULTI, Lua, Functions · `testing.md` test isolation and CI
- **Operate** — `memory-eviction.md` maxmemory, eviction, big keys, fragmentation · `persistence.md` RDB, AOF, backups, restore drills · `connections.md` pooling, timeouts, buffers, TLS · `security.md` ACLs, exposure, command policy · `cli.md` the redis-cli forensics toolkit · `managed-redis.md` ElastiCache, MemoryDB, Redis Cloud, Upstash, Memorystore, Azure
- **Scale and recover** — `cluster.md` slots, hash tags, resharding · `high-availability.md` replication, Sentinel, failover, durability · `performance.md` latency triage, pipelining, command cost · `incidents.md` symptom-to-cause playbooks · `migrations.md` upgrades, standalone-to-cluster, provider and Valkey moves

## Core Rules

1. **Every key gets a TTL or a named owner that deletes it.** Redis never reclaims what nobody expires. Budget: `memory ≈ keys × (60-100 bytes of key overhead + value size)`, so 10M TTL-less session keys of 200 bytes cost roughly 3 GB whether or not anyone reads them. Check: `redis-cli INFO keyspace` reports `keys=N,expires=M` per db — a large `N − M` is the leak.
2. **Set `maxmemory` and a policy before production.** Without `maxmemory` Redis grows until the kernel swaps or the OOM killer takes it. Sizing: with fork-based persistence enabled, `maxmemory` ≤ 55-60% of host RAM (a copy-on-write fork can approach a second copy of the dataset in the worst case); a pure cache with persistence off can go to ~80%. Default policy is `noeviction`, which turns "full" into failed writes.
3. **One command, one atomic unit — `MULTI` is not a rollback.** A runtime error inside `EXEC` (wrong type, OOM) does not undo the commands that already ran. Prefer a single atomic command (`INCR`, `SET NX`, `LMOVE`), then Lua, then `WATCH`-and-retry. Check: if your logic reads a value and writes a value derived from it, it needs one of the three.
4. **Never run an O(N) command against an unbounded collection.** Command execution is serial: one `KEYS` or `HGETALL` over 1M elements stalls *every* client for the duration, and at roughly 10^5-10^6 elements that is tens to hundreds of milliseconds. Replacements: `SCAN`/`HSCAN`/`SSCAN`/`ZSCAN` with `COUNT`, `UNLINK` instead of `DEL` for big keys, `LRANGE` with real bounds.
5. **Round trips, not Redis, are your latency.** Unpipelined cost ≈ `n × RTT`: 1000 sequential `GET`s over a 0.5 ms network is 500 ms of wall time while the server spends under 10 ms. Batch with a pipeline (500 commands per flush is a sane default), `MGET`/`HMGET`, or a Lua script that does the loop server-side.
6. **A lock needs a unique token, a TTL longer than the work, and a compare-and-delete release.** TTL ≥ 3× the p99 duration of the critical section, renewed at 1/3 of the TTL by the holder. Releasing with a plain `DEL` deletes whatever lock exists — including the one the next worker just acquired after your TTL expired. The instance must be `noeviction`: that mandatory TTL is what makes a lock the first victim under `volatile-*` (→ `locks.md`).
7. **Replication is asynchronous: an acknowledged write can be lost in failover.** The master replies before replicas see the write. `WAIT 1 100` blocks until one replica acknowledges (still not consensus — a partitioned master can acknowledge and lose), `min-replicas-to-write 1` + `min-replicas-max-lag 10` refuses writes when nobody is listening. Choose the data-loss budget explicitly (→ `high-availability.md`).
8. **Persistence is a data-loss budget, not a checkbox.** RDB alone loses everything since the last snapshot (default save points fire at 3600s/1 change, 300s/100, 60s/10000). AOF with `appendfsync everysec` loses about one second and is the default balance; `always` costs an fsync per write. Both enabled = RDB for fast restore, AOF for the tail — Redis restores from AOF when it is on (→ `persistence.md`).

## Choosing The Data Structure

Pick the smallest structure whose access pattern matches the query you will actually run. Encoding thresholds and per-type memory math: `data-types.md`.

| Need | Structure | Wins because | Cost / limit |
|---|---|---|---|
| Blob, counter, flag | String | `INCR`, `SETNX`, `GETEX`, `APPEND` are single atomic ops | 512 MB max value; whole-value read-modify-write if you store JSON |
| Object with independently updated fields | Hash | `HSET`/`HINCRBY` touch one field; small hashes are stored as a packed listpack | Field-level TTLs need Redis >=7.4; otherwise the TTL is per key |
| FIFO/LIFO of jobs, capped log | List | `LPUSH`/`BLMOVE` are O(1) at the ends | No ack, no replay; index access is O(N) |
| Membership, tags, dedup set | Set | `SISMEMBER` O(1), `SINTER`/`SDIFF` server-side | Set ops are O(N) in the inputs — bound them |
| Ranking, sliding window, priority queue, time index | Sorted Set | Score-ordered range queries, O(log N) writes | Scores are IEEE-754 doubles: integers exact only to 2^53 |
| Event log with consumers, acks, replay | Stream | Consumer groups, pending list, `XAUTOCLAIM` recovery | Entries live until trimmed — `MAXLEN`/`MINID` or it grows forever |
| Daily-active flags, per-user boolean matrix | Bitmap (String) | 1 bit per user; `BITCOUNT`/`BITOP` server-side | 2^32 bits (512 MB) ceiling; sparse ids waste space |
| Approximate unique counts | HyperLogLog | 12 KB per counter at 0.81% standard error, mergeable with `PFMERGE` | No membership test, no exact count |
| Radius / nearest search | Geo (Sorted Set) | `GEOSEARCH` (Redis >=6.2) by radius or box | Geohash precision; still one sorted set under the hood |
| Query by field, full text, vectors | JSON + Search modules | Secondary indexes over hashes/JSON | Module availability differs per deployment (→ `modules.md`) |
| Anything else | Start with Hash or Sorted Set | They cover object and ordered-collection access | Re-check against this table once the query pattern is known |

## Latency Triage

Run in order; skipping to step 4 tunes the wrong thing.

1. Separate client-side from server-side: `redis-cli --latency -h <host>` (round-trip as seen from a client) vs `redis-cli --intrinsic-latency 100` on the server box (what the kernel and CPU alone cost). A high intrinsic number means the host, not Redis.
2. `SLOWLOG GET 10` — the log records commands over `slowlog-log-slower-than`, default 10000 microseconds, keeping the last `slowlog-max-len` 128. Its times exclude network, so an entry here is genuinely a slow command.
3. `LATENCY LATEST` and `LATENCY DOCTOR` after setting `latency-monitor-threshold 100` (default 0 = disabled). Events named `fork`, `aof-fsync-always`, `expire-cycle` name their own cause.
4. `INFO commandstats` — sort by `usec_per_call`, then multiply by `calls`: a 0.2 ms command called 5k/s consumes a full second of CPU per second of wall time — the entire single thread — and beats any 40 ms outlier as a target.
5. Fork suspicion: `INFO stats` `latest_fork_usec`. Fork cost tracks page-table size, on the order of 10-20 ms per GB of RSS on ordinary Linux VMs, and transparent huge pages multiply both the pause and the copy-on-write memory (→ `persistence.md`).
6. Still unexplained: check swap (`used_memory_rss` far above `used_memory` while the host swaps), `mem_fragmentation_ratio` below 1.0 means swapping, and `blocked_clients` for a queue of `BLPOP`/`BRPOPLPUSH` waiters (→ `performance.md`).

## Expiration And Eviction

Two different mechanisms produce the same symptom, "my key is gone", and have opposite fixes.

- **Expiry** removes keys that had a TTL. Redis mixes lazy deletion on access with an active cycle that runs at `hz` (default 10) times per second: it samples 20 keys with a TTL per database, deletes the expired ones, and repeats immediately while more than 25% of the sample was expired. Consequence: a key can outlive its TTL in memory for a moment, but never in reads — Redis never returns an expired value.
- **Eviction** removes keys because `maxmemory` was hit, and depends entirely on `maxmemory-policy`: `noeviction` (default: writes fail with OOM), `allkeys-lru` / `allkeys-lfu` / `allkeys-random`, `volatile-lru` / `volatile-lfu` / `volatile-random` / `volatile-ttl`. Diagnose with `INFO stats`: `expired_keys` rising is expiry, `evicted_keys` rising is eviction.
- The trap the `volatile-*` policies set: they can only evict keys that carry a TTL. A mixed workload where the persistent half has no TTL and the cache half does will refuse writes with OOM even though most of memory is evictable — the eviction candidate pool was empty.
- Locks and queues need a `noeviction` instance, full stop. Under `allkeys-*` a lock key or a stream is an ordinary eviction candidate; under `volatile-*` a lock is the *first* victim, because the TTL every lock must carry is exactly what puts it in the candidate pool (→ `locks.md`, `queues.md`).
- LRU here is sampled, not true LRU: `maxmemory-samples` (default 5) candidates per eviction; raising it to 10 tracks true LRU closely at a small CPU cost. LFU (`allkeys-lfu`) counts frequency with a logarithmic counter, tuned by `lfu-log-factor` (default 10) and decayed by `lfu-decay-time` (default 1 minute) — better for a cache with a hot, stable working set.
- Replicas do not expire keys on their own: they wait for the master's `DEL` and meanwhile answer reads as if the key were gone. A replica's `DBSIZE` can therefore exceed the master's without anything being wrong.

## Error Messages

Match on the prefix; message tails change between versions. Full playbooks in `incidents.md`.

| Reply | Meaning | First move |
|---|---|---|
| `OOM command not allowed when used memory > 'maxmemory'` | Memory limit reached and nothing is evictable | Policy or capacity — Expiration And Eviction |
| `MISCONF Redis is configured to save RDB snapshots...` | The last background save failed; `stop-writes-on-bgsave-error yes` refuses writes | Fix the disk, permissions, or `dir`, then `BGSAVE` (→ `persistence.md`) |
| `CROSSSLOT Keys in request don't hash to the same slot` | Multi-key command over keys in different slots | Hash tag the group, or split into per-key calls (→ `cluster.md`) |
| `MOVED 3999 host:port` / `ASK ...` | Slot lives elsewhere (`MOVED`) or is migrating (`ASK`) | Use a cluster-aware client and let it refresh the slot map |
| `BUSY Redis is busy running a script` | A Lua script exceeded `busy-reply-threshold` (default 5000 ms) | `SCRIPT KILL` if it has not written; otherwise only `SHUTDOWN NOSAVE` (→ `scripting.md`) |
| `LOADING Redis is loading the dataset in memory` | Startup or a full resync is reading RDB/AOF | Wait; `INFO persistence` `loading_*` fields estimate the remainder |
| `READONLY You can't write against a read only replica` | You are talking to a replica | Client is stale or misrouted (→ `high-availability.md`) |
| `WRONGTYPE Operation against a key holding the wrong kind of value` | Key namespace collision, or a type change without a migration | Namespace by type (→ `keys-ttl.md`) |
| `NOSCRIPT No matching script` | `EVALSHA` after a restart or `SCRIPT FLUSH` | Fall back to `EVAL`, or load at connect (→ `scripting.md`) |
| `EXECABORT Transaction discarded because of previous errors` | A queued command was syntactically invalid | Fix the queued command; note that runtime errors do *not* abort (Core Rule 3) |
| `NOAUTH` / `WRONGPASS` / `NOPERM` | Missing auth, bad credentials, ACL denies this command or key pattern | `ACL WHOAMI`, `ACL GETUSER <user>` (→ `security.md`) |
| `ERR max number of clients reached` | `maxclients` (default 10000) or the file-descriptor limit | Pool and cap; check leaked connections (→ `connections.md`) |
| `NOREPLICAS Not enough good replicas to write` | `min-replicas-to-write` is unsatisfied | Replica health first, never by lowering the setting blindly |

## Output Gates

Before emitting Redis commands, a client integration, or a config change:

- Does every key this creates either carry a TTL or have a named deleter?
- Is every read-modify-write one atomic command, one Lua script, or a `WATCH` retry loop?
- Is every keyspace-wide operation a `SCAN` loop, never `KEYS`, and every large delete an `UNLINK`?
- Under `topology: cluster`, do all keys of each multi-key command or script share a hash tag?
- Do the locks and queues sit on a `noeviction` instance (never `allkeys-*`, never `volatile-*`), and does the code still survive a restart?
- Are the round trips bounded — pipeline, `MGET`, or a script — rather than one call per item in a loop?
- Does anything here claim durability the configured `persistence_mode` and replication cannot deliver?
- Is a destructive command (`FLUSHALL`, pattern delete, `CONFIG SET`, failover) gated by `destructive_confirm`?

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/redis-store/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| deployment | self-hosted \| elasticache \| memorydb \| redis-cloud \| upstash \| memorystore \| azure | self-hosted | Gates which admin commands exist (`CONFIG SET`, `DEBUG`, `BGREWRITEAOF`) and switches tuning advice to parameter groups or console equivalents (→ `managed-redis.md`) |
| topology | standalone \| sentinel \| cluster | standalone | Decides whether multi-key commands, `SELECT`, plain Pub/Sub and Lua over several keys are safe to emit, and which failover story applies |
| server_version | number (6-8) | 7 | Which version-gated features appear (`feature >=X` lines): `KEEPTTL`, `GETEX`, sharded Pub/Sub, `XAUTOCLAIM`, Functions, hash-field TTLs |
| client | redis-cli \| redis-py \| ioredis \| node-redis \| go-redis \| lettuce \| jedis \| phpredis | redis-cli | The language of every emitted example, plus which pooling and reconnection advice applies (→ `connections.md`) |
| maxmemory_policy | noeviction \| allkeys-lru \| allkeys-lfu \| volatile-lru \| volatile-lfu \| volatile-ttl \| allkeys-random \| volatile-random | noeviction | Whether generated code may assume a key still exists. Anything other than `noeviction` makes locks, queues and counters unsafe and the warning is emitted: `allkeys-*` can evict them, and `volatile-*` is worse for a lock, whose mandatory TTL puts it *in* the candidate pool (→ `locks.md`) |
| persistence_mode | none \| rdb \| aof \| both | rdb | The durability claims attached to any recipe, and which backup and restore procedure is offered |
| key_prefix | text | app | The namespace in every generated key (`app:user:1:profile`); also the prefix used for scoped `SCAN` and test isolation |
| default_ttl | duration | 1h | The expiry written into generated cache examples, and the base for the ±10% jitter in `caching.md` |
| destructive_confirm | bool | true | `FLUSHALL`, `FLUSHDB`, pattern deletes, `CONFIG SET`, `SHUTDOWN`, `CLUSTER FAILOVER` and `SCRIPT FLUSH` are emitted for review instead of run |

Preference areas — customizable dimensions; a stated preference is recorded in `config.yaml` and applied from then on:

- **Tooling** — CLI vs RedisInsight vs a provider console, migration tooling (RIOT, `--cluster` helpers), benchmark harness
- **Thresholds** — slowlog threshold worth reporting, big-key size that forces a refactor, fragmentation ratio that triggers a defrag, pipeline batch size, `SCAN COUNT`
- **Conventions** — key separator and namespace depth, hash-tag policy, stream and consumer-group naming, cache-key versioning scheme
- **Platform** — server version, container vs VM, instance memory and cores, network RTT and region, TLS on or off
- **Risk posture** — run destructive or admin commands directly vs hand back reviewed commands, whether replica reads are acceptable, whether runtime `CONFIG SET` is allowed at all
- **Output format** — `redis-cli` transcripts vs client-library code, whether Lua is welcome, how much of the reasoning to narrate
- **Work order** — measure-before-change gates, whether to rehearse on a restored copy first, review before any keyspace-wide operation
- **Integrations** — monitoring stack (`INFO` scraping, Prometheus exporter, provider metrics), backup destination, alerting targets
- **Restrictions** — commands disabled by policy or provider (`KEYS`, `FLUSHALL`, `DEBUG`, `EVAL`), compliance regimes mandating TLS, ACLs or encryption at rest
- **Cadence** — restore-drill frequency, big-key and TTL audit cycle, failover-drill schedule

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `KEYS pattern` in application code | O(N) over the whole keyspace with every other client blocked behind it | `SCAN` cursor loop, or maintain an index set alongside (→ `cli.md`) |
| `INCR` then `EXPIRE` as two calls | A crash between them leaves a counter that never expires — a rate limiter that locks a user out forever | One Lua script, or `SET k 0 EX <ttl> NX` before `INCR` (→ `patterns.md`) |
| `DEL` on a multi-million-element collection | Frees every element synchronously; a multi-second stall | `UNLINK`, plus `lazyfree-lazy-*` on the server |
| `MULTI`/`EXEC` used as a transaction with rollback | Runtime errors leave earlier commands applied; there is no rollback | Lua for all-or-nothing (→ `scripting.md`) |
| Storing a JSON blob and updating one field | Read-modify-write over the network loses concurrent updates and re-serializes the whole document | Hash fields, or the JSON module for partial updates (→ `modules.md`) |
| Pub/Sub for work that must not be lost | At-most-once, no persistence, no acks: a disconnected subscriber misses everything | Streams with a consumer group (→ `queues.md`) |
| `XACK` without trimming | Acking removes the entry from the pending list, not from the stream; memory grows unbounded | `XADD ... MAXLEN ~ N` or a periodic `XTRIM MINID` (→ `queues.md`) |
| Releasing a lock with `DEL key` | After a TTL expiry you delete the *next* holder's lock | Lua compare-and-delete on the token (→ `locks.md`) |
| Expiry events used as a reliable trigger | Keyspace notifications are Pub/Sub: fire-and-forget, and the event fires when the key is actually deleted, not at the TTL instant | A sorted set of due timestamps polled by a worker (→ `keys-ttl.md`) |
| `CONFIG SET` without `CONFIG REWRITE` | The change disappears on the next restart, usually during the next incident | Rewrite the config file, or change it in the provider's parameter group |
| `SELECT 3` for multi-tenancy | Numbered databases share the same memory, eviction and blocking; Cluster only has db 0 | Key prefixes, or separate instances (→ `keys-ttl.md`) |
| Reading from a replica for correctness | Asynchronous replication returns stale data, and a lagging replica returns very stale data | Read the master, or make staleness explicit in the API (→ `high-availability.md`) |
| `MONITOR` left running in production | It streams every command to your client and can cut throughput by more than half | `SLOWLOG`, `INFO commandstats`, `--hotkeys` (→ `cli.md`) |
| Hash tags applied to everything | All keys land in one slot: a cluster with a single hot node and no way to rebalance | Tag only the groups you actually access together (→ `cluster.md`) |

## Where Experts Disagree

- **Redis as a primary database.** One camp treats any AOF-durable Redis as a legitimate system of record; the other allows only regenerable data. The testable boundary: can you name the recovery path when the last N seconds of writes are lost and no other system has them? If not, Redis is a cache with good uptime, not a database.
- **Redlock for distributed locking.** Multi-node Redlock adds availability but not correctness: its author and its critics have never agreed on the clock and process-pause assumptions it needs. Practical stance: single-node lock plus a short TTL for mutual exclusion that is an optimization; when correctness depends on the lock, you need fencing tokens checked by the resource itself (→ `locks.md`).
- **RDB vs AOF vs both.** RDB restores fast and forks rarely; AOF loses less. The disagreement is about the fork cost on large datasets. Boundary: dataset under a few GB and both on is nearly free; tens of GB and the fork pause becomes the argument for RDB-on-a-replica-only.
- **Cluster vs a bigger instance.** Cluster costs you multi-key operations, transactions across slots, and operational complexity. Most workloads outgrow one node's *throughput* far later than they think: reach for Cluster when the dataset no longer fits one machine or a single core saturates, not at the first latency complaint.
- **Lua in the server vs logic in the app.** Scripts buy atomicity and cut round trips; they also block the server and are hard to observe and version. Boundary: short, bounded, key-scoped scripts yes; anything looping over unbounded data, no.
- **Redis vs the Valkey fork.** After the 2024 license change the ecosystem split; both remain protocol-compatible for the classic command set, and the divergence is in modules, threading and licensing rather than in the fundamentals of this skill (dated detail in `managed-redis.md`).

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/redis-store (install if the user confirms):

- `caching` — store-agnostic cache strategy: hierarchy, invalidation policy, stampede theory
- `rate-limiting` — picking the algorithm (token bucket, sliding window) before implementing it here
- `distributed-systems` — consistency models and partial failure behind the replication and lock tradeoffs
- `pg` — when the data wants durability, joins, or transactions rather than a memory store
- `observability` — turning `INFO` and latency events into alerts that fire before the incident

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/redis-store.
