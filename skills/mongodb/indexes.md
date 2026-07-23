# Index Strategies

## ESR With a Worked Example

Query: `find({status: "shipped", total: {$gt: 100}}).sort({createdAt: -1})`

- **E**quality → `status`, **S**ort → `createdAt`, **R**ange → `total` ⇒ index `{status: 1, createdAt: -1, total: 1}`.
- Wrong order `{status: 1, total: 1, createdAt: -1}`: the range on `total` breaks index order, so the sort becomes a blocking in-memory sort — fine on 1k docs, a spill or failure on millions (100MB sort limit).
- `$in` with few values can be "exploded" into parallel index scans and keep the sort order; large `$in` lists behave like a range — treat them as R when in doubt, then verify with explain.

## Prefixes, Direction, Covering

- Prefix rule (canonical: SKILL.md rule 5): audit for redundant single-field indexes whenever you add a compound one.
- Sort direction matters only in compound sorts: `{a: 1, b: -1}` serves sort `{a: 1, b: -1}` and its exact reverse `{a: -1, b: 1}` — NOT `{a: 1, b: 1}`.
- Covered query: all filter + projection fields in the index and `_id` projected out (or in the index) → explain shows IXSCAN with `totalDocsExamined: 0`. Worth engineering for the top 2-3 hottest queries only.
- Low-cardinality equality alone (a `status` with 3 values) barely narrows anything — it earns its place only as the E-prefix of a compound index that continues with a sort or selective field.

## Special Index Types

- **Partial**: `{email: 1}, {partialFilterExpression: {status: "active"}}` — smaller and cheaper, but the query must repeat the predicate: `find({email: e})` will NOT use it; `find({email: e, status: "active"})` will. Partial supersedes the legacy sparse option.
- **Partial unique** — the fix for "unique but optional" fields: a plain unique index treats two missing fields as duplicate nulls; `{partialFilterExpression: {field: {$exists: true}}}` enforces uniqueness only where the field exists.
- **TTL**: `{expireAfterSeconds: 86400}` on a date field. The sweep runs ~every 60 seconds and deletes in batches — expiry is approximate, and only the primary deletes (secondaries replicate the deletes). Change the TTL with `collMod`, no rebuild.
- **Text**: one per collection, requires `$text` in the query. For relevance scoring, fuzzy matching, or facets, use Atlas Search (Lucene) — the built-in text index is the wrong tool past basic keyword match.
- **Wildcard** (4.2+): indexes arbitrary/unknown field paths — often replaces the attribute pattern. Good for single-path equality/range; not a substitute for designed compound indexes on known hot queries.
- **Collation** (`strength: 2`): case-insensitive equality and sort. The query must pass the identical collation or the index is skipped — set it at collection level to avoid per-query mistakes.
- **Hashed**: for hashed sharding only; supports equality, never ranges.

## Lifecycle: Add, Verify, Retire

1. **Build**: since 4.2 all builds are hybrid (non-blocking) — the old `background: true` option is gone. Builds still consume IO and default to 200MB build memory (`maxIndexBuildMemoryUsageMegabytes`); schedule off-peak.
2. **Verify**: run explain on the real query; confirm IXSCAN and the examined/returned ratio (SKILL.md rule 1).
3. **Audit**: `db.collection.aggregate([{$indexStats: {}}])` — stats are per-node and reset on restart. Check EVERY replica set member (secondary reads count there) and note uptimes before declaring an index unused.
4. **Retire**: hide the index first (4.4+, `hideIndex`) and wait through a full business cycle — weekly reports, month-end jobs. Hiding reverses instantly; rebuilding a dropped index on a large collection takes hours.

## Costs and Limits

- Every write updates every index: 10 indexes ≈ 10 B-tree writes per insert. Write-heavy collections should stay in the single digits.
- Hard limits — 64 indexes per collection, 32 fields per compound index — are diagnostic: hitting either means the query patterns were never designed, not that you need more room.
- Multikey cost is canonical in SKILL.md rule 3; additional constraint here: at most ONE array field per compound index — the planner rejects the second.
