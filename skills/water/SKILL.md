---
name: water
slug: water
version: 1.0.2
description: Tracks daily water intake from casual mentions with container calibration, weight-based targets, and exercise and heat adjustments. Use when the user logs drinks, asks about hydration targets, dehydration signs, urine color, or fluid replacement during exercise.
homepage: https://clawic.com/skills/water
metadata:
  clawdbot:
    emoji: "💧"
    displayName: Water Tracker
    configPaths:
    - ~/clawic/water/
    changelog: Complete rewrite with real hydration targets
---

Hydration tracking and advice from conversational mentions. Learned data (container sizes, patterns, preferences) persists in `~/clawic/water/memory.md`; this skill reads and writes only that folder.

## When To Use

- User mentions drinking anything ("had water with lunch", "finished my bottle") and wants intake tracked
- User asks how much water they should drink, or whether they drink enough
- User reports thirst, dark urine, headache, or fatigue and hydration is a plausible factor
- User plans or logs exercise, sauna, or hot-weather activity and asks about fluid replacement
- Mode: act-as tracker (log without commenting) plus advise (targets and adjustments) when asked
- Not for meal or calorie logging (route to calories) and not for diagnosing illness (see Red Flags)

## Quick Reference

| Situation | Play |
|---|---|
| Drink mentioned, no size given | Log the default size for that container type (see Logging), note the assumption in the log |
| First mention of a personal container ("my bottle") | Ask its size once, store in memory, never ask again |
| User asks "how much should I drink" | 30-35 ml per kg body weight of fluids per day (→ Core Rule 1); ask weight if unknown |
| Exercise or sports session mentioned | Add 500-1000 ml per sweaty hour to today's target; offer the sweat-rate test if they train regularly |
| Hot day (above 30 C) or long sun exposure | Raise today's target 250-500 ml, without announcing it |
| Headache or fatigue mentioned | Check today's log first; only if intake is below half of target by mid-afternoon, suggest water once |
| Active fasting window | Water, black coffee, and plain tea do not break a fast; on fasts over 24 h add electrolytes (→ fasting skill) |
| Vomiting or diarrhea | Oral rehydration solution, not plain water; run the Red Flags table |
| Any other hydration mention (default) | Log without comment, with timestamp and estimated ml; no comment, no reminder, no target talk |

## Core Rules

1. Daily fluid target = 30-35 ml x body weight in kg. Worked example: 70 kg → 2100-2450 ml. Weight unknown → default 1600 ml (women) / 2000 ml (men), the ~80% drinking-fluid share of EFSA adequate total-water intake (2.0 L women / 2.5 L men, food water included). Never quote 3.7 L or 2.7 L as a drinking target: those NAM figures are total water including the 20-30% that comes from food.
2. Count all non-alcoholic drinks at 100%: coffee, tea, milk, soup, sparkling water. Up to 400 mg caffeine per day produces no net fluid loss in habitual drinkers (Armstrong). Alcohol counts as 0 ml.
3. Estimate first, ask later. Use the default container table, flag the entry as estimated, and only ask when the ambiguity changes the day total by more than 20% ("bottle" could be 500 or 1000 ml → ask; "a glass" → just log 250).
4. Judge status by urine, not by totals hit. Pale straw = on target regardless of ml logged; dark (5 or above on the 8-level Armstrong chart) at midday = deficit, drink 500 ml now; completely colorless at every void = overshooting, ease off. First morning urine is always dark and proves nothing.
5. Thirst is a lagging signal: it appears at roughly 1-2% body mass already lost, and measurable performance drop starts near 2% (ACSM). For planned long efforts, schedule intake instead of waiting for thirst. Adults over 65 have blunted thirst: schedule for them by default.
6. Cap intake rate at about 1 L per hour sustained; the gut absorbs little more, and plain water beyond sweat losses dilutes sodium. Body weight GAIN during exercise = overdrinking, stop fluids (exercise hyponatremia hit 13% of Boston Marathon finishers in the Almond study).
7. History of kidney stones changes the target: drink enough to produce at least 2.5 L urine per day (AUA), which means roughly 3 L intake. This overrides Rule 1 upward.

## Logging And Estimation

Default sizes when the user does not specify (store overrides in memory after one calibration):

| Container | Default |
|---|---|
| Glass, cup of water | 250 ml |
| Mug (tea, coffee) | 300 ml |
| Small bottle, can | 330-500 ml (can 330, bottle 500) |
| Large bottle | 1000 ml |
| Restaurant glass with refills | 250 ml per mention, count refills only if stated |
| "Some water", "a sip" | 100 ml |

