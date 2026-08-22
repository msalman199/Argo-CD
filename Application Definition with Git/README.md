# Application Definition with Git

A hands-on GitOps lab that demonstrates how to deploy and continuously manage a containerized Kubernetes application using **Git, Kubernetes, Helm, Docker, kind, and Argo CD**.

The lab builds a complete local GitOps workflow: application definitions are stored in Git, Argo CD continuously monitors the repository, changes are introduced through Git commits, and the Kubernetes cluster is automatically reconciled to match the declared state.

## 🎯 Objectives

By completing this lab, you will:

* Build and deploy a containerized web application to Kubernetes.
* Define the same application using both raw Kubernetes manifests and a Helm chart.
* Manage Kubernetes configuration entirely through a local Git repository.
* Install and configure Docker, kubectl, Helm, kind, and Argo CD.
* Create a local Kubernetes cluster named `gitops-lab`.
* Configure Argo CD to monitor a Git repository using a `file://` repository URL.
* Deploy two independent Argo CD Applications.
* Enable automated synchronization, pruning, self-healing, and namespace creation.
* Demonstrate a complete GitOps version update from `v1.0.0` to `v1.1.0`.
* Verify that application changes reach Kubernetes without manually changing Deployments.
* Demonstrate Argo CD self-healing by deleting a Deployment and allowing Argo CD to restore it automatically.

---

## 🏗️ Lab Architecture

```text
                         Local Git Repository
                              ~/gitops-lab
                                  |
                    +-------------+-------------+
                    |                           |
             Raw Kubernetes                  Helm Chart
              Manifests                      Definition
                    |                           |
                    +-------------+-------------+
                                  |
                                  v
                         Argo CD Repository
                              Server
                                  |
                    +-------------+-------------+
                    |                           |
             sample-app                  sample-app-helm
                    |                           |
             Kubernetes Deployment      Kubernetes Deployment
                    |                           |
                 Service                    Service
                    |                           |
                    +-------------+-------------+
                                  |
                                  v
                           kind Cluster
                           gitops-lab
```

### GitOps Flow

```text
Developer
   |
   | git commit
   v
Git Repository
   |
   | Argo CD detects change
   v
Argo CD
   |
   | Automated Sync
   v
Kubernetes Cluster
   |
   | Reconciled state
   v
Running Application
```

---

## 🧰 Technology Stack

| Technology | Purpose                                       |
| ---------- | --------------------------------------------- |
| Ubuntu     | Lab operating system                          |
| Docker     | Build the application container               |
| Kubernetes | Container orchestration                       |
| kind       | Local Kubernetes cluster                      |
| kubectl    | Kubernetes CLI                                |
| Helm       | Kubernetes package management                 |
| Git        | Application configuration and version control |
| Argo CD    | GitOps continuous delivery                    |
| Bash       | Automation and administration                 |

---

## 📋 Prerequisites

You should have:

* Basic Linux command-line knowledge.
* Familiarity with Git.
* Basic understanding of Docker containers.
* Basic Kubernetes knowledge.
* Understanding of Deployments and Services.
* Basic familiarity with Helm.
* Access to the Al Nafi AWS EC2 Ubuntu training environment.

---

# Task 1 — Install and Verify the Toolchain

The first stage prepares the Ubuntu instance with all required DevOps and GitOps tooling.

## 1. Install Docker

Install Docker Engine and configure the current user to run Docker commands.

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

printf 'deb [arch=%s signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu %s stable\n' \
  "$(dpkg --print-architecture)" \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" \
  | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

sudo usermod -aG docker "$USER"
newgrp docker

docker version
```

### Docker Repository Troubleshooting

If you encounter:

```text
E: Malformed entry 1 in list file /etc/apt/sources.list.d/docker.list
```

Inspect the repository file:

```bash
cat /etc/apt/sources.list.d/docker.list
```

The repository configuration must be a single unbroken line.

If necessary:

```bash
sudo rm /etc/apt/sources.list.d/docker.list
```

Then rerun the repository configuration command.

---

## 2. Install kubectl

```bash
KUBE_VERSION="$(curl -fsSL https://dl.k8s.io/release/stable.txt)"

