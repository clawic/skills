# Working File Templates — AWS

Everything lives in `~/Clawic/data/aws/`. `config.yaml` is what the user **declared**; the files below are what the agent **observed** or **recorded**. An observation never overwrites a declaration.

**Contents:** [config.yaml](#configyaml) · [memory.md](#memorymd) · [resources.md](#resourcesmd) · [spend-log.md](#spend-logmd)

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

## resources.md

Written by an inventory pass (SKILL.md Rule 1) so the next session starts from facts, not from a rediscovery.

```markdown
# AWS Resources — account 111122223333

## Compute
| Name | Type | Region/AZ | Purpose | Monthly |
|------|------|-----------|---------|---------|

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
<!-- Things seen but not yet understood, resources with no owner -->

---
*Inventoried: YYYY-MM-DD*
```

## spend-log.md

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
