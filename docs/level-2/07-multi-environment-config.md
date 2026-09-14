# 07 · Multi-Environment Configuration

!!! note "Not run against a live cluster"
    Manifests/commands below follow documented Kustomize/Helm behavior; not
    executed against a live cluster in this session.

## The problem: dev, staging, and prod aren't identical

Same application, different replica counts, resource limits, hostnames, and
feature flags per environment. Maintaining three full copies of every
manifest invites drift (someone fixes a bug in prod's copy and forgets
staging's). Two common approaches solve this without duplicating whole
files: **Kustomize** (overlay-based, built into `kubectl`) and **Helm
values files** (Module 06's mechanism, reused here per-environment).

## Kustomize: base + overlays

```text
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
resources:
  - ../../base
patches:
  - path: replica-patch.yaml
images:
  - name: myapp
    newTag: "1.5.0"
configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=warn
```

```yaml
# overlays/prod/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 10
```

```bash
kubectl kustomize overlays/prod | less     # preview the merged output
kubectl apply -k overlays/prod             # apply directly
kubectl apply -k overlays/dev
```

Nothing in `base/` is duplicated — each overlay references the base and
layers a small patch/config diff on top, so a fix to `base/deployment.yaml`
(a new probe, a new label) propagates to every environment automatically.

## Helm's equivalent: layered values files

```bash
helm install shop ./mychart -f values.yaml -f values-prod.yaml
```

```yaml
# values.yaml (shared defaults)
replicaCount: 2
resources:
  requests: { cpu: 100m, memory: 128Mi }
```

```yaml
# values-prod.yaml (overrides only what differs)
replicaCount: 10
resources:
  requests: { cpu: 500m, memory: 512Mi }
```

The mechanism is the same idea as Kustomize overlays — a common base plus a
small per-environment diff — just implemented via values-file merging
instead of strategic-merge patches on rendered YAML.

## ConfigMaps/Secrets that vary per environment

```yaml
configMapGenerator:
  - name: app-config
    literals:
      - API_URL=https://api-staging.example.com
      - FEATURE_NEW_CHECKOUT=false
```

Kustomize's `configMapGenerator` appends a content hash to the generated
ConfigMap's name (e.g. `app-config-8f92d6k4`) and automatically updates any
Deployment referencing it by name — this exists specifically so that
changing a ConfigMap's data *forces* a new Pod rollout (Kubernetes doesn't
otherwise restart Pods when a mounted ConfigMap's content changes; Module
08 covers this pattern's implications in more depth).

## Worked example: same base, three environments

```bash
kubectl apply -k overlays/dev
kubectl get deploy web -n dev -o jsonpath='{.spec.replicas}'
# 1

kubectl apply -k overlays/prod
kubectl get deploy web -n prod -o jsonpath='{.spec.replicas}'
# 10

kubectl get deploy web -n dev -o jsonpath='{.spec.template.spec.containers[0].image}'
# myapp:latest  (dev overlay didn't override the image tag)
kubectl get deploy web -n prod -o jsonpath='{.spec.template.spec.containers[0].image}'
# myapp:1.5.0
```

## How It Actually Works

- **`kubectl apply -k` and `kubectl kustomize` do pure client-side YAML
  transformation — no Kustomize CRD or controller runs in the cluster.**
  The kustomize engine loads the base's resources into memory, then applies
  each overlay's directives in a fixed order (generators first, then
  patches, then name/namespace/label transformers), producing final plain
  YAML that's submitted to the API server exactly like a hand-written
  manifest — this is why `kubectl kustomize` (no apply) is a safe,
  side-effect-free way to inspect the exact output before touching a
  cluster.
- **Patches are strategic merge patches by default, not naive
  overwrites.** A `patches` entry with `kind: Deployment` and a partial
  spec is merged field-by-field against the matching base resource using
  the same strategic-merge-patch rules the API server itself uses for
  `kubectl apply` — scalar fields (like `replicas`) are replaced, but list
  fields with defined merge keys (like `containers`, keyed by `name`) are
  merged element-by-element rather than replacing the whole list, so a
  patch touching only `resources` on one container doesn't accidentally
  drop the base's other containers.
- **`configMapGenerator`'s hash suffix is what forces a rollout — Kubernetes
  itself has no "watch this ConfigMap for changes" behavior for Pods.**
  kubelet only re-syncs a mounted ConfigMap's *file contents* on its
  periodic resync (eventually, but with no rollout, restart, or ordering
  guarantee); by instead generating a *new* ConfigMap object name derived
  from a content hash and rewriting the Deployment's
  `volumes[].configMap.name` (or `envFrom` reference) to point at it,
  Kustomize turns a content change into a Pod-template change, which the
  Deployment controller *does* know how to roll out safely.
- **Overlay composition is a directed acyclic reference graph, resolved
  once per invocation.** An overlay's `resources: [../../base]` is
  resolved recursively at build time (an overlay can itself be a base for
  another overlay), and Kustomize refuses cycles — there's no persistent
  "this overlay is linked to that base" relationship stored anywhere; every
  `kubectl apply -k` re-walks and re-renders the whole tree from scratch.

## Exercise

Create a `base/` with a Deployment (2 replicas) and Service, then `dev` and
`prod` overlays where `prod` patches `replicas` to 8 and sets a different
image tag via `images:`. Run `kubectl kustomize overlays/prod` and
`overlays/dev` and diff the two outputs to confirm only the intended fields
differ. Apply both to separate namespaces and verify with `kubectl get
deploy -A`.
