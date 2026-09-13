# Pod

A **Pod** is the smallest deployable unit in Kubernetes. It represents a single instance of a workload and contains one or more containers that are deployed and managed together.

Containers within the same Pod are considered **tightly coupled**. They share the same network namespace and can share storage volumes, allowing them to communicate and coordinate closely.

A Pod is not itself a container. It is a Kubernetes abstraction that provides the execution context in which one or more containers run.

## Why Kubernetes Introduced Pods

Kubernetes does not schedule and manage containers as independent workload units. Instead, it schedules **Pods**.

Scheduling, networking, and much of the workload lifecycle are therefore designed around Pods. The Pod acts as a logical wrapper around its containers and provides a common execution environment.

Pods also allow multiple tightly coupled containers to run together when they need to share resources or cooperate closely.

For example:

```text
Pod
├── Application container
└── Sidecar container
```

The application and sidecar can share:

* The same network namespace
* The same Pod IP address
* The same port space
* Storage volumes
* The Pod lifecycle context

This makes a multi-container Pod useful when the containers form a single functional unit.

---

## Key Characteristics of a Pod

### 1. One or More Containers

A Pod can contain one or more containers.

Most Pods contain a single application container. Multiple containers are appropriate when the containers are tightly coupled and must run together.

A common example is a **sidecar container**:

```text
Pod
├── Application container
└── Logging sidecar
```

The sidecar may collect logs, provide a proxy, perform synchronization, or perform another supporting function for the main application.

Multiple containers in a Pod are **not automatically sidecars**. "Sidecar" describes a supporting container pattern; containers can also have other tightly coupled roles.

---

### 2. Shared Networking

All containers in a Pod share the same network namespace.

Therefore, containers in the same Pod:

* Share the same Pod IP address
* Share the same network interfaces
* Share the same port space
* Can communicate with each other through `localhost`

For example:

```text
Container A
    │
    │ http://localhost:5000
    ▼
Container B
```

If Container B listens on port `5000`, Container A can access it through:

```text
http://localhost:5000
```

Because the containers share the same network namespace, they cannot both bind the same port on the same IP address.

From outside the Pod, networking is normally addressed using the **Pod IP**, and clients generally access workloads through Kubernetes Services rather than relying directly on Pod IPs.

---

### 3. Shared Storage

Containers in a Pod can share storage through **Volumes**.

For example:

```text
Pod
├── Application container ──┐
│                           │
└── Logging sidecar ────────┤
                            ▼
                       Shared Volume
```

The application container can write files to the volume while another container reads those files.

A common example is:

```text
Application → writes logs → shared volume → Sidecar → sends logs
```

Containers do not automatically share the same filesystem. They share storage only when a volume is configured and mounted appropriately.

---

### 4. Pods Are Ephemeral

Pods are generally **ephemeral**. Their individual identity and IP address should not be treated as permanent.

If a Pod is deleted, fails, or its node becomes unavailable, Kubernetes may create a replacement Pod when a higher-level controller is managing it.

For example:

```text
Deployment
 ↓
ReplicaSet
 ↓
Pod
```

If the Pod disappears:

```text
Pod deleted
    ↓
ReplicaSet detects fewer replicas than desired
    ↓
New Pod created
    ↓
Scheduler assigns it to a node
    ↓
Kubelet starts the Pod
```

The replacement Pod is a **new Pod**, with a different identity and potentially a different IP address.

Because of this, production workloads are normally managed through higher-level controllers such as **Deployments, StatefulSets, or DaemonSets**, rather than by creating individual Pods directly.

---

## Pod Lifecycle and Container Lifecycle

A Pod and its containers have related but distinct lifecycles.

A container can terminate and be restarted without necessarily recreating the entire Pod. For example, with the appropriate restart policy, kubelet can restart a failed application container within the existing Pod.

Therefore, it is more accurate to distinguish:

```text
Pod lifecycle
    │
    ├── Pod is created
    ├── Containers are started
    ├── Containers may restart
    └── Pod is eventually deleted/replaced
```

A Pod replacement is different from simply restarting a container.

---

## Basic Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

### Explanation

* `apiVersion: v1` → API version used for the Pod object
* `kind: Pod` → object type
* `metadata` → identifies and organizes the Pod
* `spec` → defines the desired Pod configuration
* `containers` → list of application containers in the Pod
* `containerPort` → documents the port the container is intended to listen on; it does not itself publish the port externally

---

## When Multiple Containers Should Not Be Placed in the Same Pod

Multiple containers should generally be placed in the same Pod only when they are **tightly coupled and must share the same execution context**.

