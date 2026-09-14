# 01 · StatefulSets & DaemonSets

!!! note "Not run against a live cluster"
    Manifests and output below follow documented controller behavior; not
    executed against a live cluster in this session.

## StatefulSets: when Pod identity matters

A Deployment's Pods are interchangeable — any replica can be deleted and
replaced by an identical one with a new random name and IP, which is fine
for stateless web tiers. Some workloads (databases, message brokers,
anything doing peer discovery or leader election) need each replica to have
a **stable, predictable identity**: the same name, the same DNS entry, the
same PVC, every time it's recreated.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres    # must match a headless Service
  replicas: 3
  selector:
    matchLabels: { app: postgres }
  template:
    metadata:
      labels: { app: postgres }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports: [{ containerPort: 5432 }]
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 10Gi } }
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None    # headless -- required for StatefulSet DNS
  selector: { app: postgres }
  ports: [{ port: 5432 }]
```

```bash
kubectl apply -f statefulset.yaml
kubectl get pods -l app=postgres
# postgres-0   Running
# postgres-1   Running   <- created only after postgres-0 is Ready
# postgres-2   Running   <- created only after postgres-1 is Ready

kubectl get pvc
# data-postgres-0   Bound   10Gi
# data-postgres-1   Bound   10Gi
# data-postgres-2   Bound   10Gi
```

Each Pod gets a name of `<statefulset>-<ordinal>` (`postgres-0`,
`postgres-1`, ...), a stable DNS name via the headless Service
(`postgres-0.postgres.default.svc.cluster.local`), and its **own** PVC from
`volumeClaimTemplates` — deleting `postgres-1` recreates a Pod named
`postgres-1` that reattaches to the *same* PVC, not a fresh one.

## Ordered rollout and scale-down

By default, StatefulSets create/update/delete Pods **one at a time, in
order** (`0, 1, 2, ...` up; reverse down) — the next ordinal isn't touched
until the previous one is Ready. This matters for anything doing
replication where node 0 might be a primary other nodes depend on at
startup.

```bash
kubectl scale statefulset postgres --replicas=1
kubectl get pods -l app=postgres -w
# postgres-2 terminates first, then postgres-1, postgres-0 stays
```

`OrderedReady` (default) can be relaxed to `Parallel` for stateless-ish
uses of StatefulSets that only want stable identity, not ordering:

```yaml
spec:
  podManagementPolicy: Parallel
```

## DaemonSets: exactly one Pod per node

Some workloads exist to serve *the node itself* — log shippers, node
monitoring agents, CNI plugins — and need exactly one instance on every
(or every matching) node, automatically added/removed as nodes join/leave.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels: { app: node-exporter }
  template:
    metadata:
      labels: { app: node-exporter }
    spec:
      tolerations:
        - operator: Exists    # so it also schedules on tainted/control-plane nodes
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.8.1
          ports: [{ containerPort: 9100, hostPort: 9100 }]
```

```bash
kubectl apply -f daemonset.yaml
kubectl get daemonset node-exporter
# DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
# 3         3         3       3            3           <none>

kubectl get pods -l app=node-exporter -o wide
# one Pod per node, regardless of replicas -- there's no replicas field on a DaemonSet at all
```

A DaemonSet has no `replicas` field — the desired count is implicitly "one
per node matching `spec.template.spec.nodeSelector`/affinity," recalculated
continuously as nodes are added, drained, or removed.

## Worked example: losing an ordinal vs. losing a node

```bash
kubectl delete pod postgres-1
kubectl get pods -l app=postgres
# postgres-1 recreated, same name, reattaches to data-postgres-1

kubectl cordon node-3 && kubectl drain node-3 --ignore-daemonsets
kubectl get pods -l app=node-exporter -o wide
# node-exporter Pod on node-3 terminated and NOT rescheduled elsewhere
# (a DaemonSet Pod's "elsewhere" doesn't exist -- it belongs to that node specifically)
```

## How It Actually Works

- **A StatefulSet's ordinal identity is implemented by predictable naming
  and per-ordinal PVC claim names, not by any special scheduling
  constraint tying a Pod to a specific node.** The StatefulSet controller
  creates Pods named `<name>-0`, `<name>-1`, ... deterministically and, for
  each, a PVC named `<volumeClaimTemplateName>-<name>-<ordinal>` — when
  Pod `postgres-1` is deleted and recreated, the controller creates a new
  Pod object with the *same* name and a `volumeClaimTemplate` reference
  that resolves to the *same already-existing* PVC (it checks for an
  existing PVC of that name before creating one), and the scheduler is free
  to place the new Pod on any node the PVC's access mode and topology allow.
- **Ordering is enforced by the controller waiting on `status.conditions`
  before proceeding, one ordinal at a time — a purely control-loop
  behavior, not a Kubernetes primitive like `dependsOn`.** On each
  reconcile, the StatefulSet controller looks at the lowest-ordinal Pod not
  yet Ready and does nothing to higher ordinals until it observes that Pod
  transition to Ready (or Terminated, on scale-down, in reverse) —
  `podManagementPolicy: Parallel` simply disables this wait, issuing all
  create/delete operations at once.
- **DaemonSet Pods are scheduled by the DaemonSet controller computing
  node membership directly — historically bypassing the general scheduler
  for the "which node" decision, though modern Kubernetes has the default
  scheduler handle it via a required node affinity the controller
  injects.** The DaemonSet controller watches the Node list and, for every
  node matching the DaemonSet's node selector/affinity/tolerations, ensures
  exactly one Pod exists bound to that node (via a `nodeAffinity` term
  naming that specific node, injected into the Pod spec at creation) — a
  new node joining the cluster triggers exactly one new Pod, and a node
  being removed triggers that Pod's garbage collection, with no
  "rescheduling elsewhere" concept because the Pod was never generic to
  begin with.
- **`kubectl drain --ignore-daemonsets` is a client-side flag that changes
  what `drain` will evict, not a change to the DaemonSet controller's
  behavior.** Draining a node evicts ordinary Pods (respecting
  PodDisruptionBudgets) so they get rescheduled elsewhere; DaemonSet Pods
  are deliberately skipped by this eviction step because rescheduling them
  "elsewhere" is meaningless — they still get cleaned up automatically once
  the node object itself is deleted or marked unschedulable in a way the
  DaemonSet controller's node-selector logic excludes it.

## Exercise

Deploy the 3-replica `postgres` StatefulSet with its headless Service and
confirm `postgres-0.postgres.default.svc.cluster.local` resolves from a
debug Pod. Delete `postgres-1` and verify it comes back with the same name
and reattaches to `data-postgres-1` rather than getting a fresh PVC. Then
deploy the `node-exporter` DaemonSet and confirm `kubectl get pods -o wide`
shows exactly one per node, with no `replicas` field to configure that
count directly.
