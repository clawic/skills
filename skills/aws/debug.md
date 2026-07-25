# Debugging AWS — Symptom to Cause

AWS failures are opaque because the error is emitted by the wrong layer: a permissions problem arrives as a timeout, a routing problem arrives as `AccessDenied`, a quota problem arrives as an intermittent 500. Work symptom-first; every step below is a check, not a guess.

## The Universal First Three

1. **Who am I and where am I?** `aws sts get-caller-identity` plus the region you are actually targeting. Half of "it works in the console but not the CLI" is a different principal or a different region.
2. **What did the API actually see?** CloudTrail is the ground truth for every control-plane call — it records the principal, the parameters, and the exact `errorCode`/`errorMessage`. Look up the call by event name before theorizing.
3. **Is it me or is it the service?** Check the Health Dashboard for the region before spending an hour on a service that is degraded. Rare, but it is the one cause you cannot fix.

## AccessDenied

Do NOT widen the policy first — that is how accounts end up with `"Action": "*"`.

1. Read the message: it names the principal, the action, and usually the resource. `is not authorized to perform` with a role ARN means identity policy or SCP; `Access Denied` with no principal named usually means a *resource* policy (S3 bucket policy, KMS key policy, ECR policy).
2. Distinguish the five gates in the order the evaluation applies them: explicit `Deny` anywhere → SCP ceiling → resource policy → permissions boundary (the effective permission is the intersection) → identity or session policy. The failing gate is never named in the message.
3. Cross-account: the permission must exist on BOTH sides (identity policy in the calling account, resource policy or trust policy in the target account). One side alone is always a denial.
4. Encrypted resource: an S3 object encrypted with a customer-managed key needs `kms:Decrypt` on the key, in the key policy AND the identity policy. The error names S3, the fix is in KMS.
5. Just-created role or policy: IAM is eventually consistent. A denial within seconds of creation that resolves on retry was propagation, not policy — build retry with backoff into automation that creates and then uses a role.

## Connection Timed Out

A timeout is a network fact. Credentials never produce a timeout — they produce `AccessDenied`, `403`, or `password authentication failed`.

Walk the path in this order, stopping at the first failure:

| Step | Check |
|---|---|
| 1. Security group (inbound) | Does the target's SG allow the source SG or CIDR on that port? |
| 2. Security group (outbound) | Does the *source's* SG allow egress to the target? Restricted egress breaks this quietly |
| 3. Route table | Does the source subnet have a route to the target — IGW, NAT, peering, or transit gateway? |
| 4. NACL | Stateless: an allow inbound needs a matching allow outbound on ephemeral ports 1024-65535 |
| 5. Subnet type | Is the target in a private subnet with no path from where you are calling? |
| 6. DNS | Does the name resolve, and to a private or public address? A public address for an RDS endpoint from inside the VPC means DNS hostnames/resolution are off |

Skip the guessing with VPC Reachability Analyzer — `aws ec2 create-network-insights-path` then `start-network-insights-analysis` names the blocking component without sending a packet.

## Load Balancer Status Codes

The code names the layer. Decode before touching configuration.

| Code | Meaning | Cause to check first |
|---|---|---|
| 502 | Bad gateway: the target closed the connection or answered malformed | Keep-alive race — the app's idle timeout is at or below the ALB's 60s idle timeout. Set the app higher (65-75s) |
| 503 | No healthy targets | Nothing registered, or all failing health checks. With ALB defaults (30s interval, 5 healthy threshold), a new target takes ~150s to be marked healthy — a deploy can 503 legitimately for that window |
| 504 | Gateway timeout: the target did not answer within the idle timeout | The slow path is real; raising the timeout hides it. Trace the request first |
| 400 with no target log | Malformed request rejected by the ALB itself | Header size, invalid characters in the path, HTTP/1.0 without a Host header |
| 460 / 463 | Client disconnected before the target answered / bad `X-Forwarded-For` | Client-side and proxy-header problems, not your app |

The two log sources that end these arguments: ALB access logs (per request, includes `target_status_code` and processing times) and target-group health-check status. If `elb_status_code` is 502 and `target_status_code` is `-`, the target never answered.