curl -fsSL \
  "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubectl" \
  -o /tmp/kubectl

sudo install -o root -g root -m 0755 \
  /tmp/kubectl /usr/local/bin/kubectl

kubectl version --client
```

---

## 3. Install Helm

```bash
curl -fsSL \
  https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \
  | bash

helm version
```

If the installation script returns a `404` error, use Helm's current Linux amd64 binary installation method instead.

---

## 4. Install kind

```bash
curl -fsSL -o /tmp/kind \
  "https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64"

sudo install -o root -g root -m 0755 \
  /tmp/kind /usr/local/bin/kind

kind version
```

---

## 5. Create the kind Cluster

Create the single-node Kubernetes cluster:

```bash
kind create cluster --name gitops-lab
```

Verify the cluster:

```bash
kubectl cluster-info --context kind-gitops-lab
kubectl get nodes
```

Expected result:

```text
NAME                        STATUS   ROLES           AGE
gitops-lab-control-plane    Ready    control-plane   ...
```

### kind Troubleshooting

If the cluster already exists:

```text
ERROR: failed to create cluster: node(s) already exist
```

Delete the existing cluster:

```bash
kind delete cluster --name gitops-lab
```

Then recreate it:

```bash
kind create cluster --name gitops-lab
```

---

# 6. Install Argo CD

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

Check the pods:

```bash
kubectl get pods -n argocd
```

Retrieve the initial administrator password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" \
  | base64 -d > ~/argocd-password.txt

echo "Argo CD admin password stored at ~/argocd-password.txt"
```

### Argo CD Troubleshooting

If the server times out:

```text
error: timed out waiting for the condition
```

Inspect the Argo CD pods:

```bash
kubectl get pods -n argocd
```

Look for:

* `ImagePullBackOff`
* `CrashLoopBackOff`
* `Pending`
* Failed readiness probes

Inspect a problematic pod:

```bash
kubectl describe pod <pod-name> -n argocd
```

---

# 7. Install the Argo CD CLI

```bash
ARGOCD_VERSION="$(curl -fsSL \
  https://api.github.com/repos/argoproj/argo-cd/releases/latest \
  | grep '"tag_name"' \
  | sed 's/.*"tag_name": "\(.*\)".*/\1/')"

curl -fsSL -o /tmp/argocd \
  "https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64"

sudo install -o root -g root -m 0755 \
  /tmp/argocd /usr/local/bin/argocd

argocd version --client
```

---

# 8. Login to Argo CD

Expose the Argo CD API server:

```bash
kubectl port-forward svc/argocd-server \
  -n argocd 8080:443 &
```

Wait briefly:

```bash
sleep 5
```

Login:

```bash
argocd login localhost:8080 \
  --username admin \
  --password "$(cat ~/argocd-password.txt)" \
  --insecure
```

Verify authentication:

```bash
argocd account get-user-info
```

Expected:

```text
Logged In: true
```

---

## Task 1 Acceptance Criteria

The environment is ready when:

```bash
kubectl get nodes
```

shows one node in `Ready` state.

```bash
kubectl get pods -n argocd
```

shows healthy Argo CD pods.

```bash
argocd account get-user-info
```

returns:

```text
Logged In: true
```

The following commands must also return successfully:

```bash
docker version
kubectl version --client
helm version
kind version
argocd version --client
```

---

# Task 2 — Build the Application and Define It in Git

Create the GitOps repository:

```bash
mkdir -p ~/gitops-lab
cd ~/gitops-lab

git init
```

The repository contains two independent application definitions:

```text
gitops-lab/
├── apps/
│   └── sample-app/
│       ├── manifests/
│       │   ├── namespace.yaml
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── kustomization.yaml
│       │
│       └── helm-chart/
│           ├── Chart.yaml
│           ├── values.yaml
│           └── templates/
│               ├── _helpers.tpl
│               ├── deployment.yaml
│               └── service.yaml
│
└── argocd/
    ├── sample-app-manifests.yaml
    └── sample-app-helm.yaml
```