If containers represent independent application components, they should normally run in separate Pods.

For example, consider:

```text
NGINX
WordPress
Redis
MySQL
```

Putting all four into one Pod is generally a poor design.

### 1. Independent Scaling

Different components usually have different scaling requirements.

For example:

```text
NGINX       → may need multiple replicas
WordPress   → may need multiple replicas
Redis       → has its own scaling/availability strategy
MySQL       → has its own stateful/replication strategy
```

If all four containers are placed in one Pod and that Pod is managed with three replicas, Kubernetes creates:

```text
Pod 1 → NGINX + WordPress + Redis + MySQL
Pod 2 → NGINX + WordPress + Redis + MySQL
Pod 3 → NGINX + WordPress + Redis + MySQL
```

This forces all components to scale together, which is usually incorrect and inefficient.

---

### 2. Independent Lifecycle Management

Independent services often need to be updated, restarted, or replaced separately.

If NGINX, WordPress, Redis, and MySQL are separate Pods, each workload can be managed independently.

For example:

```text
Deployment A → NGINX Pods
Deployment B → WordPress Pods
Redis workload → Redis Pods
MySQL workload → MySQL Pods
```

Each workload can then have its own:

* Replica count
* Update strategy
* Resource requirements
* Health checks
* Storage configuration
* Lifecycle

This provides much greater operational flexibility.

---

### 3. Service-Based Communication

Independent application components can communicate through **Kubernetes Services**.

For example:

```text
              ┌── WordPress Pod
Service ──────┤
              └── WordPress Pod

Redis Service ──→ Redis Pod

MySQL Service ──→ MySQL Pod
```

This allows each component to be independently deployed and scaled while still providing stable network access through Services.

### General Rule

> **If containers must always run together and share the same network/storage context, consider placing them in the same Pod. Otherwise, separate them into different Pods.**

---

# Init Containers

**Init Containers** are specialized containers that run during Pod initialization, before the regular application containers start.

They are used to perform initialization or preparation tasks that must complete successfully before the application starts.

The execution flow is:

```text
Pod created
    ↓
Init Container 1
    ↓
Init Container 2
    ↓
All Init Containers complete successfully
    ↓
Application containers start
```

If an Init Container fails, kubelet retries it according to the Pod's restart policy. The application containers do not start until all Init Containers have completed successfully.

---

## Purpose of Init Containers

Init Containers are useful for preparation tasks such as:

* Waiting for a dependency or service to become available
* Generating configuration files
* Preparing shared volumes
* Downloading required data
* Performing initialization scripts
* Checking required conditions before startup

For example:

```text
Init Container
    │
    ├── Wait for dependency
    ├── Generate configuration
    └── Prepare shared volume
            ↓
Application Container
    └── Start application
```

An Init Container can use a different image and tools from the application container. This allows the application image to remain small and focused.

---

## Characteristics of Init Containers

### Sequential Execution

Init Containers run **sequentially**.

If a Pod contains three Init Containers:

```text
init-1 → init-2 → init-3 → application containers
```

`init-2` does not start until `init-1` has successfully completed, and `init-3` waits for `init-2`.

The application containers start only after all Init Containers complete successfully.

### Completion-Based Execution

Init Containers are intended to perform initialization work and then terminate successfully.

They are not long-running application containers.

After successful completion, they do not continue running alongside the application containers.

### Independent Image and Tools

Init Containers can use different images from the application containers.

For example:

```text
Init Container → busybox
Application    → nginx
```

This allows initialization tools and dependencies to remain outside the main application image.

### Shared Pod Resources

Init Containers run in the same Pod and can access configured Pod resources such as volumes and networking.

A common pattern is using a shared volume:

```text
Init Container
      │
      │ writes configuration
      ▼
Shared Volume
      │
      ▼
Application Container
      │
      └── reads configuration
```

---

## Init Containers vs Application Containers

| Aspect              | Init Container                                | Application Container                                      |
| ------------------- | --------------------------------------------- | ---------------------------------------------------------- |
| Purpose             | Initialization/preparation                    | Main workload                                              |
| Execution           | Before application containers                 | After all Init Containers succeed                          |
| Multiple containers | Run sequentially                              | Run concurrently                                           |
| Completion          | Normally exits successfully                   | Normally keeps running                                     |
| Failure             | Prevents application containers from starting | Container may be restarted according to Pod restart policy |
| Image               | Can use a different image                     | Uses application image                                     |
| Typical tasks       | Setup, checks, configuration, preparation     | Application workload                                       |

