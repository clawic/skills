---
name: k8s
slug: k8s
version: 1.0.2
description: >-
  Kubernetes troubleshooting and manifest judgment: pod crashes, probes, resources and QoS,
  rollouts, DNS, storage, RBAC. Use when debugging clusters, reviewing manifests, or tuning
  workloads.
homepage: https://clawic.com/skills/k8s
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
metadata:
  clawdbot:
    emoji: "☸"
    displayName: Kubernetes
---

## When To Use

- A pod is Pending, CrashLoopBackOff, OOMKilled, ImagePullBackOff, or stuck Terminating
- A Service or Ingress receives no traffic, or DNS inside the cluster is flaky
- Writing or reviewing manifests: probes, resources, rollout strategy, security
- A rollout is stuck, flapping, or silently serving a broken version
- Locking down RBAC, ServiceAccounts, or Secrets handling
- Not for building container images themselves — that is `docker`

## Quick Reference

| Symptom | First move |
|---|---|
| Pod Pending | `kubectl describe pod` Events: insufficient CPU/memory (requests vs node allocatable), unbound PVC, taint without toleration, nodeSelector with no match |
| CrashLoopBackOff | `kubectl logs -p` (previous container). Exit 137 = SIGKILL (OOM or liveness kill), 143 = SIGTERM, 1 = app error. Backoff doubles 10s → 5m cap, resets after 10m of clean running |
| OOMKilled | `describe pod` → Last State: OOMKilled. Raise memory limit or fix the leak; for JVMs check `MaxRAMPercentage` first (→ Resources and QoS) |
| Service gets no traffic | `kubectl get endpoints <svc>` — empty means selector/label mismatch or every pod unready. Check `port` vs `targetPort` next |
| Rollout stuck | `kubectl rollout status`; `describe rs` on the new ReplicaSet for quota or image errors. `progressDeadlineSeconds` (default 600s) only flags Progressing=False — it never auto-rolls back |
| Pod stuck Terminating | Finalizers: `kubectl get pod -o yaml` → `metadata.finalizers`. Fix the controller that owns the finalizer; force-delete is last resort (→ Traps) |
| Intermittent DNS failures | `ndots:5` + UDP conntrack races. Use FQDNs with trailing dot (`db.prod.svc.cluster.local.`) or set `dnsConfig` `ndots:1`; NodeLocal DNSCache at cluster level |
| Works in ns A, fails in ns B | Diff ResourceQuota, LimitRange, NetworkPolicy — a quota namespace rejects pods without requests set |
| HPA shows `<unknown>` | A container is missing resource requests — utilization = usage/requests has no denominator |
| Node NotReady | `kubectl describe node` conditions: MemoryPressure/DiskPressure/PIDPressure; check kubelet on the node |
| Anything else | `kubectl get events -n <ns> --sort-by=.lastTimestamp` then `describe` the object it points at |

## Core Rules

1. Memory requests = limits on every production container. Overcommitted memory means the node OOM-kills someone when neighbors burst — and the victim is chosen by QoS class, not by who leaked. CPU: always set requests; limits are contested (→ Where Experts Disagree).
2. Never guess `initialDelaySeconds` — give slow starters a startupProbe. Boot budget = failureThreshold × periodSeconds: 30 × 10s = 300s covers a slow JVM while liveness stays tight for steady state.
3. Detection latency = periodSeconds × failureThreshold. Defaults (10s × 3) mean up to 30s of traffic to a dead pod before endpoint removal — tune both factors, not just one.
4. Readiness gates traffic, liveness restarts, neither reschedules. A restart happens in place on the same node; only eviction or deletion moves a pod. Restarting a pod cannot fix a node problem.
5. Shutdown contract: app handles SIGTERM + `preStop` sleep 5–10s + `terminationGracePeriodSeconds` (default 30s) > real drain time. Endpoint removal propagates async, so pods keep receiving traffic after SIGTERM — the sleep covers that window.
6. Diagnose in order: `describe` (events) → `logs -p` → `get endpoints` → `exec`/`debug`. Events expire after 1h by default (`--event-ttl`) — capture them before anything else in an incident.
7. Requests drive both scheduling and HPA math. HPA: desired = ceil(current × usage/target); 3 replicas at 90% usage with a 60% target → ceil(3 × 90/60) = 5. Undersized requests inflate utilization and overscale.

## Resources and QoS

- Requests are scheduling-time only — never enforced at runtime. Limits are enforced: memory by cgroup OOM kill, CPU by CFS quota per 100ms period.
- CPU throttling triggers below 100% average usage: a bursty app can exhaust its quota inside one 100ms window and stall until the next. Symptom: p99 latency spikes with low average CPU. Check `container_cpu_cfs_throttled_periods_total` before adding replicas.
- QoS classes decide eviction order under node pressure: BestEffort (no requests) dies first, then Burstable pods exceeding requests, Guaranteed (requests = limits for both CPU and memory) last.
- Units: 1 CPU = 1000m. Memory `Mi` = 2^20, `M` = 10^6 — and a lone `m` suffix on memory means millibytes: `memory: 512m` requests 0.512 bytes and the pod OOMs instantly (→ Traps).
- JVM in a container: default `MaxRAMPercentage` is 25 — a 4Gi limit yields a 1Gi heap. Set 50–75 depending on non-heap footprint; the gap between heap and limit must hold metaspace, threads, and direct buffers.
- LimitRange injects default requests/limits into pods that omit them; ResourceQuota rejects such pods outright. Same manifest, different namespace, different outcome — check both before blaming the manifest.

