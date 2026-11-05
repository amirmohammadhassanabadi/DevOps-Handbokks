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

---

## BackendRefs, ReferenceGrant, and Traffic Splitting

An `HTTPRoute` determines how a request is matched and where it should be sent. The `backendRefs` field identifies the backend destinations that receive matching traffic.

Keep three separate concepts in mind:

- **Backend selection:** Which Service should receive the request?
- **Reference permissions:** Is the Route allowed to reference that backend?
- **Traffic distribution:** If there are multiple backends, how should requests be distributed between them?

### How `backendRefs` works

A backend reference usually points to a Kubernetes Service.

```yaml
backendRefs:
  - name: orders-service
    port: 8080
```

This tells the controller to send matching traffic to port 8080 of the Service named orders-service.

If namespace is omitted, the backend is assumed to be in the same namespace as the HTTPRoute. You can specify namespace to reference a Service in another namespace, but that introduces additional permission requirements.

A simplified request flow is:

```
Client
  |
  v
Gateway / Envoy
  |
  v
HTTPRoute
  |
  | backendRefs
  v
Kubernetes Service
  |
  v
Service endpoints (typically Pod IPs)
```

The Service provides a stable backend abstraction. The Gateway implementation resolves and configures the actual destinations; the request does not necessarily pass through the Service's ClusterIP as a separate network hop.

### Cross-namespace backends and ReferenceGrant

Kubernetes namespaces provide organizational and access-control boundaries. A Route in one namespace should not automatically be able to expose a Service owned by another namespace.

Suppose the resources are organized like this:

```
Namespace: edge
  └── HTTPRoute: public-api
                    |
                    | backendRef
                    v
Namespace: applications
  └── Service: orders-service
```

The Route can specify the backend namespace explicitly:

```yaml
backendRefs:
  - name: orders-service
    namespace: applications
    port: 8080
```


But this reference alone is insufficient. 

> **The owner of the target namespace must explicitly permit the cross-namespace reference using a ReferenceGrant resource in the target namespace**

### How `ReferenceGrant` works

A `ReferenceGrant` identifies:

- **`from`**: the kinds of resources and source namespace permitted to make references.
- **`to`**: the kinds of resources that can be referenced in the namespace containing the grant.

For our example, the grant belongs in `applications`, not `edge`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: ReferenceGrant
metadata:
  name: allow-edge-routes
  namespace: applications
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: edge
  to:
    - group: ""
      kind: Service
```

This permits `HTTPRoute` resources in `edge` to reference **Services** in `applications`. It does not grant access to every kind of resource, and it does not allow Routes from every namespace.

Notice that the grant does not specify a Service name. Its to section grants permission by resource kind, so the grant applies to Services in the target namespace, subject to the specified source restrictions.

**Security principle:** the owner of the referenced resource controls whether cross-namespace references to that resource type are allowed. A Route author cannot grant themselves this permission simply by editing their own Route.

**What happens if the ReferenceGrant is missing?**

The cross-namespace backend reference is invalid. The controller should report the unresolved reference in the Route's status, typically through the `ResolvedRefs` condition with reason `RefNotPermitted`. Traffic to that backend cannot be forwarded as intended. The precise observable failure depends on the implementation and configuration.

Do not confuse two different cross-namespace relationships:

| Relationship | Main permission mechanism |
| --- | --- |
| `HTTPRoute` attaching to a Gateway in another namespace | Gateway listener's `allowedRoutes` |
| `HTTPRoute` referencing a Service in another namespace | `ReferenceGrant` in the Service's namespace |
| Gateway referencing a TLS Secret in another namespace | `ReferenceGrant` in the Secret's namespace |

These mechanisms protect different boundaries.

### Traffic splitting with backend weights

One Route rule can reference multiple backends. The weight field defines the relative proportion of requests sent to each backend.

For example, suppose you have two versions of an application:

- `orders-v1`: existing stable version
- `orders-v2`: new version being introduced

You can configure the Route to send approximately 90% of traffic to `v1` and 10% to `v2`.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: orders-route
spec:
  parentRefs:
    - name: public-gateway
  hostnames:
    - shop.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /orders
      backendRefs:
        - name: orders-v1
          port: 8080
          weight: 90
        - name: orders-v2
          port: 8080
          weight: 10
```


The distribution is calculated as:

$$
\text{Backend share} = \frac{\text{backend weight}}{\text{sum of all backend weights}}
$$

Here, the total weight is `90+10=100`, so the configured proportions are 90% and 10%. The sum does not have to equal 100; **weights are relative values**, not inherently percentages. 

For example, weights of `9` and `1` express the same ratio as `90` and `10`.

**Important details:**

- If a backend's weight is omitted, it defaults to 1.
- A weight of 0 means that backend should receive no traffic.
- Traffic distribution is approximate over finite numbers of requests; you should not expect exactly 10 out of every 100 requests to reach v2.
- Traffic splitting does not itself perform health analysis, automatically roll back a deployment, or guarantee application-level availability.
- Both backend Services must be configured correctly, and the controller must support the relevant behavior.

This is a core Gateway API capability for `HTTPRoute` implementations conforming to the relevant standard. 

### Backend errors and status conditions

A Route may be accepted while its backend reference is invalid or unavailable. Always distinguish configuration status from actual application health.

Useful concepts include:

- `Accepted` — whether the Route was accepted for processing by the relevant parent.
- `ResolvedRefs` — whether references such as backend Services can be resolved and are permitted.
- Backend readiness and health — whether the actual application endpoints can serve requests successfully.
For example, a missing Service, wrong Service port, or missing cross-namespace grant can prevent successful forwarding even when the Route itself exists.

For a Kubernetes Service backend, the port in `backendRefs` refers to the Service port, not the Pod's targetPort.

If the Service defines:

```yaml
ports:
  - port: 8080
    targetPort: 5000
```

then the Route should reference port `8080`. Kubernetes maps that Service port to the target port for the selected endpoints.

---

## Gateway Listeners and TLS Configuration

Now we need to understand how the Gateway accepts connections in the first place, especially HTTPS.

