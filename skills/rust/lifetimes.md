# Lifetimes — Elision, Variance, and `'static`

A lifetime is not a duration; it is a region of code over which a reference must stay valid. You never make something live longer by annotating it — you only tell the compiler which existing region to check against. If the answer is "I need it to live longer", the fix is ownership, not syntax.

## The Three Elision Rules

Applied in order to every function signature:

1. Each elided lifetime in an input position gets its own fresh parameter.
2. If there is exactly one input lifetime, it is assigned to every elided output lifetime.
3. If one of the inputs is `&self` or `&mut self`, that lifetime is assigned to every elided output lifetime.

Consequences worth memorizing:

- `fn f(a: &str, b: &str) -> &str` is rejected (E0106): rule 2 does not apply, rule 3 has no `self`. Name the one the result borrows from: `fn f<'a>(a: &'a str, b: &str) -> &'a str`.
- Any `&self` method returning a reference silently borrows `self` for the whole result. That is why `fn iter(&self) -> Iter<'_>` blocks mutation of the struct while the iterator lives — it is not a bug in the iterator.
- `'_` is the anonymous lifetime: `Iter<'_>` means "elided, and I am telling the reader there is one". `#[deny(elided_lifetimes_in_paths)]` makes the compiler require it, which is worth turning on in library crates.

## `'static` — Two Different Meanings

- `&'static T`: a reference valid for the entire program. String literals, `const` promotions, `Box::leak` results.
- `T: 'static` (a bound): the type contains no non-`'static` references. `String`, `Vec<u8>`, and every owned type satisfy it. This is the one `thread::spawn` and `tokio::spawn` require, and it does **not** mean the value lives forever — it means the value could, because nothing inside it borrows.

The error `argument requires that X is borrowed for 'static` almost always means you hit the bound, not the reference. Fixes in order: move owned data into the closure, `Arc` shared data, use `std::thread::scope` (`rust >=1.63`) for borrows of the parent frame, or restructure so the spawned work receives a message instead of a borrow.

## Structs That Hold References

```rust
struct Parser<'a> { input: &'a str, pos: usize }
```

- The struct is now infected: every type and function that holds a `Parser` needs the parameter too, and the struct can never outlive its input. That is the correct design for a zero-copy parser and the wrong one for a config object.
- Decision rule: hold references when the type is a short-lived view over data owned elsewhere in the same call stack (parsers, iterators, formatters). Hold owned data or `Arc` when the type is stored, returned upward, sent across threads, or kept in a collection.
- Middle ground: `Cow<'a, str>` lets one type serve both — borrowed until a caller mutates.
- A struct field cannot borrow from another field of the same struct. Store the offsets and re-slice on access.

## Bounds Between Lifetimes

- `'a: 'b` reads "`'a` outlives `'b`" — anywhere a `&'a T` must be usable where `&'b T` is expected.
- `T: 'a` means every reference inside `T` is valid for at least `'a`. Required when you store a generic `T` next to a `&'a` something.
- These bounds appear automatically in most code. When you have to write one by hand, first check whether the real fix is to return owned data.

## Variance (why some assignments are refused)

| Position | Variance in the lifetime | Practical effect |
|---|---|---|
| `&'a T` | Covariant | A longer-lived reference is accepted where a shorter one is needed |
| `&'a mut T` | Covariant in `'a`, invariant in `T` | You cannot substitute a `&mut Vec<&'long str>` where `&mut Vec<&'short str>` is wanted |
| `Cell<T>`, `RefCell<T>`, `Mutex<T>` | Invariant in `T` | Interior mutability forbids the substitution that would let you smuggle a short reference into a long-lived slot |
| `fn(T) -> U` | Contravariant in `T`, covariant in `U` | A function accepting a longer lifetime can stand in for one accepting a shorter one |
| `PhantomData<T>` | Same as `T` | The tool for making a type parameter you do not store still affect variance |

The symptom of invariance is a "lifetime may not live long enough" error where both lifetimes look identical. It usually means a `&mut` or a `Cell` is in the path; the fix is nearly always to restructure so the two regions are the same, not to add a bound.

## Higher-Ranked Bounds (HRTB)

`for<'a> F: Fn(&'a str) -> &'a str` says the closure works for **every** lifetime, not one the caller picks. You need it when storing a callback that will be handed references it has never seen:

```rust
fn register<F>(f: F) where F: for<'a> Fn(&'a Event) -> Result<(), Error> + Send + 'static
```

- Closures usually infer this automatically; the moment you write the bound by hand, put `for<'a>` in front or the lifetime binds at the wrong level.
- The classic failure is a closure returning a reference derived from a captured value rather than from the argument — HRTB rejects it correctly, because the caller could pass a shorter-lived argument.

## Generic Associated Types (`rust >=1.65`)

```rust
trait Container { type Iter<'a>: Iterator<Item = &'a str> where Self: 'a;
                  fn iter(&self) -> Self::Iter<'_>; }
```

GATs exist for exactly one shape: an associated type that must borrow from `&self`. Before them the workaround was returning `Box<dyn Iterator + '_>` (allocation plus dynamic dispatch). Reach for a GAT only when the lending iterator or lending stream shape is genuinely required — the `where Self: 'a` clause is mandatory and the error messages are still the worst in the language.

## Escape Hatches, With Their Price

| Hatch | Price |
|---|---|
| Return owned (`String`, `Vec<T>`) | An allocation; usually invisible and always correct |
| `Arc<T>` / `Rc<T>` | Refcount traffic; removes the lifetime entirely |
| Index into an arena | You now police staleness (an index into a freed slot compiles fine) |
| `Cow<'a, T>` | One extra branch; keeps the zero-copy path |
| `Box::leak` for a genuinely process-lifetime value | Memory never reclaimed; acceptable for config loaded once, never in a loop |
| `unsafe` lifetime transmute | Undefined behavior the moment the referent dies — not an option |