---

## Raw Kubernetes Manifests

The manifest-based deployment should contain:

* Namespace
* Deployment
* Service
* Kustomization file

The `kustomization.yaml` must reference all Kubernetes resources.

The Deployment must define:

* CPU requests
* CPU limits
* Memory requests
* Memory limits
* HTTP readiness probe
* Container image
* `imagePullPolicy: Never`

Example resource configuration:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Example readiness probe:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

# Helm Application

The Helm chart must contain at least:

```text
helm-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    └── service.yaml
```

The chart should define configurable values such as:

```yaml
replicaCount: 2

image:
  repository: sample-app
  tag: v1.0.0
  pullPolicy: Never
```

The Deployment must include CPU and memory requests/limits and an HTTP readiness probe.

---

## Helm Template Helpers

The chart must use `_helpers.tpl` for:

* Full application name
* Selector labels

This avoids hard-coded application identifiers inside templates and provides reusable Helm naming conventions.

---

# Build the Application Image

Build the application locally:

```bash
docker build -t <your-image-name>:v1.0.0 .
```

Load the image into kind:

```bash
kind load docker-image <your-image-name>:v1.0.0 \
  --name gitops-lab
```

Verify the image:

```bash
docker exec gitops-lab-control-plane \
  crictl images 2>/dev/null \
  | grep "<your-image-name>"
```

The output should contain the image with the `v1.0.0` tag.

---

# Validate the Helm Chart

Run:

```bash
helm lint ~/gitops-lab/apps/sample-app/helm-chart/
```

Expected result:

```text
1 chart(s) linted, 0 chart(s) failed
```

---

# Validate Kubernetes Manifests

Run:

```bash
kubectl apply \
  --dry-run=client \
  -f ~/gitops-lab/apps/sample-app/manifests/
```

The command must complete without validation errors.

---

# Commit the Initial Version

Configure Git if necessary:

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

Commit the application:

```bash
git add .
git commit -m "Add initial GitOps application definition"
```

Tag the initial release:

```bash
git tag v1.0.0
```

Verify:

```bash
git -C ~/gitops-lab log --oneline --decorate
```

The initial commit should be decorated with:

```text
v1.0.0
```

---

# Make the Repository Accessible to Argo CD

Create a bare-style Git repository that Argo CD can access through a local `file://` URL.

The repository must contain the committed application definitions and remain accessible from the environment where Argo CD is running.

Register the repository before creating the Applications:

```bash
argocd repo add file://<repository-path>
```

Verify the repository:

```bash
argocd repo list
```

---

# Create Argo CD Applications

Create two Argo CD Applications.

## Application 1 — Raw Manifests

The manifest-based application should:

* Use the local Git repository.
* Point to the manifests directory.
* Deploy to namespace `sample-app`.
* Target the `gitops-lab` Kubernetes cluster.
* Enable automated synchronization.
* Enable pruning.
* Enable self-healing.
* Automatically create the destination namespace.

Conceptually:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
```

---

## Application 2 — Helm

The Helm-based application should:

* Use the same Git repository.
* Point to the Helm chart directory.
* Deploy to `sample-app-helm`.
* Target the `gitops-lab` cluster.
* Enable automated synchronization.
* Enable pruning.
* Enable self-healing.
* Automatically create the destination namespace.
* Override `replicaCount` to `3`.

The replica count must be provided through an Argo CD parameter rather than by modifying `values.yaml`.

Example concept:

```yaml
parameters:
  - name: replicaCount
    value: "3"
