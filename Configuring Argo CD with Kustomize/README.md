# Configuring Argo CD with Kustomize

## 📌 Overview

This repository contains a hands-on lab for configuring **Argo CD with Kustomize** to implement a GitOps-based Kubernetes deployment workflow.

The lab demonstrates how to:

* Build a local Kubernetes environment with **kind**
* Install and configure **Argo CD**
* Manage Kubernetes manifests using **Kustomize**
* Create reusable base configurations
* Create environment-specific overlays
* Deploy separate development and production environments
* Configure Argo CD automated synchronization
* Implement self-healing and pruning
* Manage shared configuration through Kustomize components
* Test GitOps workflows by changing configuration in Git
* Troubleshoot common Argo CD and Kustomize problems

The lab is designed around a sample NGINX application and demonstrates how the same Kubernetes application can be customized for different environments without duplicating the complete manifest set.

---

## 🎯 Lab Objectives

By completing this lab, you will be able to:

* Install and configure Argo CD on Kubernetes
* Understand the fundamentals of Kustomize
* Create Kustomize bases and overlays
* Configure development and production environments
* Deploy Kustomize applications through Argo CD
* Manage application configuration through GitOps
* Configure automated synchronization
* Implement pruning and self-healing
* Create reusable Kustomize components
* Apply environment-specific configurations
* Verify application health and synchronization
* Troubleshoot common Argo CD and Kustomize issues

---

## 🏗️ Architecture

```text
                    ┌─────────────────────┐
                    │     Git Repository  │
                    │                     │
                    │  Kustomize Base     │
                    │  Development Overlay│
                    │  Production Overlay │
                    └──────────┬──────────┘
                               │
                               │ GitOps
                               ▼
                    ┌─────────────────────┐
                    │       Argo CD       │
                    │                     │
                    │  Application Dev    │
                    │  Application Prod   │
                    │  Auto Sync          │
                    │  Self Heal          │
                    │  Prune              │
                    └──────────┬──────────┘
                               │
                  ┌────────────┴────────────┐
                  ▼                         ▼
        ┌──────────────────┐      ┌──────────────────┐
        │ Development      │      │ Production       │
        │ Namespace        │      │ Namespace        │
        │                  │      │                  │
        │ 1 Replica        │      │ 3+ Replicas      │
        │ nginx Alpine     │      │ nginx            │
        │ DEBUG logging    │      │ WARN logging     │
        └──────────────────┘      └──────────────────┘
```

---

## 🧰 Technologies Used

| Technology | Purpose                             |
| ---------- | ----------------------------------- |
| Kubernetes | Container orchestration             |
| kind       | Local Kubernetes cluster            |
| Docker     | Container runtime                   |
| kubectl    | Kubernetes CLI                      |
| Kustomize  | Kubernetes configuration management |
| Argo CD    | GitOps continuous delivery          |
| Git        | Configuration and version control   |
| NGINX      | Sample application                  |
| YAML       | Kubernetes configuration            |

---

## 📋 Prerequisites

Before starting, you should have:

* Basic Kubernetes knowledge
* Understanding of Pods, Deployments, and Services
* Familiarity with YAML
* Basic Git knowledge
* Understanding of containers
* Linux command-line experience

The original lab uses an **Al Nafi Linux-based cloud machine** and starts from a bare-metal environment where the required tools are installed during the exercise.

---

## 🛠️ Environment Setup

### Install Docker

```bash
sudo apt update

sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io

sudo usermod -aG docker $USER

sudo systemctl start docker
sudo systemctl enable docker

newgrp docker
```

Verify:

```bash
docker --version
```

---

## ☸️ Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/

kubectl version --client
```

---

## 📦 Install kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

chmod +x ./kind

sudo mv ./kind /usr/local/bin/kind

kind version
```

---

## 🔧 Install Git

```bash
sudo apt install -y git

git --version
```

---

# ☸️ Create Kubernetes Cluster

Create the kind configuration:

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
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

Create the cluster:

