# Rollback

A Deployment rollback allows you to revert a Deployment's Pod template to a previously deployed version when a new release causes problems.

Deployments support rollback because they retain previous ReplicaSets. Each ReplicaSet represents a particular Pod template revision of the Deployment.

When the Pod template of a Deployment changes — for example, when the container image is changed — Kubernetes creates a new ReplicaSet and performs a rollout. The previous ReplicaSet is retained according to `revisionHistoryLimit`, allowing the Deployment to return to an earlier Pod template.

```text
Deployment
    │
    ├── ReplicaSet (revision 1)
    │      └── old Pods
    │
    ├── ReplicaSet (revision 2)
    │      └── old Pods
    │
    └── ReplicaSet (revision 3)
           └── current Pods
```

A rollback does **not** restore the old Pods themselves. Kubernetes uses the previous Pod template and performs another rollout to recreate Pods based on that template.

---

## Viewing Deployment History

You can view the recorded revisions of a Deployment using:

```bash
kubectl rollout history deployment/<deployment-name>
```

Example:

```bash
kubectl rollout history deployment/nginx-deployment
```

Example output:

```text
REVISION  CHANGE-CAUSE
1         Initial deployment
2         Updated nginx image
3         Updated environment variables
```

The revision number identifies a Deployment revision. It is associated with the ReplicaSet representing that revision.

### Viewing a Specific Revision

To inspect the Pod template stored for a particular revision:

```bash
kubectl rollout history deployment/<deployment-name> --revision=<number>
```

Example:

```bash
kubectl rollout history deployment/nginx-deployment --revision=2
```

This can show details such as:

* Container image
* Environment variables
* Container configuration
* Other Pod template fields

---

## CHANGE-CAUSE

`CHANGE-CAUSE` is displayed by `kubectl rollout history` to provide information about why a particular revision was created.

It comes from the `kubernetes.io/change-cause` annotation on the Deployment or its revision ReplicaSet, depending on how the information was recorded.

For example:

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Updated nginx image to 1.26"
```

Then:

```bash
kubectl rollout history deployment/nginx-deployment
```

may show:

```text
REVISION  CHANGE-CAUSE
1         Initial deployment
2         Updated nginx image to 1.26
3         Updated environment variables
```

### About `--record`

Older Kubernetes versions supported:

```bash
kubectl apply -f deployment.yaml --record
```

which automatically recorded the command as the change cause.

However, `--record` is **deprecated and removed from modern kubectl versions**. It should not be used for new documentation or workflows.

For modern Kubernetes, if you want a meaningful change cause, explicitly set the annotation:

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Updated nginx image to 1.26"
```

---

## Deployment Revision Annotation

Kubernetes automatically maintains the:

```text
deployment.kubernetes.io/revision
```

annotation on Deployment-related objects.

For example:

```yaml
deployment.kubernetes.io/revision: "3"
```

This identifies the Deployment revision represented by the ReplicaSet.

When the Pod template changes, such as changing the container image, Kubernetes creates a new ReplicaSet and assigns it a new revision.

For example:

```text
Revision 1 → Initial deployment
Revision 2 → First Pod-template update
Revision 3 → Second Pod-template update
```

The revision number is used to identify the historical Deployment state when performing operations such as:

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

A rollback should be thought of as **another Deployment update that restores an earlier Pod template**, rather than simply "switching back" to an old ReplicaSet.

---

## Rolling Back to the Previous Revision

To roll back to the previous revision:

```bash
kubectl rollout undo deployment/<deployment-name>
```

Example:

```bash
kubectl rollout undo deployment/nginx-deployment
```

The Deployment controller restores the previous Pod template and performs a new rollout according to the Deployment's update strategy.

For a `RollingUpdate` Deployment, Pods are gradually replaced according to `maxSurge` and `maxUnavailable`.

---

## Rolling Back to a Specific Revision

First, view the revision history:

```bash
kubectl rollout history deployment/nginx-deployment
```

Then specify the desired revision:

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

This restores the Pod template represented by revision `2`.

The rollback itself creates a new revision rather than permanently moving the Deployment's revision number backward.

For example:

```text
Revision 1 → nginx:1.24
Revision 2 → nginx:1.25
Revision 3 → nginx:1.26
```

If you roll back from revision 3 to revision 2:

```text
Revision 4 → nginx:1.25
```

Revision 4 is a new revision containing the old Pod template from revision 2.

---

## Checking Rollout Status

To monitor a rollout or rollback:

```bash
kubectl rollout status deployment/<deployment-name>
```

Example:

```bash
kubectl rollout status deployment/nginx-deployment
```

The command waits for the rollout to complete or report a failure.

---

## Revision History Limit

Deployments retain old ReplicaSets so that previous revisions can be inspected or restored.

The number of old ReplicaSets retained can be controlled using:

```yaml
spec:
  revisionHistoryLimit: 10
```

This keeps up to 10 old ReplicaSets for rollback history.

The currently active ReplicaSet is not counted as an old revision.

If an old ReplicaSet required for a rollback has already been removed, that revision can no longer be restored through the Deployment's retained history.

