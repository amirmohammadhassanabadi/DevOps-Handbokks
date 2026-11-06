# Gateway Listeners and TLS Configuration

Now we need to understand how the Gateway accepts connections in the first place, especially HTTPS.

The important distinction is:
- **Listener**: Defines how the Gateway accepts connections.
- **TLS configuration**: Defines how TLS is configured for a listener.
- **HTTPRoute**: Defines how decrypted HTTP requests are routed to backends.

## What is a Gateway listener?

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

> ### TCPRoute
> 
> `TCPRoute` routes TCP connections to backend Services.
> 
> Unlike **`TLSRoute`**, it does not inspect TLS SNI or other application-level information. It simply forwards `TCP` connections received on a listener to the configured backend.
> 
> For example, imagine you have an application that listens on `TCP` port `9000`.
> 
> You want clients to connect to the Gateway on port `9000`, and the Gateway should forward those connections to the application Service.
>
> **Example**
>
>```yaml
> apiVersion: gateway.networking.k8s.io/v1
> kind: Gateway
> metadata:
>   name: tcp-gateway
>   namespace: edge
> spec:
>   gatewayClassName: cilium
>   listeners:
>     - name: tcp-app
>       protocol: TCP
>       port: 9000
>       allowedRoutes:
>         kinds:
>           - kind: TCPRoute
>         namespaces:
>           from: Same
> ---
> apiVersion: gateway.networking.k8s.io/v1alpha2
> kind: TCPRoute
> metadata:
>   name: tcp-app-route
>   namespace: edge
> spec:
>   parentRefs:
>     - name: tcp-gateway
>       sectionName: tcp-app
>   rules:
>     - backendRefs:
>         - name: tcp-application
>           port: 9000
> ```
> 
> **Traffic flow:**
>
> ```
> ┌─────────────────────────────────────────────────────────────────┐
> |┌───────────────────────────────────────────────────────────────┐|
> ||                          TCP Client                           ||
> ||                   Connects to Gateway :9000                   ||
> |└───────────────────────────────────────────────────────────────┘|
> |                                ↓                                |
> |┌───────────────────────────────────────────────────────────────┐|
> ||                          tcp-gateway                          ||
> ||                       TCP listener :9000                      ||
> |└───────────────────────────────────────────────────────────────┘|
> |                                ↓                                |
> |┌───────────────────────────────────────────────────────────────┐|
> ||                        tcp-application                        ||
> ||                         Service :9000                         ||
> |└───────────────────────────────────────────────────────────────┘|
> └─────────────────────────────────────────────────────────────────┘
> ```
> 
> **Important TCPRoute limitation**
>
> `TCPRoute` has no hostname, path, or header matching.
> 
> Suppose you have two TCP applications:
> - Application `A` listens on Service port `9000`.
> - Application `B` listens on Service port `9001`.
>
> You generally need separate Gateway listeners with separate ports and a corresponding Route for each application.
> 
> You cannot use `TCPRoute` to inspect the application protocol and decide which backend should receive a connection on the same listener.
> 
> That is one of the reasons `TLSRoute` exists: it can distinguish TLS connections using **SNI**.
>
> ---
>
> ### UDPRoute
>
> `UDPRoute` routes UDP traffic from a Gateway listener to backend Services.
> 
> **UDP** is connectionless, unlike TCP. There is no TCP three-way handshake or persistent TCP connection to forward.
> 
> > A common example is **DNS traffic**.
> 
> Imagine you have a DNS server running inside Kubernetes and want to forward UDP port `5300` from the Gateway to that DNS Service.
>
> **Example**
>
> ```yaml
> apiVersion: gateway.networking.k8s.io/v1
> kind: Gateway
> metadata:
>   name: udp-gateway
>   namespace: edge
> spec:
>   gatewayClassName: cilium
>   listeners:
>     - name: dns
>       protocol: UDP
>       port: 5300
>       allowedRoutes:
>         kinds:
>           - kind: UDPRoute
>         namespaces:
>           from: Same
> ---
> apiVersion: gateway.networking.k8s.io/v1alpha2
> kind: UDPRoute
> metadata:
>   name: dns-route
>   namespace: edge
> spec:
>   parentRefs:
>     - name: udp-gateway
>       sectionName: dns
>   rules:
>     - backendRefs:
>         - name: dns-server
>           port: 53
> ```
> 
> **Traffic flow:**
>
> ```
> ┌─────────────────────────────────────────────────────────────────┐
> |┌───────────────────────────────────────────────────────────────┐|
> ||                          DNS Client                           ||
> ||                  UDP query to Gateway :5300                   ||
> |└───────────────────────────────────────────────────────────────┘|
> |                                ↓                                |
> |┌───────────────────────────────────────────────────────────────┐|
> ||                          udp-gateway                          ||
> ||                       UDP listener :5300                      ||
> |└───────────────────────────────────────────────────────────────┘|
> |                                ↓                                |
> |┌───────────────────────────────────────────────────────────────┐|
> ||                          dns-server                           ||
> ||                        Service :53/UDP                        ||
> |└───────────────────────────────────────────────────────────────┘|
> └─────────────────────────────────────────────────────────────────┘
> ```
> 
> Notice that the Gateway listener port and backend Service port are different:
> - Gateway receives traffic on UDP `5300`.
> - Backend Service receives traffic on UDP `53`.
>
> The Route connects these two ports.
> 
> The Service must expose UDP port 53, for example:
>
> ```yaml
> ports:
>   - name: dns
>     protocol: UDP
>     port: 53
>     targetPort: 53
> ```
>
> ---
>
> ### Cilium-specific compatibility
>
> There is an important implementation detail here.
> 
> According to the Cilium 1.20 documentation:
>
> | Resource | Cilium support | Additional requirement |
> | --- | --- | --- |
> | TLSRoute | Supported | Gateway API TLSRoute CRD |
> | TCPRoute | Supported | Optional TCPRoute CRD |
> | UDPRoute | Supported | Optional UDPRoute CRD |
> 
> **`Cilium 1.20`** supports all three, but `TCPRoute` and `UDPRoute` require their optional CRDs to be installed. 
> 
> You can check whether the CRDs exist:
>
> ```bash
> kubectl get crd tlsroutes.gateway.networking.k8s.io
> kubectl get crd tcproutes.gateway.networking.k8s.io
> kubectl get crd udproutes.gateway.networking.k8s.io
> ```
> 
> > One limitation to remember: **Cilium Gateway API host network mode is not compatible with `TCPRoute` and `UDPRoute`.** If you use those two Route types, you need to account for their generated Service exposure instead. 
>


