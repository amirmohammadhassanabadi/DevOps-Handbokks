## What the Official Architecture Diagram Implies

The Kubernetes architecture documentation groups the major components into two broad categories:

### Control Plane Components

* **kube-apiserver**
* **kube-scheduler**
* **kube-controller-manager**
* **etcd**
* **cloud-controller-manager**

### Node Components

* **kubelet**
* **kube-proxy**
* **container runtime**

These categories describe the **role** of the components rather than requiring every component to be deployed in exactly the same way on every cluster.

| Component                   | Worker Nodes | Control Plane Nodes | Typical Deployment                            |
| --------------------------- | -----------: | ------------------: | --------------------------------------------- |
| **kubelet**                 |          Yes |                 Yes | System service                                |
| **kube-proxy**              |          Yes |             Usually | DaemonSet Pod                                 |
| **container runtime**       |          Yes |                 Yes | Host-level service                            |
| **kube-apiserver**          |           No |                 Yes | Commonly static Pod                           |
| **kube-scheduler**          |           No |                 Yes | Commonly static Pod                           |
| **kube-controller-manager** |           No |                 Yes | Commonly static Pod                           |
| **etcd**                    |           No |                Yes* | Commonly static Pod in stacked control planes |

> **etcd** may instead run on dedicated external nodes or be managed outside the Kubernetes cluster.

Node components are generally installed on every node because every node needs the functionality required to run workloads. However, **kube-proxy is not mandatory** when another networking implementation provides equivalent Service networking functionality.

---

## Architecture Summary

Kubernetes separates responsibilities between the **control plane** and **nodes**.

### Control Plane

The control plane is responsible for managing the cluster's overall state and making decisions about what should happen.

Its major responsibilities include:

* exposing the Kubernetes API
* storing persistent cluster state
* scheduling unscheduled Pods
* running controllers that maintain desired state
* handling cloud-specific control-plane responsibilities when applicable

### Nodes

Nodes provide the environment where workloads actually run.

Their major responsibilities include:

* running Pods and containers
* managing the local workload state
* providing container runtime functionality
* providing node-level networking functionality when kube-proxy or an equivalent is used
* reporting node and Pod status back to the control plane

A useful mental model is:

```text
                CONTROL PLANE
        ┌─────────────────────────┐
        │ kube-apiserver          │
        │ scheduler               │
        │ controllers             │
        │ etcd                    │
        │ cloud-controller-manager│
        └────────────┬────────────┘
                     │
             cluster decisions
                     │
                     ▼
              ┌─────────────┐
              │    Nodes    │
              ├─────────────┤
              │ kubelet     │
              │ kube-proxy  │
              │ runtime     │
              │ Pods        │
              └─────────────┘
```

In simple terms:

> **The control plane manages the cluster's desired state and makes cluster-level decisions, while nodes execute and maintain the workloads assigned to them.**

---

# Kubernetes Components Running as Pods

In many Kubernetes installations, particularly clusters created with **kubeadm**, the main control-plane components run as **static Pods**.

Typical components include:

* **kube-apiserver**
* **kube-scheduler**
* **kube-controller-manager**
* **etcd** when using a stacked control-plane topology

These Pods are different from ordinary Pods because they are **created and managed directly by the kubelet from static-Pod manifests**, rather than being created by a Deployment, ReplicaSet, or another Kubernetes controller.

A common location for static-Pod manifests is:

```text
/etc/kubernetes/manifests/
```

For example:

```text
/etc/kubernetes/manifests/
                    ├── kube-apiserver.yaml
                    ├── kube-scheduler.yaml
                    ├── kube-controller-manager.yaml
                    └── etcd.yaml
```

The kubelet is configured to monitor this directory as a **static-Pod source**. When it detects a manifest, it ensures that the corresponding Pod is running on that node.

The resulting Pods can be observed through the Kubernetes API, for example:

```bash
kubectl get pods -n kube-system
```

However, their lifecycle is fundamentally different from an ordinary API-created Pod: the kubelet is responsible for managing the static Pod based on the local manifest.

---

## Why Static Pods Are Used for the Control Plane

Static Pods are useful for bootstrapping the control plane because they do not require the Kubernetes API server to already be functioning.

The dependency can therefore begin with:

```text
kubelet
   │
   │ reads local static-Pod manifests
   ▼
control-plane Pods
   │
   ├── kube-apiserver
   ├── etcd
   ├── scheduler
   └── controller-manager
```

Once the API server becomes available, these components participate in the normal Kubernetes control-plane architecture.

This allows a kubeadm-style control-plane node to **bootstrap itself without first requiring a functioning Kubernetes API**.

Static Pods are therefore particularly useful for components that are themselves required to establish the control plane.

> Static Pods are a common way of deploying control-plane components, but they are not a universal requirement of Kubernetes. Kubernetes allows other deployment models as well.

---

# kubelet Runs as a Host-Level Service

The kubelet is different from the control-plane components and ordinary application workloads.

A kubelet normally runs directly on the node as a **host-level system service**, commonly managed by `systemd`.

For example:

```text
Linux Node
│
├── kubelet
│
├── container runtime
│
└── Pods / containers
```

This arrangement is important because the kubelet is part of the mechanism that **starts and manages Pods on the node**. It therefore cannot depend on an ordinary Kubernetes Pod being available before it can perform its own job.

The bootstrap relationship is:

```text
Operating System
      │
      ├── starts kubelet
      │
      ▼
    kubelet
      │
      ├── starts/manages static Pods
      │
      └── manages ordinary Pods
             │
             ▼
       container runtime
```

This is why the kubelet is normally installed and started independently of the Kubernetes workloads it manages.

The important distinction is therefore:

> **The kubelet is a host-level node agent that exists outside the normal Pod lifecycle, while the kubelet can manage Pods—including static Pods—that make up other parts of the Kubernetes system.**

---

## Final Architecture Mental Model

At the highest level, the entire architecture can be understood as:

```text
                    Kubernetes API
                         │
                         ▼
                 ┌───────────────┐
                 │ Control Plane │
                 │               │
                 │ API Server    │
                 │ Scheduler     │
                 │ Controllers   │
                 │ etcd          │
                 └───────┬───────┘
                         │
              desired cluster state
                         │
          ┌──────────────┴──────────────┐
          ▼                             ▼
       Node A                         Node B
   ┌─────────────┐                ┌─────────────┐
   │ kubelet     │                │ kubelet     │
   │ kube-proxy* │                │ kube-proxy* │
   │ runtime     │                │ runtime     │
   │ Pods        │                │ Pods        │
   └─────────────┘                └─────────────┘

* when the cluster uses kube-proxy
```

The control plane maintains the **cluster-level desired state** and makes decisions. Each node's kubelet then works to make the desired state for that node match the **actual local state**.

This separation of responsibilities is the foundation of Kubernetes:

> **The control plane decides and coordinates; the nodes execute and maintain.**
