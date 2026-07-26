# Production — The Cluster You Are Responsible For

Scope: the cluster-level decisions that decide whether an incident is fifteen minutes or a day. Workload-level readiness lives in the Output Gates of SKILL.md; this is everything above the Deployment.

**Open with the `## Due` table and `## Clusters` in `~/Clawic/data/k8s/memory.md`**: an overdue restore drill, certificate sweep, or deprecated-API sweep is stated in one line before anything else here is worth discussing, and the last upgrade's record in `deploys/<year>.md` is the only honest input to planning the next one.

## Capacity With Failure Budgeted In

```
usable capacity = allocatable × (nodes − 1)          # survive one node loss
zone-resilient   = allocatable × nodes × (z−1)/z     # survive one zone of z
```

- Target 50-75% of allocatable requested cluster-wide. Below ~50% you are paying for air; 75-85% is acceptable only if you can tolerate a wait for new nodes after a failure; above ~85% a single node loss has nowhere to reschedule and eviction cascades (`resources.md`).
- Headroom is what absorbs traffic spikes; autoscaling arrives 1-3 minutes later, or 3-6 with a node provision (`autoscaling.md`).
- Spread that matters is declared, not hoped for: topology spread constraints by zone on every multi-replica workload, `DoNotSchedule` for the ones whose SLO depends on it (`scheduling.md`).
- Check the single points that are not in your Deployment: ingress controller replicas across zones, CoreDNS replicas plus a PDB, the registry your nodes pull from, and the cloud LB's own health checks.

## Upgrades

- Order is fixed: control plane, then nodes, then workloads that depend on new APIs. Skipping minor versions on the control plane is unsupported — one minor at a time.
- Version skew is a real constraint: kubelets may trail the API server by a few minor versions (widened in recent releases) and `kubectl` is supported within one minor either way. Confirm the exact policy for the target version before planning a long node rollout.
- **Deprecated APIs are the recurring hazard.** Sweep before, not during: deprecation warnings appear in `kubectl` output and in audit logs, and a scan of the manifests in Git catches what is not currently deployed. A chart pinned two years ago is the usual offender (`manifests.md`).
- Nodes upgrade by cordon → drain → replace, honoring PDBs (`nodes.md`). Budget the drain time: 200 nodes at 3-10 minutes each is 10-33 hours serial — more than one maintenance window, and multi-day once a PDB stalls a batch — unless you parallelize deliberately.
- Rehearse on a non-production cluster that has the same CRDs, webhooks, and CNI. Version-skew bugs live in extensions, not in core (`operators.md`).
- After every upgrade: webhook certificates still valid, CRDs applied, metrics-server and CoreDNS healthy, one test deploy end to end.

## Disaster Recovery

Three separate things, restored in this order, and each one needs its own rehearsal:

1. **Cluster** — a new cluster from infrastructure-as-code. The fastest recovery is recreation, not repair, if the next two are in place.
2. **Manifests** — everything in Git, applied by a GitOps controller. Objects that exist only in a cluster (hand-applied secrets, hotfixes) are the gap that turns a 30-minute recovery into a reconstruction project.
3. **Data** — PV snapshots and database backups in a different failure domain, restored and verified on a schedule (`storage.md`, `stateful.md`).

- etcd snapshots restore the API objects, never the volume contents. On managed control planes you do not own etcd at all, which makes points 2 and 3 the entire plan.
- Write down the RTO you can actually demonstrate, not the one in the document. The number is whatever the last drill measured.
- Drill list, quarterly: kill a node, drain a zone, restore one database from backup, and rebuild a namespace from Git into an empty cluster.

## Alerts Worth Paging For

| Alert | Why it is not noise |
|---|---|
| Node NotReady > 5 min | Past the eviction timeline; workloads are moving |
| Pod restart rate rising across a Deployment | A crash loop with a working readiness probe hides behind green dashboards |
| Deployment replicas available < desired for > 10 min | Rollout stuck past `progressDeadlineSeconds` (`rollouts.md`) |
| PVC free space < 15% | Storage failure has no Kubernetes-level symptom (`storage.md`) |
| Certificate expiring < 14 days | Webhook, ingress, and kubelet certificate expiry are cluster-wide events |
| CoreDNS latency or throttling | Cluster-wide latency wearing an application's clothes (`dns.md`) |
| API server 5xx or latency p99 rising | Everything downstream degrades next |
| CronJob last-success age > 2 intervals | Failure alerts never fire when the job stopped being scheduled (`jobs.md`) |
| Node allocatable requested > 85% | Your failure budget is gone before the failure |
| Pods pending > 15 min | Autoscaling is not solving it and nobody was told |

Ship events off-cluster (they expire after 1h) and keep audit logs elsewhere too — during an incident, the cluster is exactly where you cannot trust the record (`security.md`).

## Multi-Tenancy and Blast Radius

- ResourceQuota and LimitRange in every namespace, even generous ones: an unbounded namespace can starve the cluster from one bad manifest (`resources.md`).
- PriorityClasses with two tiers plus a default; anything above cluster-critical is a mistake waiting for a busy afternoon (`scheduling.md`).
- Namespaces are a policy boundary, not a kernel boundary. Untrusted workloads need separate clusters or isolated runtimes (`security.md`).
- One team's admission webhook or CRD upgrade breaking another team is the signal that separate clusters are now cheaper than the coordination.

## Cost

- Cost is driven by requests, not usage: over-requesting is directly billed as idle nodes. Fleet-wide rightsizing from VPA recommendations is usually the single largest saving available (`autoscaling.md`).
- Second largest: consolidation. Bin-packing with a right-sizing provisioner beats a static node group at almost any scale.
- Then spot for retryable workloads (`jobs.md`), and finally the boring wins — orphaned PVCs from deleted StatefulSets, unattached load balancers, retained snapshots, and log volume nobody reads.
- Attribute before optimizing: namespace and label-based cost reporting turns "the cluster is expensive" into a specific team's specific workload, which is the only version of the conversation that changes anything.

## Cluster Readiness Gate

- Control plane HA (or managed), and at least one drill proving you can rebuild the cluster from code?
- Every application namespace: quota, LimitRange, PSA label, default-deny NetworkPolicy?
- CoreDNS and the ingress controller: multiple replicas, PDBs, spread across zones?
- PDBs on everything above 1 replica, and none of them stricter than the replica count?
- Manifests in Git, one writer per object, drift detection on (`manifests.md`)?
- Backups of data and manifests in a separate failure domain, with a restore performed this quarter?
- The ten alerts above wired to a human, and events plus audit logs shipped off-cluster?
- Upgrade path known: current version, target, deprecated APIs swept, extensions verified?

Every line of that gate produces a durable fact, so write them where the next session finds them: cluster version, capacity ratio, alerting and control-plane posture into `## Clusters`; the upgrade itself, its drain time and what broke into `~/Clawic/data/k8s/deploys/<year>.md`; the measured RTO of any drill into the same file under `## Restore Drills` (`backup.md` runs the drill); every recurring item above — upgrade window, quarterly drill, monthly certificate and orphaned-storage sweeps — into the `## Due` table with a real date; and any gate item consciously left unmet into `## Known Gaps` with the date and the reason. A readiness gate whose result is not written is a gate that gets re-run from zero next quarter.
