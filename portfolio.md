# Three-Tier Application Deployment on AWS EKS

**Production-Style Cloud-Native Application Deployment on Amazon Elastic Kubernetes Service**

---

**Prepared by:** Anwarullah Khan Nuzair — Cloud & DevOps Engineer  
**Email:** myselfnuzairkhan@gmail.com  
**LinkedIn:** https://www.linkedin.com/in/anwarullah-khan-nuzair  
**GitHub:** https://github.com/Nuzairkhan

---

## Project Overview

This portfolio covers a hands-on deployment of a three-tier application on Amazon EKS — from cluster provisioning through containerization, Kubernetes configuration, networking, persistent storage, and the troubleshooting that came with getting it all working together.

The point wasn't just to deploy containers. It was to work through the complete engineering lifecycle: provision the infrastructure, build and push images, wire up Kubernetes resources, debug what breaks, and then clean everything up responsibly when done.

---

## Technology Stack

| Domain | Technologies |
|--------|-------------|
| Cloud Platform | AWS |
| Container Platform | Docker |
| Container Orchestration | Kubernetes, Amazon EKS |
| Image Registry | Amazon ECR |
| Compute | Amazon EC2 |
| Networking | VPC, Security Groups, Application Load Balancer, Ingress |
| Storage | Amazon EBS, Persistent Volumes, Persistent Volume Claims |
| Identity & Access | AWS IAM |
| Package Management | Helm |
| Application Stack | React, Node.js, MongoDB |
| Operating System | Linux |
| Version Control | Git, GitHub |

---

## Document Information

| Item | Value |
|------|-------|
| Version | 1.0 |
| Status | Production Portfolio |
| Document Type | Engineering Deployment Documentation |
| Last Updated | July 2026 |

---

---

## Chapter 2 — Executive Summary

### Background

I picked up this three-tier deployment challenge because it covered the full stack — not just spinning up a single container, but deploying a complete frontend, backend, and database, wiring them together through Kubernetes networking, and exposing everything through AWS. That's the kind of thing that looks simple from the outside and gets complicated fast in practice.

Going in, I had some AWS and DevOps background, but EKS was new to me. I understood the individual pieces — Docker, basic Kubernetes concepts, AWS services — but hadn't put them together in a real deployment. That's what this project fixed.

### The Problem This Solves

Managing a multi-service application without an orchestration platform gets messy quickly. You end up manually coordinating deployments across services, dealing with environment inconsistencies between machines, and handling failures by hand. The specific problems this deployment was designed to address:

- Consistent deployment across all three application tiers
- Reliable communication between frontend, backend, and database
- Persistent storage for MongoDB so data survives container restarts
- A single, controlled public entry point for external traffic
- Automatic recovery from failures without manual intervention

### What I Set Out to Do

Deploy a production-style Kubernetes environment on Amazon EKS, validate the complete frontend → backend → database flow end to end, and understand how the pieces actually connect — not just in theory, but by working through the issues that come up in a real deployment.

Specific outcomes:
- Deploy a multi-tier application using native Kubernetes resources
- Containerize each application component for consistent runtime behavior
- Use Amazon EKS as the managed Kubernetes control plane
- Store images in Amazon ECR
- Configure internal service discovery using Kubernetes DNS
- Set up persistent storage for MongoDB through PV/PVC backed by EBS
- Route external traffic through an AWS Application Load Balancer via Kubernetes Ingress
- Validate the complete flow from browser request to database write
- Clean up AWS resources after testing

### What the Solution Looks Like

The application runs as a standard three-tier setup. Each tier is an independent Kubernetes workload:

| Tier | Implementation | Purpose |
|------|---------------|---------|
| Presentation | React | User interface |
| Application | Node.js REST API | Business logic and request handling |
| Data | MongoDB | Persistent storage |

External traffic comes in through an AWS Application Load Balancer and gets routed by a Kubernetes Ingress resource. Services communicate internally using Kubernetes DNS. MongoDB's data persists on EBS-backed Persistent Volumes, independent of Pod lifecycle.

The end result: opening the ALB URL in a browser loaded the React frontend, and adding a task through the UI stored it in MongoDB — the complete flow working end to end.

### What's Intentionally Out of Scope

The following were deliberately excluded. They'd matter in production, but the goal here was to understand the deployment before layering automation on top:
- CI/CD automation
- Infrastructure as Code
- Multi-region deployment
- Production monitoring platforms
- GitOps workflows

### Was It Successful?

Yes. Specifically:
- EKS cluster provisioned and running
- Frontend, backend, and MongoDB deployed and healthy
- Inter-service communication working through Kubernetes Services
- External traffic routed through ALB via Ingress
- MongoDB data persisted using Persistent Volumes
- End-to-end flow verified from browser to database
- All AWS resources decommissioned after testing

This is a portfolio project. The architecture reflects real patterns used in enterprise Kubernetes deployments but is intentionally scoped to a single region, manually applied manifests, and basic monitoring. What it shows is the ability to work through a complete cloud-native deployment, debug real infrastructure issues across multiple layers, and understand how the components connect.

---

## Chapter 3 — Cloud Architecture

### Overview

The deployment uses a three-tier architecture on Amazon EKS. Each application tier — frontend, backend, and database — runs as an independent Kubernetes workload. Internal communication goes through Kubernetes-native networking; external access goes through AWS infrastructure.

The architecture came from the reference project. My job was implementing it correctly on EKS, understanding how the components actually interact, and working through the configuration and troubleshooting required to get it running.

### Architectural Principles

| Principle | Decision | Rationale |
|-----------|----------|-----------|
| Separation of Concerns | Independent frontend, backend, and database tiers | Simplifies maintenance; each tier has its own lifecycle |
| Containerization | Docker images for every component | Consistent, reproducible deployments |
| Managed Control Plane | Amazon EKS | Eliminates control plane administration overhead |
| Internal Service Discovery | Kubernetes Services | Decouples workloads from Pod IP addresses |
| Persistent Storage | PV + PVC | MongoDB data survives Pod restarts |
| External Traffic | ALB + Kubernetes Ingress | Single HTTP entry point with path-based routing |

### How Components Interact

**External access:** Users reach the application through the ALB DNS endpoint. The ALB is the only public entry point — it forwards HTTP traffic into the cluster through the Ingress resource.

**Traffic routing:** I used path-based routing rather than host-based routing. Since the deployment didn't need a custom domain, routing directly on the ALB DNS name was simpler and sufficient.

| Request Path | Destination |
|-------------|-------------|
| / | Frontend Service |
| /api | Backend Service |

**Frontend:** The React frontend uses relative API paths — `/api/tasks` — so requests resolve through the Ingress controller to the backend Service. No hardcoded backend URL in the compiled bundle.

**Backend:** Runs as a Kubernetes Deployment with two replicas. The backend Service distributes incoming traffic between them transparently.

**Database:** The backend connects to MongoDB through Kubernetes internal DNS at `mongodb-svc:27017`. MongoDB is never exposed outside the cluster.

**Persistent storage:** MongoDB's storage is managed through PV/PVC backed by Amazon EBS. If the MongoDB Pod restarts or gets rescheduled, it reattaches to the same volume — the data is still there.

### Traffic Flow

External request:
```
Browser → AWS Application Load Balancer → Kubernetes Ingress → Frontend Service → React Frontend
```

API request:
```
React App → /api → Ingress → Backend Service → Node.js API → MongoDB Service → MongoDB → Persistent Volume
```

### Kubernetes Resources

| Resource | Role |
|----------|------|
| Namespace | Logical isolation for project resources |
| Deployment | Desired application state management |
| ReplicaSet | Maintains required Pod count |
| Pod | Runs application containers |
| Service | Stable internal networking |
| Ingress | External HTTP routing |
| Secret | MongoDB credentials |
| Persistent Volume | Durable database storage |
| Persistent Volume Claim | Storage allocation abstraction |

### Key Design Decisions

**Managed Kubernetes:** EKS handles the control plane — API server, etcd, scheduler — so the focus stays on workloads rather than infrastructure maintenance.

**ClusterIP Services:** All Services are internal-only. The ALB is the only externally reachable component. Internal traffic never leaves the Kubernetes network.

**Path-based routing:** One ALB endpoint serves both frontend and API traffic. No custom domain needed for functional validation.

**Stateless vs. stateful:** Frontend and backend Pods can be freely recreated without side effects. MongoDB requires persistent storage — the deployment treats them differently by design.

**Backend replicas:** Two backend Pods means one failing doesn't take down the API. Rolling updates happen without downtime.

### Validation

The architecture was verified through end-to-end functional testing:
- Frontend accessible through the ALB
- Ingress routing traffic correctly by path
- Backend Services resolving through Kubernetes DNS
- MongoDB connectivity established from the application layer
- Persistent storage retaining data across Pod lifecycle events
- Stable communication across all three tiers

---

## Chapter 4 — Infrastructure Design

### Overview

The infrastructure runs on AWS managed services rather than self-managed components. The goal was straightforward: let AWS handle the heavy lifting on things like the Kubernetes control plane and networking, so I could stay focused on deploying and validating the application.

The approach was to set up a dedicated Ubuntu EC2 instance first as a management environment, then use it for everything else — cluster provisioning, ECR operations, and Kubernetes resource management — over SSH throughout the project.

### Infrastructure Components

| Component | AWS Service | Role |
|-----------|-------------|------|
| Identity | AWS IAM | Authentication and authorization |
| Management Compute | Amazon EC2 (t2.micro) | Cluster administration host |
| Container Platform | Amazon EKS | Managed Kubernetes control plane |
| Worker Nodes | EC2 Managed Node Group (t2.medium) | Execute Kubernetes workloads |
| Image Registry | Amazon ECR | Store Docker images |
| Load Balancing | AWS Application Load Balancer | Public HTTP entry point |
| Persistent Storage | Amazon EBS | Durable storage for MongoDB |
| Networking | Amazon VPC | Network isolation |

### IAM Setup

Before provisioning anything else, I created a dedicated IAM user named `eks-admin` with `AdministratorAccess` attached. This gave the management environment the permissions needed to provision EKS, ECR, EC2, IAM roles, and the load balancer controller without hitting authorization errors at each step.

The AWS Load Balancer Controller specifically needed its own IAM Role associated with a Kubernetes Service Account — the IRSA pattern. Rather than giving the controller broad permissions, it gets only what it needs to manage load balancers. This distinction mattered during deployment: the IAM failure we hit (`missing elasticloadbalancing:DescribeListenerAttributes`) was a specific missing permission, not a broad misconfiguration.

IAM covered:
- Administrative access for cluster provisioning
- EKS cluster and node group permissions
- Worker node permissions for ECR image pulls
- AWS Load Balancer Controller permissions via IRSA
- OIDC provider association for service account binding

### EC2 Management Instance

A dedicated Ubuntu EC2 instance (t2.micro) ran everything — cluster creation, image builds, kubectl operations, Helm installations, manifest deployments. It's purely an administration host; no application workloads run on it.

Using a dedicated EC2 instance rather than my local machine kept the environment consistent and avoided local tooling version issues or AWS credential configuration problems.

Tools installed:
- AWS CLI
- kubectl
- eksctl
- Docker
- Helm

The AWS Console ran alongside for visual verification: EKS cluster state, load balancer provisioning, target group health, security group configuration, IAM role attachments, ECR repositories, EBS volumes.

### VPC and Networking

The VPC wasn't created manually. Running `eksctl create cluster` automatically provisioned the VPC, subnets, Internet Gateway, and route tables. That's one of the advantages of eksctl — the networking infrastructure comes up correctly without having to configure each component individually.

Cluster deployed in `us-west-2`.

| Subnet Type | Purpose |
|-------------|---------|
| Public Subnets | Application Load Balancer |
| Worker Node Subnets | Kubernetes Worker Nodes |

Worker nodes sit off the public internet. The ALB is in the public subnets and forwards traffic to the nodes.

### Security Groups

| Resource | Inbound | Purpose |
|----------|---------|---------|
| Management EC2 | SSH (port 22) | Administration access only |
| Application Load Balancer | HTTP (port 80) | Public application traffic |
| Worker Nodes | ALB + control plane traffic | No direct public internet exposure |

Pods are never directly reachable from outside. All external traffic flows through ALB → Ingress → Service → Pod.

### EKS Cluster

```bash
eksctl create cluster \
  --name three-tier-cluster \
  --region us-west-2 \
  --node-type t2.medium \
  --nodes-min 2 \
  --nodes-max 2
```

This created the managed Kubernetes control plane, a managed node group with two t2.medium worker nodes, and the supporting VPC and networking infrastructure. eksctl handled control plane setup, node registration, and networking — leaving a working cluster ready for workloads.

### Worker Nodes

| Configuration | Value |
|---------------|-------|
| Node Count | 2 |
| Instance Type | t2.medium |
| Management | EKS Managed Node Group |
| OS | Amazon Linux (managed by EKS) |
| Region | us-west-2 |

Two worker nodes give basic redundancy — if one goes down, Kubernetes reschedules Pods on the other. The two backend replicas can land on separate nodes, which improves availability.

### Amazon ECR

Two private ECR repositories stored the application images:

| Repository | Purpose |
|------------|---------|
| Frontend | React application image |
| Backend | Node.js API image |

Worker nodes pull images from ECR using IAM-based authentication — no manual credential configuration needed. Separate repositories mean frontend and backend images can be versioned and updated independently.

### Persistent Storage

MongoDB needed storage that survives Pod restarts. The PV/PVC workflow handled this:
- A Persistent Volume Claim defined the storage requirement
- Kubernetes bound the PVC to an available Persistent Volume
- The PV was backed by Amazon EBS

The EBS volume was created through this Kubernetes workflow rather than manually in the AWS Console. Storage lifecycle stays tied to the Kubernetes configuration rather than requiring separate manual steps in AWS.

### Design Rationale

