---
name: Azure
slug: azure
version: 1.0.1
description: >-
  Architects, debugs, secures, and cost-optimizes Azure — VMs, App Service, Functions, AKS, Azure SQL, Cosmos DB,
  Entra ID, VNets, Storage. Use when deploying or reviewing anything on Azure, when a bill jumps or spend has to
  come down, when AuthorizationFailed, an RBAC assignment that has not propagated, a private endpoint resolving to
  a public IP, a 502 from Application Gateway, a 230-second App Service timeout, a 429 from Cosmos DB, SNAT
  exhaustion, or SkuNotAvailable has no obvious cause, when choosing between compute options (App Service,
  Functions, AKS, VMs) or databases (Azure SQL, Cosmos DB), when hardening Key Vault, NSGs, managed identities,
  storage exposure or Entra ID, when writing Bicep, ARM or Terraform against Azure, or when auditing an inherited
  subscription or tenant. Covers VNet/Private Link design, Azure Monitor/KQL, backup/DR, and az CLI context. Not
  for Kubernetes manifest authoring (`k8s`), Terraform language mechanics (`terraform`), or SQL query tuning
  (`sql`).
homepage: https://clawic.com/skills/azure
changelog: "Clearer data layout: what to read, what to save, and where"
metadata:
  clawdbot:
    emoji: 🔷
    requires:
      anyBins:
      - az
    os:
    - linux
    - darwin
    - win32
    displayName: Azure
    configPaths:
    - ~/Clawic/data/azure/
    - ~/Clawic/data/servers/
    - ~/Clawic/data/domains/
    - ~/Clawic/data/contacts/
---

**Data.** At the start of every session, read `~/Clawic/data/azure/config.yaml` (what the user declared) and `~/Clawic/data/azure/memory.md` (what you observed, plus its `## Boxes` index and `## Due` table). Open any file `## Boxes` names when the condition on its line applies — the index is the list of files, never assume the list is fixed. Read `~/Clawic/data/servers/servers.md` before any deploy, sizing, or "what do I have" question, and `~/Clawic/data/domains/domains.md` before touching a custom domain, DNS zone, or certificate. If none of it exists, work from defaults and say nothing about it.

**Write before the session ends** whenever it produced something durable: a VM or scale set created, resized, discovered or retired; an inventory pass; a spend number or a saving; a budget or alert; a subscription and who pays for it; a custom domain, certificate or credential with an expiry date; a deploy or a timed restore drill; a KQL query worth running again; or something the user will want to read again — a runbook, a custom role or policy that finally worked, an address plan, an architecture decision. `memory-template.md` has every destination, format and threshold, and is the only file you open to write.

**A VM or VM scale set is a host: it goes to the shared inventory `~/Clawic/data/servers/servers.md`**, not here, because the same file holds machines from every provider and "what servers do I have" has to answer itself. One row per host, identified by `Name` + `Provider` — update your own row in place, never append a second one: `name | provider (azure) | subscription/resource group | region | size | role | monthly cost with currency | access reference`. Managed platform resources (App Service plans, AKS clusters, SQL servers, storage accounts) are not hosts — they belong in `## Current Infrastructure`.

**No credential is ever written anywhere under `~/Clawic/data/`** — not in these files, not in a file you create, not in text the user pastes in to be saved. Store the pointer and strip the value: `azure-kv:kv-prod/db-password`, `env:AZURE_CLIENT_SECRET`, `keychain:azure-prod`, `1password:Work/Azure/prod`.

Azure gives you five ways to run a container and four places the same setting can be overridden. Pick one, name the monthly number, and say what the blast radius is. Reach for the cheapest thing that meets the requirement, and say when a cheaper thing would not. Work from defaults immediately: never open with questions about their tenant, their budget, or how proactive to be. The exceptions to silence are `default_subscription` and `default_location` — while either is unset, state which one you are assuming before acting (Rule 7). That is a statement, not a question. Precedence for any value: `config.yaml` → `~/Clawic/profile.yaml` (shared universals: currency, locale) → the Configuration table default.

