# Scalable & Resilient Microservices with MicroK8s
## Technical Report

---

## 1. Project Overview

This project demonstrates a cloud-native microservices deployment using MicroK8s on a local virtual machine. The system is composed of a stateless REST API application deployed as a containerized pod, coordinated by Kubernetes. The application is managed through a complete local DevOps pipeline — from Docker image building, to pushing to a private local registry, to deployment on a single-node MicroK8s cluster.

The system is composed of the following Kubernetes resources:

| Resource | Name | Purpose |
|---|---|---|
| Deployment | myapp | Manages application pod replicas |
| Service | myapp-service | Internal load balancing (ClusterIP) |
| Ingress | myapp-ingress | Routes external traffic to the service |
| HPA | myapp-hpa | Automatic horizontal scaling based on CPU |
| ResourceQuota | myapp-quota | Enforces hard resource limits at namespace level |

---

## 2. Application Node

The application is a stateless REST API built with Python Flask, running inside a python:3.12-slim container, exposed on port 5000. It was deliberately designed as stateless — meaning no session data, no database, and no persistent storage — so that any pod replica can handle any request interchangeably. This is the foundation that makes horizontal scaling and load balancing possible.

The service exposes two endpoints:

| Endpoint | Method | Purpose |
|---|---|---|
| / | GET | Returns a JSON response with a welcome message and the pod hostname |
| /health | GET | Returns {"status": "ok"} used by Kubernetes liveness and readiness probes |

The / endpoint returns the pod hostname in its response, which allows traffic distribution across replicas to be visually verified during load balancing demonstrations.

Technical details:
- Base image: python:3.12-slim
- Framework: Flask 3.0.0
- Exposed port: 5000
- Environment: No external dependencies, fully self-contained

---

## 3. Containerization & Image Workflow

The application is packaged using a Dockerfile. The python:3.12-slim base image was chosen over the full python:3.12 image for the following reasons:

- Size: ~150MB vs ~900MB for the full image
- Security: Smaller attack surface with fewer pre-installed packages
- Performance: Faster image pulls from the local registry
- Resource efficiency: Important on constrained VM hardware

The image workflow follows a complete local DevOps pipeline:

1. Build the image locally with Docker: docker build -t myapp:latest .
2. Tag it for the local MicroK8s registry: docker tag myapp:latest localhost:32000/myapp:latest
3. Push to the local registry: docker push localhost:32000/myapp:latest
4. MicroK8s pulls from localhost:32000 during pod creation

Using a local private registry rather than Docker Hub eliminates external network dependency during deployment, speeds up pod startup, and mirrors real production workflows where organizations maintain private registries for security and version control.

---

## 4. Kubernetes Architecture

### Deployment
The Deployment resource manages the application pod replicas. It defines the container image, resource constraints, and health probes. The replica count is managed dynamically by the HPA based on CPU load.

### Service
A ClusterIP Service named myapp-service provides stable internal DNS and load balancing across pod replicas. It listens on port 80 and forwards traffic to port 5000 on the pods. The ClusterIP type was chosen because external access is handled exclusively through the Ingress controller.

### Ingress
An NGINX Ingress controller (deployed via Helm with MetalLB providing a stable external IP at 10.0.2.100) routes external HTTP traffic from myapp.local to myapp-service. NGINX was chosen over the default MicroK8s Traefik controller due to stability issues encountered during deployment — Traefik repeatedly failed to initialize due to Gateway API CRD timeout errors on the constrained VM hardware.

### Horizontal Pod Autoscaler (HPA)
The HPA monitors CPU utilization across all pod replicas and automatically adjusts the replica count:

| Setting | Value |
|---|---|
| Minimum replicas | 2 |
| Maximum replicas | 5 |
| CPU target | 50% utilization |

Under normal conditions (low CPU), the HPA maintains 2 replicas. Under high load (CPU > 50%), it scales up to a maximum of 5 replicas automatically.

### Self-Healing
Liveness and Readiness probes are defined on each container, both targeting the /health endpoint:

- Readiness probe: prevents traffic from reaching a pod until it passes the health check, ensuring no requests are sent to an initializing container
- Liveness probe: continuously monitors pod health and triggers an automatic restart if the container becomes unresponsive

During testing, manually deleting a running pod demonstrated that Kubernetes immediately spawned a replacement, and traffic continued flowing to the remaining pods without any interruption or downtime.

### Resource Constraints
Resource requests and limits are defined on each container to ensure stable operation on constrained hardware:

| Resource | Request | Limit |
|---|---|---|
| CPU | 100m (0.1 core) | 250m (0.25 core) |
| Memory | 64Mi | 128Mi |

- Requests guarantee the pod will be scheduled on a node with sufficient resources
- Limits prevent any single pod from monopolizing CPU or memory on the single-node cluster

---

## 5. Resource Governance

### Problem Encountered During Demonstration

During the live demonstration, the professor requested a manual scale to 10 replicas using the following command:

kubectl scale deployment myapp --replicas=10

Kubernetes accepted the command and began creating pods beyond the HPA configured maxReplicas of 5. This revealed a critical gap in the original design — the HPA only governs automatic scaling and does not prevent manual operations that exceed its limits. As a result, pods were created beyond the intended maximum, which could potentially destabilize the cluster by consuming excessive CPU and memory resources on the single-node VM.

### Solution

To address this vulnerability, a ResourceQuota was implemented at the namespace level:

| Resource | Hard Limit |
|---|---|
| Pods | 6 |
| CPU requests | 600m |
| CPU limits | 1500m |
| Memory requests | 384Mi |
| Memory limits | 768Mi |

When the same scale command was run again after applying the quota, Kubernetes rejected any pod creation that would exceed the hard limit of 6 pods. Combined with the HPA which scales back down to 2 during low CPU periods, the system now has two layers of protection:

1. HPA: automatically manages replica count between 2 and 5 based on CPU load
2. ResourceQuota: enforces a hard ceiling of 6 pods regardless of manual commands

This mirrors real production environments where platform teams enforce namespace quotas to prevent any single application from consuming excessive cluster resources.

---

## 6. Hardware Constraints & Lessons Learned

The project was deployed on a QEMU virtual machine running Ubuntu 24.04 LTS with 4 vCPUs and 6GB RAM. Several challenges arose during development:

- Single CPU instability: Initially running MicroK8s on 1 vCPU caused frequent API server crashes under load. Increasing to 4 vCPUs resolved this completely.
- Ingress controller failures: The default MicroK8s Traefik ingress controller repeatedly timed out during initialization due to Gateway API CRD download failures on the slow VM network. Resolved by deploying NGINX ingress via Helm combined with MetalLB for external IP assignment.
- HPA scaling during reboot: After VM reboots, temporary CPU spikes caused the HPA to scale up to maximum replicas. Mitigated by setting a sensible maxReplicas value and implementing the ResourceQuota as an additional safety net.
- Resource right-sizing: Conservative CPU and memory limits were essential to maintain stability while running multiple pods on a single-node cluster. The chosen values (100m request, 250m limit per pod) allowed stable operation of 3-5 simultaneous replicas without destabilizing the node.

These challenges reinforced the importance of proper resource governance in Kubernetes — both at the pod level (requests/limits) and at the namespace level (ResourceQuota) — and highlighted the real-world value of the HPA ability to scale down automatically during low load periods.

---

Source code and Kubernetes manifests available at: https://github.com/jefftamatthew/microk8s-project