| Decision | Rationale |
|----------|-----------|
| Managed Kubernetes (EKS) | Avoid control plane administration overhead |
| eksctl for provisioning | Automates VPC, subnet, and node group creation |
| Dedicated management EC2 | Consistent tooling environment, no local dependency |
| Amazon ECR | Native EKS integration, no external registry auth needed |
| ALB Ingress | Single public endpoint with path-based routing |
| ClusterIP Services | Internal-only communication, no unnecessary exposure |
| Persistent Volumes | MongoDB data survives Pod recreation |
| Two Worker Nodes | Basic redundancy and scheduling flexibility |
| IAM least privilege | Controller-specific roles via IRSA |

### Infrastructure Validation

The infrastructure was operational once:
- EKS cluster showing Active in us-west-2
- Both worker nodes in Ready state
- Docker images available in ECR
- Kubernetes workloads scheduled across nodes
- Application Load Balancer provisioned and reachable
- Internal service communication working via Kubernetes DNS
- MongoDB persistent storage bound and validated
- All AWS resources decommissioned after testing

---

## Chapter 5 — Technology Stack

### Overview

The application stack — React, Node.js, MongoDB — was already part of the challenge project. My focus wasn't on selecting these technologies but on understanding how to containerize them, wire them together through Kubernetes networking, and deploy the complete three-tier architecture on EKS.

The infrastructure and tooling choices around that stack — EKS, ECR, ALB, Helm, IAM — were driven by the AWS deployment context. Using managed AWS services was the natural path when everything was already running in AWS.

### Technology Evaluation

| Technology | Purpose | Why Selected |
|-----------|---------|-------------|
| Amazon EKS | Managed Kubernetes | Native AWS integration, managed control plane |
| Docker | Containerization | Industry standard, reproducible environments |
| Amazon ECR | Image Registry | Native EKS integration, same AWS environment |
| Amazon EC2 | Administration Host | Dedicated management environment |
| AWS IAM | Identity & Access | Fine-grained permission management |
| Application Load Balancer | External Traffic Routing | Native Kubernetes Ingress support |
| Helm | Kubernetes Package Manager | Used for AWS Load Balancer Controller installation |
| MongoDB | Database | Part of the project application stack |
| React.js | Frontend | Part of the project application stack |
| Node.js | Backend | Part of the project application stack |

### Cloud Platform — AWS

Everything in this deployment runs on AWS. EKS, ECR, ALB, EBS, IAM all integrate natively, which simplified deployment significantly. ECR authenticates with EKS automatically through IAM. The Load Balancer Controller provisions ALBs directly from Kubernetes Ingress resources. EBS volumes bind to Persistent Volume Claims without manual provisioning. The value wasn't any single service — it was that they work together without custom integration work.

### Container Runtime — Docker

The Dockerfiles were already in the project repository. My work was building images from those Dockerfiles, pushing to ECR, and debugging issues along the way — including the Dockerfile naming error in the backend (`Dockefile` instead of `Dockerfile`), which blocked the entire build until caught.

### Container Orchestration — Amazon EKS

I used eksctl for cluster creation rather than the AWS Console. The goal was to understand the CLI-based provisioning workflow — how cluster, node group, VPC, and networking all get created from a single command.

One thing this deployment made clear: a running EKS cluster with healthy Pods isn't enough on its own. IAM permissions, the Load Balancer Controller, Ingress configuration, Service port mappings, and application-level configuration all have to be correct simultaneously for the application to actually work end to end. The cluster is just the foundation.

### Container Registry — Amazon ECR

ECR was the natural choice because the cluster was already in AWS. Worker nodes pull images using IAM-based authentication — no separate registry credentials needed.

| Repository | Purpose |
|------------|---------|
| Frontend | React application image |
| Backend | Node.js API image |

### Networking

**AWS Application Load Balancer:** The single external entry point. The AWS Load Balancer Controller provisions and manages it directly from the Kubernetes Ingress resource.

| Request Path | Destination |
|-------------|-------------|
| / | Frontend Service |
| /api | Backend Service |

**Kubernetes Services:** All application Services use ClusterIP — internal only. Services provide stable DNS-based endpoints so Pods can communicate without depending on individual Pod IP addresses.

### Storage — Amazon EBS + Kubernetes PV/PVC

MongoDB needs storage that survives Pod restarts. The PVC defines the storage requirement, Kubernetes binds it to a Persistent Volume, and that volume is backed by Amazon EBS. If the MongoDB Pod restarts or gets rescheduled, it reattaches to the same volume.

### Identity — AWS IAM

IAM underpins every AWS service interaction. The `eks-admin` user had `AdministratorAccess` for provisioning. The Load Balancer Controller used a dedicated IAM Role via IRSA, scoped only to load balancer management permissions.

The IAM failure during deployment — `missing elasticloadbalancing:DescribeListenerAttributes` — was a direct reminder that IAM policies have to be exact. The controller was running, the Ingress was applied, but nothing happened on the AWS side until the missing permission was attached.

### Package Management — Helm

Helm was used specifically for installing the AWS Load Balancer Controller inside the cluster. The application itself was deployed using standard Kubernetes manifests. A single Helm command handles the chart download, configuration, and deployment rather than manually applying multiple manifest files in order.

### Application Stack

**React.js — Frontend:** The frontend uses relative API paths (`/api/tasks`) rather than environment variables. This matters because React resolves environment variables at build time, not at runtime — the backend API URL gets compiled into the JavaScript bundle when `npm run build` runs. Changing a Kubernetes environment variable after the image is built has no effect on the already-compiled frontend.

**Node.js — Backend:** The REST API listens on port 3500 and handles all communication between frontend and MongoDB. Two replicas run in parallel — the Kubernetes Service distributes traffic between them transparently.

**MongoDB — Database:** Stores application task data as a stateful workload with persistent storage managed through PV/PVC. The backend reaches it through Kubernetes internal DNS at `mongodb-svc:27017`. Never exposed outside the cluster.

### What This Stack Actually Taught Me

The biggest surprise wasn't any individual technology — it was how many layers had to work correctly simultaneously for a simple application to show up in a browser.

A working EKS cluster wasn't enough. Healthy Pods weren't enough. The IAM permissions, Load Balancer Controller, Ingress configuration, Service port mappings, frontend API paths, and persistent storage configuration all had to be correct at the same time. When something was wrong, the symptom often appeared at a completely different layer from the actual cause — a frontend that couldn't reach the backend turned out to be a React build-time configuration issue, not a Kubernetes networking problem.

That's the practical value of this stack — not just how each tool works individually, but how they interact as a system.

---

## Chapter 6 — Containerization

### Overview

Containerization was the first major step before anything could run on Kubernetes. Each application tier — React frontend, Node.js backend, and MongoDB — needed to be packaged as a Docker image before EKS could schedule it as a workload. All builds ran on the Ubuntu EC2 management instance.

The Dockerfiles were already in the project repository. My work was working through the build process, fixing the issues that came up, pushing images to ECR, and then referencing those images in the Kubernetes Deployment manifests.

### Why Containerization

Running three services across different machines without containers means dealing with dependency conflicts, environment inconsistencies, and manual configuration on every host. Docker solves this by packaging each service with everything it needs to run, so behavior is identical regardless of where the container executes.

For Kubernetes specifically, containers aren't optional — they're the fundamental unit that Kubernetes schedules, monitors, and manages. Everything in this deployment flows from that model.

| Challenge | How Docker Addresses It |
|-----------|------------------------|
| Environment inconsistency | Standardized runtime with bundled dependencies |
| Dependency conflicts | Each service runs in its own isolated container |
| Deployment repeatability | Immutable images produce identical runtime behavior |
| Portability | Images run on any Docker-compatible host |
| Scaling | New containers start from the same image instantly |

### Containerized Components

| Component | Runtime | Purpose |
|-----------|---------|---------|
| Frontend | React.js | Serves the user interface |
| Backend | Node.js | Handles REST API requests and business logic |
| Database | MongoDB | Persistent data storage — stateful workload |

Each tier runs in its own container, managed independently by Kubernetes. The backend can be updated or scaled without touching the frontend. MongoDB runs as a stateful workload while the application layers stay stateless.

### Build Environment Setup

All image builds happened on the Ubuntu EC2 management instance. Docker was installed and configured before starting the build process.

```bash
sudo apt-get update
sudo apt install docker.io
docker ps
sudo chown $USER /var/run/docker.sock
```

That last step matters in practice — without it, every Docker command requires `sudo`, which adds friction and can cause permission issues when commands are chained together.

### Container Build Workflow

**Phase 1 — Source Preparation:** The project source code was cloned onto the EC2 management instance. Each application tier was in its own directory, with its own Dockerfile alongside the application source and dependency definitions.

**Phase 2 — Image Build:** Docker builds ran from inside each service directory. The backend Dockerfile typo (`Dockefile` instead of `Dockerfile`) was caught here — Docker couldn't locate the build definition and exited. Finding the misnamed file and correcting it unblocked the build.

**Phase 3 — ECR Authentication:**

```bash
aws ecr get-login-password --region us-west-2 | docker login \
  --username AWS \
  --password-stdin \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com
```

A successful authentication returns:
```
Login Succeeded
```

ECR authentication is entirely IAM-based — no separate registry password needed.

**Phase 4 — Image Tagging:**

```bash
docker tag <local-image-name>:latest \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/<repository-name>:latest
```

**Phase 5 — Image Push to ECR:**

```bash
docker push \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/<repository-name>:latest
```

Separate pushes for frontend and backend. Images were verified in ECR before referencing them in Kubernetes Deployment manifests.

**Phase 6 — Kubernetes Deployment:** With images in ECR, Deployment manifests reference the full ECR URI:

```yaml
image: <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/frontend:latest
image: <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/backend:latest
```

Worker nodes pull images directly from ECR using IAM-based authentication — no manual credential configuration needed on the nodes.

### Frontend Container

The React frontend uses relative API paths (`/api/tasks`) rather than a hardcoded backend URL. The original configuration used a build-time environment variable (`REACT_APP_BACKEND_URL`), which was already compiled into the bundle by the time the container ran. Switching to a relative path let Ingress handle routing instead.

| Property | Value |
|----------|-------|
| Framework | React.js |
| Replicas | 1 |
| Image Source | Amazon ECR |

### Backend Container

The Node.js backend handles all REST API requests, processes business logic, and communicates with MongoDB.

| Property | Value |
|----------|-------|
| Framework | Node.js |
| Container Port | 3500 |
| Replicas | 2 |
| Image Source | Amazon ECR |

The backend port (3500) had to be correctly reflected in the Kubernetes Service `targetPort`. When there was a mismatch, frontend API requests failed even though backend Pods were healthy. Correcting `targetPort` to 3500 restored communication.

### Database Container

MongoDB runs as a containerized database but is treated differently from the stateless frontend and backend. It requires persistent storage that survives container replacement. The container itself is stateless — the data it writes is persisted externally via EBS and reattached when the container restarts.

MongoDB uses the official public Docker image. No custom build is required.

### Image Repository Strategy

Two separate private ECR repositories — one for frontend, one for backend. Keeping them separate enables:
- Independent image versioning per service
- Targeted deployments — updating the backend doesn't require rebuilding the frontend image
- Simplified rollback — each service can be rolled back to a previous image independently

### Immutable Image Model

Container images are treated as immutable artifacts. Running containers are never modified directly. To update:
1. Source code is updated in the repository
2. A new Docker image is built
3. The new image is pushed to ECR
4. The Kubernetes Deployment is updated to reference the new image
5. Kubernetes performs a rolling update

This ensures what runs in the cluster is always a known, versioned artifact.

### Lessons from Containerization

**Dockerfile naming is exact.** Docker looks for a file named exactly `Dockerfile`. The backend project had `Dockefile` — one character off — which caused the build to fail immediately. An `ls -la` in the directory before building would have caught this.

**Container ports must match Service configuration.** The port a container exposes in its Dockerfile must match the `targetPort` in the Kubernetes Service. The backend listened on 3500, and the Service had to reflect that exactly. Mismatches cause silent failures where Pods appear healthy but traffic never reaches them.

**Images must be in ECR before applying Deployment manifests.** Kubernetes will attempt to pull the image as soon as a Deployment is applied. If the image isn't in ECR yet, the Pod enters `ImagePullBackOff`. The correct sequence is always build → push → deploy.

**Build-time vs. runtime configuration.** React environment variables are resolved at build time. Configuration that needs to work across environments should use relative paths or be handled at the routing layer.

---

## Chapter 7 — Kubernetes Design

### Overview

The Kubernetes manifests were part of the reference project. My work was deploying them into Amazon EKS, understanding how the resources relate to each other, configuring the environment correctly, and troubleshooting what didn't work.

This chapter documents the Kubernetes resource architecture as deployed, how the resources connect, and what it actually took to get them working together on AWS EKS.

### Namespace

The first step before deploying any application resources was creating a dedicated namespace:

```bash
kubectl create namespace workshop
```

All application resources were deployed here — Deployments, Services, Secrets, Persistent Volume Claims, and the Ingress. Keeping everything in a single namespace makes administration straightforward: you can see all application resources in one place and avoid accidental interaction with system-level Kubernetes resources in other namespaces.

Resources deployed in the `workshop` namespace:
- Frontend Deployment
- Backend Deployment
- MongoDB Deployment
- ClusterIP Services (frontend, backend, MongoDB)
- Kubernetes Secret (MongoDB credentials)
- Persistent Volume Claim
- Ingress Resource

### Applying Resources

```bash
kubectl apply -f .
```

This applied all manifest files in the current directory simultaneously. `apply` is declarative — if a resource already exists, Kubernetes updates it to match the manifest. If it doesn't exist, Kubernetes creates it. That made iterative debugging straightforward: fix the manifest, run `kubectl apply -f .` again, and Kubernetes updates only what changed.

### Deployment Strategy

| Deployment | Replicas | Purpose |
|------------|---------|---------|
| Frontend | 1 | React user interface |
| Backend | 2 | Node.js REST API |
| MongoDB | 1 | Persistent database |

The two backend replicas demonstrated how Kubernetes handles multiple instances of the same workload — the backend Service distributes traffic between Pods transparently, and if one Pod fails, traffic continues flowing to the other while Kubernetes schedules a replacement.

### ReplicaSets

