
# `nodeSelector`

`nodeSelector` is the simplest Kubernetes mechanism for controlling Pod placement.

It allows you to specify one or more node labels that must match for the scheduler to consider a node eligible.

### Example

```yaml
spec:
  nodeSelector:
    disktype: ssd
    environment: production
```

This means:

- The node must have `disktype=ssd`.
- The node must also have `environment=production`.

Both conditions must be satisfied.

### How It Works

During the **filtering** stage, the scheduler:

1. Examines available nodes.
2. Checks their labels against the Pod's `nodeSelector`.
3. Eliminates nodes that do not match all specified labels.
4. Scores the remaining eligible nodes.

If no node matches, the Pod remains Pending.

### Adding Node Labels

You must ensure that the relevant nodes have the labels required by the Pod.

```bash
kubectl label nodes worker-1 disktype=ssd
kubectl label nodes worker-1 environment=production
```

You can inspect node labels with:

```bash
kubectl get nodes --show-labels
```

Node labels can represent:

- Hardware: `disktype=ssd`, `gpu=true`
- Environment: `environment=production`
- Location: region, zone, rack
- Purpose: `workload=database`, `workload=logging`

### Limitations of nodeSelector

`nodeSelector` supports simple exact label matching.

It does not support:

- `In` or `NotIn` expressions
- `Exists` or `DoesNotExist`
- Numeric comparisons
- Multiple alternative matching expressions
- Soft scheduling preferences

For example, you cannot express:

> Run this Pod on nodes where `disktype` is either `ssd` or `nvme`.

You also cannot express:

> Prefer SSD nodes, but allow HDD nodes if no SSD node is available.

Node affinity supports these requirements.

### Full Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example
spec:
  nodeSelector:
    environment: staging
  containers:
    - name: busybox
      image: busybox
      command: ["sleep", "3600"]
```

Only nodes labeled `environment=staging` are eligible.

If `worker-1` is the only node with this label, the scheduler must select it, provided it satisfies the other scheduling requirements.

### When to Use nodeSelector

Use `nodeSelector` when:

- You need a simple, strict placement rule.
- Exact label matching is sufficient.
- You do not need advanced expressions or preferences.

Typical use cases:

- Running GPU workloads on GPU nodes.
- Placing workloads on nodes with specific hardware.
- Selecting nodes belonging to a particular environment or node pool.

---

# Node Affinity

Node affinity is a more expressive mechanism than `nodeSelector`.

It allows you to define Pod placement requirements based on node labels, supporting:

- Hard requirements
- Soft preferences
- Multiple matching operators
- AND/OR logic
- Weighted preferences

Node affinity is configured under:

```yaml
spec:
  affinity:
    nodeAffinity:
```

## Types of Node Affinity

There are two main types.

- ### requiredDuringSchedulingIgnoredDuringExecution

  This defines a **hard scheduling requirement**.

  A node must satisfy the configured rules for the scheduler to select it.

  Example:

  ```yaml
  spec:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: disktype
                  operator: In
                  values:
                    - ssd
  ```

  Meaning:

  The Pod must run on a node whose `disktype` label has the value `ssd`.

  If no node matches, the Pod remains Pending.

- ### preferredDuringSchedulingIgnoredDuringExecution

  This defines a **soft scheduling preference**.

  The scheduler prefers nodes that satisfy the configured rules but can select other eligible nodes when necessary.

  Example:

  ```yaml
  spec:
    affinity:
      nodeAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
                - key: environment
                  operator: In
                  values:
                    - production
  ```

  Meaning:

  The scheduler prefers nodes labeled `environment=production`, but this is not mandatory.

### Understanding IgnoredDuringExecution

Both rule types contain the suffix:

`IgnoredDuringExecution`

This means Kubernetes evaluates the rule when scheduling the Pod, but does not evict an already-running Pod merely because the node's labels later change and no longer satisfy the rule.

For example:

1. A Pod is scheduled on a node labeled `disktype=ssd`.
2. An administrator changes the label to `disktype=hdd`.
3. The Pod continues running.

The rule is not continuously enforced as an eviction policy.

### Weight

The `weight` field is available for soft node-affinity preferences.

Its allowed range is:

```text
1–100
```

A higher weight means that matching this preference contributes more to the node's score.

Example:

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 80
    preference:
      matchExpressions:
        - key: environment
          operator: In
          values: ["production"]

  - weight: 20
    preference:
      matchExpressions:
        - key: disktype
          operator: In
          values: ["ssd"]
```

Conceptually, matching nodes receive contributions from the applicable preferences:

| Node | Matches production | Matches SSD | Preference contributions |
|---|---:|---:|---:|
| Node 1 | Yes | Yes | 80 + 20 |
| Node 2 | Yes | No | 80 |
| Node 3 | No | Yes | 20 |

Node 1 receives the greatest contribution from these preferences.

**Important details about weight:**

- Weight is not a percentage or probability.
- It does not guarantee that a matching node will be selected.
- It does not allocate CPU or memory.
- It contributes to the node-affinity scoring process, which is combined with other scheduler scoring plugins.

## Node Affinity Operators

Node affinity supports these operators:

| Operator | Meaning |
|---|---|
| **`In`** | The label value must match one of the specified values. |
| **`NotIn`** | The label value must not match any specified value. |
| **`Exists`** | The node must have the specified label key. |
| **`DoesNotExist`** | The node must not have the specified label key. |
| **`Gt`** | The numeric label value must be greater than the specified value. |
| **`Lt`** | The numeric label value must be less than the specified value. |

