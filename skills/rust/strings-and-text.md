# Strings and Text — `String`, `&str`, UTF-8, Paths, and Formatting

Rust is the language where "just index the string" is a compile error or a panic, on purpose: a `String` is bytes with a UTF-8 guarantee, not an array of characters.

## `String` and `&str`

- `String` owns a growable UTF-8 buffer; `&str` is a borrowed view of one. `&String` derefs to `&str`, which is why parameters should be `&str` (SKILL.md rule 2).
- `s.len()` is **bytes**. `s.chars().count()` is Unicode scalar values and is O(n). Neither is "characters" as a user means them.
- Slicing on a non-boundary **panics** at runtime: `&s[0..1]` on `"é"` panics with "byte index 1 is not a char boundary". Use `char_indices()` to find real boundaries, or `s.get(0..1)` which returns `Option` instead.
- Building strings: `write!(&mut buf, ...)` into a reused `String` (needs `use std::fmt::Write`) beats `format!` in a loop, which allocates every iteration.
- `s1 + &s2` consumes `s1` and reuses its allocation — efficient but surprising. `format!("{s1}{s2}")` keeps both and allocates once.
- `push_str` over `+=` in loops; `String::with_capacity(n)` when the size is roughly known.
- Comparison and case: `to_lowercase()` is Unicode-aware and can change length (`"İ"` grows); `eq_ignore_ascii_case` is the fast, correct choice for protocol tokens like HTTP headers.
- `Box<str>` or `Arc<str>` for a string that never grows: one word smaller than `String`, and `Arc<str>` shares without cloning bytes.

## Unicode Reality

| You want | Use | Note |
|---|---|---|
| Bytes | `as_bytes()`, `bytes()` | The only O(1) indexing |
| Unicode scalar values | `chars()` | `char` is always 4 bytes in memory |
| What a user calls a character | `unicode_segmentation::graphemes` | Not in std; emoji and combining marks need it |
| Display width in a terminal | `unicode_width` | Grapheme count is not column count |
| Truncate safely | `char_indices().nth(n)` then slice | Truncating by byte count corrupts multibyte text |
| Compare user-entered text | `unicode_normalization` NFC first | `"é"` composed and decomposed are different byte strings |

## Searching, Splitting, Trimming

- Every "pattern" argument accepts a `char`, a `&str`, a `&[char]`, or a closure: `s.split(|c: char| !c.is_alphanumeric())` needs no regex.
- `split_whitespace()` collapses runs and handles Unicode spaces; `split(' ')` yields empty strings between consecutive spaces. Choosing the second by accident is the usual cause of phantom empty fields.
- `strip_prefix("foo")` returns `Option<&str>` and is the correct replacement for `starts_with` plus manual slicing, which recomputes a length you can get wrong.
- `splitn(2, '=')` for key/value: it stops splitting, so a value containing `=` survives intact. `rsplitn` when the last separator is the delimiter.
- `find` returns a **byte** index — feed it straight back into a slice, never into `chars().nth()`.
- `lines()` strips `\n` and a trailing `\r`, so it handles CRLF input; `split('\n')` leaves the `\r` on every line and breaks comparisons on Windows-authored files.
- `replace` always allocates. In a hot path, iterate with `match_indices` and write into a reused buffer, or return `Cow` (below).
- Regex: compile once. A `Regex::new` inside a function is the single most common Rust performance bug in text code — hoist it into a `static RE: LazyLock<Regex>` (`rust >=1.80`).

## Parsing and Formatting

- `"42".parse::<u32>()` returns `Result`; the type comes from the turbofish or the binding. Implement `FromStr` for your own types and `parse` works on them too.
- `Display` is for users, `Debug` for programmers. `{}` requires `Display`, `{:?}` requires `Debug`, `{:#?}` pretty-prints. Deriving `Debug` is nearly always right; writing `Display` is a deliberate act because it defines your type's user-facing text.
- Implement `Display` and you get `ToString` free — implementing `ToString` directly is a smell.
- Inline captures (`rust >=1.58`): `format!("{name} at {pos}")` reads better than positional arguments, but only for plain identifiers — expressions still need `{}` plus an argument.

