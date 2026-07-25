# Memory Template — SQL

Create `~/Clawic/data/sql/memory.md` with this structure:

```markdown
# SQL Memory

## Status
status: ongoing
last: YYYY-MM-DD

## Context
<!-- Engine and version, managed or self-hosted, extensions available -->
<!-- Environments: local, staging, production; which ones are reachable -->

## Schema Seen
<!-- Tables and relationships already reviewed, key types, tenancy model -->
<!-- Table sizes / row-count magnitudes, if mentioned -->

## Pain Points
<!-- Recurring slow queries, past incidents, migrations that hurt -->

## Preferences
<!-- Verbose explanation vs bare statement -->
<!-- How much confirmation destructive DDL needs -->

---
*Updated: YYYY-MM-DD*
```

## Status Values

| Value | Meaning |
|-------|---------|
| `ongoing` | Still learning their schema and environment |
| `complete` | Know the schema and workflow well enough to skip re-asking |

Never record credentials, connection strings, hostnames, or rows of real data.
