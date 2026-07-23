# Collections: Choice, Copying, Sorting, Iteration

## Structure Choice

| Need | Use | Why |
|---|---|---|
| user-controlled or non-string keys | Map (or `Object.create(null)`) | plain object: a `"__proto__"` key pollutes the prototype; all keys coerce to strings (`obj[{}]` → `"[object Object]"`) |
| frequent add/delete, need count | Map | `.size` is O(1); `Object.keys(o).length` walks every key |
| membership tests, dedupe | Set | `includes` is O(n): filtering 10k items against a 10k list = 100M comparisons; one Set build makes each lookup O(1) |
| metadata on objects you don't own | WeakMap | keys don't block GC — annotate/cache without leaks; intentionally not iterable |
| JSON, spread-heavy records | plain object | Map doesn't survive `JSON.stringify` (→ `{}`) |
| anything else (default) | array + object | switch to Map/Set only on a trigger above |

- Set/Map match keys by SameValueZero (→ coercion.md): object keys by reference — `{a:1} !== {a:1}` — and NaN works as a key.
- Set algebra (`union`, `intersection`, `difference`) exists natively; floor in `modern.md`.

## Copying

| Method | Depth | Keeps | Loses / throws |
|---|---|---|---|
| spread, `Object.assign` | 1 | — | nested refs shared; spread snapshots getter values |
| `structuredClone` | deep | cycles, Date, Map, Set, TypedArray | functions and DOM nodes THROW; class instances lose their prototype (become plain objects) |
| JSON round-trip | deep | plain data | undefined/functions/symbol keys dropped, Date → string, Map/Set → `{}`, cycles THROW |

Default `structuredClone`; use the JSON round-trip only when you also want to prove the value is serializable.

## Sorting

- Default comparator stringifies: `[10, 2, 1].sort()` → `[1, 10, 2]`. Numbers: `(a, b) => a - b`. Human strings: `localeCompare` with `{numeric: true}` (→ SKILL.md Strings & Unicode).
- Stable since ES2019: multi-key ordering = sort by secondary key first, then by primary — ties keep the secondary order.
- `undefined` elements always sort to the end; the comparator never sees them.
- An inconsistent comparator (random tiebreaks, non-transitive rules) yields implementation-defined garbage, not "roughly sorted".
- `sort`/`reverse`/`splice` mutate in place; `toSorted`/`toReversed`/`toSpliced`/`with` return copies (floors: `modern.md`).

## Holes & Deletion

- `Array(3)` creates holes: `map`/`forEach`/`filter` SKIP them; `join`/`fill` don't. Materialize with `Array.from({length: 3}, (_, i) => f(i))`.
- `delete arr[i]` leaves a hole → `splice(i, 1)` reindexes. `arr.length = 0` empties in place — every alias sees it.
- Removing elements inside `for...of` over the same array skips the following element — iterate a copy, walk backwards by index, or `filter`.

## Lookup & Iteration

- `find()` returning undefined is ambiguous (absent vs stored undefined) — `findIndex() === -1` disambiguates.
- `filter(Boolean)` also removes 0 and `""` — usually you meant `x => x != null`.
- `for...in` walks enumerable keys INCLUDING inherited ones — use `Object.keys/entries` for objects; `Reflect.ownKeys` for non-enumerables and symbols.
- `for...of` on a plain object throws (not iterable) — `Object.entries(obj)`.
- `at(-1)` for the last element; `findLast`/`findLastIndex` search from the end; `Object.groupBy`/`Map.groupBy` for bucketing (floors: `modern.md`).
