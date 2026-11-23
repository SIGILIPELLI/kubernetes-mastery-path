---
title: "Kubernetes Interview Questions and Answers (179 Questions)"
description: "179 Kubernetes interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Kubernetes Mastery Path."
---

# Kubernetes Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Kubernetes Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is Kubernetes &amp; Why Orchestration?](level-1/01-what-is-kubernetes.md)

**Q1.** What does container orchestration automate?

**Answer:** Scheduling containers onto machines, keeping the desired number running, routing traffic, managing rollouts and reacting to failures

It does this continuously without a human watching.

**Q2.** What does 'declarative, reconciling control system' mean?

**Answer:** You declare desired state and Kubernetes continuously works to make actual state match it

This is the control (reconciliation) loop.

**Q3.** Where did Kubernetes come from?

**Answer:** Designed at Google (based on Borg/Omega) and donated to the CNCF in 2015

It is the de facto standard for containerized workloads at scale.

**Q4.** Why can't shell scripts and cron jobs scale for production?

**Answer:** They don't scale past a handful of services for restarts, rollouts, load balancing and scaling

Orchestration automates the messy production realities.

### [02 · Kubernetes Architecture](level-1/02-architecture.md)

**Q5.** Which component is the only one that talks directly to etcd?

**Answer:** kube-apiserver

Every interaction goes through the API server.

**Q6.** What does etcd hold?

**Answer:** The entire cluster state as a consistent key-value store

If etcd is lost without backup, the cluster's state is lost.

**Q7.** What does the kube-scheduler do?

**Answer:** Picks a node for newly created Pods; it only decides, and the kubelet starts the container

It weighs resource requests, affinity rules, taints and policy.

**Q8.** What does the controller manager do?

**Answer:** Runs control loops that drive actual state toward desired state

Examples: the Node controller and the Deployment/ReplicaSet controller.

### [03 · Installing a Local Cluster (minikube/kind)](level-1/03-local-cluster-setup.md)

**Q9.** How does kind differ from minikube?

**Answer:** kind runs cluster nodes as Docker containers, making it fast to start and tear down and popular for CI

minikube runs a node in a VM or container and ships add-ons like dashboard and ingress.

**Q10.** Which command starts a minikube cluster using Docker?

**Answer:** minikube start --driver=docker

It configures kubectl to point at the new cluster.

**Q11.** How do you tear a minikube cluster down completely?

**Answer:** minikube delete

minikube stop only stops it without deleting.

**Q12.** What do both tools need as a prerequisite?

**Answer:** A container runtime such as Docker, plus kubectl

Docker Desktop works as the driver for both.

### [04 · kubectl Basics](level-1/04-kubectl-basics.md)

**Q13.** What is the general shape of a kubectl command?

**Answer:** kubectl &lt;verb&gt; &lt;resource-type&gt; [name] [flags]

For example kubectl describe deployment my-app.

**Q14.** Which section of kubectl describe is usually the fastest way to diagnose why a Pod won't start?

**Answer:** Events

Image pull errors, failed scheduling, crash loops and failed probes show up there with timestamps.

**Q15.** How do you read logs from a previously crashed container?

**Answer:** kubectl logs &lt;pod&gt; --previous

-f streams like tail -f.

**Q16.** Which is preferred: create or apply?

**Answer:** apply, declaratively from a YAML file

create is for quick one-offs.

### [05 · Pods](level-1/05-pods.md)

**Q17.** What is a Pod?

**Answer:** The smallest deployable unit: one or more containers sharing a network namespace, scheduled together onto one node

Containers in a Pod can reach each other over localhost.

**Q18.** What is the sidecar pattern?

**Answer:** A small helper container such as a log shipper, proxy or config reloader running alongside the main container in the same Pod

It's why a Pod can hold more than one container.

**Q19.** Why not create bare Pods for real workloads?

**Answer:** They have no self-healing: nothing recreates one if its node dies or it's deleted

Use a Deployment, which manages Pods via a ReplicaSet.

**Q20.** Which four top-level fields does every manifest have?

**Answer:** apiVersion, kind, metadata and spec

spec holds the desired state specific to the kind.

### [06 · Deployments &amp; ReplicaSets](level-1/06-deployments-replicasets.md)

**Q21.** What does a ReplicaSet ensure?

**Answer:** A specified number of identical Pod replicas is running at all times

It replaces dead Pods and deletes extras.

**Q22.** What does a Deployment add on top of a ReplicaSet?

**Answer:** Rollouts: a new ReplicaSet shifts replicas gradually from the old one, with history for rollback

You almost always create a Deployment, not a bare ReplicaSet.

**Q23.** What must spec.selector.matchLabels match?

**Answer:** spec.template.metadata.labels; a mismatch is a validation error