```bash
kind create cluster \
  --config=kind-config.yaml \
  --name=argocd-lab
```

Verify:

```bash
kubectl cluster-info --context kind-argocd-lab
kubectl get nodes
```

---

# 🧩 Kustomize

Kustomize is used to customize Kubernetes configurations without maintaining completely separate manifest files for every environment.

The project follows the structure:

```text
argocd-kustomize-lab/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
│
├── overlays/
│   ├── development/
│   │   ├── deployment-patch.yaml
│   │   ├── namespace.yaml
│   │   └── kustomization.yaml
│   │
│   └── production/
│       ├── deployment-patch.yaml
│       ├── namespace.yaml
│       └── kustomization.yaml
│
└── components/
    └── configmap/
        ├── configmap.yaml
        └── kustomization.yaml
```

This structure separates common configuration from environment-specific changes.

---

# 📁 Base Configuration

The base contains common application resources.

### Deployment

The sample application uses NGINX:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### Service

The application is exposed internally through a ClusterIP service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app-service
spec:
  selector:
    app: sample-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

---

# 🧪 Development Overlay

The development overlay customizes the base configuration.

Development configuration includes:

* `development` namespace
* 1 application replica
* `nginx:1.21-alpine`
* Lower CPU and memory requirements
* Development environment labels

Example:

```yaml
namespace: development

resources:
- ../../base
- namespace.yaml

commonLabels:
  environment: development
```

Test the generated configuration:

```bash
kubectl kustomize overlays/development
```

Apply it:

```bash
kubectl apply -k overlays/development
```

Verify:

```bash
kubectl get all -n development
```

---

# 🚀 Production Overlay

The production overlay provides a different configuration while reusing the same base.

Production configuration includes:

* `production` namespace
* 3 replicas initially
* `nginx:1.21` / later `nginx:1.22`
* Higher CPU and memory resources
* Production-specific labels

Build the configuration:

```bash
kubectl kustomize overlays/production
```

---

# 🔄 Argo CD Installation

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd \
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
kubectl get all -n argocd
```

---

# 🖥️ Argo CD CLI

Install the Argo CD CLI:

```bash
curl -sSL -o argocd-linux-amd64 \
https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

chmod +x argocd-linux-amd64

sudo mv argocd-linux-amd64 /usr/local/bin/argocd

argocd version --client
```

---

# 🌐 Access Argo CD

Start port forwarding:

```bash
kubectl port-forward svc/argocd-server \
-n argocd 8080:443 &
```

Retrieve the initial admin password:

```bash
ARGOCD_PASSWORD=$(kubectl -n argocd \
get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d)

echo "$ARGOCD_PASSWORD"
```

Login:

```bash
argocd login localhost:8080 \
  --username admin \
  --password "$ARGOCD_PASSWORD" \
  --insecure
```

Argo CD is then available through:

```text
https://localhost:8080
```

---

# 📚 Git Repository

Initialize the application repository:

```bash
cd ~/argocd-kustomize-lab

git init

git config user.name "Lab User"
git config user.email "labuser@example.com"

git add .

git commit -m "Initial commit: Kustomize application structure"
```

Create a local bare repository:

```bash
cd ~

git init --bare argocd-kustomize-repo.git
```

Add it as a remote:

```bash
cd ~/argocd-kustomize-lab

git remote add origin ~/argocd-kustomize-repo.git

git push -u origin master
```

---

# 🚢 Argo CD Development Application

The development Argo CD Application points to:

```text
overlays/development
```

Its synchronization policy enables:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
  - CreateNamespace=true
```

This provides automated synchronization, resource pruning, self-healing, and namespace creation.

Verify:

```bash
argocd app list
```

---

# 🚀 Argo CD Production Application

The production application points to:

```text
overlays/production
```

Verify:

```bash
argocd app list
```

Check both applications:

```bash
argocd app get sample-app-dev
argocd app get sample-app-prod
```

---

# 🔄 Synchronization

Applications can be synchronized manually:

