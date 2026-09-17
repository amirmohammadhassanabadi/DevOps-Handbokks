## Why Should We Not Use Pods Directly?

Although a Pod is the smallest deployable unit in Kubernetes, in real environments we generally do not create and manage application Pods directly. Instead, we use higher-level workload resources such as **Deployments, StatefulSets, DaemonSets, Jobs, and CronJobs**, which are managed by Kubernetes controllers.

A standalone Pod does not provide the higher-level lifecycle and availability management required by most applications.

- ## Pods Are Not Self-Healing at the Workload Level

    A Pod created directly by a user is not automatically replaced when the Pod itself is deleted or the node hosting it permanently fails.

    For example:

    ```bash
    kubectl apply -f pod.yaml
    ```

    If the Pod is deleted:

    ```text
    Pod exists
       ↓
    Pod deleted
       ↓
    No controller owns it
       ↓
    No replacement Pod is created
    ```

    Similarly, if the node permanently fails, there is no higher-level workload controller responsible for creating a replacement Pod on another node.

    A controller such as a **ReplicaSet** continuously compares the desired number of replicas with the current number of matching Pods:

    ```text
    Desired: 3 Pods
    Current: 3 Pods
            ↓
    One Pod disappears
            ↓
    Current: 2 Pods
            ↓
    ReplicaSet detects the difference
            ↓
    Creates a replacement Pod
            ↓
    Current: 3 Pods
    ```

    This reconciliation behavior provides **workload-level self-healing**.

    > **Note:** a Pod can restart its containers according to its **`restartPolicy`**. This is different from a controller creating a replacement Pod. Container restarts do not mean that the Pod itself is being recreated.

    ---

- ## Pods Do Not Provide Replica Management

    A Pod represents a single instance of a workload. If an application requires multiple instances, creating standalone Pods means managing each Pod individually.
    
    For example, creating three standalone Pods:
    
    ```text
    Pod 1
    Pod 2
    Pod 3
    ```
    
    does not provide a mechanism that continuously maintains three replicas.
    
    A ReplicaSet or Deployment instead declares the desired replica count:
    
    ```yaml
    spec:
      replicas: 3
    ```
    
    The controller continuously reconciles the workload so that the desired number of Pods exists.
    
    If one Pod disappears:
    
    ```text
    3 Pods
      ↓
    2 Pods
      ↓
    Controller creates replacement
      ↓
    3 Pods
    ```
    
    Scaling can also be performed by changing the desired replica count:
    
    ```bash
    kubectl scale deployment myapp --replicas=5
    ```
    
    For automatic scaling based on metrics, Kubernetes can use the **Horizontal Pod Autoscaler (HPA)**.

    ---

- ## 3. Pods Do Not Provide Rolling Updates

    Applications frequently need to be updated. With a standalone Pod, changing the container image generally requires replacing the Pod.

    For example:

    ```text
    nginx:1.24
        ↓
    nginx:1.25
    ```

    A standalone Pod does not provide a built-in rollout strategy for gradually replacing old instances with new ones.

    A **Deployment**, however, manages ReplicaSets and can perform a rolling update:

    ```text
    3 old Pods
        ↓
    2 old + 1 new
        ↓
    1 old + 2 new
        ↓
    3 new Pods
    ```

    The Deployment controller gradually replaces the old ReplicaSet with a new one according to the configured rollout strategy and availability constraints.

    ---

- ## Pods Do Not Provide Rollout History and Rollbacks

    A standalone Pod does not provide Deployment-style revision history or rollback management.

    Deployments maintain revisions of their ReplicaSets, allowing an application to return to a previous revision when an update causes problems.

    For example:

    ```bash
    kubectl rollout undo deployment/myapp
    ```

    This instructs the Deployment to roll back to a previous revision.

    ## Summary

    Standalone Pods are useful for testing, debugging, and simple one-off workloads, but they are generally not appropriate for managing production applications by themselves.

    The main limitation is that a standalone Pod represents **one workload instance**, while production applications usually require higher-level lifecycle management such as:

    * Maintaining the desired number of replicas
    * Replacing failed or deleted Pods
    * Scaling
    * Rolling updates
    * Rollbacks
    * Consistent Pod configuration through templates

    For this reason, production applications are typically deployed through higher-level workload resources rather than directly as standalone Pods.

    A useful mental model is:

    ```text
    Pod
    └── One workload instance

    Workload Resource
    └── Defines how the application should be managed
        ↓
    Controller
    └── Continuously reconciles desired and actual state
        ↓
    Pods
    └── Created, replaced, scaled, or updated as required
    ```