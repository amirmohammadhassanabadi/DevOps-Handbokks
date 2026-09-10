# kube-proxy

**kube-proxy** is a node-level Kubernetes component responsible for implementing the networking behavior of **Services**. It runs on each node and configures the node's networking rules so that traffic sent to a Service can be forwarded to the appropriate backend Pods.

A **Service** provides a **stable virtual IP** and **DNS name** for a group of Pods, while the actual Pods behind the Service can change over time. kube-proxy helps maintain the node-level packet-processing rules required to translate traffic destined for a Service into traffic destined for one of its backend Pods.

## Responsibilities

* watches Kubernetes API objects relevant to Service networking, primarily **Services** and **EndpointSlices**
* maintains the node's Service-related networking rules
* implements Service virtual IP (**ClusterIP**) forwarding
* implements **NodePort** forwarding
* performs network-level load distribution across Service endpoints
* handles connection forwarding/NAT between Service addresses and Pod addresses

## How kube-proxy works

kube-proxy communicates with the **kube-apiserver** and observes changes to Services and their backend endpoints.

Conceptually:

```text
Service
   │
   ├── ClusterIP
   │
   └── EndpointSlice
          │
          │ Service endpoints
          ▼
      kube-proxy
          │
          │ programs networking rules
          ▼
   Linux networking stack
          │
          ▼
       Pod IPs
```

For example, suppose we have:

```yaml
Service: nginx
ClusterIP: 10.96.10.20

Backend Pods:
Pod A → 10.244.1.10
Pod B → 10.244.2.15
Pod C → 10.244.3.8
```

When a client sends traffic to:

```text
10.96.10.20:80
```

the ClusterIP is not normally an actual network interface assigned to a Pod. kube-proxy has configured the node's networking rules so that traffic destined for the Service can be translated/forwarded toward one of the Service's endpoints:

```text
Client
   │
   │ 10.96.10.20:80
   ▼
Service ClusterIP
   │
   │ kube-proxy programmed rules
   ▼
┌────────────────┐
│ Load selection │
└────────┬───────┘
         │
    ┌────┼────┐
    ▼    ▼    ▼
  Pod A Pod B Pod C
```

## kube-proxy does not normally forward every packet itself

An important distinction is that kube-proxy is primarily a **rule/programming component**.

It observes the Kubernetes state and configures the node's packet-processing mechanisms. After those rules are installed, packet forwarding and NAT are handled by the node's networking stack.

Therefore, conceptually:

```text
kube-apiserver
      ↓
  kube-proxy
      ↓
programs networking rules
      ↓
Linux networking
      ↓
Pod
```

kube-proxy is therefore not equivalent to a traditional application-level reverse proxy such as Nginx.

### Networking implementations

Historically, kube-proxy has commonly used:

* **iptables**
* **IPVS**

Modern Kubernetes environments can also use an **nftables** mode.

The exact implementation depends on the Kubernetes version and configuration.

The important concept is not the specific mechanism but the role:

```text
Kubernetes Service state
        ↓
    kube-proxy
        ↓
Node networking rules
        ↓
Service traffic
        ↓
Backend Pod
```

### Why does kube-proxy run on every node?

A client may send Service traffic from any node.

For example:

```text
                    Kubernetes Cluster

       Node A             Node B             Node C
      kube-proxy         kube-proxy         kube-proxy
          │                  │                  │
          ▼                  ▼                  ▼
       networking        networking         networking
          │                  │                  │
          └────────────── Service ──────────────┘
                             │
                        Backend Pods
```

Each node therefore needs the appropriate Service networking rules for traffic originating from or arriving at that node.

## kube-proxy as a DaemonSet

In many standard Kubernetes installations, kube-proxy is deployed as a **DaemonSet**.

A DaemonSet ensures that a kube-proxy Pod runs on each eligible node:

```text
DaemonSet
    │
    ├── kube-proxy → Node A
    ├── kube-proxy → Node B
    └── kube-proxy → Node C
```

This normally includes both worker and control-plane nodes unless scheduling rules, taints/tolerations, or cluster configuration prevent it.

### Important distinction: Service vs kube-proxy

A **Service** is a Kubernetes API object that defines a stable network identity and selects backend Pods.

**kube-proxy** is a node component that implements the networking behavior required for that Service on the node.

So:

```text
Service = Kubernetes abstraction

kube-proxy = node-level implementation of Service networking
```

### Important modern caveat

kube-proxy is **not an absolute requirement for Kubernetes Service networking**.

Some Kubernetes networking implementations can replace or bypass kube-proxy by implementing Service load balancing themselves, often using technologies such as **eBPF**.

Therefore, the statement:

> "Without kube-proxy on each node, Service networking will not work"

is too absolute.

A more accurate statement is:

> **In a cluster using kube-proxy for Service networking, each eligible node normally needs kube-proxy or an equivalent implementation that provides the required Service networking behavior.**

### Complete mental model

```text
                    kube-apiserver
                          │
                          │
              Services / EndpointSlices
                          │
                          ▼
                     kube-proxy
                          │
                          │ programs
                          ▼
               Node networking rules
                          │
                          ▼
                    Service traffic
                          │
                    load selection
                          │
                          ▼
                    Backend Pod
```

The key idea is:

> **kube-proxy watches the Kubernetes Service and endpoint state and programs the node's networking mechanisms so that traffic destined for a Service can reach and be distributed among its backend Pods.**

