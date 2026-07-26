# Cost Control

Prices: us-east-1, on-demand, recorded early 2026. The **ratios and break-evens are stable**; verify the absolute number on the pricing page before committing money. Every threshold below scales with `monthly_budget_usd`.

**Contents:** [Set Up the Alarms Before the Resources](#set-up-the-alarms-before-the-resources) · [Diagnosing a Surprise Bill](#diagnosing-a-surprise-bill) · [The Ten Biggest Line Items](#the-ten-biggest-line-items) · [Commitment Discounts — After Right-Sizing, Never Before](#commitment-discounts--after-right-sizing-never-before) · [S3 Lifecycle Economics](#s3-lifecycle-economics) · [Free Tier and Support](#free-tier-and-support) · [Monthly Review Checklist](#monthly-review-checklist)

**Before answering any spend question**, read `## Spend` in `~/Clawic/data/aws/memory.md` — or `spend-log.md` if the `## Boxes` index points there. A current-month number with no prior months is not an answer.

**After any bill review or saving**, write it back in the same turn: the month row with its `As of` date, the top three services, and any optimization (`memory-template.md`).


## Set Up the Alarms Before the Resources

```bash
# Anomaly detection is two calls, in this order. Cost Explorer APIs are us-east-1 only.
# 1. The monitor: what gets watched. Nothing can subscribe until this exists.
aws ce create-anomaly-monitor --anomaly-monitor '{
  "MonitorName": "account-wide",
  "MonitorType": "DIMENSIONAL",
  "MonitorDimension": "SERVICE"
}'   # returns MonitorArn — paste it below

# 2. The subscription: who hears about it, and above what size.
#    Threshold near DAILY spend (monthly_budget_usd ÷ 30), not monthly: $100/mo budget → 3.
aws ce create-anomaly-subscription --anomaly-subscription '{
  "SubscriptionName": "cost-alerts",
  "MonitorArnList": ["arn:aws:ce::111122223333:anomalymonitor/EXAMPLE"],
  "Subscribers": [{"Type": "EMAIL", "Address": "you@email.com"}],
  "Frequency": "DAILY",
  "ThresholdExpression": {
    "Dimensions": {
      "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
      "Values": ["3"],
      "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
    }
  }
}'

# Budget with an 80%-actual alert, matching the $100/mo assumed above
aws budgets create-budget --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget '{
    "BudgetName": "Monthly",
    "BudgetLimit": {"Amount": "100", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[{
    "Notification": {"NotificationType": "ACTUAL", "ComparisonOperator": "GREATER_THAN", "Threshold": 80},
    "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "you@email.com"}]
  }]'
```

Add a second notification at 100% `FORECASTED` — it fires early in the month, while there is still time to act, rather than after the money is spent.

Two details that break the paste: the flat `--threshold` flag is superseded by `ThresholdExpression` (the old form is still accepted on some CLI versions and ignored by newer ones), and `"Frequency": "IMMEDIATE"` requires an SNS subscriber — `EMAIL` only accepts `DAILY` or `WEEKLY`.

## Diagnosing a Surprise Bill

```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -u -d '30 days ago' +%Y-%m-%d 2>/dev/null || date -v-30d +%Y-%m-%d),End=$(date -u +%Y-%m-%d) \
  --granularity DAILY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE
```

Four steps, in order:

1. **Group by service at DAILY granularity.** Daily, not monthly — a step change has a date, and the date is most of the diagnosis.
2. **Group the top service by usage type.** `DataTransfer-Out-Bytes` and `BoxUsage:t3.medium` are different problems with different fixes. This is the step people skip.
3. **Map the start date to a deploy.** Cost Explorer gives you a timestamp for free; CloudTrail says who changed what that day: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<Api>`.
4. **Decide: one-time or run-rate?** A single large transfer is an incident. A raised floor is a new monthly cost that compounds until someone fixes it.

Do not diagnose from the Billing console's current month: it lags, and it hides the usage-type breakdown that names the cause.

## The Ten Biggest Line Items

The summary table is in SKILL.md, Cost Reflexes; the arithmetic and commands are here.

### 1. NAT Gateway — $0.045/hr (~$33/mo) + $0.045/GB

A private-subnet app pulling 300 GB/mo from S3 through NAT pays ~$13.50/mo for traffic a gateway endpoint moves free.

```bash
aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.s3 --route-table-ids rtb-xxx
```

Gateway endpoints (S3, DynamoDB) are free — there is no case for routing that traffic through NAT. Interface endpoints cost ~$0.01/hr per AZ (~$7.30/mo) plus $0.01/GB; when the NAT stays for other traffic, they break even at `7.30 ÷ (0.045 − 0.01)` ≈ **210 GB/mo per endpoint per AZ**. If the endpoint lets you delete the NAT entirely, it wins immediately.

### 2. CloudWatch Logs Ingestion — $0.50/GB

Ingestion is 16× the storage price ($0.03/GB-mo). Retention caps the cheap part; the **log level** controls the expensive part. 1 GB/day of debug logs ≈ $15/mo per log group before a byte is stored.

```bash
aws logs describe-log-groups --query 'logGroups[?!retentionInDays].[logGroupName,storedBytes]'
aws logs put-retention-policy --log-group-name /aws/lambda/fn --retention-in-days 14
```

Logs Insights adds $0.005 per GB scanned — a debugging afternoon over a wide time range is its own line item.

### 3. Data Transfer — the whole map, once

| Path | Price |
|---|---|
| Into AWS from the internet | Free |
| Out to the internet from EC2 | $0.09/GB at the first tier, after a monthly free allowance |
| Out via CloudFront | First 1 TB/mo free, then below EC2 egress |
| Between AZs in a region | $0.01/GB **each way** ($0.02 per round trip) |
| Within one AZ, over private IPs | Free |
| Within one AZ, over a public or Elastic IP | Billed as if cross-AZ — the silent one |
| Between regions | Varies by pair; always more than cross-AZ |

Two actions follow: co-locate chatty pairs in one AZ (spend cross-AZ money on replicas, not RPC), and keep in-VPC traffic resolving to private addresses (`enableDnsSupport` + `enableDnsHostnames` on the VPC) so it never leaves and comes back.

### 4. Public IPv4 — ~$0.005/hr (~$3.6/mo) each

Billed since 2024, attached or not.

```bash
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]'
```

Ten forgotten Elastic IPs plus a public IP per instance is real money on a small account. Private subnets behind an ALB remove the line entirely.

### 5. Idle Load Balancers — ALB ~$16/mo at zero traffic

```bash
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB \
  --metric-name RequestCount --dimensions Name=LoadBalancer,Value=app/my-alb/50dc6c495c0c9188 \
  --start-time "$(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -v-7d +%Y-%m-%dT%H:%M:%S)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" --period 604800 --statistics Sum
