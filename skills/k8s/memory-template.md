# Working File Templates — Kubernetes

Read this file only when WRITING. `config.yaml` is what the user **declared**; `memory.md` and everything it indexes is what you **observed** or produced. An observation never overwrites a declaration.

## Where each thing goes

| Data | Home | How it grows |
|---|---|---|
| Declared preferences — table keys and preference areas alike | `~/Clawic/data/k8s/config.yaml` | Key by key, read-modify-write |
| Cluster topology, workloads, incident shapes, accepted gaps, habits, due dates, box index | `~/Clawic/data/k8s/memory.md` | Rewritten in place; stays small |
| Clusters and named machines | `~/Clawic/data/servers/servers.md` (**shared**) | One row per cluster, every provider in one inventory |
| Hostnames the cluster serves, with TLS issuer and expiry | `~/Clawic/data/domains/domains.md` (**shared**) | One row per hostname |
| Cluster detail past what one line holds — versions, node pools, CNI/CSI, operators and CRD versions, quotas | `## Clusters` in `memory.md` while it fits; `~/Clawic/data/k8s/clusters.md` after the split | One block per cluster |
| Per-workload facts that cost a traffic cycle to measure — observed peak memory, p90 CPU, real drain time, boot budget, SLO, owning namespace | `## Workloads` in `memory.md`, then `~/Clawic/data/k8s/workloads.md` | One row per workload |
| Recurring failure shapes and what actually fixed them | `## Incident History` in `memory.md`, then `~/Clawic/data/k8s/incidents.md` | One row per shape, updated in place when it recurs |
| Things you produced that get re-read — runbooks, a NetworkPolicy or Role that finally worked, architecture decisions, capacity plans, upgrade plans | `~/Clawic/data/k8s/artifacts/<kebab-name>.md` | Born as its own file, from the first one |
| Deploy records, cluster upgrades, and timed restore drills | `~/Clawic/data/k8s/deploys/<year>.md` | Append-only, cut by year |
| **Anything durable this table does not name** | `~/Clawic/data/k8s/<plural-noun>.md`, or `artifacts/<kebab-name>.md` if it is a long text read whole | Name the file after what it holds, never after when it was made; add its `## Boxes` line in the same turn |
| Kubeconfigs, tokens, kubeconfig contents, Secret values | Nowhere under `~/Clawic/data/` | Pointer only — see Secrets |

## When to write

No permission needed, no announcement beyond one line.

| It happened | Write |
|---|---|
| A cluster was discovered, created, upgraded or retired | Its row in `servers.md`, plus `## Clusters` |
| A node pool, CNI, CSI driver, or operator version was established | `## Clusters` |
| A workload was sized from real observation, or a VPA recommendation was applied | Its row in `## Workloads` |
| A real drain time, boot budget, or measured RTO came out of the work | `## Workloads` for a workload, `deploys/<year>.md` for a drill |
| An incident's cause was finally named | `## Incident History` — and a runbook in `artifacts/` if it will recur |
| A deploy shipped, or a rollback was needed | `deploys/<year>.md`, with the rollback digest |
| A cluster upgrade was planned or completed | `deploys/<year>.md`, and the new version in `## Clusters` |
| A restore or DR drill was rehearsed and timed | `deploys/<year>.md` under `## Restore Drills`, and the cadence in `## Due` |
| An audit ran (PSA, RBAC, exposure, orphaned PVCs) and something was left unfixed | `## Known Gaps`, with the date and why it was accepted |
| A policy, Role, NetworkPolicy or manifest finally worked after real effort | `artifacts/policy-<name>.md` |
| An architecture decision was made — mesh or no mesh, in-cluster database, cluster granularity, ingress controller | `artifacts/decision-<kebab>.md` |
| A hostname started or stopped being served by the cluster, or its certificate issuer changed | Its row in `domains.md` |
| The user declared a preference | Its key in `config.yaml` |
| Recurring work was scheduled or run | `## Due` |

## Start flat, split only when it hurts

Everything except artifacts, deploy records and the shared boxes begins inside `memory.md`. Splitting is a procedure, not a suggestion:

1. Before appending to a section, count its entries.
2. If the append would take it past **~15 entries or ~40 lines of real content** — scaffolding, headings and comments do not count — then, in the same turn: create the new file in `~/Clawic/data/k8s/`, move the whole section into it, **delete the section from `memory.md`**, add its line to `## Boxes`, and append the new entry to the new file.
3. Keep the headings identical on both sides of the move, so the split is a copy-paste and never a rewrite.
4. Never leave a copy behind. If the same data ever appears in both places, the extracted file wins and the `memory.md` copy is deleted.

