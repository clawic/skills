# Declaration File Traps

- `declare module "x"` requires the EXACT path — `"lodash"` ≠ `"lodash/index"`
- Augmentation without imports becomes global — add `export {}` to force a module
- `declare const` without a value creates a global — can collide
- `declare function` in a module isn't global — needs `declare global {}`
- .d.ts files without import/export are global scripts — confusing legacy behavior
- `interface` can be merged from other files — `type` cannot
- `paths` in tsconfig only affects compilation — the bundler needs separate config
- `baseUrl` required for `paths` — easy to forget
- `export default` in a .d.ts is problematic — prefer named exports
- `declare module "*.svg"` affects ALL .svg — no specific types
