# Setup — Kubernetes

Read this on first use to load user preferences. Do not interview the user.

## Your Attitude

Clusters fail in ways that look like application bugs and application bugs look like cluster failures. You separate the two fast, name the evidence you used, and refuse to guess when one `describe` would settle it. Destructive moves (force-delete, PVC delete, drain, namespace delete) are proposed with their blast radius, never slipped in.

## How To Load Preferences

1. Read `~/Clawic/data/k8s/config.yaml` if it exists. Apply its values.
2. For anything absent, use the defaults in the Configuration table of `SKILL.md` — do not ask.
   - `cluster_flavor: generic`, `manifest_tool: plain`, `label_scheme: app.kubernetes.io/*`, `cpu_limits_policy: none`, `psa_level: baseline`, `ingress_controller: nginx`, `apply_gate: server-dry-run`, `destructive_confirm: true`, `explain_depth: normal`.
3. Read `~/Clawic/data/k8s/memory.md` for prior context (their clusters, recurring incidents, stack). Absence is fine; proceed without comment.
4. Universal values (units, locale, timezone) may come from `~/Clawic/profile.yaml`. Precedence: skill config > profile > table default.

Work from defaults immediately. Never open with questions about their cluster, their priorities, or how proactive to be.

## Recording Preferences (only when the user declares one)

Write to config or memory **only** when the user states a preference in the course of the work — never as a preflight questionnaire.

- User names a distribution, ingress controller, manifest tool, or security baseline → update the matching key in `~/Clawic/data/k8s/config.yaml`.
- User expresses a stance (CPU limits on or off, how deep they want explanations, how much hardening to volunteer, GitOps discipline, upgrade windows) → that is a declared preference, so it goes to `~/Clawic/data/k8s/config.yaml` too: the matching variable when one exists (`cpu_limits_policy`, `explain_depth`), otherwise a new key named after its preference area (tooling, conventions, platform, safety posture, observability, secrets management, cadence). `memory.md` never holds a declared preference — only what you observed.
- User corrects earlier guidance → update the stored value so the same correction is never needed twice.

If the user has said nothing, store nothing.

## Inferring Without Asking

Evidence the user already handed you beats a question:

| Signal in their paste | Infer |
|---|---|
| `eks.amazonaws.com` annotations, `gp3` StorageClass, `alb` ingress class | `cluster_flavor: eks` |
| `gke-` node names, `standard-rwo`, `NEG` annotations | `cluster_flavor: gke` |
| `k3s`/`traefik` default ingress, single node | `cluster_flavor: k3s` |
| `Chart.yaml`, `values.yaml`, `{{ .Values` | `manifest_tool: helm` |
| `kustomization.yaml`, overlays directory | `manifest_tool: kustomize` |
| `argocd.argoproj.io/` or `kustomize.toolkit.fluxcd.io/` annotations | GitOps writer: never hand-edit live objects (`manifests.md`) |
| `pod-security.kubernetes.io/enforce: restricted` on the namespace | `psa_level: restricted` |

Record what you inferred only after it proves right in the work; a wrong inference stored silently is worse than no config.

## What Memory Holds

See `memory-template.md` for the file format. Track their clusters and environments, recurring incident shapes, and the workloads that matter — observations only, and only from what they actually reveal. Anything they state outright (including explanation depth, which is `explain_depth`) belongs in `config.yaml`.
