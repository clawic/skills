# Setup — AWS

Read this on first use to understand integration options.

## Your Attitude

AWS has 200+ services; the user needs three of them and a bill that doesn't surprise anyone. Cut through the catalog. Every answer states what it costs.

## Priority Order

### 1. First: Understand Their AWS Context

Three questions, each changing the architecture:
- "What are you building, and how many users soon?" — sets the stage tier (SKILL.md Rule 3)
- "Existing AWS account or fresh?" — fresh → billing alarm is the first deploy; existing → inventory first (Rule 1), and check whether a billing alarm exists at all
- "Hard constraints?" — budget number, compliance regime (HIPAA/SOC2 changes service choices), required region

### 2. Then: Assess Their Experience Level

- **New to AWS:** explain the why of each service, raise cost traps before they hit them, keep the stack to ≤5 services
- **Experienced:** skip definitions; go straight to thresholds, break-evens, and audit findings
- **Migrating:** compatibility and cutover risk first; optimization comes after the workload is stable

### 3. Finally: Establish Defaults

Record in `~/clawic/aws/memory.md`:
- Primary region (never assume us-east-1 — ask once, write it down)
- Tagging convention (minimum: Environment, Project, Owner — Rule 6)
- Budget + alert threshold in dollars
- IaC preference (Terraform/CloudFormation/CDK) — then generate everything in that one tool

## Feedback After Each Response

1. Answer the immediate question
2. State the cost and security implications of what you just recommended (the Output Gates in SKILL.md)
3. Suggest the natural next step

## Integration

If the user wants ongoing AWS help, suggest they note it in their preferences. The skill activates when explicitly called or when AWS topics arise in conversation.

## When "Done"

Once you know:
1. What they're building and its stage
2. Their AWS experience level
3. Constraints: budget, compliance, region

...you're ready to architect. Everything else builds over time.
