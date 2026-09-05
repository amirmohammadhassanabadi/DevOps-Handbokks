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

## etcd

**etcd** is a distributed, strongly consistent key-value store that Kubernetes uses to persist the state of its API objects. It contains:

- all Kubernetes objects
- configuration
- secrets
- cluster metadata

**Characteristics:**

- distributed
- strongly consistent
- highly available

### Deep Dive in etcd:

**etcd** is a distributed NoSQL key‑value database that Kubernetes uses as its primary datastore. It stores the persistent state of Kubernetes API objects in a structured format that Kubernetes components understand. All cluster configuration and metadata are persisted in **etcd**, including information about Pods, Deployments, Services, ConfigMaps, Secrets, nodes, and many other Kubernetes resources. When a user defines a resource, such as requesting an Nginx Pod with specific `replicas`, container images, volumes, or other configuration, this desired state is ultimately stored in **etcd** through the Kubernetes API.

In the Kubernetes architecture, the **kube‑apiserver** is the only component that communicates directly with **etcd** (Other components normally go through the API server.). When a user or tool sends a request, such as querying the list of Pods, the request first reaches the API server. The API server then reads the required data from **etcd** and returns the response. Other control plane components, such as the **kube-scheduler**, **kube-controller** and **kubelets**, do not communicate with **etcd** directly. Instead, they interact with the cluster by watching and updating resources through the API server, which then persists any changes in **etcd**.

Because **etcd** contains the complete state of the Kubernetes control plane, it is considered authoritative persistent datastore for Kubernetes state. If the **etcd** data is lost or becomes empty, Kubernetes effectively loses its knowledge of the cluster’s resources and configuration. Even if containers are still running on nodes, Kubernetes may no longer recognize or manage them because the definitions and metadata that describe those resources no longer exist in **etcd**. For this reason, protecting **etcd** is critical for cluster reliability.

In production environments, **etcd** is typically configured in a high‑availability cluster. Instead of a single instance, multiple **etcd** members run on different control plane nodes and replicate data between them using a consensus algorithm. This distributed design ensures that if one **etcd** node fails, the remaining members can continue serving the cluster without losing data. As a result, clusters with multiple control plane nodes often run multiple **etcd** instances to maintain redundancy, consistency, and fault tolerance for the Kubernetes control plane state.

The name **etcd** comes from a naming convention within the Linux directory structure. In UNIX, all system configuration files for a single system are contained in a folder called `/etc` and **d** stands for **distributed**.

### etcd implementation:

There are 2 implementation for etcd:

- ### Stacked etcd Topology
   In a Kubernetes cluster using the stacked **etcd** topology, each control plane node runs its own **etcd** member in addition to the control plane components (kube-apiserver, kube-scheduler, and kube-controller-manager). These **etcd** members do not act as independent databases; instead, they form a single distributed **etcd** cluster that stores the Kubernetes cluster state.

   A common misconception is that as long as one **etcd** member remains running, the cluster can continue operating. This is not true. **etcd** uses the **Raft consensus algorithm**, which requires a majority (quorum) of voting members to agree before processing operations. Without a quorum, the cluster cannot safely read or write the cluster state, preventing split-brain scenarios and ensuring data consistency (A healthy etcd cluster can serve certain reads without the same consensus requirements as writes, but Kubernetes needs a healthy etcd quorum for reliable cluster-state operations).

   For a cluster with N etcd voting members, the required quorum is more than half of the members. Therefore, the maximum number of failures the cluster can tolerate is:

   Maximum failures tolerated = `floor((N−1)/2)`

   This results in the following fault tolerance:

   | Number of etcd Members | Quorum Required | Maximum Failures Tolerated |
   | --- | --- | ---|
   | 1 | 1 | 0
   | 2 | 2 | 0
   | 3 | 2 | 1
   | 4 | 3 | 1
   | 5 | 3 | 2
   | 6 | 4 | 2
   | 7 | 4 | 3

   Notice that increasing the number of **etcd** members from an odd number to the next even number does not improve fault tolerance. For example, a 4-member **etcd** cluster can still tolerate only one failure, just like a 3-member cluster. Fault tolerance increases only when another member is added to create the next odd-sized cluster (for example, from 3 to 5 members). For this reason, production Kubernetes clusters typically use 1, 3, 5, or 7 voting **etcd** members. A 2-member or 4-member **etcd** cluster is generally not recommended because it does not provide better fault tolerance than the previous odd-sized configuration while introducing additional complexity.

   It is also important to distinguish between the control plane and **etcd**. Losing quorum in **etcd** does not immediately stop running workloads. Existing Pods on worker nodes continue running as long as the nodes remain healthy. However, because the API server can no longer reliably access or update the cluster state, management operations such as creating, deleting, scheduling, or updating resources cannot proceed until quorum is restored.

