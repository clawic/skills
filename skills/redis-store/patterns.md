# Recipes — Sessions, Counters, Leaderboards, Dedup, Search Aids

The standard jobs, each with its structure, its expiry story, and the failure it is designed around. Caching, queues and locks have their own guides; this file is everything else.

Contents: Sessions · Counters · Rate Limiting · Idempotency And Dedup · Leaderboards And Ranking · Autocomplete And Prefix Search · Recently-Viewed / Capped History · Feature Flags And Hot Config · Geospatial Proximity · Fan-Out Timelines

## Sessions

```bash
HSET app:session:<sid> user_id 1042 csrf <token> ip 1.2.3.4
EXPIRE app:session:<sid> 1800
# every authenticated request, sliding window:
HGETALL app:session:<sid>   # then EXPIRE app:session:<sid> 1800
```

- Hash, not a JSON string: touching `last_seen` should not rewrite the whole object.
- Sliding expiry costs one extra command per request; `GETEX` gives it for free on string sessions, hashes need the explicit `EXPIRE`.
- Logout must delete the key server-side. A stateless token you cannot revoke is not a session.
- "Log out all devices" needs a reverse index: `SADD app:user:1042:sessions <sid>` alongside, cleaned up when a session expires (a `SCAN` sweep or a TTL on the index itself, since the set does not shrink on its own).
- Never put a session store on an `allkeys-*` instance and call the result a bug — eviction will log people out.

## Counters

- Simple: `INCR app:views:post:88`. Atomic, creates at 0, no read-modify-write anywhere.
- Bucketed by time: `INCR app:views:post:88:2026-07-26` + `EXPIRE` on creation. Same TTL trap as rate limiting below — set the expiry atomically.
- Many counters for one entity: `HINCRBY app:stats:post:88 views 1` keeps them in one packed hash.
- Approximate high-cardinality: `PFADD app:uniq:2026-07-26 <user>` at 12 KB per counter and 0.81% standard error, merged across days with `PFMERGE`.
- Dense boolean per user per day: `SETBIT app:active:2026-07-26 <userid> 1`, then `BITCOUNT` for the day and `BITOP AND` across days for retention — only with dense sequential ids.

## Rate Limiting

Algorithm choice (fixed window vs sliding vs token bucket vs GCRA) belongs to the `rate-limiting` skill. The Redis mechanics:

**Fixed window**, cheapest, boundary-burst-prone:

```lua
-- KEYS[1]=app:rl:<id>:<minute>, ARGV[1]=limit, ARGV[2]=window-seconds
local n = redis.call('INCR', KEYS[1])
if n == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end
return n <= tonumber(ARGV[1]) and 1 or 0
```

The Lua is not decoration: `INCR` then `EXPIRE` as two round trips can crash in between and leave a counter with no TTL, which locks that identity out permanently (SKILL.md Traps).

**Sliding window log**, exact, costs memory per request:

```bash
ZREMRANGEBYSCORE app:rl:<id> -inf <now-window-ms>
ZADD app:rl:<id> <now-ms> <unique-request-id>
ZCARD app:rl:<id>
EXPIRE app:rl:<id> <window-seconds>
```

All four in one Lua script. Memory = one sorted-set member per request in the window per identity: at 1000 req/min per user this is fine, at 100k req/min it is not — that is when a token bucket in two hash fields (tokens, last refill timestamp) wins.

## Idempotency And Dedup

```bash
SET app:idem:<key> <result-id> NX EX 86400
```

- Reply `OK` → you are the first, do the work. Reply `nil` → someone already did it; read the stored result.
- The TTL is the dedup window and it must exceed the maximum retry horizon of every caller in the chain, including a client that retries an hour later.
- Store the *result* (or a pointer to it), not just a flag, or the duplicate caller has nothing to return.
- Two-phase variant for slow work: `SET ... NX` with a short TTL as an in-progress marker, then overwrite with the result and a long TTL. Callers that see the marker retry rather than duplicating.
- Deduplicating a firehose of ids where storage matters: a Bloom filter accepts a false-positive rate in exchange for constant memory.

## Leaderboards And Ranking

```bash
ZADD app:board:weekly GT <score> user:1042     # GT: only raise (>=6.2)
ZREVRANGE app:board:weekly 0 9 WITHSCORES      # top 10
ZREVRANK app:board:weekly user:1042            # my position, 0-based
ZCOUNT app:board:weekly <myscore> +inf         # how many are above me
```

- Ties break lexicographically by member. For "earlier submission wins", encode the timestamp inside the member string, not in the score's low bits — scores are doubles, exact only to 2^53.
- Windowed boards: one key per period (`app:board:2026-W30`) with a TTL, plus `ZUNIONSTORE` to build "last 4 weeks" on a schedule rather than per request.
- Around-me pages: `ZREVRANK` then `ZREVRANGE rank-5 rank+5` — two calls, no full scan.
- Millions of members is fine; the cost is O(log N) writes and O(log N + M) range reads.

## Autocomplete And Prefix Search

- All members at score 0 in one sorted set, then `ZRANGEBYLEX key "[pre" "[pre\xff"` returns everything with that prefix in lexicographic order.
- Store `term:id` as the member so the id travels with the match, and lowercase/normalize on write — `ZRANGEBYLEX` is byte-ordered, not locale-aware.
- Ranking by popularity inside a prefix needs a second structure (a hash of scores) or the Search module.

## Recently-Viewed / Capped History

```bash
LPUSH app:recent:user:1042 post:88
LTRIM app:recent:user:1042 0 49     # same pipeline, every write
```

Dedupe by removing first (`LREM key 1 post:88`) or use a sorted set scored by timestamp with `ZREMRANGEBYRANK 0 -51` when the same item can reappear.

## Feature Flags And Hot Config

- Hash per flag set (`HGETALL app:flags` is fine: it is small and packed), read at process start and cached locally.
- Invalidate with Pub/Sub: publish `app:flags:changed`, subscribers re-read. A missed message costs staleness, not correctness.
- Percentage rollout without storing per user: bucket a stable hash of the user id (`CRC32(uid) % 100 < rollout`). Storing a set of enrolled users is only necessary when the enrollment must be sticky and auditable.

## Geospatial Proximity

```bash
GEOADD app:drivers <lon> <lat> driver:77
GEOSEARCH app:drivers FROMLONLAT <lon> <lat> BYRADIUS 3 km ASC COUNT 20
```

- `GEOSEARCH`/`GEOSEARCHSTORE` (>=6.2) supersede `GEORADIUS`. Always bound with `COUNT`; the command is O(N+log(M)) in the searched area.
- It is a sorted set: `ZREM` removes a member, and freshness needs a separate TTL strategy (a per-driver key with a TTL plus a sweeper, since members of a sorted set cannot expire individually).

## Fan-Out Timelines

- Write-heavy fan-out: on post, `LPUSH` the post id into each follower's timeline list, `LTRIM` to a window. Read is O(1); a celebrity write is O(followers).
- Read-heavy fan-in: keep one list per author, and merge at read time with `ZUNIONSTORE` over the authors a user follows. Write is O(1); read costs more.
- The standard compromise: fan-out for normal accounts, fan-in for accounts above a follower threshold, merged at read. State the threshold explicitly; it is the whole design.