---

## Init Container Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
spec:
  initContainers:
    - name: init-script
      image: busybox
      command:
        - sh
        - -c
        - echo "Waiting for Redis..."; sleep 10

  containers:
    - name: webapp
      image: nginx
```

The execution order is:

```text
webapp-pod created
        ↓
init-script starts
        ↓
sleep 10 completes
        ↓
init-script exits successfully
        ↓
webapp container starts
```

### Important Note

An Init Container is different from a **sidecar container**.

* **Init Container** → runs during initialization and completes before application containers start.
* **Sidecar Container** → normally runs alongside the main application container and provides supporting functionality.

This distinction is important when designing multi-container Pods.

---

## Pod Lifecycle

The **Pod lifecycle** describes how a Pod progresses from creation through execution and eventually termination.

Several Kubernetes components participate in this lifecycle:

* **kube-apiserver** → accepts and exposes the Pod object
* **etcd** → stores the persistent API state
* **kube-scheduler** → assigns an unscheduled Pod to a node
* **kubelet** → manages the Pod and its containers on the assigned node
* **Controllers** → create, replace, or delete Pods when they are managed by higher-level workloads such as Deployments

A simplified lifecycle is:

```text
Pod manifest
    ↓
kube-apiserver
    ↓
Pod object stored in etcd
    ↓
Scheduler selects a node
    ↓
kubelet receives the Pod assignment
    ↓
Images pulled / containers created
    ↓
Containers started
    ↓
Pod runs
    ↓
Pod terminates
```

Once a Pod is scheduled, it is **bound to that node**. Kubernetes does not move the existing Pod to another node if the node fails. Instead, when the Pod is managed by a controller such as a Deployment, the controller can create a **new replacement Pod**, which the scheduler may assign to another node.

---

# Pod Phases

Kubernetes reports a Pod's high-level lifecycle state through `.status.phase`.

The possible Pod phases are:

* **`Pending`**
* **`Running`**
* **`Succeeded`**
* **`Failed`**
* **`Unknown`**

Pod phase is a **high-level summary**, not a detailed state machine. For detailed troubleshooting, inspect container states, container termination reasons, Pod conditions, and Events.

- ## Pending

    A Pod is in `Pending` when it has been accepted by Kubernetes but has not yet reached the `Running` or terminal phases.

    This can include:

    * Waiting for scheduling
    * Waiting for required resources
    * Waiting for volumes
    * Pulling container images
    * Preparing containers

    Therefore, a Pending Pod may already be scheduled to a node.

    ### Common causes

    * No node has sufficient CPU or memory
    * Node selectors or affinity rules prevent scheduling
    * Node taints are not tolerated
    * Required PersistentVolume/PVC is unavailable
    * Image is still being pulled
    * Other admission or initialization requirements prevent startup

    ### Key insight

    > `Pending` indicates that the Pod has not reached the Running phase. It does not necessarily mean that scheduling has failed.

    To determine the actual cause:

    ```bash
    kubectl describe pod <pod-name>
    ```

    The **Events** section is usually the first place to look.

    ---

- ## Running

    A Pod enters `Running` when:

    * It has been bound to a node, and
    * All containers have been created, and
    * At least one container is running or is in the process of starting/restarting

    `Running` does **not** necessarily mean that the application is healthy or ready to receive traffic.

    For example, a Pod can be:

    ```text
    Phase: Running
    Ready: False
    ```

    This can happen when a readiness probe is failing.

    ---

- ## Succeeded

    A Pod enters `Succeeded` when **all containers have terminated successfully** and Kubernetes will not restart them.

    This is common for short-lived workloads such as:

    * Batch processing
    * Data migration
    * One-time scripts
    * Kubernetes Jobs

    Example:

    ```text
    Job
     ↓
    Pod
     ↓
    Container completes successfully
     ↓
    Pod → Succeeded
    ```

    ---

- ## Failed

    A Pod enters `Failed` when **all containers have terminated**, and at least one container terminated unsuccessfully or was terminated by the system.

    For example:

    ```text
    Container → exit code 1
                     ↓
    Pod → Failed
    ```

    The exact reason should be determined from the container's termination state and Pod Events.

    ---

- ## Unknown

    `Unknown` means Kubernetes cannot reliably determine the current state of the Pod.

    This commonly occurs when communication with the node is unavailable.

    Possible causes include:

    * Node failure
    * Network partition
    * Kubelet failure
    * Node shutdown

    This is generally a **node/control-plane communication problem**, rather than an application-level failure.

---

# Pod Phase vs Container State vs Pod Conditions

These three concepts should not be confused.

```text
Pod
│
├── Phase
│     └── High-level Pod lifecycle
│
├── Conditions
│     └── More detailed Pod status
│
└── Containers
      └── Individual container states
