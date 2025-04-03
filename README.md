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
```
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
│       │   │   ├── _helpers.tpl
│       │   │   ├── deployment.yaml
│       │   │   └── service.yaml
│       │   ├── values.yaml
│       │   └── Chart.yaml
│       ├── staging/
│       │   ├── templates
│       │   │   ├── _helpers.tpl
│       │   │   ├── deployment.yaml
│       │   │   └── service.yaml
│       │   ├── values.yaml
│       │   └── Chart.yaml
│       ├── prod/
│       │   ├── templates
│       │   │   ├── _helpers.tpl
│       │   │   ├── deployment.yaml
│       │   │   └── service.yaml
│       │   ├── values.yaml
│       │   └── Chart.yaml
├── charts/
│   └── nginx/
│       ├── templates/                  # templates present in every app/namespace 
│       │   ├── _helpers.tpl
│       │   ├── deployment.yaml
│       │   └── service.yaml
│       ├── values.yaml                 # default values if not override locally
│       └── Charts.yaml
└── argocd/                             # apps that are being monitored by argocd
    ├── hello-nginx-1-dev.yaml
    ├── hello-nginx-1-staging.yaml
    ├── hello-nginx-1-prod.yaml
    ├── hello-nginx-2-dev.yaml
    ├── hello-nginx-2-staging.yaml
    └── hello-nginx-2-prod.yaml
```

* **apps/hello-nginx-{1,2}/dev**: overrides for dev (e.g., fewer replicas, latest tag)

* **apps/hello-nginx-{1,2}/prod**: overrides for prod (e.g., fixed tag, tighter resources)

* **charts/nginx/**: shared chart logic (templates, defaults) (#TODO : use Kustomize to reduce overhead)

* **argocd/**: config files for apps that argocd will watch

### ArgoCD : Setup

### ArgoCD : Config
 Port Forwarding (makes it available at this url : http://127.0.0.1:8080):
 ```bash
 kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Login :
```bash
argocd login localhost:8080
```
It will prompt for:
* Username: admin
* Password: (you can retrieve from secret — see below)
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Use that as the password for admin.

### ArgoCD : App deployment
You first need to create a namespace for every app. 

Let s check if it exists :
```bash
kubectl get ns hello-nginx-1-dev
```
If it does not, let s create it :
```bash
kubectl create ns hello-nginx-1-dev
```
Lets see what app are configured so far :
```bash
argocd app list
```
Found yours ? Inspect it :
```bash
argocd app get hello-nginx-1-dev
```
Look for:
* Status: Synced or OutOfSync
* Health: Healthy or Missing, Progressing, etc.
* Destination: namespace + server
* Any error message in the "Conditions" section

Finally sync the app if need be :
```bash
argocd app sync hello-nginx-1-dev
```

### ArgoCD : Debug
```bash
kubectl get applications -n argocd
kubectl logs -n argocd deploy/argocd-application-controller
kubectl get events -n hello-nginx-1-dev --sort-by=.metadata.creationTimestamp
```
