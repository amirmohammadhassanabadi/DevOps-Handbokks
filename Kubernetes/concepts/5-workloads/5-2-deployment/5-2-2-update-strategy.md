# Deployment Update and Release Strategies

A Kubernetes Deployment provides two built-in strategies for replacing Pods when its Pod template changes:

1. **RollingUpdate** — gradually replaces old Pods with new Pods.
2. **Recreate** — terminates all existing Pods before creating new ones.

Other release patterns, such as **Canary** and **Blue-Green**, are not Deployment strategy types. They are higher-level deployment approaches that can be implemented using multiple Kubernetes resources and, when necessary, additional traffic-management tools.

---

# RollingUpdate

**RollingUpdate** is the default Deployment strategy.

When the Pod template changes, for example when the container image is updated, the Deployment Controller creates a **new ReplicaSet** containing the new Pod template. It then gradually scales the new ReplicaSet up and the old ReplicaSet down.

The process is:

```text
Old ReplicaSet
      ↓
Old Pods
      ↓
Deployment updated
      ↓
Deployment Controller
      ↓
Create New ReplicaSet
      ↓
New ReplicaSet
      ↓
New Pods gradually created
      ↓
Old ReplicaSet gradually scaled down
      ↓
Old Pods gradually terminated
      ↓
All desired Pods run the new version
```

The old and new ReplicaSets can temporarily coexist:

```text
Deployment
    │
    ├── Old ReplicaSet
    │      ├── Pod v1
    │      └── Pod v1
    │
    └── New ReplicaSet
           └── Pod v2
```

The exact number of old and new Pods during the rollout is controlled primarily by two parameters.

- ### `maxUnavailable`

    `maxUnavailable` specifies the **maximum number of Pods that can be unavailable during the rolling update**, relative to the desired number of replicas.

- ### `maxSurge`

    `maxSurge` specifies the **maximum number of additional Pods that can exist above the desired replica count during the rolling update**.

Example:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

If the Deployment has:

```yaml
replicas: 5
```

then:

* Up to **1 Pod can be unavailable** during the rollout.
* Up to **1 additional Pod** can temporarily exist above the desired count.
* Therefore, the Deployment can temporarily have **up to 6 Pods**.

These values are **limits, not instructions**. They do not mean that Kubernetes must always delete one Pod and create one Pod at the same time. The Deployment Controller adjusts the old and new ReplicaSets while respecting these limits and the availability of the Pods.

A simplified rollout could look like:

```text
Initial:
5 × v1

        ↓

4 × v1 + 1 × v2
6 Pods total

        ↓

4 × v1 + 1 × v2
old Pod removed as rollout progresses

        ↓

3 × v1 + 2 × v2

        ↓

2 × v1 + 3 × v2

        ↓

1 × v1 + 4 × v2

        ↓

0 × v1 + 5 × v2
```

The exact sequence can vary depending on Pod readiness, resource availability, and the configured rollout parameters.

### Availability During Rolling Updates

Rolling updates are designed to maintain application availability while the new version is introduced, but **RollingUpdate does not guarantee zero downtime**.

Availability depends on several factors, including:

* Number of replicas
* `maxUnavailable`
* `maxSurge`
* Readiness probes
* Application startup time
* Application behavior
* Service configuration
* Available cluster resources

A readiness probe is particularly important because Kubernetes should not consider a new Pod available for normal Service traffic until it becomes ready.

> RollingUpdate controls **how Pods are replaced**. It does not directly control a percentage of user traffic.

---

## Availability & Progress

During a Deployment rollout, Kubernetes needs to determine whether new Pods are becoming ready and whether the rollout is making progress. Two settings control important parts of this behavior: **`minReadySeconds`** and **`progressDeadlineSeconds`**.

- ### minReadySeconds

    `minReadySeconds` specifies how long a newly created Pod must remain **Ready** without any container crashing or becoming unready before the Deployment considers that Pod **available**.
    
    ```yaml
    spec:
      minReadySeconds: 10
    ```
    
    With `minReadySeconds: 10`, a Pod must remain continuously Ready for 10 seconds before it counts as available.
    
    This is useful when an application can become Ready briefly and then fail shortly afterward. It prevents Kubernetes from immediately considering such a Pod reliably available.
    
    Default:
    
    ```yaml
    minReadySeconds: 0
    ```
    
    With the default value, a Pod can be considered available as soon as it becomes Ready.

