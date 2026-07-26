# Working File Templates — AWS

Read this file only when WRITING. `config.yaml` is what the user **declared**; `memory.md` and everything it indexes is what you **observed** or produced. An observation never overwrites a declaration.

## Where each thing goes

| Data | Home | How it grows |
|---|---|---|
| Declared preferences — table keys and preference areas alike | `~/Clawic/data/aws/config.yaml` | Key by key, read-modify-write |
| Account context, infrastructure, pain points, how they work, spend, due dates, box index | `~/Clawic/data/aws/memory.md` | Rewritten in place; stays small |
| Hosts and managed instances | `~/Clawic/data/servers/servers.md` (**shared**) | One row per host, every provider in one inventory |
| AWS accounts and who owns them | `## Account Context` in `memory.md` while there is one; `~/Clawic/data/aws/accounts.md` from the second | One row per account |
| Things you produced that get re-read — runbooks, policies that finally worked, architecture decisions, diagrams | `~/Clawic/data/aws/artifacts/<kebab-name>.md` | Born as its own file, from the first one |
| Deploy records and timed DR drills | `~/Clawic/data/aws/deploys/<year>.md` | Append-only, cut by year |
| **Anything durable this table does not name** | `~/Clawic/data/aws/<plural-noun>.md`, or `artifacts/<kebab-name>.md` if it is a long text read whole | Name the file after what it holds, never after when it was made; add its `## Boxes` line in the same turn |
| Credentials of any kind | Nowhere under `~/Clawic/data/` | Pointer only — see Secrets |

## When to write

No permission needed, no announcement beyond one line.

| It happened | Write |
|---|---|
| A host was created, resized, discovered or retired | Its row in `servers.md` |
| An inventory pass ran (Rule 1) | `## Current Infrastructure` |
| A bill was reviewed, or a saving landed | `## Spend` |
| A budget or anomaly subscription was created | `### Alerts Configured` |
| An account was added, or its owner or billing named | The accounts table |
| A deploy shipped, or a DR drill was timed | `deploys/<year>.md` |
| A runbook, a working policy, or an architecture decision came out of the session | `artifacts/` |
| The user declared a preference | Its key in `config.yaml` |
| Recurring work was scheduled or run | `## Due` |

## Start flat, split only when it hurts

Everything except artifacts, deploy records and the shared inventory begins inside `memory.md`. Splitting is a procedure, not a suggestion:

1. Before appending to a section, count its entries.
2. If the append would take it past **~15 entries or ~40 lines of real content** — scaffolding, headings and comments do not count — then, in the same turn: create the new file in `~/Clawic/data/aws/`, move the whole section into it, **delete the section from `memory.md`**, add its line to `## Boxes`, and append the new entry to the new file.
3. Keep the headings identical on both sides of the move, so the split is a copy-paste and never a rewrite.
4. Never leave a copy behind. If the same data ever appears in both places, the extracted file wins and the `memory.md` copy is deleted.

Artifacts are the exception: a runbook or a decision is born as its own file whatever its size, because it is read whole and only when its subject comes up.

## Secrets

Nothing under `~/Clawic/data/` ever holds a secret value — not the files named here, not files you create, not text the user pastes in and asks you to keep. Store the pointer in its place, in this shape: `<kind>:<locator>`.

`ssm:/prod/db/password` · `secretsmanager:prod/api/key` · `env:AWS_PROFILE` · `keychain:aws-prod` · `1password:Work/AWS/prod` · `profile:prod` · `file:~/.ssh/id_ed25519`

When the user pastes something to save, replace each secret value before writing and leave the pointer visible: `password: <ssm:/prod/db/password>`. Say in one line that you did it.

In this domain — **not secrets, keep them**: account IDs, ARNs, role and policy names, bucket and VPC ids, instance ids, region ids, profile names. **Secrets, strip them**: access key ids and secret access keys, session tokens, passwords, private keys and passphrases, connection strings carrying a password, `sts:ExternalId`, webhook and pager tokens.