```

---

# Verify Argo CD Applications

Apply both Application resources:

```bash
kubectl apply -f ~/gitops-lab/argocd/
```

Check Argo CD:

```bash
argocd app list
```

Expected state:

```text
NAME                    STATUS    HEALTH
sample-app-manifests    Synced    Healthy
sample-app-helm        Synced    Healthy
```

Verify the workloads:

```bash
kubectl get pods -n sample-app
kubectl get pods -n sample-app-helm
```

Pods should reach:

```text
Running
```

Verify the Helm replica override:

```bash
kubectl get deployment \
  -n sample-app-helm \
  -o jsonpath='{.items[0].spec.replicas}'
```

Expected:

```text
3
```

---

# Task 3 — Execute a GitOps Update Cycle

The third stage demonstrates the core GitOps workflow.

The application will be updated from:

```text
v1.0.0
```

to:

```text
v1.1.0
```

The update must be performed through Git.

---

## 1. Modify the Application

Change the application's visible response so that users can distinguish the new version.

For example:

```text
Welcome to Sample Application v1.1.0
```

Build the new image:

```bash
docker build -t <your-image-name>:v1.1.0 .
```

Load it into kind:

```bash
kind load docker-image \
  <your-image-name>:v1.1.0 \
  --name gitops-lab
```

---

## 2. Update Both Git Definitions

Update the image tag in the raw Kubernetes Deployment:

```text
v1.0.0 → v1.1.0
```

Update the image tag in Helm `values.yaml`:

```text
v1.0.0 → v1.1.0
```

Do not modify the running Kubernetes Deployment directly.

Forbidden approaches:

```bash
kubectl set image
kubectl edit
kubectl patch
```

The only mechanism for changing the application version is:

```text
Git change
    ↓
Git commit
    ↓
Argo CD detects change
    ↓
Argo CD sync
    ↓
Kubernetes rollout
```

---

# Commit Version 1.1.0

Review the changes:

```bash
git status
git diff
```

Commit:

```bash
git add .

git commit -m "Update sample application to v1.1.0"
```

Create the release tag:

```bash
git tag v1.1.0
```

Verify:

```bash
git log --oneline --decorate
```

---

# Refresh and Synchronize Argo CD

Allow Argo CD to detect the repository change or trigger the repository refresh as required by the lab.

Check:

```bash
argocd app list
```

Once synchronization completes, verify the image in the raw manifest application:

```bash
kubectl get deployment \
  -n sample-app \
  -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'
```

Expected:

```text
<your-image-name>:v1.1.0
```

Verify the Helm deployment:

```bash
kubectl get deployment \
  -n sample-app-helm \
  -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'
```

Expected:

```text
<your-image-name>:v1.1.0
```

---

# Verify the Rollout

```bash
kubectl rollout status deployment \
  -n sample-app
```

```bash
kubectl rollout status deployment \
  -n sample-app-helm
```

Both commands should exit successfully.

---

# Verify Updated Application Content

Forward the Service port:

```bash
kubectl port-forward \
  svc/<service-name> \
  -n sample-app \
  8081:<service-port>
```

Then:

```bash
curl -s http://localhost:8081
```

The response must show the new `v1.1.0` application content.

Repeat the process for the Helm application.

The response must differ visibly from the original `v1.0.0` response.

---

# Task 3 Requirement 2 — Verify Self-Healing

Argo CD self-healing ensures that manually changed or deleted resources are restored to the desired Git state.

Delete the manifest-based Deployment:

```bash
kubectl delete deployment \
  -n sample-app \
  <deployment-name>
```

Do **not** execute:

```bash
argocd app sync
```

Do **not** execute:

```bash
kubectl apply
```

The recovery must happen automatically through Argo CD.

---

## Watch the Recovery

Monitor the namespace:

```bash
kubectl get pods \
  -n sample-app \
  --watch
```

The expected lifecycle is:

```text
Deployment deleted
       ↓
Argo CD detects OutOfSync
       ↓
Argo CD self-heal
       ↓
Deployment recreated
       ↓
New Pod created
       ↓
Pod becomes Running
       ↓
Application returns to Synced
```

---

# Verify Deployment Recovery

```bash
kubectl get deployment -n sample-app
```

Then:

```bash
kubectl rollout status \
  deployment \
  -n sample-app
