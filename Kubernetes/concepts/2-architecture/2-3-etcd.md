## etcd

**etcd** is a distributed, strongly consistent key-value store that Kubernetes uses to persist the state of its API objects. It contains:

- all Kubernetes objects
- configuration
- secrets
- cluster metadata

**Characteristics:**

- distributed
- strongly consistent
- highly available

### Deep Dive in etcd:

**etcd** is a distributed NoSQL key‑value database that Kubernetes uses as its primary datastore. It stores the persistent state of Kubernetes API objects in a structured format that Kubernetes components understand. All cluster configuration and metadata are persisted in **etcd**, including information about Pods, Deployments, Services, ConfigMaps, Secrets, nodes, and many other Kubernetes resources. When a user defines a resource, such as requesting an Nginx Pod with specific `replicas`, container images, volumes, or other configuration, this desired state is ultimately stored in **etcd** through the Kubernetes API.

In the Kubernetes architecture, the **kube‑apiserver** is the only component that communicates directly with **etcd** (Other components normally go through the API server.). When a user or tool sends a request, such as querying the list of Pods, the request first reaches the API server. The API server then reads the required data from **etcd** and returns the response. Other control plane components, such as the **kube-scheduler**, **kube-controller** and **kubelets**, do not communicate with **etcd** directly. Instead, they interact with the cluster by watching and updating resources through the API server, which then persists any changes in **etcd**.

Because **etcd** contains the complete state of the Kubernetes control plane, it is considered authoritative persistent datastore for Kubernetes state. If the **etcd** data is lost or becomes empty, Kubernetes effectively loses its knowledge of the cluster’s resources and configuration. Even if containers are still running on nodes, Kubernetes may no longer recognize or manage them because the definitions and metadata that describe those resources no longer exist in **etcd**. For this reason, protecting **etcd** is critical for cluster reliability.

In production environments, **etcd** is typically configured in a high‑availability cluster. Instead of a single instance, multiple **etcd** members run on different control plane nodes and replicate data between them using a consensus algorithm. This distributed design ensures that if one **etcd** node fails, the remaining members can continue serving the cluster without losing data. As a result, clusters with multiple control plane nodes often run multiple **etcd** instances to maintain redundancy, consistency, and fault tolerance for the Kubernetes control plane state.

The name **etcd** comes from a naming convention within the Linux directory structure. In UNIX, all system configuration files for a single system are contained in a folder called `/etc` and **d** stands for **distributed**.

### etcd implementation:

There are 2 implementation for etcd:

- ### Stacked etcd Topology
   In a Kubernetes cluster using the stacked **etcd** topology, each control plane node runs its own **etcd** member in addition to the control plane components (kube-apiserver, kube-scheduler, and kube-controller-manager). These **etcd** members do not act as independent databases; instead, they form a single distributed **etcd** cluster that stores the Kubernetes cluster state.

   A common misconception is that as long as one **etcd** member remains running, the cluster can continue operating. This is not true. **etcd** uses the **Raft consensus algorithm**, which requires a majority (quorum) of voting members to agree before processing operations. Without a quorum, the cluster cannot safely read or write the cluster state, preventing split-brain scenarios and ensuring data consistency (A healthy etcd cluster can serve certain reads without the same consensus requirements as writes, but Kubernetes needs a healthy etcd quorum for reliable cluster-state operations).

   For a cluster with N etcd voting members, the required quorum is more than half of the members. Therefore, the maximum number of failures the cluster can tolerate is:

   Maximum failures tolerated = `floor((N−1)/2)`

   This results in the following fault tolerance:

   | Number of etcd Members | Quorum Required | Maximum Failures Tolerated |
   | --- | --- | ---|
   | 1 | 1 | 0
   | 2 | 2 | 0
   | 3 | 2 | 1
   | 4 | 3 | 1
   | 5 | 3 | 2
   | 6 | 4 | 2
   | 7 | 4 | 3

   Notice that increasing the number of **etcd** members from an odd number to the next even number does not improve fault tolerance. For example, a 4-member **etcd** cluster can still tolerate only one failure, just like a 3-member cluster. Fault tolerance increases only when another member is added to create the next odd-sized cluster (for example, from 3 to 5 members). For this reason, production Kubernetes clusters typically use 1, 3, 5, or 7 voting **etcd** members. A 2-member or 4-member **etcd** cluster is generally not recommended because it does not provide better fault tolerance than the previous odd-sized configuration while introducing additional complexity.

   It is also important to distinguish between the control plane and **etcd**. Losing quorum in **etcd** does not immediately stop running workloads. Existing Pods on worker nodes continue running as long as the nodes remain healthy. However, because the API server can no longer reliably access or update the cluster state, management operations such as creating, deleting, scheduling, or updating resources cannot proceed until quorum is restored.

