# Implementing Production-Grade RBAC in Argo CD

A hands-on lab for designing, implementing, validating, and troubleshooting **production-grade Role-Based Access Control (RBAC)** in Argo CD using its embedded **Casbin policy engine**.

The lab builds a multi-tier authorization model for developers, DevOps engineers, and administrators, then validates permissions against scoped Argo CD Projects and Applications.

---

## 🎯 Objectives

By completing this lab, you will be able to:

* Design a multi-tier RBAC policy in Argo CD.
* Configure local Argo CD users and role bindings.
* Understand how Argo CD uses the Casbin policy engine for authorization.
* Restrict application permissions by AppProject and application object.
* Separate developer, DevOps, and administrative privileges.
* Validate allowed and denied operations using `argocd account can-i`.
* Build an automated RBAC validation harness with Bash.
* Analyze Argo CD server logs for RBAC enforcement decisions.
* Troubleshoot policy inheritance, default roles, and permission gaps.
* Apply production-oriented least-privilege and separation-of-duties principles.

---

## 🏗️ Lab Architecture

```text
                         ┌──────────────────────┐
                         │      Git Repository   │
                         │ argocd-example-apps  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │       Argo CD        │
                         │                      │
                         │  Casbin RBAC Engine  │
                         └──────────┬───────────┘
                                    │
             ┌──────────────────────┼──────────────────────┐
             │                      │                      │
             ▼                      ▼                      ▼
       ┌───────────┐          ┌───────────┐          ┌───────────┐
       │   Alice   │          │    Bob    │          │  Charlie  │
       │ Developer │          │  DevOps   │          │ Org Admin │
       └─────┬─────┘          └─────┬─────┘          └─────┬─────┘
             │                      │                      │
             ▼                      ▼                      ▼
       team-alpha             Applications,          All Argo CD
       Applications            Repositories,           resources
                               Clusters, Projects
             │
             ▼
       ┌─────────────┐
       │  Kubernetes │
       │   Cluster   │
       └──────┬──────┘
              │
       ┌──────┴──────┐
       ▼             ▼
   alpha-ns       beta-ns
```

---

## 🔐 RBAC Model

The lab implements three primary roles.

| User      | Role      | Allowed Operations                                                        | Restricted Operations                           |
| --------- | --------- | ------------------------------------------------------------------------- | ----------------------------------------------- |
| `alice`   | Developer | Get/sync applications, read logs                                          | Create/delete applications, manage repositories |
| `bob`     | DevOps    | Full application lifecycle, repository management, read clusters/projects | Manage clusters/projects                        |
| `charlie` | Org Admin | All resources and actions                                                 | None                                            |

### Role Mapping

```text
alice   → role:developer
bob     → role:devops
charlie → role:org-admin
```

### Default Policy

Authenticated users who are not explicitly assigned a role receive:

```text
role:readonly
```

This provides a secure baseline rather than automatically granting privileges.

---

## 📦 Prerequisites

Before starting, you should have:

* Basic Linux command-line experience.
* Familiarity with Bash scripting.
* Understanding of Kubernetes namespaces.
* Understanding of Kubernetes RBAC concepts.
* Basic knowledge of GitOps.
* Familiarity with Argo CD Applications and AppProjects.
* Access to an AWS EC2 Ubuntu instance.
* Internet connectivity.

---

## ☁️ Lab Environment

The lab uses a dedicated Ubuntu EC2 instance provided through **Al Nafi**.

The environment consists of:

* Ubuntu Linux
* Docker
* Minikube
* Kubernetes
* kubectl
* Argo CD
* Argo CD CLI
* Bash
* Git

Minikube provides the local Kubernetes cluster used to host Argo CD.

---

# Task 1 — Provision the Environment

## 1.1 Install Required Packages

Update the operating system and install the basic utilities.

```bash
sudo apt-get update -y
sudo apt-get upgrade -y

sudo apt-get install -y \
  curl \
  wget \
  git \
  apt-transport-https \
  ca-certificates \
  gnupg \
  lsb-release
```

---

## 1.2 Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh

sudo usermod -aG docker $USER
newgrp docker

