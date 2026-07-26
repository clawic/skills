# Nodes — Pressure, Drains, and the Machine Under the Pod

**Read `~/Clawic/data/servers/servers.md` and `## Clusters` in `~/Clawic/data/k8s/memory.md` first**: node classes, pool layout and who owns upgrades are usually already recorded, and rediscovering them costs the time you need for the incident.

When the pod is innocent, the node is the suspect. Node problems present as application problems on every pod that happens to be there, which is why "it only fails sometimes" so often correlates perfectly with one hostname.

## Heartbeats and the Failure Timeline

```
kubelet posts a Lease every ~10s
  → controller marks the node NotReady after the node-monitor grace period (~40s; some distributions 50s)
  → NoExecute taint applied
  → pods evicted after tolerationSeconds (default 300s, injected by admission)
```

Worst case is therefore about 5m40s from "machine died" to "pods rescheduled". Lower `tolerationSeconds` per workload if failover speed matters; below ~60s expect churn on every transient network blip (`scheduling.md`).

- A pod on a NotReady node is not moved by anything else: restarting a Deployment will not relocate it, and the StatefulSet controller deliberately will not recreate its ordinal (`stateful.md`).
- If the cloud instance is gone but the Node object remains, nothing progresses until the object is deleted. Managed clusters usually do this; a self-managed one may not.

## Node Conditions and Their Taints

| Condition | Threshold that sets it | Effect |
|---|---|---|
| `MemoryPressure` | `memory.available` below 100Mi (default hard eviction) | Taint; BestEffort pods evicted first |
| `DiskPressure` | `nodefs.available` <10%, `imagefs.available` <15%, or inodes <5% | Taint; image GC, then pod eviction |
| `PIDPressure` | Available PIDs below the threshold | Taint; new pods rejected |
| `NetworkUnavailable` | CNI has not configured the node network | Node unschedulable in practice |
| `Ready=False`/`Unknown` | Kubelet not reporting | NoExecute taint after the grace period |

- Eviction under memory pressure ranks by QoS, then by usage over requests (`resources.md`). The victim is frequently not the cause; find the cause with `kubectl top pods --all-namespaces --sort-by=memory` restricted to that node.
- DiskPressure has a specific order: the kubelet garbage-collects unused images first (default: start at 85% of image filesystem usage, free down to 80%), then evicts pods.
- Node-level OOM is different from container OOM: the kernel picks a victim by `oom_score_adj`, which is derived from QoS. `dmesg -T | grep -i oom` on the node names the process the kernel actually chose, which may be in a different pod than the one you were investigating (`debug.md`).

## Disk: What Actually Fills a Node

| Consumer | Check | Control |
|---|---|---|
| Container images | `crictl images` | Image GC thresholds; smaller images |
| Container logs | `/var/log/pods` | Kubelet rotation (`containerLogMaxSize`, `containerLogMaxFiles`) — set fleet-wide, not per app |
| emptyDir and writable layers | `du -sh /var/lib/kubelet/pods/*` | `sizeLimit` on emptyDir; `ephemeral-storage` limits (`resources.md`) |
| Orphaned volumes from force-deleted pods | `/var/lib/kubelet/pods` entries with no pod | Node restart or manual cleanup after verifying |

A node at 100% disk cannot pull images, cannot write logs, and often cannot be repaired through the kubelet — the same "cannot fix it through the thing that is broken" shape as a full Docker host (`docker` skill).

## Draining Without Breaking Things

```bash
kubectl cordon <node>                                   # stop new placements, nothing moves yet
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=10m
kubectl uncordon <node>                                 # after the work
```

- `drain` uses the eviction API, so it honors PDBs (`rollouts.md`). A drain that hangs is usually a PDB that cannot be satisfied — read the message, it names the budget.
- `--delete-emptydir-data` acknowledges data loss for emptyDir volumes. `--force` deletes bare pods that no controller will recreate: they are gone permanently, and that is exactly the flag people add to make a drain "work".
- `--disable-eviction` bypasses PDBs entirely by deleting pods. It exists for emergencies and turns your availability guarantee off cluster-wide for the duration.
- Cordon before you investigate a suspect node. It costs nothing, stops the problem spreading to new pods, and is trivially reversible.
- After an uncordon the node stays empty: the scheduler never rebalances (`scheduling.md`). Restart a workload deliberately if you want it spread again.

## Getting Onto the Node

```bash
kubectl debug node/<node> -it --image=busybox     # node filesystem at /host, host namespaces
chroot /host                                       # then the usual tools
crictl ps -a && crictl logs <container-id>         # container runtime view when kubectl cannot reach it
journalctl -u kubelet -n 200 --no-pager            # kubelet's own story
df -h /var/lib/kubelet /var/lib/containerd && dmesg -T | tail -50
```

- `kubectl debug node/` needs no SSH and works on managed nodes where SSH is disabled by policy.
- The kubelet log is the authority on mount failures, image pulls, probe execution, and eviction decisions — all the things whose events already expired.
- Certificate expiry shows up here first: an expired kubelet client certificate makes every node go NotReady at once, and the fix is CSR approval, not a reboot (`production.md`).

## Node Lifecycle Work

- **Upgrades**: cordon → drain → upgrade → uncordon, one node at a time, with PDBs enforcing your availability floor. Managed node pools do the same thing with surge nodes; the PDB is still what protects you.
- **Spot and preemptible nodes**: a termination handler watches the provider's notice (typically 30-120s), taints the node, and drains it. Without one, pods are killed with no grace period at all — no preStop, no drain (`probes.md`).
- **Node pools by workload class**: taint GPU, spot, and memory-heavy pools and tolerate deliberately, rather than relying on resource requests to sort placement (`scheduling.md`).
- **Zombie nodes**: an instance terminated outside Kubernetes leaves a Node object that keeps attracting rescheduling attempts. `kubectl delete node <name>` after confirming the instance is genuinely gone.
- **Clock skew**: containers share the node clock. NTP drift breaks TLS validation and token expiry in ways that read as authentication bugs.

## Node Triage Order

1. `kubectl get nodes -o wide` — one NotReady, or all of them? All at once means control plane, certificates, or network, not the machines.
2. `kubectl describe node <n>` — conditions first, then `Allocated resources`, then the events at the bottom.
3. `kubectl get pods -A --field-selector spec.nodeName=<n>` — what is on it, and what is unhealthy there.
4. `kubectl debug node/<n>` → kubelet journal, disk, dmesg. Most answers are in one of those three.
5. Network-shaped symptoms (random resets, DNS failures on that node only) → conntrack table fullness, MTU, and the kube-proxy and CNI pods on that node (`networking.md`).
6. Still unexplained → cordon, drain, and replace the node. On cloud infrastructure, replacing a sick node is nearly always cheaper than diagnosing it further; capture the evidence first if the failure is novel.

After any inventory pass or pool change, write it down: the cluster's row in `~/Clawic/data/servers/servers.md` (one row per cluster — node counts and instance classes live in its `Type` field, and cattle nodes never get rows of their own), and the pool detail, taints, and upgrade ownership in `## Clusters` in `~/Clawic/data/k8s/memory.md`. A node that was decommissioned gets its row deleted, not left behind; a pet node — a bare-metal box, a single-node cluster — keeps its own row. Formats and the scale cut: `memory-template.md`.
