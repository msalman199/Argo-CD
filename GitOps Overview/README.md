# GitOps with Argo CD

A hands-on GitOps lab demonstrating how to manage Kubernetes applications using **Git as the single source of truth** and **Argo CD** as the continuous delivery and synchronization engine.

This project creates a local Kubernetes environment with **Kind**, stores Kubernetes manifests in a Git repository, deploys an NGINX application through Argo CD, and demonstrates important GitOps capabilities such as automated synchronization, drift detection, self-healing, and Git-based auditability.

---

## 📌 Overview

GitOps is an operational methodology that uses Git repositories as the authoritative source for application and infrastructure configuration.

In this lab:

* Kubernetes runs locally using **Kind**
* Kubernetes manifests are stored in Git
* **Argo CD** continuously monitors the desired state
* Applications are deployed from Git
* Configuration changes are made through Git commits
* Argo CD detects configuration drift
* Argo CD automatically restores the desired state
* Git provides an auditable history of deployment changes

The lab is designed to run on a single Linux-based cloud machine and includes all required installation and configuration steps.

---

## 🎯 Objectives

By completing this project, you will learn how to:

* Understand the principles and benefits of GitOps
* Configure a Git repository for GitOps workflows
* Create Kubernetes manifests using GitOps practices
* Install and configure Argo CD
* Deploy Kubernetes applications through Argo CD
* Use Git as the source of truth
* Configure automated synchronization
* Demonstrate drift detection
* Test Argo CD self-healing
* Monitor application health and synchronization
* Troubleshoot common GitOps deployment problems

---

## 🏗️ Architecture

```text
                  ┌──────────────────────┐
                  │      Git Repository  │
                  │                      │
                  │ Kubernetes Manifests │
                  │ Argo CD Configuration │
                  └──────────┬───────────┘
                             │
                             │ Git
                             ▼
                  ┌──────────────────────┐
                  │       Argo CD        │
                  │                      │
                  │ GitOps Controller    │
                  │ Sync Engine          │
                  │ Drift Detection      │
                  │ Self-Healing         │
                  └──────────┬───────────┘
                             │
                             │ Sync
                             ▼
                  ┌──────────────────────┐
                  │   Kind Kubernetes    │
                  │      Cluster         │
                  │                      │
                  │  sample-app Namespace│
                  │  NGINX Deployment    │
                  │  NGINX Service       │
                  └──────────────────────┘
```

---

## 🛠️ Technologies Used

| Technology     | Purpose                                     |
| -------------- | ------------------------------------------- |
| **Git**        | Source control and desired-state management |
| **Kubernetes** | Container orchestration                     |
| **Kind**       | Local Kubernetes cluster                    |
| **kubectl**    | Kubernetes command-line management          |
| **Argo CD**    | GitOps continuous delivery                  |
| **Kustomize**  | Kubernetes manifest organization            |
| **Docker**     | Container runtime                           |
| **NGINX**      | Sample application                          |
| **YAML**       | Declarative configuration                   |
| **Linux**      | Lab operating environment                   |

---

## 📋 Prerequisites

Before starting, you should have:

* Basic Git knowledge
* Basic Kubernetes knowledge
* Familiarity with Pods, Services, and Deployments
* Linux command-line experience
* Understanding of YAML
* Basic Docker/container knowledge

---

## 💻 Lab Environment

The project is designed for a Linux-based cloud machine.

The original lab uses an **Al Nafi Cloud Machine** with sufficient resources to run:

* Docker
* Kind
* Kubernetes
* Argo CD
* Git repositories

---

# 🚀 Installation

## 1. Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

Install the required utilities:

```bash
sudo apt install -y curl wget git vim nano tree jq
```

---

## 2. Install Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Add the current user to the Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker --version
```

---

## 3. Install Kind

Install Kind for running Kubernetes locally:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Verify:

```bash
kind --version
```

---

## 4. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

The lab uses Kind and kubectl to create and manage the local Kubernetes environment.

---

# ☸️ Create the Kubernetes Cluster

Create a Kind configuration:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 8080
    protocol: TCP
  - containerPort: 443
    hostPort: 8443
    protocol: TCP
```

Create the cluster:

```bash
kind create cluster \
  --config=kind-config.yaml \
  --name=gitops-lab
```

Verify:

```bash
kubectl cluster-info
kubectl get nodes
```

---

# 📁 GitOps Repository

Create the project:

```bash
mkdir -p ~/gitops-demo
cd ~/gitops-demo
```

Initialize Git:

```bash
git init

git config user.name "GitOps Student"
git config user.email "student@example.com"
```

Create the repository structure:

```bash
mkdir -p applications/sample-app
mkdir -p argocd-config
```

