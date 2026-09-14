# 03 · Cluster Upgrades & Maintenance

!!! note "Not run against a live cluster"
    Commands and version-skew rules below follow the documented Kubernetes
    upgrade policy; not executed against a live cluster in this session.

## The version-skew policy

Kubernetes supports upgrading components independently within strict skew
limits, and every upgrade plan starts from these rules:

- **kube-apiserver** is the ceiling — no other component may run a newer
  minor version than it.
- **kubelet / kube-proxy** may be up to **3 minor versions older** than
  kube-apiserver (as of recent Kubernetes releases; historically 2).
- **kube-controller-manager / kube-scheduler / cloud-controller-manager**
  may be up to **1 minor version older** than kube-apiserver.
- **kubectl** may be one minor version newer or older than the
  apiserver.

This is why the mandated upgrade order is always **control plane first,
then nodes**, one minor version at a time (never skip a minor version) —
upgrading a kubelet past the apiserver's version, even briefly, violates
skew and is unsupported.

```text
1.27 --> 1.28 --> 1.29   (never 1.27 --> 1.29 directly)
Order per hop: apiserver -> controller-manager/scheduler -> kubelet/kube-proxy
```

## Upgrading a kubeadm-managed control plane

```bash
# on the first control-plane node
apt-get update && apt-get install -y kubeadm=1.29.1-1.1
kubeadm upgrade plan
# shows: "Upgrade to the latest version in the v1.29 series: v1.29.1" and any warnings

kubeadm upgrade apply v1.29.1
# upgrades static pod manifests for apiserver/controller-manager/scheduler in place

apt-get install -y kubelet=1.29.1-1.1 kubectl=1.29.1-1.1
systemctl daemon-reload && systemctl restart kubelet

# on each additional control-plane node:
kubeadm upgrade node
```

`kubeadm upgrade apply` rewrites the static Pod manifests in
`/etc/kubernetes/manifests/`; kubelet, which watches that directory,
restarts the affected control-plane Pods automatically — no separate
"restart apiserver" step is needed or possible via `kubectl` (static Pods
aren't API objects kubelet takes deletion commands for from the API
server).

## Draining and upgrading a worker node

```bash
kubectl cordon node-3
# marks node-3 unschedulable: no NEW Pods will be placed there

kubectl drain node-3 --ignore-daemonsets --delete-emptydir-data --timeout=300s
# evicts existing Pods (via the Eviction API, respecting PodDisruptionBudgets)
# --ignore-daemonsets: DaemonSet pods aren't evicted (they're meant to run on every node)
# --delete-emptydir-data: required if any Pod uses emptyDir (data is node-local, will be lost)

# now safe to patch the OS / upgrade kubelet on node-3:
apt-get install -y kubelet=1.29.1-1.1 kubectl=1.29.1-1.1
systemctl restart kubelet

kubectl uncordon node-3
# marks node-3 schedulable again; existing Pods are NOT rescheduled back automatically
```

## PodDisruptionBudgets make drains safe

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: api
```

With this PDB in place, `kubectl drain` will refuse to evict a Pod if
doing so would drop `api`'s available replica count below 2 — the drain
command polls and retries rather than force-evicting, so a drain can
legitimately stall on a PDB that's too strict for the number of nodes
being drained concurrently. This is the single most common cause of
"drain hangs forever" in real upgrades.

## Worked example: a rolling multi-node upgrade

```bash
for node in node-1 node-2 node-3; do
  kubectl cordon "$node"
  kubectl drain "$node" --ignore-daemonsets --delete-emptydir-data --timeout=300s
  ssh "$node" "apt-get install -y kubelet=1.29.1-1.1 && systemctl restart kubelet"
  kubectl uncordon "$node"
  kubectl wait --for=condition=Ready "node/$node" --timeout=120s
done
```

Doing this one node at a time (not all three cordoned simultaneously)
keeps enough capacity live for PDBs to be satisfiable and for the
remaining nodes to absorb evicted Pods — draining all nodes at once with a
strict PDB will deadlock the drain entirely.

## How It Actually Works

- **`kubeadm upgrade apply` does not touch the etcd or kubelet
  configuration directly — it re-renders static Pod manifests and lets
  kubelet's file-watch loop do the actual restart.** kubeadm computes the
  new manifest content for `kube-apiserver.yaml`,
  `kube-controller-manager.yaml`, and `kube-scheduler.yaml` under
  `/etc/kubernetes/manifests/`, writes them atomically, and then simply
  waits — kubelet's static-Pod source (a filesystem watcher, one of
  several Pod sources kubelet supports alongside the API server) detects
  the changed file hash and recreates the container, which is why a
  control-plane "upgrade" causes a brief apiserver restart per node
  without any scheduler-driven Pod eviction.
- **`kubectl drain`'s eviction path goes through the Eviction API
  subresource, not a plain Pod delete — this is what makes it
  PDB-aware.** `POST /api/v1/namespaces/{ns}/pods/{name}/eviction`
  triggers the `disruption controller`'s admission check against any
  matching PDB's `status.disruptionsAllowed` counter; a plain `kubectl
  delete pod` bypasses this check entirely (PDBs only protect against
  voluntary disruption initiated through the Eviction API — a node dying
  outright ignores PDBs by necessity).
- **Version skew is enforced at connection time via each component's
  `--version`-negotiated API compatibility, not by a central admission
  check.** kubelet reports its version on every `NodeStatus` update; the
  API server doesn't reject an out-of-skew kubelet outright, but
  behavior becomes officially unsupported and can silently break features
  gated on newer API fields the older kubelet doesn't understand — the
  skew policy is a support boundary from upstream, enforced by convention
  and tooling (`kubeadm upgrade plan` warnings) rather than a hard runtime
  gate.
- **Uncordon does not trigger rebalancing.** `kubectl cordon`/`uncordon`
  only flip the `spec.unschedulable` field on the Node object, which the
  scheduler's node-filtering predicate checks when placing *new* Pods —
  Kubernetes has no built-in rebalancer that moves already-running Pods
  back onto a freshly uncordoned node, which is why nodes can stay
  unevenly loaded after a rolling upgrade until natural Pod churn (or a
  tool like the Descheduler) redistributes them.

## Exercise

On a local kind cluster with 3 worker nodes, create a Deployment with 4
replicas and a `PodDisruptionBudget` with `minAvailable: 3`. Attempt to
drain two of the three worker nodes at the same time and observe how the
second drain behaves relative to the PDB. Then drain them sequentially,
one at a time, and compare.
