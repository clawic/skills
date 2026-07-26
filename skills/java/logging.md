# Logging — Facades, Bindings, and Output You Can Debug With

Java's logging ecosystem is four competing APIs plus bridges between them. Nearly every logging problem is a wiring problem, not a code problem.

## The Stack, in One Picture

- **Facade** (what your code imports): SLF4J. Nothing else belongs in application or library code.
- **Implementation** (what actually writes): Logback (SLF4J's native backend, Spring Boot's default) or Log4j2. Exactly ONE on the classpath.
- **Bridges** (redirect other APIs into your facade): `jcl-over-slf4j` (Commons Logging), `log4j-over-slf4j` or `log4j-to-slf4j`, `jul-to-slf4j` (java.util.logging).
- Rule: one facade, one implementation, bridges for everything else, and the ORIGINAL implementations excluded. A bridge plus its original implementation on the same classpath is an infinite loop or a stack overflow at first log.

| Symptom | Cause | Fix |
|---|---|---|
| `SLF4J: Class path contains multiple SLF4J bindings` | Two backends; SLF4J picks one arbitrarily, so config changes appear to do nothing | Exclude all but one (`mvn dependency:tree`, `build.md`) |
| `SLF4J: Failed to load class ...StaticLoggerBinder` / `NOP` provider | Facade with no backend — every log line is discarded silently | Add `logback-classic` or `log4j-slf4j2-impl` |
| Some libraries log, others do not | The quiet ones use JUL or Commons Logging with no bridge | Add the bridge; for JUL also install `SLF4JBridgeHandler` |
| Log level changes have no effect | Two config files (`logback.xml` and `logback-spring.xml`, or a jar shipping its own), or the wrong backend won | Check which file loaded: `-Dlogback.debug=true` / `-Dlog4j2.debug` |
| Duplicate lines | A logger with `additivity` on plus its own appender, or two backends both writing | Set `additivity="false"` on the specific logger |

## Writing Log Statements

- Parameterized, always: `log.debug("order {} for {}", id, customer)`. String concatenation builds the message even when the level is off.
- The trailing throwable needs no placeholder: `log.error("saving order {}", id, e)` logs both the id and the stack (SKILL.md Traps).
- Guard only genuinely expensive arguments: `if (log.isDebugEnabled()) log.debug("{}", expensiveDump())`, or pass a `Supplier` with Log4j2's API.
- Levels with a decision attached: ERROR = a human must act tonight; WARN = degraded but handled; INFO = a business event a reader would want; DEBUG = developer detail; TRACE = firehose. An ERROR per retry attempt trains everyone to ignore ERROR (`exceptions.md`).
- Log once per failure, at the boundary that decides what to do — catch-log-rethrow at every layer produces four copies of one incident.
- Never log secrets, tokens, full request bodies, or PII. Redact in the appender/pattern, not at each call site, so a new call site cannot leak (`security.md`).
- User-controlled text with newlines forges log entries; encode or strip control characters before logging untrusted input.

## Structure and Correlation

- Structured JSON output (logstash-logback-encoder, Log4j2's `JsonTemplateLayout`) makes fields queryable. The value appears the day you need "all errors for customer X", not before.
- MDC carries the correlation/trace id for the whole request: `MDC.put("traceId", id)` in a filter, `MDC.clear()` in `finally`. Without the `finally`, a pooled thread carries a stale id into the next request.
- MDC is thread-local: it does NOT cross an `@Async` boundary, a `CompletableFuture` stage, or a thread pool hop unless you propagate it explicitly (task decorators exist for this) (`async.md`).
- One event per line. Multi-line output breaks every log aggregator except for stack traces, which every aggregator special-cases.
- Include the identifiers you would grep for: entity id, user id, operation, outcome, duration. "Failed to process" costs the same as "failed to process order=123 stage=payment".

## Configuration and Operations

- Asynchronous appenders (Logback `AsyncAppender`, Log4j2's async loggers) remove I/O from the request path — with a bounded queue. The default discards or blocks when full: know which, because "logging blocked the app" and "we lost the logs for the incident" are different outages.
- Rolling policies need BOTH a size/time trigger and a total cap (`maxHistory`, `totalSizeCap`), or logs fill the disk during the very incident you need them for.
- In containers, log to stdout and let the platform collect; a file inside a container is lost when it restarts (`docker` skill for the collection side).
- Runtime level changes: Spring Boot's `loggers` actuator endpoint or a JMX/`Configurator` call — the ability to enable DEBUG for one package for five minutes in production is worth more than any amount of extra INFO logging. Keep that endpoint off the public port (`spring.md`).
- Cost is real: a chatty INFO line in a hot path is allocation plus I/O plus storage plus index. Measure the top talkers before adding more (`performance.md`).

## Testing Logs

- Assert on behavior, not on log text, except where the log line IS the deliverable (audit trails).
- When you must: attach a `ListAppender` (Logback) to the logger under test and assert the events — string matching against captured stdout is brittle (`testing.md`).
- Keep test output quiet with a test-scoped configuration; a suite that prints 50,000 lines hides the one failure message that matters.
