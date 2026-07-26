# Debugging — Symptom to Cause in Minutes

Work symptom-first. Each chain is ordered by probability, and every step is a check that eliminates a branch, not a guess. If two steps in a row produce nothing, you are in the wrong chain — go back to the Status Decoder in SKILL.md.

**Read `## Incident History` in `~/Clawic/data/k8s/memory.md` before starting** (and whatever `## Boxes` points to for it). Most production failures are a shape that has happened before; if the row names a runbook in `artifacts/`, open that instead of re-deriving the chain.

## CrashLoopBackOff

1. `kubectl logs <p> -p --tail=200`. Empty previous logs means the process died before writing anything: bad entrypoint, missing binary, or an arch mismatch (exit 126/127 → `docker` skill).
2. `kubectl get pod <p> -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'` — reason, exit code, and the timestamps that tell you whether it ran 200ms or 200s. Sub-second lifetime = startup config; minutes = a real workload failure.
3. Exit 137 → is `OOMKilled: true`? If yes, `resources.md`. If no, something SIGKILLed it: liveness probe (`describe` shows `Killing container ... failed liveness probe`) or grace-period expiry (`probes.md`, `rollouts.md`).
4. Config suspicion: `kubectl exec` is impossible on a crashlooping pod, so clone it — `kubectl debug <p> --copy-to=<p>-dbg --set-image=app=busybox -- sleep 1d`, then read the mounted ConfigMaps, Secrets, and env the app would have received.
5. Restart cadence itself is evidence: backoff starts at 10s and doubles to a 5m cap, resetting after 10m of clean running. A pod that restarts every 5m has been failing for a long time; a pod at 10s intervals just started.
6. Crashes only in one namespace or on one node → jump to the differential chains below.

## Pod Pending

1. `kubectl describe pod <p>` — the FailedScheduling event names the predicate for every node: `Insufficient cpu`, `node(s) had untolerated taint`, `didn't match Pod's node affinity/selector`, `pod has unbound immediate PersistentVolumeClaims`.
2. Insufficient anything → compare against allocatable, not capacity: `kubectl describe node <n>` `Allocated resources`. Full arithmetic and the eviction-reserve gap: `scheduling.md`.
3. Unbound PVC → `kubectl describe pvc`: no default StorageClass, `WaitForFirstConsumer` (a chicken-and-egg with an unschedulable pod), or a zonal disk in a zone with no room (`storage.md`).
4. No events at all → no scheduler saw it: check the `schedulerName` field, a namespace ResourceQuota rejecting the pod at admission, or a paused/suspended parent object.
5. Only some replicas Pending → anti-affinity or topology spread cannot satisfy the constraint with the nodes available (`scheduling.md`).

## OOMKilled

1. Confirm attribution: `Last State: Terminated, Reason: OOMKilled`. The kernel kills the largest offender inside the cgroup — if a child process died and PID 1 exited on its own, the reason is `Error`, not `OOMKilled`, and the app logs hold the truth.
2. Measure before raising: `kubectl top pod <p> --containers` under load, or `container_memory_working_set_bytes` from Prometheus. Working set (not RSS) is what the kernel compares against the limit, and it includes dirty page cache.
3. Runtime caps come before container limits: a JVM defaults to `MaxRAMPercentage=25` (a 4Gi limit yields a 1Gi heap), Node needs `--max-old-space-size`, Go needs `GOMEMLIMIT`. Full table: `resources.md`.
4. Sawtooth kills over hours = a leak; kills within seconds of startup = a runtime cap sized off the host rather than the cgroup; kills only during batch windows = concurrency, not size.
5. The pod that OOMs may not be the guilty one: on a node under memory pressure, eviction picks by QoS class. Check `kubectl get events --field-selector reason=Evicted` before rewriting the victim's manifest (`nodes.md`).

## ImagePullBackOff

| Events line | Real cause | Fix |
|---|---|---|
| `401 Unauthorized` | Missing or wrong `imagePullSecret`, or the secret is in another namespace | Secrets are namespaced; create it where the pod runs, or attach it to the ServiceAccount |
| `manifest unknown` / `404` | Tag typo, tag deleted, or wrong registry host | `kubectl get pod -o jsonpath='{..image}'` and compare character by character |
| `toomanyrequests` | Registry rate limit (anonymous pulls are throttled per source IP) | Authenticate pulls or mirror your base images |
| `no match for platform` | amd64-only image on arm64 nodes (or the reverse) | Multi-arch build (`docker` skill) or a nodeSelector on `kubernetes.io/arch` |
| `x509: certificate signed by unknown authority` | Private registry CA not trusted by the node's container runtime | Node-level trust store, not a pod-level fix |
| Pull works on some nodes only | Stale cached credentials or a broken node-level mirror | Compare with `crictl images` on both nodes (`nodes.md`) |

