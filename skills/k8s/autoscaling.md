# Autoscaling — Pods, Nodes, and Why It Was Too Slow

Three independent systems, often confused: HPA changes replica count, VPA changes resource requests, and the cluster autoscaler changes node count. They interact, and two of the three can fight.

## HPA Arithmetic

```
desired = ceil( currentReplicas × currentMetric / targetMetric )
```

3 replicas at 90% CPU with a 60% target → `ceil(3 × 90/60)` = 5 (SKILL.md rule 7).

- Utilization is measured **against requests**, not limits and not node capacity. Halving a request doubles reported utilization and the HPA scales without any change in load. Requests are therefore an autoscaling parameter, not just a scheduling one (`resources.md`).
- A 10% tolerance suppresses tiny corrections: at a 60% target, nothing happens between 54% and 66%.
- The controller re-evaluates every 15s by default and reads metrics from metrics-server, which itself scrapes on an interval. Expect 30-60s before a load change is even visible to the HPA.
- `<unknown>` in `kubectl get hpa` has exactly two causes: a container with no request for that resource, or metrics-server not serving. Both are visible in `kubectl describe hpa`.
- Multiple metrics are evaluated independently and the **largest** desired count wins. Adding a memory target to a CPU-driven HPA can pin replicas high forever, because memory rarely falls back after a JVM grows its heap — memory is usually the wrong HPA signal.

## Behavior Tuning

- Scale-down uses a 300s stabilization window by default (it takes the highest recommendation over the last 5 minutes); scale-up has none. Asymmetry is intentional: scaling up wrongly costs money, scaling down wrongly costs an outage.
- `behavior.scaleUp.policies` bound the rate (`percent` or `pods` per `periodSeconds`). A single burst policy of 100% per 15s doubles the fleet quickly and can overwhelm a shared database — the autoscaler's blast radius is your dependencies.
- Flapping means the target sits at the natural operating point. Widen the window, lower the target, or scale on a metric with less variance (queue depth, requests per second) rather than CPU.
- `minReplicas` is your availability floor and interacts with PDBs and topology spread (`rollouts.md`, `scheduling.md`). `minReplicas: 1` plus a PDB requiring 1 available blocks every drain.

## Choosing the Signal

| Workload | Better metric than CPU |
|---|---|
| HTTP service with a latency SLO | Requests per second per pod, or concurrency (in-flight requests) |
| Queue worker | Queue depth or oldest-message age (KEDA, external metrics) |
| gRPC service | Active streams; CPU under-reports because the wait is I/O |
| Batch | Do not autoscale replicas — scale the Job's parallelism (`jobs.md`) |
| Anything memory-bound | Fix sizing; memory is a poor scaling signal because it does not shrink |

KEDA adds event-source scalers (Kafka lag, SQS depth, Prometheus queries, cron windows) and scale-to-zero for workloads that are legitimately idle. Scale-to-zero moves cold-start cost onto the first request — acceptable for internal tools, rarely for user-facing paths.

## VPA

- Three modes: `Off` (recommendations only), `Initial` (applies at pod creation), `Auto`/`Recreate` (evicts pods to resize them).
- `Off` mode is the highest-value, lowest-risk use: it produces the p90 numbers you would otherwise guess for requests. Read them, apply them in Git, keep the manifest as the source of truth (`manifests.md`).
- `Auto` evicts pods to apply changes. On a workload with a tight PDB, that eviction can stall; on a StatefulSet, it is a restart of a database member. In-place resize is landing across versions but is not something to assume — check the cluster.
- VPA and HPA on the same resource metric fight: VPA raises requests, which lowers measured utilization, which makes the HPA scale in, which raises per-pod load, which makes VPA raise requests again. Use VPA for memory and HPA for CPU, or use VPA in `Off` mode.

## Cluster Autoscaler and Karpenter

- Cluster autoscaler scales a node group up when a Pending pod would fit on a new node of that group's shape. It simulates: a pod that fits no shape stays Pending forever with no scale-up and an event saying so.
- Scale-down needs a node under its utilization threshold (50% of requests by default) for a sustained period (10 minutes by default) **and** every pod on it must be movable.
- Blockers, in order of how often they are the answer: a PDB that cannot be satisfied, kube-system pods with no PDB, pods with local storage (emptyDir counts), the `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` annotation, and restrictive affinity that has nowhere else to go.
- Karpenter-style provisioners skip node groups and create right-sized nodes per pending workload, then consolidate by replacing under-used nodes. Faster and cheaper, and consolidation is a continuous voluntary disruption — PDBs and graceful shutdown stop being optional (`rollouts.md`).
- Scale-from-zero requires the node group to advertise its labels, taints, and capacity to the autoscaler; without those hints, a pod with a nodeSelector for an empty group is never scheduled.

## The Latency Budget (why autoscaling did not save you)

```
metric visible (15-60s) + HPA decision (≤15s) + [node provision 30-120s if needed]
  + image pull (0-60s) + pod boot + readiness (startup budget)
```

Realistically 1-3 minutes for pod scaling, 3-6 with a node provision. A traffic spike that arrives in 30 seconds is absorbed by headroom, not by autoscaling.

- Buy headroom deliberately: a lower HPA target (50-60% instead of 80%) is the simplest form.
- For faster node scale-up, run overprovisioning pods — a Deployment of low-priority pause pods sized like a node's worth of capacity. Real workloads preempt them instantly and the autoscaler replaces the displaced pause pods in the background (`scheduling.md`).
- Cache images on nodes or use a registry mirror if pull time dominates; a 2 GB image is a minute of scale-up you cannot tune away (`docker` skill).

## Triage

| Symptom | Cause |
|---|---|
| `<unknown>` metric | Missing requests, or metrics-server down |
| Scales up, never down | 300s stabilization plus a metric that does not fall; or `minReplicas` already reached |
| Oscillates | Target at the operating point, or HPA and VPA fighting |
| At `maxReplicas`, still saturated | The bottleneck is downstream (database, lock, external API) — more replicas add load, not capacity |
| Pods Pending, no new nodes | No node shape fits the request, quota exhausted, or the pod's affinity excludes every group |
| Nodes never scale down | Read the blocker list above in that order |
| Scaled up but latency did not improve | Cold caches or connection-pool limits at the dependency; measure per-replica warm-up before blaming the autoscaler |

VPA recommendations, the HPA target that finally stopped flapping, the measured scale-up latency, and the downstream ceiling a workload hits at `maxReplicas` all belong in that workload's row in `## Workloads` in `~/Clawic/data/k8s/memory.md`. Each took observation under real load to establish, and the last one — "more replicas stop helping at 12 because the database connection pool is the wall" — is the fact most likely to be rediscovered painfully during a traffic spike.
