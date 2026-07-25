# Scaling — When One Node Is Not Enough

Order the options by cost, and refuse to skip a step. Most "we need to shard" conversations end at step 2 once someone reads the plans. Sharding is the last option because it removes joins, transactions, and unique constraints across the shard key — capabilities you cannot get back.

Contents: The Ladder · Read The Bottleneck · Vertical First · Read Replicas · Caching · Partitioning · Splitting Workloads · Sharding · Shard Keys · Life Without Cross-Shard Joins · Hot Rows and Contention · Write Amplification · Queues and Backpressure · Migrating to a Warehouse · Signals

## The Ladder

1. Fix the queries and indexes — the cheapest capacity ever bought.
2. Fix the access pattern: N+1s, missing pagination, reports running on the primary.
3. Add hardware. Doubling RAM to fit the working set outperforms every architectural change and costs a restart.
4. Pool connections properly.
5. Cache the reads that dominate.
6. Move reads to replicas.
7. Partition large tables.
8. Split workloads onto separate databases by bounded context.
9. Shard.

## Read The Bottleneck Before Choosing

| Symptom | Bottleneck | Right rung |
|---|---|---|
| CPU high, few slow queries | Query cost or plan quality | 1 |
| CPU high, thousands of tiny queries | Access pattern, N+1 | 2 |
| Cache hit ratio below the 99% OLTP threshold, heavy disk reads | Working set exceeds RAM | 3 |
| Connections at the ceiling, queries fast | Pool sizing | 4 |
| Reads dominate and are repetitive | Read volume | 5, 6 |
| One table is enormous, queries scan it by time | Table size | 7 |
| One noisy feature starves everything else | Workload mixing | 8 |
| Writes saturate a single node's disk or CPU | Write volume | 9 |

Measure before choosing. The most expensive scaling projects are the ones that solved a bottleneck the system did not have.

## Vertical First

- Fitting the working set in RAM is the single largest performance cliff in databases: below it, reads are memory-speed; above it, every miss is a disk I/O.
- Faster storage (local NVMe over network-attached) changes write latency more than any configuration change.
- More cores help concurrency, not single-query latency, unless the engine parallelizes the query.
- Vertical scaling ends at the largest instance the provider offers, and it does not improve availability. It buys the runway to do the rest properly.

## Read Replicas

- Correct fit: read-heavy workloads with tolerance for staleness — dashboards, search pages, exports, analytics.
- Wrong fit: anything that reads its own write. Replication is asynchronous by default, so a redirect after a POST can read the state from before the write.
- Routing patterns, cheapest first: send explicitly-tagged read-only endpoints to replicas; pin a user's session to the primary for a few seconds after they write; or track the commit position and route by it if the driver supports it.
- Replicas do not help write throughput at all, and every replica adds write cost on the primary (WAL shipping) and another thing that can lag or break.
- Long analytics queries on a replica get canceled by replay conflicts unless configured otherwise, which then holds back cleanup on the primary. Dedicate a replica to analytics rather than mixing.

## Caching

- Cache the expensive and repeated, not everything. The candidates are visible in the ranked query list (SKILL.md rule 9): high total time driven by high call count.
- Invalidation strategy, in order of reliability: short TTL (simplest, always correct within the window) → write-through on the code path that changes the data → event-driven invalidation. Manual invalidation scattered through the codebase is the one that always misses a path.
- Cache keys must include everything that varies the result: tenant, user permissions, locale, feature flags. A key missing the tenant is a cross-tenant data leak.
- The database's own buffer cache already serves repeated reads well; an application cache pays off when it avoids the query round trip and the serialization, not merely the disk read.
- Thundering herd: when a hot key expires, every request recomputes it at once. Use a lock or stale-while-revalidate.
- Materialized rollups are a cache with a schema and visible staleness — usually better than an opaque cache for aggregate reads.

## Partitioning

- One table, many physical pieces, still one database. It solves table size, not node capacity.
- The reason that pays for itself is **retention**: dropping a partition is instant and reclaims disk; `DELETE` on the same rows runs for hours and leaves bloat.
- Every query must filter on the partition key or it scans every partition. Verify pruning in the plan, not in the documentation.
- Partition creation must be automated. The failure mode is inserts failing at midnight because next month's partition does not exist.
- Partition count has a cost: planning time grows with the number of partitions, so hundreds is routine and tens of thousands is not.
- Unique constraints must include the partition key; global uniqueness across partitions is not available in PostgreSQL declarative partitioning.

## Splitting Workloads

Before sharding, separate by bounded context: move the analytics tables, the event log, the job queue, or one noisy service's tables to their own database.

