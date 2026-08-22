# Synchronizing Kubernetes Applications

A hands-on GitOps lab demonstrating how to synchronize Kubernetes applications with a Git repository using **Argo CD**. This lab covers automatic synchronization, manual synchronization, self-healing, pruning, synchronization policies, troubleshooting, and rollback workflows.

## 📌 Lab Objectives

By completing this lab, you will learn how to:

* Understand automatic vs. manual synchronization in Argo CD
* Configure automated synchronization policies
* Perform manual application synchronization
* Test Git-to-Kubernetes synchronization workflows
* Configure self-healing and pruning
* Configure retry and backoff policies
* Detect and resolve configuration drift
* Troubleshoot common Argo CD synchronization issues
* Perform application rollback
* Verify application health and synchronization status

## 🏗️ Lab Architecture

```text
                    Git Repository
                         │
                         │ Git Changes
                         ▼
                  ┌───────────────┐
                  │    Argo CD    │
                  │   Controller  │
                  └───────┬───────┘
                          │
                ┌─────────┴─────────┐
                │                   │
          Auto-Sync             Manual Sync
                │                   │
                └─────────┬─────────┘
                          ▼
                  ┌───────────────┐
                  │   Kubernetes  │
                  │    Cluster    │
                  └───────┬───────┘
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
          NGINX         Apache        Redis
```

## 🧰 Prerequisites

Before starting, you should have:

* Basic Kubernetes knowledge
* Familiarity with Git
* Understanding of YAML
* Basic Linux command-line knowledge
* Understanding of GitOps principles
* Familiarity with Kubernetes Deployments and Services
* Basic knowledge of Argo CD

## 💻 Lab Environment

This lab uses a single Linux machine provided through the **Al Nafi cloud training environment**.

The machine starts with no required tools installed. During the lab, you will install:

* Docker
* kubectl
* kind
* Git
* Argo CD CLI
* Argo CD

A local **kind** Kubernetes cluster is used, so no additional virtual machines or remote Kubernetes hosts are required.

---

# 🚀 Task 1: Set Up Auto-Sync

## 1.1 Install Required Tools

### Install Docker

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable"

sudo apt update
sudo apt install -y docker-ce

sudo usermod -aG docker $USER
newgrp docker

docker --version
```

### Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

kubectl version --client
```

### Install kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

kind version
```

### Install Argo CD CLI

```bash
curl -sSL -o argocd-linux-amd64 \
https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

argocd version --client
```

---

# ☸️ 1.2 Create the Kubernetes Cluster

Create a kind configuration:

```bash
cat << EOF > kind-config.yaml
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
EOF
```

Create the cluster:

```bash
kind create cluster \
  --config=kind-config.yaml \
  --name=argocd-lab

kubectl cluster-info --context kind-argocd-lab
```

Verify:

```bash
kubectl get nodes
```

---

# 🎯 1.3 Install Argo CD

Create the namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for the server:

```bash
kubectl wait \
  --for=condition=available \
  --timeout=300s \
  deployment/argocd-server \
  -n argocd
```

Check Argo CD pods:

```bash
kubectl get pods -n argocd
```

Forward the Argo CD API server:

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443 &
```

Retrieve the initial admin password:

```bash
ARGOCD_PASSWORD=$(kubectl \
  -n argocd \
  get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

echo "Argo CD Admin Password: $ARGOCD_PASSWORD"
```

---

# 📦 1.4 Create the Sample Git Repository

Install Git:

```bash
sudo apt install -y git
```

Create the repository:

```bash
mkdir -p ~/gitops-demo
cd ~/gitops-demo

git init

git config user.name "Lab User"
git config user.email "lab@example.com"
```

Create the NGINX application:

```bash
mkdir -p apps/nginx-app
```

### NGINX Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
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

### NGINX Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

Commit the application:

```bash
git add .
git commit -m "Initial nginx application"
```

---

# 🔄 1.5 Configure Auto-Sync

Login to Argo CD:

```bash
argocd login localhost:8080 \
  --username admin \
  --password $ARGOCD_PASSWORD \
  --insecure
```

Create the Argo CD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: file:///home/$(whoami)/gitops-demo
    targetRevision: HEAD
    path: apps/nginx-app

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
    - CreateNamespace=true
```

Apply it:

```bash
kubectl apply -f nginx-app.yaml
```

Verify:

```bash
argocd app list
argocd app get nginx-app
kubectl get pods -l app=nginx
```

### Auto-Sync Configuration

The following settings are important:

```yaml
automated:
  prune: true
  selfHeal: true
```

* `automated` enables automatic synchronization.
* `prune` removes resources deleted from Git.
* `selfHeal` corrects Kubernetes changes that differ from Git.

---

# 🖐️ Task 2: Manual Synchronization

## 2.1 Create an Application Without Auto-Sync

Create the Apache application:

```bash
mkdir -p apps/apache-app
```

### Apache Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apache
  template:
    metadata:
      labels:
        app: apache
    spec:
      containers:
      - name: apache
        image: httpd:2.4
        ports:
        - containerPort: 80
```

### Apache Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache-service
  namespace: default
spec:
  selector:
    app: apache
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

Commit the application:

```bash
git add apps/apache-app/
git commit -m "Add apache application"
```

Create an Argo CD Application without automated synchronization:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apache-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: file:///home/$(whoami)/gitops-demo
    targetRevision: HEAD
    path: apps/apache-app

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    syncOptions:
    - CreateNamespace=true
```

Apply:

```bash
kubectl apply -f apache-app.yaml
```

---

## 2.2 Perform Manual Synchronization

Check status:

```bash
argocd app get apache-app
argocd app list
```

Synchronize manually:

```bash
argocd app sync apache-app
```

Verify:

```bash
argocd app get apache-app
kubectl get pods -l app=apache
kubectl get svc apache-service
```

### Test a Git Change

Scale Apache from 1 to 3 replicas:

```bash
sed -i 's/replicas: 1/replicas: 3/' \
  apps/apache-app/deployment.yaml

git add apps/apache-app/deployment.yaml
git commit -m "Scale apache to 3 replicas"
```

Check the status:

```bash
argocd app get apache-app
```

The application should become **OutOfSync** until a manual synchronization is performed.

Run:

```bash
argocd app sync apache-app
```

Verify:

```bash
kubectl get pods -l app=apache
```

---

# 🔁 Task 3: Test Git-to-Kubernetes Synchronization

## 3.1 Test Auto-Sync

Update the NGINX image:

```bash
sed -i 's/nginx:1.21/nginx:1.22/' \
  apps/nginx-app/deployment.yaml

git add apps/nginx-app/deployment.yaml
git commit -m "Update nginx to version 1.22"
```

Monitor Argo CD:

```bash
watch -n 5 \
"argocd app get nginx-app | grep -A 5 'Sync Status'"
```

Verify the updated deployment:

```bash
kubectl get pods -l app=nginx -o wide

kubectl describe deployment nginx-deployment | grep Image
```

Because auto-sync is enabled, Argo CD should detect the Git change and reconcile the cluster automatically.

---

# 🧩 3.2 Configure Advanced Sync Policies

Create the Redis application:

```bash
mkdir -p apps/redis-app
```

Create its Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:6.2
        ports:
        - containerPort: 6379
```

Commit:

```bash
git add apps/redis-app/
git commit -m "Add redis application"
```

Configure advanced synchronization:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: file:///home/$(whoami)/gitops-demo
    targetRevision: HEAD
    path: apps/redis-app

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
```

Apply:

```bash
kubectl apply -f redis-app.yaml
```

---

# 🩹 3.3 Test Self-Healing

Manually change the desired Kubernetes state:

```bash
kubectl scale deployment nginx-deployment --replicas=5
```

Verify:

```bash
kubectl get pods -l app=nginx
```

Because `selfHeal: true` is configured, Argo CD should detect the drift and restore the replica count defined in Git.

Monitor:

```bash
watch -n 3 "kubectl get pods -l app=nginx"
```

This demonstrates an important GitOps principle:

```text
Git = Desired State
Kubernetes = Actual State
Argo CD = Reconciliation Engine
```

---

# 🗑️ 3.4 Test Pruning

Create a temporary ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: default
data:
  nginx.conf: |
    server {
        listen 80;
        location / {
            return 200 'Hello from Nginx!';
        }
    }
```