- ### progressDeadlineSeconds

  `progressDeadlineSeconds` specifies how long Kubernetes waits for a Deployment rollout to make progress before considering the rollout **stalled**.

  ```yaml
  spec:
    progressDeadlineSeconds: 600
  ```

  The default is:

  ```text
  600 seconds
  ```

  If the Deployment does not make sufficient progress within this period, Kubernetes reports:

  ```text
  ProgressDeadlineExceeded
  ```

  This does **not** automatically roll back the Deployment. Kubernetes reports the stalled rollout, but rollback must be performed separately if required.

### Deployment Conditions

Kubernetes exposes Deployment conditions through:

```bash
kubectl describe deployment <deployment-name>
```

and:

```bash
kubectl get deployment <deployment-name> -o yaml
```

The main conditions are:

| Condition        | Meaning                                                                                                                  |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `Progressing`    | The Deployment is making progress, such as creating a new ReplicaSet or increasing the number of updated/available Pods. |
| `Available`      | The Deployment has the required number of available Pods according to its availability requirements.                     |
| `ReplicaFailure` | The Deployment encountered a failure while creating or managing its ReplicaSet or Pods.                                  |

For example:

```yaml
status:
  conditions:
    - type: Progressing
      status: "True"
      reason: NewReplicaSetAvailable

    - type: Available
      status: "True"
      reason: MinimumReplicasAvailable
```

A stalled rollout may report:

```yaml
- type: Progressing
  status: "False"
  reason: ProgressDeadlineExceeded
```

### ProgressDeadlineExceeded

A Deployment can become stalled for several reasons, for example:

* New Pods cannot be scheduled because of insufficient cluster resources.
* The container image cannot be pulled.
* Pods repeatedly fail their health checks.
* Pods never become Ready.
* A required Deployment condition is not satisfied.

When the progress deadline is exceeded, Kubernetes sets the `Progressing` condition to `False` with the reason `ProgressDeadlineExceeded`.

The Deployment does **not** automatically revert to the previous version.

The operator can investigate the problem and, if necessary, perform a rollback:

```bash
kubectl rollout undo deployment/<deployment-name>
```

### Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
  minReadySeconds: 10
  progressDeadlineSeconds: 300
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.26
```

In this example:

* A newly updated Pod must remain Ready for **10 seconds** before it counts as available.
* Kubernetes allows up to **300 seconds** for the rollout to make progress.
* If progress stalls for longer than 300 seconds, the Deployment reports `ProgressDeadlineExceeded`.
* Kubernetes does **not** automatically roll back the Deployment.

---

# Recreate

The **Recreate** strategy terminates all existing Pods before creating Pods from the new Pod template.

```yaml
strategy:
  type: Recreate
```

The process is:

```text
Old Pods
   ↓
Terminate all old Pods
   ↓
No old Pods remain
   ↓
Create new Pods
   ↓
New Pods become Ready
   ↓
