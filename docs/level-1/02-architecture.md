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

## Exercise

Draw (on paper or in a text file) the control-plane/worker-node diagram from
memory: boxes for API server, etcd, scheduler, controller-manager on one
side; kubelet, container runtime, kube-proxy on the other. Then write one
sentence under each box describing its single responsibility. Being able to
redraw this from memory is the single most useful mental model for
debugging cluster issues later.