The important distinction is:
- **Listener**: Defines how the Gateway accepts connections.
- **TLS configuration**: Defines how TLS is configured for a listener.
- **HTTPRoute**: Defines how decrypted HTTP requests are routed to backends.

### What is a Gateway listener?

A listener is an entry point defined inside `Gateway.spec.listeners`.

Each listener specifies properties such as:

| Field | Purpose |
| --- | --- |
| `name` | Unique name for the listener within the Gateway |
| `protocol` | Protocol such as HTTP, HTTPS, TLS, or TCP |
| `port` | Port on which the listener accepts traffic |
| `hostname` | Optional hostname constraint |
| `tls` | TLS configuration, when applicable |
| `allowedRoutes` | Controls which Route kinds and namespaces may attach |

**For example:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gateway
  namespace: edge
spec:
  gatewayClassName: cilium
  listeners:
    - name: http
      protocol: HTTP
      port: 80

    - name: https
      protocol: HTTPS
      port: 443
      hostname: shop.example.com
      tls:
        mode: Terminate
        certificateRefs:
          - name: shop-tls
            kind: Secret
```


This Gateway defines two listeners:

- `http`: accepts HTTP connections on port 80.
- `https`: accepts HTTPS connections on port 443 and terminates TLS using the referenced certificate.

### Listener protocols

Gateway API defines several listener protocols.

| Protocol | Purpose | Typical Route types |
| --- | --- | --- |
| `HTTP` | Plain HTTP traffic | HTTPRoute |
| `HTTPS` | HTTP traffic protected by TLS | HTTPRoute |
| `TLS` | TLS traffic, potentially passed through without termination | TLSRoute |
| `TCP` | Raw TCP traffic | TCPRoute |
| `UDP` | UDP traffic | UDPRoute |

The available Route kinds and protocol combinations depend on the Gateway API specification and implementation support.

For our Cilium learning path, HTTP and HTTPS listeners are especially important because they are central to HTTP application exposure.

### TLS termination: what actually happens?

When a client connects to an HTTPS listener configured with tls.mode: Terminate, the Gateway terminates the client's TLS connection.

This means the Gateway:

1. Receives the client's TLS connection.
2. Presents its configured certificate.
3. Completes the TLS handshake with the client.
4. Decrypts the incoming HTTP traffic.
5. Processes the HTTP request and applies the matching HTTPRoute.
6. Forwards the request to the backend.

**Conceptually:**

```
       Client
          |
          | HTTPS (encrypted)
          v
  +-------------------+
  | Gateway / Envoy   |
  |                   |
  | TLS termination   |
  | HTTP routing      |
  +-------------------+
          |
          | HTTP (unencrypted)
          v
    Backend Service
          |
          v
        Backend
```

In this example, traffic between the Gateway and backend is HTTP. That does not mean Gateway API always uses HTTP for upstream connections; backend TLS can be configured separately

TLS termination at the Gateway means the original client-to-Gateway TLS session ends at the Gateway. It does not automatically encrypt the separate Gateway-to-backend connection.

**Where does the certificate come from?**

Gateway API commonly uses a Kubernetes Secret to provide the TLS certificate and private key.

A typical TLS Secret looks like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: shop-tls
  namespace: edge
type: kubernetes.io/tls
data:
  tls.crt: BASE64_ENCODED_CERTIFICATE
  tls.key: BASE64_ENCODED_PRIVATE_KEY
```

The two important keys are:

- `tls.crt`: the server certificate, normally including the required certificate chain.
- `tls.key`: the corresponding private key.e and cryptographic circumstances.

The Gateway listener references the Secret:

```yaml
tls:
  mode: Terminate
  certificateRefs:
    - kind: Secret
      name: shop-tls
```

By default, the Secret is expected in the same namespace as the Gateway. Cross-namespace references require the appropriate `ReferenceGrant` in the namespace containing the Secret.

Gateway API does not issue certificates. It only defines how a Gateway references and uses certificate material. The Gateway controller consumes the referenced certificate material.

### SNI: how does the Gateway know which certificate to present?

Imagine one Gateway has two HTTPS hostnames:

- `shop.example.com`
- `api.example.com`

Both use port `443`, but each requires a different certificate.

How can the Gateway know which certificate the client expects before it has received the HTTP request?

The answer is **SNI (Server Name Indication)**.

SNI is a TLS extension sent by the client in the `ClientHello` message. It tells the server which hostname the client is trying to reach.

For example:

```
Client
  |
  | TLS ClientHello
  | SNI: shop.example.com
  v
Gateway
  |
  | Select certificate matching shop.example.com
  v
TLS handshake continues
```


The Gateway can use SNI to select the appropriate certificate during the TLS handshake.

This is particularly important because the HTTP Host header (or HTTP/2 :authority) is sent as part of the encrypted application traffic, after TLS has been established. The Gateway cannot rely on that encrypted HTTP information to select the certificate for the initial handshake.

### Multiple HTTPS listeners

A Gateway can define multiple HTTPS listeners on the same port, provided their configurations are compatible.

For example:

```yaml
listeners:
  - name: shop-https
    protocol: HTTPS
    port: 443
    hostname: shop.example.com
    tls:
      mode: Terminate
      certificateRefs:
        - name: shop-tls

  - name: api-https
    protocol: HTTPS
    port: 443
    hostname: api.example.com
    tls:
      mode: Terminate
      certificateRefs:
        - name: api-tls
```

The two listeners use the same port but serve different hostnames and certificates.

> Gateway API defines listener compatibility and hostname-selection rules. Exact hostnames and more-specific wildcard hostnames take precedence over less-specific wildcard matches.

### istener hostname, SNI, and HTTP Host are different things

These three concepts are related, but they are not interchangeable.

| Concept | Where it appears | Purpose |
| --- | --- | --- |
| Listener `hostname` | Gateway configuration | Restricts the hostnames served by a listener |
| SNI | TLS `ClientHello` | Tells the server which hostname the client wants during TLS negotiation |
| HTTP `Host` / `:authority` | HTTP request | Identifies the requested HTTP authority |

