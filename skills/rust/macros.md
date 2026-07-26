# Macros — `macro_rules!` and Proc Macros

Order of preference, and it matters: a function, then a generic function, then a trait with a blanket impl, then `macro_rules!`, then a derive macro, and only then a full proc macro. Each step down costs compile time, debuggability, and the ability of a reader to jump to a definition.

## When a Macro Is Actually the Answer

| Need | Answer |
|---|---|
| Variable number of arguments | `macro_rules!` (`vec!`, `println!`) |
| Code that must expand at each call site to capture `file!`/`line!`/`module_path!` | `macro_rules!` |
| Implementing the same trait for a list of types | `macro_rules!` over a type list |
| Deriving behavior from a struct's fields | Derive proc macro |
| A DSL with its own syntax | Function-like proc macro |
| Rewriting a function's body (instrumentation, async transforms) | Attribute proc macro |
| Anything expressible with generics | Not a macro |

## `macro_rules!` Essentials

```rust
macro_rules! try_all {
    ($($e:expr),+ $(,)?) => { $( $e?; )+ };
}
```

- Fragment specifiers: `expr`, `ident`, `ty`, `path`, `pat`, `stmt`, `block`, `item`, `literal`, `tt`, `lifetime`, `vis`, `meta`. `tt` matches anything and is the escape hatch when the follow-set rules block you.
- **Follow-set restrictions**: after an `expr` or `stmt` fragment only `=>`, `,`, or `;` may appear; after `ty` or `path` only a small set. This is why an apparently reasonable pattern will not parse — restructure the syntax or switch to `tt` munching.
- Repetition: `$(...),*` zero or more separated by commas, `+` one or more, `?` zero or one (no separator allowed). `$(,)?` at the end accepts a trailing comma, which every ergonomic macro should.
- Once a fragment is captured as `expr`, it is an opaque AST node: you cannot match inside it later. Capture as `tt` if you need to inspect it.
- Matching is first-arm-wins, top to bottom. Put the most specific arms first, and remember an arm that fails to match falls through rather than erroring, which is why a typo produces "no rules expected this token" pointing at the whole invocation.

## Hygiene, Precisely

- Local variables introduced by the macro cannot collide with the caller's: `let tmp = ...` inside a macro is a different `tmp`. This is the guarantee people mean by "macros are hygienic".
- Hygiene does **not** cover paths, types, or traits. A macro referring to `Vec::new()` breaks in a module where `Vec` is shadowed. Always write absolute paths in expansions: `::std::vec::Vec::new()`.
- `$crate` expands to the defining crate — mandatory for any macro your crate exports, or it resolves against the caller's crate and fails.
- `#[macro_export]` puts the macro at your crate root regardless of the module it is defined in, and it is part of your public API forever. Re-export from the intended path with `pub use` and `#[doc(hidden)]` on the root name.
- Identifiers passed **in** by the caller keep the caller's hygiene, which is what makes `$name:ident` work for generating a struct the caller can then use.

## Debugging Expansions

1. `cargo expand` (installs as `cargo-expand`) prints the post-expansion source — the first move for every macro bug, declarative or procedural.
2. `cargo expand --lib path::to::module` to avoid a thousand lines of output.
3. `trace_macros!(true)` on nightly logs each expansion step for `macro_rules!`.
4. Recursion limit is 128 by default; deep `tt` munchers hit it and the error names the limit. `#![recursion_limit = "256"]` raises it, but a limit you keep raising means the macro should be a proc macro or a loop.
5. An error inside an expansion points at the macro definition, not the call site. `#[track_caller]` on generated functions and explicit `compile_error!` arms for bad input restore usable diagnostics.

## Proc Macros

- Must live in their own crate with `[lib] proc-macro = true`, and cannot export anything else. The convention is `foo` (the API) plus `foo-macros` (the derive), with `foo` re-exporting the macro so users add one dependency.
- They are compiled for the **host** and run at compile time — they are arbitrary code execution during your build, and they cannot be cross-compiled away.
- `syn` parses, `quote!` generates, `proc-macro2` bridges. `syn`'s `full` feature is heavy: enable only the features you parse, since it lands on the critical path of every dependent build.
- Emit errors with `syn::Error::new_spanned(&tokens, "message").to_compile_error()` rather than panicking: a panic reports "proc macro panicked" with no span, an error points at the user's actual field.
- Span choice controls hygiene: `Span::call_site()` resolves at the call site (what you want for names the user should see), `Span::mixed_site()` gives `macro_rules!`-like hygiene for internal temporaries.

## Derive Macros

- Derives may only **add** items; they cannot modify the annotated type. An attribute macro can, and that is the only difference that matters when choosing.
- Helper attributes must be declared: `#[proc_macro_derive(MyTrait, attributes(my_attr))]`, otherwise the user's `#[my_attr]` is an unknown-attribute error.
- Generated impls need the right bounds. Deriving `impl<T: MyTrait> MyTrait for Wrapper<T>` blindly reproduces the `#[derive(Clone)]` over-bounding problem; consider bounding the field types instead of the parameters.
- Generic parameters, lifetimes, and `where` clauses must be reproduced in the impl. `syn`'s `split_for_impl()` exists exactly for this and hand-rolling it is where most derive bugs live.

## Testing Macros

- `trybuild` compiles fixture files and compares against expected stderr — the only way to test that a **bad** invocation produces a good error message. Store the expected output and review diffs; regenerate with `TRYBUILD=overwrite`.
- Test the expansion's behavior with ordinary unit tests in a separate crate that uses the macro, since the proc-macro crate itself cannot contain them.
- Snapshot the expansion (`cargo expand` output through `insta`) for macros whose generated code is intricate: it turns silent expansion changes into a reviewable diff.
- Test with generics, lifetimes, `where` clauses, tuple structs, unit structs, and enums with all three variant shapes. Those six shapes catch nearly every derive bug.

## Cost and Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| A proc macro where a generic would do | Adds `syn`/`quote` to every dependent's build graph | Trait with a blanket impl |
| `Vec::new()` unqualified in an expansion | Breaks under shadowing at the call site | `::std::vec::Vec::new()` |
| `crate::` in an exported macro | Resolves to the caller's crate | `$crate::` |
| Panicking on invalid input | "proc macro panicked", no span, no help | `syn::Error` with a span |
| A `tt` muncher over long input | Quadratic expansion and recursion-limit errors | Iterate with repetition, or write a proc macro |
| Macro-generated code with no tests | Errors surface in users' crates | `trybuild` plus a consumer test crate |
| Reading a whole file in a proc macro without `rerun-if-changed` | Stale output; incremental builds never notice | Do file IO in `build.rs` with proper dependency tracking |
