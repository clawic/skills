---
name: fitness
slug: fitness
version: 1.0.2
description: Plans and adjusts fitness training with quantified progression rules, volume landmarks, deload triggers, and recovery heuristics. Use for program design, plateau diagnosis, missed-session resets, cardio zoning, and return-to-exercise decisions.
homepage: https://clawic.com/skills/fitness
metadata:
  clawdbot:
    displayName: Fitness
    emoji: 💪
    configPaths:
    - ~/clawic/fitness/
    changelog: 'Complete rewrite with real training programming rules'
---

Training logs, preferences, and derived baselines (e1RMs, resting HR, attendance rate) persist in `~/clawic/fitness/` as plain markdown. Advise mode: this skill coaches a human who performs the training; escalation thresholds live in the Red Flags table.

## When To Use

- Designing or adjusting a training program: strength, muscle, endurance, or general conditioning
- Diagnosing a stalled lift, pace, or body-composition plateau
- Deciding what to prescribe after missed sessions, illness, or a multi-week layoff
- Turning workout logs or wearable trends into a concrete next-session prescription
- Not for meal-level nutrition planning (dietitian/calories) or rep-by-rep set logging (gym)

## Quick Reference

| Situation | Play |
|---|---|
| Goal: strength | 3-6 reps at 80-90% e1RM, 3-5 min rest, 2-4 compound lifts per session, each lift 2-3x/week |
| Goal: muscle | 6-12 reps at RIR 1-3, 10-20 hard sets per muscle per week, each muscle trained 2x/week |
| Goal: endurance | 80/20 split (Seiler): 80% of weekly minutes conversational, 20% hard |
| Goal: fat loss | Keep lifting heavy to preserve muscle in a deficit; deficit math routes to `calories` |
| Returning after 2+ weeks off | Apply the layoff discount (Rule 6), rebuild over roughly as many weeks as were missed |
| Lift stalled 3 sessions | Run the diagnostic order in Progression and Plateaus before changing anything |
| Pain reported | Check Red Flags first, then the DOMS-vs-injury test in Recovery and Readiness |
| Anything else / no stated goal | ACSM baseline: 150 min/week moderate cardio plus 2 resistance days covering all major muscles |

## Core Rules

1. **Progress by double progression, not by feel.** Assign a rep range (e.g. 3x8-12). When every set reaches the top of the range at the same load, add 2.5% for upper-body lifts or 5% for lower-body lifts and drop back to the bottom of the range. Worked example: 3x12 at 60 kg squat completed, next session is 3x8 at 63 kg. This makes overload automatic and stall detection unambiguous.
2. **Estimate max with Epley; never test it casually.** e1RM = load x (1 + reps/30). Example: 100 kg x 5 reps gives 100 x 1.167 = ~117 kg. Program percentages from e1RM; true 1RM attempts belong only in a planned peak week.
3. **Prescribe effort in reps in reserve (RIR).** Most sets at RIR 1-3; RIR 0 only on the last set of the final week before a deload. Chronic failure training roughly doubles fatigue for a similar stimulus.
4. **Volume has a floor and a ceiling.** Start near 10 hard sets per muscle per week; add 2 sets/week only while logged performance and sleep hold; past ~20 sets/week returns flatten or reverse (Schoenfeld dose-response work). A hard set ends within 3 reps of failure.
5. **Cardio: the 80% easy share is the load-bearing number.** Easy means full sentences are speakable. For HR targets use Karvonen: target = HRrest + intensity% x (HRmax - HRrest), with HRmax = 208 - 0.7 x age (Tanaka; the 220 - age formula drifts 10+ bpm off past age 40).
6. **Layoff discount by gap length.** A single missed session (gap under one week) costs nothing: skip it, resume as planned, never stack two workouts into one. One full week missed but fewer than two weeks off: resume at the last logged load with no jump, treating the return as a re-entry rather than a discount. Two or more weeks off: apply the discount, subtracting 10% of load per week missed, floor at 50%. Three weeks off a 100 kg lift means resuming near 70 kg, adding back ~10% per week.
7. **Deload on evidence, calendar as backstop.** Trigger: two consecutive sessions below logged numbers on the same lift, or resting HR 5+ bpm above baseline for 3+ mornings. Deload = 1 week at 50% of normal set count, same loads. Even without triggers, cap hard blocks at 4-8 weeks.
8. **Every prescription = exercise + sets x reps + load or RIR + rest.** "3 sets of bench" is not programmable; "3x8 at 70 kg, RIR 2, rest 3 min" can be checked against the log next session.

## Program Design Defaults

- Frequency beats session length: three 45-min full-body sessions outperform one 2-hour session at equal weekly sets, because late-session sets degrade.
- Session order: skill/power work, then the heaviest compound, then secondary compounds, isolation, conditioning last. Hard cardio before squats taxes the lift that pays most.
- Split by available days: 2-3 days = full-body; 4 days = upper/lower; 5-6 days = push/pull/legs. Program to logged attendance plus at most one day, not to stated ambition.
- Weekly movement floor: one squat pattern, one hinge, one horizontal push, one horizontal pull, one vertical push, one vertical pull; isolation fills what is left.
- Rest: 3-5 min on compound strength sets, 1.5-2 min on isolation. Cutting rest to save time cuts next-set load; superset antagonist pairs instead.
- Protein 1.6-2.2 g/kg/day supports adaptation (Morton meta-analysis); anything deeper routes to `calories` or the dietitian skill.

