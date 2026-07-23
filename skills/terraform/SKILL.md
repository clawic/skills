---
name: terraform
slug: terraform
version: 1.0.2
description: Terraform state surgery, safe refactoring with moved/import/removed blocks, count vs for_each, plan review, and drift control. Use when writing HCL, debugging terraform plan/apply errors, restructuring state or modules, or setting up backends and CI pipelines.
homepage: https://clawic.com/skills/terraform
changelog: "Full coverage pass: deeper guides, situation-named files, and per-user configuration"
metadata:
  clawdbot:
    emoji: 🟪
    requires:
      bins:
      - terraform
    os:
    - linux
    - darwin
    - win32
    displayName: Terraform
---

## When To Use

- Writing or reviewing Terraform/HCL: resource design, count vs for_each, module boundaries
- Debugging plan/apply failures, permanent diffs, stuck state locks, drift
- Refactoring live infrastructure: renames, moving resources between modules or states, importing existing cloud resources
- Designing backends, state layout, and CI plan/apply gates
- Not for choosing the cloud resources themselves — that's `aws`, `gcp`, or `azure`

## Quick Reference

| Situation | Play |
|---|---|
| Renamed a resource or module in code | `moved` block (terraform >=1.1); apply, then delete the block in a later PR |
| Cloud resource exists but not in state | `import` block (>=1.5) + `plan -generate-config-out=gen.tf`; rewrite the draft, merge only when plan is zero-diff |
| Stop managing a resource without destroying it | `removed` block with `destroy = false` (>=1.7); `state rm` only for one-off surgery |
| Permanent diff on every plan | Find the writer first (autoscaler, console operator, other tool); only then `ignore_changes` on that exact attribute |
| Error: "Invalid for_each argument" (unknown at plan) | Key on values known at plan time — input variables or static strings, never resource outputs; two-phase `-target` apply as last resort |
| Stuck state lock | Confirm the holding process is dead (CI job, colleague), then `terraform force-unlock <LOCK_ID>` |
| Drift detection in CI | `terraform plan -detailed-exitcode`: 0 = clean, 2 = drift, 1 = error |
| Resource deleted manually in cloud | Plan shows recreate; if the deletion was intended, add a `removed` block instead of applying |
| Anything else | Smallest possible change; `plan -out=tfplan`; search the diff for "forces replacement" before applying |

## Core Rules

1. The saved plan is the contract: `terraform plan -out=tfplan` → review → `terraform apply tfplan`. A bare `apply` re-plans against a world that may have changed since review — you approve one diff and execute another.
2. Back up before state surgery: `terraform state pull > backup-$(date +%s).tfstate`; restore with `terraform state push`. One wrong `state rm` orphans a live resource that keeps running and billing with nothing tracking it.
3. Read the destroy count out loud. The summary line `X to add, Y to change, Z to destroy` with Z > 0 needs an explanation you could give in an incident review; "forces replacement" in the diff tells you which attribute caused it.
4. Pin everything: providers via `required_providers` + committed `.terraform.lock.hcl`; third-party modules to exact versions. Unpinned means CI breaks on the provider's release day, not on yours.
5. Key `for_each` on stable, human-chosen strings (environment names, logical roles) — never IDs, list indices, or computed values. A changed key is destroy + create of the real resource.
6. Refactor declaratively (`moved`, `removed`, `import` blocks) so the change is visible in the PR diff. `state mv`/`rm` are out-of-band: unreviewable, unreplayable, invisible to the next reader.
7. Nothing sensitive is safe in state: `sensitive = true` masks CLI output, but the value sits in plaintext in the state file. Encrypt the backend at rest, restrict state read access; ephemeral values (>=1.10) and write-only arguments (>=1.11) keep secrets out of state entirely.

## State Management

- Remote backend with locking, always. The S3 backend locks natively via `use_lockfile = true` (terraform >=1.10) — no DynamoDB table needed on current versions.
- Enable object versioning on the state bucket: it is the only undo after a corrupted or wrong `state push`.
- `state rm` is not destroy — the resource keeps existing and billing, just unmanaged. Destroy removes the resource; `removed` blocks let you pick per case.
- `terraform_remote_state` exposes only root outputs in config, but the reader needs read access to the entire state file — secrets included. For cross-stack values, publish to SSM/secret manager instead and read via data source.
- One state = one lock = one apply unit: everyone queues behind it. When routine plans take minutes or a state passes a few hundred resources, split by blast radius (network / data / app tiers), not by team org chart — resources that always change together stay together.

## Count vs for_each

- `count` is positional: removing item 0 renumbers everything after it, and Terraform destroys/recreates real infrastructure to match indices. Reserve `count` for identical replicas and the enable flag: `count = var.enabled ? 1 : 0`, referenced as `one(aws_x.y[*].id)`.
- `for_each` is keyed and stable, but requires a map or set of strings (`toset()` for lists) whose keys are known at plan time — keys built from resource attributes fail with "Invalid for_each argument" (→ Quick Reference).
- Migrating existing `count` resources to `for_each` without `moved` blocks per index destroys and recreates every instance. Write the `moved` blocks first, confirm zero-diff plan, then merge.

## Lifecycle Rules

