# Rollouts — Shipping Without An Outage, Rolling Back Without A Prayer

A Deployment rollout is a ReplicaSet handoff governed by four numbers. Knowing which number is stalling you is the entire skill.

## The Four Numbers

| Field | Default | What it controls |
|---|---|---|
| `maxSurge` | 25% | Extra pods above `replicas` during the update. Rounds **up** |
| `maxUnavailable` | 25% | Pods allowed missing during the update. Rounds **down** |
| `minReadySeconds` | 0 | How long a pod must stay Ready before it counts as available |
| `progressDeadlineSeconds` | 600 | When a stalled rollout is marked `Progressing=False` — it never rolls back on its own |

- Rounding matters at small scale: 2 replicas at the defaults gives surge 1, unavailable 0 — Kubernetes protects you from a full outage there. At 10 replicas you get surge 3, unavailable 2, so 8 pods carry peak traffic mid-rollout. Size for that or set `maxUnavailable: 0`.
- `maxUnavailable: 0` plus `maxSurge: 1` is the safest and slowest configuration; it requires headroom for one extra pod and a cluster autoscaler that can provide it (`autoscaling.md`).
- `minReadySeconds` is the cheapest canary available: an image that passes its first readiness check and then crashes will otherwise replace the entire fleet before the first restart. 5-30s catches immediate-crash regressions; it does not catch slow leaks.
- `progressDeadlineSeconds` only sets a condition. Automatic rollback does not exist in core Kubernetes — a CI step watching `kubectl rollout status --timeout` is how teams actually get it.

## Revisions and Rollback

- Revisions are ReplicaSets, identified by the pod-template hash. `revisionHistoryLimit` (default 10) is how many you can roll back to; setting it to 0 destroys your rollback path.
- `kubectl rollout undo` creates a NEW revision with the old template. History moves forward, never backward — which is why "undo twice" toggles between two versions instead of going further back. Use `--to-revision=<n>` with `kubectl rollout history`.
- Only pod-template changes create a revision. A ConfigMap edit changes nothing about the Deployment, so there is nothing to roll back to. Canonical fix: hash the config into a pod annotation (`kustomize` ConfigMap generators and Helm's `checksum/config` annotation both do this) so config changes are template changes.
- `kubectl rollout restart` works by stamping `kubectl.kubernetes.io/restartedAt` on the template — a normal rolling update, honoring every constraint above. It is the correct way to pick up mounted-file changes or refresh connections.
- Rolling back the code does not roll back a database migration. Expand/contract is the requirement: deploy schema changes that the previous version tolerates, ship the code, remove the old schema in a later release. Without it your rollback path is theoretical.

## Diagnosing A Stuck Rollout

1. `kubectl rollout status deploy/<d>` — it prints which side is blocked ("waiting for deployment to finish: 3 out of 5 updated replicas are available").
2. `kubectl describe rs <newest-rs>` — the ReplicaSet, not the Deployment, holds the real errors: quota rejection, admission webhook denial, image pull failure, PVC binding.
3. New pods Pending → `scheduling.md` (surge capacity does not exist). New pods running but not Ready → `probes.md`. New pods ready then dying → `debug.md` crash chain.
4. Old pods refusing to terminate → a finalizer, a preStop hook longer than the grace period, or a PDB blocking eviction during a node-level operation (`operators.md`, below).
5. Rollout completed but the old version is still serving → two Deployments matching one Service selector, or a stale endpoint (`networking.md`).
6. `Progressing=False` with `ProgressDeadlineExceeded` is a symptom, never a cause: it means one of 3-5 has been true for 10 minutes.

## Deployment Strategies Beyond RollingUpdate

- `Recreate` — all old pods terminate before any new one starts. Mandatory for a singleton holding an RWO volume or a non-concurrent-safe migration; it is a deliberate outage window, so say so out loud.
- **Blue/green** — two Deployments, one Service; flip `spec.selector` to switch. Instant rollback (flip back), double the resources, and one shared Service label scheme. The flip is atomic per new connection, not for in-flight ones.
- **Canary by replica count** — two Deployments with a shared label the Service selects, replica counts 9:1 for a 10% canary. Free, coarse, and it works everywhere. Percentage granularity below ~5% needs a proxy that splits by request (Ingress canary annotations, service mesh, Argo Rollouts, Flagger).
- Sticky sessions defeat canaries: with session affinity, the same 10% of users hit the canary every time, and their pain is invisible in aggregate metrics.

## StatefulSet and DaemonSet Rollouts

- StatefulSet `OrderedReady` updates from the highest ordinal down, waiting for Ready at each step. One pod that never becomes Ready halts the rollout forever — and the halt is the feature (`stateful.md`).
- `partition: N` in `rollingUpdate` updates only ordinals >= N: a genuine canary for stateful workloads. Set partition to `replicas`, apply, then lower it step by step.
- `podManagementPolicy: Parallel` affects scale up/down, not the rolling update ordering — a frequent misreading.
- DaemonSet: `maxUnavailable: 1` by default, so a 200-node rollout is 200 sequential steps. `maxSurge` is available for DaemonSets that can tolerate two instances briefly; without it, raise `maxUnavailable` deliberately.

## PodDisruptionBudgets

- PDBs constrain **voluntary** disruptions only: `kubectl drain`, the eviction API, cluster autoscaler scale-down, node pool upgrades. They do nothing about crashes, OOM kills, node failures, or `kubectl delete pod`.
- Express as `minAvailable` when the floor matters (quorum systems) and `maxUnavailable` when the fraction matters (stateless fleets). `maxUnavailable: 1` on a 3-replica Deployment is the sane default.
- The classic deadlock: `minAvailable` equal to `replicas`, or a PDB on a 1-replica Deployment — every drain blocks forever and cluster upgrades hang (SKILL.md Traps).
- Pods that are already unhealthy can block eviction under the default policy; `unhealthyPodEvictionPolicy: AlwaysAllow` (verify availability on your cluster version) lets a drain proceed past pods that were never going to serve anyway.
- `kubectl drain` uses the eviction API and therefore respects PDBs; `kubectl delete pod` does not. Deleting pods to "speed up a drain" is how you breach the budget you wrote.

## Rollout Gate

Before starting a deploy that matters:

- Rollback path exists: previous revision retained, or the previous image digest recorded?
- Schema changes backward compatible with the running version?
- `minReadySeconds` above 0 so a first-check-passing crash loop cannot replace the fleet?
- `maxUnavailable` compatible with current traffic, and surge capacity actually available?
- PDB present and looser than the replica count?
- Grace period and `preStop` sized for real drain time (`probes.md`)?
- A watch on `kubectl rollout status --timeout=<deadline>` in CI, since nothing rolls back by itself?

Record the deploy in `~/Clawic/data/k8s/deploys/<year>.md` — date, workload, image digest and commit, chart or overlay version, and **the digest to roll back to**. During the next incident nobody can reconstruct which digest was last good, and `revisionHistoryLimit` may already have discarded the ReplicaSet that knew. A rollback that actually happened gets its own row with what forced it, and the failure shape goes to `## Incident History` in `memory.md`.
