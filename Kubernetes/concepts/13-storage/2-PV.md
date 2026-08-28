# PersistentVolume (PV)
A PersistentVolume (PV) is a Kubernetes object that represents a piece of persistent storage available to the cluster. The storage can come from different backends.
- an NFS share
- a Ceph disk
- an AWS EBS volume
- a SAN disk
- a local SSD

With static provisioning, a cluster administrator creates PVs. With dynamic provisioning, a StorageClass and CSI driver can automatically create the PV when a PVC requests storage.

#### `PV` example:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /data
    server: 10.0.0.5
```

The PV represents 10 GiB of storage provided by the NFS export on 10.0.0.5. A Pod can ultimately use this storage through a PVC.

`PV` = the supply of storage.

## `PersistentVolume` Lifecycle States:

A `PersistentVolume (PV)` moves through several lifecycle states depending on whether it is claimed and released.

- ### Available

    The `PV` exists but is not bound to any `PVC`.

    - `PV` → Available
    - `PVC` → none

    Kubernetes can bind the PV to a compatible PVC based on its requirements and configuration. (`size`, `accessModes`, `storageClass`, ... ).

- ### Bound
    The `PV` is successfully bound to a `PersistentVolumeClaim`.

    `PV` → Bound → `PVC`

    The `PV` is now reserved for that `PVC` only. Pods use the `PVC`, not the `PV` directly.

- ### Released
    The PVC was deleted, but the PV still exists.

    #### Important detail:

    - The PV still contains data from the old claim
    - Kubernetes does not automatically make it Available again
    
    What happens next depends on the **reclaim policy**.

    #### Why the Released state exists?
    To protect data. If a `PVC` is deleted, Kubernetes assumes the volume may still contain important data, so it does not immediately give it to another claim.

- ### Failed 

    The PV encountered a failure during an operation such as reclaiming the volume. The exact recovery depends on the storage backend and the cause of the failure.

    #### Example:

    - The storage system failed to delete the underlying disk
    - Something went wrong during recycling/cleanup

### Visaul Flow:

```
Available
↓
Bound
↓
PVC deleted
↓
Released
↓
Reclaim policy determines what happens
    ├── Delete → PV/storage is deleted
    └── Retain → PV/storage is retained for manual handling
```

> The PV lifecycle phase describes the state of the PV object; it does not necessarily describe the exact state of the underlying storage system.