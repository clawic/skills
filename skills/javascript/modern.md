# Modern JS: Feature Floors, Syntax Semantics, Modules, Classes

## Feature Floors (canonical table — other files point here)

| Feature | Spec | Node |
|---|---|---|
| `?.`, `??` | ES2020 | >=14 |
| `&&=`, `\|\|=`, `??=` | ES2021 | >=15 |
| `at()`, `Object.hasOwn`, Error `cause`, `#private` + `#x in obj`, static blocks | ES2022 | >=16.11 |
| top-level await | ES2022 | >=14.8 (modules only) |
| `structuredClone` | WHATWG | >=17 |
| `AbortSignal.timeout` | WHATWG | >=17.3 |
| `AbortSignal.any` | WHATWG | >=20 |
| `toSorted`/`toReversed`/`toSpliced`/`with`, `findLast` | ES2023 | >=20 |
| `import.meta.dirname`/`filename` | Node API | >=20.11 |
| `Object.groupBy`, `Map.groupBy` | ES2024 | >=21 |
| `Promise.withResolvers`, `Array.fromAsync` | ES2024 | >=22 |
| Set `union`/`intersection`/`difference`, iterator helpers (`.map`/`.filter`/`.take` on iterators) | ES2025 | >=22 |

A missing floor is a runtime TypeError in production, not a build error — match this table against your lowest supported runtime before using anything below ES2022.

## Optional Chaining Semantics

- `obj.method?.()` guards `method`; `obj?.method()` guards `obj`. Different nulls, different fixes.
- `a?.b.c`: if `a` is nullish the whole expression short-circuits to undefined; but if `a.b` is nullish, `.c` THROWS. One `?.` protects only its own link.
- `?.` does not guard callability: `x.fn?.()` with `fn = 3` throws (→ SKILL.md Traps).
- `delete a?.b` is legal; `a?.b = 1` is a SyntaxError — you cannot assign through `?.`.

## Defaults & Destructuring

- Defaults fire on undefined ONLY: `f(null)` keeps null; `({a = 1} = {a: null})` → `a` is null. Nullish-safe: apply `?? fallback` after destructuring.
- Nested destructuring of null throws: `{a: {b}} = {a: null}` — and an `= {}` default does not save you, it only covers undefined. Destructure one level, then `a?.b`.

## Classes

- Field initializers run per instance in definition order. `field = () => ...` creates one closure per instance: costs memory, lives off the prototype, so spies/mixins/`super.method()` dispatch never see it. Prototype methods unless you specifically need bound `this`.
- `this` before `super()` in a subclass constructor → ReferenceError. Class declarations are in the TDZ (no use-before-declare). Class bodies are always strict mode.
- `#private` is enforced by the language (unlike `_convention`). Brand-check membership with `#x in obj` instead of try/catch.

## Modules (ESM/CJS)

- ESM exports are live bindings; CJS exports are an object copied at `require` time. Circular ESM → TDZ ReferenceError at first access; circular CJS → silently partial object. Fix both the same way: extract the shared piece into a third module.
- `import * as ns` is frozen — patching for tests needs dependency injection or a mocking loader, not assignment.
- No `__dirname`/`__filename` in ESM: `import.meta.dirname` (floor above) or `fileURLToPath(import.meta.url)`.
- Dual-package hazard: one package resolved once as CJS and once as ESM loads TWICE — singletons split and `instanceof` fails across the boundary. Symptom: "x is not an instance of X" with identical class names.
- `require()` of synchronous ESM works on Node >=22.12; below that, dynamic `import()` is the only bridge from CJS.
- Modules are singletons per resolved URL: a `?v=2` query yields a fresh copy — deliberate cache-bust in dev, memory leak if abused.
- Top-level await blocks every importer until it settles; a rejection fails the whole graph (→ async.md).
- ESM files are always strict mode — `"use strict"` only matters in legacy scripts.