## TLS termination: what actually happens?

When a client connects to an HTTPS listener configured with `tls.mode: Terminate`, the Gateway terminates the client's TLS connection.

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

In this example, traffic between the Gateway and backend is **HTTP**. That does not mean Gateway API always uses HTTP for upstream connections; backend TLS can be configured separately.

TLS termination at the Gateway means the original client-to-Gateway TLS session ends at the Gateway. It does not automatically encrypt the separate Gateway-to-backend connection.

**Where does the certificate come from?**

Gateway API commonly uses a Kubernetes `Secret` to provide the TLS certificate and private key.

A typical TLS `Secret` looks like this:

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

By default, the `Secret` is expected in the same namespace as the Gateway. Cross-namespace references require the appropriate `ReferenceGrant` in the namespace containing the Secret.

Gateway API does not issue certificates. It only defines how a Gateway references and uses certificate material. The Gateway controller consumes the referenced certificate material.

## SNI: how does the Gateway know which certificate to present?

Imagine one Gateway has two HTTPS hostnames:

- `shop.example.com`
- `api.example.com`

Both use port `443`, but each requires a different certificate.

> How can the Gateway know which certificate the client expects before it has received the HTTP request?

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

This is particularly important because the HTTP Host header (or HTTP/2 `:authority`) is sent as part of the encrypted application traffic, after TLS has been established. The Gateway cannot rely on that encrypted HTTP information to select the certificate for the initial handshake.

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

