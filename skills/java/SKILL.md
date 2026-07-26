---
name: Java
slug: java
version: 1.0.3
changelog: "Display name shown correctly"
description: >-
  Writes, debugs, and tunes Java and the JVM: nulls, equality, collections, generics, concurrency, streams, memory, builds, and upgrades.
  Use when Java throws NullPointerException, ClassCastException, ConcurrentModificationException, NoClassDefFoundError, NoSuchMethodError,
  UnsupportedClassVersionError or OutOfMemoryError, when a service deadlocks, hangs, burns CPU, or leaks memory, when equals/hashCode,
  Optional, generics, streams, or CompletableFuture misbehave, when dates shift or text turns into question marks, when Maven or Gradle
  resolves the wrong version or a fat jar fails only after packaging, when @Transactional does not roll back or an entity throws
  LazyInitializationException, when tests silently do not run, or when moving from Java 8 to 17/21+ and javax became jakarta.
  Also for choosing collections, sizing heap in containers, thread dumps, profiling, JDBC connection pools, HTTP timeouts, and logging setup.
  Not for Kotlin, Android app development, or JavaScript.
homepage: https://clawic.com/skills/java
metadata:
  clawdbot:
    emoji: ☕
    requires:
      bins:
      - java
      - javac
    os:
    - linux
    - darwin
    - win32
    displayName: Java
    configPaths:
    - ~/Clawic/data/java/
---

User preferences and memory live in `~/Clawic/data/java/` (see `setup.md` on first use, `memory-template.md` for the file format). If you have data at an old location (`~/java/` or `~/clawic/java/`), move it to `~/Clawic/data/java/`.

## When To Use

- Writing or reviewing Java: APIs, data modelling, collections, generics, error handling, concurrency
- Debugging a running JVM: exceptions, crash loops, deadlocks, 100% CPU, growing heap, slow startup
- Sizing heap and GC for a container, reading a thread dump or heap dump, profiling and benchmarking
- Fighting the build: dependency version conflicts, fat jars, module errors, `--release` mismatches
- Upgrading a JDK (8 → 17 → 21+), migrating javax → jakarta, replacing removed APIs
- Testing: JUnit 5, Mockito, Testcontainers, tests that silently don't run or fail only in CI
- Integrating outward: HTTP clients and timeouts, JDBC pools and batching, logging wiring
- Not for Kotlin syntax, Android SDK/UI, or JavaScript — different skills; JVM-level advice here still applies under Kotlin

## Quick Reference

| Situation | Go to |
|---|---|
| An exception or error whose name you must decode; hang, deadlock, 100% CPU, works-in-IDE-only | `debug.md` |
| OutOfMemoryError, heap grows over days, heap dump analysis, container OOM-kill | `memory.md` |
| Choosing a GC, setting `-Xmx` in a container, slow startup, JVM flags, `-Xlog` | `jvm.md` |
| Slow code and no idea where; JMH benchmark, JFR recording, allocation pressure | `performance.md` |
| Shared mutable state, locks, `volatile`, atomics, deadlock design, virtual threads | `concurrency.md` |
| CompletableFuture chains, executors, timeouts, cancellation, structured concurrency | `async.md` |
| Which collection, the `equals`/`hashCode` contract, iteration traps, comparators, maps | `collections.md` |
| Stream pipeline wrong or slow, collectors, `groupingBy`, parallel streams | `streams.md` |
| Lambdas and method references, designing a `@FunctionalInterface`, capture rules, a checked exception inside a lambda | `lambdas.md` |
| Designing against null, `Optional`, autoboxing, nullability annotations | `nulls.md` |
| Type erasure, wildcards, `List<Dog>` vs `List<Animal>`, unchecked warnings | `generics.md` |
| Class design: records, sealed types, pattern matching, immutability, inheritance | `classes.md` |
| Custom annotations, runtime metadata, `setAccessible`, `MethodHandle`/`VarHandle`, dynamic proxies, annotation processors | `reflection.md` |
| Checked vs unchecked, try-with-resources, retries, interrupts, logging failures | `exceptions.md` |
| Strings, `StringBuilder`, regex, `String.format`, charsets, locale-sensitive output | `text.md` |
| Dates, time zones, DST, `Instant` vs `LocalDateTime`, formatting patterns | `datetime.md` |
| Files, `Path`, classpath resources, temp files, streams that leak handles | `io.md` |
| Jackson/JSON mapping, records in JSON, Java serialization and its CVEs | `serialization.md` |
| Calling another service: HTTP client choice, timeouts, DNS caching, TLS handshake errors | `http.md` |
| JDBC connections, pool exhaustion, batching, fetch size, driver and transaction behavior | `jdbc.md` |
| SLF4J bindings, duplicate or missing log output, MDC, log levels, structured logs | `logging.md` |
| Maven/Gradle version conflicts, scopes, fat jars, multi-module, reproducible builds | `build.md` |
| Upgrading a JDK, javax → jakarta, removed APIs, `--add-opens`, illegal reflective access | `migration.md` |
| Deserialization gadgets, XXE, path traversal, SQL injection, TLS, secrets, crypto choices | `security.md` |
| JUnit 5, Mockito, AssertJ, Testcontainers, flaky tests, tests that never ran | `testing.md` |
| Spring Boot: `@Transactional`, proxies, JPA lazy loading, N+1, bean wiring, config precedence | `spring.md` |
| Anything else | Exception Triage and Core Rules below; then reproduce it in a single `main` with no framework |

