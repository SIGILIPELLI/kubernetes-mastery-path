---
description: "Health Checks (Liveness & Readiness Probes) — kubelet's default health signal is just 'is the process still alive' — a container can be running and yet…"
---

# 04 · Health Checks (Liveness & Readiness Probes)

!!! note "Not run against a live cluster"
    Manifests and output below follow documented kubelet probe behavior; not
    executed against a live cluster in this session.

## Why "container is running" isn't "container is healthy"

kubelet's default health signal is just "is the process still alive" — a
container can be running and yet deadlocked, stuck waiting on a dependency
forever, or still warming up a cache and unable to serve traffic. **Probes**
let kubelet ask the application itself whether it's actually working.

## Liveness probes: restart if stuck

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
    - name: app
      image: myapp:1.4
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 10
        failureThreshold: 3
        timeoutSeconds: 2
```

If `/healthz` fails 3 consecutive times (`failureThreshold`), kubelet kills
the container and the container runtime restarts it, subject to the Pod's
`restartPolicy`. This is for *unrecoverable-without-a-restart* states —
deadlocks, poisoned in-memory state — not for transient upstream issues,
which would just cause a restart loop without fixing anything.

## Readiness probes: pull out of Service traffic without restarting

```yaml
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3
```

A failing readiness probe does **not** restart the container — it removes
the Pod's IP from the Service's EndpointSlice so traffic stops being routed
to it, resuming automatically once the probe succeeds again. This is the
correct tool for "temporarily can't serve" (e.g. a downstream database
connection dropped) where restarting would be pointless or disruptive.

## Startup probes: protect slow-starting apps from premature liveness kills

```yaml
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30
        periodSeconds: 10   # up to 300s to start before liveness kicks in
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        periodSeconds: 10
        failureThreshold: 3
```

While a `startupProbe` is defined and hasn't yet succeeded, kubelet disables
the liveness *and* readiness probes entirely. This solves the classic
"JVM app takes 90s to boot but my liveness probe's `initialDelaySeconds` is
only 30s, so it gets killed before it ever finishes starting" problem
without having to loosen the liveness probe's steady-state timing.

## Probe mechanisms

```yaml
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
---
livenessProbe:
  tcpSocket:
    port: 5432
---
livenessProbe:
  grpc:
    port: 9090
```

| Mechanism | Success condition |
|---|---|
| `httpGet` | Response status `200–399` |
| `tcpSocket` | TCP connection to the port succeeds |
| `exec` | Command exits `0` |
| `grpc` | gRPC health-checking protocol responds `SERVING` |

## Worked example: readiness failure removes a Pod without restarting it

```bash
kubectl apply -f deployment.yaml   # 3 replicas, readiness on /ready
kubectl get endpoints web
# 3 IPs

kubectl exec deploy/web -- rm /tmp/ready-flag   # app now fails /ready
kubectl get pods -l app=web
# NAME          READY   STATUS    RESTARTS   AGE
# web-abc123    0/1     Running   0          3m   <- Running, NOT restarted

kubectl get endpoints web
# only 2 IPs now -- unready Pod pulled from rotation, traffic keeps flowing to the other 2
```

## How It Actually Works

- **Probes run entirely from kubelet on the container's own node, on a
  fixed timer — not event-driven.** Each probe (liveness/readiness/startup)
  is executed independently by kubelet's `probeManager` at `periodSeconds`
  intervals; `httpGet`/`tcpSocket` probes go over the network stack to the
  Pod's IP (from the node, not from inside the container's netns for
  `exec`, which does run inside via the container runtime's exec API) — the
  API server is not in the loop for the probe itself, only for reporting
  the resulting status.
- **A failing readiness probe changes cluster state via the API server, not
  locally.** kubelet doesn't touch traffic routing directly; it PATCHes the
  Pod's `status.conditions[Ready]` to `False` via the API server. The
  EndpointSlice controller, watching Pod readiness, then removes that Pod's
  IP from the EndpointSlice, and every node's kube-proxy (watching
  EndpointSlices) reprograms its iptables/IPVS rules to drop it — three
  separate reconciling components, chained by watches, produce the "traffic
  stops" effect with real (if small) end-to-end propagation delay across
  the cluster.
- **A failing liveness probe is enforced locally and immediately by
  kubelet, no API-server round trip needed.** kubelet directly calls the
  container runtime (containerd/CRI-O via the CRI gRPC API) to stop and
  restart the specific container — this is why liveness failures act faster
  and more deterministically than readiness-driven traffic changes, and why
  a liveness restart doesn't reschedule the Pod (same node, same Pod
  object, just a fresh container).
- **`failureThreshold`/`successThreshold` add hysteresis so a single blip
  doesn't flap state.** kubelet tracks consecutive failures/successes per
  probe type in memory; a probe must fail (or, after failing, succeed)
  `threshold` times in a row before kubelet acts — a single dropped TCP
  packet or one slow response under `timeoutSeconds` doesn't, by itself,
  restart anything or pull a Pod from load balancing.

## 🔀 Related lessons on other tracks

- [Docker — 06 · Health Checks](https://sigilipelli.github.io/docker-mastery-path/level-2/06-health-checks/)
- [Server Ops — 01 · High Availability Concepts (redundancy, failover, health checks)](https://sigilipelli.github.io/server-ops-mastery-path/level-3/01-ha-concepts/)

## Exercise

Add a `livenessProbe` and a separate `readinessProbe` to a Deployment,
pointed at two different endpoints on a small app you control (or `httpbin`,
using `/status/200` and a path you can flip to `/status/500`). Force the
readiness endpoint to fail and confirm via `kubectl get endpoints` that the
Pod is pulled from the Service without a restart; then force the liveness
endpoint to fail and confirm via `kubectl get pod` that `RESTARTS`
increments instead.
