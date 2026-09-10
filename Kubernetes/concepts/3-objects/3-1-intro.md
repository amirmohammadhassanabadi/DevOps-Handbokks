# Object

In Kubernetes, an **object** is a persistent representation of a resource in the Kubernetes API. It describes a piece of the cluster's state and allows Kubernetes to store and manage information about that resource.

When you create or modify an object, you are declaring how you want that resource to exist in the cluster. Kubernetes then continuously works to make the actual state match the desired state defined by the object.

For example, Kubernetes objects can represent:

* a **Pod** running one or more containers
* a **Deployment** managing a desired number of Pod replicas
* a **Service** providing a stable network endpoint for a group of Pods
* a **ConfigMap** storing non-sensitive configuration data

Objects are represented through the Kubernetes API and are commonly created or modified using **YAML or JSON manifests**.

A typical manifest contains fields such as:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 3
```

The main fields are:

* **`apiVersion`** → identifies the API group and version that defines the object's schema.
* **`kind`** → identifies the type of object being requested.
* **`metadata`** → contains identifying and organizational information such as the object's name, namespace, labels, annotations, and UID.
* **`spec`** → describes the desired state of the object.

Many Kubernetes objects also have a **`status`** field, which represents the current or observed state of the resource.

For example:

```text
spec
  ↓
"What I want"

status
  ↓
"What Kubernetes currently observes"
```

Controllers and other Kubernetes components use these states to determine whether reconciliation is required.

After an object is created through the API server, its persistent cluster state is stored in **etcd**. Kubernetes components observe changes through the API server and perform the actions necessary to bring the cluster toward the desired state.

For example:

```text
Manifest
   ↓
kube-apiserver
   ↓
Object stored in etcd
   ↓
Controllers / other components observe object
   ↓
Reconciliation
   ↓
Actual cluster state
```

It is important to understand that the **manifest is not the object itself**. The manifest is a representation of the configuration you send to the Kubernetes API. The Kubernetes API server validates and processes that request, and the resulting object becomes part of the cluster's persistent state.

In short:

> **A Kubernetes object is a persistent API representation of a resource that Kubernetes uses to describe, store, observe, and manage cluster state.**

# Manifest

A Kubernetes **manifest** is a configuration file, usually written in YAML, that describes a Kubernetes object and the desired state that Kubernetes should maintain for that object.

Instead of manually creating or modifying resources through individual commands, a manifest allows you to define the configuration of a resource in a declarative form. It specifies **what type of object should exist** and **how that object should be configured**, rather than providing a sequence of steps for creating it.

For example, a manifest can define:

* a **Pod** and the containers it should run
* a **Deployment** and the number of Pod replicas it should maintain
* a **Service** and how an application should be exposed
* a **ConfigMap** and the configuration data it should contain

A typical manifest contains fields such as **`apiVersion`**, **`kind`**, **`metadata`**, and, when applicable, **`spec`**.

Once the manifest is written, it can be submitted to the cluster using a command such as:

```bash
kubectl apply -f file.yaml
```

`kubectl` sends the requested object configuration to the **kube-apiserver**. The API server validates and processes the request, and the resulting object is persisted as cluster state in **etcd**. Kubernetes components then observe that state and reconcile the cluster toward the desired state described by the object.

The important point is that a manifest describes **desired state**, not a sequence of instructions. This makes Kubernetes configuration declarative and provides benefits such as **reproducibility, version control, automation, and consistent deployments**.

## YAML and JSON

Kubernetes manifests are most commonly written in **YAML**, although **JSON** is also supported.

YAML is preferred for manifests because it is generally easier for humans to read and maintain. Kubernetes APIs, however, use JSON as their primary wire/serialization format. When tools such as `kubectl` process a YAML manifest, the configuration is parsed and represented in the form required for communication with the Kubernetes API.

Therefore, YAML and JSON are two supported representations of Kubernetes API objects; YAML is primarily convenient for humans, while JSON is commonly used in API communication.

## Manifest Top-Level Fields

A typical Kubernetes manifest contains the following fields:

* **`apiVersion`** → specifies the API group and version used for the object, such as `v1` or `apps/v1`.

* **`kind`** → specifies the type of Kubernetes object being created, such as `Pod`, `Deployment`, `Service`, or `ConfigMap`.

* **`metadata`** → contains information that identifies and describes the object, such as `name`, `namespace`, `labels`, and `annotations`.

* **`spec`** → defines the desired configuration or state of the object. Its contents depend on the object type. For example, a Pod's `spec` defines its containers, while a Deployment's `spec` defines its replica count and Pod template.

For example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
```

### `spec` vs `status`

Kubernetes objects commonly follow a **desired state vs. observed state** model:

* **`spec`** → the desired state: what you want Kubernetes to achieve.
* **`status`** → the observed state: what Kubernetes currently reports about the resource.

`status` should not normally be written as part of the user's desired configuration. Kubernetes components, such as controllers, update it to report the current state of the object.

For example, a Deployment may contain:

```yaml
spec:
  replicas: 3
```

This means:

> The desired state is to have 3 replicas.

Its `status` may contain information such as:

```yaml
status:
  replicas: 3
  updatedReplicas: 3
  availableReplicas: 3
```

This indicates that Kubernetes currently observes three replicas, all three have been updated to the current Deployment revision, and all three are available.

If the Deployment currently has only two available replicas, for example:

```yaml
spec:
  replicas: 3

status:
  replicas: 3
  availableReplicas: 2
```

the desired state has not yet been fully achieved. The Deployment controller continues reconciling the resource until the observed state matches the desired state, assuming the cluster can satisfy that requirement.

Therefore, the simplest mental model is:

**`spec` → what you want**

**`status` → what Kubernetes observes**

This distinction is fundamental to Kubernetes' declarative and reconciliation-based architecture.

## Manifest vs. Object

A **manifest and a Kubernetes object are related but not identical**.

The manifest is the configuration representation that you provide to Kubernetes. When you submit it through the Kubernetes API, the API server validates and processes it, and the resulting **Kubernetes object** becomes part of the cluster's persistent API state.

The basic flow is:

```text
Manifest
   ↓
kubectl
   ↓
kube-apiserver
   ↓
Kubernetes API Object
   ↓
etcd
   ↓
Controllers / Other Components
   ↓
Reconciliation
   ↓
Actual Cluster State
```

In short:

> A **manifest** is a declarative configuration used to create or modify a Kubernetes object, while the **object** is the persistent representation of that resource in the Kubernetes API.
