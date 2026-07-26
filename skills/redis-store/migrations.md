# Migrations — Upgrades, Sharding, Moving Providers

Four different jobs share one name. Identify which you are doing before picking a tool: version upgrade, topology change, provider move, or key-shape change.

## Version Upgrades

- RDB is forward-compatible, not backward: a newer Redis loads an older file, an older Redis refuses a newer one. That asymmetry means a rollback plan needs a *pre-upgrade* backup, not a fresh one.
- The safe path is replication: attach a new-version replica to the old master, wait for `master_link_status:up` and a matching offset, then promote it and repoint clients.
- Read the release notes for the versions you are skipping, not just the target. The things that historically move: default config values, renamed parameters (`lua-time-limit` → `busy-reply-threshold`, `slave-*` → `replica-*`), encoding names (`ziplist` → `listpack`), and AOF layout (single file → `appendonlydir` in 7.0).
- Rehearse on a restored copy of production data and time it. Load time is your downtime if you do the naive restart.
- After upgrading, re-check the settings you rely on: an upgrade that adopts new defaults changes `save`, `appendfsync` or `maxmemory-policy` behaviour under you.

## Standalone → Cluster

The mechanical part is easy; the code audit is the work. Before anything moves, grep the application for:

| Pattern | Why it breaks | Fix |
|---|---|---|
| `SELECT n` / multiple databases | Cluster has only db 0 | Prefix-namespace the keys first |
| `MGET`/`MSET`/`SUNION`/`RENAME`/`BITOP` over unrelated keys | `CROSSSLOT` | Hash tags for the groups that must stay together, or split into per-key calls |
| Lua scripts with keys built inside the script | Cluster routes by declared KEYS | Declare every key in `numkeys`/KEYS |
| `MULTI` spanning entities | One slot only | Redesign or accept per-entity transactions |
| `KEYS`/`SCAN`/`DBSIZE` used as global | Per node | Iterate nodes (`--cluster call`) |
| Plain Pub/Sub at high volume | Broadcast to every node | Sharded Pub/Sub, Redis >=7.0 |
| A non-cluster-aware client | Fails on the first `MOVED` | Client upgrade is a prerequisite, not a follow-up |

Then: build the cluster, dual-write from the application for the overlap window (or import with the tooling below), verify counts per prefix, cut reads over, and keep the old instance readable until you are sure.

## Moving Data Between Instances

| Tool | Good for | Watch out |
|---|---|---|
| Replication (`REPLICAOF`) | Same major-version family, minimal downtime, exact copy | Target must accept a full sync; it wipes the target |
| RDB file copy | Cross-provider, offline, simplest | Downtime = load time; version-forward only |
| `redis-cli --rdb` | Pulling a snapshot from a server you cannot reach on disk | Point-in-time only |
| `DUMP`/`RESTORE` per key | Selective moves, key-by-key repair | Payload carries a version + CRC and is rejected by older targets; TTL must be passed explicitly, in ms |
| `MIGRATE` | Server-to-server move of specific keys (what resharding uses) | Blocks both ends for the transfer — brutal on a big key |
| `redis-cli --pipe` | Loading generated data or a transformed dump | You generate the RESP; it is a loader, not a copier |
| Provider import/export | Managed-to-managed | Formats and limits are provider-specific |

Whichever you pick, the verification step is the same: compare `DBSIZE` per node, spot-check `MEMORY USAGE` and `TTL` on a sample of keys per prefix, and confirm that keys with TTLs still have them (a naive copy loop that omits TTLs turns your cache into a leak).

## Cutover Patterns

1. **Dual write, read old.** Write to both, read from the old store. Cheapest rollback; needs the write path to tolerate one side failing.
2. **Dual write, shadow read.** Read both, compare, log differences, serve the old. This is where you discover the TTL and encoding bugs.
3. **Flip reads.** Read new, keep writing both, keep the old store warm for the rollback window.
4. **Stop dual write** only after the rollback window has passed with no differences logged.

For a pure cache, skip all of it: point at the new instance, accept a cold-cache period, and confirm the source of truth survives the miss storm (stampede).

## Key-Shape Migrations

Changing the structure under a key is a new key, not an operation (`WRONGTYPE` is the symptom of pretending otherwise).

1. New prefix carrying the version: `app:v2:user:1042`.
2. Dual write both shapes during the overlap.
3. Backfill the old population with a `SCAN` pass in batches, converting as you go — bounded batches, resumable by cursor, with a rate limit so the backfill does not become the incident.
4. Flip reads to v2.
5. Sweep the old prefix with `--scan` + `UNLINK`.

Lazy alternative when the data is regenerable: write v2 only, let v1 expire, and delete the remainder once its TTL horizon has passed. Zero backfill, at the cost of a temporary double memory footprint.

## Redis → Valkey (Or Back)

Protocol- and RDB-compatible for the classic command set, so the paths above apply unchanged: replicate or import an RDB. What needs checking is the periphery — modules in use, client-library defaults, and any feature gated on a version number whose meaning differs across the fork.

## Migration Checklist

- Is there a pre-migration backup that the *old* version can load?
- Has the target been sized with the same headroom rule (Core Rule 2), including reserved memory on managed platforms?
- Does the client library support the target topology (cluster-aware, Sentinel-aware, TLS)?
- Have TTLs been verified on a sample after the copy, not just key counts?
- Is the rollback a decision someone can make in one step, or does it require another migration?
- Is there a written cutover window with the measured load time from a rehearsal, not an estimate?
