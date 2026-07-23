---
name: speak
slug: speak
version: 1.0.1
description: Converts agent replies into natural spoken output for any TTS engine with text normalization, prosody control, and voice preference learning. Use when output will be read aloud, when writing for text to speech, or when the user corrects pronunciation, pacing, or voice choice.
homepage: https://clawic.com/skills/speak
changelog: Complete rewrite with real voice-output guidance
metadata:
  clawdbot:
    emoji: 🗣️
    os:
    - linux
    - darwin
    - win32
    displayName: Speak
    configPaths:
    - ~/clawic/speak/
---

Turns written agent replies into speech-ready text for any TTS engine and adapts voice, rate, and phrasing to the user over time. Learned preferences live in `~/clawic/speak/preferences.md`; the skill reads and writes only that folder.

## When To Use

- Any reply that a TTS engine will read aloud, in any runtime with a voice channel
- Converting existing text (docs, briefs, lists, code output) into listenable speech
- Configuring or switching TTS voice, rate, or pronunciation after user feedback
- Building spoken briefings, confirmations, or notifications
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
| Engine mispronounces a word | Add lexicon or phoneme fix once, permanently |
| Content over 150 words | Chunk by topic; end each chunk with a check-in question |
| TTS engine errors or unavailable | Fall back to text and say so in one line |
| Anything else | Persona default voice, rate 1.0, plain sentences, under 60 seconds |

## Core Rules

1. **Budget speech in seconds, not words.** Spoken seconds = word count / 2.5 (English conversational average is 140-160 words per minute). A 150-word reply is about 60 seconds; that is the cap for uninterrupted agent speech. 375 words = 2.5 minutes = a monologue nobody requested.
2. **Answer in the first sentence.** Listeners cannot skim or rewind. Order: verdict, then reasoning, then caveats. Check: sentence one alone would work as the entire reply.
3. **Sentences of 20 words or fewer, one clause deep.** Test: read it aloud in one breath. Nested clauses that work on a page force the listener to hold state they will drop.
4. **Normalize every non-word token before synthesis.** Numbers, dates, units, acronyms, URLs, and symbols each need a decision (Normalization Rules below). An unnormalized token is a pronunciation coin flip you did not call.
5. **Round for the ear.** Non-confirmable numbers to 2 significant figures: 1,247,893 becomes "about 1.2 million". Confirmable data (codes, phone numbers, amounts to be charged) is the exception: exact, digit by digit.
6. **One voice per persona.** Users bind identity to the voice; a switch reads as a different agent. Change voice only on explicit request; change rate only after a user signal.
7. **Two signals make a preference.** First occurrence: comply this session. Second consistent occurrence: confirm aloud in one sentence, then store it. Storing on the first signal fossilizes a one-off mood into permanent config.
8. **Degrade with notice, never by dropping output.** TTS failure, unsupported markup, or over-budget content falls back to text with a one-line notice. A silent drop teaches the user the voice channel is unreliable.

## Writing For The Ear

Rewrite, do not filter. Speech-ready text is a different artifact from screen text:

- Kill all markup: `*`, `_`, backticks, `#` headers, tables, links. Engines read them literally ("asterisk asterisk") or drop them mid-word. Links: speak the site name, keep the URL in the text channel.
- Emoji are read by name ("face with tears of joy"). Delete them; carry tone in word choice.
- Bullets become enumeration with signposting: "Three things. First... Second... Third." Announce the count so the listener knows when you are done.
- Tables become comparisons: "X costs 40 dollars; Y costs 60 but includes support." More than 3 rows: speak the winner, offer the table in text.
- Homographs: engines guess tense and part of speech for "read", "live", "lead", "record", "bass". If the guess is wrong, rewrite the sentence ("I have finished reading") instead of retrying the same string.
- Questions to the user go last and alone. A question buried mid-monologue never gets answered.

## Normalization Rules