## Progression and Plateaus

Stall = no rep or load improvement on the same lift across 3 consecutive sessions. Diagnostic order, cheapest first:

1. Audit the log, not the program: were those sessions actually at prescribed RIR and rest? Under-resting mimics a plateau.
2. Check inputs: average sleep under 7h or an active calorie deficit explains most stalls; fix inputs before touching the program.
3. Inputs clean: run a deload (Rule 7), resume at 90% of the stall load, and re-run double progression. Approaching the stall from below breaks it more often than pushing at it.
4. Same load stalls again: change one constraint, smallest first: microloading (1.25 kg plates), then rep-range shift (8-12 to 4-6), then exercise variant. One change at a time or the signal of what worked is lost.

Beginners (first ~6 months) skip double progression: add 2.5 kg (upper) or 5 kg (lower) every session until the first stall. Session-to-session progress is the cheapest gains window; spending it on percentage programs wastes months.

## Recovery and Readiness

- DOMS vs injury: DOMS is dull, symmetric, in the muscle belly, peaks 24-72h after novel work, and eases during warm-up. Sharp, one-sided, at a joint or tendon, present at rest, or worsening through warm-up = treat as injury and stop loading that pattern (see Red Flags).
- Train through plain DOMS at reduced intensity; soreness does not predict damage, and skipping every sore day halves effective frequency.
- Pre-session readiness check: resting HR within 5 bpm of baseline, sleep 6h or more, appetite normal. Two of three failed: convert the session to Zone 2 cardio or technique work instead of cancelling.
- Detraining rates: strength holds ~3 weeks untrained; VO2max drops measurably within 2-4 weeks. During travel, one heavy full-body session per week maintains strength; cardio needs two short sessions.
- Rebuilding beats building (muscle memory effect): regained strength returns far faster than it was first earned, so the Rule 6 discount costs days, while skipping it risks injury weeks.

## Red Flags

| Signal (observable) | Suspicion | Action |
|---|---|---|
| Chest pain or pressure during exertion, radiating to arm or jaw | Cardiac event | Stop immediately, emergency services |
| Fainting or near-fainting during or right after exercise | Arrhythmia or cardiac cause | Stop training, clinician this week, no intensity until cleared |
| Cola-colored urine plus severe swelling after an extreme session | Rhabdomyolysis | Emergency care the same day |
| Sharp joint pain with swelling, or pain that alters gait | Structural injury | Stop loading that pattern; clinician if not improving in 5-7 days |
| Numbness, tingling, or weakness down a limb | Nerve involvement | Stop the provoking exercise, clinician |
| Resting HR 5+ bpm elevated for a full week plus performance and sleep decline despite a deload | Overtraining syndrome or illness | Suspend hard training; clinician if unresolved in 2 weeks |
| Wheeze or chest tightness triggered by exercise | Exercise-induced bronchoconstriction | Clinician before further intensity work |

Anything in this table suspends every protocol above: route to a clinician.

## Output Gates

Before emitting any training prescription, verify:

- Does every exercise line carry sets x reps + load or RIR + rest (Rule 8)?
- Is weekly volume per muscle inside the 10-20 hard-set corridor (Rule 4)?
- Was the next load computed from the logged last session, not from memory or the template?
- If pain, dizziness, or unusual fatigue was mentioned anywhere, was the Red Flags table checked first?
- After a gap of 2+ weeks, was the layoff discount applied (Rule 6)?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Programming to stated ambition | Adherence predicts results more than program choice; 6-day plans collapse to 0 | Logged attendance + max 1 day |
| Adding volume to break a stall | Stalls are usually recovery-limited; more sets deepen the hole | Diagnostic order in Progression and Plateaus |
| Every cardio session moderately hard | The gray zone: too hard to recover from, too easy to drive adaptation | 80/20 split (Rule 5) |
| Changing the program every 4 weeks | Double progression needs 6+ weeks to expose a real stall; novelty resets the signal | One variable, only at a diagnosed stall |
| Testing 1RM to measure progress | A max attempt costs a week of recovery and spikes injury exposure | Epley e1RM from rep sets (Rule 2) |
| Guilt-framing missed sessions | Shame predicts dropout; the log is data, not a verdict | Apply Rule 6 and move on |
| Treating wearable readiness scores as ground truth | Proprietary scores blend unvalidated inputs | Raw resting HR + sleep hours + log vs performance |

## Related Skills

- `gym` for rep-by-rep workout logging and in-session routine execution
- `running` for run-specific pacing, race preparation, and running injury prevention
- `calories` for the deficit or surplus math behind fat-loss and muscle-gain goals
- `sleep` when readiness checks point at sleep as the limiting input

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/fitness.
