# Architecture and Core Resources

Kubernetes Gateway API separates the configuration of traffic entry points from the rules that determine how application traffic is routed.

Its core resource model includes:

- `GatewayClass`: identifies the controller responsible for managing a class of Gateways.
- `Gateway`: defines a traffic entry point and its listeners.
- `HTTPRoute`: defines HTTP request-matching and routing rules.
- `Service`: commonly represents a backend application destination.

These resources describe desired networking behavior. A compatible Gateway API controller reconciles the resources and implements that behavior using its networking infrastructure.

# GatewayClass

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

---

# Gateway

A `Gateway` represents a request for a particular traffic-handling capability and references a `GatewayClass`.

Example:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
    name: production-gateway
    namespace: edge
    labels:
        app.kubernetes.io/name: production-gateway
        app.kubernetes.io/managed-by: platform-team
    annotations:
        description: "Production HTTP and HTTPS entry point"
spec:
    gatewayClassName: cilium

    listeners:
    # HTTP listener
    - name: http
      hostname: shop.example.com
      protocol: HTTP
      port: 80
      allowedRoutes:
        kinds:
        - group: gateway.networking.k8s.io
          kind: HTTPRoute
        namespaces:
            from: Selector
            selector:
                matchLabels:
                    gateway-access: "true"

    # HTTPS listener
    - name: https
      hostname: shop.example.com
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - group: ""
            kind: Secret
            name: shop-example-com-tls
      allowedRoutes:
        kinds:
          - group: gateway.networking.k8s.io
            kind: HTTPRoute
        namespaces:
          from: Selector
          selector:
            matchLabels:
              gateway-access: "true"
```

- `apiVersion` → Identifies the Gateway API version.
- `kind` → Declares the resource type as Gateway.
- `metadata.name` → Name of the Gateway.
- `metadata.namespace` → Namespace containing the Gateway.
- `metadata.labels` → Labels for organization and selection.
- `metadata.annotations` → Additional metadata; annotations do not automatically configure Gateway behavior.
- `spec.gatewayClassName` → References the GatewayClass that determines which controller implements the Gateway.
- `spec.listeners` → Defines the network listeners exposed by the Gateway.
- `listeners[].name` → Unique name identifying a listener within this Gateway. (Listener names are important because an `HTTPRoute` can select a specific listener through `parentRefs.sectionName`)
- `listeners[].hostname` → Restricts the hostnames accepted by the listener.
- `listeners[].protocol` → Specifies the protocol: HTTP, HTTPS, TLS, TCP, or UDP, subject to implementation support.
- `listeners[].port` → Port on which the listener accepts traffic.
- `listeners[].allowedRoutes` → Controls which Route kinds and namespaces may attach to the listener.

    It controls two separate things:
    - Which kinds of Routes can attach to the listener.
    - Which namespaces are allowed to provide those Routes.

    The namespace-selection options are:

    - `Same` → Only Routes from the Gateway's own namespace can attach.
    - `All` → Routes from any namespace can attach, subject to other compatibility requirements.
    - `Selector` → Only Routes from namespaces matching the specified label selector can attach.

    > In this example application namespaces must have the label `gateway-access: "true"` to attach their HTTPRoutes.

- `allowedRoutes.kinds` → Specifies permitted Route kinds and their API groups.
- `allowedRoutes.namespaces.from` → Defines the namespace-selection policy.
- `allowedRoutes.namespaces.selector` → Selects namespaces using Kubernetes labels.
- `listeners[].tls.mode` → Defines how TLS is handled. Terminate means TLS terminates at the Gateway.
- `listeners[].tls.certificateRefs` → References the certificate and private key used by the HTTPS listener.

A `listener` defines the traffic that the Gateway is intended to accept. Its configuration also determines which Route kinds and namespaces may attach.

Defining a `listener` does not guarantee external reachability. The controller and underlying infrastructure must implement the configuration and expose the required networking resources.

---

# HTTPRoute

`HTTPRoute` defines how HTTP requests are matched and forwarded to backend destinations.

**Example:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop-route
  namespace: api
  labels:
    app.kubernetes.io/name: shop
    app.kubernetes.io/component: routing
spec:
  parentRefs:
    - name: production-gateway
      namespace: edge
      sectionName: https

  hostnames:
    - shop.example.com

  rules:
    # API requests
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: shop-api
          port: 8080
          weight: 1

    # Frontend requests
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: shop-frontend
          port: 80
          weight: 1
```

### Understand each section

- ### `metadata`

    The `HTTPRoute` is created in the `api` namespace. This is important because the Gateway is in the edge namespace. The Route and Gateway do not need to be in the same namespace.

    However, the Gateway's `allowedRoutes` configuration must permit Routes from the `api` namespace.

    Our previous Gateway used this selector:

    ```yaml
    namespaces:
      from: Selector
      selector:
        matchLabels:
          gateway-access: "true"
    ```

    Therefore, label the `api` namespace:
    ```bash
    kubectl label namespace api gateway-access=true
    ```

    Without that label, the Route won't be permitted to attach to this Gateway listener.

