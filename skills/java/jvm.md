# JVM — Sizing, Garbage Collectors, Flags, and Startup

Defaults come from ergonomics at startup, derived from the CPU and memory the JVM *believes* it has. In containers that belief is the whole game.

## What the JVM Picks by Default

- **Heap**: max = 25% of available memory (`-XX:MaxRAMPercentage=25`), initial = 1/64 of it. On a 4 GB container that is a 1 GB heap and 3 GB unused.
- **Collector**: G1 since JDK 9 on any "server-class" machine (≥2 CPUs and ≥1792 MB memory); below that threshold the JVM silently picks SerialGC — the reason a 1-CPU container behaves nothing like the 8-core laptop.
- **Container awareness**: on by default (`-XX:+UseContainerSupport`) since 8u191/10, so the JVM reads cgroup limits and `docker run -m 512m` really does cap the heap calculation. Older JVMs read HOST memory and OOM-killed immediately.
- **CPU count**: `availableProcessors()` derives from the cgroup CPU quota. A 0.5-CPU limit rounds to 1, which sets `commonPool` parallelism to 0 and most pool sizes to 1 (`streams.md`).

## Sizing a Container

Budget formula (canonical statement: SKILL.md rule 5): `RSS ≈ Xmx + metaspace + code cache + (threads × Xss) + direct buffers + GC structures`.

```
-XX:MaxRAMPercentage=75      # heap takes 75% of the limit; the rest covers the other terms
-XX:MaxMetaspaceSize=256m    # a ceiling turns a slow classloader leak into a clear error
-XX:MaxDirectMemorySize=256m # otherwise it defaults to roughly the max heap, doubling the worst case
-Xss512k                     # only when thread count is high; below ~256k, deep recursion breaks
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/app/
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m
```

- Worked check on a 1 GiB limit: 75% → `Xmx` 768 MB, plus ~100 MB metaspace, ~100 MB code cache, 100 threads × 512 KB = 50 MB, plus direct buffers — already over budget. Either raise the limit or drop to `MaxRAMPercentage=50`. The percentage is a starting point you verify against measured RSS, not a law.
- `-Xms` = `-Xmx` in production: it removes heap-resize pauses and makes the process fail at startup rather than at 3am if the box cannot back the memory.
- Off-heap is invisible in a heap dump. If RSS exceeds heap by more than ~40% and keeps climbing, start with `-XX:NativeMemoryTracking=summary` and read `jcmd <pid> VM.native_memory` (`memory.md`).

## Choosing a Collector

| Collector | Flag | Pause profile | Choose when |
|---|---|---|---|
| G1 | default (`-XX:+UseG1GC`) | Tens of ms, target-driven | The default answer for heaps of roughly 2-32 GB; leave it alone until you measured a problem |
| Parallel | `-XX:+UseParallelGC` | Longer pauses, highest throughput | Batch jobs where total runtime matters and pauses do not |
| ZGC | `-XX:+UseZGC` | Sub-millisecond, largely independent of heap size | Latency SLOs and large heaps; costs some throughput and more RSS |
| Shenandoah | `-XX:+UseShenandoahGC` | Sub-millisecond | Same niche as ZGC; availability depends on your JDK build |
| Serial | `-XX:+UseSerialGC` | Stop-the-world, single thread | Tiny containers and short-lived CLIs — genuinely the best choice under ~512 MB with 1 CPU |

- Tune the target before the collector: `-XX:MaxGCPauseMillis=200` is G1's default goal; lowering it makes G1 shrink the young generation, which raises GC frequency and lowers throughput. No setting gives you both.
- Never copy a CMS-era flag list. CMS was removed in JDK 14, and an unrecognized `-XX` flag makes the JVM refuse to start (`migration.md`).
- Fixed young-generation sizes (`-Xmn`) disable G1's adaptive sizing. Set them only with a measurement saying the adaptation is wrong.
- Compressed object pointers stop working above a ~32 GB heap: references become 8 bytes instead of 4, so a 33 GB heap can hold LESS live data than a 31 GB one. Either stay below the threshold or jump well past it.

## Reading the GC Log

- Unified logging replaced the old flags in JDK 9: `-Xlog:gc*` in, `-XX:+PrintGCDetails` out. An old flag on a new JVM is a startup failure, not a warning.
- Health metrics: **percentage of wall time spent in GC** and the **live set after full GCs**. Individual pause times matter only against your latency SLO.
- `Pause Full (Allocation Failure)` repeating = the heap cannot hold the live set. More heap, or fix the leak; tuning will not save it.
- `to-space exhausted` in G1 = evacuation failure, no room to copy survivors. Raise the heap or `-XX:G1ReservePercent`.
- Humongous allocations (objects larger than half a G1 region; region size defaults to heap/2048, clamped to 1-32 MB) get dedicated regions and fragment the heap. Large `byte[]` buffers are the usual source — shrink them or raise `-XX:G1HeapRegionSize`.
- `-XX:+UseStringDeduplication` (G1) reclaims duplicate `char[]`/`byte[]` backing arrays of equal strings. Worth measuring in string-heavy services, useless elsewhere.

## Startup Time

- Class loading and JIT warmup dominate the first seconds. Order of effect for a typical service: fewer classes on the startup path > AppCDS > tiered-compilation limits > everything else.
- **AppCDS** maps a pre-parsed class archive at startup:
  ```
  java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar     # once, exercising the startup path
  java -XX:SharedArchiveFile=app.jsa -jar app.jar        # every run afterwards
  ```
  The archive is tied to the exact JDK build and classpath — regenerate it in the image build; a stale archive is ignored with a warning rather than failing.
- `-XX:TieredStopAtLevel=1` (C1 only) makes short-lived CLIs and test runs start faster and long-running servers permanently slower. Never on a service.
- Lazy-initialize what the first request does not need; with Spring, `spring.main.lazy-initialization=true` measures how much of startup is bean construction (`spring.md`).
- GraalVM native images trade startup for build complexity and explicit reflection configuration — a different tool, not a flag.

## Flag Hygiene

- `jcmd <pid> VM.flags -all | grep <name>` shows the effective value and whether it came from `default`, `ergonomic`, or the command line. This is how you prove a flag took effect.
- `java -XX:+PrintFlagsFinal -version` lists every flag that exists in that build — the only authority on whether a flag survived your JDK upgrade.
- Two conflicting flags: the last one on the command line wins, silently. Keep JVM options in one place, not spread across a Dockerfile, an entrypoint script, and `JAVA_TOOL_OPTIONS`.
- `JAVA_TOOL_OPTIONS` applies to every JVM started in that environment, including Maven, Gradle, and the test JVMs — convenient for agents, dangerous for heap sizes.
- Experimental flags need `-XX:+UnlockExperimentalVMOptions` first, diagnostic ones `-XX:+UnlockDiagnosticVMOptions`. Without the unlock the JVM refuses to start rather than ignoring the flag.
- `-XX:+ExitOnOutOfMemoryError` for containerized services: a JVM that survives an OOM in a degraded state is worse than one the orchestrator restarts.
