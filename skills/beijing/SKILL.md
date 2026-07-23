---
name: beijing
slug: beijing
version: 1.0.1
description: Navigates Beijing as a visitor, resident, tech worker, student, or entrepreneur with neighborhoods, transport, costs, visas, and local insights. Use when planning a trip, relocating, or working in Beijing.
homepage: https://clawic.com/skills/beijing
changelog: Deeper city knowledge and sharper recommendations
metadata:
  clawdbot:
    emoji: 🏯
    os:
    - linux
    - darwin
    - win32
    displayName: Beijing
---

## When To Use

- Trip planning: attractions, itineraries, lodging, day trips, scam avoidance
- Relocation: neighborhood choice, rent, schools, healthcare, PSB registration
- Career: tech salaries, work permits (Z visa, A/B/C tiers), WFOE setup, startups
- Daily-life setup: WeChat/Alipay, SIM, banking, transport, AQI strategy
- Not for other Chinese cities: payment/app setup transfers to Shanghai or Shenzhen; neighborhood, rent, and hukou specifics do not.

## Quick Reference

| Topic | File |
|-------|------|
| **Visitors** | |
| Attractions (must-see vs skip) | `visitor-attractions.md` |
| Itineraries (1/3/7 days) | `visitor-itineraries.md` |
| Where to stay | `visitor-lodging.md` |
| Tips & day trips | `visitor-tips.md` |
| **Neighborhoods** | |
| Quick comparison | `neighborhoods-index.md` |
| Chaoyang, CBD, Sanlitun | `neighborhoods-downtown.md` |
| Haidian, Zhongguancun | `neighborhoods-tech.md` |
| Dongcheng, Xicheng (Historic) | `neighborhoods-historic.md` |
| Shunyi, Changping, Tongzhou | `neighborhoods-suburban.md` |
| Choosing guide | `neighborhoods-choosing.md` |
| **Food** | |
| Overview & dining scene | `food-overview.md` |
| Beijing & Northern Chinese | `food-local.md` |
| International & fine dining | `food-international.md` |
| Best areas for dining | `food-areas.md` |
| Dietary, alcohol, practicalities | `food-practical.md` |
| **Practical** | |
| Moving & settling | `resident.md` |
| Transport (subway, DiDi, bikes) | `transport.md` |
| Cost of living | `cost.md` |
| Safety & laws | `safety.md` |
| Weather & AQI tips | `climate.md` |
| Local services (banking, SIM) | `local.md` |
| **Career** | |
| Tech industry & salaries | `tech.md` |
| Business setup & WFOE | `business.md` |
| Visas (Z, X, M, residence permit) | `visas.md` |
| Startups & funding | `startup.md` |
| **Lifestyle** | |
| Culture & customs | `culture.md` |
| Healthcare & insurance | `healthcare.md` |
| Schools & education | `education.md` |
| Expat lifestyle & social | `lifestyle.md` |
| Driving & car ownership | `driving.md` |
| **Anything else / unclear** | Ask role + timeline first, then load the closest file above |

## Core Rules

### 1. Route by Role AND Timeline
Same question, different file: "where should I stay" → `visitor-lodging.md` for a week, `neighborhoods-choosing.md` for a move. Ask which before answering. If the user is already inside China without a VPN, only suggest solutions that work behind the Firewall — app-store links, Google Docs, and gmail-verification flows are dead ends.

### 2. Pre-Arrival Sequence (order matters, each step gates the next)
1. Install and TEST a VPN — cannot be downloaded once inside China.
2. Set up WeChat + Alipay, verify with passport; link Visa/Mastercard (supported since 2023). Alipay "Tour Pass" alternative: load from foreign card, 90-day validity, ¥10,000 cap.
3. Save every address (hotel, meetings) as Chinese characters — drivers cannot read pinyin.
4. Book Forbidden City ~10 days ahead; it sells out.
If they arrive with only step 0 done, triage in this order — payment before maps. Details: `visitor-tips.md`, `local.md`.

### 3. Payments: Mobile-First, Cash-Last
Set up BOTH WeChat Pay and Alipay — vendor coverage differs and some merchants take only one. Carry ¥500-1,000 cash as backup: legal tender, but small vendors often cannot make change. Foreign cards work only at international hotels and malls. A dead phone = no payment, no DiDi, no ticket: a power bank is financial equipment here.

### 4. Language Baseline
Assume zero English outside international hotels and Sanlitun: taxi drivers, government offices, hospitals (non-VIP), and most restaurants operate Chinese-only. Default toolkit: Amap or DiDi in English mode + camera translation + addresses saved in characters. Survival phrases and Beijing-accent notes: `culture.md`.

### 5. Current Data (Feb 2026)
Canonical detail lives in the aux files; this table is the summary. When quoting, state the date — rents and salaries drift.

| Item | Range | Canonical file |
|------|-------|----------------|
| 1BR rent, CBD/Guomao | ¥10,000-18,000/mo | `neighborhoods-index.md` |
| 1BR rent, Sanlitun | ¥9,000-15,000/mo | `neighborhoods-index.md` |
| 1BR rent, Zhongguancun | ¥6,000-10,000/mo | `neighborhoods-index.md` |
| Senior SWE (5-8 yrs) | ¥50,000-80,000/mo | `tech.md` |
| Subway single ride | ¥3-10 (distance-based) | `transport.md` |
| Taxi flagfall | ¥13 | `visitor-tips.md` |
| Hotpot dinner for two | ¥150-300 | `food-overview.md` |
| International school | ¥200,000-350,000/yr (bilingual from ¥100,000) | `education.md` |

