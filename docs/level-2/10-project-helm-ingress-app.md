# 10 · Project — Ingress-Fronted Helm App

!!! note "Not run against a live cluster"
    Manifests and output below follow documented behavior of the components
    used (Helm, Ingress-nginx, PVC); not executed against a live cluster in
    this session.

## Goal

Package a two-tier app (a `web` frontend, a `postgres`-backed `api`) as a
Helm chart, deploy it behind an Ingress with TLS, back the database with a
PersistentVolumeClaim, and parameterize it for dev vs. prod via values
files — combining Modules 01, 02, 06, and 07.

## Chart layout

```text
shop/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
└── templates/
    ├── deployment-web.yaml
    ├── deployment-api.yaml
    ├── service-web.yaml
    ├── service-api.yaml
    ├── pvc-db.yaml
    ├── statefulset-db.yaml
    ├── ingress.yaml
    └── secret-db.yaml
```

```yaml
# values.yaml
web:
  replicas: 2
  image: { repository: shop-web, tag: "1.0.0" }
api:
  replicas: 2
  image: { repository: shop-api, tag: "1.0.0" }
db:
  storage: 5Gi
  storageClassName: standard
ingress:
  host: shop.local
  tls: false
```

```yaml
# values-prod.yaml
web:
  replicas: 4
api:
  replicas: 4
db:
  storage: 20Gi
ingress:
  host: shop.example.com
  tls: true
```

## The database tier: PVC + StatefulSet

```yaml
# templates/statefulset-db.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ .Release.Name }}-db
spec:
  serviceName: {{ .Release.Name }}-db
  replicas: 1
  selector:
    matchLabels: { app: {{ .Release.Name }}-db }
  template:
    metadata:
      labels: { app: {{ .Release.Name }}-db }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          envFrom:
            - secretRef: { name: {{ .Release.Name }}-db-secret }
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata: { name: data }
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: {{ .Values.db.storageClassName }}
        resources:
          requests: { storage: {{ .Values.db.storage }} }
```

A single-replica StatefulSet is used here (rather than a plain Deployment)
specifically so the PVC survives Pod rescheduling with a *stable* name
(`data-shop-db-0`) — Level 3, Module 01 covers multi-replica StatefulSets in
depth; here it's just "Deployment semantics don't fit a Pod holding
irreplaceable state."

## The Ingress, parameterized for TLS

```yaml
# templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  ingressClassName: nginx
  {{- if .Values.ingress.tls }}
  tls:
    - hosts: [{{ .Values.ingress.host }}]
      secretName: {{ .Release.Name }}-tls
  {{- end }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: {{ .Release.Name }}-web, port: { number: 80 } }
          - path: /api
            pathType: Prefix
            backend:
              service: { name: {{ .Release.Name }}-api, port: { number: 8080 } }
```

## Deploying dev, then promoting to prod

```bash
helm install shop-dev ./shop -f values.yaml -n dev --create-namespace
kubectl get pods -n dev
# shop-dev-web-...   2/2 Running
# shop-dev-api-...   2/2 Running
# shop-dev-db-0      1/1 Running

kubectl get ingress -n dev
# ADDRESS assigned by the ingress-nginx controller Service

curl -H "Host: shop.local" http://<ingress-ip>/
curl -H "Host: shop.local" http://<ingress-ip>/api/health
```

```bash
helm install shop ./shop -f values.yaml -f values-prod.yaml -n prod --create-namespace
kubectl get deploy -n prod
# shop-web  4/4
# shop-api  4/4
kubectl get ingress -n prod
# tls: shop-tls -- confirm HTTPS works
curl -k -H "Host: shop.example.com" https://<ingress-ip>/
```

## Verifying resilience end to end

```bash
kubectl exec shop-db-0 -n prod -- psql -U postgres -c "insert into orders(id) values (1);"
kubectl delete pod shop-db-0 -n prod
kubectl get pods -n prod -w
# shop-db-0 recreated by the StatefulSet, same PVC re-attached
kubectl exec shop-db-0 -n prod -- psql -U postgres -c "select * from orders;"
# row 1 still present -- PVC survived Pod deletion

kubectl delete pod -l app=shop-web -n prod --all
# Ingress keeps serving once ReplicaSet-created replacements pass readiness
```

## How It Actually Works

- **The Ingress, both Services, and the StatefulSet are independent
  controllers that only agree with each other via labels/selectors —
  Helm's only role was rendering and submitting them once.** After `helm
  install` returns, Helm is out of the loop entirely; the Deployment
  controller, StatefulSet controller, EndpointSlice controller, and the
  Ingress controller's own watch loop each independently reconcile their
  slice of this system continuously, which is why the app keeps working
  (and self-healing) long after the `helm install` process has exited.
- **The chart's parameterization only changes what gets rendered once, at
  install/upgrade time — it has no runtime effect.** `.Values.web.replicas`
  is substituted into the Deployment's `spec.replicas` field during `helm
  template`; nothing about the running cluster "knows" it came from a
  values file. Scaling via `kubectl scale` directly, bypassing Helm, works
  fine at runtime but will be silently reverted on the next `helm upgrade`
  if the values file wasn't also updated — a common source of "why did my
  manual scale-up disappear" surprises.
- **The StatefulSet's `volumeClaimTemplates` create one PVC per ordinal,
  named deterministically, and Helm has no special awareness of this.**
  `data-{{release}}-db-0` is created once (by the StatefulSet controller,
  not Helm) and is *not* deleted when the StatefulSet is scaled down or the
  release is uninstalled (`helm uninstall` does not clean up
  StatefulSet-managed PVCs by design) — this is a deliberate
  data-loss guardrail, and it means a `helm uninstall` followed by
  `helm install` of the same chart will reattach the *same* PVC and reuse
  old data.
- **The two Ingress paths (`/` and `/api`) are matched independently per
  request by the Ingress controller's own routing table — there is no
  cross-talk between them at the Kubernetes level.** Each path maps to a
  distinct Service/EndpointSlice pair; a failure or full rollout of `api`
  has zero effect on the Ingress controller's ability to keep routing `/`
  to `web`, because they are entirely separate reconciliation targets
  sharing only the same external IP and controller process.

## Exercise

Build this chart (or a smaller two-service version of it) with `web`,
`api`, a StatefulSet-backed `db`, and an Ingress routing by path. Install it
with `values.yaml` into a `dev` namespace, confirm both paths respond via
`curl -H Host: ...`, then delete the `db` Pod and confirm data written
before the deletion is still present after the StatefulSet recreates it.
Finally, install a second release into `prod` using `values-prod.yaml` and
confirm the two releases run with different replica counts without
interfering with each other.
