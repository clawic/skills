# SSML — Precision With a Portability Trap

Rule zero (SKILL.md Prosody): test one tag on the target engine before templating many. Engines fall into three tiers — full SSML, break-only subset, none (tags read aloud as text). `ssml: off` in config forces punctuation-only mode regardless of tier.

## Tags by Portability (most → least)

| Tag | Does | Portability notes |
|---|---|---|
| `<break time="300ms"/>` | pause | The most widely honored tag; the `strength` attribute is less supported than `time` |
| `<say-as interpret-as="...">` | forces token class | Classes: characters, digits, telephone, date, time, ordinal, fraction, unit — support varies PER CLASS; test each class you use |
| `<sub alias="...">` | speak the alias | Useful where no lexicon API exists; display form is irrelevant in audio |
| `<prosody rate="" pitch="" volume="">` | delivery control | Percentage vs named values differ per engine; large pitch shifts produce artifacts |
| `<emphasis>` | stress | Frequently ignored silently — never let meaning depend on it |
| `<phoneme alphabet="ipa" ph="...">` | exact pronunciation | Alphabets: `ipa` vs `x-sampa`; the wrong alphabet is ignored or read aloud |
| `<lang xml:lang="...">` | language switch | Multilingual voices only (`multilingual.md`); single-language voices ignore or garble it |
| `<audio src="...">` | insert an audio clip | Cloud-specific; unavailable on local/offline engines |

`<speak>` root wrapper: some engines require it around any SSML; add it only after the tag test passes — on a no-SSML engine it gets read aloud like everything else.

## Escaping

- Escape in every SSML payload: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`; inside attribute values also `"` → `&quot;`.
- Escape exactly once — double-escaping makes the engine SAY "amp" (`debug.md`).
- One bare ampersand fails the whole request on strict parsers (SKILL.md gate 6); scan generated text before wrapping, especially text that quotes user content or URLs.

## Fallback Map (when the test fails)

| Wanted | Plain-text equivalent |
|---|---|
| break | comma (short pause), period (full stop), paragraph break (long pause) |
| say-as characters/digits | pre-normalize yourself: space the characters in the string (`normalization.md`) |
| phoneme | phonetic respelling in the speech string (`pronunciation.md`) |
| prosody rate | cut words — the budget is seconds, not rate (SKILL.md rule 1) |
| emphasis | word order: put the stressed item first or last; add "only"/"exactly" |
| lang | respell the foreign word phonetically in the base language (`multilingual.md`) |

## Test Protocol

1. Send `before <break time="500ms"/> after` to the engine.
2. Audible half-second gap, no spoken "break" → tag tier confirmed. Test each ADDITIONAL tag type once before first use; passing `break` proves nothing about `say-as`.
3. Tag read aloud or request rejected → punctuation-only mode for this engine.
4. Record the engine's tier and any per-class results in `~/Clawic/data/speak/preferences.md` (Engine Notes) so no later session re-tests.

## Templating Discipline

- Template only tags this engine passed; a template full of unsupported tags fails loudly on tier-3 engines and silently degrades on tier-2.
- Per-character billing may count SSML markup as characters (`engines.md`) — verbose prosody wrappers can double the bill for zero audible change on a subset engine.
- Keep SSML generation last in the pipeline: normalize and rewrite first, wrap tags around the finished speech string, then escape.
