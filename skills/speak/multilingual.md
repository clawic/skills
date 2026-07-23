# Multilingual — Locales, Switching, Accents

## Reply Language

- Speak the language the user last used; mixed-language input → match the dominant language of the last message.
- One signal switches ("in Spanish, please") — language is a one-signal preference like pronunciation (SKILL.md rule 7 exceptions), not a two-signal one.
- `locale` in config sets the default; bilingual splits ("work in English, home in Spanish") go under the `contexts` preference area.

## Locale Normalization (overrides the `normalization.md` defaults)

| Element | Varies how |
|---|---|
| Decimal and thousands | 1.234,56 across much of Europe — speak per the TARGET language's convention, never per the source text's formatting |
| Date order | day-month vs month-day: always speak the month by name; "3/4" spoken in digits is a booking error waiting to happen |
| Time | most non-US locales speak 24h naturally ("14 30"); `time_format` wins when set |
| Currency position | symbol placement varies in writing; speech is uniform — amount, then currency word |
| Units | metric vs imperial from `~/Clawic/profile.yaml` fallback; convert the value, never just relabel the unit |
| Phone grouping | group per national convention — French numbers read in digit pairs, US in 3-3-4 |

## Inline Foreign Words

- Multilingual voice: wrap the word in a `<lang>` tag (`ssml.md`).
- Single-language voice: choose anglicization or a phonetic respelling (`pronunciation.md`) — never leave it to the engine's language auto-detect, which flips accent mid-sentence (`debug.md`).
- Names keep origin pronunciation when the engine renders it; within one conversation, consistency beats correctness — switching pronunciation mid-talk reads as correcting yourself or the user.

## Accent and Variant

- Variant selection (en-US vs en-GB, es-ES vs es-MX) changes more than accent: word pronunciation ("schedule", the letter Z), vocabulary, and locale defaults above.
- Pick the user's own variant, not the corpus-majority one; store under `locale` on the first clear signal.
- One voice per persona still holds across languages (SKILL.md rule 6): prefer one multilingual voice over swapping voices per language — the swap reads as a different agent.

## Scripts the Voice Cannot Read

- Transliterate into the base language's phonetics in the speech string (Cyrillic, Arabic, CJK names); the original script stays in the text channel.
- Localized numerals (Arabic-Indic digits) normalize to spoken words directly — routing them through Western digits first invites a second guess by the engine.
- If the engine skips or garbles a token silently, treat it as a no-support signal and respell — silence is a drop, and drops are forbidden (SKILL.md rule 8).
