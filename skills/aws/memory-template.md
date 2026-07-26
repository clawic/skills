# Working File Templates — AWS

Read this file only when WRITING. `config.yaml` is what the user **declared**; the rest is what the agent **observed**. An observation never overwrites a declaration.

**Where each thing goes**

| Data | Home | Why |
|---|---|---|
| Declared preferences | `~/Clawic/data/aws/config.yaml` | Replaced, never grows |
| Account shape, pain points, cost history | `~/Clawic/data/aws/memory.md` | Rewritten in place; stays small |
| Hosts and managed instances | `~/Clawic/data/servers/servers.md` (**shared**) | Every provider in one inventory, so "my servers" has one answer |
| Credentials of any kind | Nowhere in these files | Profile, environment, or secret manager at call time; store at most a pointer |

**Start flat, split only when it hurts.** Everything begins inside `memory.md`. When a section passes ~40 lines or ~15 entries, move it to its own file in `~/Clawic/data/aws/` and leave one index line in `memory.md` saying when to open it — for example `Spend history (18 months) → spend-log.md; read before any cost comparison`. Until that happens, creating extra files only makes them less likely to be read.

**Contents:** [config.yaml](#configyaml) · [memory.md](#memorymd) · [shared servers inventory](#shared-servers-inventory) · [spend-log.md](#spend-logmd-only-once-it-outgrows-memorymd)

## config.yaml

Keys come from the Configuration table in `SKILL.md`. Write a key only when the user states the preference.

```yaml
default_region: eu-west-1
cli_profile: prod
iac_tool: terraform
monthly_budget_usd: 250
account_model: organization
compliance_regime: none

# Preference areas — free-form keys added as the user reveals them
conventions:
  tags: [Environment, Project, Owner, CostCenter]
  cidr_scheme: "10.<env>.0.0/16, /20 subnets"
platform:
  instance_families: [m7g, t4g]     # Graviton standard
safety_posture:
  destructive_commands: confirm-each
```

## memory.md

```markdown
# AWS Memory

## Status
status: ongoing
last: YYYY-MM-DD

## Account Context
<!-- What the account is for, its stage, who else touches it -->

## Current Infrastructure
<!-- High level only; the inventory belongs in resources.md -->

## Pain Points
<!-- Past incidents, surprise bills, things that burned them -->

## How They Work
<!-- Experience level, preferred answer shape, tolerance for detail -->

---
*Updated: YYYY-MM-DD*
```

| Status | Meaning |
|-------|---------|
| `ongoing` | Still learning their setup |
| `complete` | Know their account and workflow well |

## Shared servers inventory

Lives at `~/Clawic/data/servers/servers.md` and is shared with every other infrastructure skill. Create the file if it does not exist; append rows, never rewrite someone else's. One row per host, provider column included so an AWS box and a Hetzner box sit side by side:

```markdown
# Servers

| Name | Provider | Account / Project | Region | Type | Role | Monthly | Access reference |
|------|----------|-------------------|--------|------|------|---------|------------------|
| api-prod-1 | aws | 111122223333 | eu-west-1 | m7g.large | API | $62 | ssm:/prod/api/key-name |
```

Access reference is a pointer only (SSM parameter name, profile name, key path). Never a key, token, or password.

The rest of the AWS inventory — databases, buckets, VPCs — is AWS-shaped and stays in this skill: keep it inside `memory.md` under `## Current Infrastructure` while it fits, and split it into `~/Clawic/data/aws/resources.md` once it passes ~40 lines, leaving the index line behind:

```markdown
# AWS Resources — account 111122223333

## Databases
| Name | Engine | Class | Multi-AZ | Backup retention |
|------|--------|-------|----------|------------------|

## Storage
| Bucket / Volume | Region | Purpose | Lifecycle |
|-----------------|--------|---------|-----------|

## Networking
| VPC | CIDR | Subnets | NAT / endpoints |
|-----|------|---------|-----------------|

## Known Gaps
<!-- Seen but not yet understood; resources with no owner -->
```

## spend-log.md (only once it outgrows memory.md)

```markdown
# AWS Spend

## Monthly
| Month | Actual | Budget | Top service | Notes |
|-------|--------|--------|-------------|-------|

## Alerts Configured
- Budget: $X actual at 80%, forecast at 100%
- Anomaly subscription: $Y daily threshold

## Optimization Log
| Date | Change | Monthly saving |
|------|--------|----------------|

---
*Updated: YYYY-MM-DD*
```

The optimization log is the reason this file exists: without it, the same NAT gateway gets rediscovered every quarter and nobody can say what the last cleanup was worth.
