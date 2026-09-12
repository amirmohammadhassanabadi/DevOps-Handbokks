# Namespace

A **Namespace** in Kubernetes is a mechanism for logically organizing and separating namespaced resources within a cluster.

Namespaces allow a single Kubernetes cluster to be divided into multiple logical environments. For example, different teams or application environments such as `dev`, `staging`, and `production` can use separate namespaces while sharing the same underlying cluster infrastructure.

Namespaces are sometimes described as **“virtual clusters inside a cluster.”** This is useful as a mental model, but a namespace does **not** create an actual independent cluster. All namespaces still share the same Kubernetes control plane and underlying node infrastructure.

For example, a cluster might contain:

```text
Cluster
├── dev
│   ├── Pods
│   ├── Deployments
│   ├── Services
│   └── ConfigMaps
│
├── staging
│   ├── Pods
│   ├── Deployments
│   └── Services
│
└── production
    ├── Pods
    ├── Deployments
    └── Services
```

## Namespace Isolation

Namespaces provide **logical separation**, but a namespace by itself is not a complete security boundary.

Additional Kubernetes mechanisms can be used to control different aspects of isolation:

* **RBAC** → controls who can access or modify resources in a namespace.
* **ResourceQuota** → limits the amount of cluster resources that objects in a namespace can consume.
* **LimitRange** → defines default and maximum resource requests/limits for objects within a namespace.
* **NetworkPolicy** → controls network communication between Pods, including communication across namespaces.

Therefore, namespaces provide the organizational boundary, while other Kubernetes mechanisms can enforce access, resource, and network isolation.

## Namespaces and Resource Names

Most application resources are **namespaced resources**. A namespaced resource belongs to a particular namespace.

Resource names generally need to be unique within their namespace and resource type. The same name can therefore be used in different namespaces.

For example:

```text
dev/api
production/api
```

These are two different Pods because they belong to different namespaces.

A namespace does not, however, make different resource types share one global name space. For example, a Pod and a Service can both be named `api` in the same namespace.

## Default Namespaces

A Kubernetes cluster normally contains several namespaces created during cluster initialization:

* **`default`** → the default namespace used when no namespace is explicitly specified.
* **`kube-system`** → contains resources related to Kubernetes system components.
* **`kube-public`** → intended for resources that should be publicly readable across the cluster, including by unauthenticated users in configurations that allow anonymous access.
* **`kube-node-lease`** → contains Lease objects used by nodes for lightweight node heartbeats.

You can view them with:

```bash
kubectl get namespaces
```

or:

```bash
kubectl get ns
```

### Creating a Namespace

- A namespace can be created directly with `kubectl`:

    ```bash
    kubectl create namespace dev
    ```

- It can also be defined declaratively using a manifest:

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: dev
    ```

    Then apply it with:

    ```bash
    kubectl apply -f namespace.yaml
    ```

### Deleting a Namespace

A namespace can be deleted with:

```bash
kubectl delete namespace dev
```

Deleting a namespace also deletes the namespaced resources that belong to it. This makes namespace deletion a potentially destructive operation, so it should be performed carefully.

### Viewing Resources in a Namespace

To list Pods in a specific namespace:

```bash
kubectl get pods -n dev
```

The `-n` option is a short form of `--namespace`:

```bash
kubectl get pods --namespace=dev
```

You can also use the namespace field in a manifest:

```yaml
metadata:
  name: my-pod
  namespace: dev
