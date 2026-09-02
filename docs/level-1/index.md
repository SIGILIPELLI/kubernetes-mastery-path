# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: understand why Kubernetes exists, stand up a local cluster, and become
fluent with the core objects — Pods, Deployments, Services, ConfigMaps, and
Secrets — well enough to ship a small multi-container application.

## Modules

1. [What Is Kubernetes & Why Orchestration?](01-what-is-kubernetes.md)
2. [Kubernetes Architecture](02-architecture.md)
3. [Installing a Local Cluster (minikube/kind)](03-local-cluster-setup.md)
4. [kubectl Basics](04-kubectl-basics.md)
5. [Pods](05-pods.md)
6. [Deployments & ReplicaSets](06-deployments-replicasets.md)
7. [Services & Networking Basics](07-services-networking.md)
8. [ConfigMaps & Secrets](08-configmaps-secrets.md)
9. [Namespaces & YAML Manifests](09-namespaces-manifests.md)
10. [Project — Multi-Container App](10-project-multi-container-app.md)

By the end of this level you'll be able to explain what each control-plane and
node component does, run a local cluster, write and apply YAML manifests, and
deploy a small application composed of a Deployment, a Service, and a
ConfigMap.

!!! note "No cluster was used to verify these commands"
    Every command and manifest in this level was written and reasoned through
    against documented Kubernetes behavior (kubectl/API reference, official
    docs) rather than executed against a live cluster in this session. Run
    them yourself against a local cluster (minikube/kind) to confirm the
    exact output on your machine and Kubernetes version.
