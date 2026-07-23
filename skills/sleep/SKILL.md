---
name: sleep
slug: sleep
version: 1.0.2
changelog: Complete rewrite with real sleep protocols and red flags
description: 'Coaches sleep with quantified protocols: insomnia triage, jet lag light timing, caffeine and melatonin cutoffs, shift work anchors. Use when the user mentions poor sleep, insomnia, night waking, daytime tiredness, jet lag, naps, snoring, night shifts, CBT-I, or sleep tracker data.'
homepage: https://clawic.com/skills/sleep
metadata:
  clawdbot:
    emoji: "😴"
    displayName: Sleep
    configPaths:
    - ~/clawic/sleep/
---

# Sleep

Operational sleep coaching: triage the complaint, run the protocol with numbers, route red flags to a clinician instead of coaching past them. Advise mode only: guide the human, never touch their medication. Diary and trip plans persist in `~/clawic/sleep/` (created only when the user starts a protocol).

## When To Use

- User reports trouble falling asleep, 3am waking, or daytime tiredness.
- Trip planning across time zones: build the light and melatonin schedule before departure.
- Scheduling questions touching sleep: nap timing, caffeine cutoff, workout placement.
- Night shifts or rotating schedules: damage-control plan, not adaptation fantasies.
- User asks what their sleep tracker data means.
- Not for diagnosing sleep disorders: snoring with gasps, dream enactment, sleep attacks go to a clinician (→ Red Flags).

## Quick Reference

| Situation | Play |
|---|---|
| Bad sleep < 3 months, tied to a stressor | Acute: hold wake time, ban naps and early bedtimes, wait it out (→ Insomnia Protocol) |
| Bad sleep ≥ 3 nights/week for ≥ 3 months | Chronic insomnia (ICSD-3): run CBT-I lite (→ Insomnia Protocol) |
| Loud snoring + witnessed pauses + sleepy days | Stop coaching, refer for a sleep study (→ Red Flags) |
| Crossing ≥ 3 zones AND ≥ 3 nights there | Adapt: compute Tmin, schedule light by direction (→ Jet Lag) |
| Crossing < 3 zones OR < 3 nights there | Rule of 3: stay on home time, book meetings in the overlap window |
| Night or rotating shifts | Anchor sleep + commute light control (→ Shift Work) |
| "Should I nap?" | 10-20 min, finished ≥ 8 h before bedtime; never during an insomnia protocol |
| Tracker score bad, user feels fine | Trust daytime function; stage data is noise (→ Substances, Naps & Trackers) |
| "What supplement helps?" | Melatonin 0.5 mg timed for phase shift; everything else is weak (→ Substances, Naps & Trackers) |
| Weekend "catch-up" sleep | Cap wake-time drift at 1 h; 2 h drift = social jet lag and a Monday relapse |
| Any other sleep complaint | Start the 7-day diary in `~/clawic/sleep/diary.md`; no intervention before data |

## Core Rules

1. **Wake Anchor**: one fixed wake time ±30 min, 7 days/week; bedtime floats with sleepiness. Highest-leverage single change; check that weekend wake stays within 1 h of weekday wake.
2. Judge sleep by daytime function, not hours. Adult range is 7-9 h (AASM), not a universal 8: alert on 6.5 h = that user's number; sleepy in meetings after 8 h = a problem despite the hours.
3. Triage before advice: every complaint passes the Red Flags table first. Hygiene tips given to an apnea case cost a year of misdirection.
4. Stimulus control (Bootzin): awake ~20 min by feel (no clock-checking), leave the bed, dim light, boring analog activity, return only when sleepy. Best-evidenced single insomnia technique.
5. Effort inverts in sleep: "try to sleep more" always backfires. Prescribe the opposite: later bedtime, restricted window, worry scheduled earlier in the evening.
6. Time substances by half-life, not by feel: caffeine last dose ≥ 8 h before bed, alcohol last drink ≥ 3 h, melatonin 0.5 mg taken 5 h before target bedtime when the goal is shifting the clock.
7. Light steers the clock and direction depends on timing: light after Tmin advances the clock, before Tmin delays it. Tmin = habitual wake minus 2.5 h (wake 07:00 → Tmin 04:30). Backwards application makes jet lag worse.
8. One intervention per week, measured against the diary. Stacked changes make results unattributable; the diary is ground truth, not memory of the night.

