# Kubernetes

Kubernetes (K8s) is an open-source orchestration platform designed to automate the lifecycle management—deployment, scaling, networking, and maintenance—of containerized applications across a multi-node cluster.

Essentially, while container runtimes handle application packaging, Kubernetes bridges the gap between individual container execution and large-scale infrastructure management, ensuring high availability and reliability across distributed systems.

Kubernetes was originally developed by Google and released as open source in 2014. It was inspired by Google’s internal cluster management system called **Borg**. Today Kubernetes is maintained by the Cloud Native Computing Foundation (CNCF) and has become the industry standard for container orchestration.

## The Problem Kubernetes Solves:

Running one container is easy, but running hundreds or thousands of containers across many servers introduces problems such as:
- **Scheduling:** deciding which machine should run each container.
- **High availability:** maintaining the desired number of application instances and recovering workloads when containers or nodes fail.
- **Scaling:** increasing or decreasing the number of application instances.
- **Networking:** allowing containers across machines to communicate.
- **Load balancing:** distributing network traffic across multiple Pod instances.
- **Rolling updates:** updating applications without downtime.
- **Service discovery:** finding where services are running.
Docker alone does not manage these concerns across multiple machines. Kubernetes solves these problems automatically. Docker is primarily a container platform for building and running containers, while Kubernetes is a container orchestration platform designed to manage workloads across a cluster of machines.

Kubernetes typically uses a container runtime such as containerd or CRI-O to actually run containers.

## Cluster:

A Kubernetes cluster is a group of nodes managed by Kubernetes as a single system. It consists of a control plane and one or more worker nodes. It allows:

- applications to run across multiple servers
- workloads to be recovered or rescheduled when nodes fail
- horizontal scaling of workloads


> ## Scaling
>
> There are two main types of scaling:
>
> ### 1. Horizontal Scaling (Scaling Out / In)
> Horizontal scaling means adding or removing instances of an application instead of making a single instance more powerful. You scale by running more Pods, containers, or servers.
> 
> Example:
> 
> Instead of running one API server, you run multiple instances and distribute traffic between them using a load balancer.
>
> Adding a worker node to a Kubernetes cluster is horizontal scaling of the cluster's compute capacity:
> - **Adding nodes** = horizontal scaling of cluster infrastructure.
> - **Adding Pods/instances** = horizontal scaling of an application.
> 
> #### Characteristics of horizontal scaling:
> 
> - Improves fault tolerance
> - Can reduce the impact of a single instance failure
> - Can scale across multiple machines
> - Usually requires load balancing
> - Works especially well with stateless applications
>
> ### 2. Vertical Scaling (Scaling Up / Down)
> 
> Vertical scaling means increasing or decreasing the resources available to a single instance or machine. You make the existing instance more powerful by allocating more CPU, memory, or other resources.
> 
> Examples:
> 
> - Increasing CPU from 2 cores → 8 cores
> - Increasing RAM from 4 GB → 16 GB
> - Increasing a container's CPU or memory limits
> 
> Example scenario:
> 
> You have one database server:
> - CPU: 2 cores
> - RAM: 4 GB
> Traffic grows, so you upgrade the machine:
> - CPU: 8 cores
> - RAM: 32 GB
> The database is still running as one instance, but with more resources.
> 
> #### Characteristics of vertical scaling:
> 
> - Simple to implement
> - Often requires fewer application changes
> - Limited by the maximum resources of the machine or platform
> - May require a restart or downtime, depending on the platform
> - Can create a single point of failure when only one instance exists
> 
> **Important:** Horizontal and vertical scaling are not mutually exclusive. A system can use both—for example, each application instance can be given more CPU/RAM (vertical), while additional instances are also added (horizontal).

> ### What is the purpose of horizontal scaling if the underlying hardware resources do not increase? If a server can handle only a fixed number of requests, why would running more application instances improve performance?
> Your question is correct if all instances run on the same machine and the machine is already fully utilized. In that case, adding more instances usually gives little or no benefit.
> Suppose you have:
> -	1 server
> -	8 CPU cores
> -	16 GB RAM
> One application instance can handle 1000 requests/second. If that instance is already using all 8 cores, then 2 instances still around 1000 req/s because the hardware is the bottleneck.
> Now suppose Kubernetes runs your application on 4 nodes:
> -	Node 1 → Instance A
> -	Node 2 → Instance B
> -	Node 3 → Instance C
> -	Node 4 → Instance D
> Each node can handle 1000 requests/second. Now 4 instances can handle 4000 req/s because the workload is distributed across multiple machines. This is the main purpose of horizontal scaling.
>
> ### Another important reason: High Availability
> Even if capacity does not increase much, **multiple instances provide redundancy.**
>
> Suppose you have 3 instances, Instance A, Instance B and Instance C. If Instance B crashes, Instance A and Instance C still there. So the application remains available.
> With only one instance:
> Instance A crashes → service unavailable
>
> This is one of the biggest reasons Kubernetes runs multiple Pods.
>
> ### Summery:
> Why increase instances if hardware resources are limited?
> -	If the same hardware is already fully utilized, adding instances usually does not increase capacity.
> -	Horizontal scaling becomes useful when workload can be distributed across multiple CPU cores or multiple machines.
> -	Multiple instances also improve availability and fault tolerance because the application continues running even if some instances fail.
> -	In Kubernetes, scaling replicas is often used for both increased throughput and higher reliability. 

## Kubernetes Objects:

Most Kubernetes resources are represented as API objects defined in YAML. Examples:

- Pods
- Deployments
- Services
- ConfigMaps
- Secrets
- PersistentVolumes

These objects describe the desired state of the system, not the commands that should be executed to achieve it.