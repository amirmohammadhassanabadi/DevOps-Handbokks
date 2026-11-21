## Introduction: Kubernetes Control Plane Security

The Kubernetes control plane is responsible for managing the cluster's desired state, making scheduling decisions, coordinating workloads, and exposing the Kubernetes API.

It consists of several components, primarily:

- **kube-apiserver**: Exposes the Kubernetes API and acts as the central communication interface for the cluster.
- **kube-controller-manager**: Runs controllers that continuously reconcile the actual cluster state with the desired state.
- **kube-scheduler**: Selects appropriate worker nodes for newly created Pods.
- **etcd**: Stores the persistent state of the Kubernetes cluster.

The control plane is one of the most security-sensitive parts of Kubernetes because compromising it can give an attacker extensive control over the cluster.

For example:

- Compromising the API Server or obtaining highly privileged API credentials can allow unauthorized workload creation and modification.
- Compromising the Controller Manager or Scheduler can affect cluster operations and potentially influence workload placement.
- Compromising etcd can expose sensitive cluster information, including Kubernetes Secrets, and potentially enable cluster takeover.

Therefore, control plane security focuses on protecting component communication, authentication, authorization, configuration, certificates, and access to the Kubernetes API.

# API Server's Security Role

The kube-apiserver is the central security enforcement point for Kubernetes API operations.

When a user, administrator, controller, or application interacts with Kubernetes through the API, requests generally pass through the API Server.

The API Server processes requests through several security-related stages:

| Stage | Responsibility |
| --- | --- |
| **Authentication** | Determines who or what is making the request. |
| **Authorization** | Determines whether the authenticated identity has permission to perform the requested operation. |
| **Admission Control** | Determines whether an otherwise authorized request should be accepted, modified, or rejected based on additional policies. |
| **Validation** | Checks whether the requested object and operation satisfy Kubernetes API requirements. |
| **Persistence** | Stores accepted changes in etcd. |

An important distinction:

- **Authentication** answers: **`Who are you?`**
- **Authorization** answers: **`What are you allowed to do?`**
- Admission Control answers: **`Should this request be accepted under the cluster's policies?`**

### Consider this scenario:

An application running inside a Pod uses its ServiceAccount token to send a request to the Kubernetes API Server to create a Deployment.

1. Authentication identifies the application using its ServiceAccount credentials.
2. Authorization checks whether that ServiceAccount has permission to create Deployments in the requested namespace.
3. Admission Control evaluates the Deployment against applicable admission policies.
4. Kubernetes validates the Deployment.
5. The accepted object is persisted in etcd.

- ### Authentication

    Authentication determines the identity of the entity making an API request.

    Kubernetes does not manage ordinary user accounts through native Kubernetes API objects. Instead, it supports several authentication mechanisms, including:

    - Client certificates.
    - Bearer tokens.
    - ServiceAccount tokens.
    - OpenID Connect (OIDC).
    - Authentication webhooks.
    - Other configured authentication mechanisms.

    For example, when an application inside a Pod communicates with the API Server using its `ServiceAccount` token, Kubernetes authenticates the token and identifies the associated ServiceAccount.

    Authentication establishes identity, but it does not determine what that identity is allowed to do.

- ### Authorization

    Authorization determines whether an authenticated identity has permission to perform the requested operation.
    
    Kubernetes supports several authorization mechanisms:
    
    - **RBAC**: Role-Based Access Control.
    - **Node**: Specialized authorization for kubelet identities.
    - **Webhook**: Delegates authorization decisions to an external service.
    - **ABAC**: Attribute-Based Access Control.
    - **AlwaysAllow / AlwaysDeny**: Allow or deny all requests, respectively.
    
    The authorization stage evaluates the identity, requested resource, operation (verb), namespace, and other relevant attributes.

- ### Admission Control

    Admission control provides another security enforcement layer.

    It operates after authentication and authorization, allowing Kubernetes to modify or reject API requests before they are persisted.

    There are two main types of admission controllers:

    - **Mutating admission controllers**

        Can modify incoming API objects.

        For example:
        - Injecting default values.
        - Adding sidecar containers.
        - Applying default security-related configurations.
    
    - **Validating admission controllers**

        Can accept or reject incoming API objects without modifying them.

        For example:

        - Rejecting privileged containers.
        - Enforcing workload security policies.
        - Preventing workloads from using prohibited configurations.

    Admission control is particularly important because authorization alone cannot enforce all workload security requirements.

    > An identity might have permission to create Pods, but admission policies can still reject a Pod that attempts to run a privileged container.