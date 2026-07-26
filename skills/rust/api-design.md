# API Design — Public Surfaces and Semver Hazards

Everything reachable from your crate root is a promise. Rust's compiler enforces the promise so precisely that changes which look internal — a new trait impl, a stricter bound, an extra field — break downstream builds.

## Signature Defaults

| Parameter is | Take | Why |
|---|---|---|
| Read-only text | `&str` | Accepts `String`, `&String`, literals (SKILL.md rule 2) |
| Read-only sequence | `&[T]` | Accepts `Vec<T>`, arrays, slices |
| A filesystem path | `impl AsRef<Path>` | Accepts `&str`, `String`, `&Path`, `PathBuf` |
| Text you will store | `impl Into<String>` | One allocation, at the caller's choice of source |
| A collection you will consume | `impl IntoIterator<Item = T>` | Accepts iterators, `Vec`, arrays |
| A callback | `impl Fn(..)`, or `&dyn Fn(..)` when stored | Generic for hot paths, `dyn` at plugin boundaries |
| Configuration with more than three fields | A builder or an options struct | Positional booleans are unreadable at the call site |

- Return concrete owned types. `-> Vec<T>` beats `-> impl Iterator<Item = T>` unless laziness is the point, because `impl Trait` in return position pins you to one hidden type and leaks its auto traits.
- Never take `&Vec<T>` or `&String` — they accept strictly less than `&[T]` and `&str` for no benefit.
- Prefer several small functions to one with a `bool` parameter; `sort(true)` at a call site is unreadable and unsearchable.

## Newtypes and Type-State

- Newtype any identifier that could be confused with another: `struct UserId(u64)` makes `fn transfer(from: UserId, to: AccountId)` impossible to call wrongly. `#[repr(transparent)]` makes it free.
- Newtype to escape the orphan rule (a foreign trait on a foreign type is only implementable through a local wrapper), and to enforce an invariant at construction: if `Email` can only be built through `Email::parse`, every later function receiving one may skip validation.
- Type-state encodes a protocol in the type system: `Request<Unsent>` has `.send()`, `Request<Sent>` has `.response()`, and the wrong call is a compile error. Worth it for a protocol misuse that would otherwise be a runtime panic; overkill for a two-step builder.
- Builder pattern: `&mut self` returning `&mut Self` for optional settings, plus `build()` returning `Result` when required fields can be missing. Consuming `self` builders read better in one expression and worse in a loop.

## Errors, Panics, and Documentation

- Public functions return `Result` for anything caller-caused; document every panic in a `# Panics` section, every `unsafe` precondition in `# Safety`, and every error case in `# Errors`.
- `#![deny(missing_docs)]` on library crates. The first sentence of each doc comment is what appears in the module index, so it must stand alone.
- Every public type gets `Debug`. Missing `Debug` makes downstream `#[derive(Debug)]` fail and is the most common complaint about new crates.
- Doc examples are compiled and run — they are the only documentation that cannot rot.
- `#[doc(hidden)]` is not privacy: the item is still usable, still part of the ABI, and still breaks people when removed. Use it only for macro-support items, and say so.

## Semver Hazards

Breaking changes that do not look like changes:

| Change | Breaks | Mitigation |
|---|---|---|
| Adding a public struct field | Struct literal construction and exhaustive destructuring | `#[non_exhaustive]` from version 1.0 |
| Adding an enum variant | Exhaustive `match` downstream | `#[non_exhaustive]` on the enum |
| Adding a method to a trait | Every downstream impl, unless it has a default body | Default body, or a sealed trait |
| Adding a blanket impl | Conflicts with downstream impls (E0119) | Treat blanket impls as final from 1.0 |
| Making a function generic | Turbofish call sites and function-pointer coercions | Add a new function instead |
| Tightening a bound (`T: Clone` added) | Callers with types that lack it | New method, or a default-implemented extension |
| Loosening a return type to `impl Trait` | Anyone who named the concrete type | Keep the concrete type in the signature |
| Removing an auto trait from a returned `impl Trait` | Callers who spawned it | Write `+ Send` explicitly so it is checked |
| Renaming a lifetime or type parameter | Callers using turbofish with explicit lifetimes | Rare, but real for `struct Foo<'a, T>` |
| Raising the MSRV | Users pinned to an older toolchain | Minor bump plus a changelog line |

`cargo semver-checks` catches a large share of these mechanically. Run it in CI on every release PR; it is far cheaper than a yanked version.

## Feature Design

- Features must be additive (SKILL.md rule 8): enabling one may only add API, never change or remove it. A `no-std` feature is the classic mistake — invert it into a `std` feature that is on by default.
- Name features after capabilities (`tls`, `derive`, `serde`), not after implementations, so you can swap the implementation without a breaking rename.
- An optional dependency that appears in a public signature makes that signature feature-dependent; document it and gate the docs with `#[cfg_attr(docsrs, doc(cfg(feature = "serde")))]`.
- Keep the default feature set small and useful. Every default is something a `default-features = false` user must re-enable, and something that gets unified back on for everyone by any dependent that forgets (SKILL.md rule 8).

## The Shape of the Public Surface

- Every reachable path is a separate promise. A type re-exported at the root **and** left public in its module has two paths, and removing either one is a breaking change — keep the module private so only the root path is a commitment.
- Renaming a module is breaking even when every type keeps its name; renaming a public field is breaking even when a constructor exists. Both are semver-major, and neither is caught by tests.
- Widening visibility is always safe, narrowing never is: ship `pub(crate)` and promote on request rather than publishing a surface you must keep.
- Adding a public item is a minor bump — except a trait method without a default body, a blanket impl, or a variant on an exhaustive enum, all of which break downstream code that compiled yesterday.

## Pre-1.0 Release Checklist

- Public types implement `Debug`, and `Clone`/`PartialEq` where they are values
- `#[non_exhaustive]` on every public enum and options struct that could grow
- Errors are one enum per boundary, `#[non_exhaustive]`, with `source()` chains
- Traits intended for downstream impls have default bodies; the rest are sealed
- Feature flags are additive, documented, and tested in combination (`cargo hack`)
- `#![deny(missing_docs)]`, doc examples compile, `cargo doc` has no warnings
- MSRV declared in `rust-version` and verified by a CI job
- `cargo semver-checks` and `cargo publish --dry-run` both clean
- README examples are the doc examples (`#![doc = include_str!("../README.md")]`)
