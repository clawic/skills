---
name: mongodb
slug: mongodb
version: 1.0.4
description: >-
  Designs MongoDB schemas, indexes, and aggregation pipelines, and debugs slow queries, connection errors, and replica set failures.
  Use when modeling documents, deciding embed vs reference, reading an explain plan, or fixing a COLLSCAN, and when a query times out,
  a pipeline aborts at the memory limit, a cursor dies mid-loop, the pool exhausts and server selection times out, writes fail with
  duplicate key or "not writable primary", a secondary lags, the oplog window closes, a shard key hotspots, or WiredTiger cache stalls
  the cluster. Covers mongosh and Compass, Atlas, Mongoose and driver connection strings, transactions and retry loops, change streams,
  time-series collections, Atlas Search and vector search, sharding, backups, restores, and upgrades.
  Not for SQL or relational modeling — normalization instincts actively mislead here.
homepage: https://clawic.com/skills/mongodb
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
metadata:
  clawdbot:
    emoji: 🍃
    requires:
      anyBins:
      - mongosh
      - mongo
    os:
    - linux
    - darwin
    - win32
    displayName: MongoDB
    configPaths:
    - ~/Clawic/data/mongodb/
---

User preferences and memory live in `~/Clawic/data/mongodb/` (see `setup.md` on first use, `memory-template.md` for the file format). If you have data at an old location (`~/mongodb/` or `~/clawic/mongodb/`), move it to `~/Clawic/data/mongodb/`.

## When To Use

- Designing or reviewing document schemas: embed vs reference, growing arrays, multi-tenant layout, versioned shapes
- A query, pipeline, or dashboard is slow: reading explain plans, designing compound indexes, killing a scan
- Building or debugging aggregation pipelines, `$lookup` joins, materialized views, window functions
- Connecting an application: connection strings, pooling, timeouts, retries, ODM behavior, serverless clients
- Operating production: replica sets, sharding, write and read concerns, backups, upgrades, security, Atlas
- A live incident: no primary, connection storm, disk full, runaway operation, cache stall, blown oplog window
- Not for SQL or relational databases — normalization instincts from relational design actively mislead here (see Related Skills)

## Quick Reference

| Situation | Play |
|---|---|
| Query is slow, cause unknown | `explain("executionStats")` before touching anything; triage order in `slow-queries.md` |
| explain shows COLLSCAN | No usable index, or the index exists in the wrong field order — check ESR before creating another (→ `indexes.md`) |
| `totalDocsExamined` far above `nReturned` | Index is not selective enough for this shape; the ratio target is Core Rule 1 |
| Embed or reference this relationship? | Three questions in order: read together, bounded, independently updated (→ `schema.md`) |
| An array grows once per user action | Never embed it — child collection or time-series collection (Core Rule 3) |
| Pipeline aborts "exceeded memory limit" | A blocking stage hit its 100MB budget: index the sort or reshape upstream (→ `aggregation.md`) |
| `find().sort()` fails on memory | Different limit: 32MB in-memory sort for find, not the pipeline's 100MB (→ `slow-queries.md`) |
| `$lookup` inside a hot query | A schema signal, not a tuning problem: extended reference or embed (→ `schema.md`) |
| Deep pagination (`skip` in the tens of thousands) | Range on the sort key: `{_id: {$gt: lastSeenId}}` — constant cost per page |
| "not writable primary", server selection timed out | Topology or failover, not your query (→ `errors.md`, then `connections.md`) |
| Latency spikes right after a deploy | Pool math: pods × `maxPoolSize` (Core Rule 8) or Mongoose `autoIndex` rebuilding on boot (→ `connections.md`) |
| Cursor died mid-iteration (code 43) | 10-minute idle cursor timeout — re-query by range per batch (→ `connections.md`) |
| Duplicate key error on an optional field | Plain unique index counts two missing fields as duplicate nulls — partial unique index (→ `indexes.md`) |
| Secondary lagging, oplog window shrinking | → `replication.md`; flow control and the resync threshold live there |
| Majority writes hang after losing one node | PSA topology trap (→ `replication.md`) |
| One shard absorbs all the writes | Monotonic shard key (→ `sharding.md`) |
| Full-text, fuzzy, faceted, or vector search | The built-in text index is the wrong tool past basic keyword match (→ `search.md`) |
| React to writes: CDC, cache invalidation, outbox | Change streams, resume tokens, oplog window (→ `change-streams.md`) |
| Changing a field's shape under live traffic | Expand → dual-write → backfill in batches → contract (→ `migrations.md`) |
| Metrics, events, or sensor readings by timestamp | Time-series collection, not a normal collection with a date index (→ `time-series.md`) |
| Multi-document invariant across collections | Try to co-locate it first; transactions with a retry loop if you cannot (→ `transactions.md`) |
| Database reachable from the internet | Auth, `bindIp`, and TLS audit before anything else (→ `security.md`) |
| An error code or message to decode | Error Codes below; full catalog with fixes in `errors.md` |
| Anything else | Reproduce with the smallest `find()` that still shows it, `explain()` before and after every change; if production is stuck, read `db.currentOp()` before you change a thing |

