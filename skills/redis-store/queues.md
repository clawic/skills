# Queues — Streams, Consumer Groups, Retries, Delays

Redis is a competent queue when the broker's job is delivery, not routing topologies. Choose the mechanism from the delivery guarantee you need, not from familiarity.

| Need | Mechanism | Guarantee |
|---|---|---|
| Fire-and-forget, one consumer, order matters | List + `BLMOVE` | At-most-once unless you use the in-flight list |
| Work that must be retried if a worker dies | Stream + consumer group | At-least-once, with a pending list you can inspect |
| Fan-out to independent consumer groups | Stream, one group each | Each group gets every entry |
| Scheduled or delayed jobs | Sorted set scored by due timestamp | At-least-once, claim must be atomic |
| Notifications nobody must persist | Pub/Sub | At-most-once, nothing stored |
| Anything else | Stream + consumer group | The only option here with acks and recovery built in |

Contents: Streams: The Working Shape · Trimming Is Not Optional · Recovering Dead Consumers · Retries And Dead Letters · Delayed And Scheduled Jobs · Lists: When They Are Still Right · Ordering, Throughput, Fairness · Observability

## Streams: The Working Shape

```bash
# producer
XADD app:jobs MAXLEN ~ 100000 '*' type email payload '{"to":"..."}'

# consumer group, created once (MKSTREAM creates the stream if absent)
XGROUP CREATE app:jobs workers '$' MKSTREAM

# consumer loop
XREADGROUP GROUP workers worker-7 COUNT 10 BLOCK 5000 STREAMS app:jobs '>'
# ... process ...
XACK app:jobs workers 1690000000000-0
```

- `'>'` means "entries never delivered to this group". Any other id means "re-read *my* pending entries" — that is the crash-recovery read, and forgetting it leaves your own in-flight work stranded.
- `BLOCK 5000` parks the connection on the server instead of polling; it holds a connection from the pool for the duration.
- `COUNT` batches: 10-100 entries per read amortizes the round trip without holding work hostage in one worker.
- Entry ids are `<ms>-<seq>` and strictly increasing; they double as a cursor for replay and as a time index for `XRANGE`.

## Trimming Is Not Optional

`XACK` removes an entry from the group's pending list. It does **not** remove it from the stream. Without trimming, the stream is an unbounded log in RAM.

- `XADD key MAXLEN ~ 100000 *` — the `~` trims at radix-node boundaries, so it is cheap and approximate; exact `MAXLEN 100000` costs more per write.
- `XTRIM key MINID ~ <ms-timestamp>` when retention is a time window rather than a count.
- Trim on write, not on a cron: a cron that fails leaves you with the bill and no alarm.
- Sizing: retention must exceed your worst realistic consumer outage, or a lagging consumer's pending entries get trimmed out from under it. `XLEN` plus the group's `lag` (in `XINFO GROUPS`) tells you how close you are.

## Recovering Dead Consumers

An entry delivered but never acked stays in the group's Pending Entries List forever, owned by a consumer name that may no longer exist.

```bash
XPENDING app:jobs workers                      # summary: count, min/max id, per-consumer counts
XPENDING app:jobs workers - + 10 worker-7      # detail: idle time and delivery count per entry
XAUTOCLAIM app:jobs workers worker-9 60000 0-0 COUNT 100   # reassign anything idle >60s
```

- `XAUTOCLAIM` (>=7.0) replaces the `XPENDING` + `XCLAIM` loop and returns a cursor to continue with.
- Idle threshold: at least 3× the p99 processing time, or you will steal work from a slow-but-alive worker and process it twice.
- `delivery_count` in `XPENDING` detail is your retry counter — there is no built-in max-retries.
- Clean up consumer names that no longer exist (`XGROUP DELCONSUMER`) or `XINFO CONSUMERS` grows with every pod that ever ran.

## Retries And Dead Letters

Neither exists natively; build both explicitly.

1. On failure, do not `XACK`. The entry stays pending and is reclaimed by the `XAUTOCLAIM` sweeper.
2. On reclaim, read `delivery_count`. Under the limit → process again. Over it → `XADD` the payload to `app:jobs:dead` with the error, then `XACK` the original so it stops circulating.
3. Backoff: an immediately-reclaimed poison message burns a worker in a tight loop. Either raise the idle threshold per attempt, or move it to the delayed set below with a due time of `now + backoff`.
4. The dead-letter stream needs its own retention (`MAXLEN`) and its own alarm — an unwatched DLQ is a silent data-loss channel.

## Delayed And Scheduled Jobs

Sorted set scored by due timestamp in milliseconds:

```lua
-- claim.lua: KEYS[1]=due zset, ARGV[1]=now-ms, ARGV[2]=batch
local due = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', ARGV[1], 'LIMIT', 0, ARGV[2])
if #due > 0 then redis.call('ZREM', KEYS[1], unpack(due)) end
return due
```

- The read and the remove must be one atomic unit, or two pollers claim the same job (Core Rule 3).
- Poll interval sets your worst-case lateness; 1 s polling for jobs scheduled in minutes is fine and costs one `ZRANGEBYSCORE` per second.
- Do not use key-expiry events as the trigger: notifications are at-most-once and fire when the key is actually deleted, not at the TTL instant.
- Hand claimed jobs to the Stream so retries and acks work the same way for immediate and delayed work.

## Lists: When They Are Still Right

- `LPUSH` producer, `BRPOP` consumer is the smallest possible queue and loses the job if the worker dies after popping.
- Reliable variant: `BLMOVE queue processing LEFT RIGHT 5` moves the job to an in-flight list atomically; the worker `LREM`s it from `processing` after success, and a sweeper re-queues entries older than a threshold. This is a hand-built pending list — if you are writing the sweeper, a Stream already has it.
- Capped log: `LPUSH` + `LTRIM 0 999` on every write.
- Lists have no consumer groups, no acks, no replay, and no way to see who holds what.

## Ordering, Throughput, Fairness

- Per-stream ordering is total; per-consumer ordering is not, because entries are distributed round-robin within a group. Work that must stay ordered per entity needs one stream per entity (or a hash-partitioned set of streams, `app:jobs:<n>`).
- Throughput ceiling is the single-threaded server: batched `XADD` via pipeline and `COUNT>1` reads matter more than worker count. Adding workers to a saturated instance adds round trips, not throughput.
- Under Cluster, one stream is one slot and therefore one node. Partition into `app:jobs:{0}` … `app:jobs:{15}` to spread load, and give each worker a subset.
- Eviction is a correctness hazard: on an `allkeys-*` policy the queue itself can be evicted, and a `volatile-*` policy only protects the stream for as long as nobody sets a TTL on it — the moment one exists, the queue joins the candidate pool. Queues belong on a `noeviction` instance, full stop (→ `locks.md`, same rule for locks).

## Observability

```bash
XLEN app:jobs                    # backlog size
XINFO STREAM app:jobs            # first/last id, groups, entries-added
XINFO GROUPS app:jobs            # per group: pending count, lag, last-delivered-id
XINFO CONSUMERS app:jobs workers # per consumer: pending, idle ms
```

Alert on three numbers: `lag` growing (consumers falling behind), pending entries older than the idle threshold (something is dying), and `XLEN` approaching `MAXLEN` (trimming is about to drop unprocessed work).