```

- ### Pod Phase

    Answers:

    > **What is the overall high-level lifecycle state of the Pod?**

    Examples:

    ```text
    Pending
    Running
    Succeeded
    Failed
    Unknown
    ```

- ### Container State

    Answers:

    > **What is happening with an individual container?**

    The three container states are:

    * `Waiting`
    * `Running`
    * `Terminated`

- ### Pod Conditions

    Answers:

    > **Has a particular condition of the Pod been satisfied?**

    Pod Phase vs Pod Condition — Simple Difference

    The easiest way to understand it is:

    - **Phase** = What is the overall lifecycle state of the Pod?
    - **Condition** = Is a specific thing about the Pod currently true?
    
    For example, while a Pod is Running:

    ```
    PodScheduled    = True
    Initialized     = True
    ContainersReady = True
    Ready           = True
    ```

    But you could have:

    ```
    Phase = Running

    PodScheduled    = True
    Initialized     = True
    ContainersReady = False
    Ready           = False
    ```

    **Meaning:**

    The Pod is running, but its containers aren't ready, so the Pod isn't ready to receive traffic.

    So they answer different questions:

    | | Pod Phase | Pod Condition |
    | --- | --- | --- |
    | Question | What is the Pod's overall lifecycle state? | Is a specific condition currently satisfied? |
    | Example | Running | Ready=True |
    | Number | One phase | Multiple conditions
    | Detail | High-level | More specific |
    | Example | Succeeded | Ready=False |

    **The easiest mental model:**
    ```
    Pod
    │
    ├── Phase
    │     └── Running
    │
    └── Conditions
          ├── PodScheduled = True
          ├── Initialized = True
          ├── ContainersReady = False
          └── Ready = False
    ```

    Phase tells you the big picture. Conditions tell you the details and one particularly important point:

    > **Running does NOT mean Ready.**







---

# Container States

Each container inside a Pod has its own state.

## Waiting

The container has not yet started running.

The container may be waiting because:

* Its image is being pulled
* Container configuration is invalid
* A previous attempt failed
* Another initialization step has not completed

The state may include a `reason`, such as:

```text
ContainerCreating
ErrImagePull
ImagePullBackOff
CrashLoopBackOff
CreateContainerConfigError
```

These are **container state reasons**, not Pod phases.

---

## Running

The container has been started and its process is currently executing.

The state can include information such as:

* Start time
* Container ID

A Running container does not automatically mean the application is healthy or ready.

---

## Terminated

The container has finished execution.

Termination information can include:

* Exit code
* Reason
* Start time
* Finish time
* Termination message

A container can terminate:

```text
Successfully → exit code 0
Unsuccessfully → non-zero exit code
```

Depending on the Pod's restart policy and workload configuration, Kubernetes may restart the container.

---

# Pod Conditions

Pod conditions provide more detailed information about important lifecycle conditions.

Common conditions include:

| Condition         | Meaning                                                          |
| ----------------- | ---------------------------------------------------------------- |
| `PodScheduled`    | Pod has been successfully assigned to a node                     |
| `Initialized`     | All Init Containers have completed successfully                  |
| `ContainersReady` | All containers in the Pod are ready                              |
| `Ready`           | Pod is ready to receive traffic according to its readiness state |

Each condition can have one of three values:

```text
True
False
Unknown
```

For example:

```text
PodScheduled    → True
Initialized     → True
ContainersReady → False
Ready           → False
```

This could mean the Pod has been successfully scheduled and initialized, but one or more application containers are not currently ready.

### Important: Ready vs Running

A Pod can be:

```text
Phase: Running
Ready: False
```

This is normal when the application is running but its readiness probe is failing.

A **readiness probe does not restart the container**. It determines whether the Pod should be considered ready to receive traffic.

A **liveness probe**, when it fails according to its configuration, can cause the kubelet to restart the affected container.

---

## Common Pod Startup and Container Errors

The following statuses are frequently seen when troubleshooting Pods with:

```bash
kubectl get pods
```

They are **not Pod phases**. They generally represent container states or reasons associated with failed startup/restarts.

## ErrImagePull

`ErrImagePull` indicates that the kubelet attempted to pull the container image but the pull failed.

Common causes:

* Incorrect image name
* Incorrect image tag
* Image does not exist
* Authentication failure
* Registry unavailable
* DNS/network problems

Example:

```text
Pod
 ↓
