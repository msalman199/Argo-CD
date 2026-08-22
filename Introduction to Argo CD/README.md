# 🚀 Introduction to Argo CD

![Argo CD](https://img.shields.io/badge/Argo%20CD-GitOps-orange?style=for-the-badge\&logo=argo)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Container%20Orchestration-326CE5?style=for-the-badge\&logo=kubernetes\&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containerization-2496ED?style=for-the-badge\&logo=docker\&logoColor=white)
![Git](https://img.shields.io/badge/Git-Version%20Control-F05032?style=for-the-badge\&logo=git\&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Administration-FCC624?style=for-the-badge\&logo=linux\&logoColor=black)

> **A hands-on DevOps lab for learning Argo CD, Kubernetes, GitOps, and continuous application delivery.**

---

## 📖 Overview

This lab provides a practical introduction to **Argo CD**, a GitOps continuous delivery tool designed for Kubernetes environments.

The lab starts with a bare Linux-based environment and builds a complete Argo CD deployment using Docker, `kubectl`, and `kind`. It then explores the Argo CD Web UI, CLI, and API before creating an Argo CD project and deploying a sample Kubernetes application using GitOps principles.

The final part demonstrates how changes committed to a Git repository can be detected and synchronized with the Kubernetes environment.

---

## 🎯 Lab Objectives

By completing this lab, you will learn how to:

* 🏗️ Understand Argo CD architecture and core concepts
* ☸️ Install and configure Argo CD on Kubernetes
* 🖥️ Navigate the Argo CD Web UI
* 💻 Use the Argo CD CLI for common operations
* 🔌 Interact with Argo CD REST API endpoints
* 📁 Create and configure an Argo CD project
* 🚀 Create and deploy an Argo CD application
* 🔄 Implement GitOps-based application synchronization
* 📊 Monitor application health and synchronization status
* 🛠️ Troubleshoot common Argo CD and Kubernetes issues

These objectives are directly based on the lab requirements.

---

## 🧰 Technologies & Tools

| Technology             | Purpose                                           |
| ---------------------- | ------------------------------------------------- |
| 🐙 **Argo CD**         | GitOps continuous delivery for Kubernetes         |
| ☸️ **Kubernetes**      | Container orchestration platform                  |
| 🐳 **Docker**          | Container runtime                                 |
| 📦 **kind**            | Local Kubernetes cluster                          |
| ⚙️ **kubectl**         | Kubernetes command-line interface                 |
| 💻 **Argo CD CLI**     | Argo CD administration and application management |
| 🐙 **Git**             | Version control and GitOps source of truth        |
| 🐧 **Linux**           | Lab operating environment                         |
| 📝 **YAML**            | Kubernetes declarative configuration              |
| 🌐 **REST API / curl** | Argo CD API interaction                           |

---

## 📋 Prerequisites

Before starting the lab, you should have basic knowledge of:

* Kubernetes Pods, Services, and Deployments
* Git version control
* YAML configuration
* Containerization concepts
* Linux command-line operations

The original lab specifically identifies these as prerequisites.

---

# 🏗️ Lab Architecture

The lab creates the following workflow:

```text
                    ┌─────────────────────┐
                    │    Git Repository   │
                    │                     │
                    │ deployment.yaml     │
                    │ configmap.yaml      │
                    └──────────┬──────────┘
                               │
                               │ GitOps
                               ▼
                    ┌─────────────────────┐
                    │      Argo CD        │
                    │                     │
                    │  API Server         │
                    │  Repo Server        │
                    │  Controller         │
                    │  Redis              │
                    │  Dex (Optional)     │
                    └──────────┬──────────┘
                               │
                               │ Sync
                               ▼
                    ┌─────────────────────┐
                    │ Kubernetes / kind   │
                    │                     │
                    │  sample-app         │
                    │  Deployment         │
                    │  Service            │
                    │  ConfigMap          │
                    └─────────────────────┘
```

---

# 🚀 Task 1 — Install Argo CD in Kubernetes

## 1️⃣ Install Docker

Update the system and install Docker dependencies:

```bash
sudo apt update

sudo apt install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  software-properties-common
```

Add Docker's repository and install Docker:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io
```

Add the current user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Start and enable Docker:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

Verify:

```bash
docker --version
```

---

## 2️⃣ Install kubectl

Install the Kubernetes command-line tool:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

---

## 3️⃣ Install kind

Install **kind (Kubernetes in Docker)**:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

chmod +x ./kind

sudo mv ./kind /usr/local/bin/kind
```

Verify:

```bash
kind version
```

---

## 4️⃣ Create Kubernetes Cluster

Create the Argo CD lab cluster:

```bash
kind create cluster --name argocd-lab
```

Verify cluster connectivity:

```bash
kubectl cluster-info --context kind-argocd-lab
```

Check nodes:

```bash
kubectl get nodes
```

Expected result:

```text
NAME                      STATUS   ROLES           AGE
argocd-lab-control-plane  Ready    control-plane   ...
```

---

## 5️⃣ Install Argo CD

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

Verify the installation:

```bash
kubectl get pods -n argocd
```

---

## 6️⃣ Expose Argo CD Server

Change the Argo CD service to NodePort:

```bash
kubectl patch svc argocd-server \
-n argocd \
-p '{"spec":{"type":"NodePort"}}'
```

Check the service:

```bash
kubectl get svc argocd-server -n argocd
```

For convenient local access, use port forwarding:

```bash
kubectl port-forward \
svc/argocd-server \
-n argocd \
8080:443 \
--address=0.0.0.0 &
```

The lab uses:

```text
https://localhost:8080
```

---

# 🖥️ Task 2 — Explore Argo CD UI, API & CLI

## 1️⃣ Install Argo CD CLI

Download the CLI:

```bash
curl -sSL \
-o argocd-linux-amd64 \
https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
```

Make it executable:

```bash
chmod +x argocd-linux-amd64
```

Move it into the system path:

```bash
sudo mv argocd-linux-amd64 /usr/local/bin/argocd
```

Verify:

```bash
argocd version --client
```

---

## 2️⃣ Retrieve the Initial Admin Password

Get the automatically generated admin password:

```bash
kubectl -n argocd \
get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d && echo
```

Save the password securely.

### Argo CD Login

```text
Username: admin
Password: <generated-password>
URL:      https://localhost:8080
```

---

## 3️⃣ Login Using the Argo CD CLI

```bash
argocd login localhost:8080 \
--username admin \
--password $(kubectl -n argocd \
get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d) \
--insecure
```

Verify:

```bash
argocd account get-user-info
```

---

## 4️⃣ Explore Argo CD CLI

Display available commands:

```bash
argocd --help
```

List clusters:

```bash
argocd cluster list
```

List applications:

```bash
argocd app list
```

List projects:

```bash
argocd proj list
```

List repositories:

```bash
argocd repo list
```

---

# 🔌 Task 3 — Explore Argo CD API

Generate an API token:

```bash
API_TOKEN=$(argocd account generate-token --account admin)
```

Test the cluster API:

```bash
curl -k \
-H "Authorization: Bearer $API_TOKEN" \
https://localhost:8080/api/v1/clusters
```

Retrieve applications:

```bash
curl -k \
-H "Authorization: Bearer $API_TOKEN" \
https://localhost:8080/api/v1/applications
```

Retrieve projects:

```bash
curl -k \
-H "Authorization: Bearer $API_TOKEN" \
https://localhost:8080/api/v1/projects
```

This demonstrates that Argo CD can be managed through multiple interfaces:

```text
Web UI
  │
  ├── Argo CD CLI
  │
  └── REST API
```

---

# 📁 Task 4 — Create a Git Repository

Create the sample application directory:

```bash
mkdir -p ~/argocd-lab/sample-app
cd ~/argocd-lab/sample-app
```

Initialize Git:

```bash
git init
```

---

## 📦 Create Kubernetes Deployment

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
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
---
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

## ⚙️ Create ConfigMap

Create `configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-app-config
data:
  app.properties: |
    app.name=sample-application
    app.version=1.0.0
    environment=development
```

---

## 💾 Commit the Application

```bash
git add .
```

Configure Git identity:

```bash
git config user.email "student@alnafi.com"
git config user.name "Lab Student"
```

Create the first commit:

```bash
git commit -m "Initial commit: Add sample application manifests"
```

Store the repository path:

```bash
REPO_PATH=$(pwd)

echo "Repository path: $REPO_PATH"
```

---

# 🗂️ Task 5 — Create an Argo CD Project

Create the project:

```bash
argocd proj create lab-project \
    --description "Lab project for learning Argo CD" \
    --src "$REPO_PATH" \
    --dest https://kubernetes.default.svc,default \
    --allow-cluster-resource '*/*' \
    --allow-namespaced-resource '*/*'
```

Verify:

```bash
argocd proj list
```

Get detailed project information:

```bash
argocd proj get lab-project
```

---

# 🚀 Task 6 — Create Argo CD Application

Create the application:

```bash
argocd app create sample-app \
    --repo "$REPO_PATH" \
    --path . \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace default \
    --project lab-project \
    --sync-policy automated \
    --auto-prune \
    --self-heal
```

Verify:

```bash
argocd app list
```

Get detailed application information:

```bash
argocd app get sample-app
```

---

# 🔄 Task 7 — Sync & Deploy Application

Synchronize the application:

```bash
argocd app sync sample-app
```

Wait for synchronization:

```bash
argocd app wait sample-app --timeout 300
```

Check application status:

```bash
argocd app get sample-app
```

Verify Kubernetes resources:

```bash
kubectl get deployments
kubectl get services
kubectl get configmaps
kubectl get pods -l app=sample-app
```

---

# 🔁 Task 8 — Demonstrate the GitOps Workflow

One of the most important concepts in this lab is GitOps.

Initially, the application contains:

```yaml
replicas: 2
```

Change it to:

```yaml
replicas: 3
```

Using:

```bash
cd ~/argocd-lab/sample-app

sed -i 's/replicas: 2/replicas: 3/' deployment.yaml
```

Commit the change:

```bash
git add deployment.yaml

git commit -m "Scale up to 3 replicas"
```

Refresh Argo CD:

```bash
argocd app refresh sample-app
```

Check the application:

```bash
argocd app get sample-app
```

Synchronize:

```bash
argocd app sync sample-app
```

Verify the deployment:

```bash
kubectl get deployments sample-app
```

Check Pods:

```bash
kubectl get pods -l app=sample-app
```

The lab demonstrates the GitOps cycle:

```text
Developer
    │
    ▼
Modify Kubernetes YAML
    │
    ▼
Git Commit
    │
    ▼
Argo CD Detects Change
    │
    ▼
Argo CD Sync
    │
    ▼
Kubernetes Cluster
    │
    ▼
Updated Application
```

---

# ✅ Verification & Testing

## Verify Argo CD Components

```bash
kubectl get pods -n argocd
```

Check services:

```bash
kubectl get svc -n argocd
```

Check Argo CD version:

```bash
argocd version
```

---

## Verify Application

Check application health and synchronization:

```bash
argocd app get sample-app \
--output json | grep -E '"health"|"sync"'
```

Check Kubernetes resources:

```bash
kubectl get all -l app=sample-app
```

View application logs:

```bash
kubectl logs -l app=sample-app --tail=10
```

---

# 🖥️ Verify the Argo CD Web UI

Open:

```text
https://localhost:8080
```

Login with:

```text
Username: admin
Password: <generated-password>
```

Verify that:

* ✅ `sample-app` appears in the application list
* ✅ Application health is visible
* ✅ Synchronization status is displayed
* ✅ Resource tree can be explored
* ✅ Tree, Network, and List views are available
* ✅ Kubernetes resources are displayed correctly

---

# 🛠️ Troubleshooting

## ❌ Pods Stuck in Pending State

Check node resources:

```bash
kubectl describe nodes
```

Check Pod events:

```bash
kubectl describe pod <pod-name>
```

---

## ❌ Argo CD UI Is Not Accessible

Check whether port forwarding is running:

```bash
ps aux | grep port-forward
```

Restart the port forward:

```bash
pkill -f "port-forward"

kubectl port-forward \
svc/argocd-server \
-n argocd \
8080:443 \
--address=0.0.0.0 &
```

---

## ❌ Application Synchronization Fails

Inspect application configuration:

```bash
argocd app get sample-app --output yaml
```

Check Kubernetes events:

```bash
kubectl get events \
--sort-by=.metadata.creationTimestamp
```

---

## ❌ CLI Authentication Problems

Logout:

```bash
argocd logout
```

Login again:

```bash
argocd login localhost:8080 \
--username admin \
--insecure
```

---

# 🧠 Key Concepts

## 🏗️ Argo CD Architecture

The lab introduces the major Argo CD components:

### API Server

Provides the API consumed by:

* Web UI
* Argo CD CLI
* CI/CD systems

### Repository Server

Maintains a local cache of Git repositories used by Argo CD.

### Application Controller

Continuously monitors applications and compares the desired state with the actual Kubernetes state.

### Redis

Used for caching and message-broker functionality.

### Dex

Provides identity and authentication functionality when configured.

These components are identified in the lab's architecture section.

---

# 🔄 GitOps Principles

The lab demonstrates four important GitOps principles:

| Principle          | Description                                      |
| ------------------ | ------------------------------------------------ |
| 📜 **Declarative** | Desired system state is described declaratively  |
| 🕐 **Versioned**   | Configuration is stored in Git with history      |
| 🤖 **Automated**   | Changes can be automatically synchronized        |
| 👁️ **Monitored**  | Argo CD continuously monitors and corrects drift |

---

# ⭐ Argo CD Features Covered

The lab introduces several important Argo CD capabilities:

* 🚀 Application management
* 🔗 Git repository integration
* 👥 Multi-tenancy and projects
* 🔐 RBAC support
* ❤️ Application health monitoring
* 🔄 Automated synchronization
* 🧹 Automatic pruning
* 🩹 Self-healing
* ⏪ Rollback capabilities

These capabilities form the foundation of the lab's Argo CD learning objectives.

---

# 📂 Suggested Repository Structure

```text
argocd-lab/
│
├── README.md
│
└── sample-app/
    ├── deployment.yaml
    └── configmap.yaml
```

---

# 🎓 Learning Outcomes

After completing this lab, you should be able to:

```text
                    Argo CD Skills
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
   Installation      Application       GitOps
        │             Management        Workflow
        │                │                │
        ▼                ▼                ▼
   Kubernetes        Projects          Git
   kind Cluster      Applications      Sync
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                 Continuous Delivery
```

You will have practical experience with:

* Installing Argo CD
* Managing Kubernetes resources
* Using the Argo CD Web UI
* Using the Argo CD CLI
* Calling Argo CD APIs
* Creating projects
* Creating applications
* Configuring automated synchronization
* Testing GitOps workflows
* Monitoring application health
* Troubleshooting Argo CD deployments

---

# 🏁 Conclusion

This lab establishes a practical foundation for **Argo CD and GitOps-based Kubernetes deployments**.

During the lab, Argo CD is installed on a Kubernetes cluster, its Web UI, CLI, and REST API are explored, a project and application are created, and a sample application is deployed using declarative Kubernetes manifests.

The GitOps workflow is then demonstrated by changing the application's replica count in Git and synchronizing the updated desired state with Kubernetes.

The completed lab provides a foundation for progressing toward more advanced topics such as:

* Multi-environment deployments
* Advanced synchronization policies
* Custom health checks
* RBAC and multi-tenancy
* CI/CD pipeline integration
* Production GitOps architectures

The original lab concludes that this foundation prepares learners for these more advanced Argo CD topics and modern DevOps practices.

---

## 🏷️ Technology Badges

![Argo CD](https://img.shields.io/badge/Argo%20CD-FF6B35?style=flat-square\&logo=argo)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square\&logo=kubernetes\&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square\&logo=docker\&logoColor=white)
![Git](https://img.shields.io/badge/Git-F05032?style=flat-square\&logo=git\&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square\&logo=linux\&logoColor=black)
![YAML](https://img.shields.io/badge/YAML-CB171E?style=flat-square\&logo=yaml\&logoColor=white)

---

## 👨‍💻 Author

**Hafiz Muhammad Salman**

**Cloud DevOps Engineer | Linux Administrator**

Focused on:

`AWS` • `Azure` • `Linux` • `Docker` • `Kubernetes` • `Terraform` • `Ansible` • `Jenkins` • `GitOps` • `Argo CD` • `Prometheus` • `Grafana`

---

⭐ **If this lab helped you understand Argo CD and GitOps, consider starring the repository!**
