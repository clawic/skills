# Modules — JSON, Search, Vectors, Bloom, TimeSeries

Modules add commands and index structures that core Redis has no answer for. Before adopting one, settle two questions: is it available on your deployment, and does it change your operational story (memory, persistence, upgrade path)?

## When A Module Is The Right Answer

| Need | Core Redis answer | Module answer | Take the module when |
|---|---|---|---|
| Update one field of a nested document | Store a hash, or read-modify-write a JSON string | `JSON.SET doc $.a.b value` | The document is genuinely nested and partially updated |
| Query by a field's value | Maintain your own index sets/sorted sets | Search index over hash or JSON | You have more than two or three secondary indexes to keep in sync |
| Full-text search with ranking | `ZRANGEBYLEX` prefix tricks | Search with stemming, scoring, aggregations | Real text search, not prefix matching |
| Vector similarity | None | KNN and hybrid filter queries over vector fields | You are serving embeddings and already run Redis |
| Membership at huge scale with a false-positive budget | Set (exact, O(N) memory) | Bloom / Cuckoo filter | The exact set does not fit memory and false positives are acceptable |
| Time series with retention and downsampling | Sorted set by timestamp + a trim job | TimeSeries with rules, labels, compaction | You need downsampling and label queries, not just a window |
| Anything else | Core structures | — | Stay in core: no extra dependency, no availability question |

Redis 8 ships several of these capabilities in the core distribution rather than as separate installs; older self-hosted versions and some providers require explicit modules. Confirm against your actual server with `MODULE LIST` (state of packaging verified 2026-07).

## Secondary Indexes: The Real Trade

The reason to reach for Search is not speed, it is *synchronization*. Hand-rolled indexes mean every write path must update every index, atomically, forever:

```bash
# hand-rolled: three writes that must not diverge
HSET app:user:1042 email a@b.com status active
SADD app:idx:status:active 1042
ZADD app:idx:created 1690000000 1042
```

One code path that forgets one line, or one crash between commands (without Lua to make them one atomic unit), and the index lies. An index maintained by the server cannot drift.

The cost you take on: index memory (often comparable to the data), reindexing time on schema changes, and a query language that is not the rest of your Redis code.

## JSON

- `JSON.SET`, `JSON.GET` with JSONPath, `JSON.NUMINCRBY`, `JSON.ARRAPPEND` — partial reads and writes without moving the whole document.
- Versus a hash: hashes are cheaper and packed at small sizes; JSON wins on nesting and arrays.
- Versus a JSON string in a plain key: the string forces read-modify-write over the network for every field change, which is the lost-update trap in `SKILL.md` Traps.
- Document size still matters: a 5 MB document is a 5 MB reply on the single thread if you `JSON.GET` the root.

## Search

- Define the index once (`FT.CREATE` over a hash or JSON prefix), then query with `FT.SEARCH` / `FT.AGGREGATE`.
- The index tracks the keyspace prefix automatically: writes through normal `HSET`/`JSON.SET` update the index.
- Costs: memory for the index, and an index rebuild when the schema changes. Size both before adopting.
- In Cluster, index and data distribution have their own rules — check the version's documentation rather than assuming slot behaviour matches plain keys.

## Vectors

- A vector field inside a Search index; queries are KNN, optionally filtered by other indexed fields (hybrid search).
- Two index types: flat (exact, linear scan, fine to tens of thousands of vectors) and HNSW (approximate, sublinear, tunable recall). Choosing between them is a recall-vs-latency decision, not a correctness one.
- Memory is the honest constraint: `vectors × dimensions × 4 bytes` for float32, plus index overhead — 1M × 768 dims ≈ 3 GB before the graph structure. Compute this before designing around it.
- Redis is a reasonable vector store when the vectors are already colocated with hot metadata and the corpus fits memory; a dedicated store wins when it does not.

## Probabilistic Structures

- **Bloom** (`BF.ADD`, `BF.EXISTS`): no false negatives, tunable false-positive rate, no deletion. Sized by capacity and error rate at creation — a filter that outgrows its capacity degrades in accuracy.
- **Cuckoo** (`CF.*`): supports deletion, at a higher constant cost.
- **Count-Min Sketch** (`CMS.*`): approximate frequency counts in fixed memory.
- **Top-K** (`TOPK.*`): heavy hitters without keeping every counter.
- Core Redis already has HyperLogLog for cardinality (12 KB, 0.81% standard error) — do not add a module for that.

## TimeSeries

- `TS.CREATE` with retention and labels, `TS.ADD`, `TS.RANGE`, `TS.MRANGE` by label, plus compaction rules that downsample into coarser series automatically.
- Versus a sorted set: the sorted set is fine for a bounded recent window you trim yourself; TimeSeries earns its place when you need retention policies, label-based queries and downsampling.

## Operational Impact Of Any Module

- **Availability**: `MODULE LIST` on the actual server settles it. Some managed tiers offer a fixed subset.
- **Persistence**: module data is serialized into RDB/AOF by the module itself; a restore requires the module to be loaded *first*, or the load fails.
- **Upgrades**: engine and module versions are separately compatible; check both before a version jump.
- **Security**: `MODULE LOAD` is arbitrary native code and must be denied to every application user.
- **Portability**: a design built on Search does not move to a plain Redis, a Valkey build without it, or a provider that lacks it. That is a real lock-in cost, worth naming out loud before adopting.