Consider this configuration:

```yaml
listeners:
  - name: shop-https
    protocol: HTTPS
    port: 443
    hostname: shop.example.com
```

And this client request:

`https://shop.example.com/products`

The expected hostname information is:

- Listener hostname: `shop.example.com`
- TLS SNI: `shop.example.com`
- HTTP Host: `shop.example.com`

A successful HTTPS request needs the appropriate hostname matching at the TLS and HTTP routing layers.

> **Important subtlety:** SNI is supplied during the initial TLS handshake, while the HTTP authority is supplied later. They are separate protocol-level values. They should not be assumed to be identical merely because the user typed one hostname into a browser. Gateway API also defines behavior for mismatched or misdirected requests, including HTTP 421 in relevant situations.

## TLS termination vs TLS passthrough

These are two fundamentally different ways of handling TLS.

- ### TLS termination

  The Gateway decrypts the client's TLS traffic.

  ```
  Client
    |
    | Encrypted TLS
    v
  Gateway / Envoy
    |
    | TLS termination
    | HTTP request becomes visible
    v
  HTTPRoute matching
    |
    | HTTP or separately configured HTTPS
    v
  Backend
  ```

  **Characteristics:**

  - The Gateway can inspect HTTP requests.
  - `HTTPRoute` can match paths, headers, methods, and other HTTP attributes.
  - The Gateway can apply HTTP filters.
  - The Gateway handles the client-facing TLS certificate.

- ### TLS passthrough

  The Gateway does not decrypt the client's TLS traffic. It forwards the encrypted TLS connection to a backend.

  ```
  Client
    |
    | Encrypted TLS
    v
  Gateway / Envoy
    |
    | Inspect TLS ClientHello / SNI
    | Do not decrypt application traffic
    v
  Backend
    |
    | TLS termination
    v
  Application
  ```

  **Characteristics:**

  - The Gateway cannot inspect encrypted HTTP paths or headers.
  - Routing can use TLS information such as SNI.
  - The backend handles TLS termination and needs its own certificate.
  - `TLSRoute` is used rather than `HTTPRoute` for TLS passthrough routing.

  For example, you could route:

  - `shop.example.com` to one backend.
  - `api.example.com` to another backend.

  The Gateway can make that decision based on SNI without decrypting the HTTP traffic.

  Gateway API represents TLS passthrough with a TLS listener using **`tls.mode: Passthrough`**, together with a compatible `TLSRoute`. HTTPS listeners, by contrast, use TLS termination for HTTP routing. 

**Comparison**

| Feature | TLS termination | TLS passthrough |
| --- | --- | --- |
| Gateway decrypts traffic | Yes | No |
| Gateway can inspect HTTP paths | Yes | No |
| Gateway can inspect HTTP headers | Yes | No |
| HTTPRoute | Yes | No |
| TLSRoute | Not for ordinary HTTPS termination | Yes |
| Client-facing certificate | Gateway | Backend |
| TLS termination location | Gateway | Backend |

### What about encryption between Gateway and backend?

So far, we have discussed TLS between the client and the Gateway. But what if the backend also requires HTTPS?

For example:

```
Client
  |
  | HTTPS
  v
Gateway / Envoy
  |
  | HTTPS
  v
Backend Service
```

This is commonly called **TLS re-encryption**.

The Gateway:

1. Terminates the client's TLS connection.
2. Processes the HTTP request.
3. Establishes a separate TLS connection to the backend.
4. Validates the backend's certificate according to its configured trust and hostname requirements.

Gateway API provides `BackendTLSPolicy` for configuring TLS connections from the Gateway to backend Services. It defines such things as the expected backend hostname and certificate validation configuration.

This is separate from the listener's `tls` configuration:

| Configuration | TLS connection |
| --- | --- |
| Gateway listener `tls` | Client → Gateway |
| `BackendTLSPolicy` | Gateway → Backend |

**Cilium-specific considerations**

Cilium implements Gateway API using Envoy as its L7 data plane.

For HTTPS termination:
- The Gateway listener references a Kubernetes TLS Secret.
- Cilium configures Envoy to terminate TLS.
- Envoy can use SNI and the configured hostnames to select the appropriate TLS configuration.
- After TLS termination, HTTP routing rules can be applied.

For TLS passthrough:

- Envoy can inspect the SNI in the TLS ClientHello.
- It forwards the encrypted TLS stream to the selected backend.
- The backend terminates TLS.

**One operational detail:** with Cilium's TLS passthrough implementation, the backend sees Envoy's source IP for the forwarded TCP connection, rather than necessarily seeing the original client IP at the network layer. 

---

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
  
  The default Kubernetes NodePort range is `30000–32767`, although it can be changed through API server configuration.
  
  Advantages:
  
  - Does not require MetalLB.
  - Does not require a LoadBalancer implementation.
  - Uses existing node IP addresses.
  - Can work with an external load balancer or WAF.
  
  Disadvantages:
  
  - Requires managing node IPs and NodePort values.
  - External systems must know the nodes and ports.
  - Source IP visibility depends on Service configuration and networking behavior.
  - Exposes a node-level port rather than a dedicated load-balancer address.
  
  Cilium supports NodePort exposure for Gateway API.

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

# Cilium Gateway API Architecture

When you create a Gateway and an HTTPRoute, Kubernetes stores those resources. They do not directly forward packets. Cilium's control-plane components interpret the resources, translate them into proxy configuration, and configure Envoy. Cilium's networking data plane then directs traffic to the proxy, and Envoy applies the HTTP routing rules.

Cilium's official documentation describes this flow as the Cilium operator validating Gateway API resources and translating them into Cilium Envoy Configuration resources, followed by the Cilium agent supplying the configuration to Envoy.

The main components are:

- ### Kubernetes API server

  **Stores the desired configuration:** `GatewayClass`, `Gateway`, `HTTPRoute`, `Service`, `ReferenceGrant`, and related resources. It is the source of truth for the declared configuration, not the component that proxies HTTP requests.

