# CRDs, Controllers, Webhooks — Extensions and How They Break Everything

Extensions are where Kubernetes stops being a fixed system: a CRD adds types, a controller adds behavior, a webhook adds veto power over every write. The third one is the highest-blast-radius object in the cluster.

## Admission Webhooks

Order on every write: authentication → authorization (RBAC) → **mutating webhooks** → schema validation → **validating webhooks** → persistence.

- `failurePolicy: Fail` means an unreachable webhook blocks every matching request. Scoped to `resources: ["*"]` cluster-wide, a down webhook backend means nothing can be created anywhere — including the pods that would restore the webhook. That is the classic self-inflicted total outage.
- Prevention, all four: `namespaceSelector` excluding `kube-system` and the webhook's own namespace, `objectSelector` narrowing to what you actually care about, at least two replicas across zones with a PDB, and `timeoutSeconds` well under 30s (10s default) so a slow backend degrades instead of hanging every apply.
- `failurePolicy: Ignore` trades enforcement for availability: policy is skipped exactly when the cluster is unhealthy. Correct for advisory mutations, wrong for a security control you claim to enforce.
- Webhook serving certificates expire. When they do, every write fails at once with an x509 error — cert-manager or the operator's own rotation must be verified, not assumed (`ingress.md` for the cert-manager debugging chain).
- Mutating webhooks are invisible in your manifest: sidecar injection, default labels, image rewriting. When the running pod does not match the YAML you applied, list them: `kubectl get mutatingwebhookconfigurations`.
- Emergency break-glass, in full knowledge of what it disables: `kubectl delete validatingwebhookconfiguration <name>` (save the YAML into `~/Clawic/data/k8s/artifacts/` first — it is both the restore path and the runbook). Re-apply once the backend is healthy.

## CRDs

- A CRD adds a type with its own OpenAPI schema; `kubectl explain <kind>` works against it once installed, and printer columns decide what `kubectl get` shows — both worth setting on any CRD you own.
- **Deleting a CRD deletes every custom resource of that type, cluster-wide, immediately.** For a database operator that can mean deleting every database object and, depending on the operator's finalizers, the underlying storage. Treat `kubectl delete crd` as a destructive command in the same class as dropping a table.
- Multiple versions need a `storageVersion` and, for non-trivial changes, a conversion webhook. Without conversion, a client asking for the old version gets an error rather than a translation.
- Helm does not upgrade CRDs from `crds/` (`manifests.md`). Operator upgrade order is: read the changelog, apply new CRDs, then upgrade the controller.
- Scale and cost: CRs live in etcd like everything else. Thousands of large CRs (per-request custom resources, per-user objects) turn etcd into the bottleneck — CRDs are for configuration, not for application data.

## Finalizers and Deletion

Deletion is two phases: the API sets `deletionTimestamp` and waits while any `metadata.finalizers` remain; each controller removes its own entry after its cleanup succeeds; the object disappears when the list is empty.

- "Stuck Terminating" therefore always means: some controller has not removed its finalizer. The questions are which one, and why it cannot finish.
- Diagnose: `kubectl get <obj> -o jsonpath='{.metadata.finalizers}'` names the owner. Then read that controller's logs — the cleanup is usually failing against an external system (a cloud load balancer that no longer exists, a database it cannot reach).
- Removing the finalizer by hand (`kubectl patch ... -p '{"metadata":{"finalizers":null}}' --type=merge`) makes the object disappear and **abandons whatever it was cleaning up**: orphaned cloud load balancers, unattached disks, unreleased IPs, real money. Do it knowingly, and go clean up the external resource yourself.
- Common built-ins: `kubernetes.io/pvc-protection` (a pod still uses the PVC — delete the pod), `foregroundDeletion` (children first), and Service LB finalizers (the cloud controller is trying to delete the LB).

## The Stuck Namespace

A namespace in `Terminating` for more than a minute has one of two causes, and they are distinguishable in one command each:

```bash
kubectl get apiservice | grep -v "True"                     # an aggregated API is unavailable
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n1 kubectl get -n <ns> --show-kind --ignore-not-found   # remaining objects
```

- An unavailable APIService (a metrics or custom API whose backing pods are gone) blocks namespace deletion because the namespace controller cannot enumerate what it must delete. Fix or delete the APIService.
- Otherwise, it is an object with a finalizer inside the namespace — handle it per the section above. Editing the namespace's own finalizer list is the internet's favorite answer and it leaves every one of those objects orphaned in etcd.

## How Controllers Actually Behave

- Reconciliation is level-triggered, not event-driven: the controller compares desired to actual and acts, repeatedly. A missed event is not a lost update; a wrong `status` is a bug in the controller, not a missed message.
- Errors are retried with exponential backoff, so a controller stuck on one bad object slows down its whole work queue. One malformed CR can delay every other CR of that type.
- Leader election: a controller holds a Lease and only the leader acts. Symptom of a broken lease is a controller that runs, logs nothing interesting, and changes nothing — check `kubectl get lease -n <ns>` and the lease holder's identity before debugging the logic.
- `ownerReferences` drive garbage collection: delete the owner and children go too. Two rules that produce silent data loss — an ownerReference to a **different namespace** or to a **cluster-scoped owner from a namespaced object** is invalid, and the garbage collector may delete the child; and an ownerReference whose UID no longer matches (owner recreated with the same name) makes the child an orphan collected on the next sweep.
- Cascading deletion has three modes: `Background` (default: owner goes first, children asynchronously), `Foreground` (children first, owner shows `foregroundDeletion`), `Orphan` (children survive — the StatefulSet surgery trick in `stateful.md`).

## Choosing and Operating an Operator

- Evaluate on the failure path, never the install path: what happens when the primary's node dies mid-write, how a point-in-time restore is performed, whether upgrades are tested across your version, whether it supports the storage you have.
- Check the RBAC it asks for. Many operators request cluster-wide `*` on `*` "for simplicity"; that is a cluster-admin equivalent running unattended (`rbac.md`).
- Prefer namespace-scoped operators when the option exists: blast radius, RBAC, and upgrade independence all improve at once.
- Watch resource consumption of the controller itself. A controller listing every pod in a large cluster without field selectors is a memory problem that arrives at scale, not at install.
- Debugging an operator's CR, in order: `kubectl describe <cr>` (events and `status.conditions` — a well-built operator explains itself there), controller logs, lease holder, then the external system it manages.

Keep the extension inventory in `## Clusters` in `~/Clawic/data/k8s/memory.md`: which operators are installed, their versions and the CRD versions they own, which webhooks exist with their failure policy, and what each one is allowed to do. It is the list an upgrade plan needs first (`production.md`) and the one nobody can reconstruct mid-incident. An operator upgrade that broke something, and the CRD change that caused it, go to `## Incident History`; a break-glass procedure you had to invent — deleting a webhook configuration to unblock the cluster — belongs in `artifacts/runbook-<kebab>.md` with the YAML saved and its `## Boxes` line added.
