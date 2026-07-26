# Networking — Services, EndpointSlices, and Policy

Mental model: a Service is not a process. It is a label selector that produces EndpointSlices, plus a virtual IP that every node's kube-proxy programs into DNAT rules. Nothing listens on a ClusterIP. Most "networking bugs" are one of those three pieces pointing at the wrong thing.

## Three Ports, Three Meanings

| Field | Meaning | Common error |
|---|---|---|
| `port` | What clients dial on the Service | Assumed to equal the container's port |
| `targetPort` | Port (or named port) on the pod | Renaming a container port breaks the name reference silently |
| `containerPort` | Documentation only — traffic flows without it | Believed to publish or open anything |

Named ports are worth the indirection: `targetPort: http` keeps working when the container port changes, and it makes probes and Services agree.

## Service Types

| Type | What it adds | Use |
|---|---|---|
| ClusterIP | Virtual IP inside the cluster | Everything internal; the default |
| NodePort | Also opens a port 30000-32767 on every node | Dev, or a hand-rolled LB in front |
| LoadBalancer | NodePort plus a cloud LB provisioned by the controller | One per service is expensive; an Ingress shares one |
| ExternalName | CNAME to an outside hostname, no proxying | Migrating an external dependency behind a cluster name |
| Headless (`clusterIP: None`) | DNS returns pod IPs, no VIP | StatefulSets, client-side load balancing, gRPC |

- A Service with no selector never gets endpoints automatically — that is the point: hand-write an EndpointSlice pointing at an external database and the rest of the cluster reaches it by service name.
- LoadBalancer objects stuck in `Pending` mean no cloud controller is running (kind, bare metal). MetalLB or an equivalent supplies one on-prem.

## EndpointSlices Are The Source Of Truth

```bash
kubectl get endpointslices -l kubernetes.io/service-name=<svc> -o wide
```

- Empty slice = selector mismatch or zero ready pods. Nothing else produces it.
- The Endpoints API is deprecated as of 1.33 in favor of EndpointSlices; slices also scale (100 endpoints per slice by default) where a single Endpoints object did not.
- `conditions.ready`, `serving`, and `terminating` are separate: a terminating pod can still be `serving` so in-flight requests finish. That is the mechanism your `preStop` sleep relies on (`probes.md`).
- Endpoint propagation is asynchronous across every node's kube-proxy and every ingress controller. Under load, expect a few hundred milliseconds to seconds — never assume removal is instant.

## kube-proxy Modes

- **iptables** (default): DNAT rules with random backend selection. Rule updates are O(services × endpoints) and reload latency grows with cluster size. No retry: a connection sent to a dead backend fails rather than being re-dispatched.
- **IPVS**: hash-table based, real scheduling algorithms (rr, lc), better at thousands of services. Needs kernel modules present on the node.
- **nftables**: the newer backend that fixes the iptables reload cost; availability depends on the cluster version, so confirm before planning a migration.
- All three explain the same symptom set: connections to just-deleted pods failing during a rollout, and slow propagation on huge clusters. Neither is fixed by editing your Deployment.

## Traffic Policy and Source IP

- `externalTrafficPolicy: Cluster` (default) — the LB hits any node, which then hops to a pod on any node. Client IP is lost to SNAT; load is even.
- `externalTrafficPolicy: Local` — no second hop, real client IP preserved, but nodes without a ready pod fail the LB health check (via `healthCheckNodePort`). With fewer replicas than nodes, expect uneven load; with one replica, a single node carries everything.
- `internalTrafficPolicy: Local` keeps in-cluster traffic on the same node — a genuine latency and cost win for node-local caches and log agents, and a black hole if that node has no ready pod.
- `sessionAffinity: ClientIP` pins by source IP for 3h by default. Combined with SNAT (`Cluster` policy) it pins per node, not per user — almost never what was intended.

## NetworkPolicy

- Additive allow-listing. A pod with no policy selecting it allows everything; the moment any policy selects it, that pod is default-deny **for that direction only**. Writing an ingress policy therefore does not restrict egress at all.
- The AND/OR trap that produces accidental cluster-wide access:

