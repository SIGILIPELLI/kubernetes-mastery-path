---
description: "Cluster Networking Deep Dive (CNI) — Each Pod gets its own network namespace; the CNI plugin creates a veth (virtual ethernet) pair — one end placed…"
---

# 09 · Cluster Networking Deep Dive (CNI)

!!! note "Not run against a live cluster"
    Diagrams and commands below follow documented CNI/kube-proxy behavior;
    not executed against a live cluster in this session.

## The four networking problems Kubernetes requires solving

Kubernetes doesn't implement networking itself — it defines a **model**
(every Pod gets its own IP; Pods can reach any other Pod's IP without NAT;
a Pod sees its own IP the same way others see it) and delegates the actual
implementation to a **CNI (Container Network Interface)** plugin. Four
distinct problems sit underneath this model:

1. **Pod-to-Pod on the same node** — a local bridge/veth problem.
2. **Pod-to-Pod across nodes** — an overlay or routed-network problem.
3. **Pod-to-Service** — kube-proxy's job (Level 1, Module 07).
4. **Pod-to-external / external-to-Pod** — NAT gateways and Ingress/LoadBalancer.

## Same-node Pod networking: veth pairs and a bridge

```text
Pod A (netns) --veth--> cni0 bridge <--veth-- Pod B (netns)
```

Each Pod gets its own **network namespace**; the CNI plugin creates a
**veth (virtual ethernet) pair** — one end placed inside the Pod's
namespace (appearing as `eth0`), the other left on the host and attached to
a Linux bridge (`cni0` for the reference bridge CNI, or an equivalent
construct for Calico/Cilium). Two Pods on the same node reach each other
by ordinary Ethernet frames traversing that bridge — no routing needed at
all for this hop.

```bash
# from the node itself:
ip link show | grep veth
# veth3f2a1c@if4: ... attached to cni0
brctl show cni0
# bridge name  bridge id       STP  interfaces
# cni0         8000...         no   veth3f2a1c, veth9b8e21
```

## Cross-node Pod networking: overlay vs. routed

Two node A/node B Pods need packets to cross the underlying physical
network, which knows nothing about Pod IPs. Two dominant approaches:

**Overlay (VXLAN)** — encapsulate each Pod packet inside a UDP packet
addressed node-to-node; the receiving node's CNI agent decapsulates it and
delivers it to the local bridge. Works on any underlying network (no
changes needed to physical routers) at the cost of encapsulation overhead
and a slightly larger MTU concern.

```bash
ip -d link show flannel.1
# vxlan id 1 ... — flannel's VXLAN interface, one per node
```

**Routed (BGP, e.g. Calico in non-overlay mode)** — each node announces
its Pod CIDR block to the rest of the network via BGP; no encapsulation
happens, packets are routed natively, which is faster but requires the
underlying network (or a route reflector) to support BGP peering.

```bash
calicoctl node status
# IPv4 BGP status
# +--------------+-------------------+-------+
# | PEER ADDRESS |     PEER TYPE     | STATE |
# | 10.0.1.5     | node-to-node mesh | up    |
```

## Pod CIDR allocation

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.podCIDR}{"\n"}{end}'
# node-1: 10.244.0.0/24
# node-2: 10.244.1.0/24
# node-3: 10.244.2.0/24
```

The cluster's overall Pod CIDR (e.g. `10.244.0.0/16`) is split into
per-node subnets by the controller manager (`--allocate-node-cidrs`); each
node's CNI plugin only needs to hand out IPs from its own /24, which is
also exactly why Pod IPs are stable-ish *per node* but not portable — a Pod
rescheduled to a different node gets an IP from that node's block.

## MTU and encapsulation overhead

```bash
ip link show flannel.1 | grep mtu
# mtu 1450   <- 1500 (typical Ethernet) minus VXLAN's 50-byte overhead
```

Getting this wrong (leaving the Pod interface at 1500 while the underlying
network's real MTU is 1500 and encapsulation adds bytes) causes silent
packet fragmentation or drops for larger payloads — a classically hard bug
because small test traffic works fine and only larger requests fail.

## Worked example: tracing a cross-node packet

```bash
kubectl get pod web-abc -o wide
# NODE: node-1, IP: 10.244.0.5
kubectl get pod api-def -o wide
# NODE: node-2, IP: 10.244.1.8

kubectl exec web-abc -- ping -c 2 10.244.1.8
# leaves web-abc's netns via veth -> cni0 bridge on node-1
# -> node-1 routes 10.244.1.0/24 via node-2's physical IP (or encapsulates via VXLAN)
# -> arrives at node-2, decapsulated if VXLAN, delivered via node-2's cni0 -> api-def's veth
```

## How It Actually Works

- **kubelet never implements networking itself — it invokes the CNI
  plugin binary as a subprocess per Pod lifecycle event, per the CNI
  spec.** When a Pod is scheduled, kubelet (via the container runtime's
  CRI implementation) calls the configured CNI plugin's `ADD` command with
  the Pod's network namespace path as an argument; the plugin (a static
  binary at `/opt/cni/bin/`, configured via JSON at
  `/etc/cni/net.d/`) is responsible for creating the veth pair, assigning
  an IP from its IPAM (IP Address Management) plugin, and wiring it to the
  bridge/overlay — this handoff is the entire integration surface between
  Kubernetes and the dozens of interchangeable CNI implementations.
- **IPAM assignment is typically a separate, chained CNI plugin, not
  built into the main network plugin.** A CNI config commonly chains a
  "main" plugin (bridge, calico, cilium) with an IPAM plugin (`host-local`,
  which allocates from the node's assigned Pod CIDR range and tracks leases
  in local files under `/var/lib/cni/networks/`) — this is why Pod IP
  exhaustion within a node manifests as `host-local` running out of free
  addresses in its local lease file, not as an API-server-visible error.
- **BGP-based routed networking (Calico's default mode) relies on each
  node's kernel routing table being programmed by a userspace BGP daemon
  (BIRD, in Calico's case) that peers with every other node (or a route
  reflector) — Kubernetes itself has no involvement in propagating these
  routes.** Calico's per-node agent (`calico-node`) watches the Kubernetes
  API for Pod/IPPool objects to know what to announce, then hands the
  actual protocol exchange to BIRD, which installs standard kernel routes
  (`ip route`) for each remote node's Pod CIDR pointing at that node's
  physical IP as the next hop — packets then flow at native L3 routing
  speed with zero encapsulation.
- **VXLAN overlay networking's decapsulation happens transparently in the
  kernel's network stack via a virtual device, not in any userspace
  proxy process.** The `flannel.1`-style VXLAN interface is a real Linux
  network device the kernel treats like any NIC; packets destined for a
  remote Pod CIDR are routed to this device, which the kernel
  automatically wraps in a UDP/VXLAN header addressed to the destination
  node's flanneld-maintained MAC/IP mapping (learned via a small
  userspace daemon writing to the kernel's FDB, forwarding database) — the
  receiving node's identical VXLAN device unwraps it just as transparently
  before handing the inner packet to its local bridge.

## Exercise

On a multi-node kind or minikube (multi-node profile) cluster, identify
which CNI is installed (`kubectl get pods -n kube-system` for
flannel/calico/cilium Pods) and find each node's `podCIDR` via `kubectl get
nodes -o jsonpath=...`. Schedule two Pods onto different nodes deliberately
(via `nodeName` or anti-affinity), ping one from the other, and use `ip
route`/`ip -d link show` on a node to identify whether cross-node traffic is
being encapsulated (a VXLAN/overlay interface) or routed natively (BGP
routes with no encapsulating device).