It's how the Deployment identifies which Pods belong to it.

**Q24.** What triggers a new ReplicaSet and rolling update?

**Answer:** Changing the Pod template, e.g. a new image or env var

Scaling replicas doesn't create a new ReplicaSet.

### [07 · Services &amp; Networking Basics](level-1/07-services-networking.md)

**Q25.** Why do Services exist?

**Answer:** Pods are ephemeral and get new IPs when recreated, so a Service gives a stable IP and DNS name in front of a set of Pods

It load-balances across the Pods currently matching its selector.

**Q26.** What is the default Service type and its reach?

**Answer:** ClusterIP, reachable only from inside the cluster

kubectl port-forward can tunnel to it for testing.

**Q27.** Which Service type opens a fixed port in the 30000–32767 range on every node?

**Answer:** NodePort

It's used for simple external access in dev/test.

**Q28.** What does the Endpoints object hold?

**Answer:** The live list of Pod IPs currently matching the selector

kube-proxy watches it to program routing rules.

### [08 · ConfigMaps &amp; Secrets](level-1/08-configmaps-secrets.md)

**Q29.** Why externalize configuration from the image?

**Answer:** Otherwise you rebuild the image for every environment or config change

ConfigMaps hold non-sensitive data; Secrets hold sensitive data.

**Q30.** How can Pods consume ConfigMaps and Secrets?

**Answer:** As environment variables or as mounted files

Both objects work the same two ways.

**Q31.** What does envFrom.configMapRef do?

**Answer:** Injects every key in the ConfigMap as an environment variable

valueFrom.configMapKeyRef pulls in a single named key.

**Q32.** How are Secret values stored?

**Answer:** base64-encoded (encoding, not encryption), with special handling like exclusion from describe output

They can also be encrypted at rest.

### [09 · Namespaces &amp; YAML Manifest Structure](level-1/09-namespaces-manifests.md)

**Q33.** What does a namespace do?

**Answer:** Partitions one physical cluster into multiple virtual clusters

Names only need to be unique within a namespace.

**Q34.** Which resources are cluster-scoped rather than namespaced?

**Answer:** Nodes and PersistentVolumes

Most others live in exactly one namespace.

**Q35.** How do you reach a Service named web in the staging namespace from the default namespace?

**Answer:** http://web.staging (or web.staging.svc.cluster.local)

The short name works only within the same namespace.

**Q36.** What happens when you delete a namespace?

**Answer:** Everything in it is deleted too (cascade)

Check before running kubectl delete namespace.

### [10 · Project — Multi-Container App](level-1/10-project-multi-container-app.md)

**Q37.** How is the frontend exposed to the outside in the project?

**Answer:** Through a NodePort Service

The backend is exposed internally through a ClusterIP Service.

**Q38.** What does the API backend read for credentials?

**Answer:** A Secret named api-credentials

Non-sensitive settings live in the api-config ConfigMap.

**Q39.** Why does the project use a httpbin image as the 'API'?

**Answer:** It's a small HTTP-echo test image, so the example runs without custom app code

Swap in your own image in a real project.

**Q40.** Why does the note say to treat the write-up as expected behavior to verify?

**Answer:** The manifests were reasoned through, not run on a live cluster, so you should apply and verify them yourself

It's not a guarantee.

## Level 2 · Intermediate

### [01 · Volumes &amp; Persistent Storage](level-2/01-volumes-persistent-storage.md)

**Q41.** What happens to data in a container's writable layer on restart?

**Answer:** It's discarded and recreated from the image

Volumes attach external storage so data can survive.

**Q42.** How long does emptyDir data live?

**Answer:** Until the Pod is removed from the node (it survives a container restart but not Pod deletion)

It's good for sharing files between containers in the same Pod or for scratch space.

**Q43.** Why is hostPath a security concern?

**Answer:** Files dropped there are visible to anything else on that node

Legitimate uses are narrow, like node-level agents.

**Q44.** How do PV and PVC split responsibilities?

**Answer:** A PV is the actual storage; a PVC is a Pod's request for storage

This lets data outlive any one Pod or node.

### [02 · Ingress &amp; Load Balancing](level-2/02-ingress-load-balancing.md)

**Q45.** Why isn't one LoadBalancer Service per app enough for HTTP routing?

**Answer:** One external IP per Service gets expensive and unwieldy with many HTTP services under one domain

Ingress describes host/path routing in one object.

**Q46.** What does an Ingress controller do?

**Answer:** Reads Ingress objects and actually implements the routing

It's separate software such as NGINX Ingress, Traefik or a cloud-managed one.

**Q47.** What does ingressClassName: nginx do?

**Answer:** Tells Kubernetes which installed controller should watch this Ingress

