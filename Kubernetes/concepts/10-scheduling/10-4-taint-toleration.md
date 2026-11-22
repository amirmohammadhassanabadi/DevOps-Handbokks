Taints and tolerations control which Pods are allowed to be scheduled on particular nodes.

They work together:

- **Taints** are applied to nodes.
- **Tolerations** are configured on Pods.

A taint repels Pods that do not tolerate it.

Think of a taint as a restriction applied to a node:

> This node is reserved or restricted. Only Pods with a matching toleration are allowed to pass the scheduling restriction.

A toleration tells Kubernetes:

> This Pod can tolerate the specified taint.

### Taint Format

A taint has three components:

```text
key=value:effect
```

For example:

```text
dedicated=db:NoSchedule
```

This means the node has a taint with:

- Key: `dedicated`
- Value: `db`
- Effect: `NoSchedule`

---

## 9. Taint Effects

Kubernetes supports three main taint effects.

### A. NoSchedule

New Pods that do not tolerate the taint will not be scheduled on the node.

Existing Pods are not automatically evicted merely because this taint is added.

Example:

```bash
kubectl taint node node1 dedicated=db:NoSchedule
```

The node now repels new Pods unless they have a matching toleration.

### B. PreferNoSchedule

The scheduler tries to avoid placing Pods on the node unless other scheduling considerations make it appropriate.

This is a soft restriction, not a guarantee that Pods will avoid the node.

### C. NoExecute

This effect applies to both new and existing Pods.

- New Pods without a matching toleration cannot be scheduled on the node.
- Existing Pods without a matching toleration are evicted.
- Existing Pods with a matching toleration can remain on the node.

This is the strongest taint effect.

### Taint Effect Summary

| Effect | New Pods without toleration | Existing Pods without toleration |
|---|---|---|
| `NoSchedule` | Not scheduled | Not evicted |
| `PreferNoSchedule` | Scheduler tries to avoid node | Not evicted |
| `NoExecute` | Not scheduled | Evicted |

---

## 10. Tolerations

A toleration is configured in the Pod specification.

Example:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "db"
    effect: "NoSchedule"
```

This toleration matches the node taint:

```text
dedicated=db:NoSchedule
```

The Pod is now allowed to be scheduled on that node, provided its other scheduling requirements are satisfied.

### Toleration Operators

Two commonly used operators are:

| Operator | Meaning |
|---|---|
| `Equal` | The key, value, and specified effect must match the taint. |
| `Exists` | The taint must have the specified key; its value is not checked. |

Example using `Exists`:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Exists"
    effect: "NoSchedule"
```

This tolerates `NoSchedule` taints with the key `dedicated`, regardless of their values.

An empty key combined with `Exists` can match all taint keys for the specified effect.

### Complete Example

First, taint the node:

```bash
kubectl taint node node1 special=true:NoSchedule
```

Then create a Pod with a matching toleration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: special-pod
spec:
  tolerations:
    - key: "special"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"

  containers:
    - name: app
      image: nginx
```

This Pod is allowed to run on `node1`.

Without the toleration, the scheduler will not select that node.

### Important: Toleration Does Not Force Scheduling

A toleration only removes the restriction imposed by a matching taint.

It does not tell the scheduler that the Pod must run on that node.

For example, if a Pod tolerates a GPU node's taint, the scheduler may still select another eligible node.

To target a specific node group, combine tolerations with `nodeSelector` or node affinity.

---

## 11. NoExecute and tolerationSeconds

`NoExecute` has an additional field called `tolerationSeconds`.

It specifies how long a Pod can remain on a node after a matching `NoExecute` taint is applied.

Example:

```yaml
tolerations:
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 30
```

Meaning:

- The Pod tolerates the taint temporarily.
- If the matching taint remains, Kubernetes evicts the Pod after the specified 30 seconds.

Behavior:

| Pod toleration | Behavior after matching `NoExecute` taint |
|---|---|
| No matching toleration | Evicted |
| Matching toleration without `tolerationSeconds` | Can remain indefinitely |
| Matching toleration with `tolerationSeconds: 30` | Evicted after 30 seconds if the taint remains |

This is particularly useful for handling temporary node problems.

---

## 12. Practical Uses of Taints and Tolerations

### A. Dedicated Nodes

Suppose you have dedicated nodes for database workloads.

Apply a taint:

```bash
kubectl taint node worker-1 dedicated=db:NoSchedule
```

Configure database Pods with:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "db"
    effect: "NoSchedule"
```