Commit:

```bash
git add apps/nginx-app/configmap.yaml
git commit -m "Add nginx configmap"
```

After synchronization, verify:

```bash
kubectl get configmap nginx-config
```

Remove the resource from Git:

```bash
rm apps/nginx-app/configmap.yaml

git add apps/nginx-app/configmap.yaml
git commit -m "Remove nginx configmap"
```

Because pruning is enabled, Argo CD should remove the Kubernetes ConfigMap:

```bash
kubectl get configmap nginx-config || \
echo "ConfigMap successfully pruned"
```

---

# 🔍 3.5 Monitor and Troubleshoot Synchronization

Inspect application details:

```bash
argocd app get nginx-app --show-params
argocd app get apache-app --show-params
argocd app get redis-app --show-params
```

View synchronization history:

```bash
argocd app history nginx-app
argocd app history apache-app
```

Force synchronization:

```bash
argocd app sync nginx-app --strategy=force
```

Synchronize a specific resource:

```bash
argocd app sync apache-app \
  --resource=apps:Deployment:apache-deployment
```

---

# ↩️ Rollback an Application

Get the current revision:

```bash
CURRENT_REVISION=$(argocd app get nginx-app \
  -o json | jq -r '.status.sync.revision')

echo "Current revision: $CURRENT_REVISION"
```

Make a change:

```bash
sed -i 's/replicas: 2/replicas: 4/' \
  apps/nginx-app/deployment.yaml

git add apps/nginx-app/deployment.yaml
git commit -m "Scale nginx to 4 replicas"
```

After synchronization, perform the rollback:

```bash
argocd app rollback nginx-app $CURRENT_REVISION
```

Verify:

```bash
kubectl get deployment nginx-deployment \
  -o jsonpath='{.spec.replicas}'
```

---

# 🛠️ Troubleshooting

## Application Stuck in Progressing State

Check the operation:

```bash
argocd app get <app-name> --show-operation
```

Check Kubernetes events:

```bash
kubectl get events \
  --sort-by=.metadata.creationTimestamp
```

Refresh the application:

```bash
argocd app get <app-name> --refresh
```

---

## Auto-Sync Is Not Working

Check the synchronization policy:

```bash
argocd app get <app-name> -o yaml | \
grep -A 10 syncPolicy
```

Check application health:

```bash
argocd app get <app-name> | grep Health
```

Manually trigger synchronization:

```bash
argocd app sync <app-name>
```

---

## Resource Conflicts

Inspect differences:

```bash
argocd app diff <app-name>
```

Force synchronization:

```bash
argocd app sync <app-name> --force
```

Use the force strategy when appropriate:

```bash
argocd app sync <app-name> --strategy=force
```

> **Note:** Force/replace operations can be disruptive. Use them carefully, especially in production environments.

---

# ✅ Lab Verification

## Verify Applications

```bash
argocd app list
```

Check Kubernetes resources:

```bash
kubectl get pods --all-namespaces
```

Expected applications include:

```text
nginx-app
apache-app
redis-app
```

---

## Test NGINX

```bash
kubectl port-forward svc/nginx-service 8081:80 &
curl http://localhost:8081
```

---

## Test Apache

```bash
kubectl port-forward svc/apache-service 8082:80 &
curl http://localhost:8082
```

Clean up port forwarding:

```bash
pkill -f "kubectl port-forward"
```

---

## Verify Sync Policies

Auto-sync applications:

```bash
argocd app get nginx-app | grep "Sync Policy"
argocd app get redis-app | grep "Sync Policy"
```

Manual synchronization application:

```bash
argocd app get apache-app | grep "Sync Policy"
```

---

# 🧹 Cleanup

Delete Argo CD applications:

```bash
argocd app delete nginx-app --cascade
argocd app delete apache-app --cascade
argocd app delete redis-app --cascade
```

Delete the Argo CD namespace:

```bash
kubectl delete namespace argocd
```

Delete the kind cluster:

```bash
kind delete cluster --name=argocd-lab
```

Remove the Git repository and configuration files:

```bash
cd ~

rm -rf gitops-demo
rm -f kind-config.yaml
rm -f nginx-app.yaml
rm -f apache-app.yaml
rm -f redis-app.yaml
```

---

# 📊 Auto-Sync vs Manual Sync

| Feature            | Auto-Sync              | Manual Sync                    |
| ------------------ | ---------------------- | ------------------------------ |
| Synchronization    | Automatic              | User initiated                 |
| Git changes        | Automatically applied  | Require manual sync            |
| Production control | Lower                  | Higher                         |
| Self-healing       | Supported              | Not automatic                  |
| Pruning            | Can be enabled         | Controlled by user             |
| Best suited for    | Development/automation | Controlled production releases |
| Human approval     | Usually not required   | Required before sync           |

---

# 🔑 Key Argo CD Concepts

### Auto-Sync

Automatically reconciles Kubernetes resources when the desired state in Git changes.

### Manual Sync

Requires an operator to explicitly execute synchronization.

```bash
argocd app sync <application>
```

### Self-Healing

Automatically corrects Kubernetes resources when they drift from the Git-defined state.

```yaml
selfHeal: true
```

### Pruning

Deletes resources that were previously managed by Argo CD but have been removed from Git.

```yaml
prune: true
```

### Retry and Backoff

Allows Argo CD to retry failed synchronization operations using configurable delays.

### Rollback

Restores an application to a previous synchronization revision.

---

# 🎯 Learning Outcomes

After completing this lab, you should understand how Argo CD maintains the relationship between:

```text
Git Repository
      │
      ▼
Desired State
      │
      ▼
    Argo CD
      │
      ▼
Kubernetes Cluster
      │
      ▼
Actual State
```

You should also be able to:

* Configure automated GitOps deployments
* Configure manual application synchronization
* Identify `Synced` and `OutOfSync` states
* Configure self-healing
* Configure automatic pruning
* Test configuration drift
* Investigate synchronization failures
* Review synchronization history
* Perform targeted synchronization
* Roll back applications
* Build reliable GitOps deployment workflows

# 🏁 Conclusion

This lab demonstrates the core synchronization capabilities of **Argo CD** and shows how GitOps can be used to continuously reconcile Kubernetes applications with their desired state stored in Git.

The auto-sync exercises demonstrate how applications can be deployed automatically whenever Git changes. Manual synchronization demonstrates how teams can retain explicit control over deployments, which is particularly useful for controlled production releases.

The self-healing exercises demonstrate Argo CD's ability to detect and correct configuration drift, while pruning demonstrates how obsolete Kubernetes resources can automatically be removed when they are deleted from the Git repository.

By combining these capabilities with retry policies, synchronization options, history, troubleshooting, and rollback functionality, Argo CD provides a powerful foundation for reliable and auditable Kubernetes application delivery.

## ⭐ Key Takeaways

* **Git is the source of truth** for the desired application state.
* **Auto-sync** reduces manual deployment work.
* **Manual sync** provides explicit deployment control.
* **Self-healing** automatically corrects configuration drift.
* **Pruning** removes resources that no longer exist in Git.
* **Retry policies** improve synchronization reliability.
* **Sync history** provides deployment visibility and auditing.
* **Rollback** enables recovery from problematic changes.
* **Argo CD** provides continuous reconciliation between Git and Kubernetes.

This lab establishes the foundation for more advanced GitOps practices such as multi-environment deployments, Helm-based applications, Kustomize, RBAC, progressive delivery, application sets, and production-grade Argo CD architectures.
