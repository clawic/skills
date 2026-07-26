# Manifests — Apply Semantics, Kustomize, Helm, and Drift

Two rules carry most of this file: **one writer per object**, and **see the diff before the cluster does**. Everything that "silently reverted" violates the first; everything that "worked in staging" violates the second.

## apply, create, replace, and Server-Side Apply

| Verb | Behavior | When it hurts |
|---|---|---|
| `create` | Fails if the object exists | Fine in CI, useless for reconvergence |
| `replace` | Overwrites the whole object | Drops fields other controllers own (HPA replicas, LB status) |
| `apply` (client-side) | Three-way merge using the `last-applied-configuration` annotation | Objects created without apply have no annotation; the first apply then behaves unexpectedly |
| `apply --server-side` | Merge computed by the API server with tracked field ownership | Conflicts are explicit and must be resolved, which is the point |

- Server-side apply records a `managedFields` owner per field. When two managers write the same field, the second gets a conflict error naming the other owner — the machine-readable version of "who changed this".
- `--force-conflicts` takes ownership. Correct when you are intentionally moving ownership; a disaster as a reflex in a CI script, because it makes your pipeline win against controllers that were right.
- The canonical ownership fight: an HPA owns `spec.replicas` while your manifest also sets it. Every apply scales back to the manifest value, the HPA scales out again. Fix: remove `replicas` from the manifest entirely once an HPA exists (in Argo CD, also add an `ignoreDifferences` entry) (`autoscaling.md`).
- `kubectl apply --prune` deletes objects that vanished from the input using label selectors. It has deleted production namespaces because a selector matched more than intended; prefer a GitOps controller's prune, which tracks ownership explicitly.

## Immutable Fields

`field is immutable` means the API is protecting you, not obstructing you:

| Object | Immutable | Path forward |
|---|---|---|
| Deployment | `spec.selector` | New Deployment, shift traffic, delete old |
| Service | `clusterIP`, some type transitions | Recreate; expect a brief DNS-visible gap |
| StatefulSet | `volumeClaimTemplates`, `serviceName` | `delete --cascade=orphan`, recreate, adopt (`stateful.md`) |
| Job | Most of `spec` | Delete and recreate with a new name |
| PVC | Everything except a size increase | Create new, migrate data (`storage.md`) |

Design implication: label schemes and selectors are decisions you live with. Write every manifest with the keys `label_scheme` names — the default `app.kubernetes.io/*` set is `name`, `instance`, `component`, `part-of`, and `version` — settle them before the first apply, and never let a tool inject labels into selectors afterwards.

## Kustomize

- Bases plus overlays, no templating language. Overlays patch with strategic merge (list merge by key) or JSON 6902 (explicit path operations) — reach for 6902 when a strategic merge cannot express a list edit.
- ConfigMap and Secret generators append a content hash to the name and rewrite every reference, so a config change becomes a pod-template change and rolls out with a rollback path (`config-and-secrets.md`).
- The classic footgun: `commonLabels` injects labels into `spec.selector` as well as metadata. On an existing Deployment that is an immutable-field error; on a new one it silently ties your selector to a label you may want to change. Prefer `labels` with `includeSelectors: false` for anything cosmetic.
- Patches apply to whatever the target selector matches. A patch matching nothing is not an error — it is silently ignored, which is how an overlay stops taking effect after a rename.
- Verify what will be sent: `kubectl kustomize overlays/prod | kubectl diff -f -`.

## Helm

- Values precedence, low to high: chart `values.yaml` → parent chart values → `-f` files in order → `--set`. `--set` beating a values file is the source of "the file says 3 replicas and it deployed 1".
- Release state lives in Secrets in the release namespace, one per revision. `helm history` and `helm rollback` read them; deleting those Secrets destroys the rollback path.
- "another operation (install/upgrade/rollback) is in progress" after an interrupted upgrade means the release is stuck in a pending state. Resolve with `helm rollback` to the last good revision, or `--force` only after understanding what the interrupted run did.
- **CRDs in the `crds/` directory are installed once and never upgraded or deleted by Helm.** Operator upgrades therefore need a manual CRD apply; skipping it produces a controller reading a schema its CRs do not have (`operators.md`).
- Hooks (`pre-install`, `pre-upgrade`) run as Jobs outside the normal lifecycle. A failed hook aborts the release; a hook without `hook-delete-policy` leaves Job objects accumulating (`jobs.md`).
- `--atomic --timeout` rolls back automatically on failure — the closest thing to a safe default for CI-driven Helm.
- Review the rendered output, not the chart: `helm template ... | kubectl diff -f -` before every upgrade of anything that matters.

## GitOps

- Argo CD and Flux continuously reconcile the cluster toward Git. Two consequences: a manual `kubectl edit` is reverted within minutes (that is the feature), and a bad commit propagates without a human in the loop (that is the risk).
- Self-heal on plus a hotfix by hand is the most common way people lose an emergency fix mid-incident. Either pause the app or commit the fix.
- Sync waves order dependent objects (namespace and CRDs first, then workloads, then jobs). Without ordering, a CR applied before its CRD fails and the sync retries noisily.
- `ignoreDifferences` is the honest declaration that another controller owns a field: HPA replicas, injected sidecar containers, cert-manager-populated secrets, cloud-populated Service status.
- Drift detection is an availability signal as much as a compliance one: an object that differs from Git during an incident is either the fix or the cause, and you need to know which.

## The Pre-Apply Gate

```bash
kubectl diff -f manifest.yaml                 # exactly what changes, with defaulting applied
kubectl apply -f manifest.yaml --dry-run=server   # real admission: webhooks, quota, PSA
```

- `--dry-run=client` runs no admission and no defaulting. It catches YAML syntax and nothing that has ever caused a production incident.
- Schema validation in CI (kubeconform-class tools against the target cluster's API version) catches the deprecated-field errors before the cluster does; policy checks belong in the same step (`security.md`).
- Deprecated API versions are the recurring upgrade hazard: the API server emits deprecation warnings on every use, and they are visible in audit logs and in `kubectl` output. Sweep for them before the cluster upgrade, not during (`production.md`).
- Keep the diff small enough to read. A 900-line diff from a chart bump is a review that did not happen; split infrastructure changes from application changes so each diff has one story.

The facts this file establishes are the ones a future session cannot infer from the cluster: which controller owns which objects, where the manifests live, and which fields are deliberately ignored. Record them in `## Clusters` in `~/Clawic/data/k8s/memory.md` — GitOps controller and repository, sync and self-heal posture, the `ignoreDifferences` entries and why each exists. The user's stated tooling choice (`manifest_tool`, `gitops_controller`, `apply_gate`) is a declaration and goes to `config.yaml`, never to memory.
