# kube-apiserver

The kube-apiserver is the central entry point to the Kubernetes API. Clients and Kubernetes components communicate with the cluster through the API server.

Examples of API clients include:

- `kubectl`
- Controllers
- Scheduler
- Kubelets
- Web dashboards
- CI/CD systems
- External automation tools

**Responsibilities:**

- exposes the Kubernetes API
- authenticates requests
- authorizes requests
- runs admission control
- validates API objects
- reads and writes cluster state
- persists Kubernetes objects in etcd

**Example:**

`kubectl apply -f deployment.yaml` → kube-apiserver → etcd

The API server is the central API layer, while etcd is the persistent data store for Kubernetes state.

---

In Docker, almost every operation ultimately goes through the Docker Engine’s REST API. The Docker CLI is essentially a client for that API. For example, when a user runs `docker ps`, the CLI sends an HTTP GET request to the Docker daemon through its REST API (usually via a Unix socket such as `/var/run/docker.sock`). The daemon processes the request, retrieves the list of running containers, and returns the result to the CLI. In this architecture, the Docker daemon acts as the central authority that receives API requests and performs the requested container operations.

Kubernetes follows a similar idea but at a much larger, distributed scale. In Kubernetes, almost all interactions with the cluster occur through the kube-apiserver, which serves as the central entry point to the Kubernetes API. Tools such as `kubectl`, web dashboards, CI/CD systems, and automation tools communicate with the cluster by sending REST API requests to the API server. For example, when a user runs `kubectl get pods`, the request is sent to the API server, which authenticates and authorizes the request, validates it, retrieves the requested resource (typically from its cache or, when necessary, from etcd), and returns the response. For write operations such as creating or updating resources, the API server stores the desired state in etcd, after which Kubernetes controllers observe the changes and work to make the actual cluster state match the desired state.

The API server is also the central communication hub for internal Kubernetes components. Core components such as the scheduler, controller manager, and kubelets do not communicate directly with each other in most cases. Instead, they interact with the API server, reading the cluster state and updating it through the Kubernetes API. For example, when a new Pod is created, the request is stored in etcd via the API server. The scheduler watches the API for unscheduled Pods, selects an appropriate node, and writes that decision back through the API server. The kubelet on the chosen node then watches the API server for Pods assigned to its node and starts the required containers.

Because of this design, the API server acts as the central control point and communication layer of the entire Kubernetes system. Nearly every action, whether from users, tools, or internal components, is expressed as an API request that reads or modifies the cluster’s desired state. This API‑driven architecture is what enables Kubernetes to be extensible, observable, and controllable by a wide variety of clients and automation systems.

---

For write operations (create, update, delete), the API server does much more than simply write to etcd.

For a typical write request:

    Client
      ↓
    kube-apiserver
      ↓
    Authentication
      ↓
    Authorization
      ↓
    Admission
      ↓
    Validation
      ↓
    Persist object in etcd
      ↓
    Controllers / Scheduler / Kubelet observe the new state

For example:

    kubectl apply -f deployment.yaml
        ↓
    kube-apiserver
        ↓
    Authenticate
        ↓
    Authorize
        ↓
    Admission
        ↓
    Validate
        ↓
    Store Deployment in etcd
        ↓
    Deployment Controller
        ↓
    ReplicaSet Controller
        ↓
    Pods

## Authentication

Authentication answers:

"Who are you?"

The API server can use different authentication mechanisms, such as:

- X.509 client certificates
- Bearer tokens
- ServiceAccount tokens
- OIDC
- Webhook authentication

These authentication mechanisms are part of the API server's request-processing pipeline.

For example:

    Request
       ↓
    kube-apiserver
       ↓
    Authentication
       ├── X.509
       ├── ServiceAccount token
       ├── OIDC
       ├── Webhook
       └── ...
       ↓
    Authenticated identity

Authentication establishes an identity such as: User: alice

## Authorization

Authorization answers:

"Is this identity allowed to perform this action?"

For example:

```yaml
User:      alice
Verb:      get
Resource:  pods
Namespace: production
```

The API server evaluates the configured authorization mechanisms. The most commonly used mechanism is **RBAC**.

**Important:**

RBAC is not a separate controller or process. It is an authorization mechanism used by the API server.

**Conceptually:**

    Authenticated identity
            ↓
    Authorization
            ↓
    RBAC Authorizer
            ↓
    Allowed / Denied

