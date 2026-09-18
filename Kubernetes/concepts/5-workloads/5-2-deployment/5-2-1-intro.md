# Deployment

A **Deployment** is a higher-level Kubernetes workload resource that manages **ReplicaSets** and provides declarative management of stateless application workloads. It adds capabilities such as **rolling updates, rollbacks, revision history, and controlled scaling** on top of the replica-management functionality provided by ReplicaSets.

The relationship is:

```text
Deployment
    │
    ├── ReplicaSet
    │      └── Pods
    │
    └── ReplicaSet
           └── Pods
```

There are two separate controllers, both running inside kube-controller-manager:

```
kube-controller-manager
│
├── Deployment Controller
│      └── manages Deployment objects
│          └── creates/updates/scales ReplicaSets
│
└── ReplicaSet Controller
       └── manages ReplicaSet objects
           └── creates/deletes Pods
```

More precisely:

- Deployment Controller watches Deployments and manages their ReplicaSets. For example, when you change the image in a Deployment, it creates a new ReplicaSet with the new Pod template and scales the old/new ReplicaSets during the rollout.
- ReplicaSet Controller watches ReplicaSets and ensures that each ReplicaSet has the desired number of matching Pods. It creates replacement Pods or removes excess Pods.
- Scheduler then decides which node newly created Pods should run on.
- Kubelet on that node actually creates and runs the containers.

Deployments are commonly used for applications whose Pods are **interchangeable** and do not require a stable identity or individually attached persistent storage.

## Purpose and Behavior

A Deployment defines the desired state of an application, including:

* **Pod template** → specification used to create Pods
* **Replica count** → desired number of Pods
* **Update strategy** → how Pods are replaced during updates
* **Selector** → identifies the Pods managed by the Deployment

The Deployment controller continuously reconciles the desired state with the current state.

For example, suppose a Deployment initially uses:

```yaml
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

The resulting structure is:

```text
Deployment
    ↓
ReplicaSet (nginx:1.25)
    ↓
Pod
Pod
Pod
```

If the image is changed to `nginx:1.26`, the Deployment controller does **not** modify the existing ReplicaSet's Pod template. Instead, it creates a **new ReplicaSet** containing the new Pod template:

```text
Deployment
    │
    ├── ReplicaSet (nginx:1.25) → old Pods
    │
    └── ReplicaSet (nginx:1.26) → new Pods
```

The Deployment then scales the old and new ReplicaSets according to the configured update strategy until the rollout is complete.

> A Deployment does not directly manage individual Pods. It manages ReplicaSets, and ReplicaSets manage Pods.

## Rolling Updates

The default Deployment update strategy is **RollingUpdate**.

During a rolling update, Kubernetes gradually increases the number of new Pods and decreases the number of old Pods rather than deleting all old Pods at once.


The exact number of old and new Pods during the transition is controlled by the Deployment's rolling-update settings, primarily:

* **`maxUnavailable`** → maximum number of Pods that can be unavailable during the update
* **`maxSurge`** → maximum number of additional Pods that can temporarily exist above the desired replica count

A rolling update can maintain application availability, but **zero downtime is not guaranteed simply because a Deployment is being used**. Availability also depends on factors such as replica count, readiness probes, update configuration, application behavior, and Service configuration.

## Rollbacks and Revision History

Deployments retain information about previous ReplicaSets, allowing a rollout to be reverted when necessary.

For example:

```text
Revision 1 → nginx:1.25
Revision 2 → nginx:1.26
Revision 3 → nginx:1.27
```

If revision 3 introduces a problem, the Deployment can be rolled back to a previous revision.

```bash
kubectl rollout undo deployment nginx
```

The Deployment controller then restores the previous Pod template and performs another rollout.

The Deployment, rather than the ReplicaSet, is responsible for managing rollout history and coordinating transitions between ReplicaSets.

## Scaling

The desired number of replicas can be changed directly:

```bash
kubectl scale deployment nginx --replicas=5
```

Deployments can also be used as targets for **Horizontal Pod Autoscaling (HPA)**, allowing the number of replicas to change automatically according to resource usage or other configured metrics.

## Deployment Lifecycle

### Initial Creation

```text
Create Deployment
       ↓
Deployment Controller
       ↓
Create ReplicaSet
       ↓
ReplicaSet Controller
       ↓
Create Pods
       ↓
Scheduler assigns Pods to Nodes
       ↓
Kubelet starts Containers
       ↓
Pods become Ready
       ↓
Application is running
```

### Deployment Update

When the Deployment's Pod template changes, for example when the container image is updated:

```text
Deployment updated
       ↓
Deployment Controller
       ↓
Create new ReplicaSet
       ↓
