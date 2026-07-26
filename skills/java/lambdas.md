# Lambdas — Functional Interfaces, Method References, Capture

A lambda is an instance of a functional interface, not a block of code passed around. Every surprise below follows from that. Pipelines built out of them live in `streams.md`; async composition in `async.md`.

## The Built-in Interfaces

Declare your own only when no `java.util.function` shape fits or the domain name earns its keep (`PriceRule` beats `BiFunction<Order, Customer, BigDecimal>` in a signature a human reads).

| Shape | Interface | Method | Primitive specializations |
|---|---|---|---|
| `() -> T` | `Supplier<T>` | `get` | `IntSupplier`, `BooleanSupplier`, … |
| `T -> void` | `Consumer<T>` | `accept` | `IntConsumer`, `ObjIntConsumer<T>` |
| `T -> R` | `Function<T, R>` | `apply` | `ToIntFunction<T>`, `IntFunction<R>`, `IntUnaryOperator` |
| `(T, U) -> R` | `BiFunction<T, U, R>` | `apply` | `ToIntBiFunction<T, U>` |
| `T -> boolean` | `Predicate<T>` | `test` | `IntPredicate`, `DoublePredicate` |
| `T -> T` / `(T, T) -> T` | `UnaryOperator<T>` / `BinaryOperator<T>` | `apply` | `IntUnaryOperator`, `IntBinaryOperator` |
| `() -> void` | `Runnable` | `run` | — |
| `() -> V throws Exception` | `Callable<V>` | `call` | — |

- The primitive specializations exist to remove boxing from hot paths; in a pipeline that is `mapToInt` and friends (`streams.md`). Outside a hot path, `Function<Integer, Integer>` costs a boxed object per call and nothing else.
- `Callable<V>` is the only common built-in whose method declares `throws Exception`. When you design an API that takes a body which can fail, take a `Callable` rather than a `Supplier` and save every caller a wrapper.
- Overloads that differ only by functional interface break inference: `ExecutorService.submit(() -> save(x))` binds to `Callable` when `save` returns a value and to `Runnable` when it returns void — and the `Callable` branch parks the exception inside the `Future` where nobody looks. Cast the lambda (`submit((Runnable) () -> save(x))`) or call the value.

## Designing a @FunctionalInterface

- Exactly one abstract method. `default`, `static`, and `private` methods are unlimited, and re-declared `Object` methods (`equals`, `hashCode`, `toString`) do not count toward the single abstract method.
- The annotation is optional and worth writing anyway: it is the only compile-time guard that stops the next person adding a second abstract method and breaking every call site.
- A SAM with its own type parameters (`<T> T apply(T t)`) cannot be implemented by a lambda — lambdas are never generic. Method references and classes still work.
- Put `throws E` on your SAM if the body can fail; that single word is what lets checked exceptions cross the boundary (below).
- Name the parameters in the interface, not the lambda: the IDE shows `interface` names at the call site, which is most of the readability you get over `BiFunction`.

## Capture Semantics

- Locals are captured **by value** and must be final or effectively final; fields are read through the captured `this`, so they are live. A lambda in an instance method therefore pins the whole enclosing object — the small-listener-keeps-a-graph-alive leak (`classes.md`, `memory.md`).
- The enhanced-`for` variable is effectively final per iteration; a classic `for (int i = ...)` index is not, which is why the index loop is the one that will not compile.
- The mutable-box workarounds (`AtomicInteger`, `int[1]`) compile and are a signal you wanted a loop or a collector. In parallel code they are also a correctness bug (`streams.md`).
- Lambdas that capture no state are cached and allocate once; capturing lambdas allocate per call — relevant only in hot loops (`performance.md`).
- `this` inside a lambda is the **enclosing instance**; inside an anonymous class it is the anonymous object. Converting an anonymous class to a lambda silently changes the meaning of `this` and of any shadowed name (a lambda cannot shadow a local; an anonymous class can).
- Lambdas compile to `invokedynamic` linked by `LambdaMetafactory`, not to one class file each: no jar bloat, but a first-call linkage cost that shows up in startup-sensitive CLIs and serverless cold starts (`performance.md`).

## Method References

| Kind | Example | Equivalent |
|---|---|---|
| Static | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| Bound instance | `System.out::println`, `this::handle` | `x -> System.out.println(x)` |
| Unbound instance | `String::length` | `s -> s.length()` — the first parameter becomes the receiver |
| Constructor | `ArrayList::new`, `String[]::new` | `() -> new ArrayList<>()`, `n -> new String[n]` |

- **The receiver of a bound reference is evaluated once, at the moment the reference is created.** `getLogger()::info` calls `getLogger()` immediately; a null there is an NPE at the reference site, and a later reassignment of the field is not seen. `x -> getLogger().info(x)` is the lazy form.
- `this::method` and `super::method` are bound references: they capture `this` with all the lifetime consequences above.
- No partial application: a method reference cannot bind some arguments and leave others. Write the lambda.
- When overloads make `String::valueOf` ambiguous, the fix is a lambda with an explicitly typed parameter, not a cast.

## Checked Exceptions Through a Lambda

The compiler rejects the body because the target SAM does not declare `throws` (`exceptions.md`). Three honest answers, in order:

1. Use a loop. For a body that does I/O per element, the loop is usually the clearer artifact anyway.
2. Wrap at the source with a type that stays meaningful: `throw new UncheckedIOException(e)`.
3. Declare a throwing interface and adapt once, at one boundary:

```java
@FunctionalInterface
interface ThrowingFunction<T, R, E extends Exception> { R apply(T t) throws E; }

static <T, R> Function<T, R> unchecked(ThrowingFunction<T, R, ? extends Exception> f) {
    return t -> { try { return f.apply(t); } catch (Exception e) { throw new IllegalStateException(e); } };
}
```

- The adapter erases the checked type, so callers can no longer catch it specifically. Acceptable only where the boundary immediately rethrows a domain exception; not acceptable as a project-wide habit.
- Blacklist "sneaky throw" (`@SneakyThrows`, or the generic cast trick that rethrows a checked exception from a method that does not declare it). It compiles, and then no caller can catch the exception by type and no signature tells the truth.

## Composition and Common Shapes

- Order trips everyone: `f.andThen(g)` is `g(f(x))`; `f.compose(g)` is `f(g(x))`. Prefer `andThen` and read left to right.
- `Predicate.and`, `or`, `negate`, plus `Predicate.not(String::isBlank)` (11+) — `not` is the answer to "I cannot negate a method reference".
- `Function.identity()` is a non-capturing lambda, allocated once; semantically the same as `x -> x`.
- A `Map<Key, Function<In, Out>>` of strategies replaces a switch chain that two files must be updated to extend — and an enum with constant-specific bodies replaces both when the key set is closed (`classes.md`).
- Memoizing with `map.computeIfAbsent(k, this::compute)` is correct only for a non-recursive `compute`: a recursive one throws `IllegalStateException: Recursive update` on `ConcurrentHashMap` (9+) and can corrupt or hang a `HashMap` (`collections.md`).

## When Not To Use a Lambda

- Over about three lines: extract a named method and pass a method reference. Stack traces then read `OrderService.priceOf` instead of `lambda$process$3` (`debug.md`).
- When you must remove the listener you added. Each evaluation of a lambda expression may yield a distinct instance and lambdas have no meaningful `equals`, so `removeListener(e -> log(e))` removes nothing — keep the reference you registered.
- Recursion: a lambda cannot reference the variable it is being assigned to. Use a method, or a field assigned in two steps.
- When a profiler or an error budget needs the frame to have a name: lambda frames are synthetic and merge badly in aggregated stack traces.