Other authorization mechanisms can also be configured, such as the Node authorizer and Webhook authorizer.

### Admission

After authentication and authorization, the request enters the admission chain.

Admission controllers can:

- modify a request
- validate a request and reject it

**Examples:**

- **`ResourceQuota`**
- **`LimitRanger`**
- **`PodSecurity`**
- **`NamespaceLifecycle`**
- **`MutatingAdmissionWebhook`**
- **`ValidatingAdmissionWebhook`**

Some admission plugins are built into the API server.

Webhook-based admission is different because the API server sends an HTTP request to an external admission webhook service.

## Validation

The API server validates that the requested object is structurally and semantically valid.

For example, Kubernetes validates things such as:

- API version
- resource kind
- field structure
- field types
- required fields
- valid values

If validation fails, the API server rejects the request.

## Storage

If the request successfully passes the required processing stages, the API server persists the Kubernetes object in etcd.

For example:

    kubectl
       ↓
    kube-apiserver
       ↓
    Authentication ✓
       ↓
    Authorization ✓
       ↓
    Admission ✓
       ↓
    Validation ✓
       ↓
    etcd

**Important:**

The API server is responsible for communicating with etcd; other Kubernetes components normally do not communicate directly with etcd.

### What happens after the object is stored?

This is where the controllers become important.

Suppose you create:

```yaml
kind: Deployment
spec:
  replicas: 3
```

The API server accepts and stores the Deployment. Then the controllers observe the new API state and reconcile it.

Simplified:

    kubectl
       ↓
    kube-apiserver
       ↓
    etcd
       ↓
    Deployment Controller
       ↓
    ReplicaSet
       ↓
    ReplicaSet Controller
       ↓
    Pods
       ↓
    Scheduler
       ↓
    Pod assigned to a node
       ↓
    Kubelet
       ↓
    Container Runtime
       ↓
    Containers

The key distinction is:

    The API server accepts and records the desired state. Controllers continuously work to make the actual state match that desired state.

## kube-apiserver vs kube-controller-manager

The kube-controller-manager is a completely separate process. Its job is not to process API requests. Its job is to run reconciliation loops (controllers), such as:
- Deployment Controller
- ReplicaSet Controller
- Job Controller
- Node Controller
- ServiceAccount Controller
- Namespace Controller
- PersistentVolume Controller
- EndpointSlice Controller
- ...and many others.

Each controller watches the API server and tries to make the actual cluster state match the desired state.

Notice that the controller manager is not involved until after the object has been accepted and stored by the API server.

| Component | Main responsibility |
| --- | --- |
| kube-apiserver | API entry point; authenticates, authorizes, admits, validates, and persists API objects |
| etcd | Stores Kubernetes cluster state |
| kube-controller-manager | Runs controllers that continuously reconcile actual state toward desired state |
| kube-scheduler | Selects nodes for unscheduled Pods |
| kubelet | Ensures Pods assigned to its node are actually running |

> **kube-controller-manager** → The reconciliation engine that runs controllers and continuously works to make the actual cluster state match the desired state.

---

## How Kubernetes Components Watch the API Server

Kubernetes control plane components and controllers need to know when the state of Kubernetes resources changes. They do this primarily by communicating with the **kube-apiserver** through the Kubernetes API.

A common misunderstanding is that controllers continuously send requests such as:

```js
GET /api/v1/pods
GET /api/v1/pods
GET /api/v1/pods
...
```

This is not how Kubernetes normally works (That would be extremely inefficient). Instead, Kubernetes uses the **Watch API** together with client-side **informers and caches**.

### Watch API

The Kubernetes API provides a **Watch** mechanism that allows a client to receive notifications when resources change.

For example, a controller can ask the API server to watch Pods. The API server then sends events whenever a relevant Pod is created, modified, or deleted.

Conceptually:

```text
Controller
    │
    │ LIST Pods
    ▼
kube-apiserver
    │
    │ Current Pods
    ▼
Controller
    │
    │ WATCH Pods
    ▼
kube-apiserver
    │
    │
    │ connection remains open
    │
    ├── ADDED Pod A
    ├── MODIFIED Pod B
    └── DELETED Pod C
```

The Watch API is therefore similar in concept to a real-time event mechanism such as Socket.IO: instead of repeatedly polling the server, the client establishes a watch and receives changes as they occur.

