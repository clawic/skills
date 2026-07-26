# Observability — `tracing`, Logging, Metrics, and Seeing Inside a Running Binary

A Rust service that panics tells you nothing by default: no logger is installed, the panic goes to stderr with no context, and the async task that failed does not name the request it was serving. Observability is setup work you do once, at the binary, and never in a library.

## `log` vs `tracing`

| Axis | `log` | `tracing` |
|---|---|---|
| Unit | An event (one line) | Spans (a period of time) plus events inside them |
| Structure | Formatted string | Typed fields attached to the span and the event |
| Async | Nothing ties lines from the same task together | A span follows the future across `.await` points |
| Ecosystem | The oldest facade; most crates emit through it | The default for services; consumes `log` output via `tracing-log` |

Default: `tracing` in binaries and in libraries that do meaningful work; plain `log` only when the dependency budget is tiny. Either way the split is the same — **a library depends on the facade and never installs a subscriber**; installing one from a library steals the choice from the application and produces double output.

## Subscriber Setup (the binary's job, once)

```rust
tracing_subscriber::fmt()
    .with_env_filter(EnvFilter::try_from_default_env().unwrap_or_else(|_| "info,my_crate=debug".into()))
    .with_target(true)
    .json()                       // drop for human-readable local output
    .init();
```

- `EnvFilter` reads `RUST_LOG` and takes per-target directives: `RUST_LOG=warn,my_crate::db=trace`. Ship a sane default in the code so an unset variable is not silence.
- JSON in production, the pretty formatter locally. Decide by environment variable, not by `#[cfg]`, so a production-shaped run is reproducible on a laptop.
- Add layers, do not replace: `tracing_subscriber::registry().with(fmt_layer).with(otel_layer).with(ErrorLayer::default())`.
- `try_init()` in tests and in libraries' own test harnesses — `init()` panics if a subscriber already exists, which makes the second test fail for the wrong reason.

## Spans and Fields

- `#[instrument]` on a function creates a span around its body with the arguments as fields. Control what that costs: `skip(self)` or `skip_all` plus explicit `fields(user_id = %id)`, because the default records **every** argument with `Debug` — a large struct on a hot path, or a secret you did not think about.
- Sigils: `%value` records `Display`, `?value` records `Debug`, bare `value` requires the type to be a `tracing::Value`.
- `#[instrument(err)]` logs the `Err` return at error level; `ret` logs the `Ok`. Cheap, and it removes most hand-written "failed to X" lines.
- Fields added later: `tracing::Span::current().record("rows", count)` — the field must be declared in the span (`fields(rows = tracing::field::Empty)`) before it can be recorded.
- **Never hold `span.enter()` across an `.await`.** The guard is thread-local; when the task is parked and resumed on another worker, the span is attached to the wrong thing. Use `future.instrument(span).await`, or `#[instrument]` on the `async fn`, which does this correctly.
- Attach context as fields rather than string-concatenating it into the message: fields survive into aggregation and can be queried, a formatted string cannot.

## Levels and Cost

- `error` for "a human must act", `warn` for degraded-but-handled, `info` for lifecycle and one line per request, `debug` for developer detail, `trace` for firehose. A service logging `info` per inner loop iteration is a service whose logs nobody reads.
- A disabled event is a level comparison and a branch, but the **arguments are still evaluated** if you compute them yourself. Pass values, not `format!` results.
- Compile them out entirely where it matters: the `release_max_level_info` feature of `tracing`/`log` removes lower levels at compile time, so the branch disappears in firmware and hot paths.
- Sampling belongs in the exporter, not in your code: trace every request into the pipeline and let the collector decide, or you lose exactly the rare slow ones.

## Panics, Errors, and Crashes

- Install a panic hook so panics reach the same sink as everything else:

```rust
std::panic::set_hook(Box::new(|info| {
    tracing::error!(panic = %info, backtrace = %std::backtrace::Backtrace::force_capture(), "panic");
}));
```

- `RUST_BACKTRACE=1` must be set in the environment for a captured backtrace to be populated; set it in the container image for services, and say so in the deployment notes.
- A panic in a spawned task does not kill the process — the task dies and the request hangs. Log the `JoinError` where you join, or run a supervisor that restarts and counts.
- For a service, `panic = "abort"` plus a restart policy makes crashes visible and consistent; unwinding keeps the process alive in a half-known state. Choose deliberately and write down which one you chose.

## Metrics and Traces

- Three signals, three uses: **logs** answer "what happened in this one request", **metrics** answer "how often, how slow, right now", **traces** answer "where did the time go across services". Logs alone force you to reconstruct the other two by grep.
- `metrics` (facade) with a Prometheus or OTLP exporter mirrors the `log`/`tracing` split: counters for events, histograms for latency, gauges only for things that genuinely go up and down.
- Cardinality is the failure mode: a label carrying a user id, a request id, or a raw path creates one time series per value and takes down the metrics backend. Bucket the path (`/users/:id`), never the id.
- `tracing-opentelemetry` bridges spans to OTLP so the same `#[instrument]` annotations feed distributed traces; propagate the trace context in headers at every service boundary or the trace ends at your door.

## What to Instrument

| Situation | Instrument |
|---|---|
| HTTP or RPC handler | One span per request with method, route, status, latency; the request id as a field on the span |
| Background job | A span per job with the job kind and its outcome; a counter for retries |
| Database or external call | A span with the operation name (never the interpolated query text) and a duration histogram |
| Async task that may stall | `tokio-console` in development; a task-count gauge in production |
| A panic or a poisoned lock | An error event plus a counter — silence here is how a degraded process runs for a week |
| Anything else | One `info` at the boundary, `debug` inside; add fields before adding lines |

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| A library calling `tracing_subscriber::init()` | Steals the application's choice; double output when the app also inits | Emit through the facade only |
| `span.enter()` held across `.await` | Thread-local guard follows the wrong task after a park | `.instrument(span).await` |
| `#[instrument]` on a function taking a big struct or a secret | Every call `Debug`-formats it into the log | `skip_all` plus explicit fields |
| `println!`/`dbg!` left in a service | Unstructured, unfiltered, and invisible to the aggregator | `tracing::debug!` behind a level |
| A user id as a metric label | Unbounded cardinality kills the metrics backend | Put identity in span fields, not metric labels |
| `info!("{:?}", huge)` on a hot path | Formats even when nothing consumes it downstream | Pass fields; let the subscriber decide |
| No panic hook | Panics vanish from the log pipeline entirely | `set_hook` at startup |