A cluster can run several controllers.

**Q48.** How is TLS termination configured in the Ingress?

**Answer:** A tls section with hosts and a secretName pointing to a kubernetes.io/tls Secret

Create it with kubectl create secret tls --cert --key.

### [03 · Resource Requests, Limits &amp; Scheduling](level-2/03-resource-requests-limits.md)

**Q49.** What do requests and limits each do?

**Answer:** Requests tell the scheduler what a container needs; limits cap what it may actually use

Without them the scheduler is guessing.

**Q50.** How does the scheduler decide whether a Pod fits a node?

**Answer:** By summing requests of already-scheduled Pods plus this Pod's, never looking at live usage

The NodeResourcesFit predicate filters nodes that would exceed allocatable capacity.

**Q51.** What happens when a container exceeds its CPU limit versus its memory limit?

**Answer:** CPU is throttled; memory exceeding the limit gets the container OOMKilled

CPU limits never kill a container.

**Q52.** What does cpu: 250m mean?

**Answer:** 0.25 of a vCPU core

Mi/Gi are base-1024 memory units.

### [04 · Health Checks (Liveness &amp; Readiness Probes)](level-2/04-health-checks-probes.md)

**Q53.** What does a failing liveness probe cause?

**Answer:** kubelet kills the container and the runtime restarts it, subject to restartPolicy

It's for unrecoverable-without-restart states like deadlocks.

**Q54.** What does a failing readiness probe cause?

**Answer:** The Pod's IP is removed from the Service's endpoints with no restart, resuming when it passes

It's the right tool for temporarily being unable to serve.

**Q55.** What does a startupProbe do?

**Answer:** Disables liveness and readiness probes until it succeeds, protecting slow-starting apps from premature kills

The example allows up to 300s to start.

**Q56.** Why isn't 'container is running' the same as 'healthy'?

**Answer:** A container can be deadlocked, stuck on a dependency or still warming up while its process is alive

Probes let kubelet ask the application itself.

### [05 · Rolling Updates &amp; Rollbacks](level-2/05-rolling-updates-rollbacks.md)

**Q57.** What do maxUnavailable and maxSurge bound?

**Answer:** How many Pods may be down and how many extra Pods may exist during a rollout

With 6 replicas, maxUnavailable 1 and maxSurge 1.

**Q58.** How does a rolling update change Pods?

**Answer:** It creates a new ReplicaSet and shifts replica counts step by step, never editing Pods in place

The old ReplicaSet is kept at zero for rollback per revisionHistoryLimit.

**Q59.** How do you roll back to the previous revision?

**Answer:** kubectl rollout undo deployment/web

Add --to-revision=N for a specific one.

**Q60.** Why does CHANGE-CAUSE show &lt;none&gt;?

**Answer:** It's populated only if you set the kubernetes.io/change-cause annotation

--record is deprecated.

### [06 · Helm Basics (Charts &amp; Values)](level-2/06-helm-basics.md)

**Q61.** What is a Helm chart?

**Answer:** A templated, versioned package of manifests

It solves the problem of parameterizing and versioning raw YAML across environments.

**Q62.** Where do .Values come from?

**Answer:** values.yaml, overridable per install

For example --set image.tag=1.5.0.

**Q63.** What is a helm rollback in terms of revisions?

**Answer:** Another upgrade creating a new revision whose target is an old revision's manifest

Rollback to revision 1 produced revision 3.

**Q64.** What are the three template objects mentioned?

**Answer:** .Values, .Release and .Chart

Release has install-time metadata; Chart has Chart.yaml fields.

### [07 · Multi-Environment Configuration](level-2/07-multi-environment-config.md)

**Q65.** Why not keep three full copies of every manifest for dev, staging and prod?

**Answer:** It invites drift: a fix in one copy gets forgotten in the others

Use Kustomize overlays or Helm values files instead.

**Q66.** How does Kustomize avoid duplication?

**Answer:** A base plus per-environment overlays that reference it and layer a small patch

A fix to base/deployment.yaml propagates to every environment.

**Q67.** What does kubectl apply -k overlays/prod do?

**Answer:** Applies the Kustomize output of that overlay directly

kubectl kustomize previews the merged output.

**Q68.** How does Helm achieve the same idea?

**Answer:** Layered values files, e.g. -f values.yaml -f values-prod.yaml, overriding only what differs

It is a common base plus a small per-environment diff.

### [08 · Jobs &amp; CronJobs](level-2/08-jobs-cronjobs.md)

**Q69.** Why isn't a Deployment the right tool for a one-time batch task?

**Answer:** It keeps a process running indefinitely and restarts it forever

A Job runs a Pod to successful completion, with retries, then stops.

**Q70.** What restartPolicy does a Job's Pod template need?

