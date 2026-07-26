# Serde — Derive Traps and Format Edge Cases

`#[derive(Serialize, Deserialize)]` covers 90% of cases silently and correctly. This file is about the other 10%, where the derive does something defensible that is not what you meant.

## Attributes That Change Behavior You Care About

| Attribute | Effect | Watch out |
|---|---|---|
| `#[serde(rename_all = "camelCase")]` | Field naming for the whole struct | Applies to variants too on enums; `rename_all_fields` targets variant fields |
| `#[serde(default)]` | Missing field uses `Default` | Silently accepts truncated input; you lose "was it absent" |
| `#[serde(deny_unknown_fields)]` | Unknown key is an error | Incompatible with `flatten` — the two cannot be combined |
| `#[serde(skip_serializing_if = "Option::is_none")]` | Omits null fields | Changes the wire format; coordinate with consumers |
| `#[serde(flatten)]` | Inlines a nested struct's fields | Forces the self-describing path: breaks bincode and other compact formats, and disables `deny_unknown_fields` |
| `#[serde(tag = "type")]` | Internally tagged enum | Requires the variant content to be a struct or map, never a tuple or primitive |
| `#[serde(untagged)]` | Tries each variant in order | On failure reports only "data did not match any variant"; first match wins, so order is semantics |
| `#[serde(with = "module")]` | Custom ser/de for one field | The module must provide both `serialize` and `deserialize` |
| `#[serde(borrow)]` | Zero-copy `&'a str` fields | Fails on any string needing unescaping — use `Cow<'a, str>` |
| `#[serde(bound = "...")]` | Overrides derived bounds | Needed whenever the derive's `T: Serialize` bound is wrong |

## Enum Representations

| Form | JSON for `Msg::Text { body }` | Use |
|---|---|---|
| Externally tagged (default) | `{"Text":{"body":"hi"}}` | Rust-to-Rust, most compact of the tagged forms |
| Internally tagged (`tag = "t"`) | `{"t":"Text","body":"hi"}` | The usual shape for public JSON APIs |
| Adjacently tagged (`tag`, `content`) | `{"t":"Text","c":{"body":"hi"}}` | When variants hold non-struct payloads |
| Untagged | `{"body":"hi"}` | Last resort: unreadable errors, ambiguous matches |

Untagged is the one that costs you later. A number that fits both `u64` and `f64`, or two variants with overlapping fields, resolves by declaration order with no warning. If you must use it, order variants most-specific first and add a round-trip property test.

## Numbers, Dates, and Precision

- JSON numbers are `f64` in most consumers: any `u64`/`i64` above 2^53 loses precision when a JavaScript client reads it. Serialize large ids as strings — `#[serde(with = "serde_with::As::<DisplayFromStr>")]` or a small `with` module.
- `serde_json` parses integers exactly by default, so a Rust-to-Rust round trip is safe; the loss happens at the other language's parser. This is a wire-format decision, not a serde one.
- Floats: `f64::NAN` and infinities are not representable in JSON. `serde_json` serializes them as `null`, which then fails to deserialize back into `f64`. Reject or encode them explicitly.
- Dates have no canonical form. `chrono` with `serde` gives RFC 3339 for `DateTime<Utc>`; `time` needs `#[serde(with = "time::serde::rfc3339")]` per field. Pick one crate per project and one format per API, and write it in the schema.
- Durations: `std::time::Duration` serializes as a `{secs, nanos}` struct, which is almost never what an API wants. Use `serde_with::DurationSeconds` or a `with` module.

## Custom Implementations

- Prefer `#[serde(with = "...")]` on the field over a hand-written `impl` on the type — it keeps the derive for everything else and is scoped to the format you care about.
- `serialize_with`/`deserialize_with` individually when only one direction differs.
- A full manual `Deserialize` impl means writing a `Visitor`. Before doing that, check whether an intermediate representation works: deserialize into a simple struct, then `TryFrom` it into the real one. That pattern is shorter, testable, and where validation belongs.
- `#[serde(try_from = "Raw", into = "Raw")]` on the type wires that pattern up declaratively, and it is the cleanest place to enforce invariants at the boundary.

## Zero-Copy Deserialization

```rust
#[derive(Deserialize)]
struct Record<'a> { #[serde(borrow)] name: Cow<'a, str>, id: u64 }
```

- Borrowing avoids allocation per field, and it constrains the struct to live no longer than the input buffer.
- `&'a str` fails at runtime on any string containing an escape sequence, because the unescaped value has no home in the input. `Cow<'a, str>` borrows when it can and allocates when it must — always prefer it.
- Not all formats support it. Reading from a `Read` stream cannot borrow; `from_slice`/`from_str` can.

## Format Selection

| Format | Choose when | Cost |
|---|---|---|
| `serde_json` | Public APIs, config, anything humans read | Largest, slowest of the three; no schema |
| `bincode`/`postcard` | Rust-to-Rust over the wire or on disk | Not self-describing: `flatten`, `untagged`, and `skip_serializing_if` break or silently change meaning |
| `toml` | Human-edited config | Tables must come after scalar keys — a struct field order issue that surfaces only on serialize |
| `serde_yaml` / successors | Existing YAML ecosystems | YAML's implicit typing (`no` → `false`, `1.0` → float) causes real bugs |
| `rmp-serde` (MessagePack) | Compact and self-describing | Fewer tools for debugging the wire bytes |

The non-self-describing trap deserves the emphasis: a struct with `#[serde(flatten)]` compiles fine with bincode and produces data that cannot be read back. Test round trips in the format you actually ship.

## Errors and Validation

- `serde_json::Error` carries line and column — surface them, do not flatten the error to a string.
- Deserialization is not validation. Serde checks shape and type; range, format, and cross-field rules are yours. Do them in `try_from` or a `validate()` call, never scattered at use sites.
- `serde_path_to_error` reports the JSON pointer where a deserialization failed. On any config file bigger than a screen this converts an unusable error into an actionable one.
- Unknown fields are ignored by default. For config files that is a footgun (a typo in a key silently does nothing) — `deny_unknown_fields` on config structs, permissive for API responses you do not control.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `flatten` plus `deny_unknown_fields` | Serde cannot know which struct owns an unknown key; the combination is rejected or misbehaves | Pick one; validate unknown keys manually if you need both |
| `untagged` for an API with similar variants | First matching variant wins and the error message names none of them | Internally tagged with an explicit discriminator |
| `#[serde(default)]` on every field | Malformed input deserializes into a struct of defaults with no error | Default only fields that genuinely have one |
| Deriving `Serialize` on a type containing secrets | The token ends up in a log line the moment someone derives `Debug`-style output | `#[serde(skip)]`, and a newtype whose `Debug` and `Serialize` both redact |
| Assuming JSON field order is preserved | `serde_json::Map` is a `BTreeMap` unless the `preserve_order` feature is on | Enable `preserve_order`, or do not depend on order |
| Changing a field name without `alias` | Old persisted data stops loading | `#[serde(alias = "old_name")]` and keep it for a full deprecation cycle |
| Very deep or untrusted nested input | Recursive deserialization can overflow the stack | Depth-limit at the boundary; treat the parser as attack surface |
