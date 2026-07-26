# IAM — Permissions That Actually Evaluate

IAM is a deny-by-default evaluation engine with five independent gates. Most "IAM bugs" are a correct policy being overruled by a gate the author forgot exists.

**Contents:** [Evaluation Order (memorize this, it ends most arguments)](#evaluation-order-memorize-this-it-ends-most-arguments) · [Diagnosing a Denial](#diagnosing-a-denial) · [Trust Policy vs Permission Policy](#trust-policy-vs-permission-policy) · [iam:PassRole — The Privilege Escalation Nobody Reviews](#iampassrole--the-privilege-escalation-nobody-reviews) · [Writing a Least-Privilege Policy That Works](#writing-a-least-privilege-policy-that-works) · [Condition Keys Worth Knowing](#condition-keys-worth-knowing) · [Roles vs Users](#roles-vs-users) · [Key Hygiene](#key-hygiene) · [Root Account: Three Rules](#root-account-three-rules) · [Organization-Level Controls](#organization-level-controls) · [Access Review Checklist](#access-review-checklist)

## Evaluation Order (memorize this, it ends most arguments)

A request is allowed only if it survives every applicable gate:

1. **Explicit `Deny` anywhere** — in any identity policy, resource policy, SCP, boundary, or session policy. One deny beats every allow, always. There is no override.
2. **Service Control Policy** (if the account is in an Organization) — an SCP grants nothing; it only defines the ceiling. A permission absent from the SCP is denied no matter what the account's admin writes.
3. **Resource control policy / resource policy** — for cross-account access, the resource side must allow you. Same-account access can be granted by *either* the identity policy or the resource policy (except for KMS, which always requires the key policy to allow).
4. **Permissions boundary** — if attached to the principal, the effective permission is the *intersection* of boundary and identity policy. A boundary that omits an action denies it silently.
5. **Identity policy / session policy** — the ordinary allow. A session policy passed at `AssumeRole` further narrows what the session can do, and cannot expand it.

Worked example: a role with `AdministratorAccess` still cannot create an S3 bucket if the SCP restricts the account to two regions and the call targets a third. The error says `AccessDenied` with no mention of the SCP — the failing gate is never named in the message.

## Diagnosing a Denial

```bash
# What does the policy simulator say the effective permission is?
aws iam simulate-principal-policy --policy-source-arn arn:aws:iam::111122223333:role/AppRole \
  --action-names s3:GetObject --resource-arns "arn:aws:s3:::my-bucket/key"
# Cross-account: check the other side too
aws s3api get-bucket-policy --bucket my-bucket
aws kms get-key-policy --key-id alias/mykey --policy-key-name default
```

The simulator does NOT evaluate SCPs or resource policies from another account. Simulator says allow + reality says deny = look at gates 2 and 3.

## Trust Policy vs Permission Policy

Two policies, two different failures, one confusing error.

| Symptom | Failing document |
|---|---|
| `User ... is not authorized to perform: sts:AssumeRole on resource: <role ARN>` | The **trust policy** of the target role does not list the caller as a principal |
| Assume succeeds, then the next call is denied | The **permission policy** attached to the role |
| Assume works for a human, fails for a service | The trust policy names an IAM principal but the caller is a service principal (`lambda.amazonaws.com`, `ecs-tasks.amazonaws.com`) |

Add `sts:TagSession` to the trust policy when the caller passes session tags; without it the assume fails with a message about tags that reads like a permissions error.

## iam:PassRole — The Privilege Escalation Nobody Reviews

Creating a resource that runs as a role is not the same permission as using the role. `iam:PassRole` is the one that matters: a principal with `ec2:RunInstances` plus `iam:PassRole` on `*` can launch an instance as the admin role and own the account.

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::111122223333:role/AppTaskRole",
  "Condition": {"StringEquals": {"iam:PassedToService": "ecs-tasks.amazonaws.com"}}
}
```

Always scope `PassRole` to specific role ARNs and constrain `iam:PassedToService`. Treat `PassRole` on `*` in a review exactly like `AdministratorAccess`, because it is one.

## Writing a Least-Privilege Policy That Works

The order that succeeds:

1. Run the workload in a sandbox with broad permissions for a representative period (a full business cycle, not ten minutes).
2. Generate the policy from what it actually called: `aws accessanalyzer start-policy-generation` reads CloudTrail and emits a policy of observed actions.
3. Scope resources to ARNs. Wildcards survive only where the API genuinely has no resource-level permission (`ec2:DescribeInstances` is one — the API does not support resource ARNs, so `Resource: "*"` there is correct, not lazy).
4. Add conditions that are free to satisfy and expensive to abuse: `aws:PrincipalOrgID` on resource policies kills the confused-deputy class; `aws:SourceArn` and `aws:SourceAccount` on service trust policies do the same for service-to-service calls.
5. Attach a permissions boundary to any role that developers can create or modify — it is the only mechanism that survives a developer with `iam:CreateRole`.

`"Action": "s3:*"` on `"Resource": "*"` is not a starting point to tighten later. Later never comes, and by then the blast radius is production.

## Condition Keys Worth Knowing

| Key | Use |
|---|---|
| `aws:PrincipalOrgID` | Resource policies: allow any principal in your Organization without listing accounts |
| `aws:SourceArn` / `aws:SourceAccount` | Service trust policies: prevent another customer's resource from triggering yours |
| `aws:SecureTransport` | Deny non-TLS access to a bucket in one statement |
| `aws:MultiFactorAuthPresent` | Human-only actions; note it is *false* for machine roles, so never apply it to a workload role |
| `aws:RequestTag` / `aws:ResourceTag` | Attribute-based access control — one policy that scales instead of one policy per team |
| `aws:ViaAWSService` | Distinguish a direct call from one a service made on your behalf |

Trap: `aws:MultiFactorAuthPresent` is absent, not false, for calls made with long-term user credentials — use `BoolIfExists` or the condition never matches and the deny never fires.

## Roles vs Users

- Workloads get roles: EC2 instance profiles, Lambda execution roles, ECS task roles, IRSA/Pod Identity for EKS. Credentials rotate automatically and never sit on disk.
- Humans get federated sessions through Identity Center or an external IdP. An IAM user with a password is a legacy pattern with a rotation problem.
- If you find IAM user keys wired into an application, the fix is a role plus key deactivation — rotating the key just resets the countdown on the same failure.
- Break-glass exception: one IAM user with MFA and no console access, credentials in a physical safe, for the day your IdP is the outage. Alarm on any use of it.

## Key Hygiene

```bash
aws iam generate-credential-report && aws iam get-credential-report \
  --query Content --output text | base64 -d
