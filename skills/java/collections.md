# Collections — Choosing, Comparing, Iterating

## Choosing the Implementation

| Need | Use | Note |
|---|---|---|
| Ordered list, index access, iteration | `ArrayList` | The default. Grows by ~1.5×; presize when you know the size |
| Frequent insert/remove at both ends | `ArrayDeque` | Beats `LinkedList` at everything except `List` semantics; `LinkedList`'s only real advantage is `ListIterator` mid-list removal |
| Unique elements, no order | `HashSet` | Requires a correct `hashCode` |
| Unique elements, insertion order | `LinkedHashSet` | The predictable choice for output that humans read |
| Sorted set/map, range queries | `TreeSet` / `TreeMap` | `O(log n)`; needs `Comparable` or a `Comparator`, and comparison must agree with `equals` |
| Key → value, no order | `HashMap` | Default capacity 16, load factor 0.75 → it resizes at 12 entries. Presize with `new HashMap<>(expected / 0.75f + 1)` |
| Key → value, insertion or access order | `LinkedHashMap` | The access-order constructor plus `removeEldestEntry` is a 5-line LRU cache |
| Enum keys | `EnumMap` / `EnumSet` | Array-backed, faster and smaller than `HashMap`; `EnumSet` is a bitset |
| Concurrent access | `ConcurrentHashMap`, `CopyOnWriteArrayList` | COW copies on every write: read-mostly listener lists only (`concurrency.md`) |
| Anything else | `ArrayList` or `HashMap` | Optimize when a profile says so, not before |

- `HashMap` degrades gracefully: a bucket with 8 colliding entries becomes a red-black tree, but only once the table has ≥64 slots — below that it resizes instead. A bad `hashCode` still costs you, it just does not become quadratic.
- Sizing beats swapping implementations: a presized `ArrayList` or `HashMap` avoids repeated array copies that dominate small-collection benchmarks (`performance.md`).

## The Equality Contract

- `equals` must be reflexive, symmetric, transitive, consistent, and `x.equals(null)` must be false. `hashCode` must be equal for equal objects; unequal objects may collide.
- Never mutate a field used by `hashCode` after inserting into a hash container — the entry becomes unreachable (SKILL.md rule 2).
- `getClass() != o.getClass()` for value types; `instanceof` only when a subclass is deliberately equal to its parent — symmetry breaks otherwise, and a `List` containing both sees different results depending on iteration order.
- `Objects.equals(a, b)` and `Objects.hash(a, b, c)` for the boilerplate. `Objects.hash` allocates a varargs array — write the manual `31 * result + field` form only in a profiled hot path.
- Records generate a correct `equals`/`hashCode` over all components. Prefer them for value types (`classes.md`).
- Arrays do not override `equals`: `a.equals(b)` is identity. Use `Arrays.equals` (1-D) or `Arrays.deepEquals` (nested), and `Arrays.hashCode`/`deepHashCode` to match.
- `TreeMap`/`TreeSet` use `compareTo`, not `equals`. If they disagree (classic: `BigDecimal("2.0")` vs `BigDecimal("2.00")`), the same object can be "present" in a `TreeSet` and "absent" in a `HashSet`.

## Immutable, Unmodifiable, and Fixed-Size

| Factory | Mutable? | Nulls? | Trap |
|---|---|---|---|
| `List.of(...)` / `Map.of(...)` (9+) | No | Rejects null elements AND `contains(null)` throws | Set/Map iteration order is randomized per JVM run — code that depends on it fails one deploy in ten |
| `Arrays.asList(a, b)` | Fixed size; `set` writes through to the array | Allows nulls | `add`/`remove` throw `UnsupportedOperationException` |
| `Collections.unmodifiableList(l)` | A read-only VIEW | Allows nulls | The backing list can still be modified, and the view changes with it |
| `List.copyOf(l)` (10+) | No | Rejects nulls | A real defensive copy — the right return value from a getter |
| `stream().toList()` (16+) | No | Allows nulls | Differs from `Collectors.toUnmodifiableList()`, which rejects nulls |
| `Collectors.toList()` | Yes (currently `ArrayList`) | Allows nulls | The mutability is unspecified — do not rely on it |

