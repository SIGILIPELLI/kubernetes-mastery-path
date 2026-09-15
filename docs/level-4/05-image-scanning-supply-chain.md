---
description: "Image Scanning & Supply Chain Security — Image scanning addresses (1); image signing/provenance addresses (2) and (3)."
---

# 05 · Image Scanning & Supply Chain Security

!!! note "Not run against a live cluster"
    Tool output and admission behavior below follow documented Trivy/Cosign/
    Kyverno behavior; not executed against a live cluster in this session.

## The supply chain has three attack surfaces

1. **Build-time** — vulnerable base images, vulnerable dependencies baked
   into the image.
2. **Distribution** — a compromised registry, or a tag silently
   repointed to different content after review.
3. **Runtime** — a cluster pulling and running an image whose contents no
   one actually verified matches what was reviewed.

Image scanning addresses (1); image signing/provenance addresses (2) and
(3).

## Scanning images with Trivy

```bash
trivy image registry.example.com/api:v1.4.2
# Total: 14 (CRITICAL: 1, HIGH: 4, MEDIUM: 6, LOW: 3)
# CVE-2024-XXXX  openssl  CRITICAL  3.0.2  fixed: 3.0.13

trivy image --severity CRITICAL,HIGH --exit-code 1 registry.example.com/api:v1.4.2
# exits non-zero if CRITICAL/HIGH found — designed to fail a CI pipeline
```

Wiring this into CI as a required check (`exit-code 1` on
CRITICAL/HIGH) is what turns scanning from a dashboard nobody reads into
an actual gate — a scan that only reports, without failing the build,
gets ignored under deadline pressure.

## Signing images with Cosign (Sigstore)

```bash
cosign sign --key cosign.key registry.example.com/api:v1.4.2
# Pushes a detached signature as a separate OCI artifact alongside the image

cosign verify --key cosign.pub registry.example.com/api:v1.4.2
# Verification for registry.example.com/api:v1.4.2 --
# The following checks were performed on each of these signatures:
# - The cosign claims were validated
# - The signatures were verified against the specified public key
```

Keyless signing (Sigstore's Fulcio/Rekor) ties the signature to an OIDC
identity (e.g. "this exact GitHub Actions workflow, on this exact repo,
built this image") instead of a long-lived private key that itself
becomes a thing to protect:

```bash
cosign sign registry.example.com/api:v1.4.2   # no --key: keyless flow via OIDC + Fulcio + Rekor transparency log
cosign verify --certificate-identity-regexp '.*github.com/example/api.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  registry.example.com/api:v1.4.2
```

## Enforcing signature/scan policy at admission time with Kyverno

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "registry.example.com/*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

```bash
kubectl run unsigned --image=registry.example.com/api:v1.4.9-unsigned
# Error from server: admission webhook "validate.kyverno.svc" denied the request:
# image verification failed for registry.example.com/api:v1.4.9-unsigned:
# .attestors[0].entries[0].keys: failed to verify signature
```

This is the step that actually closes the loop: scanning and signing at
build time are advisory unless something at deploy time refuses to run an
image that wasn't verifiably produced by the pipeline you trust.

## Worked example: an enforced pipeline

```text
1. CI builds image, tags it with the Git SHA.
2. trivy image --exit-code 1 --severity CRITICAL,HIGH  -> fails build on critical CVEs
3. cosign sign (keyless, tied to the CI OIDC identity)  -> signs the passing image
4. Push to registry.
5. Deploy manifest references the image by digest, not by mutable tag:
     image: registry.example.com/api@sha256:9f2a...
6. Kyverno ClusterPolicy verifies the Cosign signature at admission time.
7. Any image that skipped steps 2-3 (or was retagged after the fact) is
   rejected by step 6, regardless of who or what tries to deploy it.
```

Pinning by digest (step 5) matters independent of signing: a mutable tag
like `:v1.4.2` can be repointed at the registry to different content after
review without changing the manifest at all; a digest reference cannot be
silently swapped.

## How It Actually Works

- **Vulnerability scanners work from SBOM-equivalent package
  manifests extracted from image layers, not from executing any code in
  the image.** Trivy walks each layer's filesystem diff, identifies
  installed package databases (`dpkg`/`rpm`/`apk` status files, language
  lockfiles like `package-lock.json`/`go.sum` baked into the image), and
  matches package name+version against a vulnerability database (mirrored
  from NVD and vendor advisories) — this is why scan results are only as
  current as the last database update and why a scan can miss a
  vulnerability in a statically-linked binary with no package-manager
  record at all.
- **Cosign's keyless signing anchors trust in a short-lived certificate
  and a public transparency log, not a stored secret.** Fulcio issues a
  10-minute X.509 certificate binding the signer's OIDC identity (e.g. a
  specific GitHub Actions workflow run) to a freshly generated keypair
  used only for that one signature; Rekor then records the signature in
  an append-only, publicly auditable Merkle-tree log. Verification later
  checks the certificate chain plus a Rekor inclusion proof rather than
  trusting a long-lived key file that would itself need protecting and
  rotating.
- **Kyverno's `verifyImages` runs as a mutating-then-validating admission
  webhook pair, not a plain validating check.** It first mutates the Pod
  spec to resolve any mutable tag reference to its immutable digest (so
  what actually gets scheduled is pinned to the exact content that was
  verified, closing the tag-repoint gap), then validates that the
  signature/attestation over that digest checks out against the
  configured attestors — rejecting at this stage means the object is
  never persisted to etcd at all, unlike a scanner that only flags an
  already-running Pod after the fact.
- **A digest reference (`@sha256:...`) is resolved by the kubelet's image
  service via CRI, and the digest itself is a hash of the image
  manifest, not of the raw layer bytes.** This is why two identically-built
  images can have different digests if manifest metadata (like the
  media-type list or an extra annotation) differs, and why "pin by digest"
  is a strictly stronger guarantee than "pin by tag" — the registry cannot
  serve different bytes for the same digest without every puller detecting
  a hash mismatch and refusing the pull.

## Exercise

Pick a public image (e.g. `nginx:1.25`), run `trivy image` against it and
identify the highest-severity CVE reported. Then install Cosign, generate
a local keypair, sign a test image you push to a local registry, and
verify it. Finally write a Kyverno `ClusterPolicy` (or read one from
Kyverno's policy library) that would have rejected an unsigned version of
that same image.
