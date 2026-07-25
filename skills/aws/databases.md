# Databases — RDS, Aurora, DynamoDB, ElastiCache

Managed does not mean automatic. The failures below are all configuration decisions made at creation time or never.

## Choosing: The Real Test

Ask "can I write down every query pattern today?"

- Yes, and they are all key lookups → **DynamoDB**. Single-digit millisecond latency at any scale, zero idle cost on on-demand, no connection limit.
- No, or the product is still discovering its queries → **RDS PostgreSQL**. Ad-hoc queries, joins, transactions, and a query planner that forgives a bad guess.
- Disqualifiers for DynamoDB: items over 400 KB, ad-hoc analytics, anything needing a join. Modeling DynamoDB like a relational table is the most expensive way to discover you needed Postgres.
- Aurora over RDS when you need read scaling (up to 15 low-lag replicas), sub-30-second failover, or storage that grows without a maintenance window. Key modeling depth for DynamoDB tables: the `dynamodb` skill.

## Connection Limits (the formula, not a lookup table)

RDS derives `max_connections` from instance memory via the default parameter group:

- **PostgreSQL**: `LEAST(DBInstanceClassMemory / 9531392, 5000)`
- **MySQL**: `DBInstanceClassMemory / 12582880`

Worked, db.t3.micro at 1 GiB (1,073,741,824 bytes):
- Postgres: 1073741824 ÷ 9531392 = **112 connections**
- MySQL: 1073741824 ÷ 12582880 = **85 connections**

Double the memory, double the ceiling. Now compare that to your application: 300 concurrent Lambda executions or a container fleet with a pool of 20 per task and 30 tasks both exceed 112 without any traffic spike.

Answers, in order of preference:

1. **RDS Proxy** in front of RDS/Aurora — pools and multiplexes connections, and holds them through a failover instead of surfacing errors to the app. This is the default fix for anything serverless.
2. A correctly sized application-side pool. `pool_size × instance_count ≤ max_connections × 0.8` leaves headroom for migrations, admin sessions, and monitoring.
3. Raising `max_connections` in a custom parameter group. This works and then does not: each connection costs memory, and Postgres at 500 idle connections on a small instance is slower than the same instance at 100.

Symptom mapping: `remaining connection slots are reserved` (Postgres) or `Too many connections` (MySQL) is the ceiling; a *timeout* connecting is the network, never credentials.

## Multi-AZ Is Not a Backup

Multi-AZ is synchronous replication for availability. It replicates your `DROP TABLE` in under a second. The recovery mechanisms are different features:

| Need | Mechanism | Reality |
|---|---|---|
| Instance or AZ failure | Multi-AZ | Automatic failover via a DNS change, typically 60-120s for RDS, usually under 30s for Aurora |
| Human error, bad migration | Point-in-time restore | Restores to a new instance at any second within the retention window; the old instance is untouched |
| Long-term retention, compliance | Manual snapshots / AWS Backup | Automated backups are deleted with the instance; manual snapshots are not |
| Region loss | Cross-region read replica or snapshot copy | Must be configured in advance; there is no retroactive option |

Backup retention defaults to a short window and caps at 35 days for automated backups. Point-in-time restore granularity is 5 minutes. A retention of 0 disables automated backups entirely and is the default in some IaC modules — check it explicitly.

**Deleting an RDS instance deletes its automated backups.** Always pass `--final-db-snapshot-identifier`, and enable deletion protection at creation.

## Failover Behavior Your Application Must Handle

