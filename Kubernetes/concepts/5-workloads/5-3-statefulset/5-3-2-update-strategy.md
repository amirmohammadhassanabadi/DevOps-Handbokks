# StatefulSet Update Strategy

A StatefulSet uses an **update strategy** to control how existing Pods are updated when the Pod template changes, such as changing a container image.

StatefulSets support two update strategies:

* `RollingUpdate` — default
* `OnDelete`

## RollingUpdate

`RollingUpdate` automatically updates StatefulSet Pods in **reverse ordinal order**, from the highest ordinal to the lowest.

For example:

```text
mysql-0
mysql-1
mysql-2
```

The update proceeds:

```text
mysql-2
   ↓
wait until updated Pod is Ready
   ↓
mysql-1
   ↓
wait until updated Pod is Ready
   ↓
mysql-0
```

The Pods retain their existing identity and storage:

```text
mysql-2 → same identity → same PVC
mysql-1 → same identity → same PVC
mysql-0 → same identity → same PVC
```

Example:

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
```

### Partitioned RollingUpdate

`RollingUpdate` supports `partition`, which limits automatic updates to Pods whose ordinal is greater than or equal to the partition value.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2
```

With:

```text
replicas: 3
```

only:

```text
mysql-2 → updated
mysql-1 → unchanged
mysql-0 → unchanged
```

This can be useful for controlled rollouts and testing a new version on selected StatefulSet members.

However, StatefulSet partitioning is **not a full canary deployment mechanism**. It selects Pods by ordinal rather than controlling traffic. It cannot provide rules such as:

* Send exactly 5% of traffic to the new version.
* Send only internal users to the new version.
* Route requests based on headers.
* Gradually increase traffic from 1% → 10% → 50%.

For traffic-based canary deployments, mechanisms such as Ingress controllers, service meshes, or progressive-delivery tools are more appropriate.

### Important

StatefulSet uses:

```yaml
spec:
  updateStrategy:
```

while Deployment uses:

```yaml
spec:
  strategy:
```

These are different API fields because StatefulSet and Deployment have different update semantics.

---

## Update Process

Suppose the StatefulSet contains:

```text
mysql-0
mysql-1
mysql-2
```

and the container image is changed.

With `RollingUpdate`, Kubernetes starts with the highest ordinal:

```text
mysql-2
```

The StatefulSet controller updates that Pod and waits for the updated Pod to become Ready before proceeding to the next ordinal.

Then:

```text
mysql-1
```

and finally:

```text
mysql-0
```

The exact termination/recreation mechanics should not be thought of as simply:

```text
delete old Pod → create new Pod
```

The important guarantee is the **ordered replacement of StatefulSet identities** while preserving their associated storage.

Each recreated Pod keeps its StatefulSet identity and can reuse its existing PVC:

```text
mysql-2 → data-mysql-2
mysql-1 → data-mysql-1
mysql-0 → data-mysql-0
```

---

## OnDelete

With `OnDelete`, changing the StatefulSet Pod template does **not** automatically update existing Pods.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

For example:

```text
1. Update StatefulSet image
        ↓
2. Existing Pods keep old image
        ↓
3. Delete mysql-2 manually
        ↓
4. StatefulSet recreates mysql-2
        ↓
5. New Pod uses the updated image
```

```bash
kubectl delete pod mysql-2
```

This gives the operator manual control over when each Pod is updated.

> ### Container Restart vs Pod Recreation
> 
> Changing the StatefulSet template does not change the specification of an existing Pod.
> 
> If a container crashes, the kubelet can restart the container according to the Pod's `restartPolicy`. The restarted container uses the **existing Pod specification**, so it continues using the old image.
> 
> The new StatefulSet template is applied when the Pod itself is recreated.

---

## RollingUpdate vs OnDelete

| Strategy                      | Behavior                                            | Typical Use                        |
| ----------------------------- | --------------------------------------------------- | ---------------------------------- |
| `RollingUpdate`               | Automatically updates Pods in reverse ordinal order | Normal StatefulSet upgrades        |
| `RollingUpdate` + `partition` | Automatically updates only selected ordinals        | Controlled/partial rollouts        |
| `OnDelete`                    | Pods update only when individually deleted          | Manual operator-controlled updates |

## StatefulSet vs Deployment Update Model

```text
Deployment
    ↓
ReplicaSet-based rollout
    ↓
New and old Pods can temporarily coexist
    ↓
Designed for interchangeable Pods


StatefulSet
    ↓
StatefulSet controller
    ↓
Reverse-ordinal Pod updates
    ↓
Stable identity + stable storage
```

StatefulSet updates are designed around preserving **Pod identity, storage mapping, and ordering**, which are important for stateful applications.