```

The Deployment must become available within **3 minutes**.

Record the elapsed recovery time between:

```text
kubectl delete deployment
```

and:

```text
Deployment Available
```

---

# Verify Argo CD Self-Healing

Run:

```bash
argocd app get sample-app-manifests
```

The output should demonstrate that Argo CD detected the drift and returned the application to the desired state.

Look for evidence of:

```text
OutOfSync
```

followed by:

```text
Synced
```

and an event or condition indicating self-healing activity.

---

# ✅ Acceptance Criteria

## Toolchain

* [ ] Docker is installed and operational.
* [ ] kubectl is installed.
* [ ] Helm is installed.
* [ ] kind is installed.
* [ ] Argo CD CLI is installed.
* [ ] kind cluster `gitops-lab` exists.
* [ ] Kubernetes node is `Ready`.
* [ ] Argo CD pods are healthy.
* [ ] Argo CD CLI login succeeds.

## Application Definition

* [ ] Git repository exists at `~/gitops-lab`.
* [ ] Raw Kubernetes manifests are present.
* [ ] `kustomization.yaml` lists all resources.
* [ ] Helm chart contains `Chart.yaml`.
* [ ] Helm chart contains `values.yaml`.
* [ ] Helm chart contains `_helpers.tpl`.
* [ ] Helm Deployment and Service use Helm helpers.
* [ ] CPU requests and limits are configured.
* [ ] Memory requests and limits are configured.
* [ ] Readiness probes target the HTTP application port.
* [ ] Application image is loaded into kind.
* [ ] `helm lint` exits successfully.
* [ ] Kubernetes dry-run validation succeeds.
* [ ] Git commit is tagged `v1.0.0`.

## Argo CD

* [ ] Local Git repository is registered with Argo CD.
* [ ] Manifest-based Application exists.
* [ ] Helm-based Application exists.
* [ ] Manifest Application deploys to `sample-app`.
* [ ] Helm Application deploys to `sample-app-helm`.
* [ ] Automated synchronization is enabled.
* [ ] `prune: true` is enabled.
* [ ] `selfHeal: true` is enabled.
* [ ] Namespace auto-creation is enabled.
* [ ] Helm `replicaCount` is overridden to `3`.
* [ ] Both Applications report `Synced`.
* [ ] Both Applications report `Healthy`.

## GitOps Update

* [ ] Application content is changed.
* [ ] Image `v1.1.0` is built.
* [ ] Image is loaded into kind.
* [ ] Raw manifests reference `v1.1.0`.
* [ ] Helm values reference `v1.1.0`.
* [ ] Changes are committed to Git.
* [ ] Commit is tagged `v1.1.0`.
* [ ] Argo CD detects the Git change.
* [ ] Both Deployments use `v1.1.0`.
* [ ] Both rollouts complete successfully.
* [ ] Updated application content is accessible through HTTP.
* [ ] No `kubectl set image` or equivalent imperative update is used.

## Self-Healing

* [ ] Deployment is manually deleted.
* [ ] No manual Argo CD sync is executed.
* [ ] No `kubectl apply` is executed after deletion.
* [ ] Argo CD automatically detects drift.
* [ ] Deployment is recreated.
* [ ] Pods become `Running`.
* [ ] Deployment becomes available within 3 minutes.
* [ ] Argo CD returns the application to `Synced`.
* [ ] Self-healing evidence is captured with `argocd app get`.

---

# 🔍 Verification Commands

Useful commands for final validation:

```bash
kind get clusters
```

```bash
kubectl get nodes
```

```bash
kubectl get pods -n argocd
```

```bash
kubectl get pods -n sample-app
```

```bash
kubectl get pods -n sample-app-helm
```

```bash
argocd app list
```

```bash
argocd app get sample-app-manifests
```

```bash
argocd app get sample-app-helm
```

```bash
helm lint ~/gitops-lab/apps/sample-app/helm-chart/
```

```bash
git -C ~/gitops-lab log --oneline --decorate
```

```bash
kubectl get deployment \
  -n sample-app \
  -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'