**Answer:** Never or OnFailure; Always makes no sense

backoffLimit caps retries.

**Q71.** What do completions and parallelism mean in a parallel Job?

**Answer:** Total successful completions needed, and how many Pods may run at once

10 completions with 3 in parallel.

**Q72.** What does completionMode: Indexed give each Pod?

**Answer:** A stable index (0..N-1) for partitioning work

NonIndexed just needs any N successful completions.

### [09 · Debugging &amp; Troubleshooting Workloads](level-2/09-debugging-troubleshooting.md)

**Q73.** What is the recommended order for debugging a workload?

**Answer:** get, describe (Events), logs, exec, then debug if the container won't start

Random kubectl get commands waste time.

**Q74.** What does CrashLoopBackOff usually mean, and where do you look?

**Answer:** The container starts then exits repeatedly; check logs --previous

--previous shows the crashed container's logs, not the current restart's.

**Q75.** What does ImagePullBackOff usually mean?

**Answer:** Wrong image name or tag, or missing private registry auth

The Events section shows the pull failure.

**Q76.** Why does describe's Events section often show a problem before logs do?

**Answer:** In many cases the container never even started, so there are no logs yet

Scheduling failures, image pulls, probes and OOM kills surface there.

### [10 · Project — Ingress-Fronted Helm App](level-2/10-project-helm-ingress-app.md)

**Q77.** Why does the project use a single-replica StatefulSet for the database?

**Answer:** So the PVC survives Pod rescheduling with a stable name (data-shop-db-0); Deployment semantics don't fit irreplaceable state

Multi-replica StatefulSets are covered in Level 3.

**Q78.** How does the chart differ between dev and prod?

**Answer:** values-prod.yaml overrides replicas, storage size, host and TLS

For example prod has 4 replicas, 20Gi and TLS on.

**Q79.** What does volumeClaimTemplates do?

**Answer:** Creates a PVC for each StatefulSet replica

It requests storage with ReadWriteOnce from the storageClassName.

**Q80.** Which components does the project combine?

**Answer:** Volumes, Ingress, Helm and multi-environment config

Modules 01, 02, 06 and 07.

## Level 3 · Advanced

### [01 · StatefulSets &amp; DaemonSets](level-3/01-statefulsets-daemonsets.md)

**Q81.** When do you need a StatefulSet instead of a Deployment?

**Answer:** When each replica needs a stable name, DNS entry and PVC, as in databases or brokers

Deployment Pods are interchangeable.

**Q82.** What name do StatefulSet Pods get?

**Answer:** &lt;statefulset&gt;-&lt;ordinal&gt;, e.g. postgres-0, postgres-1

They also get stable DNS names through the headless Service.

**Q83.** What happens when postgres-1 is deleted?

**Answer:** A Pod named postgres-1 is recreated and reattaches to the same PVC

Each Pod has its own PVC from volumeClaimTemplates.

**Q84.** Why does a StatefulSet need a headless Service (clusterIP: None)?

**Answer:** It is required for StatefulSet DNS

serviceName must match it.

### [02 · RBAC &amp; Service Accounts](level-3/02-rbac-service-accounts.md)

**Q85.** What are the two separate questions every API request answers?

**Answer:** Authentication (who are you) then authorization (what can you do)

RBAC is the standard authorization mechanism.

**Q86.** Which ServiceAccount does a Pod use if none is specified?

**Answer:** The namespace's default one, which has essentially zero permissions

ServiceAccounts give identity to Pods, not people.

**Q87.** What does a Role plus RoleBinding grant?

**Answer:** Permissions only within one namespace

log-reader can read Pods in prod but nothing in staging.

**Q88.** When do you use ClusterRole and ClusterRoleBinding?

**Answer:** For cluster-wide resources like nodes or reusable permissions

The example grants node-reader to a monitoring ServiceAccount.

### [03 · Network Policies](level-3/03-network-policies.md)

**Q89.** What is the default Kubernetes networking behavior?

**Answer:** Any Pod can reach any other Pod on any port, across namespaces

A NetworkPolicy restricts this.

**Q90.** When is a NetworkPolicy actually enforced?

**Answer:** Only if the CNI plugin implements it, such as Calico or Cilium

On a CNI that ignores it, the object is inert.

**Q91.** What does an empty podSelector with no rules mean in the default-deny policy?

**Answer:** Select every Pod and allow nothing

Policies are additive, so layer allow policies on top.

**Q92.** What does allow-web-to-api permit?

**Answer:** Inbound TCP 8080 to app: api Pods only from app: web Pods in the same namespace

Everything else stays blocked by default-deny.

### [04 · Autoscaling (HPA &amp; VPA Concepts)](level-3/04-autoscaling-hpa-vpa.md)

