# Compiler Errors — Symptom to Cause in Minutes

Read rustc output bottom-up: the last `note:` usually states the real constraint, the first line only names where it surfaced. Codes are indexed in SKILL.md; this file covers the chains, and the failures that have no code at all.

## The Universal First Three

1. Read the `help:` line literally. rustc's suggestion is machine-generated from the actual type error and is right far more often than it is wrong — apply it, then judge the result.
2. `cargo check` before `cargo build` — same diagnostics, no codegen. On a large workspace this is the difference between a 3-second loop and a 40-second one.
3. When a message is unreadable (nested generics, closures, futures), shrink it: comment out call sites until one remains, or `let x: () = expr;` to make rustc print the real type of `expr` in the mismatch.

## Move and Borrow Chains

**`value borrowed here after move` (E0382)**

1. Find the move: a `for x in collection` (consumes — use `&collection` or `.iter()`), a method taking `self`, a closure that captured by value, or passing an owned type to a function.
2. Ask whether the consumer needs ownership at all. If not, change the signature to `&T` — fixes the call site and every future one.
3. If it does, decide who owns it after: `.clone()` if the value is small or the path is cold, restructure so the last use comes first if not.
4. In a loop: the move happened on iteration 1 and the error points at iteration 2. Move the owned value out of the loop, or clone per iteration and admit the cost.

**`cannot borrow as mutable more than once` (E0499) / `as mutable because also borrowed as immutable` (E0502)**

1. Print the borrow spans mentally: NLL ends a borrow at its LAST use. Moving that last use earlier often resolves it with no restructuring.
2. `self.a.push(self.b)` fails through a method (`self.push_a(self.b())`) but succeeds on direct fields — the borrow checker sees fields individually, methods take all of `self`.
3. Slices: `split_at_mut`, `iter_mut` and `chunks_mut` hand out disjoint `&mut` the checker accepts; manual index pairs do not.
4. Maps: `if map.contains_key(k) { map.insert(...) }` double-looks-up and often double-borrows — use `entry(k).or_insert_with(...)`.
5. Still stuck: the design wants two owners. Go to the ladder in SKILL.md rather than sprinkling `RefCell`.

**`cannot move out of ... which is behind a shared reference` (E0507)**

- On a struct field: `.clone()`, or `std::mem::take(&mut x.field)` if you have `&mut`.
- On `Option`: `.take()` (leaves `None`), `.as_ref()` (borrows the inside), `.cloned()`/`.copied()` on `Option<&T>`.
- In a match: match on `&value` and let match ergonomics bind by reference, or add `ref` in older editions.
- On an index: `Vec` indexing cannot move; use `.remove(i)`, `.swap_remove(i)` (O(1), reorders), or `std::mem::replace(&mut v[i], default)`.

## Lifetime Chains

**`missing lifetime specifier` (E0106)** — elision has three rules: each elided input lifetime gets its own; if there is exactly one input lifetime it goes to all outputs; if one input is `&self`, its lifetime goes to all outputs. E0106 means none applied — two or more inputs and no `self`. Decide which input the output borrows from and name it. If the answer is "neither, it's new data", return owned.

**`borrowed value does not live long enough` (E0597)** — the owner is declared after (or drops before) the borrower. Fix by hoisting the owner's `let`, since locals drop in reverse declaration order. The version everyone hits: a temporary in a `let` binding — `let r = make_thing().field();` drops the thing at the semicolon (E0716). Bind it: `let thing = make_thing(); let r = thing.field();`.

**`argument requires that ... is borrowed for 'static`** — usually `thread::spawn` or `tokio::spawn`, both of which require `'static`. Options in order: move owned data in, `Arc` it, or use `std::thread::scope` (`rust >=1.63`) which permits borrows of the parent frame.

## Trait Chains

**`the trait bound X: Y is not satisfied` (E0277)** — read the `required by a bound introduced by this call` chain from the bottom: the last line names the concrete unmet obligation. Frequent real causes, in order:

