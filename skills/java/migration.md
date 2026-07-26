# Migration — Upgrading the JDK, javax → jakarta, Removed APIs

Upgrade order that minimizes risk: **run old bytecode on the new JDK first**, fix what breaks at runtime, and only then raise `--release` and adopt new language features. Java's backward compatibility is strong enough that step one usually works, and it separates runtime breakage from compiler breakage.

## The Upgrade Sequence

1. Update the build tooling first (Maven/Gradle, Surefire, compiler plugins, Lombok, Mockito, ASM-based libraries). Bytecode-manipulating libraries fail on a new class-file version with `UnsupportedOperationException` or `IllegalArgumentException` from ASM — before your code even runs.
2. Run the existing artifact on the new JDK. Fix startup failures (removed flags), then reflective-access errors.
3. Raise `--release` (SKILL.md rule 8), recompile, and fix compiler errors and new warnings.
4. Adopt new APIs and syntax last, guarded by the Version Floors table in SKILL.md.
5. Re-measure: GC behavior, startup time, and memory all change across major versions (`jvm.md`).

## Class-File Versions

`major = JDK + 44`. 52 = Java 8, 53 = 9, 55 = 11, 61 = 17, 65 = 21, 69 = 25. `UnsupportedClassVersionError: class file version 65.0` means a JDK 21 artifact on an older runtime — read the number, do not guess (SKILL.md Exception Triage). Inspect any jar with `javap -v Foo.class | head -3` or `unzip -p app.jar META-INF/MANIFEST.MF`.

## Breakage by Version

| Change | Version | What it breaks |
|---|---|---|
| Locale data switched to CLDR | 9 | Formatted dates, month names, and number patterns differ from Java 8's output — assertions on formatted text fail (`text.md`) |
| Old GC logging flags removed (`-XX:+PrintGCDetails`) | 9 | The JVM refuses to start; use `-Xlog:gc*` (`jvm.md`) |
| Java EE and CORBA modules removed | 11 | `javax.xml.bind`, `javax.annotation`, `javax.activation` vanish → add the Jakarta artifacts explicitly |
| CMS collector removed | 14 | `-XX:+UseConcMarkSweepGC` is now a fatal unknown flag |
| Strong encapsulation of JDK internals by default | 16/17 | `InaccessibleObjectException` on reflection into `java.*`; `--illegal-access` no longer helps (it was removed in 17) |
| SecurityManager deprecated, then permanently disabled | 17 / 24 | Code calling `System.setSecurityManager` fails; policy-based sandboxes are gone |
| Default charset became UTF-8 | 18 | File and string encoding changes silently on Windows and on JVMs that relied on the platform default (`text.md`) |
| `Thread.stop` throws `UnsupportedOperationException` | 20 | Legacy "kill the thread" code stops working; there is no replacement by design |
| `synchronized` no longer pins virtual threads | 24 | Not breakage — it removes the main reason to rewrite `synchronized` blocks on 21-23 (`concurrency.md`) |

- Finalization is deprecated for removal; run with `--finalization=disabled` (18+) to prove nothing in your stack depends on it (`classes.md`).
- `sun.misc.Unsafe` memory-access methods are on the removal path and now emit warnings — libraries using them (some caches, serializers, and off-heap stores) need upgrades, not flags.
- Dynamically loading a Java agent at runtime warns from 21 onward; profilers and APM agents should be attached at startup with `-javaagent`.

## javax → jakarta

- Jakarta EE 9 renamed every `javax.*` EE package to `jakarta.*`. This is a hard break with no compatibility shim in the JDK: Servlet, JPA, Bean Validation, JAX-RS, JMS, Mail, Annotations.
- The JDK's own `javax.*` packages (`javax.swing`, `javax.net.ssl`, `javax.crypto`, `javax.sql`) are **not** affected — do not blanket-rename.
- Spring Boot 3 requires Jakarta and Java 17: the framework upgrade and the namespace change arrive together, so plan them as one project.
- Mechanics: update dependencies to their Jakarta coordinates first, then rewrite imports (OpenRewrite recipes or the Eclipse Transformer do this reliably; a regex over `javax\.` will corrupt the JDK packages above).
- A single dependency still on `javax` puts both namespaces on the classpath: two `Servlet` types with the same simple name, and the container binds neither. Verify with `mvn dependency:tree` (`build.md`).

## The Module System, Practically

Most applications stay on the classpath (the unnamed module) and only meet JPMS through access errors:

- `InaccessibleObjectException: module java.base does not "opens java.lang" to unnamed module` = a library reflecting into the JDK. Fix by upgrading the library; the escape hatch is `--add-opens java.base/java.lang=ALL-UNNAMED` at runtime.
- `--add-exports` for compile-time access to a non-exported package; `--add-opens` for deep reflection at runtime. `exports` is compile-time visibility, `opens` is runtime reflection — they are not interchangeable.
- Put the flags where they survive: `<argLine>` for Surefire, `JAVA_TOOL_OPTIONS` or the launcher script for runtime, `Add-Opens` in the manifest for an executable jar. A flag in the developer's IDE that is missing in production is a favourite outage.
- If you do modularize: split packages (the same package in two modules) are forbidden and are the usual blocker; `requires transitive` re-exports a dependency to your consumers; `opens pkg to spring.core` scopes reflection to the framework that needs it; `provides X with Y` replaces classpath SPI scanning.
- `jdeps --jdk-internals --multi-release 21 app.jar` lists every internal API your dependencies touch — run it before the upgrade, not during the incident.

## Testing the Upgrade

- Run the full suite on both JDKs during the transition; Maven toolchains and Gradle toolchains make a two-JDK matrix cheap (`build.md`).
- Watch for behavioral, non-crashing changes: formatted dates and numbers (CLDR), `HashMap`/`Set.of` iteration order, default charset, GC pause profile, and default heap sizing in containers.
- Pin the tzdata expectation: date tests can fail because the new JDK ships newer DST rules (`datetime.md`).
- Benchmark before and after with the same load (`performance.md`); newer JDKs are usually faster, but a changed default collector can move p99 in either direction.
- OpenRewrite and Error Prone can automate a large share of the mechanical edits; review the diff — automated migrations occasionally "fix" intentional code.
