# Production — Reliability, Monitoring, and the Day It Breaks

Scope: taking something that works into something you are on call for. The gate at the bottom is the checklist; everything above it explains why each item is on it.

## Availability Is a Number You Choose and Pay For

Pick the target before designing, because each nine changes the architecture and the bill.

| Target | Downtime per month | What it takes |
|---|---|---|
| 99% | ~7.2 hours | Single AZ, restore from backup, humans in the loop |
| 99.9% | ~43 minutes | Multi-AZ, health checks, automated restart, tested restore |
| 99.95% | ~22 minutes | Multi-AZ everywhere, no single-instance components, automated failover |
| 99.99% | ~4.3 minutes | Multi-region active-active or warm standby, and an organisation that practises |

Formula: `downtime_minutes = 43,200 × (1 − target)`. Note that your availability is the *product* of every component in the request path — three components at 99.9% in series give 99.7%, not 99.9%. This is why the answer to "make it more reliable" is usually "have fewer things in the path".

## Multi-AZ Is the Default; Multi-Region Rarely Is

- Multi-AZ costs roughly double for stateful components and is the price of not being paged by a single AZ event. Every production RDS instance, every ASG, every load balancer spans at least two AZs.
- Multi-region doubles the entire operational surface: data replication, failover orchestration, quota requests in a second region, and a second set of everything to keep patched. Justify it with a regulatory requirement or a real RTO commitment, not with a diagram.
- The middle ground that is almost always correct: **pilot light** — data replicated cross-region (S3 replication, RDS cross-region read replica or copied snapshots), infrastructure defined in code, nothing running until you need it. Recovery is measured in hours and costs almost nothing until the day it matters.

## RTO and RPO Are Commitments, Not Adjectives

- **RPO** (how much data you can lose) is set by backup frequency and replication lag. Automated RDS backups with 5-minute PITR granularity give RPO ≈ 5 minutes. A nightly snapshot gives RPO = 24 hours, and someone should agree to that in writing.
- **RTO** (how long recovery takes) is only known if you have measured it. Restoring a 500 GB RDS snapshot is not instant, and the first restore is slower still because the volume lazy-loads from S3.
- Quarterly drill, timed: restore the newest snapshot into a scratch instance, point a copy of the app at it, and record how long it took and what was missing in `~/Clawic/data/aws/deploys/<year>.md`, updating the `## Due` table in `memory.md`. An untimed drill is a drill that did not happen. What breaks is never the data — it is the KMS grant, the parameter group, the security group, the DNS record, and the one credential nobody documented.
- Backups live in a different account when the threat model includes a compromised credential or an angry admin. AWS Backup with a vault in the log-archive account and a vault lock is the mechanism.

## Scaling That Works

- Target tracking beats step scaling for almost everything: set a target CPU or request-count-per-target and let AWS work out the steps. Step scaling is for metrics with known thresholds and non-linear responses.
- **Scale out fast, scale in slowly.** Aggressive scale-in during a traffic dip causes a thundering herd when it recovers. Asymmetric cooldowns are the default posture.
- The ASG health check grace period defaults to 300s; an instance that takes 400s to boot is terminated and replaced forever, which reads as a capacity problem and is a configuration problem.
- Warm pools cut scale-out latency for slow-booting AMIs at the cost of stopped-instance storage. Cheaper answer first: bake a better AMI so boot time stops mattering.
- Predictive scaling helps only with genuinely cyclical daily traffic and needs weeks of history. For spiky, unpredictable load, over-provision the floor instead.
- Everything downstream must scale too: an autoscaling app in front of a fixed-size RDS instance just moves the queue to the database.

## Quotas Are the Ceiling You Meet at 3am

```bash
aws service-quotas list-service-quotas --service-code ec2 \
  --query 'Quotas[?Adjustable==`true`].[QuotaName,Value]' --output table
aws service-quotas request-service-quota-increase \
  --service-code lambda --quota-code L-B99A9384 --desired-value 5000
```

- Defaults that bite earliest: 1,000 concurrent Lambda executions per region, 5 Elastic IPs per region, 60 inbound + 60 outbound rules per security group, 5 security groups per network interface, and per-family EC2 vCPU limits that are low on new accounts.
- Increases are approved by humans and capacity-dependent ones can take days. Request headroom before launch — a quota request filed during an incident is a request filed too late.
- Put a CloudWatch alarm on the quota-usage metrics AWS publishes (`AWS/Usage`) so you learn at 70% of a limit, not at 100%.