|                                 | `deployment.kubernetes.io/revision`                     | `revisionHistoryLimit`                             |
| ------------------------------- | ------------------------------------------------------- | -------------------------------------------------- |
| What is it?                     | An **annotation**                                       | A **Deployment spec field**                        |
| Example                         | `"3"`                                                   | `10`                                               |
| Meaning                         | Identifies the Deployment's **current revision number** | Controls how many **old ReplicaSets** are retained |
| Changes automatically?          | Yes                                                     | Only when you configure it                         |
| Controls rollback availability? | Helps identify revisions                                | Controls how much old history remains available    |


---

## Deployment Rollback vs. Persistent Data

A Deployment rollback operates at the **application configuration level**.

It restores fields from the Deployment's Pod template, such as:

* Container image
* Environment variables
* Container configuration
* Volumes and other Pod-template configuration

It does **not** restore the previous state of persistent data.

For example, if an application uses a PersistentVolume and the newer application version modifies data stored on that volume, rolling the Deployment back does not undo those data changes.

```text
Deployment rollback
        │
        ├── Application image       → restored
        ├── Pod configuration       → restored
        └── Persistent data         → NOT restored
```

Restoring persistent data requires a separate data-recovery mechanism, such as application-level backups, database backups, volume snapshots, or another storage-specific recovery process.

Therefore, a Deployment rollback should be considered an **application-level rollback**, not a database or storage rollback.

---

# Rollout Pause and Resume

A Deployment rollout can be paused and resumed.

This is useful when you want to make multiple Pod-template changes and have them rolled out together instead of creating separate rollout steps for each change.

## Pausing a Deployment

Pause a Deployment with:

```bash
kubectl rollout pause deployment/<deployment-name>
```

Example:

```bash
kubectl rollout pause deployment/nginx-deployment
```

When a Deployment is paused, changes to its Pod template do not trigger the normal rollout process.

For example, you can modify:

* Container image
* Environment variables
* Resource requests/limits
* Other Pod-template fields

The Deployment object is updated, but the changed Pod template is not rolled out while the Deployment remains paused.

You can check the paused state with:

```bash
kubectl get deployment nginx-deployment
```

or:

```bash
kubectl describe deployment nginx-deployment
```

---

## Scaling While Paused

Pausing a Deployment does **not** prevent scaling.

For example:

```bash
kubectl scale deployment/nginx-deployment --replicas=5
```

can still change the desired number of replicas while the Deployment is paused.

Therefore, pause affects the **rollout of Pod-template changes**, not every possible Deployment operation.

---

## Resuming a Deployment

Resume a paused Deployment with:

```bash
kubectl rollout resume deployment/<deployment-name>
```

Example:

```bash
kubectl rollout resume deployment/nginx-deployment
```

Once resumed, Kubernetes reconciles the Deployment and rolls out the current Pod template.

If several Pod-template changes were made while paused, they can be incorporated into the resulting rollout rather than being rolled out individually.

---

## Example Workflow

Pause the Deployment:

```bash
kubectl rollout pause deployment/nginx-deployment
```

Make multiple changes:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

kubectl set env deployment/nginx-deployment ENV=production
```

The changed Pod template is stored in the Deployment, but the changes are not rolled out while the Deployment remains paused.

Resume the Deployment:

```bash
kubectl rollout resume deployment/nginx-deployment
```

Kubernetes then processes the current Pod template and performs the rollout.

Without pausing, separate Pod-template changes can trigger separate rollouts.

---

## Important Behavior While Paused

Pausing does not stop the currently running Pods.

Existing Pods continue running with their current Pod specifications:

```text
Current Pods
     │
     ├── continue running
     ├── continue serving traffic
     └── continue using their existing Pod template

Deployment
     │
     └── Pod-template changes accumulated while paused
```

For example:

```text
Deployment running
       ↓
rollout pause
       ↓
Existing Pods continue running
       ↓
Change image
       ↓
Change environment variables
       ↓
Change resources
       ↓
No rollout of those Pod-template changes yet
       ↓
rollout resume
       ↓
Rollout begins using the resulting Pod template
```

Pausing a rollout therefore does not inherently cause downtime. The existing Pods remain available unless they independently fail or another operation affects them.

---

# Rollout Commands Summary

| Command                                                    | Purpose                                                                                   |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `kubectl rollout status deployment/<name>`                 | Shows and waits for rollout completion                                                    |
| `kubectl rollout history deployment/<name>`                | Displays Deployment revision history                                                      |
| `kubectl rollout history deployment/<name> --revision=<n>` | Shows details of a specific revision                                                      |
| `kubectl rollout undo deployment/<name>`                   | Rolls back to the previous revision                                                       |
| `kubectl rollout undo deployment/<name> --to-revision=<n>` | Rolls back to a specific revision                                                         |
| `kubectl rollout restart deployment/<name>`                | Restarts Pods by changing the Pod template without changing the application configuration |
| `kubectl rollout pause deployment/<name>`                  | Pauses Pod-template rollout                                                               |
| `kubectl rollout resume deployment/<name>`                 | Resumes a paused Deployment                                                               |
