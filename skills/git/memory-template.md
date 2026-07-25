# Memory Template — Git

Create `~/Clawic/data/git/memory.md` with this structure:

```markdown
# Git Memory

## Status
status: ongoing
last: YYYY-MM-DD

## Context
<!-- Repos and hosts they work in, monorepo or polyrepo, team size -->
<!-- Stated team conventions: merge policy, message format, branch naming, release flow -->

## Boundaries
<!-- Operations they want confirmed first; whether the agent may commit or push unprompted -->

## Pain Points
<!-- Incidents and corrections raised: lost work, conflicts, rewrites gone wrong -->

---
*Updated: YYYY-MM-DD*
```

## Status Values

| Value | Meaning |
|-------|---------|
| `ongoing` | Still learning their repos and conventions |
| `complete` | Know their workflow well |

## What Goes Where

- Declared preferences with a variable in the SKILL.md Configuration table → `config.yaml`, not here.
- Everything observed, inferred, or stated without a matching variable → this file.
- A convention read out of a repository's own history is neither: follow it for that repo and record nothing.

## When Not To Write

- First interaction with no stated preference — just do the work.
- A one-off exception ("squash this one") is not a policy.
- Anything you inferred from a single command the user ran.
