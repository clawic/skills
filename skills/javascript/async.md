# Async Traps

- `new Promise(async (resolve) => {})` — async executor swallows errors
- `.then()` without `.catch()`: silent unhandled rejection
- `return promise` vs `return await promise` in try/catch: only await catches
- `await` outside async: syntax error (top-level await only in modules)
- `await` in a loop: sequential, use `Promise.all` for parallel
- Forgetting `await`: the variable is a Promise, not the value
- `forEach(async () => {})`: does NOT wait, iterations run in parallel
- `Promise.all` one reject: everything fails, use `allSettled` for all results
- Multiple uncoordinated awaits: unpredictable order, race conditions
- Error in a Promise inside setTimeout: unhandled, not in the chain
- Top-level await error in a module: the module doesn't load
