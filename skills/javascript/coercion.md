# Coercion Traps

- `[] == false` is true: array → "" → 0
- `null == undefined` is true: but `null === undefined` is false
- `NaN !== NaN`: use `Number.isNaN(x)` to detect it
- `{} == {}` is false: objects compare by reference
- `0` and `""` are falsy: `if (count)` fails when count is 0
- `"0"` is truthy but `"0" == false`: both true
- `??` only null/undefined: `0 ?? default` returns 0
- `||` any falsy, `0 || default` returns default
- `?.` returns undefined, not null: APIs expecting null fail
- `"" + {}` vs `{} + ""`: the second is 0, parsed as an empty block
- `String(Symbol())` ok: but `"" + Symbol()` throws
- `Number("")` is 0: probably not what you wanted
- `1 + "2"` is "12" but `1 - "2"` is -1: + concatenates, - coerces
- `[1,2] + [3,4]` is "1,23,4": arrays to strings
