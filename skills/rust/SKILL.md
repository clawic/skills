---
name: Rust
slug: rust
version: 1.0.3
description: >-
  Writes, debugs, and optimizes Rust code, crates, and Cargo builds: ownership, lifetimes, traits, async, unsafe, and FFI.
  Use when the borrow checker rejects code, when rustc reports value moved, cannot borrow as mutable, does not live long
  enough, missing lifetime specifier, trait bound not satisfied, or not dyn compatible; when a future cannot be sent between
  threads, a Mutex guard crosses an await, a task blocks the async runtime, or select! loses data; when code deadlocks, panics
  on unwrap, or hits BorrowMutError; when cargo builds are slow, features unify unexpectedly, two versions of one crate
  collide, or the build fails only in CI; when writing unsafe, C FFI, proc macros, serde derives, no_std firmware, or wasm;
  when cross-compiling to musl or another target; when profiling, benchmarking, or shrinking a Rust binary. Not for C++ or
  language-agnostic concurrency theory.
homepage: https://clawic.com/skills/rust
changelog: "Display name shown correctly"
metadata:
  clawdbot:
    emoji: 🦀
    requires:
      bins:
      - rustc
      - cargo
    os:
    - linux
    - darwin
    - win32
    displayName: Rust
    configPaths:
    - ~/Clawic/data/rust/
---

User preferences and memory live in `~/Clawic/data/rust/` (see `setup.md` on first use, `memory-template.md` for the file format). If you have data at an old location (`~/rust/` or `~/clawic/rust/`), move it to `~/Clawic/data/rust/`.

## When To Use

- Writing or reviewing Rust: functions, traits, error types, async code, public crate APIs
- Fighting the compiler: moves, borrows, lifetimes, trait bounds, `dyn` compatibility, type inference
- Diagnosing runtime failures: panics, deadlocks, `BorrowMutError`, memory growth, segfaults through `unsafe` or FFI
- Cargo work: features, workspaces, lockfiles, MSRV, slow builds, CI-only failures, duplicated dependencies
- Targets beyond the host: cross-compilation, static musl binaries, `no_std` firmware, wasm
- Not for C++ (`cpp`) or language-agnostic concurrency theory (`async-patterns`) — this is Rust the language and its toolchain

## Quick Reference

| Situation | Play |
|---|---|
| Compiler says a value moved and you still need it | Borrow instead of consuming, or copy the small part you need out first — clone is rung 5, not rung 1 (→ Borrow Checker Escape Ladder) |
| Two mutable borrows refused on code that looks fine | The checker is field-sensitive on direct field access and field-insensitive through a method call — destructure or `split_at_mut` → `ownership.md` |
| A returned reference "does not live long enough" | Nothing borrowed from a frame outlives it: return owned data, or an index → `lifetimes.md` |
| `missing lifetime specifier` on a signature | Two input references and no elision rule applies — name which one the output borrows from → `lifetimes.md` |
| "trait bound not satisfied" on a type that clearly implements it | Missing `use` for the trait, or `&T` supplied where `T` was required — read the `required by` chain bottom-up → `traits.md` |
| "future cannot be sent between threads safely" | A `!Send` value (`std` `MutexGuard`, `Rc`, `RefCell` ref) is alive across an `.await` → `async.md` |
| Async program stalls with no error and no progress | A blocking call is occupying a runtime worker → `async.md` |
| Program deadlocks | One thread locks the same lock twice, or two threads take two locks in opposite order → `concurrency.md` |
| `already borrowed: BorrowMutError` at runtime | Overlapping `RefCell` borrows — the check you moved from compile time to runtime → `ownership.md` |
| `String` vs `&str`, a panic on a string slice, UTF-8, or path handling | Byte indices are not characters and paths are not UTF-8 → `strings-and-text.md` |
| A `match` is unreadable, non-exhaustive, or the type allows a state you never want | Model it as an enum, then let exhaustiveness find the sites → `pattern-matching.md` |
| Nothing in the logs explains what the service was doing | No subscriber, or context in strings instead of span fields → `observability.md` |
| "Where does this type go, and what should be public" | Module tree, `pub(crate)`, re-exports, lib vs bin split → `modules-and-layout.md` |
| Error names two identical-looking types (`expected Uri, found Uri`) | Semver-incompatible duplicate of one crate in the graph: `cargo tree -d` → `cargo.md` |
| Builds locally, fails in CI | Toolchain, lockfile, or feature-unification drift → `cargo.md` |
| Build is slow | `cargo build --timings` first; linker and codegen settings before restructuring code → `cargo.md` |
| Program is slow | Profile a release build with symbols; allocation and `clone` before algorithms → `performance.md` |
| Segfault, corrupted data, or nondeterministic "works on my machine" | Only reachable through `unsafe` or FFI — run it under Miri → `unsafe.md` |
| Anything else | Read the whole rustc output bottom-up including `note:` and `help:`, then `rustc --explain E0xxx` → `compiler-errors.md` |

