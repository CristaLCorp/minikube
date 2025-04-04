# Minikube - A Prod Ready DevSecOps Love Story

## Prerequisites

For this story to work, you need to have the following up and running :
* [Docker](https://docs.docker.com/engine/install/) ( or [Docker Desktop](https://www.docker.com/products/docker-desktop/) )
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86-64%2Fstable%2F.exe+download)
* [Helm](https://helm.sh/docs/intro/install/)

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
This will open a dashboard in your browser at : http://120.0.0.1:[random]

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
ArgoCD runs in its own namespace. Let’s deploy it using raw manifests:
```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Wait a bit, then check pods:
```bash
kubectl get pods -n argocd
```

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

## Helm
A typical Helm chart directory includes:​
```bash
nginx/
  ├── Chart.yaml
  ├── values.yaml
  ├── templates/
  │   ├── deployment.yaml
  │   ├── service.yaml
  │   └── ... (other templates)
  └── ... (optional files like README.md, LICENSE)
```
Key Components:

* Chart.yaml: Contains metadata about the chart, such as its name, version, and description.​
* values.yaml: Holds default configuration values that templates can reference.​
* templates/: Contains Kubernetes manifest templates that Helm renders using the values provided.

### Debugging
if you see this error :
```bash
[ERROR] templates/: template: nginx/templates/service.yaml:4:11:
executing "nginx/templates/service.yaml" at <include "nginx.fullname" .>:
error calling include: template: no template "nginx.fullname" associated with template "gotpl"
```
This means your Helm chart is using:
```yaml
{{ include "nginx.fullname" . }}
```
...but there’s no definition for "nginx.fullname" in your chart. These include helpers typically live in a file named:
```bash
charts/nginx/templates/_helpers.tpl
```
Quick fix ? 
Create a file at: charts/nginx/templates/_helpers.tpl

Add the following content (you can reuse this for all your apps):
```bash
{{- define "nginx.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end }}

{{- define "nginx.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "nginx.labels" -}}
app.kubernetes.io/name: {{ include "nginx.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```