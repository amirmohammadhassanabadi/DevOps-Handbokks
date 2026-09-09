# Container Runtime

The container runtime is the software responsible for running containers on a Kubernetes node. It handles the low-level operations required to create, start, stop, and remove containers.

Kubernetes communicates with the container runtime through the **Container Runtime Interface (CRI)**.

Common container runtimes include:

* **containerd**
* **CRI-O**

The container runtime is responsible for tasks such as:

* pulling container images
* creating and starting containers
* stopping and removing containers
* reporting container and Pod runtime status to the kubelet

The **kubelet does not directly manage containers**. It instructs the container runtime through the CRI, and the runtime performs the actual container operations.

The simplified relationship is:

```text
kubelet
   ↓
CRI
   ↓
container runtime
   ↓
containers
```

**What about Docker?**

Docker is not a Kubernetes container runtime in modern Kubernetes. Kubernetes removed its built-in Docker integration (**dockershim**) starting with Kubernetes 1.24.

Docker can still be used as a development tool, and Docker-built images can run on Kubernetes. However, Kubernetes nodes normally use a CRI-compatible runtime such as **containerd** or **CRI-O**.

### Summery

- **CRI** → the standardized interface/protocol Kubernetes uses to communicate with a container runtime.
- **containerd** → the actual container runtime that receives those requests and manages containers.
- **runc** → the low-level OCI runtime that actually creates and runs the container processes.