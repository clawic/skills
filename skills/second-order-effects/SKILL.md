---
name: second-order-effects
slug: second-order-effects
version: 1.0.0
description: Analyze decisions by tracing consequences beyond immediate outcomes to second and third-order effects.
homepage: https://clawic.com/skills/second-order-effects
metadata:
  clawdbot:
    emoji: 🔮
    requires:
      bins: []
    os:
    - linux
    - darwin
    - win32
    displayName: Second Order Effects
---

## Setup

If `~/Clawic/data/second-order-effects/` doesn't exist, or user's memory file shows setup incomplete, read `setup.md` first.

## When to Use

User faces a decision with non-obvious downstream effects. Agent traces consequences through multiple orders, identifies hidden risks and opportunities, and stress-tests assumptions.

## Architecture

Memory lives in `~/Clawic/data/second-order-effects/`. See `memory-template.md` for structure.

```
~/Clawic/data/second-order-effects/
├── memory.md          # Preferences + past analyses
├── decisions/         # Archived decision analyses
│   └── YYYY-MM-DD_topic.md
└── patterns.md        # Learned consequence patterns
```

## Quick Reference

| Topic | File |
|-------|------|
| Setup process | `setup.md` |
| Memory template | `memory-template.md` |
| Analysis framework | `framework.md` |
| Common patterns | `patterns.md` |

## Core Rules

### 1. Always Go Three Levels Deep
First-order: What happens immediately?
Second-order: What does that cause?
Third-order: What does THAT cause?

Most people stop at first-order. Competitive advantage lives in second and third.

### 2. Consider All Stakeholders
Map who is affected at each order:
- Direct participants
- Indirect observers
- Market/ecosystem
- Future self

Each stakeholder creates new consequence chains.

### 3. Invert the Question
After mapping positive outcomes, ask: "What could go wrong at each level?"

| Order | Optimistic | Pessimistic |
|-------|------------|-------------|
| 1st | Direct benefit | Obvious risk |
| 2nd | Compounding gain | Hidden cost |
| 3rd | Strategic advantage | Systemic risk |

### 4. Time-Weight Consequences
Near-term consequences feel larger than they are. Apply discount:
- 1st order (now): weight 0.5
- 2nd order (weeks/months): weight 1.0
- 3rd order (years): weight 1.5

Decisions that sacrifice 2nd/3rd order for 1st are usually wrong.

### 5. Document Predictions
Every analysis should include falsifiable predictions with timestamps. Review quarterly. Update `patterns.md` when patterns emerge.

## Consequence Chain Format

Use this structure for every analysis:

```markdown
## Decision: [One sentence]

### First Order (Immediate)
- Effect 1 → leads to...
- Effect 2 → leads to...

### Second Order (Weeks-Months)
- [Effect 1] causes → ...
- [Effect 2] causes → ...

### Third Order (Months-Years)
- [Second-order effect] causes → ...

### Stakeholder Map
| Who | 1st Order | 2nd Order | 3rd Order |
|-----|-----------|-----------|-----------|

### Inversion (What Could Go Wrong)
- Risk at 2nd order: ...
- Risk at 3rd order: ...

### Decision: [Proceed/Pause/Reject] because [reason tied to 2nd/3rd order]
```

## Common Traps

- Stopping at first order → miss compounding effects
- Ignoring negative second-order effects → blindsided by hidden costs
- Over-weighting immediate pain → sacrifice long-term position
- Analysis paralysis → set time limit (15-30 min), then decide
- Confident predictions → use probability ranges, not certainties

## Scope

This skill ONLY:
- Analyzes decisions using consequence chains
- Stores analyses in `~/Clawic/data/second-order-effects/`
- Learns patterns from past decisions

This skill NEVER:
- Makes decisions for the user
- Accesses external data without request
- Modifies its own SKILL.md

## Security & Privacy

**Data that stays local:**
- Decision analyses in ~/Clawic/data/second-order-effects/
- Learned patterns and preferences

**This skill does NOT:**
- Send data externally
- Access files outside its directory
- Make network requests

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):
- `first-principles-thinking` - break problems to fundamentals
- `six-thinking-hats` - parallel thinking modes
- `strategy` - strategic planning frameworks

## Feedback

- If useful, star it: https://clawic.com/skills/second-order-effects
- Latest version: https://clawic.com/skills/second-order-effects