Recommended structure:

```text
gitops-demo/
├── applications/
│   └── sample-app/
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
│
├── argocd-config/
│   └── sample-app-application.yaml
│
└── README.md
```

This structure separates application manifests from Argo CD configuration.

---

# 📦 Sample NGINX Application

The sample application consists of:

### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: sample-app
  labels:
    managed-by: argocd
```

### Deployment

The deployment runs two NGINX replicas initially:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: sample-app
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

### Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- namespace.yaml
- deployment.yaml
- service.yaml

commonLabels:
  managed-by: argocd
  environment: demo
```

The original lab defines the application using a Namespace, Deployment, Service, and Kustomization.

---

# 📝 Commit the Application

```bash
git add applications/
git commit -m "Add sample nginx application manifests"
```

View history:

```bash
git log --oneline
```

View the repository:

```bash
tree
```

---

# 🚢 Install Argo CD

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for the Argo CD server:

```bash
kubectl wait \
  --for=condition=available \
  --timeout=300s \
  deployment/argocd-server \
  -n argocd
```

Verify:

```bash
kubectl get pods -n argocd
```

---

# 🌐 Access Argo CD

Retrieve the initial administrator password:

```bash
ARGOCD_PASSWORD=$(kubectl -n argocd \
  get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

echo "$ARGOCD_PASSWORD"
```

Start port forwarding:

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443 &
```

Argo CD will be accessible at:

```text
https://localhost:8080
```

Default username:

```text
admin
```

Install the Argo CD CLI:

```bash
curl -sSL \
  -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

Login:

```bash
argocd login localhost:8080 \
  --username admin \
  --password "$ARGOCD_PASSWORD" \
  --insecure
```

---

# 🔗 Configure the Git Repository

Get the repository path:

```bash
REPO_PATH=$(pwd)

echo "$REPO_PATH"
```

Register it with Argo CD:

```bash
argocd repo add "$REPO_PATH" \
  --type git \
  --name local-gitops-repo
```

Verify:

```bash
argocd repo list
```

---

# 🎯 Create the Argo CD Application

Create:

```text
argocd-config/sample-app-application.yaml
```

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-nginx-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: /path/to/gitops-demo
    targetRevision: HEAD
    path: applications/sample-app

  destination:
    server: https://kubernetes.default.svc
    namespace: sample-app

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
    - CreateNamespace=true
```

Commit the configuration:

```bash
git add argocd-config/
git commit -m "Add Argo CD application configuration"
```

The application configuration tells Argo CD which Git repository and path contain the desired Kubernetes state.

---

# 🚀 Deploy the Application

Apply the Argo CD Application:

```bash
kubectl apply \
  -f argocd-config/sample-app-application.yaml
```

Check the application:

```bash
argocd app get sample-nginx-app
```

List applications:

```bash
argocd app list
```

Verify Kubernetes resources:

```bash
kubectl get all -n sample-app
```

---

# 🔄 Test the GitOps Workflow

Change the desired state in Git.

For example, scale NGINX from 2 to 3 replicas:

```bash
sed -i 's/replicas: 2/replicas: 3/' \
  applications/sample-app/deployment.yaml
```

Commit:

```bash
git add applications/sample-app/deployment.yaml

git commit -m "Scale nginx deployment to 3 replicas"
```

Synchronize:

```bash
argocd app sync sample-nginx-app
```

Verify:

```bash
kubectl get deployment nginx-deployment \
  -n sample-app

kubectl get pods \
  -n sample-app
```

This demonstrates the fundamental GitOps workflow:

```text
Developer
   │
   ▼
Git Change
   │
   ▼
Git Commit
   │
   ▼
Argo CD Detects Change
   │
   ▼
Synchronization
   │
   ▼
Kubernetes Desired State
```

---

# 🩹 Test Drift Detection & Self-Healing

GitOps also protects the cluster from configuration drift.

Manually change the deployment:

```bash
kubectl scale deployment nginx-deployment \
  --replicas=5 \
  -n sample-app
```

Check the current state:

```bash
kubectl get deployment nginx-deployment \
  -n sample-app
```

Wait for Argo CD to detect the drift:

```bash
sleep 60
```

Verify:

```bash
kubectl get deployment nginx-deployment \
  -n sample-app

argocd app get sample-nginx-app
```

Because `selfHeal: true` is enabled, Argo CD can restore the cluster to the state defined in Git.

---

# 🔧 Update Application Configuration

Update the NGINX image:

```bash
sed -i 's/nginx:1.21/nginx:1.22/' \
  applications/sample-app/deployment.yaml
```

Add an environment variable:

```yaml
env:
- name: ENVIRONMENT
  value: "gitops-demo"
