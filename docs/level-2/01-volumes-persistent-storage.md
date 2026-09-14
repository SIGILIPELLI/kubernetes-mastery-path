# 01 · Volumes & Persistent Storage

!!! note "Not run against a live cluster"
    Manifests and output below follow documented Volume/PV/PVC behavior; not
    executed against a live cluster in this session.

## The problem: containers are stateless by default

A container's writable filesystem lives inside its container runtime layer.
When the container is restarted (crash, image update, node reschedule), that
layer is discarded and recreated from the image — anything written to it is
gone. Kubernetes **Volumes** attach external storage to a Pod so that data
can survive a container restart, and **PersistentVolumes** let that data
survive even when the *Pod itself* is deleted and rescheduled elsewhere.

## `emptyDir`: scratch space tied to the Pod's lifetime

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: scratch
          mountPath: /cache
  volumes:
    - name: scratch
      emptyDir: {}
```

`emptyDir` is created empty when the Pod is scheduled to a node and deleted
permanently when that Pod is removed from the node — a crash and restart of
the *container* keeps the data (kubelet doesn't recreate the volume, only the
container), but deleting the *Pod* wipes it. It's ideal for sharing files
between containers in the same Pod (a sidecar writing logs another container
tails) or as disposable scratch space.

## `hostPath`: binding to the node's filesystem

```yaml
volumes:
  - name: node-logs
    hostPath:
      path: /var/log/myapp
      type: DirectoryOrCreate
```

`hostPath` mounts a path from the **node's own filesystem** into the Pod.
It's tied to that specific node — if the Pod reschedules elsewhere, the data
doesn't follow it, and any file dropped there is directly visible to
anything else running on that node (a real security concern; Level 3/4
covers restricting it via Pod Security Standards). Legitimate uses are
narrow: node-level agents (log collectors, CNI/CSI plugins) that are
*supposed* to see the host, not general application storage.

## PersistentVolumes and PersistentVolumeClaims: storage that outlives the Pod

Real application data needs storage independent of any one Pod or node. Two
objects split this concern:

- A **PersistentVolume (PV)** represents an actual piece of storage (a cloud
  disk, an NFS export, a local disk) — created by an admin or dynamically
  provisioned.
- A **PersistentVolumeClaim (PVC)** is a request for storage made by a
  workload ("I need 5Gi, ReadWriteOnce") — Kubernetes binds it to a matching
  PV.

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

```yaml
# deployment.yaml (excerpt)
spec:
  template:
    spec:
      containers:
        - name: app
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: data-pvc
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc data-pvc
# NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# data-pvc   Bound    pvc-3f2c1a9e-...                           5Gi        RWO            standard       4s
```

## StorageClasses: dynamic provisioning

Rather than an admin pre-creating PVs by hand, a **StorageClass** tells
Kubernetes *how* to create one on demand when a PVC references it:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs   # or pd.csi.storage.gke.io, disk.csi.azure.com, etc.
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

`volumeBindingMode: WaitForFirstConsumer` delays provisioning until a Pod
using the PVC is actually scheduled, so the volume gets created in the same
zone as the node the Pod lands on (a Pod can't mount a disk provisioned in a
different availability zone).

## Access modes

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | Mounted read-write by a single node at a time |
| `ReadOnlyMany` (ROX) | Mounted read-only by many nodes |
| `ReadWriteMany` (RWX) | Mounted read-write by many nodes (needs NFS/EFS-class storage, not typical block disks) |
| `ReadWriteOncePod` (RWOP) | Mounted read-write by a single *Pod* (stricter than RWO's per-node guarantee) |

Most cloud block storage (EBS, PD, Azure Disk) only supports RWO — this is
why a single PVC generally can't be shared read-write across replicas of a
Deployment; StatefulSets (Level 3) instead give each replica its *own* PVC.

## Worked example: data survives Pod deletion

```bash
kubectl apply -f pvc.yaml
kubectl apply -f deployment.yaml
kubectl exec deploy/app -- sh -c 'echo hello > /var/lib/postgresql/data/marker'

kubectl delete pod -l app=app        # Deployment recreates the Pod
kubectl exec deploy/app -- cat /var/lib/postgresql/data/marker
# hello -- the new Pod mounted the SAME PVC/PV, data survived
```

## How It Actually Works

- **The PV/PVC binding is a two-phase controller dance, not a direct
  mount.** The `PersistentVolumeController` (in kube-controller-manager)
  watches PVCs; when a new one appears unbound, it searches existing
  PVs for one whose capacity, access modes, and StorageClass satisfy the
  claim (static provisioning) or, if `storageClassName` references a
  StorageClass with a provisioner, it does nothing itself and instead the
  external CSI provisioner sidecar (`external-provisioner`, watching PVCs
  via the API server) creates a brand-new PV and its backing cloud volume,
  then the controller binds the two objects by writing each one's UID into
  the other's spec.
- **Actually attaching and mounting is a kubelet + CSI driver job, done
  per-node, at Pod start.** Once a Pod referencing the PVC is scheduled,
  kubelet calls the CSI driver's `NodeStageVolume`/`NodePublishVolume` gRPC
  methods (over a Unix socket) — this is what actually calls the cloud
  provider's API to attach the block device to the node's instance, formats
  it if unformatted, mounts it to a staging directory, then bind-mounts it
  into the container's mount namespace at the path Docker/containerd sets
  up when creating the container.
- **`WaitForFirstConsumer` exists because provisioning and scheduling would
  otherwise race.** Immediate binding mode creates the PV as soon as the PVC
  appears, picking a zone with no knowledge of where the Pod will land;
  `WaitForFirstConsumer` defers PV creation until the scheduler has already
  chosen a node, and passes that node's topology labels to the provisioner
  so the volume is created in a zone the node can actually attach to.
- **`reclaimPolicy` runs on PVC *deletion*, not Pod deletion.** `Delete`
  causes the same controller to call the CSI driver's `DeleteVolume` to
  destroy the backing cloud disk once the PVC is removed; `Retain` leaves
  the PV (and underlying disk) around in a `Released` state requiring
  manual cleanup — deliberately chosen for data an admin doesn't want a
  developer able to destroy with `kubectl delete pvc`.

## Exercise

Create the `standard` StorageClass, the `data-pvc` PVC, and a single-replica
Deployment mounting it at `/data`. Write a file into `/data`, delete the
Pod, and confirm the ReplicaSet's replacement Pod can still read the file
back — because it mounts the same PVC/PV, not a fresh `emptyDir`. Then
compare by repeating the same steps with an `emptyDir` volume instead, and
confirm the file is gone after the Pod is deleted.
