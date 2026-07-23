# AWS Cost Optimization

Prices below: us-east-1, on-demand, checked early 2026 — verify against the pricing page before committing money.

## Cost Monitoring Setup

### Billing Alerts (Do This First)
```bash
# Anomaly detection (one-time, us-east-1)
aws ce put-anomaly-subscription --subscription-name cost-alerts \
  --threshold 50 --frequency DAILY

# Budget with 80%-actual alert
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
Set the anomaly threshold near your daily spend, not monthly — anomalies are daily events.

### Reading the Bill
```bash
# Last 30 days by service — always the first diagnostic
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE
```
Diagnosis order for a surprise bill: (1) group by service, (2) group the top service by usage type — "DataTransfer-Out" vs "BoxUsage" points to completely different fixes, (3) check the anomaly console for the start date. The start date usually maps to a deploy.

## Top Cost Traps

### 1. NAT Gateway
**$0.045/hr (~$33/mo) + $0.045/GB processed.** A private-subnet app pulling 300 GB/mo from S3 through NAT pays ~$13.50/mo for traffic that a gateway endpoint moves free.
```bash
aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.s3 --route-table-ids rtb-xxx
```
Gateway endpoints (S3, DynamoDB) are free. Interface endpoints for other services cost ~$0.01/hr per AZ — still cheaper than NAT once traffic passes ~15 GB/mo per endpoint, and they keep traffic off the public path.

### 2. CloudWatch Logs Ingestion
**$0.50/GB ingested — 16× the storage price ($0.03/GB-mo).** Retention policies cap the cheap part; the log *level* controls the expensive part. 1 GB/day of debug logs ≈ $15/mo per log group before storing a byte.
```bash
# Find groups with no retention (storage grows forever)
aws logs describe-log-groups \
  --query 'logGroups[?!retentionInDays].[logGroupName,storedBytes]'
aws logs put-retention-policy --log-group-name /aws/lambda/fn --retention-in-days 14
```

### 3. Idle Load Balancers
**ALB ~$16/mo minimum** at zero traffic.
```bash
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerArn' | \
  xargs -I{} aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB \
  --metric-name RequestCount --dimensions Name=LoadBalancer,Value={} \
  --start-time $(date -v-7d +%Y-%m-%dT%H:%M:%S) --end-time $(date +%Y-%m-%dT%H:%M:%S) \
  --period 604800 --statistics Sum
```

### 4. Public IPv4 Addresses
**~$0.005/hr (~$3.6/mo) each since 2024 — attached or not.** Ten forgotten EIPs plus per-instance public IPs is real money on small accounts.
```bash
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]'
```

### 5. Unattached EBS Volumes and Old Snapshots
```bash
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Created:CreateTime}'
aws ec2 describe-snapshots --owner-ids self \
  --query 'Snapshots[?StartTime<=`'$(date -v-90d +%Y-%m-%d)'`].[SnapshotId,VolumeSize,StartTime]'
```
Terminating an instance does not delete its volumes unless DeleteOnTermination was set. Use DLM lifecycle policies so snapshots expire themselves.

### 6. gp2 Volumes
gp3 is ~20% cheaper ($0.08 vs $0.10/GB-mo) with 3,000 baseline IOPS and no burst-credit cliff. Migration is live:
```bash
aws ec2 modify-volume --volume-id vol-xxx --volume-type gp3
```

### 7. Oversized Instances
Right-sizing heuristic (canonical): avg CPU <20% over 14 days → step down one size; sustained >70% → step up or scale out. Each size step ≈ halves/doubles compute cost.
```bash
aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
  --metric-name CPUUtilization --dimensions Name=InstanceId,Value=i-xxx \
  --start-time $(date -v-14d +%Y-%m-%dT%H:%M:%S) --end-time $(date +%Y-%m-%dT%H:%M:%S) \
  --period 3600 --statistics Average
aws compute-optimizer get-ec2-instance-recommendations   # cross-check, includes memory if agent installed
```
CPU alone under-diagnoses memory-bound apps — check memory before downsizing anything running a JVM or a database.

### 8. Cross-AZ Data Transfer
**$0.01/GB each way** ($0.02 per round trip). Chatty microservices across AZs pay per hop. Co-locate chatty pairs in one AZ; spend cross-AZ money on replicas and failover, not on RPC.

## Savings Strategies

Order of operations: right-size first, then commit. Committing to an oversized fleet locks in the waste.

### Reserved Instances / Savings Plans
| Term | Payment | Typical Savings |
|------|---------|-----------------|
| 1 year, no upfront | Monthly | ~30% |
| 1 year, all upfront | One-time | ~40% |
| 3 year, all upfront | One-time | ~60% |

**Break-even formula: commit only if expected uptime ≥ (1 − discount).** Worked: 1-yr no-upfront at ~30% off costs 0.70 × always-on on-demand; an instance running <70% of hours is cheaper on-demand. A 3-yr at ~60% off breaks even at just 40% uptime — but only if the workload survives 3 years.

Prefer Compute Savings Plans over standard RIs for anything uncertain: same discount ballpark, but they follow you across instance families, Fargate, and Lambda.

### Spot Instances (up to ~90% off)
For interruptible work only — 2-minute reclaim notice: batch jobs, CI/CD workers, stateless workers behind a queue. Never for single-node databases or anything holding un-checkpointed state.

### S3 Lifecycle — With the Transition-Cost Caveat
```bash
aws s3api put-bucket-lifecycle-configuration --bucket my-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "Archive old data",
      "Status": "Enabled",
      "Filter": {"Prefix": "logs/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"}
      ],
      "Expiration": {"Days": 365}
    }]
  }'
```
Traps inside the "savings":
- **Transition requests are billed** (order of $0.01–0.05 per 1,000 depending on target tier). Millions of tiny objects → the transition can cost more than a year of the storage saved. Aggregate small objects (tar/parquet) before archiving.
- **IA has a 30-day minimum charge and ~$0.01/GB retrieval fee; objects <128 KB don't get IA rates.** IA only pays off for data read less than roughly once a month.
- **Glacier tiers have minimum storage durations** (90/180 days) — early deletion bills the remainder.

### Intelligent-Tiering
Monitoring fee ~$0.0025 per 1,000 objects/mo, no retrieval fees. The right default when access patterns are unknown and objects are >128 KB; skip it for buckets of millions of tiny objects, where the monitoring fee exceeds the storage.

## Monthly Cost Review Checklist

| Check | Action |
|-------|--------|
| Top-5 services vs last month | Explain every delta >20% or flag it |
| Idle EC2 instances | Stop or terminate |
| Unattached EBS volumes / old snapshots | Delete or DLM |
| Unassociated public IPs | Release |
| gp2 volumes remaining | Migrate to gp3 |
| Log groups without retention | Set policy; check ingestion leaders |
| Oversized instances (Rule: <20% avg CPU, 14 days) | Downsize one step |
| Commitment coverage | On-demand spend that ran all month → RI/Savings Plan candidate |
