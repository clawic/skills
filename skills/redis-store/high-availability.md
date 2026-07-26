# Replication, Sentinel, Failover — And What You Actually Lose

Availability and durability are separate purchases. Replication buys availability; nothing here buys consensus.

## Replication Basics

- `REPLICAOF <host> <port>` (or `replicaof` in the config) makes a node a replica; `REPLICAOF NO ONE` promotes it.
- The master streams writes asynchronously and **does not wait** for replicas. `INFO replication` on the master lists each replica's `offset`; the master's `master_repl_offset` minus a replica's offset is the lag in bytes.
- Replicas are read-only by default (`replica-read-only yes`) and answer `READONLY` to writes.
- Full sync: the master forks, produces an RDB, streams it, then streams the backlog. Partial resync after a brief disconnect uses the replication ID + offset if the write volume during the gap fits `repl-backlog-size` (default 1mb).
- Sizing the backlog: `repl-backlog-size ≥ write_bytes_per_second × expected_disconnect_seconds`. At 5 MB/s of writes and a 60-second network blip, 1 MB guarantees a full resync (a fork, an RDB, and minutes of extra load) where 300 MB would have avoided it.

## What A Failover Loses

The master acknowledges a write before any replica has it. A failover promotes a replica that may be behind. Writes in that gap are gone — and worse, they can *reappear as gone* after the old master rejoins as a replica and discards its own history.

Tools that narrow the window, none of which close it:

| Tool | Effect | Cost |
|---|---|---|
| `WAIT <numreplicas> <timeout-ms>` | Blocks until N replicas acknowledge the offset; returns the count reached | One round trip of added latency per write; a partitioned master can still acknowledge |
| `min-replicas-to-write 1` + `min-replicas-max-lag 10` | Master refuses writes when fewer than N replicas are within the lag window | Writes fail during replica maintenance — that is the point |
| AOF `appendfsync everysec` on both | Bounds loss to ~1s per node on restart | Disk cost |
| Application-level idempotency | A lost write can be replayed safely | Design work, and it is the only one that actually solves it |

State the accepted loss window in writing before choosing a topology. "We lose up to N seconds of writes on failover" is the honest sentence.

## Sentinel

Sentinel is a separate process set that monitors masters, elects a leader among themselves, and reconfigures replicas on failover.

- Run an **odd number, at least 3, on separate hosts/failure domains**. Sentinels on the same two boxes as the Redis nodes fail together with them.
- `quorum` is how many sentinels must agree the master is down to *start* a failover; a majority of the whole sentinel set is still required to *authorize* one. Quorum 2 of 3 is the standard shape.
- `down-after-milliseconds` is the detection delay, `failover-timeout` bounds the whole procedure. Detection shorter than your worst GC pause or network hiccup gives you flapping failovers.
- Clients must be Sentinel-aware: they ask a sentinel for the current master address and re-ask on error. Hardcoding the master's host is the most common reason a technically successful failover still causes an outage.
- Sentinel rewrites the config files of the nodes it manages. Configuration management that restores an old `replicaof` line will fight it and can create two masters.
- Sentinel does not shard and does not proxy: it only moves the "who is master" pointer.

## Split Brain

A partition can leave the old master accepting writes while a new one is promoted. Both are "the master" from someone's point of view; when the partition heals, the old master becomes a replica and **discards its divergent writes**.

- `min-replicas-to-write` is the practical mitigation: the isolated old master loses its replicas, fails its own check, and stops accepting writes.
- Cluster has the same exposure with different mechanics: a majority of masters must agree, within `cluster-node-timeout`.
- Any design where "the lock was held on the old master" matters is a correctness design: it needs a fencing token the protected resource checks, not a better failover.

## Reads From Replicas

- They are asynchronously stale by definition; the staleness is your replication lag, which is small until it is not (a big write burst, a slow network, a busy replica doing `BGSAVE`).
- `replica-serve-stale-data yes` (default) keeps serving during a lost link with the master; setting it to `no` makes the replica return an error instead — pick based on whether stale is worse than unavailable for that read path.
- Replicas do not expire keys themselves, so a replica's `DBSIZE` can exceed the master's; reads still behave correctly.
- Good uses: analytics scans, `BGSAVE` offloading, warm standby. Bad uses: read-your-own-writes flows, anything feeding a decision that a write just changed.

## Runbook: Planned Failover

1. Confirm the target replica is caught up: `INFO replication` on the master, replica `offset` equal to `master_repl_offset`.
2. Confirm it can serve: memory headroom, persistence status, same version or newer.
3. Sentinel: `SENTINEL FAILOVER <master-name>`. Cluster: `CLUSTER FAILOVER` on the replica. Plain replication: `REPLICAOF NO ONE` on the replica, then repoint the others.
4. Verify clients moved: connection counts on the new master, error rate on the old one.
5. Rejoin the old node as a replica **only after** confirming it will not be repopulated from an empty dataset (replica-side persistence).

## Monitoring That Catches This Early

- `master_link_status:down` on any replica, and how long it has been down.
- Replica offset delta in bytes and its trend, not just "connected replicas".
- `rdb_changes_since_last_save`, `latest_fork_usec`, and sync counters (`sync_full` rising means partial resync is failing — usually an undersized backlog).
- Sentinel logs for `+sdown`/`+odown` events that resolve on their own: flapping detection is a failover you narrowly avoided.
