# Incidents — Symptom To Cause

Each chain is ordered by probability, and every step is a check, not a guess. Start with the three commands that cost nothing: `INFO`, `SLOWLOG GET 10`, `CLIENT LIST`.

## Writes Rejected With OOM

1. `INFO memory` → `used_memory` vs `maxmemory`, and `maxmemory_policy`.
2. `INFO stats` → `evicted_keys`. Flat while writes fail = nothing is evictable: either `noeviction` (working as configured) or a `volatile-*` policy with no TTL-carrying keys left.
3. `INFO keyspace` → `keys=` minus `expires=` is the TTL-less population that will never leave on its own.
4. Buy time: raise `maxmemory` if the host has RAM (`CONFIG SET` **plus** `CONFIG REWRITE`), or switch to `allkeys-lru` *if and only if* every key on the instance is safely evictable — never on an instance holding locks or queues.
5. Find the growth: `--memkeys`, then delete by prefix with `--scan` + `UNLINK`. Never `FLUSHALL`.

## Server Unresponsive Or Latency Spiked

1. Is it Redis or the host? `redis-cli --intrinsic-latency 100` on the box vs `--latency` from a client.
2. `SLOWLOG GET 10` — one O(N) command against a big key explains most seconds-long stalls.
3. `INFO stats` → `latest_fork_usec`; `LATENCY LATEST` for `fork`, `aof-fsync-always`, `expire-cycle` events.
4. `INFO memory` → `mem_fragmentation_ratio` below 1.0 means the process is swapping; check the host.
5. `CLIENT LIST` for a `MONITOR` connection someone left open, and for clients with a large `omem`.
6. If a Lua script is running: `SCRIPT KILL` works only while it has not written; after that only `SHUTDOWN NOSAVE`.

## MISCONF: Writes Refused, Reads Fine

The last background save failed and `stop-writes-on-bgsave-error yes` is protecting you. `INFO persistence` → `rdb_last_bgsave_status`, `rdb_last_error`. Causes in order: disk full, `dir` not writable, fork refused for lack of overcommit, read-only filesystem. Fix the cause, then `BGSAVE` and confirm `rdb_last_bgsave_status:ok`; turning off `stop-writes-on-bgsave-error` restores writes and hides the problem.

## Disk Full

1. What is consuming it: the RDB, `appendonlydir/`, or the log?
2. An AOF that has outgrown the dataset needs a rewrite (`BGREWRITEAOF`) — which needs disk headroom to write the new file before deleting the old, so free space first.
3. Never delete `appendonlydir` files individually while Redis is running; the manifest and the base/incr set must stay consistent.
4. After recovery, set the alarm at the level where a rewrite still fits: roughly dataset size in free space, not a fixed percentage.

## Keys Vanished

- `evicted_keys` rising → eviction under an `allkeys-*` policy. The design question is why non-evictable data lives on an evictable instance.
- `expired_keys` rising → they had TTLs and reached them. Look for a code path that sets a short TTL, or `SET` overwriting a long TTL with a default.
- Neither rising and the keyspace is *empty* → `FLUSHALL` (check `CLIENT LIST`, the log and, if reachable from outside, treat as a compromise), or the instance restarted with no persistence, or it resynced from an empty master.
- `DBSIZE` differs between master and replica → normal: replicas keep expired keys until the master's `DEL` arrives.

## Connections Refused Or Reset

1. `INFO clients` → `connected_clients` against `maxclients`; `INFO stats` → `rejected_connections`.
2. `CLIENT LIST` grouped by address: one host holding hundreds of connections is a leaking pool.
3. Resets under load with no ceiling reached → a client output buffer limit disconnecting slow subscribers or replicas.
4. Nothing connects at all → bind address, firewall, TLS mismatch or auth. Test from the server host with `redis-cli` before touching the application.

## Replication Broken Or Lagging

1. On the replica: `INFO replication` → `master_link_status`, `master_last_io_seconds_ago`.
2. `INFO stats` → `sync_full` rising means partial resync keeps failing: `repl-backlog-size` too small for the disconnect window.
3. Repeated full-sync loops also come from the replica output buffer limit (`256mb 64mb 60`) killing the transfer each time — raise it *and* fix the reason the replica cannot keep up.
4. Lag with a healthy link → the replica is CPU- or disk-bound, often because it is the one doing `BGSAVE`.
5. `MASTER <-> REPLICA sync: Loading DB in memory` is progress, not a fault; `INFO persistence` `loading_*` estimates the remainder.

## After A Failover, Writes Are Missing

Expected: replication is asynchronous, so writes acknowledged by the old master and not yet replicated are gone. Establish the loss window (`master_repl_offset` delta at the time, if you captured it), then decide the durability posture deliberately — `WAIT`, `min-replicas-to-write`, or application-level idempotency. Do not "fix" it by pointing clients back at the old master: it has divergent history and will discard it when it rejoins.

## Two Masters / Split Brain

`INFO replication` on both shows `role:master`. Stop writing to both immediately, pick the one with the higher offset *and* the traffic, and demote the other with `REPLICAOF`. Then fix the cause: sentinel quorum on too few hosts, configuration management restoring an old `replicaof` line, or a detection timeout shorter than a normal network hiccup.

## CROSSSLOT / MOVED Errors Appear After A Deploy

- `CROSSSLOT` → new code issued a multi-key command over keys without a shared hash tag.
- `MOVED` reaching the application → the client is not cluster-aware or is not refreshing its slot map; also happens right after a reshard.
- Both are code-level, not cluster-level: rolling back the deploy is the fast mitigation.

## Instance Was Publicly Reachable

Treat as compromised, not as a warning: rotate credentials, inspect the host for added SSH keys and cron entries, check `MODULE LIST` and `CONFIG GET dir`, and rebuild rather than clean.

## After Any Incident

Write down three numbers before the context evaporates: what the limit was, what the actual value reached, and how long detection took. That triple is what turns the next occurrence into an alert instead of an outage (→ `observability` skill for the alerting side).