```

The dimension value is the ALB ARN suffix (`app/<name>/<id>`), not the full ARN — a mismatch returns an empty datapoint list that reads exactly like zero traffic. Zero requests over seven days usually means a load balancer nobody removed after a migration.

### 6. Orphaned Storage

```bash
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Created:CreateTime}'
aws ec2 describe-snapshots --owner-ids self \
  --query "Snapshots[?StartTime<=\`$(date -u -d '90 days ago' +%Y-%m-%d 2>/dev/null || date -v-90d +%Y-%m-%d)\`].[SnapshotId,VolumeSize,StartTime]"
```

Terminating an instance leaves its volumes behind unless `DeleteOnTermination` was set, and snapshots accumulate forever without a DLM policy. Snapshot Archive is substantially cheaper per GB for snapshots kept beyond 90 days, but it has a 90-day minimum and a restore that takes up to 72 hours — correct for compliance retention, wrong for anything you might need on Tuesday.

### 7. gp2 Volumes

gp3 is ~20% cheaper ($0.08 vs $0.10/GB-mo) with 3,000 baseline IOPS and no burst-credit cliff. The migration is live: `aws ec2 modify-volume --volume-id vol-xxx --volume-type gp3` — once per volume per 6 hours.

### 8. Oversized Instances

The right-sizing heuristic and its formula live in SKILL.md Rule 3. This is how to measure it.

```bash
aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
  --metric-name CPUUtilization --dimensions Name=InstanceId,Value=i-xxx \
  --start-time "$(date -u -d '14 days ago' +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -v-14d +%Y-%m-%dT%H:%M:%S)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%S)" --period 3600 --statistics Average
