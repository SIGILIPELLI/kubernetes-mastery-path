# 07 · Services & Networking Basics

!!! note "Not run against a live cluster"
    Manifests and output below follow documented Service/DNS behavior; not
    executed against a live cluster in this session.

## The problem Services solve

Pods are ephemeral — they get new IP addresses every time they're
recreated (a crash, a rollout, a rescheduled node). If another part of your
app hardcoded a Pod's IP, it would break the moment that Pod was replaced.
A **Service** gives a stable network identity (a fixed virtual IP and DNS
name) in front of a *set* of Pods, selected by label, and load-balances
traffic across whichever Pods currently match.

```text
Client --> Service (stable IP/DNS) --> one of the matching Pods (changes over time)
```

## A ClusterIP Service (the default type)

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80          # port the Service listens on
      targetPort: 80    # port the container listens on
```

`spec.selector` works exactly like a Deployment's — it's a label match. Any
Pod (from any Deployment, or a bare Pod) carrying `app: web` becomes a
backend for this Service.

```bash
kubectl apply -f service.yaml
kubectl get svc web
# NAME   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# web    ClusterIP   10.96.123.45   <none>        80/TCP    5s

kubectl get endpoints web
# NAME   ENDPOINTS                                 AGE
# web    10.244.1.5:80,10.244.1.6:80,10.244.1.7:80 5s
```

The `Endpoints` object (or the newer `EndpointSlice`) is the live, automatically
maintained list of Pod IPs currently matching the selector — kube-proxy on
every node watches this and programs local routing rules so traffic to the
Service's ClusterIP gets distributed across these endpoints.

**`ClusterIP`** (the default) is only reachable *from inside the cluster* —
other Pods, not your laptop's browser directly (though `kubectl
port-forward` can tunnel to it for testing, as in Module 03/05).

## Service types

| Type | Reachable from | Typical use |
|---|---|---|
| `ClusterIP` (default) | Inside the cluster only | Internal service-to-service traffic |
| `NodePort` | Any node's IP, on a fixed high port (30000-32767) | Simple external access, dev/test |
| `LoadBalancer` | The internet, via a cloud load balancer | Production external access (cloud only) |
| `ExternalName` | N/A — DNS alias to an external name | Pointing at a service outside the cluster |

`NodePort` example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080   # optional -- auto-assigned if omitted
```

```bash
kubectl apply -f service-nodeport.yaml
minikube ip   # e.g. 192.168.49.2
curl http://192.168.49.2:30080
```

`LoadBalancer` (only fully functional on a real cloud provider — on
minikube, `minikube tunnel` simulates it):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

On EKS/GKE/AKS, creating this triggers the cloud-controller-manager
(Module 02) to provision an actual cloud load balancer and populate
`EXTERNAL-IP` with its address.

## DNS: how Pods find Services by name

Kubernetes runs a cluster DNS service (usually **CoreDNS**) that
automatically creates a DNS record for every Service:

```text
<service-name>.<namespace>.svc.cluster.local
```

From any Pod in the same namespace, you can reach the `web` Service just by
its short name:

```bash
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh
# inside the temporary pod:
wget -qO- http://web        # short name, same namespace
wget -qO- http://web.default.svc.cluster.local   # fully qualified
```

From a different namespace, you need at least `web.<namespace>` since the
short name alone resolves relative to your own namespace's search domain.

## Worked example: Service survives Pod churn

```bash
kubectl apply -f deployment.yaml   # 3-replica "web" Deployment from Module 06
kubectl apply -f service.yaml

kubectl get endpoints web
# 3 IPs listed

kubectl delete pod -l app=web --all   # kill all 3 Pods at once
kubectl get pods -l app=web --watch   # ReplicaSet recreates them with NEW IPs

kubectl get endpoints web
# a DIFFERENT set of 3 IPs -- but the Service's ClusterIP and DNS name never changed
```

This is the entire point of a Service: everything that talks to `web`
(other Pods, an Ingress in Level 2) keeps using the same stable name/IP,
completely unaffected by the fact that the underlying Pod IPs changed
underneath it.

## Exercise

Apply the 3-replica `web` Deployment from Module 06 plus the `ClusterIP`
Service above. From a temporary debug Pod (`kubectl run tmp --rm -it
--image=busybox:1.36 --restart=Never -- sh`), curl/wget the Service by its
short DNS name several times and note that responses may come from
different backend Pods (if you add a per-Pod identifier to the response,
e.g. by mounting the Pod name as an env var via `fieldRef`, you'd see it
rotate — for now, just confirm the request succeeds repeatedly). Then
delete all the Pods at once and confirm the Service keeps working once the
ReplicaSet replaces them.
