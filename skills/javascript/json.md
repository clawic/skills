# JSON — Precision, Fidelity, Round-Trips

## What Survives (the fidelity table)

| Value | After stringify → parse |
|---|---|
| plain objects, arrays, strings, booleans, null | intact |
| integers beyond 2^53 − 1 | silently rounded (canonical rule: SKILL.md Numbers & Money — transport big ids as strings) |
| `NaN`, `±Infinity` | `null` |
| `undefined`, functions, symbols | dropped from objects; `null` inside arrays |
| `Date` | ISO string — one way; parse returns a string |
| `Map`, `Set` | `{}` |
| `BigInt` | TypeError at stringify |
| cycles | TypeError at stringify |
| anything with `toJSON()` | whatever that method returns (Date is the built-in case) |

- Deep-cloning via JSON round-trip inherits every loss above — `structuredClone` instead (`collections.md` Copying).

## Precision

- If an upstream API emits big integers as JSON numbers, precision is lost INSIDE `JSON.parse`, before any of your code runs — no reviver can recover digits that are gone. Fix the producer contract (ids as strings), or use a raw-text-aware parser for that endpoint.
- Float artifacts round-trip faithfully: JSON does not add error, but it faithfully preserves the error your arithmetic already baked in (money: SKILL.md Numbers & Money).

## Reviver & Replacer

- Replacer function `(key, value)`: `value` has already been through `toJSON` — a Date arrives as a string; recover the original via `this[key] instanceof Date`.
- Replacer array whitelists keys — one-line payload trimming and secret redaction: `JSON.stringify(obj, ["id", "name", "total"])`.
- Log redaction by name: a replacer that returns `"[redacted]"` for `/token|secret|password|key$/i` keys is the cheap guard against credentials in logs.
- Reviver runs leaves-first, root last. Date revival needs a strict gate — revive only strings matching a full ISO pattern (`/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(\.\d+)?(Z|[+-]\d{2}:\d{2})$/`), never "looks like a date".

## Safety

- `JSON.parse` throws on invalid input, on `""`, and on `undefined` (coerced to the string `"undefined"`). Wrap every parse of external input; an empty 204 body through `.json()` is the classic crash (`browser.md` fetch).
- Prototype pollution: `JSON.parse` itself is safe — a `"__proto__"` key becomes an ordinary own property. The pollution happens in the deep-merge AFTERWARDS: merge into a null-prototype target or skip `__proto__`/`constructor` keys (SKILL.md Objects, this & Closures).
- Embedding JSON in HTML `<script>`: a `</script>` inside a string breaks out of the tag — escape `<` as `\u003c` before embedding (any JSON-for-HTML encoder does this).

## Canonicalization & Hashing

- `JSON.stringify` follows property insertion order (integer-like keys first — SKILL.md Objects): equal-content objects built in different order produce different strings. Never compare or hash raw JSON text for data equality.
- For signing/hashing/dedupe: recursively sort keys (canonical-JSON), and remember whitespace and number formatting also vary across producers — canonicalize on YOUR side, don't trust the wire form to be stable.

## Big Payloads

- `parse`/`stringify` are synchronous and whole-payload: multi-MB on a server stalls the event loop (`performance.md`). Options in order: paginate the payload; parse in a worker; use a streaming parser.
- NDJSON / JSON Lines (one object per line): appendable, streamable line-by-line, and one corrupt record loses one line, not the file — the right shape for logs, exports, and datasets.