- Each split removes a workload's interference with the rest, and the pieces scale independently.
- The price is losing cross-database joins and transactions between them — the same price sharding charges, but paid once at a natural seam rather than across every table.
- Choose seams where the data genuinely does not need to be transactionally consistent with the rest: an audit log, metrics, sessions, search indexes, the job queue.
- Do this before sharding, always. It is often enough on its own.

## Sharding

Horizontal partitioning across independent databases. Everything below is what you give up:

- **No cross-shard joins.** Queries that span shards are fanned out and merged in the application or a proxy.
- **No cross-shard transactions.** Multi-shard writes need sagas, an outbox, or eventual consistency with reconciliation.
- **No global unique constraints** except on the shard key. Global ids must be generated externally (UUIDv7, snowflake ids).
- **No global `ORDER BY ... LIMIT`** without fetching from every shard and merging.
- **Rebalancing is a project.** Moving data between shards while serving traffic is the hardest part, and it is not optional as shards grow unevenly.
- **Operations multiply.** Every migration, backup, restore drill, and upgrade now runs N times and can partially fail.

Implementation choices: application-level routing (most control, most code), a proxy layer (Vitess for MySQL, Citus for PostgreSQL), or a distributed SQL engine (CockroachDB, Yugabyte, Spanner-style) that hides sharding at the cost of higher per-transaction latency and its own operational model.

## Choosing the Shard Key

The single most consequential and least reversible decision in the system.

- It must appear in the overwhelming majority of queries, or every read becomes a scatter-gather.
- It must distribute both data volume and traffic. `tenant_id` is natural for B2B and fails when one customer is 40% of the load; hash-based keys distribute evenly but destroy range queries.
- It must be immutable. Changing a row's shard key means moving the row between databases.
- Time is almost always a bad shard key for OLTP: all writes land on the newest shard while the rest idle.
- Write down the queries that will need scatter-gather **before** committing. If the list includes anything on a hot path, the key is wrong.

## Hot Rows and Contention

Contention on a few rows caps throughput regardless of how many nodes exist.

- A single counter row updated by every request serializes all of them. Shard it into N rows keyed by a random bucket and `SUM` at read time; N of 10-100 is the usual range, and the read cost rises with N.
- Sequences and auto-increment on a single table are a contention point at very high insert rates; client-generated UUIDv7 removes it.
- A job queue polled by many workers contends on the same head rows unless `SKIP LOCKED` is used.
- The last page of an index on a monotonically increasing key is a write hotspot — the reason random UUIDv4 is sometimes proposed, and why UUIDv7 (time-ordered but distributed by node) is the better answer.
- Lock waits reported without deadlocks are the signature; look at what every transaction touches in common.

## Write Amplification

Before scaling writes, reduce them:

- Every index is written on every insert, and updating an indexed column writes to every index on the table. Five indexes on a write-heavy table is a real cost.
- Updating unchanged values still writes a new row version. Guard with `WHERE col IS DISTINCT FROM :new`.
- Wide rows and large JSON documents rewrite entirely on any update.
- Audit triggers double the write volume by construction; batch or partition the audit table.
- Synchronous replicas add a network round trip to every commit; asynchronous ones do not.

## Queues and Backpressure

- Not every write needs to be synchronous. Moving non-critical writes (analytics events, notifications, denormalized updates) to a queue smooths spikes and takes them off the request path.
- A queue table inside the same database is legitimate and simple up to moderate rates; it stops being right when queue churn dominates the database's own write load.
- Without backpressure a queue just moves the failure: bound the queue, shed load, and make consumers idempotent.
- Batch consumers: 1,000 rows in one statement instead of 1,000 statements is the same order-of-magnitude win as any bulk load.

## Moving Analytics to a Warehouse

- The signal is workload shape, not size: full-table scans, wide aggregations, and columnar access patterns competing with OLTP traffic.
- A columnar engine (`duckdb` locally, `clickhouse`, or a cloud warehouse) reads ~25× less data on column pruning alone for a query touching 3 of 80 columns, and more once per-column compression is counted — a storage-format advantage no index gives you on a row store.
- Start with the cheapest form that works: a replica plus rollup tables, then a scheduled extract, then a full pipeline with transformations and tests (`dbt`).
- Keep one definition of each metric across both systems, or the two will disagree and nobody will know which is right.

## Signals That You Actually Need the Next Rung

- Rung 1-2 are exhausted only when the ranked query list has no cheap wins left and the plans are good.
- Vertical is exhausted when you are on the largest practical instance and the working set still does not fit.
- Replicas are exhausted when write throughput, not read throughput, is the ceiling.
- Partitioning is exhausted when a single partition is itself too large for one node.
- Only then is sharding the answer — and by then the shard key is obvious from the access patterns you have been measuring all along.