## Red Flags

| Signal | Suspicion | Action |
|---|---|---|
| Loud snoring + witnessed breathing pauses or gasp-awakenings + daytime sleepiness | Obstructive sleep apnea | Refer for a sleep study before any protocol |
| Dozing while driving, or sleep intruding mid-conversation | Severe sleepiness (Epworth-range > 10) | Refer promptly; advise against driving drowsy now |
| Acting out dreams: punching, kicking, leaping, mostly age 50+ | REM behavior disorder | Neurologist referral, not urgent but not optional |
| Evening leg discomfort with urge to move, relieved by movement | Restless legs | Clinician; low ferritin is the common driver |
| Sudden sleep attacks, knees buckling with laughter | Narcolepsy/cataplexy | Sleep specialist |
| Insomnia + hopeless 3am thoughts, mood collapse | Depression presenting as insomnia | Treat mood as primary; escalate per user's care setup |

Anything in this table suspends the protocols in this skill: route to a clinician.

## Insomnia Protocol (CBT-I Lite)

Split by the 3-3 line: ≥ 3 nights/week for ≥ 3 months = chronic (ICSD-3); anything less is acute.

**Acute** (most cases, self-resolving): the job is preventing chronification, not fixing tonight.
- Hold the Wake Anchor even after a terrible night.
- Ban compensation: no naps, no earlier bedtime, no sleeping in, no nightcap. Compensation converts 2 bad weeks into 2 bad years.
- Stressor gone but insomnia persists 4+ weeks → start the diary and treat as pre-chronic.

**Chronic**: CBT-I is first-line, ahead of medication (ACP). Do NOT run restriction with: bipolar or mania history, seizure disorder, suspected apnea, pregnancy, or a safety-critical week (commercial driving, machinery); refer instead. Prescribed sleep medication is untouchable; coaching runs alongside, tapering is the prescriber's job.

1. Diary 7 days first, one line per night: `date | in bed | lights out | min to fall asleep | wakes (n, min) | final wake | out of bed | naps | caffeine last | alcohol`. Filled each morning; no tracker data pasted in.
2. Compute sleep efficiency SE = total sleep time / time in bed. SE ≥ 85% is healthy; SE < 85% triggers restriction.
3. Prescribed TIB = average diary TST rounded to 15 min, clamped to the floor: 5 h (5.5 h if age 65+, long commute, or safety-adjacent job). The floor is a clamp, never an exit: diary TST 4 h 30 → prescribe 5 h 00 and continue.
4. Worked example: TST 5 h 50, TIB 8 h 30 → SE 69%. Prescribe TIB 5 h 45 (350 min rounded to 15 min), wake fixed 06:30 → earliest bedtime 00:45. Bedtime is a "not before" line; sleepiness is the entry ticket, fatigue is not sleepiness.
5. Weekly titration from the diary: SE ≥ 90% → extend TIB 15 min; 85% ≤ SE < 90% → hold; SE < 85% → cut 15 min (never below floor). A week with 3+ violations (naps, sleep-ins, early bedtimes) is re-run, not titrated on.
6. Weeks 1-2 feel worse: sleepier days are the mechanism (pressure building); say so upfront or the user quits at day 5. Warn about drowsy driving during this phase.
7. No improvement by week 4 with clean adherence → stop and refer to a CBT-I clinician.
8. Discharge at 4 straight weeks SE ≥ 85% with good daytime function. Keep forever: Wake Anchor + out-of-bed rule. Relapse plan: 3 bad nights/week for 2 weeks → restart diary; 2 more weeks → restart restriction at last working TIB.

Falling asleep in < 5 min routinely is sleep deprivation, not talent; treat as a duration problem.

## Jet Lag

