# Migration Traps

## Order That Works

1. `tsconfig` with `allowJs: true`, `noEmit: true`, `strict: false` — get the checker running over the untouched codebase first; renames before this step mean debugging two changes at once
2. Optional step zero for large codebases: `checkJs: true` + JSDoc annotations — real type errors surface with zero renames
3. Rename bottom-up: convert the modules with the fewest internal imports (utils, models) first, so types flow INTO the files you convert next — top-down conversion makes every file's imports `any`
4. Flags one at a time, each with its own error-burn-down: `noImplicitAny` first (biggest payoff per error fixed), then `strictNullChecks` (biggest error wave), then full `strict`
5. Ratchet the remainder: baseline the error count (`tsc --noEmit 2>&1 | grep -c "error TS"`), commit the number, fail CI when it rises — migrations without a ratchet regress silently

## Silent `any` Leaks

- `noImplicitAny: false` doesn't defer errors, it hides them — code "compiles" while half the graph is unchecked
- Untyped callback params: `arr.map(x => x.foo)` on an `any[]` never fails — the array's type, not the callback, is where the fix goes
- `JSON.parse`, `localStorage.getItem` and friends return `any`/`string | null` — wrap once in typed helpers at the boundary; validate shape at runtime for external data (→ SKILL.md Quick Reference)
- "Temporary" `any` has no expiry mechanism — write `unknown` from day one; it nags until someone narrows it, which is the point

## Assertion Debt

- `as Type` validates structure loosely at best: `"hello" as number` errors, but `{} as User` compiles — every migration `as` needs a guard or a comment (→ SKILL.md, Stop Using `any`)
- `as unknown as T` defeats even the overlap check — grep-able marker of the worst debt; count them in the ratchet too
- `// @ts-ignore` suppresses forever, even after the error is fixed — use `// @ts-expect-error`, which itself errors once the underlying error disappears, so suppressions self-expire

## Config Traps

- `strictPropertyInitialization` wants class fields set in the constructor — `!` on fields assigned by a framework (DI, ORM) is the sanctioned use of `!`, not a smell
- `skipLibCheck: true` is the right default for apps (checking all of node_modules' `.d.ts` is wasted time), but know it also skips YOUR hand-written `.d.ts` — type-check those via a test file that imports them
- CJS interop: `import x from "cjs"` requires `esModuleInterop: true`; without it use `import * as x`. Mixing styles across a codebase is how "x is not a function" ships
- One `tsconfig` can't vary strictness per directory — use project references: a strict `tsconfig` for migrated folders, a loose one for the legacy core, and move folders across as they're cleaned
- `outDir` never deletes stale output — an orphaned `.js` from a renamed file keeps getting imported; clean with `tsc --build --clean` or delete `outDir` in the build script
