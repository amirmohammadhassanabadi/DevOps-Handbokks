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

---

# ReplicaSet

A **ReplicaSet** is a Kubernetes workload controller whose primary responsibility is to maintain a specified number of Pods that match its label selector.

It continuously reconciles the desired number of replicas with the current set of matching Pods. If the numbers differ, the ReplicaSet creates or deletes Pods to move the cluster toward the desired state.

ReplicaSets provide the basic **replica management and self-healing mechanism** used by many stateless workloads.

In practice, ReplicaSets are rarely created directly by users. They are most commonly created and managed by **Deployments**, which add higher-level capabilities such as **rolling updates**, **rollbacks**, and **revision** management.

The relationship is:

```text
Deployment
 ↓
ReplicaSet
 ↓
Pods
```

## Core Responsibility

The defining responsibility of a ReplicaSet is:

> **Maintain the desired number of Pods that match its selector.**

It continuously compares:

* **Desired state** → the number specified by `spec.replicas`
* **Actual state** → the current set of Pods matching the ReplicaSet's selector

If the actual state differs from the desired state, the ReplicaSet controller takes corrective action.

For example:

```text
Desired: 3 Pods
Current: 3 Pods
        ↓
One Pod is deleted
        ↓
Current: 2 Pods
        ↓
ReplicaSet creates a replacement
        ↓
Current: 3 Pods
```

This reconciliation process runs continuously.

## ReplicaSet Structure

A ReplicaSet primarily consists of three important parts:

### 1. Replicas

The `replicas` field specifies the desired number of Pod replicas.

```yaml
spec:
  replicas: 3
```

This tells the ReplicaSet to maintain three matching Pods.

### 2. Selector

The selector identifies the Pods that belong to the ReplicaSet.

```yaml
selector:
  matchLabels:
    app: web
```

The ReplicaSet uses this selector to identify matching Pods.

For example, a Pod with:

```yaml
metadata:
  labels:
    app: web
```

matches the selector above.

The ReplicaSet's selector must correspond to the labels defined by its Pod template.

### 3. Pod Template

The Pod template defines how new Pods should be created.

```yaml
template:
  metadata:
    labels:
      app: web
  spec:
    containers:
      - name: nginx
        image: nginx:1.25
```

When the ReplicaSet needs to create a new Pod, it uses this template as the basis for the new Pod.

A basic ReplicaSet can therefore be represented as:

```yaml
apiVersion: apps/v1
kind: ReplicaSet

metadata:
  name: web-rs

spec:
  replicas: 3

  selector:
    matchLabels:
      app: web

  template:
    metadata:
      labels:
        app: web

    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

## How a ReplicaSet Works

The ReplicaSet controller operates as part of the Kubernetes control plane.

Conceptually, the process is:

```text
ReplicaSet object
      ↓
ReplicaSet controller observes state
      ↓
Find matching Pods
      ↓
Compare desired vs actual count
      ↓
      ┌───────────────┐
      │               │
      ▼               ▼
  Too few Pods    Too many Pods
      │               │
      ▼               ▼
 Create Pods       Delete Pods
      │               │
      └───────┬───────┘
              ▼
       Desired state
          restored
```

The ReplicaSet object is stored in etcd through the Kubernetes API server. The ReplicaSet controller observes relevant resources through the API server and performs reconciliation by creating or deleting Pods through the API.

## Self-Healing

ReplicaSets provide workload-level self-healing by maintaining the desired number of matching Pods.

For example:

```text
Desired replicas: 3