```

When using imperative commands, the namespace can be specified with `-n`:

```bash
kubectl run nginx --image=nginx -n dev
```

### Setting the Default Namespace for kubectl

Instead of specifying `-n dev` for every command, you can configure the current kubectl context to use a particular namespace by default:

```bash
kubectl config set-context --current --namespace=dev
```

After this, a command such as:

```bash
kubectl get pods
```

will operate against the `dev` namespace unless another namespace is explicitly specified.

This setting affects the **current kubectl context**, not the Kubernetes cluster itself.

### Namespaced vs. Cluster-Scoped Resources

Not every Kubernetes resource belongs to a namespace.

Resources are generally divided into two categories:

- **Namespaced resources**

    These exist inside a specific namespace, such as:

    * Pods
    * Deployments
    * Services
    * ConfigMaps
    * Secrets
    * PersistentVolumeClaims

- **Cluster-scoped resources**

    These exist at the cluster level and do not belong to any namespace, such as:

    * Nodes
    * PersistentVolumes
    * Namespaces
    * StorageClasses
    * ClusterRoles
    * ClusterRoleBindings

    For example, a **Node** represents a machine that provides resources for workloads across the cluster. It therefore cannot belong to one particular namespace.
    
    A **PersistentVolume (PV)** is also cluster-scoped. It represents storage that is available to the cluster. A namespaced **PersistentVolumeClaim (PVC)** can request storage and bind to an appropriate PV.

The distinction can be visualized as:

```text
Cluster
│
├── Cluster-scoped resources
│   ├── Node
│   ├── PersistentVolume
│   ├── StorageClass
│   └── ClusterRole
│
└── Namespaces
    ├── dev
    │   ├── Pod
    │   ├── Deployment
    │   ├── Service
    │   └── PVC
    │
    └── production
        ├── Pod
        ├── Deployment
        ├── Service
        └── PVC
```

### Identifying Resource Scope

The `kubectl api-resources` command displays the resource types available through the Kubernetes API and provides information about their scope.

For example:

```bash
kubectl api-resources
```

The output includes columns such as:

* **NAME** → resource name used by the API
* **SHORTNAMES** → abbreviated names, such as `po` for Pods
* **APIVERSION** → API group and version
* **NAMESPACED** → whether the resource belongs to a namespace
* **KIND** → Kubernetes object kind

The `NAMESPACED` column is particularly useful for determining whether a resource is namespaced or cluster-scoped.

For example, you may see:

```text
NAME              SHORTNAMES   APIVERSION   NAMESPACED   KIND
pods              po           v1           true         Pod
services          svc          v1           true         Service
deployments       deploy       apps/v1      true         Deployment
nodes             no           v1           false        Node
persistentvolumes pv           v1           false        PersistentVolume
```

This distinction is important because it determines whether a resource can be referenced with a namespace and whether it is isolated within a particular namespace.

### Important Note About `kubectl get all`

Despite its name, `kubectl get all` does **not** literally display every resource type in a namespace.

It displays a predefined collection of commonly used resources, such as Pods, Services, Deployments, ReplicaSets, and StatefulSets. Resources such as ConfigMaps, Secrets, PVCs, and many other object types are not necessarily included.

For example:

```bash
kubectl get all -n dev
```

should therefore be understood as:

> Show the common workload and service resources in the `dev` namespace.

It should not be interpreted as:

> Show every Kubernetes object in the namespace.

### Summary

A **Namespace** provides a logical boundary for organizing namespaced resources within a Kubernetes cluster. It allows multiple teams, applications, or environments to share the same cluster while keeping their resources logically separated.

The important distinction is:

```text
Namespace
    ↓
Logical organization and resource boundary
    ↓
RBAC / Quotas / LimitRanges / NetworkPolicies
    ↓
More specific access, resource, and network isolation
```

Namespaces do not create separate Kubernetes clusters. They provide a logical partition within one cluster, while cluster-scoped resources remain outside individual namespaces.

---

## Namespace Troubleshooting Workflow

When troubleshooting a namespace, first determine its current state and then inspect the resources and control-plane conditions associated with it.

### 1. Check the Namespace Status

Start by checking whether the namespace exists and its current status:

```bash
kubectl get namespace dev
```

For more details:

```bash
kubectl describe namespace dev
```

Normally, a namespace should have a status such as:

```text
Active
```

If it is being deleted, its status will typically be:

```text
Terminating
```

### 2. Check Namespace Events

Inspect events associated with the namespace:

```bash
kubectl get events -n dev --sort-by=.lastTimestamp
```

Events and conditions can provide clues about failed cleanup, admission problems, unavailable APIs, or other issues.

### 3. Check for Remaining Resources

When a namespace is stuck in `Terminating`, determine whether resources still exist inside it.

A useful first check is:

```bash
kubectl get all -n dev
```

However, `get all` does not include every resource type. To discover all namespaced resource types supported by the cluster:

```bash
kubectl api-resources --verbs=list --namespaced -o name
```

You can then query those resources:

```bash
for resource in $(kubectl api-resources --verbs=list --namespaced -o name); do
    kubectl get "$resource" -n dev --ignore-not-found
