# GitOps ArgoCD Kubernetes Project

<img width="4393" height="2161" alt="Untitled Diagram drawio (15)" src="https://github.com/user-attachments/assets/868bc218-21f7-44cd-b80f-79c0df9d8162" />

## Project Overview

This is a **GitOps-based deployment project** using **ArgoCD** to manage Kubernetes applications. The project demonstrates continuous synchronization of Git repository state with Kubernetes cluster state, enabling declarative infrastructure as code practices.

**Repository**: [hesxo/argocd-project](https://github.com/hesxo/argocd-project)

## Project Structure

```
argocd-project/
├── application.yaml          # ArgoCD Application manifest
├── README.md                 # Project documentation
└── dev/                      # Development environment configurations
    ├── deployment.yaml       # Kubernetes Deployment
    └── service.yaml          # Kubernetes Service
```

## Core Components

### 1. ArgoCD Application (`application.yaml`)

| Property | Details |
|----------|---------|
| **Application Name** | `myapp-argo-application` |
| **Namespace** | `argocd` |
| **Project** | `default` |
| **Git Repository** | `https://github.com/hesxo/argocd-project.git` |
| **Target Revision** | `main` |
| **Deployment Path** | `dev` |
| **Destination Cluster** | `https://kubernetes.default.svc` (in-cluster) |
| **Target Namespace** | `myapp` |

**Sync Policies**:

- **Auto-sync enabled** — continuous synchronization
- **Self-healing** — reconciles drift
- **Auto-prune** — removes resources deleted from Git
- **Create namespace** — auto-creates `myapp` namespace

### 2. Kubernetes Deployment (`dev/deployment.yaml`)

| Configuration | Value |
|---------------|-------|
| **Kind** | Deployment |
| **Name** | `myapp` |
| **Replicas** | 3 |
| **Container Image** | `nginx:latest` |
| **Container Port** | 8080 |
| **Selector Label** | `app: myapp` |

**Features**:

- Three pod replicas for redundancy
- Container selector: `app: myapp`
- Declared container port 8080 for the Service target

> **Note:** The stock `nginx` image listens on port **80** by default. If pods are not ready or probes fail, align `containerPort` / probes with port 80 or use a custom nginx config.

### 3. Kubernetes Service (`dev/service.yaml`)

| Configuration | Value |
|---------------|-------|
| **Kind** | Service |
| **Name** | `myapp-service` |
| **Service Port** | 8080 |
| **Target Port** | 8080 |
| **Protocol** | TCP |
| **Selector** | `app: myapp` |

**Purpose:** Internal service discovery and load balancing for the Deployment.

## GitOps Workflow

```
Git Repository (GitHub)
    |
Developer commits changes
    |
ArgoCD monitors Git repository
    |
Compare: Desired State (Git) vs Current State (Cluster)
    |
Auto-sync: Apply changes to Kubernetes
    |-- Self-heal: Correct manual drifts
    |-- Auto-prune: Remove deleted resources
    '-- Create namespaces if needed
    |
Application updates in Kubernetes cluster
(3 replicas in myapp namespace)
```

## Technology Stack & Languages

| Technology | Purpose | Version |
|------------|---------|---------|
| **Kubernetes (K8s)** | Container orchestration platform | 1.20+ |
| **ArgoCD** | GitOps continuous delivery tool | Latest stable |
| **Git** | Version control & source of truth | GitHub |
| **Docker** | Container runtime & image packaging | Latest |
| **YAML** | Infrastructure as Code configuration format | — |
| **Go** | ArgoCD implementation language | 1.16+ |
| **Shell/Bash** | Command-line scripting | — |

### Language breakdown

**YAML** — All Kubernetes and ArgoCD manifests (`application.yaml`, `dev/deployment.yaml`, `dev/service.yaml`).

**Kubernetes API** — `apiVersion: apps/v1` (Deployment), `v1` (Service), `argoproj.io/v1alpha1` (ArgoCD Application).

**Bash/Shell** — Installation, `kubectl`, and ArgoCD CLI usage.

**Go** — ArgoCD is implemented in Go (sync, reconciliation, API server).

**Docker** — Container image `nginx:latest`; port exposure as declared in the Deployment.

## Deployment Architecture

```
+--------------------------------------------+
|     Kubernetes Cluster                     |
+--------------------------------------------+
|                                            |
|  +--------------------------------------+  |
|  |   ArgoCD Namespace (argocd)          |  |
|  |  +--------------------------------+  |  |
|  |  | ArgoCD Controller (Go)         |  |  |
|  |  | (Monitors Git & Syncs)         |  |  |
|  |  | myapp-argo-application         |  |  |
|  |  +--------------------------------+  |  |
|  +--------------------------------------+  |
|                                            |
|  +--------------------------------------+  |
|  |   myapp Namespace (auto-created)     |  |
|  |  +--------------------------------+  |  |
|  |  | Deployment: myapp              |  |  |
|  |  | +- Pod 1 (container)           |  |  |
|  |  | +- Pod 2 (container)           |  |  |
|  |  | +- Pod 3 (container)           |  |  |
|  |  | Image: nginx:latest            |  |  |
|  |  | Port: 8080 (declared)          |  |  |
|  |  +--------------------------------+  |  |
|  |  +--------------------------------+  |  |
|  |  | Service: myapp-service         |  |  |
|  |  | Port: 8080 -> targetPort: 8080 |  |  |
|  |  +--------------------------------+  |  |
|  +--------------------------------------+  |
|                                            |
+--------------------------------------------+
        |
    GitHub Repository
    (Source of Truth)
```

## Security & Access Configuration

| Component | Configuration | Details |
|-----------|---------------|---------|
| **Namespace Isolation** | Yes | `argocd` and `myapp` separated |
| **RBAC** | Enabled | Default service accounts |
| **API Server** | In-cluster | `https://kubernetes.default.svc` |
| **Authentication** | Secret-based | `argocd-initial-admin-secret` |
| **Port Security** | Port-forward | No direct cluster exposure by default |

## Installation & Setup

### Prerequisites

```bash
kubectl version
helm version
git --version
```

### Step 1: Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

### Step 2: Access ArgoCD UI

```bash
kubectl get svc -n argocd
kubectl port-forward svc/argocd-server 8080:443 -n argocd
```

UI: `https://localhost:8080` (note: local port 8080 here is for the Argo CD server, not the app Service).

### Step 3: Get Admin Credentials

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
```

Username: `admin`

### Step 4: Change Initial Password (Recommended)

```bash
argocd account update-password --account admin --new-password <new-password>
# Or remove the bootstrap secret after changing password:
kubectl delete secret argocd-initial-admin-secret -n argocd
```

### Step 5: Deploy the Application

After you apply `application.yaml` (or register the app in ArgoCD), it will:

1. Create the `myapp` namespace (via sync option)
2. Deploy three replicas
3. Create the Service
4. Keep auto-sync, self-heal, and prune enabled

## Key Features

| Feature | Status | Benefit |
|---------|--------|---------|
| **Automated Synchronization** | Enabled | No manual apply for routine Git changes |
| **Self-Healing** | Enabled | Reconciles drift |
| **Auto-Prune** | Enabled | Removes orphaned resources |
| **Namespace Management** | Enabled | Auto-creates deployment namespace |
| **High Availability** | 3 replicas | Better availability |
| **Git-based Deployment** | GitHub | Version-controlled manifests |

## Current Status Verification

```bash
kubectl get application -n argocd
kubectl describe application myapp-argo-application -n argocd

kubectl get pods -n myapp
kubectl get svc -n myapp
kubectl get deployment -n myapp

kubectl describe pod -n myapp -l app=myapp
kubectl logs -n myapp -l app=myapp
```

## Update Workflow

1. Edit manifests under `dev/` (or `application.yaml`)
2. Commit and push to `main`
3. ArgoCD reconciles according to its poll interval (default ~3 minutes) or trigger a sync manually

Example:

```bash
git add dev/deployment.yaml
git commit -m "Update nginx image or replica count"
git push origin main
```

## Documentation & References

- [Argo CD documentation](https://argo-cd.readthedocs.io/)
- [Kubernetes documentation](https://kubernetes.io/docs/)
- [GitOps principles](https://www.gitops.tech/)

## Project Goals & Best Practices

- Declarative infrastructure in Git  
- Full change history via Git  
- Immutable deployments via container images  
- Automated sync from Git  
- Self-healing where configured  
- Multiple replicas for resilience  
- Single source of truth: the Git repository  

## Monitoring & Troubleshooting

### Application status

```bash
kubectl get application -A
argocd app list
argocd app describe myapp-argo-application
```

### Sync

```bash
argocd app sync myapp-argo-application
argocd app wait myapp-argo-application
```

### Debug

```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
kubectl describe application myapp-argo-application -n argocd
kubectl describe deployment myapp -n myapp
```

## Project Owner

**Repository**: [hesxo/argocd-project](https://github.com/hesxo/argocd-project)  
**Example image**: `nginx:latest`  
**Branch**: `main`

## License & Contribution

Follow GitOps practices: commit manifest changes to Git before expecting the cluster to match.

Last updated: April 11, 2026
