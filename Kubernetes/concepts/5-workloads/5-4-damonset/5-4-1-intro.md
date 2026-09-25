# DaemonSet

A **DaemonSet** is a Kubernetes controller that ensures a Pod runs on every node that matches its scheduling requirements.

Unlike a Deployment, a DaemonSet does not use a `replicas` field. The desired number of Pods is determined by the nodes that match the DaemonSet's scheduling constraints.

DaemonSets are mainly used for **node-level infrastructure workloads** that need to run on individual nodes.

## Why DaemonSets Exist

Some workloads need one instance on each node because they interact with the node itself rather than serving normal application traffic.

Common examples include:

* **Log collection:** Fluent Bit, Fluentd, Filebeat
* **Monitoring:** Prometheus Node Exporter
* **Networking:** CNI components
* **Security:** node-level security or intrusion-detection agents
* **Storage:** storage-related node daemons
* **Device plugins:** GPU or other hardware device plugins

These workloads may need access to the node's filesystem, network, devices, or kernel-level information.

---

## How DaemonSet Works

When a DaemonSet is created:

1. The DaemonSet controller watches the nodes in the cluster.
2. It determines which nodes match the DaemonSet's scheduling requirements.
3. It ensures that a DaemonSet Pod exists on each matching node.
4. When a new matching node joins, a Pod is created on that node.
5. When a node stops matching the requirements or is removed, the corresponding DaemonSet Pod is removed.

For example, if 4 nodes match the DaemonSet:

```text
Node 1 → Pod
Node 2 → Pod
Node 3 → Pod
Node 4 → Pod
```

The DaemonSet therefore maintains one Pod per matching node during normal operation.

> `maxSurge` can temporarily allow more than one DaemonSet Pod on a node during an update.

---

## Example DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor

  template:
    metadata:
      labels:
        app: node-monitor

    spec:
      containers:
        - name: node-monitor
          image: prom/node-exporter
          ports:
            - containerPort: 9100
```

If four nodes match the DaemonSet's scheduling requirements, Kubernetes will maintain four DaemonSet Pods.

---

## Scheduling Behavior

A DaemonSet can run on all matching nodes or only on a selected subset.

Scheduling can be controlled using:

* `nodeSelector`
* `nodeAffinity`
* `tolerations`

### nodeSelector

For example, to run the DaemonSet only on nodes labeled `monitoring=true`:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        monitoring: "true"
```

Only nodes with this label will receive the DaemonSet Pod.

### Tolerations

DaemonSets can also run on nodes that normally reject Pods through taints.

For example, control-plane nodes are commonly tainted. A DaemonSet can run there by adding an appropriate toleration:

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

Therefore, it is more accurate to think of a DaemonSet as:

```text
One Pod per node that matches its scheduling requirements
```

rather than simply:

```text
One Pod per worker node
```

---

## DaemonSet vs Deployment

| Feature         | DaemonSet                                 | Deployment                     |
| --------------- | ----------------------------------------- | ------------------------------ |
| Pod count       | One per matching node                     | Controlled by `replicas`       |
| Scaling         | Changes automatically with matching nodes | Manual or HPA                  |
| Main use case   | Node-level infrastructure                 | Application workloads          |
| Scheduling      | Targets matching nodes                    | Scheduler distributes replicas |
| Pod identity    | Node-oriented                             | Replica-oriented               |
| Update strategy | `RollingUpdate` / `OnDelete`              | `RollingUpdate` / `Recreate`   |

---

## Key Characteristics

* Runs one Pod on each matching node.
* Automatically reacts to node changes.
* Does not use `replicas`.
* Commonly used for node-level infrastructure.
* Supports `RollingUpdate` and `OnDelete`.
* Supports rollout history and rollback.
* Supports liveness, readiness, and startup probes.

---

# Update Strategy

The DaemonSet update strategy determines how existing Pods are replaced when the Pod template changes, such as when the container image is updated.

DaemonSets support two update strategies:

* `RollingUpdate` — default
* `OnDelete`

---

## RollingUpdate

`RollingUpdate` automatically replaces old DaemonSet Pods with new ones in a controlled manner.

```yaml
updateStrategy:
  type: RollingUpdate
```

By default, `maxUnavailable` is `1` and `maxSurge` is `0`. Therefore, without additional configuration, the update normally replaces Pods while allowing at most one DaemonSet Pod to be unavailable at a time.

- ### maxUnavailable

    `maxUnavailable` controls how many DaemonSet Pods may be unavailable during the update.

    For example:

    ```yaml
    updateStrategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 2
    ```

    With 10 matching nodes, up to two DaemonSet Pods can be unavailable during the update.

    A larger value can make the rollout faster, but increases the number of nodes temporarily without an available DaemonSet Pod.

    ---

