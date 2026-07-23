# Exercise Calories — Wearables, Eat-Back, and Double Counting

Exercise calories are the most over-credited number in tracking. Two honest accountings exist; the errors come from inflating the burn or counting it twice.

## Pick One Accounting (never both)

| Accounting | How it works | Best for |
|---|---|---|
| Activity-inclusive multiplier | Training is already inside the 1.375-1.725 multiplier (`targets.md`); ignore per-workout burns entirely | Consistent training schedules — simplest, hardest to game |
| Sedentary base + eat-back | TDEE at 1.2, then add back ≤50% of each workout's reported burn | Erratic schedules, endurance athletes with big session variance |

Using an active multiplier AND eating back workouts counts every session twice — the #1 reason "I'm in a deficit" coexists with a flat trend line (SKILL.md Traps).

## Why ≤50%

- Wrist wearables overestimate energy expenditure by ~27% or more (Shcherbina, Stanford) — canonical number, also in SKILL.md Traps.
- Cardio machine displays inflate too (they credit gross burn, generous assumptions, no body-composition input); treat every displayed number as an upper bound.
- Reported burns are GROSS (they include the resting calories that hour would have burned anyway); eat-back logic wants NET.
- Halving the reported number absorbs all three errors with a margin. Eating back 100% of a wearable number converts most moderate deficits to maintenance.

## Honest Rules of Thumb (when there is no device, or to sanity-check one)

- Running: ~1 kcal per kg per km, gross. 80 kg × 5 km ≈ 400 kcal.
- Walking: about half of running — ~0.5 kcal/kg/km. 10,000 steps ≈ 7-8 km → the same formula, ~280-320 kcal for 80 kg.
- Strength training: 200-300 kcal/hour for a typical session — far less than it feels like and less than most watches claim. Its value in a deficit is muscle retention (rule 7's partner), not burn.
- Swimming/cycling: too technique- and intensity-dependent for a clean constant; use the device number ×0.5 like everything else.

## NEAT — the Invisible Half

Non-exercise activity (walking, fidgeting, standing) often exceeds workout burn, and it silently DROPS in a deficit — the body economizes without asking. Practical countermeasure: a daily step floor (e.g. hold whatever the user's normal is) so adaptation can't hollow out expenditure unnoticed. A falling step count during a stall is evidence for the audit in `trend.md`.

## Framing Guardrails

- Exercise added to a plan raises TDEE for the NEXT calibration window (`calibration.md`); it is not a same-day spending account.
- "I earned this meal" / "burning off" specific foods is punishment-adjacent framing — one mention is a teaching moment (food and training are separate ledgers); a pattern is a Red Flags signal (SKILL.md).
- Training questions themselves — programming, progression, recovery — route to `gym`.
