# Pattern Matching and Type Modeling — `match`, Combinators, and Unrepresentable States

Two halves of one craft: patterns are how you take a value apart, and the type you chose decides how many patterns exist. Most "this match is ugly" complaints are a modeling problem — a `bool` and an `Option` that should have been one enum.

## Match Ergonomics and Binding Modes

- Matching a reference binds by reference automatically (`rust >=1.26`): `match &opt { Some(x) => ...}` gives `x: &T` with no `ref` keyword. `ref`/`ref mut` still exist for the cases where you match an owned value but want to borrow one field.
- The move rule: an arm that binds a non-`Copy` field by value moves the whole scrutinee. Match on `&value` — or bind with `ref` — when you need the original afterwards.
- `_` discards without binding; `..` skips the rest of a struct or tuple (`Point { x, .. }`, `(first, .., last)`). Naming a binding `_name` still binds it, and it lives to end of scope.
- Or-patterns nest (`rust >=1.53`): `Some(A | B)` is legal, and every alternative must bind the same names with the same types.
- Range patterns: `'0'..='9'`, `1..=9`, and open ranges `n if n > 100`. Exclusive `..` in patterns is stable from `rust >=1.80`.
- Slice patterns read like a parser: `[] | [_] => ...`, `[first, rest @ ..] => ...`, `[.., last] => ...`. They replace most manual length checks and index arithmetic.
- `@` binds while testing: `n @ 1..=9 => println!("digit {n}")`.
- Guards do **not** participate in exhaustiveness. `Some(x) if x > 0` leaves `Some(x)` unproven, so the compiler still demands another arm — and a guard cannot move out of the bound value.

## `let else`, `if let`, and `matches!`

- `let else` (`rust >=1.65`) flattens the extract-or-bail shape and keeps the binding in the outer scope; the `else` block must diverge (`return`, `continue`, `break`, `panic!`):

```rust
let Some(config) = load()? else { return Err(Error::Missing) };
```

- `if let ... else if let` chains are the right shape when each branch does different work; three or more branches on the same value should be a `match`.
- `while let Some(x) = stack.pop()` is the idiomatic drain loop.
- `matches!(v, Pattern)` for a boolean test without an arm: `matches!(state, State::Ready | State::Idle)`.
- Edition 2024 changed `if let` temporary scope: the scrutinee's temporaries drop before the `else` block runs, which silently fixes a class of `if let Some(x) = m.lock().unwrap().get(k)` deadlocks and changes behavior when you migrate.

## Exhaustiveness Is a Feature — Do Not Disable It

- On **your own** enums, never write a catch-all `_ => ...`. List the variants. The payoff arrives the day someone adds a variant: the compiler names every site that must change, which is the single largest maintenance advantage Rust has over dynamic modeling.
- Reserve `_` for foreign enums, integers, and genuinely open sets. `_ => unreachable!()` on your own enum is the worst of both: it silences the compiler and converts the next added variant into a runtime panic. Write `Variant::A | Variant::B | Variant::C => ...` instead — the same length, and it fails at build time.
- Consumer side of `#[non_exhaustive]` (the attribute you meet on other people's types):

| You are doing | What the attribute does | Your move |
|---|---|---|
| Matching a foreign `#[non_exhaustive]` enum | A wildcard arm is mandatory even if you covered every variant | `_ => ...` with a real fallback, not `unreachable!()` |
| Destructuring a foreign `#[non_exhaustive]` struct | Exhaustive struct patterns are rejected | `Foo { a, .. }` |
| Constructing one | Struct-literal syntax is rejected outside the defining crate | The constructor or builder the crate provides |
| Adding it to your own type | Downstream matches keep compiling when you add a variant | Add it from 1.0, before anyone depends on exhaustiveness |

## The `Option`/`Result` Combinator Vocabulary

One transform on the happy path → combinator. Two branches that both do real work → `match`. Chains longer than three combinators read worse than the `match` they replaced.

| Want | `Option` | `Result` |
|---|---|---|
| Transform the value | `map` | `map` |
| Transform the failure | — | `map_err` |
| Chain a fallible step | `and_then` | `and_then` |
| Supply a default | `unwrap_or`, `unwrap_or_else`, `unwrap_or_default` | same three |
| Convert between them | `ok_or(e)` / `ok_or_else(|| e)` | `.ok()`, `.err()` |
| Test the inside | `is_some_and(f)` | `is_ok_and(f)` |
| Drop values that fail a test | `filter(pred)` | `Result` has no `filter`: use `and_then` |
| Borrow the inside | `as_ref`, `as_mut`, `as_deref` | `as_ref`, `as_mut` |
| Take or swap in place | `take()`, `replace(v)`, `get_or_insert_with(f)` | — |
| Combine two | `zip`, `or`, `xor` | `and`, `or` |
| Swap the nesting | `transpose()` (`Option<Result<T,E>>` ↔ `Result<Option<T>,E>`) | `transpose()` |
| Collapse nesting | `flatten()` | — |
| Anything else | `match` — it is not a defeat | `match` |

- `unwrap_or(expensive())` evaluates the argument even on the happy path; `unwrap_or_else(|| expensive())` does not. Same for `ok_or` vs `ok_or_else`.
- `?` is the combinator for "propagate": it applies `From::from` to the error, which is why one `#[from]` conversion removes a whole layer of `map_err`.

## Modeling: Make the Illegal State Unrepresentable

- Two `Option`s that are never both `Some` are one enum. `struct Job { running: bool, finished_at: Option<Instant> }` allows `running && finished_at.is_some()` — `enum Job { Running(Instant), Finished { at: Instant } }` does not.
- Booleans in a struct are the smell: three `bool` fields describe eight states of which you support three. An enum names the three, and every `match` over it becomes exhaustive.
- Newtype vs enum: **newtype** when the set of values is open and the invariant is a check (`Email(String)` built only through `parse`); **enum** when the set is closed and each case carries different data. A newtype whose constructor validates gives every downstream function permission to skip re-validating.
- Parse, do not validate: convert unstructured input into the constrained type once, at the boundary, and let the type carry the proof inward. A function taking `&str` and re-checking is a function whose caller can forget.
- Prefer data in the variant over a parallel field: `enum Shape { Circle { r: f64 }, Rect { w: f64, h: f64 } }` beats a `kind` tag plus five `Option`s, and the compiler enforces the pairing.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `_ => {}` on your own enum | A new variant compiles silently and is ignored at runtime | List the variants; let the compiler find the sites |
| `if let Some(_) = x` | Tests presence but reads like a binding | `x.is_some()`, or `matches!` |
| `match x { Some(v) => Some(f(v)), None => None }` | Ten tokens for what `map` says in three | `x.map(f)` |
| A five-combinator chain | Nobody can say which step produced the `None` | `match`, or `let else` per step with a distinct error |
| `unwrap_or(build_default())` in a hot path | The default is built on every call | `unwrap_or_else` |
| `_ => unreachable!()` to silence a foreign `#[non_exhaustive]` | Turns an upstream addition into a production panic | A real fallback branch |
| Nested `match` on two values | Cartesian blow-up and unreadable arms | Match the tuple: `match (a, b) { (Some(x), Ok(y)) => ... }` |
