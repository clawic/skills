---
name: apple-search-ads
slug: apple-search-ads
version: 1.0.1
description: >-
  Plan, launch, and optimize Apple Search Ads (Apple Ads) campaigns via Campaign Management API v5.
  Covers bid math, keyword mining, AdServices and SKAdNetwork attribution, and scaling strategy.
  Use for paid user acquisition on iOS apps.
homepage: https://clawic.com/skills/apple-search-ads
changelog: Sharper strategy guidance and cleaner structure
metadata:
  clawdbot:
    emoji: 🍎
    configPaths:
    - ~/apple-search-ads/
    requires:
      bins:
      - curl
      - jq
      env:
      - ASA_CLIENT_ID
      - ASA_TEAM_ID
      - ASA_KEY_ID
      - ASA_ORG_ID
      - ASA_PRIVATE_KEY_FILE
    os:
    - linux
    - darwin
    displayName: Apple Search Ads
---

# Apple Search Ads 🍎

Toolkit for Apple Search Ads (rebranded "Apple Ads" in 2024; API paths still read `searchads`): Campaign Management API v5, attribution (AdServices + SKAdNetwork), bid math, and scaling strategy. All local state (memory, credentials, campaign notes, reports) lives in `~/apple-search-ads/`.

## When To Use

- Launching or restructuring campaigns for an iOS app
- Optimizing bids, budgets, keywords, or CPA on running campaigns
- Wiring attribution (AdServices, SKAdNetwork, MMP) into the app
- Automating reports and weekly optimization via the API
- Deciding scaling, multi-country expansion, or Custom Product Page tests
- Mode: acts directly via the API when the user grants credentials; otherwise advises on structure and strategy. Not for App Store metadata/screenshot work itself — route to `aso`; this skill only flags when CVR points there.

## Quick Reference

Route by what you need to do. Read one file; each is self-contained.

| Situation | Go to |
|-----------|-------|
| First run, `~/apple-search-ads/` missing, onboarding the user | `setup.md` |
| Saving apps, targets, campaign structure, learnings to memory | `memory-template.md` |
| Authenticating, base URL/headers, endpoints, payloads, campaign/ad group/keyword objects, report requests, metrics, error codes, rate limits | `api-reference.md` |
| Adding attribution in the iOS app (AdServices, SKAdNetwork conversion values, MMP integration, testing) | `ios-integration.md` |
| Planning structure, placements, match types, keyword tiers, bidding, benchmarks, budget allocation, scaling, multi-country, Custom Product Pages | `strategy.md` |
| Running curl/jq automation (get token, daily/search-term/keyword reports, add keywords/negatives, weekly optimization) | `scripts.md` |
| Anything else | Apply Core Rules below, then route to the closest file |

## Core Rules

### 1. Bid Backwards From LTV, Never From Competitors' Bids
Max CPT = target CPA × expected CVR. Target CPA $6 and CVR 50% → max CPT $3.00. A bid above your max CPT needs an explicit reason (brand defense); a bid copied from "what the niche pays" has none. CPA math lives in `strategy.md`.

### 2. Structure Is the Strategy: One Intent Per Campaign
Brand, category, competitor, discovery — separate campaigns. ASA has no portfolio bid strategies or shared budgets; campaign separation is the only budget and reporting control you get, so mixing intents removes your only lever.

### 3. "Exact Match" Is Not Exact
ASA exact match also serves plurals and common misspellings (Apple-documented). Before concluding a keyword works or fails, open the search term report and see which queries actually spent the money.

### 4. Graduate and Negate
Weekly: any search term with ≥2 installs and CPA ≤ target → add as exact match in its intent campaign AND as a negative where it was discovered. Skip the negative and the discovery campaign keeps buying the term, splitting its data forever.

### 5. Defend Your Brand
Competitors WILL bid your name, and your relevance advantage makes brand defense the cheapest CPA in the account. Impression share is reported as a range, not a point; if the high bound on brand terms sits below ~90%, raise brand bids.

### 6. One Country Per Campaign
CPT, CVR, and query language differ per market; blended reporting makes every bid decision wrong somewhere. Separate campaigns per country/region.

