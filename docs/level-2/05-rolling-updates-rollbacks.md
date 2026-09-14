# 05 · Rolling Updates & Rollbacks

!!! note "Not run against a live cluster"
    Manifests and output below follow documented Deployment controller
    behavior; not executed against a live cluster in this session.

## The default: rolling updates

When you change a Deployment's Pod template (most commonly the image tag),
Kubernetes doesn't stop all Pods and start new ones at once — it performs a
**rolling update**, gradually replacing old Pods with new ones while keeping
the app available throughout.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # at most 1 of 6 can be down during the rollout
      maxSurge: 1         # at most 1 extra Pod above 6 can exist during the rollout
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: app
          image: myapp:1.4
          readinessProbe:
            httpGet: { path: /ready, port: 8080 }
```

```bash
kubectl set image deployment/web app=myapp:1.5
kubectl rollout status deployment/web
# Waiting for deployment "web" rollout to finish: 2 out of 6 new replicas have been updated...
# deployment "web" successfully rolled out
```

## Watching a rollout in progress

```bash
kubectl get rs -l app=web
# NAME          DESIRED   CURRENT   READY   AGE
# web-6f9b8c7   5         5         5       10m   <- old ReplicaSet, scaling down
# web-8a2d1e4   1         1         1       8s    <- new ReplicaSet, scaling up
```

A rolling update never edits the running Deployment's Pods in place — it
creates a **new ReplicaSet** for the new template and shifts replica counts
between old and new ReplicaSets step by step, bounded by `maxUnavailable`
and `maxSurge`, until the new one holds all replicas and the old one is
scaled to zero (but kept around, per `revisionHistoryLimit`, for rollback).

## Rollout history and rollback

```bash
kubectl rollout history deployment/web
# REVISION  CHANGE-CAUSE
# 1         kubectl apply -f web.yaml
# 2         kubectl set image deployment/web app=myapp:1.5

kubectl rollout undo deployment/web
# deployment.apps/web rolled back

kubectl rollout undo deployment/web --to-revision=1
```

`CHANGE-CAUSE` is only populated if you set the
`kubernetes.io/change-cause` annotation (or used `--record`, deprecated) —
otherwise history entries show `<none>`, which is a common surprise when
you go looking for it later.

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "bump to myapp:1.5 for CVE fix"
```

## Pausing a rollout mid-flight

```bash
kubectl rollout pause deployment/web
kubectl set image deployment/web app=myapp:1.6
kubectl set resources deployment/web -c app --limits=cpu=500m
# neither change is rolled out yet -- both batched
kubectl rollout resume deployment/web
# now a single rollout applies both changes together
```

## Worked example: a bad rollout gets caught and rolled back

```bash
kubectl set image deployment/web app=myapp:1.5-broken
kubectl rollout status deployment/web --timeout=60s
# error: deployment "web" exceeded its progress deadline

kubectl get pods -l app=web
# new Pods stuck CrashLoopBackOff; old Pods still Running (maxUnavailable protected them)

kubectl rollout undo deployment/web
kubectl rollout status deployment/web
# deployment "web" successfully rolled out
```

Because `maxUnavailable: 1` capped how many old Pods could be torn down
before new ones proved ready, the bad rollout only ever affected a fraction
of capacity — readiness probes (Module 04) are what make this safety net
work at all; without them, a broken-but-still-"Running" Pod would look
ready immediately and the rollout would proceed to completion regardless.

## How It Actually Works

- **The Deployment controller never touches Pods directly — it only
  manages ReplicaSets, and ReplicaSets manage Pods.** Editing a Deployment's
  `spec.template` causes the Deployment controller to compute a hash of
  that template (the `pod-template-hash` label) and either find an existing
  ReplicaSet with a matching hash (used for rollback — the old ReplicaSet
  is reused rather than recreated) or create a new one. All replica-count
  shuffling happens by the Deployment controller PATCHing the `.spec.replicas`
  field on the *old* and *new* ReplicaSet objects; the ReplicaSet controller
  is a completely separate reconciliation loop that reacts to those
  replica-count changes by creating/deleting Pods to match.
- **`maxSurge`/`maxUnavailable` bound two independent inequalities the
  Deployment controller checks on every reconcile tick.** It computes, from
  current ready-Pod counts across both ReplicaSets, how many new Pods it's
  currently allowed to add (bounded by surge) and how many old Pods it's
  currently allowed to remove (bounded by unavailability), then issues the
  smallest safe replica-count change — this is why a rollout with a slow
  readiness probe visibly "stalls" at each step: the controller is polling,
  waiting for newly-created Pods to actually report Ready before it's
  permitted to remove more old ones.
- **Rollback is not a special code path — it's the same reconcile logic
  run against a template copied from an old ReplicaSet.** `kubectl rollout
  undo` finds the target revision's ReplicaSet (revisions are tracked via
  the `deployment.kubernetes.io/revision` annotation on each ReplicaSet),
  copies its Pod template back into the Deployment's `spec.template`, and
  the normal rolling-update machinery takes it from there — treating the
  "old" version as if it were simply the newly desired one.
- **`revisionHistoryLimit` is why rollback has a horizon.** Old
  ReplicaSets are scaled to 0 but not deleted, up to this many past
  revisions (default 10); the Deployment controller garbage-collects older
  ones on each reconcile — `kubectl rollout undo --to-revision=N` fails
  with a "cannot get revision" error for anything already collected.

## Exercise

Deploy `web` with 6 replicas, `maxUnavailable: 1`, `maxSurge: 1`, and a
readiness probe. Trigger a rollout to a deliberately broken image tag and
watch `kubectl get rs` and `kubectl get pods` while it stalls — confirm the
old ReplicaSet still holds ready Pods throughout. Run `kubectl rollout
undo` and confirm it converges back to the last-good revision, then check
`kubectl rollout history` to see both revisions recorded.
