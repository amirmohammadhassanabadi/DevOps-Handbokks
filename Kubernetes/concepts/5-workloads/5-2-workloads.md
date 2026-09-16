# Controllers and Workload Resources

A **controller** is a Kubernetes control-loop component that continuously observes the state of resources and takes corrective actions to move the actual state toward the desired state.

A **workload resource** is a Kubernetes API object that describes how an application or task should be run. Kubernetes provides several built-in workload resources, including **Deployment, StatefulSet, DaemonSet, Job, and CronJob**.

The relationship can be summarized as:

```text
Workload Resource
 ↓
Desired State
 ↓
Controller
 ↓
Reconciliation
 ↓
Pods
```

For example, when a Deployment specifies:

```yaml
spec:
  replicas: 3
```

the Deployment controller works to ensure that the desired workload state is represented by the appropriate ReplicaSet and Pods.

## Pod Templates

Most workload resources use a **Pod template** to define how their Pods should be created.

A Pod template typically specifies:

* Container images
* Containers and ports
* Environment variables
* Resource requests and limits
* Volumes and mounts
* Labels
* Security settings
* Other Pod configuration

For example:

```yaml
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:1.0
```

When additional Pods are required, the workload controller uses this template as the basis for creating them. This ensures that Pods belonging to the same workload are created with consistent configuration.

## Labels and Selectors

Controllers use **labels and selectors** to identify the Pods associated with a workload.

For example:

```yaml
selector:
  matchLabels:
    app: myapp
```

and:

```yaml
template:
  metadata:
    labels:
      app: myapp
```

The selector identifies the Pods managed by the workload.

This relationship is important because Kubernetes must be able to determine which Pods belong to a particular workload.

> The exact ownership and selection mechanism depends on the workload type. For example, Deployments manage ReplicaSets, and ReplicaSets manage Pods.

## Common Workload Resources

Kubernetes provides several built-in workload resources for different application patterns:

| Resource        | Purpose                                                                                       |
| --------------- | --------------------------------------------------------------------------------------------- |
| **ReplicaSet**  | Maintains a specified number of identical Pod replicas                                        |
| **Deployment**  | Manages stateless applications through ReplicaSets and provides rolling updates and rollbacks |
| **StatefulSet** | Manages workloads requiring stable Pod identity and commonly persistent storage               |
| **DaemonSet**   | Ensures a Pod runs on every eligible node or a selected set of nodes                          |
| **Job**         | Runs Pods until a task completes successfully                                                 |
| **CronJob**     | Creates Jobs according to a schedule                                                          |

These resources support different workload patterns:

```text
Deployment  → Stateless applications
StatefulSet → Stateful applications
DaemonSet   → Node-level workloads
Job         → One-time/batch tasks
CronJob      → Scheduled tasks
```

## ReplicaSet and Deployment Relationship

A particularly important relationship is:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

A Deployment normally does not directly maintain Pods. Instead:

1. The Deployment defines the desired application version and replica count.
2. The Deployment controller creates or updates a ReplicaSet.
3. The ReplicaSet controller maintains the required number of Pods.
4. Pods are scheduled and run by the normal Kubernetes scheduling and node components.

This layered design allows Deployments to provide features such as rolling updates and rollbacks while ReplicaSets handle replica maintenance.

## Controller Reconciliation

Controllers continuously reconcile desired and actual state.

Conceptually:

```text
Desired State
     │
     ▼
Controller observes resources
     │
     ▼
Compare desired vs actual state
     │
     ├── Match → No corrective action
     │
     └── Differ → Take corrective action
                         │
                         ▼
                    Cluster changes
                         │
                         ▼
                 Reconcile again
```

Controllers normally interact with the Kubernetes API through the API server. They do not directly modify etcd or bypass the Kubernetes API.

This reconciliation model is one of the fundamental mechanisms behind Kubernetes' declarative architecture.
