# Installing and Configuring Argo CD

## 📌 Overview

This repository contains a hands-on lab for **installing, configuring, and verifying Argo CD on a local Kubernetes cluster**. The lab demonstrates the fundamental concepts of GitOps and continuous delivery by deploying Argo CD, configuring its API and repository server, accessing the web interface, and deploying a sample application from a Git repository.

The environment uses **Docker, Minikube, Kubernetes, kubectl, and the Argo CD CLI** to build a complete local GitOps platform.

---

## 🎯 Lab Objectives

By completing this lab, you will learn how to:

* Install Argo CD on a Kubernetes cluster
* Install and configure Docker
* Install and use `kubectl`
* Create a local Kubernetes cluster with Minikube
* Configure the Argo CD API server for external access
* Configure and verify the Argo CD Repository Server
* Retrieve and use the Argo CD administrator credentials
* Install and use the Argo CD CLI
* Access the Argo CD web interface
* Create and synchronize applications
* Verify Argo CD components and deployed workloads
* Troubleshoot common Argo CD issues
* Clean up the complete lab environment

---

## 🏗️ Architecture

```text
                    Git Repository
                         │
                         │
                         ▼
              ┌─────────────────────┐
              │      Argo CD        │
              │   GitOps Controller │
              └──────────┬──────────┘
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
     Argo CD API Server       Repository Server
             │                       │
             └───────────┬───────────┘
                         │
                         ▼
                Kubernetes Cluster
                    (Minikube)
                         │
                         ▼
                  Guestbook App
```

---

## 🧰 Technologies Used

| Technology      | Purpose                                |
| --------------- | -------------------------------------- |
| **Argo CD**     | GitOps continuous delivery             |
| **Kubernetes**  | Container orchestration                |
| **Minikube**    | Local Kubernetes cluster               |
| **Docker**      | Container runtime                      |
| **kubectl**     | Kubernetes command-line management     |
| **Argo CD CLI** | Argo CD administration                 |
| **GitHub**      | Application source repository          |
| **YAML**        | Kubernetes and configuration manifests |

---

## 📋 Prerequisites

Before beginning the lab, you should have:

* Basic understanding of Kubernetes concepts such as Pods, Services, and Deployments
* Familiarity with Linux command-line operations
* Basic knowledge of YAML configuration files
* Understanding of container orchestration principles

---

## 🖥️ Lab Environment

The lab uses a Linux-based cloud machine provided through the Al Nafi training environment. The machine starts without the required tools, allowing the lab to demonstrate the installation process from the ground up.

---

# 🚀 Installation

## 1. Install Docker

Docker is installed first and configured as the container runtime for Minikube.

```bash
sudo apt update

sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io

sudo usermod -aG docker $USER

sudo systemctl start docker
sudo systemctl enable docker
```

Apply the Docker group membership:

```bash
newgrp docker
```

Verify:

```bash
docker --version
docker run hello-world
```

---

## 2. Install kubectl

Install the Kubernetes command-line utility:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/

kubectl version --client
```

---

## 3. Install Minikube

Minikube provides the local Kubernetes environment used by this lab.

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

sudo install minikube-linux-amd64 /usr/local/bin/minikube

minikube start --driver=docker
```

Verify the cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

---

# 📦 Deploy Argo CD

## 4. Create the Argo CD Namespace

```bash
kubectl create namespace argocd

kubectl get namespaces
```

## 5. Install Argo CD

Apply the official Argo CD installation manifest:

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for the Argo CD server:

```bash
kubectl wait \
--for=condition=available \
--timeout=600s \
deployment/argocd-server \
-n argocd
```

Check all Argo CD Pods:

```bash
kubectl get pods -n argocd
```

The installation includes components such as:

* `argocd-application-controller`
* `argocd-applicationset-controller`
* `argocd-dex-server`
* `argocd-notifications-controller`
* `argocd-redis`
* `argocd-repo-server`
* `argocd-server`

---

# 🌐 Configure External Access

## 6. Configure the API Server