For `Gt` and `Lt`, the comparison values must be valid integers.

Example:

```yaml
matchExpressions:
  - key: temperature
    operator: Lt
    values: ["70"]

  - key: zone
    operator: In
    values: ["us-east1-b", "us-east1-c"]
```

Both expressions must be satisfied within the same term.

> ### AND and OR Logic
> 
> Node affinity supports logical combinations through `nodeSelectorTerms` and `matchExpressions`.
> 
> **Inside one `nodeSelectorTerm`:**
> 
> Multiple `matchExpressions` are combined using AND.
> 
> **Between multiple `nodeSelectorTerms`:**
> 
> The terms are combined using OR.
> 
> Example:
> 
> ```yaml
> nodeSelectorTerms:
>   - matchExpressions:
>       # Term A
>       - key: region
>         operator: In
>         values: ["us-east"]
> 
>       - key: disktype
>         operator: In
>         values: ["ssd"]
> 
>   - matchExpressions:
>       # Term B
>       - key: environment
>         operator: In
>         values: ["production"]
> ```
> 
> The logic is:
> 
> ```text
> (Term A) OR (Term B)
> 
> Term A:
> region = us-east AND disktype = ssd
> 
> Term B:
> environment = production
> ```
> 
> A node qualifies if it satisfies all expressions in `Term A` or all expressions in `Term B`.
> 
> It does not need to satisfy both terms.

## `nodeSelector` vs Node Affinity

| Feature | nodeSelector | Node Affinity |
|---|---|---|
| Hard requirements | Yes | Yes |
| Soft preferences | No | Yes |
| Exact label matching | Yes | Yes |
| Advanced operators | No | Yes |
| AND/OR expressions | Limited to AND | Yes |
| Weighted preferences | No | Yes |

`nodeSelector` and required node affinity can also be used together.

If both are configured, **both sets of requirements must be satisfied**.

A useful way to understand `nodeSelector` is that its exact-match requirements can be expressed using required node affinity. They are separate API mechanisms, not a literal conversion performed by Kubernetes.

---

# Node Anti-Affinity

Kubernetes does not provide a separate `nodeAntiAffinity` field.

Instead, node exclusion is implemented through `nodeAffinity` using operators such as `NotIn` and `DoesNotExist`.

Example:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role
                operator: NotIn
                values: ["gpu"]
```

This prevents the Pod from being scheduled on nodes with `node-role=gpu`.

Node exclusion is useful for:

- Keeping general workloads away from special-purpose nodes.
- Avoiding noisy workloads on sensitive hardware.
- Excluding nodes reserved for specific workloads.

Remember that this is still **node affinity**, not a separate Kubernetes scheduling mechanism.

## Real-World Example: Hard Requirement and Soft Preference

Suppose a workload must run in the staging environment, but SSD nodes are preferred.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: advanced-affinity
spec:
  affinity:
    nodeAffinity:
      # Hard requirement
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: environment
                operator: In
                values: ["staging"]

      # Soft preference
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]

  containers:
    - name: busybox
      image: busybox
      command: ["sleep", "3600"]
```

Meaning:

- The Pod must run on a staging node.
- Among eligible staging nodes, the scheduler prefers SSD nodes.
- If staging nodes exist but none have SSD, the Pod can still run on another eligible staging node.

---

# `nodeName`

`nodeName` is the most direct way to assign a Pod to a specific node.

When you set `nodeName` in a Pod specification, you explicitly identify the node where the Pod should run.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  nodeName: worker-node-1
  containers:
    - name: nginx
      image: nginx
```

### How It Works

Normally, the process is:

```text
Pod created
    │
    ▼
Scheduler selects a suitable node
    │
    ▼
Scheduler binds Pod to node
    │
    ▼
Kubelet starts Pod
```

With `nodeName`:

```text
Pod created with nodeName
    │
    ▼
Scheduler is bypassed
    │
    ▼
Kubelet on specified node observes Pod
    │
    ▼
Kubelet attempts to start Pod
```

The scheduler does not select a node or perform its normal filtering and scoring process.

### Important Behavior

- If the specified node does not exist, the Pod cannot run.
- If the node is unavailable, the Pod cannot automatically be moved to another node.
- Normal scheduler filtering and scoring are bypassed.
- Resource suitability is not checked by the scheduler.
- Scheduler-based taint checks are also bypassed.

**Important taint distinction:** `nodeName` bypasses the scheduler, so a `NoSchedule` taint does not prevent the kubelet from starting the directly assigned Pod. However, a `NoExecute` taint can still cause the Pod to be evicted if it does not tolerate that taint.

### When to Use nodeName

It is uncommon for ordinary application workloads in production.

Possible uses include:

- Testing and debugging.
- Manual Pod placement.
- Special administrative scenarios.

For normal workloads managed by Deployments or StatefulSets, use scheduler-based placement mechanisms instead.

---

## nodeName vs nodeSelector vs Node Affinity

| Feature | nodeName | nodeSelector | Node Affinity |
|---|---|---|---|
| Placement | Specific node | Matching nodes | Matching nodes |
| Scheduler selects node | No | Yes | Yes |
| Hard requirements | Direct assignment | Yes | Yes |
| Soft preferences | No | No | Yes |
| Fallback to another node | No | Yes, among matching nodes | Yes, among eligible nodes |

---