This allows database Pods to use the node.

For actual workload isolation, combine the toleration with node affinity or `nodeSelector`. Otherwise, tolerated database Pods are allowed on the node but are not required to use it.

### B. Control-Plane Nodes

Kubernetes clusters commonly taint control-plane nodes with:

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

This prevents ordinary Pods without a matching toleration from being scheduled there.

Some cluster distributions or configurations use different taints, or may not apply this taint at all.

### C. GPU Nodes

A GPU node might have a taint such as:

```text
nvidia.com/gpu=true:NoSchedule
```

A workload needs a matching toleration to pass the taint restriction.

However, toleration alone does not allocate a GPU.

A GPU workload must also request the appropriate extended resource, for example:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

The scheduler uses the resource request to determine whether the node has an available GPU resource.

### D. Node Maintenance

You can apply a `NoExecute` taint to evict Pods that do not tolerate it:

```bash
kubectl taint node node1 maintenance=true:NoExecute
```

Pods without a matching toleration will be evicted.

For ordinary planned maintenance, `kubectl drain` is generally the more appropriate administrative operation because it coordinates Pod eviction and respects mechanisms such as PodDisruptionBudgets.

---

## 13. Inspecting and Managing Node Taints

Use:

```bash
kubectl describe node <node-name>
```

The output includes a `Taints` field.

Example:

```text
Name:       worker-1
Roles:      worker
Labels:     kubernetes.io/hostname=worker-1

Taints:
  special=true:NoSchedule
  key1=value1:NoExecute
  dedicated=db:PreferNoSchedule
```

If the node has no taints, the output shows:

```text
Taints: <none>
```

### Adding a Taint

```bash
kubectl taint node worker-1 env=prod:NoSchedule
```

### Removing a Taint

```bash
kubectl taint node worker-1 env=prod:NoSchedule-
```

The trailing `-` removes the specified taint.

---

# Taints vs Affinity

These mechanisms serve different purposes.

| Mechanism | Applied to | Main purpose |
|---|---|---|
| Node affinity | Pod | Select or prefer nodes |
| Node anti-affinity (implemented using node affinity expressions) | Pod | Exclude certain nodes |
| Pod affinity | Pod | Co-locate with matching Pods |
| Pod anti-affinity | Pod | Avoid co-location with matching Pods |
| Taints | Node | Repel Pods |
| Tolerations | Pod | Allow Pods to tolerate matching taints |

### Mental Model

Think of each node as a room.

**Taint:**

> This room is restricted. You need permission to enter.

**Toleration:**

> This Pod has permission to enter the restricted room.

**Affinity:**

> This Pod wants to be in a particular room or topology domain.

**Anti-affinity:**

> This Pod wants to avoid a particular room or topology domain.

The important distinction is:

- Taints and tolerations control whether a Pod is allowed through a scheduling restriction.
- Affinity and anti-affinity express where a Pod must or prefers to run.

These mechanisms can be combined to create precise placement policies.

---

# Dynamic Taints and Node Health

Kubernetes can automatically add or remove certain taints based on node conditions.

This is part of the cluster's node-health and scheduling behavior.

### Node Heartbeats

The kubelet on each node regularly communicates node health information to the control plane through the API server.

Kubernetes uses this information to determine whether a node is healthy and reachable.

If the control plane stops receiving node updates within the expected period, the node controller can determine that the node is unhealthy or unreachable.

### Node-Health Taints

Kubernetes uses taints such as:

