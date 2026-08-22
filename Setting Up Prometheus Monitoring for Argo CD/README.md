<div align="center">

# 📊 Setting Up Prometheus Monitoring for Argo CD

![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Argo CD](https://img.shields.io/badge/Argo%20CD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=helm&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

**A hands-on lab for integrating Prometheus and Grafana observability with Argo CD GitOps deployments**

</div>

---

## 📑 Table of Contents

- [🎯 Lab Objectives](#-lab-objectives)
- [📋 Prerequisites](#-prerequisites)
- [🖥️ Lab Environment](#️-lab-environment)
- [🧩 Task 1: Prepare the Environment and Install Prerequisites](#-task-1-prepare-the-environment-and-install-prerequisites)
- [⚙️ Task 2: Install and Configure Argo CD](#️-task-2-install-and-configure-argo-cd)
- [📈 Task 3: Set Up Prometheus Monitoring](#-task-3-set-up-prometheus-monitoring)
- [🔗 Task 4: Configure ServiceMonitors for Argo CD](#-task-4-configure-servicemonitors-for-argo-cd)
- [✅ Task 5: Verify Prometheus Integration and Monitoring](#-task-5-verify-prometheus-integration-and-monitoring)
- [📉 Task 6: Access Grafana Dashboard](#-task-6-access-grafana-dashboard)
- [🛠️ Task 7: Troubleshooting and Verification](#️-task-7-troubleshooting-and-verification)
- [🚑 Troubleshooting Common Issues](#-troubleshooting-common-issues)
- [📚 Key Concepts](#-key-concepts)
- [🏁 Conclusion](#-conclusion)

---

## 🎯 Lab Objectives

By the end of this lab, you will be able to:

| # | Objective |
|---|-----------|
| 1 | Install and configure Prometheus monitoring system in Kubernetes |
| 2 | Set up Prometheus to collect metrics from Argo CD applications |
| 3 | Configure ServiceMonitor resources for Argo CD monitoring |
| 4 | Access and interpret Prometheus metrics dashboard |
| 5 | Understand the integration between Prometheus and Argo CD for comprehensive application monitoring |
| 6 | Troubleshoot common monitoring setup issues |

---

## 📋 Prerequisites

Before starting this lab, you should have:

| Requirement | Details |
|-------------|---------|
| ☸️ Kubernetes Concepts | Basic understanding of pods, services, deployments |
| 📄 YAML | Familiarity with YAML configuration files |
| 🚀 Argo CD Fundamentals | Knowledge of Argo CD fundamentals from previous labs |
| 📊 Monitoring Concepts | Understanding of monitoring concepts and metrics |
| 🐧 CLI Operations | Experience with command-line interface operations |

---

## 🖥️ Lab Environment

> **☁️ Al Nafi Cloud Machine:** Al Nafi provides Linux-based cloud machines for this lab. Simply click **Start Lab** to access your dedicated environment. The provided Linux machine is bare metal with no pre-installed tools — you will install all required components during the lab exercises.
>
> **⚠️ Important:** All tasks in this lab will be performed on a single Linux machine. No additional virtual machines or remote hosts are required.

---

## 🧩 Task 1: Prepare the Environment and Install Prerequisites

### 🔹 Subtask 1.1: Update System and Install Docker

First, update your system and install Docker, which is required for Kubernetes.

```bash
# 🔄 Update system packages
sudo apt update && sudo apt upgrade -y

# 🧰 Install required packages
sudo apt install -y curl wget apt-transport-https ca-certificates gnupg lsb-release

# 🐳 Install Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# 👤 Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# ✅ Verify Docker installation
docker --version
```

### 🔹 Subtask 1.2: Install kubectl and kind

Install kubectl for Kubernetes management and kind for creating local clusters.

```bash
# 📥 Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# ✅ Verify kubectl installation
kubectl version --client

# 📥 Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# ✅ Verify kind installation
kind version
```

### 🔹 Subtask 1.3: Create Kubernetes Cluster

Create a local Kubernetes cluster using kind.

```bash
# 📝 Create kind cluster configuration
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
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 30080
    hostPort: 30080
    protocol: TCP
  - containerPort: 30090
    hostPort: 30090
    protocol: TCP
EOF

# ▶️ Create the cluster
kind create cluster --config=kind-config.yaml --name=argocd-monitoring

# ✅ Verify cluster is running
kubectl cluster-info
kubectl get nodes
```

### 🔹 Subtask 1.4: Install Helm

Install Helm package manager for Kubernetes applications.

```bash
# 📥 Download and install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# ✅ Verify Helm installation
helm version

# ➕ Add required Helm repositories
helm repo add argo https://argoproj.github.io/argo-helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## ⚙️ Task 2: Install and Configure Argo CD

### 🔹 Subtask 2.1: Install Argo CD using Helm

Install Argo CD with monitoring capabilities enabled.

```bash
# 📁 Create namespace for Argo CD
kubectl create namespace argocd

# 📝 Create values file for Argo CD with monitoring enabled
cat <<EOF > argocd-values.yaml
server:
  service:
    type: NodePort
    nodePortHttp: 30080
  metrics:
    enabled: true
    service:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8083"
        prometheus.io/path: "/metrics"
      serviceMonitor:
        enabled: true
        selector:
          matchLabels:
            app.kubernetes.io/name: argocd-server-metrics
        namespace: monitoring

controller:
  metrics:
    enabled: true
    service:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8082"
        prometheus.io/path: "/metrics"
      serviceMonitor:
        enabled: true
        selector:
          matchLabels:
            app.kubernetes.io/name: argocd-application-controller-metrics
        namespace: monitoring

repoServer:
  metrics:
    enabled: true
    service:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8084"
        prometheus.io/path: "/metrics"
      serviceMonitor:
        enabled: true
        selector:
          matchLabels:
            app.kubernetes.io/name: argocd-repo-server-metrics
        namespace: monitoring

applicationSet:
  metrics:
    enabled: true
    service:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
      serviceMonitor:
        enabled: true
        selector:
          matchLabels:
            app.kubernetes.io/name: argocd-applicationset-controller-metrics
        namespace: monitoring
EOF

# 📥 Install Argo CD
helm install argocd argo/argo-cd -n argocd -f argocd-values.yaml

# ⏳ Wait for Argo CD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### 🔹 Subtask 2.2: Access Argo CD and Get Initial Password

```bash
# 🔑 Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# ✅ Verify Argo CD is accessible
kubectl get svc -n argocd
kubectl get pods -n argocd

# 🌐 Test access to Argo CD (should show login page)
curl -k http://localhost:30080
```

---

## 📈 Task 3: Set Up Prometheus Monitoring

### 🔹 Subtask 3.1: Create Monitoring Namespace and Install Prometheus

```bash
# 📁 Create monitoring namespace
kubectl create namespace monitoring

# 📝 Create Prometheus values file
cat <<EOF > prometheus-values.yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}
    ruleSelectorNilUsesHelmValues: false
    ruleSelector: {}
    ruleNamespaceSelector: {}
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
  service:
    type: NodePort
    nodePort: 30090

grafana:
  enabled: true
  service:
    type: NodePort
    nodePort: 30091
  adminPassword: admin123

alertmanager:
  enabled: true
  service:
    type: NodePort
    nodePort: 30092

kubeStateMetrics:
  enabled: true

nodeExporter:
  enabled: true

prometheusOperator:
  enabled: true
EOF

# 📥 Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring -f prometheus-values.yaml

# ⏳ Wait for Prometheus to be ready
kubectl wait --for=condition=available --timeout=600s deployment/prometheus-grafana -n monitoring
kubectl wait --for=condition=available --timeout=600s deployment/prometheus-kube-prometheus-operator -n monitoring
```

### 🔹 Subtask 3.2: Verify Prometheus Installation

```bash
# 🔍 Check all monitoring components
kubectl get pods -n monitoring

# 🔍 Check services
kubectl get svc -n monitoring

# 🌐 Verify Prometheus is accessible
curl http://localhost:30090

# 📄 Check if Prometheus can access its configuration
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus --tail=50
```

---

## 🔗 Task 4: Configure ServiceMonitors for Argo CD

### 🔹 Subtask 4.1: Create ServiceMonitor for Argo CD Server

```bash
# 📝 Create ServiceMonitor for Argo CD Server
cat <<EOF > argocd-server-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-server-metrics
  namespace: monitoring
  labels:
    app.kubernetes.io/name: argocd-server-metrics
    app.kubernetes.io/part-of: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server-metrics
  namespaceSelector:
    matchNames:
    - argocd
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
EOF

kubectl apply -f argocd-server-servicemonitor.yaml
```

### 🔹 Subtask 4.2: Create ServiceMonitor for Argo CD Application Controller

```bash
# 📝 Create ServiceMonitor for Application Controller
cat <<EOF > argocd-controller-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-application-controller-metrics
  namespace: monitoring
  labels:
    app.kubernetes.io/name: argocd-application-controller-metrics
    app.kubernetes.io/part-of: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-application-controller-metrics
  namespaceSelector:
    matchNames:
    - argocd
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
EOF

kubectl apply -f argocd-controller-servicemonitor.yaml
```

### 🔹 Subtask 4.3: Create ServiceMonitor for Argo CD Repo Server

```bash
# 📝 Create ServiceMonitor for Repo Server
cat <<EOF > argocd-repo-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-repo-server-metrics
  namespace: monitoring
  labels:
    app.kubernetes.io/name: argocd-repo-server-metrics
    app.kubernetes.io/part-of: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server-metrics
  namespaceSelector:
    matchNames:
    - argocd
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
EOF

kubectl apply -f argocd-repo-servicemonitor.yaml
```

### 🔹 Subtask 4.4: Verify ServiceMonitors are Created

```bash
# 📋 List all ServiceMonitors
kubectl get servicemonitors -n monitoring

# 🔍 Check ServiceMonitor details
kubectl describe servicemonitor argocd-server-metrics -n monitoring
kubectl describe servicemonitor argocd-application-controller-metrics -n monitoring
kubectl describe servicemonitor argocd-repo-server-metrics -n monitoring

# ✅ Verify Argo CD metrics services exist
kubectl get svc -n argocd | grep metrics
```

---

## ✅ Task 5: Verify Prometheus Integration and Monitoring

### 🔹 Subtask 5.1: Check Prometheus Targets

```bash
# 🌐 Port forward to access Prometheus UI locally
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &

# ⏳ Wait a moment for port forwarding to establish
sleep 5

# 🔍 Check if Prometheus can reach Argo CD targets
curl -s "http://localhost:9090/api/v1/targets" | grep -i argocd

# 🌐 You can also access Prometheus UI at http://localhost:9090 in a browser
# Navigate to Status -> Targets to see Argo CD endpoints
```

### 🔹 Subtask 5.2: Query Argo CD Metrics

```bash
# 🔍 Query some basic Argo CD metrics
echo "Checking Argo CD application metrics..."
curl -s "http://localhost:9090/api/v1/query?query=argocd_app_info" | head -200

echo "Checking Argo CD cluster metrics..."
curl -s "http://localhost:9090/api/v1/query?query=argocd_cluster_info" | head -200

echo "Checking Argo CD repository metrics..."
curl -s "http://localhost:9090/api/v1/query?query=argocd_repo_pending_request_total" | head -200

# 🛑 Stop the port forwarding
pkill -f "kubectl port-forward.*prometheus"
```

### 🔹 Subtask 5.3: Create a Sample Application for Monitoring

```bash
# 📝 Create a sample application to generate metrics
cat <<EOF > sample-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

kubectl apply -f sample-app.yaml

# ⏳ Wait for the application to sync
sleep 30

# ✅ Check application status
kubectl get applications -n argocd
```

### 🔹 Subtask 5.4: Verify Metrics Collection

```bash
# 🌐 Port forward to Prometheus again
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &
sleep 5

# 🔍 Query metrics for the sample application
echo "Checking application sync metrics..."
curl -s "http://localhost:9090/api/v1/query?query=argocd_app_sync_total" | head -200

echo "Checking application health metrics..."
curl -s "http://localhost:9090/api/v1/query?query=argocd_app_health_status" | head -200

# 🛑 Stop port forwarding
pkill -f "kubectl port-forward.*prometheus"
```

---

## 📉 Task 6: Access Grafana Dashboard

### 🔹 Subtask 6.1: Access Grafana and Import Argo CD Dashboard

```bash
# 🌐 Port forward to Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 &
sleep 5

# 🔑 Get Grafana admin password (default is admin123 from our values)
echo "Grafana admin password: admin123"

# 🌐 You can access Grafana at http://localhost:3000
# Login with admin/admin123

# 📥 Download Argo CD dashboard JSON
curl -o argocd-dashboard.json https://raw.githubusercontent.com/argoproj/argo-cd/master/examples/dashboard.json

echo "Dashboard downloaded. Import this in Grafana UI:"
echo "1. Go to http://localhost:3000"
echo "2. Login with admin/admin123"
echo "3. Click '+' -> Import"
echo "4. Upload the argocd-dashboard.json file"
echo "5. Select Prometheus as data source"

# 🛑 Stop port forwarding after viewing
# pkill -f "kubectl port-forward.*grafana"
```

### 🔹 Subtask 6.2: Create Custom Argo CD Monitoring Dashboard

```bash
# 📝 Create a simple custom dashboard configuration
cat <<EOF > custom-argocd-dashboard.json
{
  "dashboard": {
    "id": null,
    "title": "Argo CD Custom Monitoring",
    "tags": ["argocd"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Application Count",
        "type": "stat",
        "targets": [
          {
            "expr": "count(argocd_app_info)",
            "refId": "A"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
      },
      {
        "id": 2,
        "title": "Sync Operations",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(argocd_app_sync_total[5m])",
            "refId": "A"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
      }
    ],
    "time": {"from": "now-1h", "to": "now"},
    "refresh": "30s"
  }
}
EOF

echo "Custom dashboard created. You can import this JSON in Grafana as well."
```

---

## 🛠️ Task 7: Troubleshooting and Verification

### 🔹 Subtask 7.1: Common Troubleshooting Steps

```bash
# 🔍 Check if all pods are running
echo "Checking Argo CD pods..."
kubectl get pods -n argocd

echo "Checking Monitoring pods..."
kubectl get pods -n monitoring

# 🔍 Check ServiceMonitor status
echo "Checking ServiceMonitors..."
kubectl get servicemonitors -n monitoring

# ✅ Verify metrics endpoints are accessible
echo "Testing Argo CD metrics endpoints..."
kubectl port-forward -n argocd svc/argocd-server-metrics 8083:8083 &
sleep 3
curl -s http://localhost:8083/metrics | head -10
pkill -f "kubectl port-forward.*argocd-server-metrics"

# 🔍 Check Prometheus configuration
echo "Checking Prometheus configuration..."
kubectl get prometheus -n monitoring -o yaml | grep -A 10 serviceMonitorSelector
```

### 🔹 Subtask 7.2: Verify End-to-End Monitoring

```bash
# 📝 Final verification script
cat <<EOF > verify-monitoring.sh
#!/bin/bash

echo "=== Argo CD Prometheus Monitoring Verification ==="

# Check cluster status
echo "1. Checking cluster status..."
kubectl get nodes

# Check Argo CD
echo "2. Checking Argo CD status..."
kubectl get pods -n argocd | grep Running | wc -l
echo "Argo CD running pods count above"

# Check Prometheus
echo "3. Checking Prometheus status..."
kubectl get pods -n monitoring | grep prometheus | grep Running | wc -l
echo "Prometheus running pods count above"

# Check ServiceMonitors
echo "4. Checking ServiceMonitors..."
kubectl get servicemonitors -n monitoring | grep argocd | wc -l
echo "Argo CD ServiceMonitors count above (should be 3)"

# Check if metrics are being collected
echo "5. Testing metrics collection..."
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &
sleep 5
METRICS_COUNT=\$(curl -s "http://localhost:9090/api/v1/query?query=argocd_app_info" | grep -o '"result":\[.*\]' | grep -o '\[.*\]' | grep -c '{')
echo "Argo CD metrics found: \$METRICS_COUNT"
pkill -f "kubectl port-forward.*prometheus"

echo "=== Verification Complete ==="
EOF

chmod +x verify-monitoring.sh
./verify-monitoring.sh
```

---

## 🚑 Troubleshooting Common Issues

<details>
<summary><strong>❗ Issue 1: ServiceMonitor Not Discovering Targets</strong></summary>

**Problem:** Prometheus is not discovering Argo CD targets.

**Solution:**

```bash
# 🔍 Check if ServiceMonitor labels match Prometheus selector
kubectl get prometheus -n monitoring -o yaml | grep -A 5 serviceMonitorSelector

# ✅ Ensure ServiceMonitor has correct labels
kubectl get servicemonitors -n monitoring --show-labels

# ✅ Check if metrics services exist
kubectl get svc -n argocd | grep metrics
```

</details>

<details>
<summary><strong>❗ Issue 2: Metrics Endpoints Not Accessible</strong></summary>

**Problem:** Argo CD metrics endpoints return errors.

**Solution:**

```bash
# 🔍 Check if metrics are enabled in Argo CD configuration
kubectl get configmap argocd-cmd-params-cm -n argocd -o yaml

# 🔄 Restart Argo CD pods if needed
kubectl rollout restart deployment/argocd-server -n argocd
kubectl rollout restart deployment/argocd-application-controller -n argocd
kubectl rollout restart deployment/argocd-repo-server -n argocd
```

</details>

<details>
<summary><strong>❗ Issue 3: Grafana Dashboard Not Showing Data</strong></summary>

**Problem:** Grafana dashboard shows no data.

**Solution:**

```bash
# ⚙️ Check Prometheus data source configuration in Grafana
# Ensure Prometheus URL is correct: http://prometheus-kube-prometheus-prometheus:9090

# 🔍 Verify time range in dashboard
# Check if metrics exist in Prometheus first
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &
curl -s "http://localhost:9090/api/v1/label/__name__/values" | grep argocd
pkill -f "kubectl port-forward.*prometheus"
```

</details>

---

## 📚 Key Concepts

| Concept | Description |
|---------|-------------|
| **Prometheus** | An open-source monitoring system that scrapes and stores time-series metrics from configured targets |
| **ServiceMonitor** | A Prometheus Operator custom resource that declaratively tells Prometheus which services to scrape and how |
| **kube-prometheus-stack** | A Helm chart bundling Prometheus, Grafana, Alertmanager, and supporting exporters as one integrated stack |
| **Grafana** | A visualization platform used to build dashboards on top of Prometheus metrics |
| **Metrics Endpoint** | An HTTP `/metrics` path exposed by an application (here, Argo CD's server, controller, and repo-server) in Prometheus exposition format |
| **prometheus.io/scrape Annotation** | A service annotation signaling that a target should be scraped for metrics |
| **Application Sync Metrics** | Argo CD metrics (e.g. `argocd_app_sync_total`, `argocd_app_health_status`) that expose GitOps deployment health and activity |

---

## 🏁 Conclusion

In this lab, you have successfully:

### 🏆 Key Accomplishments

- ✅ Set up a complete monitoring infrastructure by installing Prometheus and Grafana in your Kubernetes cluster
- ✅ Integrated Argo CD with Prometheus by enabling metrics collection and creating ServiceMonitor resources
- ✅ Configured comprehensive monitoring for all Argo CD components including the server, application controller, and repository server
- ✅ Verified metrics collection by querying Prometheus APIs and checking target discovery
- ✅ Set up visualization capabilities through Grafana dashboards for monitoring Argo CD operations

### 🌍 Real-World Applications

This monitoring setup provides crucial visibility into your Argo CD deployments, allowing you to:

- Track application sync operations and their success rates
- Monitor cluster and repository health in real-time
- Set up alerting for critical issues in your GitOps workflows
- Analyze performance trends and optimize your deployment processes

The integration between Prometheus and Argo CD creates a robust observability foundation that is essential for production GitOps environments. This monitoring capability enables proactive issue detection, performance optimization, and reliable continuous deployment operations.

**Key takeaways:** Monitoring is not just about collecting metrics — it's about gaining actionable insights into your deployment processes. The combination of Prometheus metrics collection and Grafana visualization provides the foundation for data-driven decisions in your GitOps workflows.

---

<div align="center">

![Al Nafi](https://img.shields.io/badge/Al%20Nafi-Cybersecurity%20Training-orange?style=for-the-badge)

</div>