### Listener hostname, SNI, and HTTP Host are different things

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

  > ### TLSRoute
  > `TLSRoute` routes TLS connections based on the Server Name Indication (SNI) hostname sent by the client during the TLS handshake.
  >
  > It is useful when you want to route encrypted traffic to different backends without inspecting HTTP-level information.
  > 
  > For example, imagine you have two TLS-enabled services:
  > - `postgres-db` — PostgreSQL over TLS
  > - `mqtt-broker` — MQTT over TLS
  >
  > You want both services to receive traffic through the same Gateway listener port, `443`, but route traffic based on the requested **SNI** hostname.
  > 
  > There are two TLS modes:
  > - **`Passthrough`**: Gateway does not terminate TLS. It forwards the encrypted connection to the backend.
  > - **`Terminate`**: Gateway terminates TLS and forwards the resulting unencrypted TCP stream to the backend.
  >
  > **Example: TLS passthrough**
  >
  > ```yaml
  > apiVersion: gateway.networking.k8s.io/v1
  > kind: Gateway
  > metadata:
  >   name: tls-gateway
  >   namespace: edge
  > spec:
  >   gatewayClassName: cilium
  >   listeners:
  >     - name: tls
  >       hostname: "*.example.com"
  >       protocol: TLS
  >       port: 443
  >       tls:
  >         mode: Passthrough
  >       allowedRoutes:
  >         kinds:
  >           - kind: TLSRoute
  >         namespaces:
  >           from: Same
  > ---
  > apiVersion: gateway.networking.k8s.io/v1
  > kind: TLSRoute
  > metadata:
  >   name: postgres-route
  >   namespace: edge
  > spec:
  >   parentRefs:
  >     - name: tls-gateway
  >       sectionName: tls
  >   hostnames:
  >     - postgres.example.com
  >   rules:
  >     - backendRefs:
  >         - name: postgres-db
  >           port: 5432
  > ---
  > apiVersion: gateway.networking.k8s.io/v1
  > kind: TLSRoute
  > metadata:
  >   name: mqtt-route
  >   namespace: edge
  > spec:
  >   parentRefs:
  >     - name: tls-gateway
  >       sectionName: tls
  >   hostnames:
  >     - mqtt.example.com
  >   rules:
  >     - backendRefs:
  >         - name: mqtt-broker
  >           port: 8883
  > ```
  > 
  > Both Routes attach to the same listener, but their hostnames are different.
  >
  > **Traffic flow:**
  > ```
  > ┌─────────────────────────────────────────────────────────────────┐
  > |                              Client                             |
  > |                    TLS connection to port 443                   |
  > |                                ↓                                |
  > |┌───────────────────────────────────────────────────────────────┐|
  > ||                           tls-gateway                         ||
  > ||                TLS listener :443 — Passthrough                ||
  > ||                           Reads SNI                           ||
  > |└───────────────────────────────────────────────────────────────┘|
  > |┌──────────────────────────────┐┌───────────────────────────────┐|
  > ||    postgres.example.com      ||       mqtt.example.com        ||
  > ||              ↓               ||              ↓                ||
  > ||         postgres-db          ||          mqtt-broker          ||
  > ||        Service :5432         ||         Service :8883         ||
  > ||   Backend terminates TLS     ||    Backend terminates TLS     ||
  > |└──────────────────────────────┘└───────────────────────────────┘|
  > |The encrypted TLS stream remains encrypted between the client and| 
  > |backend.                                                         |
  > └─────────────────────────────────────────────────────────────────┘
  > ``` 
  > 
  > 
  > 
  > The Gateway can inspect the SNI in the TLS **`ClientHello`**, but it cannot inspect the encrypted application data or HTTP paths in passthrough mode.
  > 
  > **Important:** SNI is a TLS hostname, not an HTTP Host header. `TLSRoute` does not provide HTTP path-based routing.
  >
  > ---

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

## `BackendTLSPolicy`

`BackendTLSPolicy` is a Gateway API resource that configures TLS encryption and certificate verification between a Gateway implementation and its backend Kubernetes Service.

The key idea is that TLS can be used on both sides of a Gateway:
1. `Client → Gateway`: TLS protects the connection from the client to the Gateway. This is configured through the Gateway listener.
2. `Gateway → Backend`: TLS protects the connection from the Gateway to the backend Service. This is where `BackendTLSPolicy` comes in.

These are two separate TLS connections. Configuring HTTPS on your Gateway listener does not automatically mean that traffic from the Gateway to the backend is encrypted.

### Why do we need `BackendTLSPolicy`?

Consider this architecture:


```
┌─────────────────────────────────────────────────────────────────┐
|                              Client                             |
|                           HTTPS request                         |
|                                ↓                                |
|                          TLS connection 1                       |
|┌───────────────────────────────────────────────────────────────┐|
||                         Gateway / Envoy                       ||
||                      Terminates client TLS                    ||
|└───────────────────────────────────────────────────────────────┘|
|                               ↓                                 |
|              Backend TLS not automatically enabled              |
|┌───────────────────────────────────────────────────────────────┐|
||                         Backend Service                       ||
||                           Application                         ||
|└───────────────────────────────────────────────────────────────┘|
└─────────────────────────────────────────────────────────────────┘
```

Without backend TLS configuration, the Gateway may terminate HTTPS and forward the request to the backend using plain HTTP.

**For example:**

```
Client (HTTPS) → Gateway (HTTP) → Backend
```

This is called **TLS termination at the Gateway**.

It is a perfectly valid architecture in some environments, but it means the Gateway-to-backend connection is not encrypted.

