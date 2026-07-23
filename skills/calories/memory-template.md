# Memory Templates — Calories

Three files under `~/Clawic/data/calories/`, each with one job. Config lives separately in `config.yaml` (SKILL.md Configuration).

## memory.md — who the user is

```markdown
# Calories Memory

## Status
status: ongoing
last: YYYY-MM-DD

## Goal & Targets
<!-- goal: cut | bulk | maintain | recomp; current target kcal + protein g; date set -->
<!-- measured TDEE: value, window it came from (calibration.md) -->

## Stats
<!-- weight (7-day avg), height, age, sex as given — only what was volunteered or needed for a target -->

## Health Context
<!-- Red Flags disclosures: meds, conditions, pregnancy, ED history — persists the guardrail (safety.md) -->

## Preferences
<!-- logging medium, goal style, food conventions, data of record, restrictions, weigh-in ritual -->

---
*Updated: YYYY-MM-DD*
```

## Status Values

| Value | Meaning |
|---|---|
| `ongoing` | Still learning their goal and habits |
| `complete` | Goal, targets, and habits established |

## library.md — the food library (the compounding asset)

One line per confirmed food, matching the format `labels.md` saves in:

```markdown
# Food Library

- Brand X protein bar: 210 kcal [per 60 g bar], 20 g protein — label, 2026-07
- Weeknight chili: 450 kcal/serving, 32 g protein — recipe math (÷6 servings), 2026-07
- Usual latte (oat, medium): ~180 kcal — user-confirmed estimate, 2026-06
```

Rules: source tag always (`label` > `recipe math` > `user-confirmed estimate` > `estimate`); a user correction overwrites and upgrades the tag; re-scanning a food already in the library is the failure `labels.md` exists to prevent.

## log.md — the daily record

```markdown
## 2026-07-23 — total ~1950 (1750-2150)
- breakfast: eggs and toast 420-520 → 470
- lunch: leftover chili (library) 450
- snack: latte (library) 180
- dinner: chain burrito bowl 700-900 → 800 [published data + guac]
- weight: 81.4 kg (7-day avg 81.7)
```

- Log the range, total the midpoints adjusted per rule 2; library hits need no range.
- **A day runs wake-to-sleep, not midnight-to-midnight** — night-shift work and 1 a.m. eating belong to the day they end. Consistency of the boundary matters more than where it sits.
- Unlogged day: enter `— unlogged` rather than nothing; calibration (`calibration.md`) must know which days to exclude, and a visible gap prevents the silent-deflation error.