## When To Use

- Architecting or deploying on Azure: service selection, VNet layout, subscription and management-group layout, infrastructure as code
- Diagnosing an Azure failure whose cause is not obvious: `AuthorizationFailed`, a private endpoint that resolves publicly, 502/504, a function that never fires, an allocation or quota error, intermittent timeouts under load
- A bill that jumped, or spend that has to come down without breaking anything
- Security work: RBAC and Entra ID, managed identities, Key Vault, storage and network exposure, credential hygiene, auditing an inherited subscription
- Operating what is already live: scaling, zone failover, backups and restores, upgrades, quotas, deploy and rollback, patching
- Moving in: assessing an on-prem or AWS estate, mapping it to Azure services, planning the cutover
- Not for Kubernetes manifest authoring (`k8s`), Terraform language mechanics (`terraform`), or SQL query and index tuning (`sql`) — this covers the Azure-platform side of all three

## Quick Reference

| Situation | Play | Depth |
|-----------|------|-------|
| "My bill exploded" | Cost Analysis by service, then by resource; the delta's start date maps to a deploy | `costs.md` |
| Fresh subscription, nothing deployed yet | Budget + anomaly alert before the first resource, then the Rule 3 stage table | `costs.md` |
| Inherited subscription or tenant | Resource Graph inventory (Rule 1), then the audit checklist top to bottom | `security.md` |
| `AuthorizationFailed` on a call that should work | Classify first: wrong scope, wrong subscription context, propagation, data-plane role, or deny assignment | `identity.md` |
| Private endpoint, DNS, NSG, or "it resolves to a public IP" | Walk the path: DNS zone link → NSG → route → service firewall → probe source | `networking.md` |
| 502/504, timeouts, or requests that hang | The layer that emits the code names the subsystem; 230s is always App Service | `debug.md` |
| Function cold, timing out, or not triggering | Hosting plan, the storage account it depends on, trigger scaling, poison queue | `functions.md` |
| Web app slow, slot swap broke prod, cert expiring | Plan sizing, Always On, swap semantics, sticky settings, managed certificates | `appservice.md` |
| AKS, Container Apps, ACI, or a pull that fails | Cluster sizing, IP exhaustion, upgrade windows, ACR auth path | `containers.md` |
| VM sizing, spot, disks, "SkuNotAvailable", maintenance | Family choice, deallocate vs stop, zones, eviction and freeze notices | `vms.md` |
| Database out of connections, throttled, failing over | Tier semantics, connection policy, RU math, HA and restore | `databases.md` |
| Blob costs, tiers, SAS, redundancy, throughput ceilings | Early-deletion fees, account-level throttles, user delegation SAS | `storage.md` |
| Choosing between two services | Decide by hard limit and break-even, never by feature list | `services.md` |
| Alerts, logs, KQL, or "why is Log Analytics so expensive" | Diagnostic settings, sampling, DCR transformations, alert types | `monitoring.md` |
| Template fails, drift, locks, or Policy blocks a deploy | What-if, deployment modes, stacks, provider registration, effects | `iac.md` |
| Second subscription, tenant, landing zone, tags, quotas | Management groups, billing model, Policy inheritance, move rules | `governance.md` |
| Taking it to production | Zones and SLA math, backup and restore drills, failover, patching | `production.md` |
| Moving from on-prem or AWS | Assessment, service mapping, data transfer, cutover checklist | `migration.md` |
| Need the exact CLI invocation | Context, JMESPath, Resource Graph, what-if, `--no-wait`, extensions | `commands.md` |
| Anything else Azure | Answer directly, then state the monthly cost and the blast radius of what you recommended | — |

