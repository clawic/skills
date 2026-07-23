# Aggregation Pipeline

## What the Optimizer Already Does

- Moves `$match` earlier past `$project`/`$addFields` when the matched fields are untouched; merges consecutive `$match` stages; coalesces `$sort` + `$limit` into a top-k sort that holds only `limit` documents in memory.
- Dependency analysis prunes unused fields automatically — the classic "add `$project` early for performance" advice is mostly obsolete and a premature `$project` can even block `$match` reordering. Add `$project` for output shape; the performance exception is cutting one huge field (raw text, blob) before a blocking stage.
- Only a `$match` that reaches the front of the (rewritten) pipeline uses indexes. Verify with `explain()` on the aggregation: the first `$cursor` stage contains the actual query plan — trust that, not your stage order.

## Streaming vs Blocking

- Streaming stages (`$match`, `$project`, `$addFields`, `$unwind`, `$limit`) pass documents through as they arrive. Blocking stages (`$group`, `$sort` without index support, `$bucket`, `$facet`, `$setWindowFields`) must consume ALL input before emitting anything.
- Each blocking stage gets 100MB of memory. Since 6.0 they spill to disk by default (`allowDiskUseByDefault: true`); before 6.0 pass `allowDiskUse: true` or the pipeline fails with QueryExceededMemoryLimit.
- Spilling shows as `usedDisk` in explain and the profiler. A pipeline that spills on every run needs an index or schema fix, not a bigger limit — disk sort is an order of magnitude slower.

## $group

- Memory is proportional to the number of groups: `_id: null` is one accumulator; grouping on a high-cardinality field materializes every distinct value.
- `$first`/`$last` are meaningless without a preceding `$sort` — they take whatever order the input arrived in.
- `$push`/`$addToSet` inside `$group` can build a single output document past 16MB → hard error at the end of an expensive run. Cap or count instead.
- `$sum` over a missing or non-numeric field contributes 0 silently — no error, wrong total. Guard dirty data with `$convert`/`$isNumber`.
- `$group` normally can't use indexes, with one exception worth checking in explain: grouping on an index prefix with only `$first` accumulators can hit DISTINCT_SCAN.

## $lookup

- Classic form (`localField`/`foreignField`): the join is one index lookup per input document — index the `foreignField` on the foreign collection or it degrades to a scan per input doc.
- Pipeline form with `$expr`: uses indexes only since 5.0. Before that, each input document triggers a full foreign-collection scan — O(N×M), the classic "worked in dev, died in prod."
- 6.0's slot-based engine made equality `$lookup` markedly faster — but a `$lookup` in every hot query is still a schema signal: extended reference or embed.
- Result is always an array, even for 1:1 — follow with `$unwind {preserveNullAndEmptyArrays: true}` or take `$first`.
- `$graphLookup` (recursive/tree traversal) ignores `allowDiskUse`: hard 100MB limit — bound the depth with `maxDepth` on deep graphs.

## $unwind and Its Alternatives

- Don't unwind just to inspect an array: `$filter` (subset), `$map` (transform), `$reduce` (fold), `$size` (count), `$sortArray` (5.2+), `$slice` all work in place without multiplying documents.
- `$unwind` emits one document per element (100-element array → 100 docs); a missing or empty array DROPS the document unless `preserveNullAndEmptyArrays: true`; `includeArrayIndex` keeps position.
- Unwind → group to recount or dedupe is fine at the END of a filtered pipeline (small N), deadly at the start of an unfiltered one.

## $facet

- Runs sub-pipelines over the same input, but sub-pipelines inside `$facet` never use indexes — do the `$match`/`$sort` BEFORE the facet stage and keep facets to reshaping and counting.
- Output is one document → the combined result caps at 16MB.
- Standard page+count in one pass: `$facet: {data: [{$skip}, {$limit}], total: [{$count: "n"}]}` — after an index-backed `$match`.

## Materialized Views: $merge and $out

- `$out` atomically replaces the target (build-then-rename), keeps the target's existing indexes, and fails if results violate them (e.g., unique) — a useful safety net.
- `$merge` (4.2+) upserts incrementally: `whenMatched: replace|merge|keepExisting|fail` or a custom pipeline. The incremental pattern: filter source by `updatedAt > lastRun`, `$merge` on `_id`, run on a schedule; dashboards read the materialized collection at find() speed.

## Debugging Procedure

1. Freeze a sample: `$match` on a known small set + `$limit: 20`; develop against it.
2. Add one stage at a time; check the output shape after each — most pipeline bugs are a stage receiving a shape it didn't expect.
3. Full-data dry run with `explain("executionStats")`: check the `$cursor` plan (IXSCAN?), `usedDisk`, and docs examined.
4. In production, slow pipelines land in the profiler with the full pipeline body — look for spills and COLLSCAN there before touching code.