- Rule of 3: adapt only when zones crossed ≥ 3 AND nights at destination ≥ 3. Otherwise keep home sleep hours, schedule commitments into the overlap window, cover with 20-min naps and morning-only caffeine.
- Shift rates for planning: advancing (east) ~1 h/day, delaying (west) ~1.5 h/day. A 6-zone eastward trip = ~6 days to alignment; give the user this number so day-3 grogginess reads as on-schedule.
- Each adapted day, move the Tmin estimate 1 h earlier (east) or 1.5 h later (west) and re-derive the light windows.
- **Eastward** (the hard direction): convert home Tmin to destination clock. Before Tmin, avoid bright light: sunglasses outdoors, no sunrise walks. After Tmin, seek 2-3 h of outdoor light. Melatonin 0.5 mg taken 5 h before target bedtime, first 3-4 nights. First night: bed no earlier than 22:00 destination even if wrecked; a 19:00 collapse produces a 3am wide-awake.
- Worked example, NYC → Paris (6 zones east, wakes 07:00): Tmin 04:30 home = 10:30 Paris on arrival. Day 1 sunglasses until 10:30 then outdoor light; melatonin 0.5 mg at 18:00 for a 23:00 bedtime; day 2 Tmin ≈ 09:30; aligned around day 5-6. Book nothing decision-heavy before 11:00 the first two days.
- **Westward**: bright light late afternoon and evening; avoid early-morning light for 2-3 days (it lands after home-Tmin and advances, fighting the delay). The signature failure is a 4am wake, not sleep onset; brief the user that it closes within 2-3 days. Melatonin mostly unnecessary westward.
- \> 8 zones east: the clock may take the shorter path and resolve as a delay (antidromic re-entrainment). Plan 9-12 zone eastward trips as westward-style delays; if day-3 sleepiness lands at odd hours, re-estimate Tmin from when the user feels worst.
- Deliver trip plans as a per-day table (avoid-light window, seek-light window, melatonin dose + time, caffeine cutoff, earliest bedtime) in `~/clawic/sleep/trip-<destination>.md`.

## Shift Work

- State once: full circadian adaptation to nights is rare outside never-flipping permanent night workers. The goal is minimizing debt and mistimed light, not "becoming nocturnal".
- Anchor sleep: one 3-4 h block at the same clock hours on both workdays and days off; the rest floats. Shift 23:00-07:00 → anchor 08:00-12:00 daily; extend to 15:00 on days off. The anchor stops the clock free-running between the two lifestyles.
- Light: bright through the first half of the shift, dim the last 2 h. The commute home is the highest-leverage 30 min: sunrise light lands right after Tmin and yanks the clock toward daytime; dark sunglasses door-to-door.
- Caffeine: front-load, first half of shift only, none within 8 h of planned day sleep. The 05:00 coffee is the classic day-sleep killer.
- Rotation, where the user has any influence: forward only (morning → evening → night); ≥ 11 h between consecutive shifts; max 3-4 consecutive nights then ≥ 48 h recovery with two full nights.
- Transition day after the last night: short block 08:00-12:00, stay up through the evening, sleep the coming night at normal hours. The full-day sleep-in eats the recovery night.
- Shift work disorder = shift sleepiness + window insomnia persisting ≥ 3 months despite a protocol like this → clinician referral.

## Substances, Naps & Trackers

| Substance | Threshold | Note |
|---|---|---|
| Caffeine | Last dose ≥ 8 h before bed; half-life ~5 h, individual range 2-10 h | 400 mg at 6 h out cut > 1 h of measured sleep with users unaware (Drake); "doesn't affect me" reports onset, not architecture |
| Alcohol | Last drink ≥ 3 h before bed; never as a sleep aid | Speeds onset, fragments the second half: REM rebound, 3am wakes |
| Melatonin | 0.5 mg (0.3-1 mg effective, Zhdanova), taken 5 h before target bedtime for phase shifts | Chronobiotic, not a sedative; 5-10 mg gummies overshoot into morning; OTC content measured -83% to +478% of label |
| Diphenhydramine / OTC "PM" | Tolerance in 3-4 days | Flag nightly reliance; anticholinergic load, groggy mornings |
| THC | Suppresses REM | Warn before a tolerance break: rebound insomnia + vivid dreams |
| Magnesium, glycine, theanine | No threshold worth stating | Comfort, not treatment |

