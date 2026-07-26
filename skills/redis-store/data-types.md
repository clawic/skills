# Data Types — What Each Structure Costs

The selection table lives in `SKILL.md` (Choosing The Data Structure). This file is the memory math and the per-type edges that decide a design once the shortlist is down to two.

## Encodings: Small Collections Are Not Hash Tables

Below a size threshold Redis stores a collection as a **listpack** (a flat, contiguous byte array) instead of a real hash table or skiplist — a fraction of the memory, O(N) operations that are fast because N is tiny. Cross the threshold once and the conversion is permanent for that key, even if it shrinks again.

| Type | Packed while | Config (Redis >=7.0 names) |
|---|---|---|
| Hash | ≤128 fields and every value ≤64 bytes | `hash-max-listpack-entries` 128, `hash-max-listpack-value` 64 |
| Sorted Set | ≤128 members and every member ≤64 bytes | `zset-max-listpack-entries` 128, `zset-max-listpack-value` 64 |
| Set (integers only) | ≤512 members | `set-max-intset-entries` 512 |
| Set (mixed) | ≤128 members, ≤64 bytes each (Redis >=7.2) | `set-max-listpack-entries` 128, `set-max-listpack-value` 64 |
| List | Quicklist of listpacks, each ≤128 entries | `list-max-listpack-size` 128 |

- Verify with `OBJECT ENCODING key` — `listpack`/`intset` is packed, `hashtable`/`skiplist`/`quicklist` is not.
- Design consequence: 1M small objects as 1M hashes of 5 fields each stay packed and cost roughly an order of magnitude less than 1M hashes of 200 fields. Sharding a huge hash into `obj:{id mod 1000}` buckets is the classic memory win — and the classic readability cost.
- Do not raise the thresholds to "save memory": beyond a few hundred entries the O(N) scans start showing in `SLOWLOG`.

## Per-Key Overhead

Budget on the order of 60-100 bytes per top-level key before its value: the dict entry, the object header, the key string, and another allocation if the key carries a TTL. That is why 10M keys of 20 bytes each cost far more than one 200 MB value.

- Measure, do not estimate: `MEMORY USAGE key SAMPLES 0` returns the real bytes including overhead (`SAMPLES 0` measures every element of a collection instead of sampling).
- Fewer, bigger keys is a memory win and a latency risk: a 100k-field hash is one `HGETALL` away from a stall (Core Rule 4).
- Key names are stored verbatim: `application:user:session:token:12345` × 10M is ~380 MB of key names alone. Short prefixes are real money at scale.

## Strings

- One value, 512 MB ceiling. Anything above a few hundred KB is a bad fit: it blocks the server during transfer and fragments the allocator.
- `INCR`/`INCRBY`/`INCRBYFLOAT` are atomic and create the key at 0 — no read-modify-write anywhere in the loop.
- `SETRANGE`/`GETRANGE` edit in place; `APPEND` is amortized O(1) but grows the allocation geometrically.
- `GETDEL` (>=6.2) and `GETEX` (>=6.2) collapse two-command patterns into one atomic call — `GETEX k EX 300` is the sliding-session read.
- Integers under 10000 are shared objects; a counter costs nothing until it grows.

## Hashes

- Field-level operations (`HSET`, `HINCRBY`, `HDEL`) beat serializing a JSON blob whenever two writers touch different fields.
- No TTL per field before Redis 7.4 (`HEXPIRE` and friends). Below that, model per-field expiry as separate keys or a sorted set of due timestamps.
- `HRANDFIELD` (>=6.2) samples without pulling the whole hash.
- `HGETALL` on a large hash is the most common accidental O(N): fetch named fields with `HMGET`, or iterate with `HSCAN`.

## Lists

- O(1) at both ends, O(N) in the middle. `LINSERT`/`LREM`/`LSET` on a long list are scans.
- `LMOVE`/`BLMOVE` (>=6.2) replace `RPOPLPUSH`/`BRPOPLPUSH` and give the reliable-queue pattern a direction argument.
- `LPUSH` + `LTRIM 0 999` is the canonical capped log — trim on every write, not on a schedule.
- `LPOS` (>=6.0.6) finds an element's index without pulling the list.
- Blocking pops hold a connection: size the pool for them.

