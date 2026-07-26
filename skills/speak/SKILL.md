---
name: Speak
slug: speak
version: 1.0.3
description: >-
  Writes and converts text into natural speech-ready output for any TTS engine:
  normalization, prosody, pronunciation, and voice preferences. Use when a reply
  will be read aloud or sent to a voice channel, when writing voiceover scripts,
  spoken briefings, voice notifications, or dialogue turns, when TTS sounds
  robotic, reads markdown, URLs, or code aloud, mispronounces names, garbles
  numbers, dates, or acronyms, speaks too fast or in the wrong accent, or when
  the user corrects pronunciation, pacing, or voice choice. Not for
  speech-to-text or live two-way call setup.
homepage: https://clawic.com/skills/speak
changelog: "Display name shown correctly"
metadata:
  clawdbot:
    emoji: 🗣️
    os:
    - linux
    - darwin
    - win32
    displayName: Speak
    configPaths:
    - ~/Clawic/data/speak/
---

Turns written replies into speech-ready text for any TTS engine and adapts voice, rate, and phrasing to the user over time. User config and learned preferences live in `~/Clawic/data/speak/` (see `setup.md` on first use, `memory-template.md` for the file format); the skill reads and writes only that folder. If you have data at an old location (`~/speak/` or `~/clawic/speak/`), move it to `~/Clawic/data/speak/`.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/speak/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| voice | text (`provider: voice-id`) | none (engine default) | Pins the persona voice everywhere speech is produced (rule 6); pin model/version too when the provider allows |
| default_rate | number (0.5-2.0) | 1.0 | Baseline speaking rate; user "faster"/"slower" signals step about 10% from here |
| speech_budget | number (seconds, 15-180) | 60 | Cap for uninterrupted speech (rule 1); content over it gets chunked or summarized |
| number_style | rounded \| exact | rounded | rounded = 2 significant figures per rule 5; exact reads full values everywhere confirmable rules don't already force it |
| time_format | 12h \| 24h | 12h | Time normalization row and every spoken time |
| locale | text (BCP 47, e.g. en-US) | en-US | Voice variant plus decimal, date, and unit conventions (`multilingual.md`) |
| ssml | auto \| off | auto | auto = test one tag per engine, then use the supported tier (`ssml.md`); off = punctuation-only prosody |
| checkins | bool | true | Chunk check-in questions in long-form; false = continuous briefing mode |

Preference areas to record as the user reveals them:

- **voice identity** — per-persona voice binding, cloning stance, register — affects rule 6 and `engines.md`
- **lexicon** — pronunciation fixes, acronym choices (SQL as "sequel" vs letters) — affects `normalization.md` and `pronunciation.md`
- **verbosity** — budget, check-in habits, question frequency — affects `long-form.md` and `dialogue.md`
- **contexts** — quiet hours, driving/shared-space profiles, what may be spoken aloud — affects `notifications.md` and `audiences.md`
- **engine** — provider choice, fallback order, streaming vs batch — affects `engines.md`

## When To Use

- Any reply that a TTS engine will read aloud, in any runtime with a voice channel
- Converting existing text (docs, briefs, lists, code output) into listenable speech
- Writing spoken briefings, voice notifications, confirmations, or dialogue turns
- Configuring or switching TTS voice, rate, or pronunciation after user feedback
- Not for speech-to-text or transcription (use `listen`) and not for real-time two-way voice session setup (use `talk`)

## Quick Reference

| Situation | Play |
|---|---|
| Normal reply will be spoken | Strip markup, answer in sentence one, cap at 150 words |
| Steps or list | Spoken enumeration ("First... Second..."), max 4 items aloud, rest to text |
| Code or logs in the answer | Speak a 1-2 sentence summary of behavior; never read syntax aloud |
| Phone number, OTP, ID, tracking code | Digit groups with pauses; never round |
| Statistic or large number | Round to 2 significant figures; exact value only on request |
| User says "slower", "faster" | Adjust rate one step (about 10%), log the signal |
| Engine mispronounces a word | Lexicon or phoneme fix once, permanently → `pronunciation.md` |
| Content over `speech_budget` | Chunk by topic with check-ins → `long-form.md` |
| Proactive alert or reminder | Speak only if time-sensitive AND actionable; else text → `notifications.md` |
| User writes in another language | Reply in the user's last language; locale rules → `multilingual.md` |
| TTS engine errors or unavailable | Fall back to text and say so in one line |
| Anything else | Persona default voice, rate `default_rate`, plain sentences, under 60 seconds |

