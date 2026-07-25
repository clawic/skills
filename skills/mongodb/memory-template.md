# Memory Template — MongoDB

Create `~/Clawic/data/mongodb/memory.md` with this structure:

```markdown
# MongoDB Memory

## Status
status: ongoing
last: YYYY-MM-DD

## Deployment
<!-- Replica set / sharded / Atlas tier, server version, region -->
<!-- Driver and framework in use; how the app connects -->

## Data Shape
<!-- Collections that keep coming up, their size and growth -->
<!-- Query patterns that matter; indexes already discussed -->

## Baselines
<!-- Observed normal values for the metrics in monitoring.md -->
<!-- e.g. cache dirty ~3%, replication lag <1s, connections ~400/1500 -->

## Incidents and Fixes
<!-- What broke, what the cause turned out to be, what was changed -->

## Preferences
<!-- Stances recorded under the preference areas in SKILL.md -->
<!-- Declared variables belong in config.yaml, not here -->

---
*Updated: YYYY-MM-DD*
```

## Status Values

| Value | Meaning |
|-------|---------|
| `ongoing` | Still learning their deployment and data |
| `complete` | Know their cluster, collections, and query patterns well |

Baselines are the highest-value section: "cache dirty at 12%" means nothing without the normal value, and the normal value is only ever learned by observing this specific cluster (→ `monitoring.md`).
