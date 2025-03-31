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
✅ Check everything is running :
```bash
kubectl get nodes
kubectl cluster-info
```

### Addons
To make our life easier we are going to install 3 addons :  
* **ingress** : Minikube [Ingress](## "In Kubernetes, an Ingress is a resource that manages external access to services, typically HTTP/HTTPS routes. It allows you to: Route traffic based on hostnames (e.g., app.local), Use path-based routing (e.g., /api, /dashboard), Terminate TLS (HTTPS)") Controller  
* **metrics-server** : A lightweight, resource-efficient service that gathers resource usage metrics (CPU, memory) from each node and pod, and exposes them through the Kubernetes API. Deploy [Kubernetes Metrics Server](https://github.com/kubernetes-sigs/metrics-server) into the local cluser. 
* **dashboard** : The [Kubernetes Dashboard](## "Provides a convenient graphical interface to inspect cluster resources (pods, deployments, services, etc.), view logs and events, scale deployments, edit YAML manifests directly in the browser, apply changes, restart pods, etc."), a web-based UI for managing and visualizing the cluster.