However, Kubernetes Watch and Socket.IO are different technologies. Kubernetes Watch is part of the Kubernetes HTTP API.

**For example:**

```text
Controller                     API Server
    │                              │
    │──── LIST Pods ──────────────►│
    │◄── current Pods ─────────────│
    │                              │
    │──── WATCH Pods ─────────────►│
    │                              │
    │    connection stays open     │
    │                              │
    │◄──── MODIFIED Pod A ─────────│
    │◄──── ADDED Pod B ────────────│
    │◄──── DELETED Pod C ──────────│
```
---

## Informers

An **informer** is client-side code used by Kubernetes controllers and other components to efficiently observe Kubernetes resources.

An informer is **not part of the kube-apiserver** and is **not a separate Kubernetes component**. It runs inside the process that uses it, such as a controller.

For example, the ReplicaSet controller can have an informer for Pods and an informer for ReplicaSets.

The informer performs two important tasks:

1. It obtains the current state of resources from the API server.
2. It watches the API server for subsequent changes.

The informer also maintains a **local cache** of the resources it is watching.

Conceptually:

```text
                 kube-apiserver
                       │
                  LIST + WATCH
                       │
                       ▼
                    Informer
                       │
                       ▼
                 Local Cache
                       │
                       ▼
              Controller Logic
```

The purpose of the cache is efficiency. The controller can usually read the current resource state from its local cache instead of sending another API request every time it needs information.

---

## Why Does the Informer First LIST?

A watch only tells the client about changes that happen after the watch is established. The controller therefore needs to know the state that already exists.

For example, suppose these Pods already exist:

```text
Pod A
Pod B
Pod C
```

The informer initially performs a **LIST** operation:

```text
Informer → API Server: LIST Pods

API Server → Informer:
Pod A
Pod B
Pod C
```

The informer puts this information into its local cache.

It then establishes a **WATCH** so that future changes can be received.

Therefore, the simplified process is:

```text
1. LIST → Get the current state

2. WATCH → Receive future changes

3. CACHE → Maintain a local copy of the observed state
```

---

## Resource Change Events

When a watched resource changes, the API server sends an event to the watcher.

The main event types are:

* **ADDED** → a resource was created
* **MODIFIED** → a resource was changed
* **DELETED** → a resource was deleted

For example:

```text
Pod A created
    ↓
ADDED Pod A

Pod A's status changes
    ↓
MODIFIED Pod A

Pod A deleted
    ↓
DELETED Pod A
```

The informer receives these events and updates its local cache accordingly.

---

## Work Queue

Controllers also commonly use a **work queue**.

The work queue is also **inside the controller process**. It is not part of the API server.

Its purpose is to tell the controller:

> "This resource may need reconciliation."

For example:

```text
API Server
    │
    │ DELETED Pod C
    ▼
Informer
    │
    ├── updates local cache
    │
    └── adds relevant object key
             to Work Queue
                    │
                    ▼
             Controller Worker
                    │
                    ▼
              Reconciliation
```

The queue normally contains a reference/key identifying the object that needs processing rather than simply being a storage location for every complete API object.

---

## Example: ReplicaSet Controller

Suppose a ReplicaSet specifies:

```yaml
desired replicas = 3
```

and currently has:

```text
Pod A
Pod B
Pod C
```

The ReplicaSet controller's local cache therefore represents:

```text
Desired state:
3 Pods

Current observed state:
3 Pods
```

Everything is correct.

Now Pod C is deleted.

The sequence is approximately:

```text
Pod C deleted
      ↓
kube-apiserver processes the deletion
      ↓
API server sends DELETED event
      ↓
ReplicaSet informer's Pod watch receives event
      ↓
Informer updates its local cache
      ↓
Informer adds the relevant ReplicaSet key
to the work queue
      ↓
ReplicaSet controller processes the queue
      ↓
Controller reads current state from its cache
      ↓
Desired = 3
Current = 2
      ↓
Controller determines that one Pod is missing
      ↓
Controller sends CREATE Pod request
to kube-apiserver
      ↓
API server stores the new Pod object
      ↓
Informer receives ADDED event
      ↓
Cache is updated
      ↓
Pod eventually becomes available
```

The important point is that the controller is **not continuously polling the API server**.

It is reacting to events and then reconciling.

---

## Where Are These Pieces Located?

It is important to distinguish the API server from the controller-side mechanisms.

