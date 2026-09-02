# 08 · ConfigMaps & Secrets

!!! note "Not run against a live cluster"
    Manifests and output below follow documented ConfigMap/Secret behavior;
    not executed against a live cluster in this session.

## Why externalize configuration

Baking configuration (database URLs, feature flags, API keys) into a
container image means rebuilding the image for every environment
(dev/staging/prod) or every config change. Kubernetes gives you two objects
to externalize this instead:

- **ConfigMap** — non-sensitive configuration data (URLs, feature flags,
  whole config files).
- **Secret** — the same mechanism, intended for sensitive data (passwords,
  tokens, TLS certs), stored base64-encoded and treated specially by
  Kubernetes (excluded from `kubectl describe` output, encryptable at rest).

Both can be consumed by Pods the same two ways: as **environment
variables** or as **mounted files**.

## Creating a ConfigMap

Declaratively:

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  greeting.txt: |
    Hello from a ConfigMap-mounted file!
    This can be any multi-line text content.
```

Imperatively (handy for quick experiments):

```bash
kubectl create configmap web-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-file=greeting.txt
```

```bash
kubectl apply -f configmap.yaml
kubectl get configmap web-config -o yaml
```

## Consuming a ConfigMap as environment variables

```yaml
# deployment-with-config.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27-alpine
          envFrom:
            - configMapRef:
                name: web-config
          env:
            - name: SINGLE_VALUE
              valueFrom:
                configMapKeyRef:
                  name: web-config
                  key: LOG_LEVEL
```

`envFrom.configMapRef` injects **every key** in the ConfigMap as an env var
(`APP_ENV`, `LOG_LEVEL`, and `greeting.txt` — note that non-identifier keys
like a filename with a dot are technically injected but awkward as env var
names, which is why file-like content is usually mounted as a volume
instead, below). `env[].valueFrom.configMapKeyRef` pulls in a **single,
specifically named** key.

```bash
kubectl exec deploy/web -- env | grep -E 'APP_ENV|LOG_LEVEL|SINGLE_VALUE'
# APP_ENV=production
# LOG_LEVEL=info
# SINGLE_VALUE=info
```

## Consuming a ConfigMap as mounted files

```yaml
spec:
  containers:
    - name: web
      image: nginx:1.27-alpine
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: web-config
```

Every key in the ConfigMap becomes a file in `/etc/config` — `APP_ENV`,
`LOG_LEVEL`, and `greeting.txt`, each with the key's value as file content.
This is the natural fit for mounting a whole config file (e.g. an nginx
`.conf`, a `.properties` file) rather than exploding it into individual env
vars.

```bash
kubectl exec deploy/web -- cat /etc/config/greeting.txt
# Hello from a ConfigMap-mounted file!
# This can be any multi-line text content.
```

## Creating a Secret

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=        # base64 of "admin"
  password: c3VwZXJzZWNyZXQ= # base64 of "supersecret"
```

`data` values must be **base64-encoded** (not encrypted — this is important:
base64 is trivially reversible, so a Secret's confidentiality depends on
Kubernetes RBAC restricting who can read it, plus encryption-at-rest for
etcd, not on the encoding itself). Generate the values:

```bash
echo -n 'admin' | base64          # -> YWRtaW4=
echo -n 'supersecret' | base64    # -> c3VwZXJzZWNyZXQ=
```

Or use `stringData` for plain-text convenience (Kubernetes base64-encodes it
for you on write):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: supersecret
```

Imperatively:

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret
```

## Consuming a Secret

Identical shape to ConfigMaps, just `secretKeyRef` / `secret` instead:

```yaml
spec:
  containers:
    - name: web
      image: nginx:1.27-alpine
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      volumeMounts:
        - name: db-creds-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: db-creds-volume
      secret:
        secretName: db-credentials
```

```bash
kubectl get secret db-credentials -o jsonpath='{.data.username}' | base64 -d
# admin
```

`kubectl describe secret db-credentials` deliberately does **not** print
values, only key names — a small guardrail against accidental exposure on
screen-shares/logs.

## Important caveats

- Base64 is **encoding, not encryption**. Anyone who can `kubectl get
  secret -o yaml` can decode it. Real secret hygiene needs RBAC (Level 3)
  restricting read access, and often an external secrets manager (Vault,
  cloud KMS-backed secret stores) integrated via a CSI driver or operator —
  out of scope for this level, but worth knowing exists.
- Env-var Secrets are visible via `kubectl exec ... -- env` and can leak
  into crash dumps/logs more easily than mounted-file Secrets — many teams
  prefer mounting Secrets as files for anything sensitive.
- Updating a ConfigMap/Secret does **not** automatically restart Pods that
  already mounted it as env vars (env vars are set once, at container
  start). Mounted **files** do eventually update in-place (kubelet syncs
  periodically), but your app still needs to notice and reload — a common
  pattern is a sidecar or app-level file-watcher for hot-reload.

## Exercise

Create the `web-config` ConfigMap and `db-credentials` Secret above. Deploy
a single-replica Deployment that consumes `LOG_LEVEL` as an env var,
mounts the whole ConfigMap at `/etc/config`, and mounts the Secret at
`/etc/secrets`. `exec` into the Pod and verify: `env | grep LOG_LEVEL`,
`cat /etc/config/greeting.txt`, and `cat /etc/secrets/username` (you should
see the *decoded* plain-text value as file content, since Kubernetes decodes
Secrets automatically when mounting — only the API representation is
base64).