Behind each Deployment, Kubernetes manages a ReplicaSet that maintains the required number of running Pods. If a Pod terminates unexpectedly — container crash, node failure, OOM kill — the ReplicaSet detects the gap between desired and actual state and schedules a replacement automatically. This happened during deployment validation without manual intervention. That's one of the more immediately practical aspects of Kubernetes.

### Kubernetes Services

All three application Services use ClusterIP — internal-only. Nothing is directly reachable from outside except through the ALB.

| Service | Port | Purpose |
|---------|------|---------|
| frontend | 3000 | React frontend |
| api | 8080 → 3500 | Backend API |
| mongodb-svc | 27017 | MongoDB database |

The backend Service port mapping (8080 → 3500) is worth noting. The Service accepts traffic on port 8080 and forwards it to the container's port 3500. That mismatch between external Service port and internal container port caused the frontend-to-backend communication failure during deployment. The Node.js application was listening on 3500, but the Service `targetPort` wasn't set correctly. Fixing that restored communication.

**Why ClusterIP?** ClusterIP Services are only reachable from within the cluster. External access comes exclusively through Ingress and ALB — no application Pod is directly reachable from the internet.

### Service Discovery

Workloads communicate through Kubernetes DNS rather than Pod IP addresses. The backend reaches MongoDB using:

```
mongodb-svc:27017
```

This DNS name resolves to the MongoDB Service's ClusterIP. If the MongoDB Pod is replaced and gets a new IP, the DNS name stays the same — the backend configuration doesn't need to change.

### Ingress Design

External HTTP traffic enters through a Kubernetes Ingress resource managed by the AWS Load Balancer Controller. The Ingress defines the routing rules:

| Request Path | Destination |
|-------------|-------------|
| / | Frontend Service |
| /api | Backend Service |

A single ALB serves both routes. One DNS endpoint for the entire application, with path-based routing handling the separation between frontend and API traffic.

The Ingress only works if the AWS Load Balancer Controller is correctly installed and authorized. During this deployment, the controller's IAM role was missing `elasticloadbalancing:DescribeListenerAttributes`, which prevented the ALB from provisioning. The Ingress resource existed in Kubernetes but had no external address until the IAM policy was corrected and the controller restarted.

### Secret Management

MongoDB credentials are stored in a Kubernetes Secret rather than hardcoded into application manifests or container images. The backend Deployment references the Secret through environment variables.

Keeping credentials in Secrets separates sensitive values from application configuration and makes credential rotation possible without rebuilding images.

> Note: Kubernetes Secrets are base64-encoded by default, not encrypted at rest unless cluster-level encryption is configured. For production, envelope encryption or an external secrets manager would add another layer of protection.

### Persistent Storage

MongoDB needs storage that survives Pod restarts:

```
MongoDB Pod → Persistent Volume Claim → Persistent Volume → Amazon EBS
```

The PVC defines what storage the application needs. Kubernetes binds the PVC to an available Persistent Volume. The PV is backed by an EBS block storage device that exists independently of the Pod lifecycle.

During deployment, the PVC remained in `Pending` state because the PV wasn't available when the claim was applied. Checking:

```bash
kubectl get pvc
```

Showed the claim waiting indefinitely. Applying storage resources in the correct order — PV before PVC — resolved it. Once bound, the MongoDB Pod scheduled successfully.

### Scaling

| Component | Replicas | Scaling Characteristics |
|-----------|---------|------------------------|
| Frontend | 1 | Horizontally scalable — stateless |
| Backend | 2 | Horizontally scalable — stateless |
| MongoDB | 1 | Stateful — requires storage coordination |

Scaling stateless workloads is straightforward: increase the replica count in the Deployment and Kubernetes schedules additional Pods. MongoDB scaling is more complex — adding replicas without proper MongoDB replication configuration would create multiple independent database instances rather than a coordinated cluster.

### Self-Healing

1. Pod fails — container crash, OOM kill, or node issue
2. ReplicaSet detects actual replica count is below desired
3. Kubernetes Scheduler selects an available node
4. New Pod is created and starts pulling its image
5. Service continues routing to healthy Pods during recovery
6. Once the new Pod passes health checks, it joins the Service load balancing pool

This behavior was observable during deployment validation — terminating a Pod resulted in Kubernetes scheduling a replacement automatically.

### Debugging Workflow

```bash
# Check worker nodes
kubectl get nodes

# Check Pod status across the namespace
kubectl get pods -n workshop

# Check Service configuration
kubectl get svc -n workshop

# Check PVC binding status
kubectl get pvc -n workshop

# Get detailed resource information and events
kubectl describe pod <pod-name> -n workshop
kubectl describe svc <service-name> -n workshop
kubectl describe pvc <pvc-name> -n workshop

# Apply updated manifests
kubectl apply -f .

# Check container logs
kubectl logs <pod-name> -n workshop

# Clean up all resources
kubectl delete -f .
```

The general debugging pattern:
```
Something isn't working
    ↓
kubectl get pods — identify unhealthy resource
    ↓
kubectl describe <resource> — read Events section
    ↓
Identify root cause — config, scheduling, networking, IAM
    ↓
Fix the manifest or configuration
    ↓
kubectl apply -f .
    ↓
Verify again
```

The `describe` command was particularly useful because the Events section shows what Kubernetes actually attempted — image pulls, scheduling decisions, probe failures, volume binding attempts. Most root causes were visible there before needing to check logs.

### Kubernetes Resource Summary

| Resource | Quantity |
|----------|---------|
| Namespace | 1 |
| Deployments | 3 |
| ReplicaSets | 3 |
| Pods | 4 |
| Services | 3 |
| Ingress | 1 |
| Secrets | 1 |
| Persistent Volumes | 1 |
| Persistent Volume Claims | 1 |

### The Biggest Kubernetes Learning

What stood out most was how interconnected everything is — and how a problem in one layer can appear as a symptom in a completely different layer.

You can have:
- EKS cluster running
- Pods in Running state
- Containers healthy

And still have:
- Frontend unable to reach backend (Service port mismatch)
- PVC stuck in Pending (resource ordering)
- ALB not provisioning (IAM permission missing)
- API requests going nowhere (React build-time configuration)

The deployment wasn't a single `kubectl apply` that produced a working application. It was an iterative process of checking each layer, identifying what was actually wrong, fixing it, and verifying again. Kubernetes abstracts a significant amount of infrastructure complexity — but that abstraction doesn't remove the need to understand what's happening underneath.

---

## Chapter 8 — Deployment Workflow

### Overview

This deployment wasn't automated through a CI/CD pipeline. It was a sequential, hands-on workflow performed entirely from the Ubuntu EC2 management instance — provisioning infrastructure, building container images, deploying Kubernetes resources, configuring AWS integrations, and validating the complete application flow step by step.

The implementation was completed iteratively rather than as a single uninterrupted run. Infrastructure provisioning, container preparation, Kubernetes configuration, AWS Load Balancer integration, and troubleshooting happened in stages until the complete frontend → backend → database flow was operational and verified in a browser.

### Deployment Phases

| Phase | Objective | Primary Tool |
|-------|-----------|-------------|
| Environment Preparation | Configure management host and credentials | EC2, AWS CLI |
| Cluster Provisioning | Create Kubernetes environment | eksctl |
| Image Build | Package application components | Docker |
| Image Distribution | Store images in registry | Amazon ECR |
| Resource Deployment | Deploy Kubernetes workloads | kubectl |
| Controller Installation | Enable ALB provisioning | Helm |
| Traffic Exposure | Publish application externally | AWS ALB + Ingress |
| Validation | Verify end-to-end functionality | Browser + kubectl |
| Cleanup | Remove AWS resources after testing | kubectl, AWS Console |

### Phase 1 — Environment Preparation

All provisioning, image building, and Kubernetes operations ran from a single Ubuntu EC2 instance throughout the project.

**Step 1 — IAM User and Credentials:** An IAM user named `eks-admin` was created with `AdministratorAccess` attached. Access keys were configured on the EC2 instance via `aws configure`.

**Step 2 — Launch Ubuntu EC2:** A dedicated Ubuntu EC2 instance (t2.micro) was launched. All subsequent steps were performed over SSH.

**Step 3 — AWS CLI Installation:**

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
aws configure
```

**Step 4 — Docker Installation:**

```bash
sudo apt-get update
sudo apt install docker.io
docker ps
sudo chown $USER /var/run/docker.sock
```

**Step 5 — kubectl Installation:**

```bash
kubectl version --short --client
```

**Step 6 — eksctl Installation:**

```bash
eksctl version
```

**Step 7 — Helm Installation:**

```bash
sudo snap install helm --classic
```

### Phase 2 — Amazon EKS Cluster Provisioning

```bash
eksctl create cluster \
  --name three-tier-cluster \
  --region us-west-2 \
  --node-type t2.medium \
  --nodes-min 2 \
  --nodes-max 2
```

| Parameter | Value |
|-----------|-------|
| Cluster Name | three-tier-cluster |
| Region | us-west-2 |
| Worker Node Type | t2.medium |
| Minimum Nodes | 2 |
| Maximum Nodes | 2 |

EKS provisioning was the longest single step — the managed control plane and worker nodes take several minutes to become operational. No application resources were deployed until this completed.

After provisioning, kubectl was connected to the cluster:

```bash
aws eks update-kubeconfig \
  --region us-west-2 \
  --name three-tier-cluster
```

Worker node registration:

```bash
kubectl get nodes
```

Expected output — both worker nodes showing Ready status:
```
NAME                                          STATUS   ROLES    AGE
ip-xxx-xxx-xxx-xxx.us-west-2.compute.internal   Ready    <none>   Xm
ip-xxx-xxx-xxx-xxx.us-west-2.compute.internal   Ready    <none>   Xm
```

### Phase 3 — Container Image Build

Source code was cloned onto the EC2 management instance. Docker builds ran from there.

The backend build failed initially because the Dockerfile was misnamed `Dockefile` instead of `Dockerfile`. Renaming the file and rebuilding resolved it:

```bash
mv Dockefile Dockerfile
docker build -t <backend-image-name> .
```

MongoDB used the official public Docker image — no custom build required.

### Phase 4 — Image Distribution to Amazon ECR

**ECR Authentication:**

```bash
aws ecr get-login-password --region us-west-2 | docker login \
  --username AWS \
  --password-stdin \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com
```

**Tag and Push:**

```bash
docker tag <local-image>:latest \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/<repository>:latest

docker push \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/<repository>:latest
```

Images were verified in ECR before referencing them in Deployment manifests.

### Phase 5 — Kubernetes Resource Deployment

```bash
kubectl create namespace workshop
kubectl apply -f .
```

Resources deployed:

| Resource | Purpose |
|----------|---------|
| Namespace | Resource isolation |
| Secret | MongoDB credentials |
| Persistent Volume | Durable storage |
| Persistent Volume Claim | Storage allocation |
| MongoDB Deployment | Database workload |
| Backend Deployment (×2) | REST API |
| Frontend Deployment | User interface |
| Services (×3) | Internal networking |
| Ingress | External routing |

The PVC remained in `Pending` state when storage resources weren't available at binding time. Understanding the dependency — PV must exist before PVC can bind, PVC must bind before MongoDB Pod can schedule — was one of the key debugging insights.

Post-deployment verification:

```bash
kubectl get pods -n workshop
kubectl get svc -n workshop
kubectl get pvc -n workshop
kubectl get ingress -n workshop
```

Expected Pod state:
```
NAME                                    READY   STATUS    RESTARTS
frontend-deployment-xxxxx               1/1     Running   0
backend-deployment-xxxxx                1/1     Running   0
backend-deployment-yyyyy                1/1     Running   0
mongodb-deployment-xxxxx                1/1     Running   0
```

### Phase 6 — AWS Load Balancer Controller

The AWS Load Balancer Controller is required for Kubernetes Ingress to provision an AWS Application Load Balancer. Without it, the Ingress resource exists in Kubernetes but no ALB gets created.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=three-tier-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

kubectl get deployment -n kube-system aws-load-balancer-controller

kubectl apply -f full_stack_lb.yaml
```

During this phase, the controller failed to provision the ALB because the IAM role was missing `elasticloadbalancing:DescribeListenerAttributes`. The Ingress existed in Kubernetes with no external address. Attaching the missing permission and restarting the controller resolved it:

```bash
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
```

### Phase 7 — Application Validation

Once the ALB DNS endpoint appeared on the Ingress resource, the application was tested end to end:

- Frontend accessible through ALB DNS endpoint
- React UI loaded in browser
- Task created through the UI
- Backend API processed the request
- Task stored in MongoDB
- Task visible on page refresh — confirming persistence
- No unexpected Pod restarts during testing

### Phase 8 — Infrastructure Cleanup

```bash
kubectl delete -f .
```

This removed all application resources and triggered cleanup of associated AWS resources — target groups and load balancers managed by the controller.

Remaining infrastructure — the EKS cluster, worker nodes, EC2 management instance, ECR repositories, and EBS volumes — was then decommissioned through the AWS Console.

Cleaning up in the correct order matters. Deleting the EKS cluster before removing Kubernetes-managed AWS resources like the ALB can leave orphaned resources that continue generating costs.

---

## Chapter 9 — Implementation & Operational Validation

### Overview

This chapter documents the operational commands, verification steps, and debugging workflow used throughout the deployment. The focus is on what was actually run — the kubectl commands, the checks at each stage, and how issues were identified and isolated.

The deployment was validated incrementally. Each phase was checked before moving to the next. When something broke, the scope of what could have caused it was already narrowed by what had already been verified.

### Phase 1 — Infrastructure Readiness

```bash
# Verify AWS CLI is configured and credentials work
aws sts get-caller-identity

# Verify Docker is running
docker ps

# Verify kubectl is installed
kubectl version --short --client

# Verify eksctl is installed
eksctl version

# Verify Helm is installed
helm version
```

Each tool had to be confirmed working before starting infrastructure provisioning. A broken tool mid-deployment is harder to debug than catching it upfront.

### Phase 2 — EKS Cluster Provisioning and Validation

```bash
eksctl create cluster \
  --name three-tier-cluster \
  --region us-west-2 \
  --node-type t2.medium \
  --nodes-min 2 \
  --nodes-max 2

aws eks update-kubeconfig \
  --region us-west-2 \
  --name three-tier-cluster

kubectl get nodes
```

