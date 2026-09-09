## kube-scheduler:

The **kube-scheduler** is a control plane component responsible for deciding which node should run an unscheduled Pod.

When a new Pod is created:

1. The Pod is created without a node assignment.
2. The scheduler detects the unscheduled Pod.
3. It evaluates the available nodes.
4. It selects the most suitable node.
5. It assigns the Pod to that node through the API server.

Scheduling decisions can consider:

- CPU and memory requests
- Node labels and selectors
- Taints and tolerations
- Affinity and anti-affinity rules
- Node conditions and resource availability
- Other scheduling constraints

> The **scheduler** does **not** create or run the Pod. It only decides where the Pod should run.

### Deep Dive in Kube-Scheduler:

The **kube-scheduler** is a control plane component responsible for deciding on which node a newly created Pod should run. Its responsibility is limited to scheduling decisions; it does not create Pods or manage their lifecycle. When a user requests a workload such as creating an Nginx Pod, the request is first processed through the kube-apiserver and stored in etcd. Controllers then ensure that the Pod object exists, but if the Pod does not yet have an assigned node, it is considered unscheduled. At this point the kube-scheduler becomes responsible for selecting a suitable node for that Pod.

To make this decision, the **scheduler** watches the API server for Pods that do not yet have a node assignment. When it detects such a Pod, it retrieves information about available nodes and their current resource usage through the API server. This information includes factors such as CPU capacity, memory availability, resource requests from other Pods, node conditions, and various scheduling constraints. Using this data, the scheduler runs a scheduling algorithm that evaluates possible nodes and selects the most appropriate one according to the configured policies and scoring rules. The default behavior typically prefers nodes with sufficient resources and balanced utilization, but administrators can modify the scheduling policies or extend them with custom plugins.

Once the scheduler selects a node, it writes that decision back to the API server by updating the Pod’s specification with the chosen node name. The kubelet running on that node then observes the assignment through the API server and proceeds to create and run the Pod’s containers using the container runtime.

Scheduling decisions are made each time a new Pod needs placement. If a Pod is deleted and recreated, such as during scaling operations, rolling updates, or controller-driven replacement, the new Pod is again unscheduled and must be evaluated by the scheduler. In this process the scheduler recalculates the best node based on the current cluster conditions, which may lead to a different node being selected than the one used previously. This dynamic scheduling process helps Kubernetes distribute workloads efficiently across the cluster and adapt to changing resource availability.

---