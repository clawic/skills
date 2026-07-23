# AWS Security — Hardening

## IAM

### Root Account: Three Rules
MFA on, zero access keys, never used for daily work. Verify:
```bash
aws iam get-account-summary --query 'SummaryMap.{RootMFA:AccountMFAEnabled,RootKeys:AccountAccessKeysPresent}'
```
Both should read `1` and `0`. Root with an access key is the single worst finding in an audit — it bypasses every policy you write.

### Roles for Services, Never Embedded Keys
An EC2/Lambda/ECS workload gets a role; credentials in code or env files eventually land in a git history. If you find IAM user keys wired into an app, the fix is a role plus key deactivation — not key rotation.
```bash
aws iam create-role --role-name MyAppRole \
  --assume-role-policy-document file://trust-policy.json
```

### Key Hygiene
```bash
# Credential report: every user, key age, last used, MFA — one call
aws iam generate-credential-report && aws iam get-credential-report \
  --query Content --output text | base64 -d
```
Rotate any access key >90 days old; delete any key with no `last_used` in 90 days — unused keys are pure liability. IAM Access Analyzer (`aws accessanalyzer create-analyzer`) finds resources shared outside the account; run it once, then on every policy change.

### Least Privilege, Practically
Start from what the app actually calls, scope to ARNs:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```
`Action: "s3:*"` on `Resource: "*"` is not a starting point to tighten later — later never comes. When you genuinely don't know the needed actions, run with broad permissions in a sandbox and read CloudTrail to write the real policy from observed calls.

## Instance Metadata: Require IMDSv2

IMDSv1 lets any SSRF bug in your app read instance credentials — the Capital One breach vector. Enforce on every instance:
```bash
aws ec2 modify-instance-metadata-options --instance-id i-xxx --http-tokens required
# Audit the fleet
aws ec2 describe-instances \
  --query 'Reservations[].Instances[?MetadataOptions.HttpTokens!=`required`].[InstanceId]'
```

## Network Security

### VPC Layout
```
┌─────────────────────────────────────────┐
│ VPC (10.0.0.0/16)                       │
│  ┌──────────────┐  ┌──────────────┐     │
│  │ Public       │  │ Private      │     │
│  │ 10.0.1.0/24  │  │ 10.0.2.0/24  │     │
│  │ ALB, NAT     │  │ EC2, RDS     │     │
│  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────┘
```
Public subnet: load balancers and NAT only. Everything with data lives private. A database that was never publicly routable can't be misconfigured into exposure.

### Security Groups
```bash
# Allow HTTPS from the ALB's SG only — never from a CIDR you'll forget to update
aws ec2 authorize-security-group-ingress --group-id sg-xxx \
  --protocol tcp --port 443 --source-group sg-alb
```
- No SSH from 0.0.0.0/0 — use SSM Session Manager: no inbound port, no bastion, sessions logged to CloudTrail (`aws ssm start-session --target i-xxx`).
- Restrict outbound too: default allow-all egress is the exfiltration path nobody monitors.
- Reference SGs, not IP ranges — SG references survive IP churn.

### NACLs: Mostly Leave Them Alone
Contrarian but standard among practitioners: keep NACLs at default allow-all and do all control in security groups. NACLs are stateless — every rule needs a matching ephemeral-port return rule (1024-65535), and a missing one produces intermittent, maddening failures. Use NACLs only for explicit subnet-level denies (blocking a known-bad CIDR).

## Data Protection

### Encryption at Rest — a Creation-Time Decision
RDS and EBS encryption cannot be toggled on later; the retrofit is snapshot → encrypted copy → restore, with downtime. So encrypt at creation, always:
```bash
aws rds create-db-instance --db-instance-identifier mydb \
  --storage-encrypted --kms-key-id alias/aws/rds ...
# Account-wide EBS default — makes the mistake impossible
aws ec2 enable-ebs-encryption-by-default
```
KMS default keys cost nothing extra; customer-managed keys ($1/mo each) only when you need key policies, rotation control, or cross-account grants.

### In Transit
TLS terminates at the ALB; require TLS to RDS (`rds.force_ssl=1` on Postgres); VPC endpoints keep AWS API traffic off the public internet.

## S3 Exposure

The classic failure: console shows ACL "private" while a bucket policy exposes everything — policies override ACLs. Kill the whole class account-wide:
```bash
aws s3control put-public-access-block --account-id $(aws sts get-caller-identity --query Account --output text) \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```
Per-bucket audit when something must be public (static sites — though CloudFront + Origin Access Control is the better pattern, no public bucket at all):
```bash
aws s3api get-bucket-policy-status --bucket my-bucket
aws s3api get-public-access-block --bucket my-bucket
```
Time-limited sharing → presigned URLs, never a public object.

## Secrets

Decision rule: SSM Parameter Store (SecureString, free tier) by default; Secrets Manager ($0.40/secret/mo) only when you need automatic rotation or cross-account access.
```bash
aws secretsmanager create-secret --name myapp/db-password --secret-string '{"password":"xxx"}'
aws secretsmanager rotate-secret --secret-id myapp/db-password \
  --rotation-lambda-arn arn:aws:lambda:...:function:SecretsManagerRotation
```
Apps fetch secrets at startup via their role — a secret injected as a plaintext env var in IaC has just moved the leak, not fixed it.

## Detection

```bash
# CloudTrail — the first trail's management events are free; no reason to skip
aws cloudtrail create-trail --name management-events \
  --s3-bucket-name my-audit-logs --is-multi-region-trail
aws guardduty create-detector --enable    # managed threat detection, pay-per-volume
```
GuardDuty's highest-value findings for small accounts: credential exfiltration (`UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration`) and crypto-mining on compromised instances.

## Audit Checklist

Run top to bottom on any account you inherit:

| Check | Command |
|-------|---------|
| Root: MFA on, no keys | `aws iam get-account-summary` (MFA=1, Keys=0) |
| Key ages + usage | `aws iam generate-credential-report` → rotate >90 days |
| CloudTrail on, multi-region | `aws cloudtrail describe-trails` |
| S3 public access blocked account-wide | `aws s3control get-public-access-block --account-id ...` |
| No public RDS | `aws rds describe-db-instances --query 'DBInstances[?PubliclyAccessible].[DBInstanceIdentifier]'` |
| IMDSv2 enforced | describe-instances filter above |
| No 0.0.0.0/0 ingress | `aws ec2 describe-security-groups --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==\`0.0.0.0/0\`]]]'` |
| EBS encryption default | `aws ec2 get-ebs-encryption-by-default` |