```text
node.kubernetes.io/not-ready:NoSchedule
node.kubernetes.io/unreachable:NoSchedule
```

These prevent new Pods from being scheduled on nodes that are not ready or reachable.

For eviction behavior, Kubernetes also uses `NoExecute` taints, including:

```text
node.kubernetes.io/not-ready:NoExecute
node.kubernetes.io/unreachable:NoExecute
```

These can cause existing Pods without matching tolerations to be evicted.

### How It Works Conceptually

```text
Node becomes unreachable
        │
        ▼
Control plane stops receiving expected updates
        │
        ▼
Node controller detects the problem
        │
        ▼
Node condition changes
        │
        ▼
Node-health taints are applied
        │
        ├── NoSchedule
        │     Prevents new scheduling
        │
        └── NoExecute
              Can evict existing Pods
```

When the node recovers and reports healthy status again, Kubernetes can remove the corresponding health-related taints.

### Default Tolerations for Node-Health Taints

Kubernetes commonly injects default tolerations for `node.kubernetes.io/not-ready` and `node.kubernetes.io/unreachable` into Pods.

These normally have a `tolerationSeconds` value of 300 seconds.

This means a Pod can tolerate these node conditions for a limited period before eviction, unless its configuration specifies different tolerations.

This behavior is important because a temporary network interruption does not necessarily cause immediate eviction of every Pod on the affected node.

**Important:** Node health detection and Pod eviction are separate processes. A node becoming unreachable does not mean every Pod is immediately terminated or that its application process has definitely stopped.

---

# Resource Requests and Scheduling

CPU and memory resource requests are important inputs to scheduling decisions.

When a Pod is scheduled, the scheduler checks whether the node has sufficient allocatable resources for the Pod's requests.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

In this example:

- The Pod requests 500 millicores of CPU.
- The Pod requests 512 MiB of memory.
- The CPU and memory limits define runtime resource limits.

**Important distinction:** The scheduler primarily uses resource requests when determining whether a node can accommodate a Pod. Resource limits do not directly determine whether a node has enough capacity for scheduling.

A Pod can remain Pending if no node has sufficient available allocatable CPU or memory for its requests, even if the node appears to have low actual resource usage.

---

# Summary

Kubernetes provides several mechanisms to control Pod placement.

| Mechanism | Purpose |
|---|---|
| `nodeSelector` | Strict node selection using exact label matching |
| Node affinity | Strict or preferred node selection using advanced expressions |
| Node exclusion | Avoid nodes using node affinity expressions such as `NotIn` |
| Pod affinity | Co-locate Pods within matching topology domains |
| Pod anti-affinity | Avoid co-locating Pods within matching topology domains |
| `nodeName` | Directly assign a Pod to a specific node, bypassing the scheduler |
| Taints | Repel Pods from nodes |
| Tolerations | Allow Pods to tolerate matching taints |
| Resource requests | Help the scheduler determine whether a node has sufficient capacity |
| Topology spread constraints | Control how evenly Pods are distributed across topology domains |

Key points:

- The kube-scheduler selects a node; the kubelet runs the Pod.
- Filtering eliminates unsuitable nodes.
- Scoring ranks the remaining eligible nodes.
- Hard constraints must be satisfied; soft preferences influence scoring.
- `nodeSelector` is simple and supports exact matching.
- Node affinity supports advanced matching expressions and weighted preferences.
- Node affinity rules ending in `IgnoredDuringExecution` do not automatically evict running Pods when node labels change.
- Pod affinity and anti-affinity use other Pods' labels and topology domains.
- `topologyKey` defines the topology domain used by Pod affinity and anti-affinity.
- `nodeName` bypasses normal scheduler decisions.
- Taints repel Pods; tolerations allow Pods to tolerate matching taints.
- A toleration does not force a Pod onto a node.
- Kubernetes can dynamically apply node-health taints when nodes become unhealthy or unreachable.
- The scheduler uses resource requests, not limits, as its primary CPU and memory capacity inputs.