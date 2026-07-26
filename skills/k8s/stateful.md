# StatefulSets and Databases — Identity, Ordering, and Failover

A StatefulSet buys three guarantees a Deployment cannot give: stable network identity, stable per-pod storage, and ordered lifecycle operations. Everything painful about running stateful workloads comes from those guarantees being enforced strictly, exactly when you want them relaxed.

## The Identity Contract

- Pods are `<sts>-0 … <sts>-N`, and the ordinal is stable across restarts, rescheduling, and node loss.
- A headless Service named in `serviceName` provides per-pod DNS: `<sts>-0.<svc>.<ns>.svc.cluster.local`. Without it, per-pod addressing does not exist and clustered software cannot form a membership list (`dns.md`).
- `volumeClaimTemplates` create one PVC per pod, named `<template>-<sts>-<ordinal>`. That PVC is bound to that ordinal forever: pod 2 always gets pod 2's disk, on whatever node it lands.
- Consequence people meet late: scaling from 5 to 3 and back to 5 gives pods 3 and 4 their **old data**. Either that is the feature (a database rejoining its replica set) or it is a stale-state bug that looks like corruption.

## PVC Retention

- Default: PVCs survive scale-down and StatefulSet deletion. Nothing you do at the StatefulSet level deletes data by accident — and nothing cleans it up either. Orphaned PVCs from deleted StatefulSets are the most common unnoticed cloud-storage bill.
- `persistentVolumeClaimRetentionPolicy` takes `whenDeleted` and `whenScaled`, each `Retain` or `Delete`. `whenScaled: Delete` is right for caches, catastrophic for databases.
- Audit with `kubectl get pvc -A --sort-by=.metadata.creationTimestamp` and match against existing StatefulSets; anything unmatched is either a restore artifact or waste.

## Ordered Operations and Their Deadlocks

- `OrderedReady` (default) creates pods 0→N, waiting for Ready at each step, and deletes N→0. Rolling updates go highest ordinal first, so pod 0 — usually the one that bootstrapped the cluster — is updated last.
- One pod that never becomes Ready halts the rollout permanently. That is the intended behavior: a database rollout that continues past a broken member is how you lose quorum.
- `rollingUpdate.partition: N` updates only ordinals ≥ N. Canary procedure: set `partition` to `replicas`, apply the new template (nothing moves), lower to `replicas-1` (one pod updates), verify, then step down to 0.
- `podManagementPolicy: Parallel` speeds up scaling and initial creation only; the rolling update stays ordered. Choosing it for a system that requires ordered bootstrap breaks the first install and nothing after.
- Surgery escape hatch: `kubectl delete sts <name> --cascade=orphan` removes the controller while leaving pods and PVCs running — the supported way to change an immutable field (`volumeClaimTemplates`, `serviceName`) without an outage. Recreate the StatefulSet with the new spec and it adopts the existing pods.

## Failover Reality

- When a node dies, the StatefulSet controller will not create a replacement pod while the old one exists in `Terminating`. The guarantee is at-most-one pod per ordinal — recreating it while the original might still be running would mean two processes writing one volume.
- So a dead node means a stuck pod, by design. It clears when the node object is deleted (the cloud controller does this for terminated instances) or when you fence the node manually.
- Force-deleting the pod breaks the at-most-one guarantee. Only do it after confirming the kubelet cannot be running the container: the instance is terminated, or the machine is powered off. "The node is unreachable" is not confirmation — a network partition looks identical and is the exact case where two writers destroy the data.
- Expect `Multi-Attach error for volume` during failover while the old attachment times out (`storage.md`). It resolves on its own; the fix is patience or node-object deletion, never force-deleting the PVC.

## Running Databases Well (if you run them)

- Anti-affinity by hostname at minimum, topology spread by zone for anything with a quorum. Three replicas in one zone is one AZ outage away from total loss (`scheduling.md`).
- PDB expressed as `minAvailable: <quorum>` — for a 3-node quorum system that is 2. `maxUnavailable: 1` says the same thing and stays correct when you scale to 5.
- Graceful membership: a `preStop` hook that removes the node from the cluster (demote a primary, transfer a shard, leave the raft group) before shutdown, with `terminationGracePeriodSeconds` sized for the slowest of those operations. Databases routinely need 120-600s here, not the 30s default (`probes.md`).
- Readiness must mean "this member can serve", not "the process is up" — a replica catching up on 40 GB of WAL is not ready. Liveness must not check cluster membership, or a leader election blip restarts the whole cluster (`probes.md`).
- Backups are logical dumps plus volume snapshots, in a different failure domain, with a restore rehearsed on a schedule. A backup nobody has restored is a hypothesis.
- Resource shape: memory request = limit, generous ephemeral storage for WAL and temp files, and CPU limits off (`cpu_limits_policy: none`) or well above the request (`explicit`, ≥2×) — `equal-requests` is the wrong policy for a database: throttling at checkpoint time creates replication lag that looks like a network problem (`resources.md`).

## Operators

- An operator is a controller plus CRDs that encodes the failover, backup, and upgrade procedures you would otherwise write as runbooks. The good ones (mature Postgres, Kafka, Redis operators) handle primary failover, PITR, and version upgrades.
- Evaluate on the failure path, not the install path: what happens when the primary's node dies mid-write, how a point-in-time restore is performed, whether upgrades are tested across the version you run. The demo is always five minutes; the incident is not (`operators.md`).
- Operator upgrades are cluster-wide events. Read the CRD changelog; a CRD schema change can invalidate every existing custom resource at once.

## When Not To Run It In-Cluster

Honest frontier: managed services win unless at least two of these hold — you already operate a platform team that practices restores; the data is reproducible from a source of truth; latency or cost genuinely requires co-location; a mature operator exists for your engine and your version. Otherwise the cluster gets the stateless workloads and the managed service gets the data, and your incidents get shorter.

## Stateful Triage

| Symptom | Cause | Move |
|---|---|---|
| `<sts>-0` Pending, others fine | Ordered creation blocked on ordinal 0's PVC or placement | `describe pvc <template>-<sts>-0`, then `scheduling.md` |
| Rollout stopped mid-way | A higher ordinal never became Ready — intended halt | Fix that pod; do not raise `partition` to skip it |
| Pod Terminating forever after node loss | At-most-one guarantee | Confirm the node is dead, delete the node object |
| New pod cannot mount | Old attachment still held | Wait out the detach timeout (`storage.md`) |
| Scaled up, old data reappeared | Retained PVCs bound to those ordinals | Delete the PVCs deliberately, or expect the data |
| Cannot edit `volumeClaimTemplates` | The field is immutable | `--cascade=orphan` recreate |
| Two pods writing one volume | RWO across nodes, or a force-delete during a partition | Move to `ReadWriteOncePod`; restore from backup |

Stateful work produces the numbers nobody wants to measure twice: the real `terminationGracePeriodSeconds` a graceful membership change needs, the quorum size and its PDB, the observed peak memory of a database under load. Write them into the workload's row in `## Workloads` in `~/Clawic/data/k8s/memory.md`. The backup schedule and the restore rehearsal go in the `## Due` table, and the rehearsal's measured RTO in `deploys/<year>.md` (`backup.md`); the decision to run this engine in-cluster rather than managed, with what it was traded against, belongs in `artifacts/decision-<kebab>.md` with its `## Boxes` line.