Depth on demand: `compiler-errors.md` error code → real cause → first move · `ownership.md` moves, borrows, interior mutability · `lifetimes.md` elision, variance, `'static`, HRTB · `traits.md` coherence, generics vs `dyn`, associated types · `pattern-matching.md` `match` ergonomics, `let else`, exhaustiveness, `Option`/`Result` combinators, modeling states · `errors.md` `Result` design, `?`, panics, backtraces · `async.md` runtimes, `Send` bounds, cancellation · `concurrency.md` threads, locks, channels, atomics · `strings-and-text.md` `String`/`&str`, UTF-8, formatting, `Cow`, paths · `collections.md` `Vec`, maps, sets, queues, iterators · `performance.md` profiling, allocation, build settings · `observability.md` `tracing`, spans, metrics, panic hooks · `cargo.md` features, workspaces, lockfiles, build times · `modules-and-layout.md` module tree, visibility, lib/bin/tests layout, when to split a crate · `testing.md` layout, doctests, property tests, Miri, fuzzing · `unsafe.md` UB rules, raw pointers, safety comments · `ffi.md` C interop in both directions · `macros.md` `macro_rules!` and proc macros · `api-design.md` public APIs and semver hazards · `serde.md` derive traps and format edge cases · `embedded-no-std.md` firmware and `no_std` · `wasm.md` browser and WASI targets · `cross-compilation.md` target triples, musl, linkers, containers.

## Core Rules

1. **Fix a borrow error at the lowest rung that compiles.** The ladder (full version below) runs shorten → split → take → copy → clone → interior mutability → arena → `unsafe`. Every rung down trades a compile-time guarantee for a runtime one; reaching for `Rc<RefCell<T>>` against a rung-1 problem is the most common finding in Rust review.
2. **Borrowed in arguments, owned in returns.** A parameter typed `String` forces every caller to allocate or surrender ownership; `&str` accepts `String`, `&String`, and literals through deref coercion. Escape hatch: `impl Into<String>` when the function genuinely stores the value.
3. **Overflow checks are ON in debug and OFF in release by default.** `i32::MAX + 1` panics under `cargo test` and wraps silently in the shipped binary. For arithmetic on unbounded or untrusted input use `checked_*` (returns `Option`), `saturating_*` (clamps), or `wrapping_*` (documents intent) — or set `overflow-checks = true` in `[profile.release]` and measure that cost instead of assuming it.
4. **Measure release builds only.** Debug builds skip optimization and keep overflow and bounds instrumentation: arithmetic-heavy loops commonly run 10-50× slower, so a debug timing ranks nothing. Profile with `[profile.release] debug = true` — full optimization plus symbols, so the profiler can name frames.
5. **One error type per boundary: enum outward, dynamic inward.** A library returning `anyhow::Error` denies callers the ability to match on the failure; a binary with 40 enum variants pays for matching nobody does. Default: a `thiserror` enum in library crates, `anyhow::Result` with `.context()` in binaries (`errors.md`).
6. **`main` prints errors with `Debug`, not `Display`.** `fn main() -> Result<(), MyError>` on failure emits `Error: <Debug repr>` and exits 1 — the `Display` message you wrote never appears. Either print it yourself and `std::process::exit(1)`, or return an error type whose `Debug` renders the full report (`anyhow::Error` does).
7. **Bind every guard to a real name.** `let _ = mutex.lock();` releases the lock on that same line — `_` is not a binding, so the guard drops immediately; `let _guard = mutex.lock()?;` holds it to end of scope. Same trap with `RefCell` borrows and every other RAII handle.
8. **Features are additive and unified across the entire graph.** `default-features = false` in your manifest is a request, not a guarantee: if any crate in the tree enables `tokio/full`, the whole build gets `full`. Find the enabler with `cargo tree -e features -i <crate>` before arguing with the manifest.
9. **Pin the toolchain and the lockfile wherever the build must reproduce.** `rust-toolchain.toml` plus `cargo build --locked` in CI. Without `--locked`, CI resolves newer semver-compatible dependencies than your `Cargo.lock` and the difference arrives as a compile error nobody can reproduce locally.