- ### maxSurge

    `maxSurge` allows Kubernetes to create an updated DaemonSet Pod before removing the old Pod on a node.

    ```yaml
    updateStrategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 1
    ```

    Normally, with `maxSurge: 0`, the old Pod is replaced without intentionally keeping an additional updated Pod on the same node.

    With `maxSurge`, the update can temporarily look like:

    ```text
    Node 1
    ├── node-agent-v1
    └── node-agent-v2
    ```

    Once the new Pod becomes available, the old Pod can be removed:

    ```text
    Node 1
    └── node-agent-v2
    ```

    This can reduce service gaps during updates, which is particularly useful for node-level agents such as monitoring and logging components.

    However, surge temporarily increases resource consumption on the affected node. If the new Pod cannot become available, the overlap can persist, so resource-intensive DaemonSets should use `maxSurge` carefully.

### maxUnavailable vs maxSurge

| Setting          | Controls                                                     |
| ---------------- | ------------------------------------------------------------ |
| `maxUnavailable` | How many DaemonSet Pods may be unavailable                   |
| `maxSurge`       | How many nodes may temporarily run an additional updated Pod |

---

## OnDelete

With `OnDelete`, updating the DaemonSet template does not automatically replace existing Pods.

```yaml
updateStrategy:
  type: OnDelete
```

Existing Pods continue running with the old configuration.

To update a specific node:

1. Update the DaemonSet template.
2. Delete the existing DaemonSet Pod.
3. The DaemonSet controller creates a replacement Pod using the new template.

For example:

```bash
kubectl delete pod <pod-name>
```

The replacement Pod is created on the same node and uses the updated DaemonSet template.

This gives administrators manual control over when each node receives the new version.

---

## RollingUpdate vs OnDelete

| Feature             | RollingUpdate          | OnDelete                  |
| ------------------- | ---------------------- | ------------------------- |
| Update process      | Automatic              | Manual                    |
| Existing Pods       | Automatically replaced | Remain unchanged          |
| New version applied | During rollout         | When old Pod is deleted   |
| Default             | Yes                    | No                        |
| Main use case       | Normal rollouts        | Controlled/manual updates |

---

# Probes

DaemonSet Pods use the same probe mechanisms as other Kubernetes Pods:

* `livenessProbe`
* `readinessProbe`
* `startupProbe`

There is no special probe mechanism for DaemonSets.

- ### Liveness Probe

    Determines whether the container is still healthy.

    If the liveness probe repeatedly fails, kubelet restarts the container.

- ### Readiness Probe

    Determines whether the Pod is ready to receive traffic.

    If readiness fails, the Pod is considered not ready and is normally removed from Service endpoints.

- ### Startup Probe

    Used for slow-starting applications.

    Until the startup probe succeeds, Kubernetes does not run the liveness and readiness probes.

Example:

```yaml
spec:
  containers:
    - name: node-agent
      image: my-agent:v1

      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
```

Probes behave the same way regardless of whether the Pod is managed by a Deployment, StatefulSet, or DaemonSet.

---

# Rollback

DaemonSets support rollout history and rollback.

Unlike Deployments, DaemonSets do not use ReplicaSets. Instead, DaemonSet revisions are stored as **ControllerRevision** objects.

You can view the rollout history with:

```bash
kubectl rollout history daemonset <name>
```

To inspect a specific revision:

```bash
kubectl rollout history daemonset <name> --revision=2
```

To roll back to a specific revision:

```bash
kubectl rollout undo daemonset <name> --to-revision=2
```

To roll back to the most recent previous revision:

```bash
kubectl rollout undo daemonset <name>
```

The rollback changes the DaemonSet Pod template back to the selected revision and starts another rollout.

Monitor the rollback with:

```bash
kubectl rollout status daemonset <name>
```

DaemonSet revisions are stored using `ControllerRevision`, and the default `revisionHistoryLimit` is 10. A rollback creates a new revision rather than moving the revision number backward.

> Rollback changes the DaemonSet Pod template. It does not restore data or external state maintained by the application.

---

# Summary

A **DaemonSet** ensures that a Pod runs on every node matching its scheduling requirements.

### Main concepts

* **One Pod per matching node**
* Automatically reacts to node joins and scheduling changes
* No `replicas` field
* Commonly used for logging, monitoring, networking, security, storage, and device plugins
* `nodeSelector`, `nodeAffinity`, and `tolerations` control where Pods run
* `RollingUpdate` is the default update strategy
* `OnDelete` provides manual update control
* `maxUnavailable` controls update availability
* `maxSurge` allows temporary overlap between old and new Pods
* Supports liveness, readiness, and startup probes
* Supports rollout history and `kubectl rollout undo`
* Uses `ControllerRevision` rather than ReplicaSets for revision history
