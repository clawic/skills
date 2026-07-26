# Storage — PVs, PVCs, and Data You Cannot Re-Derive

Decision rule first: **PVC with a dynamic StorageClass for data the workload owns**; **emptyDir with a `sizeLimit` for scratch that must not survive**; **ConfigMap/Secret for configuration** (`config-and-secrets.md`); **hostPath for nothing you would defend in a review**.

## The Binding Chain

```
PVC (what you ask for) → StorageClass (how it gets made) → PV (the real volume) → attach to node → mount into pod
```

Each arrow is a separate failure point, and `kubectl describe pvc` plus `describe pod` between them tells you which one stalled.

- `storageClassName: ""` and omitting the field are different: `""` disables dynamic provisioning entirely (static PVs only); omitted uses the cluster's default class. A cluster with no default class turns every omitted field into a Pending PVC.
- `volumeBindingMode: Immediate` provisions the volume the moment the PVC exists — in whatever zone the provisioner chooses — and the pod is then forced into that zone. `WaitForFirstConsumer` waits for the scheduler, so the disk lands where the pod fits. On any multi-zone cluster, WaitForFirstConsumer is the correct default (`scheduling.md`).
- Dynamically provisioned PVs default to `reclaimPolicy: Delete`: deleting the PVC deletes the data. Patch the class or the PV to `Retain` for anything you cannot re-derive.
- A `Retain` PV goes to `Released`, not `Available`, after its PVC is deleted, and no new claim can bind it until you clear `spec.claimRef`. That two-line patch is the restore procedure people rediscover during an outage.

## Access Modes (the "per node" that everyone reads as "per pod")

| Mode | Real meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Read-write by pods **on one node** — two pods on the same node can both mount it |
| `ReadOnlyMany` (ROX) | Read-only across many nodes |
| `ReadWriteMany` (RWX) | Read-write across many nodes; needs a network filesystem (NFS, EFS, CephFS) |
| `ReadWriteOncePod` (RWOP) | Exactly one pod, cluster-wide — the mode you actually wanted for a database |

- The RWO surprise appears only after a reschedule: the app "supported two replicas" because both landed on the same node, then corrupted itself when one moved.
- `Multi-Attach error for volume` means the old node still holds the attachment. Normal during failover of a node that died without detaching; it clears when the attachment times out or the node object is removed (`nodes.md`).
- RWX over NFS is fine for bulk artifacts and pathological for dependency trees or database files: per-file latency dominates and locking semantics differ from local filesystems.

## Expansion, Never Contraction

- `allowVolumeExpansion: true` on the StorageClass, then edit `spec.resources.requests.storage` on the PVC. Most CSI drivers resize online; some need a pod restart to grow the filesystem — `describe pvc` says which by the condition it sets.
- Shrinking is not supported by any path. Sizing generously once is cheaper than migrating later, and a PVC 4× the current need still costs less than the incident.
- A full PVC does not evict the pod and does not appear as a Kubernetes error: the application returns write failures while everything looks Ready. Alert on `kubelet_volume_stats_available_bytes` — this is the storage failure with no Kubernetes-level symptom.

## Snapshots and Backups

- `VolumeSnapshotClass` + `VolumeSnapshot` gives you a CSI snapshot; restore by creating a PVC with `dataSource` pointing at it.
- CSI snapshots are **crash-consistent**, not application-consistent: they capture what was on disk, including a half-written transaction. Databases need their own quiesce or logical dump for a guaranteed-restorable copy (`stateful.md`).
- Snapshots usually live in the same storage system as the volume — that is availability, not backup. A real backup is in a different failure domain, and it has been restored at least once.
- etcd backups contain PVC and PV objects, never the data inside them. Cluster restore without a storage-layer restore gives you pods that mount empty disks.
- Cloning (`dataSource` pointing at another PVC) is the fastest way to make a staging copy of production data — and the fastest way to copy production secrets into a less protected namespace. Decide deliberately.

## Volume Types Beyond PVCs

- `emptyDir` — node-local, deleted with the pod. Always set `sizeLimit`; unbounded it drives the node to DiskPressure and evicts your own pod (`resources.md`). `medium: Memory` makes it tmpfs, charged to the pod's memory limit.
- Generic ephemeral volumes — a PVC template inline in the pod spec, giving scratch space with a real StorageClass, deleted with the pod. The right answer for "big scratch disk, no persistence".
- `local` PVs — real disks with node affinity baked into the PV. Fast and cheap; the pod can never move, so a node failure is a data outage. Only for systems that replicate at the application layer.
- `hostPath` — writes to the node filesystem, no isolation, no scheduling awareness. In production it is a finding (`security.md`), not a design.
- `projected` volumes combine ConfigMaps, Secrets, the downward API, and bound ServiceAccount tokens into one directory — the modern way to hand a pod its identity and config together (`rbac.md`).

## Permissions and Slow Mounts

- `fsGroup` in the pod securityContext makes the kubelet chown the volume recursively at mount time. On a volume with millions of files this adds minutes to pod startup and the pod just looks stuck in `ContainerCreating`.
- `fsGroupChangePolicy: OnRootMismatch` performs the walk only when the top-level ownership is wrong — the fix for that exact stall.
- Permission denied on a mounted volume is a numeric UID mismatch, not a Kubernetes bug: match `runAsUser`/`fsGroup` to what wrote the data. `chmod 777` in an initContainer is the finding, not the fix.
- SELinux-enforcing nodes (OpenShift, RHEL) need matching labels; the CSI driver usually handles it, but a hand-made PV will not mount.

## PVC and Volume Triage

| Symptom | Cause | Move |
|---|---|---|
| PVC Pending, no events | No default StorageClass | `kubectl get sc` — look for the `(default)` marker |
| PVC Pending, `waiting for first consumer` | Normal with WaitForFirstConsumer | Fix why the pod is unschedulable (`scheduling.md`) |
| PVC Pending, provisioner error | Quota, zone capacity, or driver credentials | CSI controller logs in `kube-system` |
| Pod stuck `ContainerCreating` | Attach or mount failure | `describe pod` Events: `FailedAttachVolume`, `FailedMount`, timeout values included |
| `Multi-Attach error` | Volume still attached to a dead or slow node | Confirm the node, then `nodes.md` |
| PVC stuck Terminating | `kubernetes.io/pvc-protection` finalizer: a pod still references it | Delete the consuming pods; the finalizer clears itself |
| Volume mounts empty | Wrong `subPath`, or a fresh volume where data was expected | `kubectl exec ... ls -la` on the mount before blaming the app |
| Writes fail, pod Ready | Volume full | `kubelet_volume_stats_available_bytes`; expand |
| Data gone after a rollout | The workload used emptyDir, or the PVC was recreated | Check whether the PVC still exists and its `Bound` age |

Two things here outlive the session. Any volume whose reclaim policy, access mode, or snapshot schedule was decided deliberately — especially a `Retain` on data nobody can re-derive — goes into the workload's row in `## Workloads` in `~/Clawic/data/k8s/memory.md`; a future engineer deleting that PVC needs to find the decision before the incident, not after. And the orphaned-PVC sweep (`kubectl get pvc -A` matched against existing workloads, per `stateful.md`) is a recurring job: put it in the `## Due` table with its date, and anything found and knowingly kept in `## Known Gaps`.
