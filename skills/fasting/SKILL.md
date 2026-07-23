---
name: fasting
slug: fasting
version: 1.0.2
description: Tracks intermittent fasting, extended fasts, and eating windows with fasting-stage timing and safety thresholds. Use when logging fasts, planning 16:8 or OMAD, or breaking a fast.
homepage: https://clawic.com/skills/fasting
changelog: Complete rewrite with real fasting protocols and red flags
metadata:
  clawdbot:
    emoji: "⏳"
    displayName: Fasting Tracker
    configPaths:
    - ~/clawic/fasting/
---

## When To Use

State lives at `~/clawic/fasting/memory.md` (protocol, window, streak, symptom timing). Only write on an explicit user signal; never infer a fast from silence.

Mode: **act-as** (you log fasts and compute elapsed time directly; you do not prescribe medical treatment).

- User announces a fast boundary: "starting my fast", "last meal was 8pm", "breaking now".
- User wants a protocol picked or an eating window planned (16:8, OMAD, 5:2, 24h+).
- User asks how long they have fasted or which fasting stage they are in.
- User is mid extended fast (24h+) and needs electrolyte or break-fast guidance.
- Not for: prescribing fasting to treat a diagnosed condition, or overriding a clinician's plan. Not for anyone in the Red Flags table.

## Quick Reference

| Situation | Play |
|---|---|
| "Last meal at 8pm" | Set fast start = 20:00; elapsed = now - 20:00; report hours + current stage |
| Black coffee / plain tea / water mid-fast | Log as 0 kcal; do NOT reset the clock |
| Bulletproof coffee, gum, diet soda, BCAAs | Treat as a break for a strict fast (>~10 kcal or sweet taste); flag it, let user decide |
| Fast reaches 24h | Prompt electrolytes (sodium/potassium/magnesium); confirm intent to continue |
| Breaking a fast <24h | Normal meal is fine; start with protein to blunt glucose spike |
| Breaking a fast 24-48h | Modest portion (~1/2 normal), protein/fat first, wait 30-60 min before more |
| Breaking a fast >=48h | Small portion (~1/4 normal), protein/fat first, wait 30-60 min before more |
| User breaks early | Log neutrally: "logged, 14h"; no streak-loss framing, no encouragement to push on |
| Ramadan / religious dry fast | No water rule applies; suspend hydration prompts; treat sunset as break |
| Any Red Flags signal | Stop protocol advice; run the Red Flags action |
| Default (unclear input) | Ask one question: "start, break, or status?" then log |

## Core Rules

1. **Clock starts at last caloric intake, not bedtime.** `elapsed = now - last_kcal_timestamp`. A 20:00 last meal and a 12:00 first meal = 16h, not "8 hours of sleep". Water, black coffee, plain tea = 0 kcal and never reset it; anything above ~10 kcal or sweet-tasting does.
2. **Log only on an explicit signal.** No proactive "did you eat?" pings. Absence of a message is not data. Prevents the tracker nagging that makes users abandon it.
3. **Stage times are estimates with wide variance; always hedge and give the number.** Ketosis onset and glycogen depletion shift with the last meal's carb load, activity, and muscle mass. Say "~16h, likely early ketosis" not "you are in ketosis".
4. **Extended fast (>=24h) makes electrolytes mandatory, not optional.** Typical daily target during water fasting: sodium 3-5 g, potassium 1-3.5 g, magnesium 300-500 mg. Under-supplementing sodium is the usual cause of the headache/dizziness/palpitation cluster, not "detox".
5. **Break size scales inversely with how well the gut is fed.** Fast <24h: eat normally, protein first. Fast 24-48h: first meal ~1/2 of a normal portion, protein/fat over refined carbs, then wait 30-60 min before more. Fast >=48h: first meal ~1/4 of a normal portion, protein/fat over refined carbs, then wait 30-60 min. The longer the fast, the higher the refeeding risk from a large carb bolus.
6. **Diabetic on insulin or sulfonylurea: hypo threshold is glucose <70 mg/dL (3.9 mmol/L).** Break immediately with 15 g fast carb at or below that. These drugs plus fasting stack hypoglycemia risk; this is a Red Flag, not a coaching moment.
7. **Never push a longer fast.** Their target is the ceiling. Breaking at hour 14 of a planned 16 is logged as a completed 14h fast, full stop.

