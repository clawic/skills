# ConfigMaps and Secrets — Delivery, Rotation, and What "Encrypted" Does Not Mean

The object is the easy part. Everything that goes wrong is about **how it reaches the process** and **what happens when it changes**.

## Delivery Modes and Update Semantics

| Mode | Updates reach the pod? | Notes |
|---|---|---|
| `env` (single key) | Never | The value is copied at container start; requires a restart |
| `envFrom` (whole object) | Never | Keys that are not valid env names are skipped with an event, not an error |
| Volume mount (whole object) | Yes, within ~2 min | kubelet sync period (~1 min) + cache TTL (~1 min) |
| Volume mount with `subPath` | Never | subPath binds the inode once; the symlink-swap update works only at directory level |
| `immutable: true` | Never — updates are rejected | Also stops the kubelet watch, a real scalability win at thousands of pods |

- The update reaching the filesystem is not the update reaching the process. Unless the app watches the file or handles SIGHUP, a mounted ConfigMap change does nothing visible. Most applications need a restart either way.
- Deterministic pattern: hash the config into a pod-template annotation so any change triggers a normal rolling update with a rollback path (`rollouts.md`). kustomize's generators (name suffix by content hash) and Helm's `checksum/config` annotation both implement this.
- The immutable pattern is the same idea, stronger: `app-config-v7`, referenced by name, immutable. Changing config means a new object plus a template change — atomic, auditable, and rollback is just the previous revision.
- Both ConfigMaps and Secrets cap at 1MiB. Larger payloads belong in a volume or object store; a base64 blob in etcd is not storage.

## The Errors They Produce

| Pod status | Cause |
|---|---|
| `CreateContainerConfigError` | Referenced ConfigMap/Secret or key does not exist in this namespace |
| Container starts with an empty variable | `envFrom` skipped an invalid key name, or `optional: true` hid a missing object |
| Mount is an empty directory | `items` selected keys that do not exist |
| Works in one namespace only | Both object types are namespaced; a copied Deployment references objects that were never copied |

`optional: true` is a deliberate choice, not a default to sprinkle: it converts a loud startup failure into a subtly misconfigured running service.

## Secrets: The Honest Threat Model

- Secrets are base64-encoded, not encrypted. `kubectl get secret -o yaml` is a plaintext read for anyone with the verb.
- Anyone who can create a pod in a namespace can read every Secret in it: mount it, or mount the ServiceAccount token and use it. **RBAC on pod creation is Secret access** (`rbac.md`).
- Encryption at rest (`EncryptionConfiguration` on the API server, ideally with a KMS provider) protects etcd on disk and backups. It does nothing against an API-level read.
- Env-delivered secrets leak more broadly than file-delivered ones: they are inherited by every child process, visible in `/proc/<pid>/environ`, and routinely captured by crash handlers and error trackers. Prefer file mounts for anything long-lived.
- File-mounted secrets default to mode 0644 — readable by every process in the container. Set `defaultMode: 0400` and an appropriate `fsGroup`.
- Never put secrets in ConfigMaps: ConfigMaps end up in logs, dumps, and support bundles precisely because tools treat them as safe.

## Rotation That Actually Rotates

1. Write the new value (new object name if you use the immutable pattern).
2. Roll the workload — `kubectl rollout restart` for env-delivered values, or rely on the template change.
3. Verify the new value is in use before revoking the old credential: `kubectl exec <p> -- printenv` or read the mounted file. Revoking first turns rotation into an outage.
4. Revoke the old credential at the source. Deleting the Kubernetes object is not revocation.

Rotation frequency is a policy decision; rotation *capability* is an architecture decision. If step 2 requires a maintenance window, the rotation will not happen when it matters.

## External Secret Stores

| Approach | How it works | Cost |
|---|---|---|
| Sealed Secrets | Encrypted secret committed to Git, decrypted in-cluster by a controller | Cluster-key management; a lost key means re-sealing everything |
| External Secrets Operator | Controller syncs from Vault, AWS/GCP/Azure secret managers into native Secrets | Secrets still exist in etcd; simplest migration |
| Secrets Store CSI driver | Mounts secrets directly from the provider into the pod | No etcd copy; provider availability becomes a pod-start dependency |
| Vault agent sidecar | Sidecar authenticates and templates secrets into a shared volume | Most flexible, most moving parts; supports dynamic short-lived credentials |

Dynamic, short-lived credentials from Vault or cloud IAM beat every rotation policy, because there is nothing long-lived to rotate. Cloud workload identity (IRSA, Workload Identity, Managed Identity) removes static cloud credentials from the cluster entirely — the highest-leverage change on this page (`rbac.md`).

## Configuration Design Rules

- Config in objects, never in the image: the same digest must run in dev and prod, or your staging test proved nothing about the artifact (`manifests.md`).
- Split by change frequency: rarely changing structural config as an immutable ConfigMap, per-environment values in the overlay, secrets in a secret store. One giant map means every change restarts everything.
- The downward API exposes pod metadata (name, namespace, node, labels, resource limits) as env or files — use it instead of templating a pod name into config. `resourceFieldRef` handing the memory limit to a runtime flag is the clean way to keep heap sizing in sync with the manifest (`resources.md`).
- Keep the schema honest: fail fast at startup on a missing or malformed required value. A service that boots with an empty database URL and fails on first request converts a config error into an incident.
- Audit periodically: `kubectl get secrets -A -o json | jq -r '.items[] | select(.metadata.annotations."kubectl.kubernetes.io/last-applied-configuration") | "\(.metadata.namespace)/\(.metadata.name)"'` finds secrets applied from files — the ones most likely to be sitting in Git in plaintext.

**Nothing from this file's subject matter is ever written into `~/Clawic/data/`.** Not a Secret value, not a decoded base64 field, not a kubeconfig, not a Helm values file the user pastes — store `env:`, `keychain:`, `vault:`, `1password:` or `ssm:` pointers in place of the value and say in one line that you did (`memory-template.md` carries the full scheme and the two lists of what counts as secret here). What does get written: the store the user chose, as `secrets_backend` in `config.yaml`; which class of secret lives where and which Secrets must be recreated by hand after a restore, as a note in `## Clusters` — that list is the most common reason a namespace restore fails (`backup.md`).
