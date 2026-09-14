# 10 · Project — Resilient Stateful Service

!!! note "Not run against a live cluster"
    Manifests and output below follow documented behavior of every
    component used; not executed against a live cluster in this session.

## Goal

Build a stateful service that survives node loss, scales safely, stays
isolated from other tenants, and can be safely drained for maintenance —
combining StatefulSets (Module 01), RBAC (Module 02), NetworkPolicy
(Module 03), HPA (Module 04), observability (Module 05), and
PodDisruptionBudgets (Module 06) into one system.

## The workload: a 3-node key-value store

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kv
  namespace: prod
spec:
  serviceName: kv
  replicas: 3
  podManagementPolicy: OrderedReady
  selector: { matchLabels: { app: kv } }
  template:
    metadata:
      labels: { app: kv }
    spec:
      serviceAccountName: kv-sa
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector: { matchLabels: { app: kv } }
      containers:
        - name: kv
          image: kv-store:2.1
          ports: [{ containerPort: 7000 }]
          readinessProbe:
            httpGet: { path: /ready, port: 7000 }
            periodSeconds: 10
          livenessProbe:
            httpGet: { path: /healthz, port: 7000 }
            periodSeconds: 15
            failureThreshold: 3
          resources:
            requests: { cpu: 200m, memory: 256Mi }
            limits: { cpu: 1, memory: 512Mi }
          volumeMounts:
            - name: data
              mountPath: /var/lib/kv
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 10Gi } }
---
apiVersion: v1
kind: Service
metadata:
  name: kv
  namespace: prod
spec:
  clusterIP: None
  selector: { app: kv }
  ports: [{ port: 7000 }]
```

`topologySpreadConstraints` with `maxSkew: 1` and `whenUnsatisfiable:
DoNotSchedule` ensures the scheduler refuses to place a second replica on a
node already holding one, as long as enough distinct nodes exist — this is
what actually makes "survives node loss" true; a StatefulSet alone gives no
such guarantee if all three replicas happened to land on one node.

## Availability under maintenance: PDB

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: kv-pdb
  namespace: prod
spec:
  minAvailable: 2
  selector: { matchLabels: { app: kv } }
```

## Least-privilege identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kv-sa
  namespace: prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kv-role
  namespace: prod
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]   # kv's own peer-discovery needs this, nothing more
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kv-role-binding
  namespace: prod
subjects:
  - kind: ServiceAccount
    name: kv-sa
    namespace: prod
roleRef: { kind: Role, name: kv-role, apiGroup: rbac.authorization.k8s.io }
```

## Network isolation

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kv-policy
  namespace: prod
spec:
  podSelector: { matchLabels: { app: kv } }
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector: { matchLabels: { app: api } }
      ports: [{ protocol: TCP, port: 7000 }]
    - from:
        - podSelector: { matchLabels: { app: kv } }   # peer-to-peer replication
      ports: [{ protocol: TCP, port: 7000 }]
  egress:
    - to: [{ podSelector: { matchLabels: { app: kv } } }]
      ports: [{ protocol: TCP, port: 7000 }]
    - to: [{ namespaceSelector: {} }]
      ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
```

## Scaling and observability

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kv-frontend-hpa   # note: HPA targets a stateless frontend in front of kv, not kv itself
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: kv-frontend }
  minReplicas: 3
  maxReplicas: 12
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

Deliberately **not** HPA'ing the StatefulSet itself: adding/removing
replicas of a stateful, peer-aware store means rebalancing data, which
isn't something a generic replica-count scaler can safely trigger — only
the stateless query-serving frontend in front of it autoscales.

## Worked example: killing a node mid-traffic

```bash
kubectl get pods -l app=kv -o wide -n prod
# kv-0  node-1
# kv-1  node-2
# kv-2  node-3   <- topology spread confirmed one per node

kubectl drain node-2 --ignore-daemonsets -n prod
# evicting pod prod/kv-1
# PDB (minAvailable: 2) permits this single eviction since kv-0 and kv-2 remain Ready

kubectl get pods -l app=kv -o wide -n prod -w
# kv-1 reschedules to node-2 once uncordoned, or stays Pending if no other node
# qualifies under the topology constraint -- reattaches to the SAME PVC (data-kv-1)

kubectl exec kv-0 -n prod -- curl -s localhost:7000/status
# cluster still reports 3-node quorum once kv-1 rejoins
```

## How It Actually Works

- **Every guardrail here is enforced by an independent controller with no
  awareness of the others — resilience emerges from composition, not from
  any single "resilient StatefulSet" feature.** The scheduler enforces
  topology spread at placement time; the StatefulSet controller enforces
  ordered identity and PVC reattachment; the disruption controller enforces
  the PDB at eviction time; the NetworkPolicy's CNI enforcement is
  independent of all of them. None of these components communicate with
  each other directly — each reacts only to the shared object state in
  etcd, which is precisely the loosely-coupled design that lets you add or
  remove one guardrail without touching the others.
- **`topologySpreadConstraints` is evaluated as a scheduler scoring/filter
  plugin at Pod creation time only — it does not continuously rebalance
  already-running Pods.** If node-2 is later added to the cluster after all
  three `kv` Pods already landed on nodes 1 and 3 (skew already violated),
  Kubernetes will not proactively move an existing Pod to fix the skew;
  the constraint only binds *new* scheduling decisions, which is why
  intentional rebalancing after cluster topology changes (e.g. via
  `descheduler`, a separate tool) is sometimes needed on top of this.
- **The PDB and the StatefulSet's ordered rollout/scale-down logic are
  two separate mechanisms that happen to compose safely, not one
  integrated feature.** A `kubectl drain` respects the PDB via the
  Eviction API; a `kubectl rollout restart statefulset/kv` instead uses
  the StatefulSet controller's own one-at-a-time ordered replacement
  logic, which happens to *also* never take more than one replica down at
  once by construction — the two mechanisms arrive at similar safety
  independently, and disabling one (`podManagementPolicy: Parallel`) does
  not disable the other (the PDB still blocks eviction-based disruption
  regardless).
- **NetworkPolicy's "peer-to-peer" self-referencing rule
  (`app: kv` selecting itself) is what specifically enables StatefulSet
  replicas to replicate data between each other while still blocking
  everything else — this pattern is easy to omit by accident.** A
  default-deny-plus-allow-from-api policy that forgets to also allow
  `kv`-to-`kv` traffic will silently break intra-cluster replication while
  looking, from the outside, like a fully functional and isolated service
  (external clients still get served fine by whichever replica currently
  holds fresh-enough data) — a failure mode that often isn't caught until
  a replica is lost and the surviving ones haven't actually been
  replicating to each other.

## Exercise

Build the `kv` StatefulSet with topology spread, PDB (`minAvailable: 2`),
scoped RBAC, and the two-directional NetworkPolicy above. Confirm one
replica per node via `kubectl get pods -o wide`, then drain the node
hosting one replica and confirm the PDB permits exactly the single eviction
needed while the other two stay Ready throughout. Finally, temporarily
remove the `kv`-to-`kv` NetworkPolicy rule and observe (via each replica's
own `/status` endpoint or logs) that replication between the two remaining
healthy replicas breaks even though external traffic keeps being served.