aws compute-optimizer get-ec2-instance-recommendations   # includes memory when the CloudWatch agent is installed
```

CPU alone under-diagnoses memory-bound apps — check memory before downsizing a JVM or a database.

### 9. Non-Production Running 24/7

A dev environment used during business hours runs ~60 of 168 hours a week. Stopping it the rest of the time saves `1 − 60/168` ≈ **64%** of its compute, and it is the single largest saving available on most small accounts.

- EC2 and RDS both support scheduled stop/start (Instance Scheduler, or an EventBridge rule plus a Lambda).
- **A stopped RDS instance still bills for storage and auto-starts after 7 days.** For anything idle longer than a week, snapshot and delete instead; the snapshot costs a fraction of the instance.
- ECS and Fargate services scale to zero on a scheduled action and cost nothing while there.

### 10. Managed Services on Autopilot

| Service | The default that costs |
|---|---|
| Secrets Manager | $0.40/secret/mo — Parameter Store SecureString is free for standard parameters |
| KMS | $1/mo per customer-managed key plus $0.03 per 10,000 requests — S3 Bucket Keys cut those requests by up to 99% on an SSE-KMS bucket |
| AWS Config | Billed per configuration item recorded; all resource types in all regions on a busy account is a surprise |
| CloudWatch custom metrics | $0.30/metric/mo, multiplied by every unique value of a high-cardinality dimension |
| CloudTrail data events | $0.10 per 100,000 events; S3 and Lambda data events dwarf the free management trail |
| Athena | $5/TB scanned — partitioning plus columnar formats routinely cut a query by an order of magnitude |
| t3 unlimited mode | Sustained CPU above baseline bills surplus credits silently; watch `CPUSurplusCreditsCharged` |
| Marketplace AMIs | An hourly software charge on top of the instance, invisible in EC2's pricing page |

## Commitment Discounts — After Right-Sizing, Never Before

**Break-even rule: commit only if expected uptime ≥ (1 − discount).** A 1-year no-upfront Savings Plan at ~30% off costs 0.70 × always-on on-demand, so anything running less than 70% of hours is cheaper on-demand. A 3-year all-upfront at ~60% off breaks even at 40% uptime — if the workload survives three years.

| Term | Payment | Typical saving |
|------|---------|----------------|
| 1 year, no upfront | Monthly | ~30% |
| 1 year, all upfront | One-time | ~40% |
| 3 year, all upfront | One-time | ~60% |

- **Compute Savings Plans over standard RIs** for anything uncertain: a similar discount that follows you across instance families, regions, Fargate, and Lambda. Standard RIs buy a few more points in exchange for committing to one family.
- Commit to the **floor**, not the average. Cover your minimum sustained usage and leave the peak on-demand; that shape never strands a commitment.
- Track both sides: `aws ce get-savings-plans-utilization` (are you using what you bought) and `get-savings-plans-coverage` (how much on-demand is left uncovered). Utilization below ~95% means you over-committed.
- Spot is a different instrument: up to ~90% off with a 2-minute reclaim notice, for batch, CI runners, and stateless workers behind a queue. Never for a single-node database or un-checkpointed state.
- Graviton (arm64) is cheaper per hour for the same task shape and stacks with Savings Plans — it needs an arm64 or multi-arch image, or the task dies with `exec format error`.

## S3 Lifecycle Economics

The costs hiding inside the savings: transitions bill per request (order of $0.01-0.05 per 1,000, by target tier); IA charges a 30-day minimum and treats objects under 128 KB as 128 KB, Glacier Flexible has a 90-day minimum and Deep Archive 180; retrieval fees apply to both; and incomplete multipart uploads bill invisibly until an `AbortIncompleteMultipartUpload` rule expires them. Short version — aggregate small objects before archiving, and never transition millions of tiny ones.

## Free Tier and Support

- The 12-month free tier expires. The classic first surprise bill is month 13 on an account where nothing changed. Diary the anniversary the day you open the account.
- Always-free allowances (Lambda requests, DynamoDB storage, CloudWatch basics) do not expire, but several are per-account rather than per-Organization — a multi-account split can multiply or dilute them depending on the service.
- Business support is billed as a percentage of monthly usage with a floor, so a small account pays the floor, which can exceed its infrastructure spend. Developer support is enough until somebody is on call.

## Monthly Review Checklist

| Check | Action |
|-------|--------|
| Top-5 services vs last month | Explain every delta over 20%, or flag it |
| Usage types that did not exist last month | Something got deployed; find out what |
| Idle EC2/RDS instances | Stop, schedule, or terminate |
| Unattached EBS volumes; snapshots past the retention policy | Delete, or hand to DLM |
| Unassociated Elastic IPs | Release |
| gp2 volumes remaining | Migrate to gp3 |
| Log groups with no retention, plus the top ingestion groups | Set the policy; fix the log level on the leader |
| Instances under 20% average CPU over 14 days | Downsize one step (SKILL.md Rule 3) |
| Savings Plan utilization and coverage | Under 95% utilization = over-committed; large uncovered steady spend = under-committed |
| Non-production running outside business hours | Schedule it |
| Untagged spend | Anything unattributable is next month's argument — fix the tags now (SKILL.md Rule 6) |

Record the date this ran in the `## Due` table of `memory.md`, and the month row in `## Spend`. A checklist with no last-run date gets skipped for a quarter and nobody notices.
