---
name: period
slug: period
version: 1.0.2
description: Tracks menstrual cycles, symptoms, and fertility windows from data she shares. Use when logging periods, predicting the next cycle, or flagging changes from her baseline.
homepage: https://clawic.com/skills/period
metadata:
  changelog: Complete rewrite with real cycle-tracking guidance
  clawdbot:
    emoji: "🩸"
    displayName: Period Tracker
    configPaths:
    - ~/period/memory.md
---

Local data lives in `~/period/memory.md`. Never sync, never mention cycle content outside a session she opened. This skill acts directly (logs, predicts, flags); it advises, it does not diagnose.

## When To Use

- She reports a period start, flow, or symptom and wants it recorded.
- She asks when her next period, fertile window, or ovulation is likely.
- She wants a cycle classified (regular vs irregular, heavy vs normal) against real thresholds, not "28 days."
- She flags a change and wants to know if it is outside her own pattern.
- Not for: contraception decisions, diagnosing PCOS/endometriosis, or interpreting a positive pregnancy test. Route those to a clinician.

## Quick Reference

| Situation | Play |
|-----------|------|
| "My period started" | Log Day 1 = first day of full flow (not prior spotting). Timestamp it. |
| "When is my next period?" | Predict only with >=3 logged cycles: last Day 1 + median cycle length. |
| "Am I regular?" | Spread of last 12 cycles: <=9 days = regular; >9 = irregular (not broken). |
| "When can I get pregnant?" | Fertile window = predicted ovulation -5 through ovulation day. Opt-in only. |
| "Is this too heavy / too long?" | Flow >8 days or soaking hourly for 2h+ or clots >2.5 cm = flag (see Red Flags). |
| Symptom mentioned (cramps, mood) | Log against cycle day. Do NOT attribute to cycle unless she does first. |
| <3 cycles logged, or default | State prediction is not yet reliable; give a wide range or decline, keep logging. |

Trackable symptom catalog: `symptoms.md`. Privacy and deletion rules: `privacy.md`.

## Core Rules

1. **Day 1 is the first day of full flow, not spotting.** Prediction math anchors on Day 1; counting spotting as Day 1 shifts every downstream date by 1-2 days.
2. **Predict from her median, never a textbook constant.** `next start = last Day 1 + median(last 3-6 cycle lengths)`. Median resists one outlier cycle that a mean would drag. Below 3 cycles there is no baseline: say so.
3. **Attach a range, not a point.** `range = median +/- (longest - shortest)/2` of her logged cycles. A woman with cycles 26-34 gets "predicted day X, plus or minus 4 days," never a false-precision single date.
4. **Ovulation is back-counted, not forward-counted.** `ovulation ~= next predicted period - 14` (luteal phase is relatively fixed (~10-16 days); the follicular phase is what varies). This is why long cycles delay ovulation but not the luteal length.
5. **Classify against FIGO ranges (§Classifying), not against 28.** 24-38 day cycles are normal frequency; irregular is a description, not a defect. PCOS and perimenopause produce genuinely long cycles.
6. **Flag against HER baseline once >=3 cycles set it.** A new sustained shift of >9 days in cycle length, or any Red Flag signal, is worth surfacing. A single off cycle is noise, not a flag.

## Prediction Math (worked)

Logged Day 1 dates: Jan 3, Jan 31, Feb 26, Mar 27. Cycle lengths: 28, 26, 29 days.
- Median = 28. Shortest 26, longest 29 -> half-spread = (29-26)/2 = 1.5, round to 2.
- Next period = Mar 27 + 28 = **Apr 24, plus or minus 2 days**.
- Ovulation = Apr 24 - 14 = **~Apr 10**. Fertile window = Apr 5 through Apr 10.

If she has only 2 cycles, report "roughly late April, still learning your pattern" and refuse a hard date. Recompute the median every new cycle. The prediction window is the last 3-6 cycles (§Rule 2), which governs the estimate; the ~12-month mark is only an outer retention ceiling, discarding cycles older than that so stale perimenopausal drift never re-enters the window.

## Classifying A Cycle (FIGO thresholds)

| Axis | Normal | Outside normal |
|------|--------|----------------|
| Frequency (Day 1 to Day 1) | 24-38 days | <24 frequent; >38 infrequent |
| Regularity (spread over 12 cycles) | <=9 days | >9 days = irregular |
| Duration (days of bleeding) | 2-8 days | >8 prolonged; <2 very short |
| Flow | soaks a normal pad/tampon in 3-6h | hourly for 2h+, clots >2.5 cm, or >80 mL/cycle = heavy |

Irregular by these axes is common with PCOS, perimenopause, thyroid issues, or high stress. Report the classification; do not name the cause.

## Fertility Signals (opt-in only)

Enable only if she asks to track conception or avoidance. Three confirming signals, ranked by predictive value:
- **LH surge (ovulation predictor kit):** positive test means ovulation in ~24-36 hours. Best forward predictor.
- **Cervical mucus:** clear, stretchy, egg-white texture marks peak fertility (the 1-2 days before ovulation).
- **Basal body temperature:** a sustained rise of 0.3-0.5 C confirms ovulation already happened. It is retrospective, so it verifies a cycle but cannot predict the current one.
Cross-check predicted ovulation (§Rule 4) against these; when they disagree, the live signals win over the calendar estimate.

## Red Flags

Anything in this table suspends prediction and coaching: surface it plainly and route to a clinician.

| Signal (observable) | Suspicion | Action |
|---------------------|-----------|--------|
| Soaking a pad/tampon hourly for 2h+, or clots >2.5 cm | Heavy menstrual bleeding, anemia risk | Flag now; suggest same-week clinical review |
| Bleeding >8 days, or between periods, or after sex | Prolonged/intermenstrual bleeding | Flag; recommend evaluation |
| No period for 90+ days (not pregnant, not on continuous contraception) | Secondary amenorrhea | Flag; recommend evaluation |
| Any bleeding 12+ months after periods stopped | Postmenopausal bleeding | Urgent: always evaluated, treat as clinician-first |
| Pain that OTC relief does not touch or that stops daily activity | Possible endometriosis/other | Flag; recommend evaluation |
| Bleeding during a known pregnancy | Obstetric emergency risk | Emergency guidance first, then stop routine tracking |

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Assuming a 28-day cycle to predict | Her follicular phase varies; a 34-day cycle ovulates ~day 20, not 14 | Back-count 14 from her predicted next period (§Rule 4) |
| Predicting from 1-2 cycles | No stable baseline; a false hard date erodes trust when it misses | Log without predicting until >=3 cycles, then predict with a range |
| Using the mean cycle length | One 45-day stress cycle skews it upward | Use the median (§Rule 2) |
| Attributing her mood to her cycle | Correlation she has not endorsed feels like being told how she feels | Log the symptom against the day; let her draw the link |
| Calling irregular cycles "broken" | 60+ day cycles are normal in PCOS/perimenopause | Classify with FIGO axes; irregular is a description |
| Counting spotting as Day 1 | Shifts every predicted date 1-2 days | Day 1 = first full flow only |

## Related Skills

- `symptoms` when she wants general (non-cycle) symptom tracking and doctor-visit prep.
- `pregnancy` once a pregnancy is confirmed and tracking shifts to prenatal.
- `doctor` to turn flagged signals into a structured question list for an appointment.
- `health` for longitudinal wellness habits beyond the cycle.

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/period.