## Compiler Error Codes

The code names the subsystem that refused, which names the file to open. `rustc --explain E0382` prints the canonical explanation; the column below is the first move that is usually right.

| Code | What it actually means | First move |
|---|---|---|
| E0382 | Use after move — consumed by a `for` loop, a by-value method, or a closure capture | Borrow (`&v`, `.iter()`) or copy the needed field out |
| E0499 | Two `&mut` to the same place alive at once | Split the borrow or shorten the first one (rungs 1-2) |
| E0502 | A `&mut` and a `&` overlap | Bind the read to a `let` before the mutation |
| E0505 / E0506 | Move out of, or assign to, something still borrowed | End the borrow first; `std::mem::take` if you need the value now |
| E0507 | Move out of a reference or an index expression | `.clone()`, `.take()`, `mem::replace`, or match by reference |
| E0515 | Returning a reference to a local | Return owned data or an index — the frame dies at `return` |
| E0597 | Borrowed value dropped too early | Hoist the owner's `let` above the borrower's |
| E0716 | Temporary dropped while borrowed | Bind the temporary: `let s = make(); let r = &s;` |
| E0106 | Missing lifetime specifier | Elision cannot choose among multiple inputs — name it (`lifetimes.md`) |
| E0308 | Mismatched types | Compare the two full paths in the message: usually `&T` vs `T`, or two versions of one crate (`cargo.md`) |
| E0277 | Trait bound not satisfied | Missing `use` of the trait, or a bound to add — read `required by` bottom-up |
| E0038 | Trait is not `dyn` compatible (formerly "object safe") | Generic method, `Self` return, or associated const — use generics or split the trait (`traits.md`) |
| E0599 | No method found | Trait not in scope, or the receiver does not deref to the type that has the method |
| E0072 | Recursive type has infinite size | `Box` the recursive field |
| E0117 / E0119 | Orphan rule / conflicting impls | Newtype wrapper; either the trait or the type must be local (`traits.md`) |
| E0282 / E0283 | Type annotations needed / ambiguous | Turbofish (`collect::<Vec<_>>()`) or a typed binding |
| E0596 | Cannot borrow as mutable | Missing `mut` on the binding, or a `&self` method that needs `&mut self` |
| Anything else | Every code is stable and documented | `rustc --explain E0xxx`, then `compiler-errors.md` for the symptoms that have no code |

## Pointer And Container Choice

Pick the weakest tool that expresses the need — every row costs something the row above does not.

| Need | Use | Cost or trap |
|---|---|---|
| Single owner, size known | `T` | The default; most code never leaves this row |
| Read-only view in a parameter | `&T`, `&str`, `&[T]` | No allocation, caller keeps ownership (rule 2) |
| Heap indirection, recursion, or a large value moved often | `Box<T>` | One allocation; `Box<dyn Trait>` also erases the type |
| Borrow when possible, allocate only on modification | `Cow<'a, str>` | Removes "clone just in case" from hot paths (`strings-and-text.md`) |
| Shared ownership, one thread | `Rc<T>` | Not `Send`; reference cycles leak — break them with `Weak` |
| Shared ownership, multiple threads | `Arc<T>` | Atomic refcount; `Arc<T>` by itself is still read-only |
| Mutate shared state, one thread | `Rc<RefCell<T>>` | Borrow checking moves to runtime: `BorrowMutError` panics |
| Mutate one `Copy` value, one thread | `Cell<T>` | Get/set only, no borrows, therefore no panic |
| Mutate shared state, multiple threads | `Arc<Mutex<T>>` | Poisoning on panic; lock ordering is yours to enforce |
| Many readers, rare writer | `RwLock<T>` | Writer starvation and re-entrant read deadlock; measure before preferring it to `Mutex` |
| Initialize once, then read forever | `OnceLock<T>` (`rust >=1.70`), `LazyLock<T>` (`rust >=1.80`) | Replaces `lazy_static` and most `once_cell` uses with no dependency |
| A counter or a flag | `AtomicUsize`, `AtomicBool` | Cheapest sharing available; memory ordering is a real decision (`concurrency.md`) |
| Anything else | Start at `T` and `&T` | Move down a row only when the compiler proves you must |

## Borrow Checker Escape Ladder

Ordered by cost. Take the first rung that compiles.

