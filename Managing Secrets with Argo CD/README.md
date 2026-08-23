# Managing Secrets with Argo CD

A hands-on Kubernetes and GitOps lab demonstrating secure application secret management using **HashiCorp Vault**, **External Secrets Operator (ESO)**, and **Argo CD**.

The lab focuses on a core GitOps security principle: **Git stores configuration and intent, while sensitive secret values remain outside Git in a dedicated secrets-management system.**

## Objectives

By completing this lab, you will be able to:

* Design and deploy a GitOps-managed Kubernetes application without storing sensitive credentials in Git.
* Deploy and configure HashiCorp Vault as the external source of truth for application secrets.
* Install and configure External Secrets Operator (ESO).
* Synchronize secrets from Vault into native Kubernetes `Secret` resources.
* Configure Vault Kubernetes authentication using a dedicated Kubernetes `ServiceAccount`.
* Manage the application lifecycle through Argo CD.
* Configure automated synchronization and self-healing in Argo CD.
* Validate secret provisioning, rotation, synchronization, and pod-level consumption.
* Demonstrate a complete external secret lifecycle from Vault to a running Kubernetes workload.

---

## Architecture

The lab implements the following secret-management flow:

```text
                    ┌─────────────────────┐
                    │   Local Git Repo    │
                    │                     │
                    │ Kubernetes intent   │
                    │ Argo CD manifests   │
                    └──────────┬──────────┘
                               │
                               │ GitOps
                               ▼
                    ┌─────────────────────┐
                    │      Argo CD        │
                    │                     │
                    │ Automated Sync      │
                    │ Self-Heal           │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Kubernetes / kind   │
                    │                     │
                    │ demo-app namespace  │
                    └──────────┬──────────┘
                               │
                               │ SecretStore /
                               │ ExternalSecret
                               ▼
                    ┌─────────────────────┐
                    │ External Secrets    │
                    │ Operator (ESO)      │
                    └──────────┬──────────┘
                               │
                               │ Kubernetes Auth
                               ▼
                    ┌─────────────────────┐
                    │   HashiCorp Vault   │
                    │                     │
                    │ secret/demo-app/    │
                    │ ├── database        │
                    │ ├── api-key         │
                    │ └── jwt-signing-key │
                    └──────────┬──────────┘
                               │
                               │ Synchronization
                               ▼
                    ┌─────────────────────┐
                    │ Kubernetes Secrets  │
                    │                     │
                    │ Database Secret     │
                    │ API Secret          │
                    │ JWT Secret          │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │    Demo Application │
                    │                     │
                    │ Environment Vars    │
                    │ /etc/secrets/jwt    │
                    └─────────────────────┘
```

### Secret Flow

```text
Vault
  │
  │ Secret values
  ▼
External Secrets Operator
  │
  │ refreshInterval: 60s
  ▼
Kubernetes Secrets
  │
  ▼
Deployment
  │
  ├── Environment Variables
  │
  └── /etc/secrets/jwt
```

Argo CD manages the Kubernetes resources and application configuration, while Vault remains the authoritative source for sensitive values.

---

## Core Security Principle

The most important principle demonstrated in this lab is:

> **Git should contain secret references and deployment intent, not secret values.**

For example, the repository can contain an `ExternalSecret` definition:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: demo-api-secret
  namespace: demo-app
spec:
  refreshInterval: 60s
  secretStoreRef:
    name: vault-store
    kind: SecretStore
  target:
    name: demo-api-secret
  data:
    - secretKey: API_KEY
      remoteRef:
        key: secret/demo-app/api
        property: api_key
