---
description: "Resource Requests, Limits & Scheduling — Without resources set, a container can consume as much CPU/memory as the node has free, and the scheduler has no…"
---

# 03 · Resource Requests, Limits & Scheduling

!!! note "Not run against a live cluster"
    Manifests and output below follow documented scheduler/kubelet behavior;
    not executed against a live cluster in this session.

## Why declare resources at all

Without `resources` set, a container can consume as much CPU/memory as the
node has free, and the scheduler has no information to decide which node has
"room" for a new Pod — it would just guess. **Requests** tell the scheduler
what a container needs to be placed sensibly; **limits** cap what it's
allowed to actually use once running.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "250m"        # 0.25 of a vCPU core
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
```

CPU is measured in cores (`1` = one full core) or millicores (`250m` =
0.25 core). Memory is measured in bytes, usually written as `Mi`/`Gi`
(mebibytes/gibibytes, base-1024) rather than `M`/`G` (base-1000).

```bash
kubectl apply -f web.yaml
kubectl describe node <node-name>
# Allocated resources:
#   Resource   Requests   Limits
#   cpu        250m (12%) 500m (25%)
#   memory     128Mi (6%) 256Mi (12%)
```

## How the scheduler uses requests

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory
```

The scheduler's `NodeResourcesFit` predicate filters out any node whose
**sum of already-scheduled Pods' requests** plus this Pod's requests would
exceed the node's allocatable capacity — it never looks at *actual* live
usage for this decision, only requests. A Pod with no requests set is
treated as requesting effectively nothing, which is exactly the "unbounded
guess" problem requests solve.

```bash
kubectl describe pod web
# Events:
#   Warning  FailedScheduling  0/3 nodes are available: 3 Insufficient cpu.
```

## Limits and OOMKilled

CPU limits are enforced by *throttling* (the container is allowed to burst
briefly but gets CPU time capped over each scheduling period) — it never
gets killed for exceeding a CPU limit. Memory limits are enforced by
*killing* — a container that exceeds its memory limit is terminated
immediately by the kernel's OOM killer.

```bash
kubectl get pod web
# NAME   READY   STATUS      RESTARTS   AGE
# web    0/1     OOMKilled   1          2m

kubectl describe pod web
# Last State: Terminated, Reason: OOMKilled, Exit Code: 137
```

Exit code 137 = 128 + 9 (`SIGKILL`) — a strong signal it was the kernel OOM
killer, not the application exiting on its own.

## QoS classes

Kubernetes derives a **Quality of Service class** per Pod purely from how
requests/limits are set, with no separate field to configure it:

| Class | Condition | Behavior under node pressure |
|---|---|---|
| `Guaranteed` | Every container sets `requests == limits` for both CPU and memory | Evicted last |
| `Burstable` | At least one container sets a request or limit, but not equal for all | Evicted after BestEffort |
| `BestEffort` | No requests or limits set on any container | Evicted first |

```bash
kubectl get pod web -o jsonpath='{.status.qosClass}'
# Burstable
```

## LimitRange and ResourceQuota

A **LimitRange** sets defaults/bounds per container within a namespace so
developers can't forget resources entirely:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: team-a
spec:
  limits:
    - default:
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      type: Container
```

A **ResourceQuota** caps the *total* requests/limits across an entire
namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

## Worked example: filling a small cluster

```bash
kubectl apply -f limitrange.yaml -n team-a
kubectl apply -f quota.yaml -n team-a

# a Pod requesting 200m/300Mi with no explicit resources gets the LimitRange defaults applied
kubectl run test --image=nginx -n team-a
kubectl get pod test -n team-a -o jsonpath='{.spec.containers[0].resources}'
# {"limits":{"cpu":"500m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}

kubectl describe quota team-a-quota -n team-a
# Used   requests.cpu: 100m   requests.memory: 128Mi  ...
```

## How It Actually Works

- **Requests are consumed as scheduler-side bookkeeping, not as any kernel
  guarantee by themselves.** The scheduler keeps an in-memory
  "already-allocated" tally per node built from the requests of Pods bound
  there; `NodeResourcesFit` is a filter plugin run per candidate node during
  scheduling, and once a Pod is bound, the number is never re-checked
  against real usage — this is why a node can look "full" by requests while
  its actual CPU sits idle, or conversely why nodes can still get
  CPU-pressured even with headroom on requests (limits/actual usage are
  independent of the scheduling decision).
- **CPU limits become a cgroup CFS quota; requests become a cgroup CFS
  share.** kubelet configures each container's cgroup with `cpu.cfs_quota_us`
  derived from the limit (throttling once that many microseconds of CPU
  time are used within each `cpu.cfs_period_us`, typically 100ms) and
  `cpu.shares` derived from the request (a *relative* weight used only when
  multiple containers compete for the same idle CPU — it does no throttling
  by itself).
- **Memory limits become a cgroup hard cap enforced by the kernel, not
  kubelet.** kubelet sets `memory.limit_in_bytes` (cgroup v1) or
  `memory.max` (cgroup v2) on the container's cgroup; when the container's
  resident memory hits that ceiling, the *kernel's* cgroup OOM killer sends
  `SIGKILL` directly — kubelet only observes this after the fact via the
  container runtime and reports the `OOMKilled` reason, it doesn't do the
  killing itself.
- **QoS class also decides *eviction* order under node memory pressure via
  `oom_score_adj`.** kubelet sets a Linux `oom_score_adj` per container
  based on QoS class (very negative for Guaranteed, near-max for
  BestEffort) — when the *node itself* runs low on memory (not a per-container
  limit breach), the kernel-wide OOM killer picks victims by score, which is
  precisely why BestEffort Pods die first: their score biases the kernel to
  target them before Guaranteed ones.

## 🔀 Related lessons on other tracks

- [Docker — 03 · Resource Limits (CPU/Memory)](https://sigilipelli.github.io/docker-mastery-path/level-3/03-resource-limits/)

## Exercise

Deploy a Pod with `requests.memory: 64Mi` and `limits.memory: 128Mi`, then
run a workload inside it that allocates memory past 128Mi (e.g. `stress
--vm 1 --vm-bytes 200M`). Confirm via `kubectl describe pod` that it's
`OOMKilled` with exit code 137. Then set a LimitRange in a namespace and
confirm a Pod created without any `resources` block picks up the default
request/limit automatically.
