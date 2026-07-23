# Regex in JS — State, Backtracking, Unicode

## Statefulness (depth for SKILL.md Strings & Unicode)

- `lastIndex` lives on the regex OBJECT: a module-level `/g` or `/y` regex is shared mutable state across every caller — including across async boundaries. Either construct per use, drop `/g` for single tests, or reset `lastIndex = 0` before each use.
- `exec` with `/g` advances per call — that is the streaming-match loop. `matchAll` is the safe modern form: requires `/g`, returns a fresh iterator each call, groups included.
- `match(re)` without `/g` → first match with capture groups; with `/g` → all matches, NO groups. Needing both = `matchAll`.

## Catastrophic Backtracking (ReDoS)

- The shape: nested or overlapping quantifiers where the same text can be split many ways — `(a+)+$`, `(\w+\s?)*$`, `(.*,)*`. On failing input the engine tries every split: `(a+)+$` against `"a".repeat(30) + "!"` explores ~2^30 paths; every added character doubles the time.
- A hanging regex blocks the event loop completely — synchronous, uninterruptible, no timeout API exists. On a server that is a one-request denial of service.
- Fixes: make the repeated element unable to match the same text two ways (`[^,]*` before a literal comma instead of `.*`; unroll `(\w+\s?)*` to `\w+(\s\w+)*`); anchor early so failures fail fast; keep a pathological input (long repeat + non-matching tail) in the test suite for every nontrivial regex.
- Untrusted PATTERNS are never safe by inspection — run them in a worker you can kill, or a linear-time engine (an RE2 binding).
- Untrusted TEXT interpolated into a pattern must be escaped: `RegExp.escape(str)` (floor: `modern.md`) or the classic `str.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")`.

## Flags

| Flag | Effect | Trap it prevents / carries |
|---|---|---|
| `u` | code-point mode, `\p{...}` property escapes, strict escape errors | without it `.` and quantifiers work per UTF-16 unit: `/^.$/.test("😀")` → false |
| `v` | `u` + set operations (`[\p{L}--[aeiou]]`) + string properties (floor: `modern.md`) | stricter syntax than `u`; the two are mutually exclusive |
| `s` | `.` matches newlines | without it, multiline text silently truncates matches at `\n` |
| `m` | `^`/`$` match per line | does NOT make `.` cross lines — that is `s` |
| `y` | sticky: match exactly at `lastIndex` | tokenizer building block; fully stateful (above) |
| `d` | `.indices` spans per group | only when you need offsets, it costs a little |
| `i` | case-insensitive | with `u`, uses full Unicode case folding |

## Groups & Replacement

- Named groups: `/(?<y>\d{4})-(?<m>\d{2})/` → `match.groups.y`; in replacement strings `$<y>`; destructure directly: `const {y, m} = s.match(re).groups`.
- `replace` with a function receives `(match, ...captures, offset, whole)` — named groups arrive as one object in the LAST parameter.
- `$` is special in replacement STRINGS: replacing with user text containing `$&` or `$'` injects match fragments. Escape as `$$`, or use the function form, which never interprets `$`.
- `replaceAll("a.b", ".")` with a plain string needs no regex and no escaping — prefer it for literal replacement.
- A backreference `\1` matches the same TEXT the group captured, not the same pattern again.

## Boundaries & Non-Uses

- `\b` is ASCII-defined: it misfires around accented and non-Latin letters even with `u`. Approximate a Unicode word boundary with lookarounds: `(?<!\p{L})word(?!\p{L})` with `u`.
- Lookbehind `(?<=...)`/`(?<!...)` is ES2018 and fine on any supported Node — but Safari only since 16.4, the one floor check browser-targeted patterns still need (`browser_floor` in SKILL.md Configuration).
- Don't parse nested structures (HTML, JSON, balanced parens) with one regex — nesting is not expressible; use a real parser (`DOMParser` in browsers, `JSON.parse`).
- For literal work, string methods beat regex: `startsWith`/`endsWith`/`includes` have no escaping bugs and state their intent.
