# Scalable & Resilient Microservices with MicroK8s
## Technical Report

---

## 1. System Architecture & Design Choices

### Application
The application is a stateless REST API built with Python Flask. It exposes two endpoints: a main endpoint (`/`) that returns a JSON response including the pod hostname, and a health endpoint (`/health`) used by Kubernetes probes. The stateless design was chosen deliberately — it means any pod can handle any request, making horizontal scaling straightforward with no session affinity required.

### Containerization
The application is packaged using a `python:3.12-slim` base image, chosen for its minimal footprint (~150MB vs ~900MB for the full Python image). This reduces attack surface, speeds up image pulls from the local registry, and conserves disk space — important on constrained hardware.

### Local Registry
Rather than pulling from Docker Hub, images are pushed to MicroK8s's built-in private registry at `localhost:32000`. This eliminates external network dependency during deployment, speeds up pod startup, and mirrors real production workflows where organizations maintain private registries for security and control.

### Ingress
Traffic routing is handled by the NGINX ingress controller (deployed via Helm). An Ingress resource routes external HTTP requests to `myapp.local` through to the internal ClusterIP service, which then distributes traffic across pod replicas. This separation of concerns — Ingress for routing, Service for discovery, Deployment for compute — follows Kubernetes best practices.

---

## 2. Core Features Implementation

### Horizontal Scaling
The Deployment is configured with multiple replicas managed by a Horizontal Pod Autoscaler (HPA). The HPA monitors CPU utilization and scales between a defined minimum and maximum replica count automatically. Manual scaling via `kubectl scale` was also demonstrated, showing immediate replica adjustment without downtime.

### Self-Healing Architecture
Liveness and Readiness probes are defined on each container, both targeting the `/health` endpoint:
- **Readiness probe** — prevents traffic from reaching a pod until it is fully initialized
- **Liveness probe** — continuously monitors pod health and triggers automatic restart if the container becomes unresponsive

During testing, deleting a running pod demonstrated that Kubernetes immediately spawned a replacement, and traffic continued flowing to the remaining pods without interruption.

### Resource Constraints
Given the limited hardware (2 vCPU, 4GB RAM), resource requests and limits were carefully defined:
- CPU request: `100m` (0.1 core) — guarantees baseline scheduling
- CPU limit: `250m` (0.25 core) — prevents any single pod from monopolizing the CPU
- Memory request: `64Mi` — guaranteed allocation
- Memory limit: `128Mi` — prevents memory leaks from destabilizing the node

These conservative values allowed stable operation of multiple pods on a single-node cluster.

---

## 3. Hardware Constraints & Lessons Learned

The project was deployed on a QEMU virtual machine running Ubuntu 24.04 LTS with 4 vCPUs and 6GB RAM. Several challenges arose from this constrained environment:

- **API server instability** — Running MicroK8s on 1 vCPU caused frequent API server crashes. Increasing to 4 vCPUs and 6GB RAM resolved this.
- **Replica limits** — Attempting to run 5+ replicas simultaneously overwhelmed the VM. Keeping replicas at 2-3 maintained stability while still demonstrating scaling concepts.
- **Image pull timeouts** — Some MicroK8s addon installations timed out due to slow network conditions inside the VM. The local registry mitigated this for application deployments.

These constraints reinforced the importance of right-sizing resource requests and limits in Kubernetes manifests, and highlighted why the HPA's ability to scale down during low load is valuable in resource-constrained environments.