- ### Cilium operator — Gateway API reconciliation

  Watches Gateway API resources, checks whether they are valid and can be accepted, and translates them into Cilium Envoy Configuration resources. It also updates resource status conditions to report the result of reconciliation.

- ### Cilium agent (`cilium-agent`)

  Runs on each node. It manages node-level Cilium networking, including eBPF datapath configuration, and picks up the generated Envoy configuration resources so the appropriate proxy can be configured.

- ### Envoy proxy

  Processes application traffic at Layer 7. It handles HTTP host/path/header matching, supported filters, TLS termination when configured, and forwarding requests to the selected backend.

- ### eBPF datapath and Linux networking

  Handle packet-level networking and relevant service redirection and policy enforcement. For Cilium Gateway traffic, the datapath works with TPROXY to steer traffic to Envoy, rather than having eBPF itself evaluate every HTTP path rule.


These components cooperate but have different responsibilities. The Cilium operator is not the traffic proxy, and eBPF does not replace Envoy's HTTP-routing functionality.

In particular, the Cilium operator is not the proxy, and Envoy is not the Kubernetes Gateway API controller.

## Configuration Flow

Consider these resources:

- **`GatewayClass`**: selects Cilium as the implementation.
- **`Gateway`**: declares an HTTPS listener.
- **`HTTPRoute`**: sends `/api` requests to a backend Service.
- **`Service`**: identifies the backend application endpoints through Kubernetes service discovery.

The configuration lifecycle follows this general sequence:

1. An administrator or application team creates or modifies resources such as `GatewayClass`, `Gateway`, `HTTPRoute`, and `Service`.
2. Kubernetes stores the desired state in the API server.
3. The Cilium operator watches and reconciles Gateway API resources, checks their validity, updates status conditions, and generates Cilium Envoy Configuration resources.
4. The Cilium agent picks up the generated configuration resources.
5. The agent supplies the appropriate configuration to Envoy.
6. Envoy applies the resulting configuration to incoming traffic.

This is a continuously reconciled control-plane process, not a sequence that runs from the beginning for every incoming request. Envoy processes requests using its active configuration rather than querying the Kubernetes API server for each request.

A resource's existence does not prove that its configuration has been accepted, its references resolved, or its data plane programmed successfully.

## Request Processing Flow

Consider a client requesting `https://shop.example.com/api/orders`.



1. **External delivery:** DNS, routing, firewalls, and any external WAF or load balancer deliver traffic to the exposed Gateway endpoint.
2. **Traffic steering:** Cilium networking directs the connection to Envoy. The exact path depends on the Gateway exposure mode.
3. **TLS handling:** For an HTTPS listener using TLS termination, Envoy completes the TLS handshake and decrypts the HTTP request. With TLS passthrough, Envoy forwards the encrypted stream according to the supported TLS routing configuration instead.
4. **HTTP routing:** Envoy evaluates the active configuration, including hostnames, HTTPRoute matches, and applicable filters.
5. **Backend selection:** Envoy selects the configured backend and forwards the request to a backend endpoint.
6. **Network delivery and policy:** Cilium networking and policy mechanisms handle the relevant traffic delivery and enforcement.
7. **Response:** The backend response returns through the proxying and networking path. For TLS termination, Envoy manages the client-facing TLS connection.

The steps describe the logical processing flow; they do not necessarily correspond to separate physical network hops.

## 4. eBPF, TPROXY, and Envoy

These technologies have different responsibilities.

| Component | Responsibility |
|---|---|
| eBPF | Implements parts of Cilium's kernel-level networking, service handling, and policy datapath. |
| TPROXY | Provides transparent traffic redirection to a proxy while preserving connection properties required by the proxy. |
| Envoy | Processes Layer 7 traffic, including HTTP routing, supported filters, and TLS termination. |
| Cilium agent | Manages node-level networking and supplies proxy configuration. |
| Cilium operator | Reconciles Gateway API resources and generates configuration resources. |

Cilium's Gateway API implementation uses TPROXY to steer traffic to Envoy. In the documented default configuration, `bpf.tproxy=false`, TPROXY uses iptables and requires the relevant netfilter modules. The alternative `bpf.tproxy=true` provides an eBPF-based implementation and is documented as beta in the referenced Cilium release.

**Important:** It is inaccurate to say that eBPF performs all Gateway API HTTP routing. eBPF and Linux networking handle packet-level operations and redirection; Envoy evaluates HTTP-level routing rules and applies supported Layer 7 filters.

Cilium Gateway API requires kube-proxy replacement (`kubeProxyReplacement=true`) and the L7 proxy (`l7Proxy=true`).

## 5. Envoy Deployment Models

Cilium supports two Envoy deployment models:

- **Embedded mode:** Envoy runs as a process within the `cilium-agent` Pod on each node.
- **Standalone mode:** Envoy runs in a separate `cilium-envoy` DaemonSet.

In both models, Envoy handles the relevant Layer 7 traffic on the node. The deployment model affects operational and security configuration details.

Do not assume that every Gateway resource creates a dedicated Envoy Pod. Cilium's proxy infrastructure can handle configuration for multiple Gateway listeners and routes.

## 6. Exposure and Traffic Redirection

Cilium Gateway API commonly exposes its proxy through a `LoadBalancer` Service. Depending on the environment and configuration, NodePort or host-network exposure may be used instead.

With Service-based exposure, traffic arriving at the Gateway Service is transparently forwarded to Envoy using TPROXY.

With host-network exposure, Gateway listeners are exposed on selected node network interfaces and ports. Host-network mode changes how the listener is exposed; it does not replace Envoy as the Layer 7 proxy.

Listener configuration alone does not guarantee external reachability. Routing, firewall rules, WAF or load-balancer configuration, and node availability must also be considered.

## 7. Troubleshooting the Configuration Lifecycle

When Gateway traffic fails, check the following separately:

1. **Resource acceptance:** Is the Gateway or HTTPRoute accepted? Inspect status conditions.
2. **Reference resolution:** Are the referenced Services, Secrets, and other resources found and permitted? Inspect `ResolvedRefs` and related conditions.
3. **Gateway programming:** Is the Gateway successfully programmed? Inspect its `Programmed` condition and reported addresses.
4. **Proxy configuration:** Has the intended configuration reached Envoy?
5. **External reachability:** Can the client or WAF reach the configured Gateway endpoint and port?
6. **Backend delivery:** Is the selected Service and its endpoint reachable, and does Cilium policy permit the traffic?
7. **TLS correctness:** If HTTPS is involved, check listener configuration, certificates, hostname compatibility, and backend TLS requirements where applicable.

A successful resource status does not, by itself, prove that external networking, backend connectivity, and application behavior are all working.

---

# Deploying Cilium Gateway API from Scratch

## Scenario design

> ### Frontend application
> 
> - Namespace **apps**
> - Service **frontend** 
> - Port **80**
>
> ### API application
> 
> - Namespace **apps**
> - Service **api**
> - Port **8080**
> 
> ```
> ┌─────────────────────────────────────────┐
> |         External client / WAF           |
> |        http://NODE_IP:NODEPORT          |
> └─────────────────────────────────────────┘
>                     ↓
> ┌─────────────────────────────────────────┐
> |     Cilium Gateway — namespace edge     |
> |   GatewayClass → Gateway → HTTPRoute    |
> |   NodePort Service exposes the proxy    |
> └─────────────────────────────────────────┘
>                     ↓
> ┌───────────────────┐┌────────────────────┐
> |    / and /app     ||       /api         |
> | Frontend Service  ||    API Service     |
> └───────────────────┘└────────────────────┘
> ``` 
> We'll use two namespaces to demonstrate a realistic separation of responsibilities:
> - **edge**: owned by the platform/networking team; contains the Gateway and routes.
> - **apps**: contains application Deployments and Services.
> 
> Because the routes in edge will reference Services in `apps`, we'll configure a `ReferenceGrant` in apps. This makes the cross-namespace permission explicit.
>
> For the exposure method, Cilium normally creates a `LoadBalancer` Service for a Gateway. We'll configure a `CiliumGatewayClassConfig` to use `NodePort` instead. Cilium documents this configuration mechanism, including the generated NodePort Service.

### Requirements

- Cilium is already installed as the cluster CNI.
- Cilium is configured with kube-proxy replacement and the L7 proxy enabled.
- Gateway API CRDs are installed.
- No external Kubernetes LoadBalancer implementation is available.
- Gateway traffic is exposed through a `NodePort`.
- Applications are deployed in the `apps` namespace.
- Gateway and routing resources are managed in the `edge` namespace.

### Resources

| Resource | Namespace | Purpose |
|---|---|---|
| `GatewayClass` | Cluster-scoped | Selects the Cilium Gateway controller and its configuration |
| `CiliumGatewayClassConfig` | `edge` | Configures generated Gateway Services to use NodePort |
| `Gateway` | `edge` | Defines the HTTP listener |
| `HTTPRoute` | `edge` | Defines HTTP routing rules |
| `ReferenceGrant` | `apps` | Permits the route to reference Services in another namespace |
| `Deployment` | `apps` | Runs the application Pods |
| `Service` | `apps` | Provides stable endpoints for the applications |

The intended traffic flow is:

`Client → Node IP:NodePort → Cilium Gateway / Envoy → HTTPRoute → Application Service → Pod`

## 1. Prerequisites and Enable Gateway API

### 1.1 Verify Cilium

Check Cilium's status:

```bash
cilium status
```

Inspect the installed Cilium version and Helm values:

```bash
helm list -n kube-system
helm get values cilium -n kube-system
```

The Helm release name may differ from `cilium` in your cluster.

Cilium Gateway API requires kube-proxy replacement:

```yaml
kubeProxyReplacement: true
```

The L7 proxy must also be enabled:

```yaml
l7Proxy: true
```

### 1.2 Install Gateway API Standard CRDs

For the Cilium 1.20.4 example, install Gateway API v1.6.3 Standard CRDs:

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml

kubectl apply --server-side -f \
  https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_gateways.yaml

kubectl apply --server-side -f \
  https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml

kubectl apply --server-side -f \
  https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.6.3/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
```

These CRDs define the Kubernetes API types. Installing them does not install or enable the Cilium controller.

Verify that the resources are registered:

```bash
kubectl get crd gatewayclasses.gateway.networking.k8s.io
kubectl get crd gateways.gateway.networking.k8s.io
kubectl get crd httproutes.gateway.networking.k8s.io
kubectl get crd referencegrants.gateway.networking.k8s.io
```

### 1.3 Enable Cilium Gateway API Support

If Gateway API support is not already enabled, update the existing Cilium Helm release. For example:

```bash
helm upgrade cilium cilium/cilium \
  --version 1.20.4 \
  --namespace kube-system \
  --reuse-values \
  --set kubeProxyReplacement=true \
  --set gatewayAPI.enabled=true
```

Use the repository, chart version, and release name appropriate to your installation. Do not blindly apply this example to a cluster running another Cilium release.

Verify the result:

```bash
cilium status
kubectl get pods -n kube-system
```

Cilium's Gateway API implementation uses Envoy for Layer 7 proxying. In the default TPROXY configuration, the nodes must have the required iptables and netfilter support.

---

## 2. Create Namespaces

Create separate namespaces for platform-managed Gateway resources and application resources:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: edge
  labels:
    app.kubernetes.io/part-of: gateway-demo
---
apiVersion: v1
kind: Namespace
metadata:
  name: apps
  labels:
    app.kubernetes.io/part-of: gateway-demo
```

Apply the file:

```bash
kubectl apply -f namespaces.yaml
```

The namespaces separate ownership and provide a useful demonstration of cross-namespace permissions. They do not automatically provide network isolation; use RBAC and network policies where required.

---

## 3. Deploy the Applications

We will deploy three simple HTTP backends:

- `frontend`: the default web application.
- `api-v1`: the primary API version.
- `api-v2`: a second API version for testing weighted traffic distribution and header-based routing.

For this demonstration, all applications use the `hashicorp/http-echo` image. These are simple test servers, not production application images.