Expected output — both worker nodes showing Ready:
```
NAME                                          STATUS   ROLES    AGE
ip-xxx-xxx-xxx-xxx.us-west-2.compute.internal   Ready    <none>   Xm
ip-xxx-xxx-xxx-xxx.us-west-2.compute.internal   Ready    <none>   Xm
```

Only after seeing both nodes in Ready state were application resources deployed.

### Phase 3 — Container Image Build and Push

```bash
# Navigate to service directory and build
docker build -t <image-name> .

# Verify image was created
docker images

# List directory contents to verify filenames
ls -la

# Rename the misnamed backend Dockerfile
mv Dockefile Dockerfile
docker build -t <backend-image-name> .

# ECR Authentication
aws ecr get-login-password --region us-west-2 | docker login \
  --username AWS \
  --password-stdin \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com

# Tag and Push
docker tag <local-image>:latest \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/<repository>:latest

docker push \
  <AWS_ACCOUNT_ID>.dkr.ecr.us-west-2.amazonaws.com/<repository>:latest
```

### Phase 4 — Kubernetes Resource Deployment

```bash
kubectl create namespace workshop
kubectl apply -f .

# Immediate post-deploy checks
kubectl get pods -n workshop
kubectl get svc -n workshop
kubectl get pvc -n workshop
```

Expected Pod output:
```
NAME                                    READY   STATUS    RESTARTS
frontend-deployment-xxxxx               1/1     Running   0
backend-deployment-xxxxx                1/1     Running   0
backend-deployment-yyyyy                1/1     Running   0
mongodb-deployment-xxxxx                1/1     Running   0
```

A Pod showing anything other than Running required investigation before proceeding:

```bash
kubectl describe pod <pod-name> -n workshop
```

The Events section in `kubectl describe` output was the most useful diagnostic — it shows what Kubernetes actually attempted.

### Phase 5 — Storage Validation

```bash
kubectl get pvc -n workshop
```

Expected output:
```
NAME              STATUS   VOLUME     CAPACITY   ACCESS MODES
mongodb-pvc       Bound    pv-xxxxx   Gi         RWO
```

When the PVC showed `Pending` instead of `Bound`:

```bash
kubectl describe pvc <pvc-name> -n workshop
```

Showed the claim waiting for a matching volume. Applying storage resources in the correct order — PV before PVC — resolved the binding.

### Phase 6 — Service and Networking Validation

```bash
kubectl get svc -n workshop
```

Expected output:
```
NAME           TYPE        CLUSTER-IP      PORT(S)
frontend       ClusterIP   10.x.x.x        3000/TCP
api            ClusterIP   10.x.x.x        8080/TCP
mongodb-svc    ClusterIP   10.x.x.x        27017/TCP
```

The backend Service port mismatch was identified by comparing the Service configuration against the actual container port:

```bash
kubectl describe svc api -n workshop
```

The output showed the `TargetPort` was not matching the port the Node.js application was listening on (3500). Correcting the Service manifest and reapplying:

```bash
kubectl apply -f .
```

### Phase 7 — AWS Load Balancer Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=three-tier-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

kubectl get deployment -n kube-system aws-load-balancer-controller

kubectl apply -f full_stack_lb.yaml

kubectl get ingress -n workshop
```

Initially the `ADDRESS` field was empty. Investigation through controller logs revealed an IAM authorization failure — the missing permission `elasticloadbalancing:DescribeListenerAttributes`.

After attaching the required IAM policy and restarting the controller:

```bash
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
```

Running `kubectl get ingress` again showed the ALB DNS address populated.

### Phase 8 — Application Validation

Validation sequence:
1. Opened the ALB DNS URL in a browser
2. React frontend loaded successfully
3. Added a task through the UI
4. Task appeared — confirming frontend → backend communication
5. Refreshed the page — task still visible, confirming MongoDB persistence

| Check | Result |
|-------|--------|
| EKS cluster operational | ✓ |
| Worker nodes in Ready state | ✓ |
| Images available in ECR | ✓ |
| All Pods in Running state | ✓ |
| PVC bound to Persistent Volume | ✓ |
| Services correctly configured | ✓ |
| ALB provisioned and reachable | ✓ |
| Ingress routing functional | ✓ |
| Frontend accessible through ALB | ✓ |
| Backend API responding | ✓ |
| MongoDB storing data | ✓ |
| Browser-to-database flow verified | ✓ |

### Debugging Command Reference

```bash
kubectl get nodes
kubectl get pods -n workshop
kubectl get svc -n workshop
kubectl get pvc -n workshop
kubectl get ingress -n workshop

kubectl describe pod <pod-name> -n workshop
kubectl describe svc <service-name> -n workshop
kubectl describe pvc <pvc-name> -n workshop
kubectl describe ingress <ingress-name> -n workshop

kubectl logs <pod-name> -n workshop
kubectl logs -n kube-system deployment/aws-load-balancer-controller

kubectl apply -f .
kubectl delete -f .
```

### Implementation Issues Resolved

| Issue | Layer | Resolution |
|-------|-------|------------|
| Dockerfile named Dockefile | Containerization | Renamed file, rebuilt image |
| PVC stuck in Pending | Kubernetes Storage | Applied PV before PVC |
| Backend Service port mismatch | Kubernetes Networking | Corrected targetPort to 3500 |
| React build-time env var | Application Config | Switched to relative path /api/tasks |
| Missing IAM permission | AWS Security | Attached policy, restarted controller |
| Ingress host-based routing | Networking | Removed host field, used path-based routing |

Each issue required checking a different layer. The debugging pattern was consistent: observe the symptom, identify the layer, check that layer specifically, fix, verify.

---

## Chapter 10 — Challenges, Root Cause Analysis & Engineering Learnings

Getting this deployment to a working state wasn't a straight line. Problems came up across containerization, storage, Kubernetes networking, IAM, and application configuration — sometimes in ways that weren't obvious from the surface symptoms. This chapter documents each issue honestly: what broke, what the investigation showed, and what fixed it.

Most debugging happened over SSH on the Ubuntu EC2 management instance, running kubectl, eksctl, the AWS CLI, and Docker from the terminal. The AWS Console ran alongside for verifying infrastructure state — load balancer status, IAM role attachments, ECR repositories, and target group health.

### Challenge 1 — Docker Image Build Failure

The backend image failed to build. Docker exited without producing an image, which left the ECR repository empty and gave Kubernetes nothing to pull.

The build error indicated that Docker couldn't locate a valid build definition. Checking the backend project directory revealed the file had been named `Dockefile` instead of `Dockerfile` — one character off.

Renaming the file and rebuilding resolved it. The image pushed to ECR cleanly after that.

It's a small mistake with a disproportionate effect — the entire pipeline stalls until the build produces a valid image. An `ls -la` in the directory before building would have caught this.

### Challenge 2 — Persistent Volume Claim Stuck in Pending

MongoDB wouldn't start. The Pod sat in `Pending` and the PVC never transitioned to `Bound`, which meant the database had no storage to attach to.

Investigation showed the PVC had been applied before a matching Persistent Volume was available for binding. Kubernetes held it in pending indefinitely, waiting for something to bind against.

Applying storage resources in the correct order and redeploying MongoDB resolved it. Once the PV was in place, the PVC bound immediately.

Stateful workloads have infrastructure dependencies that stateless ones don't. The PV has to exist before the PVC can bind — and the PVC has to be bound before the Pod can schedule. Getting that sequence wrong means the whole thing stalls silently.

### Challenge 3 — Frontend Couldn't Reach the Backend

The frontend loaded in the browser, but every API call failed. The backend Pods were running and healthy — so the problem wasn't with the application itself, it was somewhere between the Service definition and the container.

Comparing the application's expected listening port against the Kubernetes Service configuration showed the mismatch. The Node.js API was listening on port 3500, but the Service wasn't forwarding traffic to the matching target port.

Correcting `targetPort` to 3500 fixed it. This one is easy to overlook because everything appears healthy at the Pod level. Running Pods don't tell you whether the Service is routing traffic correctly — you have to check the port configuration against what the application actually exposes.

### Challenge 4 — React API Requests Going Nowhere

This took more time to understand than the others because the surface symptoms looked identical to a networking problem. The frontend was loading, the backend was running, the Service config was correct — but API requests were still failing.

The issue had nothing to do with Kubernetes. I had the backend URL configured through `REACT_APP_BACKEND_URL` in the frontend source. What I hadn't accounted for is that React resolves environment variables at build time, not at runtime. By the time the container is running, the compiled JavaScript bundle already has whatever value was present when `npm run build` executed. Updating a Kubernetes environment variable after the fact does nothing.

The fix was to stop using an environment variable for the URL entirely. I changed `taskServices.js` to use a relative path instead:

```javascript
const apiUrl = "/api/tasks";
```

The full file after the change:

```javascript
import axios from "axios";

const apiUrl = "/api/tasks";

export function getTasks() {
    return axios.get(apiUrl);
}

export function addTask(task) {
    return axios.post(apiUrl, task);
}

export function updateTask(id, task) {
    return axios.put(apiUrl + "/" + id, task);
}

export function deleteTask(id) {
    return axios.delete(apiUrl + "/" + id);
}
```

Verified in place:

```bash
ubuntu@ip-172-31-21-175:~/TWSThreeTierAppChallenge/Application-Code/frontend/src/services$ cat taskServices.js
```

With a relative path, the request goes to whatever host is serving the frontend — the ALB in this case — and Ingress handles routing `/api` traffic to the backend Service. No hardcoded URL, no build-time configuration dependency.

This was probably the most conceptually interesting issue in the whole deployment. It's not a Kubernetes problem — it's a React behavior that only becomes visible when you're deploying behind a reverse proxy or Ingress controller.

### Challenge 5 — AWS Load Balancer Controller Authorization Failure

After installing the AWS Load Balancer Controller and applying the Ingress resource, no ALB appeared. The Ingress sat without an external address and nothing showed up in the AWS Console under load balancers.

The controller itself was running — the problem was that it couldn't complete the AWS API calls it needed. Investigation of the controller error output identified an AWS authorization failure. The IAM role attached to the controller was missing a specific permission:

```
elasticloadbalancing:DescribeListenerAttributes
```

Attaching the required IAM policy and restarting the controller resolved it. The ALB provisioned within a few minutes.

IAM failures in Kubernetes on AWS tend to look like silent hangs rather than obvious errors. The Ingress resource exists, the controller is running, but nothing happens on the AWS side. Going straight to the controller's error output rather than rechecking Kubernetes configuration saved time here — the 403 pointed directly at the authorization layer.

### Challenge 6 — Ingress Routing Without a Domain

The initial Ingress manifest included hostname-based routing. Without a registered domain pointing at the ALB, host-based routing couldn't be validated — requests weren't resolving correctly.

The deployment didn't require a custom domain for functional validation, so the hostname dependency was removed entirely. With the host field gone, the ALB's DNS name became the direct entry point, and traffic was routed purely on path:

```
/       →  Frontend Service
/api    →  Backend Service
```

For a deployment scoped to functional testing rather than production exposure, path-based routing is simpler and avoids a dependency on external DNS configuration.

### What Actually Took the Longest

Across all six challenges, the ALB and IAM configuration consumed the most total time. The Dockerfile typo and port mismatch were quick once identified. The React environment variable issue required understanding a non-obvious build behavior, but the fix itself was straightforward once the root cause was clear.

The Load Balancer Controller authorization failure involved the most back-and-forth. The controller appeared healthy, the Ingress was applied correctly, but nothing happened on the AWS side. That kind of silent failure takes longer to debug because the problem isn't visible in Kubernetes at all — it lives in the IAM layer, and you only find it by reading the controller's own error output.

### Debugging Approach

A few things that helped consistently:

**Check Kubernetes events before changing anything.** `kubectl describe` on the affected resource usually points toward the right layer — whether the issue is scheduling, storage binding, networking, or something else.

**Isolate the layer first.** Kubernetes config problems, application problems, and AWS permission problems present differently. Treating them as the same thing wastes time.

**Change one thing at a time.** When multiple things are wrong simultaneously, fixing them in parallel makes it impossible to know what actually resolved the issue.

**Use the AWS Console alongside kubectl.** Some things are faster to verify visually — load balancer provisioning state, target group health, IAM role attachments. Terminal and console together give a more complete picture than either alone.

### Challenge Summary

| Challenge | Category | Severity | Status |
|-----------|----------|----------|--------|
| Dockerfile naming | Containerization | Medium | Resolved |
| PVC pending | Kubernetes Storage | High | Resolved |
| Service port mapping | Kubernetes Networking | High | Resolved |
| React environment variables | Application Configuration | Medium | Resolved |
| IAM authorization | AWS Security | Critical | Resolved |
| Ingress host configuration | Networking | Medium | Resolved |

---

## Chapter 11 — Security Architecture

### Overview

Security wasn't a separate phase in this deployment — it was part of how the infrastructure was set up from the beginning. IAM roles were scoped to specific workloads, application services were kept off the public internet, credentials were separated from application code, and administration was isolated to a dedicated host.

This is a learning environment, not a production system. The security decisions reflect that — they're practical and appropriate for the scope. Where production would require additional controls, those are called out rather than claimed as implemented.

One of the most concrete security lessons came from the IAM authorization failure during ALB provisioning — covered in detail below because it demonstrates how IAM directly affects infrastructure behavior, not just access control.

### Security Design Principles

| Principle | Implementation |
|-----------|---------------|
| Least Privilege | IAM roles scoped to specific workload requirements |
| Defense in Depth | Multiple security layers across AWS and Kubernetes |
| Network Isolation | Internal ClusterIP Services for all application components |
| Controlled Exposure | Single public ALB endpoint |
| Credential Protection | Kubernetes Secrets for MongoDB credentials |
| Resource Isolation | Dedicated namespace — workshop |
| Immutable Infrastructure | Docker images as deployment artifacts |
| Secure Administration | Dedicated EC2 management host |

### Identity & Access Management

IAM was the most operationally significant security component — not just as access control, but as a direct dependency for infrastructure functionality.

**Administrative Access:** An IAM user named `eks-admin` was created with `AdministratorAccess`. This user's credentials were configured on the EC2 management instance via `aws configure`.

**AWS Load Balancer Controller — IRSA:** Rather than giving the Load Balancer Controller broad node-level AWS permissions, a dedicated IAM role was created and associated specifically with the controller's Kubernetes Service Account.

First, the OIDC provider was associated with the cluster:

```bash
eksctl utils associate-iam-oidc-provider \
  --region=us-west-2 \
  --cluster=three-tier-cluster \
  --approve
