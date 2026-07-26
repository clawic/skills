# DNS — ndots, CoreDNS, and the Five-Second Stall

Cluster DNS fails in two distinctive shapes: everything is slow by a small constant (search-path expansion), or a few requests per thousand hang for exactly 5s (conntrack race). Both have specific fixes; neither is fixed by restarting CoreDNS.

## What Is Actually In The Pod's resolv.conf

```
nameserver 10.96.0.10                                   # kube-dns Service ClusterIP
search <ns>.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- `ndots:5` means: any name with fewer than 5 dots is tried against every search domain FIRST, then as an absolute name.
- `api.stripe.com` has 2 dots → 3 search-domain attempts (each returning NXDOMAIN) before the real query, and glibc sends A and AAAA for each. That is 8 queries per lookup (3 search attempts + the real one, ×2 for A and AAAA), 6 of them wasted, on every connection your app opens.
- Cluster names benefit: `db` resolves in one hop inside its namespace. External-heavy workloads pay for it all day.

Three fixes, in order of preference:

1. Fully-qualified names with a trailing dot in the application config (`api.stripe.com.`) — zero search expansion, no cluster change.
2. Per-pod `dnsConfig: {options: [{name: ndots, value: "2"}]}` — single-label cluster names (`db`) still expand through the search list, while any hostname with 2+ dots goes straight out. `ndots:1` is the aggressive version and breaks two-label cluster names like `db.prod`.
3. Client-side connection pooling so lookups happen once, not per request.

## The 5-Second Stall

- Signature: p99 latency has a hard 5.000s cliff on a small fraction of requests; the application stack is clean; retries succeed instantly.
- Cause: glibc sends the A and AAAA queries in parallel over one socket. Two conntrack entries for the same tuple race during DNAT insertion, one packet is dropped, and the resolver waits out its 5s timeout before retrying.
- Fixes, strongest first: NodeLocal DNSCache (a per-node cache on a link-local address that talks TCP upstream — the race disappears), `options single-request-reopen` in `dnsConfig` (glibc only), or forcing TCP.
- musl-based images (Alpine) implement a subset of resolv.conf options, so per-pod workarounds that fix glibc pods do nothing there. NodeLocal DNSCache or a different base image are the real options (`docker` skill for the base-image tradeoff).

## dnsPolicy

| Value | Resolver | Use |
|---|---|---|
| `ClusterFirst` | Cluster DNS, forwarding external names upstream | Default; correct for almost everything |
| `ClusterFirstWithHostNet` | Same, but required when `hostNetwork: true` | Host-network pods that still need cluster names |
| `Default` | Inherits the node's resolv.conf | Pods that must resolve exactly like the node |
| `None` | Nothing; you supply `dnsConfig` entirely | Custom resolvers, split-horizon corporate DNS |

The trap: a `hostNetwork: true` pod left on the default policy silently uses node DNS and cannot resolve any Service name. Symptom is "works as a normal pod, breaks as a host-network pod".

## CoreDNS Operation

- The Corefile's `kubernetes` plugin answers cluster names; `forward . /etc/resolv.conf` sends the rest upstream; `cache 30` caches positive and negative answers for 30s.
- Negative caching is why a Service created seconds ago is still NXDOMAIN. Wait out the cache instead of debugging a nonexistent problem.
- CoreDNS defaults to 2 replicas regardless of cluster size. On a busy cluster they throttle (`resources.md`) and every workload gets slow together — CPU throttling on CoreDNS is a cluster-wide latency event. Scale with the cluster-proportional autoscaler, and give CoreDNS a PDB (`rollouts.md`).
- Temporary visibility: add the `log` plugin to the Corefile, reproduce, remove it. Query logs on a large cluster are enormous, so scope the window.
- `kubectl -n kube-system logs -l k8s-app=kube-dns` shows plugin errors and upstream failures; `dig @<coredns-pod-ip> <name>` from a debug pod bypasses caching layers to test a specific instance.

## Record Shapes Worth Knowing

| Name | Resolves to |
|---|---|
| `<svc>` | The Service, in the pod's own namespace only |
| `<svc>.<ns>` | The Service in another namespace — the short form never crosses namespaces |
| `<svc>.<ns>.svc.cluster.local.` | Absolute, no search expansion |
| `<pod>-0.<svc>.<ns>.svc.cluster.local` | A specific StatefulSet member via a headless Service (`stateful.md`) |
| `_https._tcp.<svc>.<ns>.svc.cluster.local` | SRV record: port and target, for clients that discover ports |
| Headless Service | A records for every ready pod, not a VIP — clients load-balance themselves |

- `publishNotReadyAddresses: true` on a headless Service exposes pods before they are ready: required for peer discovery in clustered systems (a node cannot join a quorum it cannot see), dangerous for anything else.
- `hostname` + `subdomain` on a plain pod gives it a stable DNS name without a StatefulSet.
- ExternalName Services return a CNAME. Clients doing TLS then present the original name in SNI while the certificate belongs to the target — the "works with curl, fails in the app" TLS mismatch.

## DNS Triage Chain

1. From inside the failing pod: `getent hosts <name>` (present in musl and glibc; `nslookup` and `dig` usually are not).
2. Fails for cluster names → is the Service real, and is the pod in the namespace you think? `kubectl get svc -n <ns>`.
3. Fails for everything → NetworkPolicy blocking egress to port 53 is the first suspect (`networking.md`), CoreDNS down is the second.
4. Works for cluster names, fails externally → CoreDNS upstream (`forward`) or the node's own resolver; test from the node (`nodes.md`).
5. Works but slowly → count the queries: `kubectl exec <p> -- getent hosts api.example.com` with the CoreDNS log plugin on shows the search-path expansion directly.
6. Works from one pod, fails from another on the same Service → compare `resolv.conf` between them; a `dnsConfig` or `dnsPolicy` override in one manifest explains it.

DNS failures are the most re-diagnosed shape in Kubernetes, because the symptom (a 5s stall, an intermittent timeout) never looks like DNS. When one is finally pinned, write the shape, its cause, and the fix into `## Incident History` in `~/Clawic/data/k8s/memory.md`, and the cluster-level remedy that was adopted — NodeLocal DNSCache, a CoreDNS replica count and PDB, an `ndots` default — into `## Clusters`. The next person to see a 5.000s p99 cliff should find the answer in one read.
