# Aggregation Pipeline

## What the Optimizer Already Does

- Moves `$match` earlier past `$project`/`$addFields` when the matched fields are untouched; merges consecutive `$match` stages; coalesces `$sort` + `$limit` into a top-k sort that holds only `limit` documents in memory.
- Dependency analysis prunes unused fields automatically — the classic "add `$project` early for performance" advice is mostly obsolete and a premature `$project` can even block `$match` reordering. Add `$project` for output shape; the performance exception is cutting one huge field (raw text, blob) before a blocking stage.
- Only a `$match` that reaches the front of the (rewritten) pipeline uses indexes. Verify with `explain()` on the aggregation: the first `$cursor` stage contains the actual query plan — trust that, not your stage order.

## Streaming vs Blocking

- Streaming stages (`$match`, `$project`, `$addFields`, `$unwind`, `$limit`) pass documents through as they arrive. Blocking stages (`$group`, `$sort` without index support, `$bucket`, `$facet`, `$setWindowFields`) must consume ALL input before emitting anything.
- Each blocking stage gets 100MB of memory. Since 6.0 they spill to disk by default (`allowDiskUseByDefault: true`); before 6.0 pass `allowDiskUse: true` or the pipeline fails with error 292 (→ `errors.md`).
- Spilling shows as `usedDisk` in explain and the profiler. A pipeline that spills on every run needs an index or schema fix, not a bigger limit — disk sort is an order of magnitude slower.
- One blocking stage early in a pipeline is a design decision; two blocking stages in a row means the second one waits for the first to finish entirely, and the pipeline's latency is their sum.

## $group

- Memory is proportional to the number of groups: `_id: null` is one accumulator; grouping on a high-cardinality field materializes every distinct value.
- `$first`/`$last` are meaningless without a preceding `$sort` — they take whatever order the input arrived in.
- `$push`/`$addToSet` inside `$group` can build a single output document past 16MB → hard error at the end of an expensive run. Cap or count instead.
- `$sum` over a missing or non-numeric field contributes 0 silently — no error, wrong total. Guard dirty data with `$convert`/`$isNumber`.
- `$group` normally can't use indexes, with one exception worth checking in explain: grouping on an index prefix with only `$first` accumulators can hit DISTINCT_SCAN.
- `$count` is shorthand for `$group` + `$project`; on an unfiltered collection prefer `estimatedDocumentCount()` (SKILL.md Traps).

## $lookup

- Classic form (`localField`/`foreignField`): the join is one index lookup per input document — index the `foreignField` on the foreign collection or it degrades to a scan per input doc.
- Pipeline form with `$expr`: uses indexes only since 5.0. Before that, each input document triggers a full foreign-collection scan — O(N×M), the classic "worked in dev, died in prod."
- 6.0's slot-based engine made equality `$lookup` markedly faster — but a `$lookup` in every hot query is still a schema signal: extended reference or embed (→ `schema.md`).
- Result is always an array, even for 1:1 — follow with `$unwind {preserveNullAndEmptyArrays: true}` or take `$first`.
- Put `$limit` BEFORE the `$lookup` whenever the pipeline ends in a page: joining 10,000 documents to return 20 does the join 500 times too often.
- `$graphLookup` (recursive/tree traversal) ignores `allowDiskUse`: hard 100MB limit — bound the depth with `maxDepth` on deep graphs, and prefer materialized paths for read-heavy hierarchies (→ `schema.md`).
- On a sharded cluster, the foreign collection of a `$lookup` has restrictions and the join executes on the merging node — check the plan before assuming it distributes (→ `sharding.md`).

## $unwind and Its Alternatives

- Don't unwind just to inspect an array: `$filter` (subset), `$map` (transform), `$reduce` (fold), `$size` (count), `$sortArray` (MongoDB >=5.2), `$slice` all work in place without multiplying documents.
- `$unwind` emits one document per element (100-element array → 100 docs); a missing or empty array DROPS the document unless `preserveNullAndEmptyArrays: true`; `includeArrayIndex` keeps position.
- Unwind → group to recount or dedupe is fine at the END of a filtered pipeline (small N), deadly at the start of an unfiltered one.

## $facet

