---
description: "Security Hardening (Pod Security Standards) — enforce rejects non-compliant Pods at admission time; warn and audit let you roll a stricter policy out…"
---

# 04 · Security Hardening (Pod Security Standards)

!!! note "Not run against a live cluster"
    Manifests and admission behavior below follow the documented Pod
    Security Admission spec; not executed against a live cluster in this
    session.

## Pod Security Standards: three named levels

Kubernetes defines three built-in policy levels (replacing the deprecated
PodSecurityPolicy), enforced by the in-tree **Pod Security Admission**
controller — no separate installation required since 1.25:

- **Privileged** — unrestricted; effectively opts out of enforcement.
- **Baseline** — blocks known privilege escalations while staying broadly
  compatible (no privileged containers, no host namespaces, no arbitrary
  host path mounts).
- **Restricted** — the hardened, least-privilege tier: requires running as
  non-root, drops all Linux capabilities by default, forbids privilege
  escalation, requires a seccomp profile.

## Enforcing a standard via namespace labels

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.29
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

```bash
kubectl apply -f payments-namespace.yaml
kubectl run test-priv --image=nginx --privileged -n payments
# Error from server (Forbidden): pods "test-priv" is forbidden:
# violates PodSecurity "restricted:v1.29": privileged (container "test-priv"
# must not set securityContext.privileged=true), ...
```

`enforce` rejects non-compliant Pods at admission time; `warn` and `audit`
let you roll a stricter policy out gradually — apply `warn`/`audit` first,
watch audit logs / `kubectl` warnings for a rollout period, then flip
`enforce` once nothing new is violating it.

## A Restricted-compliant Pod spec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api
  namespace: payments
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: api
      image: registry.example.com/api:v1.4.2
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

Every field here maps to a specific Restricted requirement:
`runAsNonRoot`/`runAsUser` block UID 0, `allowPrivilegeEscalation: false`
blocks `setuid`-style escalation inside the container,
`readOnlyRootFilesystem` plus an `emptyDir` for `/tmp` lets a process still
write scratch files without a writable root filesystem, and `capabilities.drop:
["ALL"]` removes every Linux capability (including ones a root-run process
would otherwise retain) unless explicitly added back.

## Least-privilege RBAC alongside Pod-level hardening

Pod Security Standards constrain what a Pod's *runtime* can do; RBAC
constrains what callers can do to the *API*. Both matter — a
Restricted-compliant Pod running under a ServiceAccount with
cluster-admin RBAC is still a serious blast-radius problem if compromised.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: payments
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-pod-reader
  namespace: payments
subjects:
  - kind: ServiceAccount
    name: api
    namespace: payments
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl auth can-i list secrets --as=system:serviceaccount:payments:api -n payments
# no
```

## Worked example: rolling out Restricted without breaking things

```bash
# 1. Warn/audit only, on the existing namespace, no enforcement yet:
kubectl label namespace payments \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted --overwrite

# 2. Deploy normally; check kubectl output and audit logs for violations:
kubectl apply -f deployment.yaml -n payments
# Warning: would violate PodSecurity "restricted:latest": ... (allowPrivilegeEscalation != false)

# 3. Fix the flagged manifests, redeploy, confirm zero warnings, then enforce:
kubectl label namespace payments pod-security.kubernetes.io/enforce=restricted --overwrite
```

## How It Actually Works

- **Pod Security Admission is a built-in *validating* admission
  webhook-equivalent compiled into the API server, not a controller
  watching objects after the fact.** It runs synchronously in the
  admission chain for every Pod create/update request, reading the
  effective policy from the target namespace's labels at request time —
  this is why enforcement is instantaneous and atomic (a violating Pod
  create is rejected outright with a 403, it never briefly exists) but
  also why it only ever evaluates Pod *creation*, not Pods already running
  when a namespace's label is tightened (existing Pods are grandfathered
  until they're next recreated).
- **The three policy levels are implemented as static, versioned rule
  sets baked into the API server binary, not CRDs or ConfigMaps.** Each
  Kubernetes minor version ships an updated definition of what
  "restricted" means (new fields get added as the API evolves); the
  `enforce-version` label pins evaluation to a specific version's rule set
  so that a cluster upgrade doesn't retroactively start rejecting
  previously-compliant Pods just because the definition of "restricted"
  gained a new check.
- **`capabilities.drop: ["ALL"]` interacts with the container runtime's
  default capability set, not with a Kubernetes-level default.** The
  container runtime (containerd/CRI-O) grants a runtime-specific default
  Linux capability set (traditionally similar to Docker's list — things
  like `NET_BIND_SERVICE`, `CHOWN`) unless told otherwise; Kubernetes'
  `securityContext.capabilities` is passed straight through to the
  runtime's OCI spec generation, so `drop: ["ALL"]` really means "start
  from an empty capability set," and any capability actually required
  (e.g. binding to port 80 as non-root, needing `NET_BIND_SERVICE`) must
  be explicitly re-added via `add`.
- **`seccompProfile: RuntimeDefault` delegates the actual syscall
  filtering to the container runtime's shipped seccomp profile via a BPF
  program loaded by the kernel, not by Kubernetes.** Kubernetes only
  records the *intent* on the Pod spec; containerd/CRI-O translates that
  into a call that loads a seccomp BPF filter into the kernel for that
  process at container-start time, blocking a curated list of dangerous
  syscalls (like `ptrace`, unless explicitly needed) — the enforcement
  boundary is the kernel, which is why a wrong or missing seccomp profile
  can't be fixed by anything short of restarting the container.

## Exercise

Create a namespace with `pod-security.kubernetes.io/enforce: baseline`,
then attempt to schedule a Pod with `hostNetwork: true` and confirm it's
rejected with a specific violation message. Fix the manifest, then bump
the namespace label to `restricted` and iterate on the same Pod spec
(adding `runAsNonRoot`, dropping capabilities, setting a seccomp profile)
until it's admitted cleanly.
