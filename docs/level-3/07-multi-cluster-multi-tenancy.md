# 07 · Multi-Cluster & Multi-Tenancy Concepts

!!! note "Not run against a live cluster"
    Manifests and output below follow documented namespace-isolation and
    multi-cluster tooling behavior; not executed against a live cluster in
    this session.

## Two different scaling problems, often confused

"Multi-tenancy" (multiple teams/customers sharing one cluster safely) and
"multi-cluster" (running several separate clusters and coordinating across
them) solve different problems and are frequently combined but not the same
decision. Multi-tenancy is about **isolation within** a cluster; multi-cluster
is about **blast-radius and locality across** clusters.

## Soft multi-tenancy: namespaces + RBAC + ResourceQuota + NetworkPolicy

The building blocks from earlier modules compose into "soft" (trusted
tenants, e.g. internal teams) multi-tenancy:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels: { tenant: team-a }
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "50"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-admin
  namespace: team-a
subjects:
  - kind: Group
    name: team-a-engineers
roleRef:
  kind: ClusterRole
  name: admin   # built-in ClusterRole, bound namespace-locally
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { tenant: team-a }
```

This is "soft" isolation: it stops accidental interference and enforces
resource fairness, but a cluster-admin-level compromise of the node/kernel
(a container escape) can still cross tenant boundaries, because all tenants
share the same kernel, kubelet, and control plane.

## Hard multi-tenancy: separate clusters (or virtual clusters) per tenant

For genuinely untrusted tenants (external customers, regulatory
separation), the stronger boundary is a **separate cluster per tenant**, or
a **virtual cluster** (e.g. vcluster) giving each tenant its own API server
and control-plane objects while still scheduling Pods onto a shared
underlying node pool — trading some resource-sharing efficiency for a much
stronger isolation guarantee than namespaces alone provide.

## Multi-cluster: why run more than one

- **Blast radius** — a control-plane outage, a bad CRD, or a
  misconfigured admission webhook affects one cluster, not everything.
- **Locality/latency** — clusters per region, close to users or data
  residency requirements.
- **Environment separation** — dev/staging/prod as physically separate
  clusters rather than namespaces in one, so a prod outage can never be
  caused by a staging experiment sharing the same API server.

## Multi-cluster service discovery (conceptual)

```yaml
# Cilium ClusterMesh / Submariner-style concept, not core Kubernetes API
apiVersion: networking.k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: api
  namespace: prod
```

Multi-cluster Services are **not** part of core Kubernetes — they require
an add-on (Cilium ClusterMesh, Submariner, or a cloud-managed multi-cluster
mesh) that establishes cross-cluster networking and republishes a Service
from one cluster's namespace into another's, typically under a
`<service>.<namespace>.svc.clusterset.local`-style name per the (still
evolving) Multi-Cluster Services API.

```bash
kubectl --context cluster-us get svc api -n prod
kubectl --context cluster-eu get svc api -n prod
# with ClusterMesh configured, a Pod in cluster-eu can resolve/reach
# the "api" Service running in cluster-us transparently
```

## Fleet management with `kubectl` contexts

```bash
kubectl config get-contexts
# CURRENT   NAME          CLUSTER    NAMESPACE
# *         cluster-us    us-east    prod
#           cluster-eu    eu-west    prod

kubectl --context cluster-eu apply -f deployment.yaml
kubectl config use-context cluster-us
```

At small scale, switching `--context` per cluster is workable; at real
fleet scale, GitOps tooling (Level 4, Module 02) applying the same
manifests to N clusters from one source of truth is the standard approach
rather than manual per-cluster `kubectl apply`.

## Worked example: quota isolation catches a runaway tenant

```bash
kubectl apply -f team-a-quota.yaml

kubectl run bulk --image=busybox -n team-a --replicas=100 2>&1 | tail -3
# Error from server (Forbidden): exceeded quota: team-a-quota,
# requested: pods=100, used: pods=12, limited: pods=50

kubectl get resourcequota team-a-quota -n team-a
# pods: 12/50, requests.cpu: 4/10
```

Team A's runaway deployment is rejected at admission time, before it ever
consumes cluster-wide capacity that other tenants (team-b, team-c) depend
on — the isolation held even though all tenants share the same nodes.

## How It Actually Works

- **Namespace-based isolation is entirely a control-plane construct — the
  underlying kernel/node resources are still fully shared.** A Pod in
  `team-a` and a Pod in `team-b` can, in the absence of NetworkPolicy and
  with weak Pod Security Standards (Level 4, Module 04), still see each
  other's traffic and, with certain kernel vulnerabilities or overly
  permissive `securityContext`, escape their container to the shared
  node — this is the precise technical reason "soft" multi-tenancy is
  described as trust-based: every enforcement layer (RBAC, quota, network
  policy) is cooperative software running with shared underlying kernel
  privilege, not a hardware/hypervisor-level boundary.
- **ResourceQuota enforcement happens as an admission plugin at object
  creation time, tracked against a live running total — not a periodic
  audit.** The `ResourceQuota` admission controller intercepts every
  Pod-creating request in a quota-bound namespace, sums that Pod's
  requested resources against the quota object's currently-tracked usage
  (updated transactionally as objects are created/deleted), and rejects
  the request outright if it would exceed `hard` limits — this is why
  quota violations surface as an immediate `403 Forbidden` on `kubectl
  apply`/`create`, not as a later reconciliation failure.
- **Virtual clusters (vcluster-style hard multi-tenancy) work by running a
  second, nested API server + control plane as a workload inside the host
  cluster, syncing a subset of objects down to real host-cluster
  resources.** Tenants interact with what looks like their own full
  Kubernetes API (their own CRDs, RBAC, even a different Kubernetes
  version) but Pods they create are transparently synced by the vcluster's
  syncer component into real Pods in a single namespace of the *host*
  cluster — giving strong API-level isolation (a tenant literally cannot
  see other tenants' objects, because they're not in their API server's
  storage at all) while still sharing the host's actual compute.
- **Multi-cluster Service meshes rely on a shared, cross-cluster identity
  and routing layer bolted on top of, not replacing, each cluster's own
  independent control plane.** Cilium ClusterMesh, for instance,
  establishes direct pod-to-pod tunnels between clusters and synchronizes
  EndpointSlice-equivalent data across the cluster boundary via each
  cluster's etcd being watched by the mesh's own agents — each cluster's
  API server remains fully authoritative for its own objects; there is no
  single federated etcd or API server spanning clusters in this pattern.

## Exercise

Create two namespaces (`team-a`, `team-b`) each with a `ResourceQuota`
capping pods at 10 and a NetworkPolicy denying cross-namespace ingress by
default. Attempt to create 15 Pods in `team-a` and confirm the quota
rejects the extra 5 at admission time with a clear error, then confirm via
`kubectl exec` that a Pod in `team-a` cannot reach a Pod in `team-b` despite
both running on the same underlying nodes.
