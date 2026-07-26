# Resources — Requests, Limits, QoS, and Throttling

**Before sizing anything, read `## Workloads` in `~/Clawic/data/k8s/memory.md`** (or the file its `## Boxes` line names). A peak already measured over a full traffic cycle beats anything you can infer in a session, and the row carries the date it was taken.

Requests buy you a place on a node. Limits are what the kernel enforces on you. Confusing the two produces both halves of the classic pair: a cluster that is 100% "full" at 15% utilization, and nodes that OOM-kill innocent pods.

## What Each Field Actually Does

| Field | Enforced by | When | Consequence of getting it wrong |
|---|---|---|---|
| `requests.cpu` | Scheduler + cgroup weight | Placement; contention only | Too low: scheduled onto a hot node, starved under contention, HPA math inflated |
| `requests.memory` | Scheduler only | Placement | Too low: node accepts more pods than fit; someone gets OOM-killed under burst |
| `limits.cpu` | CFS quota, every 100ms | Continuously | Throttling at low average CPU, tail-latency spikes |
| `limits.memory` | cgroup OOM killer | On allocation | Container killed instantly at the boundary — no warning, no graceful path |

- CPU requests become a cgroup weight (1 core ≈ 1024 shares): they matter only when the node is contended, and then they divide CPU proportionally. On an idle node a 100m pod can use four cores.
- Memory requests have no runtime effect at all. Nothing reserves that memory for you; the guarantee is that the scheduler counted it.
- Memory is incompressible: over the limit means killed. CPU is compressible: over the limit means slowed. That asymmetry is the whole reason memory requests should equal limits in production (SKILL.md rule 1) while CPU limits stay debatable.

## QoS Classes

| Class | Condition | Eviction order under node pressure |
|---|---|---|
| Guaranteed | Every container: requests = limits, for both CPU and memory | Last |
| Burstable | At least one request set, not matching limits | Middle — those most over their requests go first |
| BestEffort | No requests or limits anywhere | First |

The kubelet ranks Burstable victims by usage relative to requests, not absolute size. A well-behaved 8Gi pod that stays under its request outlives a 512Mi pod at 4× its request.

## Throttling Forensics

```
quota per period = limits.cpu × 100ms      # limits.cpu: 500m → 50ms of CPU per 100ms
throttled if the container needs more than that inside any single 100ms window
```

- A request that needs 200ms of CPU on a 500m limit takes at least 400ms wall time, spread across four periods, even on an empty node.
- Signal: `container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total` above ~10%, while average CPU sits far below the limit.
- Multi-threaded runtimes make it worse: 8 threads on a 1-core limit exhaust the quota in 12.5ms and stall for the remaining 87.5ms. Match runtime concurrency to the limit (`GOMAXPROCS`, JVM `ActiveProcessorCount`, thread pools) — the container sees the node's core count unless you tell it otherwise.
- Fix order: raise the limit, then reduce concurrency, then split the workload. Adding replicas does not fix throttling; each new replica gets the same quota.

## Runtime Memory Caps (the container limit is not the app's budget)

| Runtime | Default behavior in a container | Set this |
|---|---|---|
| JVM | `MaxRAMPercentage` 25 → a 4Gi limit yields a 1Gi heap | 50-75 depending on non-heap footprint; the gap must hold metaspace, threads, code cache, direct buffers |
| Node.js | Old-space default sized from the host on older builds; heap ignores the cgroup | `--max-old-space-size` at ~75-80% of the limit, in MB |
| Go | No heap cap; the GC grows until the kernel objects | `GOMEMLIMIT` at ~90% of the limit (soft), and `GOMAXPROCS` to the CPU limit |
| .NET | GC heap hard limit defaults to ~75% of the container limit | Usually fine; tune `DOTNET_GCHeapHardLimitPercent` only with evidence |
| Python | No cap of any kind | Bound the workload (worker count, batch size) or accept OOM as the limit |

Any runtime cap must sit below the container limit, and the container limit below what the node can honor. Three layers, each one strictly smaller.

## Sizing Without Guessing

