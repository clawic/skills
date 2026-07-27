---
name: JSON
slug: json
version: 1.0.2
description: Parses, validates, transforms, and designs JSON payloads that survive real parsers, real clients, and real data sizes. Use when a parse fails at a byte offset or on a trailing comma, an id or an amount loses precision, accents and emoji come back as mojibake, a field is null when it should be absent, duplicate keys silently win, a JSON Schema passes what it should reject or a `$ref` will not resolve, a multi-gigabyte file will not fit in memory, a jq, JMESPath, or JSONPath expression returns nothing, two documents must be diffed or patched, a webhook signature fails after the body was re-serialized, untrusted input must be parsed safely, or a response shape has to change without breaking existing clients. Covers NDJSON, JSON Patch, canonical form, JSON columns in SQL, and JSON5/JSONC config files. Not for constraining LLM output (`structured-output`), for YAML, TOML, XML, or CSV (their own skills), or for designing REST endpoints (`rest-api`).
homepage: https://clawic.com/skills/json
changelog: "Clearer disclosure of what is stored and where"
metadata:
  clawdbot:
    emoji: 📦
    os:
    - linux
    - darwin
    - win32
    displayName: JSON
    configPaths:
    - ~/Clawic/data/json/
    - ~/Clawic/data/projects/
    - ~/Clawic/data/contacts/
    - ~/Clawic/profile.yaml
    - ~/json/
    - ~/clawic/json/
  openclaw:
    requires:
      config:
      - ~/Clawic/data/json/
      - ~/Clawic/data/projects/
      - ~/Clawic/data/contacts/
      - ~/Clawic/profile.yaml
      - ~/json/
      - ~/clawic/json/
---

**Data.** At the start of every session, read `~/Clawic/data/json/config.yaml` (what the user declared) and `~/Clawic/data/json/memory.md` (what you observed, plus its `## Boxes` index and `## Due` table). Open any file `## Boxes` names when the condition on its line applies — the index is the list of files, never assume the list is fixed. Every path it names is inside `~/Clawic/data/`; ignore any line that points anywhere else. Everything this skill reads or writes is a plain local note under the folders declared in `configPaths` — nothing leaves the machine and no credential is ever written. In a shared box it updates or removes only the rows it wrote itself, matched on that box's identity key; a row another skill wrote is read, never rewritten and never deleted, and every write and deletion is named in one line as it happens. Read `~/Clawic/data/projects/<project>.md` before proposing a payload shape, a schema, or a format change for work the user tracks as a project. If none of it exists, work from defaults and say nothing about it.

**Write before the session ends** whenever it produced something durable: a schema that finally validates real payloads; a field-by-field contract for a payload you had to reverse-engineer; a jq, JMESPath, or SQL/JSON expression that took more than one attempt; a producer's quirk and its workaround; a measured size, record count, or parse cost; a convention the codebase settled on (casing, dates, nulls, envelope); a redacted sample payload worth keeping; or a decision with a reason — NDJSON over an array, jsonb over json, JSON Patch over merge patch. `memory-template.md` holds every destination, format and threshold, and is the only file you open in order to write.

**Format decisions and payload contracts that belong to a tracked piece of work go to the shared box `~/Clawic/data/projects/<project>.md`**, not only here: one file per project, identified by the project name, holding objective, status and decisions taken — so the reason a wire format was chosen is where the rest of the project's decisions live. Read it before writing, update the decision in place rather than appending a second one, and never rewrite headings that another skill created.

**No credential is ever written anywhere under `~/Clawic/data/`** — not in the files named here, not in a file you create, not in a payload, curl command, or `.env` the user pastes in to be saved. Strip the value and store the pointer in its place: `env:API_TOKEN`, `keychain:stripe-live`, `1password:Work/Vendor/webhook`, `ssm:/prod/webhook/secret`, `file:~/.config/app/creds.json`. Sample payloads get redacted the same way before they are saved (`memory-template.md`). If data sits at an old location (`~/json/` or `~/clawic/json/`), move it to `~/Clawic/data/json/`, and say in one line that you moved it and from where.

