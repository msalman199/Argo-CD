# Optimizing Argo CD Performance

A hands-on lab for tuning **Argo CD performance** on a local Kubernetes cluster running inside Docker. This lab focuses on Redis caching, application-controller concurrency, repository-server caching, resource exclusions, server-side apply, sync waves, and measurable synchronization benchmarks.

## 📌 Lab Objectives

By completing this lab, you will be able to:

* Deploy Argo CD on a local **kind** Kubernetes cluster.
* Configure Redis for improved caching performance using a **256 MB memory limit** and **LRU eviction**.
* Tune the Argo CD application controller for higher synchronization throughput.
* Enable server-side diff and server-side apply.
* Configure repository-server Git caching.
* Exclude unnecessary Kubernetes resources from Argo CD tracking.
* Configure optimized synchronization policies for multiple applications.
* Use sync waves to control resource deployment order.
* Build a reusable shell-based Argo CD synchronization benchmark.
* Compare optimized and baseline synchronization performance.

---

## 🏗️ Lab Architecture

```text
                         AWS EC2 Ubuntu
                              │
                              ▼
                         Docker Engine
                              │
                              ▼
                    ┌───────────────────┐
                    │   kind Cluster    │
                    │   argocd-perf     │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │     Argo CD       │
                    │                   │
                    │  API Server       │
                    │  Repo Server      │
                    │  Controller       │
                    │  Redis Cache      │
                    └─────────┬─────────┘
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
        Application A    Application B    Application C
          Optimized         Manual          Sync Waves
              │
              ▼
       Baseline Application
              │
              ▼
      Benchmark Comparison
```

---

## 🧰 Prerequisites

Before beginning, you should have:

* An Ubuntu-based AWS EC2 instance.
* `sudo` privileges.
* Internet connectivity.
* Basic Linux command-line knowledge.
* Basic Kubernetes knowledge.
* Familiarity with GitOps and Argo CD.
* Understanding of Deployments, Services, ConfigMaps, and namespaces.

The lab environment is designed around an EC2 Ubuntu instance provided through **Al Nafi**.

---

# 🚀 Task 1 — Install and Verify the Full Toolchain

## 1.1 Install Supporting Utilities

Update the Ubuntu system and install the utilities required throughout the lab.

```bash
sudo apt-get update && sudo apt-get upgrade -y

sudo apt-get install -y \
  git \
  curl \
  wget \
  jq \
  bc \
  apt-transport-https \
  ca-certificates \
  gnupg \
  lsb-release
```

Verify the utilities:

```bash
git --version
curl --version
wget --version
jq --version
bc --version
```

---

## 1.2 Install Docker

Install Docker and enable the service:

```bash
sudo apt-get install -y docker.io

sudo systemctl enable --now docker

sudo usermod -aG docker $USER

newgrp docker
```

Verify:

```bash
docker version
```

### Troubleshooting

If you receive:

```text
permission denied while trying to connect to the Docker daemon socket
```

Check group membership:

```bash
groups
```

Make sure `docker` appears in the output.

If it does not, log out and back in or run:

```bash
newgrp docker
```

---

## 1.3 Install kubectl

Retrieve the current stable Kubernetes version:

```bash
KUBECTL_VERSION=$(curl -fsSL https://dl.k8s.io/release/stable.txt)
```

Download and install `kubectl`:

```bash
curl -fsSL \
  "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
  -o kubectl

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

rm kubectl
```

Verify:

```bash
kubectl version --client
```

### Troubleshooting

If you receive a `404` error:

```text
curl: (22) The requested URL returned error: 404
```

Check the stable release:

```bash
curl -fsSL https://dl.k8s.io/release/stable.txt
```

Verify that the downloaded URL matches the current Kubernetes release structure.

---

## 1.4 Install kind

Install Kubernetes-in-Docker:

```bash
curl -fsSL \
  https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64 \
  -o kind

sudo install -o root -g root -m 0755 kind /usr/local/bin/kind

rm kind
```

Verify:

```bash
kind version
```

Troubleshoot installation:

```bash
which kind
```

If `exec format error` occurs, verify that the binary matches the EC2 instance architecture.

---

## 1.5 Install Helm

Download the Helm installation script:

```bash
curl -fsSL \
  https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \
  -o get-helm.sh

chmod +x get-helm.sh

./get-helm.sh

rm get-helm.sh
```

Verify:

```bash
helm version
```

If the installer fails, download a specific Helm release manually and install the binary into:

```text
/usr/local/bin/helm
```

---

## 1.6 Install Argo CD CLI

