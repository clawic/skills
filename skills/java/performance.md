# Performance — Measure, Then Change One Thing

Order of operations: reproduce with a load that resembles production → measure where the time goes → change one thing → measure again. Java performance intuition is wrong more often than in most languages because the JIT rewrites your code.

## Profiling in Production

- **JFR** is the always-available option: built into the JDK, low overhead with the `default` settings (on the order of 1%), higher with `profile`.
  ```bash
  java -XX:StartFlightRecording=settings=profile,duration=120s,filename=/tmp/app.jfr -jar app.jar
  jcmd <pid> JFR.start name=live settings=profile      # attach to a running JVM
  jcmd <pid> JFR.dump name=live filename=/tmp/app.jfr
  ```
  Read it with JDK Mission Control. Start from: hot methods, allocation by class, GC pauses, socket/file I/O time, thread parks, lock contention. JFR sees allocation and lock events that a sampling profiler cannot.
- **Safepoint bias**: profilers built on `Thread.getStackTrace`/JVMTI can only sample at safepoints, so their answers skew toward safepoint-heavy methods and away from tight loops that the JIT never polls. When JFR's picture looks implausible, confirm with a profiler that samples outside safepoints (async-profiler and similar use `AsyncGetCallTrace`/perf).
- Flame graphs answer "where is wall time going"; allocation profiles answer "why is GC running"; both are needed before touching code.
- Sampling for 30 seconds under real load beats a perfect benchmark of the wrong method.

## Benchmarking Without Fooling Yourself

Never time code with a `System.nanoTime()` loop in `main` — dead-code elimination, missing warmup, and on-stack replacement make the result meaningless. Use JMH:

```java
@BenchmarkMode(Mode.AverageTime) @OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5) @Measurement(iterations = 10) @Fork(2)
@State(Scope.Benchmark)
public class ParseBench {
    private String input;
    @Setup public void setup() { input = "..."; }
    @Benchmark public long parse() { return Long.parseLong(input); }   // returned, so it cannot be eliminated
}
```

- **Return every result** or consume it with a `Blackhole`. An unused computation is deleted by C2, and you measure an empty loop.
- **`@Fork` must be ≥1** (the default is fine, `0` is for debugging only): running in the harness JVM lets one benchmark's profile pollute the next one's compilation.
- Warmup exists because tiered compilation reaches C2 only after thousands of invocations (`Tier4InvocationThreshold=5000`, `Tier4CompileThreshold=15000`); the first iterations measure the interpreter and C1.
- Constant inputs get constant-folded. Take inputs from `@State` fields, not literals.
- Report the error bars JMH prints. A 3% difference inside overlapping intervals is not a difference.

## Where Java Time Actually Goes

| Symptom in the profile | Usual cause | Fix |
|---|---|---|
| High `GC Thread` CPU, low app CPU | Allocation rate, not heap size | Cut allocations in the hot path; check for boxing (below) |
| Many `String` and `char[]` instances | Parsing or logging in a loop | Parameterized logging, `StringBuilder` with a capacity hint, avoid `String.format` in hot code |
| `Integer`/`Long`/`Double` instances everywhere | Autoboxing in a generic collection or stream | Primitive streams and arrays; `IntArrayList`-style structures for hot data (`nulls.md`) |
| Threads parked on the same lock | Contention, not throughput | Shrink the critical section; `LongAdder` over `AtomicLong` under heavy write contention (`concurrency.md`) |
| Time in `HashMap.get` | Bad `hashCode` collapsing to few buckets | Fix the hash; HashMap converts an 8-entry bucket into a tree only when the table is ≥64 (`collections.md`) |
| Time in `ArrayList.grow`/`Arrays.copyOf` | Repeated resizing | Presize: `new ArrayList<>(expectedSize)`; `ArrayList` grows by ~1.5× |
| Time in reflection or proxies | Framework mapping per call | Cache the reflective handle; check whether the mapper can be pre-generated |
| Everything is I/O wait | Nothing to optimize in Java | Batch, cache, or parallelize the calls (`async.md`) |

## Allocation Discipline (the highest-leverage Java optimization)

- The JVM's allocation is fast (bump-the-pointer in TLAB) and young collection is proportional to SURVIVORS, not garbage — short-lived garbage really is nearly free. Optimize allocations that *survive*, and allocation *rate* in the very hottest loops.
- Escape analysis can stack-allocate an object that never escapes its method — real, but conditional on inlining and never something to rely on for correctness.
- The measurable wins in order: (1) stop boxing in loops, (2) presize collections and builders, (3) reuse expensive immutable objects (`Pattern`, `DateTimeFormatter`, `ObjectMapper` — all thread-safe), (4) avoid defensive copies of large arrays on every call.
- Object pooling is almost always a pessimization now: pooled objects survive, get promoted, and add scanning work. Pool only genuinely expensive resources (connections, threads, direct buffers).

## The JIT, Briefly

- Tiered compilation runs C1 first for fast startup, then C2 for peak throughput. Peak performance arrives seconds to minutes into the run.
- Inlining drives every other optimization; it stops at method size (`-XX:MaxInlineSize=35` bytes for cold, `-XX:FreqInlineSize=325` for hot). A giant hot method is not inlined into its caller and blocks further optimization — small methods are faster, not slower.
- Megamorphic call sites (an interface with many implementations hit at one location) lose inlining. This is why a strategy interface with 15 implementations can be slower than a switch.
- `-Xlog:class+load` and `-XX:+PrintCompilation` (or JFR's compilation events) show what is being recompiled; constant deoptimization/recompilation of the same method usually means an assumption keeps being invalidated (a rarely taken branch finally taken, a class loaded late).
- Do not micro-optimize what the JIT already handles: bounds-check elimination, loop unrolling, lock elision, and string concat via `invokedynamic` (JDK 9+) are done for you.

## Concurrency for Throughput

- Sizing: CPU-bound pool ≈ number of cores; I/O-bound pool ≈ `cores × (1 + wait/service)`. Worked: 8 cores, 90 ms waiting per 10 ms of work → 8 × (1 + 9) = 80 threads. That formula is a starting point to measure against, not a target.
- On JDK 21+, I/O-bound work with blocking calls scales better on virtual threads than on any tuned platform pool — the sizing question disappears (`concurrency.md`).
- Contention beats parallelism: two threads fighting over one lock are slower than one thread. Measure lock contention in JFR before adding threads.

## Before Claiming a Speedup

- Same JVM version, same flags, same data, same hardware, warm.
- Report a percentile, not an average — p99 is where users live, and averages hide GC pauses entirely.
- Re-run after the change with the profiler still attached: the bottleneck moves, and the second bottleneck is often bigger than the first.
