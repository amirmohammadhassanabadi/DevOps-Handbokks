# Kubernetes CLI

The **Kubernetes CLI (Command Line Interface)** is the primary command-line interface used to interact with a Kubernetes cluster. The official Kubernetes CLI is ***kubectl***.

`kubectl` is a **client application** that communicates with the **Kubernetes API server (kube-apiserver)**. It allows users and administrators to create, inspect, update, and delete Kubernetes resources such as Pods, Deployments, Services, ConfigMaps, and Nodes.

`kubectl` does **not** communicate directly with etcd or other control-plane components. Instead, it sends requests to the Kubernetes API server.

## kubeconfig

When `kubectl` is executed, it uses a configuration called **kubeconfig** to determine how and where to communicate with the cluster.

The default location is:

```bash
~/.kube/config
```

A kubeconfig can contain:

* **Clusters** — Kubernetes API server endpoints and related cluster information
* **Users** — authentication information used to access clusters
* **Contexts** — combinations of a cluster, user, and optionally a default namespace

For example, a context can determine that `kubectl` should use the `production` cluster with a particular user and the `production` namespace.

The active context can be viewed with:

```bash
kubectl config current-context
```

Available contexts:

```bash
kubectl config get-contexts
```

The current context can be changed with:

```bash
kubectl config use-context <context-name>
```

A default namespace can also be configured for the current context:

```bash
kubectl config set-context --current --namespace=production
```

## Communication with the Cluster

For example, when running:

```bash
kubectl get pods
```

the general flow is:

```text
kubectl
   │
   │ HTTPS API request
   ▼
kube-apiserver
   │
   ├── Authentication / Authorization
   ├── Admission / Validation
   │
   ▼
Kubernetes API state
   │
   ▼
kubectl
   │
   ▼
Formatted output
```

The API server is the central entry point to the Kubernetes API. Other Kubernetes components, users, automation systems, dashboards, and applications can also communicate with the cluster through this API.

---

# Imperative and Declarative Approaches

In Kubernetes and infrastructure management, **imperative** and **declarative** describe two different ways of managing system state.

- ## Imperative Approach

    In the **imperative approach**, you explicitly issue commands that perform specific operations on Kubernetes resources.

    For example:

    ```bash
    kubectl run nginx --image=nginx
    ```

    This directly requests Kubernetes to create a Pod named `nginx` using the `nginx` image.

    Another example:

    ```bash
    kubectl scale deployment nginx --replicas=3
    ```

    This directly requests Kubernetes to change the Deployment's desired replica count to `3`.

    The user explicitly specifies the **operation to perform**, while Kubernetes handles the underlying implementation.

    ### Characteristics

    * Command-based
    * Performs a specific operation
    * Useful for quick testing and troubleshooting
    * Convenient for one-time changes
    * Changes are harder to track and reproduce if commands are not recorded
    * Less suitable as the primary method for managing complex production configurations

    **Simple mental model:**

    > **Imperative = “Perform this operation.”**

    ---

- ## Declarative Approach

    In the **declarative approach**, you define the **desired state** of a Kubernetes resource, and Kubernetes determines how to achieve and maintain that state.

    For example:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
            - name: nginx
              image: nginx
    ```

    The configuration can then be submitted with:

    ```bash
    kubectl apply -f deployment.yaml
    ```

    The user specifies that the desired state is a Deployment with **three nginx Pod replicas**. The user does not specify the individual steps required to create and maintain those Pods.

    Kubernetes controllers continuously reconcile the actual state with the desired state. For example, if one of the three Pods becomes unavailable, the appropriate controller works to create a replacement so that the Deployment returns to its desired state.

    ### Characteristics

    * State-based
    * Configuration-based
    * Describes the desired state
    * Kubernetes determines how to reach that state
    * Easy to version and reproduce
    * Well suited for production, CI/CD, and GitOps workflows

    **Simple mental model:**

    > **Declarative = “Make the system look like this.”**

---

### Key Difference

| Imperative                         | Declarative                                          |
| ---------------------------------- | ---------------------------------------------------- |
| Specifies an operation             | Specifies desired state                              |
| Usually command-based              | Usually configuration-based                          |
| `kubectl run`                      | `kubectl apply -f`                                   |
| `kubectl scale`                    | Deployment manifest with `replicas: 3`               |
| Good for quick/one-time operations | Good for persistent configuration                    |
| Changes may be harder to reproduce | Configuration can be version-controlled              |
| User initiates each operation      | Kubernetes continuously reconciles the desired state |

The important distinction is **operation vs. desired state**:


> **Imperative:** "Scale this Deployment to 3 replicas."
> 
> **Declarative:** "This Deployment should have 3 replicas."


In the declarative case, Kubernetes continues working toward that state even after the original `kubectl apply` command has finished.

### Simple Analogy

**Imperative:**

> “Cook rice, boil the water, add rice, and stir.”

You specify the operations to perform.

**Declarative:**

> “I want a bowl of cooked rice.”

You specify the desired result, and the system determines the necessary steps.

### In Real Kubernetes Operations

Both approaches are useful, and declarative does **not** mean that imperative commands should never be used.

**Imperative operations** are commonly useful for:

* Quick testing
* Troubleshooting
* One-time operational changes
* Interactive administration

**Declarative management** is commonly preferred for:

* Production workloads
* Configuration management
* CI/CD
* GitOps
* Version-controlled infrastructure

A common production workflow is therefore:

```text
YAML / Configuration
        ↓
      Git
        ↓
   CI/CD or GitOps
        ↓
   Kubernetes API
        ↓
   Desired State
        ↓
     Controllers
        ↓
    Actual State
```

The key idea is that **imperative commands request individual operations, while declarative configuration defines the state Kubernetes should continuously maintain**.

---

## Common kubectl Capabilities

Beyond creating and modifying resources, `kubectl` provides operational and troubleshooting capabilities.

| Command                | Purpose                                        |
| ---------------------- | ---------------------------------------------- |
| `kubectl get`          | Display resources                              |
| `kubectl describe`     | Show detailed information about a resource     |
| `kubectl create`       | Create resources imperatively                  |
| `kubectl apply`        | Create or update resources declaratively       |
| `kubectl delete`       | Delete resources                               |
| `kubectl logs`         | View container logs                            |
| `kubectl exec`         | Execute a command inside a container           |
| `kubectl port-forward` | Forward a local port to a Pod or Service       |
| `kubectl edit`         | Edit an existing resource interactively        |
| `kubectl patch`        | Modify specific fields of an existing resource |
| `kubectl config`       | Manage kubeconfig contexts and settings        |

`kubectl` can interact with both **namespaced and cluster-scoped resources**, provided the authenticated user has the required permissions.