Retrieve the latest Argo CD release:

```bash
ARGOCD_VERSION=$(curl -fsSL \
  https://api.github.com/repos/argoproj/argo-cd/releases/latest \
  | jq -r .tag_name)
```

Download the CLI:

```bash
curl -fsSL \
  "https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64" \
  -o argocd
```

Install:

```bash
sudo install -o root -g root -m 0755 argocd /usr/local/bin/argocd

rm argocd
```

Verify:

```bash
argocd version --client
```

If the GitHub API is rate-limited, use a known Argo CD release instead of dynamically retrieving the version.

---

# ☸️ Task 2 — Create the kind Cluster

Create the kind configuration:

```bash
cat > kind-config.yaml <<EOF
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
  - containerPort: 30080
    hostPort: 30080
    protocol: TCP
EOF
```

Create the cluster:

```bash
kind create cluster \
  --config=kind-config.yaml \
  --name argocd-perf
```

Verify:

```bash
kubectl cluster-info --context kind-argocd-perf
kubectl get nodes
```

Expected cluster:

```text
argocd-perf
```

---

# 🚢 Task 3 — Install Argo CD

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

Wait for the primary Argo CD deployments:

```bash
kubectl wait \
  --for=condition=available \
  --timeout=300s \
  deployment/argocd-server \
  deployment/argocd-repo-server \
  deployment/argocd-application-controller \
  -n argocd
```

Verify:

```bash
kubectl get deployments -n argocd
```

All required deployments should report ready replicas.

---

## 🔐 Connect to Argo CD

Forward the Argo CD API server:

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443 &
```

Retrieve the initial administrator password:

```bash
ARGOCD_PASSWORD=$(kubectl -n argocd get secret \
  argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)
```

Login:

```bash
argocd login localhost:8080 \
  --username admin \
  --password "${ARGOCD_PASSWORD}" \
  --insecure
```

Verify:

```bash
argocd version
```

### Acceptance Criteria

The following must succeed:

```bash
kubectl get deployments -n argocd
```

and:

```bash
argocd version
```

The CLI should communicate successfully with the Argo CD server.

---

# ⚡ Task 4 — Configure Redis Caching and Controller Tuning

## 4.1 Redis Performance Configuration

The target Redis configuration is:

| Setting              |        Target |
| -------------------- | ------------: |
| Maximum memory       |        256 MB |
| Maximum memory bytes |   `268435456` |
| Eviction policy      | `allkeys-lru` |

Verify the configuration:

```bash
kubectl exec \
  -n argocd \
  deployment/argocd-redis \
  -- redis-cli CONFIG GET maxmemory
```

Expected:

```text
268435456
```

Verify the eviction policy:

```bash
kubectl exec \
  -n argocd \
  deployment/argocd-redis \
  -- redis-cli CONFIG GET maxmemory-policy
```

Expected:

```text
allkeys-lru
```

Inspect memory usage:

```bash
kubectl exec \
  -n argocd \
  deployment/argocd-redis \
  -- redis-cli INFO memory | grep used_memory_human
```

---

## 4.2 Application Controller Tuning

The controller should be configured for:

```text
status processors = 20
operation processors = 10
server-side diff = enabled
resource cache expiration = 1 hour
```

Create or update:

```text
argocd-cmd-params-cm
```

in the `argocd` namespace.

Verify:

```bash
kubectl get configmap \
  argocd-cmd-params-cm \
  -n argocd \
  -o yaml
```

Restart the controller:

```bash
kubectl rollout restart \
  deployment/argocd-application-controller \
  -n argocd
```

Verify:

```bash
kubectl rollout status \
  deployment/argocd-application-controller \
  -n argocd
```

---

# 🗄️ Task 5 — Tune Repository Server and Argo CD Configuration

## 5.1 Repository Server Cache

Configure the repository server with:

```text
Git cache expiration = 24 hours
Cache volume = 1 GiB emptyDir
```

The repository-server Deployment should expose:

```text
ARGOCD_REPO_CACHE_EXPIRATION=24h
```

Verify:

```bash
kubectl rollout status \
  deployment/argocd-repo-server \
  -n argocd
```

Inspect recent logs:

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-repo-server \
  --tail=30
```

Search for startup or fatal messages:

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-repo-server \
  --tail=30 \
  | grep -iE "(started|error|fatal)"
```

---

## 5.2 Configure Resource Exclusions

Update `argocd-cm` to exclude:

```text
coordination.k8s.io/Lease
```

from Argo CD resource tracking across clusters.

Also configure:

```text
timeout.reconciliation = 180s
application.diff.server.side = enabled
```

Verify:

```bash
kubectl get configmap argocd-cm \
  -n argocd \
  -o jsonpath='{.data.resource\.exclusions}'
