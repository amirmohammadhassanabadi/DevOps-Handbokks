# Probes

Kubernetes probes are health checks configured for containers in a Pod. The **kubelet** performs these checks and uses their results to determine whether a container is healthy and whether its Pod should receive traffic.

Probes are configured under the individual container:

```yaml
spec:
  containers:
    - name: app
      image: myapp:1.0
      livenessProbe:
        ...
      readinessProbe:
        ...
      startupProbe:
        ...
```

There are three types of probes:

* **Liveness Probe** → Is the application still functioning?
* **Readiness Probe** → Is the application ready to receive traffic?
* **Startup Probe** → Has the application successfully completed its startup phase?

---

# Liveness Probe

A **liveness probe** determines whether the application is still functioning.

If the liveness probe repeatedly fails, kubelet considers the container unhealthy and restarts the container according to the Pod's restart policy.

A liveness probe is useful when an application can become stuck or deadlocked while its process is still running.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

This configuration starts checking the application after the initial delay and then performs the HTTP health check periodically.

If the configured failure threshold is reached, **kubelet restarts the container**.

A liveness probe should normally check whether the application itself is functioning, rather than whether every external dependency is available.

---

# Readiness Probe

A **readiness probe** determines whether a container is ready to receive traffic.

If the readiness probe fails:

* The Pod is marked **NotReady**.
* The Pod is removed from the set of normal Service endpoints.
* New traffic should no longer be directed to the Pod through the Service.
* The container is **not restarted** merely because the readiness probe failed.

When the readiness probe succeeds again, the Pod can become Ready and receive traffic again.

Example:

```yaml
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 3
```

This checks whether a TCP connection can be established on port `8080`.

A readiness probe is useful when an application needs time to become ready, for example when it needs to:

* Load application data
* Initialize internal state
* Establish required connections
* Warm up before serving requests

Readiness is therefore primarily about **traffic eligibility**, not container health.

---

# Startup Probe

A **startup probe** is designed for applications that require a long time to start.

When a startup probe is configured, Kubernetes uses it to determine whether the application has completed its startup phase. Liveness and readiness probing are held back until the startup probe succeeds.

Example:

```yaml
startupProbe:
  httpGet:
    path: /start
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

The configured failure threshold and period allow the application time to start before Kubernetes considers the startup check failed.

Startup probes are particularly useful for applications such as large Java applications that may require significant initialization time.

Without a startup probe, an aggressive liveness probe could restart a slow-starting application before it has finished initializing.

---

# Common Probe Parameters

Several parameters control probe behavior:

| Parameter             | Purpose                                                |
| --------------------- | ------------------------------------------------------ |
| `initialDelaySeconds` | Delay before the first probe                           |
| `periodSeconds`       | Time between probes                                    |
| `timeoutSeconds`      | Maximum time allowed for a probe                       |
| `successThreshold`    | Consecutive successes required for a successful result |
| `failureThreshold`    | Consecutive failures required for a failed result      |

Example:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

---

# Probe Mechanisms

Probes can use several mechanisms:

## HTTP GET

Sends an HTTP request to the specified path and port.

```yaml
httpGet:
  path: /healthz
  port: 8080
```

The probe succeeds when the HTTP response is considered successful.

---

## TCP Socket

Attempts to establish a TCP connection to the specified port.

```yaml
tcpSocket:
  port: 8080
```

The probe succeeds when the TCP connection can be established.

---

## Exec

Executes a command inside the container.

```yaml
exec:
  command:
    - cat
    - /tmp/healthy
```

The probe succeeds when the command exits with status code `0`.

---

## gRPC

Kubernetes can also perform a gRPC health check for applications that implement the gRPC health checking protocol.

Example:

```yaml
grpc:
  port: 50051
```

This is useful for gRPC-based applications.

---

# Probe Results and Container Behavior

The effect of a failed probe depends on the probe type.

| Probe         | Purpose                                       | When it fails                                                     |
| ------------- | --------------------------------------------- | ----------------------------------------------------------------- |
| **Liveness**  | Detect an unhealthy container                 | Container is restarted                                            |
| **Readiness** | Determine whether the Pod can receive traffic | Pod becomes NotReady and is removed from normal Service endpoints |
| **Startup**   | Determine whether startup has completed       | Container is eventually restarted if startup repeatedly fails     |

A readiness failure **does not restart the container**.

A liveness or startup probe failure can result in the container being restarted.

---

# Probes During Rolling Updates

Readiness probes are particularly important during Deployment rolling updates.

Suppose a Deployment is updated:

```text
Old version
    ↓
