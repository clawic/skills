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
| Timestamped measurements (IoT, metrics, logs) | Time-series collection (MongoDB >=5.0) | Native columnar buckets; details and the pre-5.0 manual bucketing fallback in `time-series.md` |
| Products with variable/unknown attributes | Attribute pattern or wildcard index | `attrs: [{k, v}]` + index `{attrs.k: 1, attrs.v: 1}`; since 4.2 a wildcard index often removes the need to reshape |
| Expensive aggregation repeated per read | Computed pattern | Maintain the value on write (`$inc` counters are atomic — no read-modify-write); keep a rebuild job as backstop against drift |
| A few documents blow the size rules | Outlier pattern | Flag `has_overflow: true`, spill the tail to a side collection; the hot path never pays for the outliers |
| Mixed types in one collection | Polymorphic | `type` discriminator field; partial indexes per type: `{age: 1}, {partialFilterExpression: {type: "person"}}` |
| Schema evolves under live traffic | Schema versioning | `schema_version` field; handle both shapes on read, migrate lazily on write, backfill job when convenient (→ `migrations.md`) |
| Tree or hierarchy (categories, org chart, threads) | Materialized paths, or child references | Store `path: "/electronics/audio/headphones"` and query with an anchored regex `/^\/electronics\//` (index-usable); `$graphLookup` only for genuinely recursive traversal (→ `aggregation.md`) |
| Reads and writes want different shapes | Read model alongside the write model | Keep the normalized truth, maintain a denormalized projection with `$merge` on a schedule or change streams in real time (→ `change-streams.md`) |
| Needs a state machine (order, job, ticket) | Status field + `findOneAndUpdate` guard | Filter on the CURRENT state in the update: `{_id, status: "pending"}` → `{$set: {status: "claimed"}}`. The filter is the lock; no transaction needed |
| Audit trail of who changed what | Append-only event collection, never an embedded array | The array version is the unbounded-array trap wearing a compliance hat (SKILL.md rule 3) |

Default: start embedded, split the moment a row above triggers. Splitting an embedded doc is a migration; un-splitting a reference is just a slower read.

## Multi-Tenancy

Three layouts, in order of how often they are the right answer:

| Layout | When it fits | What it costs |
|---|---|---|
| `tenant_id` field, shared collections | The default for SaaS at any tenant count | Every index must lead with `tenant_id`, and every query must include it — one missed filter is a cross-tenant leak |
| Database per tenant | Tens of tenants, hard isolation or per-tenant restore requirements | Connection and cache overhead per database; migrations run N times |
| Collection per tenant | Almost never | Each collection and index is a WiredTiger file: thousands degrade checkpoints, startup, and open file handles |

- With the shared layout, `tenant_id` is also the natural shard key prefix (→ `sharding.md`), so the choice compounds well.
- Enforce the filter where it cannot be forgotten: a data-access layer that injects `tenant_id`, or per-tenant users with a role restricted by document filter (→ `security.md`).
- A single very large tenant in a shared collection is the outlier pattern again — give that one tenant its own database rather than reshaping for everyone.

## Field-Level Choices That Age Well

- Store money as integer minor units or `Decimal128`, never a double — `0.1 + 0.2` is as wrong here as anywhere else.
- Dates as BSON `Date`, always UTC. A date stored as a string sorts lexicographically and silently defeats every range query.
- Field names cost bytes in every document AND in the index: `createdAt` over `dateOfCreationOfThisRecord` at a million documents is real storage, and short cryptic keys are a false economy that costs comprehension forever. Keep names short and honest, not compressed.
- One field, one type, forever. Mixed types split range queries (SKILL.md Query Semantics); a `$jsonSchema` validator with `bsonType` is what actually holds the line.
- Prefer a missing field over `null` when "unknown" and "empty" mean the same thing — but pick one and encode it in the validator, because queries distinguish them.
- Arrays of subdocuments beat parallel arrays: `[{k, v}]` is queryable with `$elemMatch`; `keys: []` plus `values: []` correlated by position is unqueryable and drifts.

## Schema Validation

```javascript
db.createCollection("orders", {
  validator: {$jsonSchema: {
    bsonType: "object",
    required: ["tenantId", "status", "createdAt", "schemaVersion"],
    properties: {
      tenantId: {bsonType: "objectId"},
      status: {enum: ["pending", "paid", "shipped", "cancelled"]},
      total: {bsonType: "decimal"},
      createdAt: {bsonType: "date"},
      schemaVersion: {bsonType: "int", minimum: 1}
    }
  }},
  validationLevel: "moderate",   // existing invalid documents keep working
  validationAction: "error"
})
```

