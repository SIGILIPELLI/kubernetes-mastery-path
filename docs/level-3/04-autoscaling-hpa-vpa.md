---
description: "Autoscaling (HPA & VPA Concepts) — Scaling a workload means either running more copies (horizontal) or giving each copy more CPU/memory (vertical).…"
---

# 04 · Autoscaling (HPA & VPA Concepts)

!!! note "Not run against a live cluster"
    Manifests and output below follow documented autoscaler behavior; not
    executed against a live cluster in this session.

## Two different axes of "more resources"

Scaling a workload means either running **more copies** (horizontal) or
giving each copy **more CPU/memory** (vertical). Kubernetes has a built-in
controller for each, and they solve different problems — the
**HorizontalPodAutoscaler (HPA)** changes `replicas`; the
**VerticalPodAutoscaler (VPA)**, a separate add-on, changes
`resources.requests/limits`.

## HPA: scaling replica count off live metrics

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

This requires the **metrics-server** add-on (or a custom metrics adapter)
to be installed — the HPA controller reads from the `metrics.k8s.io` API,
which metrics-server serves by scraping kubelet's resource stats.

```bash
kubectl apply -f web-hpa.yaml
kubectl get hpa web-hpa
# NAME      REFERENCE      TARGETS         MINPODS   MAXPODS   REPLICAS
# web-hpa   Deployment/web 45%/70%         2         10        3

kubectl top pods -l app=web    # requires metrics-server
```

`averageUtilization: 70` means "keep the average CPU usage across all
replicas at 70% of each Pod's **CPU request**" — this is exactly why
setting an accurate `requests.cpu` (Level 2, Module 03) is a prerequisite
for HPA to make sensible decisions at all; without a request, utilization
percentage has no denominator.

## Custom and external metrics

```yaml
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
    - type: External
      external:
        metric:
          name: sqs_queue_depth
          selector:
            matchLabels: { queue: orders }
        target:
          type: AverageValue
          averageValue: "30"
```

`type: Pods` reads a metric per-Pod (via a custom metrics adapter, e.g.
Prometheus Adapter); `type: External` reads a metric with no direct Pod
association at all (a queue depth, a metric from a managed cloud service) —
useful for scaling workers based on backlog rather than their own resource
usage.

## VPA: right-sizing requests/limits automatically

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
    updateMode: "Auto"   # or "Off" (recommendation only), "Initial"
```

```bash
kubectl describe vpa api-vpa
# Recommendation:
#   Container: api
#     Target: { cpu: 340m, memory: 410Mi }
#     Lower Bound: { cpu: 200m, memory: 300Mi }
#     Upper Bound: { cpu: 600m, memory: 550Mi }
```

`updateMode: "Auto"` doesn't patch a running Pod's resources in place
(Kubernetes doesn't support that) — it **evicts and recreates** Pods with
the recommended values applied, meaning VPA in Auto mode causes real,
disruptive restarts, which is why it's commonly run in `"Off"` mode purely
for recommendations that a human (or a CI pipeline) applies deliberately.

## HPA and VPA together: usually not on CPU/memory at once

Running HPA and VPA on the *same* metric (CPU) for the same workload is
explicitly unsupported — they'd fight each other (VPA raising requests
while HPA reacts to the resulting utilization change). The common safe
combination is VPA on requests/limits for right-sizing, HPA on a
*different* signal (custom app metric, or memory while VPA handles CPU).

## Worked example: load test drives a scale-out

```bash
kubectl apply -f web-hpa.yaml
kubectl run load --image=busybox:1.36 --restart=Never -it --rm -- \
  sh -c "while true; do wget -qO- http://web; done" &

kubectl get hpa web-hpa -w
# TARGETS goes 20%/70% -> 85%/70% -> HPA scales replicas 3 -> 5 -> 8
kubectl get deploy web
# REPLICAS: 8/8
```

## How It Actually Works

- **The HPA controller is a periodic reconciliation loop, not an
  event-driven reaction to metric spikes.** Every sync period (15s
  default), the `horizontal-pod-autoscaler` controller in
  kube-controller-manager queries the metrics API for current values,
  computes `desiredReplicas = ceil(currentReplicas * (currentMetricValue /
  desiredMetricValue))` for each metric, takes the max across all metrics
  (never scaling down because one metric is low if another says scale up),
  and PATCHes the target's `.spec.replicas` — the actual Pod creation is
  then, as always, the Deployment/ReplicaSet controllers' job, completely
  decoupled from the HPA's decision.
- **Stabilization windows and scaling policies exist because raw reactive
  scaling would thrash.** `behavior.scaleDown.stabilizationWindowSeconds`
  (default 300s) makes the controller look back over a window and pick the
  *highest* recommended replica count seen in that window before scaling
  down — this deliberately makes scale-down conservative and slow while
  scale-up (default stabilization 0s) can react immediately, because the
  cost of being briefly over-provisioned is much lower than the cost of
  being under-provisioned during a real spike.
- **`metrics-server` computes utilization from kubelet's cAdvisor-derived
  stats, sampled on an interval — not a live, continuous readout.**
  metrics-server scrapes every kubelet's `/stats/summary` endpoint roughly
  every 15-60s and aggregates to the `metrics.k8s.io` API; this
  end-to-end latency chain (cAdvisor sampling -> kubelet -> metrics-server
  scrape -> HPA sync) means an HPA reacting to a sudden spike is
  realistically tens of seconds to a couple of minutes behind real load,
  which is exactly why HPA is not a substitute for proper capacity buffer
  on latency-sensitive services.
- **VPA's "Auto" mode is fundamentally an eviction-and-recreate cycle
  driven by a separate admission webhook, not an in-place patch.** The VPA
  recommender computes target resources from historical usage (stored in
  its own aggregated histograms, not raw metrics-server data); the VPA
  updater evicts Pods whose current resources deviate too far from the
  recommendation, and a mutating admission webhook (Level 3, Module 08)
  intercepts the *replacement* Pod's creation to inject the new
  resources — meaning VPA can only ever change resources at Pod
  (re)creation time, never on a live, running container.

## Exercise

Deploy `web` with an HPA targeting 70% CPU utilization, `minReplicas: 2`,
`maxReplicas: 8`, and confirm `kubectl get hpa` shows current utilization
once metrics-server is scraping. Generate load against it (a simple busybox
loop hitting the Service) and watch `kubectl get hpa -w` to see
`desiredReplicas` climb; stop the load and observe the slower scale-down
governed by the stabilization window rather than an immediate drop.
