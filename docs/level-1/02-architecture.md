---
description: "Kubernetes Architecture — A Kubernetes cluster has two categories of machines: the control plane (the 'brain') and worker nodes (where your workloads…"
---

# 02 · Kubernetes Architecture

!!! note "Not run against a live cluster"
    Component names, flags, and interactions below are drawn from the
    documented Kubernetes architecture, not from executing commands against
    a live cluster in this session.

A Kubernetes cluster has two categories of machines: the **control plane**
(the "brain") and **worker nodes** (where your workloads actually run).

## The control plane

The control plane makes global decisions about the cluster (scheduling,
detecting and responding to events) and typically runs on dedicated
machines (in managed services like EKS/GKE/AKS, the cloud provider hosts and
hides this for you).

### kube-apiserver

The front door to the cluster. Every interaction — `kubectl`, controllers,
kubelets, dashboards — goes through the **API server** over HTTPS/REST. It:

- Validates and processes requests (e.g., "create this Deployment").
- Is the *only* component that talks directly to `etcd`.
- Is stateless and horizontally scalable — you can run several behind a
  load balancer for high availability.

### etcd

A distributed, consistent **key-value store** that holds the entire cluster
state — every object (Pods, Deployments, Secrets, ConfigMaps, everything) is
persisted here. If `etcd` is lost without backup, the cluster's state is
lost. Production clusters run etcd as a clustered (typically 3 or 5 node)
quorum for fault tolerance and back it up regularly.

### kube-scheduler

Watches for newly created Pods that have no node assigned yet, and picks a
node for them to run on, based on:

- Resource requests/limits (does the node have enough free CPU/memory?)
- Affinity/anti-affinity rules, taints and tolerations (Level 3)
- Data locality, hardware/software constraints, and policy

The scheduler only **decides** where a Pod should run — it doesn't start
the container itself; that's the kubelet's job (below).

### kube-controller-manager

Runs the **controllers** — background control loops that watch the cluster
state via the API server and drive actual state toward desired state.
Examples: the Node controller (notices when a node goes unreachable), the
Deployment/ReplicaSet controller (keeps the right number of Pod replicas
running), the Job controller, and more. Conceptually each controller does:

```text
loop forever:
    observed = current state (from API server)
    desired  = spec (from API server)
    if observed != desired:
        take action to reconcile
```

### cloud-controller-manager

Present on managed cloud clusters — bridges Kubernetes to cloud-provider
APIs (provisioning load balancers for Services of type `LoadBalancer`,
attaching cloud disks for PersistentVolumes, labeling nodes with cloud
metadata). Not present on bare local clusters like a default minikube setup.

## Worker nodes

Every worker node runs the same three agents:

### kubelet

The primary "node agent." It:

- Registers the node with the API server.
- Watches the API server for Pods assigned to *its* node.
- Talks to the **container runtime** to actually start/stop containers.
- Reports Pod and node status (health, resource usage) back to the API
  server.
- Runs liveness/readiness/startup probes (Level 2) and restarts containers
  that fail them.

### Container runtime

