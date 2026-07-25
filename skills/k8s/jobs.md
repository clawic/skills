# Jobs and CronJobs — Batch That Finishes, Retries That Stop

Batch workloads fail differently from services: the failure mode is not "down", it is "ran twice", "never ran", or "retried 6 times and gave up while nobody watched".

## Job Fields That Decide Behavior

| Field | Default | Effect |
|---|---|---|
| `completions` | 1 | How many successful pods make the Job complete |
| `parallelism` | 1 | How many pods run at once |
| `backoffLimit` | 6 | Pod failures tolerated before the Job is marked Failed |
| `activeDeadlineSeconds` | none | Wall-clock kill switch; it overrides `backoffLimit` and fails the Job immediately |
| `ttlSecondsAfterFinished` | none | Automatic cleanup of the finished Job and its pods |
| `completionMode` | NonIndexed | `Indexed` gives each pod a `JOB_COMPLETION_INDEX` for sharded work |
| `suspend` | false | Creates the Job without starting pods; a queue controller flips it |

- Pod failure backoff starts at 10s and doubles to a 6-minute cap: 10 + 20 + 40 + 80 + 160 + 320 = 630s. A Job with `backoffLimit: 6` and a pod that fails instantly therefore takes about 10 minutes to be marked Failed, not 6 seconds — size alert windows to that, not to the pod's runtime.
- `restartPolicy: OnFailure` restarts the container inside the same pod (fast, but the previous logs are harder to reach). `Never` creates a fresh pod per attempt, leaving each failed pod as evidence. For anything you will need to debug, `Never` plus a generous TTL is worth the clutter.
- `activeDeadlineSeconds` is the only field that bounds total time. Without it, a hung job holds its resources indefinitely and no retry counter ever advances.
- `podFailurePolicy` distinguishes causes: ignore the exit code produced by a spot-instance SIGTERM so preemption does not consume `backoffLimit`, and fail fast on an application exit code that will never succeed. Confirm support on the cluster version before relying on it.
- Most of a Job's spec is immutable after creation. Changing anything means delete and recreate — which is why templating Jobs through the same pipeline as Deployments needs a name that changes per run.

## Three Batch Shapes

- **Fixed completion count**: `completions: 100, parallelism: 10` — the controller runs pods until 100 succeed. Work must be partitioned externally, or by `JOB_COMPLETION_INDEX` in Indexed mode.
- **Work queue**: `parallelism: N`, `completions` unset — every pod pulls from a queue and exits 0 when the queue is empty. The Job completes when one pod succeeds and the rest finish; each worker must be idempotent because a redelivered message is normal.
- **Single task**: migrations, imports, one-off repairs. The one that most needs `activeDeadlineSeconds` and a lock, because it is the one people run twice by hand.

## CronJobs

| Field | Default | Effect |
|---|---|---|
| `schedule` | required | Standard cron, evaluated in UTC unless `timeZone` is set |
| `timeZone` | UTC | IANA name; without it, "3am" drifts by an hour twice a year |
| `concurrencyPolicy` | Allow | `Allow` stacks overlapping runs, `Forbid` skips, `Replace` kills the old one |
| `startingDeadlineSeconds` | none | How late a missed run may still start |
| `successfulJobsHistoryLimit` | 3 | Completed Jobs retained |
| `failedJobsHistoryLimit` | 1 | Failed Jobs retained — the one you need is often already gone |
| `suspend` | false | Pause without deleting |

- The permanent-stop trap: if the controller sees more than 100 missed schedules and no `startingDeadlineSeconds`, it stops scheduling entirely and logs "Cannot determine if job needs to be started: too many missed start times". A controller outage over a weekend on a `*/5` schedule reaches 100 in under nine hours. Always set `startingDeadlineSeconds` (a value shorter than the interval), and alert on last-successful-run age rather than on failures.
- Execution is at-least-once in practice, not exactly-once: clock skew, controller restarts, and `Replace` can all produce a double run. Every CronJob body must be idempotent or hold a lock (a lease object, an advisory lock in the database).
- `concurrencyPolicy: Allow` plus a job that occasionally hangs is how a namespace hits its pod quota at 4am and every other workload stops scheduling (`resources.md`).
- Raise `failedJobsHistoryLimit` to at least 3 before you need it. The default keeps one failure, and the interesting one is usually the first, not the last.

## Cleanup

- `ttlSecondsAfterFinished` on every Job. Without it, finished pods accumulate, count against `count/pods` quota, and slow down every `kubectl get pods`.
- Set the TTL longer than your debugging window: 86400 for anything whose failure you would investigate, shorter for high-frequency noise.
- Deleting a Job deletes its pods (and their logs). If logs matter, ship them off-cluster; the Job object is not a log store (`observability` skill).

## Batch and the Cluster

- Cluster autoscaler will not scale down a node running a Job pod that cannot be moved. Long jobs on shrinking node pools either block scale-down (cost) or get evicted mid-run (progress lost) — annotate deliberately with `cluster-autoscaler.kubernetes.io/safe-to-evict` and make the job resumable (`autoscaling.md`).
- Spot and preemptible nodes are excellent for retryable batch and terrible for a 6-hour job with no checkpointing. Pair with `podFailurePolicy` so preemption does not burn the retry budget.
- Batch pods without resource limits are the most common cause of node memory pressure at night, when no human is watching the eviction (`nodes.md`).
- A Job's PriorityClass should sit below interactive services and above nothing else; `preemptionPolicy: Never` keeps a big batch from evicting anything while still queueing ahead of other batch (`scheduling.md`).

## Debugging A Failed Job

1. `kubectl get pods -l job-name=<job> --show-labels` — every attempt, with status. If none exist, the TTL already cleaned them or the pods were never created (quota, admission).
2. `kubectl logs job/<job>` reads one pod; for a specific attempt use the pod name. `--previous` for an `OnFailure` container restart.
3. `kubectl describe job <job>` — the `Conditions` block distinguishes `BackoffLimitExceeded` from `DeadlineExceeded`, which are entirely different investigations.
4. Job created but no pods at all → ResourceQuota rejection, a scheduling gate, or `suspend: true`. The event lands on the Job, not on a pod.
5. CronJob with no Jobs → `kubectl describe cronjob` and read `Last Schedule Time`. A stale timestamp means the too-many-missed-starts stop, a suspended CronJob, or a schedule that does not mean what its author thought.
6. Job succeeded but the work did not happen → the container exited 0 on an error path. Batch entrypoints must propagate exit codes; `set -euo pipefail` in a shell wrapper is the difference between a red Job and silent data loss (`bash` skill).
