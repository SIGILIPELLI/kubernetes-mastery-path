---
description: "Deployments & ReplicaSets — In practice, you almost always create a Deployment, not a bare ReplicaSet — the Deployment gives you the ReplicaSet's…"
---

# 06 · Deployments & ReplicaSets

!!! note "Not run against a live cluster"
    Manifests and example `kubectl` output below follow documented
    Deployment/ReplicaSet behavior; not executed against a live cluster in
    this session.

## Why Deployments exist

A bare Pod (Module 05) has no self-healing and no way to run multiple
identical copies. **ReplicaSet** and **Deployment** solve this, layered on
top of each other:

```text
Deployment  --manages-->  ReplicaSet  --manages-->  Pods
```

- A **ReplicaSet** ensures a specified number of identical Pod replicas are
  running at all times — if one dies, it creates a replacement; if there
  are too many (e.g., after a manual Pod deletion race), it deletes the
  extra.
- A **Deployment** manages ReplicaSets on your behalf and adds **rollout**
  behavior: when you change the Pod template (new image, new env var), the
  Deployment creates a *new* ReplicaSet and gradually shifts replicas from
  the old one to the new one (a rolling update), keeping a history so you
  can roll back.

In practice, you almost always create a **Deployment**, not a bare
ReplicaSet — the Deployment gives you the ReplicaSet's self-healing *plus*
safe, controlled updates.

## A Deployment manifest

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
```

Key structure to internalize:

- `spec.replicas` — desired number of Pod copies.
- `spec.selector.matchLabels` — how the Deployment (via its ReplicaSet)
  identifies *which* Pods belong to it. This **must match**
  `spec.template.metadata.labels` — a mismatch is a validation error.
- `spec.template` — the Pod spec to stamp out N times. Everything under
  `template` is exactly a Pod spec, identical in shape to Module 05's bare
  Pod manifest.

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
# NAME   READY   UP-TO-DATE   AVAILABLE   AGE
# web    3/3     3            3           10s

kubectl get replicasets
# NAME             DESIRED   CURRENT   READY   AGE
# web-7d8f9c6b7d   3         3         3       10s

kubectl get pods -l app=web
# NAME                   READY   STATUS    RESTARTS   AGE
# web-7d8f9c6b7d-2xk9p   1/1     Running   0          10s
# web-7d8f9c6b7d-h8k2p   1/1     Running   0          10s
# web-7d8f9c6b7d-m4t7q   1/1     Running   0          10s
```

Notice the Pod names are `<deployment-name>-<replicaset-hash>-<random>` —
the middle segment is a hash of the Pod template, which changes whenever you
update the template (new image, changed env, etc.), which is how the
Deployment tells "old" Pods from "new" Pods during a rollout.

## Self-healing in action

```bash
kubectl delete pod web-7d8f9c6b7d-2xk9p
kubectl get pods -l app=web --watch
# NAME                   READY   STATUS        RESTARTS   AGE
# web-7d8f9c6b7d-2xk9p   1/1     Terminating   0          2m
# web-7d8f9c6b7d-9j3fp   0/1     ContainerCreating   0     1s
# web-7d8f9c6b7d-9j3fp   1/1     Running       0          3s
```

The ReplicaSet controller noticed the actual replica count dropped to 2 and
immediately created a new Pod to bring it back to the desired 3 — this is
the reconciliation loop from Module 02, made concrete.

## Scaling

```bash
kubectl scale deployment web --replicas=5
kubectl get deployments
# NAME   READY   UP-TO-DATE   AVAILABLE   AGE
# web    5/5     5            5           4m
```

Or declaratively — edit `replicas: 5` in the YAML and `kubectl apply -f
deployment.yaml` again; both approaches converge on the same result, but the
declarative one keeps your YAML as the source of truth.

## Rolling updates (preview — Level 2 goes deep)

Changing the image and re-applying triggers a rolling update automatically:

```bash
kubectl set image deployment/web web=nginx:1.27
kubectl rollout status deployment/web
# Waiting for deployment "web" rollout to finish: 1 out of 3 new replicas have been updated...
# deployment "web" successfully rolled out
```

