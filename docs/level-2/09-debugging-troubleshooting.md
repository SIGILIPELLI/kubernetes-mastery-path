---
description: "Debugging & Troubleshooting Workloads — describe's Events section is the single richest source of 'why' — scheduling failures, image pull errors, probe…"
---

# 09 · Debugging & Troubleshooting Workloads

!!! note "Not run against a live cluster"
    Commands and output below follow documented kubectl/kubelet behavior;
    not executed against a live cluster in this session.

## A systematic order of operations

Random `kubectl get` commands waste time. A reliable order for "why isn't
this working":

1. Is the object even where you expect it? (`kubectl get`)
2. What does Kubernetes itself say went wrong? (`kubectl describe`, Events)
3. What does the application say? (`kubectl logs`)
4. If you need to see inside the running container: `kubectl exec`.
5. If the container won't even start: `kubectl debug`.

## Step 1–2: `get` and `describe`

```bash
kubectl get pods -n prod
# NAME          READY   STATUS             RESTARTS   AGE
# web-7d9f8c    0/1     ImagePullBackOff   0          2m
# api-9b2c1a    0/1     CrashLoopBackOff   5          10m
# worker-3f1e2  0/1     Pending            0          1m

kubectl describe pod web-7d9f8c -n prod
# Events:
#   Warning  Failed   kubelet  Failed to pull image "myapp:1.5.0": not found
```

`describe`'s **Events** section is the single richest source of "why" —
scheduling failures, image pull errors, probe failures, volume mount
errors, and OOM kills all surface there before anything shows up in
container logs (because in many of these cases, the container never even
started).

| STATUS | Usual cause | Where to look next |
|---|---|---|
| `Pending` | Can't be scheduled (resources, node affinity, no matching node) | `describe pod` Events |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag, private registry auth missing | `describe pod` Events |
| `CrashLoopBackOff` | Container starts then exits repeatedly | `logs --previous` |
| `Pending` on PVC | Nothing satisfies the PVC (no matching StorageClass/PV) | `describe pvc` |
| `Running` but `0/1 Ready` | Readiness probe failing | `describe pod`, app-level check |

## Step 3: logs, especially `--previous`

```bash
kubectl logs api-9b2c1a -n prod --previous   # the CRASHED container's logs, not the current restart's
kubectl logs api-9b2c1a -n prod -c sidecar    # multi-container Pod: name the container
kubectl logs -f deploy/api -n prod            # follow all Pods behind a Deployment (via label selector)
kubectl stern api -n prod                     # (if installed) tails multiple Pods with Pod-name prefixes
```

`--previous` is the single most-forgotten flag in a CrashLoopBackOff
investigation — by the time you run `kubectl logs`, the container may have
already restarted into a fresh (and initially quiet) instance, and plain
`logs` shows *that* one, not the one that actually crashed.

## Step 4: exec into a running container

```bash
kubectl exec -it api-9b2c1a -n prod -- sh
kubectl exec api-9b2c1a -n prod -- env
kubectl exec api-9b2c1a -n prod -- cat /etc/resolv.conf   # check DNS config from inside
```

`exec` only works if the container is running *and* has a shell/binary to
exec into — a `scratch`-based distroless image often has neither, which is
where `kubectl debug` comes in.

## Step 5: `kubectl debug` for containers with no shell, or that won't start at all

```bash
# attach a debug container with a full toolset to a running Pod's network/process namespace
kubectl debug -it api-9b2c1a -n prod --image=busybox:1.36 --target=api

# copy a Pod and swap in a shell-having image, for a container that crashes before you can exec
kubectl debug -it api-9b2c1a -n prod --image=busybox:1.36 --copy-to=api-debug --container=api -- sh

# debug a NODE (runs a privileged Pod chrooted into the node's filesystem)
kubectl debug node/worker-1 -it --image=busybox:1.36
```

`--copy-to` is the answer for "the container is stuck in
`ImagePullBackOff`/`CrashLoopBackOff` before I can attach anything" — it
creates a new Pod from the same spec with the target container's image and
command overridden, letting you reproduce the environment (mounts, env
vars, service account) without the broken entrypoint.

## Networking-specific checks

```bash
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh
# inside:
nslookup web.default.svc.cluster.local   # DNS resolving?
wget -qO- http://web:80                  # Service reachable?
nc -zv web 80                            # raw TCP check

kubectl get endpoints web    # are there actually any IPs behind the Service?
kubectl get networkpolicy -A # anything blocking traffic? (Level 3)
```

## Worked example: chasing a CrashLoopBackOff to its cause

```bash
kubectl get pods -n prod
# api-9b2c1a   0/1   CrashLoopBackOff   5   10m

kubectl describe pod api-9b2c1a -n prod
# Last State: Terminated, Reason: Error, Exit Code: 1

kubectl logs api-9b2c1a -n prod --previous
# panic: failed to connect to database: dial tcp 10.96.5.2:5432: connect: connection refused

kubectl get svc db -n prod
kubectl get endpoints db -n prod
# Endpoints: <none>  <- the actual root cause: the db Deployment's Pods aren't Ready, Service has no backends

kubectl describe pod -l app=db -n prod
# readiness probe failing -- the real fix is in the db Pod, not api
```

## How It Actually Works

- **Events are ephemeral and namespaced to the object's lifetime, stored
  as a separate API object with a TTL.** Every `Event` is its own resource
  in etcd (visible via `kubectl get events`), written by whichever
  component observed something (kubelet, scheduler, controllers), and
  garbage-collected by the API server after roughly one hour by default —
  this is why `describe`-ing a Pod that's been stable for hours often shows
  no Events at all, not because nothing happened, but because they expired.
- **`kubectl logs` reads from the container runtime's log files on the
  node, streamed through the API server as a proxy — it isn't stored in
  etcd at all.** kubelet configures containerd/CRI-O to write each
  container's stdout/stderr to a per-container log file under
  `/var/log/pods/` on that node; `kubectl logs` makes the API server open a
  connection to that node's kubelet, which tails the file — if the node is
  unreachable or the Pod has already been deleted and garbage-collected,
  the logs are simply gone (this is the real motivation for the log
  aggregation stack in Module 05 of Level 3).
- **`--previous` works because kubelet retains one prior terminated
  container's log file per Pod slot before overwriting it on the next
  restart.** The container runtime keeps the last-terminated container's
  filesystem/log association around specifically so `--previous` has
  something to point at; after a *second* restart, the crash-before-last is
  gone — only one generation back is recoverable this way.
- **`kubectl exec` and `kubectl debug --target` both work by attaching to
  existing Linux namespaces, not by modifying the running container.**
  `exec` calls the CRI `ExecSync`/`Exec` RPC, which runs a new process
  inside the container's existing PID/mount/network namespaces.
  `--target` on an ephemeral debug container instead creates a *new*,
  separate container that shares the *target* container's process
  namespace (`shareProcessNamespace`-style) and, for node debugging, its
  network namespace — enough to `ps`, inspect `/proc`, or curl loopback
  services as if you were inside the original container, without ever
  altering the original container's own filesystem or command.

## Exercise

Deploy an app Pod that depends on a Service (`db`) whose backing Deployment
you intentionally misconfigure (e.g. a wrong readiness probe path) so it
never becomes Ready. Watch the dependent Pod crash-loop, and walk the full
chain: `describe pod` on the crashing Pod, `logs --previous`, then trace the
root cause to `kubectl get endpoints db` showing no backends, then `describe
pod` on the `db` Pod itself to find the actual misconfiguration.