**Q93.** What does HPA change versus VPA?

**Answer:** HPA changes replicas; VPA changes resource requests/limits

Horizontal means more copies; vertical means more CPU/memory per copy.

**Q94.** What does HPA need to read CPU utilization?

**Answer:** The metrics-server add-on (or a custom metrics adapter)

It serves the metrics.k8s.io API from kubelet stats.

**Q95.** What does averageUtilization: 70 mean?

**Answer:** Keep average CPU across replicas at 70% of each Pod's CPU request

Without a request, the utilization percentage has no denominator.

**Q96.** What is an External metric type used for?

**Answer:** A metric with no direct Pod association, like a queue depth

Type Pods reads a per-Pod metric through an adapter such as Prometheus Adapter.

### [05 · Observability: Metrics &amp; Logging](level-3/05-observability-metrics-logging.md)

**Q97.** Why isn't kubectl logs enough at scale?

**Answer:** It doesn't survive Pod deletion, doesn't aggregate across replicas and gives no historical trends

Production observability needs metrics, logs and traces.

**Q98.** How does Prometheus collect metrics?

**Answer:** It pulls: discovers targets via the Kubernetes API and hits each target's /metrics endpoint

The app exposes Prometheus text format using a client library.

**Q99.** What does kube-state-metrics add over metrics-server?

**Answer:** It turns object state (replica counts, Pod phase, PVC binding) into Prometheus metrics, while metrics-server exposes only current CPU/memory

For example kube_deployment_status_replicas_available.

**Q100.** What do the prometheus.io/scrape annotations do?

**Answer:** Mark a Pod as a scrape target and say which port and path to use

The relabel config keeps only Pods annotated scrape: "true".

### [06 · Probes &amp; Health at Scale](level-3/06-probes-at-scale.md)

**Q101.** What is the thundering herd problem with readiness probes?

**Answer:** If every replica's probe depends on a shared downstream, a brief blip can make all replicas unready at once, so the Service loses all backends

It's strictly worse than one Pod being briefly unavailable.

**Q102.** What is the recommended fix?

**Answer:** Keep readiness shallow and handle downstream failures with retries/circuit breakers in app code

A deep /ready that pings the database couples the fleet's availability to it.

**Q103.** What does a PodDisruptionBudget protect against?

**Answer:** Voluntary disruption such as node drains and cluster upgrades evicting too many replicas

Probes handle involuntary failures like crashes.

**Q104.** Why jitter initialDelaySeconds?

**Answer:** So replica probes don't all fire in the same tick, which matters as probes add load at fleet scale

It isn't a native field; stagger via Helm or rely on rolling-update timing.

### [07 · Multi-Cluster &amp; Multi-Tenancy Concepts](level-3/07-multi-cluster-multi-tenancy.md)

**Q105.** How do multi-tenancy and multi-cluster differ?

**Answer:** Multi-tenancy is isolation within a cluster; multi-cluster is blast-radius and locality across clusters

They are often combined but are different decisions.

**Q106.** Which building blocks make up soft multi-tenancy?

**Answer:** Namespaces, RBAC, ResourceQuota and NetworkPolicy

They suit trusted tenants like internal teams.

**Q107.** Why is namespace isolation 'soft'?

**Answer:** All tenants share the same kernel, kubelet and control plane, so a container escape could cross boundaries

It stops accidental interference and enforces fairness.

**Q108.** What is the stronger option for untrusted tenants?

**Answer:** A separate cluster or a virtual cluster per tenant

vcluster gives each tenant its own API server on a shared node pool.

### [08 · Admission Control &amp; Policy Enforcement](level-3/08-admission-control-policy.md)

**Q109.** What is the order for write requests to the API server?

**Answer:** Authentication → authorization → admission control → persisted to etcd

Admission is content-aware, unlike RBAC.

**Q110.** How does admission differ from RBAC?

**Answer:** RBAC asks whether an identity may do a verb; admission asks whether this specific object with these fields is acceptable

RBAC has no concept of object content.

**Q111.** What gap do dynamic admission webhooks fill?

**Answer:** Custom logic beyond the plugins compiled into the API server

ResourceQuota, LimitRange and DefaultStorageClass are built-in plugins.

**Q112.** What does a validating webhook respond with?

**Answer:** allowed: true or false with an optional reason, in an AdmissionReview

The example rejects Pods without memory limits.

### [09 · Cluster Networking Deep Dive (CNI)](level-3/09-cluster-networking-cni.md)

**Q113.** What does the Kubernetes networking model require?

**Answer:** Every Pod gets its own IP, can reach any other Pod's IP without NAT, and sees its own IP as others do

Kubernetes defines the model; a CNI plugin implements it.