## Probes

- Liveness answers exactly one question: "is this process deadlocked beyond self-recovery?" It must check in-process state only. A liveness probe touching a database converts a DB outage into a fleet-wide restart storm.
- Readiness owns dependencies: DB down → fail readiness → pod leaves endpoints → recovers without a restart when the DB returns.
- Default `timeoutSeconds` is 1. One GC pause or load spike → probe timeout → liveness failure → restart under the exact load that needed the pod alive. Set 2–5s on any probe doing real work.
- startupProbe disables liveness and readiness until first success — it exists so boot time and steady-state health can have different budgets (→ Core Rules 2).
- HTTPS endpoint needs `scheme: HTTPS` on an httpGet probe; default is HTTP and the handshake failure reads as a probe failure.
- A pod that passes readiness once then crashes still advanced the rollout if `minReadySeconds` is 0 (the default) — set 5–30s so "ready" means "survived" (→ Rollouts and Shutdown).

## Rollouts and Shutdown

- RollingUpdate defaults: maxSurge 25%, maxUnavailable 25%. For latency-sensitive fleets set `maxUnavailable: 0` and pay for the surge capacity.
- `minReadySeconds` is the cheapest canary you can buy: a crash-looping image with a passing first readiness check will otherwise replace the whole fleet before the first restart.
- `kubectl rollout undo` needs the old ReplicaSet — `revisionHistoryLimit` (default 10) keeps it. `kubectl rollout restart` is the correct way to pick up ConfigMap/Secret changes delivered via env vars.
- PodDisruptionBudgets protect against voluntary disruptions only (drains, upgrades) — never against crashes or OOM. A PDB with `maxUnavailable: 0` on a 1-replica deployment blocks node drains forever.
- Shell-form `ENTRYPOINT ["sh", "-c", "..."]` makes sh PID 1: SIGTERM is not forwarded, the app never shuts down gracefully, and every stop waits the full grace period then exits 137 without an OOM. Use exec form or `tini`.
- Deployment `spec.selector` is immutable, and changing pod template labels without it fails validation. Plan the label scheme (`app`, `version`, `environment`) before first apply, not after.

## Networking and DNS

