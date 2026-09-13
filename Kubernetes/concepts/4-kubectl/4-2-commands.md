# kubectl Commands

## Basic Command Structure

The general structure of a `kubectl` command is:

```bash
kubectl [COMMAND] [TYPE] [NAME] [FLAGS]
```

- ### COMMAND

    Specifies the operation to perform.

    Common commands include:

    ```text
    get
    describe
    create
    apply
    delete
    edit
    patch
    logs
    exec
    config
    ```

- ### TYPE

    Specifies the Kubernetes resource type.

    Examples:

    ```bash
    pod
    deployment
    service
    namespace
    node
    ```

    Resource types can also be specified using their short names:

    ```bash
    kubectl get pods
    kubectl get po

    kubectl get services
    kubectl get svc
    ```

    Common short names include:

    | Resource              | Short name |
    | --------------------- | ---------- |
    | Pod                   | `po`       |
    | Service               | `svc`      |
    | Deployment            | `deploy`   |
    | ReplicaSet            | `rs`       |
    | StatefulSet           | `sts`      |
    | DaemonSet             | `ds`       |
    | Namespace             | `ns`       |
    | ConfigMap             | `cm`       |
    | Secret                | `secret`   |
    | PersistentVolumeClaim | `pvc`      |
    | PersistentVolume      | `pv`       |
    | Node                  | `no`       |

- ### NAME

    Specifies a particular resource instance.

    It is optional for commands that operate on all resources of a type.

    ```bash
    kubectl get pods
    kubectl get pod nginx-pod
    ```

    The first lists Pods in the current namespace, while the second retrieves a specific Pod.

- ### FLAGS

    Flags modify the behavior of a command.

    Examples:

    ```bash
    kubectl get pods -n kube-system
    kubectl get pod nginx -o yaml
    kubectl get pods -l app=nginx
    ```

    Common flags include:

    ```text
    -n, --namespace
    -o, --output
    -l, --selector
    -A, --all-namespaces
    --kubeconfig
    ```

---

# kubeconfig Selection

The `--kubeconfig` flag allows `kubectl` to explicitly specify which kubeconfig file to use.

```bash
kubectl --kubeconfig=/home/user/dev-cluster-config get pods
```

This is useful when multiple kubeconfig files are maintained separately.

Alternatively, the `KUBECONFIG` environment variable can be used:

```bash
export KUBECONFIG=/path/to/config
```

After setting it, subsequent `kubectl` commands use that configuration.

If the kubeconfig contains multiple contexts, the active context determines the cluster and user configuration used by `kubectl`.

---

## kubectl config

The `kubectl config` command is used to inspect and manage kubeconfig settings.

- ### View Configuration

    ```bash
    kubectl config view
    ```

    Displays the kubeconfig configuration, including clusters, users, and contexts.

- ### View Current Context

    ```bash
    kubectl config current-context
    ```

    Shows the context currently being used.

- ### List Contexts

    ```bash
    kubectl config get-contexts
    ```

    Displays all available contexts and indicates the active one.

- ### Switch Context

    ```bash
    kubectl config use-context <context-name>
    ```

    Changes the active context.

- ### Set Default Namespace

    ```bash
    kubectl config set-context --current --namespace=production
    ```

    Sets `production` as the default namespace for the current context.

---

# Cluster Inspection

These commands provide basic information about the Kubernetes cluster and its nodes.

- ### kubectl cluster-info

    ```bash
    kubectl cluster-info
    ```

    Displays basic information about Kubernetes control-plane and cluster services. It is useful for verifying that `kubectl` can communicate with the cluster.

- ### kubectl version

    ```bash
    kubectl version
    ```

    Displays Kubernetes client and server version information when the server is reachable and the user has permission to retrieve it.

- ### kubectl get nodes

    ```bash
    kubectl get nodes
    ```

    Lists the nodes registered in the cluster.

    Typical output includes:

    ```text
    NAME       STATUS   ROLES           AGE   VERSION
    master     Ready    control-plane   ...   ...
    worker-1   Ready    <none>          ...   ...
    ```

    For additional information:

    ```bash
    kubectl get nodes -o wide
    ```

