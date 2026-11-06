# Gateway Exposure and Kubernetes Networking

### Gateway Address vs Gateway Exposure

A Gateway listener defines the protocol and port on which traffic should be accepted.

However, this does not automatically make the Gateway reachable from outside the cluster.

Several separate concerns must be addressed:

- Which IP address should clients connect to?
- Which Kubernetes node receives the traffic?
- How does traffic reach Envoy?
- How is traffic exposed outside the cluster?

The Gateway API `status.addresses` field reports the address or addresses assigned by the implementation. It does not allocate or advertise those addresses by itself.

An assigned address must also be reachable through the underlying network.

For example, a Gateway controller might report:

```yaml
status:
  addresses:
    - type: IPAddress
      value: 192.168.10.50
```

That address is useful only if the underlying network actually delivers traffic to it.

Now consider this Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gateway
spec:
  gatewayClassName: cilium
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: shop-tls
```

The listener says that the Gateway should accept HTTPS connections on port 443.

But this configuration alone does not answer:

- Which IP address should clients connect to?
- Which Kubernetes node receives the traffic?
- How does the traffic reach Envoy?
- How does traffic from outside the cluster enter the node?

These are exposure and networking concerns.

## Cilium Gateway Exposure Architecture

Cilium uses Envoy for Layer 7 proxying.

In its common exposure model, Cilium creates a Kubernetes Service for the Gateway's Envoy proxy. The Service provides the entry point through which traffic reaches Envoy.

Conceptually:

```
   External Client
          ↓
  Gateway IP / DNS
          ↓
  Kubernetes Service
          ↓
     Envoy Proxy
          ↓
      HTTPRoute
          ↓
   Backend Service
```

The Service type and Cilium configuration determine how external traffic reaches Envoy.

Cilium supports these exposure models:

- **LoadBalancer**
- **NodePort**
- **Host network mode**

These are not three different Gateway API routing mechanisms. They are different ways to expose the Gateway's data plane.

Cilium uses its networking stack and TPROXY mechanism to transparently forward traffic to Envoy.

- ## LoadBalancer Service

  By default, Cilium creates a Kubernetes Service of type `LoadBalancer` for Gateway exposure.

  Kubernetes does not automatically provide an external load balancer. A load-balancer implementation is required.

  Possible implementations include:

  - Cloud provider load-balancer integration.
  - MetalLB.
  - Cilium LoadBalancer IPAM combined with an appropriate external address announcement or routing mechanism.
  - An integrated external load-balancing system.

  **Important distinction:** IP allocation and IP advertisement are separate responsibilities. Allocating an IP does not automatically make that IP reachable from the external network.

  > **Cilium LB-IPAM** can allocate addresses, but the network must still know how to route traffic to them.

- ## NodePort Service

  A NodePort Service exposes a port on the Kubernetes nodes.
  
  Example:
  
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: gateway-service
  spec:
    type: NodePort
    ports:
      - name: https
        port: 443
        targetPort: 443
        nodePort: 30443
  ```
  
  The Gateway can be reached through:
  
  `https://<node-ip>:30443`

  The typical traffic flow is:

  ```
  External Client
        ↓
  Node IP : 30443
        ↓
  NodePort Service
        ↓
  Gateway Envoy
        ↓
  HTTPRoute
        ↓
  Backend Service
  ```
  
  The default Kubernetes `NodePort` range is `30000–32767`, although it can be changed through API server configuration.
  
  Advantages:
  
  - Does not require MetalLB.
  - Does not require a `LoadBalancer` implementation.
  - Uses existing node IP addresses.
  - Can work with an external load balancer or WAF.
  
  Disadvantages:
  
  - Requires managing node IPs and NodePort values.
  - External systems must know the nodes and ports.
  - Source IP visibility depends on Service configuration and networking behavior.
  - Exposes a node-level port rather than a dedicated load-balancer address.
  
  Cilium supports `NodePort` exposure for Gateway API.