| Spec | Meaning |
|---|---|
| `{v:>8}` / `{v:<8}` / `{v:^8}` | Right, left, center in 8 columns; prefix a fill char: `{v:*^8}` |
| `{v:.3}` | Three decimals for floats, truncate to 3 chars for strings |
| `{v:08.3}` | Zero-padded width with precision |
| `{n:#x}` / `{n:#b}` / `{n:#o}` | Hex/binary/octal with the `0x`/`0b`/`0o` prefix |
| `{v:width$}` / `{v:.prec$}` | Width or precision taken from another argument or variable |
| `{{` / `}}` | A literal brace |

## `Cow` — Borrow Until You Must Own

```rust
fn normalize(input: &str) -> Cow<'_, str> {
    if input.contains('\r') { Cow::Owned(input.replace('\r', "")) } else { Cow::Borrowed(input) }
}
```

The right return type for any transform that usually changes nothing: callers pay for an allocation only in the cases that need one. Also the escape hatch for a struct that would otherwise need two versions, one borrowing and one owning.

## Bytes That Are Not Text

- `&[u8]` and `Vec<u8>` for anything that may not be UTF-8: network frames, file contents, C strings. Byte literals are `b"GET "`, byte chars `b'\n'`.
- `String::from_utf8(v)` returns `Result` and hands the bytes back on failure; `from_utf8_lossy` substitutes U+FFFD and is only acceptable for display, never for data you will store or re-emit.
- Parsers should work on `&[u8]` and validate UTF-8 once at the end — skipping validation per token is a measurable win and removes a whole class of "the input was almost text" bugs.
- `bstr` gives byte slices the string API (`split`, `trim`, `to_lowercase`) without pretending they are valid UTF-8.

## Paths and `OsStr`

- Paths are not guaranteed UTF-8 on any major platform. `Path`/`PathBuf` wrap `OsStr`/`OsString`; `to_str()` returns `Option`, `to_string_lossy()` replaces invalid sequences and must never be used to build a path you will open.
- Take `impl AsRef<Path>` in public functions — it accepts `&str`, `String`, `&Path`, and `PathBuf` with no allocation.
- `join` with an absolute argument **replaces** the whole path: `Path::new("/etc").join("/passwd")` is `/passwd`. Validate any user-supplied component before joining.
- `push` on `PathBuf` has the same replacement rule. Extension handling: `set_extension` overwrites the existing one, `with_extension` returns a new path.
- `file_stem` drops only the last extension: `archive.tar.gz` gives `archive.tar`. Match on the full name when double extensions matter.
- Comparing paths textually is wrong across `.`/`..`/symlinks; `canonicalize()` resolves them but requires the path to exist and returns the `\\?\` form on Windows.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `s.chars().nth(i)` to index | O(n) per call, so O(n²) in a loop, and it is still not a grapheme | Iterate once with `char_indices`, or work on `&[u8]` |
| `&s[..n]` to truncate a display string | Panics on a char boundary, cuts graphemes in half | `s.char_indices().nth(n)` for the byte offset, or `unicode_segmentation` |
| `to_lowercase()` for case-insensitive matching of ASCII protocol tokens | Allocates and applies full Unicode folding | `eq_ignore_ascii_case` |
| `Regex::new` inside the function | Recompiles the automaton on every call | `static RE: LazyLock<Regex>` |
| `to_string_lossy()` on a path you then open | Silently rewrites the filename with U+FFFD | Keep it as `OsStr`/`Path`; only lossy for messages |
| `format!` to build one string in a loop | One allocation per iteration | `write!` into a reused `String` with `with_capacity` |