- `validationLevel: "moderate"` applies the rule to inserts and to updates of already-valid documents only — the setting that lets you add validation to a live collection without breaking the backfill.
- `validationAction: "warn"` logs instead of rejecting: the way to measure how much data violates a rule before you enforce it.
- Add validation with `collMod` on an existing collection; it takes effect immediately with no rebuild.
- Rejections surface as error 121 with the failing path in `errInfo.details` (→ `errors.md`).

## Working With Arrays

Bounded arrays are good design; the operators for them are where correct schemas get corrupted.

- **Update one matching element**: `$` updates the FIRST element matched by the query — `updateOne({_id, "items.sku": s}, {$set: {"items.$.qty": 3}})`. It requires the array field in the query and it stops at one match.
- **Update all elements**: `$[]` — `{$inc: {"items.$[].qty": 1}}`.
- **Update a filtered subset**: `$[<id>]` with `arrayFilters` — `{$set: {"items.$[low].flag": true}}, {arrayFilters: [{"low.qty": {$lt: 5}}]}`. This is the only operator that updates several specific elements in one write, and it is the one most people never learn.
- **Cap on push**: `{$push: {recent: {$each: [r], $slice: -10, $sort: {at: -1}}}}` keeps the newest ten and nothing else — the operator form of "bounded by design" (SKILL.md rule 3).
- **Remove**: `$pull` by predicate, `$pop` by end (`1` last, `-1` first). `$pull` on a large array rewrites the whole document, like every other array write.
- **Projection**: `{items: {$slice: 5}}` returns a window; `{items: {$elemMatch: {sku: s}}}` returns only the first matching element. Both cut network and memory when the array is large and the caller wants one entry.
- Positional operators do not work through two levels of array nesting. If you need `$[a].$[b]`, the schema is one level too deep (see Anti-Patterns).

## Files and Blobs

- Documents cap at 16MB (SKILL.md rule 2), so anything larger needs GridFS or object storage.
- GridFS splits a file across a `fs.chunks` collection (default 255KB per chunk) plus an `fs.files` metadata document. It gives you replication, sharding, and range reads of the file for free.
- It has no partial update: changing a byte means rewriting the file. It is a store, not a filesystem.
- Choose GridFS when the files must live in the same backup, replication, and access-control story as the data, or when the deployment has no object storage. Choose object storage with a URL in the document in almost every other case — it is cheaper, it serves over HTTP directly, and it keeps large binary content out of the WiredTiger cache your queries need.

## Write Fan-Out Reality

- A denormalized copy means you own its update path. Enumerate every copy at design time; update via `updateMany` and accept eventual consistency, or use a transaction only when staleness is user-visible (→ `transactions.md`).
- `$inc`, `$push`, `$addToSet`, `$min`/`$max` are atomic per document — design counters and sets so concurrent writers never read-modify-write in application code.
- `findOneAndUpdate` is the atomic claim primitive (job queues, seat holds): filter on current state, set the new state, read the returned document. No transaction needed.
- Change streams turn fan-out into an eventually-consistent pipeline instead of a fragile multi-write in the request path (→ `change-streams.md`).

## Anti-Patterns

- Unbounded arrays — canonical rule and multikey cost in SKILL.md rule 3.
- Nesting beyond 3-4 levels: every query path spells the full chain, and arrays-inside-arrays are barely indexable (`$elemMatch` nesting). Flatten or split.
- Large blobs inline: a 2MB document evicts roughly five hundred 4KB documents from cache on every read. Files go to GridFS or object storage with a URL in the document.
- Collection sprawl (per-tenant, per-day collections): each collection and index is a separate WiredTiger file — checkpoint and open-handle costs grow with the count.
- Exposing `_id` as a public business identifier: enumerable and leaks creation time (SKILL.md, ObjectId and _id).
- Rebuilding relational normalization out of habit: five collections joined by `$lookup` on every page load is a relational schema paying document-store prices with none of the guarantees.
- Storing a computed value with no rebuild job: the computed pattern without the backstop drifts, and nobody discovers it until the number is on an invoice.
