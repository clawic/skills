# CLI Toolkit

The commands that answer real questions, not the ones in the getting-started guide. Every example assumes a profile is selected; add `--profile <cli_profile>` when one is configured.

**Contents:** [Identity and Profiles](#identity-and-profiles) · [Assuming a Role](#assuming-a-role) · [Querying Output](#querying-output) · [Pagination](#pagination) · [Not Destroying Things](#not-destroying-things) · [Inventory and Discovery](#inventory-and-discovery) · [Logs](#logs) · [Shell Access Without SSH](#shell-access-without-ssh) · [S3 Data Movement](#s3-data-movement) · [Configuration Worth Setting Once](#configuration-worth-setting-once)

## Identity and Profiles

```bash
aws sts get-caller-identity                    # who am I, in which account — run this before anything surprising
aws configure list                             # which config source won: env, profile, or instance role
aws configure sso                              # one-time Identity Center profile setup
aws sso login --profile prod                   # refresh an expired SSO session
export AWS_PROFILE=prod AWS_REGION=eu-west-1   # never rely on an implicit default
```

Credential precedence, highest first: command-line flags → environment variables → the profile named by `AWS_PROFILE` → `~/.aws/credentials` default → container credentials → instance role. `aws configure list` prints which one is actually in effect, and it settles most "wrong account" incidents in one line.

`ExpiredToken` on every call means an SSO session or an assumed-role session ran out — re-login rather than debugging the permission.

## Assuming a Role

```bash
# Declarative: put it in ~/.aws/config and let the CLI handle refresh
# [profile prod-admin]
# role_arn = arn:aws:iam::111122223333:role/Admin
# source_profile = default
# mfa_serial = arn:aws:iam::444455556666:mfa/you

# Imperative, when you need the raw credentials for another tool
aws sts assume-role --role-arn arn:aws:iam::111122223333:role/Admin \
  --role-session-name investigate --duration-seconds 3600
```

Prefer the declarative form: the CLI refreshes before expiry, and the credentials never land in your shell history. Session names appear in CloudTrail — use something that identifies a person or a pipeline, not `session1`.

## Querying Output

`--query` is JMESPath, evaluated server-side of the parsing but client-side of the API. It is the difference between reading JSON and answering a question.

```bash
# Table of instances with the fields you care about
aws ec2 describe-instances --output table \
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name,Name:Tags[?Key==`Name`]|[0].Value}'

# Filter server-side first (cheaper and faster), then shape
aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].InstanceId' --output text

# Sort and slice
aws ec2 describe-snapshots --owner-ids self \
  --query 'reverse(sort_by(Snapshots,&StartTime))[:10].[SnapshotId,StartTime,VolumeSize]' --output table
```

- `--filters` runs at the API and reduces what is returned; `--query` runs locally on the response. Use filters for volume, query for shape.
- `--output text` for shell pipelines, `table` for humans, `json` for `jq`. Text output separates columns with tabs, which is why `read` works and `awk` with default splitting sometimes does not.
- Backticks quote literals inside JMESPath; single quotes wrap the whole expression for the shell. The Name-tag idiom above is the one worth memorizing.

## Pagination

The CLI auto-paginates by default, which is usually right and occasionally the reason a command hangs on a huge account.

```bash
aws s3api list-objects-v2 --bucket big --max-items 100          # stop after N items, prints NextToken
aws s3api list-objects-v2 --bucket big --page-size 1000         # request size per call; does not limit total
aws logs describe-log-groups --no-paginate                      # one page only
```

`--max-items` limits results; `--page-size` limits calls. Using `--page-size` to "get fewer results" is the common mistake, and on a throttled API a smaller page size means *more* calls and more throttling.

## Not Destroying Things

```bash
aws ec2 terminate-instances --instance-ids i-xxx --dry-run   # permission and validity check, no action
aws s3 sync ./local s3://bucket --dryrun                     # prints what would transfer
aws cloudformation deploy --no-execute-changeset             # creates a change set for review
terraform plan -out=plan.out                                 # review, then apply exactly that file
```

`--dry-run` on EC2 APIs returns `DryRunOperation` when you *do* have permission and `UnauthorizedOperation` when you do not — it is also the cheapest permission test that exists. Not all services support it; `--cli-auto-prompt` and change sets cover the rest.

Before any delete, terminate, or force: state what it destroys and what depends on it, and get an explicit confirmation. Destructive commands never travel inside a block of read-only ones.

## Inventory and Discovery

```bash
# Everything tagged, across services, in one call
aws resourcegroupstaggingapi get-resources --tag-filters Key=Environment,Values=prod

# What exists that is NOT tagged — usually the interesting set
aws resourcegroupstaggingapi get-resources --query 'ResourceTagMappingList[?length(Tags)==`0`].ResourceARN'

# Cost by service, last 30 days — the fastest map of an unknown account
aws ce get-cost-and-usage --time-period Start=$(date -u -d '30 days ago' +%Y-%m-%d 2>/dev/null || date -v-30d +%Y-%m-%d),End=$(date -u +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE

# The account's own view of what it is running
aws configservice select-resource-config \
  --expression "SELECT resourceType, COUNT(*) GROUP BY resourceType"
```

`date -v-30d` is BSD/macOS; `date -d '30 days ago'` is GNU/Linux. The fallback above works on both — hardcoding either breaks half your readers' machines.

## Logs

```bash
aws logs tail /aws/lambda/myfn --follow --since 15m           # live tail, the command most people never learn
aws logs tail /ecs/myservice --filter-pattern "ERROR" --since 1h
aws logs start-query --log-group-names /aws/lambda/myfn \
  --start-time $(date -u +%s -d '1 hour ago' 2>/dev/null || date -v-1H +%s) --end-time $(date -u +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /Timeout/ | sort @timestamp desc | limit 50'
```

`aws logs tail --follow` replaces the console for almost all interactive debugging. Insights queries are billed per GB scanned, so narrow the time range before the filter, not after.

## Shell Access Without SSH

```bash
aws ssm start-session --target i-xxx                          # no inbound port, no bastion, logged to CloudTrail
aws ssm start-session --target i-xxx \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.xxx.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'
aws ecs execute-command --cluster c --task <arn> --container app --interactive --command "/bin/sh"
```

Port forwarding through Session Manager reaches a private RDS instance from a laptop without a bastion, a public subnet, or an SSH key. It requires the SSM agent, an instance role with `AmazonSSMManagedInstanceCore`, and network egress to the SSM endpoints (or interface endpoints in a private subnet).

`ecs execute-command` needs `enableExecuteCommand` on the service and `ssmmessages:*` on the **task** role — the shell that opens is inside the container, so the task role is the one that matters — the execution role only pulls the image and reads secrets before start.

## S3 Data Movement

```bash
aws s3 sync ./dist s3://bucket/prefix --delete --exact-timestamps
aws s3 cp s3://bucket/big.tar.gz - | tar xz -C /target        # stream, never land the file
aws s3api list-object-versions --bucket b --prefix p          # what versioning is actually keeping
aws s3 presign s3://bucket/key --expires-in 300               # 5-minute share, no public bucket
aws configure set default.s3.max_concurrent_requests 20       # tune parallelism for large transfers
```

`s3 sync` compares size and modification time, not content. `--exact-timestamps` matters on downloads where a same-size file was regenerated; for correctness-critical syncs, compare checksums explicitly.

## Configuration Worth Setting Once

```bash
aws configure set cli_pager ""                    # stop paging every output into less
aws configure set output json
aws configure set retry_mode adaptive             # client-side rate limiting on throttled APIs
aws configure set max_attempts 10
```

`retry_mode adaptive` is the difference between a script that dies on `ThrottlingException` and one that slows down and finishes.
