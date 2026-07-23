# Debugging — Symptom to Cause for Spoken Output

Work symptom-first. Reproduce with a minimal string — bisect: synthesize halves until the failing token is isolated — then fix the class, not the instance.

## "It read the formatting out loud"

- "asterisk", "underscore", "hash" spoken → markdown leaked into the speech string → strip markup (SKILL.md Writing For The Ear); verify the pipeline sends the speech string, not the display string.
- Angle brackets or tag names spoken → engine has no SSML support but received tags → punctuation-only mode, fallback map in `ssml.md`.
- "amp" or "&amp;" spoken → double-escaped entity → escape exactly once (`ssml.md`).
- Emoji names spoken ("rocket") → emoji survived the rewrite → delete them; tone goes in word choice.

## "It spelled a word letter by letter"

- ALL-CAPS token → engines treat unknown caps as initialisms → lowercase it in the speech string or add a lexicon entry.
- Token with digits (B2B, S3) → normalize explicitly ("B to B", "S 3") — `normalization.md` mixed alphanumerics.
- CamelCase identifier → describe it instead (SKILL.md: file path / identifier row).

## "It pronounced it wrong"

- Same word wrong only in some sentences → homograph part-of-speech guess → rewrite the sentence (`pronunciation.md`), never retry the same string.
- Proper or brand name wrong → fix ladder: lexicon > phoneme > respelling (`pronunciation.md`); persist on the first correction.
- Foreign word anglicized, or accent flips mid-sentence → language auto-detect tripped → `<lang>` tag or respelling (`multilingual.md`).

## Timing Problems

- Long silence before playback → full-reply synthesis before playback starts → sentence-level streaming (SKILL.md Prosody); also check for leading break tags or whitespace.
- Pauses in odd mid-sentence places → residual line breaks from wrapped source text treated as sentence ends → join lines before synthesis (`normalization.md` Structures).
- Breathless run-on, no pauses at all → punctuation stripped by an over-eager sanitizer, or SSML breaks dropped silently → inspect the exact string that reached the engine.
- Cuts off mid-reply → per-request character limit → chunk at sentence boundaries (`engines.md`).

## Sound Problems

- Chipmunk or slow-motion audio → sample-rate mismatch between synthesis output and playback → match rates (`engines.md`).
- Robotic, flat delivery → text carries no punctuation prosody, or the voice is a lower quality tier → restore punctuation first; try the engine's higher tier before blaming the text.
- Volume or timbre swings between replies → mixed voices or per-request settings drift → pin one voice and settings (SKILL.md rule 6, `engines.md`).
- Clicks between chunks → separately synthesized chunks concatenated gapless → add a short break at chunk boundaries or crossfade in the player.

## Request Failures

- Some replies fail, most work → unescaped `&`, `<`, `>` in the SSML payload on a strict parser → escape scan (`ssml.md`).
- Everything fails since a config change → invalid voice ID or unsupported language for that voice → revert to a known-good voice, change one field at a time.
- Intermittent failures under load → rate limit → fallback chain, then text with notice (SKILL.md rule 8).

## Regressions

- A fix works once, wrong again next session → applied inline, never persisted → lexicon line in `~/Clawic/data/speak/preferences.md`.
- "It sounded different today" → provider silently updated the voice model → confirm the pinned voice ID and model version (`engines.md`); note the date in preferences.
- Preference seems ignored → declared value in `config.yaml` vs observed line in `preferences.md` conflict → declared wins; reconcile per SKILL.md Preference Memory.

## When You Are Truly Stuck

Synthesize the plainest possible string ("This is a test sentence.") on the same engine and settings. Clean → the content is the problem, bisect it. Still broken → the engine or channel is the problem, walk `engines.md` operations before touching another word of text.