ReplicaSet Controller
       ↓
Create new Pods
       ↓
Scheduler assigns new Pods to Nodes
       ↓
Kubelet starts new Containers
       ↓
New Pods become Ready
       ↓
Deployment Controller
       ↓
Gradually scale down old ReplicaSet
       ↓
ReplicaSet Controller
       ↓
Terminate old Pods
       ↓
New ReplicaSet reaches desired replica count
       ↓
Old ReplicaSet scaled to zero
       ↓
Rollout completed
```

The key relationship is:

```text
Deployment Controller
        ↓
   ReplicaSets
        ↓
ReplicaSet Controller
        ↓
      Pods
        ↓
   Scheduler
        ↓
     Nodes
        ↓
     Kubelet
        ↓
   Containers
```

The **Deployment Controller** coordinates the rollout by managing the ReplicaSets, while the **ReplicaSet Controller** is responsible for maintaining the desired number of Pods for each ReplicaSet.

## Deployment vs ReplicaSet

| Feature                           | ReplicaSet           | Deployment                    |
| --------------------------------- | -------------------- | ----------------------------- |
| Maintains Pod replicas            | ✓                    | ✓                             |
| Creates replacement Pods          | ✓                    | Indirectly through ReplicaSet |
| Rolling updates                   | ✗                    | ✓                             |
| Rollbacks                         | ✗                    | ✓                             |
| Revision history                  | ✗                    | ✓                             |
| Manages ReplicaSets               | ✗                    | ✓                             |
| Declarative application lifecycle | Limited              | ✓                             |
| Typical application workload      | Rarely used directly | Common                        |

The important distinction is:

```text
ReplicaSet → "How many Pods should exist?"

Deployment → "Which ReplicaSet should be active, and how should
              the application transition between versions?"
```

---

## Example Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    app: web
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
          ports:
            - containerPort: 80

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

This example defines a Deployment with **three desired Pod replicas** running an NGINX container.

The `RollingUpdate` strategy controls how Pods are replaced when the Deployment's Pod template changes.

* **`maxSurge: 1`** → the Deployment can temporarily have **up to one additional Pod above the desired replica count**.
* **`maxUnavailable: 1`** → **at most one Pod can be unavailable** during the rollout.

With three desired replicas, `maxSurge: 1` allows the rollout to temporarily have up to **4 Pods**, while `maxUnavailable: 1` ensures that the rollout does not intentionally reduce availability below **2 available Pods**.

These are **limits, not instructions**. Kubernetes does not simply interpret them as "delete one Pod and create one Pod." The Deployment Controller continuously adjusts the old and new ReplicaSets while respecting these limits.

---

## Pod Naming in Deployments

Pods created by a Deployment do not have fixed names defined in the Pod template. The Deployment creates a ReplicaSet, and the ReplicaSet creates Pods with generated names.

A Pod name generally follows this structure:

```text
<deployment-name>-<replicaset-hash>-<pod-id>
```

For example:

```text
web-deployment-7c9d8f6d5b-abc12
```

The components are:

```text
web-deployment    → Deployment name
7c9d8f6d5b        → ReplicaSet hash
abc12             → unique Pod-generated suffix
```

The **ReplicaSet hash** is derived from the Pod template and is used to distinguish ReplicaSets created from different Deployment revisions.

For example:

```text
Deployment
    │
    ├── ReplicaSet: web-deployment-7c9d8f6d5b
    │       ├── web-deployment-7c9d8f6d5b-abc12
    │       ├── web-deployment-7c9d8f6d5b-def34
    │       └── web-deployment-7c9d8f6d5b-ghi56
    │
    └── ReplicaSet: web-deployment-6f8a2c4e91
            ├── web-deployment-6f8a2c4e91-jkl78
            ├── web-deployment-6f8a2c4e91-mno90
            └── ...
```

When the Deployment's Pod template changes, such as:

```yaml
image: nginx:1.25
```

to:

```yaml
image: nginx:1.26
```

the Deployment Controller creates a **new ReplicaSet** with a different Pod-template hash. Pods created by that ReplicaSet therefore receive names containing the new hash.

```text
nginx:1.25
     ↓
ReplicaSet A
     ↓
web-deployment-7c9d8f6d5b-xxxxx

        update

nginx:1.26
     ↓
ReplicaSet B
     ↓