Artifacts are the exception: a runbook, a working policy, or a decision is born as its own file whatever its size, because it is read whole and only when its subject comes up.

## Secrets

Nothing under `~/Clawic/data/` ever holds a secret value — not the files named here, not files you create, not text the user pastes in and asks you to keep. A pasted kubeconfig, `kubectl get secret -o yaml` output, a Helm values file, or an incident's command log are the densest sources of secrets in this domain; strip them **before** writing, not after. Store the pointer in place of the value, in this shape: `<kind>:<locator>`.

`env:KUBECONFIG` · `file:~/.kube/prod.yaml` · `keychain:cluster-prod` · `1password:Work/K8s/prod` · `vault:kv/prod/db` · `ssm:/prod/db/password` · `secretsmanager:prod/api/key`

When the user pastes something to save, replace each secret value before writing and leave the pointer visible: `token: <keychain:cluster-prod>`. Say in one line that you did it.

In this domain — **not secrets, keep them**: cluster and context names, namespace and object names, image names and digests, node names and instance types, StorageClass and IngressClass names, ServiceAccount and Role names, CIDRs, cluster version numbers, hostnames, certificate expiry dates. **Secrets, strip them**: kubeconfig `client-key-data`, `client-certificate-data` and `token` fields, ServiceAccount tokens, Secret `data`/`stringData` values (base64 is not encryption), `imagePullSecret` dockerconfigjson, TLS private keys, database URLs carrying a password, webhook and pager tokens, cloud credentials in any Helm values file.

