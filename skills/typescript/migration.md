# Migration Traps

- `noImplicitAny: false` hides errors — code "compiles" but types are wrong
- Untyped callback params are a silent `any` — `arr.map(x => x.foo)` doesn't fail
- `strictNullChecks: true` breaks a lot — localStorage.getItem returns `string | null`
- `strictPropertyInitialization` requires init in the constructor — or use `!`
- `as Type` validates nothing — `"hello" as number` compiles
- `as unknown as Type` is a total escape — avoid
- JSON.parse returns `any` — needs an assertion or validation
- `@types/x` can be outdated versus the package
- `skipLibCheck: true` hides errors in your .d.ts too
- `import x from "cjs"` vs `import * as x from "cjs"` — different behavior
- `// @ts-ignore` propagates — use `@ts-expect-error`, which fails if there's no error
- A temporary `any` stays forever — better `unknown` from the start
- `outDir` doesn't clean old files — orphaned .js cause bugs
