# Testing — Layout, Doctests, Properties, Miri, and Fuzzing

Rust's test harness is fast and has three defaults that surprise people: tests run in parallel threads inside **one** process, `stdout` is captured unless a test fails, and the working directory is the crate root, not the workspace root.

## Where Tests Live

| Kind | Location | Sees | Use for |
|---|---|---|---|
| Unit | `#[cfg(test)] mod tests` in the source file | Private items | Logic that has no public surface |
| Integration | `tests/*.rs`, one binary per file | Only the public API | Contract of the crate as a consumer sees it |
| Doc | ` ``` ` blocks in doc comments | Public API | Examples that must not rot |
| Benchmark | `benches/` with criterion | Public API | Measuring a change against a baseline |
| Compile-fail | `trybuild` in `tests/` | Macro and API errors | Proving a bad call is rejected |

- `#[cfg(test)]` code is not compiled into the release artifact, but `[dev-dependencies]` still affect feature unification for the whole build.
- Each file in `tests/` links the crate separately: 20 integration files means 20 link steps. Consolidating into one file with `mod` submodules is a real build-time win on large crates.
- Shared helpers go in `tests/common/mod.rs` — a bare `tests/common.rs` would be compiled as its own test binary.

## Assertions and Test Shape

- `assert_eq!` prints both values; `assert!(a == b)` prints nothing useful. The same applies to `assert_ne!` and to custom messages: `assert!(cond, "expected {x} to be under the limit {limit}")`.
- `#[should_panic(expected = "substring")]` matches a **substring** of the panic message. Bare `#[should_panic]` passes on any panic, including one from a typo in the test, so always give the expected text.
- Tests may return `Result`: `fn t() -> anyhow::Result<()>` lets you use `?` instead of a wall of `unwrap`. An `Err` fails the test with the `Debug` output.
- `#[ignore]` for slow tests, run with `--ignored`. Put them in CI on a schedule rather than deleting them.
- Table-driven tests read better than a dozen near-identical functions, but a failure reports one line number for all cases — include the case identity in the assertion message, or use `rstest` cases which report separately.

## Parallelism and Isolation

- Tests share one process: `std::env::set_var`, `set_current_dir`, global loggers, and process-wide singletons are shared mutable state across threads. Setting an environment variable in one test changes another test's behavior and the failure looks random.
- Fixes, in order: keep the state out of the test (pass config as an argument), use `tempfile::TempDir` per test rather than a fixed path, serialize the few tests that must be serial (`serial_test`), and only then reach for `--test-threads=1`, which slows the whole suite to fix two tests.
- `cargo nextest` runs each test in its **own process**, which removes most of this class and gives per-test timeouts and clean output. It does not run doctests — keep `cargo test --doc` as a separate CI step.
- Ports: binding a fixed port makes tests conflict; bind port 0 and read the assigned port back.
- Time: never `sleep` to wait for a condition. Poll with a deadline, or inject a clock.

## Doctests

- Every ` ``` ` block in a doc comment compiles and runs, with the crate implicitly imported. They are your only compiler-verified documentation.
- Hide setup with a leading `#` on the line: it runs but is not shown to readers.
- Annotations: `no_run` compiles only, `ignore` does neither (and is almost always a doctest quietly rotting), `should_panic`, `compile_fail` for showing what the type system rejects.
- Doctests are slow — one binary per block. In a crate with hundreds, mark the redundant ones `no_run` rather than deleting the examples.
- `#![doc = include_str!("../README.md")]` makes the README's examples doctests, which is how you keep the front page from lying.

## Property and Snapshot Testing

- `proptest`/`quickcheck` generate inputs and shrink failures to a minimal case. Highest value on parsers, serializers, encoders, and any function with an inverse: `parse(render(x)) == x` finds bugs that example tests structurally cannot.
- Write the invariant, not the expected output: "the result is sorted and a permutation of the input" beats fifty hand-written arrays.
- Record the failing seed and add it as a normal unit test when a property fails; proptest persists regressions in `proptest-regressions/`, which belongs in version control.
- `insta` snapshot tests fit output nobody wants to write by hand (rendered text, JSON, error messages). The discipline that makes them useful: review every diff, never `cargo insta accept` in bulk.

## Miri and Sanitizers

- `cargo +nightly miri test` interprets your code and detects out-of-bounds, use-after-free, invalid alignment, data races, and aliasing violations under Stacked/Tree Borrows. It is the only practical correctness check for `unsafe`.
- Cost: commonly one to two orders of magnitude slower than native, so run it on a subset (`cargo miri test -- unsafe_`), on a schedule, not on every PR.
- Miri cannot execute FFI calls or most syscalls. Structure `unsafe` code so the pure-Rust logic is testable without the foreign call.
- For code Miri cannot reach: `-Zsanitizer=address` and `-Zsanitizer=thread` on nightly, which do work across FFI.

## Fuzzing

- `cargo fuzz` (libFuzzer) or `afl.rs` for any parser or decoder that touches untrusted input. A fuzz target is five lines around `fuzz_target!(|data: &[u8]| { let _ = my_parse(data); });`.
- `arbitrary` derives structured input so the fuzzer explores your type space rather than random bytes.
- Every crash goes into the corpus and becomes a regression test — that conversion is what makes fuzzing pay off beyond the first week.
- Run it where it can run for hours (nightly job, OSS-Fuzz); a two-minute fuzz in a PR check finds nothing and costs everyone time.

## Coverage and What It Means

- `cargo llvm-cov` is the current standard (`cargo tarpaulin` on Linux as an alternative). Both work with nextest.
- Line coverage in Rust overstates safety: exhaustive `match` and the type system already eliminate whole classes of untested state. Use coverage to find files nobody tests, not as a percentage target.
- Uncovered `Err` branches are the honest finding — error paths are where untested code hides.

## Test Failure Triage

| Symptom | First check |
|---|---|
| Passes alone, fails in the suite | Shared process state: env vars, cwd, global logger, fixed port or path |
| Passes locally, fails in CI | Toolchain or feature drift, or a timing assumption on a slower machine |
| Flaky | Real concurrency bug until proven otherwise — loop it under `nextest` before blaming the runner |
| Fails only in release | Overflow checks and `debug_assert!` are off (SKILL.md rule 3); or UB that debug happened to tolerate |
| Fails only under Miri | Real UB. Miri false positives exist but are rare and specific |
| Anything else | `cargo test -- --nocapture --test-threads=1` to get readable, ordered output before theorizing |
