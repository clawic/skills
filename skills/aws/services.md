# AWS Services — Selection Thresholds

Pick by hard limits and break-evens, not feature lists. Prices: us-east-1, on-demand, checked early 2026 — verify before committing.

## Compute

| Service | Use Case | Pricing Model |
|---------|----------|---------------|
| EC2 | Full OS control, steady load | Per-hour + storage |
| Lambda | Event-driven, spiky | Per-request + GB-seconds |
| ECS Fargate | Containers, no cluster ops | Per vCPU/GB-hour |
| EKS | Kubernetes | ~$73/mo control plane + nodes |
| Lightsail | Fixed-price simple VPS | Flat monthly |

### Lambda vs Fargate: the Duty-Cycle Break-Even
A 0.5 GB Lambda running flat-out all month ≈ $22 compute; a 0.25 vCPU / 0.5 GB Fargate task always-on ≈ $9/mo. Break-even lands around **~40% duty cycle**: busier than that, sustained → Fargate/ECS; spikier → Lambda. Recompute for your memory size — Lambda bills GB-seconds, so the ratio holds across sizes.

Lambda hard limits that force the decision regardless of cost: 15-min max execution, 6 MB synchronous payload, 250 MB unzipped package (10 GB via container images, slower cold starts), and cold-start latency you can't fully eliminate without paying for provisioned concurrency while idle.

EKS threshold: only if the team already runs Kubernetes or needs its ecosystem (operators, Helm charts, multi-cloud portability). The $73/mo control plane is trivial; the operational overhead is not — that's the real cost.

## Database

| Service | Type | Best For |
|---------|------|----------|
| RDS | Relational | Ad-hoc queries, joins, transactions |
| Aurora | Relational (AWS) | Read scaling, faster failover |
| DynamoDB | Key-value | Known access patterns at any scale |
| ElastiCache | In-memory | Sub-ms cache/sessions |
| DocumentDB | Document | MongoDB API compatibility |

### RDS vs DynamoDB — the Real Test
Ask: "can I write down every query pattern today?" Yes, and they're all key lookups → DynamoDB (single-digit ms at any scale, zero idle cost on on-demand mode). No, or the product is still discovering its queries → RDS PostgreSQL. DynamoDB disqualifiers: items >400 KB (hard limit), ad-hoc analytics, anything needing joins. Modeling DynamoDB like a relational table is the most expensive way to discover you needed Postgres.

Aurora note: the standard configuration bills per I/O request — an I/O-heavy workload can cost more than the equivalent RDS despite the "serverless" branding. Check the I/O line in Cost Explorer before and after migrating; I/O-Optimized config trades higher instance price for free I/O.

## Storage

| Service | Type | Cost/GB/month |
|---------|------|---------------|
| S3 Standard | Object | $0.023 |
| S3 IA | Infrequent access | $0.0125 (+retrieval fee, 30-day min) |
| S3 Glacier tiers | Archive | $0.004 → $0.001 (min durations apply) |
| EBS gp3 | Block (one instance) | $0.08 |
| EFS | Shared POSIX filesystem | $0.30 |

Selection: S3 unless you need a filesystem; EBS for a single instance's disk; EFS only when multiple instances must mount the same tree — at ~4× gp3's price, "might share later" doesn't justify it. Archive-tier fine print lives in `costs.md` (transition request costs, minimum durations).

S3 request pricing is the hidden line: PUTs ~$0.005/1,000. Writing millions of tiny objects (per-event logs) makes requests, not storage, the bill — batch into larger objects.

## Networking

| Service | Purpose | Key Cost |
|---------|---------|----------|
| VPC | Isolation | Free (NAT is not — see `costs.md`) |
| ALB | HTTP/HTTPS, WebSocket, gRPC | ~$16/mo min + LCU |
| NLB | Raw TCP/UDP, static IP, extreme scale | Similar hourly + LCU |
| CloudFront | CDN | First 1 TB/mo free, then < EC2 egress |
| Route 53 | DNS | $0.50/zone + queries |
| API Gateway | Managed API front | Per-request |

### API Front Selection
- **HTTP API vs REST API**: HTTP API ~$1.00/M requests vs REST ~$3.50/M. Default to HTTP API; REST API only for API keys/usage plans, request validation, or WAF-per-stage needs.
- **API Gateway vs ALB**: per-request vs per-hour. Low/spiky volume → API Gateway; past tens of millions of requests/month, ALB's flat pricing wins — do the multiplication at your volume, the crossover is well inside startup scale.
- ALB handles WebSocket and gRPC (HTTP/2); NLB when you need static IPs, non-HTTP protocols, or preservation of source IP at L4.

## Security & Identity

| Service | Purpose | Note |
|---------|---------|------|
| IAM | Identities, policies | Free |
| Cognito | User auth | Free tier generous; check MAU pricing past it |
| Secrets Manager | Rotating secrets | $0.40/secret/mo — Parameter Store free otherwise |
| KMS | Encryption keys | AWS-managed free; CMK $1/mo |
| WAF | L7 firewall | Per-rule + per-request |
| GuardDuty | Threat detection | Pay per volume analyzed |

## Messaging & Integration

| Service | Pattern | Deciding Limit |
|---------|---------|----------------|
| SQS | Queue (1 consumer group) | 256 KB messages — larger payloads go to S3 with a pointer in the message |
| SNS | Pub/sub fan-out | Push, no replay |
| EventBridge | Event bus + routing rules | Higher latency than SNS; schema/filtering power |
| Kinesis | Ordered streams, replay | Sharding to manage |
| Step Functions | Workflow orchestration | Per state transition — chatty loops get expensive |

Selection: one consumer → SQS. Many consumers of the same event → SNS→SQS fan-out (each consumer gets its own queue and retry semantics — subscribing consumers directly to SNS loses buffering). Cross-service routing with filtering → EventBridge. Need replay/ordering → Kinesis. SQS FIFO caps at 300 msg/s per API action (3,000 with batching) — hitting that ceiling means redesigning around message groups or dropping strict ordering.

## Observability

| Service | Purpose | Cost Watch |
|---------|---------|------------|
| CloudWatch Logs | Aggregation | Ingestion $0.50/GB — the trap (`costs.md`) |
| CloudWatch Metrics | Monitoring | Custom metrics $0.30/metric/mo — high-cardinality dimensions multiply this |
| CloudWatch Alarms | Alerting | ~$0.10/alarm/mo |
| X-Ray | Tracing | Sample, don't trace 100% |
| CloudTrail | API audit | First management-event trail free |

## Cost Tiers (Typical)

| Tier | Services | Monthly |
|------|----------|---------|
| Free tier | Lambda, DynamoDB, S3 (within limits) | $0 |
| Minimal | t3.micro + RDS + S3 | $30-50 |
| Startup | t3.small + RDS + ALB + S3 | $100-200 |
| Growth | ASG + RDS Multi-AZ + ElastiCache | $300-500 |
| Scale | EKS + Aurora + CloudFront | $1000+ |
