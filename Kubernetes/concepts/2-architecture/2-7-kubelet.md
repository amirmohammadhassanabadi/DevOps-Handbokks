
# kubelet

The **kubelet** is the primary node agent that runs on **every Kubernetes node**. Its main responsibility is to ensure that the containers specified in **`PodSpecs`** are running correctly on that node.

**Responsibilities:**

* Communicates with the **kube-apiserver**
* Ensures assigned Pods and their containers are running
* Monitors container and Pod status
* Reports node and Pod status to the API server
* Works with the **container runtime** to create and manage containers
* Applies Pod configuration such as volumes, environment variables, and resource limits

The kubelet does **not** directly manage containers. It instructs the **container runtime**, such as **containerd** or **CRI-O**, to create, start, stop, and remove containers.

## Control Plane Nodes

Control plane nodes are also Kubernetes nodes and therefore run a **kubelet**.

In many Kubernetes installations, the kubelet is responsible for running **static Pods** containing control plane components such as:

* kube-apiserver
* kube-scheduler
* kube-controller-manager

On worker nodes, the kubelet manages application Pods assigned to that node.

**Important:**

> The **scheduler decides where a Pod should run; the kubelet on that node is responsible for making that Pod actually run.**

**Mental model:**

```text
kube-scheduler
      ↓
Assigns Pod to Node
      ↓
kubelet
      ↓
Container Runtime
      ↓
Containers
```
## kubelet Failure Behavior

If the kubelet fails on a node:

- The kubelet stops managing Pods on that node.
- The node eventually becomes NotReady as the control plane stops receiving kubelet heartbeats/status updates.
- Existing containers may continue running because they are managed by the container runtime, but Kubernetes can no longer reliably manage them through that kubelet.
- The control plane may eventually treat the node as failed and, depending on the workload and configuration, replace/reschedule Pods on another healthy node.
- Static Pods on that node are also no longer managed by the kubelet.


> kubelet failure does not necessarily mean containers immediately stop. It means Kubernetes loses its node agent and therefore loses reliable management and status reporting for that node.

## Example

We have:

```yaml
Deployment: nginx
Replicas: 1
Nodes:
   Node A: 10.10.10.1
   Node B: 10.10.10.2
```

Initially, nginx Pod A is running on Node A.

### 1. User creates the Deployment

You run:

```bash
kubectl apply -f nginx.yaml
```

The request goes:

```text
kubectl → kube-apiserver → etcd
```

The API server **authenticates/authorizes/admission-processes** the request and persists the Deployment object in etcd.

At this point, no container has been started yet.

### 2. Deployment Controller sees the Deployment

The Deployment controller watches the API server for Deployment changes.

It sees:

``` yaml
Deployment nginx
replicas: 1
```

It creates a ReplicaSet. The ReplicaSet object is stored through the API server.

### 3. ReplicaSet Controller sees the ReplicaSet

The ReplicaSet controller watches ReplicaSets.

It sees:

```yaml
Desired replicas: 1
Current Pods: 0
```

Therefore it needs to create a Pod. so it creates: `Pod A` through the API server.

**Important:**

The ReplicaSet controller does NOT choose the node. The Pod initially looks conceptually like:
```text
Pod A
nodeName = <empty>
```

### 4. Scheduler sees the unscheduled Pod

The kube-scheduler watches for Pods that don't have a node assigned.

It sees:

```text
Pod A
nodeName = empty
```

It evaluates the available nodes and chooses, for example:

```text
Node A = 10.10.10.1
```

Then the scheduler writes the assignment through the API server:

```text
Pod A
nodeName = 10.10.10.1
```

Now the API server contains the desired information:

```
Pod A should run on Node A
```

### 5. kubelet on Node A sees Pod A

The kubelet running on `10.10.10.1` observes Pods assigned to its node through the Kubernetes API. So Pod A becomes part of kubelet's desired state. The kubelet now needs to make reality match that desired state.

**Conceptually:**

```
API Server
    │
    │  "Pod A should exist on Node A"
    ↓
 kubelet
    │
    │ compare desired vs actual
    ↓
Container Runtime
```

### 6. kubelet checks the actual state

The kubelet communicates with the container runtime through the CRI.

**For example:**

```
kubelet → CRI → containerd
```

It asks the runtime about the local Pod/container state.

Suppose the runtime says:

```
Pod A: does not exist
```

So kubelet determines:

```
Desired:
    Pod A exists

Actual:
    Pod A does not exist
```

There is a difference. Therefore kubelet acts.

### 7. kubelet tells the runtime to create Pod A

**Conceptually:**

```
kubelet
   ↓
Create Pod sandbox
   ↓
Create containers
   ↓
Start containers
```

The runtime creates the Pod sandbox and containers.

**Eventually:**

```
Pod A
 └── nginx container
       ↓
    Running
```

### 8. kubelet continues reconciling

Kubelet doesn't say:

```
"I started Pod A, therefore my job is finished."
```

Instead, it continuously reconciles.

**For example:**