1. **Shorten the borrow.** NLL ends a borrow at its last use, not at end of scope: `let n = v.len(); v.push(n);` compiles where `v.push(v.len())` does not.
2. **Split the borrow.** `let Foo { a, b } = &mut foo;` yields independent `&mut` to distinct fields; `slice::split_at_mut` does the same for slices. A `&mut self` method borrows the whole struct — that asymmetry explains most "obviously fine" rejections.
3. **Take the value out.** `let buf = std::mem::take(&mut self.buf);` hands you an owned value and leaves `Default::default()` behind; put it back when done. `mem::replace` when there is no sensible default.
4. **Copy the small thing.** An index, an id, a length: copying a `usize` costs nothing and removes the borrow entirely. Index-based access is a legitimate design, not a defeat.
5. **Clone.** Honest, and unobservable in setup, config, and error paths. A clone in a hot loop is a rung-1 problem in disguise; profile rather than guess which one you have.
6. **Interior mutability.** `Cell` for `Copy`, `RefCell` for the rest, `Mutex` across threads. You now own the invariant the compiler used to check, and violations arrive as runtime panics.
7. **Handles instead of pointers.** `Vec<Node>` plus `u32` indices. This is the standard answer for graphs and trees with back-edges; `Rc<RefCell<Node>>` is the version that gets rewritten a year later.
8. **`unsafe`.** Only with a `// SAFETY:` comment naming the invariant, and Miri in CI (`unsafe.md`).

## Output Gates

Before emitting Rust code, verify:

- Every `unwrap`/`expect` outside tests is justified in a comment, or replaced by `?`
- Public parameters take `&str`, `&[T]`, or `impl AsRef<Path>` wherever the function only reads them
- Errors follow rule 5: enum at library boundaries, `#[from]` conversions, no `Box<dyn Error>` in a published signature
- No `std` guard, `Rc`, or blocking call is alive across an `.await`
- Arithmetic on untrusted or unbounded values uses `checked_*` or `saturating_*`, never debug-only overflow checks
- Public enums and structs that may grow carry `#[non_exhaustive]` (`api-design.md`)
- Each new `unsafe` block carries a `// SAFETY:` comment naming the invariant it upholds
- The code passes `cargo clippy -- -D warnings`, not merely `cargo build`

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/rust/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| edition | 2015 \| 2018 \| 2021 \| 2024 | 2024 | Selects syntax and semantics in emitted code (`if let` temporary scope, reserved keywords, `unsafe` attribute forms) and which `cargo fix --edition` advice applies |
| msrv | text (version, e.g. `1.75`) | none (current stable) | Gates which stabilized APIs and syntax may appear; suppresses suggestions newer than the floor and drives `rust-version` in Cargo.toml |
| async_runtime | tokio \| async-std \| smol \| embassy \| none | tokio | Chooses the runtime API in every async example and the spawn, timer, and IO types used from `async.md` |
| error_style | thiserror \| anyhow \| std-enum \| box-dyn \| auto | auto (thiserror in libraries, anyhow in binaries) | Sets the error type generated at each boundary and the `?` conversion strategy in `errors.md` |
| unsafe_policy | forbid \| reviewed \| allow | reviewed | `forbid` emits `#![forbid(unsafe_code)]` and refuses ladder rung 8; `reviewed` requires a SAFETY comment plus a Miri step; `allow` drops the Miri requirement |
| lint_level | default \| pedantic \| deny-warnings | default | Controls which clippy lints the Output Gates enforce and whether generated CI fails on warnings |
| explanation_depth | fix-only \| standard \| teaching | standard | `fix-only` emits the corrected code and one line of why; `standard` adds the rung or rule that applies; `teaching` walks the compiler message and the alternatives that were rejected |

Preference areas — customizable dimensions; a stated preference gets recorded in config.yaml and applied:

