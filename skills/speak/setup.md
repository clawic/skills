# Setup — Speak

Read this on first use of a voice channel to load user preferences. Do not interview the user.

## Your Attitude

Spoken output is a different artifact from screen text, and most agents ship the wrong one. You rewrite for the ear by default and save the user from robotic, unlistenable replies. Practical, brief, ear-first.

## How To Load Preferences

1. Read `~/Clawic/data/speak/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `voice: engine default`, `default_rate: 1.0`, `speech_budget: 60`, `number_style: rounded`, `time_format: 12h`, `locale: en-US`, `ssml: auto`, `checkins: true`.
3. Read `~/Clawic/data/speak/preferences.md` for the learned lexicon, style, and engine notes (`memory-template.md` for the format). Absence is fine; proceed without comment.

Work from defaults immediately. Never open with questions about voice, speed, or style.

## Recording Preferences (only when the user gives a signal)

- Pronunciation correction → one signal = permanent lexicon line (SKILL.md rule 7 exception).
- Language switch request → one signal = locale/language update (`multilingual.md`).
- Everything else (rate, style, check-ins, verbosity, notification muting) → two-signal rule (SKILL.md rule 7): comply the first time, confirm and store on the second.
- Situational requests ("just this once, faster", "serious tone for this document") → comply, store nothing.
- Declared settings (voice, rate baseline, time format, locale) → `config.yaml`; observed patterns, lexicon, and engine test results → `preferences.md`. An observation never overwrites a declared value without confirmation.

If the user has said nothing, store nothing.

## What Memory Holds

See `memory-template.md`. Track voice binding per persona, the lexicon, each engine's SSML tier (`ssml.md` test results), style and avoid lines, and context profiles (driving, quiet hours, shared spaces) — only from what the user actually reveals.
