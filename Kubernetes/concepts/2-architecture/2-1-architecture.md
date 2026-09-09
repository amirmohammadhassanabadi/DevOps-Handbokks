

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

## kube-controller-manager

The **kube-controller-manager** is a control plane component that runs multiple controllers. Each controller continuously watches Kubernetes resources and reconciles the actual state toward the desired state.

For example, the ReplicaSet Controller ensures that the desired number of Pods exists.

**Scenario:**

- Desired state → replicas: 3
- Actual state → 2 Pods exist

Controller action → create 1 new Pod

Other controllers include:

- Deployment Controller
- ReplicaSet Controller
- Node Controller
- Job Controller
- EndpointSlice Controller
- PersistentVolume Controller
- Namespace Controller
- And many others

**Important:**

> The **kube-controller-manager** does not process user API requests. The kube-apiserver handles API requests, while controllers watch the state through the API and take corrective actions.

**Example:**


    User
      ↓
    kubectl
      ↓
    kube-apiserver
      ↓
    etcd
      ↓
    ReplicaSet Controller
      ↓
    Detects: 2 Pods instead of 3
      ↓
    Creates a new Pod
      ↓
    kube-apiserver
      ↓
    kube-scheduler
      ↓
    Assigns Pod to a node
      ↓
    kubelet
      ↓
    Container Runtime
      ↓
    Pod runs

### Deep Dive in kube-controller-manager:

The **kube-controller-manager** is a core control plane component that runs multiple **controllers**. Each controller continuously watches Kubernetes resources through the API server and uses a **reconciliation loop** to bring the actual state toward the desired state.

User requests are handled by the **kube-apiserver**, not the controller manager. The API server stores and retrieves Kubernetes objects, while controllers observe changes and take corrective actions when necessary.

The controller manager contains specialized controllers, each responsible for a specific part of the cluster. Examples include:

* **Deployment Controller** → manages Deployments and their ReplicaSets
* **ReplicaSet Controller** → maintains the desired number of Pods
* **Node Controller** → monitors node health
* **Job Controller** → ensures Jobs complete successfully
* **EndpointSlice Controller** → maintains EndpointSlices based on Services and Pods
* **PersistentVolume Controller** → manages PV/PVC binding and lifecycle

These controllers provide many of Kubernetes' automation capabilities, including **self-healing, scaling, rolling updates, and resource management**.

**Mental model:**

> **kube-apiserver** → accepts and processes API requests
>
> **etcd** → stores persistent cluster state
>
> **kube-controller-manager** → continuously reconciles actual state with desired state

---

## cloud-controller-manager:

The cloud-controller-manager (CCM) integrates Kubernetes with a cloud provider's APIs and manages cloud-specific functionality, such as:

- Load balancers
- Cloud storage integration
- Node lifecycle
- Network routes

It is commonly used with cloud providers such as:

- AWS
- GCP
- Azure

The cloud-controller-manager is **optional** and is generally not needed for bare-metal clusters unless a cloud provider integration is being used.

---