Coverage map: `debug.md` symptom→cause · `identity.md` Entra ID and RBAC · `networking.md` VNet, Private Link, egress · `costs.md` bill control · `security.md` hardening and leaks · `services.md` selection thresholds · `functions.md` serverless · `appservice.md` web apps and certificates · `containers.md` AKS/Container Apps/ACR · `vms.md` compute and disks · `databases.md` SQL/PostgreSQL/Cosmos/Redis · `storage.md` Blob/Files/disks · `monitoring.md` Azure Monitor and KQL · `iac.md` Bicep/ARM/Terraform · `governance.md` subscriptions, Policy, quotas · `production.md` reliability · `migration.md` moving in · `commands.md` az toolkit.

## Core Rules

1. **Inventory before architecture.** Never propose infrastructure into an unknown subscription — and never rediscover one you already mapped. Read the stored inventory first: `## Current Infrastructure` in `memory.md`, whatever its `## Boxes` line points to, and `~/Clawic/data/servers/servers.md`. Then discover only what is missing or older than the last recorded pass, and write the result back. Minimum discovery: `az account show`, one Azure Resource Graph query counting resources by type and location, and 30 days of Cost Analysis grouped by service (`commands.md`). Resource Graph answers in seconds what a portal tour never finishes, and spend maps the estate faster than any diagram because anything real costs something.
2. **Budget before the first resource.** On a fresh subscription the first deployment is a budget plus an anomaly alert, not a VM. Thresholds: alert at 80% of budget actual and 100% forecast; set the anomaly threshold near *daily* spend — `monthly_budget ÷ 30`, so a 100/mo budget alerts at ~3, not 50 — because anomalies are daily events and a monthly-sized threshold never fires. EA and CSP scopes put budget creation in a different place than pay-as-you-go (`costs.md`).
3. **A monthly number with every recommendation.** Rough stages (West Europe, pay-as-you-go, list price — verify before committing money):

   | Stage | Recommended stack | Monthly |
   |-------|-------------------|---------|
   | MVP (<1k users) | App Service B1 + Azure SQL serverless (auto-pause) + Storage | ~40 |
   | Growth (1-10k) | App Service P1v3 + SQL GP serverless + Front Door Standard | ~300 |
   | Scale (10k+) | Container Apps or AKS + SQL Business Critical/Hyperscale + Redis + Front Door Premium | 1,000+ |

   Default to the smallest viable SKU: scaling up an App Service plan or a SQL vCore takes minutes, an oversized fleet bleeds silently. Right-sizing heuristic (canonical for this skill): avg CPU <20% over 14 days → step down one size (each step ≈ halves compute cost); sustained >70% → step up or scale out. On B-series burstable VMs and App Service, CPU percentage lies once credits are exhausted — check the credit metric before concluding the workload is small (`vms.md`).
4. **Smallest blast radius, decided at creation.** RBAC at the narrowest scope that works (resource or resource group, never subscription-wide Contributor); data services behind private endpoints; storage with shared-key auth disabled; encryption on by default. Azure has a long list of choices with no in-place undo — Cosmos DB partition key, storage account redundancy across some combinations, SQL Hyperscale (one-way), AKS network plugin and node subnet, VNet address space that overlaps a future peer, and the region itself. "Later" on any of them is a decision to migrate, not a setting to change.
5. **Everything in code, nothing portal-only.** The portal is for exploration and for reading; anything that survives the session goes into the tool named by `iac_tool`. Preview before every change — `what-if` for Bicep/ARM, `plan` for Terraform — and read the output: incremental mode never deletes what you removed from the template, so drift accumulates silently until someone runs Complete mode and takes production with it (`iac.md`).
6. **Tag at creation; Azure does not inherit tags.** A resource does not get its resource group's tags — inheritance only exists if a Policy `Modify` rule creates it, and Policy only fixes existing resources when a remediation task runs. Cost allocation reports from tags forward, so an untagged month is unattributable forever. Minimum set: `Environment`, `Workload`, `Owner`, `CostCenter`.
7. **Subscription and region are decisions, not ambient context.** The most expensive Azure mistake is a correct deployment into the wrong subscription because the CLI context was left where the last task put it. Name the subscription and the location in every command and every price quote; a West Europe price quoted for a workload in Brazil South is a wrong number, not a rounded one. While `default_subscription` or `default_location` is unset, say which one you are assuming before acting.
8. **Name the first quota and the first timeout.** Every design states which limit it hits first and its current value (`az vm list-usage`, the Quotas blade). Defaults that bite earliest: App Service HTTP requests die at 230s no matter what the app does; new subscriptions ship with a low per-family vCPU quota per region; outbound load-balancer SNAT gives 1,024 ports per instance by default; a Cosmos DB logical partition stops at 20 GB; a subscription takes 4,000 role assignments. A design that has not named its ceiling has not been designed.