**Q114.** How do two Pods on the same node communicate?

**Answer:** Via veth pairs attached to a Linux bridge such as cni0, with no routing needed

One veth end is inside the Pod's namespace as eth0.

**Q115.** What does an overlay (VXLAN) do for cross-node traffic?

**Answer:** Encapsulates Pod packets inside UDP packets addressed node-to-node, working on any underlying network at the cost of overhead

The routed alternative, e.g. Calico with BGP, avoids encapsulation.

**Q116.** Which problem is kube-proxy's job?

**Answer:** Pod-to-Service

The four problems also include external traffic via NAT, Ingress or LoadBalancer.

### [10 · Project — Resilient Stateful Service](level-3/10-project-resilient-stateful-service.md)

**Q117.** What do topologySpreadConstraints with maxSkew 1 and DoNotSchedule accomplish?

**Answer:** The scheduler refuses to put a second replica on a node already holding one when enough nodes exist, making 'survives node loss' true

A StatefulSet alone doesn't guarantee spread.

**Q118.** What does the PodDisruptionBudget add to the project?

**Answer:** The service can be safely drained for maintenance without evicting too many replicas at once

It protects availability during voluntary disruption.

**Q119.** Which modules does the project combine?

**Answer:** StatefulSets, RBAC, NetworkPolicy, HPA, observability and PDBs

It builds a resilient 3-node key-value store.

**Q120.** What does podManagementPolicy: OrderedReady do?

**Answer:** Creates Pods one at a time in order, waiting for each to be Ready

It's the default StatefulSet behavior.

## Level 4 · Master

### [01 · Operators &amp; CRDs](level-4/01-operators-crds.md)

**Q121.** What does a CRD let you do?

**Answer:** Register a new resource type with its own schema and validation, without recompiling the API server

You get kubectl get/apply support immediately.

**Q122.** What is an Operator?

**Answer:** A CRD paired with a custom controller that encodes operational knowledge as code

Backups, failover and upgrade steps are examples.

**Q123.** What comes from the CRD's schema with no custom code?

**Answer:** kubectl get, RBAC rules and OpenAPI validation that rejects a missing required field

The controller adds behavior; the CRD alone is just API.

**Q124.** What does scope: Namespaced mean for the CRD?

**Answer:** Its resources live inside a namespace

orders-db was created in prod.

### [02 · GitOps Patterns (ArgoCD/Flux Concepts)](level-4/02-gitops-argocd-flux.md)

**Q125.** How does the GitOps pull model differ from the push model?

**Answer:** A controller inside the cluster watches Git and reconciles, so nothing outside needs write credentials to the cluster

CI only needs push access to Git.

**Q126.** What do prune: true and selfHeal: true do together in ArgoCD?

**Answer:** Delete resources removed from Git and revert manual cluster edits, enforcing Git as the only source of change

Without them ArgoCD only reports OutOfSync.

**Q127.** What is an ArgoCD Application?

**Answer:** A custom resource pointing at a Git path and a destination cluster/namespace

repoURL, targetRevision and path define the source.

**Q128.** How is Flux structured differently?

**Answer:** A set of composable CRD-driven controllers like GitRepository and Kustomization instead of one Application object

It's Flux v2 (Flux CD).

### [03 · Cluster Upgrades &amp; Maintenance](level-4/03-cluster-upgrades-maintenance.md)

**Q129.** What is the mandated upgrade order?

**Answer:** Control plane first, then nodes, one minor version at a time

For example 1.27 → 1.28 → 1.29, never 1.27 → 1.29 directly.

**Q130.** Which component is the ceiling for version skew?

**Answer:** kube-apiserver

No other component may run a newer minor version than it.

**Q131.** What does kubectl cordon node-3 do?

**Answer:** Marks the node unschedulable so no new Pods are placed there

kubectl drain then evicts existing Pods, respecting PodDisruptionBudgets.

**Q132.** How do control-plane Pods restart after kubeadm upgrade apply?

**Answer:** kubelet watches the static Pod manifests directory and restarts them automatically

Static Pods aren't normal API objects you restart via kubectl.

### [04 · Security Hardening (Pod Security Standards)](level-4/04-security-hardening.md)

**Q133.** What are the three Pod Security Standards levels?

**Answer:** Privileged, Baseline and Restricted

They replace the deprecated PodSecurityPolicy and are enforced by the in-tree Pod Security Admission controller.

**Q134.** How do enforce, warn and audit differ?

**Answer:** enforce rejects non-compliant Pods; warn and audit let you roll out a stricter policy gradually

Apply warn/audit first, then flip enforce once nothing new violates it.

**Q135.** How is a Pod Security Standard applied to a namespace?

**Answer:** With pod-security.kubernetes.io labels on the Namespace