## Service Returns Nothing

1. `kubectl get endpointslices -l kubernetes.io/service-name=<svc> -o wide`. Empty = the Service is wired to nothing; nothing else produces an empty slice. (`kubectl get endpoints` still works but the Endpoints API is deprecated as of 1.33 — read slices.)
2. Empty and pods exist → label mismatch. Compare `kubectl get svc <svc> -o jsonpath='{.spec.selector}'` with `kubectl get pod --show-labels`. A single trailing character kills it silently.
3. Endpoints present but marked not-ready → readiness is failing; the Service is correct and the app is not (`probes.md`).
4. Endpoints ready but requests fail → port mapping: Service `port` vs `targetPort` vs what the process actually binds (`kubectl exec <p> -- ss -ltn`). Binding `127.0.0.1` inside a pod is unreachable from anywhere.
5. Works via `port-forward` but not via the Ingress → the fault is above the Service (`ingress.md`); works via ClusterIP from another pod but not from outside → LB or externalTrafficPolicy (`networking.md`).

## "Works in Namespace A, Fails in B"

Check in this order; each is a one-minute test:

| Difference | Check |
|---|---|
| ResourceQuota | `kubectl get resourcequota -n <ns>` — a quota namespace rejects pods with no requests |
| LimitRange | `kubectl get limitrange -n <ns> -o yaml` — silently injects different defaults |
| NetworkPolicy | `kubectl get netpol -n <ns>` — one policy flips selected pods to default-deny |
| PSA labels | `kubectl get ns <ns> --show-labels` — `enforce: restricted` rejects the same pod spec |
| ServiceAccount RBAC | `kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>` |
| Secret/ConfigMap presence | Both are namespaced; a copied Deployment references objects that do not exist there |
| Node pool via nodeSelector | The namespace's default node affinity (or a mutating webhook) sends pods elsewhere |

## Intermittent Failures Only Under Load

1. CPU throttling: `container_cpu_cfs_throttled_periods_total` rising while average CPU sits low. Fix the quota, not the replica count (`resources.md`).
2. Conntrack exhaustion on the node: `nf_conntrack: table full, dropping packet` in kernel logs; symptoms are random resets across all pods on that node (`nodes.md`).
3. DNS: 5s stalls with a clean application stack are the ndots + UDP race signature (`dns.md`).
4. Rolling restarts mid-test: `kubectl get pod -w` during the run — you may be measuring three different pods.
5. One replica bad, others fine: `kubectl get pods -o wide` and correlate with the node; a single sick node produces "intermittent" everywhere.

## Everything Started Failing At Once

1. `kubectl get --raw='/readyz?verbose'` — is the control plane itself healthy?
2. Every create failing with a webhook error → an admission webhook with `failurePolicy: Fail` whose backend is down. This is the highest-blast-radius failure in Kubernetes (`operators.md`).
3. Only new pods failing → the CNI or the image registry; running pods keep working, so nobody notices until a rollout.
4. Certificates: an expired kubelet or API client certificate produces authentication errors across the fleet at the same instant (`production.md`).
5. A cluster-wide change lands in Git before it lands in your memory: `kubectl get events -A --sort-by=.lastTimestamp` plus the GitOps controller's history (`manifests.md`).

## When You Are Truly Stuck

Reduce to the smallest object that still fails: `kubectl run tmp --image=<same-image> --restart=Never --command -- sleep 1d` in the same namespace, then re-add pieces one at a time — the ServiceAccount, then the volumes, then the securityContext, then the network policy. The addition that breaks it names the subsystem and the file to open next.

## Before You Close The Session

Once the cause is named, write it down — the diagnosis is the expensive part and it evaporates by Monday:

- The shape, its root cause, and what actually fixed it → a row in `## Incident History` in `~/Clawic/data/k8s/memory.md`. If the shape is already there, update `Last seen` and sharpen the fix rather than adding a second row.
- A shape seen three times, or one whose fix is a procedure rather than a sentence → a runbook at `~/Clawic/data/k8s/artifacts/runbook-<kebab>.md`, opening with the symptom that should make someone read it, with every secret replaced by its pointer — plus its `## Boxes` line in the same turn.
- Anything found and deliberately not fixed → `## Known Gaps`, with the date and why.

