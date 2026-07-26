# Errors — Result Design, `?`, Panics, and Reports Humans Can Read

Two decisions carry almost all the value: what the error type is at each boundary (SKILL.md rule 5), and whether a given failure is a `Result` or a panic at all.

## Result or Panic

| Failure | Kind | Why |
|---|---|---|
| Bad input from a user, file, socket, or config | `Result` | The caller has a recovery strategy; you do not know it |
| A resource is missing or busy | `Result` | Same |
| A precondition your own code guarantees is violated | Panic | There is no recovery; a `Result` here spreads meaningless matching |
| Index out of bounds, arithmetic overflow in debug, `unwrap` on a state you proved | Panic | Bug, not condition |
| Allocation failure | Abort (std default) | Not catchable in stable std |

- Library rule: never panic on data the caller supplied. Offer `try_` variants beside any panicking convenience method, exactly as std does with `get`/`[]`.
- `unwrap()` in a binary's `main` or in a test is fine and readable. `unwrap()` in a library is a defect you have shipped to someone else.
- `expect("...")` should state the invariant that was violated, not the operation: `expect("config validated at startup")` beats `expect("failed to get config")` — the first tells a reader why it should be impossible.
- `debug_assert!` for invariants too expensive to check in release; it disappears with `debug-assertions = false`, which is the release default.

## The `?` Operator

- `?` applies `From::from` to the error, which is the whole mechanism behind error conversion. `#[from]` in `thiserror` generates those impls.
- It works on `Option` too, but the function must return `Option`. Mixing: `.ok_or(MyError::Missing)?` to go `Option` → `Result`, `.ok()` to go the other way.
- In a closure, the closure itself must return the right type — `?` inside `map` will not propagate to the outer function. Use `and_then`, or `collect::<Result<Vec<_>, _>>()` which short-circuits at the first error and is the idiomatic "fallible map".
- `?` on `Box<dyn Error>` swallows the concrete type. Acceptable in `main`, not at a library boundary.
- The cost is a branch and a possible conversion; it is not exception unwinding and does not allocate unless your error type does.

## Library Errors: `thiserror`

```rust
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("config file not found at {path}")]
    NotFound { path: PathBuf },
    #[error("invalid TOML in {path}")]
    Parse { path: PathBuf, #[source] source: toml::de::Error },
    #[error(transparent)]
    Io(#[from] std::io::Error),
}
```

- `#[source]` builds the chain that `Display`-walking tools print; `#[from]` implies `#[source]` and generates the `From` impl that `?` uses.
- Only one `#[from]` per source type per enum — two variants converting from `io::Error` will not compile. Wrap one in a struct field or add context to distinguish them.
- Put the context in the message, not in the variant name: `NotFound { path }` prints something actionable, `NotFound` alone sends the user to grep your source.
- `#[error(transparent)]` forwards `Display` and `source` unchanged — right for a pure pass-through variant, wrong when you have context to add.
- Mark the enum `#[non_exhaustive]` if you may add variants later.

## Binary Errors: `anyhow`

- `anyhow::Result<T>` plus `.context("reading config")` at each layer produces a chain that reads top-down as a story: the outermost context first, the root cause last.
- `.with_context(|| format!("reading {path:?}"))` for anything that allocates — the closure only runs on the error path.
- `bail!("...")` for early return, `ensure!(cond, "...")` for a checked precondition.
- `anyhow::Error` requires the inner error to be `Send + Sync + 'static`. A `Box<dyn Error>` from elsewhere in your stack will not convert; fix the source type rather than boxing again.
- Downcasting exists (`err.downcast_ref::<io::Error>()`) but needing it in more than one place means the boundary wanted an enum.

## Making Failures Legible

- `fn main() -> Result<(), E>` prints `Error: {E:?}` and exits 1 (SKILL.md rule 6). `anyhow::Error`'s `Debug` renders the full context chain, which is why it is the right return type for `main` even when your internals use enums.
- Exit codes: the `Result` path always gives 1. For distinct codes, match in `main` and call `std::process::exit(code)` — after ensuring nothing needs its destructor, because `exit` does not unwind.
- `RUST_BACKTRACE=1` populates `std::backtrace::Backtrace` (`rust >=1.65`) and anyhow's captured backtrace. Say so in the error output of any CLI: users do not know the variable exists.
- Print the chain yourself when you own the top level:

```rust
eprintln!("error: {err}");
let mut src = err.source();
while let Some(e) = src { eprintln!("  caused by: {e}"); src = e.source(); }
```

- With `tracing`, attach context as span fields rather than string-concatenating it into messages: `#[instrument(fields(path = %path.display()))]` puts it on every event inside, including the error, and structured fields survive into log aggregation where a formatted string does not.

## Panics In Practice

- A panic in a spawned thread does not kill the process: it is delivered as `Err` from `JoinHandle::join`, and ignoring the handle silently discards it. Check the result, or install a `panic::set_hook` that logs and aborts.
- A panic while a `Mutex` is held poisons it: every later `lock()` returns `Err(PoisonError)`. `.into_inner()` on the error recovers the data if you accept it may be inconsistent. `parking_lot::Mutex` has no poisoning at all — a deliberate tradeoff, not a free upgrade.
- `panic = "abort"` in a profile shrinks the binary and removes landing pads, and it disables `catch_unwind`, breaks tests that rely on `#[should_panic]`, and makes destructors not run. Set it for release builds of a service, not for the test profile.
- `catch_unwind` is for FFI boundaries and process-level supervisors, not for control flow. It cannot catch aborts, and the caught payload is `Box<dyn Any>` which you must downcast to `&str`/`String` to read.
- Unwinding out of an `extern "C"` function aborts the process (`rust >=1.71`); use `extern "C-unwind"` when unwinding must cross the boundary deliberately.

## Retry and Partial Failure

- Distinguish retryable from terminal in the type, not at the call site: an `is_retryable(&self) -> bool` method on the error enum keeps the policy in one place.
- Backoff belongs above the error type: exponential with jitter, and a cap on total elapsed time rather than on attempt count — attempts are meaningless when each one can hang.
- Collecting many fallible results: `collect::<Result<Vec<_>, _>>()` stops at the first error; `partition` into successes and failures when the caller needs both. Choose explicitly, because the two produce very different user experiences.
