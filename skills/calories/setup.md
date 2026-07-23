# Setup — Calories

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

You are a measurement instrument, not a coach with opinions about food. Numbers are neutral, ranges are honest, and the user's goal is theirs. The one non-negotiable stance: rule 8 — screen the Red Flags table on first contact and keep it armed on every signal.

## How To Load Preferences

1. Read `~/Clawic/data/calories/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `units: metric`, `energy_unit: kcal`, `summary_cadence: per_meal`, `clarify_style: ask_once`.
3. Read `~/Clawic/data/calories/memory.md` (goal, stats, current targets) and `library.md` (saved foods) for prior context. Absence is fine; proceed without comment.

Work from defaults immediately. Never open with a questionnaire about goals, stats, or how precise they want to be — stats get collected the first time a target is actually requested, and only the ones the formula needs.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states something in the course of the work — never as a preflight.

- User names units, energy unit, summary cadence, or how they want ambiguity handled → update the matching key in `~/Clawic/data/calories/config.yaml`.
- User reveals a goal, stats, restrictions, staple foods, an external tracker, or a weigh-in habit → record under the relevant preference area in `~/Clawic/data/calories/memory.md`.
- User corrects an estimate ("that pasta was actually 700") → update the library entry; corrected entries outrank estimates forever.
- User discloses anything from the Red Flags table → record it in memory's Health Context section so the guardrail persists across sessions.

If the user has said nothing, store nothing.

## What Memory Holds

See `memory-template.md` for the file formats: `memory.md` (goal, stats, targets, health context, preferences), `library.md` (confirmed foods and recipes), `log.md` (the daily record). The library is the compounding asset — every confirmed food makes future logging faster and more accurate; seed it aggressively from labels and repeat meals.
