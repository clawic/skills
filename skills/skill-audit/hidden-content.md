# Hidden Content — Invisible, Encoded, and Misrendered

Audit raw bytes. The rendered view is the attacker's presentation layer: what you see in a markdown preview and what a model reads from the file are different documents.

## Invisible Characters (reject class inside instruction text)

| Codepoints | Name | Abuse |
|---|---|---|
| U+200B–U+200D, U+2060, U+FEFF | Zero-width spaces and joiners | Split trigger words past greps; encode bits invisibly |
| U+200E, U+200F, U+202A–U+202E, U+2066–U+2069 | Bidi controls | Reorder rendered text so what the human reads differs from what the model reads — the "Trojan Source" class (Boucher & Anderson) |
| U+E0000–U+E007F | Unicode tag block | "ASCII smuggling": an instruction channel models can read that no editor displays |
| U+00AD | Soft hyphen | Invisible in most renders; breaks pattern matching |

Detection commands: `checks.md` Hidden Characters. Any of these inside instruction text = reject; show the user the decoded content.

## Homoglyphs

- Cyrillic а/е/о/р/с/х and Greek ο/ν pass for Latin at reading speed: `pаypal.com` with a Cyrillic а is a different domain than `paypal.com`.
- Highest stakes in package names, domains, and slugs — the reason `supply-chain.md` verification is char-by-char.
- Detection: any non-ASCII hit (`checks.md`) inside an identifier, URL, or install command → resolve each character before trusting the string.

## Encoded Blobs

- Decode, never execute: `echo '<blob>' | base64 -d | cat -v` — output to the terminal, piped nowhere further; `cat -v` keeps control characters visible.
- Verdict on the decoded content: a URL → `exfiltration.md`; shell → Rule 3 reject; inert data (an image, a font) → verify it is declared and actually used.
- Nested encoding (base64 inside base64, hex inside base64) has exactly zero honest uses in a skill. Reject.
- A blob the skill never references is still audited: unused payloads wait for a future version to activate them.

## Markdown and HTML Concealment

| Vector | Check |
|---|---|
| Link text ≠ target: `[docs.python.org](https://evil.example)` | Read hrefs, not link text — the endpoint grep (`checks.md`) surfaces them all |
| Reference-style links defined at file bottom | Resolve every `[text][ref]` to its definition |
| Images — many UIs auto-GET on render | Every image URL is a live network endpoint (`exfiltration.md`) |
| HTML comments `<!-- -->` | Invisible in render, fully read by models: `grep -rn "<!--" <folder>` |
| `<details>` collapsed blocks, styled-invisible HTML | Read raw source, never the preview |
| Frontmatter fields no registry card shows | Read the full frontmatter on disk |

## Whitespace and Layout

- Trailing-whitespace patterns can encode data: `grep -rnE " +$" <folder>` — occasional trailing space is sloppiness; a systematic pattern across many lines is a channel.
- Lines over ~500 chars push content past the human viewport: `awk 'length > 500 {print FILENAME":"FNR}' <file>` — read the far end of every hit.

## What Is Benign

Em-dashes, curly quotes, ellipses, emoji, accented names, non-English text in a skill that declares that language. The triage rule: invisible or misrendering = hostile until explained; visible-but-foreign = context decides.
