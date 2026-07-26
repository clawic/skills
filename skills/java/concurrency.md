# Concurrency — Shared State, Locks, and Virtual Threads

Ordering rule for design: **no shared mutable state > immutable shared state > confined mutable state > guarded mutable state**. Every step down costs a class of bug. Composing asynchronous *results* is a different problem — `async.md`.

## The Java Memory Model in Four Facts

1. Without synchronization, one thread's write may never become visible to another — not "eventually", never. The JIT is allowed to hoist a field read out of a loop.
2. `volatile` gives visibility and ordering, not atomicity: `count++` is read-modify-write and still loses updates. `AtomicInteger.incrementAndGet()` is the atomic version.
3. Everything before a `synchronized` release is visible to whoever acquires that same monitor next. Different monitors give you nothing.
4. `final` fields are safely published by the constructor: a fully constructed immutable object can be shared without synchronization. That property is why immutability is the cheapest concurrency strategy.

## Publishing Objects Safely

- Double-checked locking without `volatile` is broken: the reference can be assigned before the constructor's writes are visible, so another thread sees a half-built object (SKILL.md Traps).
- Preferred singleton: the holder-class idiom — `private static class H { static final X I = new X(); }` — the classloader guarantees safe, lazy, lock-free initialization.
- Never let `this` escape from a constructor (registering a listener, starting a thread, passing to a callback): other threads can observe the object before its fields are final.
- Safe publication channels: static initializer, `volatile`/`AtomicReference` field, final field of a safely published object, or a value put into a thread-safe collection.

## Choosing the Primitive

| Need | Use | Why not the alternative |
|---|---|---|
| Counter under contention | `LongAdder` | `AtomicLong` CAS-retries under heavy write contention; `LongAdder` stripes the count and sums on read |
| One-writer flag or stop signal | `volatile boolean` | A lock is unnecessary; an `AtomicBoolean` only if you need compare-and-set |
| Read-mostly map | `ConcurrentHashMap` | `Collections.synchronizedMap` serializes readers and needs manual locking for iteration |
| Producer/consumer handoff | `ArrayBlockingQueue` (bounded) | `LinkedBlockingQueue` with no capacity is unbounded and turns backpressure into an OOM (`memory.md`) |
| Wait for N tasks once | `CountDownLatch` | `CyclicBarrier` is for repeated rendezvous; a latch cannot be reset, which is usually what you want |
| Limit concurrent access to a resource | `Semaphore` | A pool of threads limits threads, not the downstream resource |
| Guard a compound operation | `synchronized` block or `ReentrantLock` | Atomics only make single operations atomic, never a pair of them |
| Timeout or interruptible acquisition | `ReentrantLock.tryLock(t, unit)` | `synchronized` cannot time out or be interrupted while waiting |

- `synchronized` is not slow: biased locking is gone (disabled by default in JDK 15, JEP 374), and an uncontended monitor is a CAS. Choose `ReentrantLock` for what it adds — timeouts, interruptibility, fairness, multiple conditions — not for speed.
- `synchronized(this)` and `synchronized` methods publish the lock to anyone holding the object; a `private final Object lock = new Object()` cannot be captured by a caller.
- Static `synchronized` locks the `Class` object; instance `synchronized` locks the instance. Mixing them protects nothing.

## ConcurrentHashMap Specifics

- Individual operations are atomic; sequences are not. `if (!map.containsKey(k)) map.put(k, v)` is a race — use `putIfAbsent`, `computeIfAbsent`, `merge`.
- `computeIfAbsent` holds a bin lock while your mapping function runs. If that function writes to the SAME map, you get an `IllegalStateException` ("recursive update") or a stall — compute the value first, then insert.
- `size()` is an estimate under concurrent modification; never use it as a loop bound or a correctness check.
- Iteration is weakly consistent: it reflects some state at some point, never throws `ConcurrentModificationException`, and may or may not see concurrent insertions.
- It rejects `null` keys and values, deliberately — `get(k) == null` would otherwise be ambiguous between "absent" and "mapped to null".

## Thread Pools

```java
ExecutorService pool = new ThreadPoolExecutor(
    8, 8, 0L, TimeUnit.MILLISECONDS,
    new ArrayBlockingQueue<>(1000),                       // bounded: backpressure instead of OOM
    new ThreadFactoryBuilderOfYourChoice("orders-%d"),    // named threads make dumps readable
    new ThreadPoolExecutor.CallerRunsPolicy());           // full queue slows the producer down
```

- `Executors.newFixedThreadPool` uses an unbounded queue: under overload, memory grows until OOM instead of rejecting work. `newCachedThreadPool` has the mirror problem — unbounded thread creation.
- The default rejection policy throws `RejectedExecutionException`. `CallerRunsPolicy` is usually the right choice for request pipelines: the producer feels the pressure.
- Name your threads. An unnamed `pool-3-thread-7` in a dump tells you nothing (`debug.md`).
- Shutdown is three steps: `shutdown()`, `awaitTermination(t)`, then `shutdownNow()` — and the last one only works if tasks honour interrupts (SKILL.md rule 3).
- An exception thrown by a task submitted with `submit()` is captured in the `Future` and silently lost if nobody calls `get()`. With `execute()` it reaches the thread's uncaught-exception handler. Set one on the factory either way.
- Never block on a future produced by the pool you are running in: the pool deadlocks with all threads waiting on work only that pool can do (`debug.md`).

## Virtual Threads (21+)

- One virtual thread per task, always. Do not pool them — creation is cheap, and a pool reintroduces the limit virtual threads exist to remove. `Executors.newVirtualThreadPerTaskExecutor()`.
- They win for **blocking I/O**. For CPU-bound work they add nothing: the carrier pool has one thread per core either way.
- **Pinning**: on JDK 21-23, blocking inside a `synchronized` block pins the carrier thread and can starve the whole carrier pool. JDK 24 (JEP 491) removed that pinning. On 21-23, replace `synchronized` with `ReentrantLock` on any path that performs I/O. Diagnose with `-Djdk.tracePinnedThreads=full`.
- Native calls and class-initialization blocks still pin on every release.
- Limit concurrency with a `Semaphore`, not a pool size: 10,000 virtual threads hitting a 20-connection database pool queue up in the driver where you cannot see them.
- `ThreadLocal` costs scale with thread count now — millions of virtual threads mean millions of copies. Scoped values (finalized in JDK 25) are the inheritance-friendly replacement.

## Deadlock and Livelock

- Deadlock needs a cycle in lock acquisition order. The fix that always works: define a global order (e.g. by `System.identityHashCode` or a business id) and take locks in it, everywhere.
- Detection is free: `jcmd <pid> Thread.print` prints "Found one Java-level deadlock" with both stacks for monitors and `ReentrantLock`s (`debug.md`).
- `tryLock` with a timeout plus backoff converts a deadlock into a retryable failure — acceptable when a global order is impossible, never as the first design.
- Never hold a lock across I/O, a callback, or a lock in another subsystem. Most deadlocks are two subsystems calling each other while each holds its own lock.
- Livelock: threads keep responding to each other and make no progress — usually retry loops without randomized backoff.

## Testing Concurrent Code

- A passing concurrency test proves nothing about a race; run the operation from `n` threads with a `CyclicBarrier` start and assert an invariant, thousands of times.
- Make failures deterministic where you can: inject the executor so tests can pass a same-thread executor, and separate "what happens concurrently" from "what the logic computes".
- Java Flight Recorder records lock contention and thread parks in production, which is where the real interleavings live (`performance.md`).
