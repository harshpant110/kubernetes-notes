# Storage and Persistent Volumes

## Why Is Storage Needed in Kubernetes?

Containers commonly use a writable filesystem layer that is tied to the container's lifecycle. If a container is replaced, data written only to that layer may be lost.

Kubernetes workloads sometimes need data to persist beyond the lifecycle of an individual container or Pod.

Examples include:

- Database files.
- Uploaded files.
- Application-generated reports.
- Shared files used by multiple containers.
- Configuration and other application data.

Kubernetes provides volume mechanisms that allow Pods to access storage. Some volumes are temporary, while others can preserve data independently of a Pod.

**Important:** A PersistentVolumeClaim does not automatically guarantee that data survives every failure. Persistence depends on the underlying storage system, its configuration, and the volume's lifecycle policies.

--------------------------------------------------------------------

## What Is a Kubernetes Volume?

A **Volume** is a storage resource made available to containers in a Pod.

Volumes are defined in the Pod specification and mounted into containers at specified paths.

Containers within the same Pod can mount the same volume when they need to share files.

For example:

```text
                 Pod
                  |
          +-------+-------+
          |               |
      Container A     Container B
          |               |
          +-------+-------+
                  |
              Shared Volume
```

The containers can access the shared files through their respective mount paths, subject to filesystem permissions and mount configuration.

A volume's lifecycle depends on its volume type. Not every Kubernetes volume provides persistent storage.

--------------------------------------------------------------------

## `emptyDir` Volume

