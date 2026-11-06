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

- A Gateway has one HTTPS listener for `shop.example.com`.
- One HTTPRoute handles `/api`.
- Another HTTPRoute handles `/store`.
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

For example, a Route in namespace `store` can reference a Gateway in namespace `edge`:

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

> **Important:** Referencing a Gateway does not automatically grant permission to attach to it. The listener's `allowedRoutes` configuration must permit the attachment.

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

> **One subtle point:** `allowedRoutes` and **hostname** compatibility solve different problems. A Route may be allowed by `namespace` and `kind` but still fail hostname compatibility. Conversely, compatible hostnames do not override a namespace restriction.

## What happens when multiple Routes have overlapping rules?

Imagine two Routes attached to the same listener:

- Route A matches `/api`.
- Route B matches `/api/orders`.

A request for `/api/orders` matches both prefix rules. The Gateway API specification defines precedence rules so that implementations can resolve such overlaps consistently.

For path matching, the main rules are:

1. An **exact path** match takes precedence over a prefix match.
2. Among prefix matches, the longer matching prefix takes precedence.
3. When matches remain tied, additional precedence rules consider other match criteria, such as method and header matches, followed by specification-defined tie-breakers.

Do not assume that whichever Route was created last wins. Creation time is relevant only in particular tie-breaking situations defined by the specification.

Also, the precedence rules apply to matching requests; they do not mean every conflict between Route definitions is resolved simply by choosing one Route. Invalid references, incompatible attachments, and implementation-specific limitations can still affect whether a Route is accepted.

### Complete precedence order

Gateway API defines a more detailed order for resolving overlapping HTTPRoute matches. The following is the relevant precedence sequence for standard HTTP path, method, header, and query-parameter matching. 

1. **Exact path** match takes precedence over prefix matching.
2. **Longest matching prefix** takes precedence over shorter prefixes.
3. **Method match** takes precedence over a match that does not specify a method, when the earlier criteria tie.
4. **More matching header** conditions take precedence over fewer header conditions.
5. **More matching query-parameter** conditions take precedence over fewer query-parameter conditions.
6. If matches from different Routes are still tied, the **oldest Route** wins, based on its creation timestamp.
7. If the creation timestamps are also tied, the Route whose **{namespace}/{name} sorts first alphabetically** wins.

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

- **Standard behavior**: What Gateway API defines, including path-match precedence and Route attachment.
- **Implementation support**: Which features a particular Cilium release implements, especially features that are experimental or implementation-specific.

Cilium's supported features can change between releases. Its current documentation should be checked against the version you plan to run rather than assuming every available Gateway API feature works in every Cilium version.

