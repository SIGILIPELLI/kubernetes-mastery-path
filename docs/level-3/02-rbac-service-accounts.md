---
description: "RBAC & Service Accounts — Every request to the API server goes through authentication (who is making this request — a user's client cert, a bearer token…"
---

# 02 · RBAC & Service Accounts

!!! note "Not run against a live cluster"
    Manifests and output below follow documented RBAC authorization
    behavior; not executed against a live cluster in this session.

## Two separate questions: who are you, and what can you do

Every request to the API server goes through **authentication** (who is
making this request — a user's client cert, a bearer token, a
ServiceAccount token) and then **authorization** (is that identity allowed
to do *this specific* thing). **RBAC (Role-Based Access Control)** is the
standard authorization mechanism: you grant permissions to a *Role*, then
bind that Role to a *subject* (a user, group, or ServiceAccount).

## ServiceAccounts: identity for Pods, not people

Every Pod runs as a ServiceAccount — if you don't specify one, it gets the
namespace's `default` ServiceAccount, which by design has essentially zero
permissions on its own.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: log-reader
  namespace: prod
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-tool
  namespace: prod
spec:
  serviceAccountName: log-reader
  containers:
    - name: tool
      image: log-tool:1.0
```

## Role and RoleBinding: namespace-scoped permissions

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-log-reader
  namespace: prod
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: log-reader-binding
  namespace: prod
subjects:
  - kind: ServiceAccount
    name: log-reader
    namespace: prod
roleRef:
  kind: Role
  name: pod-log-reader
  apiGroup: rbac.authorization.k8s.io
```

A `Role`+`RoleBinding` pair only grants access **within one namespace** —
`log-reader` can list/watch Pods and read logs in `prod`, and nothing at
all in `staging`, even though it's the exact same rule set.

## ClusterRole and ClusterRoleBinding: cluster-wide (or reusable) permissions

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
  - kind: ServiceAccount
    name: monitoring
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

Nodes, PersistentVolumes, and Namespaces themselves are cluster-scoped
resources — a namespaced `Role` cannot grant access to them at all; only a
`ClusterRole` can. A `ClusterRole` can also be bound with a `RoleBinding`
(not just `ClusterRoleBinding`) to grant its permissions within just *one*
namespace — a common pattern for reusing one well-defined ClusterRole
(like the built-in `view`/`edit`/`admin`) across many namespaces without
duplicating rule definitions.

## Verbs and the principle of least privilege

| Verb | Meaning |
|---|---|
| `get` | Fetch one named object |
| `list` | Fetch a collection |
| `watch` | Stream changes to a collection |
| `create` / `update` / `patch` / `delete` | Mutating operations |
| `deletecollection` | Bulk delete |

```bash
kubectl auth can-i delete pods --as=system:serviceaccount:prod:log-reader -n prod
# no

kubectl auth can-i list pods --as=system:serviceaccount:prod:log-reader -n prod
# yes
```

`kubectl auth can-i` is the fastest way to check effective permissions
without trial-and-error against real workloads, and works for any
subject via `--as`/`--as-group`.

## Worked example: a CI pipeline ServiceAccount scoped to one namespace, one verb set

```bash
kubectl create serviceaccount ci-deployer -n staging
kubectl create role deployer -n staging --verb=get,list,watch,create,update,patch --resource=deployments,services
kubectl create rolebinding ci-deployer-binding -n staging --role=deployer --serviceaccount=staging:ci-deployer

kubectl auth can-i delete deployments --as=system:serviceaccount:staging:ci-deployer -n staging
# no
kubectl auth can-i create deployments --as=system:serviceaccount:staging:ci-deployer -n staging
# yes
kubectl auth can-i create deployments --as=system:serviceaccount:staging:ci-deployer -n prod
# no -- RoleBinding is namespace-scoped to staging only
```

## How It Actually Works

- **RBAC authorization is one pluggable module in a chain the API server
  runs on every single request, after authentication, before admission.**
  The API server's authorizer chain (RBAC is the default/most common one,
  often combined with Node authorization for kubelets) receives the
  authenticated identity, the verb, the resource, and the namespace for
  *every* API call, and RBAC's implementation is simply: gather every
  Role/ClusterRole reachable via any RoleBinding/ClusterRoleBinding bound
  to this subject (or a group it belongs to), union their rules, and check
  whether any single rule matches — RBAC never *denies*, only grants; there
  is no explicit-deny rule type, so the effective permission set is always
  the union of everything bound to you.
- **A ServiceAccount's credential is a projected, auto-rotated JWT mounted
  into the Pod, verified by the API server against the cluster's service
  account issuer key — not a static secret by default in modern
  Kubernetes.** Since the `BoundServiceAccountTokenVolume` feature (stable
  since 1.22), kubelet requests a short-lived, audience-bound token from
  the API server's TokenRequest API and mounts it at
  `/var/run/secrets/kubernetes.io/serviceaccount/token`, refreshing it
  before expiry; the API server's authenticator verifies the JWT signature
  and its `sub` claim
  (`system:serviceaccount:<namespace>:<name>`) becomes the authenticated
  identity RBAC then evaluates rules against — this is exactly the string
  `kubectl auth can-i --as=system:serviceaccount:...` impersonates.
- **`kubectl auth can-i` doesn't run any real request — it calls the
  `SelfSubjectAccessReview` (or `SubjectAccessReview` with `--as`) API,
  which runs the *same* authorizer chain the API server would use for a
  real request, without performing it.** This is why it's a reliable,
  side-effect-free way to test permissions: it exercises the identical RBAC
  evaluation code path, just short-circuited before the actual verb is
  executed.
- **RoleBindings/ClusterRoleBindings are immutable in their `roleRef` by
  design.** You cannot edit a RoleBinding's `roleRef` field after creation
  (the API server rejects the patch) — this exists because changing what
  Role a binding points to would silently and retroactively change granted
  permissions for every already-bound subject; the intended pattern is to
  delete and recreate the binding, making the permission change an
  explicit, auditable act.

## Exercise

Create a ServiceAccount `ci-deployer` in a `staging` namespace with a Role
granting only `get`, `list`, `create`, `update` on `deployments`. Bind it
with a RoleBinding scoped to `staging`. Use `kubectl auth can-i ... --as`
to confirm it can create/update Deployments in `staging`, cannot delete
them, and cannot do anything at all in a `prod` namespace — all without
ever actually running a Pod as that ServiceAccount.
