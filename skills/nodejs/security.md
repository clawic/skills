# Security Traps

- `exec(cmd + userInput)` is shell injection. Use `execFile(bin, [args])` — array args never touch a shell; with `spawn`, keep `shell: false` (the default). Quoting user input for a shell is the losing move.
- Path traversal, the exact check: `const p = path.resolve(root, userInput); if (p !== root && !p.startsWith(root + path.sep)) reject`. A prefix check without the separator lets `/data-evil` pass a `/data` guard.
- Prototype pollution: `JSON.parse` itself is safe — the recursive merge you run afterwards is not. Block `__proto__`, `constructor`, and `prototype` keys in merges, or keep user-keyed data in a `Map`/`Object.create(null)`.
- `process.env` values are strings or undefined. Validate the whole set once at startup and fail fast — a missing var discovered mid-request at 3am was checkable at boot.
- Secrets in argv show up in `ps` for every user on the box — pass them via env or files, and strip tokens/authorization headers before anything reaches the logger.
- `eval()`/`new Function` on any user-reachable string is remote code execution. And `node:vm` is NOT a sandbox — its own docs say so; isolate untrusted code at the process or container level.
- Supply-chain risk concentrates in install scripts and transitive deps: `npm ci --ignore-scripts` where the build allows it; pin a vulnerable transitive dep with `overrides` instead of waiting for upstream.
- ReDoS: user input reaching a regex with nested quantifiers (e.g. `(a+)+`) triggers catastrophic backtracking — matching goes exponential and one crafted string blocks the whole process. Bound input length, avoid nested quantifiers, or use a linear-time engine.
