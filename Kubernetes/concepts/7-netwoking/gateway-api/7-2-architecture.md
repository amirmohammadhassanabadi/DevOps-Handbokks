# Architecture and Core Resources

## Overview

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