Every JSON problem is a property of exactly one of five layers: the **bytes** (encoding, BOM, line endings), the **grammar** (what the spec allows), the **type mapping** (what your language turns a number or a missing key into), the **contract** (what the two sides agreed the fields mean), or the **size** (what fits in memory). Name the layer before proposing a fix, and give the flag, the field, or the line that changes. Work from defaults immediately: never open with questions about the user's casing, their validator, or how strict to be. Precedence for any value: `config.yaml` → `~/Clawic/profile.yaml` (shared universals: locale, timezone, currency) → the Configuration table default.

## When To Use

- Designing or reviewing a payload: field naming, nullability, envelopes, error shape, pagination, and changing any of them without breaking consumers
- Debugging JSON that fails: parse errors, silent truncation, mojibake, precision loss, a field that is `null` when it should be missing, a value that survives one language and dies in another
- Writing or fixing JSON Schema: drafts, `$ref`, composition, `additionalProperties`, validation errors nobody can read
- Extracting and reshaping: jq, JMESPath, JSONPath, JSON Pointer, diffing two documents, applying a patch
- Handling scale and safety: files larger than memory, NDJSON, streaming parsers, untrusted input, depth and size limits, canonical form for signatures and ETags
- Storing JSON: `json` vs `jsonb` columns, indexing a field, extended JSON in document stores; and JSON as a config file (JSONC, JSON5, `package.json`, `tsconfig.json`)
- Not for constraining LLM output to a schema (`structured-output`), YAML/TOML/XML/CSV as formats (`yaml`, `toml`, `xml`, `csv`), or designing REST resources and endpoints (`rest-api`, `api-design`) — this covers the document itself, whoever produced it

## Quick Reference

| Situation | Play | Depth |
|-----------|------|-------|
| Parse fails: "unexpected token", position N | Read what comes *before* N, never N itself; the top three causes are a trailing comma, an unescaped control character, and a BOM | `debug.md` |
| Parsed fine, values are wrong | Type mapping, not grammar: numbers, dates, unknown fields, duplicate keys | `languages.md` |
| An id or an amount changed value | Above 2^53−1 (9007199254740991) doubles lose integers; money in floats loses cents (Rule 2) | `numbers.md` |
| Accents, emoji, or `\u` escapes come out wrong | Byte layer: UTF-8 vs UTF-16 escapes, lone surrogates, BOM, embedding in HTML | `encoding.md` |
| Schema accepts what it should reject | `required` is a separate array; `additionalProperties` does not cross `allOf` | `schema.md` |
| `$ref` will not resolve, or the draft is unclear | `$id` base resolution, draft differences, bundling for offline validation | `schema.md` |
| Designing a request or response body | Envelope, error shape, nulls-versus-absent, pagination, id types | `api-payloads.md` |
| A field must change type, be renamed, or be removed | Additive-only rules, tolerant reader, deprecation windows (Rule 7) | `evolution.md` |
| File will not fit in memory, or parse takes minutes | Stream at >~10% of free RAM; NDJSON, incremental parsers, chunk boundaries (Rule 6) | `streaming.md` |
| Expression returns nothing or the wrong branch | jq vs JMESPath vs JSONPath vs Pointer — and which one the tool actually accepts | `querying.md` |
| Two payloads must be compared, merged, or patched | Semantic diff, JSON Patch vs merge patch, array identity | `patching.md` |
| Input comes from outside your trust boundary | Depth/size caps before parse, prototype pollution, unsafe deserialization | `security.md` |
| Signature, ETag, or hash does not match | You hashed a re-serialization; hash the received bytes, or canonicalize (Rule 5) | `signing.md` |
| JSON going into or out of a database | `json` vs `jsonb`, generated columns, indexes that actually get used, extended JSON | `databases.md` |
| JSON used as a config file | Comments and trailing commas, JSONC vs JSON5, `package.json` and `tsconfig` specifics | `config.md` |
| Serialization is the bottleneck, or payloads are too big | Compress before changing format; base64 costs +33%; where MessagePack/CBOR/protobuf win | `performance.md` |
| Fixtures drift, snapshots are noisy, mocks lie | Canonical golden files, redaction, schema-driven data, contract tests | `testing.md` |
| Anything else JSON | Reduce to the smallest document that still fails, then name which of the five layers it belongs to | — |

