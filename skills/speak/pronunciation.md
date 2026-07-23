# Pronunciation — Names, Homographs, Lexicons

## The Fix Ladder

Preference order (SKILL.md Prosody, expanded):

1. **Engine lexicon entry** — survives every future request; use whenever the engine supports one.
2. **`<phoneme>` tag** — per-request precision; alphabet support (IPA vs X-SAMPA) varies per engine (`ssml.md`).
3. **Phonetic respelling in the speech string only** — works on every engine; must never leak into the visible text channel.

Verify a fix by synthesizing the single word before shipping the sentence — hearing beats assuming. Persist immediately: one correction = permanent lexicon line (SKILL.md rule 7 exception).

## Homographs

Rewrite the sentence so the part of speech is unambiguous; retrying the same string re-rolls the same dice.

| Word | Ambiguity | Rewrite that disambiguates |
|---|---|---|
| read | past vs present | "I have finished reading" / "I will read it now" |
| lead | metal vs verb | "the lead engineer" -> "the engineer in charge" |
| record | noun vs verb | "a new record" / "start recording" |
| live | adjective vs verb | "the live stream" / "they are living in Madrid" |
| tear | rip vs cry | "tear it up" -> "rip it up" |
| wind | air vs verb | "wind the clock" -> "turn the clock key" |
| close | shut vs near | "close the file" / "close to done" -> "nearly done" |
| minute | time vs tiny | "a minute detail" -> "a tiny detail" |
| present | gift/now vs verb | "present the plan" -> "walk through the plan" |
| content | material vs happy | "the page content" -> "the page text" |

Also watch: bass, wound, object, produce, refuse, desert, contract. Same word right in one sentence and wrong in another is this failure, not an engine bug (`debug.md`).

## Proper Names

- The user's own name: exact pronunciation is identity — one correction goes to the top of the lexicon, permanently.
- People: prefer the origin pronunciation when the engine renders it; otherwise the accepted anglicization — and stay consistent within a conversation, because switching mid-talk reads as a correction.
- Places: local usage wins over dictionary form; when in doubt, follow how the user says it.
- Names in other scripts or languages: `multilingual.md` (transliteration, lang tags, auto-detect traps).

## Tech and Brand Names

Respellings that survive most engines (speech string only):

| Name | Respelling |
|---|---|
| Nginx | "engine x" |
| PostgreSQL | "postgres" (say the short form; the full form garbles) |
| Kubernetes | "koo-ber-net-ease" |
| kubectl | contested ("cube-control", "cube-C-T-L") — store the user's choice |
| GIF | contested (hard or soft g) — the user's call, store it |
| SQL / MySQL | "sequel" / "my-sequel" or letters — per user, store it |
| Azure | "azh-er" |
| Qt | "cute" |
| LaTeX | "lay-tech" |
| C# / .NET | "C sharp" / "dot net" — normalize the symbols first (`normalization.md`) |

Contested names are lexicon material by definition: pick the user's variant on the first signal and never argue phonetics aloud.

## Engine and Session Discipline

- Different engines may need different respellings for the same word — key lexicon lines by engine when they diverge (`memory-template.md`, Lexicon section).
- Apply the lexicon in every channel that speaks, not only the one where the correction happened.
- A fix that works once and regresses next session was applied inline and never persisted — the regression is the missing lexicon line, not engine drift (`debug.md`).
