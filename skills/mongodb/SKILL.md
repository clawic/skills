---
name: mongodb
slug: mongodb
version: 1.0.3
description: Designs MongoDB schemas, indexes, and aggregation pipelines; diagnoses slow queries and tunes production replica sets. Use for document modeling, embed vs reference decisions, explain plans, write concerns, sharding, or replica set operations.
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
---

## When To Use

- Designing or reviewing document schemas: embed vs reference, growing arrays, multi-tenant layout
- A query is slow: reading explain plans, designing compound indexes
- Building or debugging aggregation pipelines, including memory-limit errors
- Operating production: replica sets, sharding, write/read concerns, backups, connection tuning
- Not for SQL databases — normalization instincts from relational design actively mislead here

## Quick Reference

| Situation | Where |
|---|---|
| Modeling data; embed vs reference; array growth; pattern catalog | `schema.md` |
| COLLSCAN in explain; compound index design; ESR; TTL/partial/text/wildcard | `indexes.md` |
| Pipeline stages; $group/$lookup/$unwind; 100MB memory errors; materialized views | `aggregation.md` |
| Replica sets; write/read concern; sharding; cache; backups; triage | `production.md` |
| Anything else — query semantics, consistency, ObjectId | sections below |

## Core Rules

1. Read `explain("executionStats")`, never guess. Healthy ratio: totalDocsExamined / nReturned ≈ 1. Worked example: 1,240,000 examined for 25 returned = 49,600:1 — missing or wrong index (Atlas's Query Targeting alert fires at 1000:1 by default).
2. 16MB is a ceiling, not a budget. WiredTiger rewrites the whole document on every update: a 5MB document taking a 20-byte `$inc` still costs a 5MB rewrite in cache. Keep working documents in the KB range.
3. Unbounded arrays are the #1 schema failure. Anything that grows per event goes to its own collection or a time-series collection (5.0+). A multikey index adds one entry per element: a 10,000-element array = 10,000 index entries for one document.
4. Set write and read concern explicitly in the connection string. Implicit defaults changed across versions (→ `production.md`); code relying on them silently changes durability semantics on upgrade.
5. One compound index per query shape. Prefix rule: `{a: 1, b: 1, c: 1}` also serves queries on `{a}` and `{a, b}` — delete those redundant single-field indexes. Index intersection exists but the planner rarely picks it; never design for it.
6. Single-document atomicity is the concurrency primitive. Model invariants inside one document before reaching for transactions; when you do use them, stay under 1,000 modified documents and well inside the 60s default transaction lifetime.
7. Read-your-own-writes requires primary reads or a causal-consistency session. Secondary lag is usually sub-second but unbounded under load — never assume freshness on a secondary.

## Query Semantics That Bite

- `{tags: "red"}` matches docs where tags IS "red" or an array CONTAINING "red" — implicit array traversal, both directions.
- `{qty: {$gte: 5, $lte: 10}}` on an array field matches `[3, 12]`: different elements satisfy each bound. One element must satisfy all conditions → `$elemMatch`.
- `{field: null}` matches explicit null AND missing field. Distinguish with `{field: {$type: "null"}}` vs `{field: {$exists: false}}`.
- `$ne`, `$nin`, `$not` technically use the index but scan most of it — near-collection-scan cost; restructure with a positive predicate or a status field.
- `skip(100000).limit(20)` walks 100,020 index entries. Paginate by range on the sort key instead: `{_id: {$gt: lastSeenId}}` — constant cost per page.
- Anchored case-sensitive regex `/^abc/` uses the index; `/^abc/i` cannot. Case-insensitive lookups need a collation index (→ `indexes.md`).

## Consistency Model

- Acknowledged ≠ durable: a `w: 1` write vanishes into a rollback file if the primary fails before replication — someone must reconcile those by hand.
- Failovers happen; retryable writes (default on in modern drivers) hide most of them — but `updateMany` and `deleteMany` are NOT retryable writes. Wrap multi-doc mutations in idempotent logic or a transaction.
- Causal sessions give read-your-own-writes even across secondaries; use them instead of forcing `primary` everywhere.
- Arbiters are a durability trap, not a cheap third node: the PSA topology stalls majority writes when one data node is down (→ `production.md`).

## ObjectId

- 12 bytes: 4-byte unix timestamp + 5-byte random + 3-byte counter. `ObjectId.getTimestamp()` recovers creation time; sorting by `_id` approximates insertion order (per-process counter, not a global clock).
- Predictable enough to enumerate — never use as a security token or unguessable URL.
- Monotonic growth makes `_id` a hotspotting shard key (→ `production.md`).

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Treating "schemaless" as "no schema" | Schema moved into app code, unenforced; shapes drift per deploy | `$jsonSchema` validator + `schema_version` field |
| `$push` growing an array forever | Full-document rewrite per push, then the 16MB wall | `$slice` cap or a child collection |
| `countDocuments({})` for dashboard totals | Runs a scan-backed count on every load | `estimatedDocumentCount()` — metadata, O(1) |
| One collection per tenant/day | Each collection+index is a WiredTiger file; thousands degrade checkpoints and startup | `tenant_id` field + compound indexes |
| `$where` / `$function` in hot paths | JavaScript per document, no index use | Rewrite with native operators |
| Ignoring write concern on "unimportant" writes | Data appears written, lost on failover | Pick a concern per data class (→ `production.md`) |

## Where Experts Disagree

- Embed-first vs reference-first: MongoDB's own guidance is embed-first; teams from relational or microservice backgrounds reference-first for independent lifecycles. Boundary: embed when the child is always read with the parent AND bounded; reference otherwise.
- Transactions: one school treats a multi-document transaction as a schema-design smell (redesign so the invariant fits one document); the other uses them freely since 4.0. Boundary: cross-entity invariants you cannot co-locate (ledger + balance) justify them; convenience joins do not.
- Secondary reads for scale: often called a myth — every secondary applies every write anyway, so they add read capacity only. Still legitimate for analytics isolation and geo-local latency with a bounded `maxStalenessSeconds`.

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/mongodb.
