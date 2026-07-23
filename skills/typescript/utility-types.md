# Utility Type Traps

- `Partial<T>` is shallow — nested ones stay required
- `Required<T>` doesn't remove `undefined` from the union — still has undefined
- `Omit<T, K>` doesn't verify that K exists — `Omit<User, "typo">` compiles
- `Pick` with a nonexistent key also compiles — no validation
- `Record<string, T>` implies EVERY key exists — access to a missing one returns T, not T|undefined
- `Record<K, V>` with a K union doesn't guarantee all keys
- `Extract<T, U>` returns `never` if there's no match — silently empty
- `ReturnType<typeof fn>` with an overload takes only the last signature
- `Parameters` likewise with overloads — inconsistent
- `NonNullable<T>` removes null AND undefined — sometimes you want only one
- `Awaited<T>` unwraps recursively — a surprise with Promise<Promise<T>>
