# Handling Application Rollbacks with Argo CD

## 📌 Overview

This repository contains a hands-on lab for **handling Kubernetes application rollbacks using Argo CD and Git**.

The lab demonstrates how to manage multiple application versions, intentionally introduce a problematic release, identify the stable version, and roll the application back using Argo CD CLI and Git history. It also covers rollback verification, automation, monitoring, troubleshooting, and cleanup.

The environment uses a local **kind Kubernetes cluster**, making the lab suitable for practicing GitOps rollback workflows without requiring a cloud Kubernetes cluster.

> **Lab Environment:** Al Nafi Linux-based cloud machine
> **Primary Platform:** Kubernetes + Argo CD
> **Rollback Strategy:** Git history and Argo CD application history

---

## 🎯 Lab Objectives

By completing this lab, you will learn how to:

* Understand rollback strategies and mechanisms in Argo CD
* Configure Argo CD applications for rollback and revision history
* Manage application versions through Git
* Create multiple application releases
* Identify problematic application deployments
* Roll back an application to a previous Git revision
* Monitor rollback progress
* Verify application health after rollback
* Perform selective rollbacks
* Automate rollback operations with Bash
* Monitor application health and synchronization status
* Troubleshoot common rollback problems
* Apply practical rollback and release-management best practices

---

## 🏗️ Architecture

```text
                    ┌─────────────────────┐
                    │    Git Repository   │
                    │                     │
                    │ v1.0 ── v2.0 ── v3.0│
                    └──────────┬──────────┘
                               │
                               │ GitOps
                               ▼
                    ┌─────────────────────┐
                    │      Argo CD        │
                    │                     │
                    │ Application Manager │
                    │ Sync / Rollback     │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Kubernetes / kind   │
                    │                     │
                    │    demo-app         │
                    │ Deployment          │
                    │ Service             │
                    │ ConfigMap            │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Application Health  │
                    │                     │
                    │ Healthy / Unhealthy │
                    │ Synced / OutOfSync  │
                    └─────────────────────┘
```

---

## 🛠️ Technologies Used

| Technology  | Purpose                                 |
| ----------- | --------------------------------------- |
| Kubernetes  | Container orchestration                 |
| kind        | Local Kubernetes cluster                |
| Argo CD     | GitOps continuous delivery              |
| Argo CD CLI | Application management and rollback     |
| Git         | Version control and application history |
| Docker      | Container runtime for kind              |
| kubectl     | Kubernetes administration               |
| Bash        | Rollback automation and monitoring      |
| YAML        | Kubernetes and Argo CD configuration    |
| NGINX       | Demo application container              |

---

## 📋 Prerequisites

Before starting, you should have:

* Basic Kubernetes knowledge
* Understanding of Pods, Services, and Deployments
* Familiarity with Git
* Knowledge of YAML
* Basic Linux command-line skills
* Understanding of containerized applications

---

# 🚀 Lab Setup

## 1. Install Docker

Update the system and install Docker:

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt update
sudo apt install -y docker-ce

sudo usermod -aG docker $USER
newgrp docker

docker --version
```

---

## 2. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

kubectl version --client
```

---

## 3. Install kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

chmod +x ./kind

sudo mv ./kind /usr/local/bin/kind

kind version
```

---

## 4. Install Argo CD CLI

```bash
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

argocd version --client
```

---

## 5. Install Git

```bash
sudo apt install -y git

git --version
```

The lab installs Docker, kubectl, kind, Argo CD CLI, and Git before creating the Kubernetes environment.

---

# ☸️ Create the Kubernetes Cluster

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
    hostPort: 8080
    protocol: TCP
  - containerPort: 443
    hostPort: 8443
    protocol: TCP
EOF
```

Create the cluster:

```bash
kind create cluster \
  --config=kind-config.yaml \
  --name rollback-lab
```

Verify it:

```bash
kubectl cluster-info --context kind-rollback-lab
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

Wait for the server:

```bash
kubectl wait \
  --for=condition=available \
  --timeout=300s \
  deployment/argocd-server \
  -n argocd
```

---

## Access the Argo CD UI

Port-forward the Argo CD server:

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443 &
```

Retrieve the initial admin password:

```bash
ARGOCD_PASSWORD=$(kubectl -n argocd \
  get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

echo "Argo CD Admin Password: $ARGOCD_PASSWORD"
```

Login:

```bash
argocd login localhost:8080 \
  --username admin \
  --password "$ARGOCD_PASSWORD" \
  --insecure
```

---

