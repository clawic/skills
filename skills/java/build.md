# Build — Maven, Gradle, Dependency Conflicts, Packaging

Most "Java bugs" filed against application code are build problems: the wrong jar won, a resource was not packaged, or the compiler targeted a JDK the runtime does not have.

## Version Resolution: the Two Rules That Differ

- **Maven: nearest wins.** The declaration at the shortest depth in the tree; ties break by declaration order in the POM, NOT by version. A direct `1.2` beats a transitive `2.0` — even when `2.0` is what another library requires.
- **Gradle: highest wins.** The maximum version among all requests, unless a constraint or `strictly` says otherwise.
- Consequence: the same dependency graph produces different jars in the two tools, so a Maven-verified library set is not evidence for a Gradle build (SKILL.md rule 7).
- The single place to fix the version:
  - Maven: `<dependencyManagement>` — it overrides the resolved version at **any** depth, unlike a direct dependency.
  - Gradle: a `platform()`/BOM dependency, or `constraints { implementation("g:a:1.2") }`.

```bash
mvn dependency:tree -Dverbose -Dincludes=com.fasterxml.jackson.core:jackson-databind
mvn dependency:analyze              # declared-but-unused, used-but-undeclared
./gradlew dependencyInsight --dependency jackson-databind --configuration runtimeClasspath
./gradlew :app:dependencies --configuration runtimeClasspath
```

- `used-but-undeclared` is the ticking one: your code compiles against a transitive jar, and the day the intermediate library drops it, your build breaks for no local reason. Declare what you import.
- The Maven Enforcer plugin's `dependencyConvergence` rule fails the build when two versions of the same artifact are requested — noisy at first, then the cheapest guard against `NoSuchMethodError` (SKILL.md Exception Triage).

## Scopes and Configurations

| Maven scope | Gradle configuration | On compile classpath | On runtime classpath | Leaks to consumers |
|---|---|---|---|---|
| `compile` (default) | `implementation` | yes | yes | Maven: yes · Gradle: **no** |
| — | `api` | yes | yes | yes |
| `provided` | `compileOnly` | yes | no | no |
| `runtime` | `runtimeOnly` | no | yes | yes |
| `test` | `testImplementation` | test only | test only | no |

- Gradle's `implementation` vs `api` is the real difference from Maven: `implementation` keeps a dependency off consumers' compile classpath, which speeds up builds and prevents accidental coupling. Use `api` only for types that appear in your public signatures.
- `provided`/`compileOnly` for anything the container supplies (servlet API, a driver installed in the app server). Shipping your own copy alongside the container's causes the `ClassCastException`-with-the-same-class-name symptom (SKILL.md Exception Triage).
- A `runtimeOnly` JDBC driver keeps your code honest: you cannot accidentally import a vendor class.

## Compiler Settings That Matter

```xml
<properties>
  <maven.compiler.release>21</maven.compiler.release>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
```

- `release` over `source`/`target` (SKILL.md rule 8). Gradle: `java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }`, which also downloads/uses the right JDK instead of whatever launched the build.
- Set the source encoding explicitly, or literals are compiled with the platform charset and mangle on another machine (`text.md`).
- `-parameters` keeps parameter names in the bytecode — required by Jackson constructor binding and several DI frameworks (`serialization.md`).
- `-Xlint:all` and, where the team can sustain it, `-Werror`: unchecked and deprecation warnings are the early form of a runtime failure (`generics.md`).
- Reproducibility: set `<project.build.outputTimestamp>` (Maven) or `isPreserveFileTimestamps = false` + `isReproducibleFileOrder = true` (Gradle jar task), so the same source produces byte-identical artifacts.

## Packaging

- **Fat/shaded jar** (maven-shade-plugin, Gradle Shadow): merges everything into one archive.
  - Add the `ServicesResourceTransformer` or `META-INF/services` entries from different jars overwrite each other — the cause of "No suitable driver found" and missing SPI implementations that only appears after packaging (`debug.md`).
  - Add the `ManifestResourceTransformer` with `mainClass`, or `java -jar` reports "no main manifest attribute".
  - Exclude signature files (`META-INF/*.SF`, `*.DSA`, `*.RSA`) from dependencies, or the JVM refuses to start with "Invalid signature file digest".
  - Relocate (shade) a dependency only when you must isolate a version conflict from consumers; it breaks reflection-by-name and resource paths for that library.
- **Spring Boot repackage** produces a nested-jar layout with its own classloader, NOT a shaded jar: dependency jars stay intact inside `BOOT-INF/lib`. That is why resource URLs are `jar:file:...!/BOOT-INF/...` and cannot be read as files (`io.md`).
- **jlink/jpackage** build a runtime image with only the modules you need — smaller containers, but requires the module graph to be complete (`migration.md`).
- Layered container images: put dependencies in a lower layer than application classes, so a code change rebuilds a few MB instead of the whole jar.

## Multi-Module

- Build one module and what it needs: `mvn -pl service -am install`; Gradle resolves this automatically from the task path.
- A module version conflict between siblings is the same nearest/highest rule — pin in the parent's `dependencyManagement` or the root `platform`.
- Keep the reactor order implicit through dependencies, never through the `<modules>` order; the order in the file is not a build order.
- `mvn -o` (offline) and `--refresh-dependencies` (Gradle) are the two switches for "is this a network/cache problem?".

## When the Build Is the Bug

| Symptom | Cause | Fix |
|---|---|---|
| Works in IDE, fails from the jar | Resource read as a file, or a missing `META-INF` merge | `io.md`, and the shade transformers above |
| `NoSuchMethodError` at runtime | Two versions of one library; the wrong one won | `dependency:tree -Dverbose`, pin it |
| Compiles locally, fails in CI | Different JDK, or a `SNAPSHOT`/range resolving differently today | Toolchains + pinned versions |
| Stale results after a change | Incremental compilation state | `mvn clean`, `./gradlew --rerun-tasks`, or delete `~/.m2/repository/.../*.lastUpdated` for a half-downloaded artifact |
| Tests do not run at all | JUnit 5 with an old Surefire, or the vintage engine missing | Surefire ≥ 2.22, `useJUnitPlatform()` in Gradle (`testing.md`) |
| Random `OutOfMemoryError` in the build | The build JVM's own heap, not the app's | `MAVEN_OPTS` / `org.gradle.jvmargs` |
| Anything else | Verify what is actually on the classpath | `mvn dependency:build-classpath`, `./gradlew dependencies`, or `jar tf app.jar` |

- Dependency hygiene as a habit: pin every version in one place, run a vulnerability scan in CI (OWASP Dependency-Check, `gradle dependencyCheckAnalyze`, or the platform's own), and rebuild on a fixed schedule even without code changes — CVE fixes arrive as new versions (`security.md`). The interval is the user's: record it under the cadence preference area (SKILL.md Configuration).
