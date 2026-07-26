# Setup — Java

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

Java rewards precision and punishes assumptions: the same code behaves differently across JDK versions, locales, container limits, and build tools. Diagnose from evidence (an exception name, a thread dump, a dependency tree), give the default that works, and name the escape hatch.

## How To Load Preferences

1. Read `~/Clawic/data/java/config.yaml` if it exists. Apply its values.
2. For anything absent, detect from the repository before falling back to the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `jdk_version`: from `maven.compiler.release`, `<java.version>`, or the Gradle toolchain if present, else 21.
   - `build_tool`: `pom.xml` → maven, `build.gradle`/`build.gradle.kts` → gradle; anything else (Bazel, Ant, plain `javac`) → other; else maven.
   - `framework`: a `spring-boot-starter*` dependency → spring-boot; Quarkus, Micronaut, or a Jakarta EE container → other; else none.
   - `locale`, `timezone`, `default_charset`: not detected from the repo — read `~/Clawic/profile.yaml` if it exists, else use the table defaults.
   - `lombok: false`, `nullability_style: none`, `preview_features: false`, `test_stack: junit5`.
3. Read `~/Clawic/data/java/memory.md` for prior context (their stack, recurring pain points). Absence is fine; proceed without comment.

Work from defaults immediately. Never open with questions about their setup, priorities, or how proactive to be.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names a JDK target, build tool, framework, test stack, nullability annotation flavour, a Lombok/preview stance, or a display locale, time zone, or file charset ("we are UTC-only", "display in es-ES") → update the matching key in `~/Clawic/data/java/config.yaml`.
- User expresses a habit or stance (formatter, immutability default, checked-exception policy, banned APIs, GC choice, how much explanation they want, whether to propose edits before applying them, how often to rebuild for CVEs, how proactively to flag legacy patterns) → record it under the relevant preference area (tooling, conventions, platform, output, work order, cadence, safety posture, restrictions) in `~/Clawic/data/java/memory.md`.
- User corrects earlier guidance → update the stored value so it is not repeated.

If the user has said nothing, store nothing.

## What Memory Holds

See `memory-template.md` for the file format. Track their stack (framework, persistence, deployment target), recurring problems (dependency conflicts, OOMs, flaky tests), and the depth of explanation they want (the `output` preference area in `SKILL.md`) — but only from what they actually reveal.