Deployment creates new Pod
    ↓
New Pod starts
    ↓
Readiness probe
    ↓
Not Ready
    ↓
No normal Service traffic
    ↓
Readiness succeeds
    ↓
Pod becomes Ready
    ↓
Pod becomes eligible for Service traffic
```

Only after the new Pod becomes available does the Deployment continue the rollout according to its update strategy and availability constraints.

Readiness therefore helps prevent a newly started application from receiving traffic before it is actually ready.

---

# Existing Connections During Pod Termination

Consider a rolling update where an old Pod is being replaced while it has active client connections.

Kubernetes does **not** track individual HTTP requests, TCP connections, or user sessions.

It manages whether the Pod is eligible to receive new traffic through its Service endpoints.

When a Pod is terminating, Kubernetes updates its endpoint state so that normal traffic is no longer directed to that Pod. Existing connections, however, are handled by the networking stack and the application.

For example:

```text
Before termination:

Client A ────────→ Pod A
Client B ────────→ Pod A
Client C ────────→ Pod A

Pod A begins termination

New connections
        │
        └────────→ Ready Pods

Existing connections
        │
        └────────→ may continue to Pod A
```

Existing connections can continue while the Pod remains alive, but Kubernetes does not guarantee that they will survive until completion. Once the container is terminated, active connections can be interrupted.

This is particularly important for long-lived connections such as:

* WebSockets
* gRPC streams
* Server-Sent Events (SSE)
* Long-running HTTP requests

Therefore, applications that require graceful connection draining should implement graceful shutdown behavior.

---

# Graceful Pod Termination

When a Pod is deleted or replaced, Kubernetes gives its containers an opportunity to terminate gracefully.

The general process is:

```text
Pod termination requested
        ↓
Pod enters Terminating state
        ↓
Termination grace period begins
        ↓
Pod endpoint state is updated
        ↓
preStop hook runs, if configured
        ↓
SIGTERM sent to container process
        ↓
Application performs graceful shutdown
        ↓
Application exits
        ↓
Pod terminates
```

The exact endpoint update and termination events can occur concurrently, so applications should not depend on a strict ordering between endpoint removal and the `preStop` hook.

The default `terminationGracePeriodSeconds` is **30 seconds**.

Example:

```yaml
spec:
  terminationGracePeriodSeconds: 60
```

This gives the application up to approximately 60 seconds to terminate gracefully.

If the process is still running when the grace period expires, kubelet eventually terminates it forcefully using `SIGKILL`.

Any unfinished requests or connections can therefore be interrupted.

---

# Graceful Shutdown

Applications should handle `SIGTERM` and perform graceful shutdown.

A well-designed application should:

1. Stop accepting new work.
2. Allow in-flight requests to finish.
3. Close connections and resources.
4. Complete required cleanup.
5. Exit before the termination grace period expires.

For example:

```text
SIGTERM
   ↓
Stop accepting new requests
   ↓
Finish active requests
   ↓
Close database connections
   ↓
Close other resources
   ↓
Exit
```

Kubernetes does not wait indefinitely for active requests to finish.

The application must complete its shutdown within the configured termination grace period.

---

# preStop Hook

A **`preStop` hook** allows Kubernetes to execute an action inside the container immediately before the container is terminated.

Example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment

spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30

      containers:
        - name: app
          image: nginx:1.26

          lifecycle:
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - sleep 10
```

The `preStop` hook can be used to perform preparation or cleanup before the application terminates.

### Important

The termination grace-period countdown begins **before** the `preStop` hook runs.

Therefore:

```text
terminationGracePeriodSeconds = 30

```text
Pod deletion requested
        │
        ▼
Pod enters Terminating
        │
        ├── Endpoint is updated
        │   → Pod is removed from normal Service traffic
        │
        ▼
Termination grace period starts
        │
        ▼
     preStop
     hook runs
        │
        ▼
     SIGTERM
        │
        ▼
Application performs
graceful shutdown
        │
        ├── Stop accepting new work
        ├── Finish active requests
        ├── Close connections
        └── Release resources
        │
        ▼
Container exits
        │
        ▼