## Fasting Stage Timeline (estimates, individual variation is large)

| Elapsed | Dominant state | What it means for logging |
|---|---|---|
| 0-4h | Fed / absorptive | Digesting; insulin elevated |
| 4-12h | Post-absorptive | Liver glycogen supplying glucose |
| 12-16h | Glycogen depleting | Fat oxidation ramps; hunger often peaks here |
| 16-24h | Ketosis building | BHB commonly >0.5 mmol/L; hunger often eases |
| 24-48h | Deep ketosis | Growth hormone elevated; electrolytes now critical |
| 48-72h+ | Prolonged | Autophagy markers reported in this window in fasting research, but human evidence is thin and mostly indirect; label it a hypothesis, not a promise |

Hunger comes in waves tied to old meal times, not a rising line. Tell the user the 12-16h wave passes in ~20-30 min; this is the single most actionable fact for a beginner.

## Breaking A Fast And Electrolytes

- Protein or fat first blunts the post-fast glucose spike more than fiber alone; a lean-protein or eggs start beats fruit juice.
- After 72h+ or in an underweight user, refeeding syndrome (phosphate/potassium/magnesium crash) is a real risk: break small, and route to a clinician for any fast this long.
- During the fast, water alone can dilute sodium; salt is the lever. A pinch of salt in water fixes most day-2 headaches faster than "wait it out".
- Ramadan and other dry fasts forbid water: do not prompt hydration during the fasting window; front-load fluids and salt at suhoor and iftar instead.

## Red Flags

Anything in this table suspends the protocols: run the Action, then route to a clinician.

| Signal (observable) | Suspicion | Action |
|---|---|---|
| Fainting, or dizziness on standing | Orthostatic drop / low electrolytes | Break fast now; salt + water; sit down |
| Heart palpitations or irregular beat | Electrolyte imbalance (K/Mg) | Break fast; electrolytes; escalate if persists |
| Glucose <70 mg/dL on a diabetic | Hypoglycemia | Break immediately, 15 g fast carb, recheck in 15 min |
| Confusion, slurred speech, severe headache | Hypoglycemia / hyponatremia | Break; do not fast unsupervised again; clinician |
| Fast pushed past 72h without supervision | Refeeding / electrolyte risk | Stop encouraging; advise medical oversight |
| Purging, fasting to punish eating, hiding food, or a falling weight trajectory | Disordered eating | Stop tracking; give current support (text "NEDA" to 741741, Crisis Text Line); do not log |
| Pregnant or breastfeeding + fasting | Contraindicated | Do not track fasts; refer to their provider |

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Counting hours from bedtime | Ignores the last meal; overstates the fast by hours | Anchor the clock to `last_kcal_timestamp` (Rule 1) |
| "You're in autophagy at 16h" | Overstates thin human evidence; sets a false expectation | Say "~16h, ketosis building", label autophagy a hypothesis |
| Treating a broken fast as failure | Guilt framing drives users to quit the tracker | Log neutrally; report the hours completed (Rule 7) |
| Prompting more water for a headache on day 2 | Dilutes sodium further; makes it worse | Prompt salt, not just water (Rule 4) |
| Same break-fast advice for 16h and 60h | A big carb meal after a long fast risks refeeding symptoms | Scale portion to fast length (Rule 5) |
| Cheering a diabetic through low glucose | Ignores drug-stacked hypo risk | Break at <70 mg/dL; treat as Red Flag (Rule 6) |

## Related Skills

- `nutrition` when the user shifts focus to what to eat inside the window (macros, micros).
- `calories` when the eating-window meals need calorie or macro totals tracked.
- `water` when hydration or electrolyte intake becomes the primary thing to log.
- `meal-planner` when the eating window needs a weekly menu or batch-cook plan.

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/fasting.
