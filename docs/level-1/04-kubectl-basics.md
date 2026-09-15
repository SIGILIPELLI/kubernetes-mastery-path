---
description: "kubectl Basics — kubectl is the command-line client that talks to the Kubernetes API server. Nearly everything you do with Kubernetes goes through it (or…"
---

# 04 · kubectl Basics

!!! note "Not run against a live cluster"
    Command syntax and example output below follow documented `kubectl`
    behavior; they were not executed against a live cluster in this
    session. Actual output (Pod names, ages, IPs) will differ on your
    machine.

`kubectl` is the command-line client that talks to the Kubernetes API
server. Nearly everything you do with Kubernetes goes through it (or a tool
built on the same API, like Helm or a CI pipeline).

## General command shape

```text
kubectl <verb> <resource-type> [name] [flags]
```

Examples:

```bash
kubectl get pods
kubectl get pod my-pod
kubectl describe deployment my-app
kubectl delete service my-service
kubectl apply -f manifest.yaml
```

## The core verbs

| Verb | Purpose |
|---|---|
| `get` | List resources, or show one in brief |
| `describe` | Show detailed info + recent events for one resource |
| `create` | Imperatively create a resource (quick, one-off) |
| `apply` | Declaratively create/update from a YAML file (preferred) |
| `delete` | Remove a resource |
| `logs` | Print container logs |
| `exec` | Run a command inside a running container |
| `edit` | Open a resource in your editor and apply changes on save |

## `get`: listing resources

```bash
kubectl get pods
# NAME                      READY   STATUS    RESTARTS   AGE
# web-7d8f9c6b7d-2xk9p      1/1     Running   0          3m

kubectl get pods -o wide          # extra columns: node, IP
kubectl get pods --watch          # stream changes live
kubectl get all                   # pods, services, deployments, etc. at once
kubectl get pods -n kube-system   # in a specific namespace
kubectl get pods -A               # across ALL namespaces
kubectl get pod web-7d8f9c6b7d-2xk9p -o yaml   # full object as YAML
```

`-o` (`--output`) accepts `wide`, `yaml`, `json`, or a custom
`-o jsonpath='{...}'` for scripting.

## `describe`: the debugging workhorse

```bash
kubectl describe pod web-7d8f9c6b7d-2xk9p
```

Prints the full spec/status **plus an Events section** at the bottom — this
Events list is usually the fastest way to diagnose why a Pod won't start
(image pull errors, failed scheduling, crash loops, failed probes all show
up there with timestamps).

## `logs`: reading container output

```bash
kubectl logs web-7d8f9c6b7d-2xk9p            # current logs
kubectl logs web-7d8f9c6b7d-2xk9p -f          # follow (stream), like tail -f
kubectl logs web-7d8f9c6b7d-2xk9p --previous  # logs from a crashed prior instance
kubectl logs web-7d8f9c6b7d-2xk9p -c sidecar  # a specific container in a multi-container Pod
```

## `exec`: getting a shell inside a container

```bash
kubectl exec -it web-7d8f9c6b7d-2xk9p -- /bin/sh
kubectl exec web-7d8f9c6b7d-2xk9p -- env
```

`-it` allocates an interactive TTY and keeps stdin open — needed for an
interactive shell. Everything after `--` is the command run *inside* the
container, not interpreted by `kubectl` itself.

## `apply` vs `create`: declarative vs imperative

```bash
# Imperative: quick and one-off, but not repeatable/idempotent in the same way
kubectl create deployment web --image=nginx:alpine

# Declarative: write the desired state to a file, then apply it
kubectl apply -f deployment.yaml
```

`kubectl apply -f` is the standard, production-grade way to manage
resources: re-running it with an updated file **updates** the existing
resource to match (a three-way diff against the last-applied config), rather
than erroring because the resource already exists. This is what makes YAML
manifests checked into version control ("infrastructure as code") the
normal workflow — Module 09 covers manifest structure in depth.

```bash
kubectl apply -f deployment.yaml     # apply one file
kubectl apply -f ./manifests/        # apply every file in a directory
kubectl diff -f deployment.yaml      # preview what apply would change
```

