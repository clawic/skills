# Debugging — Symptom to Cause in Minutes

Work symptom-first. Each chain is ordered by probability and every step is a check, not a guess. Exception names decode in SKILL.md Exception Triage; this file covers the chains that a single exception name does not settle.

## The Universal First Four

1. **First exception, not last.** Scroll UP to the earliest failure in the log; everything after a failed initialization is a consequence.
2. **Whole cause chain.** `Caused by:` blocks end at the real defect; the top frame is usually framework plumbing. Truncated middles say `... 47 more` — the frames are in the enclosing trace.
3. **`jcmd <pid> VM.system_properties` and `VM.command_line`** — proves which JDK, which flags, and which classpath are actually running, which is frequently not what the deployment descriptor says.
4. **`jcmd <pid> Thread.print`** — three dumps, 10 seconds apart. One dump is a photograph; three are a movie, and the difference between them is the diagnosis.

## Process Hangs (no output, no exception)

1. `jcmd <pid> Thread.print | grep -A5 "Found one Java-level deadlock"` — HotSpot detects monitor and `ReentrantLock` cycles for you and names both threads. If it prints, you are done.
2. No deadlock reported: look for threads in `WAITING (parking)` on a pool queue vs `BLOCKED` on a monitor. All-BLOCKED on one monitor = a lock holder doing I/O inside the critical section.
3. Thread-pool starvation shape: every worker in the pool is `WAITING` on a future produced by the SAME pool. Fix is a separate pool, never a bigger one (`async.md`).
4. The JVM exits nothing and the process persists after `main` returns: a non-daemon thread. `Thread.print` and look for a live thread you created without `setDaemon(true)` — Executors' default factory produces non-daemon threads.
5. Hangs only on shutdown: an unclosed `ExecutorService` or a shutdown hook blocking. `shutdown()` + `awaitTermination` + `shutdownNow()` is the three-step, and it only works if tasks honour interrupts (SKILL.md rule 3).
6. Every worker parked in `SocketRead` or `getConnection`: a downstream call with no read timeout, or an exhausted connection pool — not a Java bug (`http.md`, `jdbc.md`).
7. Hangs during class loading (rare, ugly): stack shows `<clinit>` in two threads referencing each other's classes — circular static initialization deadlock. Break the cycle; no flag fixes it.

## 100% CPU

The exact procedure, no profiler needed:

```bash
top -H -p <pid>            # per-THREAD CPU; note the hot tid (decimal)
printf '%x\n' <tid>        # convert to hex — this is "nid" in the dump
jcmd <pid> Thread.print > dump.txt
grep -A20 'nid=0x<hex>' dump.txt
```

- Hot thread named `C2 CompilerThread` → the JIT is compiling, not your code; normal for the first minutes after a deploy, suspicious if permanent (huge generated methods, `performance.md`).
- Hot threads named `GC Thread#*` → not a CPU bug, a memory bug: the heap is too small or leaking (`memory.md`).
- Hot thread is yours, same frame in all three dumps → an actual loop. Same frame but a moving line number → normal work, use a profiler (`performance.md`).
- CPU high with tiny load and `epollWait` everywhere → a busy-wait loop somewhere with `while(true)` and no blocking call.

## "Works in the IDE, Fails from the Jar"

| Difference | Check |
|---|---|
| Classpath resource read as a `File` | `getResource().getFile()` works from `target/classes`, throws inside a jar — use `getResourceAsStream` (`io.md`) |
| Different JDK | `java -version` in the terminal vs the IDE's project SDK; `UnsupportedClassVersionError` is this |
| Missing `META-INF/services` | Shading merged jars without `ServicesResourceTransformer`; SPI lookups return nothing (`build.md`) |
| Duplicate class from two jars | `jar tf app.jar \| sort \| uniq -d`, or `mvn dependency:tree -Dverbose` |
| Working directory | The IDE runs in the module dir; the jar runs wherever systemd/Docker put it — relative paths break |
| Annotation processing | IDE runs its own compiler; Lombok/MapStruct output can differ from the build's |
| Default charset or locale | IDE sets `-Dfile.encoding`; the server may not. On JDK <18 this changes behavior (`text.md`) |

