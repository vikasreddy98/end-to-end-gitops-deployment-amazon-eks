# TROUBLESHOOTING.md

# Troubleshooting Guide

This document captures the issues encountered during the implementation of this project and the steps taken to resolve them.

Rather than simply documenting fixes, this guide explains the investigation process, root causes, and verification steps. Many of these issues are commonly encountered when working with Docker, Kubernetes, Amazon EKS, GitHub Actions, and Argo CD.

---

# Docker

---

# Issue 1 - Docker Build Failed Due to Go Version Mismatch

## Symptoms

The Docker image failed during the dependency download stage.

```text
go.mod requires go >= 1.22.5
(running go 1.21.13)
```

---

## Investigation

The Dockerfile specified:

```Dockerfile
FROM golang:1.21
```

However, the application's `go.mod` file required:

```text
go 1.22.5
```

Docker builds execute inside the container environment, meaning the Go version inside the Docker image must satisfy the version defined by the project.

---

## Root Cause

The Docker base image was using an older Go version than required by the application.

---

## Resolution

Update the Docker base image.

```Dockerfile
FROM golang:1.22.5
```

Rebuild the image.

```bash
docker build -t username/go-web-app .
```

---

## Verification

The build completed successfully without dependency errors.

---



![](docs/screenshots/02-docker-build-error.png)


---

## Lessons Learned

Always verify that the Docker build environment matches the application's runtime requirements.

---

# Kubernetes

---

# Issue 2 - Pod Stuck in ImagePullBackOff

## Symptoms

Pods never reached the Running state.

```bash
kubectl get pods
```

Output:

```text
ImagePullBackOff
```

---



![](docs/screenshots/11-imagepullbackoff-error.png)


---

## Initial Investigation

Several common causes were investigated.

- Incorrect image name
- Incorrect repository
- Missing image
- Incorrect tag

The Deployment manifest was verified.

The Docker Hub repository was verified.

The image was rebuilt and pushed again.

The issue still remained.

---

## Root Cause

The Pod Events revealed:

```text
no match for platform in manifest
```

The Docker image had only been built for the local architecture.

Amazon EKS worker nodes expected a different architecture.

---

## Resolution

Build a multi-platform image using Docker Buildx.

```bash
docker buildx build \
--platform linux/amd64,linux/arm64 \
-t username/go-web-app:v1 \
--push .
```

Restart the Deployment.

```bash
kubectl rollout restart deployment go-web-app
```

---

## Verification

```bash
kubectl get pods
```

Pods entered the Running state.

---



![](docs/screenshots/12-pod-running.png)


---

## Lessons Learned

ImagePullBackOff is not always caused by authentication or missing images.

Always inspect Pod events using:

```bash
kubectl describe pod <pod-name>
```

before assuming the cause.

---

# Issue 3 - Application Not Accessible Through Ingress

## Symptoms

The Ingress resource existed.

However,

```bash
kubectl get ingress
```

showed:

```text
ADDRESS

<pending>
```

The application could not be accessed externally.

---

## Investigation

The Ingress manifest itself appeared correct.

Deployment:

✔

Service:

✔

Pods:

✔

Only the Ingress lacked an external address.

---

## Root Cause

An Ingress resource only defines routing rules.

It does not actually process traffic.

A Kubernetes Ingress Controller was missing.

---

## Resolution

Install the NGINX Ingress Controller.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/aws/deploy.yaml
```

---

## Verification

```bash
kubectl get ingress
```

The ADDRESS field was now populated.

---



![](docs/screenshots/14-ingress-controller-created.png)


---

## Lessons Learned

Ingress resources require an Ingress Controller.

Creating an Ingress object alone does not expose an application.

---

# Issue 4 - HTTP 404 After Ingress Was Created

## Symptoms

The Ingress Controller was running.

The Ingress had an ADDRESS.

Opening the AWS hostname returned:

```text
404 Not Found
```

---

## Investigation

The Ingress configuration specified a custom Host value.

Requests sent directly to the AWS hostname used a different HTTP Host header.

As a result, the routing rule never matched.

---

## Root Cause

Host-based routing requires the incoming request's Host header to match the value configured in the Ingress manifest.

---

## Resolution

Map the configured hostname locally.

```bash
sudo vim /etc/hosts
```

Add:

```text
<Ingress-IP>

go-web.local
```

---

## Verification

Open:

```
http://go-web.local/courses
```

The application loaded successfully.

---



![](docs/screenshots/15-dns-mapping.png)


---


![](docs/screenshots/16-app-accessed-host.png)


---

## Lessons Learned

Host-based routing depends on DNS resolution.

Always verify both:

- DNS
- Host header

when troubleshooting Ingress issues.

---

# GitHub Actions

---

# Issue 5 - golangci-lint Failed

## Symptoms

GitHub Actions failed during the linting stage.

```text
golangci-lint exited with code 3
```

---



![](docs/screenshots/27-ci-error.png)


---

## Investigation

The workflow configuration appeared correct.

The issue was traced to an incompatible GitHub Action version.

---

## Resolution

Update the action.

```yaml
uses: golangci/golangci-lint-action@v8

with:

version: v2.2.2
```

Commit the workflow.

Push again.

GitHub Actions automatically retriggered.

---

## Verification

The workflow completed successfully.

---



![](docs/screenshots/18-ci-test-success.png)


---

## Lessons Learned

CI failures are not always caused by application code.

Workflow dependencies should also be kept up to date.

---

# Argo CD

---

# Issue 6 - Unable to Access the Argo CD UI

## Symptoms

Argo CD was installed successfully.

However,

opening the dashboard from the browser failed.

---

## Investigation

The `argocd-server` Service was configured as:

```text
ClusterIP
```

ClusterIP Services are only accessible from inside the Kubernetes cluster.

---

## Root Cause

The Service needed external exposure.

---

## Resolution

Patch the Service.

```bash
kubectl patch svc argocd-server \
-n argocd \
-p '{"spec":{"type":"LoadBalancer"}}'
```

---

## Verification

```bash
kubectl get svc -n argocd
```

The Service received an external endpoint.

The dashboard became accessible.

---



![](docs/screenshots/20-argocd-installed.png)


---

# General Debugging Commands

The following commands proved useful throughout the project.

## Kubernetes

```bash
kubectl get all

kubectl get pods

kubectl get svc

kubectl get ingress

kubectl describe pod <pod>

kubectl logs <pod>

kubectl rollout restart deployment <deployment>

kubectl get events
```

---

## Docker

```bash
docker ps

docker images

docker build

docker buildx build

docker logs <container>
```

---

## Helm

```bash
helm list

helm install

helm uninstall
```

---

## GitHub Actions

Review:

- Workflow logs
- Job output
- Secrets configuration

---

## Argo CD

Check:

- Application Health
- Sync Status
- Resource Tree
- Events

---

# Key Takeaways

Throughout this project, the majority of issues fell into one of four categories:

- Version mismatches
- Platform incompatibilities
- Networking and DNS configuration
- Service exposure

The most valuable lesson was to investigate methodically rather than assuming the cause of an issue.

Commands such as `kubectl describe`, `kubectl logs`, and GitHub Actions workflow logs provided the information needed to identify the true root cause in each case.