# Scaling with Deployment

A Deployment can be scaled manually or automatically.

### Manual Scaling

Use `kubectl scale` to change the desired number of replicas:

```bash
kubectl scale deployment/web-deployment --replicas=5
```

The Deployment updates its desired replica count, and its ReplicaSet controller creates or removes Pods to reach the new desired state.

```text
Deployment
    ↓
ReplicaSet
    ↓
5 Pods
```

Manual scaling changes the current replica count but does not modify the Pod template, so it does not create a new Deployment revision.

---

# Horizontal Pod Autoscaler (HPA)

The **Horizontal Pod Autoscaler (HPA)** automatically adjusts the number of Pod replicas in a workload based on observed metrics.

Instead of manually changing the replica count, HPA periodically evaluates the configured metrics and calculates the desired number of replicas.

```text
Current metrics
      ↓
HPA Controller
      ↓
Calculate desired replicas
      ↓
Update workload scale
      ↓
Deployment
      ↓
ReplicaSet
      ↓
Pods
```

HPA can target scalable workloads such as Deployments, StatefulSets, and other resources that expose the Kubernetes `scale` subresource.

---

> ## Horizontal vs. Vertical Scaling
> 
> - ### Horizontal Scaling
> 
>   Horizontal scaling changes the **number of Pods**.
>   
>   ```text
>   3 Pods → 10 Pods
>   ```
>   
>   The workload handles additional load by running more instances.
> 
> - ### Vertical Scaling
> 
>   Vertical scaling changes the **resources allocated to a Pod's containers**.
>   
>   ```text
>   CPU:    500m → 1
>   Memory: 512Mi → 1Gi
>   ```
>   
>   Vertical Pod Autoscaler (VPA) is a separate Kubernetes mechanism for automatically adjusting resource requests and limits.

---

# How HPA Works

The **HPA controller** periodically obtains the configured metrics and compares the current values with the target values.

For resource utilization metrics, the general calculation is based on the ratio between the current metric value and the configured target.

For example:

```text
Target CPU utilization = 50%
Current CPU utilization = 80%
```

The HPA calculates a higher desired replica count because the current utilization is above the target.

If utilization later falls significantly below the target, HPA can reduce the replica count, subject to its scaling behavior and configured limits.

HPA does not directly create or delete Pods. It changes the desired scale of the target workload, and the workload's controllers reconcile the resulting replica count.

---

# HPA Metrics

HPA supports several metric sources.

- ## Resource Metrics

    Resource metrics are commonly used for CPU and memory.

    Examples:

    * CPU utilization
    * Memory utilization
    * CPU average value
    * Memory average value

    For resource utilization, the percentage is calculated relative to the configured resource request.

    For example:

    ```yaml
    resources:
      requests:
        cpu: 500m
    ```

    If the container uses `250m` CPU:

    ```text
    CPU utilization = 250m / 500m × 100 = 50%
    ```

    Resource metrics are normally provided through the Kubernetes resource metrics API, commonly implemented by **Metrics Server**.

    ---

- ## Custom Metrics

    Custom metrics can represent application-specific measurements.

    Examples:

    * Requests per second
    * Queue length
    * Application-specific request count

    Custom metrics require a component that exposes the metrics through the Kubernetes custom metrics API.

    Prometheus is commonly used as the monitoring system, with an adapter such as Prometheus Adapter exposing selected metrics to Kubernetes.

    ---

- ## External Metrics

    External metrics come from systems outside the Kubernetes resource model.

    Examples:

    * Messages waiting in Kafka or RabbitMQ
    * Cloud monitoring metrics
    * External service metrics

    An external metrics provider is required to make these metrics available to HPA.

---

# Example HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment

  minReplicas: 2
  maxReplicas: 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50

    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
```

### Meaning

```text
Target workload    → nginx-deployment
Minimum replicas   → 2
Maximum replicas   → 10
CPU target         → 50%
Memory target      → 70%
```

Because two metrics are configured, HPA evaluates both and determines the desired replica count for each metric. The highest required replica count is used, subject to `minReplicas` and `maxReplicas`.

---

# Example HPA Behavior

Suppose:

```text
Current replicas: 2
Minimum replicas: 2
Maximum replicas: 10

CPU target:    50%
Memory target: 70%
```

### Low Load

```text
CPU:    30%
Memory: 40%
```

The desired replica count may remain at 2.

### Increased Load

```text
CPU:    85%
Memory: 75%
```

HPA calculates a higher desired replica count and may scale the Deployment up.

For example:

```text
2 Pods → 4 Pods
```

The exact number is calculated by HPA; it is not simply "add one Pod whenever CPU exceeds the target."

### Load Decreases

When the observed metrics remain sufficiently below the targets, HPA can reduce the desired replica count, but not below `minReplicas`.

```text
4 Pods → 2 Pods
```

---

## HPA with Multiple Metrics

When an HPA uses multiple metrics, Kubernetes evaluates each metric **independently**.

HPA does **not** require all metrics to reach their targets.

For example:

```text
CPU target    = 50%
Memory target = 70%

Current CPU    = 65%  → above target
Current Memory = 5%   → below target
```

In this case, the CPU metric can cause HPA to **scale up**, even though memory usage is well below its target.

The metrics work as follows:

```text
CPU    → may require more replicas
Memory → may require fewer or the same replicas
                 ↓
        HPA uses the highest
        desired replica count
