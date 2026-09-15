---
description: "Operators & CRDs — Everything so far (Deployments, Services, PVCs) is a built-in resource the API server knows about natively. A CustomResourceDefinition…"
---

# 01 · Operators & CRDs

!!! note "Not run against a live cluster"
    Manifests and output below follow the documented CRD/controller-runtime
    machinery; not executed against a live cluster in this session.

## Extending the API instead of working around it

Everything so far (Deployments, Services, PVCs) is a **built-in** resource
the API server knows about natively. A **CustomResourceDefinition (CRD)**
lets you register an entirely new resource type — with its own schema,
validation, and `kubectl get/apply` support — without modifying or
recompiling the API server at all. An **Operator** pairs a CRD with a
custom controller that encodes operational knowledge (backup procedures,
failover logic, version-specific upgrade steps) as code reacting to that
CRD.

## Defining a CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresclusters.db.example.com
spec:
  group: db.example.com
  names:
    kind: PostgresCluster
    plural: postgresclusters
    singular: postgrescluster
    shortNames: ["pgc"]
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["replicas", "version"]
              properties:
                replicas: { type: integer, minimum: 1 }
                version: { type: string }
                storageSize: { type: string, default: "10Gi" }
      subresources:
        status: {}
```

```bash
kubectl apply -f postgrescluster-crd.yaml
kubectl get crd postgresclusters.db.example.com
kubectl explain postgrescluster.spec   # works immediately -- schema is self-describing
```

Once the CRD is registered, `PostgresCluster` behaves like any built-in
kind: `kubectl get pgc`, `kubectl apply -f`, RBAC rules referencing
`db.example.com/postgresclusters`, and OpenAPI schema validation on
`kubectl apply` (rejecting a missing `version` field, for instance) all
work with zero custom code — this comes entirely from the CRD's schema.

## Using the custom resource

```yaml
apiVersion: db.example.com/v1
kind: PostgresCluster
metadata:
  name: orders-db
  namespace: prod
spec:
  replicas: 3
  version: "16.2"
  storageSize: 50Gi
```

```bash
kubectl apply -f orders-db.yaml
kubectl get pgc orders-db -n prod
# NAME        REPLICAS   VERSION   AGE
# orders-db   3          16.2      5s
```

At this point, nothing has actually happened besides an object being
stored in etcd — a CRD alone has no behavior. The **Operator's controller**
is what watches this object and does something about it.

## The controller: reconciling desired state into a StatefulSet

```go
// simplified controller-runtime reconcile loop (Go), illustrating the pattern
func (r *PostgresClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var pgc dbv1.PostgresCluster
    if err := r.Get(ctx, req.NamespacedName, &pgc); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    desired := buildStatefulSet(&pgc)      // e.g. replicas/version -> a real StatefulSet spec
    var existing appsv1.StatefulSet
    err := r.Get(ctx, req.NamespacedName, &existing)
    if apierrors.IsNotFound(err) {
        return ctrl.Result{}, r.Create(ctx, desired)
    }
    if !reflect.DeepEqual(existing.Spec, desired.Spec) {
        existing.Spec = desired.Spec
        return ctrl.Result{}, r.Update(ctx, &existing)
    }

    pgc.Status.ReadyReplicas = existing.Status.ReadyReplicas
    return ctrl.Result{}, r.Status().Update(ctx, &pgc)
}
```

This is the exact same reconcile pattern every built-in controller
(Deployment, StatefulSet, Job) uses: read desired state, read observed
state, converge the difference, requeue on change. The Operator pattern is
simply "write your own copy of this loop for a resource type you define,"
using `controller-runtime` (the library the Deployment/StatefulSet
controllers are conceptually built on) to handle watches, work queues, and
leader election for you.

## Worked example: the Operator turns a CRD into running Pods

```bash
kubectl apply -f postgrescluster-crd.yaml
kubectl apply -f operator-deployment.yaml -n operators   # the controller Pod itself
kubectl apply -f orders-db.yaml

kubectl get statefulset -n prod
# orders-db   3/3   <- created BY the operator, not directly by the user

kubectl patch pgc orders-db -n prod --type=merge -p '{"spec":{"replicas":5}}'
kubectl get statefulset orders-db -n prod
# 5/5 within a few seconds -- the operator's reconcile loop noticed the CRD change and updated the StatefulSet
```

## How It Actually Works

- **A CRD registers a new resource entirely inside the API server's
  generic storage machinery — there's no code specific to "PostgresCluster"
  anywhere in Kubernetes core.** The API server implements a generic
  `apiextensions-apiserver` aggregation layer that, given a CRD's OpenAPI
  schema, dynamically creates REST endpoints (`/apis/db.example.com/v1/...`)
  backed by the same etcd storage, watch, and validation machinery every
  built-in type uses — `kubectl explain`, schema validation on `apply`, and
  `kubectl get -o yaml` all come for free from this shared machinery, which
  is why CRDs feel completely native rather than bolted-on.
- **The Operator's controller is just another API client using the watch
  protocol — architecturally indistinguishable from kube-controller-manager
  itself, just running as a Pod instead of a static binary in the control
  plane.** `controller-runtime`'s manager sets up an informer (a
  local cache kept in sync via the API server's watch stream) for
  `PostgresCluster` and any owned resources (StatefulSet, Service); any
  change enqueues a reconcile key, and the reconcile function runs
  independently of whatever triggered it — this is why a manual
  `kubectl edit statefulset` conflicting with the CRD's desired state gets
  silently reverted on the next reconcile, exactly like editing a
  Deployment-owned ReplicaSet by hand does.
- **`subresources: status: {}` splits write permission for spec vs.
  status at the API level, enforced by the API server itself, not by
  convention.** With this subresource enabled, a `kubectl apply` (which
  writes to the main resource) cannot modify `.status`, and the
  controller's `r.Status().Update()` call hits a separate
  `/status` endpoint — this prevents a user's `kubectl apply` from
  clobbering fields the controller itself owns (like `readyReplicas`), the
  same separation built-in controllers rely on for Deployment/Pod status.
- **`ownerReferences` on the generated StatefulSet is what ties
  garbage collection to the CRD object, using the same GC controller every
  built-in cascading-delete relies on.** The operator sets the
  `PostgresCluster` as owner on the StatefulSet it creates; deleting the
  `PostgresCluster` object triggers the standard garbage-collector
  controller (the same one that deletes ReplicaSets when a Deployment is
  deleted) to cascade-delete the StatefulSet and its Pods — again, no
  operator-specific cleanup code is needed for this, only the correct
  owner reference.

## Exercise

Define a minimal CRD (`group: example.com`, kind `EchoService`, with
`spec.replicas` and `spec.message`) and apply it with no controller
running — confirm `kubectl get echoservice` works and stores the object,
but nothing else happens. Then sketch (in comments or pseudocode, no need
to fully implement a Go operator) what a reconcile function would need to
do to turn that object into a Deployment running an echo server with
`spec.replicas` Pods, each printing `spec.message` — and identify which
part of that logic the CRD's schema alone could never provide.