## Core Rules

1. **`.equals()` for content; `==` only for primitives, enums, and deliberate identity.** `Integer a = 128, b = 128; a == b` is **false**, while at 127 it is true — the autobox cache is −128..127 (`-XX:AutoBoxCacheMax` moves only the upper bound). Use `Objects.equals(a, b)` whenever either side can be null.
2. **`equals` and `hashCode` ship together, over fields that never change.** Contract: equal objects must return the same hash; unequal objects may collide. Failure mode: `set.add(o)`, then mutate a field used in `hashCode()` → `set.contains(o)` is false and the entry is unreachable forever. Hash only final fields (→ `collections.md`).
3. **Never swallow `InterruptedException`.** `catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }` — the flag is the only channel that tells the pool to stop; swallowing it makes every `shutdownNow()` wait out the full timeout and every cancellation silently fail.
4. **Every `AutoCloseable` in try-with-resources** — including `Files.lines`, `Files.walk`, JDBC `Connection`/`Statement`/`ResultSet`, and `Scanner`. Resources close in reverse order, and a failure inside `close()` arrives as `e.getSuppressed()` instead of masking the real exception. Leaked handles surface hours later as "Too many open files" (→ `io.md`).
5. **Size the container, not just the heap.** `RSS ≈ Xmx + metaspace + code cache + (threads × Xss) + direct buffers + GC structures`. Worked: `-Xmx512m` + ~100 MB metaspace + ~100 MB code cache + 200 threads × 1 MB stack + 64 MB direct ≈ 976 MB — a 1 GiB limit gets OOM-killed at peak with the heap only half full. In containers set `-XX:MaxRAMPercentage=75` (the JVM's own default is 25%) and leave the remainder for the non-heap terms (→ `jvm.md`).
6. **`Optional` is a return type.** `orElse(buildDefault())` evaluates its argument on every call even when a value is present; `orElseGet(() -> buildDefault())` does not. Never a field, parameter, or collection element — it adds a second empty state and is not serializable (→ `nulls.md`).
7. **Pin dependency versions in exactly one place.** Maven picks the **nearest** declaration in the tree (ties: first declared, not the highest); Gradle picks the **highest** version it sees. The same dependency graph therefore yields different jars in the two tools. Declare in `<dependencyManagement>` or a Gradle `platform`, and verify the winner with `mvn dependency:tree -Dverbose` (→ `build.md`).
8. **Compile with `--release N`, never `-source/-target N`.** `-source 8 -target 8` on a JDK 17 compiles happily against JDK 17 APIs and then dies at runtime on Java 8 with `NoSuchMethodError`; `--release 8` also restricts the visible API set. Class-file major version = **JDK + 44** (52 = Java 8, 55 = 11, 61 = 17, 65 = 21).
9. **Streams to transform, loops to mutate.** Go parallel only when all three hold: per-element work is real, the source splits evenly (arrays, `ArrayList`, `IntStream.range` — not `LinkedList`, `Files.lines`, `Stream.iterate`), and nothing shared is mutated. Parallel streams run on `ForkJoinPool.commonPool`, whose parallelism is `availableProcessors() − 1` — in a 1-CPU container that is 0 extra threads, so "parallel" runs entirely on the calling thread (→ `streams.md`).

## Exception Triage

Read the FIRST exception in the log, not the last: the later ones are usually consequences. The `getCause()` chain matters more than the top frame.

| Symptom | What it really means | First move |
|---|---|---|
| `NullPointerException` with a helpful message ("Cannot invoke ... because `x.y` is null") | Helpful NPE messages, on by default since JDK 15 | Read the message — it names the exact expression that was null |
| `NullPointerException` with **no stack trace** | The JIT recompiled a hot throw site to reuse a preallocated exception | Restart with `-XX:-OmitStackTraceInFastThrow` and reproduce |
| `NoClassDefFoundError` | The class existed at compile time but not at runtime — OR its static initializer threw earlier | Search upward in the log for the first `ExceptionInInitializerError`; that one carries the real cause |
| `ClassNotFoundException` | A by-name lookup (reflection, JDBC driver, SPI) failed | Check the runtime classpath, and whether shading dropped `META-INF/services` (`build.md`) |
| `NoSuchMethodError` / `NoSuchFieldError` | Version skew: compiled against one jar, running against another | `mvn dependency:tree -Dverbose -Dincludes=<artifact>` (`build.md`) |
| `UnsupportedClassVersionError: class file version 65.0` | Built for a newer JDK than the one running it; major − 44 = JDK (65 → 21) | Align `--release` with the runtime JDK (`migration.md`) |
| `ClassCastException` naming the SAME class on both sides | Two classloaders loaded it (fat jar plus a container-provided copy) | Remove the duplicate; mark the provided one `provided`/`compileOnly` |
| `ConcurrentModificationException` | Structural modification during iteration — single-threaded in most sightings, not a concurrency bug | `Iterator.remove()` or `removeIf` (`collections.md`) |
| `StackOverflowError` | Unbounded recursion, or two objects whose `toString`/`equals` call each other | Read the repeating frame cycle in the trace |
| `OutOfMemoryError` (any flavour) | Six distinct causes with different fixes | `memory.md` — the message text selects the chain |
| `IllegalStateException: stream has already been operated upon or closed` | A stream reused after its terminal operation | Rebuild the stream from its source (`streams.md`) |
| `IllegalMonitorStateException` | `wait`/`notify` called without holding that object's monitor | `concurrency.md` |
| Process hangs with no exception at all | Deadlock, a non-daemon thread that never ends, or a blocked unbounded queue | Three thread dumps 10s apart (`debug.md`) |

## Version Floors

Check before suggesting an API: it compiles on your JDK and fails on theirs.

| Feature | Minimum JDK | Note |
|---|---|---|
| `var` for locals | 10 | Lambda parameters: 11 |
| `HttpClient`, single-file source launch | 11 | Last LTS where `javax.*` was still the norm |
| Text blocks (`"""`) | 15 | Incidental trailing whitespace is stripped |
| Helpful NullPointerException messages | 15 | On by default from 15; before that, opt-in |
| `instanceof` pattern, records, `Stream.toList()` | 16 | `toList()` is unmodifiable and null-tolerant; `Collectors.toUnmodifiableList()` rejects nulls |
| Sealed classes and interfaces | 17 | First LTS enforcing strong encapsulation of JDK internals |
| UTF-8 as the default charset | 18 | JEP 400 — before this the default was platform-dependent |
| Virtual threads, pattern matching for `switch`, record patterns, sequenced collections | 21 | `SequencedCollection.getFirst()`, `reversed()` |
| `synchronized` no longer pins a virtual thread's carrier | 24 | On 21-23, use `ReentrantLock` inside virtual threads (`concurrency.md`) |
| Structured concurrency (`StructuredTaskScope`) | preview | Still a preview API through JDK 25 — requires `--enable-preview`, and its shape changed between previews |

## Output Gates

Before emitting Java code or a build change, verify:

- Every `AutoCloseable` is inside try-with-resources?
- `equals` and `hashCode` overridden together, computed from final fields only?
- Charset, `Locale`, and time zone explicit wherever text, numbers, or time cross a boundary?
- No raw types, and every `@SuppressWarnings("unchecked")` justified in a comment?
- Every caught `InterruptedException` restores the flag or rethrows?
- Every API used is at or below the configured `jdk_version` (→ Version Floors)?
- New dependency versions declared in one place, not inline per module?
- No SQL, shell command, or file path built by concatenating input (`security.md`)?

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/java/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| jdk_version | number (JDK major, 8-25) | from `maven.compiler.release`, `<java.version>`, or the Gradle toolchain if present, else 21 | Gates every API and syntax suggestion against Version Floors; selects flag syntax in `jvm.md` and the target in `migration.md` |
| build_tool | maven \| gradle \| other | detected (`pom.xml` → maven, `build.gradle*` → gradle), else maven | Selects the resolution rules, commands, and packaging advice in `build.md`, which covers Maven and Gradle only; `other` (Bazel, Ant, plain `javac`) suppresses tool-specific commands and keeps the advice at classpath and jar level |
| framework | spring-boot \| other \| none | detected from dependencies, else none | `spring-boot` enables `spring.md` routing for proxies, `@Transactional`, and JPA; `other` (Quarkus, Micronaut, Jakarta EE) keeps guidance at JDK and specification level and states that container-specific DI and transaction semantics are not covered here; `none` assumes plain Java |
| locale | text (BCP 47 tag, e.g. `es-ES`) | none — `Locale.ROOT` for machine-facing output, the caller's locale for display; `~/Clawic/profile.yaml` is the fallback | Fills the explicit `Locale` argument in formatting, collation, and case-mapping guidance (`text.md`) and picks the locale used in worked examples |
| timezone | text (IANA zone id, e.g. `Europe/Madrid`) | none — every example takes an explicit `ZoneId`, never the system default; `~/Clawic/profile.yaml` is the fallback | The zone assumed when a wall-clock time arrives without one, and the zone shown in `datetime.md` examples |
| default_charset | text (charset name) | utf-8 | The charset written into every explicit `Charset` argument in `text.md` and `io.md`; any value other than utf-8 also turns on the legacy-encoding warnings around `file.encoding` and the JDK 18 default change |
| lombok | bool | false | false writes explicit constructors, getters, and `equals`; true writes Lombok annotations and skips the boilerplate sections of `classes.md` |
| nullability_style | jspecify \| jakarta \| jetbrains \| none | none | Which `@Nullable`/`@NonNull` annotations appear in generated signatures (`nulls.md`) |
| preview_features | bool | false | When true, `--enable-preview` APIs (structured concurrency) become admissible suggestions |
| test_stack | junit5 \| junit4 \| testng | junit5 | Selects assertion and lifecycle idioms in `testing.md`; junit4 turns on the vintage-engine warnings |

Preference areas to record as the user reveals them:

- **tooling** — IDE, formatter (google-java-format, palantir, spotless), static analysis (ErrorProne, SpotBugs, NullAway)
- **conventions** — package layout, immutability default, builder vs constructor, logging facade and message style, checked-exception policy
- **platform** — container vs bare metal, target CPU architecture, GC choice, cloud provider, application server, and the deployment's locale, time zone, and charset when they differ from the `locale`/`timezone`/`default_charset` variables
- **output** — depth of explanation (one-line fix vs full diagnosis), whether the reasoning precedes or follows the patch, diff vs whole file, comment density in generated code
- **work order** — propose-then-apply vs editing directly, review gate before touching build files or dependency versions, whether to compile and run the tests before handing back, coverage gate
- **cadence** — how often to rebuild for CVEs with no code change (`build.md`), the JDK upgrade window (`migration.md`), and whether to raise upgrades between windows
- **safety posture** — how proactively to flag legacy APIs (Java serialization, `SimpleDateFormat`, raw types) and to propose dependency or JDK upgrades, vs only on request
- **restrictions** — banned APIs or libraries, no-preview-features rule, compliance regime (FIPS crypto, no reflection, offline builds)

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `log.error(e.getMessage())` | Drops the stack trace, and the message is `null` for NPEs and many wrapped exceptions | `log.error("context", e)` — the throwable is a separate argument |
| `new String(bytes)`, `getBytes()`, `FileReader`, `PrintWriter(file)` | Platform default charset; UTF-8 only became the default in JDK 18, so the same code writes different bytes on an older JVM or on Windows | Pass `StandardCharsets.UTF_8` explicitly every time |
| `list.remove(someInt)` on a `List<Integer>` | The `remove(int)` overload wins over `remove(Object)` — it removes by INDEX and can throw `IndexOutOfBoundsException` | `list.remove(Integer.valueOf(x))` |
| `SimpleDateFormat` in a static or shared field | Not thread-safe; under load it returns silently wrong dates rather than throwing | `DateTimeFormatter` — immutable and thread-safe (`datetime.md`) |
| `Collectors.toMap(k, v)` on data where a key can repeat | Throws `IllegalStateException` only for inputs that collide, so it passes tests and fails in production | Supply a merge function: `toMap(k, v, (a, b) -> b)` |
| `@Transactional` called from another method of the same class | Self-invocation bypasses the proxy: no transaction, no warning, no rollback | Move the annotated method into another bean (`spring.md`) |
| JUnit 4 annotations left in a JUnit 5 project | `org.junit.Test` classes are simply not executed by the Jupiter engine — a green build running zero tests | One engine, or add the vintage engine deliberately (`testing.md`) |
| `Files.lines()` / `Files.walk()` outside try-with-resources | Holds the file handle until GC; the failure shows up hours later as "Too many open files" | try-with-resources (Core Rule 4) |
| `printStackTrace()` in server code | Writes to stderr, detached from the request context and invisible to log aggregation | Logger with the throwable |
| `catch (Exception e) {}` around a retry loop | Also catches `InterruptedException` and programming errors, turning an outage into silence | Catch the specific exception; rethrow `Error` and restore interrupts (`exceptions.md`) |
| An HTTP or JDBC call with no read timeout | The default in most Java clients is unlimited: one slow dependency parks every worker thread and the whole service stops answering | Set connect AND read timeouts on every client (`http.md`, `jdbc.md`) |
| Double-checked locking without `volatile` | The reference can be published before the constructor finishes — another thread sees a half-built object | Holder-class idiom, or `volatile` on the field (`concurrency.md`) |
| Turning on `spring.jpa.open-in-view` to silence `LazyInitializationException` | Holds a DB connection for the whole request, converting a query bug into pool exhaustion under load | Fetch what the view needs in the query (`spring.md`) |

## Where Experts Disagree

- **Checked exceptions.** Bloch defends them for conditions a caller can actually recover from; most modern frameworks wrap everything unchecked. Working boundary: checked only when the caller has a real alternative path — and never across a lambda or stream, where they do not compose.
- **`Optional` beyond return types.** Its designers scoped it to library return values; a school uses it for fields and parameters anyway. Boundary: return types yes; entity fields, DTOs, and hot loops no (extra allocation, not serializable, two empty states to test).
- **Lombok.** Removes real boilerplate vs it is an annotation processor that breaks on JDK upgrades and hides behavior from readers and tools. Since records (16) cover immutable carriers, its honest remaining use is `@Slf4j` and `@Builder` on mutable entities.
- **Mocks vs real dependencies in tests.** Mock what you own and what is slow or non-deterministic; run repositories and SQL against a real engine (Testcontainers). A mocked JDBC layer verifies the mock, not the query.
- **Virtual threads vs reactive.** Since 21, virtual threads deliver most of the throughput of reactive code with straight-line control flow and readable stack traces. Reactive still wins where you genuinely need backpressure across a streaming pipeline — that need, not fashion, is the criterion.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/java (install if the user confirms):
- `kotlin` — the other JVM language; same bytecode, different null and concurrency model
- `android` — Android SDK, lifecycle, and app packaging
- `sql` — the queries your JDBC and JPA code actually sends
- `docker` — containerizing a JVM and matching memory limits to heap

## Feedback

- If useful, star it: https://clawic.com/skills/java
- Latest version: https://clawic.com/skills/java

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/java.
