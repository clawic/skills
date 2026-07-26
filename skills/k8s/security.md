# Security — Pod Security, Hardening, and Supply Chain

Threat model in one line: a container escape is root on the node, and root on a node is the credentials of every pod scheduled there. Everything below is about making that path longer.

**Read `## Known Gaps` in `~/Clawic/data/k8s/memory.md` before auditing anything.** Findings the user already saw and consciously accepted are not findings; re-raising them every quarter is how the real ones get ignored.

## Pod Security Admission

PodSecurityPolicy was removed in 1.25; the built-in replacement is namespace-label based:

```yaml
pod-security.kubernetes.io/enforce: baseline
pod-security.kubernetes.io/enforce-version: v1.30
pod-security.kubernetes.io/audit: restricted
pod-security.kubernetes.io/warn: restricted
```

- Three profiles: `privileged` (no restrictions), `baseline` (blocks the known escape vectors: privileged, hostPID/IPC/Network, hostPath, most capabilities), `restricted` (baseline plus runAsNonRoot, seccomp RuntimeDefault, drop ALL capabilities, no privilege escalation).
- Three modes matter together: run `enforce: baseline` with `audit`/`warn: restricted` and you get a working cluster today plus a list of exactly what blocks the upgrade to restricted tomorrow.
- Pin `enforce-version`: without it, a cluster upgrade can tighten the profile's definition and reject workloads that applied fine yesterday.
- Enforcement happens at pod creation. Labelling a namespace does not evict violating pods already running — the next rollout is when you find out.
- Audit the fleet: `kubectl get ns -L pod-security.kubernetes.io/enforce` shows every unlabelled namespace, which is every namespace with no policy at all.

## The securityContext Baseline

```yaml
securityContext:              # pod level
  runAsNonRoot: true
  runAsUser: 10001            # numeric — a username cannot be verified as non-root
  fsGroup: 10001
  seccompProfile: {type: RuntimeDefault}
containers:
- name: app
  securityContext:            # container level wins on conflict
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities: {drop: ["ALL"]}
```

- `runAsNonRoot` with a username in the image fails at admission on platforms that enforce it: the kubelet cannot resolve a name to a UID before starting the container. Numeric UIDs only (`docker` skill bakes this at build time).
- `readOnlyRootFilesystem: true` plus an `emptyDir` on `/tmp` catches an entire class of exploit that needs to write a payload. Applications that fail under it are usually writing logs or caches to the image — fix that, it was a bug anyway.
- `drop: ["ALL"]` then add back the one capability that is genuinely needed (`NET_BIND_SERVICE` for ports below 1024, or just bind above 1024 and let the Service map it).
- Container-level settings override pod-level. A hardened pod spec with one container overriding `allowPrivilegeEscalation` is a real finding and reads as compliant at a glance.

## Escape Vectors — Treat Each As A Finding

| Setting | What it hands over |
|---|---|
| `privileged: true` | Everything: all capabilities, all devices, effectively root on the node |
| `hostPID: true` | The node's process table; `nsenter` into any container on it |
| `hostNetwork: true` | Node network namespace, including localhost-only services and bypassing NetworkPolicy |
| `hostPath: /` (or `/var/run/docker.sock`, `/var/lib/kubelet`) | The node filesystem and every other pod's secrets |
| `CAP_SYS_ADMIN`, `CAP_NET_ADMIN`, `CAP_SYS_PTRACE` | Escape-adjacent capabilities that read as narrow |
| `allowPrivilegeEscalation: true` with a setuid binary | Root inside the container, the starting point for the rest |

Legitimate exceptions exist (CNI agents, CSI drivers, node exporters). They belong in a dedicated namespace labelled `privileged`, with their own ServiceAccounts, so the exception is visible instead of ambient.

## Network Boundaries

