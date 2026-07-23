---
name: typescript
slug: typescript
version: 1.0.4
description: >-
  Writes type-safe TypeScript: narrowing, generics, inference control, and strict tsconfig setup.
  Use for type errors, eliminating any, discriminated unions, .d.ts files, or migrating JavaScript to TypeScript.
homepage: https://clawic.com/skills/typescript
changelog: Deeper type patterns and migration guidance
metadata:
  clawdbot:
    emoji: 🔷
    displayName: TypeScript
---

## When To Use

- Decoding type errors: narrowing failures, inference surprises, "not assignable" walls
- Designing types for an API surface: generics, discriminated unions, utility types
- tsconfig decisions: strict flags, module resolution, JS-to-TS migration order
- Writing or fixing declaration files (`.d.ts`, module augmentation, publishing types)
- Not for runtime JavaScript semantics (closures, event loop, coercion) — that is the javascript skill

## Quick Reference

| Situation | Play |
|---|---|
| Value of unknown shape (API, `JSON.parse`, `catch`) | `unknown` + narrow before use; never `any` |
| Config/lookup object losing literal types | `satisfies` (TS >=4.9), not a type annotation |
| Union variant not narrowing | Add a literal discriminant field + exhaustive `never` check |
| 40-line "not assignable" error | Read the deepest `Types of property ... are incompatible` line first — the root cause is at the bottom, the top lines are wrappers |
| Generic won't infer / variance confusion | `generics.md` |
| `Partial`/`Omit`/`Record` behaving oddly | `utility-types.md` |
| Untyped npm package, globals, augmentation | `declarations.md` |
| JS conversion, strict-flag rollout, `as` debt | `migration.md` |
| External data crossing a trust boundary | Validate at runtime (schema lib) at the boundary; static types only downstream |
| Anything else typed | Default: full `strict`, infer locals, annotate exports |

## Stop Using `any`

- `any` is viral both directions: it silences errors on reads AND poisons everything it's assigned to. `unknown` only blocks reads until you narrow — same flexibility, none of the spread
- `catch (e)` is `unknown` under strict since TS 4.4 (`useUnknownInCatchVariables`) — check `e instanceof Error` before `.message`
- Every remaining cast (`as`) gets either a runtime guard above it or a one-line comment saying why it's safe — an uncommented cast is a deferred bug report
- Ratchet, don't boil the ocean: count `any`s (typescript-eslint `no-explicit-any` + `no-unsafe-*`), record the baseline, fail CI when the count rises

## Inference & Widening

Widening:

- `let x = "hello"` is `string`; `const x = "hello"` is `"hello"` — mutability drives widening
- Object and array literals widen their members even under `const`: `const cfg = { mode: "dark" }` has `mode: string` — `as const` freezes the whole tree (readonly + literals)
- Literal arguments: primitives infer as literals, objects/arrays widen — `<const T>` type parameters (TS >=5.0) preserve them at the call site
- Function return types widen too — annotate the return when a caller needs the literal

Inference limits:

- Inference flows from arguments to parameters, not through intermediate generics — when a nested generic fails, name the intermediate type and split into two steps
- No partial type argument inference: `fn<T, U>` with one explicit argument turns off inference for the rest — use the curried two-call pattern `make<T>()(config)` to fix one and infer the other
- `NoInfer<T>` (TS >=5.4) excludes a position from inference: `declare function set<T>(vals: T[], initial: NoInfer<T>): void` — without it, a wrong `initial` widens `T` instead of erroring
- Callback parameters get contextual types only when the expected signature is known — assigning the function to a typed variable or parameter is what enables `(x) =>` without annotation

## Discriminated Unions & Narrowing

Modeling with unions:

- Add a literal `kind` field to each variant — narrowing and exhaustive switches come free
- Exhaustiveness: `default: x satisfies never` (TS >=4.9) or `const _: never = x` — compile error when a variant is added
- Never model states as optional-field soup: `{ data?: T; error?: E }` admits 4 combinations of which 2 are illegal; a 2-variant union encodes exactly the legal ones
- Discriminant must be a literal type — a field typed `string` discriminates nothing; `as const` the source values

Narrowing failures:

