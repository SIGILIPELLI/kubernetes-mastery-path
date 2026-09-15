---
description: "Observability: Metrics & Logging — kubectl logs (Level 2, Module 09) works for one Pod you already know is broken. It doesn't survive Pod deletion…"
---

# 05 · Observability: Metrics & Logging

!!! note "Not run against a live cluster"
    Manifests and output below follow documented Prometheus/Fluent Bit
    deployment patterns; not executed against a live cluster in this
    session.

## Why "kubectl logs" isn't enough at scale

`kubectl logs` (Level 2, Module 09) works for one Pod you already know is
broken. It doesn't survive Pod deletion, doesn't aggregate across
replicas, and gives you no historical trend data. Production observability
needs three separate pillars: **metrics** (numeric time series — CPU,
request rate, error rate), **logs** (aggregated, searchable, retained past
a Pod's lifetime), and (not covered in depth here) **traces**.

## Metrics: the Prometheus pull model

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api
  labels: { app: api }
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  containers:
    - name: api
      image: myapp:1.4
      ports: [{ containerPort: 8080 }]
```

```yaml
# prometheus.yaml scrape_config (conceptual)
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
```

Prometheus **pulls** — it discovers scrape targets via the Kubernetes API
(watching Pods/Services/Endpoints, no separate service registry needed) and
periodically hits each target's `/metrics` HTTP endpoint. The application
must expose metrics in Prometheus's text exposition format itself (a
client library like `prom-client`/`client_golang` handles this); Kubernetes
core components (kubelet, API server, scheduler) already expose `/metrics`
this way natively.

```bash
kubectl apply -f prometheus-deployment.yaml -n monitoring
kubectl port-forward svc/prometheus 9090 -n monitoring
# open http://localhost:9090, query: rate(http_requests_total{app="api"}[5m])
```

## `kube-state-metrics`: cluster object state as metrics

`metrics-server` (Module 04) only exposes current CPU/memory. **
kube-state-metrics** is a separate component that watches the API server
and turns *object state* (Deployment replica counts, Pod phase, PVC
binding status) into Prometheus metrics:

```text
kube_deployment_status_replicas_available{deployment="web"} 3
kube_pod_status_phase{pod="web-abc",phase="Running"} 1
kube_pod_container_status_restarts_total{pod="api-xyz"} 5
```

This is how alerting rules like "a Deployment has had `< desired` available
replicas for 10 minutes" get built — from cluster *state*, not
resource usage.

## Logging: aggregate at the node, ship centrally

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector: { matchLabels: { app: fluent-bit } }
  template:
    metadata: { labels: { app: fluent-bit } }
    spec:
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:3.1
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
      volumes:
        - name: varlog
          hostPath: { path: /var/log }
```

A log-shipping DaemonSet (Level 3, Module 01) runs one instance per node,
reading every container's log files from `/var/log/pods` on that node
(kubelet's own log directory, the same one `kubectl logs` reads from) and
forwarding them to a central store (Elasticsearch/Loki/a cloud logging
service). This is the standard pattern precisely *because* it needs no
per-app code change — it works by tailing files kubelet already writes.

```bash
kubectl apply -f fluent-bit-daemonset.yaml -n logging
kubectl logs -l app=fluent-bit -n logging --tail=20
# forwarding logs for pod web-abc123_default_app-...
```

## Structured logging and correlation

```json
{"level":"info","msg":"order created","order_id":"o-123","trace_id":"9f2a...","pod":"api-7f9d8","ts":"2026-09-11T02:14:00Z"}
```

JSON-structured logs (rather than free text) let the central log store
index fields for querying (`order_id:"o-123"`) and, critically, let you
correlate a single request across services via a shared `trace_id` —
without structure, cross-service debugging degenerates into grepping
timestamps and hoping.

## Worked example: a Deployment's health, from both angles

```bash
# metrics angle
curl -s http://prometheus:9090/api/v1/query --data-urlencode \
  'query=kube_deployment_status_replicas_available{deployment="api"}'
# value: 2  (expected 3 -- one replica missing)

# logs angle, for WHY
kubectl logs -l app=api --since=10m | grep -i error
# "panic: connection refused to db:5432"
```

Metrics tell you *something* is wrong and roughly *when*; logs tell you
*why* — this division of labor is why production setups always run both,
not one instead of the other.

## How It Actually Works

- **Prometheus's Kubernetes service discovery re-lists/watches the API
  server continuously — scrape targets are never a static config file in
  practice.** The `kubernetes_sd_configs` mechanism watches Pod (or
  Endpoints/Service/Node) objects via the API server's watch protocol; a
  newly created Pod matching the relabel rules becomes a scrape target
  within one discovery refresh cycle with zero manual registration —
  relabeling runs entirely client-side in Prometheus against metadata
  already present on the object (annotations, labels), not against
  anything the target itself controls at scrape time.
- **A DaemonSet log shipper reads container logs via the *same* on-disk
  file convention kubelet maintains for `kubectl logs`, giving both
  identical (and identically limited) visibility.** Container runtimes
  write stdout/stderr to `/var/log/containers/*.log` (symlinked from
  `/var/log/pods/<pod-uid>/<container>/`) in a JSON-lines format kubelet
  defines; both `kubectl logs` (via the kubelet API) and a DaemonSet-based
  shipper tail these same files — this is why logs from a Pod deleted
  *and* garbage-collected before the shipper caught up are lost by both
  methods equally; centralizing logs only helps once they're shipped
  *before* that GC happens.
- **`kube-state-metrics` derives its numbers purely by watching the API
  server's object cache — it never touches nodes, containers, or cgroups
  at all.** This is the key distinction from `metrics-server`: it's
  reporting the *declared/observed state* of Kubernetes objects (what the
  Deployment controller wrote to `.status`), not live resource
  consumption — a Deployment showing `available: 2` when `replicas: 3`
  reflects a controller's own reconciliation state, sourced from Pod
  readiness the same way the EndpointSlice controller (Level 1, Module 07)
  computes membership.
- **Prometheus's pull model creates an inherent trade-off DaemonSet-based
  logging doesn't share: ephemeral, fast-lived Pods can finish and be
  garbage-collected between scrape intervals, producing zero metrics
  data.** A batch Job's Pod that starts and exits in 3 seconds may never be
  scraped if Prometheus's interval is 15s — this is exactly why short-lived
  workloads commonly use a **push gateway** pattern instead (the job
  pushes its final metric values to an intermediary Prometheus can scrape
  at its own pace) rather than being scraped directly.

## 🔀 Related lessons on other tracks

- [Server Ops — 09 · Observability at Scale (metrics, logs, traces)](https://sigilipelli.github.io/server-ops-mastery-path/level-3/09-observability-at-scale/)

## Exercise

Deploy `kube-state-metrics` and a minimal Prometheus scraping it, then
query `kube_deployment_status_replicas_available` for a Deployment you
intentionally scale past its available capacity (e.g. by setting an
unsatisfiable resource request on a new replica) to see the gap show up as
a metric. Separately, deploy a log-shipping DaemonSet and confirm via its
own logs that it picked up a test Pod's log lines that `kubectl logs`
against the (now-deleted) Pod itself would no longer be able to show.
