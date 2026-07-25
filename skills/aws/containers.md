# Containers — ECS, Fargate, EKS, ECR

Choosing: **ECS Fargate** unless the team already runs Kubernetes or needs its ecosystem, in which case **EKS**. ECS on EC2 only when you need GPU/specialised instances, per-instance cost control at scale, or daemon workloads. Image building and container debugging itself: the `docker` skill; Kubernetes manifests and cluster internals: the `k8s` skill.

## The Two Roles Nobody Distinguishes

ECS tasks have two separate IAM roles, and mixing them up produces errors that look like network failures.

| Role | Used by | Needs permission to |
|---|---|---|
| **Task execution role** | The ECS agent, before your container starts | Pull from ECR, write to the log group, read secrets from Secrets Manager/SSM and decrypt them with KMS |
| **Task role** | Your application code, at runtime | Everything the app calls: S3, DynamoDB, SQS |

`ResourceInitializationError: unable to pull secrets or registry auth` is always the *execution* role. `AccessDenied` from your own code is always the *task* role. Granting the app's permissions to the execution role does nothing for the app.

## Why a Task Will Not Start

Read `stoppedReason` first — `aws ecs describe-tasks --cluster c --tasks <arn> --query 'tasks[].stoppedReason'`.

| Reason | Cause | Fix |
|---|---|---|
| `CannotPullContainerError` | No route to ECR | Private subnet needs NAT, or the trio `ecr.api` + `ecr.dkr` + the **S3 gateway endpoint** (layers live in S3) |
| `ResourceInitializationError ... secrets` | Execution role or KMS | Grant the execution role `secretsmanager:GetSecretValue` / `ssm:GetParameters` plus `kms:Decrypt` |
| `Task failed ELB health checks` | App boots slower than the health check tolerates | Raise `healthCheckGracePeriodSeconds` above worst-case boot; ALB defaults mark a target healthy only after ~150s |
| Stuck in PROVISIONING | No free IP in the subnet, or no ENI capacity on the instance | awsvpc mode consumes one IP per task — check subnet free addresses |
| Stuck in PENDING on EC2 launch type | No instance with enough CPU/memory/ports | Capacity provider with managed scaling, or a bigger instance type |
| `OutOfMemoryError: Container killed` | Container exceeded its hard memory limit | Raise `memory`, or set `memoryReservation` (soft) and `memory` (hard) as a pair |
| Exits 0 immediately | The container's command finished — a task is not a service | Long-running process in the foreground, or use a scheduled task if it is a job |

## Fargate Sizing Is a Menu, Not a Dial

CPU and memory come in fixed valid combinations: 0.25 vCPU allows 0.5/1/2 GB; 1 vCPU allows 2-8 GB; 4 vCPU allows 8-30 GB. An invalid pair is rejected at task-definition registration, not at run time.

- Rough cost: $0.04048 per vCPU-hour and $0.004445 per GB-hour (us-east-1, Linux/x86). A 0.25 vCPU / 0.5 GB task running always ≈ **$9/month** — the number behind the ~40% duty-cycle break-even in SKILL.md, Service Defaults.
- Fargate Spot is ~70% cheaper with a 2-minute interruption notice. Correct for stateless workers and batch behind a queue; never for the only replica of anything.
- Graviton (`ARM64` runtime platform) is cheaper per vCPU-hour, but the image must be built for arm64 — a multi-arch build, or `exec format error` in production.
- Ephemeral storage defaults to 20 GB and configures up to 200 GB. There is no instance store to fall back on.

## Deployments and Draining

