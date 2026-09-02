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

## Exercise

Against your local cluster from Module 03: create a Deployment imperatively
(`kubectl create deployment demo --image=nginx:alpine`), then practice
`get`, `describe`, `logs`, and `exec -it ... -- /bin/sh` (try `ls /`, `env`,
and `exit` inside the shell) against the resulting Pod. Finally delete it
with `kubectl delete deployment demo` and confirm with `kubectl get pods`
that it's gone.