## Failure Signatures

Decode rule: the layer that emits the error names the subsystem. An HTTP status from a front door or gateway is about the *connection*; an ARM error string is about *permissions, quota or state*; a bare timeout is about *routing or SNAT*. The Activity Log plus the correlation ID from the failed operation is the ground truth for every control-plane call.

| Signature | Most likely cause | First move |
|---|---|---|
| `AuthorizationFailed` on a role you can see in the portal | Assignment at the wrong scope, the CLI pointed at another subscription, or the role grants control plane and you need data plane | Classification table in `identity.md` — never widen the role first |
| Access works in the portal, fails from code | Data-plane role missing (Contributor ≠ Storage Blob Data Contributor) | Assign the data role; portal uses your user, code uses the identity |
| Role assigned, still denied 20 minutes later | Propagation, documented at up to 30 minutes | Wait it out or force a fresh token; do not stack more assignments |
| Private endpoint created, app still hits the public IP | Private DNS zone missing, not linked to the VNet, or overridden by custom DNS | DNS chain in `networking.md`; resolve the FQDN from inside the VNet before touching anything else |
| Intermittent connection timeouts only under load | SNAT port exhaustion on the default outbound path | NAT Gateway on the subnet; 1,024 ports per instance is the default ceiling |
| Application Gateway 502 | Backend probe failing, or host-name mismatch on the backend setting | Backend health first, then `pickHostNameFromBackendAddress`, then NSG to the backend |
| Request dies at exactly ~230s | App Service front-end idle timeout — not configurable | Make the work asynchronous (`appservice.md`) |
| HTTP 429 from Cosmos DB or Storage | RU/s or account request-rate throttle, not capacity | Read the RU charge and `x-ms-retry-after-ms`; fix the partition or the indexing policy (`databases.md`) |
| `SkuNotAvailable`, `AllocationFailed`, `ZonalAllocationFailed` | Regional or zonal capacity for that family, not your quota | Another zone, another size in the family, or Flexible orchestration (`vms.md`) |
| `QuotaExceeded` / `OperationNotAllowed` on a deploy | Per-family vCPU quota in that region | `az vm list-usage`, then request headroom (Rule 8) |
| `MissingSubscriptionRegistration` | Resource provider not registered in this subscription | Register the provider; it is per subscription, not per tenant (`iac.md`) |
| VM disappeared or rebooted with no deploy | Spot eviction (30-second notice) or platform maintenance | Scheduled Events via IMDS records both (`vms.md`) |
| Function stopped firing, no errors | Its storage account is unreachable or its keys rotated | The host state lives in that account (`functions.md`) |
| `ImagePullBackOff` on AKS | ACR not attached, private endpoint DNS, or missing AcrPull role | Pull path in `containers.md` |
| Alert never fired during a real outage | Log alert frequency, or the metric stopped publishing entirely | Metric alerts on saturation, log alerts for absence (`monitoring.md`) |
| Anything else | Find the operation and correlation ID in the Activity Log, then match it here | `debug.md` |

