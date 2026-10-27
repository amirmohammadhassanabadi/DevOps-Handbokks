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

---

# Gateway API Architecture and Core Resources

## Overview

Kubernetes Gateway API separates the configuration of traffic entry points from the rules that determine how application traffic is routed.

Its core resource model includes:

- `GatewayClass`: identifies the controller responsible for managing a class of Gateways.
- `Gateway`: defines a traffic entry point and its listeners.
- `HTTPRoute`: defines HTTP request-matching and routing rules.
- `Service`: commonly represents a backend application destination.

These resources describe desired networking behavior. A compatible Gateway API controller reconciles the resources and implements that behavior using its networking infrastructure.

## GatewayClass

`GatewayClass` is a cluster-scoped resource that represents a class of Gateways with a common implementation and potentially common configuration.

Its main fields include:

- `metadata.name`: the class name referenced by Gateways.
- `spec.controllerName`: identifies the controller responsible for managing the class.
- `spec.parametersRef`: optionally references implementation-specific configuration.

Example:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
```

The controller identifier shown is Cilium's documented identifier. Creating this resource does not install or start Cilium; the corresponding controller must already be running and configured.

`GatewayClass` serves a similar high-level purpose to `IngressClass`.

## Gateway

A `Gateway` represents a request for a particular traffic-handling capability and references a `GatewayClass`.

Its `spec` defines the desired Gateway configuration, including:

- `gatewayClassName`: the referenced GatewayClass.
- `listeners`: the protocols, ports, optional hostnames, TLS configuration, and Route-attachment policies.
- `addresses`: optional requested addresses, subject to controller support and infrastructure capabilities.
- Other supported fields that influence Gateway configuration.

A `listener` defines the traffic that the Gateway is intended to accept. Its configuration also determines which Route kinds and namespaces may attach.

Defining a `listener` does not guarantee external reachability. The controller and underlying infrastructure must implement the configuration and expose the required networking resources.

## HTTPRoute

`HTTPRoute` defines how HTTP requests are matched and forwarded to backend destinations.

Important fields include:

- `spec.parentRefs`: identifies the parent Gateway or other supported parent resource.
- `spec.hostnames`: optionally restricts routing to matching hostnames.
- `spec.rules`: defines request-matching and forwarding behavior.
- `matches`: specifies conditions such as path, headers, method, and query parameters.
- `filters`: specifies supported actions or request/response modifications.
- `backendRefs`: identifies backend destinations, commonly Kubernetes Services.

An `HTTPRoute` may contain multiple `rules`, and multiple `HTTPRoutes` may attach to the same `Gateway`, subject to compatibility and attachment policies.

## Route attachment

A Route must successfully attach to an appropriate Gateway listener before the controller can implement the intended routing behavior.

Attachment involves:

1. The Route referencing its intended parent through `parentRefs`.
2. The Gateway listener allowing the Route `kind` and `namespace`.
3. The Route and listener having compatible configuration, including hostname constraints where applicable.
4. The controller accepting and programming the resulting configuration.
`
The Gateway listener's `allowedRoutes` field controls which Route `kinds` and `namespaces` can attach. Namespace restrictions allow infrastructure operators to share a Gateway while limiting which application teams can configure its routing.

Cross-namespace backend references are a separate permission concern. A `ReferenceGrant` may be required to authorize a Route to reference an object in another namespace.

## Controller and data plane

A Gateway API controller watches and reconciles Gateway API resources, validates configuration, programs the implementation, and reports status.

The controller is software, not a standard Gateway API resource `kind`.

The data plane processes actual network traffic. Depending on the implementation, it may use a proxy such as Envoy together with other networking components.

For Cilium, Gateway API support integrates Cilium's networking platform with Envoy for supported Layer 7 proxying.

## 7. Status and troubleshooting

Resource existence alone does not prove successful implementation.

Important status conditions include:

- `Accepted`: indicates whether the controller accepts the relevant configuration.
- `Programmed`: indicates whether the Gateway configuration has been programmed.
- `ResolvedRefs`: indicates whether relevant references have been resolved.

Condition availability and meaning must be interpreted in the context of the resource and specification. Route status can also report acceptance by particular parent Gateways and listeners.

When troubleshooting, inspect resource status, conditions, events, controller logs, and the underlying networking infrastructure.

## 8. Resource relationships

- A Gateway references exactly one GatewayClass.
- A GatewayClass identifies the responsible controller.
- A Gateway defines one or more listeners.
- A Route references its intended parent Gateway and must satisfy the relevant attachment rules.
- A Route's backend references identify the destinations for matching traffic.
- A Service commonly provides a stable endpoint for the application backend.

These are API relationships, not a literal sequence of network hops. The controller translates the declared configuration into behavior implemented by the data plane.