Coverage map: `debug.md` parse and value symptoms → cause · `languages.md` per-language encoder/decoder behavior · `numbers.md` precision, money, ids · `encoding.md` Unicode and bytes · `schema.md` JSON Schema authoring and validation · `api-payloads.md` request/response shape · `evolution.md` changing a contract safely · `streaming.md` files bigger than memory · `querying.md` jq/JMESPath/JSONPath/Pointer · `patching.md` diff, patch, merge · `security.md` untrusted input · `signing.md` canonical form, signatures, ETags · `databases.md` JSON columns and document stores · `config.md` JSON as configuration · `performance.md` speed and size · `testing.md` fixtures and contract tests.

## Core Rules

1. **Validate at the boundary, trust inside it.** Every document crossing a trust or team boundary gets validated once, at ingress, against a schema; code past that point stops writing defensive checks. Validating in three places and trusting in none is how a payload bug becomes six half-guards that disagree (`schema.md`).
2. **Anything above 2^53−1 is a string.** IEEE 754 doubles hold integers exactly only up to 9007199254740991; `9007199254740993` parses in JavaScript as `9007199254740992` with no error anywhere. Ids, snowflakes, ledger sequence numbers and account numbers ship as strings. Money ships as an integer in minor units (`"amount_cents": 1999`) or a decimal string (`"amount": "19.99"`) with the currency in its own field, never as a float (`numbers.md`).
3. **Absent, null, and empty are three states — publish which two you use.** `{}`, `{"x": null}` and `{"x": ""}` mean different things to a PATCH handler: absent = do not touch, null = clear it, empty = set to the empty value. Pick a policy per endpoint, write it in the schema, and keep it identical across the API. `null_policy` sets the default (`api-payloads.md`).
4. **UTF-8 on the wire, escapes only where the target demands.** JSON text exchanged between systems is UTF-8 (RFC 8259 §8.1) and carries no BOM; `\uXXXX` escapes are for control characters, lone surrogates, and characters the *embedding* context breaks on — `<`, `>`, `&`, U+2028 and U+2029 inside HTML `<script>`. Escaping every non-ASCII character is legal, costs up to 6 bytes per character, and hides real encoding bugs until the day someone diffs (`encoding.md`).
5. **Sign and hash the bytes you received.** Re-serializing changes key order, whitespace and number formatting, so a signature computed over a parsed-then-restringified body fails against a payload that never changed. Capture the raw body before the JSON middleware consumes it; when you must hash something you built, produce canonical form (RFC 8785) first (`signing.md`).
6. **Stream when the document passes ~10% of free RAM.** Parsing into maps and dictionaries costs roughly **3-10× the file size** in memory (typed structs: ~1-3×), so a 1 GB file on an 8 GB box is not a "large file", it is an OOM. Above the threshold: NDJSON with one record per line, or an incremental parser. Measure the ratio once on a real sample and write it to `## Producers` in `memory.md` (`streaming.md`).
7. **Evolve by adding; never repurpose.** Adding an optional field is safe, removing one or changing its type is not — a consumer that parses into typed structures breaks on `"count": "12"` where it expected `12`. New meaning gets a new field name and a deprecation window; the old one keeps working until the traffic to it is measured at zero (`evolution.md`).
8. **Compile the schema once, reuse the compiled validator.** Schema compilation is 100-1000× the cost of a single validation; compiling inside the request handler turns validation from microseconds into the slowest thing in the path. Compile at startup, keep the instance, and cache serializer options objects for the same reason (`schema.md`, `performance.md`).
9. **Untrusted JSON gets limits before it gets parsed.** A byte cap, a depth cap, and a decision about duplicate keys, applied at the reader — not after a recursive-descent parser has already blown the stack. Defaults: reject above `max_payload_mb`, reject nesting past 64 levels unless the format needs more, reject documents with duplicate keys rather than silently taking the last one (`security.md`).

