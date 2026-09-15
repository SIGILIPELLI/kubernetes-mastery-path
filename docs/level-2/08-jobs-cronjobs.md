---
description: "Jobs & CronJobs — A bare Pod running a batch task (a DB migration, a report generation script) has no retry logic — if it fails, it just sits Failed…"
---

# 08 · Jobs & CronJobs

!!! note "Not run against a live cluster"
    Manifests and output below follow documented Job/CronJob controller
    behavior; not executed against a live cluster in this session.

## Why not just a Pod, or a Deployment

A bare Pod running a batch task (a DB migration, a report generation
script) has no retry logic — if it fails, it just sits `Failed` forever
unless something else notices. A Deployment is the wrong tool too: it's
built to keep a process running *indefinitely*, restarting it forever, which
is exactly wrong for something meant to run once to completion. A **Job**
runs a Pod (or several) to successful completion, with retries, and stops.

## A basic Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp-migrate:1.4
          command: ["./migrate.sh"]
```

```bash
kubectl apply -f job.yaml
kubectl get jobs
# NAME         COMPLETIONS   DURATION   AGE
# db-migrate   1/1           8s         10s

kubectl logs job/db-migrate
```

`restartPolicy: Never` (or `OnFailure`) is required for a Job's Pod
template — `Always` (the Deployment default) makes no sense for something
meant to terminate. `backoffLimit` caps retries on failure; exceeding it
marks the Job `Failed` rather than retrying forever.

## Parallel Jobs

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-process
spec:
  completions: 10      # total successful Pod completions needed
  parallelism: 3        # up to 3 running at once
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: batch-worker:1.0
          env:
            - name: JOB_COMPLETION_INDEX
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
```

`completionMode: Indexed` gives each Pod a stable index (0..9) it can use to
partition work (e.g. "process shard N of the input") — without it
(`NonIndexed`, the default), the Job just needs *any* 10 successful
completions with no per-Pod identity.

## CronJobs: scheduled Jobs

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"     # 2 AM daily, standard cron syntax, cluster's configured timezone
  concurrencyPolicy: Forbid  # don't start a new run if the previous is still going
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: report
              image: report-gen:2.1
```

```bash
kubectl apply -f cronjob.yaml
kubectl get cronjobs
# NAME             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# nightly-report   0 2 * * *     False     0        <none>          5s

kubectl create job --from=cronjob/nightly-report manual-run-now   # trigger one immediately, for testing
```

`concurrencyPolicy` matters for anything non-idempotent: `Allow` (default)
lets overlapping runs pile up, `Forbid` skips a new run if the last is still
active, `Replace` kills the still-running one and starts fresh.

## Worked example: a flaky Job retries, then a CronJob runs it nightly

```bash
kubectl apply -f job.yaml
kubectl get pods -l job-name=db-migrate --watch
# db-migrate-x1  0/1  Error   0
# db-migrate-x2  0/1  Error   0    <- backoffLimit retries create new Pods, not restart the old one
# db-migrate-x3  1/1  Completed

kubectl get job db-migrate
# COMPLETIONS: 1/1

kubectl apply -f cronjob.yaml
kubectl get jobs -l job-name  # after 2 AM: nightly-report-28421840 appears, owned by the CronJob
```

## How It Actually Works

- **A Job's retries create brand-new Pods, not restarted containers.**
  Unlike a liveness-probe restart (Module 04, same Pod/container), the Job
  controller watches its owned Pods; on failure (with `restartPolicy:
  Never`) it creates an entirely new Pod object counted against
  `backoffLimit`, with an exponentially increasing backoff delay between
  attempts (10s, 20s, 40s... capped at 6 minutes) — this is why
  `kubectl get pods -l job-name=...` shows multiple distinct Pod names for
  one Job, each a full scheduling+creation cycle, not a restart count on
  one Pod.
- **Completion tracking for parallel/indexed Jobs is done via a Pod
  count and (for Indexed mode) a completion-index bitmap the controller
  keeps, not by inspecting exit codes beyond success/failure.** The Job
  controller reconciles by counting currently-succeeded, active, and
  failed Pods against `completions`/`parallelism`; `Indexed` mode
  additionally tracks *which* indices have succeeded (via the annotation)
  so it only creates replacement Pods for indices that failed, not a whole
  fresh batch — this is what makes Indexed Jobs suitable for exactly-once
  partitioned work.
- **CronJob is a Job factory driven by a wall-clock reconcile, with no
  independent scheduler daemon.** The CronJob controller (part of
  kube-controller-manager) runs on a periodic tick (~10s), computes whether
  any schedule times have been "missed" since it last checked, and creates
  a new Job object (owned by the CronJob via `ownerReferences`) for each
  due run within its `startingDeadlineSeconds` window — if the controller
  itself was down across a scheduled time and the deadline window has
  passed, that run is simply skipped, not queued.
- **`concurrencyPolicy: Forbid`/`Replace` are enforced by checking
  currently-Active child Jobs at creation time, which has a real race
  window.** The controller lists Jobs it owns with `status.active > 0`
  before deciding to skip or replace — because this check and the create
  are not one atomic operation against the API server, extremely
  fast-firing schedules or controller restarts can, in rare cases, still
  produce brief overlap; `Forbid` is a strong deterrent, not an absolute
  mutual-exclusion guarantee.

## Exercise

Create a Job with `backoffLimit: 3` running a container that exits
non-zero the first two times and succeeds the third (a short shell script
using a file-based counter written to an `emptyDir` works). Watch
`kubectl get pods -l job-name=...` to see distinct Pod names per attempt.
Then wrap the same Pod template in a CronJob scheduled a couple of minutes
out and confirm a Job gets auto-created at the scheduled time via `kubectl
get jobs`.
