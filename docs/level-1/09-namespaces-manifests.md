# 09 · Namespaces & YAML Manifest Structure

!!! note "Not run against a live cluster"
    Manifests and command output below follow documented Kubernetes
    behavior; not executed against a live cluster in this session.

## What namespaces are for

A **namespace** partitions a single physical cluster into multiple virtual
clusters. Most namespaced resources (Pods, Deployments, Services,
ConfigMaps, Secrets — but *not* cluster-scoped resources like Nodes or
PersistentVolumes) live inside exactly one namespace, and names only need
to be unique **within** a namespace — you can have a `web` Deployment in
both `staging` and `production` namespaces without conflict.

Typical uses:

- Separating environments (`dev`, `staging`, `production`) on a shared
  cluster.
- Separating teams/projects sharing a cluster, often combined with RBAC
  (Level 3) and resource quotas to isolate them from each other.
- Kubernetes itself uses `kube-system` for control-plane component Pods and
  `kube-public`/`kube-node-lease` for other internal purposes.

## Working with namespaces

```bash
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   10d
# kube-system       Active   10d
# kube-public       Active   10d
# kube-node-lease   Active   10d
```

Create one:

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
```

```bash
kubectl apply -f namespace.yaml
# or imperatively:
kubectl create namespace staging
```

Deploy into it — either put `namespace:` in the manifest's metadata:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: staging
spec:
  # ...
```

...or specify it on the command line:

```bash
kubectl apply -f deployment.yaml -n staging
kubectl get pods -n staging
```

If neither is set, resources land in whatever namespace your current
context defaults to (`default`, unless you changed it — Module 04).

## What DNS and Service discovery look like across namespaces

Recall from Module 07: a Service's DNS name is
`<name>.<namespace>.svc.cluster.local`. Within the same namespace, the
short name (`web`) works; from a *different* namespace you need at least
`web.staging`:

```bash
# From a pod in the "default" namespace, reaching a service in "staging":
wget -qO- http://web.staging
wget -qO- http://web.staging.svc.cluster.local
```

## Deleting a namespace deletes everything in it

```bash
kubectl delete namespace staging
```

This cascades — every Pod, Deployment, Service, ConfigMap, Secret, etc.
inside `staging` is deleted with it. This is powerful for tearing down a
whole environment in one command, and correspondingly dangerous if run
against the wrong namespace — always double-check
`kubectl config current-context` and `-n <namespace>` before deleting.

## YAML manifest structure, formalized

You've been writing manifests since Module 05 — here's the structure made
explicit, since Module 10's project asks you to compose several kinds
together in one workflow.

### The four top-level fields, every time

```yaml
apiVersion: <group>/<version>   # or just <version> for core resources
kind: <ResourceType>
metadata:
  name: <string>
  namespace: <string>            # optional; defaults per context
  labels: { key: value, ... }    # optional; used by selectors
  annotations: { key: value, ... } # optional; non-identifying metadata
spec:
  # shape is entirely specific to `kind`
```

`apiVersion` cheat sheet for what you've used so far:

| Kind | apiVersion |
|---|---|
| Pod, Service, ConfigMap, Secret, Namespace | `v1` |
| Deployment, ReplicaSet | `apps/v1` |

`kubectl explain <kind>` gives you the authoritative field reference
straight from the cluster's API, without leaving the terminal:

```bash
kubectl explain deployment.spec
kubectl explain deployment.spec.template.spec.containers
```

### Multiple objects in one file

YAML's `---` document separator lets you define several resources in a
single manifest file — very common for "everything one small app needs":

```yaml
# app.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  LOG_LEVEL: "info"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
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
          envFrom:
            - configMapRef:
                name: web-config
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f app.yaml     # creates/updates all three objects
kubectl delete -f app.yaml    # deletes all three
```

Ordering inside the file mostly doesn't matter — Kubernetes will happily
accept a Deployment that references a ConfigMap defined later in the same
file, since `apply` first writes all objects to the API server and
reconciliation happens asynchronously. Convention still favors config
before consumers, top to bottom, for human readability.

### Directory-based organization

For anything beyond a toy example, one file per resource (or per logical
group) in a directory is standard:

```text
manifests/
  namespace.yaml
  configmap.yaml
  deployment.yaml
  service.yaml
```

```bash
kubectl apply -f manifests/
```

`kubectl apply -f <dir>` applies every YAML file in the directory
(non-recursively by default; add `-R` to recurse into subdirectories).

## Exercise

Create a `staging` namespace. Write a directory `manifests/` containing
`configmap.yaml`, `deployment.yaml`, and `service.yaml` (reusing the
`web-config`/`web` names from Modules 06-08), each with `namespace:
staging` set, and apply the whole directory with one `kubectl apply -f
manifests/`. Confirm with `kubectl get all -n staging`, then delete the
entire environment with a single `kubectl delete namespace staging` and
confirm everything is gone.