- Runs sub-pipelines over the same input, but sub-pipelines inside `$facet` never use indexes — do the `$match`/`$sort` BEFORE the facet stage and keep facets to reshaping and counting.
- Output is one document → the combined result caps at 16MB.
- Standard page+count in one pass: `$facet: {data: [{$skip}, {$limit}], total: [{$count: "n"}]}` — after an index-backed `$match`. For deep pages, range pagination plus a cached total beats this (SKILL.md Query Semantics).

## Window Functions and Gap Filling

- `$setWindowFields` (MongoDB >=5.0) computes running totals, moving averages, rank, and lag/lead without self-joins: `partitionBy`, `sortBy`, then `output` with a `window` in documents or in a time `range`.
- It is a blocking stage and partitions independently — memory scales with the largest partition, not the collection. A partition key with one giant group is the failure mode.
- `$rank`/`$denseRank`/`$documentNumber` replace the unwind-and-count-yourself pattern entirely.
- `$densify` (MongoDB >=5.1) inserts the missing time buckets that a chart needs; `$fill` (MongoDB >=5.3) carries values forward or interpolates them. Together they remove the "zero-fill in application code" step that used to follow every time-bucketed `$group` (→ `time-series.md`).

## Stages With Position Rules

Four stages must sit in a specific place, and the error message when they do not is rarely the obvious one:

- `$search` and `$vectorSearch` — FIRST, always (→ `search.md`).
- `$geoNear` — FIRST, and it requires a 2dsphere or 2d index on the queried field. It also sorts by distance implicitly, so a later `$sort` silently throws that work away. Use `$geoWithin` inside a plain `$match` when you need containment rather than ordered proximity (→ `indexes.md`).
- `$out` and `$merge` — LAST, and nothing may follow them.
- `$indexStats`, `$collStats`, `$currentOp`, `$planCacheStats` — first only; they are metadata sources, not transformations (→ `monitoring.md`).

## Views

- `db.createView(name, source, pipeline)` stores the pipeline, not the results: every read runs it. Read-only, and it composes — a view can source another view.
- Queries against a view can use the SOURCE collection's indexes, but only for the parts of the query the optimizer can push down into the pipeline's leading `$match`. A view whose pipeline starts with `$group` is a full computation on every read.
- Right for enforcing a projection (hiding fields from a role — a view plus a role granted only on the view is real access control, → `security.md`) and for freezing a canonical join shape. Wrong as a performance tool: for that you want a materialized collection below.

## Materialized Views: $merge and $out

- `$out` atomically replaces the target (build-then-rename), keeps the target's existing indexes, and fails if results violate them (e.g., unique) — a useful safety net.
- `$merge` (MongoDB >=4.2) upserts incrementally: `whenMatched: replace|merge|keepExisting|fail` or a custom pipeline. The incremental pattern: filter source by `updatedAt > lastRun`, `$merge` on `_id`, run on a schedule; dashboards read the materialized collection at find() speed.
- `$merge` into the collection you are reading is legal but pays twice; write to a separate target and swap names if you need atomic replacement of a live read model.
- Both write to the primary and count as normal writes for replication and oplog purposes — a nightly rebuild of a large view is a replication-lag event (→ `replication.md`).

## Reading and Writing Pipeline Code

- Develop with Compass's aggregation builder or against a frozen sample; the stage-by-stage preview finds shape errors faster than any amount of reasoning (→ `mongosh.md`).
- Expression operators are prefixed and take arrays or objects inconsistently by design (`$add: [a, b]` vs `$filter: {input, cond}`). When an operator silently returns null, the arity is the first suspect.
- `$$ROOT`, `$$NOW`, `$$CURRENT` and user variables via `$let` keep pipelines readable; `$$REMOVE` conditionally omits a field, which is the clean alternative to emitting nulls.
- Pipelines are also an update syntax: `updateMany(filter, [{$set: {...}}])` lets one field's new value depend on another field's current value — impossible with the classic update operators.

## Debugging Procedure

1. Freeze a sample: `$match` on a known small set + `$limit: 20`; develop against it.
2. Add one stage at a time; check the output shape after each — most pipeline bugs are a stage receiving a shape it didn't expect.
3. Full-data dry run with `explain("executionStats")`: check the `$cursor` plan (IXSCAN?), `usedDisk`, and per-stage `nReturned` — the stage where the count explodes is the cost.
4. In production, slow pipelines land in the profiler with the full pipeline body — look for spills and COLLSCAN there before touching code (→ `slow-queries.md`).