```

```bash
kubectl get deployment \
  -n sample-app-helm \
  -o jsonpath='{.items[0].spec.template.spec.containers[0].image}'
```

---

# 🧠 Key GitOps Concepts Demonstrated

### Declarative Configuration

The desired Kubernetes state is stored as code in Git rather than being maintained manually inside the cluster.

### Git as the Source of Truth

Application configuration, image versions, Kubernetes manifests, and Helm values are version-controlled.

### Automated Reconciliation

Argo CD continuously compares the desired Git state with the actual Kubernetes state.

### Automated Synchronization

When Git changes, Argo CD can automatically apply the new desired state.

### Self-Healing

If someone deletes or modifies a managed resource, Argo CD detects the drift and restores the declared state.

### Helm and Raw Manifests

The lab demonstrates two common Kubernetes application-definition approaches:

```text
Raw Kubernetes YAML
        +
Helm Templates
        ↓
     Argo CD
        ↓
   Kubernetes
```

---

# 📁 Suggested Repository Structure

```text
gitops-lab/
├── README.md
├── apps/
│   └── sample-app/
│       ├── manifests/
│       │   ├── namespace.yaml
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── kustomization.yaml
│       │
│       └── helm-chart/
│           ├── Chart.yaml
│           ├── values.yaml
│           └── templates/
│               ├── _helpers.tpl
│               ├── deployment.yaml
│               └── service.yaml
│
└── argocd/
    ├── sample-app-manifests.yaml
    └── sample-app-helm.yaml
```

---

# 🧹 Cleanup

When the lab is complete, remove the kind cluster:

```bash
kind delete cluster --name gitops-lab
```

Optionally remove the local repository:

```bash
rm -rf ~/gitops-lab
```

Remove the stored Argo CD password:

```bash
rm -f ~/argocd-password.txt
```

---

# 🚀 Expected Outcome

At the end of this lab, you will have built a complete local GitOps workflow:

```text
                 Git Repository
                 ~/gitops-lab
                       |
          +------------+------------+
          |                         |
      Kubernetes YAML            Helm Chart
          |                         |
          +------------+------------+
                       |
                    Argo CD
                       |
             Automated Reconciliation
                       |
                kind Kubernetes
                       |
          +------------+------------+
          |                         |
     sample-app              sample-app-helm
          |                         |
       v1.1.0                    v1.1.0
```

Both applications are managed declaratively through Git, synchronized automatically by Argo CD, and protected by self-healing.

The completed `v1.0.0 → v1.1.0` workflow demonstrates that application changes can be delivered by changing Git rather than manually modifying Kubernetes resources.

---

# 🔮 Future Improvements

This local GitOps implementation can be extended into a production-oriented workflow by:

* Moving the repository from `file://` to GitHub, GitLab, or another Git server.
* Adding a CI pipeline for automated Docker image builds.
* Publishing images to a container registry.
* Using immutable image tags or image digests.
* Adding automated security scanning with tools such as Trivy.
* Introducing separate development, staging, and production environments.
* Using Argo CD Projects for access control.
* Adding Prometheus and Grafana monitoring.
* Integrating centralized logging with Loki.
* Implementing Argo Rollouts for progressive delivery.
* Adding pull-request-based GitOps workflows.
* Implementing automated testing before deployment.

---

## Conclusion

This lab demonstrates the complete lifecycle of a GitOps-managed Kubernetes application, beginning with a bare Ubuntu environment and progressing through tool installation, container creation, Kubernetes application definitions, Helm packaging, Git version control, Argo CD synchronization, automated application updates, and self-healing.

The most important principle demonstrated is:

> **Git defines the desired state, and Argo CD continuously reconciles Kubernetes to that state.**

This approach reduces manual cluster administration, provides a complete history of application changes, enables repeatable deployments, and makes Kubernetes infrastructure easier to audit and recover.
