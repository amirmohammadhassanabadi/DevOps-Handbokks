# Metadata

**Metadata** is the information that describes and identifies a Kubernetes object. It is defined under the `metadata` field of an object and is used by Kubernetes and other components to identify, organize, and associate additional information with that object.

A typical `metadata` section may contain fields such as:

```yaml
metadata:
  name: nginx
  namespace: production
  labels:
    app: nginx
    environment: production
  annotations:
    description: "Production web server"
```

Important metadata fields include:

* **`name`** → identifies the object within its namespace and resource type.
* **`namespace`** → specifies the namespace to which a namespaced object belongs.
* **`labels`** → key-value pairs used to identify, group, and select objects.
* **`annotations`** → key-value pairs used to attach additional information or configuration to an object. They are not intended for selecting objects.

Kubernetes also automatically maintains metadata fields such as **`uid`**, **`resourceVersion`**, **`creationTimestamp`**, and **`generation`**. These fields are used internally to track the object's identity, version, and lifecycle.

In simple terms:

```text
metadata
├── Identity
│   ├── name
│   ├── namespace
│   └── uid
│
├── Organization / Selection
│   └── labels
│
└── Additional Information
    └── annotations
```

Therefore, metadata describes **what an object is, where it belongs, how it can be identified or selected, and additional information associated with it**.

# Labels

**Labels** are key-value pairs attached to Kubernetes objects through the `metadata.labels` field. They are used to **identify, organize, group, and select** objects.

For example:

```yaml
metadata:
  labels:
    app: nginx
    env: production
```

Labels do not directly change the behavior of the object they are attached to. Instead, they provide metadata that Kubernetes components, controllers, and users can use to identify and operate on groups of objects.

### Purpose of Labels

Labels are fundamental to Kubernetes because they provide a flexible way to associate related resources.

They can be used to:

* Organize resources by application, environment, team, or version.
* Select groups of objects dynamically.
* Connect related Kubernetes resources, such as Services and Pods.
* Identify different application versions or releases.
* Filter resources with `kubectl`.
* Support controllers and other automation mechanisms.

For example:

```yaml
labels:
  app: payment
  environment: production
  version: v2
```

These labels allow different components and tools to identify that the object belongs to the `payment` application, is part of the production environment, and represents version `v2`.

## Label Selectors

A **label selector** is a query used to select objects based on their labels.

Selectors are extensively used by Kubernetes resources and tools. For example:

* **Services** use selectors to identify the Pods that should receive traffic.
* **Deployments and ReplicaSets** use selectors to identify the Pods they manage.
* **kubectl** can use selectors to filter resources.
* Other controllers and Kubernetes components can also use selectors to find related objects.

For example:

```bash
kubectl get pods -l app=nginx
```

This returns Pods whose `app` label is exactly `nginx`.

### Equality-Based Selectors

Equality-based selectors match a label against a specific value.

Examples:

```bash
kubectl get pods -l app=nginx
```

Selects Pods where:

```text
app = nginx
```

An inequality can also be used:

```bash
kubectl get pods -l env!=prod
```

Selects Pods where the `env` label is not equal to `prod`.

### Set-Based Selectors

Set-based selectors allow selection based on membership in a set of values.

For example:

```bash
kubectl get pods -l 'env in (dev,test)'
```

Selects Pods whose `env` label is either `dev` or `test`.

Another example:

```bash
kubectl get pods -l 'tier notin (frontend,backend)'
```

Selects Pods whose `tier` label is not `frontend` or `backend`.

Set-based selectors provide more expressive selection than simple equality matching.

## Labels and Services

A common example of labels and selectors is the relationship between a **Service** and Pods.

For example, a Pod might have:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    env: production
spec:
  containers:
    - name: nginx
      image: nginx
```

A Service can use a matching selector:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

The Service selector identifies Pods with:

```text
app = nginx
```

Kubernetes uses this selection to maintain the Service's backend endpoints. The Service networking mechanism can then direct traffic to the selected Pods.

The important relationship is:

```text
Pod
└── label: app=nginx
          ↑
          │