```yaml
# AND: pods labeled app=api IN namespaces labeled env=prod
- from:
  - namespaceSelector: {matchLabels: {env: prod}}
    podSelector: {matchLabels: {app: api}}
# OR: ALL pods in env=prod namespaces, PLUS app=api pods in this namespace
- from:
  - namespaceSelector: {matchLabels: {env: prod}}
  - podSelector: {matchLabels: {app: api}}
```

One list item with two selectors is an intersection; two list items are a union. The dash placement is the whole security boundary.

- Always ship the DNS egress rule (UDP and TCP port 53 to kube-dns) in the same change as your first egress policy, or every name resolution in the namespace dies first (`dns.md`).
- `ipBlock` matches IPs after any SNAT the CNI applies, so it is unreliable for pod-to-pod rules — select pods by label, and reserve `ipBlock` for genuinely external CIDRs.
- Probes come from the kubelet on the node, not through the pod network, so NetworkPolicy never blocks them. A pod can be perfectly healthy and completely unreachable.
- Policy enforcement is the CNI's job: Flannel enforces nothing, Calico and Cilium do, and the extras (FQDN-based egress, L7 rules, cluster-wide policies) are CNI-specific CRDs, not core Kubernetes.

## Pod Networking Facts That Explain Weird Bugs

- Every pod gets a routable-in-cluster IP that changes on every restart. Anything caching a pod IP (JVM DNS caching with `networkaddress.cache.ttl=-1` is the classic) breaks after the first rollout.
- Overlay encapsulation (VXLAN, ~50 bytes of overhead) lowers the effective MTU. Signature: TLS handshakes and small requests fine, large POSTs or downloads hang forever. Check pod MTU against node MTU before blaming the application.
- `hostNetwork: true` puts the pod in the node's namespace: no pod IP, port conflicts become scheduling failures, and DNS needs `dnsPolicy: ClusterFirstWithHostNet` or cluster names stop resolving.
- Conntrack table exhaustion on a busy node (`nf_conntrack: table full, dropping packet`) produces random resets across every pod on that node — an infrastructure symptom masquerading as an application bug (`nodes.md`).
- UDP services keep stale conntrack entries pointing at a deleted pod; long-lived UDP clients keep talking to nothing until the entry expires.
- Dual-stack requires `ipFamilyPolicy: PreferDualStack` on the Service; single-stack clusters silently ignore AAAA expectations, and a client preferring IPv6 sees connection failures nobody else reproduces.

## Reachability Triage

| From → To | Test | If it fails |
|---|---|---|
| Pod → Service (same ns) | `kubectl exec <p> -- nc -zv <svc> <port>` | EndpointSlice, then targetPort |
| Pod → Service (other ns) | Use `<svc>.<ns>` — short names only resolve in-namespace | `dns.md` |
| Pod → external | `kubectl exec <p> -- nc -zv 1.1.1.1 443` | Egress NetworkPolicy, NAT gateway, node routing |
| Your laptop → Service | `kubectl port-forward svc/<svc> 8080:<port>` | If this works, the fault is in the LB or Ingress layer |
| Internet → Service | curl the LB hostname | `ingress.md`, LB health checks, `externalTrafficPolicy` |
| Pod → Pod, both healthy | `kubectl exec <p> -- nc -zv <podIP> <port>` | CNI or NetworkPolicy, in that order |

Never use `ping` to test a Service: a ClusterIP has no interface and answers no ICMP. It is not down; it is not a host.

A NetworkPolicy that finally allows exactly what the workload needs and nothing else is expensive knowledge: it took an outage, a DNS rule, and several rounds of selector arithmetic. Save it to `~/Clawic/data/k8s/artifacts/policy-<name>.md` with the date and what it unblocked, and add its `## Boxes` line in the same turn. The CNI in use, whether it enforces policy at all, and the cluster's pod/service CIDRs go in `## Clusters` in `~/Clawic/data/k8s/memory.md` — the CIDRs are needed by every future peering, MTU, and `ipBlock` question.
