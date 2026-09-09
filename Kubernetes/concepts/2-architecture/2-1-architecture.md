

## cloud-controller-manager:

The cloud-controller-manager (CCM) integrates Kubernetes with a cloud provider's APIs and manages cloud-specific functionality, such as:

- Load balancers
- Cloud storage integration
- Node lifecycle
- Network routes

It is commonly used with cloud providers such as:

- AWS
- GCP
- Azure

The cloud-controller-manager is **optional** and is generally not needed for bare-metal clusters unless a cloud provider integration is being used.

---