- `create_before_destroy` is infectious: opting in on one resource forces it onto everything that resource depends on. Expect it to spread through the graph.
- `create_before_destroy` + a fixed unique name (IAM role, security group) = "already exists" error at apply, because both live at once. Use `name_prefix` or a `random_id` suffix.
- `prevent_destroy` only guards while its block exists — deleting the resource block deletes the guard with it. Real deletion protection lives outside Terraform: IAM deny on delete, or provider-level deletion protection flags.
- `ignore_changes` takes a static attribute list (no variables) and hides drift permanently — the attribute becomes unmanaged. Comment why, or the next engineer inherits archaeology.
- `replace_triggered_by` forces recreation when a dependency changes — the clean fix for resources that must rotate together.

## Refactoring, Import & Removal

- The acceptance test for every refactor — rename, module move, import, split — is a no-op plan: `0 to add, 0 to change, 0 to destroy` before the PR merges. Any other result means the refactor changes infrastructure.
- `moved` blocks handle renames, moves into/out of modules, and count→for_each index changes. Leave the block in place through one apply everywhere (all envs, all collaborators), then remove it.
- `import` + `-generate-config-out` produces working but ugly config: every attribute inlined, defaults included. Treat it as a draft — rewrite to your conventions, then verify zero diff.
- `removed { destroy = false }` is the reviewable form of `state rm`: the intent ships in the diff and replays in every environment.

## Modules & Providers

- `depends_on` at module level makes every data source inside the module wait until apply — their results become "known after apply", which cascades into permanently churning plans. Pass explicit values between modules instead.
- Modules used with `count`/`for_each` cannot contain `provider` blocks; providers pass in from root via the `providers` map. Design modules provider-agnostic from the start.
- Thin-wrapper antipattern: a module that renames one resource's arguments adds a version to pin and an indirection to trace, and nothing else. Wrap only when the module encodes real policy (tagging, encryption defaults, naming).
- Two levels of nesting max. Each level must re-export outputs by hand; every extra level is plumbing tax on every new output.
- CI on Linux, laptops on macOS: run `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64` once, or the lock file fails on whichever platform didn't write it.

## Variables & Outputs

- `validation` blocks fail at plan time with your error message — the cheapest place to catch bad input. `precondition`/`postcondition` (>=1.2) check facts across resources; `check` blocks (>=1.5) warn without blocking.
- Untyped variable = `type = any`: type errors surface deep inside expressions instead of at the variable. Always declare types; `nullable = false` to reject null explicitly.
- Secrets never enter committed `.tfvars` — git history is forever. Use `TF_VAR_*` environment variables, secret-manager data sources, or ephemeral values (→ Core Rules 7).

## Traps

| Trap | Why it fails | Do instead |
|---|---|---|
| CLI workspaces for dev/prod separation | Same backend, same credentials; the active env is invisible CLI state — `apply` in the wrong workspace looks identical to the right one | Directory per environment with separate backends and separate cloud roles |
| Routine `-target` applies | Leaves the graph partially applied; the next full plan is a surprise diff nobody scoped | Emergencies only, always followed by a clean full plan |
| Provisioners for configuration | Not idempotent, untracked in state; a failed provisioner taints the whole resource | `user_data`/cloud-init, config management, or `terraform_data` (>=1.4) |
| Hand-editing state JSON | Serial/lineage mismatch corrupts the backend copy — or worse, the push succeeds | `state mv`/`rm`/`push` on a pulled backup (→ Core Rules 2) |
| Treating plan success as apply safety | Plan validates config against state, not against the cloud: quotas, IAM, name collisions, eventual consistency all surface at apply | Apply early in a sandbox account; keep changes small so failures are attributable |
| `apply -auto-approve` outside CI | Removes the only human checkpoint between a typo and deleted production | Auto-approve only in pipelines that apply a reviewed saved plan |

## Where Experts Disagree

- **Vanilla Terraform vs wrapper tooling (Terragrunt, etc.)**: the frontier is duplication — one team with a handful of stacks loses more to wrapper complexity than it saves; once environments × stacks means maintaining dozens of near-identical backend/provider blocks, DRY tooling earns its cost.
- **Exact module pins vs `~>` constraints**: exact pins for third-party registry modules (supply-chain surface); pessimistic minor constraints acceptable for internal modules gated by your own CI.
- **One shared state vs many micro-states**: the frontier is change coupling — resources that always ship together belong in one state; every cross-state reference costs a data-source hop and an ordering problem between pipelines.

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/terraform (install if the user confirms):

- **[aws](https://clawic.com/skills/aws)** — provider-specific resource and service guidance
- **[devops](https://clawic.com/skills/devops)** — pipeline and delivery design around plan/apply gates
- **[github-actions](https://clawic.com/skills/github-actions)** — wiring plan-on-PR / apply-on-merge workflows
- **[ansible](https://clawic.com/skills/ansible)** — configuring what lives inside the instances Terraform creates
- **[k8s](https://clawic.com/skills/k8s)** — workloads on the clusters Terraform provisions

## Feedback

- If useful, star it: https://clawic.com/skills/terraform
- Latest version: https://clawic.com/skills/terraform

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/terraform.