If you want encryption on that connection too, you can configure:

```
Client (HTTPS) → Gateway (HTTPS) → Backend
```

`BackendTLSPolicy` tells the Gateway implementation how to establish and verify that backend TLS connection.

### What does `BackendTLSPolicy` configure?

Its main responsibilities are:

- **TLS hostname**: The hostname the Gateway uses for the backend TLS connection and certificate verification.
- **Certificate trust**: Which CA certificates the Gateway trusts when verifying the backend's certificate.
- **Backend selection**: Which Kubernetes Service the policy applies to.

**An important distinction:**

`BackendTLSPolicy` does not configure the frontend listener's TLS certificate. That belongs to `Gateway.spec.listeners[].tls.certificateRefs`.

It also does not issue certificates or renew them. You need an existing backend certificate and an appropriate trust configuration.

### A practical example

Let's extend our previous shop example.

Assume:

- `Gateway`: production-gateway in namespace `edge`.
- `HTTPRoute`: `shop-route` in namespace `api`.
- Backend Service: shop-api in namespace `api`.
- The backend serves HTTPS on Service port `443`.
- Its certificate is issued by an internal Certificate Authority (CA).
- The certificate contains `shop-api.internal.example.com` as its DNS identity.

The connection should look like this:

```
Client
  |
  | HTTPS
  v
Gateway / Envoy
  |
  | HTTPS
  | SNI: shop-api.internal.example.com
  | Validates backend certificate using internal CA
  v
shop-api Service :443
  |
  v
Backend Pods
```

### Step-By-Step Implementarion:

- **Step 1 — Prepare the CA certificate**

  The Gateway must trust the CA that issued the backend certificate.

  Create a `ConfigMap` containing the CA certificate:

  ```bash
  kubectl create configmap shop-api-ca \
    --from-file=ca.crt=internal-ca.crt \
    -n api
  ```

  The `ConfigMap` contains the CA's public certificate, not the backend's private key.

- **Step 2 — Create `BackendTLSPolicy`**

  ```yaml
  apiVersion: gateway.networking.k8s.io/v1
  kind: BackendTLSPolicy
  metadata:
    name: shop-api-backend-tls
    namespace: api
  spec:
    targetRefs:
      - group: ""
        kind: Service
        name: shop-api

    validation:
      hostname: shop-api.internal.example.com
      caCertificateRefs:
        - group: ""
          kind: ConfigMap
          name: shop-api-ca
  ```

  This policy targets the `shop-api` Service. It tells the Gateway implementation to use TLS when connecting to that backend and validate the backend's certificate.

- **Step 3 — Update the `HTTPRoute` backend port**

  Our earlier `HTTPRoute` used Service port `8080`. For this example, the backend Service now exposes **HTTPS** on port `443`, so update the API rule:

  ```yaml
  backendRefs:
    - name: shop-api
      port: 443
      weight: 1
  ```

  **Remember:** the Route references the Service port, not the Pod's targetPort.

  The Service might look like this:

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
      - name: https
        protocol: TCP
        port: 443
        targetPort: 8443
  ```

  The application Pods must actually serve HTTPS on port `8443`.

### Understand the important fields

- **`targetRefs`**

  ```yaml
  targetRefs:
    - group: ""
      kind: Service
      name: shop-api
  ```
  
  This identifies the Service to which the policy applies.
  - `group: ""` → means the core Kubernetes API group.
  - `kind: Service` → identifies the target resource type.
  - `name: shop-api` → identifies the target Service.
  
  **Important:** The `BackendTLSPolicy` must be in the **same namespace as the Service it targets**. In our example, both are in `api`.

  > **The policy is attached to the Service, not directly to the `HTTPRoute`.** This means multiple Routes that reference the same Service can use the same backend TLS configuration. 

- **`validation.hostname`**

  This field has two important purposes:

  1. It specifies the SNI hostname the Gateway sends when establishing the TLS connection to the backend.
  2. It is used to verify the backend's certificate identity.

  For example, the backend certificate must be valid for `shop-api.internal.example.com`.

  This hostname does not have to be the same as the public hostname used by clients:

  - **`Client → Gateway`**: `shop.example.com`
  - **`Gateway → Backend`**: `shop-api.internal.example.com`

  This distinction is important in real environments because internal backend services often have their own DNS names and certificates.

- **`caCertificateRefs`**

  ```yaml
  caCertificateRefs:
    - group: ""
      kind: ConfigMap
      name: shop-api-ca
  ```

  This tells the Gateway which CA certificate(s) to trust when verifying the backend's certificate.

  The ConfigMap contains PEM-encoded CA certificate data.

  The validation process is conceptually:
  1. Gateway connects to the backend using TLS.
  2. Backend presents its certificate.
  3. Gateway verifies the certificate chain against the configured CA.
  4. Gateway verifies the certificate identity against the configured hostname.
  5. If validation succeeds, the TLS connection can be established.

  If certificate verification fails, the Gateway must not treat that backend TLS connection as successfully established.

### System CA certificates versus custom CA certificates

There are two alternatives for configuring certificate trust.

- **Option A — Custom CA certificates**

  ```yaml
  validation:
    hostname: shop-api.internal.example.com
    caCertificateRefs:
      - group: ""
        kind: ConfigMap
        name: shop-api-ca
  ```

  Use this when your backend certificates are issued by an **internal CA** or a CA that is **not in the implementation's system trust store**.

- **Option B — System CA certificates**

  ```yaml
  validation:
    hostname: shop-api.example.com
    wellKnownCACertificates: System
  ```

  This tells the Gateway implementation to use its **system trust store**.

  Use it when the backend certificate is issued by a **publicly trusted CA** and the implementation supports this option.

  You must specify one of these trust mechanisms, not both. 

### Does BackendTLSPolicy provide mutual TLS (mTLS)?

**No, not by itself.**

The configuration we've used provides:
- Encryption from Gateway to backend.
- Verification of the backend's certificate by the Gateway.

This is ordinary TLS with server authentication.

In mutual TLS, the backend also verifies a client certificate presented by the Gateway.

**Ordinary backend TLS:**

```
Gateway  ---- TLS ---->  Backend
          verifies
          backend cert
