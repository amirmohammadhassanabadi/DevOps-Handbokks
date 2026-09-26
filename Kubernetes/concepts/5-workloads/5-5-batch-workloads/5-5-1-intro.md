# Batch Workloads

Kubernetes workloads can generally be divided into two categories:

* **Long-running application workloads**
* **Batch workloads**

Controllers such as **Deployment, StatefulSet, and DaemonSet** are primarily designed for long-running applications. Kubernetes continuously reconciles these resources to maintain their desired state and keep their Pods running.

For example, a web server managed by a Deployment should remain available continuously. If a Pod fails, Kubernetes creates a replacement to maintain the desired number of replicas.

In contrast, **batch workloads** are designed for tasks that eventually **finish**.

Kubernetes provides two main batch workload resources:

* **Job** — Runs Pods until a task completes successfully.
* **CronJob** — Creates Jobs according to a defined schedule.

A Job can run one or multiple Pods depending on its configuration. Once the required successful completions are achieved, the Job is considered complete.

A CronJob extends the Job concept by creating Jobs at specific times or intervals using a cron-style schedule.

For example:

```text id="6zq8h3"
CronJob
   │
   ├── creates → Job
   │               │
   │               └── creates → Pod(s)
   │                              │
   │                              └── completes
   │
   ├── creates → Job
   │               │
   │               └── creates → Pod(s)
   │                              │
   │                              └── completes
   │
   └── ...
```

Common batch workload use cases include:

* Database migrations
* Backups
* Data processing
* Report generation
* Scheduled maintenance
* One-time initialization tasks
* Cleanup operations

## Summary

**Long-running workloads** focus on keeping applications continuously available:

**Batch workloads** focus on executing tasks until completion:

The key distinction is:

> Long-running workload → Keep running
>
> Batch workload        → Run → Complete → Stop