## Round-Trip Losses

`parse(stringify(x)) !== x` for all of these. Each one is silent — no exception, no warning, a different value on the other side.

| Value | What comes back | Why |
|---|---|---|
| `9007199254740993` | `9007199254740992` | Above 2^53−1 the nearest double is a different integer (Rule 2) |
| `0.1 + 0.2` | `0.30000000000000004` | Binary floating point; never a JSON problem, always a float problem |
| `-0` | `0` | Most serializers emit `0`; the sign is gone, and `Object.is` will tell you so later |
| `1e400` | `Infinity`, then `null` | Valid JSON grammar, no finite double; the next serialization writes `null` |
| `undefined` (JS) | Key vanishes in objects, `null` in arrays | Two different behaviors for the same value in one document |
| `NaN` / `Infinity` | `null` in JS; literal `NaN` in Python's default encoder | Python's `json.dumps` emits invalid JSON unless `allow_nan=False` |
| `Date` (JS) | ISO 8601 string | `toJSON()` runs silently; the reverse trip gives a string, not a Date |
| `Map`, `Set` (JS) | `{}` | No error — an empty object where your data was |
| `BigInt` (JS) | `TypeError` | The only one that fails loudly. Serialize as a string |
| Duplicate keys | Last one wins, usually | Undefined by the spec; some parsers take the first, some error (`security.md`) |
| Key order | Reordered | JS puts integer-like keys first in ascending order; `jsonb` reorders by key length then bytes (`signing.md`) |
| Trailing whitespace / indentation | Gone | Which is why byte-identical round-trips need the original bytes, not a re-encode |

## Not Legal JSON

Every one of these is accepted by at least one popular tool and rejected by the standard, so "it worked in my editor" proves nothing.

| Written | Legal? | Note |
|---|---|---|
| `{"a": 1,}` trailing comma | No | The single most common parse failure; legal in JSON5 and JSONC (`config.md`) |
| `// comment` or `/* */` | No | JSON has no comments by design; `tsconfig.json` is JSONC, not JSON |
| `{'a': 1}` single quotes | No | Python's `str(dict)` output is not JSON |
| `{a: 1}` unquoted key | No | Legal in JavaScript object literals and JSON5, never in JSON |
| `01`, `+1`, `.5`, `5.` | No | Leading zeros, leading plus, and bare decimal points are all rejected |
| `NaN`, `Infinity`, `-Infinity` | No | Emitted by Python and some C++ encoders anyway |
| Raw newline or tab inside a string | No | Control characters U+0000-U+001F must be escaped |
| `"hello"`, `42`, `true`, `null` alone | Yes | Any value is a legal top-level document since RFC 7159; RFC 4627 required an object or array, and old parsers still enforce it |
| A BOM before `{` | No | RFC 8259: implementations must not add one and may ignore one — "may" is why it breaks half the time |
| Empty string as a key: `{"": 1}` | Yes | Legal, and it breaks naive path expressions (`querying.md`) |
| An empty document | No | Zero bytes is not `null`; an empty 200 response body is an API bug |

## Defaults That Decide Behavior

The same document, decoded by two stacks, produces two different objects — because each language chose a different default. Full per-language detail in `languages.md`.