- Failover swaps the DNS record for the endpoint. Applications that cache DNS forever (JVM's default `networkaddress.cache.ttl` of -1 is the classic) keep talking to the dead instance. Set a DNS TTL respect of ~5 seconds in the client.
- In-flight transactions are lost. The application needs retry logic around the connection, not just around the query.
- Aurora has separate writer and reader endpoints. Sending writes to the reader endpoint fails only after a failover promotes a different node — a bug that hides until the worst possible moment.
- Test it: `aws rds reboot-db-instance --force-failover` in a non-production environment, and time how long the application takes to recover. If nobody has ever run it, the failover plan is a hypothesis.

## Storage and Parameter Changes

- RDS storage grows and **never shrinks**. Shrinking means dump, create smaller, restore. Enable storage autoscaling with a maximum you are willing to pay for, rather than provisioning for the worst case up front.
- One storage modification per volume per 6 hours. A resize during an incident locks you out of a second attempt for the rest of the incident.
- Parameter groups have dynamic and static parameters. Static ones require a reboot to apply, and the console shows "pending-reboot" rather than failing — a setting can be "changed" for weeks without being in effect: `aws rds describe-db-parameters` and check `ApplyType`.
- Free storage space is the alarm nobody sets until the first outage. A full RDS volume puts the instance in `storage-full` state and it stops accepting writes.

## Aurora Specifics

- Storage is shared across the cluster and bills for what you use, growing in 10 GB increments. Replicas do not each carry a copy.
- The **standard configuration bills per I/O request**, and an I/O-heavy workload can cost more than the equivalent RDS despite the serverless branding. Aurora I/O-Optimized trades a higher instance price for zero I/O charges — the crossover is roughly when I/O exceeds a quarter of the total Aurora bill. Check the I/O line in Cost Explorer before and after any migration.
- Serverless v2 scales in 0.5-ACU increments and can be configured with a minimum of 0 ACUs, so a dev cluster costs nothing when idle. Scaling up is fast; the trap is a minimum set high "to be safe", which is just an always-on instance with extra steps.
- Read replicas lag. Read-after-write against the reader endpoint returns stale data. Route reads that must see their own write to the writer.

## DynamoDB Operations

- **Capacity mode break-even**: a provisioned WCU costs $0.00065/hour ≈ $0.47/month and delivers up to 2.63 M writes; the same writes on-demand cost ~$3.29. Provisioned wins above roughly **15% sustained utilization** — but only with autoscaling configured, otherwise you are paying for a peak you rarely reach. On-demand is correct for unknown or spiky traffic and for anything that idles.
- **Throttling with capacity to spare = hot partition.** Per-partition ceilings are 3,000 RCU and 1,000 WCU. A partition key with low cardinality, or a "current day" key, concentrates all traffic on one partition regardless of table capacity.
- GSIs have their own capacity. In provisioned mode, a throttled GSI applies back-pressure to writes on the base table — the table looks fine and writes fail.
- LSIs must be created with the table and can never be added; GSIs can be added later. This is the one modeling decision with no escape hatch.
- TTL deletes are best-effort within 48 hours of expiry and consume no write capacity. Filter expired items in the query too, or users see records that "should" be gone.
- Transactions consume double the capacity of the equivalent non-transactional operations. Use them where correctness requires it, not as a default.
- Point-in-time recovery is a per-table switch, off by default, restoring to any second in the last 35 days. Turn it on; the cost is storage-based and small next to the alternative.

## ElastiCache

- Redis vs Memcached: Redis for anything needing persistence, replication, pub/sub, or data structures; Memcached only for a pure multi-threaded cache you are willing to lose entirely.
- Cluster mode changes the client contract: multi-key operations must hit the same slot. Enabling it later is a client rewrite, so decide at creation.
- Eviction policy defaults to `volatile-lru`, which evicts only keys that have a TTL. Cache entries written without a TTL are never evicted, and the cluster fills until writes fail. Use `allkeys-lru` for a pure cache.
- A cache is not a database. Sizing that assumes 100% hit rate produces an origin that cannot survive a cold cache — test the failure mode by flushing it in staging.

## Migration and Upgrades

- Engine major-version upgrades are one-way. Snapshot first, test the restore against a copy, then upgrade — and read the engine's own breaking-change notes, not just the AWS ones.
- DMS handles the bulk load plus change data capture for a low-downtime cutover; the schema conversion is a separate problem and usually the harder one.
- Blue/green deployments (RDS/Aurora) create a synchronized staging environment and switch over in a controlled window — the safest path for a major version upgrade of a busy database.
- Post-cutover, keep the source readable but not writable until you are certain. Dual-write is a distributed-systems problem you did not sign up for.