- ### 3.1 Frontend Deployment and Service

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: frontend
    namespace: apps
    labels:
      app: frontend
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: frontend
    template:
      metadata:
        labels:
          app: frontend
      spec:
        containers:
          - name: frontend
            image: hashicorp/http-echo:1.0.0
            args:
              - "-listen=:8080"
              - "-text=frontend"
            ports:
              - name: http
                containerPort: 8080
                protocol: TCP
            readinessProbe:
              tcpSocket:
                port: http
              initialDelaySeconds: 2
              periodSeconds: 5
            livenessProbe:
              tcpSocket:
                port: http
              initialDelaySeconds: 5
              periodSeconds: 10
            resources:
              requests:
                cpu: 50m
                memory: 32Mi
              limits:
                cpu: 200m
                memory: 64Mi
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: frontend
    namespace: apps
    labels:
      app: frontend
  spec:
    type: ClusterIP
    selector:
      app: frontend
    ports:
      - name: http
        port: 80
        targetPort: http
        protocol: TCP
  ```

  The Service exposes port `80` inside the cluster and forwards traffic to the Pods named container port `http`, which is `8080`.

- ### 3.2 API v1 Deployment and Service

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: api-v1
    namespace: apps
    labels:
      app: api
      version: v1
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: api
        version: v1
    template:
      metadata:
        labels:
          app: api
          version: v1
      spec:
        containers:
          - name: api
            image: hashicorp/http-echo:1.0.0
            args:
              - "-listen=:8080"
              - "-text=api-v1"
            ports:
              - name: http
                containerPort: 8080
                protocol: TCP
            readinessProbe:
              tcpSocket:
                port: http
              initialDelaySeconds: 2
              periodSeconds: 5
            resources:
              requests:
                cpu: 50m
                memory: 32Mi
              limits:
                cpu: 200m
                memory: 64Mi
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: api-v1
    namespace: apps
    labels:
      app: api
      version: v1
  spec:
    type: ClusterIP
    selector:
      app: api
      version: v1
    ports:
      - name: http
        port: 8080
        targetPort: http
        protocol: TCP
  ```

- ### 3.3 API v2 Deployment and Service

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: api-v2
    namespace: apps
    labels:
      app: api
      version: v2
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: api
        version: v2
    template:
      metadata:
        labels:
          app: api
          version: v2
      spec:
        containers:
          - name: api
            image: hashicorp/http-echo:1.0.0
            args:
              - "-listen=:8080"
              - "-text=api-v2"
            ports:
              - name: http
                containerPort: 8080
                protocol: TCP
            readinessProbe:
              tcpSocket:
                port: http
              initialDelaySeconds: 2
              periodSeconds: 5
            resources:
              requests:
                cpu: 50m
                memory: 32Mi
              limits:
                cpu: 200m
                memory: 64Mi
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: api-v2
    namespace: apps
    labels:
      app: api
      version: v2
  spec:
    type: ClusterIP
    selector:
      app: api
      version: v2
    ports:
      - name: http
        port: 8080
        targetPort: http
        protocol: TCP
  ```

  Apply the application manifests:

  ```bash
  kubectl apply -f applications.yaml
  ```

  Check their status:

  ```bash
  kubectl get deployments -n apps
  kubectl get pods -n apps -o wide
  kubectl get services -n apps
  kubectl get endpointslices -n apps
  ```

  Wait until the Deployments have their desired available replicas before proceeding.

---

## 4. Configure a GatewayClass to Use NodePort

Cilium normally creates a `LoadBalancer` Service for a Gateway. Since this scenario does not assume a LoadBalancer implementation, we will configure Cilium to generate a `NodePort` Service.

Cilium provides a custom resource named `CiliumGatewayClassConfig` for implementation-specific Gateway settings.

**Note:** `CiliumGatewayClassConfig` uses Cilium's `cilium.io/v2alpha1` API in this example. It is a Cilium-specific, alpha API, not a portable Gateway API resource. Check its schema and compatibility against your installed Cilium release.

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumGatewayClassConfig
metadata:
  name: nodeport-config
  namespace: edge
spec:
  description: "Cilium Gateway configuration for NodePort exposure"
  service:
    type: NodePort
    externalTrafficPolicy: Cluster
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium-nodeport
spec:
  controllerName: io.cilium/gateway-controller
  description: "Cilium GatewayClass using NodePort exposure"
  parametersRef:
    group: cilium.io
    kind: CiliumGatewayClassConfig
    name: nodeport-config
    namespace: edge
```

Apply the resources:

```bash
kubectl apply -f gateway-class.yaml
```

Inspect the GatewayClass:

```bash
kubectl get gatewayclass
kubectl describe gatewayclass cilium-nodeport
kubectl get ciliumgatewayclassconfig -n edge
```

Important fields:

- `controllerName`: identifies the controller responsible for this GatewayClass. Cilium uses `io.cilium/gateway-controller`.
- `parametersRef.group`: the API group containing the implementation-specific configuration.
- `parametersRef.kind`: the custom resource kind.
- `parametersRef.name`: the referenced configuration resource.
- `parametersRef.namespace`: the namespace containing the configuration resource.
- `service.type`: tells Cilium to generate a NodePort Service instead of the default LoadBalancer Service.
- `service.externalTrafficPolicy`: controls how external traffic is routed by the generated Service. `Cluster` permits routing to eligible endpoints across nodes; source-IP visibility depends on the network path and proxy behavior.

The `GatewayClass` is cluster-scoped, but the `CiliumGatewayClassConfig` referenced by it is namespaced.

## 5. Create the Gateway

The Gateway defines the HTTP listener and which HTTPRoutes may attach to it.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: applications-gateway
  namespace: edge
  labels:
    app.kubernetes.io/part-of: gateway-demo
  annotations:
    description: "HTTP Gateway for frontend and API applications"
spec:
  gatewayClassName: cilium-nodeport
  listeners:
    - name: http
      hostname: demo.example.com
      protocol: HTTP
      port: 80
      allowedRoutes:
        kinds:
          - kind: HTTPRoute
        namespaces:
          from: All