**Contents:** [config.yaml](#configyaml) · [memory.md](#memorymd) · [shared servers inventory](#shared-servers-inventory) · [accounts.md](#accountsmd) · [artifacts/](#artifacts) · [deploys/](#deploys) · [split-out files](#split-out-files)

## config.yaml

Keys come from the Configuration table in `SKILL.md`, plus free-form keys nested under a preference area. Write a key only when the user states the preference.

**Writing is read-modify-write**: load the existing file, set or replace only the key just declared, keep every other key byte for byte. Never emit a `config.yaml` from this template — the template shows shape, not content. Create `~/Clawic/data/aws/` if it does not exist.

```yaml
default_region: eu-west-1
cli_profile: prod
iac_tool: terraform
monthly_budget_usd: 250
account_model: organization
compliance_regime: none
cost_review_day: 15

# Preference areas — free-form keys added as the user reveals them.
# A preference the user states is a declaration and belongs here, never in memory.md.
conventions:
  tags: [Environment, Project, Owner, CostCenter]
  cidr_scheme: "10.<env>.0.0/16, /20 subnets"
platform:
  instance_families: [m7g, t4g]     # Graviton standard
safety_posture:
  destructive_commands: confirm-each
```

If you find a preference recorded in `memory.md`, move it here and note the move.

## memory.md

Write only the sections you have content for — a heading with nothing under it is noise, and it inflates the line count that decides a split. Never copy these hints into the user's file. `## Boxes` is the one section that is never dropped when `memory.md` is rewritten: deleting a line there orphans a file forever. This is what a populated file looks like:

```markdown
# AWS Memory

## Status
status: ongoing
last: 2026-07-26

## Boxes
- Spend history (18 months) → `spend-log.md`; read before any cost comparison
- Checkout outage runbook → `artifacts/runbook-checkout.md`; read the moment checkout is the symptom
- Accounts (3) → `accounts.md`; read before any billing, profile or cross-account question

## Due
| What | Every | Last run | Next due |
|------|-------|----------|----------|
| Cost review | month, day 15 | 2026-06-15 | 2026-07-15 |
| DR drill | quarter | 2026-04-02 | 2026-07-02 |
| Idle-ALB sweep | week | 2026-07-20 | 2026-07-27 |

## Account Context
111122223333, production SaaS, two engineers, Terraform-managed, eu-west-1.

## Current Infrastructure
One VPC 10.0.0.0/16 · ALB → 2× m7g.large · RDS Postgres db.t4g.medium Multi-AZ · S3 assets bucket behind CloudFront.

## Spend
### Monthly
| Month | Actual | As of | Budget | Top services | Notes |
|-------|--------|-------|--------|--------------|-------|
| 2026-06 | 412 USD | 2026-06-30 | 500 USD | EC2 180 · RDS 95 · NAT 40 | closed |
| 2026-07 | 96 USD | 2026-07-08 | 500 USD | EC2 44 · RDS 24 · NAT 11 | month-to-date |

### Alerts Configured
- Budget 500 USD: alert at 80% actual, 100% forecast
- Anomaly subscription: 17 USD daily threshold

### Optimization Log
| Date | Change | Monthly saving |
|------|--------|----------------|
| 2026-05-11 | S3 gateway endpoint, NAT traffic removed | 14 USD |

## Pain Points
March 2026: 1,400 USD from CloudWatch debug logs. Sensitive to log cost since.

## How They Work
Strong on EC2, new to IAM. Wants the command, not the theory.

---
*Updated: 2026-07-26*
```

Rules that keep this readable next month:

- **`## Boxes`**: one line per file that exists — `<what> (<volume>) → <file>; read when <condition>`. Written in the same turn the file is created. Never delete a line without deleting the file it points to. A box with no index line does not exist.
- **`## Due`**: check it against today's date at the start of a session and state any overdue item in one line — a statement, not a question. Every recurring thing this skill schedules belongs here.
- **`## Spend`**: `As of` is the day the number was read. A row whose `As of` is not the last day of the month is month-to-date and must never be compared against a closed month. Re-checking the current month **overwrites** its row; never a second row for the same month. Amounts always carry their currency. `Top services` is the top three, descending, always the same shape — it is what makes a six-month comparison possible without re-querying Cost Explorer.
- With `account_model: organization`, add a `Scope` column to `### Monthly`: one row for the payer total, plus a row for any account above ~20% of it.
- These three headings are exactly the ones `spend-log.md` gets when `## Spend` outgrows this file, so the split stays a copy-paste.

| Status | Meaning |
|-------|---------|
| `ongoing` | Still learning their setup |
| `complete` | Know their account and workflow well |

## Shared servers inventory

Lives at `~/Clawic/data/servers/servers.md` and is shared with every other infrastructure skill — the user may not have any of them installed, so the format travels with this skill.

```markdown
# Servers

| Name | Provider | Account / Project | Region | Type | Role | Monthly | Access reference |
|------|----------|-------------------|--------|------|------|---------|------------------|
| api-prod-1 | aws | 111122223333 | eu-west-1 | m7g.large | API | 62 USD | ssm:/prod/api/key-name |
```

- **Identity is `Name` + `Provider`.** Read the file before adding. If that pair is already there, update the row in place — it is yours. The rule against rewriting protects rows whose `Provider` is not `aws`; never touch those.
- **Retirement is part of the inventory.** When a host is terminated, delete its row and note the date in `memory.md`. An inventory that only grows stops being an inventory.
- **Amounts carry their currency in the value** (`62 USD`), because Hetzner rows next to yours are in EUR and someone will add the column up.
- **`Monthly` is a planning estimate, not a bill.** Cost Explorer is the source of truth; after a cost review, refresh any AWS row whose real cost moved more than ~20%.
- **Scale cut**: one row per host while there are ≤15. Past that, one file per host at `~/Clawic/data/servers/<name>.md` with the same fields, and `servers.md` becomes the index (`Name | Provider | Role | → file`). If you arrive and the folder already looks like that, follow it — do not start a parallel `servers.md`.
- **Foreign columns win.** If `servers.md` already exists with a different column set, match its columns and add anything missing as a trailing note. Never rewrite its header.
- Access reference is a pointer only. Never a key, token, or password.

## accounts.md

One AWS account lives in `## Account Context`. From the second, this file:

```markdown
# AWS Accounts

| Account ID | Alias | Purpose | Owner / client | Billing | Profile | Region |
|------------|-------|---------|----------------|---------|---------|--------|
| 111122223333 | prod | production SaaS | us | consolidated | prod | eu-west-1 |
| 444455556666 | acme | Acme delivery work | Acme (see contacts) | invoiced separately | acme | eu-west-1 |
```

When an account belongs to a client, the client goes in the shared box `~/Clawic/data/contacts/contacts.md` (`name | role | preferred channel | context`) and is referenced here by name only. Never duplicate the client record inside the AWS box.

## artifacts/

One file per thing, at `~/Clawic/data/aws/artifacts/<kebab-name>.md`, created the first time it exists. Canonical types here: **runbook**, **policy that finally worked**, **architecture decision**. Every artifact opens with when to read it, and gets its `## Boxes` line in the same turn.

```markdown
# Runbook — checkout 502s
*Read when: checkout returns 502/504. Written 2026-07-26.*

...steps, with every secret replaced by its pointer...
```

```markdown
# Architecture decision — API on Fargate, not Lambda
*Read before any change to the request path, and before sizing anything. 2026-07-26.*

Decision: ...one sentence...
Diagram: ...mermaid or ASCII...
Rejected: Lambda — duty cycle ~60%, break-even crossed.
Estimated monthly: 210 USD, eu-west-1.
First quota: 1,000 concurrent tasks. First timeout: ALB idle 60s.
```

If the user tracks this work as a project, the decision summary also belongs in the shared `~/Clawic/data/projects/<project>.md`, with the diagram staying here and referenced by name.

## deploys/

```markdown
# Deploys — 2026

| Date | Service | Image digest / commit | Template version | Rollback target | Notes |
|------|---------|-----------------------|------------------|-----------------|-------|
| 2026-07-24 | api | sha256:9f2c… / a41b7e | tf 1.14.2 | sha256:71ad… | — |

## DR Drills
| Date | What was restored | Measured RTO | What was missing |
|------|-------------------|--------------|------------------|
| 2026-04-02 | RDS snapshot → scratch | 47 min | KMS grant, parameter group |
```

## Split-out files

Created only by the split procedure above, never on day one. Each keeps the exact headings it had inside `memory.md`.

`spend-log.md` — `## Monthly`, `## Alerts Configured`, `## Optimization Log`. The optimization log is the reason this file exists: without it the same NAT gateway gets rediscovered every quarter and nobody can say what the last cleanup was worth.

`resources.md` — the AWS-shaped inventory (`## Databases`, `## Storage`, `## Networking`, `## Known Gaps`), one `## <account-id>` heading per account when there is more than one.
