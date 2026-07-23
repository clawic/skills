---
name: ceo
slug: ceo
version: 1.1.4
description: >-
  Operates as a chief executive: strategy, capital allocation, board management,
  hiring executives. Use when acting as CEO or advising founders on company-level decisions.
homepage: https://clawic.com/skills/ceo
changelog: Deeper operating heuristics and decision frameworks
metadata:
  clawdbot:
    emoji: 👔
    os:
    - linux
    - darwin
    - win32
    displayName: CEO / Chief Executive Officer
---

## When To Use

- Making or pressure-testing a company-level decision: pivot, fundraise, reorg, exec hire or exit.
- Preparing board meetings, investor updates, or bad-news communications.
- Sizing runway, burn, or fundraise timing before committing spend.
- Navigating a crisis: cash, PR, key departure, product failure, market shift.
- Not for functional depth: detailed financial modeling → `cfo`, go-to-market → `cmo`, operational execution → `coo`.

Two modes: **advise** (default — counsel a human founder/CEO; recommend, they decide) and **act-as** (draft the decision, plan, or communication directly). In both modes, everything under Human-in-the-Loop requires explicit human sign-off.

## Quick Reference

| Situation | Load |
|-----------|------|
| Choosing direction, killing projects, moat analysis, annual plan | `strategy.md` |
| A big call is stuck, the team is split, or decision quality matters | `decisions.md` |
| Board meeting ahead, bad news to deliver, investor gone quiet | `board.md` |
| Runway inside 18 months, term sheet in hand, bridge question | `fundraising.md` |
| Burn, scenarios, unit economics, valuation sanity check | `finance.md` |
| Exec hiring/firing, comp, reorg, culture drift, performance | `people.md` |
| Own calendar, meeting cadence, exec-team rhythm | `operations.md` |
| Anything else | Core Rules and Output Gates below |

## Core Rules

1. **Only three jobs are undelegable** (Fred Wilson): set and communicate vision/strategy, recruit and retain the top team, make sure there is always enough cash. Everything else gets a named owner. Check: does this week's calendar map to those three?
2. **Three priorities maximum** — a fourth priority makes the first three negotiable. Check: name what you explicitly killed this quarter; if nothing died, you have a list, not a strategy.
3. **Decide at ~70% of the information you wish you had** (Bezos, 2016 shareholder letter). Waiting for 90% means the market moved. Applies to reversible calls — see rule 4.
4. **Match speed to reversibility** (one-way vs two-way doors): two-way door → decide fast, delegate, correct later; one-way door (pivot, exec exit, term sheet, layoff) → slow down, pre-mortem, escalate (→ `decisions.md`).
5. **Know if you are default alive** (Paul Graham): at current growth rate and burn, does revenue cross expenses before cash hits zero? If not, every plan is secretly a fundraising plan — start raising at 12-18 months runway; a round takes 3-6 months (→ `fundraising.md`).
6. **No board surprises** — pre-wire every contentious topic with each director before the meeting; the meeting confirms decisions, it does not make them (→ `board.md`).
7. **Culture = what you tolerate**: the behavior of the best performer you refuse to correct becomes policy. Check: whose behavior are you currently excusing because of their numbers?
8. **Declare peacetime or wartime** (Horowitz): most CEO advice assumes peacetime. Under existential threat, centralize decisions, tolerate less deviation from plan, communicate bluntly. Pick the mode explicitly; blending them reads as inconsistency.

## By Company Stage

| Stage | CEO focus | Stage-specific failure |
|-------|-----------|------------------------|
| **Pre-PMF** | Weekly customer contact, fast iteration, retention signal, guard runway | Scaling spend before retention proves PMF |
| **Series A** | Repeatable go-to-market, first exec hires, board rhythm | Hiring execs built for a company two stages larger |
| **Series B** | Delegate operations, org design, second bet | CEO still deciding everything — becomes the rate limiter |
| **Series C+** | Multi-product, M&A, succession, public-company hygiene | Managing to reported metrics instead of operating metrics |

## Output Gates

Before issuing a recommendation or decision, check:

- Did I establish stage, runway in months, and board composition? Advice that skips these defaults to generic Series-B peacetime advice.
- Is this a one-way or two-way door, and does my recommended speed match?
- Does the decision carry a named owner and a review date?
- If it is bad news, does the plan travel with it? Never hand a board or team a problem without the proposed response.
- Can the current team execute this, or does it assume execs the company does not have?

## Crisis Navigation

| Crisis type | First 24h | First week | Recovery |
|-------------|-----------|------------|----------|
| Cash crisis | Freeze all spending, inventory options | Cut once, cut deep, communicate the plan | Execute plan, rebuild trust |
| PR crisis | One voice, acknowledge established facts only | Investigate root cause | Systemic fix, not optics |
| Key person leaves | Secure knowledge transfer | Interim owner, start search | Use as culture reset |
| Product failure | Contain damage, customer comms | Root cause analysis | Ship fix, rebuild confidence |
| Market shift | Assess exposure, scenario model | Strategic options to board | Pivot or double down |
| Mixed / unclear | Treat as cash crisis first | Reassess with real data | Cash converts any other crisis into a fatal one |

**Crisis principles:**
- The team mirrors your demeanor — move fast, never visibly panic.
- Communicate more than feels necessary; silence creates a rumor vacuum.
- One decision-maker; committees dissolve in crisis.
- A second layoff destroys more trust than one deep cut (Horowitz) — size the first cut for the bear case.
- Document everything in writing; you will need the timeline later.

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Skipping board pre-wiring | Directors decide cold, in the room, anchored by the loudest voice | 15-30 min call with each director before every consequential meeting |
| Hiring a senior exec mid-crisis | A wrong exec costs 6-12 months, landing exactly when you cannot afford it | Interim internal owner or fractional exec until stable |
| Ignoring runway until under 6 months | Desperation raise: bad terms, wrong partners, zero leverage | Review runway weekly; start the process at 12-18 months |
| Five or more priorities | Teams optimize locally; nothing compounds | Three max, published with the kill list |
| Avoiding the hard conversation | The problem compounds and the team learns you tolerate it | Same-week feedback; `decisions.md` if genuinely stuck |
| Delegating culture to HR | Culture is set by promotion and firing decisions — only the CEO makes those at the top | Use the levers in `people.md` yourself |
| Founder mode on everything, forever | You become the org's bottleneck | Delegate by task-relevant maturity (Grove): per task, not per person |
| Reading quick consensus as alignment | On a big bet, fast agreement means the debate never happened | Assign a dissenting voice before deciding (→ `decisions.md`) |

## Founder vs Professional CEO

The operating question is not the label — it is whether the founder should still hold the seat.

- **Founder keeps the seat while**: their learning rate matches company growth, top talent still joins to work with them, and the board trusts them in a crisis.
- **Hand-off signals**: repeated exec turnover traced back to the founder, the founder managing around their own gaps instead of hiring for them, boredom in a pure-execution phase.
- **Try the middle path first**: a strong COO covering the founder's gaps beats a CEO swap — replacing the founder resets vision, conviction, and often the top team.
- **Keeper test, both directions** (Hastings/Netflix): would the board fight to keep this CEO? Would the CEO fight to keep each exec? A "no" either way is the real conversation.

## Security & Privacy

This skill provides strategic guidance only.

**Data handling:**
- No external API calls
- No data leaves your machine
- No persistent storage required

**This skill does NOT:**
- Store confidential business data
- Make network requests
- Access files outside its auxiliaries

## Human-in-the-Loop

Escalate to human for:
- Major pivots or shutdowns
- Executive terminations
- Fundraising term negotiations
- M&A decisions
- Crisis public communications
- Board seat changes

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/ceo (install if the user confirms):
- `cfo` — financial modeling and capital allocation
- `coo` — operations and scaling execution
- `cmo` — marketing strategy and growth
- `cto` — technical leadership and architecture
- `business` — strategy validation and planning

## Feedback

- If useful, star it: https://clawic.com/skills/ceo
- Latest version: https://clawic.com/skills/ceo

---

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/ceo.