## Limits That Force Designs

Architecture constraints, not trivia: each one has killed a design that was already half-built. Verify a limit before betting a quarter on it — the ceiling moves, the shape of the constraint does not.

| Service | Limit that decides the design |
|---|---|
| App Service | 230s hard HTTP timeout · plan is the scale unit, and Linux and Windows apps cannot share one · Always On is unavailable on Free/Shared, so the app unloads when idle |
| Functions | Consumption: 5 min default timeout, 10 min max, no VNet · Premium/Flex: 30 min default, longer configurable · one storage account is a hard dependency |
| Cosmos DB | 20 GB per logical partition (the partition key value, not the container) · 2 MB item · partition key immutable · autoscale bills at 1.5× the manual rate and floors at 10% of max |
| Azure SQL | Hyperscale is one-way · connection limits scale with the SKU, not a lookup table · serverless auto-pause needs a minimum idle delay of 1 hour |
| Storage account | The account, not the container, is the throttle boundary (~20,000 requests/s standard) · cool 30-day, cold 90-day, archive 180-day minimum retention with early-deletion fees |
| Managed disks | Size and IOPS come as tiers that round up (Premium SSD v2 is the exception) · a disk cannot shrink |
| VNet / subnet | Azure reserves 5 IPs per subnet · address space cannot overlap a peer · peering is non-transitive · a subnet in use cannot be resized down |
| AKS | Azure CNI (non-overlay) consumes a VNet IP per pod: `nodes × (max_pods + 1)` must fit the subnet · a minor version leaves support roughly 12 months after GA |
| Load Balancer | 1,024 SNAT ports per instance by default; 4-minute idle timeout on outbound flows |
| Subscription | 4,000 role assignments · 980 resource groups · 800 deployments of history per resource group (older ones must be purged before a deploy fails) |
| Key Vault | Soft-delete is mandatory (90 days) — the name is unusable until purged; purge protection, once on, cannot be turned off |
| Entra ID | A subscription moved to another tenant loses every role assignment and every system-assigned managed identity (`governance.md`) |

## Cost Reflexes

The ten line items that produce most surprise bills. Prices: West Europe, pay-as-you-go, recorded early 2026 — the **ratios are stable, the absolute numbers need verifying** (`costs.md` has the commands and the savings playbook).

| Driver | Why it bites | Do instead |
|------|--------------|------------|
| Azure Firewall on a small VNet | Standard runs ~1.25/hr (~900/mo) before a byte flows | NSGs plus a NAT Gateway cover most VNets; Firewall Basic exists for the rest (`networking.md`) |
| Front Door Premium chosen for WAF | Premium's base fee is ~10× Standard's | Standard + WAF policy unless you need Private Link origins or managed rule depth |
| Application Gateway / VPN Gateway / Bastion left up | Each is a fixed hourly resource that bills at zero traffic | Bastion Developer or just-in-time VM access for occasional admin (`security.md`) |
| Stopped VMs | Portal "Stop" leaves the VM allocated: compute still bills | Deallocate, and remember disks and static public IPs bill regardless (`vms.md`) |
| Log Analytics ingestion | Priced per GB ingested, an order of magnitude above storage; verbose app logs and AKS container stdout dominate | Daily cap, table-level Basic Logs, and a DCR transformation that drops the noisy columns (`monitoring.md`) |
| Cosmos DB provisioned RU/s | Bills whether or not anything queries, and a default indexing policy indexes every path | Serverless for bursty, autoscale with a sane floor, exclude unqueried paths (`databases.md`) |
| Orphaned disks, NICs and public IPs | Deleting a VM does not delete them unless that was set at creation | Monthly sweep for unattached disks and idle public IPs (`costs.md`) |
| Premium SSD by default | The portal defaults new VM disks to Premium; dev workloads rarely notice Standard SSD | Match the disk tier to the actual IOPS requirement, then check burst credits |
| Cross-region and peering traffic | Inter-region egress bills, and VNet peering bills on both sides of every hop | Co-locate chatty pairs; Private Link for cross-region access instead of a chatty peer |
| Reservations and savings plans bought before right-sizing | Locks the oversized fleet in for 1-3 years | Right-size (Rule 3), observe two weeks, then commit — and put the term end date in `## Due` |

