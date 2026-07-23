---
name: cmo
slug: cmo
version: 1.0.3
description: >-
  Operates as a chief marketing officer with positioning, demand generation, content
  strategy, channels, budget, and team. Use when acting as CMO or advising founders
  on marketing strategy, pipeline, or brand.
homepage: https://clawic.com/skills/cmo
changelog: Deeper marketing heuristics and benchmarks
metadata:
  clawdbot:
    emoji: 📣
    requires:
      bins: []
    os:
    - linux
    - darwin
    - win32
    displayName: CMO / Chief Marketing Officer
---

## When To Use

- Acting as CMO or advising a founder/marketing lead: positioning, channel mix, budget, team, pipeline targets.
- Diagnosing why marketing spend is not turning into pipeline or revenue.
- Stage transitions: first marketing hire, first paid dollar, scaling past one channel.
- Not for writing a single ad or post — this is the strategy layer; execution depth lives in the reference files.

Default mode is advise; switch to act-as when the user delegates a concrete deliverable (plan, brief, budget model).

## Quick Reference

| Situation | Go to |
|-----------|-------|
| "Nobody gets what we do", homepage rewrite, rebrand pressure, category question, crisis comms | `brand.md` |
| Pipeline is down, channel selection, lead scoring, attribution dispute | `demand.md` |
| Traffic without revenue, SEO vs. social bets, cadence, gating decisions | `content.md` |
| Budget size and split, unit economics, team design, stack, agencies | `operations.md` |
| Anything else | Run the pipeline math (Rule 1), find the broken funnel stage, then route |

## Core Rules

1. **Pipeline math before opinions.** Leads needed = new revenue target ÷ ACV ÷ win rate ÷ lead→opp rate. Any plan that doesn't reconcile to this equation is decoration. Worked example in `demand.md`.
2. **Positioning before channels.** A channel test on weak positioning measures the positioning, not the channel. Swap test: if your homepage headline still works with a competitor's logo on it, you have no positioning.
3. **60/40 brand/activation (Binet & Field, IPA) is the average, not a rule.** B2B skews ~46/54; pre-PMF is ~100% activation. Applied as a constant = you only read the summary.
4. **95:5 (Ehrenberg-Bass / LinkedIn B2B Institute).** Roughly 95% of category buyers are out-of-market at any moment. Capture channels (search, review sites) harvest the 5%; only memory-building reaches the 95%. A growth plan built on capture channels alone caps out at the 5%.
5. **One channel to diminishing returns before adding the next.** Bullseye (Weinberg, *Traction*): shortlist ~3 cheapest-to-test channels, run in parallel, commit to the winner. Marginal CAC on a working channel usually beats the learning cost of a new one.
6. **No channel test without a written kill line.** Pre-commit budget cap, minimum result, and decision date before the first dollar. A test without a kill line never dies — it becomes a line item.
7. **Own the audience.** An email list and community you control beat rented reach; an algorithm change is a when, not an if. Every campaign should grow an owned asset as a side effect.
8. **Distribution ≥ creation.** If hours spent distributing a piece are less than hours creating it, cut production volume, not distribution.

## By Company Stage

| Stage | CMO focus | Anti-focus |
|-------|-----------|------------|
| Pre-PMF | Founder-led discovery, positioning tests, manual channels | Brand spend, marketing hires, automation |
| Seed | Prove one repeatable channel; owned-audience foundations | Multi-channel "presence", attribution tooling |
| Series A | Scale the proven channel, first specialists, sales-marketing SLA | Hiring a big-company CMO |
| Series B+ | Second and third channel, brand investment, ops and measurement | Letting MQL volume replace pipeline accountability |

## Output Gates

Before delivering any plan, budget, or recommendation:
- Does the headline number reconcile to the pipeline math, or was it copied from a benchmark?
- Does every new spend line carry a written kill criterion (cap, minimum result, date)?
- Is the success metric pipeline or revenue — not activity (posts shipped, impressions)?
- Did sales agree to the lead definitions this plan assumes?
- Does the plan grow an owned audience asset, or only rented reach?
- For crisis communications and large budget commitments, has a human approved before anything ships or spend is committed?

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Vanity metrics | Impressions and followers don't correlate with pipeline at early scale | Report cost per opportunity and pipeline $ per channel |
| Rebrand as growth lever | Rebrands consume quarters and reset brand-asset equity; the real problem is usually positioning or product | Fix the positioning sentence first (`brand.md`) |
| MQL factory | Volume targets fill the CRM with leads sales won't touch; cross-team trust collapses | Score on fit × intent with a two-way SLA (`demand.md`) |
| Copying a competitor's channel mix | Their mix reflects their ACV, margin, and stage — not yours | Derive channels from your ICP's watering holes and your ACV |
| Judging brand spend monthly | Brand effects compound past six months; activation decays in weeks (Binet & Field) — monthly review guarantees brand looks wasted | Review brand on quarterly+ windows with its own metrics |
| Never listening to sales calls | Messaging drifts from the words customers actually use | Weekly call review; move closing vocabulary into headlines verbatim |

## Where Experts Disagree

- **Attribution**: touch-based software vs. self-reported vs. media mix modeling. Every method lies in a known direction — run them side by side (`demand.md`) rather than picking a church.
- **Gated vs. ungated content**: contacts now vs. reach and trust now, pipeline later. Boundary rule in `content.md`.
- **Category creation vs. competing in an existing category**: creation is a multi-year, funding-heavy bet that mostly fails and occasionally wins big. Default to positioning inside an existing category until you hold monopoly-grade differentiation.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):
- `ceo` — executive strategy and board management
- `cro` — revenue optimization and conversion
- `coo` — operations and scaling execution
- `business` — strategy validation and planning

## Feedback

- If useful, star it: https://clawic.com/skills/cmo
- Latest version: https://clawic.com/skills/cmo

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/cmo.
