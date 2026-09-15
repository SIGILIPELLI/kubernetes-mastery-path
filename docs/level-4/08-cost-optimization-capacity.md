---
description: "Cost Optimization & Capacity Planning — Cloud bills for a Kubernetes cluster are driven almost entirely by requested, not used, resources — the scheduler…"
---

# 08 · Cost Optimization & Capacity Planning

!!! note "Not run against a live cluster"
    Commands and sizing numbers below follow documented autoscaler/VPA
    behavior; not executed against a live cluster in this session.

## Where Kubernetes cost actually goes

Cloud bills for a Kubernetes cluster are driven almost entirely by
**requested**, not used, resources — the scheduler places Pods based on
`resources.requests`, and nodes are provisioned (and billed) to satisfy
those requests whether or not the Pod ever uses what it asked for.
Over-requesting is the single largest source of Kubernetes waste in
practice.

```bash
kubectl top pod -n payments
# NAME          CPU(cores)   MEMORY(bytes)
# api-7f8b9     45m          210Mi

kubectl get pod api-7f8b9 -n payments -o jsonpath='{.spec.containers[0].resources}'
# {"requests":{"cpu":"500m","memory":"1Gi"},"limits":{"cpu":"1","memory":"2Gi"}}
```

Here the Pod requests 500m CPU / 1Gi memory but uses 45m / 210Mi — the
scheduler reserved 10x the CPU actually needed on some node, capacity that
sits idle and billed regardless.

## Right-sizing with the Vertical Pod Autoscaler

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Off"   # "Off" = recommendation-only, doesn't evict Pods
```

```bash
kubectl describe vpa api-vpa
# Recommendation:
#   Container: api
#     Target:   {cpu: 80m, memory: 256Mi}
#     Lower Bound: {cpu: 50m, memory: 200Mi}
#     Upper Bound: {cpu: 140m, memory: 340Mi}
```

`updateMode: "Off"` is the safe way to introduce VPA into an existing
workload — it only computes and reports a recommendation, letting a human
apply the new `requests` deliberately, rather than `"Auto"` mode's
behavior of evicting and recreating Pods to apply new sizing live (which
is disruptive for anything that isn't tolerant of restarts).

## Scaling node count with Cluster Autoscaler

```yaml
# node group annotation (cloud-specific, AWS example via a ASG tag)
k8s.io/cluster-autoscaler/enabled: "true"
k8s.io/cluster-autoscaler/cluster-name: "prod-cluster"
```

```bash
kubectl logs -n kube-system deployment/cluster-autoscaler | tail -5
# scale_up.go: Pod payments/api-8f9c2 is unschedulable, scaling up node group "general-pool"
# scale_up.go: node group general-pool scaled up from 6 to 7
```

Cluster Autoscaler reacts to **unschedulable Pods** (a Pod stuck Pending
because no node has room) and to **underutilized nodes** it can safely
drain and remove — it does not look at CPU/memory utilization percentages
directly the way a naive "scale when 80% busy" system would.

## Bin-packing and Pod density

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: api
          resources:
            requests:
              cpu: 100m      # tightened from 500m after VPA recommendation
              memory: 256Mi
            limits:
              memory: 512Mi   # no CPU limit: avoid CFS-throttling a bursty workload
```

Dropping over-requested `resources.requests` doesn't just save money
directly — it changes the scheduler's bin-packing math, letting more Pods
land on the same node, which is what actually lets Cluster Autoscaler
scale the node count down.

## Spot/preemptible capacity for interruption-tolerant workloads

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      tolerations:
        - key: cloud.google.com/gke-spot
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        cloud.google.com/gke-spot: "true"
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway   # tolerate imbalance rather than fail to schedule
          labelSelector:
            matchLabels:
              app: batch-worker
```

Spot/preemptible nodes are typically 60-90% cheaper but can be reclaimed
by the cloud provider with short notice (seconds to 2 minutes) — suitable
for stateless, retryable, horizontally-scaled workloads (batch jobs,
CI runners, stateless API replicas behind enough total capacity to absorb
a reclaim) but a poor fit for anything stateful or singleton.

## Worked example: a cost-reduction pass

```text
1. kubectl top pod across every namespace + 2 weeks of VPA "Off" recommendations
   -> identify Deployments requesting >3x their observed usage.
