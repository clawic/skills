# Generics — Erasure, Variance, and the Warnings That Matter

## Erasure, and What It Costs

At runtime `List<String>` is just `List`. Consequences you cannot design around:

- `new T()` and `new T[n]` do not compile. Pass a factory (`Supplier<T>`) or a `Class<T>` token and call `clazz.getDeclaredConstructor().newInstance()`.
- `instanceof List<String>` does not compile; `instanceof List<?>` does and tells you nothing about the element type.
- Overloads cannot differ only by type argument: `f(List<String>)` and `f(List<Integer>)` have the same erasure and clash.
- A `catch (MyException<T> e)` is illegal — exceptions cannot be generic.
- Static fields are shared across all parameterizations: there is one `Box.count`, not one per `Box<String>`.
- The compiler inserts casts for you, so a broken generic type surfaces later as a `ClassCastException` in code you did not write (a "heap pollution" report).

Erasure is also why generics are free at runtime: no reification, no per-type code, no boxing beyond what you write.

## Variance: PECS

`List<Dog>` is **not** a `List<Animal>` — if it were, you could put a `Cat` into it. Wildcards restore the flexibility safely.

- **Producer Extends**: `List<? extends Animal>` — you can READ `Animal`s, you cannot add anything (except `null`), because the actual list might be `List<Dog>`.
- **Consumer Super**: `List<? super Dog>` — you can ADD `Dog`s, reads give you `Object`.
- Mnemonic in a signature: `void copy(List<? super T> dst, List<? extends T> src)`.
- `Collections.max(Collection<? extends T>, Comparator<? super T>)` is the canonical shape: read the elements, compare against a comparator that may handle supertypes.
- `<?>` (unbounded) is not `<Object>`: `List<?>` accepts a `List<String>`, `List<Object>` does not.
- Wildcards belong in **parameters**, not return types: `List<? extends Foo>` returned forces every caller to deal with the wildcard.

## Bounds and Type Parameters

- `<T extends Comparable<? super T>>` is the correct bound for "sortable T" — the `super` allows a class whose parent implements `Comparable`.
- Multiple bounds: `<T extends Number & Comparable<T>>`; the class bound, if any, must come first.
- Recursive bound (`<T extends Builder<T>>`) is how a fluent builder returns the subtype from inherited methods.
- A method type parameter is inferred from arguments; when it cannot be, supply it explicitly: `Collections.<String>emptyList()`.
- `var` interacts badly with generic inference in a chain — the inferred type may be a capture or intersection type you cannot name. Annotate the variable when the compiler complains about something unpronounceable.

## Arrays vs Generics

- Arrays are covariant and reified: `Object[] a = new String[1]; a[0] = 1;` compiles and throws `ArrayStoreException` at runtime. Generics are invariant and erased: the same mistake is a compile error.
- `new T[10]` is illegal; the standard workaround inside a generic class is `(T[]) new Object[10]` with a documented `@SuppressWarnings("unchecked")` and the array kept private. Never return it as `T[]` — the caller's implicit cast fails.
- `list.toArray(new String[0])` is the safe conversion (`collections.md`).
- Never mix generics and arrays in a public API. Return `List<T>`.

## Warnings You Must Not Ignore

- **Raw types** (`List` with no argument) disable ALL generic checking for that expression, including unrelated type arguments — one raw usage can silently un-type a whole call chain. They exist only for pre-2004 compatibility.
- `unchecked cast` means the compiler cannot prove the cast; if you keep it, add `@SuppressWarnings("unchecked")` on the **smallest possible scope** with a comment explaining why it holds.
- `@SafeVarargs` on a `final`/`static`/`private` method declares that you do not store or expose the generic varargs array. Adding it to a method that DOES leak the array converts a compile warning into a runtime `ClassCastException`.
- Build with `-Xlint:all` (and `-Werror` where the team can sustain it). Unchecked warnings are the only signal that erasure is about to bite.

## Getting Type Information Back

- **Class token**: pass `Class<T>` and use `clazz.cast(o)` — the `Map<Class<?>, Object>` heterogeneous container pattern is type-safe if writes go through `cast`.
- **Super type token**: a generic supertype's type argument IS retained in the class file, which is why `new TypeReference<List<Foo>>() {}` (Jackson) and `new TypeToken<List<Foo>>() {}` (Gson/Guava) work. This is the only reliable way to deserialize a generic type (`serialization.md`).
- Field, parameter, and return types keep their generic signature and are readable via `getGenericType()`/`getGenericReturnType()`; the *values* never carry it.
- Do not try to recover the type argument of a local variable at runtime. It is genuinely gone.
