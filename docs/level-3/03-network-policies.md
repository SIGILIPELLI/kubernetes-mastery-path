# 03 · Network Policies

!!! note "Not run against a live cluster"
    Manifests below follow the documented NetworkPolicy spec and common CNI
    (Calico/Cilium) enforcement behavior; not executed against a live
    cluster in this session.

## The default: every Pod can talk to every Pod

Out of the box, Kubernetes networking is a flat space — any Pod can reach
any other Pod's IP on any port, across namespaces, with no isolation at
all. A **NetworkPolicy** restricts this, but only if the cluster's CNI plugin
actually implements policy enforcement (Calico, Cilium, Weave — not the
basic `kubenet` or a CNI running purely in "connectivity only" mode); the
object itself is inert on a CNI that ignores it.

## Default-deny: the essential first policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}      # matches ALL Pods in this namespace
  policyTypes:
    - Ingress
    - Egress
```

An empty `podSelector: {}` with no `ingress`/`egress` rules means "select
every Pod, and allow nothing" — this is the recommended starting point for
any namespace, because NetworkPolicies are purely additive: apply this
first, then layer on specific `allow` policies for exactly the traffic you
expect.

## Allowing specific traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: prod
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { app: web }
      ports:
        - protocol: TCP
          port: 8080
```

This policy targets Pods labeled `app: api` and allows inbound TCP:8080
*only* from Pods labeled `app: web` in the **same namespace** — every other
source (other namespaces, other Pods, outside the cluster) stays blocked by
the default-deny.

## Cross-namespace traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: prod
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: monitoring }
          podSelector:
            matchLabels: { app: prometheus }
      ports:
        - protocol: TCP
          port: 9090
```

Combining `namespaceSelector` **and** `podSelector` in one `from` entry
means "this specific Pod label, in this specific namespace" (an AND). Two
separate entries in the `from` list would instead be an OR — matching
either condition independently.

## Egress: restricting what a Pod can reach outward

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress
  namespace: prod
spec:
  podSelector:
    matchLabels: { app: api }
  policyTypes: [Egress]
  egress:
    - to:
        - podSelector:
            matchLabels: { app: db }
      ports: [{ protocol: TCP, port: 5432 }]
    - to:      # allow DNS -- almost always required, easy to forget
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Forgetting the DNS egress rule is the single most common NetworkPolicy
mistake — a Pod under egress default-deny that can't reach CoreDNS on
UDP/TCP 53 will fail to resolve *any* name, including Service DNS, and
everything downstream looks like a mysterious connectivity failure that's
actually a DNS failure.

## Worked example: isolating a database tier

```bash
kubectl apply -f default-deny-all.yaml -n prod
kubectl apply -f allow-web-to-api.yaml -n prod
kubectl apply -f allow-api-to-db.yaml -n prod

kubectl exec -it deploy/web -n prod -- wget -qO- http://api:8080/health
# succeeds -- explicitly allowed

kubectl exec -it deploy/web -n prod -- nc -zv db 5432
# times out -- web is NOT in the allow-list for db's ingress

kubectl exec -it deploy/api -n prod -- nc -zv db 5432
# succeeds
```

## How It Actually Works

- **NetworkPolicy objects are pure declarative intent stored in etcd like
  any other resource — the API server enforces nothing about network
  traffic itself.** Enforcement is entirely delegated to whatever CNI
  plugin is installed; the plugin's agent (Calico's `calico-node`,
  Cilium's `cilium-agent`) runs a controller that watches NetworkPolicy,
  Pod, and Namespace objects and translates the *selector-based* rules into
  actual packet-filtering primitives on each node.
- **Selector-based rules are re-evaluated continuously against live Pod
  labels, not resolved once to a fixed IP list.** When a new Pod matching
  `app: web` appears (a scale-up, a rolling update), the CNI's controller
  recomputes the policy's effective IP set and reprograms enforcement
  immediately — there's no manual step to "add the new Pod to the
  allow-list" because the rule was never about IPs, only labels, from the
  start.
- **Enforcement mechanism differs by CNI, with real performance and
  granularity implications.** Calico in iptables mode compiles policies
  into iptables rules per node, matched against Pod IPs recorded in
  ipsets (for fast set-membership checks instead of one rule per Pod);
  Cilium instead attaches eBPF programs at the veth/socket layer that
  evaluate policy per-packet in kernel space using identity-based labels
  attached to each packet (not raw IPs) — this is why Cilium can enforce
  L7-aware policies (e.g. "only allow HTTP GET to /health") that pure L3/L4
  iptables-based enforcement cannot express.
- **Multiple NetworkPolicies selecting the same Pod are unioned, and
  `policyTypes` determines whether "no matching policy" means allow or
  deny.** Once *any* NetworkPolicy selects a Pod for `Ingress`, that Pod's
  default flips from allow-all to deny-all for ingress, and every
  ingress-type policy selecting it contributes an OR'd set of allow rules —
  a Pod is never made *more* restrictive by adding another allow policy;
  policies can only add exceptions to an implicit deny, never subtract
  from one, which is why there's no "deny" rule type at all in the base API.

## Exercise

In a namespace with `web`, `api`, and `db` Deployments, apply a
`default-deny-all` policy for both ingress and egress, then add exactly the
rules needed for `web -> api` (port 8080) and `api -> db` (port 5432), plus
DNS egress for both. Confirm with `kubectl exec ... -- nc -zv` that `web`
can reach `api` but not `db` directly, and that `api` can still resolve
`db`'s DNS name and connect to it.
