# Service Selection — Thresholds and Break-Evens

Pick by hard limit and break-even, never by feature list. Prices: us-east-1, on-demand, recorded early 2026 — the ratios hold, verify the absolute numbers. Stage-to-stack costs live in SKILL.md Rule 3; the per-service hard limits that override every preference are in SKILL.md, Limits That Force Designs.

**Contents:** [Compute](#compute) · [Database](#database) · [Storage](#storage) · [Networking and Edge](#networking-and-edge) · [Messaging and Integration](#messaging-and-integration) · [Data and Analytics](#data-and-analytics) · [Security and Identity](#security-and-identity) · [Observability](#observability) · [Email, Notification, and the Sandbox Trap](#email-notification-and-the-sandbox-trap) · [Region Selection](#region-selection)

## Compute

| Service | Use case | Pricing model |
|---------|----------|---------------|
| Lambda | Event-driven, spiky, short | Per request + GB-seconds |
| ECS Fargate | Containers, no cluster ops | Per vCPU-hour + GB-hour |
| ECS on EC2 | Containers where you need GPU, specialised instances, or per-node cost control | Instance hours |
| EKS | Kubernetes ecosystem needed | ~$73/mo control plane + nodes |
| EC2 | Full OS control, steady load, legacy | Per hour + storage |
| Lightsail | Fixed-price simple VPS | Flat monthly |
| Batch | Long-running queued jobs, Spot-friendly | Underlying compute only |
| App Runner | A container URL with no infrastructure decisions | Per vCPU/GB with a scale-to-zero option |

### Lambda vs Fargate: the Duty-Cycle Break-Even

A 0.5 GB Lambda running flat out all month ≈ **$22** of compute. A 0.25 vCPU / 0.5 GB Fargate task always on ≈ **$9/mo**. Break-even lands near **40% duty cycle**: busier and sustained → Fargate/ECS; spikier → Lambda. Recompute for your memory size — Lambda bills GB-seconds, so the ratio holds across sizes.

Limits that force the decision regardless of cost: 15-minute maximum execution, 6 MB synchronous payload, 250 MB unzipped package, and a cold start you cannot eliminate without paying for provisioned concurrency while idle.

### EKS Threshold

Only if the team already runs Kubernetes or needs its ecosystem (operators, Helm, portability). The $73/mo control plane is trivial; the upgrade cadence, add-on drift, and IP planning are the real cost.

## Database

| Service | Type | Best for |
|---------|------|----------|
| RDS | Relational | Ad-hoc queries, joins, transactions |
| Aurora | Relational, AWS-native | Read scaling, fast failover, storage that grows itself |
| DynamoDB | Key-value | Known access patterns at any scale, zero idle cost |
| ElastiCache | In-memory | Sub-millisecond cache and sessions |
| DocumentDB | Document | MongoDB API compatibility |
| Neptune | Graph | Relationship traversal that SQL makes painful |
| Timestream | Time series | High-cardinality metrics with automatic tiering |
| Redshift | Columnar warehouse | Analytics over hundreds of GB and up |

### RDS vs DynamoDB — the Real Test

Ask: "can I write down every query pattern today?" Yes, and they are all key lookups → DynamoDB. No, or the product is still discovering its queries → RDS PostgreSQL. DynamoDB disqualifiers: items over 400 KB, ad-hoc analytics, anything needing a join. Key modeling for DynamoDB tables: the `dynamodb` skill.

Aurora note: the standard configuration bills per I/O request, so an I/O-heavy workload can cost more than the equivalent RDS. I/O-Optimized trades a higher instance price for free I/O and wins when I/O exceeds roughly a quarter of the Aurora bill.

## Storage

| Service | Type | Cost/GB/month |
|---------|------|---------------|
| S3 Standard | Object | $0.023 |
| S3 Standard-IA | Infrequent access | $0.0125 + retrieval fee, 30-day minimum |
| S3 Glacier tiers | Archive | $0.004 → $0.001, minimum durations apply |
| EBS gp3 | Block, one instance | $0.08 |
| EFS | Shared POSIX filesystem | $0.30 |
| FSx | Windows/Lustre/NetApp filesystems | Per-family |

S3 unless you need a filesystem; EBS for a single instance's disk; EFS only when several instances must mount the same tree, at nearly 4× gp3's price. The two numbers that decide S3 designs: requests cost more than bytes (PUT/LIST ≈ $0.005 per 1,000 against $0.023/GB-mo of storage), and throughput caps at 5,500 GET / 3,500 PUT per second **per prefix**.

## Networking and Edge

| Service | Purpose | Cost driver |
|---------|---------|-------------|
| VPC | Isolation | Free — NAT is not: ~$33/mo + $0.045/GB |
| ALB | HTTP/HTTPS, WebSocket, gRPC, path routing | ~$16/mo minimum + LCU |
| NLB | Raw TCP/UDP, static IPs, source-IP preservation | Similar hourly + LCU |
| CloudFront | CDN and public entry point | First 1 TB/mo egress free |
| Route 53 | DNS, health checks, failover routing | $0.50/hosted zone/mo + queries |
| API Gateway | Managed API front | Per request |
| Global Accelerator | Anycast IPs, non-HTTP global routing | Fixed hourly + data premium |

### API Front Selection

- **HTTP API vs REST API**: ~$1.00/M requests vs ~$3.50/M. Default to HTTP API; choose REST only for API keys and usage plans, request validation, or per-stage WAF.
- **API Gateway vs ALB**: per request vs per hour. Low or spiky volume → API Gateway. Past tens of millions of requests a month, ALB's flat pricing wins — do the multiplication at your volume, the crossover is well inside startup scale.
- **CloudFront in front of either** turns egress from $0.09/GB into the free tier and adds caching, TLS, and WAF attachment. For a public API it is usually the cheapest change available.
- ALB handles WebSocket and gRPC; NLB when you need static IPs, non-HTTP protocols, or L4 source-IP preservation.

## Messaging and Integration

| Service | Pattern | Deciding limit |
|---------|---------|----------------|
| SQS | Queue, one consumer group | 256 KB message — larger payloads go to S3 with a pointer |
| SNS | Pub/sub fan-out | Push only, no replay |
| EventBridge | Event bus with routing rules and schemas | Higher latency than SNS; far more filtering power |
| Kinesis Data Streams | Ordered, replayable streams | Shard management; per-shard throughput ceiling |
| Data Firehose | Buffered delivery to S3/Redshift/OpenSearch | Delivery latency measured in buffer intervals |
| Step Functions | Workflow orchestration | Billed per state transition — chatty loops get expensive |
| MQ | Managed ActiveMQ/RabbitMQ | Only when a protocol (JMS, AMQP) is a hard requirement |

Selection: one consumer → SQS. Many consumers of the same event → SNS→SQS fan-out, giving each consumer its own queue and retry semantics (subscribing consumers directly to SNS loses buffering, and one slow consumer then drops messages). Cross-service routing with content filtering → EventBridge. Replay or strict ordering → Kinesis. SQS FIFO caps at 300 messages/second per API action (3,000 batched); hitting that means redesigning around message groups or dropping strict ordering.

Step Functions Express workflows cost per GB-second rather than per transition — correct for high-volume short workflows, while Standard is correct for long-running ones that need durable history.

## Data and Analytics

| Service | Purpose | Cost driver |
|---------|---------|-------------|
| Athena | SQL over S3, no infrastructure | $5/TB scanned — partition and use Parquet |
| Glue | Catalog + ETL jobs | Per DPU-hour; the catalog itself is nearly free |
| Redshift | Warehouse for repeated heavy queries | Node hours, or per-RPU on Serverless |
| EMR | Spark/Hadoop at scale | Instance hours; Spot-friendly for transient clusters |
| OpenSearch | Search and log analytics | Node hours + storage |
| QuickSight | BI dashboards | Per user or per session |

The threshold that matters: **Athena until query cost or latency hurts, then Redshift.** Athena has no idle cost and no cluster, so it wins for exploration and periodic reporting. A dashboard that runs the same scan hundreds of times a day is paying per run for work a warehouse does once — that is the migration signal, not data volume.

Partitioning plus columnar format is the highest-leverage change in this whole section: the same query over Parquet partitioned by date routinely scans an order of magnitude less than raw JSON.

## Security and Identity

| Service | Purpose | Note |
|---------|---------|------|
| IAM | Identities and policies | Free |
| Identity Center | Workforce SSO across accounts | Free |
| Cognito | Application user auth | Generous free tier; check MAU pricing past it |
| Secrets Manager | Rotating secrets | $0.40/secret/mo — Parameter Store is free otherwise |
| KMS | Encryption keys | AWS-managed free; customer-managed $1/mo |
| WAF | L7 firewall | Per rule + per request |
| GuardDuty | Threat detection | Per volume analyzed |

## Observability

| Service | Purpose | Cost watch |
|---------|---------|------------|
| CloudWatch Logs | Aggregation | $0.50/GB ingested — 16× the $0.03/GB-mo storage price |
| CloudWatch Metrics | Monitoring | Custom metrics $0.30/metric/mo; high-cardinality dimensions multiply it |
| CloudWatch Alarms | Alerting | ~$0.10/alarm/mo |
| X-Ray | Distributed tracing | Sample; never trace 100% |
| CloudTrail | API audit | First management-event trail free; data events billed |
| Managed Grafana / Prometheus | Dashboards and metrics at scale | Per active user / per metric ingested |

## Email, Notification, and the Sandbox Trap

- SES starts every account in a **sandbox**: you can only send to verified addresses, with a low daily cap, until you request production access. This is discovered at launch, not before it — request the move out of the sandbox days ahead, since approval is manual.
- SNS sends SMS and mobile push; SES sends email. Using SNS for transactional email is a common wrong turn — no templates, no reputation management, no bounce handling.
- Bounce and complaint rates are monitored by AWS and a bad reputation gets sending paused. Wire the bounce and complaint SNS topics into something that suppresses addresses automatically, from day one.

## Region Selection

- us-east-1 is usually cheapest and has the widest service availability; it also has the largest correlated failure history and hosts global control planes (IAM, CloudFront, Organizations, billing), which is why an outage there feels global.
- Latency to your users beats price for anything interactive: pick the region nearest the users, or put CloudFront in front and keep the origin wherever it is cheapest.
- Data residency requirements are not negotiable and are the first filter, before price or latency.
- New services and instance families reach us-east-1 first. Building on something brand-new in a distant region means waiting for it there.