Service
└── selector: app=nginx
```

Therefore, labels and selectors provide a **dynamic relationship** between Kubernetes resources. The Service does not need to know the individual Pod names or IP addresses.

If a new Pod with `app: nginx` is created, it can automatically become a Service endpoint. If an existing matching Pod is removed or its labels change so that it no longer matches, Kubernetes updates the Service's endpoints accordingly.

## Label Syntax

Labels consist of a **key** and an optional **value**:

```yaml
labels:
  app: nginx
```

Some common conventions are:

```yaml
labels:
  app: payment
  environment: production
  version: v2
```

Label keys can optionally contain a DNS-style prefix followed by `/`:

```yaml
labels:
  example.com/team: platform
```

The prefix is useful when a label is defined by an organization or automation system and helps avoid collisions with labels created by other parties.

Label values are intended to be relatively small and follow Kubernetes label syntax rules. Labels should therefore contain concise identifying information rather than large descriptive data.

---

## Annotations

**Annotations** are key-value metadata attached to Kubernetes objects through the `metadata.annotations` field.

Like labels, annotations provide additional information about an object. However, their purpose is different:

> **Labels are intended for identification and selection; annotations are intended for additional metadata that is not used for selection.**

For example:

```yaml
metadata:
  annotations:
    description: "Frontend web server"
    owner: "platform-team"
```

Annotations can be used to store information such as:

* Documentation or descriptions
* Build and deployment information
* Tool-specific configuration
* References to external systems
* Operational or debugging information
* Configuration hints consumed by controllers or other tools

Annotations cannot be used by label selectors.

## How Annotations Affect Behavior

An annotation does not automatically change the behavior of a Kubernetes object.

Kubernetes stores the annotation as part of the object's metadata. A controller, operator, application, or other tool must specifically recognize and interpret that annotation for it to have an effect.

For example, an ingress controller may define specific annotations that modify how it handles an Ingress resource:

```yaml
metadata:
  annotations:
    <controller-specific-key>: <value>
```

If that controller recognizes the annotation, it can use the value to modify its behavior.

If no component recognizes the annotation, Kubernetes simply stores it as metadata and it has no operational effect.

Therefore:

```text
Annotation
     ↓
Stored as object metadata
     ↓
Recognized by a controller/tool?
     ├── Yes → Component may change its behavior
     └── No  → Remains metadata with no operational effect
```

This is why annotations are commonly used as an extension mechanism for Kubernetes controllers, operators, CI/CD systems, and other tools.

### Annotations vs. Application Configuration

Annotations should not be confused with application configuration.

For example, adding:

```yaml
annotations:
  timeout: "60"
```

to a Pod does **not** automatically configure the NGINX process running inside that Pod.

For an annotation to affect NGINX-related behavior, some Kubernetes component—such as an NGINX Ingress Controller—must specifically support and interpret that annotation.

If no component understands it, the NGINX container will not see or use the value.

---

## Example with Labels and Annotations

A Kubernetes object can contain both:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: production
    version: v2
  annotations:
    description: "Frontend web server"
    owner: "platform-team"
spec:
  containers:
    - name: nginx
      image: nginx
```

Here:

```text
Labels
├── app=nginx
├── environment=production
└── version=v2

        ↓
Used for identification, grouping, and selection


Annotations
├── description="Frontend web server"
└── owner="platform-team"

        ↓
Used for additional metadata or tool-specific information
```

## Labels vs. Annotations

|                                          | **Labels**                          | **Annotations**             |
| ---------------------------------------- | ----------------------------------- | --------------------------- |
| Primary purpose                          | Identification, grouping, selection | Additional metadata         |
| Used by selectors                        | **Yes**                             | **No**                      |
| Used by Kubernetes components            | Commonly                            | When specifically supported |
| Suitable for grouping resources          | **Yes**                             | No                          |
| Suitable for tool-specific configuration | Limited                             | **Yes**                     |
| Intended data                            | Small identifying values            | Additional metadata         |
| Example                                  | `app: payment`                      | `git-commit: a93f1c2`       |

The simplest way to remember the difference is:

```text
Labels
→ "What does this object belong to?"
→ Used to identify and select objects.

Annotations
→ "What additional information or instructions are associated with it?"
→ Used to store metadata for tools, controllers, or humans.
```

Both are stored under the object's `metadata`, but they serve fundamentally different purposes.
