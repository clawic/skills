# Targets — Setting Calories and Macros

Every target derives from a formula (rule 3) or the user's own data (rule 5, `calibration.md`). Floors from rule 4 apply to everything on this page.

## BMR: Two Formulas

- **Mifflin-St Jeor (default):** 10×kg + 6.25×cm − 5×age, +5 men / −161 women.
- **Katch-McArdle (when body-fat % is known):** 370 + 21.6 × lean mass kg — better for very muscular or very lean users, where Mifflin's population average drifts.
- Worked example: 80 kg, 180 cm, 35-year-old man → Mifflin BMR = 800 + 1125 − 175 + 5 = 1755. Same man at 20% body fat → Katch = 370 + 21.6×64 = 1752. Agreement like this is typical; a >150 kcal gap means the body-fat estimate is probably off.

## Activity Multipliers — the Honest Read

| Level | Multiplier | Reality check |
|---|---|---|
| Sedentary | 1.2 | Desk job, no training. Also the right pick when planning to eat back exercise (`exercise.md`) |
| Light | 1.375 | Desk job + training 3-4×/week. Where most gym-going office workers actually belong |
| Moderate | 1.55 | Physical job, or training ~daily |
| Heavy | 1.725 | Physical job + hard training, or athlete volume |

Users systematically pick one level too high. When unsure, choose the lower level — an underestimate self-corrects upward via hunger and the 2-3 week recalibration (`calibration.md`); an overestimate produces "I'm in a deficit but not losing".

Example continued: 1755 × 1.375 ≈ 2410 TDEE; loss target 1900-2100 (deficit ~300-500).

## Sizing the Deficit or Surplus

- **Deficit:** 300-500 kcal/day default, or from rate: daily deficit = target %/week × kg × 7700 ÷ 700. At 80 kg targeting 0.75%/week → 0.6 kg/week → 660 kcal/day. Stay inside the 0.5-1.0%/week band (rule 3); leaner users sit at the low end — less fat available means more muscle in every kg lost.
- **Surplus:** +200-300 kcal/day over measured TDEE. Gaining faster than ~0.25-0.5% bodyweight/week is mostly fat for anyone past their first year of training; first-year trainees can ride the top of that band.
- **Maintenance:** measured TDEE once it exists, formula TDEE until then; managed as a band, not a number (`maintenance.md`).
- Aggressive requests (>1%/week, "1000 kcal deficit"): cap at the band top, state why (muscle loss, adherence collapse, the 1.5 kg/week Red Flags row), and let the user decide within the safe range.

## Macros

- **Protein first:** 1.6-2.2 g/kg in a deficit, 0.8 g/kg RDA at sedentary maintenance (rule 7; 65+ and BMI 30+ adjustments below). In a surplus, 1.6 g/kg is plenty; more is fine but buys nothing measured. If optimizing distribution: ~0.4 g/kg per meal across 3-4 meals (Schoenfeld-Aragon), though daily total dominates timing.
- **Fat floor:** ~0.5 g/kg minimum — a coaching convention for hormonal health, not an RDA; going below it is the classic error of protein+carb-only cutting.
- **Carbs:** the remainder. No metabolic magic either direction at equal calories and protein; assign by preference and training demand.
- **Fiber:** 14 g per 1000 kcal (IOM). The target nobody hits in a deficit without planning; route food choices to `dietitian`.
- Macro-split requests ("40/30/30?"): translate to g/kg and check protein and fat floors — a percentage split that breaks a floor at low calories is wrong even if the percentages look canonical.

## Special Populations (formula adjustments that matter)

- **65 and older:** protein rises to 1.0-1.2 g/kg even at sedentary maintenance (PROT-AGE) — the 0.8 RDA exception in rule 7 stops applying with age, because anabolic resistance means the same gram buys less muscle retention. Deficits stay shallow (300 kcal side of the band): muscle lost past 65 is the hardest to rebuild.
- **BMI 30+:** compute protein from goal weight, not current weight — 1.6-2.2 g/kg of current mass at high BMIs produces inflated, unmeetable gram counts; the lean tissue being protected scales closer to goal weight. Calorie formulas still use current stats.
- **Menopause:** expenditure and body-composition shifts are real but modest; the practical answer is not a different formula — it is faster recalibration (`calibration.md` every 2-3 weeks) plus the 65+ protein logic arriving early. Never attribute a stall to menopause before running the stall protocol (`trend.md`); the audit order does not change.
- **Very lean or very muscular:** Katch-McArdle (above) plus measured TDEE as early as the data allows; population formulas drift most at the physique extremes.

## "Do Calories From Protein Count Less?"

Partially yes — the thermic effect of food: digesting protein costs 20-30% of its calories, carbs 5-10%, fat 0-3%. Two consequences, one warning: a protein-forward deficit genuinely burns slightly more at equal intake (a real tiebreak on top of rule 7's satiety and muscle-retention case), and measured TDEE (`calibration.md`) already contains the user's actual TEF — so never add TEF math on top of a calibrated target. That would be the same double-count family as exercise (`exercise.md`).

## Sanity Gates Before Stating Any Target

- Stats give BMI <18.5, age <18, pregnancy, or a listed condition → Red Flags table, not a target.
- Computed target below 1200/1500 → raise to the floor and slow the expected rate; never hand out a sub-floor number "just this once".
- User already has 14+ days of logs → skip the formula entirely, calibrate (`calibration.md`).
- Goal is recomposition, not scale movement → `maintenance.md` (the scale is the wrong instrument).
