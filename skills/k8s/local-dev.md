# Local Clusters and Context Safety

Two things belong together here: the cluster on your laptop behaves differently from the real one in specific, predictable ways, and the fastest way to cause a production incident is to run a correct command against the wrong context.

## Context Safety

```bash
kubectl config current-context                      # before every destructive command
kubectl config get-contexts                         # what is even in this kubeconfig
kubectl --context=kind-dev apply -f .               # explicit beats remembering
```

- The failure shape is always the same: a `delete`, `scale --replicas=0`, or `apply` that was correct for dev, executed against prod because a previous command switched the context. Nothing in kubectl warns you.
- Make the current context visible in the shell prompt. A prompt segment showing context and namespace is the cheapest incident prevention in this file.
- Separate kubeconfig files per environment (`KUBECONFIG=~/.kube/prod.yaml`) beat one file with many contexts: switching becomes an explicit act, and a stale context cannot follow you into a new terminal.
- Give production contexts names that read as a warning (`prod-eu-DANGER`), and where the platform supports it, use read-only credentials by default with a separate escalation path.
- `--dry-run=server` and `kubectl diff` are also context-safety tools: both show you which cluster answered before anything changes (`manifests.md`).
- Automation never inherits an ambient context: CI and scripts pass `--context` or a dedicated kubeconfig explicitly, and `destructive_confirm` governs whether a destructive command is proposed or executed.

## What A Local Cluster Does Not Have

| Missing | Symptom | Local workaround |
|---|---|---|
| Cloud load balancer controller | `LoadBalancer` Service stuck `Pending` forever | `port-forward`, NodePort, or MetalLB / the distribution's built-in LB |
| Real StorageClass | PVCs bind to a hostPath-backed provisioner with no zones, no snapshots, no expansion | Test storage behavior on a real cluster; local proves only that the YAML applies |
| Multiple nodes and zones | Anti-affinity and topology spread silently do nothing, or make pods unschedulable | Multi-node kind/k3d config when testing spread (`scheduling.md`) |
| Node capacity | Everything OOMs, or nothing does, depending on the laptop | Do not size requests from local observation (`resources.md`) |
| Cloud identity (IRSA, Workload Identity) | Auth works locally with your credentials, fails in-cluster | Test the ServiceAccount path in a real cluster early (`rbac.md`) |
| Ingress controller by default | 404 or connection refused on every Ingress | Install the controller explicitly; k3s ships Traefik, kind ships nothing |

The rule that follows: local clusters validate **manifests and application wiring**, never capacity, storage durability, identity, or failure behavior. Treating a green local run as production evidence is how "it worked in dev" happens on infrastructure as much as on code.

## Distribution Differences That Change Instructions

- **kind** — nodes are containers. Images must be loaded into the cluster (`kind load docker-image app:dev`) or pulled from a registry; a locally built image is invisible otherwise, and the pod sits in `ErrImagePull` while the image plainly exists on your machine.
- **minikube** — a VM or container runtime with its own daemon. Either point your build at it (`eval $(minikube docker-env)`) or load explicitly; `minikube tunnel` provides LoadBalancer addresses.
- **k3s / k3d** — real distribution, small footprint, Traefik and a local-path provisioner preinstalled. Closest to a real cluster of the three, and its opinionated defaults are also what differs from your production distribution.
- **Docker Desktop Kubernetes** — shares the local image store, so no loading step; single node, no realistic networking or storage.

Whatever the flavor, record it as `cluster_flavor` in `~/Clawic/data/k8s/config.yaml` once the user reveals it: it changes the LoadBalancer, StorageClass, and Ingress advice in every other file.

## The Inner Loop

- Fast iteration is a rebuild-and-replace loop, not a `kubectl edit` loop: image tag by content hash, apply, watch. Skaffold and Tilt automate exactly that; hand-rolling it with `rollout restart` on a mutable tag reintroduces the "which code is running" problem (SKILL.md rule 8).
- `imagePullPolicy: IfNotPresent` with a locally loaded image is the combination that works in kind and minikube. `Always` on a tag that exists only locally fails on every pod start.
- Remote debugging: `kubectl port-forward` for one service, or a traffic-interception tool (Telepresence-class) when the service needs to receive cluster traffic while running on your machine. Both are development conveniences with real security implications on a shared cluster — never point one at production.
- Test manifests without a cluster at all: `kubectl apply --dry-run=server` against a scratch cluster catches admission and defaulting; schema validation in CI catches deprecated APIs (`manifests.md`).
- Keep one throwaway namespace per developer with a ResourceQuota rather than one cluster per developer where the platform allows it — the quota teaches the same lesson production will (`resources.md`).

## Reproducing A Production Bug Locally

Ordered by how often it works:

1. Same image digest, same manifests, real config values replaced with equivalents — most application bugs reproduce here.
2. Same resource limits, including the CPU limit: throttling-shaped bugs disappear on an unlimited laptop (`resources.md`).
3. Multi-node kind config when the bug involves scheduling, anti-affinity, RWO volumes across nodes, or `externalTrafficPolicy`.
4. Do not attempt: cloud LB behavior, cloud IAM, CSI driver semantics, node pressure eviction, or anything about etcd. Those reproduce only on a real cluster — use a staging namespace with a copy of the workload instead.

Two things learned here are declarations, so they go straight to `~/Clawic/data/k8s/config.yaml` the moment the user reveals them: the local distribution as `cluster_flavor`, and the kubeconfig context that must never be treated casually as `prod_context`. The second one is the cheapest incident prevention in this skill — once it is recorded, every state-changing command against that context is proposed with its blast radius instead of executed. Local clusters and scratch contexts do not belong in the shared `servers.md` inventory; only real clusters and pet machines do.
