# `HTTPRoute` Filters

An `HTTPRoute` does more than match requests and select a backend. It can also define filters that modify how a matching request is handled.

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

## Standard Filter Types

| Filter | Purpose | Gateway API support level |
|---|---|---|
| `RequestHeaderModifier` | Adds, sets, or removes request headers | Core |
| `ResponseHeaderModifier` | Adds, sets, or removes response headers | Extended |
| `RequestRedirect` | 	Returns a redirect response, instructing the client to request another URL. | Core |
| `URLRewrite` | 	Rewrites the request hostname and/or path before forwarding. | Extended |
| `RequestMirror` | Sends a request copy to another backend and ignores the mirrored response | Extended |
| `CORS` | Sends a copy of the request to another destination while the primary request continues normally. | Extended |
| `ExtensionRef` | References an implementation-specific extension | Implementation-specific |

The first five are standardized filter types, although individual options and support can vary. `ExtensionRef` is explicitly for implementation-specific behavior.

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

- ### CORS and ExtensionRef

  Gateway API also defines a `CORS` filter for Cross-Origin Resource Sharing. `CORS` controls which browser origins may access responses; it is not an authentication or authorization mechanism.
  
  `ExtensionRef` allows a Route to refer to an implementation-specific extension. That can provide functionality beyond the standard API, but it reduces portability between Gateway API implementations.
  
  Gateway API groups features into support levels:
  
  - **Core**: implementations conforming to the relevant API must support the feature.
  - **Extended**: implementations are encouraged to support it, but support is not universally required.
  - **Implementation-specific**: behavior depends on the implementation.
  
  These levels describe the API's conformance expectations, not a guarantee that every filter works identically in every controller version.

---

# `BackendRefs`, `ReferenceGrant`, and Traffic Splitting

An `HTTPRoute` determines how a request is matched and where it should be sent. The `backendRefs` field identifies the backend destinations that receive matching traffic.

Keep three separate concepts in mind:

- **Backend selection:** Which Service should receive the request?
- **Reference permissions:** Is the Route allowed to reference that backend?
- **Traffic distribution:** If there are multiple backends, how should requests be distributed between them?

## How `backendRefs` works

A backend reference usually points to a Kubernetes Service.

```yaml
backendRefs:
  - name: orders-service
    port: 8080
```

This tells the controller to send matching traffic to port `8080` of the Service named `orders-service`.

If namespace is omitted, the backend is assumed to be in the same namespace as the `HTTPRoute`. You can specify namespace to reference a Service in another namespace, but that introduces additional permission requirements.

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

## Cross-namespace backends and `ReferenceGrant`

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

> **The owner of the target namespace must explicitly permit the cross-namespace reference using a `ReferenceGrant` resource in the target namespace**

## How `ReferenceGrant` works

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

This permits `HTTPRoute` resources in `edge` to reference **Services** in `applications` namespace. It does not grant access to every kind of resource, and it does not allow Routes from every namespace.

Notice that the grant does not specify a Service name. Its to section grants permission by resource kind, so the grant applies to Services in the target namespace, subject to the specified source restrictions.

**Security principle:** the owner of the referenced resource controls whether cross-namespace references to that resource type are allowed. A Route author cannot grant themselves this permission simply by editing their own Route.

### What happens if the `ReferenceGrant` is missing?

The cross-namespace backend reference is invalid. The controller should report the unresolved reference in the Route's status, typically through the `ResolvedRefs` condition with reason `RefNotPermitted`. Traffic to that backend cannot be forwarded as intended. The precise observable failure depends on the implementation and configuration.

Do not confuse two different cross-namespace relationships:

| Relationship | Main permission mechanism |
| --- | --- |
| `HTTPRoute` attaching to a Gateway in another namespace | Gateway listener's `allowedRoutes` |
| `HTTPRoute` referencing a Service in another namespace | `ReferenceGrant` in the Service's namespace |
| Gateway referencing a TLS Secret in another namespace | `ReferenceGrant` in the Secret's namespace |

These mechanisms protect different boundaries.

## Traffic splitting with backend weights

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
- **Backend readiness and health** — whether the actual application endpoints can serve requests successfully.
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