An `emptyDir` volume is created when a Pod is assigned to a node. It is available to containers in that Pod and can be used for temporary data or shared files.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-demo
spec:
  containers:
    - name: writer
      image: busybox:stable
      command: ["sh", "-c", "echo hello > /data/message.txt; sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: reader
      image: busybox:stable
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
  volumes:
    - name: shared-data
      emptyDir: {}
```

In this example, both containers mount the same volume at `/data`. The writer creates a file that the reader can also access.

Important properties:

- The volume is scoped to the Pod.
- Containers in the Pod can share its files.
- Its contents survive a container restart while the Pod remains on the node.
- Its contents are removed when the Pod is removed from the node.

`emptyDir` is useful for scratch space, temporary files, and sharing data between containers. It is not a replacement for persistent storage.

--------------------------------------------------------------------

## What Is a PersistentVolume?

A **PersistentVolume (PV)** is a Kubernetes resource representing storage made available to a cluster.

It can be provisioned manually by an administrator or dynamically through a StorageClass and a supported storage provisioner.

A PV exists independently of an individual Pod. However, its actual lifecycle and durability depend on the underlying storage and reclaim policy.

A PV describes storage properties such as:

- Capacity.
- Access modes.
- Storage type.
- Reclaim policy.
- Storage class association.

--------------------------------------------------------------------

## What Is a PersistentVolumeClaim?

A **PersistentVolumeClaim (PVC)** is a request for storage made by a user or workload.

A PVC can specify requirements such as:

- Requested storage capacity.
- Access modes.
- StorageClass.
- Other supported volume requirements.

Kubernetes attempts to bind the PVC to a suitable PV. If dynamic provisioning is configured, a new PV may be created to satisfy the claim.

The Pod references the PVC rather than needing to manage the underlying storage resource directly.

```text
                  Pod
                   |
                   v
      PersistentVolumeClaim (PVC)
                   |
                   v
          PersistentVolume (PV)
                   |
                   v
          Underlying Storage
```

This separation allows workloads to request storage without depending on every implementation detail of the storage infrastructure.

--------------------------------------------------------------------

## PersistentVolume YAML Example

The following is an illustrative PV manifest using a node-local directory. It is suitable only for understanding the resource structure and is not a production storage design.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data/example
    type: DirectoryOrCreate
```

Important fields:

- `capacity.storage` specifies the advertised storage capacity.
- `volumeMode: Filesystem` requests filesystem-style volume use.
- `accessModes` describes supported access semantics.
- `persistentVolumeReclaimPolicy` determines what happens to the PV after its claim is released.
- `storageClassName` associates the PV with a StorageClass name.
- `hostPath` references a path on a particular node.

**Warning:** `hostPath` uses node-local storage. It is tied to the node's filesystem and is not automatically replicated, backed up, or available on other nodes. For production workloads, use an appropriate storage solution with suitable durability and recovery characteristics.

--------------------------------------------------------------------

## PersistentVolumeClaim YAML Example

The following PVC requests one gibibyte of storage from a PV or provisioner matching its requirements.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```

The claim requests:

- `ReadWriteOnce` access semantics.
- A StorageClass named `manual`.
- At least `1Gi` of requested capacity.

The request must match a suitable PV or be supported by a provisioner for the claim to bind.

In this example, the `example-pv` and `example-pvc` manifests use the same StorageClass name and compatible capacity and access modes. Kubernetes still needs to evaluate all binding requirements before the claim becomes bound.

--------------------------------------------------------------------

## Using a PVC in a Pod

A Pod can mount a PVC as a volume.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: persistent-storage-demo
spec:
  containers:
    - name: app
      image: busybox:stable
      command: ["sh", "-c", "echo persistent-data > /data/message.txt; sleep 3600"]
      volumeMounts:
        - name: app-storage
          mountPath: /data
  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: example-pvc
```

In this example:

- `volumes` defines a volume backed by the PVC.
- `claimName` identifies the existing PVC.
- `volumeMounts` mounts the volume inside the container.
- The application writes a file to `/data`.

The PVC must exist in the same namespace as the Pod and must be usable for the requested workload.

A PVC can remain bound when a Pod is deleted, allowing another Pod to use the claim when the storage and access requirements permit it.

--------------------------------------------------------------------

## What Is a StorageClass?

A **StorageClass** describes a class of storage that can be dynamically provisioned.

It lets a workload request storage without requiring an administrator to manually create a PV for every claim.

A StorageClass may define:

- A provisioner.
- Parameters for the storage system.
- Reclaim policy.
- Volume binding behaviour.
- Expansion-related settings, where supported.

Example structure:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: example-storage
provisioner: example.com/placeholder
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

The provisioner shown here is a placeholder, not a real storage driver. This manifest is illustrative and will not provision storage unless replaced with a supported provisioner and appropriate configuration.

### Dynamic Provisioning

When a PVC requests a StorageClass that supports dynamic provisioning, Kubernetes can request that the provisioner create suitable storage and a corresponding PV.

The process is generally:

1. A workload or user creates a PVC.
2. The PVC requests storage capacity and a StorageClass.
3. A compatible provisioner creates or allocates storage.
4. Kubernetes binds the claim to the resulting PV.
5. A Pod references the PVC and mounts the volume.

A cluster may have a default StorageClass. If a PVC omits `storageClassName`, Kubernetes may use the default StorageClass when one is configured. Setting `storageClassName: ""` explicitly disables default StorageClass selection.

--------------------------------------------------------------------

## PersistentVolume Access Modes

Access modes describe how a volume can be mounted by nodes or workloads.

| Access mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | The volume can be mounted read-write by a single node at a time. Multiple Pods on that same node may be able to use it, depending on the storage and workload configuration. |
| `ReadOnlyMany` (ROX) | The volume can be mounted read-only by multiple nodes, when supported. |
| `ReadWriteMany` (RWX) | The volume can be mounted read-write by multiple nodes, when supported. |
| `ReadWriteOncePod` (RWOP) | The volume can be mounted read-write by a single Pod, subject to Kubernetes version and storage-driver support. |

**Important:** Access-mode support depends on the storage implementation. An access mode is not a guarantee that every storage backend supports the requested behaviour.

--------------------------------------------------------------------

## PersistentVolume Reclaim Policies

The reclaim policy determines what happens to a PV after its PVC is deleted and the claim is released.

### Retain

- The PV and underlying storage are retained for manual recovery or cleanup.
- An administrator may need to handle the released volume before it can be reused.
- This can help protect data from automatic deletion.

### Delete

- Kubernetes and the storage provisioner can delete the PV and its associated storage asset when the claim is released.
- The exact behaviour depends on the provisioner and storage implementation.

The older `Recycle` policy is deprecated and should not be used for new storage designs.

**Important:** Deleting a PVC can have serious data consequences. Understand the PV reclaim policy and the storage provider's behaviour before deleting a claim.

--------------------------------------------------------------------

## Inspecting Kubernetes Storage

Use these commands to inspect storage resources:

```bash
# List PersistentVolumes
kubectl get pv

# List PersistentVolumeClaims
kubectl get pvc

# List StorageClasses
kubectl get storageclass

# Inspect a specific PVC
kubectl describe pvc example-pvc

# Inspect a specific PV
kubectl describe pv example-pv

# Inspect a Pod that uses storage
kubectl describe pod persistent-storage-demo
```

PersistentVolumes are cluster-scoped resources. PersistentVolumeClaims are namespaced resources.

If a PVC remains `Pending`, inspect its events and check whether a compatible PV or dynamic provisioner is available.

--------------------------------------------------------------------

## Common Storage Troubleshooting

### PVC Is Pending

Possible causes:

- No suitable PV exists.
- The requested capacity or access modes cannot be satisfied.
- The requested StorageClass does not exist.
- Dynamic provisioning is unavailable or failing.
- Volume binding is waiting for a suitable Pod scheduling decision.

Inspect the claim:

```bash
kubectl describe pvc <pvc-name>
kubectl get pv
kubectl get storageclass
```

### Pod Cannot Mount a Volume

Possible causes:

- The referenced PVC does not exist in the Pod's namespace.
- The claim is not bound or cannot be used by the Pod.
- The storage backend is unavailable.
- Node-level mount requirements cannot be satisfied.
- File permissions prevent the application from accessing the mounted directory.

Inspect the Pod and relevant events:

```bash
kubectl describe pod <pod-name>
kubectl get pvc
kubectl get pv
```

### Data Appears to Be Missing

Check whether the Pod used a temporary volume such as `emptyDir`, whether the correct PVC was mounted, and whether the workload is using the expected storage backend.

Also check the storage system's recovery, backup, and reclaim policies. Persistence is not the same as backup.

--------------------------------------------------------------------

## Key Takeaways

- Volumes provide storage to containers in a Pod.
- `emptyDir` is temporary and tied to the Pod's lifecycle.
- A PersistentVolume represents cluster storage.
- A PersistentVolumeClaim requests storage for a workload.
- A StorageClass enables storage provisioning based on defined parameters.
- Pods reference PVCs to mount persistent storage.
- Access modes describe supported volume-mount semantics.
- Reclaim policies determine what happens to storage when a claim is released.
- `hostPath` is node-local and requires special care.
- Persistent storage does not replace backups, disaster recovery, or storage security.

--------------------------------------------------------------------

## Revision Questions

1. Why might data in a container's writable layer be lost?
2. What is the difference between a Volume, a PersistentVolume, and a PersistentVolumeClaim?
3. What is the purpose of a StorageClass?
4. How does dynamic provisioning work?
5. What is the difference between `emptyDir` and persistent storage?
6. What does `ReadWriteOnce` mean?
7. What is the difference between the `Retain` and `Delete` reclaim policies?
8. Why should `hostPath` be used carefully?
9. Which commands help troubleshoot a PVC in the `Pending` state?
10. Why is persistent storage not a substitute for backups?

--------------------------------------------------------------------