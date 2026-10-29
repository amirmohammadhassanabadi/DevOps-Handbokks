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

---

# Gateway API — Traffic Flow

## Overview

A typical request routed through Gateway API passes through the following logical stages:

1. DNS resolution and external network delivery.
2. Gateway listener acceptance.
3. TLS handling, when applicable.
4. HTTP request matching and routing.
5. Backend selection and forwarding.
6. Application processing and response delivery.

These stages describe logical behavior, not necessarily separate network hops. The exact packet path depends on the Gateway implementation and cluster networking configuration.

- ### DNS and external reachability

  The client resolves the requested hostname to an IP address that can reach the Gateway's exposed endpoint.

  DNS configuration and Gateway API configuration are separate concerns. Gateway API does not automatically create public DNS records.

  The endpoint may be exposed through a load balancer, NodePort-based arrangement, host networking, or another supported mechanism.

  A correctly configured Gateway can still be unreachable if DNS, routing, firewall rules, or external exposure is incorrect.

- ### Gateway listener

  The Gateway listener defines the protocol, port, optional hostname restrictions, TLS configuration, and Route-attachment policies.

  For HTTPS, TLS negotiation takes place before the Gateway's HTTP routing logic can inspect the encrypted HTTP request. With TLS termination, the data plane decrypts the request and can then evaluate HTTP routing rules.

  A listener does not independently define every HTTP path's backend destination.

- ### HTTPRoute matching and routing

  An `HTTPRoute` defines how HTTP requests are matched and forwarded.

  Matching conditions can include:

  - Hostnames.
  - URL paths.
  - HTTP methods.
  - HTTP headers.
  - Query parameters.

  Filters can modify a request or response, or perform supported actions such as redirects. Backend references identify the destinations for matching requests.

  The controller reconciles Gateway API resources and programs the data plane. The data plane uses the resulting configuration to process requests; it does not normally query Kubernetes API resources for each incoming request.

- ### Backend forwarding

  A matching Route commonly references a Kubernetes Service and a Service port.

  The Service provides a stable abstraction for the backend. The implementation resolves backend and endpoint information and forwards the request to a selected endpoint, commonly a Pod.

  The **HTTPRoute** determines the intended routing destination, while the controller, data plane, and cluster networking implement the forwarding behavior.

  A Service's virtual IP is not necessarily an additional proxy hop. Depending on the implementation, the proxy may forward directly to a selected Pod endpoint.

- ### Response path

  The backend application processes the request and generates a response.

  The response travels back through the networking path to the client. When TLS terminates at the Gateway, the proxy typically handles the client-side TLS connection and returns the response through that connection.

  Successful communication also depends on backend health, endpoint reachability, network policies, protocol configuration, and timeouts.

## Cilium implementation

Cilium integrates Gateway API with its networking stack and Envoy for supported Layer 7 proxying.

In Cilium's documented architecture, traffic arriving at an exposed Gateway Service can be transparently forwarded to Envoy using TPROXY. This differs from implementations that expose a separate proxy Deployment through a Service.

The precise behavior depends on the Cilium version and configuration.

### Key distinctions

- **DNS:** maps a hostname to an IP address.
- **Gateway listener:** defines how incoming traffic is accepted.
- **HTTPRoute:** defines request matching and routing behavior.
- **Service:** provides a stable backend abstraction.
- **Controller:** reconciles API resources and programs the implementation.
- **Data plane:** processes actual traffic.
- **Network infrastructure:** provides the reachability required for requests and responses to travel.

> ### Question: 
> Why would it be a mistake to immediately assume the HTTPRoute is broken? Name two other parts of the request path that could cause the failure.
> ### Answer:
> the Gateway might have no listener at all, or it might have a listener that doesn't accept the request because of its protocol, port, hostname, or Route attachment configuration. If the Gateway and Route truly report successful acceptance and programming, however, we should inspect the exact status conditions before assuming this is the cause.
>
> Also the hostname must resolve to an address that can reach the Gateway's exposed endpoint. Even if DNS resolves correctly, external routing, firewall rules, or other network configuration could still prevent the request from reaching the cluster.

---

## Multiple HTTPRoutes sharing a Gateway

This is an important part of Gateway API because it lets different teams or namespaces manage their own routes while sharing a common entry point.