```

The actual API key is stored only in Vault.

This prevents sensitive credentials from becoming part of:

* Git history
* Pull requests
* GitHub/GitLab repositories
* CI/CD logs
* Configuration backups
* Developer workstations

---

# Prerequisites

Before beginning the lab, you should have:

* Basic Linux command-line knowledge.
* Experience editing files from the terminal.
* Familiarity with Kubernetes.
* Understanding of:

  * Deployments
  * Services
  * Namespaces
  * Secrets
  * ServiceAccounts
* Basic Git knowledge.
* Access to an AWS EC2 Ubuntu instance provided by Al Nafi.
* Internet connectivity from the EC2 instance.

---

# Lab Environment

The lab uses a dedicated Ubuntu EC2 instance.

The Kubernetes environment is created using **kind**, allowing all required components to run locally inside Docker.

The environment contains:

```text
Ubuntu EC2
│
├── Docker
│
├── kind Kubernetes Cluster
│   │
│   ├── Argo CD
│   ├── HashiCorp Vault
│   ├── External Secrets Operator
│   └── Demo Application
│
├── kubectl
├── Helm
├── Argo CD CLI
└── Vault CLI
```

---

# Task 1: Install and Verify the Full Toolchain

## 1.1 Install Required Tools

The lab requires:

* Docker
* kubectl
* kind
* Argo CD CLI
* Vault CLI
* Helm

Each tool should be installed through its official distribution channel.

After installation, verify every tool.

### Docker

```bash
docker version
```

### kubectl

```bash
kubectl version --client
```

### kind

```bash
kind version
```

### Argo CD CLI

```bash
argocd version --client
```

### Vault CLI

```bash
vault version
```

### Helm

```bash
helm version
```

Every command must exit successfully and display a recognizable version.

---

## 1.2 Create the kind Cluster

Create a single-node Kubernetes cluster:

```bash
kind create cluster --name argocd-vault-lab
```

Verify the cluster:

```bash
kind get clusters
```

Verify Kubernetes access:

```bash
kubectl cluster-info
```

Verify the node:

```bash
kubectl get nodes
```

Expected result:

```text
NAME                         STATUS   ROLES           AGE
argocd-vault-lab-control-plane   Ready    control-plane   ...
```

---

# 1.3 Deploy Argo CD

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Deploy Argo CD using the official installation manifest:

```bash
kubectl apply \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Monitor the deployment:

```bash
kubectl get pods -n argocd -w
```

Verify all pods:

```bash
kubectl get pods -n argocd
```

All Argo CD pods should eventually report:

```text
Running
```

with all containers ready.

---

## Argo CD Troubleshooting

### Pods stuck in `Pending`

Inspect the affected pod:

```bash
kubectl describe pod -n argocd <pod-name>
```

Review the events at the bottom of the output.

### Pods stuck in `ImagePullBackOff`

Check the pod:

```bash
kubectl describe pod -n argocd <pod-name>
```

Verify that the kind node has internet connectivity:

```bash
docker exec <kind-container> curl -fsSL https://quay.io
```

### Official Documentation

* Docker Ubuntu installation: https://docs.docker.com/engine/install/ubuntu/
* kind quick start: https://kind.sigs.k8s.io/docs/user/quick-start/
* Argo CD getting started: https://argo-cd.readthedocs.io/en/stable/getting_started/

---

# Task 2: Deploy and Configure HashiCorp Vault

## 2.1 Add the HashiCorp Helm Repository

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Update the repository:

```bash
helm repo update
```

Check available Vault versions:

```bash
helm search repo hashicorp/vault --versions
```

---

## 2.2 Install Vault

Create a dedicated namespace:

```bash
kubectl create namespace vault
```

Install Vault using the official Helm chart.

For a lab environment, Vault can be deployed in development mode.

Example:

```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true"
```

Verify the deployment:

```bash
kubectl get pods -n vault
```

Check the service:

```bash
kubectl get svc -n vault
```

---

## 2.3 Configure the Vault CLI

Set the Vault address:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```

Forward the Vault service:

```bash
kubectl port-forward -n vault svc/vault 8200:8200
```

In another terminal:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```

Check Vault:

```bash
vault status
```

The Vault instance must report:

```text
Initialized: true
Sealed: false
```

---

## 2.4 Enable the KV-v2 Secrets Engine

Verify the secrets engines:

```bash
vault secrets list
```

Enable KV-v2 at `secret/` if it is not already enabled:

```bash
vault secrets enable -path=secret kv-v2
```

Verify:

```bash
vault secrets list
```

You should see:

```text
secret/    kv    ...
```

---

## 2.5 Perform a Vault Smoke Test

Write a test secret:

```bash
vault kv put secret/smoke-test value=ok
```

Read it:

```bash
vault kv get secret/smoke-test
```

Expected result:

```text
value    ok
```

This confirms that Vault's KV-v2 engine is functional.

---

## Vault Troubleshooting

### Helm chart compatibility error

If you see:

```text
chart requires kubeVersion: >= 1.x.x
```

or:

```text
no matches for kind "PodDisruptionBudget"
```

check the Kubernetes version:

```bash
kubectl version
```

Search compatible Vault versions:

```bash
helm search repo hashicorp/vault --versions
```

Install a compatible chart version:

```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --version <compatible-version>
```

Official documentation:

https://developer.hashicorp.com/vault/docs/platform/k8s/helm

---

# Task 3: Store Application Secrets in Vault

The demo application requires three groups of secrets:

1. Database credentials
2. API key
3. JWT signing key

Create the database secret:

```bash
vault kv put secret/demo-app/database \
  username="demo-user" \
  password="demo-password"
```

Create the API secret:

```bash
vault kv put secret/demo-app/api \
  api_key="initial-api-key"
```

Create the JWT secret:

```bash
vault kv put secret/demo-app/jwt \
  signing_key="demo-jwt-signing-key"
```

Verify the secrets:

```bash
vault kv list secret/demo-app/
```

Read individual values when required:

```bash
vault kv get secret/demo-app/database
vault kv get secret/demo-app/api
vault kv get secret/demo-app/jwt
```

**Important:** Never commit these values to Git.

---

# Task 4: Install External Secrets Operator

Add the ESO Helm repository:

```bash
helm repo add external-secrets \
  https://charts.external-secrets.io
```

Update Helm repositories:

```bash
helm repo update
```

Create the ESO namespace:

```bash
kubectl create namespace external-secrets
```

Install ESO:

```bash
helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace
```

Verify:

```bash
kubectl get pods -n external-secrets
```

All ESO components should become ready.

---

# Task 5: Configure Vault Kubernetes Authentication

ESO needs a secure method for authenticating against Vault.

The lab uses:

```text
Kubernetes ServiceAccount
        │
        ▼
Vault Kubernetes Auth
        │
        ▼
Vault Policy
        │
        ▼
secret/demo-app/*
```

Create the application namespace:

```bash
kubectl create namespace demo-app
```

Create a dedicated ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo-app-vault
  namespace: demo-app