- Naps: 10-20 min restores alertness without inertia; 30-60 min lands in the grogginess window; ~90 min is a full cycle. Finish ≥ 8 h before bedtime (≈ 15:00 for a 23:00 sleeper). Banned during an insomnia protocol. Coffee nap for emergencies: caffeine immediately before a 20-min nap.
- Debt is nonlinear: 2 weeks at 6 h/night matches a night of total deprivation on performance while subjective sleepiness plateaus (Van Dongen). "Adapted to 6 hours" = adapted to feeling impaired.
- 17-19 h continuously awake degrades psychomotor performance like ~0.05% blood alcohol; use to veto late-night driving.
- Trackers: asleep-vs-awake is acceptable; stage claims agree with lab measurement only ~50-60% of epochs. Use for timing consistency and duration trends; ignore nightly scores and stage breakdowns. If the user quotes their score with anxiety (orthosomnia), prescribe 2 weeks of blind tracking: collect, do not look.

## Output Gates

- Did this complaint pass the Red Flags table before any protocol advice?
- Does every melatonin mention carry both dose and clock time (0.5 mg, 5 h before target bedtime for phase shifts)?
- Is prescribed TIB clamped to ≥ 5 h and the bedtime phrased as "not before"?
- Are jet lag light windows derived from Tmin converted to destination clock, not from local sunrise?
- Am I prescribing exactly one new intervention this week, with the diary as the measure?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Sleeping in after a bad night | Dilutes sleep pressure, delays the clock, seeds the next bad night | Same wake time; earlier sleepiness tonight is the repayment |
| Earlier bedtime to get more sleep | More time awake in bed conditions bed = frustration | Later bedtime until SE ≥ 85%, then extend by 15 min steps |
| Weekend catch-up ≥ 2 h | Social jet lag (Roenneberg): Sunday-night insomnia, Monday impairment | Cap drift at 1 h; 20-min Saturday nap if needed |
| Hygiene tips for chronic insomnia | Hygiene alone shows near-zero effect on chronic cases; it is prevention | CBT-I components: restriction + stimulus control |
| Nightcap for sleep | Onset improves, second half fragments | ≥ 3 h alcohol buffer; treat latency with restriction |
| 10 mg melatonin at lights-out for jet lag | Wrong dose and wrong hour; sedation misread as adaptation | 0.5 mg, 5 h before target bedtime, eastward only |
| Morning sunlight on arrival in Europe from the US | Lands before body-clock Tmin, delays the clock, worsens the lag | Sunglasses until converted Tmin, bright light 2-3 h after |
| Coaching a loud snorer on bedtime routine | Misses apnea; months lost while AHI stays high | Red Flags first, referral before protocol |
| "Relax and clear your mind" | Sleep-effort paradox: monitoring for sleep prevents it | Stimulus control; paradoxical intention for high performers |
| Adjudicating tracker deep-sleep deficits | Stage data is noise at consumer accuracy | Re-anchor on daytime function and the diary |

## Where Experts Disagree

- Blue light: photobiology shows real melatonin delay; behavioral trials show content arousal dominates in adults. Teens and severe insomniacs get strict screen cutoffs; average adults get engagement rules (no feeds in bed), not amber glasses.
- Napping: performance school prescribes it, insomnia school bans it. Sleeps well → nap freely within the ≥ 8 h cutoff; in protocol → no naps until discharged.
- Melatonin for plain insomnia: trials average ~7 min faster onset; strong effects only for circadian problems. Circadian use yes, nightly-forever use no.

## Related Skills

- `fitness`: when the lever is training load, overtraining, or workout timing rather than the night itself.
- `fasting`: when late eating windows or fasting schedules collide with the sleep window.
- `plan`: when the fix is calendar surgery, moving deep work to the user's alert hours instead of fixing sleep.

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/sleep.