For example pod-security.kubernetes.io/enforce: restricted.

**Q136.** Which are Restricted requirements?

**Answer:** Run as non-root, drop all capabilities, forbid privilege escalation and require a seccomp profile

The compliant Pod spec sets runAsNonRoot, allowPrivilegeEscalation: false and capabilities drop ALL.

### [05 · Image Scanning &amp; Supply Chain Security](level-4/05-image-scanning-supply-chain.md)

**Q137.** What are the supply chain's three attack surfaces?

**Answer:** Build-time, distribution and runtime

Scanning addresses build-time; signing and provenance address distribution and runtime.

**Q138.** What turns Trivy scanning from a dashboard nobody reads into a real gate?

**Answer:** Wiring it into CI with --exit-code 1 on CRITICAL/HIGH so the build fails

A scan that only reports gets ignored under deadline pressure.

**Q139.** What does keyless Cosign signing tie the signature to?

**Answer:** An OIDC identity (e.g. a specific GitHub Actions workflow), instead of a long-lived private key

It uses Fulcio and the Rekor transparency log.

**Q140.** What does the Kyverno ClusterPolicy do here?

**Answer:** Enforces at admission time that images are signed (or scanned) before Pods are allowed

It blocks unverified images at the admission step.

### [06 · Designing Production-Grade Cluster Architecture](level-4/06-production-cluster-architecture.md)

**Q141.** Why does etcd need an odd number of members?

**Answer:** So a majority (quorum) is always well-defined: 3 tolerate 1 failure, 5 tolerate 2

It's a Raft-based cluster.

**Q142.** Why spread 3 control-plane nodes across 3 AZs?

**Answer:** A single AZ outage still leaves 2 members, a quorum, so the cluster keeps writing

Each AZ holds one etcd member.

**Q143.** What do taints and tolerations achieve for a GPU node pool?

**Answer:** Ordinary workloads don't land on expensive specialized nodes; only Pods that tolerate the taint do

The Pod also uses a nodeSelector for the pool.

**Q144.** What questions does a production design answer?

**Answer:** Control plane count and AZs, etcd deployment and backup, worker node pooling and cluster-wide policy

The earlier levels sidestepped these for simplicity.

### [07 · Disaster Recovery &amp; Backup Strategies](level-4/07-disaster-recovery-backup.md)

**Q145.** Why does restoring etcd alone not recover a corrupted database?

**Answer:** etcd holds declared API objects, while application data lives on PV storage outside etcd

Both halves need independent, tested backup and restore procedures.

**Q146.** Why must etcd snapshots ship off-node immediately?

**Answer:** A snapshot only on the control-plane disk dies with that disk in a real disaster

The cron job copies each snapshot to S3 or GCS.

**Q147.** What does etcdctl snapshot save do?

**Answer:** Writes a point-in-time snapshot of etcd to a file

etcdctl snapshot status shows its hash, revision, key count and size.

**Q148.** How does the restore procedure make etcd use the restored data?

**Answer:** Point the etcd static Pod manifest at the restored data-dir; kubelet restarts etcd against it

Restoring is a full replace, not a merge.

### [08 · Cost Optimization &amp; Capacity Planning](level-4/08-cost-optimization-capacity.md)

**Q149.** What drives most of a cluster's cost?

**Answer:** Requested, not used, resources

Nodes are provisioned and billed to satisfy requests whether or not Pods use them.

**Q150.** A Pod requests 500m CPU but uses 45m. What is the problem?

**Answer:** The scheduler reserved about 10x the CPU actually needed, which sits idle and billed

Over-requesting is the largest source of Kubernetes waste.

**Q151.** Why start VPA with updateMode: "Off"?

**Answer:** It only reports recommendations without evicting Pods, so a human applies new requests deliberately

Auto mode evicts and recreates Pods, which is disruptive for restart-intolerant workloads.

**Q152.** What does Cluster Autoscaler react to?

**Answer:** Unschedulable (Pending) Pods, scaling a node group up

The log shows a node group scaled from 6 to 7.

### [09 · Platform Engineering on Kubernetes](level-4/09-platform-engineering.md)

**Q153.** What two failure modes does platform engineering avoid?

**Answer:** Every team reinventing its own boilerplate, or a central team becoming a ticket queue

It provides a self-service golden path.

**Q154.** What does a Crossplane Composite Resource Definition (XRD) let a platform team do?

**Answer:** Define a simplified developer-facing API that expands into the real Kubernetes and cloud resources

A developer writes a small DatabaseClaim; a Composition provisions the real resources.

**Q155.** What does a developer submit in the lesson's example?

**Answer:** A DatabaseClaim with size and engine

The claim yields a connection Secret.

**Q156.** What is a golden path?