docker version
```

Verify that Docker is operational before continuing.

### Troubleshooting

If you receive:

```text
permission denied while trying to connect to the Docker daemon socket
```

Check the user's groups:

```bash
groups
```

If `docker` is missing, run:

```bash
newgrp docker
```

Alternatively, log out and log back into the instance.

---

## 1.3 Install kubectl

```bash
KUBECTL_VERSION=$(curl -fsSL https://dl.k8s.io/release/stable.txt)

curl -fsSL \
  "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
  -o kubectl

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

rm kubectl

kubectl version --client
```

### Troubleshooting

If the binary download returns `404`, verify the current Kubernetes version:

```bash
curl -fsSL https://dl.k8s.io/release/stable.txt
```

Then retry using the returned version.

---

## 1.4 Install Minikube

```bash
curl -fsSL \
  https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  -o minikube

sudo install -o root -g root -m 0755 minikube /usr/local/bin/minikube

rm minikube

minikube version
```

---

## 1.5 Start the Kubernetes Cluster

Allocate at least 4 GB of memory and 2 CPUs.

```bash
minikube start \
  --driver=docker \
  --memory=4096 \
  --cpus=2
```

Verify Kubernetes connectivity:

```bash
kubectl cluster-info
kubectl get nodes
```

Expected result:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...   ...
```

### Troubleshooting

If Minikube hangs or Docker reports container errors:

```bash
minikube delete
```

Then recreate the cluster:

```bash
minikube start \
  --driver=docker \
  --memory=4096 \
  --cpus=2
```

If Docker is not running:

```bash
sudo systemctl start docker
sudo systemctl enable docker

docker info
```

---

# Task 2 — Deploy Argo CD

## 2.1 Create the Argo CD Namespace

```bash
kubectl create namespace argocd
```

Deploy Argo CD:

```bash
kubectl apply \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

## 2.2 Verify Argo CD Components

Wait for the major control-plane components:

```bash
kubectl wait \
  --for=condition=available \
  --timeout=300s \
  deployment/argocd-server \
  -n argocd

kubectl wait \
  --for=condition=available \
  --timeout=300s \
  deployment/argocd-repo-server \
  -n argocd

kubectl wait \
  --for=condition=available \
  --timeout=300s \
  deployment/argocd-application-controller \
  -n argocd
```

Check all pods:

```bash
kubectl get pods -n argocd
```

All critical components should eventually reach `Running` or an appropriate healthy state.

### Troubleshooting

If `argocd-server` does not become available:

```bash
kubectl describe pod \
  -n argocd \
  -l app.kubernetes.io/name=argocd-server
```

Look for:

* `ImagePullBackOff`
* insufficient memory
* failed mounts
* scheduling problems
* container crashes

If resources are insufficient:

```bash
minikube stop

minikube start \
  --driver=docker \
  --memory=6144 \
  --cpus=2
```

---

# Task 3 — Install and Configure the Argo CD CLI

Install the latest Argo CD CLI:

```bash
ARGOCD_VERSION=$(curl -fsSL \
  https://api.github.com/repos/argoproj/argo-cd/releases/latest \
  | grep '"tag_name"' \
  | sed 's/.*"tag_name": "\(.*\)".*/\1/')

curl -fsSL \
  "https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64" \
  -o argocd

sudo install -m 0755 argocd /usr/local/bin/argocd

rm argocd

argocd version --client
```

---

## 3.1 Expose Argo CD

Create a local port-forward:

```bash
kubectl port-forward \
  svc/argocd-server \
  -n argocd \
  8080:443 &
```

Wait briefly:

```bash
sleep 5
```

---

## 3.2 Retrieve the Initial Admin Password

```bash
ARGOCD_INITIAL_PASSWORD=$(kubectl -n argocd get secret \
  argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" \
  | base64 --decode)
```

---

## 3.3 Authenticate

```bash
argocd login localhost:8080 \
  --username admin \
  --password "${ARGOCD_INITIAL_PASSWORD}" \
  --insecure
```

Verify the account:

```bash
argocd account list
```

The `admin` account should be enabled.

---

# Task 4 — Create Namespaces

Create the two namespaces used by the AppProjects:

```bash
kubectl create namespace alpha-ns
kubectl create namespace beta-ns
```

Verify:

```bash
kubectl get namespaces
```

---

# Task 5 — Configure Local Argo CD Users

Argo CD stores local account configuration in `argocd-cm`.

Patch the ConfigMap instead of replacing it:

```bash
kubectl patch configmap argocd-cm \
  -n argocd \
  --type merge \
  -p '{
    "data": {
      "accounts.alice": "login",
      "accounts.bob": "login",
      "accounts.charlie": "login"
    }
  }'
```

Verify:

```bash
kubectl get configmap argocd-cm \
  -n argocd \
  -o jsonpath='{.data}'
```

---

## 5.1 Configure User Passwords

Use the bootstrap administrator credentials to configure the local accounts:

```bash
argocd account update-password \
  --account alice \
  --new-password "AlicePass#2024" \
  --current-password "${ARGOCD_INITIAL_PASSWORD}"
```

```bash
argocd account update-password \
  --account bob \
  --new-password "BobPass#2024" \
  --current-password "${ARGOCD_INITIAL_PASSWORD}"
```

```bash
argocd account update-password \
  --account charlie \
  --new-password "CharliePass#2024" \
  --current-password "${ARGOCD_INITIAL_PASSWORD}"
```

Verify:

```bash
argocd account list
```

> **Security note:** The passwords above are lab credentials only. Never reuse lab passwords in production.

---

# Task 6 — Create AppProjects

The lab uses two isolated AppProjects:

```text
team-alpha → alpha-ns
team-beta  → beta-ns
```

## Team Alpha

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-alpha
  namespace: argocd
spec:
  description: "Project scoped to the alpha team namespace"
  sourceRepos:
    - "https://github.com/argoproj/argocd-example-apps.git"
  destinations:
    - namespace: alpha-ns
      server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap
```

## Team Beta

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-beta
  namespace: argocd
spec:
  description: "Project scoped to the beta team namespace"
  sourceRepos:
    - "https://github.com/argoproj/argocd-example-apps.git"
  destinations:
    - namespace: beta-ns
      server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap
```

Apply them with:

```bash
kubectl apply -f team-alpha.yaml
kubectl apply -f team-beta.yaml
```

Or apply the manifests directly with `kubectl apply -f -`.

Verify:

```bash
argocd proj list
```

---

# Task 7 — Configure Casbin RBAC

Argo CD RBAC policies are stored in:

```text
argocd-rbac-cm
```

The primary policy format is:

```text
p, subject, resource, action, object, effect
```

For example:

```text
p, role:developer, applications, sync, team-alpha/*, allow
```

means that the developer role can synchronize applications belonging to `team-alpha`.

---

## 7.1 RBAC Policy

Apply the following policy:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
data:
  policy.default: role:readonly

  policy.csv: |
    # Baseline readonly role
    p, role:readonly, applications, get, */*, allow
    p, role:readonly, projects,     get, *,   allow

    # Developer
    p, role:developer, applications, get,  */*,          allow
    p, role:developer, applications, sync, team-alpha/*, allow
    p, role:developer, logs,         get,  */*,          allow

    # DevOps
    p, role:devops, applications, get,    */*, allow
    p, role:devops, applications, create, */*, allow
    p, role:devops, applications, update, */*, allow
    p, role:devops, applications, delete, */*, allow
    p, role:devops, applications, sync,   */*, allow
    p, role:devops, repositories, get,    *,   allow
    p, role:devops, repositories, create, *,   allow
    p, role:devops, repositories, update, *,   allow
    p, role:devops, repositories, delete, *,   allow
    p, role:devops, clusters,     get,     *,   allow
    p, role:devops, projects,     get,     *,   allow
    p, role:devops, logs,         get,     */*, allow

    # Organization administrator
    p, role:org-admin, *, *, *, allow

    # User-to-role assignments
    g, alice,   role:developer
    g, bob,     role:devops
    g, charlie, role:org-admin
```

Apply the configuration:

```bash
kubectl apply -f argocd-rbac-cm.yaml
```

Verify:

```bash
kubectl get configmap argocd-rbac-cm \
  -n argocd \
  -o jsonpath='{.data.policy\.csv}'
```

---

## 7.2 Reload the Argo CD Server

Restart the server so the configuration is reloaded:

```bash
kubectl rollout restart deployment/argocd-server -n argocd
```

Wait for availability:

```bash
kubectl wait \
  --for=condition=available \
  --timeout=120s \
  deployment/argocd-server \
  -n argocd
```

---

# Task 8 — Deploy Test Applications

The lab uses two Applications to verify project-level authorization.

```text
guestbook-alpha → team-alpha → alpha-ns
guestbook-beta  → team-beta  → beta-ns
```

## Alpha Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-alpha
  namespace: argocd
spec:
  project: team-alpha
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: alpha-ns
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
```

## Beta Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-beta
  namespace: argocd
spec:
  project: team-beta
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: beta-ns
  syncPolicy:
    automated:
      prune: false
      selfHeal: false
```

Apply both:

```bash
kubectl apply -f guestbook-alpha.yaml
kubectl apply -f guestbook-beta.yaml
```

Verify:

```bash
argocd app list
```

Expected applications:

```text
guestbook-alpha
guestbook-beta
```

The `PROJECT` column should show their respective AppProjects.

---

# Task 9 — Validate RBAC Boundaries

The central objective of this lab is to verify that each user receives exactly the permissions defined by the policy matrix.

Use:

```bash
argocd account can-i
```

for pre-flight authorization checks.

---

## RBAC Test Matrix

| User    | Resource     | Action | Object                     | Expected |
| ------- | ------------ | ------ | -------------------------- | -------- |
| Alice   | applications | get    | team-alpha/guestbook-alpha | Allow    |
| Alice   | applications | sync   | team-alpha/guestbook-alpha | Allow    |
| Alice   | applications | sync   | team-beta/guestbook-beta   | Deny     |
| Alice   | applications | create | team-alpha/*               | Deny     |
| Alice   | applications | delete | team-alpha/*               | Deny     |
| Bob     | applications | create | team-alpha/*               | Allow    |
| Bob     | applications | delete | team-alpha/*               | Allow    |
| Bob     | repositories | create | *                          | Allow    |
| Bob     | projects     | create | *                          | Deny     |
| Bob     | clusters     | delete | *                          | Deny     |
| Charlie | projects     | create | *                          | Allow    |
| Charlie | clusters     | delete | *                          | Allow    |

---

# Task 10 — Build the RBAC Validation Harness

Create:

```text
rbac-validator.sh
```

The script should implement these functions:

```text
rbac_test_case()
rbac_run_suite()
login_as()
restore_admin_session()
```

### `rbac_test_case`

Arguments:

```text
username
password
resource
action
object
expected_result
```

Output:

```text
PASS
```

or:

```text
FAIL: <reason>
```

Return codes:

```text
0 → PASS
1 → FAIL
```

### `rbac_run_suite`

The function should:

1. Execute every test case.
2. Record the result.
3. Print each result.
4. Generate a summary.
5. Exit with a non-zero status if any test fails.

---

## Run the Validator

Make the script executable:

```bash
chmod +x rbac-validator.sh
```

Run:

```bash
./rbac-validator.sh
```

The final summary should contain:

```text
FAIL: 0
```

Every required test case must report:

```text
PASS
```

---

# Task 11 — Audit RBAC Decisions

RBAC troubleshooting becomes significantly easier when server logs provide detailed policy enforcement information.

## 11.1 Enable Debug Logging

Patch the Argo CD command parameters:

```bash
kubectl patch configmap argocd-cmd-params-cm \
  -n argocd \
  --type merge \
  -p '{
    "data": {
      "server.log.level": "debug",
      "server.log.format": "json"
    }
  }'
