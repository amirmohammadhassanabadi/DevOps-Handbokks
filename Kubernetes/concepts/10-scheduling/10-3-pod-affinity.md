# Pod Affinity

Node affinity selects nodes based on **node labels**.

Pod affinity selects placement based on the location of **other Pods with matching labels**.

It tells the scheduler:

> Place this Pod in the same topology domain as Pods matching these labels.

The meaning of closeness depends on `topologyKey`.

For example:

- Same node
- Same availability zone
- Same region

Pod affinity is useful when workloads benefit from being located close to one another, such as applications that communicate frequently and need low network latency.

### How Pod Affinity Works

Suppose backend Pods have this label:

```yaml
labels:
  app: backend
```

You want frontend Pods to run on the same nodes as backend Pods.

Example:

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: backend
          topologyKey: kubernetes.io/hostname
```

Meaning:

- Find existing Pods labeled `app=backend`.
- Identify the nodes on which those Pods are running.
- Schedule the new Pod within the matching node topology domain.

Because `topologyKey` is `kubernetes.io/hostname`, the topology domain represents an individual node.

---

# Pod Anti-Affinity

Pod anti-affinity tells the scheduler:

> Do not place this Pod in a topology domain containing Pods that match the specified labels.

It is commonly used to spread application replicas across nodes or availability zones.

Example:

You have a Deployment with multiple replicas and want to prevent two Pods labeled `app=myapp` from running on the same node.

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname
```

Meaning:

- The scheduler checks the topology domain of existing Pods labeled `app=myapp`.
- A node that would place the new Pod in the same hostname domain is excluded.
- If no eligible node exists, the Pod remains Pending.

This is useful for improving availability by preventing replicas from sharing the same node.

## Hard and Soft Pod Affinity Rules

Pod affinity and anti-affinity support both hard requirements and soft preferences.

| Rule | Behavior |
|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | Matching is mandatory. |
| `preferredDuringSchedulingIgnoredDuringExecution` | Matching is preferred but not mandatory. |

- ### Soft Pod Affinity Example

    Prefer scheduling frontend Pods on the same nodes as backend Pods:

    ```yaml
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 70
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: backend
                topologyKey: kubernetes.io/hostname
    ```

    The scheduler prefers nodes in the same topology domain as matching backend Pods, but can select other eligible nodes.

- ### Soft Pod Anti-Affinity Example

    Prefer spreading database Pods across availability zones:

    ```yaml
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: database
                topologyKey: topology.kubernetes.io/zone
    ```

    This expresses a preference for placing the new Pod in a zone that does not already contain matching database Pods.

    It does not guarantee that every replica will be in a different zone.

---

## topologyKey

`topologyKey` defines the node-label key used to identify topology domains for Pod affinity and anti-affinity.

It tells Kubernetes what constitutes being close to or separated from another Pod.

Common topology labels include:

| Topology key | Domain |
|---|---|
| `kubernetes.io/hostname` | Individual node |
| `topology.kubernetes.io/zone` | Availability zone |
| `topology.kubernetes.io/region` | Region |

Examples:

**Same node:**

```yaml
podAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: backend
      topologyKey: kubernetes.io/hostname
```

**Different nodes:**

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: myapp
      topologyKey: kubernetes.io/hostname
```

**Prefer different zones:**

```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: database
        topologyKey: topology.kubernetes.io/zone
```

The relevant topology labels must exist on the nodes for these rules to work as intended.

### Pod Affinity vs Node Affinity

| Mechanism | Matches against | Example |
|---|---|---|
| Node affinity | Node labels | Run on GPU nodes |
| Pod affinity | Other Pods' labels and topology | Run near backend Pods |
| Pod anti-affinity | Other Pods' labels and topology | Keep replicas apart |

**Important:** Pod anti-affinity is not the only way to distribute replicas. Kubernetes also provides `topologySpreadConstraints`, which is specifically designed to control how evenly matching Pods are distributed across topology domains.

---