1. The trait is implemented but not imported — traits must be in scope to call their methods (`use std::io::Write;`).
2. You have `&T` and the bound is on `T` (or vice versa). `impl Trait for T` does not give `&T` the trait.
3. The bound is on an associated type, not the type you are looking at (`I::Item: Display`, not `I: Display`).
4. Two versions of the crate defining the trait — the impl exists, for the other version (`cargo tree -d`).
5. Generic function missing the bound: add it to the `where` clause and let the caller satisfy it.

**`the trait cannot be made into an object` (E0038)** — a method is generic, returns `Self`, takes `self` by value without `where Self: Sized`, or the trait has an associated const. Fix by marking those methods `where Self: Sized` (they become unavailable through `dyn`, which is usually fine) or splitting the trait into an object-safe core plus an extension trait.

**`type annotations needed` (E0282/E0283)** — inference has more than one valid answer. `collect()` is the usual site: `collect::<Vec<_>>()`, or a typed binding. `.into()`, `.parse()`, and `Default::default()` behave the same way.

## Async and Send Chains

**`future cannot be sent between threads safely`** — this error names the future and the offending type in a `note:` that says "the trait `Send` is not implemented for X, held across an await point here". The three sources:

1. `std::sync::MutexGuard` alive across `.await` → shrink the scope with a block, or use the runtime's async mutex.
2. `Rc`, `RefCell`'s `Ref`/`RefMut`, or a raw pointer alive across `.await` → use `Arc`, or restructure.
3. A trait object without `+ Send`: `Box<dyn Error>` is not `Send`; write `Box<dyn Error + Send + Sync>` for anything crossing a spawn.

The scope fix in practice: `let v = { let g = m.lock().unwrap(); g.value.clone() }; do_async(v).await;` — the guard dies at the closing brace, before the await.

**`cannot be shared between threads safely` (`Sync`)** — the value is captured by reference in a spawned future. Same three sources, one level of indirection out.

## Cargo and Toolchain Errors

| Message | Real cause | Fix |
|---|---|---|
| `expected struct X, found struct X` | Two semver-incompatible versions of the same crate; the types are genuinely different | `cargo tree -d`, then align versions or bump the lagging dependency |
| `linker 'cc' not found` | No system C toolchain | Install build-essential / Xcode CLT / MSVC build tools |
| `failed to run custom build command for openssl-sys` | Native OpenSSL headers absent, or cross-compiling | `rustls` instead, or the `vendored` feature |
| `package X does not have feature Y` | Feature renamed, or you enabled it on the wrong crate in the graph | `cargo tree -e features` to see what the resolver actually built |
| `error: could not compile ... (signal: 9, SIGKILL)` | rustc ran out of memory, almost always in CI | Lower `-j`, set `codegen-units` higher, drop `lto = "fat"` for CI builds |
| `can't find crate for 'std'` | Target installed as `no_std`-only, or the target's std was never added | `rustup target add <triple>`, or you really are on a `no_std` target |
| `feature X is not stable` | The code needs nightly, or your MSRV floor is below the stabilization version | Check `msrv` in config before suggesting the API |
| `perhaps two different versions of crate proc-macro2` | Duplicate proc-macro dependency across a workspace | Unify with `[workspace.dependencies]` |

## "It Compiled Yesterday"

Check in this order; each is a one-minute test:

| Difference | Check |
|---|---|
| Toolchain moved | `rustc -V` vs the CI log; `rust-toolchain.toml` absent means everyone floats |
| Lockfile resolved differently | `git diff Cargo.lock`; build with `--locked` to reproduce CI exactly |
| A feature got enabled by a new dependency | `cargo tree -e features -i <crate>` (SKILL.md rule 8) |
| Edition changed | `cargo fix --edition` output left half-migrated code |
| A dependency published a semver-compatible break | Pin the exact version, confirm, then report upstream |
| Only `--all-features` fails | Two features are mutually exclusive; feature-gate the conflict or test the real combinations, not the union |

## When You Are Truly Stuck

Minimize: copy the failing function into a scratch crate and delete everything that still reproduces it. The remaining ten lines either make the cause obvious or make a good question. `cargo expand` when a macro is involved; `cargo +nightly rustc -- -Ztreat-err-as-bug` only when you suspect a compiler bug, which is almost never the answer.
