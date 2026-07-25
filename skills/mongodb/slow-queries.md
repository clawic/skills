# Slow Queries — Explain Plans and Triage

Tune from a plan, never from query text (SKILL.md rule 1). This file is the order in which to look and how to read what you find.

## Triage Order

Skipping to step 4 spends the fix on the wrong query.

1. **Find the real offender.** Profiler: `db.setProfilingLevel(1, {slowms: 100})`, then `db.system.profile.find().sort({millis: -1}).limit(10)`. A 5ms query called 10k/min beats a 2s report as a target — sort by total time, not by worst single run. On Atlas, Performance Advisor and the Query Profiler do this without enabling anything (→ `atlas.md`).
2. **Explain the offender**: `db.coll.find(q).sort(s).explain("executionStats")`. For an aggregation, `db.coll.explain("executionStats").aggregate(pipeline)`.
3. **Read three numbers before anything else**: `nReturned`, `totalKeysExamined`, `totalDocsExamined`. The diagnosis table below is entirely these three.
4. **Check for a blocking SORT stage** — a `SORT` node in the winning plan means the index did not provide the order.
5. **Fix one thing, re-explain, compare the same three numbers.** A prettier plan shape with the same `totalDocsExamined` fixed nothing.

## Reading the Three Numbers

| Pattern | Meaning | Fix |
|---|---|---|
| `keys = 0`, `docs` = collection size | COLLSCAN: no usable index | Create the index in ESR order (→ `indexes.md`) |
| `docs` ≫ `returned`, `keys` ≈ `docs` | Index found rows, but the filter that discards them is not in the index | Extend the compound index with the discarding field |
| `keys` ≫ `returned`, `docs` ≈ `returned` | Index scan is wide: wrong field order, or a range before the sort | Reorder Equality → Sort → Range |
| `docs = 0`, `keys` ≈ `returned` | Covered query — the best possible outcome | Nothing; protect it (any added projection field breaks coverage) |
| All three ≈ equal and small, still slow | Not the query: network round trips, cursor batching, cache misses, or a busy server (→ `monitoring.md`) |

## Plan Stages Worth Recognizing

- `COLLSCAN` — full scan. Fine on a collection of a few thousand documents that never grows; a bug anywhere else.
- `IXSCAN` → `FETCH` — normal: read index keys, then load documents. The `FETCH` count is `totalDocsExamined`.
- `IXSCAN` with no `FETCH` — covered query, `totalDocsExamined: 0`. Requires every filter and projection field in the index and `_id` excluded.
- `SORT` + `SORT_KEY_GENERATOR` — in-memory sort. Hard cap 32MB for `find()`; over it, the query errors instead of degrading (MongoDB >=4.4 accepts `.allowDiskUse()` to spill instead). This is a different limit from the aggregation pipeline's 100MB per blocking stage (→ `aggregation.md`).
- `LIMIT` above a `SORT` — top-k sort, holds only `limit` documents: cheap. `SORT` with no limit over a large input is the expensive shape.
- `PROJECTION_COVERED` vs `PROJECTION_SIMPLE` — the first confirms coverage; the second means documents were fetched.
- `DISTINCT_SCAN` — the planner skipped duplicate index keys; the good outcome for `distinct()` and for `$group` on an index prefix with `$first` only.
- `SHARDING_FILTER` — orphan filtering on a sharded cluster; its presence with a high discard count means orphaned documents are accumulating (→ `sharding.md`).

## When the Plan Looks Fine but the Query Is Slow

- **First run only.** Cold cache: the documents were not resident and every fetch was a disk read. Compare `executionTimeMillis` across two consecutive runs before concluding anything.
- **Plan changed under you.** The planner caches a winning plan per query shape and re-evaluates it periodically or when indexes change. A query that got slow with no code change is often a plan flip: `db.coll.aggregate([{$planCacheStats: {}}])` shows what is cached; `hint()` proves the better index exists; `planCacheSetFilter` pins the choice durably where `hint()` in application code would not survive a refactor.
- **Different shape, same text.** `{status: "a"}` and `{status: {$in: [...]}}` are different shapes with different plans. Explain the shape production actually sends, parameters included.
- **The sort is the cost.** `nReturned` small, `executionTimeMillis` large, `SORT` present: the index served the filter and nothing else.
- **Contention, not the query.** Check `db.currentOp()` for a long write on the same collection and WiredTiger ticket availability (→ `monitoring.md`).

## Aggregation Explain

- The plan lives in the first `$cursor` stage — that is the only part index selection applies to. Everything after is pipeline execution.
- `explain("executionStats")` on an aggregation reports per-stage `nReturned` and `executionTimeMillisEstimate`: the stage where document count explodes or time jumps is the target.
- `usedDisk: true` means a blocking stage spilled. Spilling on every run is an index or schema problem, not a memory setting (→ `aggregation.md`).

## Making a Query Bounded Rather Than Fast

Some queries cannot be made fast; make them safe:

- `maxTimeMS` on every user-facing query — without it a runaway query holds cursors and cache until someone notices. Error 50 is this working as designed (SKILL.md Error Codes).
- Cap the result: a query returning 200k documents to an API is a design bug that no index fixes.
- Move the recurring expensive aggregation to a materialized collection refreshed on a schedule with `$merge` (→ `aggregation.md`); the dashboard then reads at `find()` speed.
- Hide the index before dropping it (→ `indexes.md`) so a wrong "this index is useless" conclusion is reversible in seconds.

## Profiler Hygiene

- Level 1 logs operations over `slowms`; level 2 logs everything and is never acceptable in production — the writes themselves become the load.
- `system.profile` is a capped collection, default 1MB: it wraps in minutes on a busy database. To keep more, turn profiling off, drop it, recreate it larger, turn profiling back on.
- `slowOpSampleRate` (default 1.0) samples the log line rather than the profiler; lowering it reduces log volume but makes rare slow queries invisible.
- The profiler is per-database and does not survive a restart of the member. Set it in configuration if you want it always on.
