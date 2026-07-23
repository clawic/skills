# Coercion & Equality

## The == Algorithm (compressed)

- `null == undefined` — and they equal nothing else. No further coercion for them.
- number vs string → string converted to number.
- boolean vs anything → boolean to number FIRST (`true`→1, `false`→0).
- object vs primitive → object through ToPrimitive: `valueOf()`, then `toString()`.
- Worked chain: `[] == false` → false→0; `[]`→`""`→0; `0 == 0` → true. Same route makes `"0" == false` true.
- The only `==` worth writing is `x == null` (→ SKILL.md Core Rule 1).

## Three Equality Regimes

| Regime | NaN vs NaN | 0 vs -0 | Used by |
|---|---|---|---|
| `===` | not equal | equal | operators, `indexOf`, `switch` |
| SameValueZero | equal | equal | `includes`, Set, Map keys |
| `Object.is` | equal | different | `Object.is` only |

- Consequence: `arr.indexOf(NaN)` is always -1; `arr.includes(NaN)` works; a Set deduplicates NaN.

## Truthiness

- Exactly 8 falsy values: `false, 0, -0, 0n, "", null, undefined, NaN`. Everything else is truthy — including `"0"`, `[]`, `{}`.
- `if (count)` drops a legitimate 0 → `count != null`. `if (str)` drops `""` — decide whether that's intended, per call site.
- `if (arr)` is always true; you meant `if (arr.length)`.

## ?? vs || vs ?.

| x | `x \|\| d` | `x ?? d` |
|---|---|---|
| `0` | d | `0` |
| `""` | d | `""` |
| `false` | d | `false` |
| `null` / `undefined` | d | d |

- `??=` assigns only on null/undefined, not on falsy. Mixing `??` with `||`/`&&` without parentheses is a SyntaxError by design — the parens are not style.
- `?.` yields `undefined`, never `null` — APIs that distinguish them (JSON bodies, DB drivers) need an explicit `?? null`.
- `?.` guards only the value it follows: `a?.b.c` still throws when `a.b` is null — repeat `?.` per nullable link (semantics: → modern.md).

## Operator Coercion

- `+` concatenates if either side becomes a string; `- * / %` always coerce to number: `"5" + 1` → `"51"`, `"5" - 1` → `4`.
- Arrays stringify via join: `[1,2] + [3,4]` → `"1,23,4"`. At a REPL, a leading `{}` parses as a block statement: `{} + []` → `0`.
- Relational operators on two strings compare code units: `"10" < "9"` is true. Mixed types go numeric.
- Template literals call `toString`: `` `${obj}` `` → `"[object Object]"`. Log the object itself, or `JSON.stringify` deliberately.
- Symbols refuse implicit string coercion (`"" + sym` throws); `String(sym)` is explicit and allowed.
