# Calibration — Measured TDEE From the User's Own Logs

The single biggest accuracy upgrade this skill offers: after 2 weeks of data, stop trusting formulas (±10% per-person error) and compute the user's actual expenditure.

## The Formula (rule 5, canonical form)

TDEE = mean daily intake − (weekly weight change in kg × 7700 ÷ 7), weight change signed (loss = negative, gain = positive).

- Worked, losing: averages 2200 kcal/day, 7-day average down 0.4 kg/week → TDEE = 2200 − (−0.4 × 1100) = ~2640. All future targets derive from 2640, not from Mifflin.
- Worked, gaining: averages 2800, up 0.3 kg/week → TDEE = 2800 − 330 = ~2470; the planned "surplus" was 330 kcal.
- Weight change comes from comparing 7-day averages of the first and last week of the window (rule 6) — never endpoint scale readings.

## Data Hygiene (what makes a window valid)

- **14+ days minimum**, and skip week 1 of any new deficit — the 1-2 kg glycogen/water drop (`trend.md`) poisons the math.
- **Only fully-logged days count.** Unlogged days silently deflate mean intake and inflate nothing — the result reads as "slow metabolism" when it is missing data. If more than 1-2 days/week went unlogged, treat measured TDEE as an upper bound on intake accuracy, not on metabolism.
- Consistent weigh-in conditions (`trend.md` protocol); a scale swap or time-of-day change mid-window invalidates the comparison.

## Refresh Cadence

Recalibrate every 2-3 weeks while in an active deficit or surplus; monthly at maintenance. Two forces make old numbers rot:

- **Mass loss lowers expenditure** — the 7700 kcal/kg constant (Wishnofsky) is a planning tool, not a promise; real loss runs slower than it predicts over months (the NIH body weight planner models exactly this). Expect the gap; never frame it as user failure.
- **Metabolic adaptation** adds roughly 10-15% below mass-predicted expenditure in prolonged deficits (Leibel) — the mechanism behind "eat less, forever" spirals, and the argument for diet breaks (`maintenance.md`).

## When Formula and Measured Disagree Badly

A gap >25% between Mifflin TDEE and measured TDEE is almost never metabolism. Check in order:

1. **Logging drift** — untracked oils, weekend meals, drinks. Base rate is high: self-reported intake in "diet-resistant" subjects under-reported by ~47% (Lichtman, NEJM). Audit before believing anything exotic.
2. **Water masking** (`trend.md`) — new training block, high-sodium stretch, cycle phase sitting on top of real fat change.
3. **Wrong activity multiplier** on the formula side — the most common benign explanation (`targets.md`).
4. Only after 1-3: accept the measured number. The user's data, cleanly collected, outranks every formula and every intuition — including theirs and yours.

## What Calibration Unlocks

- Targets that survive plateaus: the stall protocol's step 3 (`trend.md`) is a recalibration, not a guess.
- Honest maintenance: the number to return to after a cut (`maintenance.md`) is measured TDEE at the new bodyweight, minus the adaptation caveat above.
- An end to "my metabolism is broken" conversations — replaced by a 4-step audit with a number at the end.