```

Then the service account and its IAM role were provisioned:

```bash
eksctl create iamserviceaccount \
  --cluster=three-tier-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name=AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=... \
  --approve \
  --region=us-west-2
```

The controller authenticates to AWS using its own scoped role rather than inheriting permissions from the worker node. Worker nodes don't need ALB management permissions — only the controller does.

| Component | Permission Scope |
|-----------|-----------------|
| eks-admin user | AdministratorAccess for infrastructure provisioning |
| Worker Nodes | ECR image pull, basic EKS node operations |
| ALB Controller | ALB provisioning and management only via IRSA |
| Application Pods | No direct AWS access |

### Real Security Finding — IAM Blocked ALB Provisioning

The most concrete security lesson from this deployment was an IAM authorization failure that directly prevented infrastructure from functioning.

The AWS Load Balancer Controller was installed and running. The Ingress was applied. But no ALB appeared — the Ingress had no external address and nothing showed up in the AWS Console.

The controller was making AWS API calls it didn't have permission to complete. The missing permission was:

```
elasticloadbalancing:DescribeListenerAttributes
```

The controller's error output showed HTTP 403 responses from AWS. Not a Kubernetes configuration problem, not a networking issue — a permissions gap in the IAM role.

Attaching the missing permission and restarting the controller resolved it. The ALB provisioned within minutes.

The practical lesson: when an EKS workload interacts with AWS services, IAM policies become part of the application's operational dependency chain. A Kubernetes workload can appear completely healthy while being unable to perform required AWS operations. When something isn't happening on the AWS side, check IAM before rechecking Kubernetes configuration.

### Network Isolation

All application Services use ClusterIP — internal to the cluster only. Nothing is directly reachable from the internet except through the ALB.

```
Internet → AWS Application Load Balancer (public) → Kubernetes Ingress
    → Frontend Service (ClusterIP — internal)
    → Backend Service (ClusterIP — internal)
    → MongoDB Service (ClusterIP — internal)
```

| Service | Type | Publicly Accessible |
|---------|------|---------------------|
| Frontend | ClusterIP | No |
| Backend | ClusterIP | No |
| MongoDB | ClusterIP | No |
| Ingress | ALB | Yes |

MongoDB never leaves the cluster network. The backend communicates with it through Kubernetes internal DNS at `mongodb-svc:27017`.

### Security Groups

Security Groups for the EKS control plane and worker nodes were provisioned automatically through eksctl cluster creation.

- **Management EC2:** Inbound SSH (port 22) for remote administration
- **Application Load Balancer:** Inbound HTTP (port 80) for public application access
- **Worker Nodes:** Traffic from ALB and Kubernetes control plane only — no unrestricted public internet access

> Production improvement: SSH access to the management EC2 should be restricted to trusted source IP ranges, or replaced with AWS Systems Manager Session Manager to eliminate the need for open SSH ports.

### Namespace Isolation

```bash
kubectl create namespace workshop
```

This keeps application workloads logically separated from system-level Kubernetes resources. It also simplifies administration — all application resources are visible in one place.

> Namespaces are not a complete security boundary on their own. For production, Kubernetes Network Policies would add Pod-to-Pod traffic restrictions that namespaces alone don't provide.

### Kubernetes Secrets

MongoDB credentials were stored using a Kubernetes Secret rather than being hardcoded into Deployment manifests or container images.

Benefits:
- Credentials separated from application code and manifests
- Credential rotation possible without rebuilding images
- Sensitive values not visible in plain deployment YAML

> Production consideration: Kubernetes Secrets are Base64-encoded by default — not encrypted at rest. In production, envelope encryption using AWS KMS or integration with AWS Secrets Manager would provide stronger protection.

### Security Assessment

| Domain | Status |
|--------|--------|
| Identity Management | Implemented — IAM user + IRSA for controller |
| OIDC Integration | Implemented via eksctl |
| Namespace Isolation | Implemented — workshop namespace |
| Internal Service Networking | Implemented — ClusterIP only |
| Kubernetes Secrets | Implemented — MongoDB credentials |
| Persistent Storage | Implemented — EBS-backed PV |
| Load Balancer Security | Implemented — single ALB entry point |
| HTTPS Encryption | Future Enhancement |
| Kubernetes RBAC | Future Enhancement |
| Network Policies | Future Enhancement |
| AWS WAF | Future Enhancement |
| Image Vulnerability Scanning | Future Enhancement |

---

## Chapter 12 — Monitoring & Observability

### Overview

This deployment didn't use a full observability stack — no Prometheus, no Grafana, no centralized log aggregation. The monitoring approach was practical for the scope: Kubernetes-native diagnostic commands, AWS Console verification, and container logs when needed.

What this chapter documents is what was actually used during the deployment — the specific tools, the specific situations where they helped, and what they revealed. Where production would require additional tooling, that's called out as a future enhancement.

### What Was Actually Used

**kubectl — Primary Diagnostic Tool**

```bash
kubectl get nodes
kubectl get pods -n workshop
kubectl get svc -n workshop
kubectl get pvc -n workshop
kubectl get ingress -n workshop

kubectl describe pod <pod-name> -n workshop
kubectl describe svc <service-name> -n workshop
kubectl describe pvc <pvc-name> -n workshop
kubectl describe ingress <ingress-name> -n workshop
```

The `kubectl describe` command was particularly useful — its Events section shows what Kubernetes actually attempted, which is often different from what the high-level `kubectl get` status suggests.

**Container Logs — AWS Load Balancer Controller**

The most concrete use of container logs during this deployment was investigating why the ALB wasn't provisioning.

The controller was running. The Ingress resource existed. But no ALB appeared. Rather than continuing to check Kubernetes manifests, the controller logs were the next step:

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

The logs showed HTTP 403 responses from AWS — an authorization failure, not a Kubernetes configuration problem. This immediately shifted the investigation from Kubernetes to IAM, where the missing permission `elasticloadbalancing:DescribeListenerAttributes` was identified and corrected.

This is the most practical observability lesson from the project: a healthy Kubernetes workload does not mean the underlying AWS integration is functioning correctly. Logs were necessary to distinguish a Kubernetes configuration issue from an AWS IAM authorization failure.

**AWS Console — Infrastructure Verification**

Used alongside kubectl for visual verification:
- EKS cluster status and worker node health
- Load balancer provisioning state
- Target group configuration and health
- IAM role attachments
- ECR repository contents
- EBS volume state

Some things are faster to verify visually — whether an ALB exists, whether target groups are healthy, whether an IAM policy is attached. The Console and terminal together gave a more complete picture than either alone.

**Browser-Based Validation**

The final validation layer was the application itself:
- React frontend loaded
- Added a task through the UI
- Task appeared — frontend → backend communication confirmed
- Refreshed the page — task still present, MongoDB persistence confirmed

This end-to-end validation was the definitive check that everything was working correctly. Kubernetes showing healthy Pods is necessary but not sufficient — the application has to actually function from a user's perspective.

### Incremental Validation Approach

Monitoring wasn't saved for after deployment was complete. Each phase was verified before moving to the next:

```
kubectl get nodes → both nodes Ready
  ↓
AWS Console → ECR repositories visible
  ↓
kubectl get pods → all Running
  ↓
kubectl get pvc → Bound
  ↓
kubectl get ingress → ADDRESS populated
  ↓
