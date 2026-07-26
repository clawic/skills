# Caching — Patterns, Invalidation, Stampede

Store-agnostic strategy (what to cache, hierarchy, hit-rate economics) belongs to the `caching` skill. This file is the Redis-side mechanics.

## Cache-Aside (The Default)

```
value = GET app:user:1042
if miss:
    value = load_from_source()
    SET app:user:1042 <value> EX <default_ttl ± jitter>
return value
```

- Write path: update the source of truth, then **delete** the cache key. Writing the new value into the cache instead re-introduces the race where two concurrent writers leave the older value resident.
- Cache the miss too, briefly: `SET app:user:1042 <sentinel> EX 30` stops a hot nonexistent id from hammering the database. Distinguish the sentinel from a real empty value or you will serve "not found" for real records.
- `GETEX k EX <ttl>` (>=6.2) turns read-and-extend into one round trip when the object should live as long as it is being used.

## TTL Is A Correctness Budget

- Pick the TTL from staleness tolerance, not from memory pressure: "how wrong may this be?" in seconds. Memory pressure is what `maxmemory` and eviction are for.
- Always jitter: `ttl = default_ttl × (1 ± 0.10)`. Every key created in the same deploy with the same fixed TTL expires in the same second, and the reload storm hits the source of truth at once.
- No TTL is only defensible when an explicit invalidation path exists *and* has a fallback. Otherwise you have built a leak (Core Rule 1).

## Stampede (Dogpile)

One popular key expires; N concurrent requests all miss and all recompute. Cost = N × the recompute, at exactly the moment traffic is highest.

Three fixes, in order of how much they cost to implement:

1. **Single-flight lock.** On a miss, `SET app:lock:user:1042 <token> NX EX 10`. Winner recomputes; losers sleep 20-50 ms and retry the read (bounded retries, then fall through to the source). Unique token, TTL and compare-and-delete release apply here too (SKILL.md Core Rule 6).
2. **Early recompute (probabilistic).** Store the value with its computation cost and recompute when the remaining TTL drops below a window — refresh when `TTL k` is under 10-20% of the original. One request refreshes while the others still get a valid value.
3. **Never-expiring value + separate freshness key.** `app:user:1042` has no TTL; `app:user:1042:fresh` has one. When the freshness key is gone, one caller refreshes and everyone else serves slightly stale data. Highest availability, most moving parts, and it needs its own eviction story.

## Invalidation

| Situation | Play |
|---|---|
| One record changed | `UNLINK` its key after the source-of-truth commit, not before |
| A computed view depends on many records | Version key: `GET app:ver:user:1042` participates in the cache key, and bumping it with `INCR` invalidates every derived entry at once |
| Whole class of keys must go (deploy, schema change) | Prefix version (`app:v2:...`) so the old generation simply stops being read and expires on its own; a `SCAN`+`UNLINK` sweep is the cleanup, not the switch |
| Two writers race on the same key | Delete-on-write plus a short TTL bounds the damage; if it must be exact, the cache is not the right place for that value |
| Anything else | Delete rather than update — a delete is idempotent and cannot install a stale value |

Never `FLUSHDB` to invalidate. It drops locks, sessions and rate-limit counters that share the instance, and the reload storm follows immediately.

## Consistency Order

Delete the cache **after** the source-of-truth write commits, and accept that a reader can slip in between. If that gap matters:

- Second delete after a short delay (delayed double delete) closes the common window at the cost of one extra operation.
- A write-through cache (write both in one path) removes the miss but not the race, and it makes the cache a participant in write availability.
- Full correctness across two systems is a distributed-transaction problem, not a caching one (`distributed-systems`).

## Client-Side Caching (Tracking)

Redis >=6 with RESP3 lets the server tell clients when a cached value changed: `CLIENT TRACKING ON` puts invalidation messages on the connection.

- Default mode: the server remembers which keys each client read and invalidates precisely, at the cost of a per-client key table.
- `BCAST` mode with `PREFIX app:user:` broadcasts invalidations for a prefix without per-client bookkeeping — cheaper server-side, noisier client-side.
- `OPTIN`/`OPTOUT` control which reads are tracked, so you can cache only the hot subset locally.
- Worth it when the same small set of keys is read thousands of times per second per process: it removes the round trip entirely (Core Rule 5). Not worth it when the working set is large or churn is high.
- Not all client libraries implement invalidation handling — check before designing around it.

## Measuring

- `INFO stats` → `keyspace_hits` and `keyspace_misses`. Hit rate = `hits / (hits + misses)`, server-wide; per-prefix rates need application metrics.
- A hit rate below ~80% on a read-through cache usually means the TTL is shorter than the reuse interval, or the key includes something unique per request (a timestamp, a request id, an unsorted query string).
- A hit rate near 100% with rising `evicted_keys` means the cache is doing its job while shrinking — capacity, not correctness.
- Cache what is expensive *and* reused. A 2 ms query cached for 60 s saves nothing; a 400 ms aggregate reused 50 times a minute is the whole point.

## Anti-Patterns Specific To Redis Caches

- Cache keys built from unsorted query parameters produce one key per permutation of the same request — canonicalize before hashing.
- Storing an entire rendered page as a 2 MB string: one `GET` moves 2 MB through the single-threaded server per request.
- `KEYS app:user:*` to count or clear a cache: a `SCAN` loop with `MATCH` and `COUNT 500` is the replacement (SKILL.md Core Rule 4).
- Using `volatile-lru` while some cache keys were written without a TTL: those keys are un-evictable and the instance OOMs with a mostly-cache dataset.