```

Apply it:

```bash
kubectl apply -f serviceaccount.yaml
```

The Vault Kubernetes authentication configuration should associate this ServiceAccount and namespace with a dedicated Vault role.

The role should be able to read:

```text
secret/data/demo-app/*
```

but should not receive unnecessary permissions elsewhere.

---

# Task 6: Create the Vault Policy

Create a Vault policy that permits read access to the demo application's secrets.

Example policy:

```hcl
path "secret/data/demo-app/*" {
  capabilities = ["read"]
}
```

Save it as:

```text
demo-app-policy.hcl
```

Load the policy:

```bash
vault policy write demo-app demo-app-policy.hcl
```

Verify:

```bash
vault policy read demo-app
```

The policy should provide only the permissions required by the application.

---

# Task 7: Configure the Vault Kubernetes Auth Role

Enable Kubernetes authentication if it is not already enabled:

```bash
vault auth enable kubernetes
```

Configure the Kubernetes auth method according to the cluster environment.

Create a Vault role:

```text
demo-app
```

The role should bind to:

```text
ServiceAccount: demo-app-vault
Namespace: demo-app
```

and attach:

```text
demo-app-policy
```

Verify the role:

```bash
vault read auth/kubernetes/role/demo-app
```

Confirm that the ServiceAccount and namespace are correct.

---

# Task 8: Create the ESO SecretStore

Create a `SecretStore` in the `demo-app` namespace.

Conceptually, the resource connects ESO to Vault:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-store
  namespace: demo-app
spec:
  provider:
    vault:
      server: "http://vault.vault.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "demo-app"
          serviceAccountRef:
            name: demo-app-vault
```

Apply the resource:

```bash
kubectl apply -f secretstore.yaml
```

Check the status:

```bash
kubectl get secretstore -n demo-app
```

Describe it if necessary:

```bash
kubectl describe secretstore vault-store -n demo-app
```

---

# Task 9: Create ExternalSecret Resources

Create three `ExternalSecret` resources.

Each resource must use:

```yaml
refreshInterval: 60s
```

This ensures ESO periodically checks Vault for changes.

---

## Database ExternalSecret

The database credentials should be synchronized into a Kubernetes Secret.

Example:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: demo-database-secret
  namespace: demo-app
spec:
  refreshInterval: 60s
  secretStoreRef:
    name: vault-store
    kind: SecretStore
  target:
    name: demo-database-secret
  data:
    - secretKey: DB_USERNAME
      remoteRef:
        key: demo-app/database
        property: username
    - secretKey: DB_PASSWORD
      remoteRef:
        key: demo-app/database
        property: password
```

---

## API Key ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: demo-api-secret
  namespace: demo-app
spec:
  refreshInterval: 60s
  secretStoreRef:
    name: vault-store
    kind: SecretStore
  target:
    name: demo-api-secret
  data:
    - secretKey: API_KEY
      remoteRef:
        key: demo-app/api
        property: api_key
```

---

## JWT ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: demo-jwt-secret
  namespace: demo-app
spec:
  refreshInterval: 60s
  secretStoreRef:
    name: vault-store
    kind: SecretStore
  target:
    name: demo-jwt-secret
  data:
    - secretKey: signing_key
      remoteRef:
        key: demo-app/jwt
        property: signing_key
```

Apply the resources:

```bash
kubectl apply -f externalsecrets/
```

---

# Task 10: Validate Secret Synchronization

Check ExternalSecrets:

```bash
kubectl get externalsecret -n demo-app
```

Expected state:

```text
READY   STATUS
True    SecretSynced
```

All three ExternalSecrets must be ready.

Check the generated Kubernetes Secrets:

```bash
kubectl get secrets -n demo-app
```

Expected resources include:

```text
demo-database-secret
demo-api-secret
demo-jwt-secret
```

---

## Verify Secret Values

Decode the database username:

```bash
kubectl get secret demo-database-secret \
  -n demo-app \
  -o jsonpath='{.data.DB_USERNAME}' | base64 -d
```

Decode the database password:

```bash
kubectl get secret demo-database-secret \
  -n demo-app \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

Decode the API key:

```bash
kubectl get secret demo-api-secret \
  -n demo-app \
  -o jsonpath='{.data.API_KEY}' | base64 -d
```

Decode the JWT signing key:

```bash
kubectl get secret demo-jwt-secret \
  -n demo-app \
  -o jsonpath='{.data.signing_key}' | base64 -d
```

These values must match the values stored in Vault.

---

# ESO Troubleshooting

If an ExternalSecret reports:

```text
SecretSyncError
```

or:

```text
403 permission denied
```

inspect the resource:

```bash
kubectl describe externalsecret <name> -n demo-app
```

Check the Vault role:

```bash
vault read auth/kubernetes/role/demo-app
```

Verify:

* Correct ServiceAccount name
* Correct namespace
* Correct Vault policy
* Correct Vault authentication mount
* Correct secret path

The policy must grant read access to:

```text
secret/data/demo-app/*
```

Official documentation:

https://external-secrets.io/latest/provider/hashicorp-vault/

---

# Task 11: Create the Argo CD Application

Initialize a local Git repository:

```bash
mkdir -p ~/demo-app-gitops
cd ~/demo-app-gitops

git init
```

A recommended repository structure is:

```text
demo-app-gitops/
├── README.md
├── namespace.yaml
├── serviceaccount.yaml
├── secretstore.yaml
├── externalsecrets/
│   ├── database.yaml
│   ├── api.yaml
│   └── jwt.yaml
├── deployment.yaml
├── service.yaml
└── application.yaml
```

The repository must **never contain actual secret values**.

---

# Task 12: Create the Demo Application Deployment

The Deployment consumes the Kubernetes Secrets generated by ESO.

Two secrets are consumed as environment variables.

Example:

```yaml
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: demo-database-secret
        key: DB_USERNAME

  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: demo-database-secret
        key: DB_PASSWORD

  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: demo-api-secret
        key: API_KEY
```

The JWT signing key is mounted as a file:

```yaml
volumeMounts:
  - name: jwt-secret
    mountPath: /etc/secrets/jwt
    readOnly: true
```

The corresponding volume:

```yaml
volumes:
  - name: jwt-secret
    secret:
      secretName: demo-jwt-secret
```

This provides the JWT secret at:

```text
/etc/secrets/jwt
```

---

# Task 13: Configure Argo CD Automated Synchronization

Create an Argo CD `Application`.

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default

  source:
    repoURL: <LOCAL-GIT-REPOSITORY>
    targetRevision: HEAD
    path: .

  destination:
    server: https://kubernetes.default.svc
    namespace: demo-app

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply the Application:

```bash
kubectl apply -f application.yaml
```

Check the Application:

```bash
kubectl get application demo-app -n argocd
```

Check synchronization:

```bash
kubectl get application demo-app \
  -n argocd \
  -o jsonpath='{.status.sync.status}'
```

Expected:

```text
Synced
```

Check health:

```bash
kubectl get application demo-app \
  -n argocd \
  -o jsonpath='{.status.health.status}'
```

Expected:

```text
Healthy
```

No manual:

```bash
argocd app sync
```

should be required after the initial Application creation.

---

# Task 14: Validate Pod-Level Secret Consumption

Check the application pods:

```bash
kubectl get pods -n demo-app
```

The pod should reach:

```text
Running
```

Check the API key:

```bash
kubectl exec -n demo-app <pod-name> -- env | grep API_KEY
```

Verify the JWT secret file:

```bash
kubectl exec -n demo-app <pod-name> \
  -- ls -l /etc/secrets/jwt
```

The application should be able to consume the JWT signing key from:

```text
/etc/secrets/jwt
```

---

# Task 15: Demonstrate Secret Rotation

The lab must demonstrate that Vault is the source of truth and that changes propagate through ESO.

First, record the original API key:

```bash
vault kv get secret/demo-app/api
```

Update the API key:

```bash
vault kv put secret/demo-app/api \
  api_key="rotated-api-key"
```

Because the `ExternalSecret` uses:

```yaml
refreshInterval: 60s
```

ESO should detect the change during its next refresh cycle.

Monitor the ExternalSecret:

```bash
kubectl get externalsecret -n demo-app -w
```

Check the Kubernetes Secret:

```bash
kubectl get secret demo-api-secret -n demo-app
```

Decode the API key:

```bash
kubectl get secret demo-api-secret \
  -n demo-app \
  -o jsonpath='{.data.API_KEY}' | base64 -d
```

The value should now be:

```text
rotated-api-key
```

---

# Task 16: Restart the Application

Environment variables sourced from Kubernetes Secrets are normally populated when a container starts.

Restart the Deployment:

```bash
kubectl rollout restart deployment/<deployment-name> \
  -n demo-app
```

Monitor the rollout:

```bash
kubectl rollout status deployment/<deployment-name> \
  -n demo-app
```

Verify the new pod:

```bash
kubectl get pods -n demo-app
```

Confirm the rotated API key:

```bash
kubectl exec -n demo-app <pod-name> \
  -- env | grep API_KEY
```

Expected:

```text
API_KEY=rotated-api-key
```

The pod must return to `Running` within 120 seconds.

---

# Validation Checklist

## Toolchain

* [ ] Docker installed and verified.
* [ ] kubectl installed and verified.
* [ ] kind installed and verified.
* [ ] Argo CD CLI installed and verified.
* [ ] Vault CLI installed and verified.
* [ ] Helm installed and verified.
* [ ] kind cluster created successfully.
* [ ] All Argo CD pods are Running and ready.

## Vault

* [ ] Vault deployed inside the kind cluster.
* [ ] Vault initialized.
* [ ] Vault unsealed.
* [ ] Vault UI/API accessible through port forwarding.
* [ ] KV-v2 enabled at `secret/`.
* [ ] Smoke-test secret successfully written and retrieved.
* [ ] Database credentials stored in Vault.
* [ ] API key stored in Vault.
* [ ] JWT signing key stored in Vault.

## External Secrets Operator

* [ ] ESO installed using Helm.
* [ ] Dedicated ServiceAccount created.
* [ ] Vault Kubernetes authentication configured.
* [ ] Vault policy created.
* [ ] Vault role configured.
* [ ] SecretStore created.
* [ ] Three ExternalSecrets created.
* [ ] Every ExternalSecret uses `refreshInterval: 60s`.
* [ ] All ExternalSecrets report `READY: True`.
* [ ] Kubernetes Secrets are generated successfully.

## Argo CD

* [ ] Local Git repository initialized.
* [ ] Kubernetes manifests committed.
* [ ] No secret values stored in Git.
* [ ] Argo CD Application created.
* [ ] Automated synchronization enabled.
* [ ] Self-healing enabled.
* [ ] Application reports `Synced`.
* [ ] Application reports `Healthy`.

## Secret Rotation

* [ ] Original API key verified.
* [ ] API key rotated in Vault.
* [ ] ESO detected the updated value.
* [ ] Kubernetes Secret updated.
* [ ] Deployment restarted.
* [ ] New pod reached `Running`.
* [ ] Rotated API key verified inside the pod.

---

# Troubleshooting Guide

## Check All Namespaces

```bash
kubectl get pods -A
```

## Check Argo CD

```bash
kubectl get pods -n argocd
kubectl get applications -n argocd
```

## Check Vault

```bash
kubectl get pods -n vault
kubectl get svc -n vault
vault status
```

## Check ESO

```bash
kubectl get pods -n external-secrets
kubectl get externalsecret -n demo-app
kubectl get secretstore -n demo-app
```

## Check Demo Application

```bash
kubectl get all -n demo-app
```

## Inspect ExternalSecret Events

```bash
kubectl describe externalsecret \
  <externalsecret-name> \
  -n demo-app
```

## Inspect SecretStore

```bash
kubectl describe secretstore \
  vault-store \
  -n demo-app
```

## Check Application Events

```bash
kubectl describe deployment \
  <deployment-name> \
  -n demo-app
```

## Check Pod Logs

```bash
kubectl logs \
  -n demo-app \
  <pod-name>
```

---

# Security Best Practices

Although this lab uses Vault development mode for simplicity, a production implementation should improve the security architecture.

Recommended production practices include:

* Use Vault HA rather than development mode.
* Enable Vault audit logging.
* Use TLS for Vault communication.
* Restrict Vault policies using least privilege.
* Use dedicated ServiceAccounts.
* Avoid sharing Vault authentication roles between applications.
* Keep secret paths isolated by application and environment.
* Never commit secret values to Git.
* Never place secret values directly into Kubernetes manifests.
* Protect Git repositories with appropriate access controls.
* Rotate credentials regularly.
* Monitor secret access.
* Implement backup and disaster recovery for Vault.
* Use automated Vault unseal mechanisms where appropriate.
* Protect Kubernetes ServiceAccount credentials.
* Use network policies where appropriate.
* Monitor ESO synchronization failures.

---

# Git Repository Security

Before committing changes, inspect the repository:

```bash
git status
```

Search for potentially exposed credentials:

```bash
grep -RniE \
  'password|secret|api[_-]?key|token|private[_-]?key' \
  .
```

This command can produce false positives because manifests contain references to secrets, but it helps identify accidental plaintext credentials.

Review the Git diff:

```bash
git diff
```

Only commit references such as:

```text
secret/demo-app/api
secret/demo-app/database
secret/demo-app/jwt
```

Never commit:

```text
api_key=real-secret-value
password=real-password
jwt_signing_key=real-private-key
```

---

# End-to-End Secret Lifecycle

The complete lifecycle demonstrated by this lab is:

```text
1. Secret Creation
       │
       ▼
2. Secret stored in Vault
       │
       ▼
3. ESO authenticates to Vault
       │
       ▼
4. ESO reads the secret
       │
       ▼
5. Kubernetes Secret created
       │
       ▼
6. Argo CD manages application manifests
       │
       ▼
7. Pod consumes Kubernetes Secret
       │
       ▼
8. Secret changed in Vault
       │
       ▼
9. ESO refreshes within configured interval
       │
       ▼
10. Kubernetes Secret updated
       │
       ▼
11. Deployment restarted
       │
       ▼
12. Pod consumes rotated secret
```

---

# Expected Outcomes

After completing the lab, you should have a fully functional GitOps-based secret management pipeline.

The final environment should provide:

```text
Git
 │
 │ Configuration only
 ▼
Argo CD
 │
 │ Reconciliation
 ▼
Kubernetes
 │
 ├── External Secrets Operator
 │
 ├── SecretStore
 │
 ├── ExternalSecrets
 │
 └── Demo Application
        │
        ▼
     Kubernetes Secrets
        ▲
        │
        │ Synchronization
        │
     Vault
```

The application secrets never need to be stored in Git.

Vault remains the source of truth, ESO provides automated synchronization, and Argo CD manages the desired Kubernetes application state.

---

# Learning Outcomes

This lab provides practical experience with:

* Kubernetes Secrets
* HashiCorp Vault
* Vault KV-v2
* Vault Kubernetes authentication
* Vault policies
* External Secrets Operator
* SecretStore
* ExternalSecret
* Kubernetes ServiceAccounts
* Argo CD Applications
* GitOps
* Automated synchronization
* Argo CD self-healing
* Secret rotation
* Pod-level secret consumption
* Secure GitOps architecture

---

# Conclusion

This lab demonstrates how **HashiCorp Vault, External Secrets Operator, and Argo CD** can work together to provide a secure GitOps-based secret management architecture.

The central design principle is simple:

```text
Git = Desired configuration and intent
Vault = Secret source of truth
ESO = Secret synchronization
Kubernetes = Runtime secret delivery
Argo CD = GitOps reconciliation
```

Instead of placing sensitive credentials directly into Git, the application declares **which secrets it requires** and ESO retrieves the actual values from Vault.

The final rotation exercise proves that a secret can be changed at its source, synchronized into Kubernetes, and consumed by a newly restarted application pod without modifying the secret value in Git.

For production environments, this architecture can be extended with **Vault HA, TLS, audit logging, automated secret rotation, stronger access controls, monitoring, disaster recovery, and automated lease management**.

---

## References

* Docker Ubuntu installation: https://docs.docker.com/engine/install/ubuntu/
* kind documentation: https://kind.sigs.k8s.io/docs/user/quick-start/
* Argo CD documentation: https://argo-cd.readthedocs.io/en/stable/getting_started/
* HashiCorp Vault Helm deployment: https://developer.hashicorp.com/vault/docs/platform/k8s/helm
* External Secrets Operator Vault provider: https://external-secrets.io/latest/provider/hashicorp-vault/

---

## Author

**Hafiz Muhammad Salman**

Cloud DevOps Engineer | Linux Administrator

GitHub: https://github.com/msalman199