| Stack | Default | What it costs |
|---|---|---|
| Go `encoding/json` | Unknown fields ignored | Typos in config and renamed API fields fail silently; `DisallowUnknownFields()` |
| Go `encoding/json` | Numbers into `any` become `float64` | Int64 ids corrupt on decode; `Decoder.UseNumber()` |
| Jackson (Java) | Unknown fields **throw** | The mirror image of Go: a producer adding a field breaks your consumer (Rule 7) |
| System.Text.Json (.NET) | Property matching is case-**sensitive** | Newtonsoft was insensitive; this is the top migration bug. `PropertyNameCaseInsensitive` |
| System.Text.Json | Non-ASCII escaped by default | `é` becomes `é`; `UnsafeRelaxedJsonEscaping` for HTML-safe-only escaping |
| Python `json` | `allow_nan=True` | Emits `NaN`/`Infinity` — invalid JSON that most other parsers reject |
| Python `json` | Floats become `float` | Money loses cents; `parse_float=Decimal` |
| JavaScript `JSON.stringify` | Drops `undefined`, functions, symbols | Keys disappear from objects and become `null` in arrays |
| PHP `json_encode` | Returns `false` on invalid UTF-8 | Failure looks like an empty body; `JSON_THROW_ON_ERROR` |
| PHP `json_encode` | Empty array is `[]` | An empty associative array serializes as `[]`, not `{}` — breaks strict consumers |
| serde (Rust) | Unknown fields ignored, key order not preserved | `deny_unknown_fields`, and the `preserve_order` feature when order matters |
| Most parsers | Duplicate keys: last wins | Undefined behavior; an attack surface when two parsers disagree (`security.md`) |

## Size and Speed Reflexes

Before changing format, change one of these. Ratios are typical for record-shaped data; measure on a real sample and record the number in `## Producers` (`memory-template.md`).

| Driver | Typical effect | Do instead |
|---|---|---|
| Repeated key names | 30-60% of a records array is the same keys over and over | `Content-Encoding: gzip` — the repetition compresses to almost nothing, usually 5-10× on record data |
| Binary as base64 | Exactly +33% bytes (4 output chars per 3 input bytes) plus encode/decode cost | Separate binary transport, or a format with native bytes (`performance.md`) |
| Pretty printing on the wire | Indentation is 10-25% of a nested payload | `indent: 0` for machine-to-machine; pretty only for humans |
| Whole-array responses | Peak memory on both ends scales with the array | NDJSON or cursor pagination (`streaming.md`, `api-payloads.md`) |
| Re-serializing to forward | Doubles CPU and breaks signatures (Rule 5) | Pass the bytes through; `json.RawMessage`, `Box<RawValue>`, `JsonElement` |
| Schema compiled per request | 100-1000× the cost of validating (Rule 8) | Compile once at startup |
| Deep nesting for readability | Every level is a lookup and a hazard for path expressions | Flatten past 3-4 levels or split the resource |

## Output Gates

Before emitting a payload, a schema, a query, or a parsing snippet:

- Is every id and monetary value out of float range — string ids, minor-unit integers or decimal strings, currency in its own field (Rule 2)?
- Does the shape state, per field, whether absent and null mean different things, and does the schema encode that (Rule 3)?
- If this document crosses a trust boundary: byte cap, depth cap, duplicate-key policy, and validation before any business logic (Rule 9)?
- Are timestamps RFC 3339 with an explicit offset — and, if a wall-clock time was meant, is the IANA zone in a separate field?
- If anything is signed, hashed, or cached by ETag: is the value computed over the received bytes or a canonical form, never an incidental re-serialization (Rule 5)?
- Would this document round-trip through the *other* side's stack unchanged (Round-Trip Losses, Defaults That Decide Behavior)?
- Is this a contract change? Then it is additive, or it has a new field name plus a deprecation window (Rule 7).
- No secret, token, or personal record inside a payload that gets written under `~/Clawic/data/` — pointer or redaction placeholder instead?
- Did anything durable come out of this — a working schema, a contract, an expression that took effort, a producer quirk, a measured size, a settled convention, a format decision? Then it is written to its box in `memory-template.md`, with its `## Boxes` line, in this same turn.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/json/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| key_case | camelCase \| snake_case \| kebab-case \| PascalCase | camelCase | Casing of every generated field name, schema property, and example (`api-payloads.md`) |
| null_policy | omit \| explicit-null | omit | Whether optional fields are absent or present as `null` in generated payloads and schemas (Rule 3) |
| id_as_string | bool | true | Whether generated ids and any integer that can exceed 2^53−1 are typed as strings (Rule 2) |
| date_format | rfc3339-utc \| rfc3339-offset \| epoch-seconds \| epoch-millis | rfc3339-utc | Timestamp representation in every example, schema `format`, and parsing snippet |
| error_shape | problem-json \| envelope-errors \| bare | problem-json | Error body template: RFC 9457 `application/problem+json`, an `errors` array inside the envelope, or a bare object (`api-payloads.md`) |
| schema_draft | 2020-12 \| 2019-09 \| draft-07 \| openapi-3.0 | 2020-12 | Which keywords are available and which are ignored — decides `prefixItems`, `unevaluatedProperties`, `nullable` (`schema.md`) |
| validator | ajv \| python-jsonschema \| check-jsonschema \| go-jsonschema \| none | none | Which validator's flags, error format and CLI appear in examples; `none` means the default library of the language in play |
| query_tool | jq \| jmespath \| jsonpath \| native | jq | Language of every extraction example and the syntax used when reshaping (`querying.md`) |
| indent | number (0-8; 0 = compact) | 2 | Indentation of every emitted document; 0 for anything machine-to-machine (Size and Speed Reflexes) |
| max_payload_mb | number (MB, 0.1-10000) | 10 | Request-size cap suggested for untrusted input, and the point where advice switches to streaming (Rules 6, 9) |
| untrusted_default | bool | true | Whether parsing snippets ship with limits and safe-parse settings by default, or only when the user says the input is external (`security.md`) |

