# Text — Strings, Regex, Formatting, Charsets

## Strings

- Immutable and interned when they are literals: `"a" == "a"` is true, `new String("a") == "a"` is false. Never compare with `==` (SKILL.md rule 1); `s.intern()` exists but pins the string for the JVM's life.
- Concatenation in a **single expression** compiles to an efficient `invokedynamic` call since JDK 9 — `"a" + b + "c"` is fine. Concatenation **in a loop** is still O(n²) because each iteration copies the accumulated string. Use `StringBuilder`, presized when you can estimate: `new StringBuilder(n * 16)`.
- `StringBuilder` (unsynchronized) over `StringBuffer` (synchronized, legacy) unless the builder is genuinely shared, which it should not be.
- `String.format` and `MessageFormat` parse the pattern on every call: readable for messages, wasteful inside a hot loop (`performance.md`).
- Useful and underused: `isBlank()`, `strip()` (Unicode-aware, unlike `trim()` which only strips ≤ U+0020), `lines()`, `repeat(n)`, `formatted(args)` (11+).
- `substring` copies since JDK 7 — no more accidental retention of a huge backing array, but also no free slicing.
- Compact strings (9+) store Latin-1 text in one byte per character; a single non-Latin-1 character doubles the array. Relevant only when profiling string-heavy memory.
- `split` returns a regex-split and drops **trailing** empty strings unless you pass a negative limit: `"a,,b,,".split(",")` has length 3, `split(",", -1)` has length 5.
- `"a.b".split(".")` returns an empty array — `.` is a regex metacharacter. Use `split("\\.")` or `Pattern.quote(sep)`.

## Text Blocks (15+)

```java
String json = """
    {"id": %d}
    """.formatted(id);
```

- Incidental leading whitespace is computed from the least-indented line **including the closing delimiter** — moving the closing `"""` changes the indentation of the whole block.
- Trailing spaces on each line are stripped; keep one with `\s`. Suppress the line break with a trailing `\`.
- A text block is a compile-time constant when it has no interpolation — usable in annotations and `case` labels.

## Regex

- Compile once: `private static final Pattern P = Pattern.compile(...)`. `String.matches`, `replaceAll`, and `split` recompile the pattern on every call — the single most common regex performance bug in Java.
- `matches()` requires the WHOLE input to match; `find()` searches. Half the "my regex doesn't work" reports are this.
- `Matcher` is stateful and NOT thread-safe; `Pattern` is immutable and thread-safe. Share the pattern, create a matcher per use.
- Escaping is doubled: `\\d` in Java source is `\d` in the regex. `Pattern.quote(s)` for literal user input.
- Catastrophic backtracking: nested quantifiers over overlapping alternatives (`(a+)+b`) go exponential on a non-matching input, and a single request pins a CPU forever. Prefer possessive quantifiers (`a++`) or atomic groups `(?>...)`, and never build a regex from untrusted input (`security.md`).
- Useful flags: `Pattern.CASE_INSENSITIVE | Pattern.UNICODE_CASE` for non-ASCII, `DOTALL` for `.` across newlines, `MULTILINE` to make `^`/`$` match per line, `COMMENTS` to write a readable multi-line pattern.
- Named groups (`(?<year>\\d{4})` → `m.group("year")`) survive refactoring; numbered groups do not.
- `replaceAll` treats `$` and `\` in the replacement as special: use `Matcher.quoteReplacement(s)` for literal text.
- Do not parse HTML, XML, JSON, or CSV with a regex. Use a parser; the edge case that breaks you is always quoting.

## Formatting Numbers and Text for Humans

- `String.format("%.2f", 1.5)` prints `1,50` under a comma-decimal default locale — a top cause of tests that pass locally and fail in CI (`debug.md`). Pass the locale explicitly: `String.format(Locale.ROOT, ...)` for machine output, the user's locale for display.
- `Locale.ROOT` for anything parsed by a machine (logs, protocols, filenames, generated code). Never `Locale.getDefault()` by accident. Display code uses the configured `locale` when the user set one (SKILL.md Configuration), the caller's locale otherwise.
- `NumberFormat.getCurrencyInstance(locale)` for money display; the amount itself stays `BigDecimal` (`debug.md`).
- `String.CASE_INSENSITIVE_ORDER` and `toLowerCase(Locale.ROOT)` — `toLowerCase()` with the default Turkish locale maps `I` to `ı`, which has broken real authentication code. Always pass a locale to case conversion.
- Sorting user-visible text uses `Collator.getInstance(locale)`, not `compareTo` — `String.compareTo` is UTF-16 code-unit order, so accented letters sort after `z`.

## Unicode Realities

- `String.length()` counts UTF-16 code units. An emoji or any character above U+FFFF counts as 2, so `length()`, `charAt`, and `substring` can split a character in half.
- Iterate code points with `s.codePoints()`; count them with `s.codePointCount(0, s.length())`.
- User-perceived characters (grapheme clusters: flags, skin-tone emoji, combining accents) need `BreakIterator.getCharacterInstance()` — even code points are not enough.
- The same text can have two byte forms (`é` as one code point or `e` + combining accent). Compare and store after `Normalizer.normalize(s, Form.NFC)`.
- Uppercasing can change length: `"ß".toUpperCase()` is `"SS"`. Never assume a case conversion preserves indices.

## Charsets and Encoding Bugs

- The default charset is UTF-8 only from JDK 18 (JEP 400). On earlier JVMs and on Windows it followed the platform, so the same code produced different bytes (SKILL.md Traps).
- Always pass the charset: `new String(bytes, UTF_8)`, `s.getBytes(UTF_8)`, `Files.newBufferedReader(path, UTF_8)` (its default already is UTF-8), `new InputStreamReader(in, UTF_8)`.
- Console output is separate: `stdout.encoding` governs `System.out` (19+), so text can be right in a file and mojibake in a terminal.
- Diagnosing the symptom: `Ã©` where `é` belongs means UTF-8 bytes were decoded as Latin-1. A `?` means the target charset cannot represent the character (encoding step). A `�` replacement character means the decoder already gave up — the original bytes are gone.
- Set the build's encoding too: `<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>` (Maven) or `options.encoding` (Gradle), or literals in source get mangled at compile time (`build.md`).
