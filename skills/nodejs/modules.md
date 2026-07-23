# Module Traps

## CommonJS
- `exports = x` rebinds a local alias — only `module.exports = x` changes what require returns.
- `require()` caches by resolved path: everyone gets the same object, mutations are global. Corollary: a package installed twice in the tree loads twice — two caches, two singletons, `instanceof` fails across them.
- Circular require returns whatever the partial export was at that moment — restructure, or require lazily inside the function that needs it.

## ESM
- `__dirname`/`__filename` don't exist — `import.meta.dirname` (node >=20.11), else `fileURLToPath(import.meta.url)`.
- Relative imports require the extension (`./util.js`) — CJS's extension guessing is gone; TypeScript compiled to ESM still needs `.js` in source imports.
- `import` is hoisted and static — conditional or lazy loading needs dynamic `import()`, which returns a promise.
- Top-level await blocks every importer — a module that awaits at the top level pauses evaluation of the whole import graph that depends on it until the promise settles.

## Interop
- CJS → ESM: `require()` of synchronous ESM graphs works on node >=22.12; on older versions, dynamic `import()` is the only door.
- ESM → CJS: the default import receives `module.exports`; named imports work only when the exports are statically detectable (cjs-module-lexer) — fallback is `pkg.default.thing`.
- `"type": "module"` flips every `.js` file in the package; `.cjs`/`.mjs` override per file.
- Dual-package hazard: a package shipped as both CJS and ESM can load twice into one app — two module instances, `instanceof` and module-level state break across them. Ship one format plus a thin wrapper, or go ESM-only (→ SKILL.md Where Experts Disagree).
- The `exports` map blocks deep imports (`pkg/lib/internal`) by design — that's your API surface; add subpath exports rather than removing the field when a consumer complains.