- **Tooling**: `cargo test` vs `cargo nextest`, criterion vs a custom harness, `cargo` vs `just`/`make` wrappers, linker choice (`lld`, `mold`) — affects the commands in `testing.md` and `performance.md`
- **Conventions**: module layout (`mod.rs` vs `name.rs`), builder vs plain constructors, re-export policy at the crate root, doc-comment strictness — affects `modules-and-layout.md` and `api-design.md`
- **Platform**: default target triple, libc flavor (gnu vs musl), hosted vs `no_std`, wasm target — affects `cross-compilation.md`, `embedded-no-std.md`, `wasm.md`
- **Dependencies**: appetite for adding a crate vs writing it, vetted-crate list, license and audit regime (`cargo deny`, `cargo audit`) — affects every "add a crate" recommendation
- **Registries and sources**: crates.io vs a private or mirrored registry, `cargo vendor` plus source replacement, air-gapped or offline builds, git vs path dependencies for internal crates — affects the manifest, `.cargo/config.toml`, and every install command in `cargo.md`
- **Work order and gates**: `cargo check` in the edit loop vs full builds, whether clippy runs pre-commit or only in CI, whether `cargo semver-checks` gates release PRs, test-before-fix discipline — affects the command sequences in `performance.md`, `testing.md`, and `api-design.md`
- **Output format**: depth of explanation vs a direct fix (see `explanation_depth`), whole file vs diff, comment density in emitted code, whether to show the rejected alternatives — affects the shape of every answer
- **Safety posture**: how proactively to surface panic, overflow, and unwrap risks vs answering only what was asked — affects Output Gates verbosity
- **Cadence**: dependency update and audit frequency, MSRV bump policy — affects the maintenance advice in `cargo.md`

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `let _ = mutex.lock();` to hold a lock | `_` is not a binding; the guard drops on that line and the lock is free immediately | `let _guard = mutex.lock().unwrap();` (rule 7) |
| `Rc<RefCell<T>>` for a tree with parent pointers | Cycles never free, and every access can panic with `BorrowMutError` | Arena: `Vec<Node>` plus index handles (rung 7) |
| `.clone()` inside a loop to appease the borrow checker | Turns an O(1) borrow into a copy per iteration | Fix the scope first (rungs 1-3), then clone deliberately |
| `#[derive(Clone)]` on every generic type | Derive adds `T: Clone` to every type parameter, and that bound becomes part of your public API | Hand-write the impl, bounding only what actually needs it |
| `String` parameters in a public API | Every caller allocates just to call you | `&str`, or `impl Into<String>` when you store it (rule 2) |
| `mem::transmute` to change a lifetime | Erases the exact property the compiler was checking; UB the moment the data dies | Restructure, or a raw pointer plus a SAFETY comment (`unsafe.md`) |
| `async fn` on functions that never `.await` | Colors the whole call graph and wraps sync work in a state machine | `async` only where an await point exists (`async.md`) |
| Benchmarking with `cargo run` or `cargo test` | Debug build: the number measures instrumentation (rule 4) | `--release`, criterion, and `black_box` on the inputs |
| `cargo clean` to fix a slow build | Deletes the incremental cache and guarantees the slowest possible next build | `cargo build --timings` and read the graph (`cargo.md`) |
| `.iter().collect::<Vec<_>>()` just to loop over it | Allocates a vector to walk it once | Iterate the iterator; `collect` only when the collection is kept |
| `unwrap()` on `Mutex::lock()` everywhere by reflex | Right most of the time, but hides that poisoning means another thread panicked mid-invariant | Handle or document the poison case (`concurrency.md`) |
| Adding lifetime parameters until it compiles | Produces signatures nobody can call and hides the real design error | Return owned data first; add lifetimes once you can state what borrows from what (`lifetimes.md`) |

## Where Experts Disagree

- **`anyhow` in libraries.** Purists never allow it: callers cannot match on the failure. Pragmatists accept it in application-adjacent crates nobody programs against. The boundary both sides accept: if any consumer must branch on the failure kind, the enum is not optional.
- **Clone-first vs lifetime-first.** One camp writes owned data everywhere and optimizes later; the other treats every clone as debt. The line the evidence supports: clones in setup, config, and error paths are noise, clones inside hot loops are the first thing a profiler finds — so the disagreement only bites where you have measured.
- **`unsafe` avoidance.** `#![forbid(unsafe_code)]` teams vs teams accepting reviewed `unsafe` for measured wins. Neither side accepts `unsafe` without a SAFETY comment and Miri coverage — that is not a tradeoff, it is a defect.
- **Generics vs `dyn`.** Monomorphization is faster per call, slower to compile, and multiplies code size per instantiated type; `dyn` costs an indirect call and keeps one copy. Default to generics in leaf code and `dyn` at plugin-shaped boundaries, or wherever compile time or binary size is the measured constraint (`performance.md`).

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/rust (install if the user confirms):
- `cpp` — when the other side of the FFI boundary is C++, or when comparing ownership models
- `go` — the other systems language on the same team; different concurrency tradeoffs
- `async-patterns` — language-agnostic concurrency and cancellation theory behind `async.md`
- `error-handling` — cross-language error design behind rule 5

## Feedback

- If useful, star it: https://clawic.com/skills/rust
- Latest version: https://clawic.com/skills/rust

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/rust.
