# Kubernetes Pod Scheduling

The **kube-scheduler** is the Kubernetes control-plane component responsible for selecting a suitable node for each newly created Pod that does not already have a node assigned.

The scheduler evaluates several factors, including:

- CPU and memory resource requests
- Node labels and scheduling constraints
- Pod affinity and anti-affinity
- Node taints and Pod tolerations
- Topology and hardware requirements
- Scheduling policies and priorities

After selecting a suitable node, the scheduler binds the Pod to that node. The kubelet running on the selected node then takes responsibility for running the Pod.

By default, Kubernetes clusters commonly apply a `NoSchedule` taint to control-plane nodes to prevent ordinary workloads from being scheduled there. This behavior depends on the cluster distribution and configuration.

Kubernetes provides several mechanisms to influence Pod placement, including:

- `nodeSelector`
- Node affinity and anti-affinity
- Pod affinity and anti-affinity
- Taints and tolerations
- `nodeName`
- Topology spread constraints
- Resource requests and scheduling policies

These mechanisms allow administrators to express requirements such as:

- Run GPU workloads only on GPU-equipped nodes.
- Prefer nodes with SSD storage.
- Keep replicas on different nodes or availability zones.
- Reserve dedicated nodes for specific workloads.
- Avoid scheduling workloads on unhealthy or unsuitable nodes.

---

## How the Scheduler Selects a Node

The default kube-scheduler follows a scheduling process that includes several stages.

Conceptually:

```text
Pod created
    │
    ▼
Scheduler observes an unscheduled Pod
    │
    ▼
Filtering
    │
    │ Removes nodes that cannot run the Pod
    ▼
Scoring
    │
    │ Ranks the remaining eligible nodes
    ▼
Node selected
    │
    ▼
Binding
    │
    ▼
Pod assigned to the selected node
    │
    ▼
Kubelet starts the Pod
```

- ### Filtering

    The scheduler evaluates nodes and eliminates those that do not satisfy the Pod's scheduling requirements.

    Examples:

    - Insufficient available CPU or memory for the Pod's requests.
    - Node does not match `nodeSelector`.
    - Required node affinity is not satisfied.
    - A node has an untolerated `NoSchedule` taint.
    - Required Pod affinity or anti-affinity rules are not satisfied.

    The remaining nodes are considered eligible for the Pod.

    If no node passes the filtering stage, the Pod remains `Pending`.

- ### Scoring

    The scheduler scores the eligible nodes using its scheduling plugins and configured policies.

    For example:

    - A node may receive a higher score because it matches a preferred node-affinity rule.
    - A node may be preferred because of resource distribution.
    - A node may be preferred because it satisfies topology-related preferences.

    The scheduler selects a node based on the resulting scores.

    Soft scheduling preferences influence this stage, but they do not guarantee placement.

- ### Binding

    After selecting a node, the scheduler binds the Pod to it by assigning the node name.

    The kubelet on that node then observes the assigned Pod and attempts to run it.

    **Important:** The scheduler does not start containers itself. Its responsibility is to select and assign a node.

---

## Resource Requests and Scheduling

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

## Hard Constraints vs Soft Preferences

Kubernetes scheduling rules can express either mandatory requirements or preferences.

- ### Hard Constraints

    A hard constraint must be satisfied for a node to be considered eligible.

    If no node satisfies the requirement, the Pod cannot be scheduled.

    Examples:

    - `nodeSelector`
    - `requiredDuringSchedulingIgnoredDuringExecution` in node affinity
    - `requiredDuringSchedulingIgnoredDuringExecution` in Pod affinity or anti-affinity

- ### Soft Preferences

    A soft preference tells the scheduler which nodes are preferable, but does not make matching nodes mandatory.

    If no node satisfies the preference, the scheduler can still select another eligible node.

    Examples:

    - `preferredDuringSchedulingIgnoredDuringExecution` in node affinity
    - `preferredDuringSchedulingIgnoredDuringExecution` in Pod affinity or anti-affinity

**Conceptually:**

| Hard constraint | Soft preference |
|---|---|
| Mandatory requirement | Optional preference |
| Filters out unsuitable nodes | Influences node scoring |
| Pod may remain Pending if no node matches | Scheduler can choose another eligible node |

This distinction allows Kubernetes to combine strict placement requirements with flexible scheduling preferences.

---