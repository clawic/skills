# RBAC — Permissions, Identity, and the Escalation Paths Auditors Miss

RBAC is purely additive: there is no deny rule. A subject's effective permission set is the union of every binding that touches it, so "we removed their access" means "we found and deleted every binding", not "we added a restriction".

## The Object Model in One Pass

- **Role** and **RoleBinding** are namespaced. **ClusterRole** and **ClusterRoleBinding** are not.
- The useful asymmetry: a RoleBinding may reference a ClusterRole, granting those verbs **in that one namespace**. Define the permission once, bind it narrowly — this is how you avoid twenty copies of the same Role drifting apart.
- Rules are `apiGroups × resources × verbs`, plus optional `resourceNames`. The core group is the empty string `""`; forgetting that is why a rule for `pods` written under `apiGroups: ["v1"]` silently grants nothing.
- Subresources are separate resources: `pods/exec`, `pods/log`, `pods/portforward`, `pods/attach`, `deployments/scale`. Granting `pods` does not grant exec; granting `pods/exec` without `pods` still lets someone exec into a pod whose name they know.
- `resourceNames` cannot restrict `list` or `watch` — those verbs are collection-level. A "read only this one Secret" rule that includes `list` reads all of them.

## Default ClusterRoles and What They Really Grant

| Role | Reality |
|---|---|
| `view` | Read most namespaced objects, **excluding** Secrets |
| `edit` | `view` plus write on workloads — and creating a pod means reading any Secret in the namespace |
| `admin` | `edit` plus managing Roles and RoleBindings inside the namespace |
| `cluster-admin` | Everything, everywhere, including the ability to grant everything |

`edit` in a namespace is therefore equivalent to "read every Secret in that namespace". If that is not acceptable, the boundary must be a separate namespace, not a smaller role.

Aggregated ClusterRoles matter when extending: labelling a ClusterRole with `rbac.authorization.k8s.io/aggregate-to-view: "true"` adds its rules to `view` cluster-wide. Convenient for CRDs, and a silent grant if someone aggregates a sensitive resource.

## Escalation Paths

- **`create pods`** ≈ read every Secret in the namespace (mount them) and act as any ServiceAccount in it (mount its token). Anything that creates pod templates — Deployments, Jobs, CronJobs, StatefulSets, DaemonSets — inherits this.
- **`escalate`** lets a subject grant permissions it does not hold; **`bind`** lets it bind an existing role to anyone; **`impersonate`** lets it act as another user or group. All three are administrator-equivalent, and all three look like ordinary verbs in a YAML review.
- **`create serviceaccounts/token`** mints a token for another identity: the same as holding that identity's credentials.
- **`pods/exec` into a privileged pod** is root on the node. So is any grant over `nodes/proxy`.
- **Updating a workload's `serviceAccountName`** promotes it to whatever that ServiceAccount can do — a write on Deployments is a write on identity.
- **Cloud identity crossover**: with IRSA, Workload Identity, or Managed Identity, editing a ServiceAccount's annotations can map it to a cloud IAM role. Namespace `admin` then reaches outside the cluster entirely — audit these annotations as if they were IAM policy documents.
- Wildcards (`verbs: ["*"]`, `resources: ["*"]`) in a Role are a finding, not a convenience: they silently absorb every CRD installed afterwards.

## ServiceAccount Tokens

- Every pod gets the `default` ServiceAccount unless told otherwise. Set `automountServiceAccountToken: false` on the ServiceAccount or the pod wherever the workload never calls the API — which is most workloads, and it removes the mounted token an attacker would otherwise find first.
- Modern tokens are projected, audience-bound, and time-limited, and the kubelet rotates them in place. Long-lived Secret-based tokens are no longer created automatically (1.24+); a non-expiring token Secret in a manifest is legacy and worth removing.
- One ServiceAccount per workload. Sharing one across a namespace means every audit answer is "some pod, we cannot tell which".
- Requesting a specific audience via a projected volume is how a workload authenticates to something other than the API server (a mesh, a cloud provider, an internal service) without any static credential.

## Verify Instead of Reasoning From YAML

```bash
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>          # the effective set
kubectl auth can-i create pods --as=system:serviceaccount:<ns>:<sa> -n <ns>
kubectl auth can-i '*' '*' --as=<user>                                  # is this subject cluster-admin in effect
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'
kubectl get clusterrolebindings,rolebindings -A -o json | jq -r '.items[] | select(.subjects[]?.name=="system:authenticated") | .metadata.name'
```

The last query finds bindings to `system:authenticated` or `system:unauthenticated` — the two group names that turn a scoped grant into a public one.

## Designing Permissions

1. Start from the workload's actual API calls, not from a role template. Most applications need zero API access; the honest first draft is `automountServiceAccountToken: false`.
2. For controllers, enumerate resource+verb pairs from the code (or run with a permissive role in staging and read the audit log), then write the minimal Role.
3. Bind ClusterRoles through RoleBindings for namespace scoping; reserve ClusterRoleBindings for genuinely cluster-scoped controllers.
4. Humans get groups from the identity provider (OIDC) or client certificates, never individual bindings — offboarding must be one action in one system.
5. Re-verify with `auth can-i --list` after every change. RBAC has no dry run for effect, only for admission.

## What RBAC Cannot Express

After a least-privilege Role finally works, save it to `~/Clawic/data/k8s/artifacts/policy-<serviceaccount>.md` — the YAML, the date, what it unblocked, and which verbs were removed and proved unnecessary — and add its `## Boxes` line to `memory.md`. Deriving one from audit logs costs a full staging cycle; nobody should pay it twice. A binding deliberately left broader than ideal goes to `## Known Gaps` with its reason.

RBAC operates on verbs and resources, never on field values. "Nobody may create a privileged pod", "images must come from our registry", "every Service must have a team label" are all unexpressible here — those are admission-control policies (`security.md`, `operators.md`). Trying to encode them as narrower roles produces a permission model that blocks legitimate work and still allows the thing you were worried about.
