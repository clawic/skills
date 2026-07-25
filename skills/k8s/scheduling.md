# Scheduling — Why This Pod Is Not On A Node

The scheduler filters, then scores. Filtering answers "which nodes *could* take this pod" using requests, taints, affinity, volumes, and ports; scoring picks among survivors. A Pending pod means filtering emptied the list, and the FailedScheduling event names the predicate that did it.

## The Arithmetic The Scheduler Used

```
allocatable = capacity − kube-reserved − system-reserved − eviction-hard threshold
fits if: sum(requests of pods already bound) + this pod's requests ≤ allocatable
```

- A 16 GiB node commonly exposes ~14.5 GiB allocatable; on managed distributions the reservation scales with node size. Sizing a pod against `capacity` is how a manifest that "obviously fits" stays Pending.
- `kubectl describe node <n>` prints `Allocated resources` — requests, not usage. A node at 12% CPU usage can be 100% requested and legitimately refuse pods.
- Limits are invisible to the scheduler. A cluster can be fully requested and idle, or fully idle and overcommitted into eviction (`resources.md`).
- Pod requests = `max( max(init container requests), sum(app container requests) + sum(native sidecar requests) )` + pod overhead from the RuntimeClass. One fat initContainer sets the floor for the whole pod's scheduling — a common surprise on migration jobs.

## Taints and Tolerations

| Effect | Behavior | Typical use |
|---|---|---|
| `NoSchedule` | New pods without the toleration are rejected | Dedicated node pools (GPU, spot, infra) |
| `PreferNoSchedule` | Soft; used in scoring, not filtering | Nudge away from expensive nodes |
| `NoExecute` | Also evicts already-running pods that do not tolerate it | Node lifecycle taints, forced drains |

- Built-in lifecycle taints do the real work: `node.kubernetes.io/not-ready`, `unreachable`, `memory-pressure`, `disk-pressure`, `pid-pressure`, `unschedulable`. Admission adds tolerations for not-ready and unreachable with `tolerationSeconds: 300` — that 5-minute default is why pods sit on a dead node for five minutes before rescheduling. Lower it per workload if failover speed matters; below ~60s expect churn on every transient blip.
- Tolerating a taint is permission, not attraction: pods land on tainted nodes only by accident unless paired with node affinity. Dedicated pools need both.
- `key` with `operator: Exists` and no `effect` tolerates everything on that key — the accidental "runs anywhere, including the broken node" configuration.

## Affinity, Anti-Affinity, Spread

- `requiredDuringSchedulingIgnoredDuringExecution` is a hard filter; `preferred` only adds score. "IgnoredDuringExecution" is literal: a running pod is never moved when the labels change.
- `podAntiAffinity` with `topologyKey: kubernetes.io/hostname` is the standard "one replica per node" — but required anti-affinity across a large cluster is O(pods × nodes) at scheduling time and gets slow above a few thousand pods. Prefer topology spread for scale.
- Topology spread is the modern tool: `maxSkew` (max difference in pod count between domains), `topologyKey`, `whenUnsatisfiable: DoNotSchedule | ScheduleAnyway`. `maxSkew: 1` on `topology.kubernetes.io/zone` with `DoNotSchedule` gives real multi-AZ guarantees; the same with `ScheduleAnyway` is a preference that silently degrades during a zone outage — which is often what you want.
- `minDomains` prevents the degenerate case where every replica lands in one zone because only one zone has schedulable nodes.
- Cluster-level default constraints exist (hostname `maxSkew: 3`, zone `maxSkew: 5`, both `ScheduleAnyway`). If spread "already works" without you asking, that is why; and it also means the default is not a guarantee.

## Priority and Preemption

- A PriorityClass with a higher `value` lets the scheduler evict lower-priority pods to make room. Preemption respects PDBs on a best-effort basis only — it will break a PDB rather than leave a critical pod Pending forever.
- `preemptionPolicy: Never` gives a pod scheduling precedence in the queue without letting it evict anything: the right setting for important-but-not-urgent batch.
- Reserve `system-node-critical` and `system-cluster-critical` for actual cluster components. Handing production apps a priority above cluster DNS means an autoscaling event can evict DNS.
- Preemption cascades: the victim reschedules elsewhere and may preempt again. Two priority tiers plus a default is usually the whole design; five tiers is a queueing system nobody can reason about.

## Scheduling Gates and Deliberate Waiting

- `spec.schedulingGates` holds a pod in `SchedulingGated` before the scheduler ever considers it; a controller removes the gate when its precondition is met (quota reserved, external resource ready). This is the supported alternative to "create the pod and let it fail until things exist".
- `WaitForFirstConsumer` on a StorageClass inverts the usual order: the volume is provisioned in the zone where the pod lands, instead of the pod being forced into the volume's zone (`storage.md`).

## The Scheduler Never Rebalances

Placement decisions are permanent until something deletes the pod. After adding nodes, scaling down, or fixing taints, the old distribution persists — hot nodes stay hot.

- Rebalance deliberately: a rolling restart redistributes, and the descheduler project automates the policy (duplicates, low utilization, violated affinity).
- Anti-pattern: scaling a Deployment up and down to "spread it out". You get a new random distribution, not a better one.

## Extended and Scarce Resources

- GPUs and other extended resources (`nvidia.com/gpu`) must be integers, and request must equal limit — there is no fractional GPU in core Kubernetes. Sharing needs a device-plugin strategy (time-slicing, MIG), decided at the node level.
- Extended resources are advertised by the node, so a pod requesting one is Pending until the device plugin registers — which looks identical to "no capacity" in the event message.
- Spot and preemptible nodes: taint them and tolerate deliberately, budget for 30-120s termination notices, and never place the only replica of a stateful workload there (`nodes.md`).

## Pending Triage Table

| Event text | Cause | Fix |
|---|---|---|
| `Insufficient cpu` / `Insufficient memory` | Requests exceed remaining allocatable everywhere | Right-size requests or add capacity; check the allocatable gap above |
| `node(s) had untolerated taint {k: v}` | Dedicated pool or a lifecycle taint | Add the toleration, or fix the node condition |
| `didn't match Pod's node affinity/selector` | Label typo, or the labeled nodes are full | `kubectl get nodes -l <selector>` — often returns nothing |
| `pod has unbound immediate PersistentVolumeClaims` | No default StorageClass, or provisioning failed | `storage.md` |
| `node(s) didn't match pod anti-affinity rules` | More replicas than domains | Raise domains, or relax to `preferred` |
| `node(s) didn't have free ports` | `hostPort` collision | Drop hostPort; use a Service |
| `0/N nodes are available` with no reason listed | Every node filtered by a different predicate — read all clauses, they are summed per predicate | Fix them in the order printed |
| `SchedulingGated` | A gate is still present | Find the controller that owns the gate |
