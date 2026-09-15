---
description: "Helm Basics (Charts & Values) — A real app is often a dozen+ manifests (Deployment, Service, Ingress, ConfigMap, PVC...) that need slightly different…"
---

# 06 · Helm Basics (Charts & Values)

!!! note "Not run against a live cluster"
    Commands and output below follow documented Helm v3 behavior; not
    executed against a live cluster in this session.

## The problem: raw YAML doesn't parameterize or version well

A real app is often a dozen+ manifests (Deployment, Service, Ingress,
ConfigMap, PVC...) that need slightly different values per environment
(dev vs. prod replica counts, image tags, hostnames). Copy-pasting and
hand-editing YAML per environment doesn't scale and has no versioning or
rollback story of its own. **Helm** packages a set of manifests as a
templated, versioned unit called a **chart**.

## Chart anatomy

```text
mychart/
├── Chart.yaml          # name, version, description
├── values.yaml         # default configuration values
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── _helpers.tpl    # reusable template snippets
```

```yaml
# Chart.yaml
apiVersion: v2
name: mychart
version: 0.1.0
appVersion: "1.4.0"
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myapp
  tag: "1.4.0"
service:
  port: 80
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-mychart
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
```

Templates use Go's `text/template` syntax; `.Values` comes from
`values.yaml` (overridable per install), `.Release` carries install-time
metadata (name, namespace, revision), `.Chart` carries `Chart.yaml`
fields.

## Installing, upgrading, rolling back

```bash
helm install shop ./mychart
# NAME: shop
# STATUS: deployed
# REVISION: 1

helm upgrade shop ./mychart --set image.tag=1.5.0
# REVISION: 2

helm history shop
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         deployed    Upgrade complete

helm rollback shop 1
# REVISION: 3 (a new revision recording "went back to revision 1's config")
```

Every `helm install`/`upgrade`/`rollback` creates a new numbered
**release revision** — rollback isn't a special undo mechanism, it's just
another upgrade whose target manifest happens to be an old revision's.

## Overriding values

```bash
helm install shop ./mychart --set replicaCount=5 --set image.tag=1.5.0

helm install shop ./mychart -f values-prod.yaml
```

```yaml
# values-prod.yaml
replicaCount: 10
image:
  tag: "1.5.0"
```

`-f` merges on top of the chart's own `values.yaml`; `--set` merges on top
of that — later sources win field-by-field, not whole-file replacement.

## Inspecting what a chart will actually produce

```bash
helm template shop ./mychart --set replicaCount=5
# renders final YAML to stdout, no cluster contact -- great for reviewing/diffing before applying

helm install shop ./mychart --dry-run --debug
# renders AND validates against the connected cluster's API without creating anything
```

## Public charts

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm show values bitnami/postgresql | less
helm install my-db bitnami/postgresql --set auth.postgresPassword=changeme
```

## Worked example: chart install, upgrade, rollback

```bash
helm install shop ./mychart --set replicaCount=3
kubectl get deploy shop-mychart
# 3/3 ready

helm upgrade shop ./mychart --set replicaCount=3,image.tag=1.5.0-broken
kubectl get pods -l app=shop
# CrashLoopBackOff

helm rollback shop 1
kubectl get deploy shop-mychart -o jsonpath='{.spec.template.spec.containers[0].image}'
# myapp:1.4.0 -- back to the known-good release
```

## How It Actually Works

- **Helm has no in-cluster server component (since v3) — it's a client
  that renders templates locally and applies plain manifests via the
  Kubernetes API.** `helm install` runs the Go template engine against
  `templates/*.yaml` with `.Values`/`.Release`/`.Chart` in scope, produces
  final YAML, and submits it the same way `kubectl apply` would — Helm adds
  no controller, no CRD reconciliation, no admission webhook by default.
- **A release's state is itself stored as a Kubernetes Secret in the
  release's namespace.** Each revision is persisted as a Secret (type
  `helm.sh/release.v1`, name like `sh.helm.release.v1.shop.v2`) whose data
  includes the fully rendered manifest and the values used — `helm
  history`/`rollback` work by reading these Secrets back, not by asking the
  live objects what they currently look like, which is why deleting these
  Secrets by hand (or a namespace) silently destroys Helm's memory of a
  release even if the underlying Deployment/Service objects still exist.
- **Upgrade computes a three-way diff, not a blind re-apply.** `helm
  upgrade` diffs the *previous* rendered manifest (from the stored release
  Secret), the *new* rendered manifest, and the *live* cluster state, then
  issues a strategic merge patch — this is how Helm detects and removes
  resources that existed in the old release but were deleted from the new
  chart version (a plain re-`apply` of only the new templates would leave
  orphans behind).
- **Rollback is possible purely because past fully-rendered manifests are
  retained, not because Kubernetes tracks "chart history" natively.**
  `helm rollback shop 1` re-reads revision 1's stored Secret, takes its
  already-rendered manifest verbatim, and applies it as a *new* revision —
  this is why a chart's `templates/` directory or `values.yaml` changing
  later has zero effect on what a rollback to an old revision produces.

## Exercise

Write a minimal chart with `Chart.yaml`, `values.yaml` (image repo/tag,
replica count), and a `templates/deployment.yaml` that uses both values.
Run `helm template` to inspect the rendered output before installing,
then `helm install`, `helm upgrade --set image.tag=<bad-tag>`, and `helm
rollback` back to revision 1 — check `helm history` after each step to see
the revision count grow.
