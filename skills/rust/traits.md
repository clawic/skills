# Traits — Coherence, Dispatch, and the Shapes That Refuse to Compile

Traits are Rust's only abstraction mechanism, so every design question eventually becomes a trait question: generic or `dyn`, associated type or parameter, and who is allowed to write the impl.

## Coherence and the Orphan Rule

You may write `impl Trait for Type` only if the trait is local to your crate, or the type is (roughly — the full rule also allows a local type nested inside a foreign generic, with the local type appearing before any type parameter).

- E0117 (orphan) is the "I want `impl Display for Vec<MyThing>`" error. Answer: newtype. `struct Things(Vec<MyThing>);` then implement on `Things`, with `Deref` to `[MyThing]` if callers need the collection API. `#[repr(transparent)]` makes the newtype free at runtime.
- E0119 (conflicting impls) usually comes from a blanket impl you or a dependency wrote. `impl<T: Foo> Bar for T` claims every present and future type implementing `Foo` — the compiler must reject any specific `impl Bar for X` even if `X: Foo` is false today, because the crate could add it later.
- There is no stable specialization. Design as if a blanket impl is final, because for downstream users it is.
- Adding a blanket impl to a published crate is a breaking change even though nothing in the signature changed.

## Generics vs `dyn`

| Axis | Generic (`impl Trait` / `<T: Trait>`) | `dyn Trait` |
|---|---|---|
| Dispatch | Static, inlinable | One vtable indirection, rarely inlined |
| Code size | One copy per concrete type | One copy total |
| Compile time | Grows with instantiations | Flat |
| Heterogeneous collections | Impossible | The reason `dyn` exists |
| Recursion / plugin boundaries | Awkward or impossible | Natural |
| Return position | `-> impl Trait` hides the type but fixes it to one | `-> Box<dyn Trait>` allows different types per branch |

- Default: generics in leaf code, `dyn` where the set of types is open or the compile-time and binary-size cost is measured (SKILL.md, Where Experts Disagree).
- The outlining trick when a generic API is convenient but monomorphization is expensive: keep the generic wrapper tiny and immediately call a non-generic inner function.

```rust
pub fn write_all(path: impl AsRef<Path>, data: &[u8]) -> io::Result<()> {
    fn inner(path: &Path, data: &[u8]) -> io::Result<()> { /* the real body */ }
    inner(path.as_ref(), data)
}
```

- `-> impl Trait` leaks auto traits: if the hidden type stops being `Send`, every caller that spawned it breaks, with no signature change to point at. Write `-> impl Future<Output = T> + Send` when it matters.

## `dyn` Compatibility (E0038)

A trait can become a trait object only if no method: is generic over types, returns or takes `Self` by value, has no receiver, or uses an associated const. Associated types are allowed but must be named at the use site (`dyn Iterator<Item = u32>`).

- Fix 1: mark the offending methods `where Self: Sized` — they stay callable on concrete types and vanish from the `dyn` view.
- Fix 2: split into an object-safe core trait plus an extension trait with a blanket impl (the `Iterator`/`IteratorExt` shape the ecosystem uses).
- Fix 3: replace the generic parameter with `&dyn` in the method signature.
- `async fn` in traits is `dyn`-incompatible for the same reason: it desugars to `-> impl Future`, and a generic return type has no vtable slot.

## Associated Types vs Type Parameters

- Associated type = **one** impl per implementing type (`Iterator::Item`). Callers write `T::Item` and never have to name it.
- Type parameter = **many** impls per type (`From<T>`, `AsRef<T>`). `String` implements `From<&str>` and `From<char>`; that would be impossible with an associated type.
- Choosing a parameter where an associated type belonged forces inference annotations on every caller; choosing an associated type where a parameter belonged blocks the second impl you will want in six months.

## Bounds That Behave Unexpectedly

- `#[derive(Clone)]` on `struct Wrapper<T>` generates `impl<T: Clone> Clone for Wrapper<T>` — even when the `T` is behind an `Arc` and never needs `Clone`. Hand-write the impl when the derived bound is wrong; this is a public-API decision, not a style one.
- Blanket std impls you get for free and should not duplicate: `Into` from `From`, `TryInto` from `TryFrom`, `ToString` from `Display`. Implementing `ToString` directly is a smell.
- `PartialEq`/`Eq`/`Hash` must agree: two values that are equal must hash equally, or `HashMap` silently loses entries. Deriving all three together is the safe path; hand-writing one of them obliges you to check the others.
- `Ord` must be a total order. Returning inconsistent orderings from `sort_by` triggers a panic on modern std rather than silent corruption, but the bug is yours.
- Auto traits (`Send`, `Sync`, `Unpin`, `UnwindSafe`) are implemented structurally — a single `Rc` field removes `Send` from the whole struct, and the error surfaces at the spawn site far away.

## Deref Is Not Inheritance

- `Deref` exists for smart pointers (`Box`, `Rc`, `Arc`, `String` → `str`, `Vec<T>` → `[T]`). Using it to fake inheritance between your own types produces methods that appear from nowhere and disappear when a name collides.
- Deref coercion applies to method receivers and to `&`-typed arguments, not to generic parameters: a function taking `T: AsRef<str>` will not accept `&String` coerced — it accepts it via the `AsRef` impl instead, which is why `AsRef` bounds beat `&str` parameters in generic APIs.
- Inherent methods win over trait methods, and the closest deref target wins over further ones. That resolution order is why adding an inherent method to your type can break downstream code that was calling a trait method of the same name.

## Sealed Traits

Prevent downstream impls while keeping the trait public:

```rust
mod private { pub trait Sealed {} }
pub trait Format: private::Sealed { fn write(&self) -> String; }
impl private::Sealed for Json {}
```

Use it when you must be free to add methods later without a breaking change, and for traits whose invariants you cannot check in downstream impls. Document that it is sealed — the compiler error alone is baffling.

## `Sized`, `?Sized`, and Unsized Values

- Every type parameter has an implicit `T: Sized`. `T: ?Sized` opts out and is what lets a function take `&T` where `T` is `str`, `[u8]`, or `dyn Trait`.
- Unsized types can only exist behind a pointer. The last field of a struct may be unsized (making the struct unsized too), which is how `Rc<str>` works.
- Writing `fn f<T: ?Sized>(x: &T)` costs nothing and widens the API; writing it on a `T` you actually store by value will not compile, which is the compiler telling you the bound was wrong.