2. Tighten `resources.requests` on those Deployments to VPA's target
   recommendation (+ modest headroom, not the raw observed average).
3. Re-run Cluster Autoscaler; observe node count drop as bin-packing improves.
4. Move stateless, horizontally-scaled workloads (with >=3 replicas and no
   local state) onto a spot node pool with matching taints/tolerations.
5. Re-measure the cloud bill's compute line item after one full billing cycle.
```

## How It Actually Works

- **The scheduler's Filter phase only ever looks at `requests`, never
  `limits` or actual usage — this is the mechanical reason over-requesting
  wastes capacity.** The `NodeResourcesFit` scheduler plugin sums a
  candidate node's already-*requested* (not used) CPU/memory against the
  new Pod's requests and rejects nodes that can't fit the sum; a node can
  be otherwise fully idle and still be considered "full" if the Pods on it
  requested more than the node has, which is exactly the gap between
  billed capacity and used capacity.
- **`limits` are enforced by a completely different mechanism than
  `requests` — the kernel's CFS bandwidth controller for CPU, and the
  cgroup OOM killer for memory — which is why a CPU limit throttles
  (slows down) while a memory limit kills.** The kubelet configures each
  container's cgroup with a CFS quota/period pair derived from the CPU
  limit; exceeding it doesn't error, it just gets scheduled less CPU time
  by the kernel within each period (visible as `nr_throttled` in
  `/sys/fs/cgroup/.../cpu.stat`) — this is why an unset CPU limit on a
  bursty workload (as in the example above) avoids artificial throttling,
  while memory limits have no equivalent "slow down" option and instead
  trigger the kernel OOM killer, which is why memory limits should
  usually be set close to actual need and CPU limits are more often
  omitted deliberately.
- **Cluster Autoscaler's scale-down decision requires simulating whether
  a node's Pods can be rescheduled elsewhere, not just checking if the
  node looks idle.** Before removing a node, it checks PodDisruptionBudgets,
  local storage usage (Pods using `emptyDir` or local PVs block removal by
  default), and whether every Pod on that node could be placed on the
  remaining nodes given their current requests — a node hosting even one
  Pod that can't be rescheduled (a DaemonSet Pod aside, or a Pod pinned by
  `nodeSelector` to that specific node) is never removed regardless of how
  idle it otherwise looks.
- **VPA's recommendation engine is a statistical model (a
  decaying-histogram percentile estimator) over the actual resource-usage
  metrics history, running as a separate `recommender` component, not a
  Kubernetes control-loop that watches live metrics in real time.** It
  periodically queries the Metrics API (backed by metrics-server or a
  custom Prometheus adapter) for historical usage, feeds each container's
  history into a per-resource histogram with time-decayed weighting, and
  derives the Target/Lower/Upper Bound recommendations from percentiles of
  that histogram — a workload with a genuinely bursty, spiky usage pattern
  will see a wide Lower/Upper Bound band precisely because the histogram
  reflects that variance.

## 🔀 Related lessons on other tracks

- [Server Ops — 02 · Capacity Planning](https://sigilipelli.github.io/server-ops-mastery-path/level-4/02-capacity-planning/)
- [AI/ML — 08 · Cost Optimization & Efficient Inference](https://sigilipelli.github.io/ai-ml-mastery-path/level-4/08-efficient-inference/)
- [AWS — Cost Optimization at Scale](https://sigilipelli.github.io/aws-mastery-path/level-4/06-cost-optimization-at-scale/)

## Exercise

On a cluster with `metrics-server` installed, run `kubectl top pod`
across a namespace and compare actual usage against each Pod's
`resources.requests`. Install the Vertical Pod Autoscaler in `"Off"` mode
against one Deployment, wait for it to accumulate enough history to
produce a recommendation, and compute the CPU/memory percentage waste
between the current request and VPA's Target recommendation.
