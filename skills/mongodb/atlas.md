# Atlas — The Managed Platform

Atlas runs the same MongoDB with the host layer taken away. That removes most of `tuning.md` and all of the OS settings table, and it adds a set of platform behaviors that produce failures nothing in the database documentation explains.

## Tiers and What Changes With Them

- **Free and shared tiers (M0 and the Flex/shared family)** run on multi-tenant hosts with capped storage, capped connections, and rate limits. The free tier's connection cap is a few hundred; a normal application fleet exhausts it during a deploy (→ `connections.md`).
- Shared tiers omit features: no dedicated CPU guarantees, limited or absent Performance Advisor, no VPC peering, restricted backup options. Prototyping there and planning capacity from what you observe produces numbers that do not transfer.
- **Dedicated tiers (M10 and above)** get real isolation, private networking, backup policies, and every feature. M10 is the smallest tier from which production conclusions are valid.
- Storage, IOPS, and RAM scale with the tier as a bundle. When one dimension is the constraint, you buy the other two as well — worth knowing before assuming the next tier is "a bit more expensive".

## Connection Limits Are the Most Common Surprise

- Every tier has a hard maximum connection count. Exceeding it produces server selection timeouts while the cluster's CPU sits idle, which reads as "the database is down" and is not.
- Fleet arithmetic is the same as anywhere: processes × `maxPoolSize` (SKILL.md rule 8). Atlas simply makes the ceiling explicit and low on small tiers.
- Serverless application fleets multiply this by their concurrency limit. Cache the client outside the handler and set a small `maxPoolSize` (→ `connections.md`).
- The Atlas metrics tab graphs connections against the limit — put that graph on the dashboard before you need it (→ `monitoring.md`).

## Network Access

- The IP access list is checked before authentication: an unlisted address fails as a connection error, never as an auth error, which sends people to debug credentials. `0.0.0.0/0` on the list is the shortcut that makes the database internet-reachable behind a password only (→ `security.md`).
- VPC peering or private endpoints remove the public address entirely. This is the correct production configuration, and it is the one that makes an IP access list irrelevant.
- `mongodb+srv` requires outbound DNS SRV and TXT resolution. Corporate networks and some container DNS configurations block them, producing server selection failures with no useful message.

## Performance Advisor and the Query Profiler

- Performance Advisor watches slow queries and suggests indexes with an estimated impact. Its suggestions are good raw material and bad blind instructions: it optimizes one query at a time and does not know about the write cost or the redundant prefix (→ `indexes.md`).
- Treat each suggestion as a hypothesis: check whether an existing index already covers it in a different order, apply ESR reasoning, then verify with explain.
- The Query Profiler shows slow operations without enabling the database profiler, which makes it the fastest first look during an incident (→ `slow-queries.md`).
- The Query Targeting alert (examined:returned above 1000:1 by default) is the single most useful default alert Atlas ships — it names the index problems before users report them (SKILL.md rule 1).

## Backups and PITR

- Cloud backups are volume snapshots on a policy: frequency, retention, and region are per-policy settings, not defaults you can ignore.
- Continuous cloud backup adds point-in-time restore within the retention window — the recovery mode that matters for "a script deleted the wrong documents" (→ `backups.md`).
- Restore targets a NEW cluster or an existing one. Restoring over a live cluster during an incident is the same mistake it is anywhere: restore to scratch, verify, then move data.
- Snapshots do not include Atlas Search index definitions or Triggers. Keep those in version control, or a restored cluster comes back without its search (→ `search.md`).
- Retention is a policy nobody re-reads after the data grows. Alert on backup age, not on job success.

## Autoscaling

- Compute autoscaling changes tiers, which means a rolling replacement of members — a brief election, not a seamless resize. Applications that survive a stepdown survive it; applications with a single-host URI do not (→ `connections.md`).
- Scaling reacts to sustained load, not to spikes. A traffic spike is absorbed or it is not; autoscaling arrives after.
- Set a maximum tier. Autoscaling without a ceiling turns a runaway query into a bill.
- Storage autoscaling only grows. It never shrinks, and the higher storage tier persists as cost after the incident that caused it.

## Atlas-Specific Features Worth Knowing

- **Atlas Search and Vector Search** — Lucene indexes maintained alongside the collection; the whole of `search.md` applies and is Atlas-only.
- **Triggers** — managed change-stream consumers running your function. Same resume-window semantics and the same at-least-once delivery (→ `change-streams.md`).
- **Online Archive** — moves cold documents to cheaper storage while keeping them queryable through a federated endpoint. Query latency against archived data is object-storage latency, not database latency; design the read path knowing that.
- **Data Federation** — query across clusters and object storage with the aggregation language. Powerful for analytics, wrong for anything in a request path.
- **Database Users vs Atlas Users** — two separate systems. A person with Atlas project access can grant themselves database access; audit both (→ `security.md`).

## What Atlas Does Not Remove

- Schema design, index design, and query shape: the entire cause of most performance problems is untouched by managed hosting (→ `schema.md`, `indexes.md`).
- Working-set-versus-RAM arithmetic. Atlas sizes the cache correctly for the tier; it cannot make your working set smaller (→ `tuning.md`).
- Shard key decisions and their consequences (→ `sharding.md`).
- The obligation to rehearse a restore (→ `backups.md`).
