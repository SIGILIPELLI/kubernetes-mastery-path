# 03 · Installing a Local Cluster (minikube/kind)

!!! note "Not run against a live cluster"
    Every command below is the documented, standard install/usage flow for
    minikube and kind. It was not executed in this session — run it on your
    own machine and check the actual output, which will vary by OS, CPU
    architecture, and tool version.

You need a real (if small) Kubernetes cluster to practice against. Two good
free options for local development:

- **minikube** — runs a single- (or multi-) node Kubernetes cluster inside a
  VM or container on your machine. Closest to "a real cluster," with add-ons
  (dashboard, ingress, metrics-server) built in.
- **kind** ("Kubernetes IN Docker") — runs cluster nodes as Docker
  containers instead of VMs. Very fast to start/tear down; popular for CI.

Either is fine for this course. Examples below show both; pick one.

## Prerequisites

- A container runtime installed locally: **Docker Desktop** (or Docker
  Engine on Linux) is the simplest choice and works as the driver for both
  tools.
- `kubectl` installed (Module 04 covers it, but you need it now too).

Install `kubectl` (macOS example):

```bash
brew install kubectl
kubectl version --client
```

## Option A: minikube

Install (macOS via Homebrew; see minikube's docs for Linux/Windows):

```bash
brew install minikube
```

Start a cluster:

```bash
minikube start --driver=docker
```

This downloads a Kubernetes node image, boots it, and configures `kubectl`
to point at it. Typical output includes lines like:

```text
😄  minikube v1.33.0 on Darwin 14.5
✨  Using the docker driver based on user configuration
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
🔥  Creating docker container (CPUs=2, Memory=4000MB) ...
🐳  Preparing Kubernetes v1.30.0 on Docker 26.1.1 ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster
```

Verify:

```bash
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.30.0
```

Useful minikube commands you'll use throughout this course:

```bash
minikube status          # is it running?
minikube dashboard       # opens the web UI
minikube addons enable ingress   # needed for Level 2's Ingress module
minikube stop            # stop without deleting
minikube delete          # tear the cluster down completely
```

## Option B: kind

Install (macOS via Homebrew):

```bash
brew install kind
```

Create a cluster:

```bash
kind create cluster --name learning
```

Typical output:

```text
Creating cluster "learning" ...
 ✓ Ensuring node image (kindest/node:v1.30.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-learning"
```

Verify:

```bash
kubectl cluster-info --context kind-learning
kubectl get nodes
```

A multi-node kind cluster (useful later for scheduling/affinity experiments
in Level 3) needs a config file:

```yaml
# kind-multi-node.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --name multi --config kind-multi-node.yaml
```

Tear down:

```bash
kind delete cluster --name learning
```

## Checking your kubectl context

Both tools configure `~/.kube/config` and set the "current context" to
point at the cluster you just created. If you have multiple clusters
(minikube, kind, a cloud cluster), always confirm which one `kubectl` is
about to talk to before running anything destructive:

```bash
kubectl config current-context
kubectl config get-contexts
kubectl config use-context kind-learning   # switch if needed
```

This matters a lot in real jobs — accidentally running a command meant for
a local cluster against a production context is a classic, very avoidable
mistake.

## Worked example: first deployment smoke test

Once your cluster is up, confirm end-to-end functionality with a throwaway
Deployment:

```bash
kubectl create deployment hello --image=nginx:alpine
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# hello-6d4b9f8b7f-xk2pl   1/1     Running   0          10s

kubectl port-forward deployment/hello 8080:80
# Forwarding from 127.0.0.1:8080 -> 80
```

Visiting `http://localhost:8080` in a browser (or `curl localhost:8080`)
should return the default nginx welcome page, confirming the cluster can
pull images, schedule Pods, and route traffic. Clean up:

```bash
kubectl delete deployment hello
```

## Exercise

Install either minikube or kind, bring up a cluster, and run
`kubectl get nodes` and `kubectl get pods -A` (all namespaces) to see the
control-plane component Pods running. Then run the smoke-test Deployment
above and confirm you can reach nginx through `kubectl port-forward`. Keep
this cluster running — every module for the rest of Level 1 assumes you
have one available.