```

Restart the server:

```bash
kubectl rollout restart deployment/argocd-server -n argocd
```

Wait for it:

```bash
kubectl wait \
  --for=condition=available \
  --timeout=120s \
  deployment/argocd-server \
  -n argocd
```

---

# Task 12 — Trigger a Denied Operation

Authenticate as Alice:

```bash
argocd login localhost:8080 \
  --username alice \
  --password "AlicePass#2024" \
  --insecure
```

Attempt to synchronize the beta application:

```bash
argocd app sync guestbook-beta || true
```

Alice should not have permission to synchronize applications in `team-beta`.

---

# Task 13 — Inspect Argo CD Server Logs

Retrieve recent server logs:

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-server \
  --since=2m \
  | grep -i "enforc" \
  | tail -20
```

Look for:

* the requesting identity
* the resource
* the action
* the target object
* the authorization decision
* Casbin enforcement information

A denied operation should produce evidence of the authorization decision.

---

# Task 14 — Validate with `account can-i`

Re-authenticate as administrator:

```bash
ARGOCD_INITIAL_PASSWORD=$(kubectl -n argocd get secret \
  argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" \
  | base64 --decode

argocd login localhost:8080 \
  --username admin \
  --password "${ARGOCD_INITIAL_PASSWORD}" \
  --insecure
```

