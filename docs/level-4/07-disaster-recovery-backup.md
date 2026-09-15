---
description: "Disaster Recovery & Backup Strategies — Restoring etcd alone gets you back a cluster that declares the right Deployments and PVCs — it does not restore a…"
---

# 07 · Disaster Recovery & Backup Strategies

!!! note "Not run against a live cluster"
    Commands and restore procedures below follow documented etcd/Velero
    behavior; not executed against a live cluster in this session.

## What actually needs backing up

Kubernetes state lives in two very different places, and a DR plan has to
cover both:

1. **etcd** — every API object (Deployments, Services, Secrets, RBAC,
   CRDs — the entire declared cluster state).
2. **Application/PersistentVolume data** — the actual bytes a database or
   stateful workload has written, which lives outside etcd entirely on
   whatever storage backend backs the PVs.

Restoring etcd alone gets you back a cluster that *declares* the right
Deployments and PVCs — it does **not** restore a corrupted database's
rows. Both halves need independent, tested backup and restore procedures.

## Backing up etcd

```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%F-%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

etcdctl snapshot status /backup/etcd-snapshot-2026-09-14-0200.db --write-out=table
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# | 8f2a1c9d |    48213 |       9871 |    112 MB  |
```

```bash
# a cron job on a control-plane node (or a CronJob with hostPath access + etcd certs mounted):
0 */6 * * * etcdctl snapshot save /backup/etcd-$(date +\%F-\%H\%M).db ... && \
            aws s3 cp /backup/etcd-$(date +\%F-\%H\%M).db s3://cluster-backups/etcd/
```

Snapshots must ship off-node immediately (to S3/GCS/etc.) — a snapshot
sitting only on the control-plane node's disk is not a disaster-recovery
backup, it's a copy that dies with the same disk in a real disaster.

## Restoring etcd from a snapshot

```bash
etcdctl snapshot restore /backup/etcd-snapshot-2026-09-14-0200.db \
  --data-dir=/var/lib/etcd-restored \
  --name=control-plane-1 \
  --initial-cluster=control-plane-1=https://10.0.1.5:2380 \
  --initial-advertise-peer-urls=https://10.0.1.5:2380

# stop the kubelet's static-pod-managed etcd, point its manifest at the restored data-dir:
sed -i 's#/var/lib/etcd#/var/lib/etcd-restored#' /etc/kubernetes/manifests/etcd.yaml
# kubelet detects the manifest change and restarts etcd against the restored data
```

Restoring is a **full replace**, not a merge — any object created after
the snapshot was taken is gone once the restore completes. This is why
snapshot frequency (every 6 hours above) directly bounds the recovery
point objective (RPO): a restore from a 6-hour-old snapshot loses up to 6
hours of API object changes.

## Backing up application/PV data with Velero

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket cluster-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1

velero backup create payments-daily \
  --include-namespaces payments \
  --snapshot-volumes \
  --ttl 720h0m0s

velero schedule create payments-nightly \
  --schedule="0 2 * * *" \
  --include-namespaces payments \
  --snapshot-volumes
```

Velero backs up both the namespace's API objects (like etcd, but scoped
and portable across clusters) **and** triggers cloud-provider volume
snapshots for any PVs referenced by those objects (`--snapshot-volumes`)
— covering the second half of the DR problem that a raw etcd snapshot
does not.

## Restoring with Velero (including cross-cluster)

```bash
velero restore create --from-backup payments-daily
velero restore describe payments-daily-20260914020000
# Phase: Completed
# Warnings: 0
# Errors: 0

kubectl get pods -n payments
kubectl get pvc -n payments   # PVCs re-bound to restored volume snapshots
```

Because Velero backups are self-contained (object manifests + volume
snapshot references, stored in object storage independent of the source
cluster), the same backup can restore into a **different** cluster
entirely — the actual DR test for "we lost the whole cluster/region," not
just "we deleted one namespace by mistake."

## Worked example: a full DR runbook

```text
1. etcd snapshots every 6h, shipped to S3, cross-region replicated.
2. Velero backup of all namespaces nightly + on-demand before risky changes,
   snapshotting PVs, shipped to a separate S3 bucket in a second region.
3. Quarterly DR drill: stand up a fresh cluster in the DR region,
   `velero restore create --from-backup <latest>`, verify application
   health checks pass, measure time-to-restore against the target RTO.
4. Document actual RTO/RPO achieved in the drill (not the theoretical
   number) and feed it back into backup frequency decisions.
```

Step 3 is the part most real incidents reveal was skipped — a backup that
has never been restored in anger is a hypothesis, not a DR plan.

## How It Actually Works

- **`etcdctl snapshot save` works by opening etcd's own bbolt (B+tree)
  data file through etcd's internal snapshot API and streaming a
  point-in-time consistent copy, not by simply copying files off disk.**
  This is why it's safe to run against a live, serving etcd member without
  stopping it — etcd's Raft/MVCC storage layer already maintains
  historical revisions internally, and the snapshot mechanism reads a
  single consistent revision through that same interface rather than
  risking a torn read of files being concurrently written.
- **Restore is destructive to member identity, which is why
  `--name`/`--initial-cluster` must be respecified.** A restored etcd data
  directory starts a *brand new* single-member Raft cluster with a fresh
  cluster ID — it cannot simply rejoin the old Raft group's peer set,
  because the old peers' logs and the restored snapshot now disagree about
  history; this is why a full etcd restore, for a multi-member control
  plane, means tearing down and re-initializing every member from the same
  snapshot rather than restoring just one node into the existing quorum.
- **Velero's PV backup depends entirely on the cloud provider's volume
  snapshot API — it does not read PV bytes itself.** The Velero plugin
  for a given cloud calls that cloud's native snapshot API (an EBS
  snapshot, a GCE persistent disk snapshot) referenced from the backed-up
  PVC/PV objects; this means Velero's actual data-consistency guarantee is
  only as good as the underlying snapshot API's guarantee (typically
  crash-consistent, not application-consistent) — a database that needs a
  quiesced/flushed state before snapshotting needs an explicit pre-backup
  hook (`velero backup create --the pre-hook running `.
- **A cross-cluster Velero restore re-resolves storage classes and
  admission-time defaults on the destination cluster, not on the source.**
  Because the restore replays object manifests through the destination
  API server's normal create path, anything the destination cluster's
  admission chain would inject or reject (a different default
  StorageClass, a stricter Pod Security Admission policy) applies to the
  restored objects exactly as it would to any newly created object — a
  restore into a cluster with `restricted` Pod Security enforcement can
  fail on Pods that were perfectly valid in the (less strict) source
  cluster.

## Exercise

On a local kind or minikube cluster, take an etcd snapshot with
`etcdctl snapshot save`, then deliberately delete a namespace and
everything in it with `kubectl delete namespace`. Restore the snapshot
into a fresh data directory and point a test etcd instance at it (or, more
simply, install Velero with a local MinIO backend, back up a namespace,
delete it, and restore from the Velero backup) — confirm the deleted
objects reappear and note how much, if any, state created after the
backup was lost.