```

Commit:

```bash
git add applications/sample-app/deployment.yaml

git commit -m \
  "Update nginx to version 1.22 and add environment variable"
```

Sync:

```bash
argocd app sync sample-nginx-app
```

Verify:

```bash
kubectl describe deployment nginx-deployment \
  -n sample-app

kubectl get pods \
  -n sample-app \
  -o wide
```

---

# 📊 Monitor Application Health

Get detailed application information:

```bash
argocd app get sample-nginx-app --show-params
```

Wait for the application to become healthy:

```bash
argocd app wait sample-nginx-app --health
```

View deployment history:

```bash
argocd app history sample-nginx-app
```

Check Argo CD Applications:

```bash
kubectl get applications -n argocd
```

---

# 🧹 Cleanup

Delete the Argo CD application:

```bash
argocd app delete sample-nginx-app --cascade
```

Verify:

```bash
kubectl get all -n sample-app
kubectl get namespace sample-app
```

Stop port forwarding:

```bash
kill $PORT_FORWARD_PID 2>/dev/null || true
```

Optionally delete the Kind cluster:

```bash
kind delete cluster --name=gitops-lab
```

---

# 🔍 Troubleshooting

## Argo CD Pods Are Not Starting

Check the pods:

```bash
kubectl get pods -n argocd
```

Check server logs:

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-server
```

Restart the server:

```bash
kubectl rollout restart \
  deployment/argocd-server \
  -n argocd
```

---

## Application Sync Failure

Inspect the Application:

```bash
kubectl describe application \
  sample-nginx-app \
  -n argocd
```

Check sync operation details:

```bash
argocd app get \
  sample-nginx-app \
  --show-operation
```

Force synchronization:

```bash
argocd app sync \
  sample-nginx-app \
  --force
```

---

## Repository Connection Problems

Check configured repositories:

```bash
argocd repo list
```

Test the repository:

```bash
argocd repo get "$REPO_PATH"
```

These troubleshooting procedures are included in the original lab for Argo CD startup, synchronization, and repository connectivity issues.

---

# 🧠 Key GitOps Concepts

## Declarative Configuration

The desired Kubernetes state is represented using YAML manifests stored in Git.

## Git as the Source of Truth

Git contains the authoritative desired state for the application.

## Automated Synchronization

Argo CD compares Git with the Kubernetes cluster and synchronizes changes.

## Drift Detection

Argo CD detects differences between the desired state and the actual cluster state.

## Self-Healing

Argo CD can automatically restore resources when they are changed outside the GitOps workflow.

## Audit Trail

Git commits provide a history of application configuration and deployment changes.

---

# 📚 What This Project Demonstrates

By completing this project, you have implemented a complete local GitOps workflow:

```text
Git Repository
      │
      │ Desired State
      ▼
   Argo CD
      │
      │ Synchronization
      ▼
 Kubernetes Cluster
      │
      │ Application
      ▼
  NGINX Deployment
```

You also demonstrated:

* Git-based deployment
* Declarative Kubernetes configuration
* Argo CD application management
* Automated synchronization
* Manual synchronization
* Drift detection
* Self-healing
* Application health monitoring
* Deployment history
* Git-based auditability
* Kubernetes resource management

The lab's final objectives specifically include establishing a GitOps environment, configuring the Git repository, installing Argo CD, deploying applications through Git, and demonstrating drift detection, self-healing, and synchronization.

---

# 🎓 Learning Outcomes

After completing this project, you should be able to explain and demonstrate:

1. What GitOps is and why it is useful.
2. Why Git can serve as the source of truth.
3. How Argo CD connects Git repositories to Kubernetes.
4. How declarative Kubernetes manifests represent desired state.
5. How automated synchronization works.
6. How configuration drift is detected.
7. How self-healing restores the desired state.
8. How Git provides an audit trail for operational changes.
9. How to troubleshoot common Argo CD deployment issues.

---

# ⭐ Conclusion

This project provides a practical introduction to **GitOps using Kubernetes and Argo CD**. It demonstrates how traditional deployment operations can be transformed into a declarative, Git-driven workflow.

Instead of manually changing Kubernetes resources, changes are committed to Git and Argo CD reconciles the Kubernetes cluster with the desired state.

The result is a deployment model that improves **consistency, reliability, traceability, and operational control** while providing a foundation for managing larger application environments with GitOps principles.

---

## 🏷️ Technologies

`GitOps` `Argo CD` `Kubernetes` `Kind` `kubectl` `Docker` `Kustomize` `NGINX` `Linux` `YAML` `Continuous Delivery`

---

## 📄 License

This project is intended for educational and lab purposes.
