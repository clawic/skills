# Lambda — Serverless That Behaves Under Load

Lambda's surprises come from three places: memory is CPU, concurrency is a shared account resource, and every event source has its own retry semantics. Hard limits: SKILL.md, Limits That Force Designs.

## Memory Is CPU (the single highest-leverage knob)

Lambda allocates CPU proportionally to memory. **1,769 MB = one full vCPU**; below that you get a fraction, above it you get more than one core (usable only by multi-threaded runtimes).

Consequence: for a CPU-bound function, doubling memory can *reduce* cost, because billing is GB-seconds and the duration halves.

```
cost ∝ memory_GB × duration_s
```

Worked: 512 MB × 4.0s = 2.00 GB-s. Raise to 1,024 MB, duration drops to 1.9s → 1.95 GB-s — slightly cheaper AND twice as fast. Raise to 2,048 MB, duration drops only to 1.8s (no more parallelism to exploit) → 3.6 GB-s, nearly double the cost. The curve has a minimum; find it by measuring three or four sizes on real input rather than reasoning about it.

I/O-bound functions (waiting on a database or an API) get nothing from extra memory — only the CPU-bound part scales.

## Cold Starts

- Anatomy: download/unpack the package → start the runtime → run your init code (everything outside the handler) → invoke the handler. Only the last part happens on a warm invocation.
- During the init phase Lambda gives the function extra CPU regardless of its memory setting. Move SDK client construction, connection pools, and config parsing **outside the handler** and they run in that cheaper window — and then persist across invocations.
- Typical magnitudes: interpreted runtimes (Node, Python) cold-start in the low hundreds of milliseconds; JVM and .NET run into seconds without ahead-of-time work (SnapStart for Java, native AOT for .NET). Package size dominates within a runtime — 250 MB of dependencies is a slower start than 5 MB of them.
- VPC attachment no longer adds the multi-second penalty it did before Hyperplane ENIs. Do not choose architecture around a cold-start cost that was fixed years ago.
- Provisioned concurrency removes cold starts for the provisioned slots and bills while idle. It is a p99-latency purchase for a user-facing endpoint, never a default. SnapStart is the cheaper answer where the runtime supports it.

## Concurrency — Three Different Things

| Term | Meaning | Failure when wrong |
|---|---|---|
| Account concurrency | 1,000 simultaneous executions per region by default, shared by every function | One runaway function starves every other function in the account |
| Reserved concurrency | A ceiling *and* a guarantee carved out of the account pool for one function | Set to 0 and the function is effectively disabled — this is also the emergency stop |
| Provisioned concurrency | Pre-initialized environments, billed hourly | Costs money while idle; does not raise any ceiling |

- Concurrency needed ≈ `requests_per_second × average_duration_seconds`. 200 rps × 0.4s = 80 concurrent. Compare that to what is left of the account pool, not to 1,000.
- Reserve concurrency on the function that must never be starved (the payment webhook) and cap the one that could run away (the S3-triggered thumbnailer). Both directions matter.
- Throttled synchronous invocations return `429 TooManyRequestsException` to the caller. Throttled async invocations are retried by Lambda for up to 6 hours — the work is not lost, it is late, which is worse when the caller already gave up.

## Retries by Event Source (they are all different)

| Source | Retry behavior | What you must do |
|---|---|---|
| Synchronous (API Gateway, ALB, direct invoke) | None — the caller sees the error | Retry in the caller, with backoff |
| Asynchronous (S3, SNS, EventBridge) | 2 retries by default, then the event is discarded | Configure a dead-letter queue or an on-failure destination, or failures are silent |
| SQS | The whole batch returns to the queue after the visibility timeout | Set visibility timeout ≥ 6× function timeout; enable `ReportBatchItemFailures` or one poison message reprocesses the entire batch forever |
| Kinesis / DynamoDB Streams | Retries the batch until it succeeds or the record expires — **blocking the shard** | Set `BisectBatchOnFunctionError` and `MaximumRetryAttempts`, plus a failure destination |
| SQS FIFO | Same as SQS, but a stuck message blocks its message group | Design message groups so one poison message cannot stall a tenant |