```

Check reconciliation timeout:

```bash
kubectl get configmap argocd-cm \
  -n argocd \
  -o jsonpath='{.data.timeout\.reconciliation}'
```

Expected:

```text
180s
```

---

## 🔄 Restart Tuned Components

Restart Redis:

```bash
kubectl rollout restart \
  deployment/argocd-redis \
  -n argocd
```

Restart the controller:

```bash
kubectl rollout restart \
  deployment/argocd-application-controller \
  -n argocd
```

Restart the repository server:

```bash
kubectl rollout restart \
  deployment/argocd-repo-server \
  -n argocd
```

Verify all rollouts:

```bash
kubectl rollout status deployment/argocd-redis -n argocd

kubectl rollout status \
  deployment/argocd-application-controller \
  -n argocd

kubectl rollout status \
  deployment/argocd-repo-server \
  -n argocd
```

---

# 📦 Task 6 — Create Local Git Repositories

The applications use local Git repositories under `/tmp`.

Create the directories:

```bash
mkdir -p /tmp/argocd-app-a
mkdir -p /tmp/argocd-app-b
mkdir -p /tmp/argocd-app-c
mkdir -p /tmp/argocd-baseline
```

Initialize each repository:

```bash
cd /tmp/argocd-app-a
git init

cd /tmp/argocd-app-b
git init

cd /tmp/argocd-app-c
git init

cd /tmp/argocd-baseline
git init
```

Add valid Kubernetes manifests to each repository and commit them:

```bash
git add .
git commit -m "Initial Kubernetes manifests"
```

These repositories become the GitOps source for the Argo CD Applications.

---

# 🚀 Task 7 — Deploy Optimized Applications

Four applications are used in the performance comparison:

```text
Application A     Optimized automated sync
Application B     Manual optimized sync
Application C     Sync-wave optimized deployment
Baseline          Default/no-option configuration
```

---

## Application A — Fully Optimized Sync

Application A should use:

```yaml
automated:
  prune: true
  selfHeal: true
```

and:

```text
ApplyOutOfSyncOnly=true
ServerSideApply=true
```

The retry configuration should use a backoff capped at three minutes.

Conceptually:

```text
Git Repository
      │
      ▼
Argo CD
      │
      ├── ServerSideApply
      ├── ApplyOutOfSyncOnly
      ├── Auto Prune
      ├── Self Heal
      └── Retry Backoff
              │
              ▼
        Kubernetes Cluster
```

---

## Application B — Manual Sync

Application B should demonstrate controlled manual synchronization.

Required options:

```text
ServerSideApply=true
RespectIgnoreDifferences=true
```

Ignore:

```text
/spec/replicas
```

for Kubernetes Deployments.

Set:

```text
revisionHistoryLimit = 3
```

This configuration is useful when replica-count changes should not automatically cause Argo CD to continuously reconcile the Deployment.

---

# 🌊 Application C — Sync Waves

Application C demonstrates ordered resource deployment.

Required order:

```text
Wave 0 → ConfigMap
          ↓
Wave 1 → Service
          ↓
Wave 2 → Deployment
```

The resources should be placed in:

```text
sync-waves-demo
```

The application should also use:

```text
ApplyOutOfSyncOnly=true
retry limit = 3
```

Verify the resulting creation order:

```bash
kubectl get \
  configmap,service,deployment \
  -n sync-waves-demo \
  -o custom-columns="KIND:.kind,NAME:.metadata.name,CREATED:.metadata.creationTimestamp" \
  --sort-by=.metadata.creationTimestamp
```

The expected ordering is:

```text
ConfigMap
Service
Deployment
```

The creation timestamps should demonstrate that each wave was processed in sequence.

---

# 🧪 Task 8 — Baseline Application

Create a fourth application for comparison.

The baseline should intentionally avoid the optimization options used by Application A.

Use:

```text
No sync optimization options
revisionHistoryLimit = 10
```

This provides a control configuration against which Application A can be measured.

---

# 🔎 Task 9 — Verify Application Health

List applications:

```bash
argocd app list
```

Verify Kubernetes Application objects:

```bash
kubectl get applications \
  -n argocd \
  -o custom-columns=\
"NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status"
```

All four applications should eventually report:

```text
SYNC     HEALTH
Synced   Healthy
```

Expected applications:

```text
Application A
Application B
Application C
Baseline
```

---

# 📊 Task 10 — Build the Synchronization Benchmark

Create:

```text
benchmark-sync.sh
```

The script should accept:

```bash
./benchmark-sync.sh <application-name> <iteration-count>
```

Example:

```bash
./benchmark-sync.sh application-a 5
```

The benchmark should:

1. Accept an application name.
2. Accept an iteration count.
3. Trigger `argocd app sync`.
4. Record the start timestamp.
5. Record the end timestamp.
6. Calculate wall-clock duration using nanoseconds.
7. Report each iteration.
8. Count successful syncs.
9. Count failed syncs.
10. Calculate the arithmetic mean of successful syncs.

The timing mechanism should use:

```bash
date +%s%N
```

Example calculation:

```bash
START=$(date +%s%N)

# Run synchronization here

END=$(date +%s%N)

DURATION_NS=$((END - START))
```

Convert the result into milliseconds or another readable unit.

---

# 📈 Task 11 — Compare Optimized and Baseline Performance

Run five iterations against Application A:

```bash
./benchmark-sync.sh application-a 5
```

Run five iterations against the baseline:

```bash
./benchmark-sync.sh baseline 5
```

The benchmark must record:

```text
Successful sync count
Failed sync count
Per-iteration duration
Average successful sync duration
```

---

## Comparison Summary

Create:

```text
comparison-summary.txt
```

The summary should contain:

```text
Argo CD Performance Comparison
==============================

Optimized Application: application-a
Iterations: 5
Average Sync Time: <value>

Baseline Application: baseline
Iterations: 5
Average Sync Time: <value>

Percentage Difference: <value>%
```

The percentage difference can be calculated using the measured averages.

For example:

```text
Percentage Difference =
((Baseline Average - Optimized Average) / Baseline Average) × 100
```

A positive result indicates that the optimized configuration completed synchronization faster than the baseline.

---

# 🔬 Task 12 — Validate the Benchmark

Display the generated comparison:

```bash
cat comparison-summary.txt
```

The output must show:

* A non-zero optimized average.
* A non-zero baseline average.
* A calculated percentage difference.
* Five benchmark iterations for each tested application.

---

# 🌊 Task 13 — Validate Sync Wave Ordering

Run:

```bash
kubectl get \
  configmap,service,deployment \
  -n sync-waves-demo \
  -o custom-columns="KIND:.kind,NAME:.metadata.name,CREATED:.metadata.creationTimestamp" \
  --sort-by=.metadata.creationTimestamp
```

Expected order:

```text
ConfigMap   → Wave 0
Service     → Wave 1
Deployment  → Wave 2
```

This demonstrates that Argo CD processed resources according to their synchronization waves.

---

# ✅ Final Acceptance Criteria

The lab is complete when all of the following conditions are satisfied.

### Toolchain

The following commands work:

```bash
docker version
kubectl version --client
kind version
helm version
argocd version --client
```

### Kubernetes Cluster

The cluster exists:

```bash
kind get clusters
```

Expected:

```text
argocd-perf
```

### Argo CD

Argo CD deployments are ready:

```bash
kubectl get deployments -n argocd
```

The CLI connects successfully:

```bash
argocd version
```

### Redis

Redis reports:

```text
maxmemory = 268435456
maxmemory-policy = allkeys-lru
```

### Controller

The application controller has the required performance parameters:

```text
20 status processors
10 operation processors
server-side diff
1-hour resource cache expiration
```

### Repository Server

The repository server has:

```text
24-hour Git cache expiration
1 GiB cache volume
```

and restarts successfully.

### Resource Exclusions

Argo CD excludes:

```text
coordination.k8s.io/Lease
```

and reconciliation timeout is:

```text
180s
```

### Applications

All four applications report:

```text
Synced
Healthy
```

### Sync Waves

Resources appear in this order:

```text
ConfigMap
Service
Deployment
```

### Benchmark

The following file exists:

```text
benchmark-sync.sh
```

and is executable:

```bash
ls -l benchmark-sync.sh
```

The comparison file exists:

```text
comparison-summary.txt
```

and contains measurable averages for both configurations.

---

# 📁 Suggested Repository Structure

```text
.
├── README.md
├── benchmark-sync.sh
├── comparison-summary.txt
├── kind-config.yaml
│
├── manifests/
│   ├── application-a.yaml
│   ├── application-b.yaml
│   ├── application-c.yaml
│   └── baseline.yaml
│
├── redis/
│   └── redis-config.yaml
│
├── argocd/
│   ├── argocd-cm.yaml
│   └── argocd-cmd-params-cm.yaml
│
└── applications/
    ├── app-a/
    ├── app-b/
    ├── app-c/
    └── baseline/
