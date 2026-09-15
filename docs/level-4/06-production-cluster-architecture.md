---
description: "Designing Production-Grade Cluster Architecture — etcd is a Raft-based cluster and needs an odd member count so a majority (quorum) is always…"
---

# 06 · Designing Production-Grade Cluster Architecture

!!! note "Not run against a live cluster"
    Topology and manifests below follow documented multi-AZ / control-plane
    design guidance; not executed against a live cluster in this session.

## The shape of a production cluster

A production Kubernetes deployment is a set of deliberate answers to
questions the earlier levels mostly sidestepped for simplicity:

- How many control-plane nodes, and across how many availability zones?
- How is etcd deployed and how is it backed up?
- How are worker nodes pooled (general-purpose vs. specialized) and
  spread across zones?
- What enforces security, cost, and reliability policy cluster-wide
  rather than per-team?

## Control plane: odd-numbered, multi-AZ etcd

```text
AZ-a: control-plane-1 (etcd member 1)
AZ-b: control-plane-2 (etcd member 2)
AZ-c: control-plane-3 (etcd member 3)
```

etcd is a Raft-based cluster and needs an **odd** member count so a
majority (quorum) is always mathematically well-defined — 3 members
tolerate 1 failure, 5 tolerate 2. Spreading the 3 control-plane nodes
across 3 AZs means a single AZ outage still leaves 2 members, a quorum, so
the cluster keeps writing.

```bash
kubectl get componentstatuses   # deprecated but illustrative on older clusters
etcdctl --endpoints=https://10.0.1.5:2379,https://10.0.2.5:2379,https://10.0.3.5:2379 \
  endpoint status --write-out=table
# +----------------+------------------+---------+
# |    ENDPOINT    |        ID        | IS LEADER |
# | 10.0.1.5:2379  | 8211f1d0f64f3269 | true      |
# | 10.0.2.5:2379  | 91bc3c398fb3c146 | false     |
```

## Worker node pools by workload shape

```yaml
apiVersion: v1
kind: Node
metadata:
  name: worker-gpu-1
  labels:
    node-pool: gpu
    topology.kubernetes.io/zone: us-east-1a
spec:
  taints:
    - key: nvidia.com/gpu
      value: "true"
      effect: NoSchedule
```

```yaml
# workloads that need the GPU pool must explicitly tolerate the taint:
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        node-pool: gpu
```

Taints on specialized pools (GPU, high-memory, spot/preemptible) plus
matching tolerations on the Pods that need them is the standard pattern —
it prevents ordinary workloads from accidentally landing on expensive or
unstable specialized capacity, while still letting the scheduler place
everything else freely across the general pool.

## Spreading workloads across zones for real HA

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api
```

Without `topologySpreadConstraints`, the scheduler's default behavior can
still legally place 5 of 6 replicas in one zone — nothing prevents it.
`maxSkew: 1` with `DoNotSchedule` forces at most a 1-replica imbalance
between the fullest and emptiest zone, which is what actually survives a
zone outage without an availability cliff.

## Ingress and load-balancing at the edge

```text
Internet -> Cloud L4 LoadBalancer (per-zone, health-checked)
         -> Ingress Controller Pods (spread across zones via topologySpreadConstraints)
         -> Service (ClusterIP) -> backend Pods
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["api.example.com"]
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

## Worked example: sizing a reference cluster

```text
Control plane: 3 x m5.xlarge, one per AZ (us-east-1a/b/c)
etcd: co-located on control-plane nodes, dedicated SSD-backed volume, backed
      up via etcdctl snapshot every 15 min (see Module 07)
General worker pool: 6-30 x m5.2xlarge, cluster-autoscaler managed,
      topologySpreadConstraints across the 3 AZs
GPU pool: 0-4 x g5.xlarge, tainted, scaled from 0 via a separate
      autoscaler node group, only used by ML workloads that tolerate the taint
Ingress: 3 nginx-ingress-controller replicas, one per AZ, behind a
      cloud L4 load balancer with cross-zone health checks
```

This shape (odd multi-AZ control plane, tainted specialized pools,
zone-spread ingress and workloads, autoscaled general pool) is the
recurring reference architecture that the rest of Level 4's modules
(upgrades, DR, cost, platform engineering) all assume as the starting
point.

## How It Actually Works

- **etcd's quorum requirement comes directly from the Raft consensus
  algorithm's majority-vote rule, not from a Kubernetes-specific
  design choice.** A Raft write is only committed once a majority of
  members have durably persisted it to their local WAL (write-ahead log);
  with 3 members, 2 constitutes a majority, so 1 member can be lost with
  zero write unavailability, while with an even count like 4, losing 2
  members still leaves exactly 2 — not a majority — meaning an even-sized
  cluster gains no extra fault tolerance over the next-lower odd size while
  paying for an extra member's write latency (every write waits on a
  majority ack across all members).
- **Zone-aware scheduling relies entirely on the `topology.kubernetes.io/zone`
  node label being populated correctly by the cloud provider's
  cloud-controller-manager, not by kubelet itself.** kubelet reports basic
  node info at registration, but zone/region labels are set by the
  cloud-controller-manager querying the cloud API's instance metadata for
  each node — on a bare-metal or misconfigured cloud-integration cluster
  this label is simply absent, silently making every
  `topologySpreadConstraints`/zone-aware feature a no-op rather than an
  error.
- **`topologySpreadConstraints` is evaluated by a scheduler plugin during
  the Filter/Score phases of the scheduling framework, computed fresh per
  Pod at schedule time — it is not a standing invariant the control plane
  continuously re-enforces.** The plugin counts existing matching Pods per
  topology domain at the moment a new Pod is being scheduled and
  rejects placements that would exceed `maxSkew`; if zone distribution
  later becomes skewed through independent events (a zone's nodes all
  failing and Pods rescheduling elsewhere), nothing proactively rebalances
  already-running Pods back — the constraint only governs future placement
  decisions.
- **A cloud L4 load balancer's per-zone health checks are what actually
  provide "zone outage tolerance" for the ingress path, not Kubernetes.**
  The cloud LB independently health-checks each ingress-controller Pod's
  node/target; if an entire zone's targets stop responding, the LB simply
  stops routing traffic there at the cloud infrastructure layer — this
  happens outside etcd, the scheduler, and any Kubernetes control loop
  entirely, which is why ingress-controller replica placement (spread
  across zones) matters even though Kubernetes itself has no concept of
  "zone health."

## Exercise

Sketch (in YAML) a control-plane and worker-node topology for a
hypothetical 3-AZ production cluster serving a workload that needs 99.9%
availability. Include: control-plane node count and AZ placement, at least
one worker pool with a taint and a matching toleration, and a Deployment
with `topologySpreadConstraints` set to tolerate the loss of one full AZ
without dropping below 2/3 of its replica count.
