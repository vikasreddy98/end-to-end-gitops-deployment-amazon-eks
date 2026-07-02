# ARCHITECTURE.md

# System Architecture

This document provides a detailed overview of the architecture used to deploy a Go web application on Amazon Elastic Kubernetes Service (EKS) using modern DevOps and GitOps practices.

Unlike the implementation guide, this document focuses on the design decisions, interactions between components, request flow, and deployment lifecycle.

---

# Architecture Overview

The project follows a modern GitOps deployment model where Git serves as the single source of truth for both the application source code and deployment configuration.

Application changes are automatically built through GitHub Actions and deployed to Amazon EKS using Argo CD, eliminating the need for manual Kubernetes deployments.

---

# High-Level Architecture

![Architecture Diagram](./architecture.png)

---

# System Components

The architecture consists of six major layers.

```text
Developer

↓

Source Control

↓

Continuous Integration

↓

Continuous Deployment

↓

Kubernetes Platform

↓

Application Delivery
```

Each layer has a dedicated responsibility.

---

# Component Breakdown

## 1. Developer

The deployment lifecycle begins when a developer pushes changes to the GitHub repository.

Responsibilities:

- Develop application features
- Commit source code
- Push changes to GitHub

Example:

```bash
git add .

git commit -m "Update application"

git push origin main
```

No manual deployment commands are required after this point.

---

## 2. GitHub Repository

GitHub serves as the central source of truth for the project.

The repository contains:

- Go application source code
- Dockerfile
- Kubernetes manifests
- Helm chart
- GitHub Actions workflow
- Project documentation

Git also provides:

- Version control
- Collaboration
- Change history
- Rollback capability

---

## 3. GitHub Actions (Continuous Integration)

GitHub Actions automates the build process whenever new code is pushed.

The workflow performs the following operations:

```text
Checkout Repository

↓

Setup Go

↓

Download Dependencies

↓

Run Static Analysis

↓

Build Docker Image

↓

Authenticate to Docker Hub

↓

Push Docker Image
```

### Responsibilities

- Validate code
- Prevent broken builds
- Build production images
- Publish images automatically

This eliminates repetitive manual build steps.

---

## 4. Docker Hub

Docker Hub stores the application container images.

Instead of Kubernetes building images directly, Kubernetes simply pulls pre-built images from Docker Hub.

Benefits include:

- Central image repository
- Versioned images
- Reusable deployments
- Consistent application runtime

---

## 5. Argo CD (GitOps)

Argo CD continuously monitors the deployment configuration stored in Git.

Whenever the desired state changes, Argo CD automatically synchronizes the Kubernetes cluster.

Unlike traditional deployment pipelines, Argo CD does not require manually running:

```bash
kubectl apply
```

Instead, Git becomes the deployment interface.

### Responsibilities

- Watch Git repository
- Detect changes
- Compare desired state with cluster state
- Synchronize automatically
- Perform rolling deployments

---

## 6. Amazon EKS

Amazon Elastic Kubernetes Service provides the managed Kubernetes control plane.

The cluster hosts the application and continuously maintains the desired application state.

The EKS cluster contains:

- Worker Nodes
- Deployments
- Pods
- Services
- Ingress Resources

AWS manages the Kubernetes control plane while worker nodes run the application workloads.

---

# Kubernetes Architecture

Within the cluster, Kubernetes resources work together to expose the application.

```text
Deployment

↓

ReplicaSet

↓

Pods

↓

Service

↓

Ingress

↓

Ingress Controller
```

---

## Deployment

The Deployment resource defines the desired state of the application.

Responsibilities:

- Create Pods
- Replace failed Pods
- Perform rolling updates
- Maintain replica count

Deployments enable Kubernetes' self-healing capabilities.

---

## ReplicaSet

ReplicaSets ensure that the requested number of Pods remain available.

If a Pod crashes, Kubernetes automatically creates another one.

Developers rarely interact directly with ReplicaSets because Deployments manage them automatically.

---

## Pods

Pods are the smallest deployable unit in Kubernetes.

Each Pod runs:

- Go application
- Distroless container image

Pods are ephemeral.

If one fails, Kubernetes replaces it automatically.

---

## Service

Pods receive dynamic IP addresses.

A Service provides a permanent endpoint that other Kubernetes resources can use.

Benefits:

- Stable networking
- Internal load balancing
- Service discovery

---

## Ingress

Ingress provides HTTP routing.

It maps:

```
go-web.local

↓

Service
```

instead of exposing Pods directly.

Ingress supports:

- Host-based routing
- Path-based routing
- TLS termination

