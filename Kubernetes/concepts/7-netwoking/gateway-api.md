# Kubernetes Gateway API — Fundamentals

## Kubernetes Service

A Kubernetes `Service` provides a stable network endpoint for accessing a group of backend Pods. It abstracts away individual Pod IP addresses, which can change when Pods are recreated.

A `ClusterIP` Service is commonly used for internal cluster communication. Other Service types, such as `NodePort` and `LoadBalancer`, can provide external reachability.

A standard Service does not perform HTTP application routing based on hostnames, URL paths, or HTTP headers. These functions require a protocol-aware routing component.

## Kubernetes Ingress

`Ingress` is a Kubernetes API resource for describing external HTTP/HTTPS routing to backend Services.

An Ingress resource defines routing intent, while an **Ingress controller** implements that configuration using networking infrastructure such as a reverse proxy or load balancer.

Creating an Ingress resource alone does not make the application reachable; a compatible controller is required.

### Limitations of Ingress

- Its standard API focuses on basic HTTP/HTTPS routing.
- Advanced capabilities often depend on controller-specific annotations.
- Annotation behavior and support can differ between implementations.
- Infrastructure configuration and application routing are not separated as explicitly as in Gateway API.

The Ingress API remains supported, but its development is frozen. Kubernetes recommends Gateway API for new networking designs.

## Kubernetes Gateway API

Gateway API is a family of Kubernetes API resources for configuring service networking and Layer 4 and Layer 7 traffic routing.

It defines a standardized way to express networking intent. A compatible controller interprets that intent and implements it using its own networking infrastructure.

Gateway API is an API specification, not a proxy, load balancer, or controller itself.

### Foundational resources

- **`GatewayClass`:** Identifies the controller responsible for managing a class of Gateways.
- **`Gateway`:** Defines a Gateway instance, including listeners, protocols, ports, and route-attachment rules.
- **`HTTPRoute`:** Defines HTTP routing rules, including hostname and path matching, filters, and backend destinations.
- **`Service`:** Commonly represents the application backend receiving the routed traffic.

A Gateway references one GatewayClass. Routes reference their intended parent Gateway and must satisfy its route-attachment rules.

### Gateway API design principles

- **Role-oriented:** Separates infrastructure, cluster, and application responsibilities.
- **Expressive:** Provides standardized advanced routing capabilities, including header matching and weighted traffic splitting.
- **Portable:** Defines a common API across implementations, while allowing for differences in supported features.
- **Extensible:** Supports additional capabilities through policies and extension mechanisms.

Gateway API includes protocol-specific Route kinds beyond HTTP, subject to the applicable specification and controller support.

## Service, Ingress, and Gateway API

- **Service:** Provides a stable endpoint for accessing application backends.
- **Ingress:** Describes HTTP/HTTPS routing using the older Kubernetes Ingress API.
- **Gateway API:** Provides a more expressive, role-oriented, and extensible model for configuring networking infrastructure and traffic routing.

Gateway API is the recommended direction for new Kubernetes networking designs. Existing Ingress deployments can continue to operate, but migration requires configuring the corresponding Gateway API resources and a compatible controller.

### Here a question arises:

> Does Kubernetes has a separate resource kind named something like `GatewayController` or `GatewayAPIController`, analogous to how we talk about an Ingress controller.

**The answer is no**. Gateway API does not define a Kubernetes resource kind named `GatewayController` or `GatewayAPIController`.

Here's the distinction:

| Ingress | Gateway API |
| --- | --- |
| Ingress resource | Gateway resource |
| Ingress controller (software) | Gateway API controller (software) |
| No standard IngressController resource kind | No standard GatewayController resource kind |
| `IngressClass` | `GatewayClass` resource identifies the responsible controller |

The key detail is that both Ingress controller and Gateway API controller refer to software, not Kubernetes resource kinds.

Gateway API has a `GatewayClass` resource, which contains a `spec.controllerName` field identifying the controller responsible for managing that class of Gateways. The controller itself runs as software in the cluster; it is not created by defining a GatewayController YAML resource.

**GatewayClass Example:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
```

`IngressClass.spec.controller` and `GatewayClass.spec.controllerName` serve equivalent high-level purposes, but they are fields in different API specifications, not interchangeable fields.

## Gateway API versus its implementation

Gateway API defines the resource model and expected behavior. The controller implements it.

For example, Cilium provides a Gateway API implementation using its networking stack and Envoy integration. Its prerequisites, supported features, traffic exposure mechanisms, and implementation-specific behavior must be verified against the selected Cilium version.