web-deployment-6f8a2c4e91-xxxxx
```

This naming structure makes it possible to identify which ReplicaSet a Pod belongs to, although the **Pod name itself should not be treated as a stable identity**.

---

> # Stateless vs Stateful Applications
> 
> The distinction between **stateless** and **stateful** applications is important when selecting a Kubernetes workload resource.
> 
> ## Stateless Applications
> 
> A **stateless application** does not depend on unique, persistent state stored inside a particular application instance. Each instance can generally process requests independently, and another instance can replace it if it fails.
> 
> Required information is either provided with the request or stored in an external system such as:
> 
> * Database
> * Cache
> * Object storage
> * External session store
> 
> For example:
> 
> ```text
> Client
>    ↓
> Service / Load Balancer
>    ↓
> ┌───────────────┐
> │ API Pod       │
> ├───────────────┤
> │ API Pod       │
> ├───────────────┤
> │ API Pod       │
> └───────────────┘
>         │
>         ↓
>  External Database
> ```
> 
> If one API Pod fails:
> 
> ```text
> API Pod 1 → failed
> API Pod 2 → continues serving
> API Pod 3 → continues serving
> ```
> 
> The failed Pod can be replaced without losing application state because the important state is stored outside the Pod.
> 
> ### Advantages
> 
> * **Easy horizontal scaling** → instances can be added or removed independently
> * **Fault tolerance** → failed instances can be replaced
> * **Simple load balancing** → requests can be distributed across interchangeable instances
> * **Fast recovery** → replacement instances do not require the identity or local state of the failed instance
> 
> Common examples include:
> 
> * Web servers such as NGINX and Apache
> * REST APIs
> * Backend microservices
> * Frontend applications
> * Proxy services
> 
> These characteristics make stateless applications well suited to **Deployments**.
> 
> ## Stateful Applications
> 
> A **stateful application** depends on persistent data, stable identity, or instance-specific state that must be preserved across Pod replacement or rescheduling.
> 
> For example, a database cluster may consist of individually identifiable members:
> 
> ```text
> Database Cluster
> ├── db-0
> ├── db-1
> └── db-2
> ```
> 
> Each instance may have:
> 
> * **Persistent storage**
> * **Stable network identity**
> * **Instance-specific data or role**
> * **Ordered startup or termination requirements**
> * **Cluster membership information**
> 
> For example, **`db-0`** may need to remain identifiable as **`db-0`** even if the Pod is rescheduled to another node.
> 
> Stateful workloads commonly include:
> 
> * PostgreSQL
> * MySQL
> * MongoDB
> * Kafka
> * Distributed storage systems
> * Clustered systems that maintain membership
> 
> Kubernetes provides **StatefulSet** for workloads that require stable identity and/or persistent storage with predictable Pod lifecycle behavior.
> 
> ## Important Distinction
> 
> Using a StatefulSet does **not automatically make an application stateful**, and using a Deployment does not inherently prevent persistent storage.
> 
> The important question is whether the **application instances themselves require stable identity or persistent, instance-specific state**.
> 
> For example:
> 
> ```text
> Stateless:
> 
> Pod A = Pod B = Pod C
> Instances are interchangeable
>         ↓
> Deployment is commonly appropriate
> ```
> 
> ```text
> Stateful:
> 
> Pod A ≠ Pod B ≠ Pod C
> Instances may have distinct identity/state
>         ↓
> StatefulSet may be appropriate
> ```
> 
> A Deployment can mount persistent storage, and a stateful application can sometimes be operated without StatefulSet. However, StatefulSet exists specifically to provide Kubernetes-level guarantees useful for workloads requiring stable identity, persistent storage association, and ordered lifecycle behavior.
> 
> ## Deployment vs StatefulSet
> 
> | Characteristic     | Deployment                   | StatefulSet                             |
> | ------------------ | ---------------------------- | --------------------------------------- |
> | Typical workload   | Stateless                    | Stateful                                |
> | Pod identity       | Interchangeable              | Stable, ordinal identity                |
> | Pod names          | Random/generated suffix      | Stable ordinal, e.g. `db-0`             |
> | Persistent storage | Possible                     | Designed for per-Pod persistent storage |
> | Scaling            | Interchangeable replicas     | Ordered/stable replicas                 |
> | Network identity   | Typically not stable per Pod | Stable per Pod                          |
> | Typical examples   | APIs, web apps, frontends    | Databases, clustered systems            |
> 
> ### Summary
> 
> A **Deployment** manages stateless or interchangeable application instances by managing ReplicaSets, which in turn manage Pods. Its main advantages over directly using ReplicaSets are **rolling updates, controlled transitions, rollback, and revision history**.
> 
> The key relationship is:
> 
> ```text
> Deployment
>     ↓
> ReplicaSet
>     ↓
> Pods
> ```
> 
> For applications where Pods are interchangeable, a Deployment is typically the appropriate workload resource. Applications that require stable identity, persistent per-instance storage, or ordered lifecycle behavior may instead require a StatefulSet.
> 