Pod is removed
```

The important point is that **the `preStop` hook consumes part of the same 30-second grace period**. It does not provide an additional 10 seconds. A `preStop` hook should therefore be short and should not consume the entire termination grace period.

If the container is still running when the grace period expires, kubelet can forcefully terminate it with `SIGKILL`.

Therefore, the application should be designed to complete its shutdown **within the available grace period**.

---

# Connection Draining

For applications with long-lived connections or long-running requests, graceful termination may require coordination between Kubernetes, the application, and the traffic-routing layer.

A typical approach is:

```text
Pod becomes terminating
        ↓
Stop sending new traffic
        ↓
Application stops accepting new work
        ↓
Existing requests/connections drain
        ↓
Application exits
```

Ingress controllers and external load balancers may also provide connection-draining behavior.

Examples include:

* NGINX Ingress
* HAProxy
* Cloud load balancers

The exact draining behavior depends on the traffic-routing component and its configuration.

Kubernetes itself does not track individual application requests and cannot guarantee that every existing connection will finish before a Pod is terminated.

Connection draining does not mean moving existing connections from one Pod to another.

When a Pod starts terminating, the traffic-routing layer can stop sending new connections or requests to that Pod while allowing existing connections to continue.

The existing connections remain connected to Pod A because an established TCP connection cannot simply be transferred to another Pod. If those connections finish before the Pod terminates, they can complete normally. If the termination grace period expires first, the remaining connections may be interrupted.

---

# Session Affinity

**Session affinity**, also called sticky sessions, is a Service feature that attempts to send traffic from the same client to the same Pod.

By default:

```yaml
spec:
  sessionAffinity: None
```

A client can therefore have different requests handled by different Pods:

```text
Client
  │
  ├── Request 1 → Pod A
  ├── Request 2 → Pod B
  ├── Request 3 → Pod C
  └── Request 4 → Pod A
```

Session affinity can be enabled with:

```yaml
spec:
  sessionAffinity: ClientIP
```

This uses the client IP address as the basis for affinity.

The timeout can be configured:

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

`10800` seconds equals 3 hours.

Session affinity is useful for applications that keep session state locally inside a Pod.

For example:

```text
Pod A
└── User session

Pod B
└── No session
```

If subsequent requests from the same client are sent to Pod B, the application may not find the session.

### Limitations

Client-IP affinity has limitations:

* Multiple users may share the same public IP through NAT.
* A reverse proxy or load balancer may affect the source IP seen by the Service.
* Mobile clients may change IP addresses.
* If the selected Pod becomes unavailable, traffic must be sent to another available Pod.

Session affinity therefore does not guarantee that a user's session will always remain on the same Pod.

For many cloud-native applications, it is preferable to keep application state outside the Pod, for example in Redis or a database, or use stateless authentication such as JWTs.

---

# Probes and Graceful Termination Together

These features solve different problems:

```text
Readiness Probe
      ↓
"Should this Pod receive traffic?"

Liveness Probe
      ↓
"Is this container still functioning?"

Startup Probe
      ↓
"Has this application finished starting?"

preStop + SIGTERM
      ↓
"How should this container shut down?"

terminationGracePeriodSeconds
      ↓
"How long does the application have to terminate gracefully?"
```

Together, they allow Kubernetes to handle both application health and controlled Pod replacement during operations such as rolling updates.

---

# Summary

* **Liveness probe** detects containers that are unhealthy and can cause them to be restarted.
* **Readiness probe** determines whether a Pod is eligible to receive normal Service traffic.
* **Startup probe** protects slow-starting applications from premature liveness/readiness checks.
* Probes can use `httpGet`, `tcpSocket`, `exec`, or `grpc`.
* A failed readiness probe does **not** restart the container.
* Kubernetes does not track individual HTTP requests, sessions, or TCP connections.
* Removing a Pod from normal Service endpoints prevents new traffic from being directed to it, but existing connections may continue while the Pod remains alive.
* Applications should implement graceful shutdown for long-running requests and connections.
* `terminationGracePeriodSeconds` controls the termination grace period.
* `preStop` runs before the container receives the normal termination signal, but its execution time is part of the termination grace period.
* If the application does not terminate before the grace period expires, kubelet can forcefully terminate it.
* Session affinity uses `ClientIP` to attempt to keep a client's traffic on the same Pod, but it does not provide guaranteed session persistence.
