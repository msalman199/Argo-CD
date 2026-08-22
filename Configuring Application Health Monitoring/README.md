# Configuring Application Health Monitoring with Argo CD

A hands-on GitOps lab demonstrating how to configure **Argo CD application health monitoring**, detect configuration drift, implement self-healing, troubleshoot unhealthy workloads, and build practical health-monitoring tools for Kubernetes applications.

## 📋 Lab Objectives

By the end of this lab, you will be able to:

* Understand how Argo CD monitors application health and detects configuration drift
* Configure and enable application health monitoring in Argo CD
* Deploy Kubernetes applications through a GitOps workflow
* Test discrepancies between the desired Git state and live Kubernetes state
* Analyze Argo CD health and synchronization statuses
* Implement automated self-healing for configuration drift
* Configure custom health checks for Kubernetes resources
* Troubleshoot unhealthy and misconfigured applications
* Build shell-based health monitoring dashboards
* Apply best practices for reliable application health monitoring

## 🏗️ Lab Architecture

```text
                    ┌─────────────────────┐
                    │    Git Repository   │
                    │                     │
                    │ Kubernetes Manifests│
                    └──────────┬──────────┘
                               │
                               │ Desired State
                               ▼
                    ┌─────────────────────┐
                    │       Argo CD       │
                    │                     │
                    │ ┌─────────────────┐ │
                    │ │ Health Monitor  │ │
                    │ ├─────────────────┤ │
                    │ │ Drift Detection │ │
                    │ ├─────────────────┤ │
                    │ │ Self-Healing    │ │
                    │ └─────────────────┘ │
                    └──────────┬──────────┘
                               │
                         Reconciliation
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Kubernetes / kind   │
                    │                     │
                    │ Deployments         │
                    │ Pods                │
                    │ Services            │
                    │ ConfigMaps          │
                    └─────────────────────┘
```

## 🔄 GitOps Health Monitoring Workflow

```text
Developer
   │
   ▼
Git Repository
   │
   │ Desired Configuration
   ▼
Argo CD
   │
   ├── Compare Git State
   │
   ├── Compare Live State
   │
   ├── Detect Drift
   │
   ├── Evaluate Health
   │
   └── Reconcile
          │
          ▼
    Kubernetes Cluster
          │
          ▼
   Application Resources
```

## 📚 Key Concepts

### Application Health

Argo CD evaluates the runtime health of Kubernetes resources and applications. Typical health states include:

| Health Status | Meaning                                              |
| ------------- | ---------------------------------------------------- |
| `Healthy`     | Application resources are operating as expected      |
| `Progressing` | Resources are still being deployed or becoming ready |
| `Degraded`    | One or more resources are unhealthy                  |
| `Suspended`   | Resource execution has been intentionally suspended  |
| `Missing`     | Expected resource is missing                         |
| `Unknown`     | Argo CD cannot determine the resource health         |

### Sync Status

| Sync Status | Meaning                                        |
| ----------- | ---------------------------------------------- |
| `Synced`    | Live cluster state matches Git                 |
| `OutOfSync` | Live state differs from Git                    |
| `Unknown`   | Argo CD cannot determine synchronization state |

### Configuration Drift

Configuration drift occurs when the live Kubernetes environment differs from the desired configuration stored in Git.

Example:

```text
Git:
replicas: 2

        ↓

Kubernetes:
replicas: 4

        ↓

Argo CD:
OutOfSync
```

### Self-Healing

When automated synchronization and `selfHeal` are enabled, Argo CD can automatically reconcile certain changes made directly to the Kubernetes cluster.

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

## 🛠️ Prerequisites

Before starting this lab, you should have:

* Basic Kubernetes knowledge
* Understanding of Pods, Deployments, Services, and ConfigMaps
* Familiarity with Git
* Basic YAML knowledge
* Linux command-line experience
* Basic understanding of containers
* Familiarity with Kubernetes manifests

## 💻 Lab Environment

The lab uses an **Al Nafi Linux-based cloud machine**.

