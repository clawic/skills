# Storage — S3, EBS, EFS

Decision rule first: **S3 unless you need a filesystem; EBS for one instance's disk; EFS only when several instances must mount the same tree.** EFS at $0.30/GB-mo is nearly 4× gp3's $0.08 — "we might share it later" does not justify it.

**Contents:** [S3: The Bill Is Requests, Not Bytes](#s3-the-bill-is-requests-not-bytes) · [S3 Lifecycle — Including the Costs Inside the Savings](#s3-lifecycle--including-the-costs-inside-the-savings) · [S3 Versioning: The Silent Doubling](#s3-versioning-the-silent-doubling) · [S3 Access Precedence](#s3-access-precedence) · [S3 Throughput and Consistency](#s3-throughput-and-consistency) · [Encryption](#encryption) · [EBS](#ebs) · [EFS](#efs) · [Picking Between Them, Concretely](#picking-between-them-concretely)

## S3: The Bill Is Requests, Not Bytes

Storage is cheap ($0.023/GB-mo Standard). Requests and transfers are where buckets surprise people.

- PUT/COPY/POST/LIST ≈ $0.005 per 1,000; GET/SELECT ≈ $0.0004 per 1,000. Writing one object per event at 10 M events/month costs ~$50 in PUTs; if those objects are 1 KB each, the 10 GB they occupy costs ~$0.23. Requests are 200× the storage. Batch into larger objects before writing.
- `LIST` is billed at the PUT rate and returns 1,000 keys per call. A job that lists a million-object prefix on every run is a real line item; keep an index instead.
- Egress to the internet is the other half — front public reads with CloudFront (first 1 TB/mo free) rather than serving from the bucket.
- Cross-region replication bills the transfer plus a second copy of the storage. It is a disaster-recovery decision, never a latency one; use CloudFront for latency.

## S3 Lifecycle — Including the Costs Inside the Savings

```bash
aws s3api put-bucket-lifecycle-configuration --bucket my-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "archive-and-clean",
      "Status": "Enabled",
      "Filter": {"Prefix": "logs/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"}
      ],
      "Expiration": {"Days": 365},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }]
  }'
```

Four traps inside the savings:

- **Transitions are billed per request** (order of $0.01-0.05 per 1,000 depending on the target tier). Millions of tiny objects can cost more to move than a year of the storage saved. Aggregate into larger objects first.
- **Minimums**: IA charges a 30-day minimum and treats objects under 128 KB as 128 KB; Glacier Flexible has a 90-day minimum, Deep Archive 180 days. Early deletion bills the remainder.
- **Retrieval fees** apply to IA and Glacier. IA pays off only for data read less than roughly once a month.
- **Incomplete multipart uploads are billed and invisible** in a normal object listing. Without the `AbortIncompleteMultipartUpload` rule above, a failed upload job leaves paid-for fragments forever. Find them with `aws s3api list-multipart-uploads --bucket my-bucket`.

Intelligent-Tiering is the right default when access patterns are unknown and objects exceed 128 KB: no retrieval fees, a monitoring fee of ~$0.0025 per 1,000 objects per month. Skip it for buckets of millions of tiny objects, where monitoring exceeds the storage.

## S3 Versioning: The Silent Doubling

- Versioning never deletes anything. `DELETE` writes a delete marker; the object still bills. A bucket with versioning on and no lifecycle rule grows forever.
- The rule you need alongside it: `NoncurrentVersionExpiration` (plus `NewerNoncurrentVersions` to keep a few), or the archive strategy applies only to current versions.
- MFA Delete protects against a compromised credential deleting versions, can only be enabled by the root account, and can only be managed with the CLI. Worth it for the audit-log bucket, disruptive elsewhere.

## S3 Access Precedence

Precedence, highest first: an explicit deny anywhere → account-level Block Public Access → bucket policy → ACL. **Policies override ACLs**, which is why a bucket whose ACL reads "private" can still be world-readable. Turn the whole class off at the account level rather than auditing bucket by bucket: `aws s3control put-public-access-block` with all four flags true.

- Static websites do not need a public bucket: CloudFront with Origin Access Control reaches a private bucket and gives you TLS, caching, and logs.
- Time-limited sharing → presigned URLs. A presigned URL is a bearer token: anyone holding it has the access, so make the expiry minutes, not days, and generate it server-side.
- Enforce TLS in one statement: a bucket policy denying `s3:*` when `aws:SecureTransport` is `false`.
- Cross-account reads need both sides — the bucket policy in this account and the identity policy in the caller's — and objects written by another account keep that account as owner unless the bucket sets `BucketOwnerEnforced` — the cause of "I own the bucket but cannot read the file".

## S3 Throughput and Consistency

- 5,500 GET/HEAD and 3,500 PUT/COPY/POST/DELETE per second **per prefix**, and prefixes partition automatically. A high-throughput job that writes everything under `data/` is capped; sharding the key space raises the ceiling linearly.
- Strong read-after-write consistency has applied to all operations since 2020. Code that sleeps or retries to "wait for consistency" is dead weight — delete it.
- Objects: 5 TB maximum, but a single PUT caps at 5 GB. Above ~100 MB use multipart, which also gives you parallelism and resumability.
- S3 Transfer Acceleration and multipart parallelism solve different problems: acceleration helps distant clients over the public internet; parallelism helps everyone.

## Encryption

- SSE-S3 is on by default for all new objects and costs nothing. SSE-KMS adds auditability and key control, plus $0.03 per 10,000 KMS requests — which on a high-request bucket becomes the dominant cost.
- **S3 Bucket Keys reduce those KMS request charges by up to 99%** by caching a bucket-level key. Enable them on any SSE-KMS bucket; there is no downside.
- A customer-managed key means every reader needs `kms:Decrypt` in both the key policy and their identity policy. The resulting error names S3, not KMS.

## EBS

| Type | Use | Numbers |
|---|---|---|
| gp3 | Default for everything | 3,000 IOPS and 125 MB/s baseline included; IOPS and throughput are provisioned independently of size |
| gp2 | Legacy only | IOPS tied to size (3 per GB) with burst credits; gp3 is ~20% cheaper ($0.08 vs $0.10/GB-mo) — migrate |
| io2 / io2 Block Express | Databases needing a guarantee | Up to 64,000 IOPS (256,000 on Block Express) with a durability SLA |
| st1 / sc1 | Sequential big-data, cold archives | Throughput-optimized HDD, up to 500 MiB/s; terrible for random I/O |

- `modify-volume` is live: type, size, and IOPS change without downtime — but only **once per volume per 6 hours**, and the filesystem still needs growing (`growpart` + `resize2fs`/`xfs_growfs`) after a size increase.
- Volumes never shrink. The path is snapshot → restore into a smaller volume → migrate data.
- Instance throughput caps EBS throughput. A gp3 volume provisioned to 1,000 MB/s attached to an instance whose EBS bandwidth is 600 MB/s delivers 600 — check the instance type's EBS bandwidth before buying IOPS.
- Terminating an instance does **not** delete attached volumes unless `DeleteOnTermination` was set. Sweep `status=available` volumes monthly.
- Snapshots are incremental and stored in S3; deleting an old snapshot never breaks a newer one. Automate expiry with a DLM lifecycle policy; manual snapshot hygiene decays the week after someone sets it up.
- Instance store (NVMe on some instance families) is free, fast, and **gone on stop or termination**. Correct for scratch, caches, and shuffle space; catastrophic for anything else.

## EFS

- Elastic throughput is the sane default; Bursting throughput scales with stored data, so a small filesystem with heavy access throttles and the metric to watch is `BurstCreditBalance`.
- Per-operation latency is much higher than EBS. A dependency tree of tens of thousands of small files (`node_modules`, a Python venv) on EFS is pathologically slow — put those on EBS or bake them into the image.
- Lifecycle management moves untouched files to Infrequent Access automatically; enable it, and remember IA has a per-GB retrieval charge.
- Mount targets are per-AZ, and traffic to a mount target in another AZ bills as cross-AZ. Create one per AZ you run in.
- Access points give a per-application root directory and enforced POSIX identity — the clean way to share one filesystem between workloads without them seeing each other's files.

## Picking Between Them, Concretely

| Situation | Answer |
|---|---|
| Uploaded files, images, backups, logs, data lake | S3 |
| A database's data directory | EBS (gp3, or io2 with a measured IOPS requirement) |
| Shared config or content across a fleet | S3 with a local cache, or EFS if the app requires POSIX |
| Scratch space, shuffle, temp | Instance store if available, else gp3 |
| Something that must survive an instance replacing itself | Anything but instance store |