**Contents:** [config.yaml](#configyaml) · [memory.md](#memorymd) · [shared servers inventory](#shared-servers-inventory) · [shared domains inventory](#shared-domains-inventory) · [artifacts/](#artifacts) · [deploys/](#deploys) · [split-out files](#split-out-files)

## config.yaml

Keys come from the Configuration table in `SKILL.md`, plus free-form keys nested under a preference area. Write a key only when the user states the preference, or when an inference from the Configuration signal table has proved right in the work.

**Writing is read-modify-write**: load the existing file, set or replace only the key just declared, keep every other key byte for byte. Never emit a `config.yaml` from this template — the template shows shape, not content. Create `~/Clawic/data/k8s/` if it does not exist.

```yaml
cluster_flavor: eks
manifest_tool: kustomize
gitops_controller: argocd
label_scheme: app.kubernetes.io/*
cpu_limits_policy: none
psa_level: restricted
ingress_controller: nginx
secrets_backend: external-secrets
default_namespace: platform
prod_context: prod-eu
apply_gate: server-dry-run
destructive_confirm: true
explain_depth: normal

# Preference areas — free-form keys added as the user reveals them.
# A preference the user states is a declaration and belongs here, never in memory.md.
conventions:
  namespaces: per-team
  image_tags: git-sha
platform:
  cni: cilium
  node_classes: [spot-batch, ondemand-api]
cadence:
  upgrade_window: "second Tuesday, 20:00-23:00 UTC"
```

If you find a preference recorded in `memory.md`, move it here and note the move.

## memory.md

Write only the sections you have content for — a heading with nothing under it is noise, and it inflates the line count that decides a split. Never copy these hints into the user's file. `## Boxes` is the one section that is never dropped when `memory.md` is rewritten: deleting a line there orphans a file forever. This is what a populated file looks like:

```markdown
# Kubernetes Memory

## Status
status: ongoing
last: 2026-07-26

## Boxes
- Checkout 502 runbook → `artifacts/runbook-checkout-502.md`; read the moment checkout returns 502 or 504
- Ingress NetworkPolicy that finally worked → `artifacts/policy-ingress-egress.md`; read before editing any NetworkPolicy in prod
- Deploys and restore drills (2026) → `deploys/2026.md`; read before a rollback, an upgrade, or an RTO claim
- Workload sizing (22 workloads) → `workloads.md`; read before sizing, before an HPA change, before a quota argument

## Due
| What | Every | Last run | Next due |
|------|-------|----------|----------|
| Restore drill (one namespace from Git, one PVC from snapshot) | quarter | 2026-04-11 | 2026-07-11 |
| Certificate expiry sweep (webhooks, ingress, kubelet) | month | 2026-07-02 | 2026-08-02 |
| Orphaned PVC and unattached LB sweep | month | 2026-06-30 | 2026-07-30 |
| Deprecated-API sweep before the upgrade window | before each upgrade | 2026-05-12 | with 1.32 |
| PSA and RBAC audit | quarter | 2026-05-04 | 2026-08-04 |

## Clusters
prod-eu — eks 1.31, eu-west-1, 3 AZ, 14 nodes (m7g.large spot + ondemand api pool), cilium, EBS CSI, argocd self-heal on. Upgrades owned by platform.
staging — eks 1.31, single AZ, 3 nodes. Same charts, no PDBs.

## Workloads
| Workload | Namespace | Shape | Peak mem observed | p90 CPU | Real drain | SLO |
|---|---|---|---|---|---|---|
| api | prod | Deployment, 6 replicas | 780Mi (2026-06-14) | 340m | 12s | 99.9% |
| checkout | prod | Deployment, 4 replicas | 1.2Gi (2026-07-02) | 600m | 25s | 99.9% |
| postgres | data | StatefulSet, 3 | 6Gi | 1200m | 180s | quorum 2 |

## Incident History
| Shape | Root cause | What fixed it | Last seen |
|---|---|---|---|
| 5s stalls on external API calls | ndots:5 + conntrack race | NodeLocal DNSCache | 2026-03-08 |
| Checkout 502 under low traffic at night | backend keepalive shorter than the proxy's | backend idle 75s > proxy 60s | 2026-07-02 |
| Every create failing cluster-wide | Kyverno webhook, failurePolicy Fail, backend down | namespaceSelector + 2 replicas + PDB | 2026-01-19 |

## Known Gaps
- No default-deny NetworkPolicy in `data` (accepted 2026-05-04: the Postgres operator's own policies were not mapped yet)
- `legacy-jobs` namespace unlabelled for PSA (accepted 2026-05-04, owner left)

## Observed Habits
Acts on availability findings immediately, defers cost findings. Wants the command, not the theory. Reads diffs, not prose.

---
*Updated: 2026-07-26*
```

Rules that keep this readable next month:

- **`## Boxes`**: one line per file that exists — `<what> (<volume>) → <file>; read when <condition>`. Written in the same turn the file is created. Never delete a line without deleting the file it points to. A box with no index line does not exist.
- **`## Due`**: check it against today's date at the start of a session and state any overdue item in one line — a statement, not a question. Every recurring thing this skill schedules belongs here: restore drills, certificate sweeps, orphaned-storage sweeps, the deprecated-API sweep that gates an upgrade, security audits.
- **`## Workloads`**: every number carries the date it was measured, because a peak observed before a dependency change is a guess. Re-measuring overwrites the cell; never a second row for the same workload. These are the numbers Core Rule 10 exists to protect.
- **`## Incident History`** holds *shapes*, not a chronology: when the same shape recurs, update `Last seen` and sharpen the fix rather than appending a row. A shape that recurs three times has earned a runbook in `artifacts/`, and the row then points at it.
- **`## Known Gaps`** records what was found and consciously not fixed, with the date and the reason. Without it, the next audit re-raises every accepted finding and the real ones drown.
- These headings are exactly the ones `clusters.md`, `workloads.md` and `incidents.md` get when their sections outgrow this file, so each split stays a copy-paste.

| Status | Meaning |
|-------|---------|
| `ongoing` | Still learning their clusters and workloads |
| `complete` | Know their topology and conventions well |

## Shared servers inventory

Lives at `~/Clawic/data/servers/servers.md` and is shared with every other infrastructure skill — the user may not have any of them installed, so the format travels with this skill.

```markdown
# Servers

| Name | Provider | Account / Project | Region | Type | Role | Monthly | Access reference |
|------|----------|-------------------|--------|------|------|---------|------------------|
| prod-eu | aws | 111122223333 | eu-west-1 | eks 1.31, 14× m7g.large | k8s cluster (api, checkout, data) | 1900 USD | file:~/.kube/prod.yaml |
| build-1 | hetzner | — | fsn1 | AX41 | bare-metal k3s node | 46 EUR | file:~/.ssh/id_ed25519 |
```

- **Identity is `Name` + `Provider`.** Read the file before adding. If that pair is already there, update the row in place — it is yours. The rule against rewriting protects rows whose `Provider` is not yours; never touch those.
- **One row per cluster, not per node.** Cattle nodes in a managed pool are a property of the cluster's row (`Type` carries the count and instance class). A machine only earns its own row when it is a pet: a single-node k3s box, a bare-metal node, a node someone SSHes into by name.
- **Retirement is part of the inventory.** When a cluster is deleted or a node decommissioned, delete its row and note the date in `memory.md`. An inventory that only grows stops being an inventory.
- **Amounts carry their currency in the value** (`1900 USD`), because Hetzner rows next to yours are in EUR and someone will add the column up.
- **`Monthly` is a planning estimate, not a bill.** The cloud provider's cost report is the source of truth; refresh a row whose real cost moved more than ~20%.
- **Scale cut**: one row per cluster while there are ≤15. Past that, one file per cluster at `~/Clawic/data/servers/<name>.md` with the same fields, and `servers.md` becomes the index (`Name | Provider | Role | → file`). If you arrive and the folder already looks like that, follow it — do not start a parallel `servers.md`.
- **Foreign columns win.** If `servers.md` already exists with a different column set, match its columns and add anything missing as a trailing note. Never rewrite its header.
- Access reference is a pointer only. Never a kubeconfig, token, or key.

## Shared domains inventory

Lives at `~/Clawic/data/domains/domains.md`, shared with DNS, hosting, and registrar skills. Write only the hostnames this cluster serves.

```markdown
# Domains

| Domain | Registrar | Expires | Points to | Notes |
|--------|-----------|---------|-----------|-------|
| app.example.com | — | — | prod-eu ingress-nginx LB (eu-west-1) | TLS: cert-manager LE, renews 2026-09-14 |
```

- **Identity is the hostname.** Read the file before adding; if the row exists, update only the fields you own (`Points to`, and the TLS note) and leave `Registrar` and `Expires` exactly as another skill wrote them.
- **Never invent registrar or domain-expiry data** from cluster objects: a certificate expiry is not a domain expiry, and overwriting one with the other is how a domain silently lapses. Certificate expiry goes in `Notes`, prefixed `TLS:`.
- **Retirement**: when the cluster stops serving a hostname, remove only your `Points to` and TLS note; delete the whole row only if you created it and nothing else in it is owned elsewhere.
- **Foreign columns win**: match the header you find, never rewrite it.
- **Scale cut**: a single `domains.md` table holds them all; past ~40 hostnames, group by apex domain into `~/Clawic/data/domains/<apex>.md` and leave `domains.md` as the index.

## artifacts/

One file per thing, at `~/Clawic/data/k8s/artifacts/<kebab-name>.md`, created the first time it exists. Canonical types here: **runbook**, **policy or manifest that finally worked**, **architecture decision**, **capacity or upgrade plan**. Every artifact opens with when to read it, and gets its `## Boxes` line in the same turn.

```markdown
# Runbook — checkout 502s
*Read when: checkout returns 502 or 504. Written 2026-07-02.*

...steps, with every secret replaced by its pointer...
```

```markdown
# Architecture decision — Postgres stays managed, Kafka moves in-cluster
*Read before any change to the data path, and before sizing the data namespace. 2026-07-26.*

Decision: ...one sentence...
Diagram: ...mermaid or ASCII...
Rejected: in-cluster Postgres — nobody has rehearsed a PITR.
First constraint hit: 3 zones, quorum 2, PDB minAvailable 2.
Estimated monthly: 640 USD, eu-west-1.
```

A NetworkPolicy, Role, or securityContext that took real effort to make work belongs here as `policy-<name>.md`: the YAML with every secret value replaced by its pointer, the date, and what it unblocked. Deriving one costs a full audit cycle; nobody should pay it twice.

## deploys/

```markdown
# Deploys — 2026

| Date | Workload | Image digest / commit | Chart or overlay version | Rollback target | Notes |
|------|----------|-----------------------|--------------------------|-----------------|-------|
| 2026-07-24 | api | sha256:9f2c… / a41b7e | api-1.8.2 | sha256:71ad… | minReadySeconds 20 |

## Cluster Upgrades
| Date | Cluster | From → To | Deprecated APIs swept | Drain time | What broke |
|------|---------|-----------|----------------------|------------|------------|
| 2026-05-12 | prod-eu | 1.30 → 1.31 | 2 charts repinned | 4h10m, 14 nodes | webhook cert expired mid-drain |

## Restore Drills
| Date | What was restored | Measured RTO | What was missing |
|------|-------------------|--------------|------------------|
| 2026-04-11 | `checkout` namespace from Git into a scratch cluster | 38 min | hand-applied Secret, ExternalDNS record |
```

The rollback digest is the reason this file exists: during an incident nobody can reconstruct which digest was last good, and `revisionHistoryLimit` may already have discarded it.

## Split-out files

Created only by the split procedure above, never on day one. Each keeps the exact headings it had inside `memory.md`.

`clusters.md` — one `## <cluster-name>` block per cluster: version, region and zones, node pools, CNI and CSI, ingress controller, installed operators with their CRD versions, namespace and quota layout, who owns upgrades. Written when the fleet passes three clusters or one cluster's detail passes ~40 lines.

`workloads.md` — the `## Workloads` table, unchanged, plus a `## Retired` section so a workload that comes back does not start from zero measurements.

`incidents.md` — the `## Incident History` table, unchanged. The value is the shape-to-fix mapping; keep rows updated in place rather than appending a chronology, and point recurring shapes at their runbook in `artifacts/`.
