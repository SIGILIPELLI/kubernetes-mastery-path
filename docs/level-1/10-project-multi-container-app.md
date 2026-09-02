# 10 · Project — Multi-Container App

!!! note "Not run against a live cluster"
    This manifest set was reasoned through against documented Kubernetes
    behavior and standard `nginx`/reverse-proxy configuration patterns, not
    executed against a live cluster in this session. Apply it yourself and
    treat this write-up as the expected behavior to verify against, not a
    guarantee.

## The goal

Combine everything from this level into one small, realistic deployment: an
**API backend** and a **frontend/proxy** running as two separate
Deployments in their own namespace, both configured via a ConfigMap and a
Secret, exposed internally via Services, with the frontend reachable from
outside the cluster.

```text
                     Namespace: capstone
                     ┌─────────────────────────────────────────┐
 you (kubectl        │   Service: frontend (NodePort)           │
 port-forward /      │        │                                 │
 minikube ip) ───────┼──────► Deployment: frontend (nginx proxy)│
                     │        │  reads ConfigMap: proxy-config   │
                     │        ▼                                 │
                     │   Service: api (ClusterIP)                │
                     │        │                                 │
                     │        ▼                                 │
                     │   Deployment: api (httpbin-style backend) │
                     │      reads ConfigMap: api-config          │
                     │      reads Secret: api-credentials         │
                     └─────────────────────────────────────────┘
```

## 1. Namespace

```yaml
# 00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: capstone
```

## 2. Backend configuration

```yaml
# 01-api-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: capstone
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: api-credentials
  namespace: capstone
type: Opaque
stringData:
  API_TOKEN: "demo-token-change-me"
```

## 3. Backend Deployment + Service

We use `kennethreitz/httpbin` (a widely used, small HTTP-echo test image) as
a stand-in "API" so the example is runnable without writing custom app code
— swap in your own image in a real project.

```yaml
# 02-api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: capstone
  labels:
    app: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: kennethreitz/httpbin:latest
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: api-config
          env:
            - name: API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: api-credentials
                  key: API_TOKEN
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: capstone
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 80
```

`api` is only reachable inside the cluster (`ClusterIP`) — exactly what we
want, since the frontend proxy is the only thing that should talk to it
directly.

## 4. Frontend proxy configuration

The frontend is an nginx reverse proxy, configured via a ConfigMap-mounted
`nginx.conf` that forwards `/api/` requests to the `api` Service by its
in-cluster DNS name:

```yaml
# 03-frontend-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: proxy-config
  namespace: capstone
data:
  default.conf: |
    server {
      listen 80;

      location / {
        return 200 'frontend ok\n';
        add_header Content-Type text/plain;
      }

      location /api/ {
        proxy_pass http://api.capstone.svc.cluster.local/;
        proxy_set_header Host $host;
      }
    }
```

Note the fully-qualified Service DNS name (`api.capstone.svc.cluster.local`)
— using the FQDN rather than the short name is a defensive habit inside
proxy configs, since it works regardless of which namespace the *proxy
container* itself resolves DNS relative to.

## 5. Frontend Deployment + Service

```yaml
# 04-frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: capstone
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: proxy-config
              mountPath: /etc/nginx/conf.d
      volumes:
        - name: proxy-config
          configMap:
            name: proxy-config
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: capstone
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

`frontend` is `NodePort` so it's reachable from outside the cluster (via
any node's IP on port 30080), matching Module 07's Service types.

## Deploying it

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-api-config.yaml
kubectl apply -f 02-api-deployment.yaml
kubectl apply -f 03-frontend-config.yaml
kubectl apply -f 04-frontend-deployment.yaml

# or, since apply accepts a directory, just:
# kubectl apply -f manifests/
```

## Verifying it

```bash
kubectl get all -n capstone
# NAME                            READY   STATUS    RESTARTS   AGE
# pod/api-5f6b8c9d7f-2xk9p        1/1     Running   0          30s
# pod/api-5f6b8c9d7f-h8k2p        1/1     Running   0          30s
# pod/frontend-6d9f7b8c5d-m4t7q   1/1     Running   0          28s
# pod/frontend-6d9f7b8c5d-p9x2r   1/1     Running   0          28s
#
# NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# service/api        ClusterIP   10.96.44.201    <none>        80/TCP         30s
# service/frontend   NodePort    10.96.201.9     <none>        80:30080/TCP   28s
#
# NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/api        2/2     2            2           30s
# deployment.apps/frontend   2/2     2            2           28s

minikube ip
# 192.168.49.2

curl http://192.168.49.2:30080/
# frontend ok

curl http://192.168.49.2:30080/api/get
# (httpbin's JSON echo of the request, proxied through frontend -> api Service -> api Pods)
```

(With `kind` instead of minikube, use `kubectl port-forward
svc/frontend 8080:80 -n capstone` and hit `http://localhost:8080` instead,
since `kind` doesn't expose node IPs directly to your host by default.)

## Confirming self-healing end-to-end

```bash
kubectl delete pod -n capstone -l app=api --all
kubectl get pods -n capstone --watch
# api Pods recreated by the ReplicaSet within seconds

curl http://192.168.49.2:30080/api/get
# still works -- the frontend never needed to know the api Pods changed,
# because it always talks to the stable "api" Service, not a Pod IP
```

## Tearing down

```bash
kubectl delete namespace capstone
```

One command removes every object created in this project, since they're
all namespaced.

## What you've demonstrated

- A **Namespace** isolating this project from anything else in the cluster.
- Two **Deployments** (each backed by a ReplicaSet, each self-healing).
- Two **Services** — one internal-only (`ClusterIP`), one externally
  reachable (`NodePort`) — decoupling both frontend and backend from Pod
  IP churn.
- **ConfigMaps** driving both application env vars and a mounted
  configuration file (the nginx proxy config).
- A **Secret** injected as an environment variable into the backend.
- Service-to-service communication over in-cluster DNS
  (`api.capstone.svc.cluster.local`).

This is, in miniature, the same shape as most real production Kubernetes
deployments — Level 2 builds directly on it with persistent storage,
Ingress (replacing this project's `NodePort` with a proper HTTP router),
resource limits, and health checks.
