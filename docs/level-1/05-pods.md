# 05 · Pods

!!! note "Not run against a live cluster"
    The manifests and command output below were reasoned through against
    documented Kubernetes API behavior, not executed against a live
    cluster. Apply them yourself and compare.

## What a Pod is

A **Pod** is the smallest deployable unit in Kubernetes — not a container.
A Pod wraps **one or more containers** that:

- Share the same network namespace (same IP address and port space — they
  can reach each other over `localhost`).
- Can share storage volumes (Module 08/Level 2's volumes module).
- Are always scheduled together, onto the same node, and live/die together.

Most Pods run exactly one container — the "one or more" matters for the
**sidecar pattern**: a small helper container (a log shipper, a proxy, a
config reloader) running alongside your main container in the same Pod.

## A minimal Pod manifest

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    app: hello
spec:
  containers:
    - name: hello
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
```

Every Kubernetes manifest has this same top-level shape:

- `apiVersion` — which version of the Kubernetes API this object belongs to
  (`v1` for core objects like Pod, Service; `apps/v1` for Deployment,
  StatefulSet).
- `kind` — the resource type.
- `metadata` — name, namespace, labels, annotations.
- `spec` — the desired state, specific to the `kind`.

Apply and inspect it:

```bash
kubectl apply -f pod.yaml
kubectl get pods
# NAME        READY   STATUS    RESTARTS   AGE
# hello-pod   1/1     Running   0          5s

kubectl describe pod hello-pod
kubectl logs hello-pod
```

## Pods are usually not created directly

In practice you will almost never write a bare Pod manifest for a real
workload — a Pod created directly like this has no self-healing: if the
node it's on dies, or the Pod is deleted, **nothing recreates it**. Real
workloads use a **Deployment** (Module 06), which manages Pods for you via a
ReplicaSet and recreates them automatically. You're learning bare Pods here
because Deployments create Pods that look exactly like this under the hood —
understanding the Pod spec is understanding the unit everything else
manages.

## Multi-container Pods (sidecar pattern)

```yaml
# pod-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logger
spec:
  containers:
    - name: web
      image: nginx:1.27-alpine
      ports:
        - containerPort: 80
      volumeMounts:
        - name: logs
          mountPath: /var/log/nginx
    - name: log-shipper
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /var/log/nginx/access.log"]
      volumeMounts:
        - name: logs
          mountPath: /var/log/nginx
  volumes:
    - name: logs
      emptyDir: {}
```

Both containers share the `logs` volume (an `emptyDir` — ephemeral storage
tied to the Pod's lifetime, covered fully in Level 2) and the same network
namespace, so `log-shipper` could equally reach `web` via `localhost:80`.

```bash
kubectl logs web-with-logger -c web            # logs from the "web" container
kubectl logs web-with-logger -c log-shipper    # logs from the sidecar
```

The `-c` flag is required whenever a Pod has more than one container.

## The Pod lifecycle (phases)

`kubectl get pods` shows a Pod's **phase** in the `STATUS` column:

| Phase | Meaning |
|---|---|
| `Pending` | Accepted by the cluster, but not yet scheduled or still pulling images |
| `Running` | Bound to a node, at least one container is running |
| `Succeeded` | All containers exited with status 0 (normal for Jobs, not long-running apps) |
| `Failed` | All containers terminated, at least one with non-zero exit |
| `Unknown` | The Pod's state couldn't be determined (usually a node communication problem) |

Within `Running`, the `READY` column (`1/1`, `0/1`, etc.) reflects how many
containers are passing their readiness checks — a container can be
`Running` but not `Ready` (Level 2 covers readiness probes).

## Common bad states you'll meet immediately

- **`ImagePullBackOff` / `ErrImagePull`** — the image name/tag is wrong, or
  the registry requires auth you haven't configured.
- **`CrashLoopBackOff`** — the container starts, then exits (crashes) —
  Kubernetes keeps retrying with exponential backoff. Check
  `kubectl logs <pod> --previous` for the crash's error output.
- **`Pending` forever** — usually insufficient cluster resources, or a
  scheduling constraint (node selector, taint) that no node satisfies;
  `kubectl describe pod` Events will say why.

## Worked example: diagnosing a CrashLoopBackOff

```yaml
# pod-crash.yaml
apiVersion: v1
kind: Pod
metadata:
  name: crashy
spec:
  containers:
    - name: crashy
      image: busybox:1.36
      command: ["sh", "-c", "echo starting; sleep 2; exit 1"]
```

```bash
kubectl apply -f pod-crash.yaml
kubectl get pods --watch
# NAME     READY   STATUS              RESTARTS   AGE
# crashy   0/1     ContainerCreating   0          2s
# crashy   1/1     Running             0          4s
# crashy   0/1     Error               0          6s
# crashy   0/1     CrashLoopBackOff    1          20s

kubectl logs crashy --previous
# starting
```

The container legitimately exits with code 1 after 2 seconds every time, so
kubelet keeps restarting it with growing backoff delays — this is the
expected, documented behavior for `restartPolicy: Always` (the default for
bare Pods).

## Exercise

Apply the `hello-pod` manifest above, `describe` it and read the Events
section, then `exec` into it (`kubectl exec -it hello-pod -- sh`) and run
`hostname` and `curl localhost:80` inside the container (install curl or use
`wget -qO-` if curl is missing from the alpine image). Then apply
`pod-crash.yaml`, watch it enter `CrashLoopBackOff`, and use
`kubectl logs crashy --previous` to see the exit output before cleaning
both Pods up with `kubectl delete pod hello-pod crashy`.
