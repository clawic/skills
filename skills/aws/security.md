# Security — Hardening, Detection, and the Leak Runbook

Identity is the perimeter, and the permissions half of the subject — evaluation order, PassRole, boundaries, SCPs, root rules, key hygiene — is a separate route from SKILL.md's Quick Reference. This file covers exposure, encryption, detection, and what to do when a credential leaks.

**Contents:** [Threat Model in Three Lines](#threat-model-in-three-lines) · [The Exposure Sweep](#the-exposure-sweep) · [Block Public Access Account-Wide](#block-public-access-account-wide) · [Instance Metadata](#instance-metadata) · [Network Exposure](#network-exposure) · [Encryption](#encryption) · [Secrets](#secrets) · [Detection Stack, in Cost Order](#detection-stack-in-cost-order) · [Leaked Credential Runbook](#leaked-credential-runbook) · [Compliance Regimes](#compliance-regimes) · [Audit Checklist](#audit-checklist)

## Threat Model in Three Lines

1. **Credentials leak** — from a repository, a laptop, a log, or an SSRF against the metadata endpoint. This is the overwhelming majority of real AWS incidents.
2. **Something is public that should not be** — a bucket, a snapshot, an AMI, an RDS instance, a security group open to the world.
3. **Cost is the payload** — a stolen credential's first act is usually to launch GPU instances for mining, which is why a cost anomaly alert doubles as a security alert.

Everything below defends one of those three.

## The Exposure Sweep

Run this on any account you inherit, and monthly afterwards.

```bash
# Buckets exposed by a policy
aws s3api get-bucket-policy-status --bucket my-bucket
# Databases reachable from the internet
aws rds describe-db-instances --query 'DBInstances[?PubliclyAccessible].[DBInstanceIdentifier,Endpoint.Address]'
# Security groups open to the world
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]].[GroupId,GroupName]'
# Snapshots and AMIs shared publicly — the forgotten one
aws ec2 describe-snapshots --owner-ids self --restorable-by-user-ids all --query 'Snapshots[].SnapshotId'
aws ec2 describe-images --owners self --query 'Images[?Public].[ImageId,Name]'
# Instances still answering IMDSv1
aws ec2 describe-instances \
  --query 'Reservations[].Instances[?MetadataOptions.HttpTokens!=`required`].[InstanceId]'
```

A public EBS snapshot is a full copy of a disk, including whatever credentials were on it, downloadable by anyone in the region. It is the least-known item on this list and the most complete compromise.

## Block Public Access Account-Wide

The classic failure: the console shows ACL "private" while a bucket policy exposes everything — **policies override ACLs**. Kill the whole class at the account level rather than auditing bucket by bucket:

```bash
aws s3control put-public-access-block --account-id $(aws sts get-caller-identity --query Account --output text) \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

Static sites do not need an exception: CloudFront with Origin Access Control reaches a private bucket and adds TLS, caching, and logs. The three bucket-level follow-ups: set `BucketOwnerEnforced` so objects written by another account are actually yours, share time-limited access with presigned URLs (expiry in minutes, generated server-side), and deny `s3:*` when `aws:SecureTransport` is `false`.

## Instance Metadata

IMDSv1 lets any SSRF bug in an application read the instance's credentials — the vector behind the Capital One breach. Require IMDSv2 everywhere:

```bash
aws ec2 modify-instance-metadata-options --instance-id i-xxx --http-tokens required
# Make it the default for anything launched from now on
aws ec2 modify-instance-metadata-defaults --http-tokens required
```

Launch templates and AMIs carry this setting too — fixing running instances without fixing the launch template means the next scale-out reintroduces it. For containers, give the task its own role and block access to the host's.

## Network Exposure

- No SSH from `0.0.0.0/0`. Use SSM Session Manager: no inbound port, no bastion, no key to lose, and every session recorded in CloudTrail: `aws ssm start-session --target i-xxx`.
- Restrict egress. Default allow-all outbound is the exfiltration path nobody monitors and the reason a compromised container can reach anything it likes.
- Reference security groups instead of CIDRs; segment by trust boundary so a compromised frontend cannot reach the admin service.
- Keep databases in subnets with no route to the internet. A resource that was never routable cannot be misconfigured into exposure by a later edit.
- WAF in front of anything public that accepts user input: the managed rule groups (core, known-bad-inputs, IP reputation) cost little and remove the automated noise. Shield Standard is automatic and free; Shield Advanced is an enterprise DDoS-response contract, not a firewall.

## Encryption

**At rest — a creation-time decision.** RDS and EBS encryption cannot be toggled on later; the retrofit is snapshot → encrypted copy → restore, with downtime.

```bash
aws ec2 enable-ebs-encryption-by-default        # makes the mistake impossible for new volumes
aws rds create-db-instance --db-instance-identifier mydb --storage-encrypted --kms-key-id alias/aws/rds ...
```

AWS-managed keys cost nothing extra. Customer-managed keys ($1/mo each) buy key policies, rotation control, cross-account grants, and the ability to revoke access to data you do not otherwise control — choose them when one of those is a requirement, not by default.

**In transit.** TLS terminates at the ALB or CloudFront; require TLS onward to the database (`rds.force_ssl=1` on Postgres) rather than assuming the VPC is trusted. VPC endpoints keep AWS API traffic off the public internet. Deny non-TLS access to buckets with an `aws:SecureTransport` condition.

## Secrets

Decision rule: **SSM Parameter Store SecureString by default; Secrets Manager ($0.40/secret/mo) when you need automatic rotation or cross-account access.**

```bash
aws ssm put-parameter --name /myapp/db-password --type SecureString --value 'xxx'
aws secretsmanager rotate-secret --secret-id myapp/db-password \
  --rotation-lambda-arn arn:aws:lambda:...:function:SecretsManagerRotation
```

- Applications fetch at startup using their role. A secret injected as a plaintext environment variable by IaC has moved the leak, not fixed it — it now lives in the task definition, the state file, and the console.
- Rotation only helps if the application handles a mid-flight change: fetch again on an auth failure, rather than once at boot forever.
- Never put a secret in: a Lambda environment variable, a container image layer, an AMI, EC2 user data, a CloudFormation parameter without `NoEcho`, or Terraform state that is not access-controlled like production.

## Detection Stack, in Cost Order

| Layer | What it catches | Cost posture |
|---|---|---|
| CloudTrail (management events, multi-region) | Who did what — the forensic record | First trail is free; no reason to skip it |
| Cost anomaly detection | Mining and exfiltration, via the bill | Free |
| GuardDuty | Credential exfiltration, mining, malicious IPs, unusual API patterns | Per volume analyzed; the highest signal per dollar AWS sells |
| AWS Config | Configuration drift and compliance rules | Per configuration item — scope the recorder or it surprises you |
| Security Hub | Aggregated findings against CIS and AWS benchmarks | Per check; earns its cost once there is more than one account |
| Inspector | CVEs in EC2, ECR images, and Lambda | Per resource scanned |
| Macie | Sensitive-data discovery in S3 | Per GB scanned — run it as a periodic job, not continuously |

```bash
aws cloudtrail create-trail --name management-events --s3-bucket-name my-audit-logs --is-multi-region-trail
aws cloudtrail start-logging --name management-events   # creating a trail does NOT start it
aws guardduty create-detector --enable
```

An account with a trail that was never started has the paperwork and none of the evidence. Check `get-trail-status`, not `describe-trails`.

The two GuardDuty findings worth an immediate page on a small account: `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration` (your instance's credentials are being used from somewhere else) and anything under `CryptoCurrency:*`.

## Leaked Credential Runbook

Speed beats elegance. In this order:

1. **Revoke before rotating.** For an IAM user key: `aws iam update-access-key --access-key-id AKIA... --status Inactive`. For a role, deleting it does not invalidate sessions already issued — attach a deny-all policy conditioned on `aws:TokenIssueTime` before that timestamp, which is what the console's "Revoke sessions" button does.
2. **Find the blast radius.** CloudTrail filtered to that key or role session: every call, in every region. Check regions you do not use — that is where mining instances get launched.
3. **Terminate what it created.** Instances, users, roles, access keys, new trust relationships. Attackers build a second door before using the first.
4. **Check the durable footholds**: new IAM principals, changed trust policies, SSH keys added to launch templates, Lambda functions with a scheduled trigger, modified bucket policies.
5. **Preserve evidence before rebuilding.** Snapshot the affected volumes and export the relevant CloudTrail range first — a rebuilt instance is an erased crime scene.
6. **Close the source.** A key committed to a repository is still in git history after the delete commit: the credential must be dead, not hidden. Turn on push protection or an equivalent scanner so the next one never lands.

Rehearse this once in a sandbox. The first run should not be the real one.

## Compliance Regimes

When `compliance_regime` is set, these stop being optional: encryption at rest and in transit everywhere, CloudTrail with log-file validation delivered to a separate account, defined log retention, access reviews with evidence, and service selection limited to eligible services — the eligibility list differs per regime and changes, so check it rather than assuming.

- HIPAA additionally requires a Business Associate Addendum with AWS and restricts which services may touch protected data.
- PCI DSS pushes network segmentation and key management into scope, which usually means a dedicated account for the cardholder data environment.
- Whatever the regime, auditors want *evidence over time*, not a screenshot. Config rules plus Security Hub produce that continuously; a manual checklist produces it once.

## Audit Checklist

Run top to bottom on any account you inherit.

| Check | Command / signal |
|-------|------------------|
| Root: MFA on, zero access keys | `aws iam get-account-summary` → MFA=1, Keys=0 |
| No access key older than 90 days | `aws iam generate-credential-report`, then `get-credential-report` |
| CloudTrail exists, multi-region, and is actually logging | `describe-trails` + `get-trail-status` |
| S3 public access blocked account-wide | `aws s3control get-public-access-block --account-id ...` |
| No publicly accessible RDS | Exposure sweep above |
| No public snapshots or AMIs | Exposure sweep above |
| IMDSv2 required, including in launch templates | Exposure sweep above |
| No 0.0.0.0/0 ingress on SSH or database ports | Exposure sweep above |
| EBS encryption by default enabled | `aws ec2 get-ebs-encryption-by-default` |
| GuardDuty enabled in every region in use | `aws guardduty list-detectors`, per region |
| Budget and cost anomaly alerts exist | `aws budgets describe-budgets` + `aws ce get-anomaly-monitors` |
| External access surfaces reviewed | `aws accessanalyzer list-findings` |

Write the sweep result into `## Current Infrastructure` in `~/Clawic/data/aws/memory.md`, and any host it turned up into `~/Clawic/data/servers/servers.md`. The next session should start from the gaps, not from `describe-*`.
