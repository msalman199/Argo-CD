# Application Health Monitoring with Argo CD

## 📌 Overview

This lab demonstrates how to build an advanced **application health monitoring system with Argo CD** using custom Lua health checks and Argo CD Notifications.

The implementation extends Argo CD's default health evaluation by defining custom health logic for **Deployments, Services, and ConfigMaps**. It also establishes a webhook-based notification pipeline that reports application health and synchronization state changes to a receiver running inside the Kubernetes cluster.

The lab validates the complete workflow by deliberately introducing application failures, verifying that Argo CD detects the degraded resources, confirming that notifications are delivered, and finally restoring the application to a healthy state.

---

## 🎯 Objectives

By completing this lab, you will learn how to:

* Configure a Kubernetes environment for Argo CD health monitoring
* Install and configure Argo CD and the Argo CD CLI
* Create custom Lua-based health checks
* Monitor Deployment, Service, and ConfigMap resources
* Extend Argo CD's default health evaluation
* Configure Argo CD Notifications
* Build a webhook notification receiver
* Send structured application health notifications
* Inject intentional application failures
* Detect `Degraded`, `Progressing`, and `Healthy` states
* Validate notification delivery
* Verify recovery after failures
* Implement end-to-end GitOps health observability

---

## 🏗️ Architecture

```text
                    ┌────────────────────────┐
                    │      Git Repository     │
                    │                        │
                    │ Kubernetes Manifests   │
                    │ Deployment             │
                    │ Service                │
                    │ ConfigMap              │
                    └────────────┬───────────┘
                                 │
                                 │ GitOps
                                 ▼
                    ┌────────────────────────┐
                    │        Argo CD         │
                    │                        │
                    │ Application Controller  │
                    │ Custom Lua Health       │
                    │ Evaluation              │
                    └────────────┬───────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
          ┌──────────────────┐     ┌────────────────────┐
          │ Kubernetes       │     │ Argo CD            │
          │ Application      │     │ Notifications      │
          │                  │     │ Controller         │
          │ Deployment       │     └─────────┬──────────┘
          │ Service          │               │
          │ ConfigMap        │               │ Webhook
          └──────────────────┘               ▼
                                   ┌────────────────────┐
                                   │ Webhook Receiver   │
                                   │                    │
                                   │ :9000/notify       │
                                   │                    │
                                   │ /tmp/webhook-      │
                                   │ alerts.log         │
                                   └────────────────────┘
```

---

## 🧰 Technologies Used

| Technology            | Purpose                                     |
| --------------------- | ------------------------------------------- |
| Kubernetes            | Container orchestration                     |
| kind                  | Local Kubernetes cluster                    |
| Docker                | Container runtime                           |
| kubectl               | Kubernetes management CLI                   |
| Argo CD               | GitOps continuous delivery                  |
| Argo CD CLI           | Argo CD command-line management             |
| Argo CD Notifications | Health and synchronization notifications    |
| Lua                   | Custom Argo CD health evaluation            |
| Python                | Webhook receiver interface                  |
| Git                   | Application configuration and GitOps source |
| NGINX                 | Sample application                          |

---

## 📋 Prerequisites

You should have:

* Linux command-line experience
* Understanding of Kubernetes Pods, Deployments, Services, and ConfigMaps
* Understanding of Kubernetes reconciliation
* Basic Git knowledge
* Familiarity with structured logs
* Ability to edit configuration files

The lab environment uses a dedicated **AWS EC2 Ubuntu instance provided by Al Nafi**. The base system is Ubuntu and the required tooling is installed during the lab.

---

# 🚀 Task 1 — Environment Setup

## 1. Install System Dependencies

```bash
sudo apt-get update -y
sudo apt-get install -y curl wget git jq
```

---

## 2. Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh

sudo usermod -aG docker "$USER"

newgrp docker

docker version
```

If Docker cannot connect to its daemon:

```bash
sudo systemctl start docker
sudo systemctl enable docker

