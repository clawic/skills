---
name: calories
slug: calories
version: 1.0.2
changelog: Complete rewrite with real calorie-tracking guidance
description: Tracks calories and macros from meal photos, text logs, and labels with honest error ranges, TDEE calibration, and eating disorder guardrails. Use when the user logs food, asks how many calories a meal has, wants a deficit or surplus target, or asks why the scale is not moving.
homepage: https://clawic.com/skills/calories
metadata:
  clawdbot:
    displayName: Calorie Tracker
    emoji: 🍎
    configPaths:
    - ~/clawic/calories/
---

Persistent data lives in `~/clawic/calories/` (memory plus food library). Create the folder on first use; if a legacy `~/calories/memory.md` exists, migrate it there once.

## When To Use

- User logs a meal by photo, text, or label and wants a calorie/macro estimate
- User asks for a daily calorie or protein target for loss, gain, or maintenance
- User reports weight changes and wants the trend interpreted
- User asks "how accurate is this" or whether to eat back exercise calories
- Not for meal planning or recipes (that is `dietitian` / `meal-planner` territory)
- Mode: act-as tracker for logging and math; advise-only for targets, and any Red Flags hit suspends both

## Quick Reference

| Situation | Play |
|---|---|
| Meal photo | Identify items, size portions against plate/utensils, add hidden-calorie adjustments, output a range |
| Vague text log ("had pasta") | Apply portion defaults, ask at most ONE clarifying question, output a range |
| Packaged food | Ask for one label photo, extract, save to library, reuse forever |
| Repeat meal | Match against library, confirm "same as last time?", skip re-estimation |
| Restaurant meal | Chain: look up published menu data; local: photo estimate +20-30%, flag as approximate |
| Wants a target | Mifflin-St Jeor x activity, deficit/surplus sized per Core Rule 3, floors per Rule 4 |
| 14+ days of logs exist | Replace formula TDEE with measured TDEE (Rule 5) |
| Scale stalled 2+ weeks | Plateau protocol (Reading The Weight Trend), never "just cut more" |
| Any Red Flags signal | Suspend all protocols, follow the Red Flags table |
| Anything else | Log it with a range, save to memory, zero commentary on whether the number is good or bad |

## Core Rules

1. Estimates are ranges, never single numbers: single foods run +/-10-15%, mixed dishes +/-25-40%, restaurant meals +20-30% vs homemade. "350-450" is honest; "412" is theater.
2. Round in the safe direction for the goal: weight loss rounds intake UP 10-15%, muscle gain rounds DOWN 10-15%, maintenance takes the midpoint. Estimation bias should oppose the goal's failure mode.
3. Targets come from a formula, not vibes. BMR (Mifflin-St Jeor): men 10xkg + 6.25xcm - 5xage + 5; women same - 161. TDEE = BMR x activity (1.2 sedentary, 1.375 light, 1.55 moderate, 1.725 heavy). Deficit 300-500 kcal/day, or sized to lose 0.5-1.0% bodyweight/week.
4. Hard floors: never set or endorse targets below 1200 kcal/day (women) / 1500 (men) without clinician oversight. Repeated logs below floor trigger the Red Flags table, not encouragement.
5. After 14+ days of consistent logs, measured TDEE beats any formula: TDEE = mean daily intake + (weekly weight change in kg x 7700 / 7). Formulas carry +/-10% error per person; the user's own data does not.
6. Judge 7-day rolling averages, never day-to-day scale readings: daily weight swings 1-2 kg on water and glycogen alone (each gram of glycogen binds ~3 g water).
7. Protein is the second number that matters: 1.6-2.2 g/kg bodyweight during a deficit (Morton meta-analysis); below that, the deficit eats muscle, not just fat. Sedentary maintenance can run at the 0.8 g/kg RDA.
8. Screen before tracking: run the Red Flags table on first contact and on every concerning signal. A calorie tracker in the wrong hands is a harm amplifier.

## Estimating A Meal

Worked example for Rule 3: 80 kg, 180 cm, 35-year-old man, light activity: BMR = 800 + 1125 - 175 + 5 = 1755; TDEE = 1755 x 1.375 = ~2410; loss target = 1900-2100.

Procedure for any photo or text log:
1. Itemize foods; for photos use plate diameter (~27 cm standard) and utensils as scale references.
2. Portion via hand heuristics when weight is unknown (Precision Nutrition method): palm of cooked protein = 100-120 g (~25-30 g protein); cupped hand of carbs = ~25-30 g carbs; thumb of fat = ~10 g (~90 kcal); fist of vegetables = ~1 cup.
3. Add hidden calories, the #1 undercounting source: unknown cooking method +50-100 kcal; fried +15-20%; sauce or dressing not on the side +50-100 kcal; 1 tbsp cooking oil = ~120 kcal and is invisible in photos.
4. Apply context: restaurant +20-30% over homemade equivalent; energy from macros checks the total (4 kcal/g protein and carbs, 9 fat, 7 alcohol, Atwater factors).
5. Output the range, log the midpoint adjusted per Rule 2, save recurring items to the library.

Defaults when the user is vague: pizza slice 250-350 kcal, cooked pasta portion 1.5 cups (not restaurant-size), salad 150 kcal base plus dressing. One clarifying question maximum; if still vague, estimate and say which assumption moves the number most.

Cost discipline: text first, photo only when the dish is mixed or unfamiliar; one label photo yields permanent accurate data and beats ten photo estimates.

## Targets And Calibration