## Security Baseline

Non-negotiables. Anything unchecked here outranks whatever feature work is in flight; commands and the full audit are in `security.md`.

| Check | Passing looks like |
|---|---|
| Global Administrator accounts | Few, MFA-enforced, and not used for daily work; two break-glass accounts excluded from Conditional Access and monitored |
| Human access | Entra ID with Conditional Access and MFA; privileged roles eligible-only through PIM, not standing |
| Workload credentials | Managed identity everywhere it exists; federated credentials for CI; client secrets only where nothing else works, with their expiry in `## Due` |
| Storage | Shared-key auth disabled, public blob access off at the account, SAS from user delegation with a short expiry |
| Key Vault | RBAC data plane (not legacy access policies), purge protection on for anything holding production keys, private endpoint or a firewall that is not "all networks" |
| Inbound | No `*`/`Internet` source on 22, 3389, 1433, 3306 or 5432; admin access through Bastion or just-in-time |
| Azure SQL firewall | The "Allow Azure services" rule (0.0.0.0) is off — it admits resources from *other tenants*, not just yours |
| Data at rest | Platform encryption everywhere; encryption at host or customer-managed keys where the regime requires it |
| Audit trail | Diagnostic settings ship Activity Log and the noisy-but-critical resource logs to a workspace; Defender for Cloud enabled at least at the free tier |

## Service Defaults

One default per need, with the escape hatch. Thresholds and break-evens: `services.md`.

| Need | Default | Switch when |
|------|---------|-------------|
| Web app | App Service (Linux) | You need per-request scale-to-zero (→ Container Apps) or Kubernetes primitives (→ AKS) |
| Event-driven code | Functions on Flex Consumption | Sustained duty cycle, or a need for long-running orchestration you already run in containers |
| Containers, no Kubernetes needed | Container Apps | The team runs operators, service meshes, or multi-tenant namespaces (→ AKS) |
| Relational database | Azure SQL (serverless for small, GP for steady) | Open-source engine, extensions like pgvector, or cost-per-vCore matters more (→ PostgreSQL Flexible Server) |
| Global, key-based access | Cosmos DB | Anything you would join, filter or report on (→ Azure SQL) |
| Cache / sessions | Azure Cache for Redis Standard | Persistence, clustering, or private endpoint required (→ Premium) |
| Queue | Service Bus queue | Cheapest possible with no ordering or sessions (→ Storage Queue); high-throughput streams (→ Event Hubs); reactive fan-out (→ Event Grid) |
| Global HTTP entry | Front Door Standard | Regional-only traffic with backend autoscaling (→ Application Gateway) |
| Secrets | Key Vault with RBAC | App settings that are not secret belong in configuration, not a vault |
| Templates | Whatever `iac_tool` says | — |

## Output Gates

Before delivering an architecture, a policy, a template, or a command:

- Did I state the monthly cost of what I recommended, in the region and subscription it will actually run?
- Did I check the stored inventory *and* the live subscription before proposing something new?
- Is anything holding data reachable from the public internet, or open to "all Azure services"?
- Does this design name the first quota and the first timeout it will hit (Rule 8)?
- Is anything that cannot be changed later — partition key, redundancy, network plugin, address space, region, Hyperscale — set correctly at creation?
- Is any command destructive (delete, purge, Complete-mode deploy, `--force`)? Then it ships with an explicit confirmation step, never inside a copy-paste block of read-only commands.
- **Did anything durable come out of this session — a host, a spend number, an expiry date, a decision, a runbook, a query worth keeping? Then it is written to its box, and any new box has its `## Boxes` line, before I finish.**

