# Rollback

StatefulSets support revision history and rollback of the Pod template. When the Pod template is changed, such as updating the container image, a `RollingUpdate` replaces Pods according to StatefulSet ordering rules.

## Update Order

With the default `RollingUpdate` strategy, Pods are updated in **reverse ordinal order**, from the highest ordinal to the lowest:

1. `pod-4`
2. `pod-3`
3. `pod-2`
4. `pod-1`
5. `pod-0`

With `OrderedReady`, Kubernetes updates one Pod at a time and waits for the updated Pod to become Ready before proceeding to the next Pod.

If an update causes problems, the StatefulSet can be rolled back:

```bash
kubectl rollout undo statefulset <name>
```

Check the available revisions with:

```bash
kubectl rollout history statefulset <name>
```

Rollback restores the Pod template to a previous revision and starts another StatefulSet update. Because StatefulSets preserve ordering and wait for readiness, a failed or stuck Pod can prevent the rollback from progressing and may require manual intervention.

> Rollback changes the Pod template; it does not automatically restore or roll back data stored in persistent volumes.

---

# HPA with StatefulSets

A Horizontal Pod Autoscaler (HPA) can scale a StatefulSet based on metrics such as:

* CPU utilization
* Memory utilization
* Custom metrics
* External metrics

For example, if a StatefulSet has 3 replicas and HPA increases the desired replica count to 5:

```text
mysql-0
mysql-1
mysql-2
mysql-3  ← created
mysql-4  ← created
```

When scaling down from 5 to 3 replicas:

```text
mysql-4  ← removed
mysql-3  ← removed

mysql-0
mysql-1
mysql-2
```

StatefulSet scaling preserves ordinal identity. With `OrderedReady`, scale-up occurs in increasing ordinal order and the controller waits for each Pod to become Ready before creating the next one. Scale-down occurs in reverse ordinal order.

With `podManagementPolicy: Parallel`, Pods can be created or terminated in parallel, so these ordering guarantees do not apply to Pod creation/deletion.

---

# Probes

**Liveness**, **readiness**, and **startup** probes work the same way in StatefulSets as they do in Deployments.

- ## Liveness Probe

    Detects whether a container is unhealthy or stuck.

    If the liveness probe repeatedly fails, Kubernetes restarts the container.

- ## Readiness Probe

    Determines whether a Pod is ready to receive traffic.

    With `podManagementPolicy: OrderedReady`, readiness also affects StatefulSet progression:

    ```text
    pod-0 → Running → Not Ready
                 ↓
            pod-1 waits
    ```

    Once `pod-0` becomes Ready:

    ```text
    pod-0 → Running + Ready
                 ↓
            pod-1 can start
    ```

    Therefore, a failing readiness probe can prevent the StatefulSet from progressing to the next ordinal.

    With `podManagementPolicy: Parallel`, readiness does not block creation of other Pods.

- ## Startup Probe

    Used for applications that require significant time to start.
    
    While the startup probe has not succeeded, Kubernetes does not run the liveness and readiness probes. This prevents slow-starting applications from being restarted or considered unready prematurely.

---

## Summary

* **Rollback:** StatefulSets support revision history and `rollout undo`; rollback follows StatefulSet update ordering and can be blocked by unhealthy Pods.
* **Scaling:** HPA can scale StatefulSets; StatefulSet identity and storage mappings are preserved.
* **OrderedReady:** Pods are created sequentially and readiness controls progression.
* **Parallel:** Pods can be created and terminated in parallel.
* **Probes:** Liveness, readiness, and startup probes work the same way as in Deployments, but readiness has an additional role in `OrderedReady` StatefulSets.
* **Persistent data:** StatefulSet rollback does not roll back data stored in PVCs.
