# 10 · Capstone Project

!!! note "Not run against a live cluster"
    This capstone's manifests and commands compose patterns from earlier
    Level 4 modules; not executed end-to-end against a live cluster in this
    session. Each referenced technique was already worked through
    individually in its own module.

## The brief

Design and document a production-grade cluster architecture for a
fictional service, **"orders-api"**, that combines every Level 4 topic
into one coherent system: GitOps-driven deployment, security hardening,
supply-chain verification, multi-AZ production topology, disaster
recovery, cost awareness, and a self-service platform surface. This
module doesn't introduce new mechanisms — it's the integration exercise
that shows how Modules 01-09 fit together as one operating system for a
real workload, not nine separate tricks.

## 1. Cluster topology (Module 06)

```text
Control plane: 3 nodes, 1 per AZ (us-east-1a/b/c), etcd co-located
General worker pool: 6-24 nodes, autoscaled, spread across all 3 AZs
Batch/spot pool: 0-10 nodes, tainted, for interruption-tolerant jobs only
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: orders
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels: {app: orders-api}
```

## 2. Namespace policy: Pod Security + RBAC (Module 04)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: orders
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: orders-api-deployer
  namespace: orders
subjects:
  - kind: Group
    name: team-orders
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

`team-orders` gets `edit` scoped to the `orders` namespace only — no
cluster-wide access, and every Pod they deploy must satisfy `restricted`.

## 3. Supply chain: signed, scanned images only (Module 05)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-orders-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        resources:
          kinds: ["Pod"]
          namespaces: ["orders"]
      verifyImages:
        - imageReferences: ["registry.example.com/orders-api*"]
          attestors:
            - entries: [{keyless: {issuer: "https://token.actions.githubusercontent.com"}}]
```

CI pipeline (outside the cluster): `trivy image --exit-code 1
--severity CRITICAL,HIGH` gates the build; `cosign sign` (keyless)
signs the passing image; the Kyverno policy above then refuses to admit
any Pod in `orders` referencing an unsigned image, closing the loop
end to end.

## 4. Deployment mechanism: GitOps (Module 02)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: orders-api
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/example/orders-api-manifests.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: orders
  syncPolicy:
    automated: {prune: true, selfHeal: true}
```

No one runs `kubectl apply` against production directly — the promotion
path is a Git commit to `overlays/production`, reviewed via PR, applied by
ArgoCD's reconcile loop, and any manual drift is auto-reverted.

## 5. Upgrades and maintenance (Module 03)

```bash
# rolling one node at a time, PDB-protected:
kubectl apply -f orders-api-pdb.yaml   # minAvailable: 4 (of 6 replicas)
for node in $(kubectl get nodes -l node-pool=general -o name); do
  kubectl cordon "$node"
  kubectl drain "$node" --ignore-daemonsets --delete-emptydir-data --timeout=300s
  # OS/kubelet patch happens here
  kubectl uncordon "$node"
  kubectl wait --for=condition=Ready "$node" --timeout=120s
done
```

## 6. Disaster recovery (Module 07)

```bash
# etcd snapshots every 6h, cross-region replicated (cluster-wide, covers all namespaces)
0 */6 * * * etcdctl snapshot save /backup/etcd-$(date +\%F-\%H\%M).db ...

# Velero, scoped and portable, covers orders-api's own PV data specifically:
velero schedule create orders-nightly \
  --schedule="0 3 * * *" \
  --include-namespaces orders \
  --snapshot-volumes
```

Quarterly DR drill: restore the `orders` namespace from the latest Velero
backup into a scratch cluster, confirm `orders-api` becomes healthy, and
record the actual time-to-restore against the target RTO.

## 7. Cost discipline (Module 08)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: orders-api-vpa
  namespace: orders
spec:
  targetRef: {apiVersion: apps/v1, kind: Deployment, name: orders-api}
  updatePolicy: {updateMode: "Off"}
```

`requests` on `orders-api` are set from VPA's Target recommendation
(reviewed by a human, not auto-applied), which lets Cluster Autoscaler
bin-pack the general pool tighter and scale it down — the mechanism from
Module 08 applied to this specific workload rather than described in the
abstract.

## 8. Developer self-service (Module 09)

```yaml
apiVersion: platform.example.com/v1alpha1
kind: DatabaseClaim
metadata:
  name: orders-db
  namespace: orders
spec:
  size: medium
  engine: postgres
```

The `team-orders` engineers provision `orders-db` through this claim, not
by filing an infra ticket or writing Terraform directly — the platform
team's `Composition` handles the real cloud resource underneath.

## How It Actually Works

- **These nine mechanisms compose because each one operates at a
  different point in the object lifecycle, and Kubernetes' control-loop
  model lets independent controllers layer without coordinating directly.**
  Pod Security Admission and the Kyverno image-verification policy both
  run at admission time (before persistence to etcd); ArgoCD's
  reconciler and Cluster Autoscaler both run as ongoing watch loops after
  persistence; etcd/Velero backups run on independent timers unaware of
  either. None of these controllers call each other — they each watch
  the API server's shared state and react, which is the same pattern
  explored per-module now operating concurrently on one real workload.
- **The actual failure mode this capstone architecture defends against is
  correlated, not independent, failure.** A single misconfigured `kubectl
  apply` to production is caught by GitOps (no direct apply path exists);
  an unsigned/vulnerable image is caught by admission-time verification
  even if CI is bypassed by a compromised credential; a full AZ loss is
  survived by topology spread and multi-AZ etcd; a full cluster loss is
  survived by Velero's cross-cluster restore. Each control only closes
  one specific gap — the production-readiness claim rests on the
  combination covering the union of realistic failure modes, not on any
  single control being perfect.
- **Verifying this design without a live cluster still has real value,
  and real limits.** Each manifest above is internally consistent with
  documented API behavior (field names, required RBAC verbs, admission
  ordering) and can be checked with `kubectl apply --dry-run=server` or
  `kubeconform` against the real API schema without touching production —
  but dry-run validation cannot verify runtime behavior (whether the VPA
  recommendation is actually well-sized, whether the DR restore actually
  completes within RTO, whether the Kyverno policy's `keyless` issuer
  matches the real CI identity) — those require the exercise below.

## Exercise

Build this capstone for real on a local kind cluster (scaled down: single
control-plane node is fine, skip real cloud provisioning). At minimum:
create the `orders` namespace with `restricted` Pod Security enforcement,
install ArgoCD and point it at a Git repo with a simple Deployment plus a
`PodDisruptionBudget`, install Kyverno with an image-verification policy
against a test signed/unsigned image pair, and take an etcd snapshot
before and after a deliberate `kubectl delete namespace orders` to confirm
your restore procedure actually recovers it. Write up which of the nine
controls you implemented, which you only sketched, and what broke that the
YAML alone didn't predict.