- ## Host Network Mode

  ### General Kubernetes Concept

  Normally, a Pod has its own network namespace and receives a Pod IP.


  For example:
  
  ```
  Node IP: 192.168.10.11
              |
              ┌─────────────────────────────┐
              | Pod network namespace       |
              |                             |
              | Pod IP: 10.0.1.25           |
              | Application port: 8080      |
              └─────────────────────────────┘
  ```

  With Kubernetes `hostNetwork: true`, a Pod uses the host's network namespace instead of having an isolated network namespace.

  Conceptually:

  ```
  Node IP: 192.168.10.11
              |
              ┌─────────────────────────────┐
              | Host network namespace      |
              |                             |
              | Pod process                 |
              | Listening port: 8080        |
              └─────────────────────────────┘
  ```

  The process listens directly on the node's network interfaces. Therefore, a client can reach the process using `192.168.10.11:8080`

  There is no need to expose that process through a conventional `ClusterIP` or `NodePort` Service.

  ### Cilium Gateway API Host Network Mode

  Cilium supports Gateway API host network exposure starting with Cilium 1.16.

  This exposes the Cilium Gateway API Envoy listeners directly on the host network.

  Enable it using Helm values:

  ```yaml
  gatewayAPI:
    enabled: true
    hostNetwork:
      enabled: true
  ```

  **This is a Cilium configuration**, not a field added to the Gateway resource.

  It is useful when:

  - No LoadBalancer Service implementation is available.
  - MetalLB is not being used.
  - An external load balancer or WAF is available.
  - Traffic should reach Kubernetes nodes directly.

  ### Traffic Flow

  For example, with a node IP of `192.168.10.11` and a Gateway listener on port `8443`:

  ```text
  External Client / WAF
   ↓
  192.168.10.11:8443
   ↓
  Host network interface
   ↓
  Cilium networking / TPROXY
   ↓
  Envoy
   ↓
  HTTPRoute
   ↓
  Backend Service
  ```

  Cilium uses its networking stack and TPROXY mechanism to transparently forward traffic to Envoy. It is not simply a conventional standalone Envoy Pod exposed through a Service.

  ### Host Network Characteristics
    
  - **Listeners bind to all interfaces**
    
    Cilium exposes listeners on all interfaces:
    
    - `0.0.0.0` for IPv4.
    - `::` for IPv6.
    
    This means the listener is not automatically restricted to one particular node interface or IP address. Use firewall and network-level controls to restrict access where necessary.
    
  - **Gateway listener ports must be unique**

    The listener port configured in `Gateway.spec.listeners[].port` is used as the host network listening port.

    Avoid port conflicts. Configure unique ports for different Gateways and ensure that those ports are available on every selected node.

    For example, two Gateways cannot both independently bind to the same host port on the same node.

  - **Prefer ports above 1023**

    Linux normally restricts binding to ports below 1024 to processes with the appropriate capability.

    Cilium Envoy does not have `NET_BIND_SERVICE` by default. If ports such as 80 or 443 are required, configure the necessary capability through Cilium Helm values.

  - **Restrict exposure to selected nodes**

    Cilium supports a node label selector:

    ```yaml
    gatewayAPI:
      enabled: true
      hostNetwork:
        enabled: true
        nodes:
          matchLabels:
            role: gateway
    ```

    Only nodes matching the configured labels expose the Gateway API Envoy listeners.

    An empty selector exposes the functionality on all Cilium nodes.

  - **Mutual exclusion with LoadBalancer mode**

    Enabling Gateway API host network mode automatically disables the normal LoadBalancer Service exposure mode. These modes are mutually exclusive.

    Cilium also documents that host network mode is not compatible with TCPRoute and UDPRoute.

### Host Network vs NodePort vs LoadBalancer

| Characteristic | LoadBalancer | NodePort | Host network |
|---|---|---|---|
| External address | Usually dedicated LB IP | Node IP | Node IP |
| External port | Service/LB port | NodePort | Gateway listener port |
| Requires LB implementation | Yes | No | No |
| Exposure through Service | Yes | Yes | No conventional Service exposure for HTTP/HTTPS listeners |
| Port conflicts | Mainly Service/LB configuration | NodePort allocation | Host port conflicts |
| Cilium support | Yes | Yes | Yes, since 1.16 |

Host network mode concerns Gateway listener exposure. It does not mean that all application Pods must use host networking.

## Cilium's Internal Traffic Forwarding

Cilium integrates Gateway API with its networking stack and Envoy.

Host network mode and LoadBalancer mode are mutually exclusive

Incoming traffic is transparently forwarded to Envoy using TPROXY.

Conceptually:

```text
Incoming traffic
       ↓
Cilium networking / eBPF
       ↓
TPROXY redirection
       ↓
Envoy
       ↓
HTTP routing
       ↓
Backend
```

TPROXY is a Linux mechanism that allows a transparent proxy to receive traffic while preserving information about the original connection.

Cilium Gateway API prerequisites include:

- Kube-proxy replacement enabled.
- L7 proxy enabled.
- Appropriate TPROXY support and kernel/network dependencies.

With the default `bpf.tproxy=false`, Cilium uses iptables to implement TPROXY, requiring the relevant iptables and netfilter modules.

The eBPF-based `bpf.tproxy=true` option is documented as beta.

Do not confuse the eBPF interception with the actual HTTP routing. eBPF handles networking and traffic redirection; Envoy handles Layer 7 proxying and HTTP routing.

### Important Distinctions

- Gateway API defines routing intent; the controller implements it.
- `status.addresses` reports assigned addresses; it does not guarantee external reachability.
- LoadBalancer, NodePort, and host network are exposure mechanisms, not HTTP routing mechanisms.
- Host network mode exposes Cilium Gateway API listeners directly on node interfaces.
- Cilium uses TPROXY to forward incoming traffic to Envoy.

---