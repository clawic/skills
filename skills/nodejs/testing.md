# Testing Traps

- `node:test` is stable on node >=20: native ESM, no transform layer, no mock hoisting magic — default it for new libraries; Jest earns its weight on existing suites and heavy matcher/snapshot use (→ SKILL.md Where Experts Disagree).
- `jest.mock()` is hoisted above imports — its factory cannot close over file-level variables; use `jest.doMock` when it must.
- Mock state persists between tests — set `clearMocks`/`restoreMocks: true` once in config instead of remembering `clearAllMocks()` in every file.
- A missing `await` lets the test pass before the assertion runs — the failure surfaces as an unhandledRejection after the suite went green. Lint test files with `no-floating-promises` too.
- Fake timers don't advance microtasks: after `advanceTimersByTime`, pending promise callbacks still haven't run — use `advanceTimersByTimeAsync`, or `await Promise.resolve()` before asserting.
- Parallel test files: bind test servers to port 0 (OS-assigned) — hardcoded ports mean EADDRINUSE under parallel runners, the classic "passes alone, fails in CI".
- Module-level state is shared within a file: a cache mutated by test 1 silently changes test 2 — `jest.resetModules()` or import a fresh instance via dynamic `import()`.
- `toBe` is `===` — fails on structurally equal objects; use `toEqual` for structure. Keep snapshots small enough to review in a diff: a 200-line snapshot gets rubber-stamped, which is worse than no assertion.
