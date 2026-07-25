---
name: fishing
slug: fishing
version: 1.0.0
description: Track fishing spots, gear, catches, and conditions with personalized recommendations.
homepage: https://clawic.com/skills/fishing
metadata:
  clawdbot:
    emoji: 🎣
    requires:
      bins: []
    os:
    - linux
    - darwin
    - win32
    displayName: Fishing
---

## When to Use

User wants to track their fishing activity, remember favorite spots, log catches, or get personalized gear and technique recommendations based on their history.

## Architecture

Memory lives in `~/Clawic/data/fishing/`. See `memory-template.md` for setup.

```
~/Clawic/data/fishing/
├── memory.md          # HOT: preferences, gear, active spots
├── catches.md         # WARM: catch log with dates, species, conditions
├── spots.md           # WARM: saved locations with notes
└── archive/           # COLD: past seasons
```

## Quick Reference

| Topic | File |
|-------|------|
| Memory setup | `memory-template.md` |
| Species guide | `species.md` |
| Tackle reference | `tackle.md` |

## Core Rules

### 1. Check Memory First
Before any recommendation, read `~/Clawic/data/fishing/memory.md` for:
- User's gear inventory
- Preferred species
- Skill level
- Local regulations they've noted

### 2. Log Catches Proactively
After user reports a catch, update `~/Clawic/data/fishing/catches.md`:

| Date | Species | Weight | Spot | Conditions | Technique |
|------|---------|--------|------|------------|-----------|
| YYYY-MM-DD | Bass | 3.5 lb | Lake X | Cloudy, 65F | Texas rig |

### 3. Learn Spot Patterns
Track what works at each location in `~/Clawic/data/fishing/spots.md`:
- Best times (dawn, dusk, tide)
- Productive techniques
- Seasonal notes

### 4. Personalize Recommendations
Use catch history to suggest:
- "Last 3 bass at Lake X were on cloudy mornings with plastics"
- "You haven't tried spot Y since spring—spawning season now"

### 5. Match Tackle to Inventory
Only recommend gear the user owns (from memory.md). If suggesting new gear, mark it clearly as a purchase suggestion.

## Fishing Traps

- Recommending gear user doesn't own → check inventory first
- Generic advice ignoring history → always reference past catches
- Forgetting seasonal patterns → review catches.md by month
- Not updating spots.md → stale recommendations

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):
- `plan` — trip planning
- `remind` — trip reminders

## Feedback

- If useful, star it: https://clawic.com/skills/fishing
- Latest version: https://clawic.com/skills/fishing