Application available again
```

Unlike `RollingUpdate`, old and new application Pods are not intentionally kept running simultaneously.

### Characteristics

* All existing Pods are terminated before new Pods are created.
* Normally causes an application availability gap.
* Simpler than a rolling update.
* Useful when old and new application versions cannot safely run at the same time.

For example, an application might require exclusive access to a resource that cannot be safely used by both versions simultaneously.

---

# RollingUpdate vs Recreate

| Feature                             | RollingUpdate                     | Recreate                      |
| ----------------------------------- | --------------------------------- | ----------------------------- |
| Default Deployment strategy         | Yes                               | No                            |
| Old and new Pods coexist            | Yes, temporarily                  | No                            |
| Pods replaced gradually             | Yes                               | No                            |
| All old Pods terminated first       | No                                | Yes                           |
| Application availability            | Designed to maintain availability | Availability gap expected     |
| Resource usage during update        | Can temporarily increase          | No surge from old/new overlap |
| Useful when versions cannot coexist | Usually not                       | Yes                           |

---

> # Canary Deployment
> 
> A **Canary deployment** releases a new application version to a **small portion of users or traffic first**. The new version is then monitored before the rollout is expanded.
> 
> The name comes from the historical use of canary birds as an early warning for dangerous gases in mines.
> 
> The software deployment concept follows the same idea:
> 
> ```text
> Stable version
>      ↓
> Most traffic
> 
> Canary version
>      ↓
> Small portion of traffic
> ```
> 
> For example:
> 
> ```text
> Initial:
> v1 → 100%
> v2 →   0%
> 
> Canary:
> v1 → 95%
> v2 →  5%
> 
> If healthy:
> 
> v1 → 90%
> v2 → 10%
> 
>         ↓
> 
> v1 → 50%
> v2 → 50%
> 
>         ↓
> 
> v1 →  0%
> v2 → 100%
> ```
> 
> The important characteristic is that the new version receives a **deliberately limited portion of traffic** while the old version continues serving the majority.
> 
> ### RollingUpdate vs Canary
> 
> A RollingUpdate and a Canary deployment can look similar because both may temporarily run old and new Pods.
> 
> However, their goals are different:
> 
> 
> **RollingUpdate** → Gradually replace old Pods with new Pods
> 
> **Canary** → Deliberately expose a small portion of traffic to the new version first
> 
> 
> A normal Kubernetes Service does not provide precise percentage-based traffic splitting between two versions. Simply running 9 Pods of v1 and 1 Pod of v2 behind the same Service does **not guarantee exactly 90%/10% user traffic**.
> 
> Precise canary traffic management can be implemented using technologies such as:
> 
> * Ingress controllers with canary-routing capabilities
> * Service meshes
> * Progressive-delivery controllers such as Argo Rollouts
> * Flagger
> 
> These can provide mechanisms such as percentage-based traffic splitting, header-based routing, cookie-based routing, or other routing rules, depending on the technology being used.
> 
> ---
> 
> # Implementing Canary Deployments
> 
> - ## Option 1: Two Deployments + One Service
> 
>   A simple approach is to create two Deployments:
>   
>   ```text
>   Stable Deployment
>   my-app-v1
>   ├── Pod v1
>   ├── Pod v1
>   ├── Pod v1
>   └── Pod v1
>   
>   Canary Deployment
>   my-app-v2
>   └── Pod v2
>   ```
>   
>   Both Pod sets can have the same application label:
>   
>   ```yaml
>   labels:
>     app: my-app
>   ```
>   
>   and the Service can select:
>   
>   ```yaml
>   selector:
>     app: my-app
>   ```
>   
>   This allows the Service to send traffic to both versions.
>   
>   However, traffic distribution is only approximate and depends on the networking implementation and connection/request behavior. It should **not** be treated as an exact percentage-based canary mechanism.
>   
>   ### Advantages
>   
>   * Simple
>   * Uses standard Kubernetes resources
>   * No additional progressive-delivery controller required
>   
>   ### Limitations
>   
>   * No precise traffic percentage
>   * No built-in user-based routing
>   * Limited control over which users receive the canary
>   * Monitoring and rollback logic must be handled separately
>   
>   ---
> 
> - ## Option 2: Ingress-Based Canary
> 
>   An Ingress controller can provide more intentional routing between stable and canary versions, depending on the controller and its supported features.
>   
>   For example:
>   
>   ```text
>   Normal users
>        ↓
>       v1
>   
>   Users matching canary rule
>        ↓
>       v2
>   ```
>   
>   Possible routing mechanisms include:
>   
>   * Percentage-based traffic
>   * Specific HTTP headers
>   * Cookies
>   * Selected users or request characteristics
>   
>   For example:
>   
>   ```text
>   x-canary: true
>           ↓
>          v2
>   ```
>   
>   The exact configuration depends on the Ingress controller being used.
>   
>   ---
> 
> - ## Option 3: Progressive Delivery
> 
>   For more advanced canary workflows, a progressive-delivery controller can automate the rollout process.
>   
>   A typical workflow is:
>   
>   ```text
>   v1 → 100%
>   v2 →   0%
>         ↓
>   Deploy v2
>         ↓
>   v1 → 90%
>   v2 → 10%
>         ↓
>   Monitor metrics
>         ↓
>   Healthy?
>      ↙       ↘
>    Yes       No
>     ↓         ↓
>   Increase   Pause/Rollback
>   traffic
>     ↓
>   v2 → 100%
>   ```
>   
>   Metrics can include:
>   
>   * Error rate
>   * Request latency
>   * CPU/memory usage
>   * HTTP response codes
>   * Application-specific business metrics
>   
>   Tools such as **Argo Rollouts** and **Flagger** can provide progressive-delivery capabilities beyond what a standard Deployment provides.
 
---

> # Blue-Green Deployment
> 
> A **Blue-Green deployment** maintains two separate versions of an application:
> 
> * **Blue** → current production version
> * **Green** → new version being prepared
> 
> Only one environment normally receives production traffic at a time.
> 
> ```text
>                  ┌── Blue → v1
> Traffic ─────────┤
>                  └── Green → v2
> ```
> 
> Before the switch:
> 
> ```text
> Traffic
>    ↓
> Blue (v1) → 100%
> 
> Green (v2) → 0%
> ```
> 
> After the switch:
> 
> ```text
> Traffic
>    ↓
> Green (v2) → 100%
> 
> Blue (v1) → 0%
> ```
> 
> The key idea is that the new version is deployed and validated **before production traffic is switched to it**.
> 
> ## Blue-Green Process
> 
> ### 1. Blue is serving production traffic
> 
> ```text
> Blue → v1 → 100% traffic
> Green → v2 → 0% traffic
> ```
> 
> ### 2. Deploy Green
> 
> The new version is deployed separately:
> 
> ```text
> Blue → v1 → 100%
> Green → v2 → 0%
> ```
> 
> ### 3. Test Green
> 
> Green can be tested using health checks, automated tests, or controlled access before receiving production traffic.
> 
> ### 4. Switch Traffic
> 
> The routing layer is changed so that production traffic goes to Green.
> 
> ```text
> Before:
> Service → Blue
> 
> After:
> Service → Green
> ```
> 
> ### 5. Monitor Green
> 
> Blue can remain available temporarily as a fallback.
> 
> ### 6. Roll Back if Necessary
> 
> If a problem is discovered:
> 
> ```text
> Service → Green
>            ↓
>         problem
>            ↓
> Service → Blue
> ```
> 
> The rollback can be performed by changing the routing back to Blue rather than rebuilding the old version.
> 
> ### 7. Clean Up
> 
> Once Green is confirmed stable, Blue can be removed or retained for the next release cycle.
> 
> ---
> 
> # Blue-Green in Kubernetes
> 
> A simple Blue-Green implementation can use **two Deployments and a Service**.
> 
> For example:
> 
> ```text
> Deployment: my-app-blue
>         ↓
>       Pods v1
> 
> Deployment: my-app-green
>         ↓
>       Pods v2
> 
>         ↓
>       Service
>         ↓
>    Production Traffic
> ```
> 
> Initially, the Service selects the Blue Pods:
> 
> ```yaml
> selector:
>   app: my-app
>   version: blue
> ```
> 
> After Green has been validated, the Service selector can be changed to:
> 
> ```yaml
> selector:
>   app: my-app
>   version: green
> ```
> 
> Traffic is then directed to the Green Pods.
> 
> If a rollback is required, the selector can be changed back to:
> 
> ```yaml
> selector:
>   app: my-app
>   version: blue
> ```
> 
> The exact traffic-switch behavior depends on the Service/networking layer, and existing long-lived connections may require connection draining or additional handling.
> 
> ## Advantages
> 
> * New version can be tested before receiving normal production traffic
> * Fast traffic switch
> * Straightforward rollback by changing routing back
> * Old and new environments remain isolated from each other during validation
> 
> ## Disadvantages
> 
> * Higher resource consumption because both environments may run simultaneously
> * Requires a mechanism for switching traffic
> * Stateful applications require careful data-management planning
> * Long-lived connections may require connection draining
> * The new environment may need access to the same external dependencies and data, which can complicate compatibility
> 
> ---
> 
> # Blue-Green vs RollingUpdate
> 
> | Feature                                       | Blue-Green                                      | RollingUpdate                                                     |
> | --------------------------------------------- | ----------------------------------------------- | ----------------------------------------------------------------- |
> | Main idea                                     | Switch traffic between two environments         | Gradually replace Pods                                            |
> | Deployment resources                          | Usually two Deployments                         | Usually one Deployment                                            |
> | Old/new versions coexist                      | Yes                                             | Yes, temporarily                                                  |
> | Traffic control                               | Explicit switch between environments            | Normally both versions may be selected during rollout             |
> | Rollout                                       | Prepare new environment, then switch            | Gradually replace Pods                                            |
> | Rollback                                      | Switch traffic back to old environment          | Roll out the previous version again                               |
> | Resource usage                                | Higher                                          | Usually lower                                                     |
> | Testing new version before production traffic | Yes                                             | Limited; new Pods become part of rollout as they become available |
> | Typical use                                   | Releases requiring a separate ready environment | Normal application updates                                        |

# Summary

Kubernetes Deployments provide two native update strategies:

```text
Deployment
│
├── RollingUpdate
│     └── Gradually replace old Pods with new Pods
│
└── Recreate
      └── Remove old Pods before creating new Pods
```

Canary and Blue-Green are **release strategies rather than Deployment strategy types**:

> **RollingUpdate** → How Pods are gradually replaced
> 
> **Canary** → How a small portion of traffic/users receives the new version first
> 
> **Blue-Green** → How two complete environments are maintained and traffic is switched
  between them


The choice depends on the application's availability requirements, compatibility between versions, traffic-management capabilities, resource constraints, and rollback requirements.
