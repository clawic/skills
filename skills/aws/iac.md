# Infrastructure as Code on AWS

Everything below applies to the tool named by `iac_tool`. HCL language mechanics, module design, and state surgery in depth: the `terraform` skill. This file is the AWS-specific half — what the provider does to you regardless of the language.

## Split Stacks by Lifecycle, Not by Team

One stack per lifecycle: **network** (VPC, subnets, endpoints — changes yearly), **data** (RDS, S3, DynamoDB — changes rarely and destructively), **application** (compute, load balancers, DNS records — changes daily).

- CloudFormation caps a stack at 500 resources, but that is not the reason. The reason is that a failing application change should not block a network change, and nobody should be able to `destroy` a database while shipping a feature.
- Cross-stack references: SSM Parameter Store parameters or remote-state data sources beat CloudFormation exports, because an export cannot be changed while another stack imports it — a dependency that turns a rename into a two-day migration.
- Anything with data gets `prevent_destroy` / `DeletionPolicy: Retain` and deletion protection on the resource itself. Two independent locks, because one of them will be bypassed.

## Drift Is the Precondition, Not a Report

Run the plan and get a clean result **before** making any change. A dirty plan means someone hotfixed in the console, and applying your change also reverts their fix — usually during the incident their fix was addressing.

```bash
terraform plan -detailed-exitcode        # exit 0 = no changes, 2 = changes pending, 1 = error
aws cloudformation detect-stack-drift --stack-name mystack
aws cloudformation describe-stack-resource-drifts --stack-name mystack \
  --stack-resource-drift-status-filters MODIFIED DELETED
```

`-detailed-exitcode` is what makes drift a CI gate instead of something a human is supposed to notice. CloudFormation drift detection is asynchronous and does not cover every resource type — treat "no drift detected" as weaker evidence than a clean plan.

## CloudFormation Failure States

| State | What happened | Way out |
|---|---|---|
| `ROLLBACK_COMPLETE` | Create failed and rolled back | The stack cannot be updated — delete it and create again |
| `UPDATE_ROLLBACK_FAILED` | The rollback itself failed, usually because a resource was deleted by hand | `continue-update-rollback --resources-to-skip <LogicalIds>`, then fix the template |
| `DELETE_FAILED` | A resource refuses to delete (non-empty S3 bucket, ENI in use, retained snapshot) | Delete the blocker manually, or `delete-stack --retain-resources` |
| `UPDATE_IN_PROGRESS` for hours | Waiting on a resource that will never signal (ASG creation policy, custom resource with no response) | Cancel the update, or send the custom resource's response manually |
| `IMPORT_ROLLBACK_*` | An import failed the resource-identifier check | Fix the identifier and re-import; imports never modify the resource |

Change sets are the CloudFormation equivalent of a plan. An update executed without one is a deploy with no review step.

## Adopting Existing Resources

Console-created resources are the normal starting state; importing them is routine, not exotic.

- Terraform 1.5+ takes `import` blocks in configuration, so an import is reviewed in a pull request and applied by CI rather than typed into someone's terminal.
- `terraform plan -generate-config-out=generated.tf` writes a first draft of the configuration for imported resources — expect to trim it heavily; the provider emits every attribute, including read-only ones.
- CloudFormation imports need a template where the resource carries `DeletionPolicy: Retain` and an exact resource identifier.
- Import in dependency order: network before compute. An imported instance whose subnet is unmanaged produces a plan that wants to replace it.
- After every import, plan must be clean. A non-empty plan right after an import means your configuration does not match reality, and applying it will change production.

## Provider Traps That Cost Real Time

- **Everything is eventually consistent.** IAM roles propagate after creation, so a role used immediately by another resource fails with `AccessDenied` on the first apply and succeeds on the retry. Depend on the attachment, not just the role.
- **`for_each` over a computed value** fails with "value depends on resource attributes that cannot be determined until apply". Key the map on something static — a name from a variable, not an ARN from another resource.
- **`count` on a list** renumbers everything after a removed element: deleting the second of five subnets destroys and recreates three. Use `for_each` with stable keys for anything you will ever remove from the middle.
- **Tags that fight**: services that add their own tags (ASG propagation, EKS, ECS) make every plan dirty. Use `ignore_changes` on those tag keys or a provider-level `default_tags` policy, not a manual reconciliation each time.
- **Secrets in state**: an RDS password, a generated key, or a secret's value is stored in plaintext in state. Encrypt the state bucket, restrict access to it as tightly as to production, and prefer `manage_master_user_password` (Secrets Manager-managed) so the value never enters state at all.
- **Resource replacement you did not ask for**: changing an EBS volume's availability zone, an RDS instance's subnet group, or a Lambda function name replaces the resource. Read the plan's `# forces replacement` lines every single time — that phrase is the only warning you get before a database is deleted.

## Remote State on AWS

- S3 backend with versioning and default encryption on the bucket. Versioning is what lets you recover from a corrupted or truncated state file, and it is the only recovery that exists.
- Locking: Terraform 1.10+ supports S3-native locking (`use_lockfile = true`), replacing the separate DynamoDB lock table. On older versions, keep the DynamoDB table — running without any lock means two concurrent applies silently corrupt state.
- One state file per stack per environment. A single state file for everything means every apply risks everything, and `terraform apply -target` becomes routine, which is the road to permanent drift.
- Never edit state by hand. `state mv`, `state rm`, and `import` cover the legitimate operations; a text editor covers none of them.

## CI/CD for Infrastructure

- Plan on pull request, apply on merge, with the plan output posted for review. A human approving an apply they have not read is a rubber stamp with extra steps.
- The pipeline assumes a role via OIDC — GitHub Actions and GitLab federate directly to IAM with no stored credentials. Long-lived deploy keys in CI are the most common way an AWS account is compromised through a repository.
- Scope the deploy role per environment and constrain the trust policy to the specific repository and branch (`token.actions.githubusercontent.com:sub`). A trust policy allowing the whole GitHub organisation lets any repository deploy to production.
- Policy checks before apply: `checkov`, `tfsec`, or CloudFormation Guard catch public buckets and open security groups at review time, where fixing them is free.
- Keep the plan artifact and apply exactly it (`terraform apply plan.out`). Re-planning at apply time means you deployed something nobody reviewed.

## When Not to Use IaC

Genuine exceptions, kept short so they stay exceptions: one-off exploration in a sandbox account, break-glass incident response (recorded and reconciled afterwards), and resources a managed service creates on your behalf. Everything else — including "just this once" — goes into code. The console hotfix survives until the next apply, and its author is never the person debugging its disappearance.
