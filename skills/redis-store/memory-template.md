# Memory Template — Redis

Create `~/Clawic/data/redis-store/memory.md` with this structure:

```markdown
# Redis Memory

## Status
status: ongoing
last: YYYY-MM-DD

## Context
<!-- What Redis is used for here: cache, sessions, queue, rate limiting, primary store -->
<!-- Observed facts: version, topology, provider, instance size, whether persistence is on -->

## Incidents
<!-- What has already gone wrong: OOM, latency spike, lost writes, exposed instance -->

## Constraints
<!-- Commands the platform forbids, compliance requirements, tables/keyspaces that must not be touched -->

## Preferences
<!-- Key naming, Lua welcome or not, verbosity, whether admin commands may be run directly -->

---
*Updated: YYYY-MM-DD*
```

## Status Values

| Value | Meaning |
|-------|---------|
| `ongoing` | Still learning their workload and topology |
| `complete` | Their setup, limits and constraints are known |
