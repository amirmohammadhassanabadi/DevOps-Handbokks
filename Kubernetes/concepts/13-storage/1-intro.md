## Two Ways Kubernetes Provides Storage

In Kubernetes, volumes are implemented in two main ways:

1. **Built-in (in-tree) volumes**
2. **CSI-based volumes**

### 1. Built-in (In-Tree) Volumes

These are volume sources that Kubernetes supports directly. They do **not** require a separate CSI driver.

Examples:

- `emptyDir`
- `configMap`
- `secret`
- `downwardAPI`
- `projected`
- `hostPath`
- `nfs`
- `local`

These work immediately because Kubernetes already knows how to mount them.

> Kubernetes includes a built-in NFS volume source. You do **not** need an NFS CSI driver merely to use the native `nfs` volume source.
> An NFS CSI driver also exists, but it is a separate implementation. It is useful when you want to use NFS through the CSI / PV / StorageClass model.

#### Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo Hello > /data/message && sleep 3600"]
    volumeMounts:
    - name: temporary-data
      mountPath: /data
  volumes:
  - name: temporary-data
    emptyDir: {}
```

### 1. CSI-Based Volumes
Modern Kubernetes prefers CSI drivers for storage systems. These require installing a driver inside the cluster.    
Examples:
- `AWS EBS`
- `Azure Disk`
- `GCE Persistent Disk`
- `Ceph RBD`
- `CephFS`
- `Longhorn`
- `OpenEBS`
- `vSphere volumes`

Example:

> Pod → Kubernetes PV/PVC → Ceph CSI driver → Ceph cluster (storage engine)

If the CSI driver is not installed, Kubernetes cannot talk to that storage system.

    Kubernetes does not implement external storage systems such as Ceph, EBS, or SANs. For storage systems integrated through CSI, Kubernetes uses the Container Storage Interface (CSI) to communicate with CSI drivers, which handle the storage-specific operations. Kubernetes also provides several built-in volume sources that do not require CSI.

### So lets see which need CSI?

- `emptyDir` → No - simple temp directory on node
- `hostPath` → No - mounts node filesystem
- `local` → No - node disk used via PV
- `nfs` → No, when using the built-in nfs volume source
- `CephFS` → Yes (modern clusters) - requires Ceph CSI driver

### Important modern Kubernetes rule:
Today Kubernetes prefers this architecture:

> PVC → StorageClass → CSI Driver → Storage Engine → PV → Pod

This is the standard production storage pattern.

### Summery:
A CSI driver is required when the chosen storage integration is provided through CSI. Many modern external storage systems use CSI drivers, while Kubernetes also has built-in volume sources such as emptyDir, hostPath, local, and nfs that can be used without a CSI driver.
