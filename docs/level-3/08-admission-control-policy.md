---
description: "Admission Control & Policy Enforcement — Every write request to the API server passes, in order: authentication (who are you) → authorization (RBAC — are…"
---

# 08 · Admission Control & Policy Enforcement

!!! note "Not run against a live cluster"
    Manifests and output below follow the documented admission-control
    request flow; not executed against a live cluster in this session.

## Where admission sits in a request's lifecycle

Every write request to the API server passes, in order: **authentication**
(who are you) → **authorization** (RBAC — are you allowed to do this verb
on this resource) → **admission control** (should this *specific object*,
as submitted, be allowed, and should it be modified first) → **persisted
to etcd**. RBAC answers "can this identity create Pods"; admission answers
"is *this* Pod, with *these* fields, acceptable" — a completely different,
content-aware check that RBAC has no concept of.

## Built-in admission plugins you've already used

`ResourceQuota` (Module 07), `LimitRange` (Level 2, Module 03), and the
`DefaultStorageClass` plugin (auto-filling `storageClassName` if a PVC
omits it) are all built-in admission plugins compiled into the API server
and enabled via `--enable-admission-plugins`. They run for every relevant
request, unconditionally, with no way to add custom logic beyond what's
compiled in — which is exactly the gap dynamic admission webhooks fill.

## ValidatingAdmissionWebhook: reject non-compliant objects

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: require-resource-limits
webhooks:
  - name: require-limits.policy.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    clientConfig:
      service:
        name: policy-webhook
        namespace: policy-system
        path: /validate-pods
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
        operations: ["CREATE"]
    failurePolicy: Fail
```

The named Service (`policy-webhook`) fronts a Pod that implements the
webhook: the API server sends it an `AdmissionReview` JSON payload
containing the object being created, and the webhook responds `allowed:
true/false` with an optional reason.

```json
// AdmissionReview response (webhook's own logic, e.g. rejecting a Pod with no resource limits)
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "...",
    "allowed": false,
    "status": { "message": "every container must set resources.limits.memory" }
  }
}
```

```bash
kubectl apply -f pod-no-limits.yaml
# Error from server: admission webhook "require-limits.policy.example.com" denied the request:
# every container must set resources.limits.memory
```

## MutatingAdmissionWebhook: rewrite objects before they're stored

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: inject-sidecar
webhooks:
  - name: sidecar-injector.policy.example.com
    admissionReviewVersions: ["v1"]
    sideEffects: None
    clientConfig:
      service: { name: sidecar-injector, namespace: policy-system, path: /mutate }
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
        operations: ["CREATE"]
```

A mutating webhook returns a **JSONPatch** describing edits, applied by the
API server to the object before it's persisted — this is exactly how
service-mesh sidecar injection (Istio, Linkerd) works: every Pod creation
is intercepted and a proxy container is patched in, with no change needed
to any Deployment manifest itself. Mutating webhooks always run **before**
validating ones, so a mutation can itself be checked by subsequent
validation.

## Policy-as-code: OPA Gatekeeper / Kyverno

Hand-writing a webhook server for every rule doesn't scale; policy engines
let you express rules declaratively and run as the webhook implementation
for you.

```yaml
# Kyverno ClusterPolicy example
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-team-label
      match:
        resources: { kinds: ["Pod"] }
      validate:
        message: "every Pod must have a 'team' label"
        pattern:
          metadata:
            labels:
              team: "?*"
```

```bash
kubectl apply -f pod-no-team-label.yaml
# Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:
# policy require-labels/require-team-label fail: validation error: every Pod must have a 'team' label
```

Kyverno/Gatekeeper register **one** webhook with the API server and
dispatch internally to however many declarative policies are loaded — this
is why real clusters run one policy engine rather than dozens of
hand-rolled webhooks: fewer network hops per admission request, and a
single, auditable place to see every active rule.

## Pod Security Admission: the built-in baseline

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
```

Since Kubernetes 1.25, **Pod Security Admission** is a built-in admission
plugin (no separate webhook install needed) enforcing the `privileged`,
`baseline`, or `restricted` Pod Security Standards purely via namespace
labels — Level 4, Module 04 covers what `restricted` actually requires in
depth.

## Worked example: a policy rejects a privileged Pod at creation

```bash
kubectl apply -f privileged-pod.yaml
# Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:
# policy disallow-privileged fail: Privileged mode is disallowed

kubectl get pods
# never created -- the API server never persisted the object at all
```

## How It Actually Works

- **Admission runs synchronously, in the request path, with a strict
  timeout — a slow or down webhook has direct, immediate consequences for
  every matching request.** The API server calls each applicable webhook
  over HTTPS (mutual TLS, via the referenced Service, which the API
  server's in-cluster networking must be able to reach) and blocks
  completing the request until it responds or `timeoutSeconds` (max 30s)
  elapses; `failurePolicy: Fail` then rejects the request outright on
  timeout/error, while `Ignore` lets it through unpoliced — an
  under-provisioned or crash-looping policy webhook with `Fail` set can
  therefore block *all* matching object creation cluster-wide, which is
  why webhook availability itself needs the same production rigor as any
  critical-path service.
- **Multiple webhooks for the same resource/operation all run, and all
  must approve — but mutating webhooks execute serially (each seeing the
  previous one's patch) while validating webhooks run against the final
  post-mutation object.** This ordering guarantee (all mutating, then all
  validating) is why policy engines validate the sidecar-injected version
  of a Pod, not the version the user originally submitted — a validation
  rule checking "does this Pod have a resource limit" will see limits a
  mutating webhook added, correctly, even though the user never wrote them.
- **JSONPatch mutations are diffed and applied by the API server itself,
  not executed as arbitrary code from the webhook.** The webhook's response
  contains a `patch` field (base64-encoded JSONPatch operations) and
  `patchType: JSONPatch`; the API server applies this patch using the
  standard JSONPatch RFC 6902 semantics against the object as it existed
  when the webhook was called — this bounded, declarative patch format is a
  deliberate security boundary: a webhook can only describe *what* to
  change, never directly write to etcd or bypass subsequent validation.
- **Gatekeeper implements its policy language (Rego, via OPA) as a
  library invoked from within its webhook server process — Kubernetes has
  no native concept of Rego or CEL-based policy at the webhook layer
  itself (CEL-based `ValidatingAdmissionPolicy`, a newer built-in
  alternative, is the one exception, compiled and evaluated directly by
  the API server with no webhook round-trip at all).** This is a real
  performance distinction: a `ValidatingAdmissionPolicy` (stable since
  1.30) runs in-process in the API server with no network hop, while a
  Gatekeeper/Kyverno-based policy always costs at least one HTTPS
  round-trip per admission check.

## Exercise

Deploy Kyverno (or write a minimal validating webhook by hand) with a
policy requiring every Pod to declare `resources.limits.memory`. Attempt to
create a Pod without it and confirm the API server rejects the request with
the webhook's message before the object is ever persisted (`kubectl get
pods` shows nothing created). Then add a mutating policy that injects a
default `team: unspecified` label on any Pod missing one, and confirm a
Pod created without that label ends up with it set after creation.