kubelet
 ↓
Container runtime
 ↓
Image registry
       ✕
   image pull failed
```

---

## ImagePullBackOff

`ImagePullBackOff` occurs when image pulling continues to fail.

Kubernetes applies a **backoff delay** between retry attempts instead of continuously retrying immediately.

Typical sequence:

```text
Image pull
    ↓
Failure
    ↓
ErrImagePull
    ↓
Retry
    ↓
Failure
    ↓
ImagePullBackOff
    ↓
Retry after backoff
```

### Key distinction

- **`ErrImagePull`** = image pull attempt failed
- **`ImagePullBackOff`** = repeated image pull failures with increasing retry delay

---

## CreateContainerConfigError

This indicates that Kubernetes could not create the container because required configuration could not be resolved or validated.

Common causes include:

* Referencing a non-existent ConfigMap
* Referencing a non-existent Secret
* Invalid environment configuration
* Invalid volume configuration
* Other invalid container configuration

The first troubleshooting command should generally be:

```bash
kubectl describe pod <pod-name>
```

Then inspect the **Events** section.

---

## CreateContainerError

This indicates that Kubernetes attempted to create the container, but container creation failed.

Possible causes include:

* Invalid container configuration
* Volume-related problems
* Runtime errors
* Filesystem or permission problems
* Other container-creation failures

The exact cause should be determined from the Events and container status.

---

## CrashLoopBackOff

`CrashLoopBackOff` indicates that a container repeatedly starts and then terminates, causing Kubernetes to repeatedly restart it with an increasing delay between attempts.

Typical sequence:

```text
Container starts
      ↓
Application exits/crashes
      ↓
Kubelet restarts container
      ↓
Application crashes again
      ↓
Restart delay increases
      ↓
CrashLoopBackOff
```

Common causes:

* Application crashes
* Missing configuration
* Missing environment variables
* Incorrect command or entrypoint
* Dependency failures
* Application exits immediately
* Liveness/startup probe causing repeated restarts

### Important distinction

A **readiness probe failure alone does not cause `CrashLoopBackOff`**, because readiness probes do not restart containers.

---

## OOMKilled

`OOMKilled` indicates that a container was terminated after the Linux kernel's out-of-memory mechanism killed it.

A common cause is the container exceeding its configured memory limit.

For example:

```text
Container memory usage
        ↑
        │
Memory limit
────────┼────────
        │
        ✕
     OOMKilled
```

Common causes:

* Application consumes too much memory
* Memory limit is too low
* Memory leak
* Unexpected workload spike

The container's previous termination reason can usually be inspected with:

```bash
kubectl describe pod <pod-name>
```

or:

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```
> ### Memory Leak
> 
> A **memory leak** occurs when an application keeps memory that it no longer needs, causing memory usage to continuously increase over time.
> 
> Common causes include:
> 
> * Objects or data that remain referenced unnecessarily
> * Global variables or collections that grow indefinitely
> * Event listeners or callbacks that are never removed
> * Caches without size limits or expiration
> 
> Memory leaks can occur in both managed languages such as Java, Python, and JavaScript, and unmanaged languages such as C and C++.
> 
> In Kubernetes, a memory leak can cause a container to exceed its memory limit. The Linux kernel may then terminate the container, resulting in:
> 
> ```text
> Memory leak
>     ↓
> Memory usage increases
>     ↓
> Memory limit exceeded
>     ↓
> OOMKilled
>     ↓
> Container restarts
>     ↓
> Repeated failures
>     ↓
> CrashLoopBackOff
> ```
> 
> **Important:** Restarting the container only temporarily releases the accumulated memory. The actual solution is to identify and fix the cause of the memory leak.

---

## ContainerCannotRun

This indicates that the container runtime could not successfully start the container's process.

Possible causes include:

* Invalid executable or command
* Executable does not exist
* Permission problems
* Incompatible image/architecture
* Runtime or filesystem problems

The exact error returned by the runtime is important when troubleshooting this condition.

---

## RunContainerError

This indicates that an error occurred while the container runtime was attempting to start or initialize the container.

The exact meaning can vary depending on the underlying runtime and error.

Possible causes include:

* Startup command problems
* Runtime configuration problems
* Mount failures
* Permission problems
* Other runtime-level failures

The kubelet/container runtime Events provide the actual error and should be checked instead of relying only on the displayed reason.

---

## Troubleshooting Workflow

When a Pod is not working correctly, do not rely only on the value shown by:

```bash
kubectl get pods
```

Use a layered approach:

```text
Pod Phase
    ↓
Pod Conditions
    ↓
Container State
    ↓
Container termination reason
    ↓
Events
    ↓
Logs
    ↓
Node / kubelet / runtime
```

Useful commands:

```bash
kubectl get pod <pod-name>
kubectl describe pod <pod-name>
kubectl get pod <pod-name> -o yaml
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

For multi-container Pods:

```bash
kubectl logs <pod-name> -c <container-name>
```

`--previous` is particularly useful for containers that are repeatedly crashing because it retrieves logs from the previous container instance.

---

## Quick Mental Model

| Symptom                      | Usually investigate                                               |
| ---------------------------- | ----------------------------------------------------------------- |
| `Pending`                    | Scheduling, resources, volumes, initialization                    |
| `ErrImagePull`               | Image name, tag, registry, authentication, network                |
| `ImagePullBackOff`           | Repeated image-pull failure                                       |
| `CreateContainerConfigError` | ConfigMap, Secret, volume, container configuration                |
| `CreateContainerError`       | Container creation/runtime problem                                |
| `CrashLoopBackOff`           | Application crash, command, configuration, liveness/startup probe |
| `OOMKilled`                  | Memory usage and memory limits                                    |
| `ContainerCannotRun`         | Container process/runtime startup                                 |
| `RunContainerError`          | Container runtime startup/initialization                          |
| `Unknown`                    | Node/kubelet/control-plane communication                          |

### Final Mental Model

```text
Pod Phase
    │
    ├── Pending
    ├── Running
    ├── Succeeded
    ├── Failed
    └── Unknown
          │
          ▼
Pod Conditions
    │
    ├── PodScheduled
    ├── Initialized
    ├── ContainersReady
    └── Ready
          │
          ▼
Container State
    │
    ├── Waiting
    ├── Running
    └── Terminated
          │
          ▼
Container Reasons / Exit Codes
    │
    ├── ImagePullBackOff
    ├── CrashLoopBackOff
    ├── OOMKilled
    ├── CreateContainerError
    └── etc.
```

The key distinction to remember is:

> **Phase tells you the Pod's high-level lifecycle, Conditions tell you whether important Pod conditions are satisfied, and Container State/Reason tells you what is happening to individual containers.**

---

## Pod Restart Policy

The **restart policy** determines how kubelet handles terminated containers in a Pod. It is defined at the **Pod level** and applies to the Pod's containers.

The available policies are:

| Policy      | Behavior                                                   | Typical Use                 |
| ----------- | ---------------------------------------------------------- | --------------------------- |
| `Always`    | Restart containers whenever they terminate                 | Long-running workloads      |
| `OnFailure` | Restart containers only when they terminate unsuccessfully | Batch/short-lived workloads |
| `Never`     | Do not restart terminated containers                       | One-time tasks              |

`Always` is the default restart policy.

For example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  restartPolicy: Always
  containers:
    - name: my-container
      image: nginx
```

### Exit Codes

A container process normally returns an exit code when it terminates:

* `0` → successful termination
* Non-zero → unsuccessful termination

With `OnFailure`, a non-zero exit code causes the container to be restarted.

The restart policy controls **container restarts**, not Pod replacement. A higher-level controller such as a Deployment or Job is responsible for managing the desired number and lifecycle of Pods.

### Init Containers

Init Containers are also subject to the Pod's restart policy, but their behavior is different from application containers.

An Init Container must complete successfully before the next Init Container or the application containers can start. If it fails, kubelet retries it according to the Pod's restart policy.

---

## Pod Termination

When a Pod is deleted or terminated, Kubernetes normally attempts to shut it down gracefully.

A simplified process is:

```text
Pod deletion requested
        ↓
Pod marked for deletion
        ↓
Pod enters termination process
        ↓
Container termination requested
        ↓
SIGTERM
        ↓
Termination grace period
        ↓
SIGKILL if necessary
        ↓
Pod removed
```

The default `terminationGracePeriodSeconds` is **30 seconds**, unless configured otherwise.

For example:

```yaml
spec:
  terminationGracePeriodSeconds: 30
```

During termination, Kubernetes updates the Pod's endpoint/serving state so that terminating Pods can be removed from normal Service traffic.

Applications should therefore handle termination signals gracefully, for example by:

* Stopping acceptance of new work
* Finishing in-progress requests when possible
* Closing connections
* Flushing important data
* Exiting cleanly

If the container does not terminate within the grace period, it may be forcibly terminated with `SIGKILL`.