## Configuration

User-dependent variables. Defaults apply until the user states a preference; store them in `~/Clawic/data/azure/config.yaml`.

| Variable | Type | Default | Effect |
|---|---|---|---|
| default_subscription | text (name or id) | none | Subscription every command, quote and deployment assumes; while unset, name the assumed subscription before acting (Rule 7) |
| default_location | text (region) | none | Region for deployments, price quotes and zone advice; while unset, name the assumed region |
| iac_tool | bicep \| terraform \| arm \| none | bicep | Language of every generated template and the preview command in `iac.md` (Rule 5) |
| monthly_budget | number (profile currency, USD if unset) | 100 | Budget and anomaly thresholds in `costs.md` (Rule 2) and the bar for calling a recommendation expensive |
| tenancy_model | single-subscription \| management-group | single-subscription | Whether guidance uses one subscription plus budgets, or management groups, Policy and landing zones (`governance.md`) |
| billing_model | payg \| mca \| ea \| csp \| devtest | payg | Where cost data and reservations live, which discounts apply, and who can create a budget (`costs.md`) |
| compliance_regime | none \| pci \| hipaa \| soc2 \| fedramp | none | Restricts selection to eligible SKUs and forces the logging, encryption and residency defaults that regime requires |
| cloud_environment | AzureCloud \| AzureUSGovernment \| AzureChinaCloud | AzureCloud | Endpoints, region names and service availability; sovereign clouds lag features and some services never arrive |
| naming_pattern | text | `<abbr>-<workload>-<env>-<region>-<nn>` | Shape of every resource name generated, and which abbreviations are used (`governance.md`) |

Preference areas — customizable dimensions; a stated preference gets recorded in `config.yaml` and applied from then on:

- **Tooling** — Bicep module style (registry vs local), `az` vs the Az PowerShell module, portal vs CLI for exploration, Terraform `azurerm` vs `azapi` for preview resources — affects every command example
- **Conventions** — tag keys beyond the required four, resource-group granularity (per-workload vs per-lifecycle), naming abbreviations, address-plan scheme — affects generated resources and `networking.md`
- **Platform** — home region and its pair, zone posture, VM families the org standardizes on, ARM64 vs x86, Linux vs Windows hosting — affects sizing advice and price quotes
- **Safety posture** — resource locks by default, purge protection, whether destructive commands are emitted at all, appetite for Complete-mode deployments — affects Output Gates and `production.md`
- **Cost reporting** — review cadence, currency for quotes, showback dimension (resource group vs tag vs subscription) — affects `costs.md`
- **Service preferences** — the standing pick where this skill offers a default (App Service vs Container Apps, SQL vs PostgreSQL, Service Bus vs Storage Queue) — affects Service Defaults and `services.md`

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| Treating a resource group as a security boundary | It is a lifecycle and RBAC-scope container with no network isolation whatsoever | NSGs, private endpoints and subscription boundaries do isolation (`networking.md`) |
| Granting Contributor to fix an access problem | It usually does not fix it — data-plane access is a separate role family — and now the fix is permanent | Classify the denial first (`identity.md`) |
| Client secrets for service-to-service auth | They expire on a date nobody diarized, and rotating them is an outage with a calendar date attached | Managed identity, or federated credentials for CI; if a secret is unavoidable, its expiry goes in `## Due` |
| "Allow Azure services to access this server" on SQL or Storage | That checkbox admits resources belonging to other tenants, not just yours | Private endpoint, or an explicit VNet rule (`security.md`) |
| Deleting a VM in the portal and assuming it is gone | Disks, NICs and public IPs survive unless delete-with-VM was set at creation | Sweep unattached disks and idle IPs monthly (`costs.md`) |
| Portal hotfix on IaC-managed resources | The next deployment silently reverts it, usually mid-incident | Fix in code; preview before every change (Rule 5) |
| Complete-mode deployment at subscription or resource-group scope | It deletes every resource not in the template, including ones another team owns | Incremental plus deployment stacks with deny settings (`iac.md`) |
| Azure CNI with a small node subnet | Every pod takes a VNet IP; the cluster stops scaling at an IP ceiling nobody computed | Compute `nodes × (max_pods + 1)` before creating, or use CNI Overlay (`containers.md`) |
| Testing DR by reading the runbook | Restores fail on details nobody wrote down: key vault access, firewall rules, connection strings, DNS | Restore into a scratch resource group quarterly and time it (`production.md`) |
| Quota increase requested during the incident | Regional capacity approvals are not instant, and zonal capacity may not exist at any price | Request headroom before the launch (Rule 8) |
| Reading spend from the current month's Cost Analysis total | It lags and hides the per-resource breakdown that names the cause | Group by service then by resource, and compare closed months only (`costs.md`) |
| One giant resource group for everything | 800-deployment history limits, and a lock or a Policy denial blocks unrelated work | Split by lifecycle: platform, data, application (`governance.md`) |
| Enabling every Defender plan on day one | Per-resource pricing across servers, storage and databases can outspend the workload | Free tier and secure score first, paid plans where there is something worth detecting (`security.md`) |

