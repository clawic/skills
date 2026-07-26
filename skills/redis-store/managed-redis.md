# Managed Redis — What Changes Off Your Own Box

Every managed offering removes some commands, renames some settings, and reserves some memory. The skill's advice is unchanged; the way you apply it is not.

## Capability Matrix

| Capability | Self-hosted | ElastiCache / MemoryDB | Redis Cloud | Memorystore / Azure Cache | Upstash and serverless |
|---|---|---|---|---|---|
| `CONFIG SET` | Yes | No — parameter groups | Partial, via console/API | No — provider settings | No |
| `CONFIG REWRITE` | Yes | N/A | N/A | N/A | N/A |
| `DEBUG`, `SHUTDOWN`, `MODULE LOAD` | Yes | No | No | No | No |
| `BGSAVE`/`BGREWRITEAOF` | Yes | Provider-managed snapshots | Provider-managed | Provider-managed | N/A |
| Persistence choice | Full control | RDB snapshots; MemoryDB is durable by design | Configurable per plan | Snapshot/AOF depending on tier | Provider-defined |
| Replication and failover | You operate it | Automatic multi-AZ | Automatic | Automatic | Transparent |
| Cluster mode | Your choice | "Cluster mode enabled/disabled" toggle | Transparent sharding | Provider tiers | Transparent |
| Modules (JSON, Search…) | Install yourself | Limited set | Broad support | Limited | Subset |
| Reserved memory | You size it (Core Rule 2) | `reserved-memory-percent`, default 25 | Plan-defined | Plan-defined | N/A |
| Per-command billing | No | No | No | No | Yes — round trips cost money |

Verify the current per-provider details in their documentation before making a plan depend on a row: these matrices move (row detail verified 2026-07).

## The Consequences That Bite

- **Tuning advice that ends in `CONFIG SET` needs a translation step.** `maxmemory-policy`, `appendfsync`, `timeout`, `client-output-buffer-limit` all exist, but as parameter-group entries applied at maintenance time — some require a reboot, which is a failover.
- **Reserved memory is already deducted.** Do not apply Core Rule 2's 55-60% on top of a provider that already reserves 25%: read the effective `maxmemory` from `INFO memory` and size against that.
- **`DEBUG` being unavailable removes several diagnostic recipes** (`DEBUG SLEEP` for timeout testing, `DEBUG SET-ACTIVE-EXPIRE` for expiry tests) — use a local container for those.
- **Snapshots are the provider's, on their schedule.** Your restore drill is *their* restore procedure, and it must still be rehearsed and timed.
- **Failover is automatic and unannounced.** Clients need reconnection and retry logic regardless of how good the provider is.
- **Serverless pricing makes round trips a line item.** The pipelining rule (Core Rule 5) stops being a latency optimization and becomes a cost optimization; a REST-style API adds HTTP overhead per command on top.
- **Blocking and long-running commands are often capped** on serverless tiers: `BRPOP`, `XREAD BLOCK`, long Lua scripts. Design queues around short blocks or polling when that applies.

## ElastiCache And MemoryDB Specifics

- ElastiCache is a cache with optional snapshots; MemoryDB is positioned as a durable primary store with a multi-AZ transaction log. The choice is the durability question — how many seconds of writes you may lose — expressed as a product.
- "Cluster mode disabled" is one shard with replicas — you still get failover, but no sharding, and multi-key commands keep working.
- "Cluster mode enabled" is Cluster: every Cluster constraint applies, including `CROSSSLOT` and per-node `SCAN`.
- Parameter groups apply per node group; changing one is a maintenance event with a reboot for some parameters.
- Auth is either a password (AUTH token) or IAM-based; ACL users exist on recent engine versions.

## Redis vs Valkey

The 2024 licence change (Redis moved off the BSD licence) produced Valkey, a Linux Foundation fork of the last BSD version, adopted by several cloud providers as the default engine. Practical consequences for this skill:

- The command set, RESP protocol, RDB format and every pattern in these files are shared — nothing you learn here stops applying.
- Divergence is in threading work, module ecosystems, and version numbering. Feature gates written as `feature >=X` refer to Redis version numbers; check the Valkey equivalent before relying on one.
- Migration between them is a replication or RDB-import exercise, not a rewrite.
- Which engine a provider actually runs behind the name "Redis" is worth confirming — several switched. (State of the split verified 2026-07.)

## Choosing

1. Regenerable cache, spiky traffic, small ops budget → serverless or a managed cache, and pipeline hard because round trips are the bill.
2. Steady high throughput, cost-sensitive → provisioned managed nodes; you pay for RAM, not per command.
3. Needs durability guarantees the cache tiers do not give → a durable-by-design product (MemoryDB-style) or self-hosted with AOF plus replication, chosen against the loss window you wrote down.
4. Needs modules, custom configuration, or unusual sizing → self-hosted, and budget the operational work honestly: monitoring, backups, restore drills, upgrades.
5. Anything else → managed. The incident playbooks are still yours, but the fork tuning, the disk alarms and the failover plumbing are not.
