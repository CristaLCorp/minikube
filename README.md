# Minikube - A Prod Ready DevSecOps Love Story

## Prerequisites

For this story to work, you need to have the following up and running :
* [Docker](https://docs.docker.com/engine/install/) ( or [Docker Desktop](https://www.docker.com/products/docker-desktop/) )
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download)

## Minikube
We are going to start a cluster with Docker as the driver and enough resources for Helm, ArgoCD, Prometheus, Keycloak, etc.
```bash
minikube start --driver=docker --cpus=4 --memory=8192 --disk-size=30g
```
Check everything is running :
```bash
kubectl get nodes
kubectl cluster-info
```

### Addons
