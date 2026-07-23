---
name: aws
slug: aws
version: 1.0.3
description: >-
  Architect, deploy, and optimize AWS infrastructure — service selection, cost control,
  security hardening. Use when the user mentions AWS, EC2, S3, Lambda, RDS, VPC, IAM,
  CloudFront, DynamoDB, or a surprise cloud bill.
homepage: https://clawic.com/skills/aws
changelog: Deeper service selection and cost guidance
metadata:
  clawdbot:
    emoji: ☁️
    requires:
      bins:
      - aws
    install:
    - id: brew
      kind: brew
      formula: awscli
      bins:
      - aws
      label: Install AWS CLI (Homebrew)
    os:
    - linux
    - darwin
    - win32
    displayName: AWS | Amazon Web Services
    configPaths:
    - ~/clawic/aws/
---

Working data lives in `~/clawic/aws/` — account context, resource inventory, cost notes. If you have data at the old `~/aws/` location, move it to `~/clawic/aws/`.

## Setup

On first use, read `setup.md` for the intake questions. The skill works immediately — setup only personalizes it.

```
~/clawic/aws/
├── memory.md        # Account context + preferences (template: memory-template.md)
├── resources.md     # Active infrastructure inventory
└── costs.md         # Cost tracking + alerts
```

## When To Use

- Architecting or deploying anything on AWS: service selection, VPC layout, IaC
- Diagnosing a surprise bill or cutting existing spend
- Security hardening: IAM, S3 exposure, network isolation, credential hygiene
- Performance work: Lambda cold starts, RDS connection exhaustion, EBS throughput
- Not for provider-agnostic architecture debates — that is `cloud`

## Quick Reference

| Situation | Play | Detail |
|-----------|------|--------|
| "My bill exploded" | Cost Explorer grouped by service, then walk the Cost Traps table | `costs.md` |
| Greenfield project | Stage table (Rule 3), smallest viable stack, billing alarm before first resource | Rule 2 |
| "Is my account secure?" | Run the audit checklist end to end | `security.md` |
| Picking between services | Decide by threshold (payload size, duty cycle, latency), not feature lists | `services.md` |
| Inherited/unknown account | Inventory commands (Rule 1), record findings in `~/clawic/aws/resources.md` | Rule 1 |
| Anything else AWS | Answer directly; state cost and security implications of whatever you recommend | — |

## Core Rules

### 1. Inventory Before Architecture
Never propose new infrastructure into an unknown account. Minimum discovery:
```bash
aws sts get-caller-identity
aws ec2 describe-vpcs --query 'Vpcs[].{ID:VpcId,CIDR:CidrBlock,Default:IsDefault}'
aws ce get-cost-and-usage --time-period Start=$(date -v-30d +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE
```
The spend report tells you what actually exists faster than any console tour. Confirm region explicitly — "default us-east-1" is an assumption, not a fact.

### 2. Billing Alarm Before First Resource
On a fresh account, the first deploy is a budget + anomaly alert, not an EC2 instance (commands in `costs.md`). Every AWS horror story starts with "I didn't know it was still running." Alert thresholds: 80% of budget actual, 100% forecast.

### 3. Cost With Every Recommendation
State the monthly number when you recommend anything. Rough stages (us-east-1, on-demand — verify current pricing before committing):

| Stage | Recommended Stack | Monthly |
|-------|-------------------|---------|
| MVP (<1k users) | Single EC2 + RDS single-AZ | ~$50 |
| Growth (1-10k) | ALB + ASG + RDS Multi-AZ | ~$200 |
| Scale (10k+) | ECS/EKS + Aurora + ElastiCache | $500+ |

Default to the smallest viable instance: scaling up is a 2-minute operation; an oversized fleet bleeds silently. Right-sizing heuristic: avg CPU <20% over 14 days → step down one size (each step ≈ halves compute cost); sustained >70% → step up or scale out.

### 4. Smallest Blast Radius
Least-privilege IAM (scope to resource ARNs, not `*`), databases in private subnets, security groups referencing other security groups instead of CIDRs, encryption at rest by default. RDS and EBS encryption can only be enabled at creation — retrofitting means snapshot-copy-restore downtime, so it is never "later".

### 5. Everything in Code, Nothing Console-Only
Console is for exploration; anything that survives the session goes into Terraform/CloudFormation/CDK. Drift check before any change: `terraform plan` must be clean first, or you're about to codify someone's console hotfix as an accident.

### 6. Tag at Creation, Not Retroactively
Cost allocation tags only report from activation forward — tagging later never backfills history, so an untagged month is unattributable forever. Minimum set: `Environment`, `Project`, `Owner`.

## Traps

### Cost