```
Desired:
Pod A → Running

Actual:
Pod A → Running

        ↓

Nothing needs to be done
```

### 9. Now a second Pod is assigned to Node A

Suppose another Pod, Pod B, gets scheduled to Node A.

The API server now says:

```
Pod A → Node A
Pod B → Node A
```

The kubelet on Node A observes the new Pod assignment.

Its desired state becomes:

```
Pod A should exist
Pod B should exist
```

Now it checks the local runtime. Runtime says:

```
Pod A → Running
Pod B → Doesn't exist
```

So kubelet compares:

| | Desired | Actual |
|---|---|---|
| Pod A | exists | exists |
| Pod B | exists | missing |

Therefore:

```
Pod A → do nothing
Pod B → create/start
```

> So Kubelet doesn't need to remember: **"Did I previously start Pod B?"**
>
> It determines the answer from the current desired state + current actual state.

### 10. What identifies the Pod?

Kubernetes doesn't simply rely on the Pod name. Pods have a unique **UID**.

**For example:**

```yaml
Pod A
name: nginx-abc123
UID: 7f8c...
```

The runtime/container metadata allows kubelet to associate local Pod sandboxes and containers with their Kubernetes Pod identity.

So kubelet can distinguish:

```yaml
Pod A UID: XXXXX
Pod B UID: YYYYY
```

This becomes particularly important after kubelet restarts.

### 11. Now suppose kubelet on Node A crashes

Before the failure:

```
Node A

kubelet
   │
   └── nginx Pod A
          │
          └── nginx container → Running
```

Then kubelet crashes. The kubelet stops communicating with the API server.

But **importantly:**

> The container runtime doesn't necessarily stop.

So you can temporarily have:

```
Node A

kubelet → DOWN

containerd → RUNNING

nginx container → STILL RUNNING
```

### 12. Control plane notices the node problem

The Node controller in kube-controller-manager monitors node health/status.

Because kubelet isn't reporting normally, Node A eventually becomes **NotReady** after the appropriate failure-detection/grace periods.

The Node controller handles the node-failure consequences.

Eventually, depending on the workload and configured tolerations/grace periods, the Pod running on the failed node is considered for eviction/deletion.

### 13. ReplicaSet notices the missing Pod

Suppose Pod A is eventually deleted from the API server.

Now the ReplicaSet controller sees:

```yaml
Desired: 1 Pod
Current: 0 Pods
```

ReplicaSet controller creates a replacement, **Pod B**.

### 14. Scheduler schedules Pod B

Pod B initially has:
```
nodeName = empty
```

Scheduler sees it and chooses `Node B`. So:

```
Pod B
nodeName = 10.10.10.2
```

### 15. kubelet on Node B starts Pod B

The kubelet on Node B sees:

```
Pod B → assigned to Node B
```

It checks its runtime:

```
Pod B → doesn't exist
```

Therefore:

```
kubelet Node B
      ↓
container runtime
      ↓
create sandbox
      ↓
create container
      ↓
start nginx
```

Now:

```
Node B
 └── Pod B
      └── nginx → Running
```

### 16. Then kubelet on Node A comes back

Suppose Node A's kubelet starts again. It does not simply assume:

```
"I was running nginx before, so I'll start managing that old nginx again."
```

It reconnects/synchronizes with the API server and reconstructs the desired Pod state for Node A.

Suppose the API server now says:

```
There is no longer a Pod assigned to Node A
```

Meanwhile, kubelet checks the local runtime and discovers:

```
Old nginx containers still exist
```

So kubelet sees:

```yaml
API / desired state:  Pod A → does NOT exist

Local actual state: Pod A containers → still exist
```

Therefore it cleans up the obsolete local Pod/container state.

**Conceptually:**

```
API Server                 Node A

Pod A doesn't exist   vs   Pod A containers exist
                              ↓
                           kubelet
                              ↓
                         cleanup old Pod
```

Meanwhile, Node B's Pod is completely independent.

**The complete picture**

Putting everything together:

```
                    ┌───────────────┐
                    │    kubectl    │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ kube-apiserver│
                    └───────┬───────┘
                            │
                            ▼
                         ┌─────┐
                         │etcd │
                         └─────┘

                            │
              ┌─────────────┼──────────────┐
              │             │              │
              ▼             ▼              ▼
       Deployment       ReplicaSet      Scheduler
       Controller       Controller
              │             │              │
              └─────────────┼──────────────┘
                            │
                            ▼
                    Pod assigned to Node
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
             Node A                 Node B
                 │                     │
              kubelet               kubelet
                 │                     │
                CRI                   CRI
                 │                     │
            containerd            containerd
                 │                     │
              Pod A                  Pod B
                 │                     │
            containers             containers
```

And the most important mental model is:

> **Controllers** → Determine / maintain cluster desired state
> 
> **Scheduler** → Chooses where an unscheduled Pod should run
> 
> **API Server** → Stores/exposes the cluster state
> 
> **kubelet** → Makes the desired Pods for THIS NODE match reality
>
> **Container Runtime** → Actually creates/runs/stops containers