Test Alice:

```bash
argocd account can-i \
  sync applications \
  --as alice \
  "team-alpha/guestbook-alpha"
```

Expected:

```text
yes
```

Test Alice against beta:

```bash
argocd account can-i \
  sync applications \
  --as alice \
  "team-beta/guestbook-beta"
```

Expected:

```text
no
```

Test Alice creating an application:

```bash
argocd account can-i \
  create applications \
  --as alice \
  "team-alpha/*"
```

Expected:

```text
no
```

Test Bob:

```bash
argocd account can-i \
  create applications \
  --as bob \
  "team-alpha/*"
```

Expected:

```text
yes
```

Bob should not manage projects:

```bash
argocd account can-i \
  delete projects \
  --as bob \
  "*"
```

Expected:

```text
no
```

Charlie should have unrestricted access:

```bash
argocd account can-i \
  delete projects \
  --as charlie \
  "*"
```

Expected:

```text
yes
```

---

# 🔍 Troubleshooting

## `can-i` Returns `yes` When `no` Was Expected

Investigate the complete policy evaluation chain.

Check the default role:

```bash
kubectl get configmap argocd-rbac-cm \
  -n argocd \
  -o jsonpath='{.data.policy\.default}'
```

Inspect the complete policy:

```bash
kubectl get configmap argocd-rbac-cm \
  -n argocd \
  -o jsonpath='{.data.policy\.csv}'
```

