# Distributed Locks — Mutual Exclusion You Can Defend

First question, always: is the lock an **optimization** (avoid duplicate work, save money) or a **correctness requirement** (two writers would corrupt data)? A Redis lock is a good optimization and a weak correctness guarantee. If it is correctness, the resource itself must reject stale writers.

## The Single-Node Lock

```bash
# acquire: unique token, TTL longer than the work
SET app:lock:invoice:88 <uuid> NX PX 30000
```

```lua
-- release.lua: KEYS[1]=lock key, ARGV[1]=my token
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
else
  return 0
end
```

Four properties, all required:

1. **`NX`** — acquire only if free, in one atomic command. `EXISTS` then `SET` is not a lock.
2. **A unique token** — a UUID per acquisition, never a constant, never the hostname. It is what makes release safe.
3. **A TTL** — a lock without expiry deadlocks forever the first time a holder is killed. Rule: TTL ≥ 3× the p99 duration of the critical section.
4. **Compare-and-delete release** — the Lua above. A plain `DEL` releases whoever holds it now, which after your TTL expired is the *next* worker.

## TTL, Renewal, And The Real Race

The lock's TTL is a bet that the work finishes first. When it does not, two holders exist and neither knows.

- Renewal ("watchdog"): the holder extends the TTL at 1/3 of its length (`PEXPIRE` guarded by the same token comparison, or `SET ... XX KEEPTTL` patterns in your client's lock helper). A renewal loop that stops when the process stalls is the point — a GC pause or a stopped-world VM misses the renewal and the lock expires, which is correct behaviour.
- Do not renew forever: cap total holding time, then abandon and let the work be re-driven.
- After finishing, verify before acting on shared state: if `TTL` is already `-2`, someone else may have started. That check is why the critical section must be idempotent regardless.
- Never make the TTL "generous to be safe" — a 10-minute TTL on a 5-second job means a crashed worker blocks the resource for 10 minutes.

## Fencing Tokens: The Correctness Answer

Every lock can be lost without the holder noticing (process pause, network partition, clock skew, expiry). The only defence is a monotonically increasing token that the protected resource checks:

```bash
INCR app:lock:invoice:88:fence     # returns 42
# pass 42 to the resource with every write
# the resource rejects any write whose fence < the highest it has seen
```

- This requires the resource (database row, file store, API) to participate. If it cannot, no lock algorithm — Redis or otherwise — gives you correctness; you get probability.
- Cheap substitute when the resource cannot fence: make the operation idempotent and keyed (write with `SET result:<jobid> ... NX`), so a duplicate execution converges instead of corrupting.

## Redlock

The multi-node algorithm acquires the lock on a majority of N independent masters within a bounded time, subtracting elapsed time from the validity window.

- What it adds: survival of a single Redis node's failure without losing mutual exclusion.
- What it does not add: safety under process pauses or clock jumps — that critique is the reason fencing tokens exist, and it is unresolved between the algorithm's author and its critics (→ `SKILL.md`, Where Experts Disagree).
- Cost: N independent masters (not a cluster, not replicas), N round trips per acquire, and a drift allowance in the validity computation.
- Practical stance: use single-node with a token for optimization locks; use fencing for correctness; reach for Redlock only when losing one node must not lose mutual exclusion *and* you have accepted the pause caveat in writing.

## Why Replicas Do Not Make A Lock Safer

Replication is asynchronous: a lock acquired on the master and not yet replicated disappears in a failover, and a second holder acquires it immediately. `WAIT 1 100` after the acquire narrows the window at the cost of latency, but a partitioned master can still acknowledge. Sentinel and Cluster do not change this: they provide availability, not consensus over lock state.

## Eviction Eats Locks

Under an `allkeys-*` policy, a lock key is an eviction candidate like any other — mutual exclusion can end because the instance got busy. A `volatile-*` policy is strictly worse, not a mitigation: those policies evict *only* keys carrying a TTL, and Rule 3 above makes a TTL mandatory on every lock, so the lock is in the candidate pool while the TTL-less keys around it are untouchable. Locks belong on a `noeviction` instance, full stop; if the workload also needs a cache, the cache goes on a different instance.

## Alternatives Worth Preferring

| Situation | Better than a lock |
|---|---|
| Prevent duplicate processing of the same job | Idempotency key: `SET app:done:<jobid> 1 NX EX 86400` — the winner processes, everyone else no-ops |
| Serialize per entity, high throughput | Partition the work so one consumer owns the entity (one stream or one worker per hash bucket) |
| Rate-limit rather than exclude | Counter or token bucket; a lock is a limit of 1 with worse ergonomics |
| Leader election among workers | The same lock plus renewal, but state the failure mode: two leaders during a partition unless the resource fences |
| A single scheduled task across N pods | Idempotency key scoped to the tick (`app:cron:<name>:<minute>` with `NX`) — no renewal, no TTL guessing |

## Checklist Before Shipping A Lock

- Unique token per acquisition, generated by the holder?
- TTL ≥ 3× p99 critical-section duration, and renewed at 1/3?
- Release is a token-comparing Lua script, never a bare `DEL`?
- The critical section is idempotent even if the lock is lost mid-flight?
- The instance policy is `noeviction` — not `allkeys-*`, and not `volatile-*`, which would evict the lock first?
- Acquisition failure has a defined behaviour (retry with backoff, skip, queue) rather than an unbounded spin?