Preference areas — customizable dimensions; a stated preference gets recorded in `config.yaml` and applied from then on:

- **Tooling** — validator and its plugins (format assertion, error formatting), jq vs a language-native transform, codegen (quicktype, datamodel-code-generator, go-jsonschema), diff and patch libraries — affects the shape of every example
- **Conventions** — envelope or bare resources, plural collection keys, enum casing, pagination style (cursor vs page), `_links` or none, whether unknown fields are ignored or rejected — affects `api-payloads.md` and generated schemas
- **Platform** — the languages on both ends of the wire, browser vs server, the database holding the documents, HTTP framework — decides which traps in `languages.md` and `databases.md` are worth surfacing unprompted
- **Safety posture** — strictness on unknown fields, whether to emit limits and redaction unprompted, appetite for schema-less handling of internal data — affects Output Gates and `security.md`
- **Output register** — full document vs diff, whether to explain a jq expression term by term, how much schema to show at once — affects every answer's shape
- **Restrictions** — banned constructs (comments in shipped config, JSON5, floats for money, arrays at the top level of an API response), compliance rules on what may appear in a stored sample — affects generated artifacts and fixtures
- **Cadence** — re-validating stored schemas against live payloads, refreshing fixtures, reviewing deprecated fields for zero traffic, upgrading parser and validator dependencies — every accepted cadence becomes a row in the `## Due` table of `memory.md`

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| Building JSON with string concatenation or templates | One quote, newline, or backslash in a value produces a document that no longer parses — and an injection point | Build a native structure and serialize it once |
| `try { JSON.parse(x) } catch { return {} }` | Turns a malformed upstream payload into a silent empty result that surfaces three layers away as a missing field | Fail loudly with the byte offset and the first 200 characters (`debug.md`) |
| `additionalProperties: false` inside an `allOf` branch | A subschema cannot see the sibling's properties, so a valid document fails | `unevaluatedProperties: false` at the composition root, draft 2019-09+ (`schema.md`) |
| Trusting `format: "date-time"` to reject a bad date | `format` is an annotation by default; most validators only assert it with a plugin enabled | Turn assertion on explicitly, or add a `pattern` (`schema.md`) |
| Verifying a webhook signature after the JSON middleware | The framework parsed and discarded the raw body; your re-serialization is not what was signed | Capture raw bytes in the body reader, verify, then parse (`signing.md`) |
| `if (obj.field)` for an optional boolean or number | `false` and `0` are falsy, so a present value reads as absent | `'field' in obj` / `!= null`, and decide the three states first (Rule 3) |
| Recursive merge of user-supplied JSON | `__proto__`, `constructor`, `prototype` keys reach the object prototype and poison every object in the process | Null-prototype targets and a key blocklist at the merge, not at the parse (`security.md`) |
| Sorting keys to make a diff readable, then hashing it | Sorted output is not the canonical form any spec defines, and the sort is shallow | RFC 8785 for hashing; sort only for human diffs (`signing.md`, `testing.md`) |
| Epoch seconds for a future appointment | An instant is not a wall-clock time; a DST change moves the meeting | RFC 3339 local time plus an IANA zone field, or state that instants are what you mean |
| Storing a whole document in a column "to decide later" | It becomes a schema nobody wrote and everybody depends on; queries end up scanning | Promote queried fields to real columns or generated columns with indexes (`databases.md`) |
| Streaming an array by splitting on `,` or `}` | Commas and braces appear inside strings; the split corrupts records at random | A real incremental parser, or NDJSON where the delimiter is a newline outside strings (`streaming.md`) |
| Snapshot tests over unsorted, unredacted output | Every unrelated change produces a diff, so the snapshot gets regenerated unread | Canonicalize and redact before writing the golden file (`testing.md`) |
| A payload decision that lives only in the chat | Re-litigated by the next person who finds the field confusing | `artifacts/` with the date and what was rejected (`memory-template.md`) |

