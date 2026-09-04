Kubernetes is designed as a distributed system that manages containerized applications across a group of machines called a cluster.

A Kubernetes cluster consists of two main components:

- **Control Plane:** manages and coordinates the cluster, including scheduling workloads and maintaining the desired state.
- **Worker Nodes:** provide the compute resources where application workloads run.

A cluster is therefore a group of physical or virtual machines that work together and are managed by Kubernetes as a single system.


> A **distributed system** is a system where multiple independent machines work together as one system. For Kubernetes, Multiple nodes work together and are managed as a single cluster.

# Control Plane

The control plane is responsible for managing and maintaining the desired state of the cluster. It:

- receives and processes API requests
- schedules workloads onto nodes
- monitors the cluster state
- performs reconciliation to bring the actual state toward the desired state

Main components:

- **kube-apiserver**
- **etcd**
- **kube-scheduler**
- **kube-controller-manager**
- **cloud-controller-manager** (optional; used when integrating with a cloud provider)

## kube-apiserver

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

### Authentication

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

### Authorization

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

### Validation

The API server validates that the requested object is structurally and semantically valid.

For example, Kubernetes validates things such as:

- API version
- resource kind
- field structure
- field types
- required fields
- valid values

If validation fails, the API server rejects the request.

### Storage

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

### kube-apiserver vs kube-controller-manager

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