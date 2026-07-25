# Memory Template — Kubernetes

Create `~/Clawic/data/k8s/memory.md` with this structure:

```markdown
# Kubernetes Memory

## Status
status: ongoing
last: YYYY-MM-DD

## Clusters
<!-- Name, distribution, environments, rough node count, who owns upgrades -->

## Workloads
<!-- The services that matter, their shape (stateless, StatefulSet, batch), their SLO -->

## Incident History
<!-- Recurring failure shapes and what actually fixed them -->

## Observed Habits
<!-- Only what you inferred and they never declared: which findings they act on, which they wave off, how they phrase a request. Anything declared (manifest tool, CPU limits stance, explanation depth, hardening appetite) lives in config.yaml -->

---
*Updated: YYYY-MM-DD*
```

## Status Values

| Value | Meaning |
|-------|---------|
| `ongoing` | Still learning their clusters and workloads |
| `complete` | Know their topology and conventions well |

Keep declared preferences in `config.yaml`, observations here. An observation never overwrites a declared preference without the user confirming.