## Where Experts Disagree

- **Bicep vs Terraform.** The frontier is who owns state and how much you live outside Azure: Bicep needs no state file, gets same-day support for new resource types, and its `what-if` reads the live resource graph; Terraform wins with multi-cloud footprints, mature module reuse, and teams who already have remote-state discipline. Both lose to portal clicking (`iac.md`).
- **AKS vs Container Apps.** Below roughly four always-on containers with no operator requirements, Container Apps wins on total cost and on the hours nobody spends upgrading a control plane; above that, or when the team needs CRDs, meshes or multi-tenant namespaces, AKS is not a preference but a requirement. The break-even math, not the taste, decides (`services.md`).
- **Hub-and-spoke with Azure Firewall.** The reference architecture is genuinely right for regulated estates and genuinely disproportionate for a two-subscription startup, where the firewall can cost more than the workloads it inspects. Start with NSGs plus NAT Gateway and a documented address plan; the migration to a hub is cheap if the address plan was written down, expensive if it was not.
- **Subscription per environment vs resource group per environment.** Subscriptions give hard quota and policy isolation and clean cost attribution; resource groups keep the estate small and avoid the tax of multiplying every shared resource. Any compliance regime, or any team above a handful of engineers, ends up at subscriptions — moving live resources across them later is a migration project, not a setting (`governance.md`).

## Security & Privacy

**Credentials:** this skill drives the Azure CLI, which reads credentials from `~/.azure/`, the OS keychain, environment variables, or a managed identity. It does NOT store, log, copy, or transmit Azure credentials, and never writes a credential into `~/Clawic/data/azure/`.

**Local storage:** preferences, memory, inventory, spend history and artifacts stay in `~/Clawic/data/azure/` on this machine — subscription and tenant identifiers, resource names, and cost figures only, no secrets.

**Guardrails:** commands are read-only by default. Destructive operations (delete, purge, Complete-mode deployment, lock removal) are presented with their blast radius and require explicit user confirmation before running.

## Related Skills
More Clawic skills, get them at https://clawic.com/skills/azure (install if the user confirms):
- `terraform` — HCL authoring, state surgery, module design
- `k8s` — Kubernetes manifests and cluster debugging, for AKS workloads
- `aws` — the other side of a multi-cloud estate, and the source estate in most migrations
- `sql` — query and index tuning for Azure SQL and PostgreSQL Flexible Server
- `infrastructure` — provider-agnostic architecture decisions

## Feedback

- If useful, star it: https://clawic.com/skills/azure
- Latest version: https://clawic.com/skills/azure

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/azure.
