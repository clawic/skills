---
name: career
slug: career
version: 1.0.3
description: Advises on career decisions, salary negotiation, promotions, transitions, and layoff response with quantified plays and decision rules. Use when the user weighs an offer, plans a career move, builds a promotion case, or responds to a layoff.
homepage: https://clawic.com/skills/career
metadata:
  clawdbot:
    emoji: "💼"
    displayName: Career
  changelog: Complete rewrite with real career-move heuristics
---

Advise mode: this skill guides a human making their own career moves; it never acts on their behalf. User career context lives in `~/clawic/career/profile.md` (comp history, market band, constraints, stated values, current target); create it on first substantive session and update after each decision.

## When To Use

- An offer or counteroffer is on the table and the user must respond
- Deciding whether to stay, leave, or pivot role, company, or industry
- Building a promotion case or diagnosing a stalled level
- Responding to a layoff, PIP, or rescinded offer
- Periodic comp benchmarking and market testing cadence
- Not for writing the resume itself (use resume) or running the application pipeline (use job-search)

## Quick Reference

| Situation | Play |
|---|---|
| Offer received | Never accept in the call; get it in writing, take 48 hours, counter once minimum (see Offer And Negotiation Playbook) |
| Asked for salary expectations first | Deflect once; if forced, state a range whose bottom is your target |
| Exploding deadline (under 1 week) | Request one week in writing; treat refusal as data about the employer |
| "Should I quit?" | No decision without a BATNA: generate a live alternative first (Rule 1) |
| Stalled 2+ promotion cycles | Diagnose scope, sponsor, or calibration timing (see Promotion Engineering) |
| Wants to change industry and role at once | One-variable rule: change one axis per move (Rule 4) |
| Laid off | 48-hour sequence; sign nothing on day one (see Layoff Response) |
| Counteroffer after resigning | Decline by default: the root cause persists and flight risk is now on record |
| Happy, no active decision | Market check every 18-24 months: 2-3 recruiter conversations to reprice the band |
| Anything else | Collect three facts before advising: current total comp, market band, live alternatives |

## Core Rules

1. **No BATNA, no decision.** "Stay or go" without a concrete outside option is a mood poll. Check: can the user name a specific alternative and its comp? If not, the first step is generating options, not deciding.
2. **Counter = 2 x target - offer.** Offer 100k, target 112k, counter 124k: negotiations gravitate toward the midpoint of the two stated numbers, so countering at your target guarantees landing below it.
3. **Switchers outprice stayers.** Atlanta Fed Wage Growth Tracker: job switchers beat stayers by roughly 1-2 percentage points of annual wage growth in most years. This is the base-rate argument behind the 18-24 month market check even when content.
4. **One-variable transitions.** Per move, change role OR industry OR track (IC to manager), not two. Each simultaneous change resets leverage to beginner on that axis; two at once typically costs both comp and title.
5. **Promotion is a lagging indicator.** You get promoted for having already done the next level's job for 6-12 months with witnesses outside your team. If nobody above your manager can describe your work, the packet fails at calibration.
6. **Negotiate the package, not the base.** Typical employer flexibility, most to least: sign-on bonus, equity, start date and vacation, title and review timing, base. Sign-on is a one-time cost to them; moving base reprices their whole band.
7. **Reversibility sets the risk bar.** If a move can be undone within about 12 months (boomerang hire, internal transfer back, contract work), bias toward acting; irreversible moves (relocation with family, visa changes, walking away from a cliff) get the full BATNA-plus-runway treatment.

## Offer And Negotiation Playbook

In order:

1. Get the full offer in writing: base, bonus target with payout history, equity grant with vest schedule and cliff, sign-on, refresher policy.
2. Compute annualized total comp yourself; recruiters quote best case. "Bonus target 20%" is not "bonus 20%": ask what percent of employees hit target last cycle.
3. Set target from market band (levels.fyi, peer conversations, recruiter ranges), never from current comp. "Current plus 15%" is an anchor, not a target.
4. Counter once at 2 x target - offer (Rule 2), justified with one sentence of market data, never personal need. Need justifies nothing; band data justifies everything.
5. If base is capped, redirect down the flexibility ladder (Rule 6). A 10-20k sign-on closes most sub-10% gaps. Also cheap for them: title, an earlier 6-month review (an early raise window), start date.
6. Read vesting mechanics before signing: a standard 4-year vest with 1-year cliff means leaving at month 11 forfeits everything; include cliff dates in any "one more year" math.
7. Rescind fear check: polite, single, data-backed counters almost never kill offers. A company that rescinds over a professional counter revealed its management culture at zero cost to you.

Internal negotiation is a different game: your performance is already known, internal peer equity matters more than market rate, and you need a champion in the room, not just a request. Never wave an external offer you are unwilling to take.