Behind the scenes, a **new ReplicaSet** is created for the new Pod template,
and the Deployment scales it up while scaling the old ReplicaSet down,
respecting the update strategy's `maxSurge`/`maxUnavailable` settings
(Level 2 · Module 05 covers the mechanics and rollback in full).

## Why you rarely touch ReplicaSets directly

You *can* create a bare ReplicaSet manifest (`kind: ReplicaSet`), but there's
almost no reason to: it gives you replica-count self-healing without
rollout management, which is strictly a subset of what a Deployment gives
you. `kubectl get replicasets` is something you'll use to *inspect* what a
Deployment created, not something you'll author directly.

## Worked example: label selectors matter

```bash
kubectl get pods --show-labels
# NAME                   READY   STATUS    RESTARTS   AGE   LABELS
# web-7d8f9c6b7d-9j3fp   1/1     Running   0          1m    app=web,pod-template-hash=7d8f9c6b7d

kubectl get pods -l app=web            # filter by label
kubectl label pod web-7d8f9c6b7d-9j3fp tier=frontend   # add a label
kubectl get pods -l app=web,tier=frontend
```

Labels are how nearly everything in Kubernetes — Deployments selecting
Pods, Services routing to Pods, `kubectl` filtering — finds the right
objects. There's no hidden parent/child pointer; it's all label matching.

## How It Actually Works

A Deployment never touches Pods directly — it manages through a chain of
owner references, and the `pod-template-hash` label you saw above is the
load-bearing detail that makes rolling updates possible:

- **Three objects, three controllers, one chain.** A Deployment object is
  watched by the Deployment controller, which creates/updates a
  ReplicaSet; that ReplicaSet is watched by the ReplicaSet controller,
  which creates/deletes Pods. Each level sets an `ownerReferences` entry
  on the object it creates, pointing back to its owner's UID — this is
  what makes `kubectl delete deployment` cascade (the garbage collector
  controller watches for owner deletion and deletes dependents) and
  what `kubectl get pods -o yaml` reveals in `metadata.ownerReferences`.
- **`pod-template-hash` is a real hash, computed by the Deployment
  controller.** It hashes the Pod template (the `spec.template` field,
  after normalizing it) to a short string, and uses that hash both as a
  label on the ReplicaSet and as a `selector` the ReplicaSet uses to
  claim only Pods generated from that exact template version. This is
  the actual mechanism that lets multiple ReplicaSets (old and new)
  coexist under one Deployment during a rolling update without their
  Pod sets overlapping — change the template even slightly and you get
  a new hash, hence a brand-new ReplicaSet, never a mutation of the old
  one.
- **The ReplicaSet controller's reconciliation is a label-selector
  count-and-diff, exactly as in Module 01.** On every relevant watch
  event, it lists Pods matching `spec.selector`, counts how many are
  not terminating, and compares that count to `spec.replicas`. Too few:
  it issues that many `Pod` create calls (via the API server, which then
  triggers the scheduler). Too many: it deletes the newest Pods first by
  default. Because this is selector-based rather than identity-based, if
  you manually attach a matching label to an unrelated Pod, the
  ReplicaSet controller will "adopt" it (and may delete one of its own
  Pods to keep the count correct) — exactly why label hygiene matters.
- **Scaling (`kubectl scale`) is just a `PATCH` to `spec.replicas`.**
  There's no separate "scaling subsystem" — it's the same reconciliation
  loop reacting to a spec change like any other, which is also why
  scaling and healing use identical code paths and are indistinguishable
  to the controller.

## Exercise

Apply the `web` Deployment above with 3 replicas. Use `kubectl get pods -l
app=web --watch` in one terminal while you delete one Pod in another and
observe the replacement appear. Then scale to 1 replica, watch two Pods
terminate, scale back to 3, and finally run `kubectl set image
deployment/web web=nginx:1.26-alpine` and watch `kubectl rollout status
deployment/web` report the rollout completing. Finish with `kubectl get
replicasets` and note that both the old and new ReplicaSet objects still
exist (old one scaled to 0) — this history is what makes rollback possible.
