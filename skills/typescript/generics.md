# Generic Traps

- `useState<User>()` infers `User | undefined` — handle the initial undefined
- `Array.filter(x => x.active)` doesn't narrow — needs a type guard: `.filter((x): x is Active => x.active)`
- `Promise.all([a(), b()])` infers a tuple only with `as const`
- `<T = any>` leaks the `any` into the rest of the code
- `<T extends object>` allows arrays — use `Record<string, unknown>` for objects
- `<T extends string>` with a literal infers `string`, not the literal
- `keyof T` in a generic function is `string | number | symbol`
- Arrays are covariant — `Dog[]` assignable to `Animal[]` but pushing a Cat breaks at runtime
- Function params are contravariant — `(Animal) => void` is NOT assignable to `(Dog) => void`
- `{ [K in keyof T]: X }` loses modifiers — use `-?` or `-readonly`
- `Partial<T>` and `Required<T>` are shallow — they don't affect nested ones
