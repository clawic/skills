# Accounts, Organizations, and Access

Applies when `account_model` is `organization`, and describes what a `single`-account setup is choosing not to have. The decision matters early: **moving live resources between accounts is a migration project, not a setting.**

## When One Account Stops Being Enough

Any one of these is sufficient reason to open an Organization:

- More than one person can touch production, and you want prod and dev separated by an account boundary rather than by a tag and good intentions.
- A compliance regime (`compliance_regime` other than `none`) that requires demonstrable isolation of scope.
- A blast radius you cannot accept: an account is the only hard boundary AWS offers. IAM mistakes stop at it; service quotas are per-account, so a runaway dev workload cannot exhaust production's Lambda concurrency.
- Cost attribution that tags cannot deliver, because someone will always forget a tag and nobody forgets which account they deployed to.

Counterweight: every account needs its own baseline (CloudTrail, GuardDuty, budgets, IAM roles, VPC). Without automation that is real recurring work — which is the argument for Control Tower rather than for staying single-account.

## The Layout That Ages Well

```
Root
├── Management account         ← billing and Organizations ONLY. No workloads.
├── Security OU
│   ├── Log archive            ← immutable CloudTrail/Config destination
│   └── Audit                  ← read-only cross-account roles for security tooling
├── Infrastructure OU
│   └── Shared services        ← shared VPC/Transit Gateway, ECR, CI runners
└── Workloads OU
    ├── prod
    ├── staging
    └── sandbox
```

The management account stays empty for one specific reason: **SCPs do not apply to it**. A workload there sits outside every guardrail you write, which is exactly the account where a mistake is worst.

## Service Control Policies

SCPs restrict; they never grant. The maintainable shape is `FullAWSAccess` plus explicit denies.

High-value denies that cost nothing and break little:

| Deny | Prevents |
|---|---|
| Regions outside your approved list (with a `NotAction` exemption for global services: IAM, Organizations, CloudFront, Route 53, Support) | Resources appearing where nobody is watching them |
| `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail`, `guardduty:Delete*`, `config:Delete*` | Turning off the evidence |
| `s3:DeleteBucket` and `s3:PutBucketPolicy` on the log-archive bucket | Destroying the audit trail |
| `iam:CreateUser`, `iam:CreateAccessKey` | Long-lived credentials appearing after you moved everyone to SSO |
| `organizations:LeaveOrganization` | An account walking out of your guardrails |
| Root-user actions except a short allow-list | Daily work as root |

Test against one OU before applying at the root. An SCP mistake at the root is an outage with a slow rollback, and the resulting `AccessDenied` never mentions the SCP.

## Identity Center (SSO) Instead of IAM Users

- One identity source (Identity Center's own directory, or an external IdP), permission sets mapped to groups, groups assigned to accounts. Adding an engineer becomes a group membership, and removing one actually removes their access everywhere at once.
- Permission sets are templates that materialize as roles in each assigned account — write them once, and the same `PowerUserAccess` boundary exists in twelve accounts consistently.
- Session duration is per permission set. Long sessions are convenient and are also how a stolen laptop stays useful; 8 hours for daily work, 1 hour for anything with production write access.
- CLI access: `aws configure sso` writes a profile; `aws sso login --profile prod` refreshes it. Credentials never touch `~/.aws/credentials`.
- Keep one break-glass IAM user per organisation, MFA-protected, credentials in a physical safe, with a CloudWatch alarm on any use — for the day the IdP is the outage.

## Cross-Account Access

Two mechanisms, chosen by direction:

- **Role assumption** — the target account's role trusts the source account's principal; the source's identity policy allows `sts:AssumeRole`. Both sides required, always. Add `sts:ExternalId` when the trusting party is a third party; it is the defence against the confused-deputy problem, and it is why every SaaS vendor asks for one.
- **Resource policies** — S3 bucket policies, KMS key policies, ECR repository policies grant access without assuming anything. Condition on `aws:PrincipalOrgID` to allow the whole Organization without maintaining an account list.

Role chaining (assume A, then assume B from A) caps the session at 1 hour regardless of the role's maximum duration, and cannot be extended. A pipeline that assumes twice and runs a 90-minute job fails at minute 60 — flatten the chain instead of fighting it.

## Billing Across Accounts

- Consolidated billing pools usage, so free-tier and volume-tier discounts apply across the whole Organization. RIs and Savings Plans purchased in one account share with others by default; turn sharing off per account when a team must see its own true cost.
- Cost allocation tags must be *activated* in the management account, and they report only from activation forward (SKILL.md Rule 6). Activate the standard set on day one, before there is history to lose.
- Per-account budgets plus one Organization-wide budget. The account boundary is what makes an alert actionable: "the Organization is over budget" tells nobody what to do.
- Cost Categories map accounts and tags into the dimensions your business actually reports on (team, product, environment), which is the layer Cost Explorer alone cannot produce.

## Landing Zone Automation

- **Control Tower** sets up the Organization, the log archive and audit accounts, guardrails, and Account Factory for new accounts. Correct for most teams that reach multi-account: it encodes the layout above without a project to design it.
- **Account Factory for Terraform / custom pipelines** where you already have IaC discipline and want account creation in the same review flow as everything else.
- Whatever creates accounts must also apply the baseline: CloudTrail to the log archive, Config recorder, GuardDuty, default EBS encryption, account-level S3 Block Public Access, a budget, and the standard roles. An account created without its baseline is the one that shows up in the next audit.
- Closing an account is a 90-day process with a suspension period, and the account still counts against the Organization's quota during it. Reuse a sandbox rather than churning accounts.

Record the account → alias → owner/client → billing mapping (`memory-template.md`): with one account it lives in `## Account Context`, from the second it is `~/Clawic/data/aws/accounts.md`. Per-account budgets are unactionable if nobody can say whose account it is. If an account belongs to a client, the client goes in the shared `~/Clawic/data/contacts/contacts.md` and is referenced here by name only.