## Monitoring the Right Things

Four metrics answer "is it healthy" for almost any AWS service: **saturation** (how full), **errors**, **latency**, and **traffic**. Alarm on saturation and errors; graph latency and traffic.

| Layer | The alarm that earns its keep |
|---|---|
| Load balancer | `HTTPCode_ELB_5XX_Count` and `TargetResponseTime` p99 — the user's view |
| ASG / ECS | Running count below desired for more than one interval |
| RDS | `FreeStorageSpace`, `DatabaseConnections` against the formula ceiling, `CPUUtilization`, replica lag |
| Lambda | `Throttles`, `Errors`, and `IteratorAge` for stream sources |
| DynamoDB | `ThrottledRequests` and `SystemErrors` |
| Account | Budget actual and forecast, plus `AWS/Usage` against quotas |

- **`TreatMissingData` defaults to `missing`, which means an alarm does not fire when the metric stops being published** — the exact situation of a dead instance. Set it to `breaching` for liveness alarms deliberately.
- Composite alarms suppress the flood: one page for "the service is down" instead of fourteen for its components. Alarm fatigue is an outage cause, not a nuisance.
- Standard EC2 monitoring publishes at 5-minute intervals; detailed monitoring gives 1-minute and bills the seven instance metrics at the custom-metric rate (~$0.30/metric/month, so ~$2.10 per instance). Turn it on where a 5-minute blind spot matters, not fleet-wide.
- CloudWatch Logs Insights costs $0.005 per GB scanned. Narrow the time range before the query, or a debugging session becomes a line item.
- Metric filters turn a log pattern into a metric you can alarm on — the cheapest way to alert on an application error string without shipping logs anywhere.

## Deploying Without an Outage

- Health-gated rolling deploys are the baseline: new instances or tasks must pass health checks before the old ones drain.
- Blue/green when rollback must be instant; canary when the failure mode is subtle and needs real traffic to reveal. Both cost more than rolling; pick per service, not per company.
- Database migrations are the part that cannot roll back. Expand-contract: deploy the schema change that is compatible with both versions, deploy the code, then remove the old column in a later release. A migration and a code deploy in the same step means the rollback plan is fiction.
- Record what you deployed — image digest, commit sha, template version — at deploy time, in `~/Clawic/data/aws/deploys/<year>.md`. Rollback is only possible if the previous artifact is identified, and "the previous tag" is not an identity.
- Deploy during hours when the people who can fix it are awake. This is not a technical control and it prevents more incidents than most technical controls.

## Patching and Maintenance

- Managed services have maintenance windows and will apply mandatory updates in them. An unset window means AWS picks one, and it will not be your quiet hour. Set it explicitly on every RDS, ElastiCache, and OpenSearch resource.
- RDS minor-version auto-upgrade is on by default and restarts the instance in the maintenance window. Multi-AZ makes that a failover instead of an outage — another reason it is the production default.
- EC2 patching via SSM Patch Manager with a maintenance window and a patch baseline. The alternative is an instance nobody has logged into for two years, running whatever was current when it was created.
- Immutable infrastructure sidesteps the whole category: rebuild the AMI or the container image, deploy, terminate the old. Patching is then a build-pipeline problem, which is a solved one.

## Production Gate

Before calling something production:

- Multi-AZ for every stateful component; no single instance in the request path
- Automated backups on with a retention that matches the agreed RPO, and a restore that has actually been timed
- Alarms on saturation and errors, with `TreatMissingData` set deliberately, routed to a human who is awake
- Autoscaling configured, with the downstream dependency verified to handle the top of the range
- The first quota the design will hit is named, its current value known, and headroom requested
- Deploy path is health-gated with an identified rollback artifact; database migrations are expand-contract
- Deletion protection and `prevent_destroy` on data resources; the state and template are in version control
- Runbook exists for the top three failure modes, saved to `~/Clawic/data/aws/artifacts/` with its `## Boxes` line in `memory.md`, and the DR drill has been run once with a recorded time