The Kinesis case is the one that takes down pipelines: a single unparseable record blocks its shard indefinitely, and the metric that shows it is `IteratorAge`, not error count. Alarm on `IteratorAge`.

## Lambda in a VPC

- A VPC-attached function has **no internet access** unless its subnets route through a NAT gateway. The symptom is a timeout calling a third-party API, and the fix is either NAT or a VPC endpoint — not a security-group change.
- Attach only when the function must reach a private resource (RDS, ElastiCache, an internal service). Attaching "for security" costs you NAT charges and an egress problem in exchange for nothing.
- Each concurrent execution needs an IP in the subnets. Give VPC-attached functions their own generously sized private subnets, not the `/26` someone made for a bastion.

## Database Connections

Lambda's concurrency model and a connection-limited database are natural enemies: 300 concurrent functions each opening a connection will exhaust an RDS instance that allows 112 (Postgres derives it from memory: `LEAST(DBInstanceClassMemory / 9531392, 5000)`, so 112 on a 1 GiB db.t3.micro).

- RDS Proxy is the answer for RDS/Aurora: it pools and reuses connections across invocations, and it holds them through a failover instead of surfacing errors.
- Open the connection outside the handler so it is reused by warm invocations, but never assume a connection survives — always handle reconnection.
- Aurora Serverless v2 Data API or DynamoDB (no connection concept at all) removes the problem instead of managing it.

## Packaging

- Zip: 50 MB compressed on direct upload, 250 MB unzipped including layers. Container image: 10 GB, with a slower cold start.
- Layers are shared dependencies, not a size cheat — they count against the same 250 MB unzipped limit, and a maximum of 5 layers attach to one function.
- `/tmp` is 512 MB by default and configurable up to 10 GB. It persists across warm invocations on the same environment, which is a cache opportunity and a data-leak hazard between tenants.
- Environment variables are visible to anyone with `lambda:GetFunctionConfiguration`. Fetch secrets at init from Parameter Store or Secrets Manager using the function's role; an encrypted-at-rest env var is still plaintext in the console.

## API Gateway in Front

- Integration timeout ceiling is ~29 seconds. Anything longer must be asynchronous: return a job id immediately, do the work in a queue-triggered function, and let the client poll or receive a webhook.
- HTTP API costs ~$1.00 per million requests; REST API ~$3.50 per million. Default to HTTP API; choose REST only for API keys and usage plans, request validation, or per-stage WAF.
- Payload limit is 10 MB. Large uploads go directly to S3 with a presigned URL, never through the gateway.
- Lambda authorizers cache by the identity source for the TTL you set (default 300s). A permission change appears to be ignored until the cache expires — this is the "I updated the policy and nothing happened" report.

## Cost Model

Billed per request plus GB-seconds of duration, at 1 ms granularity. Two consequences worth acting on:

- A function that sleeps waiting on another AWS call is billed the whole time. Step Functions `.sync` integrations and EventBridge callbacks wait for free; a Lambda polling in a loop does not.
- Beyond ~40% duty cycle, an always-on Fargate task is usually cheaper than the equivalent Lambda — a 0.5 GB Lambda running flat out all month ≈ $22 of compute against ≈ $9 for an always-on 0.25 vCPU / 0.5 GB Fargate task. Compute it before scaling a Lambda-based service that never goes idle.

## Observability

- Every invocation's REPORT line carries `Duration`, `Billed Duration`, `Memory Size`, `Max Memory Used`, and `Init Duration` when cold. `Max Memory Used` pinned at the limit means the next OOM is a matter of input size.
- Alarm on `Throttles` and on `Errors` separately — they have different causes and different fixes.
- For async and stream sources, alarm on `DeadLetterErrors` and `IteratorAge`; error count alone stays at zero while events pile up.
- X-Ray with sampling (never 100%) is what separates "the function is slow" from "the database call inside it is slow".