Browser → task creation → page refresh → data persists
```

This incremental approach made root cause identification faster. When something broke, the scope of possible causes was already narrowed by what had already been verified.

### Current Monitoring Coverage

| Capability | Status |
|------------|--------|
| Pod Monitoring | ✓ |
| Deployment Monitoring | ✓ |
| Container Logs (ALB Controller) | ✓ |
| Kubernetes Resource Events | ✓ via kubectl describe |
| Node Health | ✓ |
| Ingress Status | ✓ |
| Persistent Storage Validation | ✓ |
| AWS Console Infrastructure Verification | ✓ |
| CloudWatch Active Monitoring | Future Enhancement |
| Prometheus Metrics | Future Enhancement |
| Grafana Dashboards | Future Enhancement |
| Centralized Log Aggregation | Future Enhancement |

### Recommended Production Additions

| Technology | Purpose |
|-----------|---------|
| Prometheus | Metrics collection from Kubernetes and applications |
| Grafana | Dashboard visualization |
| Alertmanager | Alert routing and notification |
| Fluent Bit | Log forwarding from containers |
| Elasticsearch | Centralized log storage |
| Kibana | Log visualization and search |
| Jaeger | Distributed tracing across services |
| OpenTelemetry | Unified telemetry collection |
| CloudWatch Alarms | Automated AWS infrastructure alerting |

---

## Chapter 13 — Performance, Scalability & Reliability

### Overview

This chapter is honest about scope. No load testing was performed, no benchmarks were run, and no performance metrics were collected during this deployment. The project was validated functionally — the application worked correctly end to end, data persisted, and the infrastructure remained stable throughout testing.

What this chapter covers is the architectural decisions that support performance and reliability, what was actually observed, and what would need to be added to make this production-ready from a performance standpoint.

### What Was Actually Validated

- Frontend loaded successfully through the ALB DNS endpoint
- Tasks submitted through the React UI reached the Node.js backend
- Data stored in MongoDB and persisted across page refreshes
- Both backend replicas were running and available as Service endpoints
- No unexpected application failures during functional testing
- Application remained stable throughout the validation period

What was not measured:
- Response time or latency
- Throughput or requests per second
- Concurrent user capacity
- CPU and memory utilization under load
- Pod recovery time after failure
- Load distribution between backend replicas

This is an honest assessment of a portfolio deployment. The architecture is designed for these capabilities — they just weren't exercised under load during this project.

### Scalability Strategy

| Component | Scaling Characteristics |
|-----------|------------------------|
| Frontend | Horizontally scalable — stateless |
| Backend API | Horizontally scalable — stateless |
| MongoDB | Stateful — requires storage coordination |

Stateless workloads like the frontend and backend can be scaled by increasing the Deployment replica count. No application code changes needed — Kubernetes handles scheduling additional Pods and the Service distributes traffic automatically.

MongoDB scaling is more complex. Adding replicas without configuring proper MongoDB replication would create multiple independent database instances rather than a coordinated cluster.

### High Availability

| Capability | Status |
|------------|--------|
| Managed Control Plane | ✓ — EKS handles control plane availability |
| Multiple Worker Nodes | ✓ — 2 × t2.medium nodes |
| Multiple Backend Replicas | ✓ — 2 replicas configured |
| Automatic Pod Recovery | ✓ — via Deployment and ReplicaSet |
| Load Balanced Traffic | ✓ — via ALB and Kubernetes Services |

The managed EKS control plane removes a significant availability concern — AWS is responsible for the API server, scheduler, and etcd.

### Fault Tolerance

**Pod Failure:** The Kubernetes Deployment and ReplicaSet provide automatic recovery:
1. Pod terminates unexpectedly
2. ReplicaSet detects actual replica count is below desired
3. Kubernetes Scheduler selects an available node
4. New Pod is created and starts
5. Service routes traffic to healthy Pods during recovery

**Node Failure:** If a worker node becomes unavailable, Pods scheduled on that node are rescheduled onto remaining healthy nodes. With two worker nodes, a single node failure doesn't take the application offline.

**Storage Protection:** MongoDB data persists independently of the container lifecycle through Persistent Volumes backed by Amazon EBS. If the MongoDB Pod is restarted or rescheduled, it reattaches to the same EBS volume.

### Cost Management

Keeping AWS costs under control after testing was a deliberate part of the deployment lifecycle:

```bash
kubectl delete -f .
```

AWS resources created through the Ingress integration followed controller-managed cleanup when the Ingress was deleted. The EKS cluster, EC2 management instance, and remaining resources were decommissioned after project completion.

Running EKS worker nodes, load balancers, and EBS volumes continue generating costs even when idle. Cleaning up promptly after validation is part of responsible cloud resource management.

### Production Readiness Assessment

| Area | Current Status | Production Recommendation |
|------|---------------|--------------------------|
| Availability | Good | Multi-AZ worker node groups |
| Scalability | Good | Horizontal Pod Autoscaler |
| Storage | Good | Managed database service or StatefulSets |
| Monitoring | Basic | Prometheus + Grafana |
| Security | Good | RBAC, Network Policies, AWS WAF |
| Resource Governance | Not confirmed | Define CPU/memory requests and limits |
| Performance Testing | Not performed | k6, JMeter, or Locust for load testing |

---

## Chapter 14 — Project Outcome & Engineering Impact

### What This Project Actually Was

This wasn't a tutorial where everything worked on the first try. It was a hands-on deployment that required working through real configuration issues across IAM, Kubernetes networking, persistent storage, and application behavior — debugging each one systematically until the complete application was running end to end.

The goal was never just to get containers running. It was to understand how AWS infrastructure, Kubernetes orchestration, and application configuration interact in a real deployment — and to develop the troubleshooting instinct that comes from actually breaking things and fixing them.

### Project Objectives vs. Outcomes

| Objective | Outcome | Status |
|-----------|---------|--------|
| Deploy a three-tier application on Amazon EKS | Successfully deployed | Completed |
| Containerize application components | Docker images created and deployed | Completed |
| Store images in Amazon ECR | Private repositories configured | Completed |
| Configure Kubernetes networking | Services and Ingress implemented | Completed |
| Expose application through AWS ALB | Path-based routing operational | Completed |
| Configure persistent storage | MongoDB data persisted using PV/PVC | Completed |
| Validate end-to-end functionality | Browser → Backend → MongoDB verified | Completed |
| Clean up AWS resources | Infrastructure successfully decommissioned | Completed |

### What Actually Surprised Me

Two things stood out that I hadn't fully anticipated going in.

**React environment variables.** I initially expected that updating the environment variable in Kubernetes would change the backend URL the frontend was targeting. It didn't — because React bakes environment variables into the compiled bundle at build time, not at runtime. The Kubernetes config was irrelevant at that point. The fix was switching to a relative API path in `taskServices.js` so requests routed through Ingress instead of depending on a hardcoded URL. It's not a complicated fix, but understanding why it was broken required understanding how React builds work, not just how Kubernetes works.

**IAM and silent failures.** The AWS Load Balancer Controller was installed, the Ingress was applied, and nothing happened — no ALB appeared. Everything looked correct from the Kubernetes side. The problem was a single missing IAM permission: `elasticloadbalancing:DescribeListenerAttributes`. The controller couldn't complete its AWS API calls and failed silently from the Kubernetes perspective. That experience reinforced something that matters in AWS deployments: when something isn't happening that should be, check the permissions layer before assuming the configuration is wrong.

### Engineering Challenges Overcome

| Challenge | Category | Severity | Status |
|-----------|----------|----------|--------|
| Dockerfile naming — Dockefile | Containerization | Medium | Resolved |
| PVC stuck in Pending | Kubernetes Storage | High | Resolved |
| Service port mismatch — 3500 | Kubernetes Networking | High | Resolved |
| React build-time env vars | Application Config | Medium | Resolved |
| Missing IAM permission | AWS Security | Critical | Resolved |
| Ingress host configuration | Networking | Medium | Resolved |

These weren't theoretical problems. Each one blocked a real part of the deployment and required actual investigation to resolve.

### What I'd Do Differently

I manually created and configured most of the environment to understand how everything worked — which was the right approach for learning, but not for repeatability. If I ran this project again, I'd use Terraform for infrastructure provisioning from the start rather than manual setup, and add a CI/CD pipeline to handle image builds, ECR pushes, and EKS deployments automatically. The manual process made the dependencies between components visible, which helped with troubleshooting. But for anything running beyond a single deployment cycle, automation is the right answer.

### Key Takeaways

**Small configuration details have large consequences.** A single missing IAM permission stopped the entire load balancer from provisioning. A one-character filename typo blocked the image build pipeline.

**Kubernetes and AWS are separate layers that both need to be correct.** Healthy Pods don't mean correct networking. Correct Kubernetes configs don't mean correct AWS permissions.

**React build behavior is different from server-side runtime behavior.** Build-time configuration needs to be understood before deployment, not discovered during debugging.

**Incremental validation is faster than end-to-end testing.** Checking each layer before moving to the next caught issues earlier and made root causes easier to isolate.

---

## Chapter 15 — Future Enhancements & Platform Evolution

### Overview

This deployment works. The application runs, the infrastructure held up under real debugging conditions, and the complete frontend → backend → database flow was verified end to end. But it was built and deployed manually — every resource provisioned through CLI commands, every manifest applied by hand.

That's the right approach for learning. It's not the right approach for a repeatable, production-scale deployment. This chapter outlines what would need to change to evolve this from a portfolio implementation into something closer to an enterprise platform.

### Enhancement Roadmap

| Phase | Focus Area | Primary Goal |
|-------|-----------|-------------|
| Phase 1 | Infrastructure as Code | Eliminate manual provisioning |
| Phase 2 | CI/CD | Automated application delivery |
| Phase 3 | Observability | Centralized monitoring and alerting |
| Phase 4 | GitOps | Declarative continuous delivery |
| Phase 5 | Production Operations | Scalability, resilience, and governance |

### Phase 1 — Infrastructure as Code with Terraform

The AWS infrastructure for this project was provisioned using eksctl and AWS CLI commands. eksctl automated a significant amount of cluster setup, but the overall provisioning process is manual and not version-controlled as infrastructure code.

Terraform is the next infrastructure tool I want to learn. The goal is to provision the same AWS environment — VPC, EKS cluster, node groups, ECR repositories, IAM roles — through declarative Terraform configuration files rather than CLI commands.

Proposed scope:
- Amazon VPC and subnet configuration
- Internet Gateway and route tables
- Security Groups
- IAM Roles and policies
- Amazon EKS cluster and managed node groups
- Amazon ECR repositories
- CloudWatch resources

### Phase 2 — CI/CD Pipeline

Container images are currently built manually on the EC2 management instance and pushed to Amazon ECR by hand. Every application update requires SSHing into the instance, rebuilding the image, and reapplying Kubernetes manifests.

I have hands-on experience with both Jenkins and GitHub Actions from previous CI/CD work — including build automation, SonarQube integration, Nexus Repository, and deployment to AWS EC2. Applying that experience to this EKS deployment would automate the full image build and deployment pipeline.

Proposed CI workflow:
- Source code validation on every push
- Dependency installation and unit testing
- Docker image build and tagging
- Security scanning before push
- Automated push to Amazon ECR
- Kubernetes Deployment update on successful image push

### Phase 3 — Observability Stack

I'm familiar with Prometheus and Grafana and understand their role in Kubernetes monitoring, but haven't implemented them hands-on in this project. Adding a proper observability stack would be the next meaningful improvement after CI/CD.

| Technology | Purpose |
|-----------|---------|
| Prometheus | Metrics collection from Kubernetes and applications |
| Grafana | Dashboard visualization |
| Alertmanager | Alert routing and notification |
| Fluent Bit | Log forwarding from containers |
| Elasticsearch | Centralized log storage |
| Kibana | Log visualization and search |
| OpenTelemetry | Unified telemetry collection |

### Phase 4 — GitOps with Argo CD

Kubernetes resources are currently applied manually using `kubectl apply -f .`. There's no automated synchronization between the Git repository and the cluster state.

I have conceptual awareness of Argo CD and GitOps workflows but haven't implemented a full GitOps deployment for this project. The idea is straightforward — the Git repository becomes the single source of truth for cluster state, and Argo CD continuously ensures the cluster matches what's defined in Git.

Advantages:
- Declarative deployments — Git is the audit trail
- Automatic synchronization — cluster drift is detected and corrected
- Rollback is a Git revert, not a manual kubectl operation
- Every deployment change is reviewed through a pull request

### Phase 5 — Autoscaling

Replica counts are currently static. The backend runs two replicas regardless of actual traffic. Worker node count is fixed at two t2.medium instances.

**Horizontal Pod Autoscaler:** Automatically adjusts replica counts based on CPU or memory metrics. For stateless workloads like the frontend and backend, this means the deployment scales to meet demand automatically.

**Cluster Autoscaler:** When the cluster runs out of scheduling capacity, the Cluster Autoscaler adds worker nodes automatically. Combined with HPA, the entire infrastructure scales up and down with demand.

### Phase 6 — Advanced Deployment Strategies

**Blue-Green Deployment:** Maintain two production environments. Traffic switches from the current version to the new version atomically — instant rollback if something goes wrong.

**Canary Deployment:** Release new versions to a small percentage of traffic first. Observe behavior before gradually increasing the rollout. Catch issues affecting real users without exposing everyone to a potentially broken release.

### Phase 7 — Security Enhancements

| Enhancement | Benefit |
|-------------|---------|
| AWS Secrets Manager | Centralized secret lifecycle management |
| AWS KMS | Encryption of Kubernetes Secrets at rest |
| Kubernetes RBAC | Fine-grained cluster authorization |
| Network Policies | Pod-to-Pod traffic restrictions |
| AWS WAF | Web application protection |
| Image Scanning | Vulnerability detection in ECR |
| SSM Session Manager | SSH-free EC2 administration |
| Admission Controllers | Policy enforcement at deployment time |

### Honest Assessment

The current implementation demonstrates the core engineering workflow — provisioning infrastructure, containerizing an application, deploying it on Kubernetes, debugging real issues, and validating the result. That foundation is solid.

The next logical steps are clear:
- **Terraform** — replace manual provisioning with version-controlled infrastructure code. Highest priority next learning area.
- **CI/CD pipeline** — apply existing Jenkins and GitHub Actions experience to automate image builds and EKS deployments.
- **Observability** — add Prometheus and Grafana to move from reactive debugging to proactive monitoring.
- **GitOps** — introduce Argo CD to make cluster state declarative and auditable.

These aren't aspirational additions — they're the natural next layer on top of what's already working.

### Platform Maturity Assessment

| Capability | Current | Future Target |
|-----------|---------|--------------|
| Kubernetes Deployment | Functional | Production Grade |
| Infrastructure Automation | Manual — eksctl/CLI | Terraform |
| CI/CD | Manual — SSH + kubectl | GitHub Actions |
| Monitoring | kubectl + AWS Console | Prometheus + Grafana |
| Logging | kubectl logs | ELK Stack |
| Security | Good — IAM, Secrets, ClusterIP | RBAC, Network Policies, WAF |
| GitOps | Not implemented | Argo CD |
| Autoscaling | Static replicas | HPA + Cluster Autoscaler |
| Service Mesh | Not implemented | Istio |
| Deployment Strategy | Rolling updates | Blue-Green and Canary |

---

## Chapter 16 — Troubleshooting & Debugging Journey

### Overview

Deploying this application involved more than writing Kubernetes manifests and applying them to a cluster. The deployment crossed multiple layers — and a failure at any point in that chain could prevent the application from working even when everything above or below it appeared healthy.

```
Application Code → Docker → Amazon ECR → Amazon EKS → Kubernetes Networking
    → Persistent Storage → IAM → AWS Load Balancer Controller
    → Application Load Balancer → Browser
```

The most useful habit I developed was learning not to treat the deployment as a single system. When something failed, the approach was to isolate the layer first:

```
Application not working
    ↓
Is the ALB reachable?
    ├── No → Check Ingress / ALB Controller / IAM
    ↓
Does routing reach the Service?
    ├── No → Check Service / Ports / Selectors
    ↓
Are Pods healthy?
    ├── No → Check Pod status / Events / Logs
    ↓
Can the backend reach MongoDB?
    ├── No → Check Service / PVC / Storage
    ↓
Check application-level configuration
```

### Debugging Environment

All debugging happened from the dedicated Ubuntu EC2 management instance over SSH.

| Tool | Primary Use |
|------|-------------|
| kubectl | Inspecting and managing Kubernetes resources |
| eksctl | Creating and managing the EKS cluster |
| AWS CLI | Authenticating with and interacting with AWS services |
| docker | Building and managing container images |
| helm | Installing the AWS Load Balancer Controller |
| AWS Console | Verifying EKS, ECR, IAM, and ALB resources |
| Browser | End-to-end application validation |

Common validation commands:
```bash
kubectl get nodes
kubectl get pods -n workshop
kubectl get svc -n workshop
kubectl get ingress -n workshop
kubectl get pvc -n workshop
kubectl describe <resource-name> -n workshop
```

### Incident 01 — Docker Build Blocked by Invalid Dockerfile Name

**Symptom:** The backend container image failed during the initial build phase. Without a successful image there was nothing to push to ECR, and Kubernetes had nothing to pull. The failure occurred before Kubernetes was involved — which immediately narrowed the problem to the local containerization layer.

**Root Cause:** The file in the backend directory was named `Dockefile` instead of `Dockerfile`. One missing character blocked the entire build.

**Resolution:** Renamed the file to `Dockerfile` and reran the image build. The image built successfully and pushed to Amazon ECR cleanly.

**Debugging takeaway:** Start at the point where the failure first occurs. Kubernetes was not the problem here — the image hadn't even been created yet.

### Incident 02 — MongoDB Could Not Start Because Storage Was Not Bound

**Symptom:** MongoDB couldn't start because the Persistent Volume Claim remained in `Pending` instead of transitioning to `Bound`. The storage dependency chain:

```
MongoDB Pod → Persistent Volume Claim → Persistent Volume → Amazon EBS-backed Storage
```

If the claim can't bind to storage, the database workload can't attach the required volume.

**Root Cause:** The PVC was applied before a matching Persistent Volume was available for binding. The dependency sequence matters:

```
Persistent Volume → Persistent Volume Claim → MongoDB Deployment / Pod
```

**Resolution:** Applied storage resources in the correct order — PV before PVC before MongoDB Deployment. Once the PV was in place, the PVC bound immediately:

```bash
kubectl get pvc -n workshop
# STATUS: Bound
```

**Debugging takeaway:** When a stateful workload fails to start, inspect the storage dependency chain before assuming the container configuration is wrong.

### Incident 03 — Healthy Backend Pods but Failed API Connectivity

**Symptom:** The frontend loaded in the browser. Backend Pods showed Running. But API calls were failing.

Pod Status = Running did not mean Application Networking = Correct.

**Root Cause:** The Service `targetPort` didn't match the backend application's listening port (3500).

**Resolution:** Corrected the Service manifest:

```yaml
targetPort: 3500
```

**Debugging takeaway:** A healthy Pod only proves the container is running. It doesn't prove traffic can reach the application. Always verify Service port configuration matches what the application actually exposes.

### Incident 04 — React API Configuration Failed After Deployment

**Symptom:** Frontend accessible. Backend Pods running. Service configuration correct. Yet API requests kept failing.

The frontend originally used a build-time environment variable for the backend URL: `REACT_APP_BACKEND_URL`. The natural assumption was that updating this in Kubernetes would change where the frontend sent requests. It didn't — because React resolves environment variables at build time, not runtime. Once the image was built, changing a Kubernetes environment variable had no effect on the already-compiled bundle.

**Root Cause:** API URL configuration existed at build time rather than being dynamically resolved at runtime.

**Resolution:** Changed `taskServices.js` to use a relative API path:

```javascript
const apiUrl = "/api/tasks";
```

With a relative path, the request goes to whatever host is serving the frontend — the ALB — and Ingress handles routing `/api` traffic to the backend.

**Debugging takeaway:** Not every deployment problem belongs to Kubernetes. When infrastructure appears healthy but the application still fails, trace the request back into the application configuration.

### Incident 05 — AWS Load Balancer Controller Could Not Provision the ALB

**Symptom:** Kubernetes Ingress created. Controller running. No ALB appeared. The Ingress had no external address.

The controller was running successfully while failing to complete the AWS API operations needed to provision infrastructure.

**Root Cause:** The IAM role attached to the Load Balancer Controller was missing:

```
elasticloadbalancing:DescribeListenerAttributes
```

The controller's logs showed HTTP 403 authorization failures from AWS. That immediately changed the direction of investigation away from Kubernetes manifests and toward IAM.

**Resolution:** Added the missing permission to the IAM policy. After correcting permissions and restarting the controller:

```bash
kubectl get ingress -n workshop
```

The Ingress received an external ALB DNS address.

**Debugging takeaway:** When an AWS-integrated controller appears healthy but creates no infrastructure, check controller logs and IAM permissions before modifying Kubernetes manifests.

### Incident 06 — Ingress Routing Required a Domain That Didn't Exist

**Symptom:** The initial Ingress configuration included hostname-based routing, but the deployment didn't use a custom domain — just the ALB DNS name directly.

**Root Cause:** The routing model assumed hostname-based access when the actual requirement was simpler.

**Resolution:** Removed the host field from the Ingress configuration. Simplified to path-based routing only:

```
/       → Frontend Service
/api    → Backend Service
```

**Debugging takeaway:** Infrastructure should match the actual requirement. If a dependency isn't needed for the deployment objective, removing it simplifies both configuration and troubleshooting.

### Key Engineering Lessons

**Work from the failing layer.** When the Docker image failed, Kubernetes was irrelevant. When the PVC was pending, the frontend was irrelevant. When the ALB didn't provision, the application Pods weren't the first thing to check.

**Running doesn't mean working.** A Pod can be healthy while Services route to the wrong port, IAM blocks infrastructure, or application configuration points to the wrong endpoint.

**Logs are more useful than assumptions.** The Load Balancer Controller issue demonstrated this directly. The controller appeared healthy — but its logs showed HTTP 403. That prevented unnecessary Kubernetes manifest changes and pointed straight at IAM.

**Stateful workloads have dependency ordering.** MongoDB wasn't just a container — it required a Persistent Volume to exist and bind before the Pod could schedule.

**Application configuration can look like an infrastructure problem.** The React environment variable issue looked like a Kubernetes networking problem. The real issue was the frontend build process.

### The Most Valuable Outcome

The most valuable outcome wasn't resolving six individual problems. It was developing a clearer way to reason about a cloud-native system under failure conditions:

```
Observe Failure → Identify Affected Layer → Inspect Resource State
    → Check Logs / Events / Configuration → Validate Dependencies
    → Correct Root Cause → Redeploy Only What Is Required → Verify End-to-End
