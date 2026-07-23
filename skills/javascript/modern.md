# Modern JS Traps

- `obj?.method()` vs `obj.method?.()`: the first checks obj, the second method
- `a?.b.c` throws if a.b is null: only short-circuits the right side of the chain
- `??=` only assigns if null/undefined: not falsy
- Don't mix `??` with `&&`/`||` without parentheses, syntax error
- Destructuring default only for undefined, not null: `{a=1}={a:null}` → a is null
- Nested destructuring — `{a:{b}}={a:null}` throws
- `this` before `super()`: error in the constructor
- Private `#field` accessible via devtools: not really private
- `class` doesn't hoist: reference error if you use it before declaring
- Circular imports: can yield undefined, depends on order
- `import *` frozen object: you can't modify exports
- `structuredClone()` doesn't clone functions: throws an error
- Top-level await only in modules: normal scripts throw a syntax error