The software that actually pulls images and runs containers, implementing
the **Container Runtime Interface (CRI)** that kubelet talks to. Modern
Kubernetes uses **containerd** or **CRI-O** (Docker Engine itself is no
longer used directly as the runtime as of Kubernetes 1.24+ — `dockershim`
was removed; container images built with `docker build` still run fine,
since they're just standard OCI images).

### kube-proxy

Maintains network rules on each node so that traffic to a Service's virtual
IP gets routed to one of the Pods backing it (Module 07 covers this in
detail). Implemented via `iptables` or `IPVS` rules on the node, depending
on configuration.

## Putting it together: what happens when you `kubectl apply` a Deployment

```text
1. kubectl sends the Deployment manifest to kube-apiserver (HTTPS, as JSON).
2. kube-apiserver validates it and writes it into etcd.
3. The Deployment controller (in kube-controller-manager) notices a new
   Deployment and creates a ReplicaSet object for it.
4. The ReplicaSet controller notices the ReplicaSet wants N Pods and creates
   N Pod objects (with no node assigned yet).
5. kube-scheduler notices unscheduled Pods, picks a node for each, and
   writes that assignment back via the API server.
6. The kubelet on each assigned node notices a Pod is scheduled to it, and
   tells the container runtime to pull the image and start the container(s).
7. kubelet reports Pod status (Running, Ready, etc.) back through the API
   server, which updates etcd.
8. kube-proxy on every node updates its routing rules so the Service
   (if any) load-balances to the new Pods once they're Ready.
```

No single component does all of this — it's a chain of independent
controllers, each watching the API server and reacting, which is why
Kubernetes is often described as a system built from cooperating control
loops rather than one monolithic program.

## Worked example: locating the pieces with kubectl

Once you have a cluster (Module 03), these commands show you the
architecture in action:

```bash
# List the nodes in the cluster and their roles
kubectl get nodes -o wide

# See control-plane components running as Pods (in kubeadm-style clusters)
kubectl get pods -n kube-system

# Inspect one node's capacity and conditions in detail
kubectl describe node <node-name>
```

On a single-node local cluster (minikube/kind), the same machine plays both
control-plane and worker roles, but the same logical components (API
server, scheduler, controller-manager, kubelet, kube-proxy) are all present
and doing their jobs — you can see most of them as Pods in the `kube-system`
namespace with `kubectl get pods -n kube-system`.

## How It Actually Works

Each control-plane component has a distinct internal mechanism worth
knowing precisely, because most debugging eventually points at one of them:

- **etcd** stores every object as a key under a path like
  `/registry/pods/<namespace>/<name>`, using the Raft consensus algorithm
  across an odd number of members (3 or 5 in production) so that a
  majority quorum must agree before a write is committed. Every write
  gets a monotonically increasing `revision` number — this revision is
  what watch streams use to resume exactly where they left off after a
  disconnect, and it's also what powers optimistic concurrency: every
  object carries a `resourceVersion` (derived from that revision), and a
  write that doesn't match the version it read against is rejected with a
  409 Conflict rather than silently overwriting a concurrent change.
- **kube-apiserver** is the only component that talks to etcd directly.
  Every request — from `kubectl`, from controllers, from kubelets — goes
  through a fixed pipeline: authentication (who are you), authorization
  (RBAC — are you allowed to do this verb on this resource),
  admission control (mutating webhooks that can modify the object, then
  validating webhooks that can reject it), and only then a read/write to
  etcd. This is why the API server, not etcd, is the single source of
  truth for validation logic and the only safe integration point.
- **kube-scheduler** does not get invoked directly by anything creating a
  Pod. It runs its own watch loop specifically for Pods whose
  `spec.nodeName` is empty, then for each one runs a **filter phase**
  (predicates: does the node have enough allocatable CPU/memory, does it
  satisfy nodeSelector/affinity/taints-tolerations) to produce a feasible
  set, followed by a **score phase** (priorities: spread pods across
  nodes, prefer nodes with more free resources, etc.) that ranks the
  survivors — the highest-scoring node wins, and the scheduler commits
  the decision with a `Bind` API call that sets `spec.nodeName`, which
  is itself just a normal write back to the API server/etcd.
- **kube-controller-manager** runs many independent control loops
  (Node controller, ReplicaSet controller, Endpoints controller, and
  dozens more) in one process, each watching only the object types it
  owns and reconciling toward desired state as described in Module 01.
- **kubelet** on each worker node doesn't just "run containers" — it
  reconciles the set of Pods assigned to its node (`spec.nodeName ==
  <this node>`) against the container runtime via the **CRI**
  (Container Runtime Interface), a gRPC API implemented by containerd or
  CRI-O. It also runs the periodic liveness/readiness probe loop and
  reports node/Pod status back to the API server roughly every 10
  seconds (`--node-status-update-frequency`), which is what
  `kubectl describe node` is actually displaying.
- **kube-proxy** doesn't proxy traffic in the literal sense on most
  clusters — it watches Service and EndpointSlice objects and
  translates them into either iptables NAT rules or IPVS virtual server
  entries directly in the Linux kernel's networking stack, so that a
  packet to a Service's ClusterIP is DNAT'd to a Pod IP by the kernel
  itself with no proxy process in the data path at all.

## Exercise

Draw (on paper or in a text file) the control-plane/worker-node diagram from
memory: boxes for API server, etcd, scheduler, controller-manager on one
side; kubelet, container runtime, kube-proxy on the other. Then write one
sentence under each box describing its single responsibility. Being able to
redraw this from memory is the single most useful mental model for
debugging cluster issues later.