**Answer:** A self-service route that gives developers safe, consistent infrastructure

Developers don't touch the underlying cloud details.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q157.** What does the capstone introduce?

**Answer:** No new mechanisms: an integration exercise showing how Modules 01–09 fit together for orders-api

It combines GitOps, security, supply chain, topology, DR, cost and platform surface.

**Q158.** What do topologySpreadConstraints on the zone key do for orders-api?

**Answer:** Spread replicas evenly across the three AZs

maxSkew 1 with DoNotSchedule enforces even spread.

**Q159.** What access does team-orders get?

**Answer:** The edit ClusterRole, scoped to the orders namespace only

Every Pod they deploy must also satisfy the restricted Pod Security level.

**Q160.** What does the Kyverno policy require in the orders namespace?

**Answer:** Signed images only, enforced at admission

It uses verifyImages with validationFailureAction: Enforce.

## Mixed questions across all levels

**Q161.** You deploy a bare Pod for a real workload and its node dies. What happens?

**Answer:** Nothing recreates it; use a Deployment for self-healing

A bare Pod has no controller behind it.

**Q162.** You change a Deployment's image tag. What does the Deployment controller do?

**Answer:** Creates a new ReplicaSet and gradually shifts replicas from the old one (rolling update)

The old ReplicaSet is kept at zero for rollback.

**Q163.** A rollout is broken. Which command returns to the previous revision?

**Answer:** kubectl rollout undo deployment/web

rollout undo reverts to the previous revision; --to-revision picks a specific one.

**Q164.** Pods get new IPs on each recreation, but other services must reach them reliably. What do you create?

**Answer:** A Service with a label selector

A Service gives a stable IP and DNS name in front of matching Pods.

**Q165.** You need one external domain with path-based routing to ten HTTP services and TLS. What do you use?

**Answer:** An Ingress plus an Ingress controller

Ingress describes host/path routing; the controller implements it.

**Q166.** A database password must not be baked into the image or stored in plain config. What do you use?

**Answer:** A Secret consumed as an env var or mounted file

Secrets are for sensitive data (base64-encoded, optionally encrypted at rest).

**Q167.** A container is deadlocked but its process is still running. Which probe restarts it?

**Answer:** Liveness probe

A failing liveness probe makes kubelet kill and restart the container.

**Q168.** A Pod is temporarily unable to serve because its database connection dropped. What is the right probe response?

**Answer:** Failing readiness removes it from Service endpoints without a restart

Readiness protects traffic without needless restarts.

**Q169.** A Pod shows OOMKilled. What happened?

**Answer:** It exceeded its memory limit and the kernel OOM killer terminated it

Memory limits are enforced by killing; CPU limits by throttling.

**Q170.** A Pod is stuck Pending with 'Insufficient cpu'. What does it tell you?

**Answer:** No node has enough unrequested CPU for its requests

The scheduler compares requests to allocatable capacity.

**Q171.** A pod shows CrashLoopBackOff. Which command shows why the previous container crashed?

**Answer:** kubectl logs &lt;pod&gt; --previous

--previous shows the crashed container's logs.

**Q172.** You want to run a database migration once, retry on failure and stop. What kind of object?

**Answer:** A Job (restartPolicy Never or OnFailure)

A Job runs to completion; backoffLimit caps retries.

**Q173.** A database needs stable Pod names, stable DNS and its own PVC per replica. What do you use?

**Answer:** A StatefulSet with a headless Service

StatefulSets give each replica a stable identity.

**Q174.** You want a service account to read Pod logs only in the prod namespace. What do you create?

**Answer:** A Role and a RoleBinding in prod

Role + RoleBinding grants permissions within one namespace.

**Q175.** By default any Pod can reach any other Pod. What is the recommended first NetworkPolicy for a namespace?

**Answer:** Default-deny, then allow-list the traffic you need

Policies are additive; start with deny-all.

**Q176.** You want replicas to scale with CPU, keeping average utilization at 70%. What do you use, and what must be set?

**Answer:** HorizontalPodAutoscaler, with accurate CPU requests and metrics-server

Utilization is a percentage of the Pod's CPU request.

**Q177.** You must stop a node drain from evicting too many replicas of your app at once. What protects availability?

**Answer:** A PodDisruptionBudget

PDBs limit voluntary disruptions.

**Q178.** You want to reconcile cluster state continuously from Git, reverting manual edits. What approach and settings?

**Answer:** GitOps (ArgoCD) with prune and selfHeal

The pull model corrects drift automatically.

**Q179.** You must reject Pods that run as root or lack resource limits across a namespace. Which mechanisms apply?

**Answer:** Pod Security labels (restricted) and admission policy

Admission control checks the object's content.

---

More free practice: the [Kubernetes Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