## Throttling and Rate Limits

- `ThrottlingException`, `Rate exceeded`, `RequestLimitExceeded`, `TooManyRequestsException`, `ProvisionedThroughputExceededException` are all the same shape: you exceeded a *rate*, not a capacity.
- The fix order is always: (1) exponential backoff with jitter in the client, (2) reduce call frequency — cache describes, batch, stop polling, (3) request a quota increase. Skipping to (3) hides a polling loop that will hit the new ceiling too.
- Control-plane calls (`describe*`, `list*`) throttle far more aggressively than data-plane calls. A dashboard that calls `describe-instances` every 5 seconds throttles the whole account's automation, including deploys.
- DynamoDB throttling with capacity to spare = hot partition, not a quota: the per-partition ceilings are 3,000 RCU and 1,000 WCU no matter what the table is provisioned for.

## The Deploy That Broke It

When something worked yesterday, find the change before debugging the symptom.

1. CloudTrail filtered to write events in the window: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=<Api>` — someone changed something, and CloudTrail names them.
2. IaC drift check — `terraform plan -detailed-exitcode` (2 = changes pending) or `aws cloudformation detect-stack-drift`. If the plan is dirty, the drift IS the change.
3. Config timeline: AWS Config records resource configuration history; a diff of a security group or a policy across the incident window is faster than reading either version.
4. Cost anomaly start date: a bill delta and an outage often share a root cause. The anomaly's start date is a timestamp for free.

## Lambda Failed, But Where

| Observation | Meaning |
|---|---|
| No log group at all | The function has never been invoked, or its role lacks `logs:CreateLogGroup` — check the execution role before the code |
| `Task timed out after N seconds` | Function timeout; if N is exactly your API Gateway ceiling instead, the gateway gave up first |
| `Runtime exited with error: signal: killed` | Out of memory — Lambda kills at the configured memory ceiling |
| Duration near the timeout only sometimes | Cold start plus a slow dependency; separate init duration from invoke duration in the REPORT line |
| Invoked but no effect, no error | Async invocation that failed and retried twice into nothing — configure a dead-letter queue or on-failure destination |

Retry semantics differ per source and explain most "it ran but nothing happened": async invocations retry twice then discard, SQS returns the whole batch after the visibility timeout, and Kinesis retries block the shard until the record expires — watch `IteratorAge`, not error count.

## Container Task Never Runs

1. `aws ecs describe-tasks` → read `stoppedReason` verbatim before anything else; it usually names the cause outright and the strings below are the common ones.
2. `CannotPullContainerError` → the task cannot reach ECR: private subnet with no NAT, or missing `ecr.api`, `ecr.dkr`, and S3 gateway endpoints. The S3 endpoint is required because image layers live in S3.
3. `ResourceInitializationError ... unable to pull secrets` → the *task execution* role lacks access to the secret or the KMS key. Two roles exist and they are not interchangeable: the execution role pulls images and reads secrets before start, the task role is what your code uses.
4. Task starts then dies within a minute → application crash; the logs are in the log group named by the task definition's `awslogs` configuration, not in ECS.
5. Task healthy in ECS, unhealthy in the target group → two different health checks. The ELB health check needs a grace period longer than worst-case boot.

## "It Works in the Console but Not from Code"

| Difference | Check |
|---|---|
| Different principal | Console is your SSO user; code is a role. Compare `sts get-caller-identity` from both |
| Different region | The console remembers a region per service; the SDK reads `AWS_REGION` or the profile |
| Different endpoint | VPC endpoint policies can allow the console path and deny the in-VPC path |
| Encryption context | Console operations sometimes supply KMS encryption context your code omits |
| MFA condition | A policy with `aws:MultiFactorAuthPresent` passes in the console session and fails for a machine role |

## When You Are Truly Stuck

Cut the problem in half with a minimal reproduction from inside the same network position: launch a throwaway instance or task in the *same subnet with the same role*, and run the single failing call. If it works there, the difference is your application's configuration; if it fails there, the difference is IAM or the network — and you have just halved the search space.
