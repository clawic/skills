# Streams — Pipelines, Collectors, and When Not To

Streams express transformations. A `for` loop that mutates something is clearer than a stream that mutates something — and `forEach` with a side effect is a loop wearing a costume. CompletableFuture lives in `async.md`.

## Mechanics That Explain Most Surprises

- A stream is lazy: nothing runs until the terminal operation. A pipeline with no terminal operation executes zero times, silently.
- A stream is single-use. Reusing one throws `IllegalStateException: stream has already been operated upon or closed`; build a new one from the source, or store the source, not the stream.
- Elements flow one at a time through the whole pipeline (except for stateful operations), so `filter` before `map` does less work — and `sorted` and `distinct` must buffer everything, which is why they break short-circuiting on infinite streams.
- `peek` exists for debugging and may be skipped entirely: with `findFirst` or `count` (which can elide the pipeline when the size is known), your peek never runs. Never put logic in it.
- Streams from I/O sources hold resources: `Files.lines`, `Files.walk`, and `Files.list` are `AutoCloseable` and must be in try-with-resources (SKILL.md rule 4).

## Collectors Worth Knowing Cold

| Goal | Collector |
|---|---|
| List/Set | `toList()` (16+, unmodifiable) or `Collectors.toCollection(ArrayList::new)` when you need mutability |
| Map | `toMap(k, v, merge)` — **always** pass the merge function (SKILL.md Traps) |
| Group | `groupingBy(Foo::type)` → `Map<Type, List<Foo>>` |
| Group and aggregate | `groupingBy(Foo::type, counting())`, `groupingBy(Foo::type, summingInt(Foo::qty))`, `groupingBy(Foo::type, mapping(Foo::name, toList()))` |
| Group into a chosen map type | `groupingBy(Foo::type, TreeMap::new, toList())` |
| Split in two | `partitioningBy(pred)` → `Map<Boolean, List<Foo>>`, and **both keys always exist** (unlike `groupingBy`) |
| Join text | `joining(", ", "[", "]")` |
| Stats in one pass | `summarizingInt(Foo::qty)` → count, sum, min, average, max |
| Reduce to one field | `reducing`, or better `mapToInt(...).sum()` |

- `toMap` throws NPE on a null **value** even when a merge function is supplied — `groupingBy` throws NPE on a null **key**. Filter nulls before collecting, or use `Collectors.toMap` into a `HashMap` supplier and accept the null handling you write yourself.
- `Collectors.toMap` with a merge function returns a `HashMap`: unordered. For insertion order pass `LinkedHashMap::new` as the fourth argument.
- Downstream collectors compose: `groupingBy(a, groupingBy(b, counting()))` builds a two-level map in one pass.
- `teeing` (12+) runs two collectors over one stream and merges the results — the one-pass answer to "average and max together".

## Primitive Streams

- `mapToInt`, `mapToLong`, `mapToDouble` remove boxing from the pipeline; `boxed()` puts it back when you need objects.
- `IntStream.range(0, n)` is the idiomatic index loop; `rangeClosed` includes the end.
- `average()` returns `OptionalDouble`, `max()` returns `OptionalInt` — empty input has no answer, and the compiler makes you decide.
- `sum()` on `IntStream` returns `int` and overflows silently; use `mapToLong(...).sum()` for counts that can exceed 2^31.
- `Collectors.averagingInt` returns a `Double` (0.0 for an empty stream) while `IntStream.average()` returns empty — two different answers to the same question; pick one and be consistent.

## Parallel Streams

Three conditions, all required (SKILL.md rule 9): real per-element work, an evenly splittable source, no shared mutable state.

- Splittability by source: arrays and `ArrayList` split perfectly; `HashSet`/`HashMap` acceptably; `LinkedList`, `Files.lines`, `Stream.iterate`, and `BufferedReader.lines` split badly or not at all.
- Everything runs on `ForkJoinPool.commonPool`, shared JVM-wide. One blocking task in a parallel stream stalls unrelated parallel work everywhere in the process. If you must control the pool, submit the terminal operation from inside your own `ForkJoinPool` — an undocumented-but-real behavior, and a reason to prefer an executor and plain futures instead (`async.md`).
- `forEach` in parallel does not preserve encounter order; `forEachOrdered` does, at the cost of the parallelism you were buying.
- `reduce(identity, acc)` requires a true identity (`acc(identity, x) == x`) and an associative accumulator, or parallel results differ from sequential ones — and the sequential test passes.
- Rough threshold from practice: below a few thousand elements of trivial work, parallel is slower than sequential because of split and merge overhead. Benchmark before believing any number, including this one (`performance.md`).

## Common Rewrites

- Loop with `if` + `add` → `filter().toList()`.
- Nested loops over a collection of collections → `flatMap(List::stream)`.
- Manual `Map<K, List<V>>` building → `groupingBy`.
- Manual counter map → `groupingBy(f, counting())` or `merge(k, 1L, Long::sum)`.
- `stream().forEach(list::add)` → `toList()`, or just a loop.
- `stream().collect(toList()).get(0)` → `findFirst().orElseThrow()`.

## When Not To Use a Stream

- The body needs to `break` on a condition other than a simple predicate, `continue` in a complex shape, or mutate two accumulators at once.
- The body throws a checked exception. Lambdas cannot declare them, and wrapping everything in `RuntimeException` to satisfy the compiler makes the code worse than the loop (`exceptions.md`).
- The pipeline exceeds about five operations or needs comments to be read — the loop is now the clearer artifact.
- You need the index. `IntStream.range` plus `list.get(i)` works but is less readable than the loop it replaces.
- Debugging matters more than elegance: stack traces through stream frames are long and generic, and stepping through a pipeline is painful (`debug.md`).