Pod A
Pod B
Pod C
```

If Pod B is deleted:

```text
Pod A
Pod C
```

The ReplicaSet detects that only two matching Pods remain and creates a replacement:

```text
Pod A
Pod C
Pod D
```

The replacement Pod is a **new Pod object with its own identity**. It is not the same Pod that was deleted.

If a node fails and Pods running on that node are lost, the ReplicaSet can also create replacement Pods. The scheduler then determines where those new Pods should run.

> ReplicaSet maintains the desired replica count; it does not itself decide which node a replacement Pod runs on. Scheduling is handled by the kube-scheduler.

## Scaling

The number of replicas can be changed manually:

```bash
kubectl scale rs web-rs --replicas=5
```

The desired state becomes:

```text
3 Pods → 5 Pods
```

The ReplicaSet controller creates additional Pods until the desired count is reached.

Scaling can also be controlled by an **Horizontal Pod Autoscaler (HPA)**. In typical applications, the HPA targets a Deployment rather than a ReplicaSet directly.

## Pod Template Changes

A ReplicaSet does **not** perform rolling updates.

For example, suppose the ReplicaSet currently uses:

```yaml
image: nginx:1.25
```

and its template is changed to:

```yaml
image: nginx:1.26
```

Existing Pods are **not automatically replaced** with Pods using the new image.

The ReplicaSet maintains the replica count, but it does not provide a mechanism for gradually replacing existing Pods with a new version.

This is one of the main reasons Deployments are normally used instead of managing ReplicaSets directly.

A ReplicaSet continuously reconciles the number of Pods against `spec.replicas`, but changing its Pod template (for example, changing the image from `nginx:1.25` to `nginx:1.26`) does not automatically replace existing Pods. Existing Pods continue running with the old image because each Pod has its own already-created specification. If an existing Pod is later deleted or fails and the ReplicaSet needs to create a replacement, the new Pod is created from the ReplicaSet’s current template and therefore uses `nginx:1.26`. As a result, old and new Pod versions can temporarily coexist, but only because Pods are being replaced for another reason; the ReplicaSet itself does not perform a rollout when its template changes.

## Limitations

ReplicaSets provide basic replica management, but they do not provide the full application deployment lifecycle.

ReplicaSets do not provide Deployment-level features such as:

* Rolling updates
* Rollback management
* Revision history
* Versioned rollout management
* Controlled application updates

Their responsibility is primarily:

```text
Maintain the desired number of matching Pods
```

rather than:

```text
Manage the complete lifecycle of application versions
```

## ReplicaSet and Deployment

A **Deployment** provides a higher-level abstraction over ReplicaSets.

The typical relationship is:

```text
Deployment
    │
    ├── ReplicaSet
    │      ├── Pod
    │      ├── Pod
    │      └── Pod
    │
    └── ReplicaSet
           ├── Pod
           ├── Pod
           └── Pod
```

During a Deployment update, a new ReplicaSet is created for the new Pod template. The Deployment then gradually scales the new ReplicaSet up and the old ReplicaSet down.

For example:

```text
Old ReplicaSet
    3 Pods
       ↓
New ReplicaSet
    1 Pod

       ↓

Old ReplicaSet
    2 Pods
New ReplicaSet
    2 Pods

       ↓

Old ReplicaSet
    1 Pod
New ReplicaSet
    3 Pods

       ↓

Old ReplicaSet
    0 Pods
New ReplicaSet
    3 Pods
```

This layered design separates responsibilities:

| Component      | Responsibility                                                   |
| -------------- | ---------------------------------------------------------------- |
| **Deployment** | Manages application versions and rollout strategy                |
| **ReplicaSet** | Maintains the desired number of Pods for a specific Pod template |
| **Pod**        | Runs the actual application containers                           |

Therefore, when you create a Deployment, you normally do not create the ReplicaSet yourself. The Deployment controller creates and manages the appropriate ReplicaSets.

## When Would You Use a ReplicaSet Directly?

Direct ReplicaSet usage is uncommon, but it can be useful when:

* You need a fixed number of identical Pods
* You do not need rolling updates or rollbacks
* You are learning how ReplicaSets work
* You have a specialized use case where Deployment features are unnecessary

For normal stateless application deployments, a **Deployment is generally preferred**.

## ReplicationController → ReplicaSet → Deployment

ReplicaSet is historically related to the older **ReplicationController**.

ReplicationController provided the basic ability to maintain a desired number of Pod replicas:

```text
ReplicationController
        ↓
      Pods
```

ReplicaSet later provided a more flexible selector model, including **set-based label selectors**, while retaining the core responsibility of maintaining Pod replicas:

```text
ReplicaSet
    ↓
  Pods
```

However, both mechanisms focused primarily on replica management and did not provide the complete application update lifecycle.

Deployment introduced a higher-level abstraction that manages ReplicaSets and adds capabilities such as rolling updates, rollbacks, and revision history.

Today, ReplicaSets remain an important part of Kubernetes architecture, but users generally interact with **Deployments** rather than creating ReplicaSets directly.

## Summary

A ReplicaSet is responsible for maintaining a desired number of Pods that match its selector.

Its core mechanism is:

```text
Desired replicas
       ↓
ReplicaSet controller
       ↓
Compare with matching Pods
       ↓
Create/Delete Pods
       ↓
Desired replica count maintained
```

ReplicaSets provide:

* Replica management
* Workload-level self-healing
* Manual scaling
* Consistent Pod creation through Pod templates

However, they do not provide rolling updates, rollbacks, or revision management.

For this reason, ReplicaSets are usually managed indirectly through **Deployments**, which use ReplicaSets as the mechanism for maintaining Pods while providing higher-level application lifecycle management.