```

This project required debugging across Docker, ECR, Kubernetes workloads, Services, persistent storage, application configuration, Ingress, IAM, the Load Balancer Controller, and ALBs. The deployment worked because each failure was traced back to its responsible layer rather than treating the entire EKS environment as one black box.

---

## Chapter 17 — Deployment Validation & Operational Evidence

### Overview

Getting `kubectl apply` to run without errors was never the finish line. The deployment was validated progressively — each layer checked independently before moving to the next. The principle throughout: a running Pod does not necessarily mean a working application.

Validation covered six distinct layers: infrastructure, Kubernetes resources, AWS integration, frontend accessibility, backend API communication, and database persistence. Each had to pass before the deployment was considered complete.

### Layer 1 — EKS Cluster Validation

```bash
kubectl get nodes
```

Both worker nodes showed Ready status — confirming the control plane was responding, nodes were registered, and kubectl was correctly connected. Only after both nodes showed Ready were application resources deployed.

### Layer 2 — Namespace and Resource Validation

```bash
kubectl get all -n workshop
kubectl get pods -n workshop
kubectl get svc -n workshop
kubectl get deployments -n workshop
kubectl get pvc -n workshop
kubectl get ingress -n workshop
```

### Layer 3 — Pod Health Validation

Expected running workloads: 1 React frontend Pod, 2 Node.js backend Pods, 1 MongoDB Pod — 4 total.

```bash
kubectl get pods -n workshop
```

All four Pods reached Running status. A Pod not reaching Running required investigation. `kubectl apply` accepts a manifest without guaranteeing the workload starts successfully — Pod status was the actual proof.

### Layer 4 — Service Connectivity Validation

```bash
kubectl get svc -n workshop
kubectl describe svc <service-name> -n workshop
```

All three Services — frontend, backend (api), MongoDB — were confirmed with correct port mappings.

The backend Service required specific attention: the Node.js application listened on port 3500, and the Service `targetPort` had to match exactly. A mismatch causes frontend API calls to fail even when the backend Pod is healthy.

### Layer 5 — Persistent Storage Validation

```bash
kubectl get pvc -n workshop
```

The PVC transitioned to `Bound` status — confirming Kubernetes successfully completed the storage binding.

During implementation, the PVC initially remained in `Pending` because it was applied before a matching Persistent Volume was available. Correcting the apply order resolved the binding. Once Bound, MongoDB started with persistent storage attached.

### Layer 6 — Ingress Validation

```bash
kubectl get ingress -n workshop
```

| Path | Destination |
|------|-------------|
| / | Frontend Service |
| /api | Backend Service |

Initially the Ingress had no external address — the AWS Load Balancer Controller needed to provision the ALB first. After the IAM permission issue was resolved and the controller restarted, the Ingress received an ALB DNS address.

### Layer 7 — AWS Load Balancer Validation

The controller encountered an IAM authorization failure during provisioning — the missing permission `elasticloadbalancing:DescribeListenerAttributes` prevented it from completing AWS API calls. After attaching the required policy and restarting the controller, the ALB provisioned successfully.

Validation confirmed:
- ALB visible in AWS Console
- Ingress showing external ALB DNS address
- Target groups healthy
- ALB DNS endpoint reachable

### Layer 8 — Browser-Level Application Validation

The React frontend loaded successfully through the ALB DNS endpoint in a browser. This confirmed the ALB was reachable, Ingress routing was working, the frontend Service was correctly configured, and the frontend Pod was serving the application.

### Layer 9 — API Flow Validation

The frontend used a relative API path:

```javascript
const apiUrl = "/api/tasks";
```

This avoided embedding a fixed backend hostname into the compiled bundle. The ALB and Ingress determined where `/api` traffic went — frontend, backend, and AWS networking aligned correctly.

### Layer 10 — End-to-End Data Validation

A task was submitted through the React UI:
1. Frontend sends POST request to `/api/tasks`
2. Request reaches Node.js backend through Ingress
3. Backend processes request and writes to MongoDB
4. MongoDB stores data to persistent EBS-backed volume
5. Frontend receives response and displays the task
6. Page refresh — task still visible, confirming persistence

All steps completed successfully. The complete Frontend → Backend → Database path was operational.

### Final Operational Status

```
EKS Cluster                 ✓ Operational
Worker Nodes                ✓ Ready
Frontend Pod                ✓ Running
Backend Pods                ✓ Running (×2)
MongoDB Pod                 ✓ Running
Kubernetes Services         ✓ Operational
Persistent Storage          ✓ Bound
AWS Load Balancer           ✓ Provisioned
Ingress Routing             ✓ Operational
Frontend Access             ✓ Verified
Backend API Connectivity    ✓ Verified
Database Persistence        ✓ Verified
End-to-End Application Flow ✓ Successful
Infrastructure Cleanup      ✓ Completed
```

### Engineering Takeaway

The deployment required multiple independent systems to work together simultaneously — AWS infrastructure, IAM permissions, Docker images, ECR, Kubernetes scheduling, Service networking, persistent storage, Load Balancer Controller, Ingress routing, and application-level API communication.

A failure in any one of these layers prevented the final application from working. The completed validation demonstrated the full operational path:

```
Browser → ALB → Ingress → Kubernetes Services → Application Pods → MongoDB → Persistent Storage
```

That end-to-end verification is the evidence that the deployment functioned as an integrated cloud-native system — not merely as a collection of successfully applied Kubernetes manifests.

---

## Chapter 18 — Architecture Decisions & Engineering Trade-offs

### Overview

This chapter documents the key architectural decisions behind the EKS deployment and the reasoning behind each one. The goal isn't to present every choice as the only valid approach — it's to explain why the selected architecture supported the objective of deploying and validating a complete three-tier application on Kubernetes, and what trade-offs came with each decision.

### Decision Summary

| Decision | Selected Approach | Primary Reason |
|----------|------------------|---------------|
| Application architecture | Three-tier | Clear separation between UI, API, and data layers |
| Container orchestration | Amazon EKS | Managed Kubernetes control plane on AWS |
| Container registry | Amazon ECR | AWS-native registry — native EKS integration |
| External access | ALB + Kubernetes Ingress | Single entry point for HTTP traffic |
| Internal networking | ClusterIP Services | Keep application components private |
| Backend availability | Two replicas | More than one API Pod available |
| Database access | Internal Kubernetes Service | MongoDB not directly exposed |
| Database persistence | PV/PVC backed by EBS | Preserve data beyond Pod lifecycle |
| API routing | Path-based routing | Avoid dependency on a custom domain |
| Deployment method | Manual Kubernetes manifests | Focus on understanding the underlying workflow |

### Decision 1 — Three-Tier Application Architecture

Each layer operates independently — the frontend can be updated without touching the database, the backend can scale without affecting the frontend, and database storage is managed separately from stateless application workloads.

Trade-off: Three tiers means more infrastructure and networking complexity than a single-container application. That complexity was acceptable here because the objective was specifically to understand how multiple workloads interact inside Kubernetes.

### Decision 2 — Amazon EKS Instead of Self-Managed Kubernetes

EKS provides a managed control plane while still requiring direct work with standard Kubernetes resources — Deployments, Services, Ingress, PV/PVC, Secrets. The managed aspect removes the need to operate the API server, scheduler, and etcd manually.

Trade-off: EKS simplifies control plane management, but Kubernetes itself remains operationally complex. Application networking, IAM integration, storage, and debugging still require direct engineering work.

### Decision 3 — Kubernetes Services Instead of Direct Pod Access

Pods are not stable endpoints. They restart, get recreated, receive new IP addresses, and move between nodes. Services solve this — the application communicates with a stable Service name rather than a specific Pod IP. Pods can change without requiring application-level networking changes.

### Decision 4 — ClusterIP for Internal Services

All three Services use ClusterIP. Nothing is directly accessible from outside the cluster except through the ALB.

Trade-off: ClusterIP Services require an additional routing layer for external traffic — provided here by ALB + Ingress. But keeping internal components private reduces the attack surface and eliminates unnecessary public endpoints.

### Decision 5 — AWS ALB and Kubernetes Ingress

A single ALB handles all external traffic. The Kubernetes Ingress defines the routing logic, letting the React frontend send requests using `/api/tasks` rather than requiring a separate public backend URL.

Trade-off: This approach depends on correct coordination between the ALB, Load Balancer Controller, IAM permissions, Ingress configuration, Services, and application ports. This became visible during implementation — the controller appeared healthy while the AWS integration layer was failing because of a missing IAM permission.

### Decision 6 — Path-Based Routing Instead of Host-Based Routing

The initial Ingress used hostname-based routing, but the project didn't use a custom domain. Removing the hostname dependency simplified everything. Path-based routing handled the separation between frontend and API traffic without any external DNS dependency.

Trade-off: A larger production system might use separate hostnames for stronger separation. For validating this deployment, path-based routing removed an unnecessary dependency without sacrificing functionality.

### Decision 7 — Relative API Paths in the Frontend

React resolves environment variables at build time — not runtime. Changing a Kubernetes environment variable after the image was built had no effect on the already-compiled bundle.

Switching to a relative path:

```javascript
const apiUrl = "/api/tasks";
```

Trade-off: This couples frontend API routing to the Ingress configuration — the `/api` path must remain correctly configured. But the benefit was significant: no hardcoded backend URL, no domain dependency, no image rebuild just to change the backend hostname.

### Decision 8 — Stateless vs. Stateful Workloads

Frontend and backend Pods are stateless — Kubernetes can replace them without any impact on application behavior. MongoDB is different. Without persistent storage, a Pod deletion means permanent data loss.

Storage was decoupled from the Pod lifecycle through PV/PVC backed by Amazon EBS. The Pod is replaceable. The data is not.

### Decision 9 — Two Backend Replicas

The backend ran two replicas. This demonstrated how Kubernetes Services interact with multiple Pods. If one Pod becomes unavailable, the Service continues routing to healthy endpoints while Kubernetes restores the desired state.

Trade-off: Additional replicas consume additional compute resources. For a demo environment one replica would have been sufficient functionally — but two replicas made the architecture more representative of a real distributed deployment.

### Decision 10 — Amazon ECR for Image Storage

Images were pushed to ECR because the infrastructure was already running in AWS. ECR integrates natively with EKS — worker nodes authenticate to ECR automatically through IAM without requiring separate registry credentials.

Trade-off: ECR increases dependency on the AWS ecosystem. A multi-cloud deployment would benefit from a cloud-neutral registry. For this single-AWS-region project, native integration was the right choice.

### Engineering Takeaway

The most important outcome wasn't that React, Node.js, and MongoDB were deployed to Kubernetes. The value came from understanding the connections between the layers.

A successful request depended on every layer working correctly simultaneously:

```
Browser → ALB → Load Balancer Controller → Ingress → Service → Pod → Application → Database → Persistent Storage
```

Those weren't isolated problems. They demonstrated a property of cloud-native systems that documentation doesn't fully convey:

> The application is only as reliable as the interaction between its infrastructure layers.

---

## Chapter 19 — Engineering Reflection & Professional Takeaways

### Overview

This chapter documents the engineering perspective gained from implementing the AWS EKS deployment — not generic Kubernetes theory, but what the project actually revealed during hands-on implementation.

Before this project, I had experience with individual technologies — AWS, Docker, Linux, Kubernetes concepts. What I hadn't experienced to the same extent was how quickly a deployment becomes dependent on the interaction between those layers. That's the main thing this project changed.

### The Deployment as a Connected System

The project only worked when the complete chain worked:

```
Source Code → Docker Image → Amazon ECR → Amazon EKS → Kubernetes Deployment
    → Service Networking → Ingress → AWS Load Balancer Controller
    → Application Load Balancer → Browser
