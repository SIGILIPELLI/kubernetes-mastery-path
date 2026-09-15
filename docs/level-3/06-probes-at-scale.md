---
description: "Probes & Health at Scale — Level 2, Module 04 covered liveness/readiness/startup probes on one Pod. At scale, the questions shift: how do you avoid an…"
---

# 06 · Probes & Health at Scale

!!! note "Not run against a live cluster"
    Manifests and output below follow documented kubelet/PDB behavior; not
    executed against a live cluster in this session.

## Beyond a single probe: health as a system property

Level 2, Module 04 covered liveness/readiness/startup probes on one Pod.
At scale, the questions shift: how do you avoid an entire fleet failing
its health check simultaneously (a "thundering herd" of restarts), how do
you protect *availability* during voluntary disruptions (node drains,
rollouts), and how do you make probes cheap enough to run against hundreds
of replicas without becoming load themselves.

## Cascading probe failures: the thundering herd problem

If every replica's readiness probe depends on a shared downstream (a
database, a cache), a brief blip in that dependency can flip *every*
replica unready at once — the Service loses all backends simultaneously,
which is strictly worse than a single Pod being briefly unavailable.

```yaml
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
```

A shallow `/ready` that only checks "is the HTTP server listening" avoids
this coupling; a deep `/ready` that pings the database on every probe
couples the *entire fleet's* availability to that single dependency's
health. The common resolution: keep readiness shallow (or check a
locally-cached "last known good" status updated asynchronously) and handle
downstream failures with retries/circuit breakers in application code
instead of via the probe.

```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
  # jitter isn't a native field -- stagger initialDelaySeconds per replica via a Helm range or similar
```

Jittering `initialDelaySeconds` (or accepting the natural staggering from
rolling-update timing) avoids every replica's probe firing in the exact
same tick, which matters more as fleet size grows and probes themselves
start to add measurable load.

## PodDisruptionBudgets: protecting availability during voluntary disruption

Probes protect against *involuntary* failure (a crash). A
**PodDisruptionBudget (PDB)** protects against *voluntary* disruption — a
node drain, a cluster upgrade — ensuring the cluster won't evict so many
replicas at once that the app effectively goes down.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2          # or maxUnavailable: 1
  selector:
    matchLabels: { app: web }
```

```bash
kubectl apply -f web-pdb.yaml
kubectl drain node-1 --ignore-daemonsets
# evicting pod default/web-abc123
# evicting pod default/web-def456
# error when evicting pod "web-ghi789": Cannot evict pod as it would violate the pod's disruption budget.
```

`drain` (and any tool using the Eviction API — cluster autoscalers, managed
node upgrades) respects the PDB automatically; it will evict Pods one at a
time, waiting for replacements to become Ready, and outright refuses an
eviction that would breach `minAvailable`. A PDB has **zero effect** on
involuntary disruption — a node crashing outright doesn't ask permission.

## Probe cost at scale

```yaml
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  periodSeconds: 15   # not 1s -- 200 replicas probing every second is real, avoidable load
  timeoutSeconds: 2
  failureThreshold: 2
```

At 200 replicas with `periodSeconds: 1`, that's 200 requests/second of pure
overhead against the app before it serves a single real user — tuning
`periodSeconds` upward (with a correspondingly lower `failureThreshold` if
faster reaction is needed) is a real, easily overlooked cost lever.

## Worked example: PDB stops a drain from taking down a service

```bash
kubectl get deploy web
# 3/3 ready
kubectl apply -f web-pdb.yaml   # minAvailable: 2

kubectl drain node-1 --ignore-daemonsets
kubectl get pods -l app=web -o wide -w
# one Pod evicted and rescheduled at a time -- PDB blocks the second eviction
# until the first replacement Pod is Ready, never dropping below 2 available
```

## How It Actually Works

- **A PDB is enforced by the Eviction subresource's admission check, not
  by intercepting `kubectl delete`.** `kubectl drain` (and the cluster
  autoscaler, and managed-node-group upgrade tooling) calls the
  `pods/eviction` subresource rather than a plain delete; the API server's
  eviction handling consults the `disruption-controller`'s continuously
  maintained `PodDisruptionBudget.status.disruptionsAllowed` count for any
  PDB selecting that Pod and rejects the eviction request outright if
  granting it would push `disruptionsAllowed` below zero — a plain
  `kubectl delete pod` bypasses this entirely, which is why PDBs only ever
  protect *eviction-based* workflows, never direct deletion.
- **`disruptionsAllowed` is computed the same way readiness feeds into
  EndpointSlices — the disruption controller counts currently-Ready Pods
  matching the PDB's selector and subtracts the `minAvailable` (or applies
  `maxUnavailable`) on every reconcile tick, independent of any actual
  drain in progress.** This is why a PDB can already show
  `disruptionsAllowed: 0` even before anyone tries to drain anything — if a
  replica is already unready for unrelated reasons (a bad rollout, a
  crash), the budget for *voluntary* disruption is already spent by an
  *involuntary* one.
- **Readiness-probe coupling failures cascade because EndpointSlice
  updates are eventually consistent and identical for every replica hitting
  the same downstream at the same moment — there's no built-in
  jitter or circuit-breaking at the platform level.** kubelet on every node
  independently runs its own probe schedule against its own Pods with no
  cross-node coordination; if all replicas share a probe implementation
  hitting the same failing dependency, they will, absent deliberate
  staggering, converge on failing within one probe period of each other —
  Kubernetes provides the *mechanism* (readiness gates traffic) but no
  protection against a badly designed probe *body*.
- **`kubectl drain`'s eviction loop is a client-side retry loop, not a
  server-side queued operation.** `drain` repeatedly attempts eviction of
  remaining Pods on the node, backing off and retrying on
  `TooManyRequests`/PDB-violation responses, until either all Pods are
  evicted or a timeout is hit — there is no server-side "drain job" object;
  killing the `kubectl drain` process mid-drain simply stops further
  evictions with the node left partially drained and no in-cluster memory
  that a drain was ever in progress.

## Exercise

Deploy a 4-replica `web` Deployment with a `minAvailable: 3` PDB, then
attempt `kubectl drain` on a node hosting 2 of its replicas. Observe that
the drain evicts one Pod, waits for its replacement to become Ready, then
proceeds to the second — never allowing available replicas to drop below
3. Then scale `web` down to 3 replicas first (so all 3 must stay available)
and confirm the same drain now fails outright with a PDB violation error
for any of the two Pods being drained.