- ### external etcd topology
   In the external etcd topology, the Kubernetes control plane and etcd are separated into different sets of machines. This is a more advanced architecture that is common in larger production environments. Instead of running etcd on each control plane node, you have a dedicated etcd cluster:
   
   ```
                            +---------------------------+
                            |      Load Balancer        |
                            | (API VIP / etcd VIP/DNS)  |
                            +-------------+-------------+
                                          |
             _____________________________|_____________________________
            |                             |                             |
   +--------v------------+      +---------v-----------+      +----------v----------+
   | Control Plane 1     |      | Control Plane 2     |      | Control Plane 3     |
   | +-----------------+ |      | +-----------------+ |      | +-----------------+ |
   | | kube-apiserver  | |      | | kube-apiserver  | |      | | kube-apiserver  | |
   | | kube-scheduler  | |      | | kube-scheduler  | |      | | kube-scheduler  | |
   | | control-manager | |      | | control-manager | |      | | control-manager | |
   | +-------|---------+ |      | +-------|---------+ |      | +--------|--------+ |
   +---------|-----------+      +---------|-----------+      +----------|----------+
             |                            |                             |
             |              (API Requests to etcd Cluster)              |
             +----------------------------+-----------------------------+
                                          |
                         +----------------+----------------+
                         |      External etcd Cluster      |
                         |  (Running on dedicated nodes)   |
                         +----------------+----------------+
                                          |
                         +----------------+----------------+
                         |                |                |
               +---------v-------+ +------v-------+ +------v---------+
               |    etcd Node 1  | |  etcd Node 2 | |   etcd Node 3  |
               |  (etcd member)  | | (etcd member)| |  (etcd member) |
               +-----------------+ +--------------+ +----------------+
   ```
   **Why use external etcd?**
   The main benefit is **isolation**. In a stacked topology, a control plane node failure also removes an etcd member. In an external topology, the control plane and the database layer fail independently.

   **Fault tolerance**
   The important thing to remember is:

   > High availability depends on the etcd cluster size, not just the number of control plane nodes.

   **For example:**

   | Control Plane Nodes | External etcd Members | etcd Failures Tolerated |
   |---|---|---|
   | 2 | 3 | 1 |
   | 3 | 3 | 1 |
   | 5 | 5 | 2 |

   **Trade-offs**
   - **Advantages**
      - Better separation of responsibilities.
      - Control plane failures do not automatically remove etcd members.
      - Easier to scale control plane and etcd independently.
      - Often preferred for large production clusters.
   - **Disadvantages**
      - More machines to manage.
      - More networking complexity.
      - More operational overhead.

### When should we use which one of them?

- **Stacked etcd:** Recommended for most clusters because it is simpler, requires fewer machines, and has lower operational complexity.
- **External etcd:** Useful when you need to isolate etcd from the control plane and manage them independently, but it requires additional infrastructure and operational overhead.

**Key point:** External etcd provides isolation and flexibility, not inherently better fault tolerance.

Many people say "External etcd is more highly available." → That's not necessarily true. 
**For example:**

- 3 control plane nodes with stacked etcd → tolerate 1 etcd member failure.
- 3 control plane nodes + 3 external etcd nodes → also tolerate 1 etcd member failure.

The etcd fault tolerance is the same because both have three voting etcd members.

The advantage of external etcd is isolation and operational flexibility, not inherently greater fault tolerance.

---

## kube-scheduler:

The **kube-scheduler** is a control plane component responsible for deciding which node should run an unscheduled Pod.

When a new Pod is created:

1. The Pod is created without a node assignment.
2. The scheduler detects the unscheduled Pod.
3. It evaluates the available nodes.
4. It selects the most suitable node.
5. It assigns the Pod to that node through the API server.

Scheduling decisions can consider:

- CPU and memory requests
- Node labels and selectors
- Taints and tolerations
- Affinity and anti-affinity rules
- Node conditions and resource availability
- Other scheduling constraints

> The **scheduler** does **not** create or run the Pod. It only decides where the Pod should run.

### Deep Dive in Kube-Scheduler:

The **kube-scheduler** is a control plane component responsible for deciding on which node a newly created Pod should run. Its responsibility is limited to scheduling decisions; it does not create Pods or manage their lifecycle. When a user requests a workload such as creating an Nginx Pod, the request is first processed through the kube-apiserver and stored in etcd. Controllers then ensure that the Pod object exists, but if the Pod does not yet have an assigned node, it is considered unscheduled. At this point the kube-scheduler becomes responsible for selecting a suitable node for that Pod.

To make this decision, the **scheduler** watches the API server for Pods that do not yet have a node assignment. When it detects such a Pod, it retrieves information about available nodes and their current resource usage through the API server. This information includes factors such as CPU capacity, memory availability, resource requests from other Pods, node conditions, and various scheduling constraints. Using this data, the scheduler runs a scheduling algorithm that evaluates possible nodes and selects the most appropriate one according to the configured policies and scoring rules. The default behavior typically prefers nodes with sufficient resources and balanced utilization, but administrators can modify the scheduling policies or extend them with custom plugins.

Once the scheduler selects a node, it writes that decision back to the API server by updating the Pod’s specification with the chosen node name. The kubelet running on that node then observes the assignment through the API server and proceeds to create and run the Pod’s containers using the container runtime.

Scheduling decisions are made each time a new Pod needs placement. If a Pod is deleted and recreated, such as during scaling operations, rolling updates, or controller-driven replacement, the new Pod is again unscheduled and must be evaluated by the scheduler. In this process the scheduler recalculates the best node based on the current cluster conditions, which may lead to a different node being selected than the one used previously. This dynamic scheduling process helps Kubernetes distribute workloads efficiently across the cluster and adapt to changing resource availability.

---