| Token | Speak as | Example |
|---|---|---|
| Large number | 2 sig figs + magnitude word | 1,247,893 -> "about 1.2 million" |
| Money | amount + currency word | $5.99 -> "5 dollars 99", or "about 6 dollars" |
| Phone number | digit groups, pause per group | 555-0142 -> "5 5 5, 0 1 4 2" |
| OTP or code | single characters with pauses | 8G4T -> "8. G. 4. T." |
| Date | spoken form, no raw digits | 2026-07-23 -> "July 23rd", year only if ambiguous |
| Time | 12-hour with am/pm unless user uses 24-hour | 14:30 -> "2 30 pm" |
| Percent | the word "percent" | 12.5% -> "12 and a half percent" |
| Acronym spoken as a word | leave as is | NASA, RAM |
| Acronym read letter by letter | space or dot the letters | FBI -> "F B I"; SQL -> "S Q L" or "sequel" per user |
| Unit | full word, correct plural | 3km -> "3 kilometers"; 1ms -> "1 millisecond" |
| URL or email | site or handle name only | "on github dot com"; full string goes to text |
| Version number | digits with "point" | v2.14 -> "version 2 point 14" |
| File path or code identifier | describe, never spell | `config.yml` -> "the app config file" |

## Prosody And Engine Config

- Rate: default 1.0. Briefings and re-listens tolerate 1.1 to 1.25 on request. Above roughly 1.5, retention of numbers and names collapses; cut words instead of adding speed.
- Punctuation is the portable prosody control: comma = short pause, period = full stop. For explicit control use SSML `<break time="300ms"/>` between digit groups and topic shifts.
- SSML portability is the trap, not the syntax. Some engines accept full SSML (`say-as`, `phoneme`, `prosody`), some honor only `break`, some accept none and read the tags aloud as text. Test one tag on the target engine before templating many; on failure, fall back to punctuation and rewriting.
- Escape `&`, `<`, and `>` in any SSML payload; one bare ampersand fails the whole request on strict parsers.
- Pronunciation fixes in preference order: engine lexicon entry, then `<phoneme>` tag, then phonetic respelling in the speech string only ("engine x" for Nginx). Respellings must never leak into the visible text channel.
- Interactive use: synthesize sentence by sentence and start playback on the first completed sentence. Waiting for full-reply synthesis adds the entire generation time to perceived latency.

## Preference Memory

Store in `~/clawic/speak/preferences.md`, one line per confirmed preference:

```
voice: <provider>: <voice-id>
rate: 1.15 (asked "faster" 2026-07-12, 2026-07-19)
lexicon: Nginx -> "engine x"; SQL -> "sequel"
style: no chunk check-ins during briefings
avoid: reading URLs aloud
```

- Write only after the two-signal rule (Core Rule 7); record the evidence dates so a later session can tell preference from mood.
- Pronunciation corrections are the exception: one correction = permanent lexicon line. Nobody corrects the same name twice for fun.
- Read the file at session start whenever a voice channel is active; apply them without mentioning them, never recite stored preferences unprompted.

## Output Gates

Before sending any string to a TTS engine:

1. Markup, emoji, and code syntax stripped or summarized?
2. Does the first sentence answer the question on its own?
3. Word count / 2.5 at or under 60 seconds, or chunked with check-ins?
4. Every number, date, acronym, and URL normalized per the table?
5. Confirmable data exact and digit-grouped; everything else rounded?
6. Any SSML verified against this engine, ampersands escaped?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Piping the text reply straight to TTS | Markup and emoji are read literally; listeners hear "asterisk" | Rewrite for the ear, every time |
| Reading code or logs aloud | Syntax has no spoken form; 10 lines of code is a minute of noise | Speak the behavior in 1-2 sentences, deliver code as text |
| Speeding up rate to fit a long reply | Above ~1.5x, numbers and names stop being retained | Cut words; the budget is seconds, not rate |
| Speaking exact big numbers | "1,247,893.42" takes several seconds to say and is not retained | 2 sig figs; exact only for confirmable data |
| Respelling words in the shared text channel | Transcript shows "engine x" garbage to readers | Lexicon or phoneme tag; respell only in speech-only strings |
| Assuming SSML is portable | Unsupported tags are read aloud as angle-bracket text | Test one tag per engine before templating |
| Switching voices for variety | Voice is identity; a switch reads as a different agent | One voice per persona, change only on request |
| Storing a preference on the first signal | A one-off mood becomes permanent config | Two consistent signals, confirm, then store |
| Burying a question mid-speech | Listeners respond to what they heard last | Question last, alone, nothing after it |

## Related Skills

- `talk` - set up the real-time two-way voice session this skill writes for
- `listen` - the input side: speech-to-text and transcription accuracy
- `audio` - process the audio files themselves: conversion, cleanup, normalization

---
Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/speak.
