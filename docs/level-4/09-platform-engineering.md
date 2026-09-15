---
description: "Platform Engineering on Kubernetes — Once an organization has more than a handful of teams deploying to Kubernetes, exposing raw kubectl/YAML to every…"
---

# 09 · Platform Engineering on Kubernetes

!!! note "Not run against a live cluster"
    Manifests and CLI output below follow documented Crossplane/Backstage
    behavior; not executed against a live cluster in this session.

## Why platform engineering exists

Once an organization has more than a handful of teams deploying to
Kubernetes, exposing raw `kubectl`/YAML to every developer creates two
opposing failure modes: either every team reinvents its own
Deployment/Ingress/monitoring boilerplate (inconsistent, insecure,
expensive to support), or a central platform team becomes a ticket queue
manually provisioning things for everyone (slow, doesn't scale with
headcount). **Platform engineering** is the practice of building a
self-service layer — a golden path — so developers get safe, consistent
infrastructure without either extreme.

## Golden-path abstraction: a Crossplane Composite Resource

Crossplane lets a platform team define a simplified custom API
(`XRD` — Composite Resource Definition) that developers use, which expands
into the full set of real Kubernetes/cloud resources underneath.

```yaml
# platform team defines the developer-facing API:
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: XDatabase
    plural: xdatabases
  claimNames:
    kind: DatabaseClaim
    plural: databaseclaims
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                size: {type: string, enum: ["small", "medium", "large"]}
                engine: {type: string, enum: ["postgres", "mysql"]}
```

```yaml
# a developer's entire interaction with cloud provisioning:
apiVersion: platform.example.com/v1alpha1
kind: DatabaseClaim
metadata:
  name: payments-db
  namespace: payments
spec:
  size: medium
  engine: postgres
```

```bash
kubectl apply -f payments-db-claim.yaml
kubectl get databaseclaim payments-db -n payments
# NAME          SYNCED   READY   CONNECTION-SECRET
# payments-db   True     True    payments-db-conn
```

Behind that one `DatabaseClaim`, the platform team's `Composition` (not
shown) provisions whatever the real cloud resources are (an RDS instance,
security groups, a connection Secret) — the developer never touches
Terraform, cloud IAM, or VPC configuration, and the platform team can
change the underlying implementation (swap RDS for CloudSQL, change
instance sizing defaults) without any developer-facing API change.

## Developer portal surface: Backstage software catalog

```yaml
# catalog-info.yaml, checked into each service's repo
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payments-api
  annotations:
    backstage.io/kubernetes-id: payments-api
spec:
  type: service
  lifecycle: production
  owner: team-payments
  dependsOn:
    - resource:default/payments-db
```

Backstage aggregates every service's `catalog-info.yaml` into a searchable
catalog with ownership, dependency graphs, and (via its Kubernetes plugin)
live cluster status — the goal is a developer can find "who owns this
service, what does it depend on, is it healthy right now" without needing
`kubectl` access or tribal knowledge of who to ask.

## Self-service scaffolding: a Backstage software template

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: new-microservice
spec:
  parameters:
    - title: Service details
      properties:
        name: {type: string}
        team: {type: string}
  steps:
    - id: fetch-base
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: "${{ parameters.name }}"
    - id: publish
      action: publish:github
      input:
        repoUrl: "github.com?repo=${{ parameters.name }}"
    - id: register
      action: catalog:register
      input:
        catalogInfoPath: /catalog-info.yaml
```

This is the "golden path" made concrete: a developer fills out a form,
gets a new repo with CI, a Dockerfile, base Kubernetes manifests, RBAC,
and observability wiring already correct by construction — the platform
team's opinions about "how services should be built" are encoded once, in
the template, instead of re-explained per team per onboarding.

## Worked example: request-to-running-service flow

```text
1. Developer runs "new-microservice" Backstage template -> new repo scaffolded,
   registered in the catalog, CI pipeline wired automatically.
2. Developer's first commit triggers CI: build, trivy scan, cosign sign
   (Module 05), push image.
3. GitOps controller (Module 02) picks up the new manifest in Git, deploys
   into a namespace already governed by a Pod Security Standard (Module 04)
   and network policy the platform team set as a default.
4. Developer needs a database: applies a `DatabaseClaim` (Crossplane) instead
   of filing a ticket or writing Terraform.
5. Backstage catalog shows the new service, its owner, its dependency on the
   claimed database, and live health from the cluster.
```

None of steps 2-5 require the platform team to be paged or to click
anything — every step after the initial template run is self-service,
governed by policy encoded in the platform rather than enforced by a human
gatekeeper per request.

## How It Actually Works

- **Crossplane's `Composition` mechanism is itself just a Kubernetes
  controller pattern layered on CRDs — "Composite Resources" (XRs) are
  real custom resources, and Crossplane's core controller reconciles them
  the same way any operator reconciles its CRD.** When a `DatabaseClaim`
  is created, Crossplane's claim controller creates a corresponding `XDatabase`
  composite resource; the Composition (a template mapping XR fields to a
  set of "managed resources," each itself a CRD representing one real
  cloud API object, e.g. `RDSInstance`) drives the actual provider
  controller (Crossplane's AWS/GCP provider, running its own reconcile
  loop against the cloud API) — the entire chain is composed of ordinary
  Kubernetes control loops, several layers deep, each unaware of the
  layers around it.
- **The claim/composite split exists specifically for namespace
  isolation and RBAC.** `DatabaseClaim` is namespaced (so a team's RBAC
  can be scoped to claims in their own namespace, the normal Kubernetes
  RBAC boundary) while the underlying `XDatabase` composite and the real
  cloud-provider managed resources are cluster-scoped — this lets the
  platform team keep the powerful, cluster-wide provisioning objects out
  of reach of individual developer RBAC roles while still letting those
  developers create and own claims freely within their namespace.
- **Backstage's catalog is a static-ish graph built from repository
  metadata plus periodic Kubernetes plugin polling — it is not itself
  a Kubernetes controller and has no admission-time enforcement power.**
  `catalog-info.yaml` files are discovered by Backstage's catalog
  processor (via configured integrations — GitHub search, a static
  location list) and re-ingested on a polling interval; the Kubernetes
  plugin separately queries the cluster's API server (scoped via a
  service account) for live status keyed by the
  `backstage.io/kubernetes-id` label/annotation. This means catalog
  freshness lags real cluster state by the poll interval, and a
  service running in the cluster without a matching `catalog-info.yaml`
  is simply invisible to the portal — the catalog reflects what's been
  registered, not what's actually deployed.
- **A scaffolder template's `publish` and `register` steps chain
  ordinary external API calls (GitHub's REST API, the Backstage catalog's
  own ingestion API) inside a Backstage-managed workflow engine — no
  Kubernetes object is created until the developer's first CI run
  actually deploys something.** This is why "self-service scaffolding"
  and "GitOps deployment" are separate systems that happen to compose:
  the template produces a Git repo and a catalog entry; only the
  subsequent CI/CD pipeline (wired into the scaffolded repo by the
  template) is what eventually causes any actual Kubernetes resource to
  exist.

## Exercise

Design (in YAML, no live cluster required) a Crossplane `CompositeResourceDefinition`
and a matching `DatabaseClaim` example for a "cache" abstraction (e.g. a
managed Redis instance) that exposes only `size` and lets the platform
team's `Composition` decide the actual cloud resource, instance class, and
network configuration. Then sketch a `catalog-info.yaml` for a service
that depends on that cache resource, and describe what a developer would
see in a Backstage catalog page for it.
