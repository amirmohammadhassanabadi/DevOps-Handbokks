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

For our Cilium learning path, HTTP and HTTPS listeners are especially important because they are central to HTTP application exposure.

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