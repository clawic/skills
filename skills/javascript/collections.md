# Collection Traps

- `sort()` mutates the original: and without a comparator it's lexicographic: [10, 2, 1]
- `reverse()` mutates: use `toReversed()` (ES2023) for a copy
- `splice()` mutates AND returns the removed items: return confusion
- `find()` returns undefined: same as an undefined element in the array
- `indexOf()` with NaN returns -1: NaN !== NaN, use `includes()`
- `filter(Boolean)` removes falsy: including the 0 and "" you wanted
- `[...array]` is shallow: nested objects share a reference
- `structuredClone()` is deep but doesn't clone functions, DOM nodes
- JSON parse/stringify loses undefined, functions, Dates
- `for...in` includes inherited keys: use `Object.keys()` or `for...of`
- `delete obj.prop` is slow: assign undefined if it doesn't matter
- `obj[key]` with an object key: gets converted to "[object Object]"
- A Set of objects compares by reference: {a:1} !== {a:1}
