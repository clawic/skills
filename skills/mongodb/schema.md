# Schema Design Patterns

## Decision Procedure (workload first)

1. List the top queries and writes with expected frequency BEFORE drawing entities. A schema optimized for the wrong access pattern is worse than no design.
2. For each relationship ask three questions in order: read together with the parent? (pull toward embed) — bounded? (unbounded → never embed) — shared across parents or updated on its own cadence? (→ reference).
3. Shape documents so every hot query resolves in one round trip; accept duplication and write fan-out on cold paths to buy that.
4. Stress the shape at P99: estimate document size after a year of growth. If the answer involves "the array keeps growing," you already failed step 2.

## Embed vs Reference Thresholds

- Embed tens of subdocuments comfortably; at low hundreds, check document size and multikey index cost; unbounded, never — growth per user action is the tell.
- Reference when the child is shared across parents, queried independently, larger than the parent, or hot while the parent is cold (every child update rewrites the entire parent document — SKILL.md rule 2).
- The middle options most teams miss:
  - **Extended reference**: store the id PLUS the 2-3 fields you always display (`{authorId, authorName, authorAvatar}`). Kills the majority of `$lookup`s; you own updating the copies when the source changes.
  - **Subset**: embed the newest N (e.g., 10 most recent reviews) for the product page; full history lives in its own collection. Hot path stays one read.

## Pattern Catalog

| Situation | Pattern | Mechanics |
|---|---|---|
| Timestamped measurements (IoT, metrics, logs) | Time-series collection (5.0+) | Native columnar buckets; pre-5.0, manual bucketing: `{sensor, hour, readings: [], count}` — roll a new bucket at a fixed count |
| Products with variable/unknown attributes | Attribute pattern or wildcard index | `attrs: [{k, v}]` + index `{attrs.k: 1, attrs.v: 1}`; since 4.2 a wildcard index often removes the need to reshape |
| Expensive aggregation repeated per read | Computed pattern | Maintain the value on write (`$inc` counters are atomic — no read-modify-write); keep a rebuild job as backstop against drift |
| A few documents blow the size rules | Outlier pattern | Flag `has_overflow: true`, spill the tail to a side collection; the hot path never pays for the outliers |
| Mixed types in one collection | Polymorphic | `type` discriminator field; partial indexes per type: `{age: 1}, {partialFilterExpression: {type: "person"}}` |
| Schema evolves under live traffic | Schema versioning | `schema_version` field; handle both shapes on read, migrate lazily on write, backfill job when convenient — never a big-bang migration |

Default: start embedded, split the moment a row above triggers. Splitting an embedded doc is a migration; un-splitting a reference is just a slower read.

## Write Fan-Out Reality

- A denormalized copy means you own its update path. Enumerate every copy at design time; update via `updateMany` and accept eventual consistency, or use a transaction only when staleness is user-visible.
- `$inc`, `$push`, `$addToSet`, `$min`/`$max` are atomic per document — design counters and sets so concurrent writers never read-modify-write in application code.
- `findOneAndUpdate` is the atomic claim primitive (job queues, seat holds): filter on current state, set the new state, read the returned document. No transaction needed.

## Anti-Patterns

- Unbounded arrays — canonical rule and multikey cost in SKILL.md rule 3.
- Nesting beyond 3-4 levels: every query path spells the full chain, and arrays-inside-arrays are barely indexable (`$elemMatch` nesting). Flatten or split.
- Large blobs inline: a 2MB document evicts roughly five hundred 4KB documents from cache on every read. Files go to GridFS or object storage with a URL in the document.
- Collection sprawl (per-tenant, per-day collections): each collection and index is a separate WiredTiger file — checkpoint and open-handle costs grow with the count.
- Exposing `_id` as a public business identifier: enumerable and leaks creation time (SKILL.md, ObjectId).