```bash
argocd app sync sample-app-dev
argocd app sync sample-app-prod
```

Wait for healthy status:

```bash
argocd app wait sample-app-dev --health
argocd app wait sample-app-prod --health
```

Verify Kubernetes resources:

```bash
kubectl get all -n development
kubectl get all -n production
```

---

# 🔁 GitOps Workflow

One of the main objectives of this lab is to demonstrate the GitOps reconciliation process.

The workflow is:

```text
Developer
   │
   ▼
Modify Kubernetes Configuration
   │
   ▼
Git Commit
   │
   ▼
Git Push
   │
   ▼
Argo CD Detects Change
   │
   ▼
Kustomize Builds Desired State
   │
   ▼
Argo CD Synchronizes
   │
   ▼
Kubernetes Cluster
```

For example, update the NGINX version:

```bash
sed -i 's/nginx:1.21/nginx:1.22/g' base/deployment.yaml
```

Commit and push:

```bash
git add .

git commit -m "Update nginx to version 1.22"

git push origin master
```

Refresh Argo CD:

```bash
argocd app refresh sample-app-dev
argocd app refresh sample-app-prod
```

Verify:

```bash
kubectl describe deployment sample-app \
-n development | grep Image

kubectl describe deployment sample-app \
-n production | grep Image
```

---

# 🧩 Kustomize Components

The lab also demonstrates reusable Kustomize components.

Example structure:

```text
components/
└── configmap/
    ├── configmap.yaml
    └── kustomization.yaml
```

The component creates:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    app.name=sample-app
    app.version=1.0.0
    log.level=INFO
```

The base configuration references the component:

```yaml
components:
- ../components/configmap
```

The Deployment mounts the ConfigMap into:

```text
/etc/config
```

This allows shared configuration to be reused across environments.

---

# 🌍 Environment-Specific Configuration

The lab applies different ConfigMap values to development and production.

### Development

```text
app.name=sample-app
app.version=1.0.0-dev
log.level=DEBUG
environment=development
```

### Production

```text
app.name=sample-app
app.version=1.0.0
log.level=WARN
environment=production
```

This demonstrates how Kustomize overlays can modify shared configuration without duplicating the complete base configuration.

---

# 🔍 Verification

Check Argo CD applications:

```bash
argocd app list
```

Check application health:

```bash
argocd app get sample-app-dev --show-params
argocd app get sample-app-prod --show-params
```

Check namespaces:

```bash
kubectl get namespaces
```

Check development:

```bash
kubectl get all,configmap -n development
```

Check production:

```bash
kubectl get all,configmap -n production
```

---

# 🌐 Test Application Access

Development:

```bash
kubectl port-forward \
svc/sample-app-service \
-n development 8081:80 &
```

Test:

```bash
curl -s http://localhost:8081 | head -5
```

Production:

```bash
kubectl port-forward \
svc/sample-app-service \
-n production 8082:80 &
```

Test:

```bash
curl -s http://localhost:8082 | head -5
```

---

# 📈 Test Production Scaling

The lab also demonstrates changing production replicas through Git.

Update:

```bash
sed -i 's/replicas: 3/replicas: 5/g' \
overlays/production/deployment-patch.yaml
```

Commit:

```bash
git add .

git commit -m "Scale production to 5 replicas"

git push origin master
```

Refresh Argo CD:

```bash
argocd app refresh sample-app-prod
```

Watch the deployment:

```bash
kubectl get deployment sample-app \
-n production -w
```

Verify:

```bash
kubectl get deployment sample-app \
-n production
```

This demonstrates a complete GitOps change from Git to the running Kubernetes workload.

---

# 🐛 Troubleshooting

## Argo CD Application Not Syncing

Check:

```bash
argocd app get sample-app-dev
```

Try a manual synchronization:

```bash
argocd app sync sample-app-dev --force
```

Inspect the Application resource:

```bash
kubectl describe application \
sample-app-dev \
-n argocd
```

---

## Kustomize Build Errors

Build locally:

```bash
kubectl kustomize overlays/development
```

Check YAML files:

```bash
find . -name "*.yaml" \
-exec yamllint {} \;
```

---

## Resource Creation Problems

Check namespaces:

```bash
kubectl get namespaces
```

Check permissions:

```bash
kubectl auth can-i create deployments \
--namespace=development

