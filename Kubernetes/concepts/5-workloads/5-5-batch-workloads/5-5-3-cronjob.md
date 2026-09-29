# CronJob

A **CronJob** is a Kubernetes controller used to run **Jobs on a repeating schedule**. It is similar to the traditional Linux `cron` system.

The execution flow is:

```text
CronJob
   │
   │ creates according to schedule
   ▼
 Job
   │
   │ creates/manages
   ▼
 Pods
   │
   │ run until completion
   ▼
Complete / Failed
```

Each scheduled execution creates a **new Job**. The Job then manages its own Pods, retries, completions, and failure handling.

CronJobs are commonly used for:

* Database backups
* Log cleanup
* Report generation
* Data synchronization
* Periodic maintenance
* Scheduled data processing
* Cleanup tasks

---

## Schedule

A CronJob uses a standard **five-field cron expression**:

```text
* * * * *
│ │ │ │ └── Day of week (0–7, Sunday is 0 or 7)
│ │ │ └──── Month (1–12)
│ │ └────── Day of month (1–31)
│ └──────── Hour (0–23)
└────────── Minute (0–59)
```

For example:

```yaml
schedule: "0 2 * * *"
```

means:

```text
Every day at 02:00
```

Another example:

```yaml
schedule: "*/15 * * * *"
```

means:

```text
Every 15 minutes
```

---

## Important Fields

- ### `schedule`

    Defines when the CronJob should create a Job.

    ```yaml
    schedule: "0 2 * * *"
    ```

    ---

- ### `jobTemplate`

    Defines the **Job specification** that will be created for every scheduled execution.

    The fields belong to the Job and therefore control how that particular execution behaves.

    For example:

    ```yaml
    jobTemplate:
      spec:
        completions: 1
        parallelism: 1
        backoffLimit: 4
        template:
          spec:
            restartPolicy: OnFailure
            containers:
              - name: backup
                image: backup-tool:1.0
    ```

    Important Job fields that can be configured inside `jobTemplate` include:

    * `completions`
    * `parallelism`
    * `backoffLimit`
    * `activeDeadlineSeconds`
    * `ttlSecondsAfterFinished`
    * Pod/container configuration

    The CronJob itself does **not** manage these Job execution details.

    ---

- ### `concurrencyPolicy`

    Controls what happens when a new schedule occurs while a previous Job created by the **same CronJob** is still running.

    Options:

    - #### `Allow` — default

        Allows multiple Jobs from the same CronJob to run concurrently.

        ```text
        Schedule 1 → Job-1 ────────────────┐
        Schedule 2 → Job-2 ────────────┐   │
        Schedule 3 → Job-3 ────────┐   │   │
                                    ▼   ▼   ▼
                                 Running simultaneously
        ```

    - #### `Forbid`

        If the previous Job is still running, the new scheduled execution is skipped.

        ```text
        Schedule 1 → Job-1 ──────────────→ Complete
        Schedule 2 → skipped
        Schedule 3 → skipped
        ```

        The skipped executions are considered **missed schedules** and can be affected by `startingDeadlineSeconds`.

    - #### `Replace`

        If the previous Job is still running when the next schedule occurs, Kubernetes terminates the currently running Job and starts a new one.

        ```text
        Job-1 ───────────────X
                             │
                             ▼
                          Job-2
        ```

        `concurrencyPolicy` only controls Jobs created by the **same CronJob**.

    ---

- ### `startingDeadlineSeconds`

    Defines how late a scheduled Job is allowed to be created.

    For example:

    ```yaml
    startingDeadlineSeconds: 300
    ```

    allows Kubernetes to create a missed Job up to **300 seconds after its scheduled time**.

    Example:

    ```text
    Scheduled:       02:00
    Controller sees: 02:05

    Deadline = 300 seconds → Job can still be created
    ```

    But:

    ```text
    Scheduled:       02:00
    Controller sees: 02:10

    Deadline = 300 seconds
    → Execution is skipped
    ```

    If `startingDeadlineSeconds` is not specified, there is no configured deadline for missed executions.

    ---

- ### `successfulJobsHistoryLimit`

    Controls how many completed successful Jobs are retained.

    Default:

    ```yaml
    successfulJobsHistoryLimit: 3
    ```

    Example:

    ```yaml
    successfulJobsHistoryLimit: 3
    ```

    Kubernetes retains the most recent successful Jobs and removes older ones.

    ---

- ### `failedJobsHistoryLimit`

    Controls how many failed Jobs are retained.

    Default:

    ```yaml
    failedJobsHistoryLimit: 1
    ```

    Example:

    ```yaml
    failedJobsHistoryLimit: 1
    ```

    Only the most recent failed Job is retained.

    These fields control **Job history cleanup**; they do not control whether the Job itself succeeds or fails.

    ---

- ### `suspend`

    Temporarily stops new executions of the CronJob.
    
    ```yaml
    suspend: true
    ```
    
    This does **not** terminate Jobs that are already running.
    
    While suspended, scheduled executions can become **missed schedules**. When the CronJob is resumed, those missed executions can potentially be started depending on `startingDeadlineSeconds`.
    
---

## CronJob Time Zone

CronJob schedules are evaluated by the **CronJob controller**, which runs inside the `kube-controller-manager`.

A CronJob can explicitly specify its timezone:

```yaml
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Tehran"
```

`timeZone` specifies the timezone used to interpret the schedule.

If `timeZone` is **not specified**, Kubernetes uses the local timezone of the `kube-controller-manager`.

Therefore:

```text
CronJob
   │
   │ schedule interpreted using
   ▼
CronJob Controller
(kube-controller-manager)
   │
   │ creates Job
   ▼
Job
   │
   ▼
Pod
```