## `delete`

```bash
kubectl delete pod web-7d8f9c6b7d-2xk9p
kubectl delete -f deployment.yaml     # delete everything defined in a file
kubectl delete deployment web --grace-period=0 --force   # force-delete (use sparingly)
```

## Namespaces and context

```bash
kubectl get pods -n staging                 # one-off namespace override
kubectl config set-context --current --namespace=staging   # change the default
```

Module 09 covers namespaces themselves; for now, know that most `kubectl`
commands default to the `default` namespace unless you say otherwise.

## Shortcuts worth knowing immediately

```bash
kubectl get po        # "po" = pods
kubectl get svc        # "svc" = services
kubectl get deploy      # "deploy" = deployments
kubectl get cm          # "cm" = configmaps
kubectl get ns          # "ns" = namespaces
kubectl api-resources    # full list of resource types and their shortnames
```

## Worked example: full inspect-and-fix loop

A typical debugging session, chaining the verbs above:

```bash
kubectl get pods
# NAME                  READY   STATUS             RESTARTS   AGE
# web-6c9d8f5b6d-h8k2p  0/1     ImagePullBackOff   0          45s

kubectl describe pod web-6c9d8f5b6d-h8k2p
# ... Events:
#   Warning  Failed   kubelet  Failed to pull image "nginx:alpin":
#            not found: manifest unknown

# Found it -- typo in the image tag. Fix the manifest, then:
kubectl apply -f deployment.yaml

kubectl get pods --watch
# NAME                  READY   STATUS    RESTARTS   AGE
# web-7d8f9c6b7d-2xk9p  1/1     Running   0          8s
```

## How It Actually Works

Every `kubectl` command is a thin REST/JSON client — understanding the
request it actually sends demystifies most "why didn't that work" moments:

- **`get`/`describe`/`delete` are HTTP verbs against a REST resource
  path.** `kubectl get pods -n foo` performs `GET
  /api/v1/namespaces/foo/pods`; `kubectl describe` does the equivalent
  `GET` on the object plus a separate `GET` for related Events
  (`/api/v1/namespaces/foo/events?fieldSelector=involvedObject.name=...`)
  and stitches them together client-side — there is no single
  "describe" API, it's kubectl composing multiple calls into readable
  text.
- **`apply` computes a three-way merge patch, not an overwrite.**
  `kubectl apply` stores the last-applied configuration as JSON in the
  `kubectl.kubernetes.io/last-applied-configuration` annotation on the
  live object. On the next `apply`, it diffs three things — the
  last-applied config, your new local file, and the current live
  object on the server — to figure out which fields you intentionally
  removed (present in last-applied, absent in new file) versus fields
  something else changed that you should leave alone (present in live
  object but never mentioned by you). `create`, by contrast, is a
  simple `POST` that fails outright if the object already exists.
- **`logs` and `exec` don't go through etcd at all.** These commands
  have the API server open a streaming connection (an upgraded
  HTTP/SPDY or WebSocket connection) directly to the kubelet on the
  node hosting the Pod, which in turn asks the container runtime (via
  CRI's `Exec`/`Attach` or the container's log file on disk) to stream
  the output back. This is why `logs`/`exec` fail with a distinct
  "error dialing backend" class of error when the node is unreachable,
  even though `kubectl get pod` for that same Pod succeeds (get only
  needs etcd/API server, not the node).
- **`--watch` keeps the same long-lived watch connection described in
  Module 01** rather than polling — this is why watched output appears
  event-by-event with no fixed delay, and why killing the network
  briefly causes kubectl to silently reconnect and resync via a fresh
  `List` + `Watch` rather than losing events.

## Exercise

Against your local cluster from Module 03: create a Deployment imperatively
(`kubectl create deployment demo --image=nginx:alpine`), then practice
`get`, `describe`, `logs`, and `exec -it ... -- /bin/sh` (try `ls /`, `env`,
and `exit` inside the shell) against the resulting Pod. Finally delete it
with `kubectl delete deployment demo` and confirm with `kubectl get pods`
that it's gone.
