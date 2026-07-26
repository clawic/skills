# Setup — Redis

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

Redis is fast and unforgiving: the failure modes are memory, atomicity and durability, and they surface in production rather than in review. Be concrete, name the command, and say out loud what a recipe does and does not guarantee.

## How To Load Preferences

1. Read `~/Clawic/data/redis-store/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `deployment: self-hosted`, `topology: standalone`, `server_version: 7`, `client: redis-cli`, `maxmemory_policy: noeviction`, `persistence_mode: rdb`, `key_prefix: app`, `default_ttl: 1h`, `destructive_confirm: true`.
3. Read `~/Clawic/data/redis-store/memory.md` for prior context (their workload, incidents already discussed). Absence is fine; proceed without comment.

Work from defaults immediately. Never open with questions about their setup, priorities, or how proactive to be.

## Cheap Facts Worth Reading Instead Of Asking

When a live connection is available and the work depends on it, one round of reads beats a questionnaire:

- `INFO server` → version and OS, which settles every `feature >=X` gate
- `INFO replication` → `role`, connected replicas, whether this is a replica at all
- `INFO memory` → `maxmemory`, `maxmemory_policy`, `used_memory`, `mem_fragmentation_ratio`
- `INFO persistence` → `aof_enabled`, `rdb_last_bgsave_status`, `loading`
- `CLUSTER INFO` → `cluster_enabled:1` settles `topology` without asking

Record what these reveal in `~/Clawic/data/redis-store/memory.md`, not in `config.yaml` — observed state is not a declared preference.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names a provider, topology, version, client library, eviction policy or persistence mode → update the matching key in `~/Clawic/data/redis-store/config.yaml`.
- User expresses a habit or stance (key naming, whether Lua is welcome, how aggressively to trim streams, whether you may run admin commands) → record it under the relevant preference area (tooling, thresholds, conventions, platform, risk posture, output format, work order, integrations, restrictions, cadence) in `~/Clawic/data/redis-store/memory.md`.
- User corrects earlier guidance → update the stored value so you do not repeat it.

If the user has said nothing, store nothing.

## What Memory Holds

See `memory-template.md` for the file format. Track their workload shape (cache, queue, primary store), the incidents they have already hit, the commands their platform forbids, and how much explanation they want — but only from what they actually reveal.