The worker node where the Pod eventually runs does **not** determine when the CronJob is scheduled.

Therefore, accurate and synchronized time on the control-plane environment is important, particularly when relying on the controller manager's local timezone.

---

## How the CronJob Controller Works

CronJobs are **controller-reconciled**, not event-driven timers.

The CronJob controller periodically examines CronJob resources and determines whether any scheduled execution is due.

The controller approximately checks every **10 seconds**. This means a CronJob should not be thought of as executing at exactly the scheduled second.

For example:

```text
Schedule: 02:00:00

Controller checks:
01:59:50
02:00:00  ← detects schedule
02:00:10
```

Depending on when reconciliation occurs, the Job may start slightly after the scheduled time.

The ~10-second interval is an implementation behavior, not a guarantee that every Job starts exactly 10 seconds after or at the schedule.

---

## Reconciliation Process

Conceptually, during reconciliation the controller:

1. Reads the CronJob configuration.
2. Determines the relevant schedule.
3. Calculates scheduled execution times.
4. Checks the CronJob's observed state and existing Jobs.
5. Determines whether a schedule was missed.
6. Applies `concurrencyPolicy`.
7. Applies `startingDeadlineSeconds`.
8. Creates a new Job when appropriate.
9. Updates the CronJob's status.

Important information used by the controller includes:

* Cron schedule
* `status.lastScheduleTime`
* Existing Jobs owned by the CronJob
* `concurrencyPolicy`
* `startingDeadlineSeconds`
* `suspend`
* Current controller time
* CronJob timezone configuration

The controller does not simply keep a timer in memory for every CronJob. It continuously reconciles the desired schedule against the observed cluster state.

---

## Missed Schedules

A schedule can be missed if the controller cannot create the Job at the scheduled time.

Examples include:

* Controller downtime
* `concurrencyPolicy: Forbid`
* CronJob being suspended
* Other scheduling/controller conditions

When reconciliation resumes, Kubernetes can determine whether missed executions should be started.

`startingDeadlineSeconds` controls how far back Kubernetes considers a missed execution eligible for creation.

For example:

```yaml
schedule: "*/1 * * * *"
startingDeadlineSeconds: 120
```

The controller considers missed executions within the configured deadline.

There is also a protection against excessive missed schedules. If more than **100 schedules** are missed in the relevant calculation window, Kubernetes does not start the Job and reports the problem in the controller logs.

---

## CronJob Execution Is Approximate

A CronJob is designed to create approximately one Job per scheduled execution.

Kubernetes attempts to avoid duplicate or missing Job creation, but under certain failure conditions it cannot guarantee exactly one Job for every scheduled time.

Therefore, **CronJob workloads should be idempotent**.

For example, a backup task should be designed so that accidentally executing it twice does not corrupt the system or produce an invalid result.

Do not design a CronJob assuming:

```text
1 scheduled time = guaranteed exactly 1 Job
```

Instead, design the workload so that duplicate execution can be safely handled.

---

## Job History vs Job Execution

The CronJob creates Jobs, while each Job manages its own Pods.

For example:

```text
CronJob
 │
 ├── Job-1
 │    └── Pod(s)
 │
 ├── Job-2
 │    └── Pod(s)
 │
 ├── Job-3
 │    └── Pod(s)
 │
 └── Job-4
      └── Pod(s)
```

Each Job is an independent execution.

Therefore, Job-level settings such as:

```text
completions
parallelism
backoffLimit
activeDeadlineSeconds
ttlSecondsAfterFinished
restartPolicy
```

apply to each individual Job created by the CronJob.

The CronJob-level settings such as:

```text
schedule
concurrencyPolicy
startingDeadlineSeconds
timeZone
suspend
successfulJobsHistoryLimit
failedJobsHistoryLimit
```

control how those Jobs are scheduled and retained.

---

## Example CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Tehran"

  concurrencyPolicy: Forbid

  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1

  jobTemplate:
    spec:
      backoffLimit: 4
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: backup-tool:1.0
              command: ["sh", "-c", "backup.sh"]
```

### Meaning

```text
Every day at 02:00 Asia/Tehran
        │
        ▼
CronJob creates a Job
        │
        ▼
Job creates a Pod
        │
        ▼
Pod runs backup.sh
        │
        ├── Success → Job Complete
        │
        └── Failure → Job retry handling
```

Because `concurrencyPolicy: Forbid` is configured, Kubernetes will not start another Job from this CronJob while the previous Job is still running.

---

## Summary

A **CronJob is a scheduler for Jobs**:

```text
CronJob
   │
   │ schedule
   ▼
Job
   │
   │ manages
   ▼
Pod(s)
   │
   │ execute
   ▼
Complete / Failed
```

Key points:

* `schedule` defines when executions should occur.
* Every scheduled execution creates a **new Job**.
* `jobTemplate` defines the Job that will be created.
* The Job manages Pods, retries, completions, and failures.
* `concurrencyPolicy` controls overlapping executions.
* `startingDeadlineSeconds` controls how late a missed execution may be started.
* `timeZone` explicitly defines the timezone used for schedule evaluation.
* Without `timeZone`, the kube-controller-manager's local timezone is used.
* `suspend` temporarily stops new executions.
* History limits control how many successful/failed Jobs are retained.
* CronJob scheduling is approximate, not an exact real-time timer.
* CronJob workloads should be **idempotent** because Kubernetes cannot guarantee exactly one Job for every scheduled instant.
* The CronJob controller periodically reconciles CronJobs and creates Jobs when appropriate.