- `minimumHealthyPercent` / `maximumPercent` default to 100/200: a rolling deploy briefly runs double the tasks, so cluster capacity and any per-task IP must cover 2× the desired count.
- With a desired count of 1 and `minimumHealthyPercent: 100`, the deploy needs room for a second task or it deadlocks. Single-task services in tight subnets hang here.
- Target-group deregistration delay defaults to 300s. The deploy looks stuck for five minutes after the new tasks are already serving — lower it to just above your longest request, not to zero.
- `stopTimeout` defaults to 30s: SIGTERM, then SIGKILL. An app that ignores SIGTERM drops in-flight requests on every deploy. Handle the signal and finish or shed work inside the window.
- Circuit breaker (`deploymentCircuitBreaker` with rollback) turns a bad deploy into an automatic revert instead of a service of crash-looping tasks. Enable it; it costs nothing.
- Blue/green via CodeDeploy gives a test listener and a bake period. Worth it when a rollback needs to be instant rather than another rolling deploy.

## Service Discovery and Load Balancing

- ECS Service Connect or Cloud Map gives service-to-service DNS inside the cluster without a load balancer per service. Service Connect also gives you per-service traffic metrics for free.
- One target group per service; the container port in the task definition must match the target-group port, and in awsvpc mode the target type must be `ip`, not `instance`. A mismatch registers targets that never pass health checks.
- Health checks exist at two levels: the container health check (in the task definition, decides task health) and the target-group health check (decides traffic). They can disagree; the target group is what the user experiences.

## ECR

- Immutable tags (`imageTagMutability: IMMUTABLE`) make a deployed tag mean exactly one image forever. Without it, `:latest` and even `:v1.2.3` can be overwritten, and rollback becomes archaeology.
- Deploy by digest (`repo@sha256:...`) or by an immutable tag. Promotion between environments retags the same digest — a promotion that rebuilds means staging validated a different artifact.
- Lifecycle policies are the only thing standing between you and a repository that grows forever: expire untagged images after a few days, and cap the count of tagged ones per prefix.
- Scan on push (basic) is free; enhanced scanning via Inspector is continuous and billed. CI-only scanning approves images that rot in place — the registry is where CVEs published after the build show up.
- Cross-region or cross-account pulls need a repository policy plus, for pull-through caches, the upstream credentials in Secrets Manager. Anonymous Docker Hub pulls from a busy CI farm hit rate limits; a pull-through cache in ECR fixes that class permanently.

## EKS, Briefly

The cluster control plane costs ~$0.10/hour (~$73/month) per cluster; that is not the real cost. The real cost is that you now own upgrades, add-ons, and networking.

- **Pod identity**: give pods their own IAM role via EKS Pod Identity or IRSA. Attaching permissions to the node role gives every pod on the node those permissions — the most common EKS security finding.
- **VPC CNI IP exhaustion**: each pod gets a real VPC IP, and each instance type caps how many ENIs and IPs it can hold. A `/24` subnet and a busy node group run out of addresses; prefix delegation or a secondary CIDR is the fix.
- **Upgrades are on a clock**: a Kubernetes minor version leaves standard support after roughly a year, then bills at a higher extended-support rate. Plan an upgrade cadence at cluster creation, not when the invoice changes.
- **Add-on drift**: VPC CNI, CoreDNS, and kube-proxy versions must track the control-plane version. Managed add-ons handle this; self-installed ones become the reason an upgrade fails.
- Fargate profiles for EKS remove node management for specific namespaces but do not support DaemonSets, privileged containers, or GPUs — check the exclusions before designing around them.

## Cost Control for Container Platforms

| Lever | Effect |
|---|---|
| Right-size task CPU/memory from actual utilization | Fargate bills the requested size, not the used size — over-requesting is 100% waste |
| Fargate Spot for interruptible workers | ~70% off, 2-minute notice |
| Graviton where the image supports arm64 | Lower per-vCPU-hour price for the same task shape |
| Scale to zero off-hours (dev/staging) | A scheduled scaling action costs nothing and halves non-prod spend |
| Compute Savings Plans | Cover Fargate as well as EC2 and Lambda — commit only after right-sizing |
| ECR lifecycle policies | Stops the storage line item nobody notices until it is large |
