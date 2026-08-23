<div align="center">

# 🔗 Argo CD and CI/CD Integration

![Argo CD](https://img.shields.io/badge/Argo_CD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![GitOps](https://img.shields.io/badge/GitOps-000000?style=for-the-badge&logo=git&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)

**Build a complete Jenkins → Argo CD GitOps pipeline, from CI build to automated CD sync**

</div>

---

## 📖 Table of Contents

- [🎯 Lab Objectives](#-lab-objectives)
- [📋 Prerequisites](#-prerequisites)
- [🖥️ Lab Environment](#️-lab-environment)
- [🧰 Task 1: Environment Setup and Tool Installation](#-task-1-environment-setup-and-tool-installation)
- [🚀 Task 2: Install and Configure Argo CD](#-task-2-install-and-configure-argo-cd)
- [⚙️ Task 3: Set up Jenkins CI Pipeline](#️-task-3-set-up-jenkins-ci-pipeline)
- [🔗 Task 4: Integrate Argo CD with CI/CD Pipeline](#-task-4-integrate-argo-cd-with-cicd-pipeline)
- [📊 Task 5: Advanced CI/CD Integration Features](#-task-5-advanced-cicd-integration-features)
- [🔐 Task 6: Security and Best Practices](#-task-6-security-and-best-practices)
- [🛠️ Troubleshooting Guide](#️-troubleshooting-guide)
- [🛡️ MITRE ATT&CK Mapping](#️-mitre-attck-mapping)
- [📚 Key Concepts](#-key-concepts)
- [✅ Conclusion](#-conclusion)

---

## 🎯 Lab Objectives

By the end of this lab, students will be able to:

| # | Objective |
|---|-----------|
| 1 | Install and configure Argo CD on a Kubernetes cluster |
| 2 | Set up a Jenkins CI/CD pipeline for automated builds |
| 3 | Integrate Jenkins with Argo CD for GitOps-based deployments |
| 4 | Create a complete CI/CD workflow that automatically deploys applications using Argo CD |
| 5 | Understand the GitOps methodology and its benefits |
| 6 | Configure automated synchronization between Git repositories and Kubernetes deployments |
| 7 | Implement best practices for CI/CD pipeline security and efficiency |

## 📋 Prerequisites

Before starting this lab, students should have:

| # | Requirement |
|---|-------------|
| 1 | Basic understanding of Kubernetes concepts (pods, services, deployments) |
| 2 | Familiarity with Git version control system |
| 3 | Basic knowledge of Docker containers |
| 4 | Understanding of YAML configuration files |
| 5 | Experience with Linux command line operations |
| 6 | Knowledge of CI/CD pipeline concepts |

## 🖥️ Lab Environment

> Al Nafi provides Linux-based cloud machines for this lab. Simply click **Start Lab** to access your dedicated environment. The provided Linux machine is bare metal with no pre-installed tools — you will install all required components during the lab exercises.

---

## 🧰 Task 1: Environment Setup and Tool Installation

### 🐳 Subtask 1.1: Install Docker

```bash
# 🔄 Update the system packages
sudo apt update

# 📦 Install required packages
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# 🔑 Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 📥 Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 🔄 Update package index
sudo apt update

# 🐳 Install Docker
sudo apt install -y docker-ce docker-ce-cli containerd.io

# 👤 Add current user to docker group
sudo usermod -aG docker $USER

# ▶️ Start and enable Docker service
sudo systemctl start docker
sudo systemctl enable docker

# ✅ Verify Docker installation
docker --version
```

### ☸️ Subtask 1.2: Install kubectl

```bash
# ⬇️ Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# 🔓 Make kubectl executable
chmod +x kubectl

# 📂 Move kubectl to system path
sudo mv kubectl /usr/local/bin/

# ✅ Verify kubectl installation
kubectl version --client
```

### 🎡 Subtask 1.3: Install Kind (Kubernetes in Docker)

```bash
# ⬇️ Download Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

# 🔓 Make Kind executable
chmod +x ./kind

# 📂 Move Kind to system path
sudo mv ./kind /usr/local/bin/kind

# ✅ Verify Kind installation
kind version
```

### 🏗️ Subtask 1.4: Create Kubernetes Cluster

```bash
# 📝 Create a Kind cluster configuration file
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
  - containerPort: 30080
    hostPort: 30080
    protocol: TCP
  - containerPort: 30443
    hostPort: 30443
    protocol: TCP
EOF

# 🚀 Create the cluster
kind create cluster --config=kind-config.yaml --name=argocd-lab

# ✅ Verify cluster is running
kubectl cluster-info --context kind-argocd-lab
kubectl get nodes
```

### 🌱 Subtask 1.5: Install Git

```bash
# 📦 Install Git
sudo apt install -y git

# ⚙️ Configure Git (replace with your information)
git config --global user.name "Lab User"
git config --global user.email "labuser@example.com"

# ✅ Verify Git installation
git --version
```

---

## 🚀 Task 2: Install and Configure Argo CD

### 🚀 Subtask 2.1: Install Argo CD

```bash
# 📁 Create argocd namespace
kubectl create namespace argocd

# 📥 Install Argo CD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ⏳ Wait for Argo CD pods to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# ✅ Verify Argo CD installation
kubectl get pods -n argocd
```

### 🌐 Subtask 2.2: Access Argo CD UI

```bash
# 🌐 Patch the Argo CD server service to use NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":8080,"nodePort":30080}]}}'

# 🔑 Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d > argocd-password.txt

# 📋 Display the password
echo "Argo CD Admin Password:"
cat argocd-password.txt
echo ""

# 🖥️ Display access information
echo "Argo CD is accessible at: http://localhost:30080"
echo "Username: admin"
echo "Password: $(cat argocd-password.txt)"
```

### 💻 Subtask 2.3: Install Argo CD CLI

```bash
# ⬇️ Download Argo CD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# 🔓 Make it executable
chmod +x argocd-linux-amd64

# 📂 Move to system path
sudo mv argocd-linux-amd64 /usr/local/bin/argocd

# ✅ Verify installation
argocd version --client

# 🔐 Login to Argo CD
argocd login localhost:30080 --username admin --password $(cat argocd-password.txt) --insecure
```

---

## ⚙️ Task 3: Set up Jenkins CI Pipeline

### ⚙️ Subtask 3.1: Install Jenkins

```bash
# ☕ Install Java (required for Jenkins)
sudo apt install -y openjdk-11-jdk

# 🔑 Add Jenkins repository key
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -

# 📥 Add Jenkins repository
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

# 🔄 Update package index
sudo apt update

# ⚙️ Install Jenkins
sudo apt install -y jenkins

# ▶️ Start and enable Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# 🔑 Get Jenkins initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword > jenkins-password.txt

echo "Jenkins is accessible at: http://localhost:8080"
echo "Initial admin password: $(cat jenkins-password.txt)"
```

### 🔧 Subtask 3.2: Configure Jenkins

```bash
# ⏳ Wait for Jenkins to start
sleep 30

# ⬇️ Install Jenkins CLI
wget http://localhost:8080/jnlpJars/jenkins-cli.jar

# 📝 Create a script to install plugins
cat << 'EOF' > install-plugins.sh
#!/bin/bash
JENKINS_URL="http://localhost:8080"
JENKINS_USER="admin"
JENKINS_PASSWORD=$(cat jenkins-password.txt)

# 🔌 Install required plugins
java -jar jenkins-cli.jar -s $JENKINS_URL -auth $JENKINS_USER:$JENKINS_PASSWORD install-plugin git
java -jar jenkins-cli.jar -s $JENKINS_URL -auth $JENKINS_USER:$JENKINS_PASSWORD install-plugin workflow-aggregator
java -jar jenkins-cli.jar -s $JENKINS_URL -auth $JENKINS_USER:$JENKINS_PASSWORD install-plugin docker-workflow
java -jar jenkins-cli.jar -s $JENKINS_URL -auth $JENKINS_USER:$JENKINS_PASSWORD install-plugin kubernetes

# 🔁 Restart Jenkins
java -jar jenkins-cli.jar -s $JENKINS_URL -auth $JENKINS_USER:$JENKINS_PASSWORD restart
EOF

chmod +x install-plugins.sh
```

### 📁 Subtask 3.3: Create Sample Application Repository

```bash
# 📂 Create application directory
mkdir -p ~/sample-app
cd ~/sample-app

# 🌱 Initialize Git repository
git init
```

```python
# app.py — 🐍 simple Flask web application
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    version = os.environ.get('APP_VERSION', 'v1.0.0')
    return f'<h1>Hello from Sample App!</h1><p>Version: {version}</p>'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```text
# requirements.txt
Flask==2.3.3
```

```dockerfile
# 🐳 Dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

```yaml
# k8s/deployment.yaml — ☸️ deployment manifest
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
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
        image: sample-app:latest
        ports:
        - containerPort: 5000
        env:
        - name: APP_VERSION
          value: "v1.0.0"
```

```yaml
# k8s/service.yaml — 🌐 service manifest
apiVersion: v1
kind: Service
metadata:
  name: sample-app-service
  namespace: default
spec:
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 5000
    nodePort: 30081
  type: NodePort
```

```groovy
// Jenkinsfile — 🛠️ CI pipeline definition
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'sample-app'
        DOCKER_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} ."
                    sh "docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} ${DOCKER_IMAGE}:latest"
                }
            }
        }
        
        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    sh """
                        sed -i 's|image: sample-app:.*|image: sample-app:${DOCKER_TAG}|g' k8s/deployment.yaml
                        sed -i 's|value: "v.*"|value: "v1.0.${BUILD_NUMBER}"|g' k8s/deployment.yaml
                    """
                }
            }
        }
        
        stage('Commit Updated Manifests') {
            steps {
                script {
                    sh """
                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"
                        git add k8s/deployment.yaml
                        git commit -m "Update image to ${DOCKER_TAG}" || true
                    """
                }
            }
        }
        
        stage('Load Image to Kind') {
            steps {
                script {
                    sh "kind load docker-image ${DOCKER_IMAGE}:${DOCKER_TAG} --name argocd-lab"
                    sh "kind load docker-image ${DOCKER_IMAGE}:latest --name argocd-lab"
                }
            }
        }
    }
}
```

```bash
# ➕ Add all files to Git
git add .
git commit -m "Initial commit with sample application"

echo "Sample application repository created at: $(pwd)"
```

---

## 🔗 Task 4: Integrate Argo CD with CI/CD Pipeline

### ➕ Subtask 4.1: Create Argo CD Application

```bash
# 📝 Create Argo CD application manifest
cat << EOF > argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: file://$(pwd)
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF

# ✅ Apply the Argo CD application
kubectl apply -f argocd-app.yaml

# 🔍 Verify the application is created
argocd app list
```

### 📦 Subtask 4.2: Configure Git Repository for Argo CD

Since we're working on a single machine, we'll use a local Git repository that Argo CD can access.

```bash
# 📦 Create a bare Git repository that can be accessed by Argo CD
cd ~
git clone --bare ~/sample-app sample-app.git

# 🔄 Update the Argo CD application to use the bare repository
cd ~/sample-app
cat << EOF > argocd-app-updated.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: file:///home/$(whoami)/sample-app.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF

# ✅ Update the application
kubectl apply -f argocd-app-updated.yaml

# 🔄 Sync the application
argocd app sync sample-app
```

### 🛠️ Subtask 4.3: Create Jenkins Pipeline Job

```xml
<!-- jenkins-job-config.xml -->
<?xml version='1.1' encoding='UTF-8'?>
<flow-definition plugin="workflow-job@2.40">
  <actions/>
  <description>Sample App CI/CD Pipeline</description>
  <keepDependencies>false</keepDependencies>
  <properties>
    <org.jenkinsci.plugins.workflow.job.properties.PipelineTriggersJobProperty>
      <triggers/>
    </org.jenkinsci.plugins.workflow.job.properties.PipelineTriggersJobProperty>
  </properties>
  <definition class="org.jenkinsci.plugins.workflow.cps.CpsScmFlowDefinition" plugin="workflow-cps@2.87">
    <scm class="hudson.plugins.git.GitSCM" plugin="git@4.8.3">
      <configVersion>2</configVersion>
      <userRemoteConfigs>
        <hudson.plugins.git.UserRemoteConfig>
          <url>file:///home/USER_HOME/sample-app</url>
        </hudson.plugins.git.UserRemoteConfig>
      </userRemoteConfigs>
      <branches>
        <hudson.plugins.git.BranchSpec>
          <name>*/main</name>
        </hudson.plugins.git.BranchSpec>
      </branches>
      <doGenerateSubmoduleConfigurations>false</doGenerateSubmoduleConfigurations>
      <submoduleCfg class="list"/>
      <extensions/>
    </scm>
    <scriptPath>Jenkinsfile</scriptPath>
    <lightweight>true</lightweight>
  </definition>
  <triggers/>
  <disabled>false</disabled>
</flow-definition>
```

```bash
# 🔄 Replace USER_HOME placeholder
sed -i "s|USER_HOME|$(whoami)|g" jenkins-job-config.xml

echo "Jenkins job configuration created. You can now create the job through the Jenkins UI."
echo "Job configuration file: jenkins-job-config.xml"
```

### 🧪 Subtask 4.4: Test the Complete CI/CD Pipeline

```bash
# ✏️ Make a change to the application
cd ~/sample-app
sed -i 's/Hello from Sample App!/Hello from Updated Sample App!/g' app.py

# ➕ Commit the change
git add app.py
git commit -m "Update application message"

# ⬆️ Push to bare repository
git push ~/sample-app.git main

# 🐳 Build the Docker image manually to test
docker build -t sample-app:v1.0.1 .
docker tag sample-app:v1.0.1 sample-app:latest

# 📤 Load image into Kind cluster
kind load docker-image sample-app:v1.0.1 --name argocd-lab
kind load docker-image sample-app:latest --name argocd-lab

# 🔄 Update the deployment manifest
sed -i 's|image: sample-app:.*|image: sample-app:v1.0.1|g' k8s/deployment.yaml
sed -i 's|value: "v.*"|value: "v1.0.1"|g' k8s/deployment.yaml

# ➕ Commit the updated manifest
git add k8s/deployment.yaml
git commit -m "Update image to v1.0.1"
git push ~/sample-app.git main

# 🔄 Trigger Argo CD sync
argocd app sync sample-app

# ✅ Check application status
argocd app get sample-app
kubectl get pods -l app=sample-app
```

### ✅ Subtask 4.5: Verify Deployment

```bash
# 🔍 Check if pods are running
kubectl get pods -l app=sample-app

# 🔍 Check service
kubectl get svc sample-app-service

# 🧪 Test the application
curl http://localhost:30081

# 📊 Check Argo CD application status
argocd app get sample-app

# 🌐 View application in Argo CD UI
echo "Access Argo CD UI at: http://localhost:30080"
echo "Username: admin"
echo "Password: $(cat ~/argocd-password.txt)"
```

---

## 📊 Task 5: Advanced CI/CD Integration Features

### 🔔 Subtask 5.1: Implement Webhook Integration

Create a simple webhook receiver to trigger Jenkins builds automatically.

```python
#!/usr/bin/env python3
# webhook-receiver.py — 🔔 triggers a Jenkins build on incoming webhook
from flask import Flask, request, jsonify
import subprocess
import os

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    try:
        # 🚀 Trigger Jenkins build
        result = subprocess.run([
            'curl', '-X', 'POST',
            'http://localhost:8080/job/sample-app-pipeline/build',
            '--user', f'admin:{os.environ.get("JENKINS_PASSWORD", "")}'
        ], capture_output=True, text=True)
        
        return jsonify({'status': 'success', 'message': 'Build triggered'})
    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=9000)
```

```bash
chmod +x webhook-receiver.py

echo "Webhook receiver created. Run with: python3 webhook-receiver.py"
```

### 📊 Subtask 5.2: Create Monitoring Dashboard

Set up basic monitoring for the CI/CD pipeline.

```bash
cat << 'EOF' > monitor-pipeline.sh
#!/bin/bash

echo "=== CI/CD Pipeline Status ==="
echo ""

echo "1. Jenkins Status:"
systemctl is-active jenkins
echo ""

echo "2. Argo CD Status:"
kubectl get pods -n argocd | grep -E "(server|controller|repo-server)"
echo ""

echo "3. Sample Application Status:"
kubectl get pods -l app=sample-app
echo ""

echo "4. Argo CD Application Status:"
argocd app get sample-app --output json | jq -r '.status.sync.status, .status.health.status'
echo ""

echo "5. Recent Jenkins Builds:"
if [ -f jenkins-password.txt ]; then
    curl -s -u admin:$(cat jenkins-password.txt) http://localhost:8080/job/sample-app-pipeline/api/json | jq -r '.builds[0:3][] | "\(.number): \(.result // "RUNNING")"'
fi
echo ""

echo "6. Application Accessibility:"
curl -s -o /dev/null -w "Sample App HTTP Status: %{http_code}\n" http://localhost:30081
EOF

chmod +x monitor-pipeline.sh

echo "Monitoring script created. Run with: ./monitor-pipeline.sh"
```

### ⏪ Subtask 5.3: Implement Rollback Strategy

Create a rollback mechanism for failed deployments.

```bash
cat << 'EOF' > rollback-deployment.sh
#!/bin/bash

DEPLOYMENT_NAME="sample-app"
NAMESPACE="default"

echo "Current deployment status:"
kubectl get deployment $DEPLOYMENT_NAME -n $NAMESPACE

echo ""
echo "Rolling back to previous version..."
kubectl rollout undo deployment/$DEPLOYMENT_NAME -n $NAMESPACE

echo ""
echo "Waiting for rollback to complete..."
kubectl rollout status deployment/$DEPLOYMENT_NAME -n $NAMESPACE

echo ""
echo "Rollback completed. Current status:"
kubectl get pods -l app=$DEPLOYMENT_NAME -n $NAMESPACE

# 🔄 Sync Argo CD to reflect the rollback
echo ""
echo "Syncing Argo CD application..."
argocd app sync sample-app
EOF

chmod +x rollback-deployment.sh

echo "Rollback script created. Use with: ./rollback-deployment.sh"
```

---

## 🔐 Task 6: Security and Best Practices

### 🔐 Subtask 6.1: Implement RBAC for Argo CD

```bash
# 📝 Create RBAC configuration
cat << 'EOF' > argocd-rbac.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    p, role:admin, applications, *, */*, allow
    p, role:admin, clusters, *, *, allow
    p, role:admin, repositories, *, *, allow
    g, admin, role:admin
EOF

kubectl apply -f argocd-rbac.yaml

# 🔁 Restart Argo CD server to apply RBAC
kubectl rollout restart deployment/argocd-server -n argocd
```

### 🛡️ Subtask 6.2: Secure Jenkins Configuration

```bash
cat << 'EOF' > secure-jenkins.sh
#!/bin/bash

echo "Implementing Jenkins security best practices..."

# 🚫 Disable Jenkins CLI over remoting
echo "jenkins.CLI.get().setEnabled(false)" | sudo tee -a /var/lib/jenkins/init.groovy.d/disable-cli.groovy

# 🛡️ Set up CSRF protection
cat << 'GROOVY' | sudo tee /var/lib/jenkins/init.groovy.d/csrf-protection.groovy
import hudson.security.csrf.DefaultCrumbIssuer
import jenkins.model.Jenkins

def instance = Jenkins.instance
instance.setCrumbIssuer(new DefaultCrumbIssuer(true))
instance.save()
GROOVY

# 🔁 Restart Jenkins to apply security settings
sudo systemctl restart jenkins

echo "Jenkins security configuration applied."
EOF

chmod +x secure-jenkins.sh
./secure-jenkins.sh
```

### 🔑 Subtask 6.3: Implement Secret Management

```bash
# 🔑 Create Kubernetes secrets for the application
kubectl create secret generic app-secrets \
  --from-literal=database-url="postgresql://localhost:5432/myapp" \
  --from-literal=api-key="your-secret-api-key"
```

```yaml
# k8s/deployment-with-secrets.yaml — 🔑 deployment updated to consume secrets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
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
        image: sample-app:latest
        ports:
        - containerPort: 5000
        env:
        - name: APP_VERSION
          value: "v1.0.0"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
```

```bash
echo "Secret management configuration created."
```

---

## 🛠️ Troubleshooting Guide

<details>
<summary><strong>Click to expand common issues and solutions</strong></summary>

#### Issue 1: Argo CD pods not starting

```bash
# 🔍 Check pod status and logs
kubectl get pods -n argocd
kubectl logs -n argocd deployment/argocd-server

# ✅ Solution: Ensure sufficient resources
kubectl describe nodes
```

#### Issue 2: Jenkins build failures

```bash
# 🔍 Check Jenkins logs
sudo journalctl -u jenkins -f

# ✅ Verify Docker permissions
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

#### Issue 3: Application not accessible

```bash
# 🔍 Check service and endpoints
kubectl get svc sample-app-service
kubectl get endpoints sample-app-service

# ✅ Verify NodePort configuration
kubectl get svc sample-app-service -o yaml
```

#### Issue 4: Argo CD sync failures

```bash
# 🔍 Check application status
argocd app get sample-app

# 🔄 Force sync
argocd app sync sample-app --force

# 🔍 Check repository connectivity
argocd repo list
```

</details>

### ⚡ Subtask 6.4: Optimize Pipeline Performance

```bash
# 📝 Create pipeline optimization script
cat << 'EOF' > optimize-pipeline.sh
#!/bin/bash

echo "Optimizing CI/CD pipeline performance..."

# 🐳 Enable Docker layer caching
cat << 'DOCKERFILE' > Dockerfile.optimized
FROM python:3.9-slim

# Install dependencies first (better caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

WORKDIR /app
COPY app.py .

EXPOSE 5000
CMD ["python", "app.py"]
DOCKERFILE

# ⚙️ Configure Jenkins for parallel builds
echo "Configuring Jenkins for optimal performance..."

# ⚡ Set up Argo CD for faster sync
kubectl patch configmap argocd-cm -n argocd --patch '{"data":{"timeout.reconciliation":"10s"}}'

echo "Pipeline optimization completed."
EOF

chmod +x optimize-pipeline.sh
```

---

## 🛡️ MITRE ATT&CK Mapping

| Technique ID | Technique Name | Relevance to This Lab |
|---|---|---|
| T1195.002 | Compromise Software Supply Chain | The Jenkinsfile automatically builds and tags container images from Git commits; an unsecured pipeline step here is a supply-chain injection point |
| T1610 | Deploy Container | Jenkins loads built images into the cluster and Argo CD auto-syncs them into running Deployments, automating container deployment end-to-end |
| T1078 | Valid Accounts | Argo CD RBAC differentiates `role:developer` from `role:admin` identities used to operate the pipeline and cluster |
| T1098 | Account Manipulation | `argocd-rbac-cm` policy.csv and group-to-role bindings (`g, admin, role:admin`) govern how permissions are granted across accounts |
| T1552.001 | Unsecured Credentials: Credentials in Files | Jenkins and Argo CD admin passwords are written to plaintext files (`jenkins-password.txt`, `argocd-password.txt`), and an `api-key` is passed as a literal Secret value |

---

## 📚 Key Concepts

| Concept | Description |
|---------|-------------|
| **GitOps** | A deployment methodology where Git is the single source of truth and Kubernetes state is continuously reconciled to match it |
| **CI vs. CD** | Continuous Integration (Jenkins: build, tag, push images) is distinct from Continuous Delivery (Argo CD: detect Git changes and sync cluster state) |
| **Jenkinsfile** | A declarative pipeline-as-code definition describing build stages, checked into the same repository as the application |
| **kind (Kubernetes in Docker)** | Runs a full Kubernetes cluster inside Docker containers, used here as the lab's local cluster |
| **Argo CD Application CRD** | A custom resource that tells Argo CD which Git repo/path to watch and which cluster/namespace to deploy it to |
| **Sync Policy** | Argo CD's `automated` block (`prune`, `selfHeal`) controls whether drift is auto-corrected and whether removed resources are pruned |
| **RBAC policy.csv** | Argo CD's role-based access control format mapping subjects/groups to permitted actions on applications, clusters, and repositories |
| **Kubernetes Secret** | An API object for storing sensitive values (credentials, API keys) separately from Pod specs, consumed via `secretKeyRef` |
| **Webhook** | An HTTP callback that lets an external event (e.g., a Git push) automatically trigger a Jenkins build |

---

## ✅ Conclusion

In this comprehensive lab, you have successfully:

- ✅ Set up a complete CI/CD environment using Jenkins and Argo CD on a single Linux machine
- ✅ Implemented GitOps methodology by integrating Argo CD with your application deployment process
- ✅ Created an automated pipeline that builds, tests, and deploys applications using modern DevOps practices
- ✅ Configured monitoring and security features to ensure reliable and secure deployments
- ✅ Learned troubleshooting techniques for common CI/CD pipeline issues

### 🏆 Key Achievements

- **Jenkins Integration:** Successfully configured Jenkins to build Docker images and update Kubernetes manifests automatically
- **Argo CD Deployment:** Implemented GitOps-based deployment using Argo CD for automated synchronization
- **End-to-End Automation:** Created a complete workflow from code commit to production deployment
- **Security Implementation:** Applied RBAC, secret management, and security best practices
- **Monitoring Setup:** Established monitoring and alerting for pipeline health

### 🌍 Why This Matters

This lab demonstrates real-world DevOps practices that are essential in modern software development. The GitOps approach with Argo CD provides:

- **Declarative Configuration:** Infrastructure and applications defined as code
- **Automated Synchronization:** Continuous deployment based on Git repository state
- **Audit Trail:** Complete history of all changes and deployments
- **Rollback Capabilities:** Easy recovery from failed deployments
- **Security:** Role-based access control and secret management

The skills learned in this lab are directly applicable to enterprise environments where reliable, automated, and secure deployment pipelines are crucial for business success. You now have hands-on experience with industry-standard tools and practices that will serve you well in professional DevOps roles.

### 🔭 Next Steps

Consider exploring advanced topics such as:

- Multi-cluster deployments with Argo CD
- Advanced Jenkins pipeline features
- Integration with cloud providers
- Implementing comprehensive monitoring with Prometheus and Grafana
- Setting up disaster recovery procedures

---

<div align="center">

![Al Nafi](https://img.shields.io/badge/Al_Nafi-Cybersecurity_Training-blue?style=for-the-badge)

</div>