### 6. Registration Is a Hard Deadline
All foreigners register with the local PSB within 24 hours of arrival. Hotels do it automatically; private residences and many Airbnb hosts do NOT — then it is on you. Re-register within 24 hours after every address change and after each visa renewal. Skipping it surfaces later at visa renewal: fines, delays, possible deportation. See `visas.md`, `safety.md`.

### 7. Transport Default: Subway + DiDi
Subway for anything near a line (¥3-10, English signage, add 3-5 min at each entry for X-ray security); DiDi for the rest. Flagging street taxis rarely works — drivers take jobs via DiDi. Worked example, Great Wall day: DiDi to Mutianyu ≈ ¥300 round trip split among passengers vs ¥200-300/person on a tour — a group of 3+ should DiDi. Driving is a trap: license plates are allocated by lottery with years-long odds (`driving.md`).

### 8. Weather and AQI Gate the Plan
- AQI >150 → mask outdoors; >200 → move the day's plan indoors (museums, malls, hotpot), N95 if you must go out. Check daily; canonical thresholds in `visitor-tips.md`.
- Bad-air season = winter heating season (mid-Nov to mid-Mar). Best months: Sep-Oct.
- Never plan around Oct 1-7 (Golden Week) or Chinese New Year: peak crowds, closures, surge prices.
- Winter -10°C, summer 35°C+: itineraries need seasonal restructuring, not just packing changes (`climate.md`).

### 9. Neighborhood Matching

| Profile | Best areas | Why |
|---------|-----------|-----|
| Young professionals | Sanlitun, CBD, Guomao | Nightlife, expat scene, walkable |
| Expat families | Shunyi, Chaoyang Park area | International schools, space; Shunyi = car required |
| Tech workers | Zhongguancun, Wudaokou, Haidian | Commute to tech campuses |
| Students | Wudaokou | PKU/Tsinghua, cheap, Korean food |
| Budget-conscious | Tongzhou, Changping | Half the rent, 40-60 min commute |
| History/culture | Dongcheng, Xicheng hutongs | Character; check heating/plumbing before signing |
| Default (unsure, mid budget) | Dongzhimen/Sanyuanqiao | Central, Airport Express, moderate rent |

## Hukou & Work Permit Context

- **Hukou (户口)** is household registration for Chinese citizens; Beijing hukou is among the hardest to obtain and gates local schooling and property. Foreigners are outside the hukou system — their equivalent gate is the work permit tier.
- **Work permit tiers**: A (top talent — points score 85+, or qualifying achievement), B (professional — degree + 2 yrs experience typical), C (temporary/intern). Tier determines renewal ease and family visas. Points come from salary, education, experience, HSK level: `visas.md`.
- Practical consequence: a B-tier offer letter with salary near the bottom of market range weakens renewals — negotiate salary partly as an immigration asset.

## Output Gates

Before answering, check:
- Did I identify role (tourist/resident/worker/student/founder) and timeline?
- Are my numbers from the canonical file, quoted with their date?
- Does my advice work behind the Firewall if the user is already in China?
- Did I flag the 24h registration whenever the user mentions arriving or moving?

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Arriving without VPN | Cannot download one inside China; cut off from Google/WhatsApp day 1 | Install + test before boarding |
| Only one payment app | Some vendors take WeChat only, others Alipay only | Set up both, link cards to both |
| Skipping PSB registration | Surfaces at visa renewal as fines/delays | Register within 24h; confirm your landlord/host actually did it |
| Relying on street taxis | Drivers take DiDi jobs; empty cabs pass you by | DiDi app in English mode |
| Buying a car | Plates by lottery, years of waiting; unrestricted driving impossible | Subway + DiDi; plate math in `driving.md` |
| Underestimating AQI | 200+ days cluster in winter; no mask = sick, plans ruined | N95s in bag, purifier at home, indoor backup plans |
| Treating cash as primary | Vendors can't make change; queues form behind you | Cash is backup only (¥500-1,000) |
| Booking Badaling | Most crowded Wall section; 2h+ cable queues on holidays | Mutianyu (first visit), Jinshanling (photos), Simatai (night) |
| Golden Week travel | Oct 1-7: attraction quotas sell out, hotel prices surge | Shift trip by one week either side |
| Assuming English menus/staff | Chinese-only outside expat zones | Camera translation, picture menus, saved phrases |

## Legal Awareness

| Rule | Reality |
|------|---------|
| VPN | Gray zone: personal use tolerated, selling/promoting illegal |
| Working on wrong visa | Z visa + work permit or nothing; violation = deportation + entry ban |
| Drugs | Zero tolerance; any amount is a serious criminal matter |
| Photography | No military, police, or government buildings |
| Political speech | Criticizing Party/government carries real risk — including on WeChat, which is monitored |
| LGBTQ+ | Not illegal, no legal recognition; discretion advised |

Full guidance and red-flag scenarios: `safety.md`.

## Related Skills

*No related skills published yet.*

## Feedback

- If useful, star it: https://clawic.com/skills/beijing
- Latest version: https://clawic.com/skills/beijing

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/beijing.