- Baseline per namespace: default-deny ingress and egress, plus an explicit allow for DNS (`networking.md`). Without egress control, a compromised pod exfiltrates freely and reaches every internal service.
- Block the cloud metadata endpoint (`169.254.169.254`) from pods. Otherwise an SSRF in any application becomes cloud credential theft; on AWS, enforce IMDSv2 with a hop limit of 1 at the instance level as well.
- Namespaces are an RBAC and policy boundary, not a kernel boundary. Hard multi-tenancy — untrusted code from different customers — needs separate clusters, or VM-isolated runtimes (gVisor, Kata) via RuntimeClass.

## Supply Chain

- Pin by digest in production (SKILL.md rule 8). A tag can be repointed at a different image after you vetted it; a digest cannot.
- Verify at admission, not only in CI: a policy controller checking signatures (cosign/sigstore) and enforcing an allowed-registry list is the only control that survives a compromised pipeline.
- Scan at two points — CI (blocks bad builds) and the registry (catches CVEs published after the build). CI-only scanning approves images that rot in place for a year.
- Rebuild regularly even with no code change: base-image CVE fixes only reach you through a rebuild.
- Keep the deployed digest inventory queryable: `kubectl get pods -A -o jsonpath='{..imageID}' | tr ' ' '\n' | sort -u`. When a CVE lands, that command is the difference between an hour and a week.

## Policy Beyond RBAC

RBAC cannot express field-level rules (`rbac.md`). Those need admission:

- **ValidatingAdmissionPolicy** — CEL expressions evaluated in-process by the API server. No webhook to keep alive, no availability risk. First choice for simple invariants ("every Deployment must set a memory limit", "no `latest` tags").
- **Kyverno / Gatekeeper** — full policy engines with mutation, generation, and reporting. Choose when you need policies that create objects (default NetworkPolicy per namespace) or a compliance report.
- Webhook-based policy is a cluster-wide dependency: `failurePolicy: Fail` with a down backend blocks every create in the cluster (`operators.md`). Scope with `namespaceSelector`, exclude `kube-system`, and run at least two replicas across zones.

## Detection and Response

- Enable API audit logging and ship it off-cluster. In-cluster logs are exactly what an attacker with cluster access can delete.
- The queries that matter: `exec` into pods, Secret reads, RoleBinding and ClusterRoleBinding creation, ServiceAccount token requests, anything from `system:anonymous`.
- Runtime detection (Falco-class syscall monitoring) catches what admission cannot see: a shell spawning inside a container that has never spawned one, an outbound connection to a new destination.
- Incident containment order for a suspected compromised pod: cordon the node, apply a deny-all NetworkPolicy to the pod's labels, snapshot for forensics (`kubectl debug`, node disk snapshot), then delete. Deleting first destroys the evidence and leaves the entry point intact (`incident-response` skill).

## Review Checklist

- Namespaces labelled with PSA `enforce`, version pinned?
- No privileged, hostPID/IPC/Network, or broad hostPath outside a declared exception namespace?
- `runAsNonRoot` with numeric UID, `allowPrivilegeEscalation: false`, capabilities dropped, seccomp RuntimeDefault?
- Default-deny NetworkPolicy plus explicit DNS allow in every application namespace?
- Metadata endpoint blocked; no static cloud credentials in Secrets (`config-and-secrets.md`)?
- Images pinned by digest, signature verified at admission, registry allow-list enforced?
- `automountServiceAccountToken: false` wherever the pod never calls the API (`rbac.md`)?
- Audit logs shipped off-cluster and someone actually reads the five queries above?

Write the sweep result rather than repeating it next quarter: anything found and deliberately not fixed goes to `## Known Gaps` in `~/Clawic/data/k8s/memory.md` with the date and the reason it was accepted; the PSA level, policy engine, and audit destination the cluster ended up with go to `## Clusters`; a securityContext, NetworkPolicy, or policy rule that finally satisfied both the workload and the profile goes to `~/Clawic/data/k8s/artifacts/policy-<name>.md` with its `## Boxes` line. Set the next audit date in `## Due`. **Nothing from an audit ever carries a secret value into those files** — a leaked-credential runbook stores `keychain:` and `vault:` pointers, never the credential it is about (`memory-template.md`).