1. Observe a full traffic cycle including the daily peak: `container_memory_working_set_bytes` and `rate(container_cpu_usage_seconds_total[5m])` per container.
2. Memory request = limit = observed peak × 1.3. Working set includes dirty page cache, so heavy file I/O inflates it — that is real memory pressure, not a measurement artifact.
3. CPU request = p90 of usage. Not the peak: requests are a reservation you pay for on every node, every replica, all day.
4. CPU limit per `cpu_limits_policy`. `none`: leave it unset (requests still divide contended CPU). `explicit`: never below 2× the request, or the burst headroom you just sized for disappears. `equal-requests`: the limit IS the request — that buys QoS Guaranteed and predictable neighbors and pays for it with throttling on every burst, so it is a conscious multi-tenancy trade, not a safe default (→ SKILL.md Where Experts Disagree).
5. Re-measure after any dependency or traffic-shape change. A VPA in `recommender`-only mode gives you the numbers without letting it restart pods (`autoscaling.md`).

Startup is the exception that breaks step 2: JVM and Node apps often need 2-3× steady-state CPU for the first 30-60s. Sizing CPU limits to steady state turns a 20s boot into a 3-minute one, which then fails the liveness probe (`probes.md`).

## Namespace Governors

- **ResourceQuota** caps totals per namespace (`requests.cpu`, `limits.memory`, `pods`, object counts, `requests.storage` per StorageClass). The rule that surprises everyone: once a quota names a resource, every pod in that namespace MUST set that field or be rejected at admission — the error appears on the ReplicaSet, not the Deployment (`rollouts.md`).
- Quota scopes make policy expressible: `BestEffort`, `NotTerminating`, `PriorityClass`. A quota scoped to a high PriorityClass is how you stop teams promoting their own workloads.
- **LimitRange** injects `defaultRequest` and `default` into containers that omit them, and enforces `min`, `max`, and `maxLimitRequestRatio`. It applies at pod creation only: existing pods keep whatever they had, so the effect of a change appears one rollout later.
- LimitRange and ResourceQuota interact: LimitRange defaults can satisfy a quota that would otherwise reject the pod. That is why the same manifest works in one namespace and fails in another (`debug.md`).
- Quota usage is not recalculated retroactively; if it drifts (`kubectl describe quota` disagreeing with reality), the controller resyncs on the next object change rather than on demand.

## Ephemeral Storage and Other Silent Resources

- `ephemeral-storage` requests/limits cover the container writable layer, emptyDir, and logs. Exceeding the limit evicts the pod with `Evicted: ephemeral local storage usage exceeds`. Unset, a runaway log file takes the node to DiskPressure and evicts neighbors (`nodes.md`).
- Always set `sizeLimit` on emptyDir. `medium: Memory` emptyDir counts against the pod's memory limit — a tmpfs cache that "leaks" is really your memory budget being spent on files.
- Pod-level `resources` (sizing the pod as a whole instead of per container) and in-place resize of a running pod are recent additions that arrive on different schedules per distribution: confirm with `kubectl explain pod.spec.resources` and `kubectl explain pod.spec.containers.resizePolicy` on the actual cluster before designing around either. Default to per-container sizing.
- PID limits per pod exist at the kubelet level (`--pod-max-pids`); a fork bomb without them takes down the node, not just the pod.

## Fleet Audit

```bash
# Workloads with no memory limit — the pods that make the node OOM someone else
kubectl get deploy -A -o json | jq -r '.items[] | select(.spec.template.spec.containers[].resources.limits.memory == null) | "\(.metadata.namespace)/\(.metadata.name)"'
# Requests vs allocatable per node — the true fullness of the cluster
kubectl describe nodes | grep -A5 "Allocated resources"
```

Compare cluster-wide `sum(requests)` against `sum(allocatable)`: below ~50% you are paying for air; above ~85% a single node failure has nowhere to reschedule (`production.md`).

Write every number this produced into the workload's row in `## Workloads` in `~/Clawic/data/k8s/memory.md` — observed peak memory with the date it was measured, p90 CPU, and the quota ceiling the namespace enforces (`memory-template.md` for the columns). Observing a full traffic cycle costs a day; re-observing it next quarter costs another one.
