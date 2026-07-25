# Time-Series Collections — Measurements Over Time

Use a time-series collection whenever documents are measurements with a timestamp and a source: metrics, sensor readings, prices, events per device. A normal collection with a date index works and is the wrong shape — it stores every field name in every document and indexes every row individually.

## Creating One Correctly

```javascript
db.createCollection("readings", {
  timeseries: {
    timeField: "ts",              // required, must be a BSON Date
    metaField: "meta",            // the source identity: {sensorId, site} — everything you group and filter by
    granularity: "minutes"        // seconds | minutes | hours
  },
  expireAfterSeconds: 7776000     // 90 days; retention is native, no TTL index needed
})
```

- `timeField` is immutable after creation, and so is `metaField`. Choosing them wrong means creating a new collection and copying — decide before the first write.
- **`metaField` is the whole design.** It holds the identity that rarely changes (device id, symbol, tenant); the measurement fields go at the top level. Putting a high-cardinality-per-write value inside `meta` (a request id, a random uuid) defeats bucketing entirely: each write becomes its own bucket and you get the storage of a normal collection plus the constraints of this one.
- `granularity` sets how much time one bucket spans. Match it to your write interval: readings every few seconds → `seconds`; a reading per minute per device → `minutes`. Too coarse means oversized buckets, too fine means bucket sprawl.
- MongoDB >=6.3 replaces `granularity` with explicit `bucketMaxSpanSeconds` and `bucketRoundingSeconds` when you need a span the three presets do not offer.

## What You Gain

- Storage: documents in a bucket share field names and are column-compressed. An order-of-magnitude reduction against the naive one-document-per-reading layout is typical, and it is storage AND cache — the working set shrinks by the same factor.
- Queries filtered by `metaField` plus a time range hit bucket-level metadata and skip whole buckets without unpacking them.
- Retention by `expireAfterSeconds` deletes whole buckets, not individual documents: far cheaper than a TTL index doing row-at-a-time deletes on a normal collection (→ `indexes.md`).

## What You Give Up

- Updates and deletes of individual measurements are restricted, and were unsupported at all in early versions — treat measurements as immutable and correct by writing a compensating reading. If you need to edit history routinely, this is the wrong collection type.
- Secondary indexes are limited to `metaField` subfields, `timeField`, and combinations of them — you cannot index an arbitrary measurement value and expect the same behavior as a normal collection.
- Sharding a time-series collection is supported from MongoDB >=5.2 only, and the shard key must be built from `metaField` and/or `timeField`.
- Change streams on time-series collections behave differently from normal collections: events arrive at bucket granularity, not per measurement (→ `change-streams.md`).

## Querying Them

```javascript
db.readings.aggregate([
  {$match: {"meta.sensorId": "s-42", ts: {$gte: from, $lt: to}}},
  {$group: {_id: {$dateTrunc: {date: "$ts", unit: "hour"}}, avg: {$avg: "$temp"}, max: {$max: "$temp"}}},
  {$densify: {field: "_id", range: {step: 1, unit: "hour", bounds: [from, to]}}},
  {$fill: {sortBy: {_id: 1}, output: {avg: {method: "locf"}}}},
  {$sort: {_id: 1}}
])
```

- `$dateTrunc` (MongoDB >=5.0) is the bucketing operator; the old `{$subtract: ["$ts", {$mod: ...}]}` arithmetic is obsolete and unreadable.
- `$densify` inserts the empty hours a chart needs; `$fill` decides whether a gap is zero (`{value: 0}`) or "unchanged" (`locf`) — that decision is domain truth, not a display detail (→ `aggregation.md`).
- Always filter by `metaField` AND a time range. A query with only a time range reads every device's buckets for that window.
- `$setWindowFields` gives moving averages and rates over the result without a self-join.

## The Pre-5.0 Fallback (and legacy collections you inherit)

Manual bucketing: one document per source per time window, with an array of measurements and a count.

```javascript
{ sensorId: "s-42", hour: ISODate("2026-07-25T14:00:00Z"), count: 60,
  first: ISODate(...), last: ISODate(...),
  readings: [{t: ISODate(...), temp: 21.4}, ...] }
```

- Roll a new bucket at a fixed count, not "when the hour ends": `updateOne({sensorId, hour, count: {$lt: 200}}, {$push: {readings: r}, $inc: {count: 1}, $min: {first: r.t}, $max: {last: r.t}}, {upsert: true})`. The `count` guard in the filter is what keeps the array bounded (SKILL.md rule 3).
- Keep `first`/`last` at the top level so range queries never need to look inside the array.
- This is strictly worse than a native time-series collection on every axis. When you find it in an existing system, the migration is a copy into a native collection (→ `migrations.md`), not an optimization of the bucket size.

## Sizing and Retention

- Estimate before building: measurements per source per day × sources × days retained. A thousand devices at one reading per minute is 1.44M measurements/day — the storage question is settled by the compression ratio, the CACHE question by how much of that window queries actually touch.
- Retention and query window should match. Keeping 5 years because "storage is cheap" while every dashboard reads 7 days means 99% of the data is cold weight in backups and restores (→ `backups.md`).
- Downsample instead of retaining raw: a scheduled `$merge` into an hourly rollup collection, then a shorter `expireAfterSeconds` on the raw collection. Two collections, each sized for its own readers (→ `aggregation.md`).