---

## iptables, IPVS, and nftables
iptables, IPVS, and nftables are not Kubernetes components. They are Linux networking/packet-processing mechanisms that kube-proxy can use to implement Service traffic.

### The common problem

Suppose we have:

```
Service
ClusterIP: 10.96.0.10:80

Backend Pods:
Pod A → 10.244.1.10:8080
Pod B → 10.244.2.20:8080
Pod C → 10.244.3.30:8080
```

A client sends:

```
10.96.0.10:80
```

But there isn't necessarily a real process listening on `10.96.0.10:80`. Something on the node needs to implement:

```
10.96.0.10:80
       ↓
choose backend
       ↓
10.244.1.10:8080
```

Historically, kube-proxy has done this using **iptables** or **IPVS**, and modern Kubernetes also has an **nftables** mode.
       
## iptables

**iptables** is a Linux userspace tool used to configure the kernel's netfilter packet-processing framework.

The important distinction is:

- **iptables** = tool/interface used to configure rules
- **netfilter** = kernel packet-processing framework

The rules tell the Linux kernel what to do with packets.

For example, conceptually:

```
Packet arrives
 ↓
destination = 10.96.0.10:80
 ↓
iptables/netfilter rules
 ↓
select backend
 ↓
DNAT
 ↓
10.244.1.10:8080
```

**DNAT** means Destination Network Address Translation.

The destination address is changed from `10.96.0.10:80` to something like `10.244.1.10:8080`. The kernel then routes the packet toward that Pod.

### How kube-proxy uses iptables

kube-proxy watches Services and EndpointSlices.

Suppose the Service has three endpoints:

```
Pod A → 10.244.1.10
Pod B → 10.244.2.20
Pod C → 10.244.3.30
```

kube-proxy creates a set of iptables rules representing that Service and its endpoints.

Conceptually:

```
Service ClusterIP
10.96.0.10:80
       ↓
   kube-proxy
       ↓
iptables rules
       ↓
 ┌─────┼─────┐
 ↓     ↓     ↓
Pod A Pod B Pod C
```

The actual iptables configuration is considerably more complicated than this; kube-proxy uses chains and rules to implement Service and endpoint selection.

**Important point:**

```
kube-proxy isn't sitting in the packet path like Nginx:

❌ Packet → kube-proxy process → Pod
```

Instead:
```
✅ kube-proxy → programs kernel networking rules

Packet → Linux kernel/netfilter → Pod
```

## IPVS

**IPVS** stands for IP Virtual Server. It is a Linux kernel-based Layer 4 load-balancing mechanism.

Instead of representing Service load balancing primarily through large numbers of individual iptables rules, IPVS maintains virtual services and real servers.

**For example:**

```
Virtual Service
10.96.0.10:80
       │
       ├── 10.244.1.10:8080
       ├── 10.244.2.20:8080
       └── 10.244.3.30:8080
```

Here:

- **Virtual Service** = Kubernetes Service ClusterIP:port
- **Real Servers** = Service endpoints / Pod IPs

When a packet arrives for `10.96.0.10:80`, IPVS selects one of the backend destinations according to its configured scheduling algorithm.

For example:

```
Request 1 → Pod A
Request 2 → Pod B
Request 3 → Pod C
Request 4 → Pod A
...
```

IPVS supports different load-balancing algorithms, such as:

- round-robin
- least connections
- weighted variants
- others

### Why was IPVS attractive?

One of the historical motivations was **scaling**. With very large numbers of Services and endpoints, representing everything as huge numbers of iptables rules could become expensive to process and manage.

IPVS provides a more purpose-built kernel load-balancing mechanism.

However, there is an important modern point:

> IPVS is no longer automatically considered the "better/newer" choice. Kubernetes has introduced nftables mode, and current Kubernetes networking discussions generally shouldn't be reduced to "iptables is old, IPVS is better."

## nftables

**nftables** is the newer Linux packet-filtering framework intended to replace the traditional iptables framework.

Conceptually:

- **iptables** → older interface/ecosystem
- **nftables** → newer Linux packet-filtering framework

nftables also operates through the Linux kernel's packet-processing infrastructure.

Kubernetes has a nftables mode for kube-proxy, where kube-proxy programs nftables rules to implement Services.

The basic flow remains:

```
Service state
     ↓
kube-proxy
     ↓
nftables rules
     ↓
Linux kernel
     ↓
backend Pod
```

So nftables isn't fundamentally changing the Kubernetes Service abstraction. You still have:

```
Service
   ↓
ClusterIP
   ↓
backend endpoints
```

The difference is how the node implements that behavior.

The most important distinction is:

```
              Kubernetes
                  │
              kube-proxy
                  │
    ┌─────────────┼─────────────┐
    ↓             ↓             ↓
iptables         IPVS        nftables
    │             │             │
    └─────────────┼─────────────┘
                  ↓
             Linux kernel
                  ↓
              Pod traffic
```

## where does eBPF fit?

This is worth knowing because you'll encounter it later with **Cilium**.

Some Kubernetes networking implementations don't rely on kube-proxy at all.

**For example:**

**Traditional:**

```
Service
   ↓
kube-proxy
   ↓
iptables/IPVS/nftables
   ↓
Pod
```

**eBPF-based:**

```
Service
   ↓
eBPF-based networking
   ↓
Linux kernel
   ↓
Pod
```