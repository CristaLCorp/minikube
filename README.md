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

```bash
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable dashboard
minikube dashboard &
```
The dashboard will then be accessible at : http://120.0.0.1:58420

### Context
In Kubernetes, a context defines which cluster, which user, and which namespace your kubectl command is targeting.  
A context is just a named configuration that wraps these three pieces:  
* Cluster (e.g., minikube, prod-cluster)  
* User (e.g., admin, developer)  
* Namespace (optional; e.g., default, dev, staging)  
In our case, we will use minikube as the context :
```bash
kubectl config use-context minikube
```

### Testing Minikube
Deploy a quick app to test:
```bash
kubectl create deployment hello-minikube --image=kicbase/echo-server:1.0
kubectl expose deployment hello-minikube --type=NodePort --port=8081
minikube service hello-minikube
```

This should open your browser with a working echo server. 

You can then delete your deployment, like this :
```bash
kubectl delete deployment hello-minikube -n default 
```
Specifying the namespace is a good habit to avoid mistakes...

Congratz ! Your cluster is working !

Let s move on into making it production ready.

## Multi-Environment GitOps Layout
Let s set up our cluster with a multi-env production ready directory layout. It will look like this :
```graphql
gitops-repo/
├── .gitignore
├── .github/
│   └── workflows/
│       └── ci-cd.yaml
├── apps/
│   ├── hello-nginx-1/
│   │   ├── dev/
│   │   │   ├── templates
│   │   │   │   ├── _helpers.tpl
│   │   │   │   ├── deployment.yaml
│   │   │   │   └── service.yaml
│   │   │   ├── values.yaml
│   │   │   └── Chart.yaml
│   │   ├── staging/
│   │   │   ├── templates
│   │   │   │   ├── _helpers.tpl
│   │   │   │   ├── deployment.yaml
│   │   │   │   └── service.yaml
│   │   │   ├── values.yaml
│   │   │   └── Chart.yaml
│   │   ├── prod/
│   │   │   ├── templates
│   │   │   │   ├── _helpers.tpl
│   │   │   │   ├── deployment.yaml
│   │   │   │   └── service.yaml
│   │   │   ├── values.yaml
│   │   │   └── Chart.yaml
│   └── hello-nginx-2/
│       ├── dev/
│       │   ├── templates
│       |   |   ├── _helpers.tpl
│       |   |   ├── deployment.yaml
│       |   |   └── service.yaml
│       │   ├── values.yaml
│       │   └── Chart.yaml
│       ├── staging/
│       │   ├── templates
│       |   |   ├── _helpers.tpl
│       |   |   ├── deployment.yaml
│       |   |   └── service.yaml
│       │   ├── values.yaml
│       │   └── Chart.yaml
│       ├── prod/
│       │   ├── templates
│       |   |   ├── _helpers.tpl
│       |   |   ├── deployment.yaml
│       |   |   └── service.yaml
│       │   ├── values.yaml
│       │   └── Chart.yaml
├── charts/
│   └── nginx/
│       └── templates/                  # templates present in every app/namespace 
│       |   ├── _helpers.tpl
│       |   ├── deployment.yaml
│       |   └── service.yaml
|       ├── values.yaml                 # default values if not override locally
|       └── Charts.yaml
└── argocd/                             # apps that are being monitored by argocd
    ├── hello-nginx-1-dev.yaml
    ├── hello-nginx-1-staging.yaml
    ├── hello-nginx-1-prod.yaml
    ├── hello-nginx-2-dev.yaml
    ├── hello-nginx-2-staging.yaml
    └── hello-nginx-2-prod.yaml
```
* **charts/nginx/**: shared chart logic (templates, defaults) (#TODO : use Kustomize to reduce overhead)

* **apps/hello-nginx-{1,2}/dev**: overrides for dev (e.g., fewer replicas, latest tag)

* **apps/hello-nginx-{1,2}/prod**: overrides for prod (e.g., fixed tag, tighter resources)

* **argocd/**: apps that argocd should watch