### Can multiple HTTPRoutes use the same listener?

Yes. You do not need one listener per HTTPRoute.

**For example:**

- A Gateway has one HTTPS listener for shop.example.com.
- One HTTPRoute handles /api.
- Another HTTPRoute handles /store.
- Both Routes attach to the same listener, assuming they satisfy its attachment rules.

**Conceptually:**

```
            Gateway
               |
      HTTPS listener :443
      hostname: shop.example.com
               |
       +-------+-------+
       |               |
  HTTPRoute A      HTTPRoute B
     /api             /store
       |               |
   API Service     Store Service
```

The listener defines which traffic can enter and which Routes may attach. The Routes define how matching requests are handled.

### What controls whether a Route can attach?

Three important settings work together.

| Setting | Purpose |
| --- | --- |
| `HTTPRoute.spec.parentRefs` | Specifies which Gateway the Route wants to attach to. |
| `Gateway.spec.listeners[].allowedRoutes` | Controls which Route kinds and namespaces may attach to that listener. |
| Listener and Route hostnames | Determine whether their hostname constraints are compatible. |

For example, a Route in namespace store can reference a Gateway in namespace edge:

```yaml
spec:
  parentRefs:
    - name: public-gateway
      namespace: edge
      sectionName: https
```

Here:

- `name` identifies the Gateway.
- `namespace` identifies the Gateway's namespace.
- `sectionName` identifies a specific listener by its name.

The `sectionName` field is optional. If omitted, the Route can potentially attach to multiple compatible listeners on the referenced Gateway, subject to the other attachment rules.

> **Important:** Referencing a Gateway does not automatically grant permission to attach to it. The listener's allowedRoutes configuration must permit the attachment.

### How does hostname compatibility work?

Suppose the listener is configured with:

```yaml
hostname: shop.example.com
```

And a Route specifies:

```yaml
hostnames:
  - shop.example.com
```

These are compatible.

But if the Route specifies only:

```yaml
hostnames:
  - api.example.net
```

it is not compatible with that listener because the hostname constraints do not intersect.

Wildcard hostnames can also match, according to Gateway API's hostname matching rules.

> **One subtle point:** allowedRoutes and hostname compatibility solve different problems. A Route may be allowed by namespace and kind but still fail hostname compatibility. Conversely, compatible hostnames do not override a namespace restriction.

### What happens when multiple Routes have overlapping rules?

This is where things get more interesting.

Imagine two Routes attached to the same listener:

- Route A matches `/api`.
- Route B matches `/api/orders`.

A request for `/api/orders` matches both prefix rules. The Gateway API specification defines precedence rules so that implementations can resolve such overlaps consistently.

For path matching, the main rules are:

1. An exact path match takes precedence over a prefix match.
2. Among prefix matches, the longer matching prefix takes precedence.
3. When matches remain tied, additional precedence rules consider other match criteria, such as method and header matches, followed by specification-defined tie-breakers.

Do not assume that whichever Route was created last wins. Creation time is relevant only in particular tie-breaking situations defined by the specification.

Also, the precedence rules apply to matching requests; they do not mean every conflict between Route definitions is resolved simply by choosing one Route. Invalid references, incompatible attachments, and implementation-specific limitations can still affect whether a Route is accepted.

### Complete precedence order

Gateway API defines a more detailed order for resolving overlapping HTTPRoute matches. The following is the relevant precedence sequence for standard HTTP path, method, header, and query-parameter matching. 

1. Exact path match takes precedence over prefix matching.
2. Longest matching prefix takes precedence over shorter prefixes.
3. Method match takes precedence over a match that does not specify a method, when the earlier criteria tie.
4. More matching header conditions take precedence over fewer header conditions.
5. More matching query-parameter conditions take precedence over fewer query-parameter conditions.
6. If matches from different Routes are still tied, the oldest Route wins, based on its creation timestamp.
7. If the creation timestamps are also tied, the Route whose {namespace}/{name} sorts first alphabetically wins.

**useful distinction:**

- **`parentRefs`** — Where does the Route want to attach?
- **`allowedRoutes`** — Is this Route allowed to attach to the listener?
- **Hostname compatibility** — Do the listener and Route agree on which hostnames they can serve?
- **Route status conditions** — Did the controller accept and program the attachment successfully?

### What is a Route conflict?

