# Nulls, Optional, and Boxing

Design goal: make null impossible in most of the code, and explicit in the rest. NPE frequency is a design property, not bad luck.

## Eliminating Null by Construction

- Validate at the boundary and never again: `this.name = Objects.requireNonNull(name, "name")` in the constructor makes every downstream method null-free by contract.
- Return empty collections, never null. `Collections.emptyList()` costs nothing and deletes an entire class of caller bug. Same for empty strings and empty arrays.
- Null as a sentinel for "not found" is a design smell in a public API: return `Optional` (query) or throw (invariant violation).
- A three-state boolean (`Boolean` that can be null) is a hidden enum. Name the third state.
- `Map.getOrDefault` / `merge` remove most null-checking around maps (`collections.md`).

## Optional, Precisely

- Return type only. Not a field (not serializable, extra allocation, two empty states), not a parameter (the caller must build an `Optional` to say "nothing"), not a collection element (an empty collection already means that).
- `orElse(x)` evaluates `x` every call; `orElseGet(() -> x)` only when empty (SKILL.md rule 6). Reserve `orElse` for constants.
- `orElseThrow()` (10+) with no argument replaces `get()` and reads as intent. `get()` is deprecated in spirit if not in signature.
- `map` returning null yields an empty `Optional` — convenient, and a silent way to lose a bug.
- `flatMap` when the function itself returns an `Optional`; otherwise you get `Optional<Optional<T>>`.
- `Optional.ofNullable(x).map(...).orElse(default)` is the whole idiom for "transform if present". A chain of `isPresent()`/`get()` is the null check you were trying to avoid, with more syntax.
- `stream()` (9+) turns `Optional` into a 0-or-1 stream: `list.stream().map(this::find).flatMap(Optional::stream)` keeps only the hits.
- `ifPresentOrElse(action, emptyAction)` (9+) removes the last common `if`.
- `Optional` is not free: it allocates. Do not put it in a hot loop or an entity (`performance.md`).

## Nullability Annotations

- They document intent for tools; the JVM enforces nothing at runtime. Value comes from a checker running in the build (NullAway, ErrorProne, the IDE's inspector), not from the annotation itself.
- Pick ONE flavour and use it everywhere: JSpecify (`org.jspecify.annotations`) is the modern cross-tool choice; the JetBrains and Jakarta variants exist mostly for legacy reasons. Mixed flavours are worse than none because each tool ignores the others (`nullability_style` in SKILL.md Configuration).
- Default-non-null at the package level plus explicit `@Nullable` is far less annotation noise than the reverse.
- Kotlin callers see your annotations as platform-type information — annotating an API used from Kotlin has real, immediate value.

## Boxing and Unboxing

- Unboxing null throws NPE at the point of use, not at assignment: `map.get(missing)` returning `Integer` into an `int` throws with a stack trace pointing at arithmetic, not at the lookup.
- The autobox cache is −128..127 for `Integer`, and equivalent small ranges for `Byte`, `Short`, `Long`, and `Character`. `==` on boxed values is therefore right in tests and wrong in production (SKILL.md rule 1). `Boolean` and small `Character` values are always cached, which makes the bug even more intermittent.
- Ternaries mix types silently: `cond ? 1 : nullableInteger` unboxes the `Integer` because the expression type is `int` — an NPE where nothing appears to dereference.
- `Integer.valueOf(x)` uses the cache; the `new Integer(x)` constructors are deprecated for removal and always allocate.
- Boxing in a loop is the most common accidental allocation in Java: `Map<String, Integer>` counters, `List<Double>` accumulators, `Comparator.comparing` with a numeric key. Use primitive streams, primitive arrays, and `comparingInt` (`collections.md`).
- `Double.NaN != Double.NaN`, but `Double.valueOf(NaN).equals(Double.valueOf(NaN))` is true, and `-0.0 == 0.0` while `Double.compare(-0.0, 0.0) < 0`. Sorting and equality therefore disagree for doubles — matters in `TreeSet` and grouping keys.

## Reading a NullPointerException

- Since JDK 15, the message names the exact expression: "Cannot invoke `String.length()` because the return value of `Map.get(Object)` is null" (`debug.md`).
- No stack trace at all = the JIT swapped in a preallocated exception at a hot throw site; rerun with `-XX:-OmitStackTraceInFastThrow` (SKILL.md Exception Triage).
- NPE from a line with no visible dereference: unboxing, or an implicit `toString`/`switch` on a null.
- `switch` on a null `String` or enum throws NPE before JDK 21; from 21 a `case null` label is allowed in pattern switches, and a switch without it still throws.

## Defensive Style That Pays

- `"literal".equals(variable)` — or better, `Objects.equals(variable, "literal")`, which reads in the natural order and handles both sides.
- `Objects.requireNonNullElse(a, b)` (9+) instead of a ternary.
- `Objects.toString(o, "")` for a null-safe string conversion; `String.valueOf(null)` is ambiguous and can even fail to compile.
- Do not add null checks everywhere "just in case": a null that cannot legally occur should crash loudly at the boundary, not be silently absorbed three layers deep.