# 📦 Create the Demo Application

Create the Git repository:

```bash
mkdir -p ~/rollback-demo-app
cd ~/rollback-demo-app

git init

git config user.name "Lab User"
git config user.email "lab@example.com"
```

Create the Kubernetes manifests directory:

```bash
mkdir -p k8s-manifests
```

The lab creates a demo application consisting of:

* Deployment
* Service
* ConfigMap

The initial release is **version 1.0** using `nginx:1.20` with two replicas.

---

# 🔖 Application Versioning

The lab creates three application releases.

## Version 1.0 — Stable Initial Release

Version 1.0 contains:

* 2 replicas
* `nginx:1.20`
* Application version `1.0`
* Basic resource limits
* Initial release configuration

Commit and tag:

```bash
git add .

git commit -m "Initial commit - Application version 1.0"

git tag v1.0
```

---

## Version 2.0 — Feature Release

Version 2.0 introduces:

* 3 replicas
* `nginx:1.21`
* Increased resource allocation
* `FEATURE_FLAG=enabled`
* Enhanced UI
* Performance improvements

Commit and tag:

```bash
git add .

git commit -m "Update to version 2.0 - Added new features"

git tag v2.0
```

---

## Version 3.0 — Problematic Release

Version 3.0 intentionally introduces instability.

Changes include:

* 4 replicas
* `nginx:1.22`
* Higher resource requirements
* Debug mode
* Additional features
* A deliberately incorrect liveness probe

The health check points to:

```text
/nonexistent
```

This causes the application to demonstrate a realistic rollback scenario.

Tag the problematic release:

```bash
git add .

git commit -m "Update to version 3.0 - Major changes (contains issues)"

git tag v3.0
```

---

# 🔄 Configure Argo CD Application

Create an Argo CD Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rollback-demo-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io

spec:
  project: default

  source:
    repoURL: file://$(pwd)
    targetRevision: HEAD
    path: k8s-manifests

  destination:
    server: https://kubernetes.default.svc
    namespace: demo-app

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true

    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  revisionHistoryLimit: 10
```

Apply it:

```bash
kubectl create namespace demo-app

kubectl apply -f argocd-application.yaml
```

Sync the application:

```bash
argocd app sync rollback-demo-app
```

Wait for health:

```bash
argocd app wait rollback-demo-app --health
```

The application is configured with automated synchronization, pruning, self-healing, retries, and revision history.

---

# 🔍 Monitor Application State

Check Argo CD:

```bash
argocd app get rollback-demo-app
```

Check Kubernetes resources:

```bash
kubectl get all -n demo-app
```

Check the deployed version:

```bash
kubectl get configmap demo-app-config \
  -n demo-app \
  -o yaml
```

Check Pods:

```bash
kubectl get pods -n demo-app
```

Inspect the Deployment:

```bash
kubectl describe deployment demo-app -n demo-app
```

Review Kubernetes events:

```bash
kubectl get events \
  -n demo-app \
  --sort-by='.lastTimestamp'
```

---

# 📜 Review Application History

View Argo CD application history:

```bash
argocd app history rollback-demo-app
```

Review Git history:

```bash
cd ~/rollback-demo-app

git log --oneline --graph
```

Compare releases:

```bash
git diff v2.0 v3.0
```

Inspect previous versions:

```bash
git show v2.0
git show v1.0
```

This allows you to determine which release should be used as the rollback target.

---

# ⏪ Perform an Argo CD Rollback

Get the Git revision for version 2.0:

```bash
REVISION_V2=$(git rev-parse v2.0)

echo "Rolling back to revision: $REVISION_V2"
```

Perform the rollback:

```bash
argocd app rollback \
  rollback-demo-app \
  --revision "$REVISION_V2"
```

Alternatively, an application history ID can be used:

```bash
argocd app rollback rollback-demo-app 2
```

---

# 📊 Monitor the Rollback

Refresh the application:

```bash
argocd app get rollback-demo-app --refresh
```

Wait for healthy status:

```bash
argocd app wait rollback-demo-app --health
```

Synchronize if necessary:

```bash
argocd app sync rollback-demo-app
```

---

# ✅ Verify Rollback Success

Check Pods:

```bash
kubectl get pods -n demo-app
```

Verify the application version:

```bash
kubectl get configmap demo-app-config \
  -n demo-app \
  -o jsonpath='{.data.version}'
```

Inspect the Deployment:

```bash
kubectl describe deployment demo-app \
  -n demo-app | grep -A 5 -B 5 "Image:"
