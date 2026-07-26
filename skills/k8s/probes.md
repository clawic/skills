# Probes and Lifecycle — Health That Helps Instead of Hurting

Three probes, three questions. Liveness: "is this process wedged beyond self-recovery?" Readiness: "should traffic go here right now?" Startup: "has it finished booting yet?" Answering the wrong question with the wrong probe is the most common self-inflicted outage in Kubernetes.

## Defaults You Are Silently Accepting

| Field | Default | Why it bites |
|---|---|---|
| `initialDelaySeconds` | 0 | Probing starts immediately; a slow boot fails liveness before the app exists |
| `periodSeconds` | 10 | Detection latency is `periodSeconds × failureThreshold` (SKILL.md rule 3) |
| `timeoutSeconds` | 1 | One GC pause or load spike reads as a failure |
| `failureThreshold` | 3 | 3 for liveness is reasonable; 30 is normal for startup |
| `successThreshold` | 1 | Must be 1 for liveness and startup; only readiness may raise it |
| `terminationGracePeriodSeconds` (probe-level) | inherits the pod's | A liveness kill can use a shorter grace than a normal delete — useful for wedged processes |

The kubelet on the node runs every probe. Probes never traverse the cluster network, so NetworkPolicy cannot block them and a probe passing proves nothing about pod-to-pod reachability (`networking.md`).

## Choosing the Probe Mechanism

- `httpGet` — default choice. Any 200-399 counts as success. `host` defaults to the pod IP; an HTTPS endpoint needs `scheme: HTTPS` or the handshake failure reads as a health failure. Certificates are not verified, so self-signed is fine here.
- `tcpSocket` — proves a listener accepts connections and nothing else. An app that accepts and then hangs stays "healthy" forever. Acceptable for readiness on protocols without an HTTP surface, never a good liveness check.
- `exec` — forks a process every period, on every replica. At 1000 pods × 10s that is 100 forks/second of node overhead, and a hung exec probe holds a slot until its timeout. Prefer HTTP; keep exec for the cases where nothing else can express it.
- `grpc` — native gRPC health checking (kubelet >=1.24, GA 1.27) removes the grpc-health-probe binary from the image. Verify the field exists on the target cluster before relying on it.

## Liveness Discipline

- The probe handler must read in-process state only: event loop responsive, worker threads alive, no deadlock flag set. Zero dependencies, zero network calls.
- A liveness probe that touches the database converts a database outage into a fleet-wide restart storm on top of the outage — restarting every pod at once is the worst possible response to a dependency being down.
- If restarting the process cannot plausibly fix the condition, liveness must not test it. That single sentence eliminates most bad liveness probes.
- Liveness failure → the kubelet kills the container per `restartPolicy`, increments `restartCount`, and the event reads `Killing container ... failed liveness probe`. Exit 137 with `OOMKilled: false` is usually this (`debug.md`).
- Slow-start protection is startupProbe's job, not `initialDelaySeconds`: a fixed delay is a number you invented, and it is wrong on the day the database is slow.

## Readiness Discipline

- Readiness owns dependencies: database unreachable → fail readiness → the pod leaves the EndpointSlice → traffic stops → the pod recovers with no restart when the dependency returns.
- The danger case: every replica sharing one dependency fails readiness at once and the Service has zero endpoints, turning a degraded service into a total outage. For read-mostly services, prefer serving degraded over serving nothing — fail readiness only for dependencies without which every request is wrong.
- Readiness is also your load-shedding valve: a queue-depth threshold that fails readiness removes the pod from rotation until it catches up, which is gentler than timeouts at the client.
- Readiness failures do not restart anything and never touch `restartCount`. A pod stuck NotReady for an hour with 0 restarts is a readiness bug, not a crash.
- `readinessGates` extend readiness with external conditions (a cloud load balancer confirming target registration). Without them, a pod is "ready" before the LB has actually started sending it traffic — the source of 502s during otherwise clean rollouts (`ingress.md`).

## Startup Probes

- While a startupProbe is running, liveness and readiness are suppressed entirely. It exists so boot budget and steady-state budget can differ by an order of magnitude.
- Boot budget = `failureThreshold × periodSeconds`. 30 × 10s = 300s (SKILL.md rule 2). Size it to the worst boot you have ever seen — a cold cache, a slow migration, a throttled CPU limit — not the median.
- Cost of over-sizing: a genuinely broken container takes the full budget before its first restart. Cost of under-sizing: an infinite crash loop that looks like an app bug. Over-size.
- The startup probe can be the same endpoint as readiness with a laxer threshold; it does not need its own handler.

## Lifecycle Hooks and Shutdown

Ordering on delete, and the trap inside it:

1. The pod is marked Terminating; `deletionTimestamp` is set.
2. **Concurrently**: endpoint removal propagates through EndpointSlices to every kube-proxy and ingress controller, AND the `preStop` hook runs, AND after preStop the container gets SIGTERM.
3. When the grace period expires, SIGKILL. `terminationGracePeriodSeconds` (default 30s) covers preStop *plus* the SIGTERM window — it is not additive.

Because step 2 is concurrent, a pod that exits immediately on SIGTERM drops in-flight requests that were routed microseconds earlier. The fix is a `preStop` sleep of 5-10s (SKILL.md rule 5): keep serving, stop being routed, then shut down.

- The built-in `lifecycle.preStop.sleep` action avoids needing a shell in the image (distroless has none); on older clusters use `exec: ["sh","-c","sleep 10"]` and verify the binary exists.
- `postStart` runs concurrently with the entrypoint with no ordering guarantee — it cannot be used to prepare state the app needs at startup. A failing postStart kills the container.
- Budget arithmetic: grace period must exceed `preStop sleep + longest in-flight request + shutdown work`. A 10s sleep with a 30s grace leaves 20s for draining; a 60s batch job's grace period must be sized from the batch, not copied from the web service.
- Native sidecars (initContainers with `restartPolicy: Always`) shut down after the app containers, which finally makes log shippers and proxies drain in the right order. Verify support on the cluster before depending on the ordering.

## Probe Failure Triage

| Observation | Cause | Fix |
|---|---|---|
| Probe fails only under load | `timeoutSeconds: 1` versus a busy event loop or GC pause | 2-5s timeout; check CPU throttling first (`resources.md`) |
| Probe fails on every deploy, then settles | Boot slower than `initialDelaySeconds` | startupProbe |
| `connection refused` in the event | App binds `127.0.0.1`, or the wrong `containerPort` | Bind `0.0.0.0`; probe the port the process actually listens on |
| HTTP 401/403 in the probe event | Auth middleware in front of the health endpoint | Exempt the health path, or use a port the middleware does not cover |
| All replicas unready at once | Shared dependency in the readiness handler | Reconsider what readiness owns |
| Restart storm during a dependency outage | Liveness touching that dependency | In-process only |
| Pod ready, traffic still 502 | LB registration lag | `readinessGates`, or `minReadySeconds` (`rollouts.md`) |
| Probe passes, users see errors | Health endpoint returns 200 unconditionally | Make it assert something that can fail |

The two numbers this file produces are worth more than the manifest they end up in: the **boot budget** that actually covers a cold start, and the **real drain time** a graceful shutdown needs. Both come from watching a live rollout, and both are guessed wrong by every later engineer who did not. Write them into the workload's row in `## Workloads` in `~/Clawic/data/k8s/memory.md` the moment they are measured (Core Rule 10).