## Sets

- `SISMEMBER` is O(1); `SMISMEMBER` (>=6.2) batches the check.
- `SINTERCARD` (>=7.0) with a `LIMIT` answers "do these overlap by at least N" without building the intersection.
- `SPOP count` is a random sample *and* a delete; `SRANDMEMBER count` samples without removing (negative count allows repeats).
- `SMEMBERS` on a big set is the same trap as `HGETALL`; use `SSCAN`.

## Sorted Sets

- Scores are IEEE-754 doubles: integers are exact only up to 2^53. Millisecond timestamps (~2^41 today) are safe; nanosecond timestamps and 64-bit ids are not.
- Ties break by member lexicographically. To break ties by insertion time inside a score, keep the timestamp *in the member string*, not in the score's low bits.
- Range families: `ZRANGEBYSCORE` (numeric window), `ZRANGEBYLEX` (all-equal scores, prefix search), `ZRANGESTORE` (>=6.2, store the slice server-side instead of shipping it).
- `ZADD GT`/`LT`/`NX`/`XX` (>=6.2) make "only raise the score" a single atomic op — the basis of last-seen indexes and sliding windows.
- `ZRANGEBYSCORE ... LIMIT 0 N` + `ZREM` is the priority queue; make the claim atomic in Lua or two workers claim the same member.

## Streams

- Append-only log with consumer groups. Entry ids are `<ms>-<seq>`; `XADD key *` generates them monotonically.
- Memory is the retained entries, not the unacked ones: `XACK` only clears the pending list. Trim explicitly — `XADD key MAXLEN ~ 100000 *` (the `~` trims at radix-node boundaries, which is cheap) or `XTRIM key MINID ~ <ms>` when the retention rule is time, not count.
- One stream, many groups: each group has its own cursor and pending list, so fan-out is free. Within a group, each entry goes to exactly one consumer.
- `XAUTOCLAIM` (>=7.0) replaces the `XPENDING` + `XCLAIM` loop for recovering entries from a dead consumer.
- Length is O(1) (`XLEN`), range reads are O(log N) to seek plus the returned count.

## Bitmaps, HyperLogLog, Bitfield

- Bitmaps are Strings: `SETBIT users:active:2026-07-26 <userid> 1`, `BITCOUNT`, `BITOP AND/OR` across days. Cost = highest id / 8 bytes, so dense sequential ids only — a bitmap keyed by random 64-bit ids is a 512 MB accident.
- HyperLogLog: 12 KB per counter at 0.81% standard error, `PFADD`/`PFCOUNT`/`PFMERGE`. Sparse encoding keeps small cardinalities far under 12 KB. No membership test, no removal, and `PFCOUNT` over several keys is a merge — not free.
- `BITFIELD` packs many small counters into one string (e.g. 8-bit per-user counters with `OVERFLOW SAT`), trading readability for density.
- Choosing between them: exact and needs membership → Set; exact count only, dense ids → Bitmap; approximate count, any ids → HyperLogLog.

## Geo And Time Series

- Geo commands are a sorted set of geohash scores. `GEOSEARCH`/`GEOSEARCHSTORE` (>=6.2) supersede `GEORADIUS`; radius queries are O(N+log(M)) in the returned area, so bound the radius and the `COUNT`.
- Time series in plain Redis is a sorted set scored by timestamp plus an `XTRIM`-style retention job; when downsampling, compaction or labels are needed, that is the TimeSeries module.

## Type Migration

Changing a key's type is not an operation — it is a new key. `WRONGTYPE` in production usually means a deploy started writing the new shape under the old name. Migrate with a versioned prefix (`app:v2:user:1`), dual-write during the overlap, then delete the old prefix with a `SCAN`-and-`UNLINK` pass.