Look for:

* overly broad `allow` rules
* incorrect object patterns
* unintended group assignments
* inherited permissions
* incorrect default role
* wildcard resources
* wildcard actions

Remember that a policy such as:

```text
p, role:developer, *, *, *, allow
```

would effectively defeat a least-privilege design.

---

## RBAC Changes Are Not Taking Effect

Check that the ConfigMap was updated:

```bash
kubectl get configmap argocd-rbac-cm -n argocd
```

Inspect the server:

```bash
kubectl get pods -n argocd
```

Restart the server:

```bash
kubectl rollout restart deployment/argocd-server -n argocd
```

Wait for readiness:

```bash
kubectl wait \
  --for=condition=available \
  --timeout=120s \
  deployment/argocd-server \
  -n argocd
```

---

## No RBAC Enforcement Logs Appear

Verify the command parameters:

```bash
kubectl get configmap argocd-cmd-params-cm \
  -n argocd \
  -o yaml
```

Confirm:

```yaml
server.log.level: debug
server.log.format: json
```

Then restart:

```bash
kubectl rollout restart deployment/argocd-server -n argocd
```

Check current logs:

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-server \
  --since=2m
```

---

## Argo CD Server Does Not Become Ready

Inspect the pods:

```bash
kubectl get pods -n argocd
```

Inspect the server:

```bash
kubectl describe deployment argocd-server -n argocd
```

Inspect recent events:

```bash
kubectl get events \
  -n argocd \
  --sort-by=.lastTimestamp
```

If Minikube is resource constrained, increase memory:

```bash
minikube stop

minikube start \
  --driver=docker \
  --memory=6144 \
  --cpus=2