The Argo CD service can be exposed using NodePort:

```bash
kubectl patch svc argocd-server \
-n argocd \
-p '{"spec":{"type":"NodePort"}}'

kubectl get svc argocd-server -n argocd
```

For this lab, port forwarding is recommended:

```bash
kubectl port-forward \
svc/argocd-server \
-n argocd \
8080:443 &
```

The Argo CD interface can then be accessed through:

```text
https://localhost:8080
```

---

# 🔐 Retrieve Administrator Credentials

Retrieve the initial Argo CD administrator password:

```bash
kubectl -n argocd get secret \
argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d && echo
```

Store it in an environment variable:

```bash
ARGOCD_PASSWORD=$(kubectl -n argocd get secret \
argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d)

echo "$ARGOCD_PASSWORD"
```

The initial credentials are then used to authenticate through the CLI and web interface.

> **Security Note:** Never commit administrator passwords or Kubernetes secrets to Git.

---

# 🖥️ Install Argo CD CLI

Install the Argo CD command-line client:

```bash
curl -sSL -o argocd-linux-amd64 \
https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

chmod +x argocd-linux-amd64

sudo mv argocd-linux-amd64 /usr/local/bin/argocd

argocd version --client
```

---

# 🔑 Login to Argo CD

Login through the CLI:

```bash
argocd login localhost:8080 \
--username admin \
--password "$ARGOCD_PASSWORD" \
--insecure
```

Verify the authenticated user:

```bash
argocd account get-user-info
```

The lab uses `--insecure` because the local port-forwarded interface uses a self-signed certificate.

---

# 📂 Repository Server

The Argo CD Repository Server handles communication with Git repositories and provides repository content to Argo CD.

Check the Repository Server:

```bash
kubectl get deployment \
argocd-repo-server \
-n argocd
```

Verify its Pods:

```bash
kubectl get pods \
-n argocd \
-l app.kubernetes.io/name=argocd-repo-server
```

The lab also demonstrates repository configuration using a ConfigMap and a GitHub repository.

---

# 🔎 Verify Argo CD

Check Services:

```bash
kubectl get svc -n argocd
```

Check Deployments:

```bash
kubectl get deployments -n argocd
```

Check Argo CD server logs:

```bash
kubectl logs \
deployment/argocd-server \
-n argocd \
--tail=20
```

Check Repository Server logs:

```bash
kubectl logs \
deployment/argocd-repo-server \
-n argocd \
--tail=20
```

List applications:

```bash
argocd app list
```

---

# 🌍 Access the Web Interface

Get the Minikube IP:

```bash
minikube ip
```

With port forwarding enabled:

```text
https://localhost:8080
```

Use:

```text
Username: admin
Password: <ARGOCD_PASSWORD>
```

Because the local setup uses a self-signed certificate, the browser may display a certificate warning.

---

# 🚀 Deploy a Test Application

To demonstrate the GitOps workflow, the lab deploys the Argo CD Guestbook example application.

Create the application:

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

List applications:

```bash
argocd app list
```

View application details:

```bash
argocd app get guestbook
```

Synchronize the application:

```bash
argocd app sync guestbook
```

Check its status:

```bash
argocd app get guestbook
```

The lab uses this application to demonstrate the core GitOps workflow of defining application state in Git and synchronizing that state into Kubernetes.

---

# 🔍 Verify the Guestbook Application

Check the Guestbook Pods:

```bash
kubectl get pods \
-l app=guestbook-ui
```

Check the Service:

```bash
kubectl get svc guestbook-ui
```

Check all resources associated with the application:

```bash
kubectl get all \
-l app.kubernetes.io/instance=guestbook
```

---

# 🛠️ Troubleshooting

## Pods Stuck in Pending State

Check node resources:

```bash
kubectl describe nodes
kubectl top nodes
```

Inspect Pod events:

```bash
kubectl describe pod <pod-name> -n argocd
```

## Argo CD Web Interface Is Not Accessible

Check whether port forwarding is active:

```bash
ps aux | grep kubectl
```

Restart it if necessary:

```bash
pkill -f "kubectl port-forward"

kubectl port-forward \
svc/argocd-server \
-n argocd \
8080:443 &
```

## Authentication Problems

Reset the administrator password state:

```bash
kubectl -n argocd patch secret argocd-secret \
-p '{"data":{"admin.password":null,"admin.passwordMtime":null}}'

kubectl -n argocd scale deployment argocd-server --replicas=0

kubectl -n argocd scale deployment argocd-server --replicas=1
```

Retrieve the new password:

```bash
kubectl -n argocd get secret \
argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```

## Repository Server Problems

Inspect logs:

```bash
kubectl logs \
deployment/argocd-repo-server \
-n argocd
```

Test GitHub connectivity:

```bash
kubectl exec -it \
deployment/argocd-repo-server \
-n argocd \
-- nslookup github.com
```

These troubleshooting procedures are included in the lab for common Pod, web-access, authentication, and repository connectivity issues.

---

# ✅ Verification Checklist

Run the following commands for a complete installation check:

```bash
echo "=== Argo CD Installation Verification ==="

echo "1. Checking namespace:"
kubectl get ns argocd

echo "2. Checking pods:"
kubectl get pods -n argocd

echo "3. Checking services:"
kubectl get svc -n argocd

echo "4. Checking CLI version:"
argocd version

echo "5. Checking applications:"
argocd app list

echo "6. Checking cluster info:"
kubectl cluster-info

echo "=== Verification Complete ==="
```

The lab uses these checks to validate the Argo CD namespace, Pods, Services, CLI, applications, and Kubernetes cluster.

---

# 🧹 Cleanup

Delete the test application:

```bash
argocd app delete guestbook --cascade
```

Remove the Argo CD installation:

```bash
kubectl delete -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Delete the namespace:

```bash
kubectl delete namespace argocd
```

Optionally stop Minikube:

```bash
minikube stop
```

These cleanup commands remove the test application, Argo CD installation, namespace, and optionally stop the local Kubernetes cluster.

---

# 📚 What You Learn

After completing this lab, you gain practical experience with:

* Kubernetes cluster preparation
* Docker container runtime configuration
* Minikube-based Kubernetes environments
* Argo CD installation
* Argo CD API Server configuration
* Repository Server configuration
* Argo CD CLI administration
* Git repository integration
* GitOps application deployment
* Application synchronization
* Kubernetes resource verification
* Argo CD troubleshooting
* Environment cleanup

---

# 🎓 Conclusion

This lab provides a complete hands-on introduction to **Argo CD and GitOps-based application delivery**. You install the underlying container and Kubernetes tooling, deploy Argo CD, configure access and repository integration, authenticate through the CLI and web interface, and deploy a sample application from Git.

The resulting environment provides a practical foundation for understanding how Git repositories can serve as the source of truth for Kubernetes deployments and how Argo CD can automate application synchronization and continuous delivery.

---

## ⭐ Key Takeaway

> **Git → Argo CD → Kubernetes**

Argo CD continuously connects the desired application state stored in Git with the actual state running in Kubernetes. This GitOps approach provides a foundation for automated, consistent, and auditable application delivery.

---

## 👨‍💻 Skills Demonstrated

```text
Linux Administration
Docker
Kubernetes
Minikube
kubectl
Argo CD
Argo CD CLI
GitOps
Git Repository Integration
Continuous Delivery
YAML Configuration
Kubernetes Troubleshooting
Application Synchronization
```

---

## 📁 Suggested Repository Structure

```text
argo-cd-installation/
├── README.md
├── repo-server-config.yaml
└── screenshots/
    ├── argocd-dashboard.png
    ├── argocd-applications.png
    └── guestbook-app.png
```

---

## 📖 Lab Source

This README is based on the provided **Installing and Configuring Argo CD** lab content, including its objectives, installation workflow, configuration procedures, verification commands, troubleshooting guidance, and cleanup process.