- Three ports, three meanings: Service `port` (what clients dial), `targetPort` (container's listening port), `containerPort` (documentation only — traffic flows without it).
- ClusterIP is virtual — kube-proxy DNAT, no interface, no ping. Test with `curl`/`nc` to the port, never `ping`.
- `kubectl get endpoints` is the single source of truth for "is my Service wired to anything". Empty endpoints = selector mismatch or zero ready pods; nothing else produces it.
- `externalTrafficPolicy: Local` preserves client source IP and skips the second hop, but nodes without a ready pod fail the LB health check — expect uneven load with few replicas.
- Headless Service (`clusterIP: None`) returns pod IPs from DNS instead of a VIP — required for StatefulSet per-pod DNS (`pod-0.svc.ns.svc.cluster.local`).
- NetworkPolicy is additive allow-listing: the moment any policy selects a pod, that pod flips to default-deny for that direction. Ship the DNS egress rule (UDP/TCP 53) in the same change or everything breaks at name resolution.
- NodePort range is 30000–32767; fine for dev, one-hop-short for production — put a LoadBalancer or Ingress in front.
- Ingress `pathType: Prefix` matches path segments (`/api` matches `/api/v1`, not `/apiv1`); `ImplementationSpecific` differs per controller — pin Prefix/Exact in anything portable.

## Storage and Config

- ReadWriteOnce is per node, not per pod: two pods on the same node can both mount an RWO volume — which is why the bug only appears after a reschedule. True multi-node write needs RWX (NFS/CephFS-class).
- `storageClassName: ""` and omitting it are different: `""` disables dynamic provisioning (static PVs only); omitted uses the cluster default class.
- Dynamically provisioned PVs default to `reclaimPolicy: Delete` — deleting the PVC deletes the data. Patch to `Retain` on anything you cannot re-derive.
- StatefulSet scale-down and deletion keep PVCs (unless `persistentVolumeClaimRetentionPolicy` says otherwise) — scale-up resurrects old data, which is either the feature you wanted or a stale-state bug.
- Volumes grow (`allowVolumeExpansion: true` on the class) but never shrink — size generously once instead of migrating later.
- ConfigMap delivery modes have different update semantics: env vars never update (restart required); volume mounts update within ~2min (kubelet sync period + cache); `subPath` mounts never update (→ Traps).
- `immutable: true` on ConfigMaps/Secrets prevents accidental live edits and stops kubelet watches (a real win at scale); updates then flow through new-name + rollout, which also gives you rollback.
- Secrets are base64, not encrypted. Threat model honestly: anyone who can create pods in the namespace can mount and read any Secret in it — RBAC on pod creation is Secret access. Enable etcd encryption at rest or use an external secrets manager.
- ConfigMaps and Secrets cap at 1MiB — anything bigger belongs in a volume or object store.

## RBAC

- One ServiceAccount per workload; set `automountServiceAccountToken: false` wherever the pod never calls the API — most workloads don't.
- Role/RoleBinding are namespaced; ClusterRole/ClusterRoleBinding are not. Useful asymmetry: a RoleBinding can reference a ClusterRole to grant it in one namespace only — define once, bind narrowly.
- Privilege escalation paths auditors miss: `create pods` in a namespace ≈ read every Secret in it (mount them) and act as any SA in it (mount its token). The verbs `escalate`, `bind`, and `impersonate` are admin-equivalent. Wildcard verbs or resources in a Role are a finding, not a convenience.
- Verify instead of reasoning from YAML: `kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>` shows the effective permission set; `kubectl auth can-i <verb> <resource> --as=...` answers point questions.

## Debugging

- `kubectl describe pod` first — Events explain scheduling failures, probe failures, image errors, OOM kills. Then `logs -f`, and `logs -p` after any crash (the current container's logs are empty precisely because it just restarted).
- Distroless or crashed-too-fast images: `kubectl debug -it <pod> --image=busybox --target=<container>` attaches an ephemeral container sharing the process namespace (kubectl >=1.25). `kubectl debug node/<node> -it --image=busybox` gets a node shell without SSH.
- `kubectl exec` proves the pod's own view: `nslookup <svc>` for DNS, `nc -zv <svc> <port>` for reachability — cluster networking bugs are invisible from your laptop.
- `kubectl explain deployment.spec.strategy` beats searching docs for field semantics and matches the cluster's actual API version.
- `kubectl apply` for everything declarative; `create` fails on existing objects. Mixing `edit`/imperative patches with GitOps applies produces drift where the next apply silently reverts a hotfix — pick one writer per object.
- Image pins: `latest` + `imagePullPolicy: IfNotPresent` means each node runs whatever it cached — different code per node, undebuggable. Pin version tags (or digests for supply-chain rigor); `Always` + `latest` is only fit for dev.

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| ConfigMap mounted with `subPath` | subPath binds the file inode once; kubelet's symlink-swap update mechanism works only at directory level — updates never arrive | Mount the whole ConfigMap as a directory, or use env vars + `rollout restart` |
| `memory: 512m` | `m` = millibytes → limit of 0.512 bytes, instant OOM | `512Mi` |
| Liveness probe checks a dependency | Dependency outage → every pod fails liveness → cluster-wide restart storm on top of the outage | Liveness = in-process only; readiness owns dependencies |
| Keeping `timeoutSeconds: 1` | One GC pause or load spike kills the pod under the load that needed it | 2–5s timeout on real-work probes |
| First NetworkPolicy in a namespace | Selected pods flip to default-deny; DNS breaks first, everything else follows | Include DNS egress (port 53) in the same policy change |
| Deleting a PVC on a dynamic PV | Default `reclaimPolicy: Delete` removes the data with the claim | `Retain` on data you can't re-derive; verify before deleting |
| Force-deleting a Terminating pod | API forgets the pod while kubelet may still run the container — StatefulSets can split-brain on the "freed" identity | Remove the stuck finalizer or fix the node; force only when the node is confirmed dead |
| CronJob with defaults | `concurrencyPolicy: Allow` piles up overlapping runs when one hangs; >100 missed schedules with no `startingDeadlineSeconds` stops future scheduling entirely | Set `concurrencyPolicy: Forbid` (or `Replace`) + `startingDeadlineSeconds` |
| PDB stricter than replica count | `maxUnavailable: 0` (or minAvailable = replicas) blocks every node drain; cluster upgrades hang | Leave ≥1 pod of slack or accept the drain pause consciously |
| Debugging from `latest` logs | You can't know which code produced them (→ Debugging, image pins) | Pin tags; log image digest at startup |

## Where Experts Disagree

- CPU limits: one camp drops them entirely — CFS quota throttling adds tail latency even at low average usage, and requests already guarantee fair share under contention. The other camp keeps limits = requests for predictability and multi-tenant fairness. Frontier: trusted workloads on dedicated clusters → requests only; multi-tenant platforms, chargeback, or compliance regimes → limits on.
- Liveness probes: default-off camp argues most apps never deadlock without crashing, so liveness only adds restart storms and masks bugs; default-on camp wants a self-healing floor. Frontier: add liveness only when the process demonstrably hangs without exiting and the probe reads purely in-process state — otherwise readiness alone.

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/k8s (install if the user confirms):
- `docker` — building and hardening the images k8s runs
- `devops` — CI/CD pipelines that deploy to the cluster
- `incident-response` — coordinating the humans when the cluster page fires
- `linux` — node-level debugging under the kubelet

## Feedback

- If useful, star it: https://clawic.com/skills/k8s
- Latest version: https://clawic.com/skills/k8s

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/k8s.