Depth on demand: `normalization.md` every token type · `pronunciation.md` names, homographs, lexicons · `ssml.md` tags, escaping, portability · `long-form.md` briefings and documents aloud · `dialogue.md` questions, confirmations, recovery · `notifications.md` proactive speech · `multilingual.md` locales and language switching · `audiences.md` listening contexts · `engines.md` choosing and operating TTS · `debug.md` symptom→cause playbooks.

## Core Rules

1. **Budget speech in seconds, not words.** Spoken seconds = word count / 2.5 (English conversational average is 140-160 words per minute). A 150-word reply is about 60 seconds; that is the default cap (`speech_budget`) for uninterrupted agent speech. 375 words = 2.5 minutes = a monologue nobody requested.
2. **Answer in the first sentence.** Listeners cannot skim or rewind. Order: verdict, then reasoning, then caveats. Check: sentence one alone would work as the entire reply.
3. **Sentences of 20 words or fewer, one clause deep.** Test: read it aloud in one breath. Nested clauses that work on a page force the listener to hold state they will drop.
4. **Normalize every non-word token before synthesis.** Numbers, dates, units, acronyms, URLs, and symbols each need a decision (Normalization Rules below; full catalog in `normalization.md`). An unnormalized token is a pronunciation coin flip you did not call.
5. **Round for the ear.** Non-confirmable numbers to 2 significant figures: 1,247,893 becomes "about 1.2 million". Confirmable data (codes, phone numbers, amounts to be charged) is the exception: exact, digit by digit.
6. **One voice per persona.** Users bind identity to the voice; a switch reads as a different agent. Change voice only on explicit request; change rate only after a user signal.
7. **Two signals make a preference.** First occurrence: comply this session. Second consistent occurrence: confirm aloud in one sentence, then store it. Storing on the first signal fossilizes a one-off mood into permanent config. Exceptions that store on one signal: pronunciation corrections and language switches.
8. **Degrade with notice, never by dropping output.** TTS failure, unsupported markup, or over-budget content falls back to text with a one-line notice. A silent drop teaches the user the voice channel is unreliable.

## Writing For The Ear

Rewrite, do not filter. Speech-ready text is a different artifact from screen text:

- Kill all markup: `*`, `_`, backticks, `#` headers, tables, links. Engines read them literally ("asterisk asterisk") or drop them mid-word. Links: speak the site name, keep the URL in the text channel.
- Emoji are read by name ("face with tears of joy"). Delete them; carry tone in word choice.
- Bullets become enumeration with signposting: "Three things. First... Second... Third." Announce the count so the listener knows when you are done.
- Tables become comparisons: "X costs 40 dollars; Y costs 60 but includes support." More than 3 rows: speak the winner, offer the table in text.
- Homographs: engines guess tense and part of speech for "read", "live", "lead", "record", "bass". If the guess is wrong, rewrite the sentence ("I have finished reading") instead of retrying the same string (`pronunciation.md`).
- Questions to the user go last and alone. A question buried mid-monologue never gets answered.

## Normalization Rules

| Token | Speak as | Example |
|---|---|---|
| Large number | 2 sig figs + magnitude word | 1,247,893 -> "about 1.2 million" |
| Money | amount + currency word | $5.99 -> "5 dollars 99", or "about 6 dollars" |
| Phone number | digit groups, pause per group | 555-0142 -> "5 5 5, 0 1 4 2" |
| OTP or code | single characters with pauses | 8G4T -> "8. G. 4. T." |
| Date | spoken form, no raw digits | 2026-07-23 -> "July 23rd", year only if ambiguous |
| Time | per `time_format`, default 12-hour | 14:30 -> "2 30 pm" |
| Percent | the word "percent" | 12.5% -> "12 and a half percent" |
| Acronym spoken as a word | leave as is | NASA, RAM |
| Acronym read letter by letter | space or dot the letters | FBI -> "F B I"; SQL -> "S Q L" or "sequel" per user |
| Unit | full word, correct plural | 3km -> "3 kilometers"; 1ms -> "1 millisecond" |
| URL or email | site or handle name only | "on github dot com"; full string goes to text |
| Version number | digits with "point" | v2.14 -> "version 2 point 14" |
| File path or code identifier | describe, never spell | `config.yml` -> "the app config file" |

Ordinals, fractions, ranges, years, coordinates, symbols, mixed alphanumerics, and locale variants: `normalization.md`.

## Prosody And Engine Config

