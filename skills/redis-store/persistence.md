# Persistence — RDB, AOF, Backups, Restores

Persistence answers one question: how many seconds of writes are you willing to lose, and how long may recovery take? Pick the numbers first, then the mechanism.

## The Two Mechanisms

| | RDB | AOF |
|---|---|---|
| What it stores | Point-in-time binary snapshot of the whole dataset | Every write command, appended |
| Loss window | Everything since the last snapshot | `appendfsync everysec` → ~1s; `always` → ~0; `no` → OS buffer (~30s) |
| Restart time | Fast — a compact file loaded directly | Slower — commands are replayed |
| Ongoing cost | A fork per save | An fsync per second (or per write with `always`) plus periodic rewrite (which also forks) |
| File | `dump.rdb` in `dir` | `appendonlydir/` with a manifest, base and incr files (Redis >=7.0) |

Defaults worth knowing: `save 3600 1 300 100 60 10000` (a save after 1 change in an hour, 100 in 5 minutes, or 10000 in a minute), `appendonly no`, `appendfsync everysec`, `stop-writes-on-bgsave-error yes`.

**Both enabled** is the common production choice: AOF bounds the loss window, RDB gives a fast-loading artifact for backups and clones. On startup Redis loads the AOF when `appendonly yes`, so the RDB is a backup, not the recovery path.

## Choosing

- Regenerable cache → `persistence_mode: none`. Persistence buys nothing and the fork costs latency. Say so explicitly rather than leaving defaults on by accident.
- Sessions, rate limits, anything users would notice losing for an hour → AOF `everysec`.
- Queue, ledger, anything where a lost write is a lost job → AOF `everysec` **plus** replication with `min-replicas-to-write`; `appendfsync always` only after measuring the throughput cost on your disk.
- Large dataset where the fork pause is the problem → keep RDB saves off on the master and let a replica do the saving.

## The Fork

Both `BGSAVE` and `BGREWRITEAOF` fork. The child gets a copy-on-write view; every page the parent then writes is duplicated.

- Pause cost tracks page-table size: on the order of 10-20 ms per GB of RSS on ordinary Linux VMs. Measure yours in `INFO stats` → `latest_fork_usec`.
- Transparent huge pages make it dramatically worse — a 2 MB page is copied when a single byte changes. Disable them: `echo never > /sys/kernel/mm/transparent_hugepage/enabled` (Redis logs a warning at startup when they are on).
- Set `vm.overcommit_memory = 1`. Without it the kernel can refuse the fork on a large instance and `BGSAVE` fails outright, which then triggers MISCONF.
- Extra memory during a save scales with the write rate while the child runs: that is the headroom in Core Rule 2 (`maxmemory` ≤ 55-60% of host RAM with persistence on).

## MISCONF: Writes Refused

`MISCONF Redis is configured to save RDB snapshots, but it's currently unable to persist to disk` means the last background save failed and `stop-writes-on-bgsave-error yes` is protecting you from writing data that is not being saved.

1. `INFO persistence` → `rdb_last_bgsave_status:err`, `aof_last_write_status`, `rdb_last_error`.
2. Usual causes, in order: disk full, `dir` not writable by the redis user, fork refused (overcommit), or a read-only filesystem after a host event.
3. Fix the cause, then `BGSAVE` and confirm `rdb_last_bgsave_status:ok`.
4. Turning off `stop-writes-on-bgsave-error` restores writes and removes the alarm. Do it only as a deliberate, time-boxed decision on a cache — on durable data it converts a visible failure into silent data loss.

## Backups That Are Real Backups

- The artifact is `dump.rdb`. Take it with `BGSAVE` (never `SAVE`, which blocks the whole server) and copy the file after `rdb_last_save_time` advances; the file is replaced atomically by rename, so a copy is never half-written.
- `redis-cli --rdb /path/backup.rdb` pulls a fresh snapshot over the wire from a live server — the portable way to back up a node you do not have filesystem access to.
- AOF is not a convenient backup format: it grows, and a rewrite can happen mid-copy. Back up RDB, keep AOF for the tail.
- Store backups off the instance and off the same disk. A backup on the volume that filled up is not a backup.
- Retention arithmetic: an hourly RDB kept for 48 hours costs 48 × dataset size compressed. State it before promising a retention window.

## Restore Drill

A restore you have never performed is a hypothesis.

1. Stop the target instance (a running Redis overwrites your file at the next save).
2. Place the file at `dir`/`dbfilename` with the redis user as owner.
3. If `appendonly yes`, Redis will load the AOF and ignore the RDB. Either start with `appendonly no`, then `CONFIG SET appendonly yes` (which rewrites the AOF from the loaded dataset and re-enables it), or clear `appendonlydir` deliberately.
4. Start, watch the log for `DB loaded from disk`, and check `DBSIZE` against the expected count.
5. Time it and write the number down: load time scales with dataset size, and that number is your RTO.

`redis-check-rdb` and `redis-check-aof --fix` validate and truncate a damaged tail — `--fix` discards the incomplete final commands, which is exactly what you want after a hard power loss and exactly what you must not do without a copy of the original file.

## Replica-Side Persistence

Running saves only on a replica removes the fork pause from the master's latency profile. Two conditions make it safe:

- The replica must have `appendonly`/`save` configured, and the master must not (otherwise you kept the pause).
- Auto-restart of an *empty* master with a replica attached wipes the replica: the replica syncs the empty dataset. When a master restarts without persistence, either keep it out of the replication topology until it is repopulated, or ensure a failover has already promoted the replica.

## Diskless Options

- `repl-diskless-sync yes` (default on modern versions) streams the RDB to replicas straight from the fork, skipping the disk on the master.
- `repl-diskless-load` on the replica side (`swapdb` or `on-empty-db`) avoids writing the incoming RDB to disk — useful on small-disk containers, at the cost of holding two datasets in memory with `swapdb`.
- Neither changes the loss window; they change I/O, not durability.