- `filter(Boolean)` doesn't narrow in any TS version — write `.filter((x): x is T => Boolean(x))`. TS >=5.5 infers predicates from simple lambdas, so `.filter(x => x != null)` narrows there; below 5.5 it doesn't
- Narrowing dies inside callbacks: after `if (x)`, `arr.map(() => x.foo)` re-errors because TS assumes `x` may be reassigned — copy to a `const` before the callback
- `typeof x === "object"` includes `null` — check `x !== null` in the same condition
- `Object.keys(obj)` returns `string[]`, not `keyof typeof obj` — intentional: structural types can carry extra keys. Cast only when you own the object literal
- `Array.isArray()` on `unknown` narrows to `any[]` — element type needs its own check
- `in` narrows unions only when the property appears in exactly one branch; on unlisted properties it narrows since TS 4.9
- Destructured discriminants (`const { kind } = action; if (kind === ...)`) narrow only in TS >=4.6 — on older versions switch on `action.kind` directly

## `satisfies` vs Annotation vs Cast

Three tools, three guarantees — pick by what you need to keep:

- `const x: T = v` — checks AND widens to `T`: literal info gone, excess properties rejected
- `const x = v satisfies T` (TS >=4.9) — checks compatibility, keeps the inferred narrow type: the default for config/route/theme objects
- `v as T` — checks nothing beyond rough overlap; `"hello" as number` fails but `{} as User` compiles. It's an assertion, not a conversion

## Strict Null Handling

- `??` vs `||`: `port || 3000` turns a legitimate `0` into 3000; `port ?? 3000` only replaces `null`/`undefined`. Same for `""` in string options
- Optional chaining `?.` produces `undefined`, never `null` — APIs contracted to `null` need an explicit `?? null`
- Non-null `!` is a cast in disguise — prefer early return or narrowing; each `!` is a runtime crash candidate the compiler can no longer see

## Compiler Config Beyond `strict`

`strict: true` is not the strictest configuration. Flags it does NOT include:

- `noUncheckedIndexedAccess` (TS >=4.1) — `arr[i]` and `record[key]` become `T | undefined`. Catches the most common real crash (indexing off the end); enable on greenfield day one, on legacy expect a large error wave and stage it
- `exactOptionalPropertyTypes` (TS >=4.4) — distinguishes "absent" from "explicitly undefined"; breaks code that assigns `undefined` to optional props
- `noPropertyAccessFromIndexSignature`, `noImplicitOverride` — cheap to adopt, low churn
- `verbatimModuleSyntax` (TS >=5.0) — forces `import type` for type-only imports; the modern replacement for `isolatedModules` import hygiene

Module boundaries:

- `import type { X }` / `export type { X }` — erased at compile time, so bundlers and transpile-only tools (esbuild, Babel) never chase a phantom runtime dependency
- `moduleResolution: "bundler"` (TS >=5.0) for apps built by a bundler; `"nodenext"` for packages run by Node — nodenext ESM requires explicit `.js` extensions on relative imports, bundler doesn't
- Prefer `as const` objects + `keyof typeof` over `enum`: `const enum` breaks under transpile-only compilers, and plain enums are the one TS feature with runtime emit. TS >=5.8 `erasableSyntaxOnly` bans them outright for Node type-stripping compatibility

## Where To Annotate

- Default: annotate exported function signatures (parameters AND return), infer everything inside bodies — errors then surface at the boundary that caused them, not at 30 call sites
- Explicit return types on the public surface also unlock `isolatedDeclarations` (TS >=5.5) and faster declaration emit
- A type parameter must relate at least two positions (two params, or param and return). Used once, it's decoration — replace with `unknown` or the concrete type
- Type-level performance: name intermediate types instead of inline chains, and prefer `interface extends` over long `&` intersections — interfaces cache relationships, intersections are recomputed. Diagnose with `tsc --extendedDiagnostics` (check time), then `--generateTrace`

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/typescript (install if the user confirms):

- **javascript** — runtime semantics: coercion, closures, event loop
- **react** — typing components, hooks, and props in practice
- **nodejs** — Node runtime, packaging, and ESM/CJS beyond the type layer
- **deno** — TypeScript-native runtime without a build step

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/typescript.