Depth on demand, by phase:

- **Diagnose** — `slow-queries.md` explain plans and triage · `errors.md` code to cause · `incidents.md` no primary, disk full, connection storm, cache stall · `monitoring.md` what to watch and alert on · `mongosh.md` the shell toolkit
- **Design** — `schema.md` embed vs reference and the pattern catalog · `indexes.md` ESR, index types, lifecycle · `aggregation.md` pipeline craft · `time-series.md` measurements over time · `search.md` text and vector search
- **Change** — `migrations.md` online schema change, backfills, bulk loading · `transactions.md` sessions and retry loops · `change-streams.md` reacting to writes
- **Operate** — `connections.md` drivers, pools, timeouts, ODMs · `replication.md` replica sets, concerns, oplog, upgrades · `sharding.md` shard keys and the balancer · `tuning.md` WiredTiger cache and host settings · `backups.md` dumps, snapshots, PITR, drills · `security.md` auth, roles, TLS, injection, encryption · `atlas.md` the managed platform

## Core Rules

1. **Read `explain("executionStats")`, never guess.** Healthy ratio: `totalDocsExamined / nReturned` ≈ 1. Worked example: 1,240,000 examined for 25 returned = 49,600:1 — missing or wrong index (Atlas's Query Targeting alert fires at 1000:1 by default). A ratio near 1 with a slow query is a different bug: look at `nReturned` itself, then at the sort.
2. **16MB is a ceiling, not a budget.** WiredTiger rewrites the whole document on every update: a 5MB document taking a 20-byte `$inc` still costs a 5MB rewrite in cache. Keep working documents in the KB range; a document you routinely read in full should fit in a single-digit number of 4KB pages.
3. **Unbounded arrays are the #1 schema failure.** Anything that grows per event goes to its own collection or a time-series collection (MongoDB >=5.0). A multikey index adds one entry per element: a 10,000-element array = 10,000 index entries for one document, paid on every write to that document.
4. **Set write and read concern explicitly in the connection string.** Implicit defaults changed across versions (→ `replication.md`); code relying on them silently changes durability semantics on upgrade. Baseline URI: `?retryWrites=true&w=majority`.
5. **One compound index per query shape, ordered Equality → Sort → Range.** Prefix rule: `{a: 1, b: 1, c: 1}` also serves queries on `{a}` and `{a, b}` — delete those redundant single-field indexes. Index intersection exists but the planner rarely picks it; never design for it.
6. **Single-document atomicity is the concurrency primitive.** Model invariants inside one document before reaching for transactions; when you do use them, stay under 1,000 modified documents and well inside the 60s default transaction lifetime (→ `transactions.md`).
7. **Read-your-own-writes requires primary reads or a causal-consistency session.** Secondary lag is usually sub-second but unbounded under load — never assume freshness on a secondary; bound it with `maxStalenessSeconds` when you read one deliberately.
8. **One `MongoClient` per process, sized from concurrency, not from the default.** Total server connections = processes × `maxPoolSize` (driver default 100): 40 pods × 100 = 4,000 connections ≈ 4GB of mongod RAM before a single document is served. Creating a client per request or per serverless invocation is the same bug at higher speed (→ `connections.md`).

## Query Semantics That Bite

- `{tags: "red"}` matches docs where `tags` IS "red" or an array CONTAINING "red" — implicit array traversal, both directions, and you cannot turn it off.
- `{qty: {$gte: 5, $lte: 10}}` on an array field matches `[3, 12]`: different elements satisfy each bound. One element must satisfy all conditions → `$elemMatch`.
- `{field: null}` matches explicit null AND missing field. Distinguish with `{field: {$type: "null"}}` vs `{field: {$exists: false}}`.
- `$ne`, `$nin`, `$not` technically use the index but scan most of it — near-collection-scan cost; restructure with a positive predicate or a status field.
- `skip(100000).limit(20)` walks 100,020 index entries. Paginate by range on the sort key instead: `{_id: {$gt: lastSeenId}}` — constant cost per page.
- Anchored case-sensitive regex `/^abc/` uses the index; `/^abc/i` cannot. Case-insensitive lookups need a collation index (→ `indexes.md`).
- Comparisons cross BSON types in a fixed order rather than erroring: the string `"5"` is never `$gt: 4`. Mixed types in one field silently split every range query — enforce the type with a `$jsonSchema` validator.
- `$in` with a large array is a range, not an equality, for index-order purposes; a short `$in` gets exploded into parallel index scans and keeps the sort order (→ `indexes.md`).
- Dotted paths into arrays are positional-ambiguous: `{"items.0.sku": x}` targets a position, `{"items.sku": x}` targets any element. `$` and `$[]` update operators follow the same split.

## Consistency Model

- Acknowledged ≠ durable: a `w: 1` write vanishes into a rollback file if the primary fails before replication — someone must reconcile those by hand, and nothing in the application will ever know.
- Failovers happen; retryable writes (default on in modern drivers) hide most of them — but `updateMany` and `deleteMany` are NOT retryable writes. Wrap multi-doc mutations in idempotent logic or a transaction.
- Causal sessions give read-your-own-writes even across secondaries; use them instead of forcing `primary` everywhere.
- Read concern `local` can return writes that later roll back; `majority` cannot. `available` on a sharded cluster skips orphan filtering and can return documents twice — never use it for correctness-sensitive reads.
- Arbiters are a durability trap, not a cheap third node: the PSA topology stalls majority writes when one data node is down (→ `replication.md`).

## ObjectId and _id

- 12 bytes: 4-byte unix timestamp + 5-byte random + 3-byte counter. `ObjectId.getTimestamp()` recovers creation time; sorting by `_id` approximates insertion order (per-process counter, not a global clock).
- Predictable enough to enumerate — never use as a security token or unguessable URL.
- Monotonic growth makes `_id` a hotspotting shard key (→ `sharding.md`).
- `_id` is immutable: changing it means insert-new + delete-old, which is a migration, not an update (error 66 in `errors.md`).
- Custom `_id` is legitimate and free of a second index when the natural key is stable and short (`{_id: "<tenant>:<sku>"}`). Random UUIDv4 as `_id` costs B-tree locality on inserts; UUIDv7 or a prefixed key keeps writes clustered.

## Error Codes

Codes are stable; message text is not. Match on the code. Full catalog with recovery steps in `errors.md`.

| Code | Name | First move |
|---|---|---|
| 11000 | DuplicateKey | Genuine duplicate, a race, or a unique index counting missing fields as null (→ `indexes.md`) |
| 112 | WriteConflict | Two writers hit the same document; inside a transaction, retry the whole transaction (→ `transactions.md`) |
| 50 | MaxTimeMSExpired | Your `maxTimeMS` fired as designed — the query is slow, not broken (→ `slow-queries.md`) |
| 43 | CursorNotFound | Cursor idled past 10 minutes; batch and re-query by range (→ `connections.md`) |
| 292 | QueryExceededMemoryLimitNoDiskUseAllowed | Blocking stage over 100MB with disk use off (→ `aggregation.md`) |
| 10107 | NotWritablePrimary | You are talking to a secondary or a stepped-down node — connect with the full replica set URI |
| 189 | PrimarySteppedDown | Failover in progress; retryable writes cover single-document writes only |
| 133 | FailedToSatisfyReadPreference | No member matches the read preference or `maxStalenessSeconds` (→ `replication.md`) |
| 251 | NoSuchTransaction | The transaction expired (60s default) or the session was lost (→ `transactions.md`) |
| 121 | DocumentValidationFailure | A `$jsonSchema` validator rejected the write — read `errInfo` for the failing path |
| 18 / 13 | AuthenticationFailed / Unauthorized | Credentials vs privileges: 18 is who you are, 13 is what you may do (→ `security.md`) |
| 66 | ImmutableField | Something tried to modify `_id` or a shard key field the wrong way |

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/mongodb/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| server_version | number (4.4-8.0) | 7.0 | Which version-gated advice applies (`feature >=X` lines) when the live server version is unknown; also gates upgrade recommendations |
| deployment | atlas \| self-hosted \| docker \| documentdb \| cosmos-mongo | atlas | Switches between Atlas console recipes and `mongod.conf`/`setParameter`, and suppresses features the emulation layers lack (→ `atlas.md`) |
| driver | mongosh \| node \| python \| java \| go \| csharp | mongosh | Language of every emitted code example and connection string |
| odm | none \| mongoose \| prisma \| beanie \| spring-data | none | Whether examples are raw driver or ODM-shaped, and which ODM traps get surfaced (→ `connections.md`) |
| id_style | objectid \| uuid \| natural | objectid | `_id` type in generated schemas, migrations, and shard-key advice |
| field_naming | camelCase \| snake_case | camelCase | Field names in generated documents, index specs, and pipelines |
| write_concern_default | majority \| w1 | majority | Write concern written into emitted URIs and examples (Core Rule 4) |
| slow_ms | number (ms, 20-500) | 100 | Profiler threshold in `monitoring.md` recipes and what counts as "slow" when reporting |
| backfill_batch | number (docs, 100-10000) | 1000 | Batch size in every backfill and migration loop (→ `migrations.md`) |
| destructive_confirm | bool | true | `drop`, `dropDatabase`, `dropIndex`, unfiltered `deleteMany`/`updateMany`, and `killOp` are emitted for review instead of run |

Preference areas — customizable dimensions; a stated preference is recorded in `config.yaml` and applied from then on:

- **Tooling** — shell vs Compass vs driver REPL, migration framework (plain scripts, migrate-mongo, Mongock), diagram and schema-analysis tooling
- **Thresholds** — profiler `slowms`, the examined:returned ratio worth reporting, pool size, replication-lag alarm, index-size budget per collection
- **Conventions** — collection naming (plural/singular), timestamp field names, soft-delete policy, `schema_version` usage, index naming, enum-as-string vs code
- **Platform** — server version, deployment target, storage class, instance memory and core count — affects every number in `tuning.md`
- **Risk posture** — whether to run mutations directly or hand back reviewed scripts, whether secondary reads are allowed, how aggressive index changes may be on a live collection
- **Output format** — snippets only vs snippets plus an explain walkthrough, how much plan detail to narrate, whether to include the rollback path by default
- **Integrations** — monitoring stack (Atlas metrics, Prometheus exporter, FTDC), backup tooling, search backend (Atlas Search vs external engine)
- **Restrictions** — compliance regimes that mandate field-level encryption or auditing, collections that must never be touched online, features the platform forbids
- **Cadence** — restore-drill frequency, index-usage review cycle, oplog-window review, upgrade windows

## Output Gates

Before emitting a schema, an index, a pipeline, or a migration:

- Is every array in this schema bounded by something other than optimism?
- Does each new index correspond to a real query shape, in ESR order, with its redundant single-field prefixes removed?
- Was the query run through `explain("executionStats")`, with examined:returned checked, and not just read?
- Does the pipeline's first stage reach an index, and does no blocking stage sit ahead of the filter?
- Is the write concern explicit, and does it match the data class (telemetry vs money)?
- Is the backfill batched, resumable, and safe to run twice?
- Does the destructive step (`drop`, unfiltered `deleteMany`, index drop) ship separately from the code change, after the code that stopped using it?
- For anything user-supplied reaching a query object: is it type-checked so `{$ne: null}` cannot arrive where a string was expected (→ `security.md`)?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Treating "schemaless" as "no schema" | Schema moved into app code, unenforced; shapes drift per deploy | `$jsonSchema` validator + `schema_version` field (→ `schema.md`) |
| `$push` growing an array forever | Full-document rewrite per push, then the 16MB wall | `$slice` cap or a child collection |
| `countDocuments({})` for dashboard totals | Runs a scan-backed count on every load | `estimatedDocumentCount()` — metadata, O(1); accepts orphan drift on sharded clusters |
| One collection per tenant or per day | Each collection and index is a WiredTiger file; thousands degrade checkpoints and startup | `tenant_id` field + compound indexes (→ `schema.md`) |
| `$where` / `$function` in hot paths | JavaScript per document, no index use — and an injection surface | Rewrite with native operators (→ `security.md`) |
| Ignoring write concern on "unimportant" writes | Data appears written, lost on failover | Pick a concern per data class (→ `replication.md`) |
| Building an index on a live collection at peak | Non-blocking since 4.2, but it still burns IO and cache on every member | Schedule off-peak, or roll it member by member (→ `indexes.md`) |
| A new `MongoClient` per request or per Lambda invocation | Every invocation opens a fresh pool; the server hits its connection cap while the app looks idle | One client per process, cached outside the handler (Core Rule 8) |
| Iterating a huge cursor while doing slow work per document | The cursor idles out at 10 minutes mid-loop, code 43 | Page by range key, one query per page (→ `connections.md`) |
| `find()` in a loop instead of one `$in` or `$lookup` | N round trips at network latency each; the database is not the bottleneck, the trips are | Batch the keys, or fix the shape so one read serves the page |
| Trusting `retryWrites` to cover everything | `updateMany`/`deleteMany` and multi-statement work are not retryable | Idempotent logic or a transaction (→ `transactions.md`) |
| Deleting millions of documents with one `deleteMany` | Long-running write, oplog flood, replication lag, no resume point | Batched deletes by range, or a TTL index that does it continuously |

## Where Experts Disagree

- **Embed-first vs reference-first.** MongoDB's own guidance is embed-first; teams from relational or microservice backgrounds reference-first for independent lifecycles. Boundary: embed when the child is always read with the parent AND bounded; reference otherwise.
- **Transactions.** One school treats a multi-document transaction as a schema-design smell (redesign so the invariant fits one document); the other uses them freely since 4.0. Boundary: cross-entity invariants you cannot co-locate (ledger + balance) justify them; convenience joins do not.
- **Secondary reads for scale.** Often called a myth — every secondary applies every write anyway, so they add read capacity only. Still legitimate for analytics isolation and geo-local latency with a bounded `maxStalenessSeconds`.
- **Schema validators.** One camp enforces `$jsonSchema` at the database because application-only validation always drifts; the other keeps validation in code because a validator change is a live DDL-shaped operation. Testable boundary: more than one writer (services, scripts, humans in the shell) means the database has to hold the rule.
- **Aggregation in the database vs the application.** Pushing analytics into pipelines competes with the operational workload for the same WiredTiger cache. Boundary: sub-second, index-backed, user-facing aggregations belong in MongoDB; anything scanning history belongs in a materialized collection, an analytics node, or a warehouse (→ `aggregation.md`).

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/mongodb (install if the user confirms):

- `pg` — PostgreSQL; jump there when the model is genuinely relational or you need cross-row constraints and SQL
- `db` — general database operations, reliability, and scaling patterns
- `database-indexing` — index theory beyond MongoDB specifics: write-cost budgets, structures, composite ordering
- `vector-databases` — comparing Atlas Vector Search against dedicated vector stores by scale and filter needs
- `elasticsearch` — when text search outgrows Atlas Search: analyzers, relevance tuning, aggregations

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/mongodb.