### kube-apiserver

The API server:

* receives API requests
* authenticates and authorizes them
* performs admission and validation
* reads/writes Kubernetes objects
* provides the Watch API

It does **not** contain the ReplicaSet controller's informer or work queue.

### kube-controller-manager

The `kube-controller-manager` runs many controllers.

Each controller contains its own controller logic and commonly uses client-side mechanisms such as:

* informers
* local caches
* work queues

For example:

```text
kube-controller-manager
│
├── ReplicaSet Controller
│    ├── Informer
│    ├── Local Cache
│    └── Work Queue
│
├── Deployment Controller
│    ├── Informer
│    ├── Local Cache
│    └── Work Queue
│
├── Node Controller
│    ├── Informer
│    ├── Local Cache
│    └── Work Queue
│
├── EndpointSlice Controller
│    ├── Informer
│    ├── Local Cache
│    └── Work Queue
│
└── ...
```

These are software mechanisms inside the controller process. They are not independent Kubernetes components.

---

## How Does Reconciliation Actually Work?

The controller's job is not simply to react to an event and perform one fixed action.

An event is better understood as a signal saying:

> "Something changed. Re-check the state."

The controller then compares the desired state with the currently observed state and determines what action, if any, is necessary.

For example:

```text
Desired state:
3 replicas

Observed state:
2 replicas

        ↓

Controller reconciliation

        ↓

Create 1 Pod
```

If the observed state is already correct:

```text
Desired state:
3 replicas

Observed state:
3 replicas

        ↓

No action required
```

This is why Kubernetes controllers are called **control loops**.

They repeatedly perform the conceptual process:

```text
Observe
   ↓
Determine current state
   ↓
Compare with desired state
   ↓
Take corrective action if necessary
   ↓
Observe again
```

---

## Does the Controller Get the Actual Container State?

Normally, **no**.

This is a very important distinction.

A controller such as the ReplicaSet controller primarily works with **Kubernetes API objects**.

It does not normally ask:

```text
"Is container nginx actually running inside containerd?"
```

The kubelet is responsible for managing the actual workload state on its node.

The architecture can therefore be simplified as:

```text
                  kube-apiserver
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
     Controllers                    kubelet
          │                           │
          │                           │
          ▼                           ▼
   Kubernetes objects          Container runtime
   and cluster state            and node state
```

The controller works primarily at the **cluster/API-object level**.

The kubelet works at the **node/container level**.

---

## The Kubelet Is Different

The kubelet also communicates with the API server and observes Pod specifications assigned to its node.

For example:

```text
kube-apiserver
      │
      │ PodSpec
      ▼
    kubelet
      │
      │
      │ compares desired Pod state
      │ with actual node state
      ▼
container runtime
```

The kubelet therefore has to deal with two sides:

```text
API PodSpec
   ↓
kubelet
   ↓
Desired Pod state vs actual node/container state
   ↓
Container Runtime
```

For example:

```text
API Server says:
Pod B should exist on Node A

Container runtime says:
Pod B does not exist

        ↓

kubelet
        ↓

Create/start Pod B
```

This is different from a controller such as the ReplicaSet controller, which normally does not directly inspect the container runtime.

---

## Complete Mental Model

The easiest way to remember the architecture is:

```text
                       kube-apiserver
                              │
                   Kubernetes API / Watch
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
       Controller side                     kubelet
              │                               │
          Informer                            │
              │                               │
          Local Cache                         │
              │                               │
         Work Queue                           │
              │                               │
         Controller                           │
              │                               │
       Reconciliation                         │
              │                               │
              └────────── API changes ────────┘
                                              │
                                              ▼
                                      Container Runtime
```

The key concepts are:

* **API Server** → provides the Kubernetes API and Watch mechanism.
* **Informer** → client-side mechanism that uses LIST/WATCH and maintains a local cache.
* **Local Cache** → locally stored view of the resources being watched.
* **Work Queue** → holds resources that should be reconciled.
* **Controller** → processes queued work and reconciles desired and observed Kubernetes state.
* **Kubelet** → observes Pod specifications assigned to its node and reconciles them with the actual state of the node/container runtime.

### The most important sentence

> **Kubernetes components do not normally discover changes by constantly polling the API server. They use the API's Watch mechanism; controllers commonly use informers to maintain local caches and work queues, and their reconciliation loops use that information to decide what actions are necessary.**