```

Therefore:

> **Only one metric needs to indicate a higher replica count for HPA to scale up.**


---

# Requirements for Resource-Based HPA

For CPU or memory metrics to work correctly, the cluster must provide resource metrics.

You can check resource metrics with:

```bash
kubectl top nodes
kubectl top pods
```

If these commands return resource usage data, the resource metrics API is available.

## Resource Requests

For utilization-based CPU or memory HPA, the corresponding resource **request** must be defined.

Example:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 128Mi
```

If a container has no CPU request, HPA cannot calculate CPU utilization as a percentage of the request for that container.

The same applies to memory utilization.

---

> ## Resource Requests and Limits
> 
> **Containers** can define resource requests and limits:
> 
> ```yaml
> resources:
>   requests:
>     cpu: "250m"
>     memory: "128Mi"
> 
>   limits:
>     cpu: "500m"
>     memory: "256Mi"
> ```
> 
> These values influence **scheduling**, resource management, and container resource enforcement.
> 
> ## Resource Requests
> 
> A **request** specifies the amount of a resource that Kubernetes uses when scheduling the Pod.
> 
> For example:
> 
> ```yaml
> requests:
>   cpu: "500m"
>   memory: "256Mi"
> ```
> 
> The scheduler considers **`500m`** CPU and **`256Mi`** memory when determining whether a node has sufficient allocatable capacity for the Pod.
> 
> The request is therefore used for **scheduling and resource accounting**, not as a hard runtime usage limit.
> 
> A container can use more than its request when additional resources are available and its limit allows it.
> 
> For example:
> 
> ```text
> CPU request = 500m
> CPU limit   = 1
> 
> Application may use:
> 500m → 700m → 900m
> ```
> 
> provided the node has available CPU and the container does not exceed its limit.
> 
> ---
> 
> ## Resource Limits
> 
> A **limit** defines the maximum resource usage enforced for the container.
> 
> Example:
> 
> ```yaml
> limits:
>   cpu: "500m"
>   memory: "256Mi"
> ```
> 
> ### CPU Limit
> 
> When a container attempts to use more CPU than its CPU limit, the container is generally **CPU throttled** rather than killed.
> 
> This can reduce application performance.
> 
> ### Memory Limit
> 
> Memory limits are enforced differently.
> 
> If a container exceeds its memory limit, it can be terminated by the **kernel's out-of-memory mechanism** and reported as **OOMKilled**
> 
> 
> For a Pod managed by a Deployment, the controllers will normally ensure that a replacement Pod is created if the Pod itself is terminated.
> 
> ---
> 
> ## CPU Units
> 
> CPU resources are measured in CPU cores.
> 
> ```text
> 1      = 1 CPU core
> 500m   = 0.5 CPU core
> 250m   = 0.25 CPU core
> 100m   = 0.1 CPU core
> ```
> 
> `m` means **millicpu**.
> 
> For example:
> 
> ```yaml
> cpu: "500m"
> ```
> 
> represents half of one CPU core.
> 
> ---
> 
> ## Memory Units
> 
> Memory is specified in bytes and commonly expressed using binary units:
> 
> ```text
> Ki = Kibibyte
> Mi = Mebibyte
> Gi = Gibibyte
> ```
> 
> For example:
> 
> ```yaml
> memory: "128Mi"
> ```
> 
> means 128 MiB.
> 
> ---
> 
> # Example Deployment with Resources
> 
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx-deployment
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: nginx
>   template:
>     metadata:
>       labels:
>         app: nginx
>     spec:
>       containers:
>         - name: nginx
>           image: nginx:1.26
>           resources:
>             requests:
>               cpu: "250m"
>               memory: "128Mi"
>             limits:
>               cpu: "500m"
>               memory: "256Mi"
> ```
> 
> In this example:
> 
> ```text
> CPU request    → 250m
> CPU limit      → 500m
> 
> Memory request → 128Mi
> Memory limit   → 256Mi
> ```
> 
> The **scheduler** uses the **`requests`** when selecting a node.
> 
> At runtime, CPU usage can be throttled at the configured CPU limit, while exceeding the memory limit can result in an OOM kill.
>

---

# Resource Requests, Limits, and HPA

These concepts are related but have different purposes:

```text
Resource Request
 ├── Scheduling
 ├── Resource accounting
 └── HPA utilization baseline
          
Resource Limit
 └── Runtime resource enforcement

HPA
 └── Changes number of Pod replicas
```

For CPU utilization-based HPA:

```text
CPU utilization = actual CPU usage / CPU request × 100
```

Example:

```text
CPU request = 500m
Actual usage = 250m

250m / 500m × 100 = 50%
```

Therefore, if the HPA target is:

```yaml
averageUtilization: 50
```

the target is **50% of the requested CPU**, not 50% of the CPU limit.

---

# Summary

* **Manual scaling** changes the Deployment's desired replica count.
* **HPA** automatically adjusts the replica count based on metrics.
* **Horizontal scaling** changes the number of Pods.
* **Vertical scaling** changes the resources allocated to Pods.
* HPA changes the workload's desired scale; the workload controllers create or remove Pods.
* CPU/memory utilization targets are calculated relative to **resource requests**.
* **Requests** are primarily used for scheduling and resource accounting.
* **Limits** define runtime resource boundaries.
* CPU limit exhaustion generally causes **throttling**.
* Memory limit exhaustion can result in **OOMKilled**.
* **`kubectl top`** can be used to verify that resource metrics are available.
* CPU/memory HPA commonly uses the resource metrics API, often provided by Metrics Server.
* Custom and external metrics require appropriate metrics providers.