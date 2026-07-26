# Concurrency — Threads, Locks, Channels, and Atomics

Rust removes data races, not deadlocks, not livelocks, not logic races, and not lock convoys. `Send` and `Sync` are the guarantees you get for free; everything below is still yours.

## `Send` and `Sync` in One Table

| Trait | Means | Missing on |
|---|---|---|
| `Send` | The value can be moved to another thread | `Rc`, `RefCell`'s guards, most raw-pointer wrappers, `MutexGuard` |
| `Sync` | `&T` can be shared with another thread (equivalently `&T: Send`) | `Cell`, `RefCell`, `Rc` |

Both are auto traits: derived structurally, so one `Rc` field silently removes `Send` from a whole struct and the error appears at the distant `spawn`. `unsafe impl Send`/`Sync` is a promise you are making on the compiler's behalf — only with a written argument.

## Choosing a Threading Model

| Shape | Tool | Why |
|---|---|---|
| A few long-lived workers | `std::thread::spawn` + channels | Simple, no dependency |
| Borrowing parent-stack data in workers | `std::thread::scope` (`rust >=1.63`) | Removes the `'static` bound; joins on scope exit |
| Data-parallel map/reduce over a collection | `rayon` `par_iter` | Work-stealing pool, near-zero code change |
| Many short tasks with IO waits | An async runtime | Threads are the wrong unit when the work is waiting |
| Pipeline of stages | Bounded channels between stages | Backpressure comes for free |
| Anything else | Start single-threaded and measure | Most "needs threads" is an algorithm problem |

- `thread::spawn` requires `'static`: closures must own their data or hold an `Arc`. Scoped threads are the escape hatch, and they join at the end of the scope whether you called `join` or not.
- A panicking thread does not stop the process; `join()` returns `Err`. A worker pool that ignores join results loses failures silently.
- Rayon inside an async runtime worker starves that worker. Bridge with `spawn_blocking` or a channel, never by calling `par_iter` inside a task.

## Locks

- Lock ordering is the only reliable deadlock defence: define a global order over your locks and take them in that order everywhere. Document it next to the struct definitions; a comment is cheaper than the incident.
- Never call unknown code (a callback, a trait method, a `Drop`) while holding a lock. That is how re-entrant locking and cross-module deadlocks appear.
- `RwLock` read locks are not re-entrant in a useful sense: taking a second read while a writer waits deadlocks on most implementations, because writers are queued ahead to avoid starvation. Two nested reads of the same `RwLock` on one thread is a bug even though it looks harmless.
- Shrink critical sections by cloning the small thing out inside a block that ends before the work; the cost of a clone is almost always less than the cost of the contention.
- Poisoning: a panic while holding a `std::sync::Mutex` makes every later `lock()` return `Err`. `into_inner()` recovers the data if you accept it may be mid-update. `parking_lot` drops poisoning and is smaller and faster, and it removes a real signal — decide, do not drift.
- One `Mutex` around a big struct serializes unrelated fields. Split into several locks (respecting the order rule) or move to per-shard locking before reaching for lock-free structures.

## Channels

| Channel | Semantics | Use |
|---|---|---|
| `std::sync::mpsc::channel` | Unbounded, multi-producer single-consumer | Simple fan-in with no backpressure need |
| `std::sync::mpsc::sync_channel(n)` | Bounded; `n = 0` is a rendezvous | When the sender must be slowed down |
| `crossbeam_channel` | Bounded or unbounded, multi-consumer, `select!` | Work queues with several workers |
| `tokio::sync::mpsc` | Async, bounded or unbounded | Between async tasks |
| `tokio::sync::watch` | Latest value only, many receivers | Config reload, shutdown signal |

- Unbounded channels replace backpressure with memory growth. If the producer can outrun the consumer even briefly, use a bounded one and let the send block; if blocking is unacceptable, drop explicitly and count the drops.
- Closing is by drop: the receive loop ends when **all** senders are dropped. A forgotten sender clone held in a struct is the classic "the pipeline never finishes".
- A rendezvous channel (`sync_channel(0)`) is a synchronization primitive, not an optimization — every send waits for a receive.

## Atomics

- Reach for atomics only for a single scalar: counter, flag, generation number, index. Anything requiring two values to change together needs a lock.
- Orderings, in the order you should consider them:

| Ordering | Guarantee | When |
|---|---|---|
| `Relaxed` | Atomicity only, no ordering with other operations | Statistics counters where you never read another variable because of this one |
| `Acquire` (load) / `Release` (store) | Everything written before the release is visible after the matching acquire | Publishing data behind a flag — the standard pair |
| `AcqRel` | Both, for read-modify-write | `compare_exchange` that publishes and consumes |
| `SeqCst` | A single total order across all `SeqCst` operations | The default when you cannot prove a weaker one is correct |

- Start with `SeqCst`, weaken only with a written argument and a Miri/loom check. A wrong ordering produces a bug that appears once a month on one CPU architecture.
- `fetch_add` wraps on overflow with no panic in any profile — unlike `+` (SKILL.md rule 3).
- `compare_exchange` can fail spuriously only in its `_weak` form; use `_weak` inside a retry loop, the strong form for a one-shot.
- False sharing: two atomics in the same 64-byte cache line make independent threads fight. Pad hot per-thread counters with `#[repr(align(64))]` when a profile shows the contention.

## Testing Concurrent Code

- Determinism first: `loom` exhaustively explores interleavings for lock-free code, and it is the only practical way to gain confidence in a hand-rolled atomic protocol.
- Race detection: `cargo +nightly miri test` catches data races inside `unsafe`, and ThreadSanitizer (`-Zsanitizer=thread`) catches them across FFI.
- Deadlock hunting: run the test suite with a watchdog thread that dumps all stacks after a timeout; a hung CI job with no output tells you nothing.
- Loop the flaky test: `cargo nextest run --retries 0 --test-threads 16` repeated in a shell loop reproduces most schedule-dependent bugs faster than reasoning does.

## Common Concurrency Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `Arc<Mutex<HashMap<K, V>>>` for a hot shared cache | One lock serializes every reader and writer | Sharded map, or a concurrent map crate, after measuring |
| Spawning one thread per unit of work | Thread creation and stack memory dominate the work | A pool, `rayon`, or an async runtime |
| Holding a lock across an IO call | Latency of the IO becomes the latency of every other thread | Clone the needed data out, release, then do IO |
| Using `Relaxed` because "it's just a counter" and then branching on it | The load is unordered with respect to the data it guards | `Acquire`/`Release` pair, or `SeqCst` |
| `while flag.load(Relaxed) {}` busy-wait | Burns a core and can spin forever without a fence | `Condvar`, a channel, or `std::thread::park` |
| Ignoring `JoinHandle` results | Worker panics vanish | Collect and check every join result |
