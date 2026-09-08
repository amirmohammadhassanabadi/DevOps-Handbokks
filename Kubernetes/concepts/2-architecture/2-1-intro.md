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

---

# Worker Nodes

Worker nodes are the primary execution units within a Kubernetes cluster, responsible for hosting and running application workloads. To ensure seamless orchestration, every node must host three essential components:

- **Kubelet**
- **Kube-proxy**
- **Container Runtime**

> **Worker Node components are expected to run on every node.**