```

---

# 🧠 Performance Concepts Demonstrated

This lab demonstrates that Argo CD performance can be improved at multiple layers.

### Redis Optimization

Redis caching reduces repeated data retrieval and improves responsiveness when Argo CD handles application state and cached information.

### Controller Concurrency

Increasing status and operation processors allows the application controller to process more work concurrently, which can improve throughput when sufficient cluster resources are available.

### Server-Side Apply and Diff

Server-side operations move more processing toward the Kubernetes API server and can reduce client-side processing overhead for suitable workloads.

### Resource Exclusions

Ignoring resources that Argo CD does not need to manage reduces unnecessary reconciliation and comparison activity.

### Repository Caching

A longer repository cache lifetime reduces repeated Git operations and can improve repository-server efficiency.

### Sync Waves

Sync waves provide deterministic deployment ordering and are useful when one Kubernetes resource must exist before another resource is created or updated.

### Benchmarking

Performance tuning should be measured rather than assumed. The benchmark provides a repeatable method for comparing configuration changes.

---

# 📌 Troubleshooting

## Docker Permission Error

```bash
groups
newgrp docker
```

If necessary, log out and back in.

## kind Cluster Already Exists

Check:

```bash
kind get clusters
```

Delete the existing cluster if it is no longer needed:

```bash
kind delete cluster --name argocd-perf
```

Then recreate it.

## Argo CD Pods Not Ready

Inspect:

```bash
kubectl get pods -n argocd
```

Then inspect a failing pod:

```bash
kubectl describe pod <pod-name> -n argocd
```

Check logs:

```bash
kubectl logs <pod-name> -n argocd
```

## Repository Server Problems

Check:

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-repo-server \
  --tail=100
```

Verify:

```bash
kubectl get deployment argocd-repo-server -n argocd -o yaml
```

## Redis Problems

Check:

```bash
kubectl get pods -n argocd
```

Then:

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-redis
```

Verify Redis configuration:

```bash
kubectl exec \
  -n argocd \
  deployment/argocd-redis \
  -- redis-cli CONFIG GET maxmemory
```

## Application Not Synced

Check:

```bash
argocd app get <application-name>
```

Then inspect Kubernetes resources:

```bash
kubectl get applications -n argocd
kubectl describe application <application-name> -n argocd
```

---

# 🧹 Lab Cleanup

Remove the kind cluster:

```bash
kind delete cluster --name argocd-perf
```

Remove temporary repositories:

```bash
rm -rf \
  /tmp/argocd-app-a \
  /tmp/argocd-app-b \
  /tmp/argocd-app-c \
  /tmp/argocd-baseline
```

The Docker installation and command-line tools can remain installed if the EC2 instance will be reused for additional labs.

---

# 🎯 Expected Outcomes

After completing this lab, you should have:

* A functioning kind Kubernetes cluster named `argocd-perf`.
* A production-oriented Argo CD configuration running locally.
* Redis configured with a 256 MB memory ceiling and LRU eviction.
* Tuned application-controller concurrency.
* Server-side diff and apply enabled.
* Lease resources excluded from tracking.
* A 180-second reconciliation timeout.
* A 24-hour repository cache.
* Three applications demonstrating different synchronization optimization strategies.
* A baseline application for performance comparison.
* A reusable `benchmark-sync.sh` script.
* Measured synchronization performance stored in `comparison-summary.txt`.
* Verified sync-wave ordering.

---

# 🏁 Conclusion

This lab demonstrates a systematic approach to **optimizing Argo CD performance** rather than relying on a single configuration change.

The optimization covers the major performance layers of an Argo CD deployment:

```text
Redis Cache
     │
     ▼
Application Controller
     │
     ▼
Repository Server
     │
     ▼
Resource Tracking
     │
     ▼
Sync Policies
     │
     ▼
Sync Waves
     │
     ▼
Benchmarking
```

The most important lesson is to **measure performance before and after tuning**. Controller concurrency, cache configuration, server-side operations, resource exclusions, and synchronization policies can affect performance differently depending on workload characteristics.

The resulting benchmark provides a reusable foundation for testing future Argo CD configuration changes and determining whether an optimization produces an actual improvement or an unintended regression.

## ⭐ Key Takeaway

> **Optimize → Measure → Compare → Validate → Repeat**

This workflow is essential when tuning Argo CD for higher throughput and more predictable GitOps deployments in production environments.
