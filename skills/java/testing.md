# Testing — JUnit 5, Mockito, Testcontainers, and Flakiness

## Wiring That Silently Fails

- **JUnit 4 annotations in a JUnit 5 project are not executed.** `org.junit.Test` is invisible to the Jupiter engine: the build stays green while running zero tests (SKILL.md Traps). Grep for `import org.junit.Test;` after any migration.
- Maven needs Surefire ≥ 2.22 to discover the JUnit Platform; older versions run nothing and report success. Gradle needs `test { useJUnitPlatform() }`.
- Assert that tests actually ran: fail the build below a minimum test count, or check the Surefire/Gradle report's total. A test count that drops by 300 in a diff is the cheapest regression detector there is.
- Test classes and methods no longer need to be `public` in JUnit 5, but they must not be `private` or `static` (except `@BeforeAll`/`@AfterAll`).
- Mixing engines is legal and explicit: add `junit-vintage-engine` when JUnit 4 tests must keep running during a migration (`build.md`).

## JUnit 5 Essentials

| Need | Use |
|---|---|
| Expect an exception | `var e = assertThrows(X.class, () -> code()); assertEquals("...", e.getMessage())` |
| Multiple independent assertions | `assertAll(() -> ..., () -> ...)` — reports every failure, not just the first |
| Same test, many inputs | `@ParameterizedTest` + `@ValueSource` / `@CsvSource` / `@MethodSource` (a static supplier) / `@EnumSource` |
| Group related tests, shared setup | `@Nested` inner class — inherits the outer `@BeforeEach` |
| Temp directory | `@TempDir Path dir` — created and deleted per test |
| Conditional skip | `@EnabledOnOs`, `@EnabledIfEnvironmentVariable`, `Assumptions.assumeTrue(...)` |
| Custom lifecycle/injection | An `Extension` (`@ExtendWith`) — the replacement for runners and rules |
| Timeout | `@Timeout(5)` — but a timeout on a hang is a symptom, not a fix (`debug.md`) |

- `@BeforeAll`/`@AfterAll` must be static unless the class is `@TestInstance(PER_CLASS)`. Per-class lifecycle also means state leaks between tests unless you reset it.
- `@Disabled` (not JUnit 4's `@Ignore`) and always with a reason string; a disabled test with no reason is deleted code with extra tokens.
- `assertEquals(expected, actual)` — the order only affects the failure message, and the failure message is the whole value of the assertion.
- Prefer AssertJ (`assertThat(list).extracting(Foo::name).containsExactly("a", "b")`) for anything beyond scalar equality: the failure output shows the actual collection instead of "expected true".
- Assert on values, never on log output or on the absence of an exception. A test whose only assertion is "it did not throw" passes when the method does nothing.

## Mockito Without the Traps

- Mockito 5 defaults to the inline mock maker and requires Java 11+: final classes and static methods are mockable without an extra artifact — which does not mean they should be.
- `when(spy.method())` **calls the real method** while stubbing. On a `@Spy`, always use `doReturn(x).when(spy).method()`.
- Stubbing a method that is never called fails the strict stubs check (default in JUnit 5's `MockitoExtension`) — that is a feature: it finds tests asserting against a path the code no longer takes. `lenient()` is an escape hatch to justify, not a default.
- `verify(mock).save(any())` after an action; `verifyNoMoreInteractions` sparingly, since it makes the test fail on unrelated refactors.
- `ArgumentCaptor` when you must assert on the shape of what was passed; `ArgumentMatchers.argThat` for a predicate.
- Mixing raw values and matchers in one call throws `InvalidUseOfMatchersException` — if any argument uses a matcher, all must (`eq("x")`).
- `@InjectMocks` silently leaves a field null when no matching mock exists, producing an NPE inside the test rather than a wiring error. Constructor injection in production code makes the test constructor explicit and removes the magic.
- `reset(mock)` mid-test means the test is doing two things. Split it.
- Do not mock types you do not own (HTTP clients, JDBC, the JDK). You end up asserting your assumptions about the library rather than its behavior — use a fake server or a real container.

## Integration Tests

- Testcontainers gives a real Postgres/Kafka/Redis per test run. A real engine catches SQL dialect issues, constraint violations, and transaction semantics that an in-memory H2 quietly accepts (SKILL.md, Where Experts Disagree).
- Reuse the container across the whole suite (a static field or the singleton pattern) — starting one per class is the usual reason an integration suite takes 20 minutes.
- Isolate by data, not by restart: unique ids per test, or a transaction rolled back at the end. Truncating shared tables makes parallel execution impossible.
- WireMock/MockWebServer for outbound HTTP: assert the request your code SENDS, which a mocked client cannot verify.
- Keep unit and integration tests in separate executions (Surefire vs Failsafe, or a Gradle test suite) so a slow suite does not gate the fast feedback loop.

## Flaky Tests

| Cause | Symptom | Fix |
|---|---|---|
| Shared static state | Passes alone, fails in the class; order-dependent | Reset in `@BeforeEach`, or remove the static |
| Real clock | Fails at midnight, month end, or DST | Inject a `Clock` (`datetime.md`) |
| Default time zone / locale | Passes locally, fails in UTC CI | Pin them in the test, or assert on values not formatted text (`text.md`) |
| `Thread.sleep` for async | Fails on a loaded CI runner | Await a condition with a timeout (Awaitility or a polling loop) |
| Real network or DNS | Random failures unrelated to the change | Fake the boundary |
| Iteration-order assumption | Fails on a JVM restart with `Set.of`/`Map.of` | Assert order-insensitively, or use a sorted/linked collection (`collections.md`) |
| Test parallelism | Only fails when the suite runs parallel | Isolate the state, or mark the class non-parallel |
| Random data | Fails once every hundred runs | Log and pin the seed so failures are reproducible |

- Quarantine, do not `@Disabled` and forget: a flaky test that nobody owns becomes a test suite nobody trusts.
- Reproduce a flake by running the single test hundreds of times (`--tests` with a loop, or JUnit's `@RepeatedTest`) before believing it is fixed.

## What Is Worth Testing

- Behavior at boundaries: empty, one, many, null, max value, duplicate key, concurrent access.
- Every bug fix gets the failing test FIRST. A test you never saw red does not prove your fix works.
- Round-trip properties (serialize → deserialize, encode → decode, save → load) catch whole categories in one assertion (`serialization.md`).
- Coverage is a map of what is untested, not a quality score. 100% line coverage with no assertions is achievable and worthless; use it to find the untouched branch, not to hit a number.
- Concurrency needs its own approach — a repeated multi-threaded invariant check, not a single happy-path run (`concurrency.md`).