| Trap | Why it bites | Do instead |
|------|--------------|------------|
| NAT Gateway | $0.045/hr + $0.045/GB processed; S3/DynamoDB traffic through NAT is pure waste | Gateway VPC endpoints for S3/DynamoDB — free |
| CloudWatch Logs ingestion | $0.50/GB ingested; retention only caps storage ($0.03/GB-mo), the cheap part | Fix log level first: debug logging at 1 GB/day ≈ $15/mo per log group before storage |
| Public IPv4 addresses | Billed since 2024 (~$3.6/mo each), including attached ones | Count them; release idle EIPs, prefer private subnets + ALB |
| Cross-AZ traffic | $0.01/GB each way; chatty microservices pay double per hop | Co-locate chatty pairs in one AZ; keep replicas cross-AZ, not RPC |
| Idle load balancers | ALB ~$16/mo minimum at zero traffic | Weekly sweep for zero-request ALBs (`costs.md`) |
| gp2 volumes | gp3 is ~20% cheaper with better baseline IOPS | `modify-volume` to gp3 — live, no downtime |
| Snapshots + orphaned EBS | Automated backups accumulate forever | DLM lifecycle policy; sweep `status=available` volumes |
| Secrets Manager by default | $0.40/secret/mo | SSM Parameter Store (free tier) unless you need rotation |
| Egress from EC2 | $0.09/GB to internet | Serve assets via CloudFront — first 1 TB/mo free |
| t3 unlimited mode | Sustained CPU above baseline bills credit overage silently | Watch `CPUSurplusCreditsCharged`; sustained load → bigger instance |

### Security

| Trap | Why it bites | Do instead |
|------|--------------|------------|
| "Private" ACL, public bucket | Bucket policies override ACLs — console shows private, policy exposes all | Account-level Block Public Access; audit commands in `security.md` |
| IMDSv1 enabled | SSRF → instance credentials theft (the Capital One vector) | Require IMDSv2: `--http-tokens required` |
| RDS publicly accessible | Console default was Yes for years; scanners find it in hours | Verify `PubliclyAccessible` on every instance |
| Long-lived IAM user keys | Keys in code/laptops leak; age is risk | Roles + temporary credentials; rotate anything >90 days old |
| SSH open to 0.0.0.0/0 | Brute-force magnet, and unnecessary | SSM Session Manager — no inbound port at all |
| All-open outbound SGs | Exfiltration path nobody watches | Restrict egress to required ports/destinations |

## Performance Patterns

**Lambda cold starts:** init SDK clients outside the handler; keep the package small (limits: 50 MB zipped / 250 MB unzipped); interpreted runtimes cold-start in hundreds of ms, JVM in seconds. Provisioned concurrency fixes p99 but bills while idle — only for latency-sensitive endpoints.

**RDS connections:** limits derive from instance memory, not a lookup table. Postgres: `LEAST(DBInstanceClassMemory/9531392, 5000)` → db.t3.micro (1 GiB) ≈ 110; MySQL: `DBInstanceClassMemory/12582880` ≈ 85 on the same box. Lambda at scale exhausts these instantly — put RDS Proxy in front.

**EBS volume types:**

| Type | Use Case | IOPS |
|------|----------|------|
| gp3 | Default (consistent, no burst credits) | 3,000 base, tunable |
| io2 | Databases needing guarantees | Up to 64,000 |
| st1 | Sequential big-data throughput | 500 MiB/s |

## Service Selection

Defaults — thresholds and break-evens in `services.md`:

| Need | Default | Why |
|------|---------|-----|
| Static site | S3 + CloudFront | Pennies/month, 1 TB free egress |
| API backend | Lambda + API Gateway (HTTP API) | Zero idle cost; HTTP API is 3.5× cheaper than REST API |
| Container app | ECS Fargate | No cluster management; EKS only if the team already runs k8s |
| Database | RDS PostgreSQL | Ad-hoc queries and joins; DynamoDB only for known access patterns |
| Cache/session | ElastiCache Redis | Sub-ms, beats DynamoDB latency |
| Queue | SQS | Default; SNS only for fan-out |
| Search | OpenSearch | Managed Elasticsearch |

## Output Gates

Before delivering an architecture or command, check:
- Did I state the monthly cost of what I just recommended?
- Did I check what already exists before proposing something new?
- Is anything holding data reachable from the public internet?
- Does this account have a billing alarm? If unknown, say so and offer to set one.
- Is any command destructive (delete/terminate/modify)? Require explicit user confirmation first.

## Where Experts Disagree

- **Lambda-first vs containers-first:** spiky or low traffic → Lambda wins on idle cost; duty cycle above ~40% or strict p99 latency → Fargate/ECS (break-even math in `services.md`).
- **Multi-account from day 1:** solo builder → one account + budgets is fine; any team or compliance need → AWS Organizations now, because moving live resources between accounts later is a migration project.
- **NACLs:** most practitioners leave them at default allow-all and do all control in security groups — stateless NACL rules cause return-traffic bugs that cost hours. Use NACLs only for explicit subnet-level deny lists.

## CLI Essentials

```bash
aws configure --profile myproject        # credentials per project
export AWS_PROFILE=myproject             # never rely on an implicit default profile
aws sts get-caller-identity              # confirm who/where before anything
aws ce get-cost-forecast --time-period Start=$(date +%Y-%m-01),End=$(date -v+1m +%Y-%m-01) \
  --metric UNBLENDED_COST --granularity MONTHLY
```

## Security & Privacy

**Credentials:** This skill uses the AWS CLI, which reads credentials from `~/.aws/credentials` or environment variables. The skill never stores, logs, or transmits AWS credentials.

**Local storage:** Preferences and context stored in `~/clawic/aws/` — no data leaves your machine.

**CLI commands:** All commands shown are read-only by default. Destructive operations (delete, terminate) require explicit user confirmation.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/<slug> (install if the user confirms):
- `infrastructure` — architecture decisions
- `cloud` — multi-cloud patterns
- `docker` — container basics
- `backend` — API design

## Feedback

- If useful, star it: https://clawic.com/skills/aws
- Latest version: https://clawic.com/skills/aws

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/aws.