## "Works Locally, Fails in CI/Prod"

- **Test ordering.** Locally you ran one test; CI ran the class. Static state leaking between tests is the top cause (`testing.md`).
- **Time zone.** CI runs UTC, your laptop does not. Any assertion on a `LocalDate` derived from `Instant` flips at the offset boundary (`datetime.md`).
- **Locale.** `String.format("%.2f", x)` prints `1,50` under a comma locale — the assertion compares `1.50` (`text.md`).
- **Parallelism.** CI runners have fewer CPUs: `availableProcessors()` changes pool sizes, and `commonPool` parallelism can drop to zero (SKILL.md rule 9).
- **Memory.** The container limit, not the laptop's 32 GB, decides the default heap (`jvm.md`).
- **Dependency resolution.** A `SNAPSHOT` or a version range resolves to a different jar on a different day. Pin (`build.md`).

## Wrong Values, No Exception

1. Equality: is the comparison `==` on boxed types or strings? (SKILL.md rule 1.)
2. Mutation: did something hand out an internal `List` that a caller then modified? Return `List.copyOf(...)` and see if the bug moves to an exception.
3. Floating point: `0.1 + 0.2 != 0.3`. Money in `BigDecimal` (with an explicit `RoundingMode`) or integer minor units; `BigDecimal.equals` compares scale too, so `2.0` ≠ `2.00` — use `compareTo() == 0`.
4. Integer overflow is silent: `Math.multiplyExact`/`addExact` throw instead. `Math.abs(Integer.MIN_VALUE)` is negative.
5. Integer division: `1/2*2.0` is `0.0`. Promote before dividing.
6. Character encoding: `?` or `Ã©` in output means a charset mismatch at a boundary, not corrupt data (`text.md`).

## Remote and Live Inspection

```bash
jcmd -l                                   # PIDs and main classes
jcmd <pid> VM.uptime                      # how long it has been running
jcmd <pid> VM.flags -all                  # what the JVM actually chose
jcmd <pid> GC.heap_info                   # live heap breakdown
jcmd <pid> Thread.print                   # thread dump
jcmd <pid> GC.class_histogram             # top classes by instance count (safepoint pause; ok in prod, briefly)
jcmd <pid> JFR.start name=r settings=profile duration=60s filename=/tmp/r.jfr
```

- Remote debugging: `-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=127.0.0.1:5005`. **Bind to localhost and tunnel over SSH** — an open JDWP port is unauthenticated remote code execution (`security.md`).
- Conditional breakpoints in a hot loop can slow the JVM by orders of magnitude — the debugger evaluates the condition in the interpreter. Prefer a log statement with a guard.
- Cannot attach at all: the target runs as a different user, or `-XX:+DisableAttachMechanism` is set, or the container lacks `SYS_PTRACE`. Run `jcmd` from inside the container namespace.

## Logging That Survives Debugging

- Log the throwable object, never `e.getMessage()` (SKILL.md Traps). One `log.error("saving order {}", id, e)` gives context and stack; SLF4J takes the trailing throwable without a placeholder.
- Add a correlation id at the entry point and put it in the MDC; without it, concurrent requests interleave and every trace is unreadable.
- Guard expensive log arguments with parameterized logging (`log.debug("{}", obj)`), not string concatenation — concatenation runs even when DEBUG is off.
- A log line that says only "error occurred" costs the same as one that names the input. Always name the input.
- Logs missing entirely, duplicated, or ignoring your level configuration is a wiring problem, not a code problem (`logging.md`).

## When You Are Truly Stuck

Reduce to a single class with a `main`, no framework, no DI, no build plugin — copy in the failing call and its inputs. Either it fails (you now own a 20-line reproducer) or it passes (the defect is in the wiring, not the logic, and `spring.md` or `build.md` is your next file).