### 7. Spend Gates Before Scale
Keyword spend ≥ 2× target CPA with 0 installs → cut the bid sharply (30-50%) or pause. Example: target CPA $6 → any keyword that burned $12 with no install acts today, not at month end.

### 8. Bid Your True Max — the Auction Is Second-Price
Apple has described the auction as second-price: you pay just above the next-highest bid. Shading below your max CPT mostly loses auctions; it rarely saves money. Cap risk with the spend gate (Rule 7), not with timid bids.

## Output Gates

Before pushing any change through the API:

- Is the new bid computed from max CPT (Rule 1), not from current bid ± gut feel?
- One variable per keyword/ad group per cycle (bid OR creative OR match type)?
- Is the decision backed by the minimum window in `strategy.md` (7 days; 14 for competitor campaigns)?
- Budget increase within the 20-30% step limit (`strategy.md` → Scaling)?
- Change and expected effect logged to `~/apple-search-ads/memory.md`?

## Setup

On first use, read `setup.md` for onboarding guidelines.

## Architecture

Memory lives in `~/apple-search-ads/`. See `memory-template.md` for structure.

```
~/apple-search-ads/
├── memory.md          # Active campaigns, preferences, learnings
├── credentials.md     # OAuth config (NEVER commit real secrets)
├── campaigns/         # Campaign-specific notes and performance
│   └── {app-id}/
├── reports/           # Generated reports
└── scripts/           # Custom automation
```

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Treating `cpaGoal` as a spend cap | It is advisory; Apple can spend far past it | Control CPA with bids and the spend gate (Rule 7) |
| Refining age/gender to "focus" targeting | Excludes every user with Personalized Ads off — a large, invisible slice | Stay broad; segment by keyword intent instead |
| Search Match ON in every ad group | Your own ad groups compete for the same queries; data fragments | Search Match only in the discovery campaign |
| Same bid across all keywords | Brand, category, and long-tail have different values | Tier bids off max CPT (`strategy.md`) |
| Judging dayparting by install timestamps | SKAN postbacks arrive hours to days late | Daypart on tap timestamps |
| Comparing ASA dashboard installs to MMP installs | Different attribution windows and models; they never match | Pick one source of truth per KPI; record it in memory.md |
| Launching on a weak product page | You pay per tap; low CVR taxes every dollar | Fix ASO first, then buy traffic |
| Mixing timezones across reports | ASA reports accept UTC or ORTZ; mixing misaligns days | Pick one and pass it in every request |
| No cross-campaign negatives | Category/discovery campaigns quietly buy your brand queries at brand-level CPTs | Add brand terms as negatives in every non-brand campaign |
| Scaling budget in big jumps | Volume spikes re-enter auctions at worse positions; CPA jumps | 20-30% per step (`strategy.md` → Scaling) |

## External Endpoints

| Endpoint | Data Sent | Purpose |
|----------|-----------|---------|
| `https://appleid.apple.com/auth/oauth2/token` | Client credentials (JWT) | Get access token |
| `https://api.searchads.apple.com/api/v5/*` | Campaign/keyword data | Campaign management |
| `https://api-adservices.apple.com/api/v1/` | Attribution token | Attribution data |

No other data is sent externally.

## Security & Privacy

**Data that leaves your machine:**
- Campaign configurations sent to Apple Ads API
- Attribution tokens sent to Apple (from iOS app)

**Data that stays local:**
- Credentials in `~/apple-search-ads/credentials.md`
- Reports and analysis
- Strategy notes

**This skill does NOT:**
- Store API secrets in plain text (use environment variables)
- Access user-level data (attribution is aggregated)
- Make requests to undeclared endpoints

## Trust

By using this skill, data is sent to Apple's Search Ads API and AdServices.
Only install if you trust Apple with your advertising data.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills (install if the user confirms):
- `app-store-connect` — manage apps and releases
- `aso` — App Store Optimization
- `analytics` — track metrics and KPIs
- `ios` — iOS development patterns

## Feedback

- If useful, star it: https://clawic.com/skills/apple-search-ads
- Latest version: https://clawic.com/skills/apple-search-ads

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/apple-search-ads.