The environment starts with a bare Linux system, so the required tooling is installed during the lab.

## 🔧 Required Tools

The lab uses:

* Docker
* Kubernetes
* kind
* kubectl
* Argo CD
* Argo CD CLI
* Git
* jq
* Bash

## 🚀 Task 1 — Set Up the Lab Environment

### 1.1 Install Required Tools

Update the operating system and install Docker:

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

Install `kubectl`:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Install kind:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Install Argo CD CLI:

```bash
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

rm argocd-linux-amd64
```

Install Git:

```bash
sudo apt install -y git
```

Apply the Docker group change:

```bash
newgrp docker
```

Verify the tools:

```bash
docker --version
kubectl version --client
kind version
argocd version --client
git --version
```

## ☸️ 1.2 Create the Kubernetes Cluster

Create a kind configuration:

```bash
cat <<EOF > kind-config.yaml
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
  --name=argocd-lab
```

Verify:

```bash
kubectl cluster-info --context kind-argocd-lab
kubectl get nodes
```

Expected result:

```text
NAME                   STATUS   ROLES           AGE
argocd-lab-control-plane   Ready    control-plane   ...
```

## 🎯 1.3 Install Argo CD

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

Check the components:

```bash
kubectl get pods -n argocd
```

## 🌐 1.4 Access the Argo CD UI

