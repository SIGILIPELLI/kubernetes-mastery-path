# 01 · What Is Kubernetes & Why Orchestration?

!!! note "Not run against a live cluster"
    The commands and output in this module were reasoned through against
    documented Kubernetes behavior, not executed against a real cluster.
    Try them yourself once you have a cluster (Module 03).

## The problem orchestration solves

Say you containerize an app with Docker. Running one container on one machine
is easy: `docker run myapp`. Production reality is messier:

- You need **multiple replicas** for load and redundancy.
- A container (or the whole machine) **crashes** — something has to notice
  and restart it, ideally on a healthy machine.
- You need to **roll out a new version** without downtime, and roll back fast
  if it's broken.
- Traffic has to find the right replicas even as they come and go, and be
  **load-balanced** across them.
- Configuration and secrets differ per environment (dev/staging/prod) and
  shouldn't be baked into the image.
- You want to **scale up and down** based on load, and use hardware
  efficiently across many services sharing a fleet of machines.

Doing all of this by hand with shell scripts and cron jobs does not scale
past a handful of services. **Container orchestration** is the class of
software that automates it: scheduling containers onto machines, keeping the
desired number running, routing traffic, managing rollouts, and reacting to
failures — continuously, without a human watching.

## What Kubernetes is

Kubernetes (often abbreviated **K8s** — "K", 8 letters, "s") is an
open-source container orchestration system, originally designed at Google
(based on their internal Borg/Omega systems) and donated to the Cloud Native
Computing Foundation (CNCF) in 2015. It is the de facto standard for running
containerized workloads at scale.

At its core, Kubernetes is a **declarative, reconciling control system**:

1. You **declare** the desired state — "I want 3 replicas of this container
   image, listening on port 8080, with this environment config."
2. You submit that declaration to Kubernetes as data (usually YAML).
3. Kubernetes continuously works to make the **actual state match the
   desired state** — starting containers, restarting failed ones,
   rescheduling work from a dead node, and so on — via a design called the
   **control loop** (or reconciliation loop).

This is fundamentally different from an imperative script that runs once
("start these 3 containers now") — Kubernetes keeps enforcing the desired
state forever, until you change or delete it.

## Core concepts, at a glance

You'll go deep on each of these in later modules — for now, a map of the
territory:

| Concept | What it is |
|---|---|
| **Cluster** | A set of machines (nodes) that Kubernetes manages as one unit |
| **Node** | A single machine (VM or physical) in the cluster, running workloads |
| **Pod** | The smallest deployable unit — one or more tightly-coupled containers |
| **Deployment** | Declares "run N replicas of this Pod template, keep them running" |
| **Service** | A stable network identity/load balancer in front of a set of Pods |
| **Namespace** | A way to partition a cluster into virtual sub-clusters |
| **ConfigMap / Secret** | Externalized configuration and sensitive data |

## Why not "just" Docker Compose?

Docker Compose is a fine tool for defining and running multi-container
applications **on a single machine** — great for local development. It has
no concept of:

- Multiple machines / scheduling across a fleet
- Self-healing (restarting containers on a *different* healthy node after one
  dies)
- Rolling updates with health-check-gated rollout
- Built-in service discovery and load balancing across a cluster
- Horizontal autoscaling based on live metrics
- A pluggable ecosystem of extensions (CRDs, operators, service meshes)

Kubernetes exists to fill exactly this gap: running containerized workloads
reliably across many machines, in production, at scale.

## Where Kubernetes runs

You'll typically meet Kubernetes in one of these forms:

- **Managed cloud services** — Amazon EKS, Google GKE, Azure AKS — the cloud
  provider runs the control plane for you.
- **Self-managed clusters** — you run the control plane yourself (kubeadm,
  or fully manual) on your own VMs or bare metal.
- **Local development clusters** — minikube, kind, k3d — a full (if scaled
  down) Kubernetes cluster running on your laptop, usually inside a VM or
  Docker containers. This is what Module 03 sets up.

## Worked example: the mental model in one sentence

> "I don't want to manage *containers* — I want to declare *desired state*
> and let the system keep reality matching it."

Concretely, when you later run:

```bash
kubectl apply -f deployment.yaml
```

you are not telling Kubernetes "start 3 containers right now" as a one-shot
command. You are submitting a **desired state record** ("3 replicas of this
Pod spec should exist") to the API server, which is then continuously
reconciled by controllers running in the control plane — if a Pod dies five
minutes later, a controller notices the actual count (2) no longer matches
the desired count (3) and creates a replacement, with no human involved.

## Exercise

Without touching a cluster yet, write down (in a notes file or scratch
document) your own one-paragraph answer to: "Why would a team running 15
microservices on 3 shared VMs want Kubernetes instead of hand-rolled
deploy scripts + `docker run`?" Reference at least three of: self-healing,
rolling updates, service discovery, resource-aware scheduling, and
horizontal scaling. This framing will make the mechanics in the rest of
this level click faster.
