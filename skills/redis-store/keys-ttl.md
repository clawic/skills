# Keys, TTLs and Scanning — The Keyspace You Have To Live With

Key design is schema design: it is the only index Redis gives you for free, and it is the thing you cannot refactor without a migration.

## Key Naming

- Shape: `<prefix>:<entity>:<id>:<attribute>` — `app:user:1042:profile`. Colon is the convention every tool (RedisInsight, exporters, `--bigkeys` output) already groups on.
- Put the type in the name when two shapes could collide (`...:profile` hash vs `...:profile:json` string). `WRONGTYPE` is almost always a naming collision.
- Version the prefix, not the value, when the shape changes: `app:v2:user:1042`. It makes the cutover a `SCAN` over the old prefix instead of a per-key inspection.
- Never build a key from unbounded user input without a bound: `app:search:<query>` grows one key per distinct query. Hash long or free-form components (`app:search:<sha1[:16]>`) and keep the mapping only if you need to debug it.
- Under Cluster, the hash tag is part of the name: `app:{user:1042}:profile` fixes which slot the key lands in.
- Prefixes are how you find things later: `SCAN 0 MATCH app:session:* COUNT 500` only works if sessions actually share a prefix.

## Numbered Databases Are Not Namespaces

`SELECT 3` gives you a separate keyspace on the same server, sharing memory, eviction, CPU, persistence and blocking. It is not isolation. Cluster only supports db 0, so any use of `SELECT` blocks a future move to Cluster. Use prefixes for logical separation and separate instances for real isolation.

## TTL Semantics

| Command | Effect on TTL |
|---|---|
| `SET k v` on an existing key | **Clears** the TTL — the number-one cause of keys that never expire |
| `SET k v KEEPTTL` (>=6.0) | Preserves the existing TTL |
| `SET k v EX 300` / `SETEX k 300 v` | Sets value and TTL in one atomic step |
| `GETEX k EX 300` (>=6.2) | Reads and re-arms the TTL — the sliding-session read |
| `GETEX k PERSIST` | Reads and removes the TTL |
| `EXPIRE k 300 NX\|XX\|GT\|LT` (>=7.0) | Conditional TTL: `GT` only extends, which is what a heartbeat wants |
| `HSET` / `LPUSH` / `ZADD` on an existing key | TTL is untouched — it belongs to the key, not the elements |
| `RENAME src dst` | The TTL travels with the value; any TTL on `dst` is discarded |
| `PERSIST k` | Removes the TTL, making the key immortal |

- `TTL k` returns `-1` (exists, no expiry) and `-2` (does not exist). Confusing the two turns a missing key into "cached forever" in monitoring dashboards.
- TTLs are absolute internally: they survive a restart and are replicated as an absolute time, so a replica promoted late does not restart the clock.
- Before Redis 7, a write to a *volatile* key on a replica could resurrect it in edge cases; on modern versions expiry is master-driven and replicas simply mask expired keys on read.

## How Expiry Actually Runs

Redis mixes two mechanisms (canonical description in `SKILL.md`, Expiration And Eviction):

1. **Lazy** — a key found expired on access is deleted then, and the read behaves as a miss.
2. **Active** — a cycle at `hz` (default 10) per second samples 20 keys with a TTL per database, deletes the expired ones, and repeats immediately while more than 25% of the sample was expired.

Consequences to design around:

- Memory is freed *after* the TTL, not at it. A million keys expiring at the same second are reclaimed over the following seconds, and `used_memory` lags the logical size.
- Mass-expiry at a round timestamp (everything set with `EX 3600` at deploy time) produces a reclaim spike and a stampede on the read side. Jitter TTLs.
- Keys that are never read and never sampled cost memory until the active cycle happens to reach them — one more reason bulk deletes use `SCAN` + `UNLINK` rather than "they will expire eventually".

## Deleting Many Keys Safely

```bash
redis-cli --scan --pattern 'app:session:*' | xargs -L 500 redis-cli UNLINK
```

- `--scan` uses `SCAN` under the hood; `KEYS` would block the server for the whole match (Core Rule 4).
- `UNLINK` frees the value in a background thread; `DEL` frees it on the main thread.
- Batch at 500 keys per call (`xargs -L 500`): one round trip per batch, and no single `UNLINK` argument list long enough to become its own stall.
- Under Cluster this must run per master node (`redis-cli --cluster call`), because `SCAN` is per node.

## SCAN Guarantees (And Non-Guarantees)

- Guaranteed: every element present in the keyspace for the whole iteration is returned at least once.
- Not guaranteed: no duplicates (dedupe on your side), and nothing at all about elements added or removed mid-iteration.
- The cursor is opaque state, not an offset: pass it back verbatim, and stop only when it returns `0`.
- `COUNT` is a hint per call, default 10 — far too small for a large keyspace. Use 500-1000; every call still returns quickly because the work per call is bounded.
- `MATCH` filters *after* the scan, so a highly selective pattern can return empty pages for many iterations. That is normal; do not treat an empty page as the end.
- `TYPE` (>=6.0) filters server-side: `SCAN 0 TYPE zset COUNT 500`.
- Collection variants (`HSCAN`, `SSCAN`, `ZSCAN`) have the same contract; `HSCAN ... NOVALUES` (>=7.4) returns field names only.

## Keyspace Notifications

Off by default: `notify-keyspace-events` is an empty string. Enable the classes you need, e.g. `Kx` for keyspace events about expirations, `KEA` for everything (loud and expensive).

- Delivery is Pub/Sub: fire-and-forget, at-most-once, nothing buffered for a disconnected subscriber. A restart during a deploy drops every event in that window, with nothing logged and no way to replay.
- The `expired` event fires when the key is *deleted* (lazily or by the active cycle), not at the TTL instant — so it is a poor scheduler.
- In Cluster, events are local to the node holding the key, so subscribers must connect to every node (or sharded Pub/Sub, Redis >=7.0).
- Reliable alternative for scheduling: a sorted set scored by due-timestamp, polled with `ZRANGEBYSCORE ... LIMIT` and claimed atomically.

## Auditing A Keyspace You Inherited

```bash
redis-cli INFO keyspace                 # keys= vs expires= per db: the TTL-less population
redis-cli --bigkeys                     # largest key per type (SCAN-based, safe on a live server)
redis-cli --memkeys                     # same walk, ranked by MEMORY USAGE
redis-cli --scan --pattern 'app:*' | head -100   # what the prefixes actually look like
redis-cli OBJECT ENCODING app:user:1    # packed or promoted encoding
```

Read `keys=` minus `expires=` first: that difference is the population that will never leave on its own, and it decides whether the next incident is an eviction storm or an OOM.