- Rate: default `default_rate`. Briefings and re-listens tolerate 1.1 to 1.25 on request. Above roughly 1.5, retention of numbers and names collapses; cut words instead of adding speed.
- Punctuation is the portable prosody control: comma = short pause, period = full stop. For explicit control use SSML `<break time="300ms"/>` between digit groups and topic shifts.
- SSML portability is the trap, not the syntax. Some engines accept full SSML, some honor only `break`, some read the tags aloud as text. Test one tag on the target engine before templating many (`ssml.md`); on failure, fall back to punctuation and rewriting.
- Escape `&`, `<`, and `>` in any SSML payload; one bare ampersand fails the whole request on strict parsers.
- Pronunciation fixes in preference order: engine lexicon entry, then `<phoneme>` tag, then phonetic respelling in the speech string only ("engine x" for Nginx). Respellings must never leak into the visible text channel (`pronunciation.md`).
- Interactive use: synthesize sentence by sentence and start playback on the first completed sentence. Waiting for full-reply synthesis adds the entire generation time to perceived latency (`engines.md`).

## Preference Memory

Store in `~/Clawic/data/speak/preferences.md` (full template: `memory-template.md`), one line per confirmed preference:

```
voice: <provider>: <voice-id>
rate: 1.15 (asked "faster" 2026-07-12, 2026-07-19)
lexicon: Nginx -> "engine x"; SQL -> "sequel"
style: no chunk check-ins during briefings
avoid: reading URLs aloud
```

- Write only after the two-signal rule (Core Rule 7); record the evidence dates so a later session can tell preference from mood.
- Pronunciation corrections and language switches are the exceptions: one signal = permanent line. Nobody corrects the same name twice for fun.
- Declared settings (voice, rate baseline, time format) go to `config.yaml`; observed patterns and lexicon go to `preferences.md`. An observation never overwrites a declared value without confirmation.
- Read both files at session start whenever a voice channel is active; apply them without mentioning them, never recite stored preferences unprompted.

## Output Gates

Before sending any string to a TTS engine:

1. Markup, emoji, and code syntax stripped or summarized?
2. Does the first sentence answer the question on its own?
3. Word count / 2.5 at or under `speech_budget`, or chunked with check-ins?
4. Every number, date, acronym, and URL normalized per the table?
5. Confirmable data exact and digit-grouped; everything else rounded?
6. Any SSML verified against this engine, ampersands escaped?
7. Nothing sensitive spoken uninvited in a possibly shared space (`audiences.md`)?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Piping the text reply straight to TTS | Markup and emoji are read literally; listeners hear "asterisk" | Rewrite for the ear, every time |
| Reading code or logs aloud | Syntax has no spoken form; 10 lines of code is a minute of noise | Speak the behavior in 1-2 sentences, deliver code as text |
| Speeding up rate to fit a long reply | Above ~1.5x, numbers and names stop being retained | Cut words; the budget is seconds, not rate |
| Speaking exact big numbers | "1,247,893.42" takes several seconds to say and is not retained | 2 sig figs; exact only for confirmable data |
| Respelling words in the shared text channel | Transcript shows "engine x" garbage to readers | Lexicon or phoneme tag; respell only in speech-only strings |
| Assuming SSML is portable | Unsupported tags are read aloud as angle-bracket text | Test one tag per engine before templating (`ssml.md`) |
| Switching voices for variety | Voice is identity; a switch reads as a different agent | One voice per persona, change only on request |
| Storing a preference on the first signal | A one-off mood becomes permanent config | Two consistent signals, confirm, then store |
| Burying a question mid-speech | Listeners respond to what they heard last | Question last, alone, nothing after it |
| Speaking every notification | Interruptions train the user to mute the channel | Gate on time-sensitive AND actionable (`notifications.md`) |
| Chaining two questions in one turn | The user answers only the last one heard | One question per turn (`dialogue.md`) |

## Where Experts Disagree

- **Verbatim vs rewritten.** Accessibility practice reads documents faithfully — screen-reader users expect the text, not a summary; assistant practice rewrites for the ear. Boundary: user-authored content read back → verbatim with normalization only; the agent's own replies → rewritten (`audiences.md`).
- **SSML vs punctuation.** SSML gives precision on one engine; punctuation survives every engine. Default: punctuation first, SSML only after a per-engine test passes (`ssml.md`).
- **Faked disfluencies.** One voice-UX school injects "hmm" and discourse markers for warmth; the other calls faked hesitation deceptive. Default: none — carry warmth in word choice; switch only on explicit user request.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/speak (install if the user confirms):
- `talk` — set up the real-time two-way voice session this skill writes for
- `listen` — the input side: speech-to-text and transcription accuracy
- `audio` — process the audio files themselves: conversion, cleanup, normalization

## Feedback

- If useful, star it: https://clawic.com/skills/speak
- Latest version: https://clawic.com/skills/speak

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/speak.