```

And after the frontend loaded, another chain still had to work:

```
React Frontend → /api/tasks → Ingress Path Rule → Backend Service
    → Node.js API → MongoDB Service → MongoDB Pod → Persistent Storage
```

Understanding the deployment as one connected system — rather than a collection of independent tools — was the core learning.

### Tooling Context

The project involved multiple tools working together:

```
eksctl      → Creates and manages EKS infrastructure
kubectl     → Interacts with Kubernetes resources
Helm        → Installs packaged Kubernetes applications
Docker      → Builds application container images
AWS CLI     → Authenticates and interacts with AWS services
Git/GitHub  → Manages source code and configuration
```

Learning individual commands was only part of the challenge. The harder part was understanding where each tool fit into the larger workflow — which layer it controlled, what it depended on, and what depended on it.

### Observability Was More Important Than Expected

During troubleshooting, a small set of commands became the primary diagnostic toolkit:

| Command | Primary Use |
|---------|-------------|
| kubectl get | Quick resource status |
| kubectl describe | Events and detailed configuration |
| kubectl logs | Application and controller output |
| AWS Console | AWS resource provisioning and status |

The debugging pattern that worked consistently:

```
Something Failed → Check Status → Check Events → Check Logs
    → Identify the Failing Layer → Make One Targeted Change → Validate Again
```

This is significantly more effective than repeatedly changing YAML files without first understanding what's failing and why.

### What I Would Do Differently

If rebuilding this project, I would reduce manual infrastructure and deployment steps from the start.

A future architecture would move toward:

```
Developer → Git Push → CI/CD Pipeline (Build, Test, Security Checks, Docker Image, Push to ECR)
    → Deployment Update → Amazon EKS
```

And infrastructure through code:

```
Terraform → AWS Infrastructure (VPC, Networking, EKS, IAM, Supporting Resources)
```

These are future improvements — not claimed as part of this implementation. The manual workflow was right for learning. It's not right for production scale.

### Final Reflection

The biggest difference between understanding this architecture on paper and implementing it was discovering how dependent every layer is on the others.

From the browser, the finished application looked simple:
```
Open Application → Add Task → Task Appears
```

Behind that interaction, the request passed through an entire infrastructure chain. Getting that chain working required more than applying manifests — it required understanding failures, reading logs, checking resource states, correcting configuration mismatches, resolving AWS permission issues, and validating the final result from the user's perspective.

That's the main professional takeaway:

> Cloud-native engineering is not about deploying individual technologies. It's about understanding how those technologies behave as one system — and being able to trace, diagnose, and fix the system when one part breaks.

---

## Chapter 20 — Production Readiness Assessment

### Honest Assessment

Completing a deployment successfully and being production-ready are not the same thing. Here's where this implementation actually stands.

### Architecture Status

| Area | Current State |
|------|--------------|
| Application Deployment | Implemented |
| Containerization | Implemented |
| Image Registry | Implemented — Amazon ECR |
| Kubernetes Orchestration | Implemented — Amazon EKS |
| External Traffic | Implemented — ALB and Ingress |
| Internal Networking | Implemented — ClusterIP Services |
| Persistent Storage | Implemented — PV/PVC backed by EBS |
| Backend Replication | Two replicas configured |
| IAM Integration | Implemented and troubleshot |
| CI/CD | Not implemented |
| Infrastructure as Code | Not implemented |
| Production Monitoring Stack | Not implemented |
| Automated Scaling | Not validated |
| Disaster Recovery | Not implemented |
| Multi-Region Availability | Not implemented |

### What's Already Production-Relevant

Several decisions in this deployment follow patterns used in real environments — not because the project claims to be production-ready, but because the reference architecture reflected sound engineering practices.

**Containerized application components:** Frontend and backend packaged independently, each with its own image lifecycle. A frontend update doesn't require rebuilding the backend.

**Managed Kubernetes control plane:** EKS removes the need to operate the API server, scheduler, and etcd manually. The focus stays on application deployment, networking, storage, and IAM integration.

**Internal service isolation:** Backend and MongoDB are not directly exposed to the internet. Only the ALB is public-facing. This creates a clear security boundary.

**Persistent storage for MongoDB:** PV/PVC backed by EBS means data survives Pod restarts and rescheduling. This is significantly more appropriate for a database workload than temporary container filesystem storage.

**IAM integration troubleshot under real conditions:** The Load Balancer Controller IAM failure demonstrated that cloud-native deployments require both Kubernetes configuration and AWS authorization to be correct simultaneously. That's a production reality, not just a learning exercise.

### What's Missing for Production

Being direct about the gaps:

- **No CI/CD** — every deployment step was manual. Consistent, repeatable production releases require automation.
- **No Infrastructure as Code** — the environment was provisioned through eksctl and CLI commands, not version-controlled Terraform. Rebuilding from scratch requires memory rather than code.
- **No formal monitoring** — operational visibility came from kubectl and the AWS Console. A production system needs centralized metrics, dashboards, and automated alerting.
- **No automated scaling** — replica counts are static. HPA and Cluster Autoscaler were not implemented.
- **No backup strategy** — MongoDB persistence was validated at the Pod level but there's no documented plan for storage failure, cluster loss, or data recovery.
- **No TLS** — the deployment used HTTP. Production traffic should be encrypted.
- **Single region** — the entire deployment runs in us-west-2. No cross-region failover.

### Final Production Readiness Verdict

Current status: Production-inspired architecture with a successful end-to-end implementation — intentionally not a fully production-ready platform.

What was implemented and validated:
- Containerized three-tier application architecture
- Amazon EKS deployment — three-tier-cluster, us-west-2
- Amazon ECR private image repositories
- Kubernetes Services, Ingress, path-based routing
- AWS Application Load Balancer integration
- Backend replica configuration — 2 × Node.js Pods
- MongoDB persistent storage — PV/PVC backed by EBS
- IAM configuration and real-world troubleshooting
- End-to-end browser-to-database validation
- Infrastructure cleanup after testing

What remains as future enhancements:
- CI/CD automation
- Infrastructure as Code — Terraform
- Formal monitoring and alerting — Prometheus, Grafana
- Automated scaling — HPA and Cluster Autoscaler
- Backup and disaster recovery
- TLS and HTTPS
- Advanced secret management
- Image security scanning
- GitOps workflows — Argo CD
- Progressive deployment strategies

### Engineering Perspective

The value of this project is not in claiming equivalence to an enterprise production platform. The value is in having implemented the foundation manually and understanding what happens between each layer — and then being able to look at a working deployment and honestly identify what's still missing.

That ability to assess your own work critically is itself a production engineering skill.

```
CURRENT STATE
Functional Application + Containerization + Kubernetes + AWS Infrastructure
    + Networking + Persistent Storage + Hands-On Troubleshooting
    = Strong Cloud-Native Engineering Foundation

NEXT EVOLUTION
Automation + Infrastructure as Code + Observability + Security Hardening
    + Autoscaling + GitOps
    = Production-Style Platform
```

The project established the foundation. The next stage is systematically improving repeatability, security, observability, resilience, and delivery automation — not randomly adding tools, but evolving the architecture in a deliberate sequence.

---

## Chapter 21 — Appendix & References

### Project Information

| Item | Details |
|------|---------|
| Project Name | Three-Tier Application Deployment using Kubernetes on AWS EKS |
| Project Type | Cloud-Native Infrastructure Deployment |
| Architecture | Three-Tier Web Application |
| Deployment Platform | Amazon Elastic Kubernetes Service (Amazon EKS) |
| Cluster Name | three-tier-cluster |
| Region | us-west-2 |
| Namespace | workshop |
| Project Status | Successfully Completed |

### Project Repository

**GitHub Repository:** https://github.com/Nuzairkhan/k8s-three-tier-app-eks

Repository contents:
- Application source code — React frontend, Node.js backend
- Dockerfiles for frontend and backend
- Kubernetes manifests — Deployments, Services, Ingress, Secrets, PV/PVC
- Ingress and ALB configuration
- Project documentation

### Project Demonstration

**LinkedIn Demo:** https://www.linkedin.com/posts/anwarullah-khan-nuzair_devops-aws-eks-activity-7415357318206418945-DehZ

The demonstration shows the deployed application running through the AWS Application Load Balancer — React frontend loading, API communication working, and task data persisting in MongoDB.

### Infrastructure Summary

| Resource | Configuration |
|----------|--------------|
| Amazon EKS Cluster | 1 — three-tier-cluster, us-west-2 |
| Worker Nodes | 2 × t2.medium |
| Amazon EC2 Management Host | 1 × t2.micro — Ubuntu |
| Amazon ECR Repositories | 2 — frontend, backend |
| Application Load Balancer | 1 — provisioned via Kubernetes Ingress |
| Deployments | 3 — frontend, backend, MongoDB |
| Services | 3 — all ClusterIP |
| Ingress | 1 — path-based routing |
| Persistent Volumes | 1 — Amazon EBS backed |
| Persistent Volume Claims | 1 — MongoDB storage |
| Namespaces | 1 — workshop |
| IAM User | eks-admin with AdministratorAccess |

### Real Issues Resolved

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Docker build failure | Backend Dockerfile named Dockefile | Renamed file, rebuilt image |
| PVC stuck in Pending | PVC applied before PV was available | Applied storage resources in correct order |
| Frontend couldn't reach backend | Service targetPort didn't match container port 3500 | Corrected targetPort in Service manifest |
| React API requests failing | REACT_APP_BACKEND_URL resolved at build time not runtime | Switched to relative path /api/tasks in taskServices.js |
| ALB not provisioning | IAM role missing elasticloadbalancing:DescribeListenerAttributes | Attached missing permission, restarted controller |
| Ingress routing failure | Host-based routing without a registered domain | Removed host field, used path-based routing only |

### Key Engineering Decisions

| Decision | Rationale |
|----------|-----------|
| eksctl for cluster provisioning | Automated VPC, subnet, and node group creation |
| ClusterIP for all Services | Internal-only communication — single ALB entry point |
| Path-based Ingress routing | Single public endpoint — / frontend, /api backend |
| Relative API path /api/tasks | Avoided React build-time environment variable issues |
| IRSA for Load Balancer Controller | Scoped IAM permissions — no node-level AWS access |
| PV/PVC for MongoDB | Storage independent of container lifecycle |
| Dedicated EC2 management host | Consistent administration environment |
| Separate ECR repositories | Independent versioning per service |

### Lessons Learned

**Infrastructure failures don't always originate from Kubernetes.** When EKS workloads interact with AWS services, IAM policies become part of the operational dependency chain.

**A healthy Pod doesn't mean a healthy application.** Services, ports, storage, Ingress rules, and AWS permissions all have to align simultaneously.

**React environment variables are resolved at build time.** Configuration that needs to change across environments should use relative paths or routing-layer abstractions rather than compiled-in values.

**Incremental validation is faster than end-to-end debugging.** Verifying each layer independently narrows the scope of possible root causes significantly.

**Responsible cloud engineering includes cleanup.** Leaving EKS clusters, load balancers, and EBS volumes running generates costs — decommissioning after validation is part of the workflow.

### Glossary

| Term | Description |
|------|-------------|
| ALB | AWS Application Load Balancer — distributes HTTP traffic to Kubernetes workloads |
| Amazon ECR | Private container image registry for storing Docker images |
| Amazon EKS | AWS managed Kubernetes control plane service |
| ClusterIP | Kubernetes Service type providing internal-only cluster networking |
| Deployment | Kubernetes controller managing desired Pod state and replica count |
| Docker Image | Immutable application package containing runtime and dependencies |
| eksctl | CLI tool for Amazon EKS cluster provisioning and management |
| Helm | Kubernetes package manager — used here for Load Balancer Controller |
| Ingress | Kubernetes resource managing external HTTP routing into the cluster |
| IRSA | IAM Roles for Service Accounts — scoped AWS permissions for Kubernetes workloads |
| Namespace | Logical isolation boundary for Kubernetes resources |
| Persistent Volume | Storage resource existing independently of Pod lifecycle |
| Persistent Volume Claim | Workload request for persistent storage |
| Pod | Smallest deployable unit in Kubernetes |
| ReplicaSet | Controller maintaining the required number of running Pod replicas |
| Service | Stable network endpoint abstracting individual Pod IP addresses |

### References

**AWS Documentation**
- Amazon EKS Documentation — https://docs.aws.amazon.com/eks
- Amazon ECR Documentation — https://docs.aws.amazon.com/ecr
- AWS IAM Documentation — https://docs.aws.amazon.com/iam
- AWS Load Balancer Controller — https://kubernetes-sigs.github.io/aws-load-balancer-controller
- Amazon EBS Documentation — https://docs.aws.amazon.com/ebs

**Kubernetes Documentation**
- Kubernetes Official Documentation — https://kubernetes.io/docs
- Kubernetes Ingress — https://kubernetes.io/docs/concepts/services-networking/ingress
- Kubernetes Storage — https://kubernetes.io/docs/concepts/storage
- Kubernetes Deployments — https://kubernetes.io/docs/concepts/workloads/controllers/deployment

**Containerization**
- Docker Documentation — https://docs.docker.com
- Docker Best Practices — https://docs.docker.com/develop/dev-best-practices

---

## Final Statement

This project documents the hands-on deployment of a three-tier application on Amazon EKS — from infrastructure provisioning through containerization, Kubernetes orchestration, networking, persistent storage, security, and operational troubleshooting.

The implementation followed a reference architecture and focused on understanding how the components interact, debugging the real issues that came up, and validating the complete application flow end to end.

The goal was practical engineering experience — not just getting something running, but understanding why it works and what to do when it doesn't.

---

*Validated outcome: User → React Frontend → Node.js API → MongoDB → Persistent Storage*  
*Result: Complete three-tier application deployed on Amazon EKS, end-to-end communication validated, infrastructure cleaned up after completion.*

---

<div align="center">

**Three-Tier Application Deployment using Kubernetes on AWS EKS**  
Version 1.0 — Engineering Portfolio Document  
Anwarullah Khan Nuzair — Cloud & DevOps Engineer  
myselfnuzairkhan@gmail.com | https://github.com/Nuzairkhan

</div>
