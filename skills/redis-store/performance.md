# Performance — Latency, Round Trips, Command Cost

Redis executes commands one at a time on one thread. Every performance question is therefore one of three: how long each command takes, how many round trips you make, and what else is stealing the thread.

## Order Of Investigation

The triage sequence lives in `SKILL.md` (Latency Triage). This file is what each step means and what to do with the answer.

1. **Client-side vs server-side.** `redis-cli --latency` measures round trip; `--intrinsic-latency 100` on the server box measures what the kernel and CPU cost with no Redis work at all. A high intrinsic number (tens of ms) means a noisy VM, CPU steal or power management — nothing you tune in Redis will fix it.
2. **Slow commands.** `SLOWLOG GET 10`, threshold `slowlog-log-slower-than` default 10000 microseconds, retaining `slowlog-max-len` 128 entries. Times exclude network, so anything logged is genuinely slow execution.
3. **Latency events.** `CONFIG SET latency-monitor-threshold 100` (default 0, disabled), then `LATENCY LATEST`, `LATENCY HISTORY <event>`, `LATENCY DOCTOR`. Event names are the diagnosis: `fork`, `expire-cycle`, `aof-fsync-always`, `command`.
4. **Aggregate cost.** `INFO commandstats` gives `calls`, `usec`, `usec_per_call` per command; `INFO latencystats` adds percentiles on modern versions. Rank by `calls × usec_per_call`, not by `usec_per_call` alone — the cheap command called 50k/s usually owns the CPU.
5. **Keyspace attribution.** `redis-cli --hotkeys` (requires an LFU policy) finds the keys taking the traffic. Combined with `--bigkeys`, you get "which key" and "how heavy".

## Round Trips Dominate

Unpipelined cost is `n × RTT`. Concretely: 1000 sequential `GET`s at 0.5 ms RTT is 500 ms of wall time while the server spends under 10 ms of CPU. The client is idle 98% of that.

- **Pipeline.** Send a batch without waiting; 500 commands per flush is a sane default. Bigger batches keep growing the client output buffer and the reply you must hold in memory.
- **Multi-key commands.** `MGET`/`MSET`/`HMGET`/`SMISMEMBER` are one round trip and one command.
- **Lua** collapses a read-decide-write loop into a single round trip *and* makes it atomic.
- Do not confuse pipelining with concurrency: 50 connections each doing one command at a time is 50× the sockets and the same round-trip problem, plus contention.
- Expected throughput reference: a single Redis core serves on the order of 10^5 simple commands per second (`redis-benchmark` on commodity hardware). Seeing a small fraction of that with an idle CPU on the server is a round-trip problem, always.

## Command Cost

| Pattern | Cost | Replace with |
|---|---|---|
| `KEYS pattern` | O(N) over the whole keyspace, blocking | `SCAN` loop with `COUNT 500` |
| `HGETALL` / `SMEMBERS` / `LRANGE 0 -1` on big collections | O(N) plus N elements over the wire | `HMGET`, `SSCAN`, bounded `LRANGE` |
| `DEL` on a huge collection | Synchronous free of every element | `UNLINK` + `lazyfree-lazy-*` |
| `ZRANGEBYSCORE` without `LIMIT` | Returns the whole matching range | Add `LIMIT offset count` |
| `SORT` on a large list | O(N log N) on the main thread | Sort in the application, or keep a sorted set |
| `SINTER`/`SUNION` over large sets | O(N) in the inputs | `SINTERCARD` with `LIMIT`, or precompute |
| `SMEMBERS` for a membership test | O(N) for an O(1) question | `SISMEMBER` / `SMISMEMBER` |
| `FLUSHALL`/`FLUSHDB` sync | Frees everything on the main thread | `ASYNC` variant, and reconsider entirely |
| Anything else | Read the command's documented complexity before shipping it in a hot path | — |

The general rule is Core Rule 4: complexity × N is a *global* stall, not a slow request, because execution is serial.

## Things That Steal The Thread

- **Fork** for `BGSAVE`/`BGREWRITEAOF`: on the order of 10-20 ms per GB of RSS, far worse with transparent huge pages. Visible as `latest_fork_usec` and a `fork` latency event.
- **AOF fsync** with `appendfsync always`, or `everysec` on a saturated disk: `LATENCY LATEST` shows `aof-fsync-always` or write stalls.
- **Swap**: `mem_fragmentation_ratio` below 1.0 means pages are on disk; a microsecond operation becomes a millisecond one.
- **Eviction storms**: a burst of large writes against a full instance evicts in a loop before serving the command.
- **Expiry cycles** when millions of keys expire at once — the `expire-cycle` latency event.
- **`MONITOR`** left connected: every command is copied to that client and throughput can drop by more than half.
- **Big replies**: a 50 MB `LRANGE` result is serialized on the main thread and buffered before it is sent.

## Threading Reality

- Command execution is single-threaded, and that is the design: no locks, atomic commands, predictable ordering.
- I/O threads (`io-threads`, Redis >=6) parallelize reading and writing sockets, not command execution. They help when the bottleneck is syscall overhead from many clients, and do nothing for a slow command. Leave at 1 until `INFO cpu` shows the process CPU-bound with cheap commands.
- Background threads already handle `UNLINK`, lazy free, and `BGSAVE`'s fork child.
- Scaling out means more instances (by workload or by slot), not more threads.

## Benchmarking Honestly

```bash
redis-benchmark -h <host> -p 6379 -t get,set -n 100000 -c 50 -P 16 -d 100
```

- `-P` (pipeline depth) changes the result by an order of magnitude; a benchmark without it measures your network, and one with `-P 100` measures nothing your application does. Report the depth with the number.
- `-d` (payload size) matters as much: 100-byte and 10 KB values are different systems.
- `-t` limits the command set; the default runs a suite that may have nothing to do with your workload. Better: replay your own command mix with `--hotkeys`-informed key distribution.
- Benchmark against a comparable dataset size. An empty instance has no eviction, no fragmentation, and a warm allocator.
- Compare like for like: same client library, same TLS setting (TLS adds handshake and per-message cost), same pipeline depth, before and after the change.
