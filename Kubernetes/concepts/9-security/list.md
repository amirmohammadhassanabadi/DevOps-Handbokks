# 8-2. Control Plane Security & Architecture Internals

This document covers the advanced theoretical mechanics, request pipelines, cryptographic trust boundaries, and authorization semantics of the Kubernetes control plane. It maps upstream architectural internals directly to **CIS Kubernetes Benchmark Level 2** requirements, highlighting RKE2-specific behaviors where applicable.

---

## 8-2-1. The API Server Request Pipeline as a Security Boundary

Every request hitting the Kubernetes API server traverses a rigid, non-bypassable sequential pipeline. Understanding this flow is essential for diagnosing access failures and preventing privilege escalation.

$$\text{Transport Layer (TLS)} \longrightarrow \text{Authentication} \longrightarrow \text{Impersonation} \longrightarrow \text{Authorization} \longrightarrow \text{Mutating Admission} \longrightarrow \text{Validating Admission} \longrightarrow \text{Persist to etcd}$$

### 1. Transport Layer & Cryptographic Enforcement
* **TLS Termination:** The API server terminates TLS using `--tls-cert-file` and `--tls-private-key-file`. 
* **Cipher Suites & Min Version:** Enforcing `--tls-min-version=VersionTLS12` (or `VersionTLS13`) prevents downgrade attacks. Cipher suites must be restricted to authenticated encryption with associated data (AEAD) algorithms (e.g., `TLS_AES_128_GCM_SHA256`, `TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256`).
* **Client Certificate Verification:** The `--client-ca-file` validates client certificates presented to the secure port (default 6443). If a certificate is signed by an untrusted CA, the connection is dropped before authentication begins.

### 2. Union vs. Intersection Semantics
* **Authentication (Union):** Multiple authenticators run in order (X.509, Webhook, Token). **The first authenticator that succeeds wins.** If all fail, the request falls back to anonymous authentication (if enabled).
* **Authorization (Intersection/Voting):** Multiple authorizers run concurrently (e.g., `--authorization-mode=Node,RBAC`). **If any authorizer votes to approve the request, it is granted.** Rejection by one authorizer does not block the request unless *all* authorizers reject or none approve.

---

## 8-2-2. Anonymous Authentication and the `system:anonymous` Identity
* **Mechanics:** When a request arrives without bearer tokens, client certificates, or valid auth headers, the API server assigns it the identity `system:anonymous` belonging to the `system:unauthenticated` group (if `--anonymous-auth=true`).
* **CIS 1.2.1 Requirement:** Disable anonymous requests unless explicitly required. When disabled (`--anonymous-auth=false`), unauthenticated TCP connections receive an immediate `401 Unauthorized` response—except for unauthenticated health endpoints (`/healthz`, `/livez`, `/readyz`).
* **Cluster Discovery Bindings:** Even with anonymous auth disabled, certain core discovery mechanisms require the `system:public-info-viewer` cluster role bound to `system:unauthenticated` to let unauthenticated tools read cluster version endpoints (`/version`).

---

## 8-2-3. Authentication Mechanisms & Trust Roots

### 1. X.509 Client Certificates
* The Subject Common Name (`CN`) maps to the username (e.g., `CN = system:node:worker-01`).
* The Subject Organization (`O`) maps to group memberships (e.g., `O = system:nodes` or `O = system:masters`).
* **Hardening Note:** X.509 certificates have no built-in revocation mechanism (OCSP/CRL are not natively checked by the API server). Revocation requires rotating the cluster CA or waiting for certificate expiration.

### 2. ServiceAccount Token Internals
* Modern Kubernetes uses **bound projected service account tokens** leveraging TokenRequest API (`--service-account-issuer`, `--service-account-signing-key-file`).
* Unlike legacy long-lived Secret tokens (which never expire and can be stolen), projected tokens are cryptographically signed JWTs with short, configurable lifetimes, audience claims (`aud`), and optional pod/node binding assertions.

### 3. Webhook Token Authentication & OIDC
* **Webhook Auth (`--authentication-token-webhook-config-file`):** Offloads token verification to an external service via a `TokenReview` API payload. Cache TTL (`--authentication-token-webhook-cache-ttl`) dictates how long negative/positive results are cached.
* **OIDC:** Integrates external identity providers (Keycloak, Azure AD) via `--oidc-issuer-url` and `--oidc-client-id`. The API server locally validates the JWT signature against the OIDC provider's public JSON Web Key Set (JWKS).

---