- ### kubectl api-resources

    ```bash
    kubectl api-resources
    ```

    Lists the resource types available through the Kubernetes API.

    Typical columns include:

    | Column       | Description                        |
    | ------------ | ---------------------------------- |
    | `NAME`       | Resource name used with `kubectl`  |
    | `SHORTNAMES` | Available short names              |
    | `APIVERSION` | API group and version              |
    | `NAMESPACED` | Whether the resource is namespaced |
    | `KIND`       | Kubernetes object kind             |

    For example:

    ```bash
    kubectl api-resources
    ```

    can be used to determine whether a resource is namespaced or cluster-scoped.

    Useful options:

    ```bash
    kubectl api-resources --namespaced=true
    kubectl api-resources --namespaced=false
    kubectl api-resources -o wide
    ```

    This command is particularly useful when working with **Custom Resource Definitions (CRDs)** because their resource types are also exposed through the Kubernetes API.

---

# Resource Inspection

- ## kubectl get

    The `kubectl get` command retrieves resources from the Kubernetes API and displays their current state.

    ### Syntax

    ```bash
    kubectl get [TYPE] [NAME]
    ```

    Examples:

    ```bash
    kubectl get pods
    kubectl get services
    kubectl get deployments
    kubectl get pod nginx-pod
    ```

    Without a specific name, `kubectl get` lists resources of the specified type.

    ### Namespace Selection

    By default, namespaced resources are queried in the namespace configured by the current context.

    ```bash
    kubectl get pods -n kube-system
    ```

    To query a resource across all namespaces:

    ```bash
    kubectl get pods -A
    ```

    ### Output Formats

    Default output provides a concise table.

    For Pods, common columns include:

    * `NAME` — Pod name
    * `READY` — Ready containers / total containers
    * `STATUS` — human-readable Pod status
    * `RESTARTS` — container restart count
    * `AGE` — time since creation

    Additional output formats are available:

    ```bash
    kubectl get pod nginx-pod -o yaml
    kubectl get pod nginx-pod -o json
    kubectl get pods -o wide
    ```

    `-o yaml` and `-o json` are especially useful when inspecting the complete Kubernetes API object.

    ---

- ## kubectl describe

    `kubectl describe` provides a detailed, human-readable view of a specific resource.

    ### Syntax

    ```bash
    kubectl describe TYPE NAME
    ```

    Examples:

    ```bash
    kubectl describe pod nginx-pod
    kubectl describe deployment nginx-deployment
    kubectl describe node worker-node-1
    ```

    Depending on the resource type, the output can include:

    * Metadata
    * Labels and annotations
    * Resource configuration
    * Node assignment
    * Container information
    * Volumes
    * Conditions
    * Events

    The **Events** section is particularly useful for troubleshooting.

    For example:

    ```bash
    kubectl describe pod nginx-pod
    ```

    can reveal problems such as:

    * Failed scheduling
    * Image-pull failures
    * Volume-mount errors
    * Container startup failures
    * Failed probes

### `get` vs `describe`

```text
kubectl get       → quick overview / current state
kubectl describe  → detailed inspection / troubleshooting
```

A common workflow is:

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

---

- ## kubectl explain

    `kubectl explain` provides documentation for Kubernetes API resources and their fields directly from the command line.

    ### Syntax

    ```bash
    kubectl explain RESOURCE
    kubectl explain RESOURCE.FIELD
    ```

    Examples:

    ```bash
    kubectl explain pod
    kubectl explain pod.spec
    kubectl explain pod.spec.containers
    ```

    Fields can be navigated hierarchically:

    ```text
    pod
    ├── metadata
    ├── spec
    │   ├── containers
    │   ├── volumes
    │   └── restartPolicy
    └── status
    ```

    To display the available fields recursively:

    ```bash
    kubectl explain pod --recursive
    ```

    `kubectl explain` is particularly useful when creating or inspecting manifests because it exposes the API schema and field descriptions available to the installed Kubernetes API resources.

---

# Creating Resources

- ## kubectl create

    `kubectl create` creates a new Kubernetes resource.

    ### From a Manifest

    ```bash
    kubectl create -f pod.yaml
    ```

    If the resource already exists, the command fails instead of updating it.

    ### Directly from the Command Line

    ```bash
    kubectl create deployment nginx-deployment --image=nginx
    kubectl create namespace dev
    kubectl create configmap app-config --from-file=config.properties
    ```

    ### Generate YAML Without Creating

    The `--dry-run=client` option can be combined with output formatting to generate a manifest without sending the resource to the cluster:

    ```bash
    kubectl create deployment nginx \
      --image=nginx \
      --dry-run=client \
      -o yaml
    ```

    This is useful for quickly generating a starting manifest.

---

# Applying Configuration