```

---

# 🛡️ Production RBAC Best Practices

This lab demonstrates several principles that should be applied to real GitOps environments.

### Least Privilege

Grant only the actions a user actually requires.

### Separation of Duties

Separate development, operations, and administrative responsibilities.

### Project Scoping

Use AppProjects to restrict:

* source repositories
* destination clusters
* destination namespaces
* resource types

### Avoid Broad Wildcards

Rules such as:

```text
*, *, *, allow
```

should be reserved for carefully controlled administrative roles.

### Protect Credentials

Do not commit real passwords to Git repositories or scripts.

For production:

* use SSO/OIDC where appropriate
* use external identity providers
* rotate credentials
* protect administrative accounts
* use secret-management systems

### Audit Authorization Decisions

Use logs and authorization checks to investigate unexpected permissions.

### Test Policy Changes

Every RBAC change should be validated against both positive and negative test cases.

---

# 📊 Expected Results

At the end of the lab:

* Minikube is running successfully.
* Argo CD is installed and healthy.
* Local users `alice`, `bob`, and `charlie` exist.
* `team-alpha` and `team-beta` AppProjects are configured.
* Applications are scoped to their respective projects.
* Alice can synchronize only `team-alpha` applications.
* Bob can manage applications and repositories but cannot manage projects or clusters.
* Charlie has unrestricted administrative permissions.
* The RBAC validation script reports zero failures.
* `argocd account can-i` decisions match the policy matrix.
* Argo CD server logs provide evidence of RBAC enforcement decisions.

---

# 🧪 Validation Checklist

* [ ] Docker installed and operational
* [ ] kubectl installed and operational
* [ ] Minikube cluster running
* [ ] Cluster has at least 4 GB memory and 2 CPUs
* [ ] Argo CD namespace created
* [ ] Argo CD control-plane components healthy
* [ ] Argo CD CLI installed
* [ ] Admin authentication verified
* [ ] `alpha-ns` created
* [ ] `beta-ns` created
* [ ] Alice account configured
* [ ] Bob account configured
* [ ] Charlie account configured
* [ ] `team-alpha` AppProject created
* [ ] `team-beta` AppProject created
* [ ] RBAC policy applied
* [ ] RBAC ConfigMap verified
* [ ] `guestbook-alpha` deployed
* [ ] `guestbook-beta` deployed
* [ ] `rbac-validator.sh` implemented
* [ ] All RBAC test cases pass
* [ ] Debug logging enabled
* [ ] Denied action generated
* [ ] Server logs inspected
* [ ] `argocd account can-i` results verified

---

# 🧠 Key Concepts Learned

This lab demonstrates how Argo CD combines several security controls:

```text
                    Argo CD Authorization
                           │
             ┌─────────────┴─────────────┐
             │                           │
        Casbin RBAC                 AppProjects
             │                           │
       User → Role                 Scope boundaries
             │                           │
       Resource/Action          Repository/Destination
             │                           │
             └─────────────┬─────────────┘
                           │
                    Application Access
```

The important distinction is that **RBAC determines what a user can do**, while **AppProjects constrain where applications are allowed to source from and deploy to**.

Together they provide a strong foundation for secure GitOps authorization.

---

# 🏁 Conclusion

This lab implemented a production-oriented RBAC model in Argo CD using the embedded Casbin policy engine.

You created separate developer, DevOps, and administrator roles, configured scoped AppProjects, deployed test applications, and validated authorization boundaries programmatically.

The combination of:

* declarative RBAC policies
* AppProject isolation
* least-privilege permissions
* automated validation
* `argocd account can-i`
* server-side authorization logs

provides a repeatable approach for designing and troubleshooting secure Argo CD environments.

The patterns demonstrated here can be extended to enterprise GitOps platforms where **least privilege, separation of duties, auditability, and controlled application deployment** are critical security requirements.

---

## 📚 Official Documentation

* [Argo CD Documentation](https://argo-cd.readthedocs.io/)
* [Argo CD RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/)
* [Argo CD AppProjects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
* [Argo CD CLI](https://argo-cd.readthedocs.io/en/stable/user-guide/commands/argocd/)
* [Kubernetes Documentation](https://kubernetes.io/docs/)
* [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)
* [Docker Documentation](https://docs.docker.com/)