## Stay, Leave, Or Pivot

Diagnose before prescribing; a bad manager and a bad career have opposite fixes.

- Three-question razor: (a) Would a different manager fix this? Internal transfer before quitting. (b) Would the same role at a better company fix it? Switch companies, keep the role (Rule 4). (c) Does the work itself drain you even in good weeks, for 3+ months? Pivot, planned as a one-variable move.
- Quitting without an offer requires runway of at least 2x realistic search time. Senior searches commonly run 3-6 months, so hold 6-12 months of expenses in cash, not "some savings".
- Pivot mechanics: bridge through the overlap. Target the role where your current skill is the differentiator (engineer to technical PM, not engineer to brand designer). Name the gap honestly in your story; claiming the experience is "basically the same" fails the first expert interviewer.
- A comp haircut on a true pivot is normal; a haircut over about 20% usually means you skipped an intermediate bridge role.
- Tenure math: sunk years are not an asset; unvested equity is. Compute the dollar cost of leaving before the next vest date and treat only that number as the price of leaving.

## Promotion Engineering

- Sponsor vs mentor: a mentor talks with you; a sponsor talks about you in calibration rooms you are not in. A case without a sponsor above your manager is a coin flip at best. Build sponsorship by making work legible to skip-levels: demos, written updates, cross-team projects.
- Timing: raise the topic 2 review cycles before you want the outcome and ask exactly one question: "What specifically is missing from my packet?" A vague answer ("more impact") means no sponsor or no headcount; probe which.
- Evidence beats effort: the packet is 3-5 artifacts with your name on them that someone outside your team has already seen. Start collecting the day you decide to push, not review week.
- Passed over twice with a concrete packet and a sponsor? The constraint is structural (headcount, level ratios). Rule 3 applies: the level you cannot get internally is usually purchasable externally as a hiring level.

## Layoff Response

First 48 hours, in order:

1. Sign nothing on day one. In the US, workers 40+ get 21 days to consider a severance release (45 for group layoffs) under OWBPA; taking days to review is normal, not adversarial. Outside the US, check statutory notice periods before signing anything.
2. Before access is cut: lock 2-3 reference commitments, export contacts, your comp history, and your own artifact list (never employer IP).
3. File for unemployment immediately; severance rarely blocks eligibility and delays cost real money.
4. Severance is negotiable exactly once, before signing. Standard asks: extra weeks, healthcare extension or COBRA subsidy, equity acceleration, removal of non-compete language.
5. Only then start the search (route to job-search). A same-day panic application spree with a stale resume converts worse than a 3-day setup with a targeted list.

## Output Gates

Before delivering career advice, check:

- Did I get current total comp, market band, and live alternatives before recommending any number or any stay/quit call?
- Is every counter I proposed computed as 2 x target - offer, not the midpoint of the visible range?
- Did I check unvested equity and cliff dates before endorsing a departure timeline?
- When no live alternative exists, did I recommend generating a BATNA instead of deciding?

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| Accepting on the call | Peak-emotion decision at their anchor | 48 hours and written details, always |
| Naming your number first | Anchors at your guess of their band, usually low | Deflect once; if forced, range with target at the bottom |
| Resigning to force a counteroffer | Root cause persists and you are flagged as flight risk | Resign only when ready to leave |
| "I have invested 5 years here" | Sunk cost; tenure is not an asset, unvested equity is | Price only the forward-looking cost of leaving |
| Chasing title over scope | Inflated title with unchanged scope fails the next interview loop | Trade title for scope, budget, or reports |
| Calling a mentor a sponsor | Advice does not move calibration decisions | Verify who speaks for you in the room |
| Negotiating base only | Base is their stickiest number | Sign-on, equity, dates, title first (Rule 6) |
| Prestige capture | Optimizing for logo approval over fit compounds misery | Score offers against stored profile values, not brand names |

## Where Experts Disagree

- Passion-first vs skill-first: both schools hold conditionally. Under about 5 years of experience, skills compound faster than self-knowledge, so bias skill-first (the career-capital position, Newport); with rare skills banked, passion picks between good options.
- Generalist vs specialist: specialize while the field expands (early markets pay depth); generalize as it consolidates (mature markets pay range and management).
- Loyalty premium: tenure-track systems (government, academia, some East Asian conglomerates) genuinely pay tenure; the switcher math of Rule 3 is calibrated to open-market private sectors and reverses there.

## Related Skills

- **job-search** for running the pipeline: applications, company research, interview prep
- **resume** when the decision is made and materials need targeting
- **negotiate** for negotiation mechanics beyond comp: limits, principals, approvals
- **comp-design** when the user is on the employer side designing base, bonus, and equity mix

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/career.
