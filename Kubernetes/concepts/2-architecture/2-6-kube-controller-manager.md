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