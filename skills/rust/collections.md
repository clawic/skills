# Collections — `Vec`, Maps, Sets, Queues, and Iterators

Choosing the container is a performance decision made once and a correctness decision made forever: iteration order, invalidation, and complexity all follow from it.

## Choosing a Container

| Need | Use | Why not the obvious one |
|---|---|---|
| A sequence, default | `Vec<T>` | Contiguous and cache-friendly; beats fancier structures far past the size intuition suggests |
| Fixed length known at compile time | `[T; N]` | No allocation, `Copy` when `T` is |
| Never grows after construction | `Box<[T]>` | One word smaller than `Vec` and it documents the intent |
| Push and pop at both ends | `VecDeque<T>` | `Vec::insert(0, _)` is O(n) per call |
| Key lookup | `HashMap<K, V>` | Unordered iteration, no range queries |
| Key lookup with ordered iteration or ranges | `BTreeMap<K, V>` | Slower per lookup, no hasher to tune, but `range(a..b)` exists |
| Membership only | `HashSet`/`BTreeSet` | A `HashMap<K, ()>` is the same thing spelled worse |
| Top-n, priority queue | `BinaryHeap<T>` | Max-heap; wrap in `Reverse` for a min-heap |
| Small collection keyed by a small enum | An array indexed by `as usize` | A `HashMap` with four keys is mostly hashing overhead |
| Anything else | `Vec` plus a linear scan | It wins below a few hundred elements more often than intuition says |

## `Vec` and Slices

- Growth is amortized doubling; each reallocation copies. `Vec::with_capacity(n)` removes that when `n` is known even approximately.
- `remove(i)` is O(n) (shifts the tail); `swap_remove(i)` is O(1) and reorders — correct whenever order does not matter.
- `retain(|x| ...)` filters in place in one pass; a manual index loop with `remove` is O(n²) and gets written surprisingly often.
- `drain(..)` yields owned items and leaves the allocation for reuse; `clear()` keeps capacity, `shrink_to_fit()` gives it back.
- `sort` is stable (allocates); `sort_unstable` is faster and allocation-free — use it unless equal elements must keep their order. `sort_by_key` recomputes the key on every comparison; `sort_by_cached_key` computes it once per element and wins as soon as the key is expensive.
- `binary_search` requires a sorted slice and returns `Result` — `Ok(i)` found, `Err(i)` the insertion point, which is exactly what you want for sorted inserts.
- `chunks`, `windows`, `split_first`, `split_at_mut` cover most manual index arithmetic, and they are bounds-checked once instead of per access.
- Indexing panics; `get(i)` returns `Option`. In library code that receives an index from a caller, `get` is the difference between an error and a crash in someone else's process.

## Maps and Sets

- `HashMap`'s default hasher (SipHash) is DoS-resistant and not the fastest. For internal maps with short keys and no untrusted input, `rustc-hash`/`ahash` are typically meaningfully faster — benchmark, and never swap the hasher on a map keyed by user-controlled data.
- Iteration order is unspecified and varies between runs. A test that depends on it passes locally and fails in CI.
- The `entry` API does one lookup instead of two: `*map.entry(k).or_insert(0) += 1`, `or_insert_with(Vec::new)` when the default allocates, `and_modify(...).or_insert(...)` for update-or-create.
- Keys must have consistent `Eq` and `Hash`: two values that are equal must hash equally, or lookups miss entries that are present. Mutating a key through interior mutability after insertion corrupts the map with no error.
- `HashMap<String, V>` can be looked up with `&str` because of the `Borrow` trait; a `HashMap<MyKey, V>` cannot unless you implement `Borrow` for it — that missing impl is why some lookups force an allocation.
- `BTreeMap` requires `Ord` and gives you `range(a..b)`, `first_key_value`, and `split_off` — the reason to accept its slower lookups.
- Sets compose: `intersection`, `difference`, `union`, `symmetric_difference` return iterators, so `.count()` costs nothing extra and `.cloned().collect()` is the only allocation.

## Iterators

- Lazy: nothing runs until a consuming adapter (`collect`, `sum`, `for`, `count`, `fold`, `any`) pulls. An iterator chain with no consumer is a no-op the `unused_must_use` lint will flag.
- `iter()` yields `&T`, `iter_mut()` yields `&mut T`, `into_iter()` yields `T` and consumes the collection. On arrays and `&Vec` the third one differs by edition — say which you mean.
- `collect::<Result<Vec<_>, E>>()` short-circuits at the first `Err`; `collect::<Vec<Result<_, E>>>()` keeps them all. The turbofish is the whole difference.
- `collect` also builds a `HashMap` from an iterator of pairs and a `String` from `char`s; the target type is always the thing to name first when inference complains.
- `filter_map` over `filter().map()` when the predicate and the transform share work; `flat_map` over `map().flatten()`.
- `zip` stops at the shorter side, silently. `enumerate` after `filter` numbers the survivors, not the originals — order the adapters deliberately.
- `cloned()` for `T: Clone`, `copied()` for `T: Copy`; `copied` is a compile-time assertion that the operation is free.
- `fold` accumulates, `try_fold` stops at the first error, `scan` yields intermediate states, `reduce` folds without an initial value and returns `Option`.
- `peekable` for lookahead, `by_ref` to consume part of an iterator and keep the rest. Chunking an iterator needs `itertools`; std has `chunks` only on slices, and `windows` has no iterator form at all because the yielded borrows would overlap.
- Iterator chains compile to the same code as the equivalent loop in release builds and additionally eliminate bounds checks that manual indexing keeps. Readability is the deciding factor, not speed.
- Writing your own: implement `Iterator::next` and get 70+ adapters free; add `size_hint` so `collect` can preallocate; implement `ExactSizeIterator`/`DoubleEndedIterator` only when they are genuinely exact and reversible.

## Cheat Table

| Task | Reach for |
|---|---|
| Count occurrences | `*map.entry(k).or_insert(0) += 1` |
| Deduplicate, keep order | `HashSet` for seen + `retain` |
| Deduplicate, sorted output | `sort_unstable` then `dedup` |
| Group by key | `map.entry(k).or_default().push(v)` |
| Top-n without full sort | `BinaryHeap` of size n, or `select_nth_unstable` |
| Split by predicate | `partition::<Vec<_>, _>(pred)` |
| Fixed-size ring buffer | `VecDeque` with a capacity check |
| Frequent front insertion | `VecDeque`, never `Vec::insert(0, _)` |
| Set operations | `HashSet::intersection`/`difference`/`union` |
| Anything else | A `Vec` and a linear scan; it wins below a few hundred elements more often than intuition says |

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `Vec::insert(0, x)` in a loop | O(n) per insert, O(n²) total | `VecDeque::push_front`, or push and reverse once |
| Iterating a map and asserting the order | Order is unspecified and changes between runs | `BTreeMap`, or sort the collected pairs |
| `contains_key` then `insert` | Two lookups, and often two borrows the checker refuses | `entry(k)` |
| `.iter().collect::<Vec<_>>()` to loop once | Allocates a vector to walk it once | Iterate the iterator directly |
| A fast hasher on a map keyed by request data | Turns hash-collision DoS back on | Fast hashers only for internal, non-attacker-controlled keys |
| `shrink_to_fit()` after every removal | Reallocates and copies, undoing amortized growth | Shrink once, when the collection is done growing |
