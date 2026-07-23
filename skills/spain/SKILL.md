---
name: spain
slug: spain
version: 1.0.1
description: Plans Spain trips with local-level picks - named restaurants, regional rules, timing, tourist-trap avoidance. Use when the user plans travel to Spain or asks about Spanish cities, food, culture, or logistics.
homepage: https://clawic.com/skills/spain
changelog: Deeper local knowledge across every guide
metadata:
  clawdbot:
    emoji: 🇪🇸
    requires:
      bins: []
      config:
      - ~/spain/
    os:
    - linux
    - darwin
    - win32
    displayName: Spain
---

## Setup

If `~/spain/` doesn't exist or is empty, read `setup.md` and begin.

## When To Use

User planning a trip to Spain or wanting local insight: where to eat, what to skip, regional differences, festivals, timing, and practical logistics. Not for learning the Spanish language — that's the `spanish` skill.

## Architecture

Memory lives in `~/spain/`. See `memory-template.md` for structure.

```
~/spain/
└── memory.md     # Trip context
```

## Quick Reference

| Topic | File |
|-------|------|
| **Cities** | |
| Madrid complete guide | `madrid.md` |
| Barcelona complete guide | `barcelona.md` |
| Sevilla complete guide | `sevilla.md` |
| San Sebastián & pintxos | `san-sebastian.md` |
| **Planning** | |
| Sample itineraries | `itineraries.md` |
| Where to stay by city | `accommodation.md` |
| Useful apps | `apps.md` |
| **Food & Drink** | |
| Regional dishes, restaurants | `food-guide.md` |
| Wine regions & bodegas | `wine.md` |
| **Experiences** | |
| Places, festivals, tips | `experiences.md` |
| Beach guide by coast | `beaches.md` |
| Hiking routes | `hiking.md` |
| Nightlife by city | `nightlife.md` |
| **Reference** | |
| 17 regions, what makes each special | `regions.md` |
| Culture, eating times, customs | `culture.md` |
| Traveling with children | `with-kids.md` |
| **Practical** | |
| Getting around | `transport.md` |
| Phone & internet | `telecoms.md` |
| Emergencies & safety | `emergencies.md` |
| Anything else (visas, weather, currency) | Answer inline; log the gap in `~/spain/memory.md` |

## Core Rules

### 1. Specific Over Generic
Every recommendation names a place + neighborhood + time. Not "try tapas in Spain" — "Casa Dani, Mercado de la Paz (Salamanca), best tortilla in Madrid; go before 13:30 or queue."
Check: if the answer would fit any European city, it fails.

### 2. Local Perspective
What locals actually do, not what guides say:
- Mercado de San Miguel = tourist trap → San Fernando, Antón Martín better
- La Rambla = pickpocket corridor → Gothic Quarter side streets, Gràcia
- Sangría = tourist tell → tinto de verano (what Spaniards drink)
- Flamenco dinner-show in Barcelona → flamenco is Andalusian; see it in Sevilla (Triana) or skip

### 3. Regional Rules Override National Rules
| Region | What changes |
|--------|----------------|
| País Vasco | Pintxos, not tapas. Bar tallies by toothpicks; you self-report your count. |
| Granada, Jaén, León | Free tapa with every drink — order drinks, food arrives |
| Valencia | Paella ONLY at lunch; a kitchen serving dinner paella is cooking for tourists |
| Cataluña | Catalan on signage. Politics sensitive — no opinions unless asked. |
| Galicia, Asturias | Atlantic climate: rain gear even in July |

### 4. Timing Is Everything
- Lunch 14:00-16:00, dinner 21:00-23:00; kitchens close between services — at 19:00 you will not be fed (bar snacks at best)
- Monday: many restaurants closed — check before crossing town
- August: family restaurants close the whole month; Madrid/Sevilla hotels drop prices while the coast doubles
- Sunday evening: much is shut — plan a pintxos crawl or a long lunch instead

### 5. Book-Ahead Ladder
Work backwards from the hardest ticket; if it anchors the trip, book it before flights:

| Target | Lead time |
|--------|-----------|
| El Celler de Can Roca, 3-star tables | Waitlist 1+ year |
| San Fermín hotels (Pamplona, 6-14 July) | 6+ months |
| Semana Santa rooms in Sevilla | Months ahead; prices x3-4 |
| Alhambra (Granada) | 2-3 months in season — book the day dates are fixed |
| Michelin 1-2 star | 1-3 months |
| AVE promo fares | 2-3 weeks (€25-40 vs €100+ last minute) |
| Sagrada Família, Park Güell | Days-weeks; timed entry, no summer walk-ins |

### 6. Tourist-Trap Detection
Any two signals = walk away:
- Photos on the menu, or menu in 5+ languages
- Host outside pulling people in
- "Paella + sangría + flamenco" advertised together
- Terrace on the main square (≈2x price, half the quality)
- Giant display paella at Barcelona beach (reheated)

### 7. Match Trip Style

| Traveler | Focus on |
|----------|----------|
| Foodie | food-guide.md, wine.md, san-sebastian.md |
| Beach | beaches.md, regions.md |
| Culture | madrid.md, barcelona.md, sevilla.md |
| Adventure | hiking.md, experiences.md |
| Family | with-kids.md, beaches.md |
| Nightlife | nightlife.md, barcelona.md, madrid.md |

## Output Gates

Before giving a recommendation, check:
- Did I name a specific place, not a category?
- Is the timing valid? (no dinner at 19:00, no museum on its closing Monday)
- Is the advice region-correct? (no free-tapas promises outside Granada/Jaén/León, no dinner paella)
- Did I check their dates against August, Semana Santa, and local festivals?
- Are prices honest ranges, not invented exact figures?

## Traps

| Trap | Why it fails | Do instead |
|------|--------------|------------|
| Eating at 19:00 | Kitchens closed between services | Snack at 18:00, dine at 21:00 |
| Barcelona/Madrid in August | Locals gone, closures, 35-40°C | Go north, or take the hotel deals knowing restaurants close |
| Tipping 15-20% like USA | Not expected; staff are salaried | Round up or leave coins |
| Paying with €50 bills | Small places have no change | Break big bills at supermarkets |
| Beachwear in the city | Fined in some coastal cities; locals dress up | Cover up off the sand |
| Trusting "best paella" signs in tourist zones | Frozen and reheated | Rice restaurants at lunch, away from the seafront |
| Treating Spain as one culture | Basque, Catalan, Galician identities are real | Load regions.md before advising |

## Security & Privacy

**Data that stays local:** Trip preferences in ~/spain/

**This skill does NOT:** Access files outside ~/spain/ or make network requests.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):
- `travel` — Travel planning
- `food` — Food and cooking
- `spanish` — Spanish language

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/spain.

## Feedback

- If useful, star it: https://clawic.com/skills/spain
- Latest version: https://clawic.com/skills/spain