---

# Pod Lifecycle Hooks

Kubernetes provides container lifecycle hooks that allow applications to perform actions during important lifecycle events.

The two hooks are:

- ### PostStart

  `PostStart` is executed after the container is created.

  It can be used for initialization actions, but it should **not be relied upon to execute before the container's main process starts**, because Kubernetes does not guarantee the ordering between the `PostStart` hook and the container entrypoint.

- ### PreStop
  
  `PreStop` is executed when Kubernetes begins terminating the container.
  
  It can be used for graceful shutdown or cleanup tasks.

Example:

```yaml
lifecycle:
  postStart:
    exec:
      command:
        - sh
        - -c
        - echo "Container started"

  preStop:
    exec:
      command:
        - sh
        - -c
        - echo "Container stopping"
```

`PreStop` runs as part of the termination process and consumes time from the Pod's termination grace period.

---

# Pod Lifecycle Summary

A simplified lifecycle for a Pod is:

```text
Manifest submitted
        ↓
kube-apiserver
        ↓
Pod object stored
        ↓
Scheduler assigns node
        ↓
kubelet receives Pod assignment
        ↓
Init Containers run
        ↓
Application containers start
        ↓
Pod → Running
        ↓
Containers may restart
        ↓
Pod eventually deleted
        ↓
Succeeded / Failed
```

The final phase depends on how the Pod terminates.

For example:

```text
Job
 ↓
Pod
 ↓
Container completes successfully
 ↓
Pod → Succeeded
```

Whereas a long-running Deployment Pod normally remains:

```text
Pod → Running
```

until it is replaced or deleted.

---

# Comprehensive Pod Manifest Example

The following example demonstrates several important Pod features without attempting to include every available Pod field.

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: advanced-pod
  namespace: default
  labels:
    app: demo
    tier: backend
  annotations:
    description: "Example Pod demonstrating common Pod features"

spec:
  restartPolicy: Always

  nodeSelector:
    disktype: ssd

  volumes:
    - name: data-volume
      emptyDir: {}

  initContainers:
    - name: init-script
      image: busybox:1.36
      command:
        - sh
        - -c
        - echo "Initializing application"; sleep 5

      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "200m"
          memory: "128Mi"

      volumeMounts:
        - name: data-volume
          mountPath: /init-data

  containers:
    - name: app-container
      image: nginx:1.25
      imagePullPolicy: IfNotPresent

      ports:
        - name: http
          containerPort: 80

      env:
        - name: ENVIRONMENT
          value: production

      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"

      volumeMounts:
        - name: data-volume
          mountPath: /var/data

      lifecycle:
        postStart:
          exec:
            command:
              - sh
              - -c
              - echo "Container started"

        preStop:
          exec:
            command:
              - sh
              - -c
              - echo "Container stopping"

      securityContext:
        runAsUser: 1000
        allowPrivilegeEscalation: false

  terminationGracePeriodSeconds: 30
```

## Important Sections

### `apiVersion`

Pods belong to the core Kubernetes API group and use:

```yaml
apiVersion: v1
```

### `kind`

Defines the Kubernetes object type:

```yaml
kind: Pod
```

### `metadata`

Identifies and organizes the Pod.

Important fields include:

* `name`
* `namespace`
* `labels`
* `annotations`

### `spec`

Defines the desired configuration of the Pod.

Important fields demonstrated here include:

* `restartPolicy`
* `nodeSelector`
* `volumes`
* `initContainers`
* `containers`
* `terminationGracePeriodSeconds`

### `imagePullPolicy`

Controls when kubelet asks the container runtime to pull the image.

| Policy         | Behavior                                              |
| -------------- | ----------------------------------------------------- |
| `Always`       | Always attempt to pull the image                      |
| `IfNotPresent` | Pull only if the image is not already present locally |
| `Never`        | Never pull; image must already exist locally          |

### `ports`

Documents the ports that the container is intended to use. Defining `containerPort` **does not expose the Pod outside the cluster**. Services and other networking mechanisms are used for that.

### `env`

Defines environment variables available inside the container.

### `resources`

Defines CPU and memory requirements:

* `requests` → resources used for scheduling decisions
* `limits` → maximum resource usage enforced by Kubernetes/Linux mechanisms

### `volumeMounts`

Mounts a Pod volume into the container's filesystem.

---

# What Happens When We Create a Pod?

When we run:

```bash
kubectl apply -f pod.yaml
```

Kubernetes processes the request through several components.

## 1. kubectl

`kubectl`:

1. Reads the manifest.
2. Parses the YAML.
3. Uses the kubeconfig to determine the API server and authentication information.
4. Sends an HTTP/HTTPS request to the kube-apiserver.

Conceptually:

```text
kubectl
   │
   │ HTTP/HTTPS
   ▼