```

Verify replicas and version information:

```bash
kubectl get deployment demo-app \
  -n demo-app \
  -o yaml | grep -A 10 -B 5 "replicas\|version"
```

The expected rollback scenario moves the application from the problematic **v3.0** release back to the stable **v2.0** release.

---

# 🌿 Git-Based Rollback

A Git-based rollback can also be performed by creating a rollback branch and merging the desired state back into the main branch.

```bash
cd ~/rollback-demo-app

git checkout -b rollback-to-v2.0

git reset --hard v2.0

git checkout main

git merge rollback-to-v2.0 \
  --no-ff \
  -m "Rollback to version 2.0 due to issues in v3.0"

git tag v2.0-rollback
```

Then synchronize Argo CD:

```bash
argocd app sync rollback-demo-app

argocd app wait rollback-demo-app --health

argocd app get rollback-demo-app
```

---

# 🎯 Selective Rollback

A selective rollback allows individual resources to be restored while retaining other changes.

For example:

```bash
cd ~/rollback-demo-app

git checkout v3.0 -- k8s-manifests/configmap.yaml

git checkout v2.0 -- k8s-manifests/deployment.yaml

git add .

git commit -m \
  "Selective rollback: deployment to v2.0, keeping v3.0 config"

argocd app sync rollback-demo-app
```

This demonstrates how Git can be used to combine different resource versions.

---

# ⚙️ Advanced Rollback Configuration

The lab also demonstrates an advanced Argo CD Application configuration with:

* `PruneLast=true`
* Automated pruning
* `selfHeal=false`
* `CreateNamespace=true`
* Foreground pruning
* `Replace=true`
* Retry configuration
* Extended revision history

This provides additional practice with rollback-related synchronization behavior.

---

# 🧪 Rollback Verification

Verify all Kubernetes resources:

```bash
kubectl get all -n demo-app -o wide
```

Check the ConfigMap:

```bash
kubectl get configmap demo-app-config \
  -n demo-app \
  -o yaml
```

Check resource utilization:

```bash
kubectl top pods -n demo-app
```

Test the application:

```bash
kubectl port-forward \
  -n demo-app \
  svc/demo-app-service \
  8081:80 &

sleep 5

curl -s http://localhost:8081 | head -10

pkill -f "port-forward.*8081"
```

---

# 📝 Rollback Report

The lab generates a rollback report containing:

* Application name
* Rollback source version
* Target version
* Rollback reason
* Rollback method
* Pod status
* Current version
* Replica count
* Post-rollback status

Example:

```text
Application: rollback-demo-app

From Version: 3.0
To Version: 2.0

Reason:
Application instability and health check failures

Method:
Argo CD CLI rollback command
```

This provides a useful foundation for documenting production rollback events.

---

# 🤖 Rollback Automation

The repository includes a Bash automation script:

```text
rollback-automation.sh
```

The script:

1. Accepts a target Git revision
2. Resolves the Git commit hash
3. Executes the Argo CD rollback
4. Waits for the application to become healthy
5. Verifies Pods
6. Checks the deployed application version
7. Reports success or failure

Make it executable:

```bash
chmod +x rollback-automation.sh
```

Run a rollback:

```bash
./rollback-automation.sh v1.0
```

The automation implements health checks with retry and timeout behavior before declaring the rollback successful.

---

# 📡 Rollback Monitoring

The lab also provides:

```text
monitor-rollback.sh
```

The monitoring script continuously checks:

* Argo CD health status
* Argo CD synchronization status
* Pod count
* Ready Pod count
* Current application version
* Recent Kubernetes events

Example status information:

```text
Health: Healthy
Sync: Synced
Pods: 3/3
Version: 2.0
```

Warnings are logged when the application becomes unhealthy, unsynchronized, or when Pods are not ready.

---

# 🐛 Troubleshooting

## Rollback Command Fails

Check the application:

```bash
argocd app get rollback-demo-app
```

Check the logged-in account:

```bash
argocd account get-user-info
```

Re-authenticate:

```bash
argocd login localhost:8080 \
  --username admin \
  --password "$ARGOCD_PASSWORD" \
  --insecure
```

Try a specific Git revision:

```bash
argocd app rollback \
  rollback-demo-app \
  --revision "$(git rev-parse v2.0)"
```

---

## Application Stuck in Progressing

Force a refresh:

```bash
argocd app get \
  rollback-demo-app \
  --refresh \
  --hard-refresh
