# Kubernetes Namespace Stuck in `Terminating` — RKE1 Incident

## Incident Summary

We had an **RKE1 Kubernetes cluster** running a microservice. A GitLab CI/CD pipeline was responsible for uninstalling and then reinstalling the microservice.

I ran the pipeline, but the expected changes did not occur to the microservice Pod. As a troubleshooting step, I decided to delete the namespace containing the microservice:

```bash
kubectl delete namespace <namespace>
```

However, the namespace did not disappear. Its status remained:

```text
Terminating
```

I verified this with:

```bash
kubectl get namespace <namespace>
```

---

## Investigation: Namespace Finalizer

I checked the namespace configuration and found a finalizer:

```yaml
metadata:
  finalizers:
    - kubernetes
```

A **finalizer** is a mechanism Kubernetes uses to prevent an object from being completely deleted until required cleanup has been completed.

When a resource with a finalizer is deleted, Kubernetes does not immediately remove it from `etcd`. Instead, it marks the object for deletion by setting a `deletionTimestamp` and waits for the responsible cleanup process to remove the finalizer.

Conceptually:

```text
kubectl delete namespace
        ↓
   kube-apiserver
        ↓
   deletionTimestamp set
        ↓
   finalizer still exists
        ↓
   namespace remains Terminating
        ↓
   cleanup must complete
        ↓
   finalizer removed
        ↓
   namespace finally deleted
```

---

## Attempt to Remove the Finalizer

I first tried to remove the finalizer manually:

```bash
kubectl edit namespace <namespace>
```

and changed:

```yaml
finalizers:
  - kubernetes
```

to:

```yaml
finalizers: []
```

After saving and exiting with `:wq`, I checked the namespace again, but the finalizer was still present.

This indicated that simply editing the normal namespace object was not successfully removing the finalizer in this situation.

I then used the namespace finalization API to remove the finalizer. After doing so, the namespace was successfully deleted.

---

## Further Investigation

After the namespace was removed, I checked the RKE1 control-plane nodes:

```bash
docker ps -a
```

I found that two important Kubernetes control-plane components were stopped:

```text
kube-controller-manager   Exited
kube-scheduler            Exited
```

This was an important finding.

The **kube-controller-manager** runs Kubernetes controllers responsible for continuously reconciling cluster state. Namespace lifecycle and finalization involve controller-based cleanup, so a failed or unavailable controller-manager can contribute to resources becoming stuck during deletion.

The **kube-scheduler** being stopped was a separate issue. Its responsibility is to assign unscheduled Pods to appropriate nodes. Therefore, if the scheduler is unavailable, newly created Pods that require scheduling may remain unscheduled.

---

## Important Distinction

The `kube-controller-manager` does **not** receive the original `kubectl delete` request.

The basic deletion flow is:

```text
kubectl
   ↓
kube-apiserver
   ↓
Authentication
   ↓
Authorization (RBAC)
   ↓
Admission
   ↓
etcd
```

The API server handles the API request and records the deletion state.

After that, Kubernetes controllers may need to react to the deletion and perform cleanup. If a finalizer is present, the object cannot be completely removed until the finalization process is completed.

Therefore, in this incident, the namespace being stuck in `Terminating` was not simply because **"the API server couldn't delete it."** The API server had accepted the deletion, but the deletion process was not completing.

---

## Important Troubleshooting Lesson

When a Kubernetes resource remains in `Terminating`, do not immediately remove its finalizer.

First investigate:

```bash
kubectl describe namespace <namespace>
kubectl get namespace <namespace> -o yaml
```

Check:

- `deletionTimestamp`
- `finalizers`
- Remaining resources inside the namespace
- API/resource discovery problems
- Controller-manager health
- Controller-manager logs
- Admission/webhook problems
- Whether the responsible controller is running

**Removing a finalizer forces Kubernetes to skip the cleanup that the finalizer was intended to guarantee.** It can therefore leave dependent resources or external resources behind.

---

## Key Takeaway

    A resource stuck in `Terminating` usually means deletion has been requested, but Kubernetes cannot complete the required cleanup/finalization. Finalizers are safeguards against premature deletion; they should normally be removed by the responsible controller. Manually removing a finalizer should be treated as a troubleshooting/remediation step, not the first solution.