Port-forward the Argo CD server:

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443 &
```

Retrieve the initial admin password:

```bash
ARGOCD_PASSWORD=$(kubectl -n argocd get secret \
  argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

echo "$ARGOCD_PASSWORD"
```

Login through the CLI:

```bash
argocd login localhost:8080 \
  --username admin \
  --password "$ARGOCD_PASSWORD" \
  --insecure
```

Access:

```text
https://localhost:8080
```

---

# 📦 Task 2 — Configure Application Health Monitoring

## 2.1 Create the Application Repository

Create the Git repository:

```bash
mkdir -p ~/argocd-health-lab
cd ~/argocd-health-lab

git init

git config user.name "Lab User"
git config user.email "lab@example.com"

mkdir -p k8s-manifests
```

## Create a Sample Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
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
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

Save it as:

```text
k8s-manifests/deployment.yaml
```

## Create a Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app-service
  namespace: default
  labels:
    app: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

Save it as:

```text
k8s-manifests/service.yaml
```

## Create a ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-app-config
  namespace: default
data:
  app.properties: |
    environment=production
    debug=false
    log.level=info
```

Save it as:

```text
k8s-manifests/configmap.yaml
```

Commit the initial application:

```bash
git add .
git commit -m "Initial application manifests"
```

## 2.2 Create an Argo CD Application

Create:

```text
argocd-application.yaml
```

Example configuration:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app-health-monitor
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: file:///home/$USER/argocd-health-lab
    targetRevision: HEAD
    path: k8s-manifests

  destination:
    server: https://kubernetes.default.svc
    namespace: default

  syncPolicy:
    automated:
      prune: true
      selfHeal: false

    syncOptions:
    - CreateNamespace=true

    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas

  revisionHistoryLimit: 10
```

Apply it:

```bash
kubectl apply -f argocd-application.yaml
```

Verify:

```bash
argocd app list
```

## ⚙️ 2.3 Configure Health Checks

Argo CD supports custom health assessments for resources where the default health logic is insufficient.

A custom health configuration can be stored in the Argo CD configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations.health.argoproj.io_Application: |
    hs = {}
    hs.status = "Progressing"
    hs.message = ""

    if obj.status ~= nil then
      if obj.status.health ~= nil then
        hs.status = obj.status.health.status

        if obj.status.health.message ~= nil then
          hs.message = obj.status.health.message
        end
      end
    end

    return hs

  timeout.reconciliation: 180s

  application.instanceLabelKey: argocd.argoproj.io/instance

  application.resourceTrackingMethod: annotation
```

Apply:

```bash
kubectl apply -f health-config.yaml
```

Restart the relevant Argo CD components:

```bash
kubectl rollout restart \
  deployment/argocd-server \
  -n argocd

kubectl rollout restart \
  deployment/argocd-application-controller \
  -n argocd
```

Verify:

```bash
kubectl rollout status \
  deployment/argocd-server \
  -n argocd

kubectl rollout status \
  deployment/argocd-application-controller \
  -n argocd
```

---

# 🔄 Task 3 — Test Configuration Drift

## 3.1 Synchronize the Application

Sync:

```bash
argocd app sync sample-app-health-monitor
```

Wait:

```bash
argocd app wait \
  sample-app-health-monitor \
  --timeout 300
```

Inspect:

```bash
argocd app get sample-app-health-monitor
```

Verify Kubernetes resources:

```bash
kubectl get all -l app=sample-app
kubectl get configmap sample-app-config
```

At this stage, the application should be synchronized with Git.

## 3.2 Create Configuration Drift

Manually change the live Kubernetes state:

```bash
kubectl scale deployment sample-app --replicas=4
```

Modify the ConfigMap:

```bash
kubectl patch configmap sample-app-config \
  --patch='{"data":{"app.properties":"environment=development\ndebug=true\nlog.level=debug"}}'
```

Add a label:

```bash
kubectl label service \
  sample-app-service \
  environment=modified
```

Inspect the live state:

```bash
kubectl get deployment sample-app -o yaml
kubectl get configmap sample-app-config -o yaml
kubectl get service sample-app-service --show-labels
```

## 3.3 Detect the Drift

Check Argo CD:

```bash
argocd app get sample-app-health-monitor
```

View the difference:

```bash
argocd app diff sample-app-health-monitor
```

Inspect synchronization information:

```bash
argocd app get sample-app-health-monitor \
  --output json | jq '.status.sync'
```

Inspect application events:

```bash
kubectl describe \
  application sample-app-health-monitor \
  -n argocd
```

The important GitOps relationship is:

```text
Desired State
     │
     ▼
    Git
     │
     ▼
  Argo CD
     │
     │ Compare
     ▼
 Live Cluster
     │
     ▼
Drift Detected
```

## 3.4 Build a Health Monitoring Script

Create:

```text
monitor-health.sh
```

Example:

```bash
#!/bin/bash

echo "=== Argo CD Application Health Monitor ==="
echo "Application: sample-app-health-monitor"
echo "Press Ctrl+C to stop monitoring"
echo

while true; do
    echo "--- $(date) ---"

    STATUS=$(argocd app get \
      sample-app-health-monitor \
      --output json)

    HEALTH=$(echo "$STATUS" |
      jq -r '.status.health.status // "Unknown"')

    SYNC=$(echo "$STATUS" |
      jq -r '.status.sync.status // "Unknown"')

    REVISION=$(echo "$STATUS" |
      jq -r '.status.sync.revision // "Unknown"')

    echo "Health Status: $HEALTH"
    echo "Sync Status: $SYNC"
    echo "Git Revision: $REVISION"

    OUT_OF_SYNC=$(echo "$STATUS" |
      jq -r '.status.resources[] |
      select(.status != "Synced") |
      .name' 2>/dev/null)

    if [ -n "$OUT_OF_SYNC" ]; then
        echo "Out of Sync Resources:"
        echo "$OUT_OF_SYNC"
    fi

    echo "------------------------"
    sleep 10
done
```

Make it executable:

```bash
chmod +x monitor-health.sh
```

Run:

```bash
./monitor-health.sh
```

## 3.5 Test Self-Healing

Enable self-healing:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Apply the updated Application configuration:

```bash
kubectl apply -f argocd-application-selfheal.yaml
```

Create drift:

```bash
kubectl scale deployment sample-app --replicas=6
```

Modify the ConfigMap:

```bash
kubectl patch configmap sample-app-config \
  --patch='{"data":{"app.properties":"environment=testing\ndebug=true\nlog.level=trace"}}'
```

Monitor:

```bash
for i in {1..30}; do
    echo "Check $i:"
    kubectl get deployment sample-app \
      -o jsonpath='{.spec.replicas}'

    echo " replicas"

    argocd app get sample-app-health-monitor \
      --output json |
      jq -r '.status.sync.status'

    sleep 10
done
```

### Important Note

The lab configuration intentionally ignores Deployment replica differences:

```yaml
ignoreDifferences:
- group: apps
  kind: Deployment
  jsonPointers:
  - /spec/replicas
```

Therefore, changes to the replica count may not be reconciled in the same way as other drift. This is useful for demonstrating how `ignoreDifferences` affects Argo CD reconciliation behavior.

## 3.6 Simulate an Application Failure

Create a Deployment using an invalid image:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: failing-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: failing-app
  template:
    metadata:
      labels:
        app: failing-app
    spec:
      containers:
      - name: failing-container
        image: nonexistent-image:latest
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /health
            port: 80
```

Commit:

```bash
git add k8s-manifests/failing-deployment.yaml
git commit -m "Add failing deployment for health monitoring test"
```

Synchronize:

```bash
argocd app sync sample-app-health-monitor
```

Inspect:

```bash
argocd app get sample-app-health-monitor
kubectl get pods -l app=failing-app
```

This demonstrates how an application can be synchronized with Git while still having an unhealthy runtime state.

---

# 🧠 Task 4 — Advanced Health Monitoring

## 4.1 Custom Deployment Health Check

Custom health logic can evaluate whether all expected replicas are ready:

```lua
hs = {}

if obj.status ~= nil then
  if obj.status.replicas ~= nil and
     obj.status.readyReplicas ~= nil then

    if obj.status.replicas == obj.status.readyReplicas then
      hs.status = "Healthy"
      hs.message = "All replicas are ready"
    else
      hs.status = "Progressing"
      hs.message = "Waiting for replicas to be ready"
    end

  else
    hs.status = "Progressing"
    hs.message = "Deployment is starting"
  end
else
  hs.status = "Progressing"
  hs.message = "Deployment status unknown"
end

return hs
```

This type of customization can provide more meaningful health information for specific environments.

## 4.2 Custom Service Health Check

For LoadBalancer services, health can depend on whether an ingress address has been assigned:

```lua
hs = {}

if obj.spec.type == "LoadBalancer" then

  if obj.status.loadBalancer.ingress ~= nil and
     #obj.status.loadBalancer.ingress > 0 then

    hs.status = "Healthy"
    hs.message = "LoadBalancer has assigned an address"

  else
    hs.status = "Progressing"
    hs.message = "Waiting for LoadBalancer address"
  end

else
  hs.status = "Healthy"
  hs.message = "Service is ready"
end

return hs
```

## 📊 4.3 Build a Health Dashboard

A shell-based dashboard can display:

* Application health
* Sync state
* Git revision
* Resource states
* Application conditions
* Deployment status
* Pod status
* Service information

Example:

```bash
#!/bin/bash

clear_screen() {
    clear

    echo "=== Argo CD Application Health Dashboard ==="
    echo "Application: sample-app-health-monitor"
    echo "Timestamp: $(date)"
    echo "Press Ctrl+C to exit"
    echo
}

get_health_status() {

    local app_name=$1

    local status=$(argocd app get \
      "$app_name" \
      --output json 2>/dev/null)

    if [ $? -eq 0 ]; then

        local health=$(echo "$status" |
          jq -r '.status.health.status // "Unknown"')

        local sync=$(echo "$status" |
          jq -r '.status.sync.status // "Unknown"')

        local revision=$(echo "$status" |
          jq -r '.status.sync.revision[0:8] // "Unknown"')

        echo "Health Status: $health"
        echo "Sync Status: $sync"
        echo "Git Revision: $revision"

        echo
        echo "Resource Status:"

        echo "$status" |
          jq -r '.status.resources[]? |
          "  \(.kind)/\(.name): \(.status)"'

        echo
        echo "Application Conditions:"

        echo "$status" |
          jq -r '.status.conditions[]? |
          "  \(.type): \(.message)"'

    else
        echo "Failed to retrieve application status"
    fi
}

get_cluster_resources() {

    echo
    echo "Cluster Resources:"

    echo
    echo "Deployments:"

    kubectl get deployments \
      -l app=sample-app

    echo
    echo "Pods:"

    kubectl get pods \
      -l app=sample-app

    echo
    echo "Services:"

    kubectl get services \
      -l app=sample-app
}

while true; do

    clear_screen

    get_health_status \
      "sample-app-health-monitor"

    get_cluster_resources

    echo
    echo "Next update in 15 seconds..."

    sleep 15

done
```

Run:

```bash
chmod +x health-dashboard.sh
./health-dashboard.sh
```

---

# 🔍 Task 5 — Troubleshooting

## 5.1 Common Health Monitoring Problems

Typical causes of unhealthy applications include:

### Incorrect Image

```text
ImagePullBackOff
ErrImagePull
```

Check:

```bash
kubectl describe pod <pod-name>
```

### Failed Readiness Probe

```text
Readiness probe failed
```

Check:

```bash
kubectl describe pod <pod-name>
```

Verify:

* Endpoint
* Port
* Initial delay
* Timeout
* Application startup time

### Failed Liveness Probe

Repeated liveness failures can cause Kubernetes to restart the container.

Check:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Resource Constraints

A pod may remain pending when requested resources exceed available node capacity.

Check:

```bash
kubectl describe pod <pod-name>
kubectl describe node
```

### OutOfSync Application

Check:

```bash
argocd app get sample-app-health-monitor
```

Then:

```bash
argocd app diff sample-app-health-monitor
```

### Application Events

```bash
kubectl describe \
  application sample-app-health-monitor \
  -n argocd
```

## 5.2 Troubleshooting Workflow

Use the following workflow:

```text
Application Unhealthy
        │
        ▼
Check Argo CD Status
        │
        ▼
Check Sync Status
        │
        ▼
Check Resource Status
        │
        ▼
Check Kubernetes Events
        │
        ▼
Check Pod Status
        │
        ▼
Check Pod Logs
        │
        ▼
Check Probes
        │
        ▼
Check Resources
        │
        ▼
Fix Git Configuration
        │
        ▼
Sync / Reconcile
        │
        ▼
Healthy + Synced
```

---

# ✅ Best Practices

## 1. Use Both Readiness and Liveness Probes

Readiness determines whether a Pod should receive traffic.

Liveness determines whether a container should be restarted.

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80

livenessProbe:
  httpGet:
    path: /
    port: 80
```

## 2. Use Startup Probes for Slow Applications

Startup probes prevent liveness checks from restarting applications before they have completed initialization.

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  failureThreshold: 10
  periodSeconds: 5
```

## 3. Define Resource Requests and Limits

```yaml
resources:
  requests:
    memory: "32Mi"
    cpu: "100m"
  limits:
    memory: "64Mi"
    cpu: "200m"
```

## 4. Keep Health Endpoints Lightweight

Health endpoints should be:

* Fast
* Reliable
* Independent where possible
* Safe to call frequently

Example:

```text
GET /health
→ HTTP 200 OK
```

## 5. Use Git as the Source of Truth

Avoid making permanent manual changes directly in Kubernetes.

Preferred workflow:

```text
Change
  ↓
Git
  ↓
Pull Request
  ↓
Review
  ↓
Merge
  ↓
Argo CD
  ↓
Kubernetes
```

## 6. Use Self-Healing Carefully

Self-healing is powerful:

```yaml
automated:
  selfHeal: true
```

However, understand which resources and differences should be ignored before enabling it in production.

## 7. Avoid Excessive Health Customization

Use built-in Argo CD health assessment whenever it is sufficient.

Custom health checks should be introduced when they provide meaningful application-specific information.

## 8. Monitor Both Health and Sync Status

An application can be:

```text
Synced + Healthy
```

or:

```text
Synced + Degraded
```

or:

```text
OutOfSync + Healthy
```

These statuses answer different questions:

```text
Sync Status → Does the cluster match Git?

Health Status → Is the application operating correctly?
```

---

# 📁 Suggested Repository Structure

```text
argocd-health-lab/
├── README.md
├── argocd-application.yaml
├── argocd-application-selfheal.yaml
├── health-config.yaml
├── advanced-health-config.yaml
├── monitor-health.sh
├── health-dashboard.sh
├── troubleshoot-health.sh
└── k8s-manifests/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── failing-deployment.yaml
    └── optimized-deployment.yaml
```

---

# 🧪 Verification Commands

Check the Argo CD application:

```bash
argocd app get sample-app-health-monitor
```

Check synchronization:

```bash
argocd app get sample-app-health-monitor \
  --output json |
  jq '.status.sync'
```

Check health:

```bash
argocd app get sample-app-health-monitor \
  --output json |
  jq '.status.health'
```

Check resources:

```bash
kubectl get all
```

Check Pods:

```bash
kubectl get pods -o wide
```

Check events:

```bash
kubectl get events \
  --sort-by=.metadata.creationTimestamp
```

Check Argo CD components:

```bash
kubectl get pods -n argocd
```

Check application resources:

```bash
argocd app resources sample-app-health-monitor
```

View differences:

```bash
argocd app diff sample-app-health-monitor
```

---

# 🎯 Learning Outcomes

After completing this lab, you should understand:

* How Argo CD evaluates application health
* How GitOps detects configuration drift
* The difference between health status and sync status
* How Kubernetes probes contribute to application health
* How Argo CD automated synchronization works
* How self-healing can automatically reconcile changes
* How custom health checks can extend Argo CD
* How to investigate unhealthy workloads
* How to build lightweight health-monitoring scripts
* How to design reliable GitOps health-monitoring workflows

---

# 🏁 Conclusion

This lab demonstrates a complete **Argo CD application health-monitoring workflow**, from deploying applications through GitOps to detecting configuration drift and troubleshooting unhealthy workloads.

You configured Kubernetes health probes, deployed applications through Argo CD, created intentional configuration drift, inspected Argo CD synchronization and health information, tested self-healing, created failing workloads, and explored custom health-check logic.

The most important operational principle demonstrated by this lab is:

```text
Git = Desired State
       ↓
Argo CD = Reconciliation + Health Monitoring
       ↓
Kubernetes = Runtime State
       ↓
Health + Sync Status = Operational Visibility
```

Application health monitoring is essential for reliable GitOps environments because **successful synchronization does not necessarily mean an application is healthy**. Argo CD provides the visibility and reconciliation mechanisms needed to continuously compare desired state with actual state while assessing whether deployed workloads are functioning correctly.

## ⭐ Key Takeaways

* **Health Monitoring** provides visibility into application runtime health.
* **Drift Detection** identifies differences between Git and the Kubernetes cluster.
* **Self-Healing** can automatically restore the desired state.
* **Health Probes** provide Kubernetes with application-level health information.
* **Custom Health Checks** allow specialized resources to expose meaningful health states.
* **Troubleshooting Tools** help identify deployment, probe, image, and resource problems.
* **GitOps Best Practices** keep Git as the authoritative source of application configuration.

---

## 🏷️ Technologies Used

```text
Kubernetes
kind
Docker
kubectl
Argo CD
Argo CD CLI
Git
Bash
jq
YAML
GitOps
```

## 👨‍💻 Lab Focus

**Primary Topic:** Argo CD Application Health Monitoring

**Environment:** Linux + kind Kubernetes

**Deployment Model:** GitOps

**Monitoring Areas:**

* Application Health
* Synchronization
* Configuration Drift
* Self-Healing
* Kubernetes Probes
* Custom Health Checks
* Troubleshooting
* Operational Visibility