- `Arrays.asList(intArray)` where `intArray` is `int[]` gives a `List<int[]>` of size 1, not a list of ints. Use `Arrays.stream(intArray).boxed().toList()`.
- `subList` is a view: structurally modifying the parent invalidates it, and modifying the sublist writes through to the parent.

## Iteration Traps

- `ConcurrentModificationException` is a fail-fast heuristic, not a guarantee, and it is usually single-threaded: you removed from the collection while a for-each iterator was open. Fixes: `iterator.remove()`, `list.removeIf(pred)`, or iterate a copy.
- `removeIf` is the correct answer for conditional removal on any `Collection` and is `O(n)` on `ArrayList`, unlike a loop of `remove(index)`.
- Removing from an `ArrayList` inside an index loop skips elements: after removing index `i`, everything shifts left. Iterate backwards or use `removeIf`.
- Modifying a `HashMap` value in place is fine; adding or removing keys during iteration is not. Use `entry.setValue(...)`, `map.replaceAll(...)`, or `map.entrySet().removeIf(...)`.
- `map.keySet()`, `values()`, and `entrySet()` are views: `map.keySet().remove(k)` removes the entry from the map.
- `Entry` objects from `entrySet()` are only valid during the iteration for some implementations — do not hoard them.

## Maps Beyond get/put

- `getOrDefault(k, d)` reads a default without inserting; `computeIfAbsent(k, f)` inserts and returns — the difference matters when the map is a cache.
- `merge(k, 1, Integer::sum)` is the idiomatic counter. `computeIfAbsent(k, x -> new ArrayList<>()).add(v)` is the idiomatic multimap.
- `compute*` and `merge` remove the entry when the function returns `null` — deliberate, and surprising the first time.
- `HashMap` permits one null key and null values; `ConcurrentHashMap` and `TreeMap` (with natural ordering) reject nulls; `Hashtable` rejects both and should not appear in new code.
- `map.get(k) == null` is ambiguous between "absent" and "mapped to null" — `containsKey` disambiguates.

## Sorting and Comparators

- `Comparator.comparing(Foo::getAge).thenComparing(Foo::getName).reversed()` — note `reversed()` applies to the WHOLE chain built so far, which is rarely what people mean; reverse individual keys with `Comparator.comparing(f, Comparator.reverseOrder())`.
- Use `comparingInt`/`comparingLong`/`comparingDouble` for primitive keys: `comparing` boxes every element on every comparison.
- Subtraction comparators (`(a, b) -> a.value - b.value`) overflow for large or negative values and silently invert the order. Use `Integer.compare(a, b)`.
- A comparator inconsistent with itself (non-transitive, or not antisymmetric) throws `IllegalArgumentException: Comparison method violates its general contract!` — from TimSort, only on inputs large enough to trigger a merge, so it appears in production and not in tests.
- `Collections.sort`/`List.sort` are stable; `Arrays.sort` on primitives is not stable (dual-pivot quicksort) but stability is meaningless for primitives.
- `nullsFirst`/`nullsLast` wrap a comparator to tolerate nulls instead of throwing NPE from inside the sort.

## Arrays and Conversion

- `list.toArray(new String[0])` is the recommended form: the JIT optimizes the zero-length allocation, and a presized array can be measurably slower (Shipilëv's benchmarks).
- `Arrays.fill(new Object[n], someList)` puts the SAME reference in every slot; the same trap applies to `Collections.nCopies` and to `new ArrayList[n]` of shared lists.
- 2-D arrays are arrays of arrays: rows can differ in length, and `clone()` is shallow — the rows are shared.
- `System.arraycopy` and `Arrays.copyOfRange` beat any manual loop and are intrinsified.