```

Perform a forced sync:

```bash
argocd app sync rollback-demo-app --force
```

Inspect events:

```bash
kubectl get events \
  -n demo-app \
  --sort-by='.lastTimestamp'
```

---

## Git Repository Problems

Check repository status:

```bash
cd ~/rollback-demo-app

git status

git log --oneline
```

Check repository integrity:

```bash
git fsck
git gc
```

If necessary, recreate the repository:

```bash
cd ~

rm -rf rollback-demo-app
```

Then repeat the repository setup steps.

---

# 🧹 Cleanup

Delete the Argo CD applications:

```bash
argocd app delete rollback-demo-app --cascade

argocd app delete \
  rollback-demo-advanced \
  --cascade 2>/dev/null || true
```

Delete namespaces:

```bash
kubectl delete namespace demo-app

kubectl delete namespace demo-app-advanced \
  2>/dev/null || true
```

Delete the kind cluster:

```bash
kind delete cluster --name rollback-lab
```

Remove generated files:

```bash
rm -rf ~/rollback-demo-app

rm -f kind-config.yaml
rm -f argocd-application*.yaml
rm -f rollback-*.sh
rm -f monitor-rollback.sh
rm -f /tmp/rollback-monitor.log
rm -f rollback-report.md
```

---

# 📚 Key Concepts Demonstrated

### GitOps Rollback

Git provides a version-controlled source of truth for Kubernetes application configuration.

### Argo CD Application History

Argo CD maintains deployment history that can be inspected before performing a rollback.

### Revision-Based Rollback

A specific Git commit or Argo CD history revision can be selected as the rollback target.

### Application Health

Rollback success should be validated using both application health and synchronization status.

### Automated Recovery

Bash automation can reduce the time required to perform repeatable rollback operations.

### Monitoring

Continuous health monitoring helps identify failed deployments and confirm successful recovery.

### Version Management

Tags such as `v1.0`, `v2.0`, and `v3.0` provide identifiable release points for recovery.

---

# 🎓 Learning Outcomes

After completing this lab, you should be able to:

* Build a local Kubernetes environment with kind
* Install and configure Argo CD
* Manage Kubernetes applications through GitOps
* Maintain multiple application releases
* Identify unstable deployments
* Inspect Git and Argo CD history
* Roll back applications to known-good versions
* Perform selective rollbacks
* Automate rollback procedures
* Monitor application health
* Troubleshoot rollback failures
* Document rollback activities
* Apply rollback practices to production-style DevOps workflows

---

# 🌍 Real-World Use Cases

Rollback capabilities are essential in production environments where a deployment can introduce:

* Application instability
* Configuration errors
* Failed health checks
* Performance degradation
* Unexpected behavior
* Failed feature releases
* Resource-related problems

The lab specifically demonstrates how an unstable release can be reverted to a stable version and how automation and monitoring can improve recovery procedures.

---

# 🔐 Production Best Practices

For production GitOps environments, consider:

* Keep application configuration in Git
* Use meaningful release tags
* Maintain sufficient revision history
* Test rollback procedures regularly
* Monitor application health after deployment
* Automate rollback verification
* Document rollback reasons and results
* Use controlled access to Argo CD
* Validate changes before production deployment
* Maintain reliable backups of critical configuration
* Prefer reproducible Git-based recovery procedures

---

# 📁 Suggested Repository Structure

```text
handling-application-rollbacks/
├── README.md
├── k8s-manifests/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── argocd-application.yaml
├── argocd-application-advanced.yaml
├── rollback-automation.sh
├── monitor-rollback.sh
└── rollback-report.md
```

---

# 🏁 Conclusion

This lab provides practical experience with **application rollback strategies in Argo CD** using GitOps principles.

You create multiple application releases, introduce an intentionally problematic version, identify a stable revision, perform a rollback through Argo CD and Git, verify the recovered application, and automate future rollback and monitoring operations.

The combination of **Git version control, Kubernetes, Argo CD, automation, and health monitoring** creates a reliable foundation for managing application releases and reducing deployment risk in cloud-native environments.

---

## 👨‍💻 Skills Demonstrated

```text
Kubernetes
Argo CD
GitOps
Git
Docker
kind
kubectl
Argo CD CLI
Bash Scripting
YAML
Application Rollbacks
Release Management
Health Monitoring
Deployment Automation
Troubleshooting
DevOps
Cloud-Native Operations
```

---

## ⭐ Project Focus

**Handling Application Rollbacks with Argo CD**

> Practice safe application recovery, Git-based version management, automated rollback procedures, and production-oriented GitOps operations.