kubectl auth can-i create configmaps \
--namespace=production
```

These troubleshooting checks are included in the original lab workflow.

---

# 🧹 Cleanup

Delete Argo CD applications:

```bash
kubectl delete application sample-app-dev -n argocd
kubectl delete application sample-app-prod -n argocd
```

Delete application namespaces:

```bash
kubectl delete namespace development
kubectl delete namespace production
```

Delete Argo CD:

```bash
kubectl delete namespace argocd
```

Delete kind cluster:

```bash
kind delete cluster --name=argocd-lab
```

Remove lab files:

```bash
cd ~

rm -rf argocd-kustomize-lab
rm -rf argocd-kustomize-repo.git
```

---

# 📂 Repository Structure

```text
argocd-kustomize-lab/
│
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
│
├── overlays/
│   ├── development/
│   │   ├── deployment-patch.yaml
│   │   ├── configmap-patch.yaml
│   │   ├── namespace.yaml
│   │   └── kustomization.yaml
│   │
│   └── production/
│       ├── deployment-patch.yaml
│       ├── configmap-patch.yaml
│       ├── namespace.yaml
│       └── kustomization.yaml
│
└── components/
    └── configmap/
        ├── configmap.yaml
        └── kustomization.yaml
```

---

# 🧠 Key Concepts Learned

### Kustomize

Kustomize allows Kubernetes manifests to be customized using bases, overlays, patches, labels, images, namespaces, and components.

### Base

The base contains reusable Kubernetes resources shared across environments.

### Overlay

An overlay modifies the base for a particular environment such as development or production.

### Argo CD

Argo CD continuously compares the desired state stored in Git with the live state of the Kubernetes cluster.

### GitOps

Git becomes the source of truth for application configuration.

### Self-Healing

Argo CD can automatically reconcile resources when the live state differs from the desired Git state.

### Pruning

Resources removed from the desired configuration can be removed from the Kubernetes environment when pruning is enabled.

### Components

Kustomize components allow reusable configuration pieces to be shared across multiple overlays.

---

# ✅ Lab Completion Checklist

* [ ] Install Docker
* [ ] Install kubectl
* [ ] Install kind
* [ ] Install Git
* [ ] Create the kind Kubernetes cluster
* [ ] Install Kustomize
* [ ] Create the Kustomize base
* [ ] Create the development overlay
* [ ] Create the production overlay
* [ ] Test Kustomize configurations
* [ ] Install Argo CD
* [ ] Install Argo CD CLI
* [ ] Access the Argo CD server
* [ ] Create the Git repository
* [ ] Deploy the development Argo CD Application
* [ ] Deploy the production Argo CD Application
* [ ] Configure automated synchronization
* [ ] Test GitOps updates
* [ ] Create a reusable ConfigMap component
* [ ] Configure environment-specific ConfigMaps
* [ ] Verify application health
* [ ] Test application accessibility
* [ ] Test production scaling
* [ ] Troubleshoot synchronization and Kustomize issues
* [ ] Clean up the lab environment

---

# 🏁 Conclusion

This lab demonstrates a complete **Argo CD + Kustomize GitOps workflow** on Kubernetes.

By combining Kustomize bases and overlays with Argo CD automated synchronization, the same application can be managed consistently across development and production environments while keeping environment-specific configuration separate.

The lab covers the complete lifecycle from cluster creation and configuration management through Git-based deployment, automated synchronization, reusable components, environment-specific settings, verification, troubleshooting, and cleanup.

The resulting workflow provides a practical foundation for implementing **GitOps-based Kubernetes deployments** with reusable and maintainable configuration management.