## 8-2-4. Impersonation Semantics & Audit Trails
* **Mechanics:** Clients with permission to impersonate can pass special HTTP headers:
  * `Impersonate-User`: Target username.
  * `Impersonate-Group`: Target group(s).
  * `Impersonate-Extra-<Key>`: Additional auxiliary claims.
* **Security Risk:** The `impersonate` verb in RBAC is equivalent to `cluster-admin` if granted broadly. 
* **Audit Impact:** When impersonation occurs, audit logs record both the *acting* identity (the controller or user executing the request) and the *impersonated* identity under the `impersonatedUser` field in the audit event.

---

## 8-2-5. Authorization Modes as a Voting System
* **`Node,RBAC` Mode:** 
  * The **Node Authorizer** inspects incoming requests from kubelets. It ensures a kubelet can only read/write objects tied to its specific node (its own Node object, Pods scheduled to it, Secrets/ConfigMaps referenced by those pods).
  * **RBAC** evaluates roles and cluster roles against the authenticated user or service account.

---

## 8-2-6. The `system:masters` Escape Hatch
* **The Bypass:** Any client certificate or token that maps to the `system:masters` group **bypasses RBAC authorization entirely**. The authorizer short-circuits and grants immediate approval.
* **CIS Implication:** Hardening benchmarks strictly audit against any bindings, static tokens, or kubeconfigs that map administrative users or machine accounts into `system:masters`. Break-glass admin access should use strict, scoped cluster-admin RBAC roles rather than `system:masters` group membership where possible.

---

## 8-2-7. Admission Control Internals
Admission plugins intercept requests after authz succeeds but before etcd writes occur.

* **Mutating Webhooks:** Can rewrite incoming object schemas (e.g., injecting Istio sidecars or setting default resource limits). They execute serially; a subsequent mutator sees changes made by the previous one.
* **Validating Webhooks:** Enforce policy boundaries. Can reject objects entirely.
* **Critical CIS Level 2 Built-in Plugins:**
  * `NodeRestriction`: Prevents a compromised kubelet from modifying node objects belonging to other nodes or rescheduling alien pods.
  * `AlwaysPullImages`: Forces kubelets to evaluate image pull credentials and check registries on every container start, mitigating image-cache poisoning attacks in multi-tenant environments.
  * `EventRateLimit`: Throttles event generation per namespace/server to protect etcd from event-flooding Denial of Service attacks.

---

## 8-2-8. Audit Logging Internals
* **Four Stages:** `RequestReceived`, `ResponseStarted`, `ResponseComplete`, `Panic`.
* **Policy Rule Evaluation:** Rules are evaluated **top-down (first match wins)**. If a request matches multiple rules, the first rule's level (`None`, `Metadata`, `Request`, `RequestResponse`) applies.
* **Secret Leakage Risk:** Setting audit levels to `RequestResponse` on resources like `Secrets`, `ConfigMaps`, or `ServiceAccount/token` writes base64-encoded or raw sensitive data directly to audit log files. Audit log files themselves must therefore be protected with strict file permissions (`0600`) and encrypted storage.

---

## 8-2-9. Controller Manager and Scheduler Security Internals
* **Secure Ports:** Controller Manager (10257) and Scheduler (10259) serve metrics and health checks exclusively over TLS with mutual authentication, replacing deprecated insecure ports (10252/10251).
* **`--use-service-account-credentials=true`:** Ensures the Controller Manager creates individual service account tokens for each controller loop rather than executing all controllers under a single overly-permissive service account.

---

## 8-2-10. File Permission, Ownership, and Certificate Cryptographic Boundaries
* **CIS 1.1.x Requirements:** Control plane configuration files, kubeconfigs (`admin.conf`, `controller-manager.conf`, `scheduler.conf`), and private keys (`--etcd-key-file`, `--client-ca-file`) must have ownership set to `root:root` and file permissions restricted to `600` or `644`.
* **RKE2 Delta:** In RKE2, control plane components do not run as standalone static pods with file flags directly exposed in `/etc/kubernetes/manifests/`. Instead, configuration is declared in `/etc/rancher/rke2/config.yaml`, and system files reside inside RKE2-managed directories (`/var/lib/rancher/rke2/`), where file permissions are enforced by the RKE2 packaging scripts.

---

## 8-2-11. RKE2-Specific Control-Plane Posture
* RKE2 embeds API server, scheduler, controller manager, and etcd inside the `rke2 server` binary executed via containerd.
* Component arguments are passed downstream via YAML block flags in `/etc/rancher/rke2/config.yaml`:
```yaml
  kube-apiserver-arg:
- "anonymous-auth=false"
- "enable-admission-plugins=NodeRestriction,AlwaysPullImages,EventRateLimit"
  