Memory file `~/clawic/water/memory.md` sections: `Containers` (name: ml), `Schedule` (detected habits, e.g. "water after coffee, AM"), `Correlations` (e.g. "gym days: +500 ml"), `Preferences` (e.g. "no reminders, weekly summary only"), `Baseline` (weight, computed target, medical overrides). Update on new evidence; never re-ask what is stored.

Reporting default: silent logging, summary only when asked or weekly if the user opted in. Never send missed-glass reminders; nagging is the main reason users abandon hydration tracking.

## Targets, Heat, And Exercise

- Heat: above 30 C ambient or direct sun work, add 250-500 ml per day of exposure; hard labor in heat can sweat 1-2 L per hour, so treat it as exercise, not weather.
- Exercise flat adjustment: +500-1000 ml per hour of sweaty activity, drunk during and after. During sessions over 60-90 min, use a drink with sodium (typical sports drinks carry 400-700 mg sodium per liter) instead of plain water.
- Sweat-rate test for regular trainers: sweat rate (L/h) = (pre-weight kg - post-weight kg + fluid drunk L) / hours. Example: 72.0 kg before, 71.2 after a 1 h run while drinking 0.5 L → (0.8 + 0.5)/1 = 1.3 L/h. Replace 125-150% of the net mass lost over the next 2-4 hours (0.8 kg lost → 1000-1200 ml), not all at once.
- Fever, vomiting, diarrhea: losses are water plus electrolytes; plain water alone worsens the sodium balance. Recommend WHO-style oral rehydration solution and run the Red Flags table before anything else.
- Illness, pregnancy, or breastfeeding mentioned: raise the target only with a clinician-sourced number; do not extrapolate from Rule 1.

## Red Flags

| Signal (observable) | Suspicion | Action |
|---|---|---|
| No urination for 8+ h despite drinking, dizziness on standing, very dry mouth | Severe dehydration | Same-day clinician; emergency care if confusion or fainting |
| Headache, nausea, bloating, or confusion during or after a long endurance event with heavy drinking | Exercise-associated hyponatremia | Emergency care; do NOT give more water |
| Extreme thirst plus urinating far more than usual, persisting over a week | Diabetes (mellitus or insipidus) | Clinician within days, mention both symptoms |
| Dark urine with yellowed skin or eyes, or flank pain | Liver or kidney problem, not simple dehydration | Clinician; urgent if pain is severe |
| Vomiting or diarrhea beyond 24 h in a child or elderly person, or fluids not staying down | Dehydration risk in a vulnerable person | Same-day care; oral rehydration solution meanwhile |
| User has heart failure, kidney disease, cirrhosis, or takes lithium or diuretics, and asks to raise intake | Fluid restriction may apply; formulas here are unsafe | Clinician sets the target; suspend Rules 1-7 |

Anything in this table suspends the protocols above: route to a clinician.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Pushing "8 glasses" or a flat 2 L on everyone | The famous figures are total-water averages that include food water; a 55 kg and a 95 kg user differ by over a liter | Weight formula (Rule 1) plus activity adjustments |
| Counting only plain water | Coffee, tea, milk, and soup hydrate at effectively 100%; excluding them undercounts by 30-50% for many users | Log all non-alcoholic fluids (Rule 2) |
| Asking container size on every log | Each question adds friction; users stop reporting within days | Calibrate once, store in memory, reuse forever |
| Answering every headache with "drink more water" | Post-race headache with heavy drinking can be hyponatremia; more water is the wrong direction | Check the day's log and context first (Quick Reference row) |
| Reading dark first-morning urine as dehydration | Overnight concentration is normal physiology | Judge from a midday sample (Rule 4) |
| Sending reminder pings for missed intake | Nagging converts a passive habit into a chore; abandonment follows | Silent logging; surface trends weekly or on request |
| Applying the standard formula to users with heart or kidney conditions | These conditions can require fluid restriction; the formula can cause harm | Red Flags last row: clinician sets the number |

## Related Skills

- fasting: eating-window rules and electrolyte protocol for fasts over 24 h
- calories: meal and drink logging when the user tracks food, not just fluids
- running: race-day fueling and pacing where fluid plans meet training plans
- gym: workout logging that feeds the exercise adjustment automatically

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/water.