```

One call, every user: key age, last used, MFA status, password age. Rotate any access key older than 90 days; delete any key with no recorded use in 90 days — an unused key is pure liability with no offsetting benefit.

## Root Account: Three Rules

MFA on, zero access keys, never used for daily work.

```bash
aws iam get-account-summary --query 'SummaryMap.{RootMFA:AccountMFAEnabled,RootKeys:AccountAccessKeysPresent}'
```

Expect `1` and `0`. A root access key bypasses every policy and every SCP you will ever write — it is the single worst finding in an audit. The handful of tasks that genuinely require root (closing the account, changing the support plan, some S3 bucket-policy recoveries) are rare enough to justify a documented, alarmed procedure.

## Organization-Level Controls

- SCPs restrict; they never grant. An SCP of `FullAWSAccess` plus a deny for what you forbid is the maintainable shape.
- SCPs do not apply to the management account. A workload in the management account is outside your guardrails — this is the strongest practical argument for keeping it empty.
- Useful denies that cost nothing: region restriction, disabling CloudTrail/GuardDuty, deleting the audit-log bucket, creating IAM users, leaving the Organization.
- Test an SCP against one OU before the root. An SCP mistake at the root is an outage with a slow rollback.

## Access Review Checklist

| Check | Command / signal |
|---|---|
| Root MFA on, no root keys | `aws iam get-account-summary` |
| No key older than 90 days | Credential report above |
| No `iam:PassRole` on `*` | Grep policies for `PassRole` with wildcard resources |
| No inline admin policies on users | `aws iam list-user-policies` per user |
| External access surfaces known | `aws accessanalyzer list-findings` — resources shared outside the account or Organization |
| Unused roles and permissions | IAM Access Advisor: services not accessed in 90+ days are candidates for removal |
| Boundaries on developer-creatable roles | `aws iam get-role --query 'Role.PermissionsBoundary'` |

After a least-privilege policy finally works, save it to `~/Clawic/data/aws/artifacts/policy-<role>.md` — the JSON with every secret value replaced by its pointer, the date, and what it unblocked — and add its `## Boxes` line to `memory.md`. Deriving one costs a full business cycle; nobody should pay it twice.