```

Apply the manifest:

```bash
kubectl apply -f gateway.yaml
```

### Explanation of the important Gateway fields

- `metadata.name` and `metadata.namespace`: identify the Gateway.
- `metadata.labels` and `metadata.annotations`: provide metadata for organization and tooling. Annotations do not automatically change Gateway behavior.
- `spec.gatewayClassName`: selects the GatewayClass and therefore the implementation.
- `listeners`: defines the entry points handled by the Gateway.
- `listeners[].name`: identifies the listener; Routes can refer to it using `parentRefs.sectionName`.
- `listeners[].hostname`: limits the hostnames accepted by the listener.
- `listeners[].protocol`: declares the listener protocol. Here it is HTTP, not HTTPS.
- `listeners[].port`: defines the listener port. This is not the NodePort number.
- `listeners[].allowedRoutes.kinds`: permits HTTPRoute attachment.
- `listeners[].allowedRoutes.namespaces.from`: controls which namespaces may attach Routes.

In this example, `from: All` permits compatible HTTPRoutes from any namespace to attach. This is deliberately broad for demonstration. In a production environment, restrict attachment according to the intended ownership model.

The listener's hostname is `demo.example.com`, so requests should use that hostname. A request to the node IP with a different Host header will not match this listener's hostname.

## 6. Permit Cross-Namespace Backend References

The `HTTPRoute` will live in `edge`, while the backend Services live in `apps`.

Gateway API uses `ReferenceGrant` to permit references to resources in another namespace. The grant must be created in the namespace containing the target resources—in this case, `apps`.

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-edge-routes
  namespace: apps
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: edge
  to:
    - group: ""
      kind: Service
```

Apply it:

```bash
kubectl apply -f reference-grant.yaml
```

This grant permits HTTPRoutes in `edge` to reference Services in `apps`. It is not limited to a single Service name, so the permission is broader than a single backend.

`ReferenceGrant` does not grant Kubernetes RBAC permissions, does not replace application authentication, and does not authorize arbitrary traffic at the network layer.

## 7. Configure HTTP Routing

We will use one `HTTPRoute` with several rules:

1. Requests to `/api/canary` with the header `x-release: canary` go to API v2.
2. Requests to `/api/orders` are split between API v1 and API v2.
3. Requests to `/api` use API v1 and receive a request header added by the Gateway.
4. Requests to `/app` are rewritten to `/` and forwarded to the frontend.
5. Other paths are forwarded to the frontend.

The order and specificity of matching matter. The canary route has a more specific path and an additional header condition; the API order route is also more specific than the general `/api` rule.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: applications-route
  namespace: edge
  labels:
    app.kubernetes.io/part-of: gateway-demo
spec:
  parentRefs:
    - name: applications-gateway
      namespace: edge
      sectionName: http
  hostnames:
    - demo.example.com
  rules:
    - name: api-canary
      matches:
        - path:
            type: PathPrefix
            value: /api/canary
          headers:
            - type: Exact
              name: x-release
              value: canary
      backendRefs:
        - name: api-v2
          namespace: apps
          port: 8080
          weight: 1

    - name: api-orders-split
      matches:
        - path:
            type: PathPrefix
            value: /api/orders
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: x-gateway
                value: cilium
      backendRefs:
        - name: api-v1
          namespace: apps
          port: 8080
          weight: 90
        - name: api-v2
          namespace: apps
          port: 8080
          weight: 10

    - name: api-general
      matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            set:
              - name: x-environment
                value: demo
            remove:
              - x-debug
      backendRefs:
        - name: api-v1
          namespace: apps
          port: 8080
          weight: 1

    - name: frontend-app
      matches:
        - path:
            type: PathPrefix
            value: /app
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
      backendRefs:
        - name: frontend
          namespace: apps
          port: 80
          weight: 1

    - name: frontend-default
      matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend
          namespace: apps
          port: 80
          weight: 1
```

Apply the route:

```bash
kubectl apply -f http-route.yaml
```

### Explanation of HTTPRoute fields

- `metadata.name` and `metadata.namespace`: identify the Route.
- `parentRefs`: specifies the Gateway to which the Route attaches.
- `parentRefs[].namespace`: identifies the Gateway's namespace. If omitted, the Route's own namespace is assumed.
- `parentRefs[].sectionName`: selects the specific listener named `http`.
- `hostnames`: limits this Route to requests matching `demo.example.com`.
- `rules`: contains the routing rules.
- `rules[].name`: provides an optional human-readable identifier for a rule.
- `matches`: defines request-matching conditions.
- `matches[].path.type`: selects the path matching mode, such as `Exact` or `PathPrefix`.
- `matches[].path.value`: specifies the path value.
- `matches[].headers`: matches request headers. Here, the match is exact.
- `filters`: modifies the request before forwarding or performs another supported routing action.
- `RequestHeaderModifier`: adds, sets, or removes request headers.
- `URLRewrite`: modifies the request URL before forwarding. `ReplacePrefixMatch` replaces the matched path prefix.
- `backendRefs`: identifies the backend Services.
- `backendRefs[].name`: identifies the Service.
- `backendRefs[].namespace`: identifies the Service's namespace. Cross-namespace references require permission.
- `backendRefs[].port`: identifies the **Service port**, not the Pod's `containerPort`.
- `backendRefs[].weight`: defines the relative traffic weight for a backend when multiple backends appear in the same rule.

### Understanding the routing behavior

**Canary rule:** `/api/canary` is sent to API v2 only when `x-release: canary` is present. Requests to that path without the matching header fall through to the more general `/api` rule.

**Weighted split:** `/api/orders` is split with weights 90 and 10. This represents a 90:10 relative distribution, not a guarantee that every group of ten requests will contain exactly nine requests to v1 and one to v2.

**General API rule:** `/api` routes to API v1. The request header modifier sets `x-environment: demo` and removes `x-debug`.

**Frontend rewrite:** `/app` is rewritten to `/` before forwarding. For example, a request to `/app/products` is forwarded with the prefix replaced, resulting in `/products`. The browser's original URL does not change.

**Default frontend rule:** `/` is a broad path prefix that matches requests not captured by more-specific matching rules. Gateway API's defined route precedence determines which matching rule wins.

`RequestHeaderModifier` and `URLRewrite` are standard Gateway API filters, but implementation support must always be checked for the chosen controller and version.

## 8. Verify the Gateway and NodePort

First, inspect the Gateway and Route status:

```bash
kubectl get gatewayclass
kubectl get gateway -n edge
kubectl describe gateway applications-gateway -n edge
kubectl get httproute -n edge
kubectl describe httproute applications-route -n edge
```

Check the relevant status conditions:

- `Accepted`: whether the resource was accepted.
- `Programmed`: whether the Gateway configuration has been programmed, where applicable.
- `ResolvedRefs`: whether references such as backend Services were resolved successfully.

Now locate the generated NodePort Service:

```bash
kubectl get svc -A | grep cilium-gateway
```

Cilium commonly generates a Service named using the pattern `cilium-gateway-<gateway-name>`. Verify the actual name in your cluster instead of assuming it.

Inspect the Service:

```bash
kubectl get svc -n edge
kubectl describe svc -n edge <generated-gateway-service-name>
```

A NodePort Service will show a mapping similar to:

```text
TYPE       PORT(S)
NodePort   80:30xxx/TCP
```

Here, `80` is the Service port and `30xxx` is the assigned NodePort. The actual value is assigned by Kubernetes unless explicitly configured by a supported mechanism. Do not assume it is `30080`.

Obtain the assigned NodePort:

```bash
kubectl get svc -n edge <generated-gateway-service-name> \
  -o jsonpath='{.spec.ports[*].nodePort}{"\n"}'