- ## kubectl apply

    `kubectl apply` creates or updates resources from declarative configuration.

    ### Syntax

    ```bash
    kubectl apply -f <file>
    ```

    Examples:

    ```bash
    kubectl apply -f pod.yaml
    kubectl apply -f deployment.yaml
    kubectl apply -f configs/
    ```

    If the resource does not exist, it is created. If it already exists, Kubernetes updates the resource according to the configuration being applied.

    A single YAML file can contain multiple resource definitions separated by `---`:

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    # ...

    ---
    apiVersion: apps/v1
    kind: Deployment
    # ...

    ---
    apiVersion: v1
    kind: Service
    # ...
    ```

    The configuration can be stored in version control and reused across environments, making `kubectl apply` a common tool for declarative Kubernetes management.

---

# Updating Resources

- ## kubectl edit

    `kubectl edit` retrieves an existing resource and opens its editable representation in the configured editor.

    ### Syntax

    ```bash
    kubectl edit TYPE NAME
    ```

    Example:

    ```bash
    kubectl edit deployment nginx-deployment
    ```

    For a namespaced resource:

    ```bash
    kubectl edit service my-service -n production
    ```

    The editor can be configured through `KUBE_EDITOR` or `EDITOR`:

    ```bash
    export KUBE_EDITOR=nano
    ```

    After the edited object is saved and accepted, `kubectl` sends the update to the API server.

    `kubectl edit` is convenient for quick operational changes, but for persistent production configuration, modifying the source manifest and applying it is generally preferable.

    ---

- ## kubectl patch

    `kubectl patch` modifies specific fields of an existing object without editing the entire resource.

    ### Syntax

    ```bash
    kubectl patch <resource> <name> --type=<patch-type> -p='<patch-data>'
    ```

    Common patch types are:

    * **Strategic Merge Patch** — Kubernetes-specific merge behavior for supported built-in resources
    * **JSON Merge Patch** — merges a JSON object into the existing resource
    * **JSON Patch** — performs explicit operations such as `add`, `remove`, and `replace`

    Example:

    ```bash
    kubectl patch deployment nginx \
      -p='{"spec":{"replicas":3}}'
    ```

    JSON Patch can perform precise modifications:

    ```bash
    kubectl patch deployment nginx \
      --type=json \
      -p='[{"op":"replace","path":"/spec/replicas","value":3}]'
    ```

    It can also be used for targeted troubleshooting operations such as removing a finalizer:

    ```bash
    kubectl patch <resource> <name> \
      --type=json \
      -p='[{"op":"remove","path":"/metadata/finalizers"}]'
    ```

    The distinction is:

    ```text
    kubectl apply  → declarative configuration
    kubectl edit   → interactive modification
    kubectl patch  → targeted field modification
    ```

---

# Debugging

- ## kubectl logs

    `kubectl logs` retrieves logs produced by containers in a Pod.

    ### Syntax

    ```bash
    kubectl logs POD_NAME
    ```

    Examples:

    ```bash
    kubectl logs nginx-pod
    kubectl logs nginx-pod -c nginx-container
    kubectl logs -f nginx-pod
    kubectl logs nginx-pod --previous
    kubectl logs nginx-pod -n production
    ```

    Important options:

    * `-c` / `--container` — select a specific container
    * `-f` / `--follow` — continuously stream logs
    * `--previous` — retrieve logs from the previous container instance after a restart
    * `-n` / `--namespace` — specify the namespace

    For a multi-container Pod:

    ```bash
    kubectl logs my-pod -c app-container
    ```

    The logs are retrieved through the Kubernetes node's kubelet, which obtains them from the container runtime's logging mechanism.

    ---

- ## kubectl exec

    `kubectl exec` executes a command inside a running container.

    ### Syntax

    ```bash
    kubectl exec POD_NAME -- COMMAND
    ```

    Examples:

    ```bash
    kubectl exec nginx-pod -- ls /
    kubectl exec -it nginx-pod -- /bin/sh
    kubectl exec -it my-pod -c app-container -- /bin/sh
    kubectl exec -it nginx-pod -n production -- env
    ```

    The `-i` option keeps standard input open, while `-t` allocates a terminal for interactive use.

    For Pods containing multiple containers, `-c` specifies the target container.

    `kubectl exec` is primarily an operational and troubleshooting tool. It does not modify the Pod's declared configuration.

---

---

## Remaining parts:

`Resource Inspection`

`kubectl get`

`kubectl describe`

`kubectl explain`

`Creating Resources
`
`kubectl create
`
`kubectl apply
`
4. Updating Resources
`kubectl edit`

`kubectl patch`

kubectl rollout

5. Deleting Resources
kubectl delete
6. Debugging

kubectl logs
kubectl exec
kubectl port-forward
7. Namespace Management

kubectl get ns
kubectl create namespace