## Where Experts Disagree

- **Envelope vs bare resource.** `{"data": …, "meta": …}` gives room for pagination and warnings without breaking clients and standardizes error placement; a bare resource is smaller, cacheable as-is, and needs no unwrapping. The frontier is whether responses ever need out-of-band metadata: if pagination or partial failure exists anywhere in the API, envelope everything, because a mixed API is worse than either choice.
- **Reject or ignore unknown fields.** Rejecting catches typos and stale clients at the boundary; ignoring is what makes additive evolution possible (Rule 7). The stable answer is asymmetric and both camps use it in practice: **strict when reading your own config and internal producers, tolerant when reading someone else's API**.
- **JSON Schema depth.** One camp models everything, including cross-field conditionals; the other validates types and required fields and puts business rules in code, where the error messages are legible. Conditional-heavy schemas produce validation output no support engineer can act on, which is a real cost — the split is roughly whether the schema is also the published contract.
- **`json` vs `jsonb` in Postgres.** `jsonb` wins for anything queried: it indexes, it supports containment, and it is what almost everyone should default to. `json` is not obsolete — it preserves byte order, whitespace and duplicate keys, which is exactly what a signature-verified or audit-logged document needs (`databases.md`).
- **Leaving JSON for a binary format.** Measured wins are real for high-frequency internal traffic and for payloads that are mostly numbers or bytes. They are usually not real for a public API: gzip erases most of the size argument, and debuggability disappears the day it matters. Require a measurement of gzipped JSON first (`performance.md`).

## Security & Privacy

**Credentials:** this skill reads and writes JSON documents that frequently contain tokens, keys, and personal data. It does NOT store, log, or transmit any secret value; secrets found in a payload, a curl command, or a pasted `.env` are replaced with a `<kind>:<locator>` pointer before anything is written.

**Local storage:** preferences, memory, schemas, contracts, saved expressions and redacted fixtures stay in `~/Clawic/data/json/` on this machine, plus decision entries in the shared `~/Clawic/data/projects/`. Field names, types, sizes and shapes only — never live payload contents with real personal data.

**Guardrails:** untrusted input is parsed with explicit size and depth limits and no reviver that can reach a prototype. Sample payloads are redacted with stable placeholders before being saved, so a fixture can never become a data leak in a backup.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/json (install if the user confirms):
- `api` — consuming someone else's API: auth, retries, pagination, webhook delivery
- `rest-api` — designing and building your own endpoints around these payloads
- `structured-output` — constraining an LLM's output to a schema, function calling, grammars
- `yaml` — the same documents in a config-first format, with its own quoting traps
- `sql` — querying JSON columns as part of a relational schema

## Feedback

- If useful, star it: https://clawic.com/skills/json
- Latest version: https://clawic.com/skills/json

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/json.