```

The Gateway's listener port and the Service's NodePort are different values:

- Gateway listener: port 80.
- NodePort: a node-level port from the cluster's configured NodePort range, commonly 30000–32767.

## 10. Test the Application

Set these shell variables using the actual values in your cluster:

```bash
NODE_IP=<reachable-kubernetes-node-ip>
NODEPORT=<assigned-nodeport>
```

Because the listener hostname is `demo.example.com`, include that hostname in requests. For testing without DNS, use `curl`'s `--resolve` option:

```bash
curl --resolve demo.example.com:${NODEPORT}:${NODE_IP} \
  http://demo.example.com:${NODEPORT}/
```

Expected body:

```text
frontend
```

Test the rewritten frontend path:

```bash
curl --resolve demo.example.com:${NODEPORT}:${NODE_IP} \
  http://demo.example.com:${NODEPORT}/app/products
```

Expected body:

```text
frontend
```

Test the general API route:

```bash
curl --resolve demo.example.com:${NODEPORT}:${NODE_IP} \
  http://demo.example.com:${NODEPORT}/api/status
```

Expected body:

```text
api-v1
```

Test the canary rule:

```bash
curl --resolve demo.example.com:${NODEPORT}:${NODE_IP} \
  -H 'x-release: canary' \
  http://demo.example.com:${NODEPORT}/api/canary
```

Expected body:

```text
api-v2
```

Test the weighted route repeatedly:

```bash
for i in $(seq 1 20); do
  curl -s --resolve demo.example.com:${NODEPORT}:${NODE_IP} \
    http://demo.example.com:${NODEPORT}/api/orders
  echo
done
```

Responses should include both `api-v1` and `api-v2` over a sufficiently large number of requests, though a short sample may not reflect the configured 90:10 ratio.

The `hashicorp/http-echo` applications return fixed text and do not display request headers, so the header-modifier behavior should be verified using a backend that echoes request headers or through suitable proxy-level diagnostics.

## 10. Troubleshooting

- ### GatewayClass is not accepted

  Check:

  ```bash
  kubectl describe gatewayclass cilium-nodeport
  kubectl get ciliumgatewayclassconfig -n edge
  ```

  Confirm the controller name, the parameters reference, and the Cilium-specific configuration API version.

- ### Gateway is not programmed

  Check:

  ```bash
  kubectl describe gateway applications-gateway -n edge
  kubectl get svc -n edge
  ```

  Confirm that the selected GatewayClass is accepted and that the listener configuration is valid.

- ### HTTPRoute is not accepted or references are unresolved

  Check:

  ```bash
  kubectl describe httproute applications-route -n edge
  kubectl get referencegrant -n apps
  kubectl get svc -n apps
  ```

  Common causes include a missing Gateway, a listener that does not allow the Route, hostname incompatibility, a missing ReferenceGrant, or an incorrect backend Service port.

- ### NodePort is unreachable

  Verify:

  1. The generated Service is actually `NodePort`.
  2. The assigned NodePort is correct.
  3. The node IP is reachable from the client or WAF.
  4. Node firewalls and network ACLs allow the NodePort.
  5. Cilium is healthy and the required L7 proxy and TPROXY dependencies are available.
  6. The Gateway's listener and Route are programmed successfully.

  A working NodePort Service does not automatically make the port reachable from the internet. External routing and firewall configuration are still required.

- ### Requests reach the wrong backend

  Check the request hostname, path, headers, and query parameters against the Route's matching conditions. Review rule precedence and verify that the request actually matches the intended rule.

- ### Backend endpoints are missing

  Check:

  ```bash
  kubectl get pods -n apps
  kubectl get svc -n apps
  kubectl get endpointslices -n apps
  ```

  Confirm that the Pods are ready and that the Service selectors match their labels.

## 11. Production Considerations

This scenario is intended to teach Gateway API concepts and verify a working Cilium implementation. Before using it in production:

- Use real application images and appropriate resource settings.
- Configure HTTPS and certificate management.
- Restrict Route attachment and cross-namespace references according to ownership boundaries.
- Apply RBAC and appropriate network policies.
- Restrict external access to the intended node addresses and NodePort.
- Configure WAF health checks and redundant Gateway targets where availability requires them.
- Verify source-IP and forwarded-header behavior across the complete network path.
- Use supported, version-compatible Cilium features and monitor the Gateway and backend health.

NodePort provides an exposure mechanism, not a load-balancing or high-availability architecture by itself.

---