- ### `parentRefs`

    This tells the `HTTPRoute` which Gateway it wants to attach to.

    - `name` → Name of the Gateway
    - `namespace` → Namespace containing the Gateway
    - `sectionName` → Specific Gateway listener to attach to

    **`sectionName: https`** refers to the listener named https in our previous Gateway resource.

    This is useful because the Gateway has two listeners:
    - `http` on port `80`
    - `https` on port `443`

    By specifying `https`, this Route attaches only to the HTTPS listener.

- ### `hostnames`

    This defines the hostname that incoming HTTP requests must match.

    For example: `https://shop.example.com/api/orders`

    > **The hostname must be compatible with the hostname configured in the Gateway listener.**

    In our example, both use `shop.example.com`, so they are compatible.


- ### `rules`

    `Rules` define what happens after a request matches the Route's hostname and is accepted by the listener.

    Each rule can contain:

    - `matches`: Conditions that determine which requests match.
    - `filters`: Optional request/response processing.
    - `backendRefs`: Destination backends.
    - `timeouts`: Optional request and backend timeout configuration.

    Our example has two rules.

    - **Rule 1 — API requests**

        Requests whose path begins with /api are forwarded to the shop-api Service.

        **Examples:**

        | Request path | Matches? |
        | --- | --- |
        | `/api` | Yes |
        | `/api/orders` | Yes |
        | `/api/v1/users` | Yes |
        | `/apiary` | NO |
        | `/orders` | No |

        > `PathPrefix` uses path-element boundaries. Therefore, `/api` matches `/api/orders`, but does not match `/apiv2`.

    - **Rule 2 — Frontend requests**

        This matches all remaining paths and forwards them to the frontend Service.

        For example:
        - `/`
        - `/products`
        - `/account/profile`

        The `/api` rule is more specific than `/`, so API requests are routed to `shop-api`, not the `frontend`.

- ### `backendRefs`

    This identifies the destination Kubernetes Service.

    **Important details:**

    - `name` is the Kubernetes Service name, not the Deployment or Pod name.
    - `port` is the Service's spec.ports[].port, not its targetPort.
    - Since no namespace is specified, the Service is expected to be in the same namespace as the HTTPRoute.
    - `weight` controls the relative distribution of traffic when multiple backends are configured.

    For example, if the Service is:

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: shop-api
      namespace: api
    spec:
      selector:
        app: shop-api
      ports:
        - port: 8080
          targetPort: 5000
    ```

    The HTTPRoute must use port 8080, not 5000.

An `HTTPRoute` may contain multiple `rules`, and multiple `HTTPRoutes` may attach to the same `Gateway`, subject to compatibility and attachment policies.

## Route attachment

A Route must successfully attach to an appropriate Gateway listener before the controller can implement the intended routing behavior.

Attachment involves:

1. The Route referencing its intended parent through `parentRefs`.
2. The Gateway listener allowing the Route `kind` and `namespace`.
3. The Route and listener having compatible configuration, including hostname constraints where applicable.
4. The controller accepting and programming the resulting configuration.

The Gateway listener's `allowedRoutes` field controls which Route `kinds` and `namespaces` can attach. Namespace restrictions allow infrastructure operators to share a Gateway while limiting which application teams can configure its routing.

Cross-namespace backend references are a separate permission concern. A `ReferenceGrant` may be required to authorize a Route to reference an object in another namespace.

# Controller and data plane

A Gateway API controller watches and reconciles Gateway API resources, validates configuration, programs the implementation, and reports status.

The controller is software, not a standard Gateway API resource `kind`.

The data plane processes actual network traffic. Depending on the implementation, it may use a proxy such as Envoy together with other networking components.

For Cilium, Gateway API support integrates Cilium's networking platform with Envoy for supported Layer 7 proxying.

##  Status and troubleshooting

Resource existence alone does not prove successful implementation.

Important status conditions include:

- `Accepted`: indicates whether the controller accepts the relevant configuration.
- `Programmed`: indicates whether the Gateway configuration has been programmed.
- `ResolvedRefs`: indicates whether relevant references have been resolved.

Condition availability and meaning must be interpreted in the context of the resource and specification. Route status can also report acceptance by particular parent Gateways and listeners.

When troubleshooting, inspect resource status, conditions, events, controller logs, and the underlying networking infrastructure.

## Resource relationships

- A Gateway references exactly one GatewayClass.
- A GatewayClass identifies the responsible controller.
- A Gateway defines one or more listeners.
- A Route references its intended parent Gateway and must satisfy the relevant attachment rules.
- A Route's backend references identify the destinations for matching traffic.
- A Service commonly provides a stable endpoint for the application backend.

These are API relationships, not a literal sequence of network hops. The controller translates the declared configuration into behavior implemented by the data plane.

---

# Traffic Flow

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