A conflict occurs when multiple Route configurations compete for the same traffic in a way that requires conflict-resolution rules.

For example, two teams might independently create Routes for the same hostname and path:

- Team A creates a Route for `shop.example.com` with an exact match on`/checkout`.
- Team B creates another Route for the same hostname and exact path, but sends traffic to a different backend.

The controller must determine which matching rule takes precedence according to the Gateway API specification. In a tie across Routes, **creation timestamp** and then **namespace/name** provide deterministic tie-breakers.

### Multiple rules inside one HTTPRoute

A single `HTTPRoute` can contain multiple rules, and each rule can have multiple matches.

Conceptually:

```
HTTPRoute: shop
  |
  +-- Rule 1
  |     Match: /api/orders
  |     Backend: orders-service
  |
  +-- Rule 2
  |     Match: /api/products
  |     Backend: products-service
  |
  +-- Rule 3
        Match: /health
        Backend: health-service
```

You do not need to create a separate `HTTPRoute` for every path. Multiple Routes become especially useful when you want separate ownership, namespaces, permissions, or lifecycle management.

For example, one team can own the public storefront Route while another owns the API Route, with both attaching to the same Gateway listener.

### How this applies to Cilium

The routing model above comes from Gateway API, not from a Cilium-specific routing syntax. Cilium implements the supported Gateway API behavior using its Gateway API controller and Envoy data plane. 

Keep two things separate:

- Standard behavior: What Gateway API defines, including path-match precedence and Route attachment.
- Implementation support: Which features a particular Cilium release implements, especially features that are experimental or implementation-specific.

Cilium's supported features can change between releases. Its current documentation should be checked against the version you plan to run rather than assuming every available Gateway API feature works in every Cilium version.

## `HTTPRoute` Filters

## 1. Purpose

An HTTPRoute does more than match requests and select a backend. It can also define filters that modify how a matching request is handled.

For example, a client might request `/api/v1/orders`, while the backend application expects `/v1/orders`. A URL rewrite filter can adapt the request before forwarding it, without requiring the client to change its URL.

Gateway API defines standard filter types, but an implementation does not necessarily support every filter or every option. For Cilium, we need to distinguish standardized behavior from the features supported by the specific Cilium release.

Filters are configured under `HTTPRoute.spec.rules[].filters`.

### Where filters fit into the request flow

The logical processing order is:

1. The listener accepts the connection.
2. The `HTTPRoute` matches the request.
3. The selected rule's filters are applied, where applicable.
4. The request is forwarded to the selected backend, unless a filter terminates or redirects the request.

**For example:**

```
Client request: /api/v1/orders
          ↓
HTTPRoute matches /api
          ↓
Rewrite path: /v1/orders
          ↓
Forward to orders-service
```

The client still sees the original URL in its address bar. A rewrite changes the request sent upstream; a redirect instead instructs the client to make another request.

## 2. Standard Filter Types

| Filter | Purpose | Gateway API support level |
|---|---|---|
| `RequestHeaderModifier` | Adds, sets, or removes request headers | Core |
| `ResponseHeaderModifier` | Adds, sets, or removes response headers | Extended |
| `RequestRedirect` | 	Returns a redirect response, instructing the client to request another URL. | Core |
| `URLRewrite` | 	Rewrites the request hostname and/or path before forwarding. | Extended |
| `RequestMirror` | Sends a request copy to another backend and ignores the mirrored response | Extended |
| `CORS` | Sends a copy of the request to another destination while the primary request continues normally. | Extended |
| `ExtensionRef` | References an implementation-specific extension | Implementation-specific |

The first five are standardized filter types, although individual options and support can vary. ExtensionRef is explicitly for implementation-specific behavior.

Support levels are defined by Gateway API. Always check the feature support matrix for the particular controller and version being used.

- ### RequestHeaderModifier

  The filter supports three operations:

  - `add`: adds a header value.
  - `set`: replaces existing values for a header with the configured value.
  - `remove`: removes a header.

  Example:

  ```yaml
  filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
          - name: x-gateway
            value: cilium
        set:
          - name: x-environment
            value: production
        remove:
          - x-debug
  ```

  A common use case is adding metadata that helps an application distinguish requests or identify an environment. However, do not treat a client-supplied header as trusted identity information. If a header is used for security decisions, the system must ensure that untrusted clients cannot spoof it.