---

## NGINX Ingress Controller

The Ingress resource only defines routing rules.

The NGINX Ingress Controller implements those rules by configuring a reverse proxy.

Without an Ingress Controller:

- Ingress resources do nothing.

---

# End-to-End Request Flow

The following diagram illustrates how an end user accesses the application.

```text
Browser

↓

DNS Resolution

↓

AWS Load Balancer

↓

NGINX Ingress Controller

↓

Ingress Resource

↓

ClusterIP Service

↓

Application Pod
```

---

## Step 1

The user enters:

```
http://go-web.local/courses
```

---

## Step 2

The local hosts file resolves the hostname.

---

## Step 3

Traffic reaches the AWS Load Balancer.

---

## Step 4

The Load Balancer forwards traffic to the NGINX Ingress Controller.

---

## Step 5

The Ingress Controller evaluates the routing rules defined in the Ingress resource.

---

## Step 6

Traffic is forwarded to the ClusterIP Service.

---

## Step 7

The Service forwards traffic to one of the healthy application Pods.

---

## Step 8

The application processes the request and returns the response to the browser.

---

# Continuous Integration Flow

Every push triggers GitHub Actions.

```text
Developer

↓

Push Code

↓

GitHub Actions

↓

Checkout Repository

↓

Run Linter

↓

Build Docker Image

↓

Push Image to Docker Hub
```

No manual image builds are required.

---

# Continuous Deployment Flow

Deployment is managed using GitOps.

```text
Git Repository

↓

Argo CD

↓

Detect Changes

↓

Synchronize Cluster

↓

Rolling Update

↓

Application Available
```

Argo CD continuously reconciles the desired cluster state with the actual cluster state.

---

# Networking Architecture

The application uses several networking layers.

```text
Browser

↓

Local DNS

↓

AWS Load Balancer

↓

NGINX Ingress Controller

↓

Ingress

↓

ClusterIP Service

↓

Pods
```

Each layer has a specific responsibility.

| Component | Responsibility |
|------------|----------------|
| DNS | Resolve hostname |
| Load Balancer | Accept external traffic |
| Ingress Controller | Reverse proxy |
| Ingress | Routing rules |
| Service | Internal load balancing |
| Pod | Application execution |

---

# Security Considerations

Several security best practices were implemented.

## Multi-stage Docker Build

Only the compiled application is included in the runtime image.

---

## Distroless Images

The runtime image contains no unnecessary operating system packages.

Benefits include:

- Reduced attack surface
- Smaller images
- Fewer vulnerabilities

---

## GitHub Secrets

Sensitive information such as Docker Hub credentials is stored as encrypted GitHub Secrets rather than hardcoded in workflows.

---

## Kubernetes Secrets

Argo CD credentials are stored securely within Kubernetes Secrets.

---

# Design Decisions

## Why Amazon EKS?

Amazon EKS provides a managed Kubernetes control plane, reducing operational overhead while maintaining compatibility with upstream Kubernetes.

---

## Why Docker?

Docker packages the application and its dependencies into a portable, reproducible container image that can run consistently across environments.

---

## Why Helm?

Helm replaces repetitive Kubernetes YAML files with reusable templates and centralized configuration through `values.yaml`.

---

## Why GitHub Actions?

GitHub Actions integrates directly with the source repository and automates the build pipeline without requiring additional CI infrastructure.

---

## Why GitOps?

GitOps makes Git the single source of truth for deployments.

Benefits include:

- Version-controlled infrastructure
- Auditable changes
- Automatic reconciliation
- Simplified rollbacks

---

## Why Argo CD?

Argo CD continuously compares the desired state stored in Git with the actual state of the Kubernetes cluster.

If they differ, Argo CD automatically reconciles the cluster.

---

# Scalability Considerations

The architecture can be extended for production environments by introducing:

- Horizontal Pod Autoscaler (HPA)
- Cluster Autoscaler
- Amazon ECR
- Route 53
- cert-manager
- Prometheus
- Grafana
- ExternalDNS
- Argo CD Image Updater

These additions improve scalability, observability, security, and operational maturity.

---

# Conclusion

This architecture demonstrates a complete cloud-native application delivery workflow using modern DevOps and GitOps practices.

Beginning with a simple Go application, the project evolves into an automated deployment pipeline that combines:

- Containerization
- Kubernetes orchestration
- Managed cloud infrastructure
- Continuous Integration
- GitOps Continuous Deployment

The resulting system closely resembles deployment workflows used in modern production environments while remaining simple enough to understand, reproduce, and extend.
