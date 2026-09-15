---
description: "GitOps Patterns (ArgoCD/Flux Concepts) — GitOps flips the normal deployment flow. Instead of a CI pipeline running kubectl apply against a cluster (a push…"
---

# 02 · GitOps Patterns (ArgoCD/Flux Concepts)

!!! note "Not run against a live cluster"
    Manifests and CLI output below follow documented ArgoCD/Flux behavior;
    not executed against a live cluster in this session.

## The core idea: Git as the single source of truth

GitOps flips the normal deployment flow. Instead of a CI pipeline running
`kubectl apply` against a cluster (a **push** model), a controller running
*inside* the cluster continuously watches a Git repository and reconciles
the live cluster state to match what's declared there (a **pull** model).

```text
Push model:  CI pipeline --kubectl apply--> Cluster
Pull model:  Git repo <--watches-- GitOps controller (in-cluster) --reconciles--> Cluster
```

The distinction matters operationally: in the pull model, nothing outside
the cluster needs write credentials to it — the controller already runs
with a service account inside the cluster, and CI only needs push access to
Git. Drift (someone running `kubectl edit` by hand) gets corrected
automatically on the next reconcile loop instead of silently persisting.

## ArgoCD: the Application CRD

ArgoCD models each deployed thing as an `Application` custom resource that
points at a Git path and a destination cluster/namespace:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/example/payments-api-manifests.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: payments
  syncPolicy:
    automated:
      prune: true       # delete resources removed from Git
      selfHeal: true     # revert manual cluster edits back to Git state
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f payments-api-application.yaml -n argocd
argocd app get payments-api
# Health Status:  Healthy
# Sync Status:    Synced
argocd app sync payments-api   # manual sync if automated is off
```

`prune: true` and `selfHeal: true` together are what make ArgoCD
enforce Git as the *only* legitimate source of change — without them,
ArgoCD only reports drift ("OutOfSync") rather than correcting it.

## Flux: reconciliation via Kustomization/HelmRelease

Flux (v2, "Flux CD") is built as a set of composable CRD-driven
controllers rather than one Application object. A `GitRepository` source
defines what to watch; a `Kustomization` defines what to apply from it:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: payments-api
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/example/payments-api-manifests.git
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: payments-api
  namespace: flux-system
spec:
  interval: 5m
  path: "./overlays/production"
  prune: true
  sourceRef:
    kind: GitRepository
    name: payments-api
```

```bash
flux get sources git
flux get kustomizations
# NAME           READY   MESSAGE
# payments-api   True    Applied revision: main@sha1:8f2a1c
```

The `GitRepository`/`Kustomization` split lets one Git source feed
multiple Kustomizations (e.g. staging and production overlays pointed at
different paths of the same repo) — a composition ArgoCD achieves with
separate `Application` objects instead.

## Worked example: promoting a change through environments

A typical GitOps repo layout separates environment overlays (Kustomize)
so promotion is a Git operation, not a pipeline deploy step:

```text
manifests/
  base/
    deployment.yaml
    service.yaml
  overlays/
    staging/kustomization.yaml     (replicas: 2, image tag: staging-latest)
    production/kustomization.yaml  (replicas: 6, image tag: v1.4.2)
```

```bash
# CI, on merge to main, updates only the image tag via a tool like `kustomize edit`:
cd manifests/overlays/production
kustomize edit set image payments-api=registry.example.com/payments-api:v1.4.3
git commit -am "bump payments-api to v1.4.3" && git push
# ArgoCD/Flux notices the new commit within its poll interval (or via webhook)
# and reconciles the live Deployment's image field to match — no kubectl needed.
argocd app diff payments-api   # confirms what changed before/if manual sync
```

Promoting to production is then just merging the same commit (or a PR)
into the production overlay's branch/path — the full deployment history is
the Git log, and rollback is `git revert` plus a reconcile.

## How It Actually Works

- **The reconciliation loop is a standard Kubernetes controller pattern,
  not something GitOps-specific.** ArgoCD's `application-controller` and
  Flux's `kustomize-controller` are both just controllers watching their
  own CRDs (`Application`, `Kustomization`) the same way the built-in
  `kube-controller-manager` watches Deployments — a control loop that
  computes desired state (rendered manifests from Git) vs. observed state
  (live cluster objects via the API server) and calls `Apply`/`Patch` to
  close the gap, repeating on a timer (`interval`) and on webhook-triggered
  wakeups.
- **Drift detection relies on comparing live object state to a
  re-rendered manifest, field by field, not on a checksum of the whole
  object.** Both tools compute a structured diff so that fields Kubernetes
  itself sets (`status`, `resourceVersion`, defaulted fields added by
  admission webhooks) are excluded — this is why "OutOfSync" specifically
  means a field *you* declared in Git differs from the live cluster, not
  that any byte of the object changed.
- **`selfHeal`/pruning re-triggers on watch events, not only on the poll
  timer.** ArgoCD additionally watches the destination cluster's resources
  (via its own informer against the target API server) so a manual
  `kubectl edit` is detected and reverted almost immediately, independent
  of the Git-repo poll interval — this is the mechanism that actually
  enforces "no manual changes survive," not just periodic re-sync.
- **Multi-tenancy in ArgoCD is enforced via the `AppProject` CRD, which
  is a policy object the application-controller consults before every
  sync, not a Kubernetes RBAC construct.** `AppProject` restricts which
  Git repos, destination clusters/namespaces, and resource kinds an
  `Application` in that project may reference; this lets one shared ArgoCD
  instance safely serve many teams without giving each team cluster-wide
  RBAC — the enforcement point is inside ArgoCD's own reconcile logic, in
  addition to (not instead of) the service account's real Kubernetes RBAC.

## Exercise

Install ArgoCD (or Flux) into a local kind/minikube cluster, point it at a
public Git repo containing a simple Deployment manifest, and confirm it
syncs. Then run `kubectl scale` or `kubectl edit` directly against the
resulting Deployment to introduce drift, and observe (via `argocd app get`
or `flux get kustomizations`) how quickly and by what mechanism the
controller reverts your manual change back to the Git-declared state.