docker version
```

The lab specifically uses Docker as the container runtime for the kind Kubernetes cluster.

---

## 3. Install kubectl

```bash
KUBECTL_VERSION=$(curl -fsSL https://dl.k8s.io/release/stable.txt)

curl -fsSL \
"https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
-o kubectl

sudo install -o root -g root -m 0755 \
kubectl /usr/local/bin/kubectl

rm kubectl

kubectl version --client
```

---

## 4. Install kind

```bash
curl -fsSL \
https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64 \
-o kind

sudo install -o root -g root -m 0755 \
kind /usr/local/bin/kind

rm kind

kind version
```

---

## 5. Install Argo CD CLI

```bash
ARGOCD_VERSION=$(curl -fsSL \
https://api.github.com/repos/argoproj/argo-cd/releases/latest \
| jq -r '.tag_name')

curl -fsSL \
"https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64" \
-o argocd

sudo install -o root -g root -m 0755 \
argocd /usr/local/bin/argocd

rm argocd

argocd version --client
```

The source lab also provides recovery guidance for installation problems involving Docker, kubectl, kind, and the Argo CD CLI.

---

# ☸️ Task 1.2 — Create the Kubernetes Cluster

Create a kind cluster:

```bash
kind create cluster --name argocd-lab
```

Verify:

```bash
kubectl cluster-info \
--context kind-argocd-lab
```

---

## Install Argo CD

Create the namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for the core Argo CD deployments:

```bash
kubectl wait deployment \
argocd-server \
argocd-repo-server \
argocd-applicationset-controller \
argocd-dex-server \
-n argocd \
--for=condition=Available \
--timeout=300s
```

Verify:

```bash
kubectl get pods -n argocd
```

The lab requires the Argo CD components to become available before continuing.

---

# 🔐 Authenticate with Argo CD

Expose the Argo CD API server:

```bash
kubectl port-forward \
svc/argocd-server \
-n argocd 8080:443 &
```

Retrieve the initial password:

```bash
ARGOCD_PASSWORD=$(kubectl -n argocd \
get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" \
| base64 --decode)
```

Login:

```bash
argocd login localhost:8080 \
--username admin \
--password "$ARGOCD_PASSWORD" \
--insecure
```

Verify:

```bash
argocd account get-user-info
```

Expected result:

```text
Logged In: true
Username: admin
```

---

# 🩺 Task 2 — Custom Health Checks

Argo CD normally evaluates resource health using built-in health logic. This lab extends that functionality with **custom Lua health checks**.

The scripts are stored in the `argocd-cm` ConfigMap and receive the live Kubernetes resource as `obj`.

Each health script must return:

```lua
hs = {}
hs.status = ""
hs.message = ""
return hs
```

Valid health states include:

```text
Healthy
Progressing
Degraded
Missing
Suspended
```

The custom health-check contract is defined by the lab source.

---

# 📦 Deployment Health

The custom `apps_Deployment` health check must implement:

### Healthy

When:

```text
readyReplicas == replicas
```

and:

```text
replicas > 0
```

### Degraded

When:

```text
readyReplicas == 0
```

and the Deployment has existed longer than:

```text
progressDeadlineSeconds
```

### Progressing

All other temporary states.

Every returned message must contain numeric replica information.

---

# 🌐 Service Health

The custom `v1_Service` health check uses different logic depending on the Service type.

### ClusterIP

Always:

```text
Healthy
```

### NodePort

Always:

```text
Healthy
```

### LoadBalancer

Initially:

```text
Progressing
```

until:

```text
status.loadBalancer.ingress
```

is populated.

Once ingress exists:

```text
Healthy
```

The health message should include the first available ingress IP or hostname.

---

# 🗂️ ConfigMap Health

The custom `v1_ConfigMap` health check is intentionally simple:

```text
Healthy
```

The health message must report the number of keys contained in:

```text
obj.data
```

---

# ⚙️ Register Lua Health Checks

Patch the Argo CD ConfigMap:

```bash
kubectl patch configmap argocd-cm \
-n argocd \
--type merge \
--patch '
{
  "data": {
    "resource.customizations.health.apps_Deployment": "REPLACE_WITH_YOUR_LUA_SCRIPT",
    "resource.customizations.health.v1_Service": "REPLACE_WITH_YOUR_LUA_SCRIPT",
    "resource.customizations.health.v1_ConfigMap": "REPLACE_WITH_YOUR_LUA_SCRIPT"
  }
}'
```

Restart the relevant Argo CD components:

```bash
kubectl rollout restart \
deployment/argocd-server \
deployment/argocd-application-controller \
-n argocd
```

Verify:

```bash
kubectl rollout status \
deployment/argocd-server \
-n argocd \
--timeout=120s

kubectl rollout status \
deployment/argocd-application-controller \
-n argocd \
--timeout=120s
```

Confirm the health customization keys exist:

```bash
kubectl get configmap argocd-cm \
-n argocd \
-o jsonpath='{.data}' \
| jq 'keys'
```

The expected keys are:

```text
resource.customizations.health.apps_Deployment
resource.customizations.health.v1_Service
resource.customizations.health.v1_ConfigMap
```

---

# 📚 Task 2.2 — Git-Backed Application

Create the application directory:

```bash
mkdir -p ~/argocd-health-demo/manifests

cd ~/argocd-health-demo
```

Initialize Git:

```bash
git init

git config user.name "Lab User"
git config user.email "lab@example.com"
```

The application must contain at least:

* A `Namespace` named `health-demo`
* A Deployment with two replicas
* `nginx:1.25`
* Liveness and readiness probes
* A ClusterIP Service
* A ConfigMap containing at least three keys

The Argo CD Application must use:

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

and:

```yaml
syncOptions:
- CreateNamespace=true
```

The repository must use the local `file://` scheme.

---

## Commit the Application

```bash
git add .

git commit -m "feat: initial application manifests"
```

Create the local bare repository:

```bash
git clone --bare \
~/argocd-health-demo \
~/argocd-health-demo.git
```

Register it with Argo CD:

```bash
argocd repo add \
"file://${HOME}/argocd-health-demo.git"
```

Verify:

```bash
argocd app list

argocd app get health-demo-app
```

Expected result:

```text
Health Status: Healthy
Sync Status: Synced
```

---

# 🔔 Task 3 — Notification Pipeline

Argo CD Notifications monitors Application state changes and can send notifications to external or internal services.

This lab creates a webhook pipeline:

```text
Argo CD Application
        │
        ▼
Notifications Controller
        │
        ▼
Webhook Service
        │
        ▼
Webhook Receiver
        │
        ▼
/tmp/webhook-alerts.log
```

---

# 📡 Webhook Receiver

The receiver must accept HTTP POST requests.

The required interface is:

```python
class WebhookReceiver:
    def handle_post(
        self,
        headers: dict,
        body: bytes
    ) -> tuple[int, str]:
        ...
```

The receiver must:

* Accept JSON requests
* Return HTTP `200` on success
* Append timestamped records to `/tmp/webhook-alerts.log`
* Run as a Kubernetes Deployment
* Use one replica
* Run in the `default` namespace
* Listen through the Kubernetes Service
* Be reachable at:

```text
http://webhook-receiver.default.svc.cluster.local:9000/notify
```

The lab intentionally does not require authentication because this is a controlled lab environment.

---

# 🔔 Install Argo CD Notifications

Install the Notifications controller:

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/notifications_install/install.yaml
```

Wait for it:

```bash
kubectl wait \
deployment/argocd-notifications-controller \
-n argocd \
--for=condition=Available \
--timeout=120s
```

Check:

```bash
kubectl get pods -n argocd
```

---

# 📨 Notification Configuration

Configure `argocd-notifications-cm` with:

### Template

Create:

```text
app-health-change
```

The payload should contain:

```text
app.metadata.name
app.status.health.status
app.status.sync.status
```

### Triggers

Configure:

```text
on-health-degraded
on-sync-succeeded
on-sync-failed
```

The required trigger behavior is:

| Trigger              | Condition            |
| -------------------- | -------------------- |
| `on-health-degraded` | Health is `Degraded` |
| `on-sync-succeeded`  | Sync is `Synced`     |
| `on-sync-failed`     | Sync is `OutOfSync`  |

The webhook service should point to:

```text
http://webhook-receiver.default.svc.cluster.local:9000/notify
```

---

# 🔗 Subscribe the Application

Apply all three notification subscriptions:

```bash
kubectl patch application health-demo-app \
-n argocd \
--type merge \
--patch '
{
  "metadata": {
    "annotations": {
      "notifications.argoproj.io/subscribe.on-health-degraded.webhook": "health-webhook",
      "notifications.argoproj.io/subscribe.on-sync-succeeded.webhook": "health-webhook",
      "notifications.argoproj.io/subscribe.on-sync-failed.webhook": "health-webhook"
    }
  }
}'
```

Check Notifications logs:

```bash
kubectl logs \
-n argocd \
deployment/argocd-notifications-controller \
--tail=20
```

There should be no `failed to send` errors after the application becomes synced.

---

# 💥 Failure Injection

The lab validates the monitoring pipeline using two deliberate failure scenarios.

## Failure Scenario A — Invalid Image

Change the Deployment image to:

```text
nginx:this-tag-does-not-exist
```

Commit and push the change, then synchronize the application.

The Deployment should enter:

```text
ImagePullBackOff
```

The custom health check should classify the application/resource as:

```text
Degraded
```

---

## Failure Scenario B — Replica Starvation

Restore the image and configure an excessive CPU request:

```text
cpu: 8000m
```

Because the node cannot satisfy the request, Pods should remain:

```text
Pending
```

The custom health check should classify this failure as:

```text
Degraded
```

The source lab defines both scenarios as required end-to-end validation cases.

---

# 🔍 Validate Failure Detection

Check Application health:

```bash
argocd app get health-demo-app \
--output json \
| jq '.status.health'
```

Check individual resource health:

```bash
kubectl get application health-demo-app \
-n argocd \
-o jsonpath='{range .status.resources[*]}{.kind}/{.name}: {.health.status} - {.health.message}{"\n"}{end}'
```

Check webhook notifications:

```bash
kubectl exec deployment/webhook-receiver \
-- cat /tmp/webhook-alerts.log \
| tail -10
```

The validation must demonstrate all three parts:

```text
Argo CD → Degraded
       ↓
Resource → Degraded
       ↓
Webhook → Notification logged
```

---

# ♻️ Recovery Validation

After both failure scenarios have been confirmed, restore the Deployment:

```text
image: nginx:1.25
cpu: 250m
```

Commit and push:

```bash
git add .

git commit -m "fix: restore healthy application"

git push
```

Synchronize the application:

```bash
argocd app sync health-demo-app
```

Verify:

```bash
argocd app get health-demo-app
```

Expected:

```text
Health Status: Healthy
```

Check the webhook log:

```bash
kubectl exec deployment/webhook-receiver \
-- cat /tmp/webhook-alerts.log
```

The expected log should contain at least:

```text
1 × Degraded — Failure Scenario A
1 × Degraded — Failure Scenario B
1 × Synced   — Recovery
```

This confirms that both degradation and recovery propagated through the notification pipeline.

---

# 📜 Code Contracts

## Lua Health Contract

All custom health scripts must return:

```lua
hs = {}
hs.status = ""
hs.message = ""
return hs
```

`hs.status` must contain one of:

```text
Healthy
Progressing
Degraded
Missing
Suspended
```

`hs.message` must be human-readable and contain numeric context where required.

---

## Notification Payload

The expected notification payload contains:

```python
class NotificationPayload:
    app_name: str
    health: str
    sync: str
    timestamp: str
```

The timestamp should use an ISO-8601 UTC representation.

---

## Webhook Receiver

The receiver follows:

```python
class WebhookReceiver:
    def handle_post(
        self,
        headers: dict,
        body: bytes
    ) -> tuple[int, str]:
        ...
```

The receiver must process the incoming JSON notification and record it before returning the HTTP response.

---

# 🔎 Verification Checklist

* [ ] Docker installed and operational
* [ ] kubectl installed
* [ ] kind installed
* [ ] Argo CD CLI installed
* [ ] kind cluster created
* [ ] Argo CD deployed
* [ ] Argo CD CLI authenticated
* [ ] Custom Deployment health check configured
* [ ] Custom Service health check configured
* [ ] Custom ConfigMap health check configured
* [ ] Argo CD components restarted successfully
* [ ] Git repository created
* [ ] Application manifests committed
* [ ] Application registered with Argo CD
* [ ] Application reports `Healthy`
* [ ] Application reports `Synced`
* [ ] Automated sync enabled
* [ ] Self-healing enabled
* [ ] Argo CD Notifications installed
* [ ] Webhook receiver deployed
* [ ] Notification template configured
* [ ] Notification triggers configured
* [ ] Application subscribed to triggers
* [ ] Invalid image failure detected
* [ ] Replica starvation failure detected
* [ ] `Degraded` notifications received
* [ ] Application recovered successfully
* [ ] `Synced` recovery notification received

---

# 🧪 Expected Outcomes

At the end of the lab:

1. Argo CD should display accurate resource-level health based on the custom Lua logic.
2. Deployment health should distinguish between `Healthy`, `Progressing`, and `Degraded`.
3. Service health should correctly handle ClusterIP, NodePort, and LoadBalancer behavior.
4. ConfigMap health should report its data-key count.
5. Git should act as the desired-state source.
6. Automated synchronization should reconcile application changes.
7. Deliberate failures should produce `Degraded` health.
8. Argo CD Notifications should send structured webhook payloads.
9. The webhook receiver should persist timestamped notifications.
10. Recovery should return the application to `Healthy` and generate a successful synchronization notification.

These expected outcomes are directly aligned with the source lab's validation requirements.

---

# 🛠️ Troubleshooting Guide

## Custom Health Check Is Not Applied

Check whether the customizations exist:

```bash
kubectl get configmap argocd-cm \
-n argocd \
-o jsonpath='{.data}' \
| jq 'keys'
```

Verify the Application Controller restarted:

```bash
kubectl rollout status \
deployment/argocd-application-controller \
-n argocd
```

Then distinguish between:

```text
Configuration loading problem
        vs.
Lua logic problem
```

The source lab specifically asks you to investigate how to confirm that `argocd-cm` was loaded after the controller restart.

---

## Webhook Connection Refused

If the receiver is running but Notifications cannot connect, investigate in this order:

```text
1. DNS resolution
        ↓
2. Service existence
        ↓
3. Service selector
        ↓
4. Pod IP
        ↓
5. Container listening port
        ↓
6. Receiver process binding
```

Useful commands include:

```bash
kubectl get svc webhook-receiver

kubectl get endpoints webhook-receiver

kubectl get pods -l app=webhook-receiver

kubectl logs deployment/webhook-receiver
```

The lab specifically identifies DNS resolution, Service selector problems, and receiver binding as the main diagnostic areas.

---

# 🧠 Key Concepts

### Custom Health Checks

Custom Lua scripts allow Argo CD to evaluate application resources using application-specific health rules rather than relying exclusively on built-in behavior.

### GitOps

Git stores the desired application configuration while Argo CD continuously reconciles the Kubernetes cluster against that desired state.

### Health Monitoring

Health monitoring provides resource-level visibility into whether an application is healthy, progressing, or degraded.

### Notifications

Argo CD Notifications converts application state changes into actionable events that can be delivered through webhooks.

### Failure Injection

Intentional failures provide a controlled way to prove that monitoring and alerting actually detect real application problems.

### Recovery Propagation

A successful recovery demonstrates that the same automated pipeline can communicate both degradation and restoration.

---

# 🏁 Conclusion

This lab demonstrates an end-to-end **Argo CD application health monitoring and notification workflow**.

Custom Lua health checks extend Argo CD's built-in health evaluation and allow resource-specific correctness criteria to be defined for Deployments, Services, and ConfigMaps.

The Argo CD Notifications controller then transforms application state changes into structured webhook events. Deliberate failures validate that the monitoring system detects actual runtime degradation, while the recovery test confirms that the application can return to a healthy state and communicate that recovery automatically.

Together, these capabilities provide a strong foundation for **GitOps observability**, where automated health feedback reduces the need for continuous manual Kubernetes inspection.

---

## ⭐ Skills Demonstrated

```text
Kubernetes
    │
    ├── kind
    ├── Deployments
    ├── Services
    └── ConfigMaps
          │
          ▼
       Argo CD
          │
          ├── GitOps
          ├── Custom Health Checks
          ├── Lua
          └── Application Reconciliation
                    │
                    ▼
          Argo CD Notifications
                    │
                    ▼
             Webhook Pipeline
                    │
                    ▼
             Health Monitoring
                    │
                    ▼
        Failure Detection & Recovery
```

**Core takeaway:** this lab shows how Argo CD can evolve from a GitOps deployment controller into an application-aware monitoring and notification platform through **custom health evaluation and automated event delivery**.
