---
name: Kubernetes
slug: k8s
version: 1.0.3
description: >-
  Debugs Kubernetes workloads and reviews manifests: pods, probes, resources, rollouts, Services, storage, RBAC.
  Use when a pod is Pending, CrashLoopBackOff, ImagePullBackOff, OOMKilled or stuck Terminating, when a Service
  or Ingress serves nothing or returns 502/503/504, when cluster DNS is flaky, a rollout hangs or silently ships
  a broken version, an HPA refuses to scale, a PVC stays unbound, a node goes NotReady or a drain never finishes,
  when writing or reviewing YAML, Helm charts or kustomize overlays, when tuning requests, limits, QoS, probes
  and graceful shutdown, or when locking down RBAC, NetworkPolicy, Pod Security and Secrets. Covers kubectl
  triage, StatefulSets and PVCs, Jobs and CronJobs, autoscaling, admission webhooks, node drains and cluster
  upgrades. Not for building container images — that is `docker`.
homepage: https://clawic.com/skills/k8s
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
metadata:
  clawdbot:
    emoji: "☸"
    displayName: Kubernetes
    requires:
      bins:
      - kubectl
    os:
    - linux
    - darwin
    - win32
    configPaths:
    - ~/Clawic/data/k8s/
---

User preferences and memory live in `~/Clawic/data/k8s/` (see `setup.md` on first use, `memory-template.md` for the file format). If you have data at an old location (`~/k8s/` or `~/clawic/k8s/`), move it to `~/Clawic/data/k8s/`.

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/k8s/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| cluster_flavor | eks \| gke \| aks \| openshift \| k3s \| kind \| generic | generic | Selects LoadBalancer, StorageClass, node-pool and Ingress defaults; gates advice that only exists on managed control planes (`nodes.md`, `storage.md`) |
| manifest_tool | plain \| kustomize \| helm | plain | Shape of every manifest emitted, plus the rollback and drift procedure in `manifests.md` |
| label_scheme | text (label-key prefix) | app.kubernetes.io/* | Label keys written into every manifest and into the Deployment selector (`manifests.md`); the Output Gates label check reads this value |
| cpu_limits_policy | none \| equal-requests \| explicit | none | Whether generated manifests carry CPU limits; resolves the standing disagreement below and the throttling advice in `resources.md` |
| psa_level | privileged \| baseline \| restricted | baseline | securityContext block written into every pod template and which findings `security.md` raises |
| ingress_controller | nginx \| traefik \| haproxy \| istio \| alb \| gateway-api \| none | nginx | Annotation syntax, timeout and body-size knobs, path-matching semantics in `ingress.md` |
| apply_gate | server-dry-run \| diff \| none | server-dry-run | Verification step required before any apply (`manifests.md`); `none` suppresses the reminder |
| destructive_confirm | bool | true | Force-delete, PVC delete, `drain --force`, and namespace delete are proposed with the blast radius spelled out, never run implicitly |
| explain_depth | brief \| normal \| deep | normal | Length of every answer and whether a diagnosis is walked step by step: `brief` = command plus verdict, `normal` = verdict plus the evidence that settled it, `deep` = the full chain including the branches ruled out |

Preference areas to record as the user reveals them:

- **tooling** — kubectl plugins (`kubectx`, `stern`, `kubectl-neat`), Helm vs kustomize vs raw YAML, GitOps controller (Argo CD, Flux)
- **conventions** — annotation scheme beyond `label_scheme`, namespace-per-team vs per-env, image tagging, resource naming
- **platform** — cluster count and sizing, node classes (spot vs on-demand), CNI, CSI driver, multi-AZ posture
- **safety posture** — how proactively to raise hardening, PDB, and cost findings vs only on request
- **observability** — metrics stack (Prometheus, Datadog, cloud-native), log pipeline, whether `kubectl top` even works
- **secrets management** — Sealed Secrets, External Secrets, Vault, cloud secret manager (the choice, never the credentials)
- **cadence** — upgrade window, drain policy, when maintenance is allowed to touch running workloads

## When To Use

- A pod is Pending, CrashLoopBackOff, OOMKilled, ImagePullBackOff, Evicted, or stuck Terminating
- A Service, Ingress, or Gateway receives no traffic, or cluster DNS is intermittently slow
- Writing or reviewing manifests, Helm charts, or kustomize overlays: probes, resources, rollout strategy, security
- A rollout is stuck, flapping, or silently serving a broken version; a rollback needs to be safe
- Sizing requests, limits, QoS, HPA, or a cluster autoscaler that will not scale in or out
- Locking down RBAC, ServiceAccounts, NetworkPolicy, Pod Security Admission, or Secrets handling
- Node-level and cluster-level work: drains, pressure eviction, upgrades, etcd backup
- Not for building container images themselves — that is `docker`

## Quick Reference

| Symptom | First move |
|---|---|
| Pod Pending | `kubectl describe pod` Events name the predicate: insufficient CPU/memory (requests vs node allocatable), unbound PVC, taint without toleration, no nodeSelector match → `scheduling.md` |
| CrashLoopBackOff | `kubectl logs -p` (previous container). Exit 137 = SIGKILL (OOM or liveness kill), 143 = SIGTERM, 1 = app error. Backoff starts at 10s and doubles to a 5m cap, resetting after 10m of clean running → `debug.md` |
| OOMKilled | `describe pod` → Last State: OOMKilled. Raise the limit or fix the leak; JVM, Node, and Go runtimes each need their own cap below it → `resources.md` |
| ImagePullBackOff | The Events line carries the real reason: 401 (missing or wrong `imagePullSecret`), 404 (tag typo, wrong registry), or a registry rate limit → `debug.md` |
| Service gets no traffic | `kubectl get endpointslices -l kubernetes.io/service-name=<svc>` — empty means selector/label mismatch or zero ready pods; then check `port` vs `targetPort` → `networking.md` |
| Ingress returns 502/503/504 | 503 = no endpoints behind the Service; 502 = backend closed or spoke the wrong protocol; 504 = backend slower than the controller's read timeout → `ingress.md` |
| Intermittent DNS, ~5s stalls | `ndots:5` search expansion plus UDP conntrack races. FQDN with a trailing dot (`db.prod.svc.cluster.local.`) or `dnsConfig` `ndots:2`; NodeLocal DNSCache at cluster level → `dns.md` |
| Rollout stuck | `kubectl rollout status`, then `describe rs` on the newest ReplicaSet for quota, image, or admission errors. `progressDeadlineSeconds` (default 600s) only flips Progressing=False — it never auto-rolls back → `rollouts.md` |
| Pod stuck Terminating | `metadata.finalizers` plus `deletionTimestamp`: fix the controller that owns the finalizer; force-delete is last resort → `operators.md` |
| Works in namespace A, fails in B | Diff ResourceQuota, LimitRange, NetworkPolicy, and PSA labels — a quota namespace rejects pods with no requests set → `resources.md`, `security.md` |
| HPA shows `<unknown>` | A container is missing resource requests (utilization = usage/requests has no denominator), or metrics-server is down → `autoscaling.md` |
| PVC stuck Pending | No default StorageClass, `volumeBindingMode: WaitForFirstConsumer` waiting on a schedulable pod, or a zone mismatch between disk and node → `storage.md` |
| StatefulSet pod-0 won't start | Ordered rollout blocks on the previous ordinal; a retained PVC may hold stale data → `stateful.md` |
| Node NotReady, or a drain never ends | Node conditions (MemoryPressure/DiskPressure/PIDPressure), then the kubelet; drains block on PDBs, bare pods, and local storage → `nodes.md` |
| CronJob stopped firing | >100 missed schedules with no `startingDeadlineSeconds` disables it permanently → `jobs.md` |
| An apply silently reverts | Two writers on one object (GitOps vs hotfix) or a field-manager conflict → `manifests.md` |
| Every create suddenly fails | An admission webhook with `failurePolicy: Fail` whose backend is down → `operators.md` |
| Works locally, fails in the cluster (or the reverse) | Local clusters have no cloud LB, no real StorageClass, one node, and your credentials → `local-dev.md` |
| About to run a destructive command | `kubectl config current-context` first; `destructive_confirm` governs the rest → `local-dev.md` |
| Anything else | `kubectl get events -n <ns> --sort-by=.lastTimestamp`, then `describe` the object it names |

Depth on demand: `debug.md` symptom→cause chains · `commands.md` kubectl incident toolkit · `scheduling.md` placement, taints, affinity, preemption · `resources.md` requests, limits, QoS, throttling, quotas · `probes.md` liveness, readiness, startup, lifecycle hooks · `rollouts.md` deploy strategies, PDB, rollback · `networking.md` Services, EndpointSlices, kube-proxy, NetworkPolicy wiring · `dns.md` CoreDNS, ndots, conntrack · `ingress.md` Ingress, Gateway API, TLS · `storage.md` PV, PVC, StorageClass, snapshots · `stateful.md` StatefulSets and databases · `config-and-secrets.md` ConfigMaps, Secrets, external stores · `rbac.md` permissions and escalation paths · `security.md` Pod Security, securityContext, supply chain · `jobs.md` Jobs and CronJobs · `autoscaling.md` HPA, VPA, KEDA, cluster autoscaler · `nodes.md` node lifecycle, drains, pressure · `manifests.md` apply semantics, kustomize, Helm, GitOps · `operators.md` CRDs, controllers, finalizers, webhooks · `local-dev.md` kind/minikube/k3d and kubeconfig context safety · `production.md` readiness gate, upgrades, DR, cost.

## Core Rules

1. **Memory requests = limits on every production container.** Overcommitted memory means the node OOM-kills someone when neighbors burst — and the victim is chosen by QoS class, not by who leaked. CPU: always set requests; limits follow `cpu_limits_policy` (→ Where Experts Disagree).
2. **Never guess `initialDelaySeconds` — give slow starters a startupProbe.** Boot budget = `failureThreshold × periodSeconds`: 30 × 10s = 300s covers a slow JVM while liveness stays tight for steady state.
3. **Detection latency = `periodSeconds × failureThreshold`.** Defaults (10s × 3) mean up to 30s of traffic to a dead pod before endpoint removal — tune both factors, not just one.
4. **Readiness gates traffic, liveness restarts, neither reschedules.** A restart happens in place on the same node; only eviction or deletion moves a pod. Restarting a pod cannot fix a node problem.
5. **Shutdown contract: app handles SIGTERM + `preStop` sleep 5-10s + `terminationGracePeriodSeconds` (default 30s) > real drain time.** Endpoint removal propagates asynchronously, so pods keep receiving traffic after SIGTERM — the sleep covers exactly that window.
6. **Diagnose in fixed order: events → previous logs → endpoints → exec.** Guessing skips the one cheap step that would have named the cause (→ The First Five Minutes). Events expire after 1h by default (`--event-ttl`) — capture them before touching anything.
7. **Requests drive both scheduling and HPA math.** `desired = ceil(current × usage/target)`; 3 replicas at 90% usage with a 60% target → `ceil(3 × 90/60)` = 5. Undersized requests inflate utilization and overscale.
8. **Pin images by digest or an immutable tag.** `latest` + `imagePullPolicy: IfNotPresent` means each node runs whatever it cached — different code per node, undebuggable. Note the default: pull policy is `Always` when the tag is `latest` or absent, `IfNotPresent` otherwise.
9. **See the diff before the cluster does.** `kubectl diff -f` and `--dry-run=server` run defaulting, admission, and webhooks; `--dry-run=client` runs none of them and passes manifests that the API server will reject (`apply_gate`).

## The First Five Minutes

1. `kubectl get events -n <ns> --sort-by=.lastTimestamp | tail -30` — capture first; the 1h TTL destroys evidence while you theorize.
2. `kubectl describe pod <pod>` — the Events block explains scheduling failures, image errors, probe failures, and OOM kills in one place.
3. `kubectl logs <pod> -p --tail=100` — previous container after any restart. Current logs are empty precisely because it just restarted.
4. `kubectl get endpointslices -l kubernetes.io/service-name=<svc>` — separates "the app is broken" from "the app was never wired up".
5. `kubectl debug -it <pod> --image=busybox --target=<container>` (kubectl >=1.25) — the pod's own view of DNS, reachability, and files.

Only when 1-5 all point outside the pod does the node become the suspect (`nodes.md`). Write down which step produced the finding: it names the file to open next. How much of that chain you narrate back follows `explain_depth`.

## Status Decoder

| Status / signal | Means | Next |
|---|---|---|
| `Pending`, no node assigned | Scheduler rejected every node; Events name the predicate | `scheduling.md` |
| `ContainerCreating` > 2 min | Volume attach, image pull, or CNI IP allocation is stuck | `describe` Events, then `storage.md` or `networking.md` |
| `Init:0/2` | An initContainer is failing or waiting; app containers never start | `logs -c <init-container>` |
| `ImagePullBackOff` / `ErrImagePull` | Auth, name/tag, or registry rate limit | `debug.md` |
| `CreateContainerConfigError` | A referenced ConfigMap/Secret or key does not exist | `config-and-secrets.md` |
| `CrashLoopBackOff` | Container exits repeatedly; backoff 10s → 5m cap | `logs -p`, `debug.md` |
| Exit 1 / app-specific code | Application exited on its own | App logs; config or dependency |
| Exit 137 with `OOMKilled: true` | cgroup memory limit hit | `resources.md` |
| Exit 137 with `OOMKilled: false` | SIGKILL after the grace period expired — a liveness kill or a delete whose SIGTERM was ignored | `probes.md`, `rollouts.md` |
| Exit 143 | Clean SIGTERM shutdown; usually not a bug | — |
| Exit 126 / 127 | Entrypoint not executable / not found (wrong arch, musl vs glibc) | `docker` skill |
| `Evicted` | Node pressure; QoS class chose the victim | `nodes.md` |
| `Completed` but restarting | `restartPolicy: Always` on a batch workload | `jobs.md` |
| `Terminating` past the grace period | Finalizer held, or the kubelet is unreachable | `operators.md`, `nodes.md` |
| `Unschedulable` after running fine | Node cordoned, drained, or its taints changed | `nodes.md` |
| Any other status, or a status that contradicts the symptom | The phase string is a summary, not a diagnosis — Events and the previous container's logs are | `describe` the pod, then `debug.md` |

## Resources and QoS

- Requests are scheduling-time only, never enforced at runtime. Limits are enforced: memory by cgroup OOM kill, CPU by CFS quota inside every 100ms period.
- QoS decides eviction order under node pressure: BestEffort (no requests) dies first, then Burstable pods exceeding requests, Guaranteed (requests = limits for both CPU and memory) last.
- Units: 1 CPU = 1000m. Memory `Mi` = 2^20, `M` = 10^6 — and a lone `m` suffix on memory means millibytes (→ Traps).
- CPU throttling starts well below 100% average usage: a bursty app drains its quota inside one 100ms window and stalls until the next. Symptom is p99 latency spikes at low average CPU.
- LimitRange injects defaults into pods that omit requests; ResourceQuota rejects those pods outright. Same manifest, different namespace, different outcome.

Sizing method, runtime-specific heap caps, quota arithmetic, and throttling forensics: `resources.md`.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| `memory: 512m` | `m` = millibytes → a limit of 0.512 bytes, instant OOM | `512Mi` |
| ConfigMap mounted with `subPath` | subPath binds the file inode once; the kubelet's symlink-swap update works only at directory level, so updates never arrive | Mount the whole ConfigMap as a directory, or env vars + `rollout restart` |
| Liveness probe checks a dependency | Dependency outage → every pod fails liveness → cluster-wide restart storm on top of the outage | Liveness = in-process only; readiness owns dependencies |
| Keeping `timeoutSeconds: 1` (the default) | One GC pause or load spike kills the pod under the exact load that needed it alive | 2-5s on any probe doing real work |
| First NetworkPolicy in a namespace | Selected pods flip to default-deny for that direction; DNS breaks first, everything else follows | Ship the DNS egress rule (UDP/TCP 53) in the same change |
| Deleting a PVC on a dynamically provisioned PV | Default `reclaimPolicy: Delete` destroys the data with the claim | `Retain` on anything you cannot re-derive; verify before deleting |
| Force-deleting a Terminating pod | The API forgets the pod while the kubelet may still be running the container — StatefulSets split-brain on the "freed" identity | Remove the stuck finalizer or fix the node; force only when the node is confirmed dead |
| CronJob with defaults | `concurrencyPolicy: Allow` piles up overlapping runs when one hangs; >100 missed schedules with no `startingDeadlineSeconds` stops scheduling entirely | `Forbid` (or `Replace`) + `startingDeadlineSeconds` |
| PDB stricter than the replica count | `maxUnavailable: 0`, or `minAvailable` = replicas, blocks every node drain and hangs cluster upgrades | Leave ≥1 pod of slack, or accept the pause consciously |
| Changing a Deployment's `spec.selector` | The field is immutable; the apply fails, and label edits without it fail validation | Fix the label scheme before first apply; otherwise create a new Deployment and shift traffic |
| Unbounded `emptyDir` | It consumes node ephemeral storage; the node hits DiskPressure and evicts your own pod | `sizeLimit` on every emptyDir, or a real volume |
| `hostPort` or `hostNetwork` for convenience | One pod per node per port, silent Pending on the next replica, and it bypasses NetworkPolicy | Service + Ingress; hostNetwork only for genuine node agents |
| Two Services selecting the same pods | Both work, both look correct, and traffic splits in ways no dashboard shows | One selector per workload; check with `kubectl get svc -o wide` before adding |
| Debugging from `latest` logs | You cannot know which code produced them (→ Core Rule 8) | Pin tags; log the image digest at startup |

## Output Gates

Before emitting a manifest (or approving one in review), verify:

- Memory request = limit; CPU request present, CPU limits per `cpu_limits_policy`?
- Readiness probe present; liveness in-process only; startupProbe wherever boot can exceed 30s?
- `terminationGracePeriodSeconds` above real drain time, and a `preStop` sleep for endpoint propagation?
- Image pinned to a digest or immutable tag, never `latest`?
- securityContext matches `psa_level`: `runAsNonRoot`, numeric UID, `allowPrivilegeEscalation: false`, capabilities dropped?
- Labels follow `label_scheme`, and the Deployment selector is the one you can live with forever?
- PDB present for every workload above 1 replica, and looser than the replica count?
- Namespace realities checked: ResourceQuota, LimitRange, NetworkPolicy, PSA labels?
- Verified through `apply_gate` (`kubectl diff -f` or `--dry-run=server`), not client-side validation?

## Where Experts Disagree

- **CPU limits.** One camp drops them entirely: CFS throttling adds tail latency even at low average usage, and requests already guarantee a fair share under contention. The other keeps limits = requests for predictability and multi-tenant fairness. Frontier: trusted workloads on dedicated clusters → requests only; multi-tenant platforms, chargeback, or compliance regimes → limits on.
- **Liveness probes.** Default-off argues most apps never deadlock without crashing, so liveness only adds restart storms and hides bugs; default-on wants a self-healing floor. Frontier: add liveness only when the process demonstrably hangs without exiting, and the probe reads purely in-process state — otherwise readiness alone.
- **Databases in the cluster.** Operators have made in-cluster Postgres and Kafka genuinely viable; the counter-argument is that storage failure modes, backups, and upgrades are where the operator's maturity gets tested during your outage. Frontier: managed service unless you already run a platform team that practices restores (`stateful.md`).
- **Helm vs kustomize vs plain YAML.** Helm wins at distributing software to strangers; kustomize wins at your own manifests across environments; plain YAML wins below roughly a dozen objects. Templating a chart for a single cluster you own is a cost with no buyer.
- **Cluster granularity.** One large multi-tenant cluster maximizes bin-packing and minimizes control-plane cost; clusters per team or per environment maximize blast-radius isolation and upgrade freedom. Frontier: the moment one team's admission webhook or CRD upgrade can break another team, isolation is cheaper than the incident.

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/k8s (install if the user confirms):
- `docker` — building and hardening the images k8s runs
- `devops` — CI/CD pipelines that deploy to the cluster
- `incident-response` — coordinating the humans when the cluster page fires
- `observability` — metrics, logs, and traces behind the dashboards you read here
- `linux` — node-level debugging under the kubelet

## Feedback

- If useful, star it: https://clawic.com/skills/k8s
- Latest version: https://clawic.com/skills/k8s

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/k8s.