kube-apiserver
```

`kubectl` does **not** communicate directly with etcd.

---

## 2. kube-apiserver

The API server receives the request and performs processing such as:

### Authentication

Determines who is making the request.

Examples include:

* Client certificates
* Bearer tokens
* OpenID Connect
* Service account credentials

### Authorization

Determines whether the authenticated identity is allowed to perform the requested operation.

For example:

```text
Can Alice create Pods in namespace default?
```

RBAC is commonly used for authorization.

### Admission and Validation

The API server processes admission controls and validates the resource against the Kubernetes API schema.

If the request is accepted, the Pod object is persisted.

---

## 3. Pod Stored in etcd

The API server persists the Pod's API object in etcd.

At this point, the Pod has been accepted by the cluster, but it may not yet have a node assigned.

Conceptually:

```text
Pod
├── Desired configuration
├── Node: not assigned
└── Status: Pending
```

The API server is the component that communicates with etcd.

```text
kube-apiserver
       │
       ▼
     etcd
```

---

## 4. Scheduler Detects the Pod

The kube-scheduler observes Pods that do not yet have a node assignment.

It evaluates available nodes based on scheduling requirements.

Examples include:

* Resource requests
* `nodeSelector`
* Node affinity
* Taints and tolerations
* Pod affinity/anti-affinity
* Other scheduling constraints

---

## 5. Scheduler Selects a Node

The scheduler selects a suitable node.

For example:

```text
Pod nginx-pod
      ↓
Scheduler
      ↓
nodeA
```

The scheduler does **not** start the Pod. It records the scheduling decision through the API server.

The Pod now contains a node assignment such as:

```yaml
spec:
  nodeName: nodeA
```

---

## 6. kubelet Receives the Assignment

The kubelet on `nodeA` observes that a Pod has been assigned to its node.

It then begins reconciling the desired Pod state with the actual state on the node.

```text
API Server
     ↓
kubelet
     ↓
Pod preparation
```

---

## 7. Pod and Container Preparation

The kubelet coordinates with the container runtime through the **Container Runtime Interface (CRI)**.

The runtime is responsible for tasks such as:

* Pulling images
* Creating containers
* Starting containers
* Stopping containers
* Managing container processes

The kubelet also coordinates Pod networking and volume setup through the relevant Kubernetes components/plugins.

Simplified:

```text
kubelet
   ↓
CRI
   ↓
container runtime
   ↓
containers
```

---

## 8. Init Containers Run

If the Pod contains Init Containers, they run before the application containers.

```text
Init Container 1
       ↓
Init Container 2
       ↓
Application containers
```

Each Init Container must complete successfully before the next one starts.

---

## 9. Application Containers Start

After all Init Containers complete successfully, kubelet starts the regular application containers.

The Pod can then reach the `Running` phase when the conditions for that phase are satisfied.

---

## 10. Status Updates

The kubelet continuously observes the Pod and its containers and reports status information through the API server.

For example:

```text
kubelet
   ↓
kube-apiserver
   ↓
Pod status
```

The API server persists the API object's state in etcd.

The status may include:

* Pod phase
* Pod conditions
* Container states
* Container termination information
* Pod IP information

---

## 11. kubectl Gets the Status

When you run:

```bash
kubectl get pods
```

the flow is:

```text
kubectl
   ↓
kube-apiserver
   ↓
API object/status
```

`kubectl` does **not** directly query etcd.

The API server provides the Kubernetes API view of the object's current state.

---

# Complete Creation Flow

The entire process can be summarized as:

```text
kubectl apply -f pod.yaml
          ↓
    kube-apiserver
          ↓
 Authentication
 Authorization
 Admission / Validation
          ↓
       etcd
          ↓
   Pod exists in API
          ↓
    kube-scheduler
          ↓
   Node selected
          ↓
    kube-apiserver
          ↓
        kubelet
          ↓
   Pod preparation
          ↓
    Init Containers
          ↓
 Application Containers
          ↓
  Container Runtime
          ↓
      Pod Running
          ↓
    kubelet reports status
          ↓
    kube-apiserver
          ↓
        etcd
```

The key architectural idea is:

> **The API server is the central API gateway and source of cluster state for Kubernetes components. The scheduler decides where a Pod should run, while kubelet on the selected node makes the Pod actually run.**