- Recalibrate every 2-3 weeks with Rule 5. Example: user averages 2200 kcal/day and loses 0.4 kg/week, so measured TDEE = 2200 + (0.4 x 7700 / 7) = ~2640. All future targets derive from 2640, not from the formula.
- The 7700 kcal/kg (3500 kcal/lb, Wishnofsky) conversion is a planning constant, not a promise: over months, real loss runs slower than it predicts because expenditure drops as mass drops (NIH body weight planner models this). Expect the gap; do not treat it as user failure.
- Prolonged deficits also add metabolic adaptation of roughly 10-15% below mass-predicted expenditure (Leibel). This is why measured TDEE must be refreshed, and why "eat less, forever" spirals.
- Surplus for muscle gain: +200-300 kcal/day over measured TDEE; gaining faster than ~0.25-0.5% bodyweight/week is mostly fat for anyone past their first year of training.
- Goal changes intensity, not math: casual users get logging with zero prompts or check-ins; loss/gain users get weekly trend reviews. Match the user's stated style; never escalate tracking intensity uninvited.

## Reading The Weight Trend

- First week of any deficit drops 1-2 kg of glycogen and water. Report it as such; counting it as fat sets up week-two disappointment.
- Compare this week's 7-day average against last week's. Two daily weigh-ins prove nothing (Rule 6 noise swamps a 500 kcal/day deficit, which moves true mass only ~0.45 kg/week).
- Stall protocol, in order: (1) confirm 2+ full weeks of flat 7-day averages, (2) audit logging for drift, since weekends and untracked oils typically hide 200-500 kcal/day, (3) recalibrate TDEE per Rule 5, (4) only then adjust intake by 100-200 kcal. Cutting first is the classic error the adaptation data punishes.
- Sustained loss faster than 1.5 kg/week raises gallstone and muscle-loss risk: raise intake toward the 0.5-1.0%/week band even if the user is pleased.

## Memory

Store in `~/clawic/calories/memory.md`, sections: Goal (loss/gain/maintenance/casual plus target), Targets (calories, protein, measured TDEE with date), Patterns (weekend drift, skipped meals), Preferences (photos vs text, summary cadence, prompt tolerance), Library (one line per saved food: `name: kcal, protein`, including label scans and "Dish @ Place" restaurant entries). Update the library automatically; surface matches when a log resembles a saved item.

## Red Flags

| Signal (observable) | Suspicion | Action |
|---|---|---|
| Logs below floor (1200 W / 1500 M) on 3+ days in a week | Unsupervised over-restriction | Pause targets, state the floor and why, suggest clinician review |
| Guilt or shame language, panic over imprecise entries, logging every gram | Disordered eating pattern | Stop tracking entirely; share NEDA 1-800-931-2237 (US) / BEAT 0808 801 0677 (UK) |
| Skipping meals then large uncontrolled eating, or exercise framed as punishment for food | Binge-restrict cycle | Stop tracking, no calorie feedback, route to clinician |
| Mentions pregnancy or breastfeeding | Deficit unsafe for fetus/infant | Decline deficit tracking; neutral logging only if their clinician approved it |
| Diabetes on insulin, kidney disease, thyroid or metabolism-affecting meds | Targets require medical coordination | Track only against clinician-set numbers, never self-derived ones |
| Under 18, or stated stats give BMI under 18.5 | Growth needs / underweight | Decline deficit tracking, route to clinician or pediatrician |
| Losing more than 1.5 kg/week for 2+ weeks | Gallstone and muscle-loss risk | Recommend raising intake; clinician if it continues |

Anything in this table suspends every protocol above: route to a clinician.

## Output Gates

Before sending any tracking reply, check:
- Is every estimate a range with context adjustments applied, not a lone exact number?
- Does any target I state clear the 1200/1500 floor and derive from formula or measured TDEE?
- Am I citing a 7-day average for any trend claim, not two scale readings?
- Did I ask about cooking fat and drinks on this savory or restaurant log?
- Is my reply free of praise or criticism of the day's total, and free of good/bad food labels?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Treating nutrition labels as exact | US labeling tolerance allows ~20% deviation | Keep a mental 10-20% margin; labels are the best single source, not ground truth |
| Crediting wearable exercise burns at face value | Wrist devices overestimate energy expenditure by ~27% or more (Shcherbina, Stanford) | Eat back at most 50% of any reported burn |
| Cutting intake at the first stall | Water retention and logging drift mask real progress for 1-2 weeks | Run the stall protocol: confirm, audit, recalibrate, then adjust |
| Escalating precision (gram-weighing everything) | Inputs carry 15-40% error, so gram precision is fake accuracy and an obsession on-ramp | Ranges plus library reuse; precision only where it is cheap (labels) |
| Moralizing foods as good/bad or clean/dirty | Drives hiding, guilt, and binge-restrict cycles | Neutral logging; context ("high for its satiety") over judgment |
| Ignoring liquids and cooking fat | Oil (~120 kcal/tbsp), lattes (150-250), alcohol (7 kcal/g) are the biggest silent gap | Prompt once per log for drinks and cooking method |
| Applying formula TDEE forever | Individual variance is +/-10% and adaptation grows over time | Measured TDEE from logs after week 2, refreshed every 2-3 weeks |

## Related Skills

- `dietitian` - turning targets into actual meal plans and timing
- `nutrition` - micronutrients and full dietary tracking beyond calories and macros
- `gym` - the training side of a recomposition, surplus, or cut
- `fasting` - eating windows and fast tracking when the user time-restricts

---

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/calories.
