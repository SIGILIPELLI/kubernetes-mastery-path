# 02 · Ingress & Load Balancing

!!! note "Not run against a live cluster"
    Manifests and output below follow documented Ingress/controller behavior;
    not executed against a live cluster in this session.

## Why Services aren't enough for HTTP routing

A `LoadBalancer` Service (Level 1, Module 07) gives one external IP per
Service — fine for a single app, expensive and unwieldy once you have ten
HTTP services that all want to live under one domain with path- or
host-based routing (`api.example.com`, `example.com/admin`, TLS
termination, etc.). An **Ingress** describes that routing as one object;
an **Ingress controller** (a separate piece of software you install, e.g.
NGINX Ingress Controller, Traefik, or a cloud-managed one) reads Ingress
objects and actually implements the routing.

```text
Internet --> one LoadBalancer/NodePort --> Ingress controller Pod --> routes by host/path --> Service --> Pods
```

## A basic Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 8080
```

```bash
kubectl apply -f web-ingress.yaml
kubectl get ingress web-ingress
# NAME          CLASS   HOSTS               ADDRESS         PORTS   AGE
# web-ingress   nginx   shop.example.com    203.0.113.10    80      30s
```

`ingressClassName: nginx` tells Kubernetes *which* installed controller
should watch this object — a cluster can run several controllers
simultaneously (e.g. one internal, one internet-facing), each watching only
Ingresses that name its `IngressClass`.

## TLS termination

```yaml
spec:
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-tls   # kind: Secret, type: kubernetes.io/tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

```bash
kubectl create secret tls shop-tls --cert=cert.pem --key=key.pem
```

The controller terminates TLS at the edge using the certificate/key in
`shop-tls`, then forwards plain HTTP to the backend Service — this is why
`targetPort` on the backend Service is usually the plain-HTTP container
port, not 443. In production, `cert-manager` (a separate controller) is
typically layered on top to request and auto-renew these certificates from
Let's Encrypt.

## Installing an Ingress controller (NGINX example)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.0/deploy/static/provider/cloud/deploy.yaml
kubectl get pods -n ingress-nginx
# ingress-nginx-controller-... Running
kubectl get svc -n ingress-nginx ingress-nginx-controller
# TYPE: LoadBalancer -- this is the single external entry point for ALL Ingresses in the cluster
```

An Ingress object with no controller installed just sits there inert —
`kubectl get ingress` will show no `ADDRESS` — because nothing is watching
it.

## Worked example: host-based routing to two Services

```bash
kubectl apply -f deployment-web.yaml deployment-api.yaml service-web.yaml service-api.yaml
kubectl apply -f web-ingress.yaml

curl -H "Host: shop.example.com" http://203.0.113.10/
# routed to the "web" Service
curl -H "Host: shop.example.com" http://203.0.113.10/api
# routed to the "api" Service, same external IP, same port 80
```

One external IP, one LoadBalancer, routing decided entirely by HTTP
`Host`/path — this is the cost saving Ingress exists for.

## How It Actually Works

- **The Ingress object is pure declared intent; the controller is a
  separate reconciliation loop with its own data plane.** The API server
  stores Ingress objects like any other resource — it does not implement
  HTTP routing itself. The Ingress controller Deployment runs a watch loop
  against Ingress, Service, EndpointSlice, and Secret objects and
  regenerates its *own* runtime configuration (for NGINX, an actual
  `nginx.conf` reloaded via `nginx -s reload`, or for Envoy-based
  controllers, an xDS config push) whenever anything relevant changes.
- **The controller talks to Pods directly, bypassing kube-proxy in most
  implementations.** NGINX Ingress Controller reads EndpointSlices and
  proxies straight to backend Pod IPs rather than routing through the
  backend Service's ClusterIP — this shaves off one extra hop of NAT and
  lets it do L7-aware load balancing (least-connections, session affinity)
  that a Service's random/round-robin L4 balancing can't express.
- **The one external IP is a single Service under the hood.** The
  controller's own Pods sit behind one `LoadBalancer`-type (or `NodePort`)
  Service; every Ingress object in the cluster shares that same entry
  point, and host/path matching happens entirely inside the controller
  process after the packet already arrived — the cloud load balancer has
  no idea Ingress objects exist.
- **Ordering and merge semantics for overlapping rules follow the
  controller's implementation, not a Kubernetes-wide spec.** The Ingress
  API only weakly specifies conflict resolution (e.g. two Ingresses
  claiming the same host+path); NGINX Ingress Controller resolves this by
  Ingress creation timestamp and object name — this is a common source of
  "why did my Ingress change stop working" surprises when multiple
  Ingress objects target overlapping routes.

## Exercise

Deploy two simple Services (`web`, `api`) and a single Ingress that routes
`/` to `web` and `/api` to `api` under one host. Confirm both paths resolve
to the right backend using `curl -H "Host: ..."` against the controller's
external IP. Then add a TLS block referencing a self-signed `kubernetes.io/tls`
Secret and confirm `curl -k https://...` terminates TLS at the Ingress while
the backend Pods still only speak plain HTTP.