done
```

This can reveal resources that are not shown by `kubectl get all`, such as ConfigMaps, Secrets, PVCs, or custom resources.

### 4. Check for Finalizers

If resources remain or the namespace cannot complete deletion, inspect finalizers.

For the namespace:

```bash
kubectl get namespace dev -o yaml
```

Look for:

```yaml
spec:
  finalizers:
```

Finalizers are mechanisms that prevent an object from being deleted until a required cleanup operation has been completed.

Resources inside the namespace can also have finalizers:

```bash
kubectl get <resource> <name> -n dev -o yaml
```

Check for:

```yaml
metadata:
  finalizers:
```

A namespace may remain in `Terminating` when Kubernetes is unable to complete the cleanup associated with a finalizer.

### 5. Check for Unavailable API Resources

Namespace deletion requires Kubernetes to discover and process the resources belonging to that namespace.

Check the API resources available in the cluster:

```bash
kubectl api-resources
```

If a CRD or aggregated API service is unavailable, Kubernetes may be unable to successfully enumerate resources during namespace cleanup.

For custom resources, check the corresponding CRDs:

```bash
kubectl get crd
```

And check their status:

```bash
kubectl describe crd <crd-name>
```

For aggregated APIs, inspect:

```bash
kubectl get apiservice
```

Look for services whose `AVAILABLE` status is `False`.

### 6. Check the Namespace Controller

Namespace lifecycle management is handled by the **kube-controller-manager**.

If the namespace remains stuck despite the resources and API services appearing healthy, inspect the controller-manager logs.

For a kubeadm-style control plane where the controller-manager runs as a static Pod:

```bash
kubectl get pods -n kube-system
```

Then inspect its logs:

```bash
kubectl logs -n kube-system <kube-controller-manager-pod>
```

The exact method depends on how the control plane is deployed.

### 7. Remove a Problematic Finalizer Only When Necessary

Do **not** immediately remove finalizers just to force deletion.

First determine:

1. Which object has the finalizer.
2. Why the finalizer was added.
3. Which cleanup operation it is waiting for.
4. Why that cleanup operation cannot complete.

If the responsible controller is permanently unavailable or the cleanup is no longer required, removing the finalizer may be appropriate.

For example, a resource's finalizers can be inspected with:

```bash
kubectl get <resource> <name> -n dev -o json
```

If you have confirmed that the finalizer is stale and safe to remove, it can be patched:

```bash
kubectl patch <resource> <name> -n dev \
  --type=json \
  -p='[{"op":"remove","path":"/metadata/finalizers"}]'
```

The exact operation depends on the resource and its finalizer structure.

### 8. Verify Deletion

After resolving the underlying problem, check the namespace again:

```bash
kubectl get namespace dev
```

Then verify that the namespace and its namespaced resources are gone:

```bash
kubectl get namespaces
```

If necessary, check whether objects from the namespace are still discoverable.

### Namespace `Terminating` Troubleshooting Flow

The overall workflow can be summarized as:

```text
Namespace stuck in Terminating
            │
            ▼
     Check namespace status
            │
            ▼
     Check events/conditions
            │
            ▼
   Check remaining resources
            │
            ▼
     Check for finalizers
            │
            ▼
 Check CRDs / APIService health
            │
            ▼
 Check kube-controller-manager
            │
            ▼
     Fix underlying issue
            │
            ▼
      Verify deletion
```

Only when the underlying cleanup mechanism is understood and the finalizer is confirmed to be stale should manually removing a finalizer be considered.

**Important:** Force-removing namespace finalizers can leave resources or external infrastructure behind. It should therefore be treated as a recovery procedure, not the normal method for deleting a namespace.
