# Modules and Project Layout — Files, Visibility, and When to Split a Crate

Rust's module tree is independent of the filesystem until you say otherwise, and `pub` means "visible to whoever can see this module", not "exported". Both surprises produce the same two errors — unresolved path and private item — and the same fix: draw the tree before moving files.

## The Module Tree

- `mod foo;` **declares** the module and tells the compiler to load it; `use` only brings a name into scope. A file nobody declares is not compiled, which is why a new `src/foo.rs` can sit there with syntax errors and a green build.
- Modern layout (edition 2018+): `src/foo.rs` for the module, `src/foo/bar.rs` for its child, and `src/foo/mod.rs` only if you prefer it. Do not mix `foo.rs` and `foo/mod.rs` for the same module — that is an error, not a fallback.
- Paths: `crate::` from the crate root, `self::` from the current module, `super::` from the parent. Inside `tests/` and `examples/` your crate is an **external** dependency, so the root is `my_crate::`, not `crate::`.
- `use` supports grouping (`use std::io::{self, Read, Write}`), renaming (`as`), and glob (`use prelude::*`) — globs belong to preludes and test modules only, because they make every name's origin unsearchable.
- `#[path = "..."]` overrides the file mapping. Legitimate for generated code and platform variants; anywhere else it is a puzzle for the next reader.
- Modules inside one crate may reference each other in both directions. There is no cycle problem below the crate level — that constraint appears only between crates.

## Visibility, Precisely

| Marker | Visible to |
|---|---|
| (nothing) | The defining module and its descendants |
| `pub(super)` | The parent module and below |
| `pub(crate)` | Everything in this crate, nothing outside |
| `pub(in crate::a::b)` | That module subtree only |
| `pub` | Everywhere the containing module is reachable |
| `pub use` | Re-export: the name becomes reachable at this path too |

- `pub` inside a private module reaches nobody outside: `mod internal { pub struct Thing; }` keeps `Thing` unreachable. That combination is the standard way to write "public within the crate's own layers" — and the standard cause of "I marked it pub and it still says private" (E0603).
- Fields have their own visibility. A `pub struct` with private fields cannot be built by literal syntax downstream, which is how you force construction through a validating constructor.
- Trait methods inherit the trait's visibility and cannot be marked individually; if a method should not be public, it belongs on an inherent impl or a private trait.
- `pub(crate)` is the right default for anything in a library that is not part of the documented API. It costs nothing and it makes the public surface greppable.

## Shaping the Public Surface

- Re-export the types users need from the crate root: `pub use error::Error;`. A user should not have to know your module tree to write a signature.
- Once re-exported, both paths are public and both are promises. Keep the internal module private (`mod error;`, not `pub mod error;`) so only the root path is a commitment.
- `#[doc(inline)]` on a re-export renders the item in place instead of as a link, which is what makes a flat root feel like the real API in rustdoc.
- A `prelude` module is appropriate for crates with many traits that must be in scope; it is noise for crates with three types.
- Group modules by domain concept, not by kind: `mod parser`, `mod render` beat `mod traits`, `mod structs`. Kind-based grouping puts every change in every file.

## Binaries, Libraries, and the Rest of the Tree

```
src/lib.rs          the library; the only thing other crates and tests/ can see
src/main.rs         the default binary; it should be thin and call into the lib
src/bin/tool.rs     extra binaries, each its own crate root
benches/ examples/ tests/   compiled as separate crates against the public API
build.rs            runs on the host before the crate compiles
```

- Put the logic in `lib.rs` and keep `main.rs` to argument parsing plus one call. Anything only reachable from `main.rs` cannot be tested from `tests/`, benchmarked, or reused — this single decision causes more untestable Rust than any other.
- `examples/` are compiled by `cargo test` but not run; they are cheap compile-checked documentation of the public API, and the only place a reader looks before the docs.
- A binary that grows submodules keeps them under `src/bin/tool/` with `main.rs` inside, or moves them into the library — `src/bin/*.rs` siblings cannot import each other.

## When to Split a File, a Module, or a Crate

| Signal | Move to |
|---|---|
| A file passes roughly 500 lines, or two unrelated concepts share it | A sibling module |
| A module's items are all `pub(crate)` and used by one caller | Inline it back — indirection with one user is cost, not structure |
| Rebuilds are slow and the crate is large | Separate crates: rustc parallelizes poorly inside one crate and recompiles it whole |
| A piece is genuinely reusable, or has heavy dependencies most users do not need | Its own crate, or a feature flag |
| Two crates need each other | Neither: extract the shared types into a third crate, or invert with a trait — crate cycles are rejected |
| Anything else | Leave it. A workspace of eight crates for one binary is a build-time tax with no reader benefit |

## Common Errors

| Message | Real cause | Fix |
|---|---|---|
| `file not found for module 'foo'` | `mod foo;` with no `foo.rs` or `foo/mod.rs` next to the declaring file | Create it at the path the error names, or delete the `mod` |
| `unresolved import` / `use of undeclared crate or module` (E0432/E0433) | The module was never declared with `mod`, or the path starts at the wrong root | Add `mod`, or fix `crate::` vs `self::` vs `super::` |
| `... is private` (E0603) | A `pub` item inside a private module | `pub(crate)` on the module, or re-export the item |
| `cannot find type in this scope` inside `tests/` | Integration tests see only the public API | `pub` or `pub use` it, or move the test into a `#[cfg(test)]` module |
| `unused import` on something you clearly use | The use is behind a `#[cfg]` that is off in this configuration | Gate the import with the same `cfg` |