```

**Mutual TLS:**

```
Gateway  <--- TLS ---->  Backend
          verifies       verifies
          backend cert   gateway cert
```

`BackendTLSPolicy` has an Extended feature called `subjectAltNames` that can provide more precise backend certificate identity matching, including URI-based identities such as SPIFFE IDs. It does not, by itself, configure the Gateway's client certificate for mTLS. 

### Important operational considerations

A few things are easy to misunderstand:

| Point | Explanation |
| --- | --- |
| Does it encrypt Gateway-to-backend traffic? | Yes, when supported and successfully configured. |
| Does it issue backend certificates? | No. |
| Does it configure the Gateway's frontend certificate? | No. |
| Does it configure the backend application to serve HTTPS? | No. The backend must already support TLS. |
| Does it configure backend certificate trust? | Yes. |
| Does it automatically enable mTLS? | No. |
| Does it apply to a Route or a Service? | It targets a Service. |
| Can it reference a CA ConfigMap in another namespace? | No. |

Also, don't confuse `BackendTLSPolicy` with `ReferenceGrant`. A `ReferenceGrant` is not needed for the CA `ConfigMap` in our example because the `ConfigMap` and policy are in the same namespace. Cross-namespace CA references are not permitted by this policy specification. 

### Cilium implementation

Cilium supports `BackendTLSPolicy` with its Gateway API implementation. Its documentation demonstrates the same workflow:

1. Configure a backend application to serve TLS.
2. Expose the backend's HTTPS port through a Kubernetes Service.
3. Create a CA ConfigMap.
4. Create a `BackendTLSPolicy` targeting the Service.
5. Update the HTTPRoute backend port to the Service's HTTPS port.

After applying the policy, inspect its status:

```bash
kubectl get backendtlspolicy -n api
kubectl describe backendtlspolicy shop-api-backend-tls -n api
kubectl get backendtlspolicy shop-api-backend-tls -n api -o yaml
```

Check its `status.ancestors[].conditions` to see whether the implementation accepted and successfully programmed the policy. Also test actual requests: an accepted policy alone does not prove the backend TLS handshake succeeds.

### The most important takeaway

Remember these three different responsibilities:

| Resource / field | Responsibility |
| --- | --- |
| `Gateway.spec.listeners[].tls` | TLS between client and Gateway |
| `HTTPRoute.spec.rules[].backendRefs` | Selects the backend Service and its Service port |
| `BackendTLSPolicy` | Configures TLS and backend certificate validation between Gateway and backend |

The complete architecture can therefore look like this:

```
Client
  |
  | HTTPS
  | Certificate: shop.example.com
  |
  v
Gateway / Envoy
  |
  | HTTPRoute selects shop-api:443
  |
  | HTTPS
  | SNI: shop-api.internal.example.com
  | Certificate verified using internal CA
  |
  v
Backend Service
  |
  v
Backend Pod
```

---