- ### external etcd topology
   In the external etcd topology, the Kubernetes control plane and etcd are separated into different sets of machines. This is a more advanced architecture that is common in larger production environments. Instead of running etcd on each control plane node, you have a dedicated etcd cluster:
   
   ```
                            +---------------------------+
                            |      Load Balancer        |
                            | (API VIP / etcd VIP/DNS)  |
                            +-------------+-------------+
                                          |
             _____________________________|_____________________________
            |                             |                             |
   +--------v------------+      +---------v-----------+      +----------v----------+
   | Control Plane 1     |      | Control Plane 2     |      | Control Plane 3     |
   | +-----------------+ |      | +-----------------+ |      | +-----------------+ |
   | | kube-apiserver  | |      | | kube-apiserver  | |      | | kube-apiserver  | |
   | | kube-scheduler  | |      | | kube-scheduler  | |      | | kube-scheduler  | |
   | | control-manager | |      | | control-manager | |      | | control-manager | |
   | +-------|---------+ |      | +-------|---------+ |      | +--------|--------+ |
   +---------|-----------+      +---------|-----------+      +----------|----------+
             |                            |                             |
             |              (API Requests to etcd Cluster)              |
             +----------------------------+-----------------------------+
                                          |
                         +----------------+----------------+
                         |      External etcd Cluster      |
                         |  (Running on dedicated nodes)   |
                         +----------------+----------------+
                                          |
                         +----------------+----------------+
                         |                |                |
               +---------v-------+ +------v-------+ +------v---------+
               |    etcd Node 1  | |  etcd Node 2 | |   etcd Node 3  |
               |  (etcd member)  | | (etcd member)| |  (etcd member) |
               +-----------------+ +--------------+ +----------------+
   ```
   **Why use external etcd?**
   The main benefit is **isolation**. In a stacked topology, a control plane node failure also removes an etcd member. In an external topology, the control plane and the database layer fail independently.

   **Fault tolerance**
   The important thing to remember is:

   > High availability depends on the etcd cluster size, not just the number of control plane nodes.

   **For example:**

   | Control Plane Nodes | External etcd Members | etcd Failures Tolerated |
   |---|---|---|
   | 2 | 3 | 1 |
   | 3 | 3 | 1 |
   | 5 | 5 | 2 |

   **Trade-offs**
   - **Advantages**
      - Better separation of responsibilities.
      - Control plane failures do not automatically remove etcd members.
      - Easier to scale control plane and etcd independently.
      - Often preferred for large production clusters.
   - **Disadvantages**
      - More machines to manage.
      - More networking complexity.
      - More operational overhead.

### When should we use which one of them?

- **Stacked etcd:** Recommended for most clusters because it is simpler, requires fewer machines, and has lower operational complexity.
- **External etcd:** Useful when you need to isolate etcd from the control plane and manage them independently, but it requires additional infrastructure and operational overhead.

**Key point:** External etcd provides isolation and flexibility, not inherently better fault tolerance.

Many people say "External etcd is more highly available." → That's not necessarily true. 
**For example:**

- 3 control plane nodes with stacked etcd → tolerate 1 etcd member failure.
- 3 control plane nodes + 3 external etcd nodes → also tolerate 1 etcd member failure.

The etcd fault tolerance is the same because both have three voting etcd members.

The advantage of external etcd is isolation and operational flexibility, not inherently greater fault tolerance.