- ### ResponseHeaderModifier

  This filter modifies response headers before the response is sent to the client.

  Example:

  ```yaml
  filters:
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        add:
          - name: x-served-by
            value: gateway
        remove:
          - x-internal-debug
  ```

  `ResponseHeaderModifier` has Extended support status in Gateway API, meaning implementations are encouraged, but not universally required, to support it. Verify support in your target Cilium release.

- ### RequestRedirect

  A redirect tells the client to make a new request to a different URL. The Gateway returns a redirect response instead of forwarding that request to the selected backend.

  **Common status codes:**

  | Code | Meaning |
  |---|---|
  | `301` | Moved Permanently |
  | `302` | Found; default when no status is specified |
  | `303` | See Other |
  | `307` | Temporary Redirect, preserving the HTTP method |
  | `308` | Permanent Redirect, preserving the HTTP method |

  If you do not specify a status code, Gateway API uses `302` by default.

  The method-preservation distinction matters for requests such as `POST:` `307` and `308` preserve the method, while clients may change the method when following `301` or `302` redirects.

  Example of redirecting HTTP to HTTPS:

  ```yaml
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: redirect-to-https
  spec:
    parentRefs:
      - name: public-gateway
        sectionName: http
    hostnames:
      - shop.example.com
    rules:
      - filters:
          - type: RequestRedirect
            requestRedirect:
              scheme: https
              statusCode: 301
  ```

  This Route attaches to the Gateway's http listener. A request such as:

  `http://shop.example.com/products`

  receives a redirect to:

  `https://shop.example.com/products`

  The HTTPS listener and its application Route must also be configured. The redirect does not automatically create that listener or make the HTTPS endpoint reachable.

- ### URLRewrite

  Unlike a redirect, a rewrite changes the request that the Gateway sends to the backend. The client does not need to make another request, and its address bar does not change.

  Suppose the client requests:

  `https://shop.example.com/api/v1/orders`

  But the backend expects:

  `/v1/orders`

  A rewrite can remove the `/api` prefix.

  ```yaml
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: api-route
  spec:
    parentRefs:
      - name: public-gateway
    hostnames:
      - shop.example.com
    rules:
      - matches:
          - path:
              type: PathPrefix
              value: /api
        filters:
          - type: URLRewrite
            urlRewrite:
              path:
                type: ReplacePrefixMatch
                replacePrefixMatch: /
        backendRefs:
          - name: api-service
            port: 8080
  ```

  **Conceptually:**

  ```
  Client requests:        /api/v1/orders
                              ↓
  Gateway rewrites path:  /v1/orders
                              ↓
  Backend receives:       /v1/orders
  ```

  `ReplacePrefixMatch` replaces the matching prefix with the configured value. The resulting path depends on the matched path and replacement value; it is not a general-purpose string substitution.

  > Gateway API also defines `ReplaceFullPath`, which replaces the entire path.

  `URLRewrite` and `RequestRedirect` cannot be combined in the same Route rule.

- ### RequestMirror

  Request mirroring sends a copy of a request to an additional backend while the normal request continues to its primary backend. The mirrored backend's response is ignored by the Gateway.

  This can be useful for testing a new application version with production-like traffic without using that version's response for the live user request.

  ```
                           +--> Primary backend
  Client --> Gateway ------|
                           +--> Mirror backend
                                      |
                                      +--> Response ignored
  ```

  Considerations:

  - Mirrored requests consume resources.
  - Requests may contain sensitive data.
  - Mirrored requests can cause side effects.
  - The feature is Extended support and must be verified for the target implementation.

### CORS and ExtensionRef

Gateway API also defines a `CORS` filter for Cross-Origin Resource Sharing. `CORS` controls which browser origins may access responses; it is not an authentication or authorization mechanism.

`ExtensionRef` allows a Route to refer to an implementation-specific extension. That can provide functionality beyond the standard API, but it reduces portability between Gateway API implementations.

Gateway API groups features into support levels:

- **Core**: implementations conforming to the relevant API must support the feature.
- **Extended**: implementations are encouraged to support it, but support is not universally required.
- **Implementation-specific**: behavior depends on the implementation.

These levels describe the API's conformance expectations, not a guarantee that every filter works identically in every controller version.
