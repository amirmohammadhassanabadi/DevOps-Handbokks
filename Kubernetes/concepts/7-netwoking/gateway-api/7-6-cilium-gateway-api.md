
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