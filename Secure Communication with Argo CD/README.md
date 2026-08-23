<div align="center">

# 🔒 Secure Communication with Argo CD

![Argo CD](https://img.shields.io/badge/Argo_CD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![TLS](https://img.shields.io/badge/TLS%2FSSL-000000?style=for-the-badge&logo=letsencrypt&logoColor=white)
![Keycloak](https://img.shields.io/badge/Keycloak-4D4D4D?style=for-the-badge&logo=keycloak&logoColor=white)
![OpenID Connect](https://img.shields.io/badge/OpenID_Connect-F78C40?style=for-the-badge&logo=openid&logoColor=white)
![NGINX](https://img.shields.io/badge/NGINX_Ingress-009639?style=for-the-badge&logo=nginx&logoColor=white)

**A design-brief lab: harden a production-grade Argo CD deployment with TLS, Keycloak SSO, and RBAC**

</div>

---

## 📖 Table of Contents

- [🎯 Objectives](#-objectives)
- [📋 Prerequisites](#-prerequisites)
- [🖥️ Lab Environment](#️-lab-environment)
- [🧰 Task 1: Install and Verify the Complete Toolchain](#-task-1-install-and-verify-the-complete-toolchain)
- [🔐 Task 2: Secure Argo CD with TLS and Expose It via HTTPS Ingress](#-task-2-secure-argo-cd-with-tls-and-expose-it-via-https-ingress)
- [🔑 Task 3: Integrate Keycloak SSO and Enforce RBAC](#-task-3-integrate-keycloak-sso-and-enforce-rbac)
- [🎯 Expected Outcomes](#-expected-outcomes)
- [🛡️ MITRE ATT&CK Mapping](#️-mitre-attck-mapping)
- [📚 Key Concepts](#-key-concepts)
- [✅ Conclusion](#-conclusion)

---

## 🎯 Objectives

| # | Objective |
|---|-----------|
| 1 | Design and deploy a production-grade Argo CD installation secured with self-signed TLS certificates and HTTPS-only ingress |
| 2 | Integrate Argo CD with Keycloak as an OpenID Connect (OIDC) provider to replace local authentication with SSO |
| 3 | Enforce role-based access control (RBAC) policies that map Keycloak groups to Argo CD permission roles |

## 📋 Prerequisites

| # | Requirement |
|---|-------------|
| 1 | Comfort with Linux command-line operations including file editing, process management, and reading command output |
| 2 | Conceptual understanding of TLS certificates (public/private key pairs, certificate signing, Subject Alternative Names), OAuth 2.0 / OpenID Connect flows, and Kubernetes resource types (Deployment, Service, Ingress, ConfigMap, Secret) |

## 🖥️ Lab Environment

> You will work on a dedicated **AWS EC2 Ubuntu instance** provided by Al Nafi. The instance has a base Ubuntu installation; you will install all required tools in Task 1.

---

## 🧰 Task 1: Install and Verify the Complete Toolchain

### 📝 Requirement 1

Design and execute an installation sequence that produces a working local Kubernetes cluster reachable via `kubectl`, with an NGINX Ingress Controller running inside it, and with `helm`, `openssl`, `jq`, and the Argo CD CLI (`argocd`) available on the `PATH`. The cluster must expose host ports 80 and 443 so that Ingress resources are reachable from the EC2 instance itself. Install Docker first, then kind (Kubernetes in Docker — a tool that runs a full Kubernetes cluster inside Docker containers), then the remaining tools.

Reference installation guides if any URL below has changed:

- Docker: https://docs.docker.com/engine/install/ubuntu/
- kind: https://kind.sigs.k8s.io/docs/user/quick-start/#installation
- kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
- Helm: https://helm.sh/docs/intro/install/
- Argo CD CLI: https://argo-cd.readthedocs.io/en/stable/cli_installation/

```bash
# --- 🐳 Docker ---
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

printf 'deb [arch=%s signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu %s stable\n' \
  "$(dpkg --print-architecture)" "$(. /etc/os-release && echo "$VERSION_CODENAME")" \
  | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker "$USER"
newgrp docker <<'DOCKERCHECK'
docker version --format 'Docker Engine: {{.Server.Version}}'
DOCKERCHECK
```

> **🔧 Troubleshoot this step:**
> - **Error seen:** `E: Malformed entry 1 in list file /etc/apt/sources.list.d/docker.list` — the file contains a literal backslash instead of a newline.
> - **Recovery:** Run `cat /etc/apt/sources.list.d/docker.list` — the file must be one unbroken line; delete it with `sudo rm /etc/apt/sources.list.d/docker.list` and re-run the `printf | sudo tee` command above.
> - **Reference:** https://docs.docker.com/engine/install/ubuntu/

```bash
# --- 🎡 kind ---
curl -fsSL -o /tmp/kind \
  "https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64"
# Official guide: https://kind.sigs.k8s.io/docs/user/quick-start/#installation
sudo install -o root -g root -m 0755 /tmp/kind /usr/local/bin/kind
kind version

# --- ☸️ kubectl ---
KUBE_VERSION="$(curl -fsSL https://dl.k8s.io/release/stable.txt)"
# Official guide: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
curl -fsSL -o /tmp/kubectl \
  "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 /tmp/kubectl /usr/local/bin/kubectl
kubectl version --client

# --- ⎈ Helm ---
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
# Official guide: https://helm.sh/docs/intro/install/
helm version

# --- 🔧 openssl and jq (already packaged) ---
sudo apt-get install -y openssl jq
openssl version
jq --version

# --- 🚀 Argo CD CLI ---
ARGOCD_VERSION="$(curl -fsSL https://api.github.com/repos/argoproj/argo-cd/releases/latest \
  | jq -r '.tag_name')"
# Official guide: https://argo-cd.readthedocs.io/en/stable/cli_installation/
curl -fsSL -o /tmp/argocd \
  "https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64"
sudo install -o root -g root -m 0755 /tmp/argocd /usr/local/bin/argocd
argocd version --client
```

> **🔧 Troubleshoot this step:**
> - **Error seen:** `curl: (22) The requested URL returned error: 404` when downloading the Argo CD CLI — the computed version tag may not match the release asset filename.
> - **Recovery:** Run `echo $ARGOCD_VERSION` to confirm the tag looks like `v2.x.y`; then browse https://github.com/argoproj/argo-cd/releases/latest in a browser to confirm the exact asset name.
> - **Reference:** https://argo-cd.readthedocs.io/en/stable/cli_installation/

### 📝 Requirement 2

Create a kind cluster named `argocd-lab` that maps host ports 80 and 443 to the control-plane node, then deploy the NGINX Ingress Controller into it and confirm the controller pod reaches `Running` status. Finally, install Argo CD into a namespace named `argocd` using the upstream stable manifest and confirm all pods in that namespace reach `Running` status before proceeding.

```bash
# --- 🎡 kind cluster with port mappings ---
cat > /tmp/kind-cluster.yaml <<'EOF'
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
EOF

kind create cluster --name argocd-lab --config /tmp/kind-cluster.yaml
kubectl cluster-info --context kind-argocd-lab
kubectl get nodes
```

> **🔧 Troubleshoot this step:**
> - **Error seen:** `ERROR: failed to create cluster: node(s) already exist for a cluster with the name "argocd-lab"` — a previous run left a partial cluster.
> - **Recovery:** Run `kind delete cluster --name argocd-lab` and then re-run the `kind create cluster` command.
> - **Reference:** https://kind.sigs.k8s.io/docs/user/quick-start/#creating-a-cluster

```bash
# --- 🌐 NGINX Ingress Controller ---
# Official guide: https://kind.sigs.k8s.io/docs/user/ingress/#ingress-nginx
kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

kubectl get pods -n ingress-nginx

# --- 🚀 Argo CD ---
# Official guide: https://argo-cd.readthedocs.io/en/stable/getting_started/
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
kubectl get pods -n argocd
```

> **🔧 Troubleshoot this step:**
> - **Error seen:** `error: timed out waiting for the condition on pods/argocd-repo-server-...` — image pulls are slow on the first run.
> - **Recovery:** Run `kubectl describe pod -n argocd -l app.kubernetes.io/name=argocd-repo-server` and look for `ErrImagePull`; if present, wait two minutes and re-run the `kubectl wait` command.
> - **Reference:** https://argo-cd.readthedocs.io/en/stable/getting_started/

**✅ Acceptance criteria:**

- [ ] `kubectl get nodes` shows the control-plane node in `Ready` status, and `kubectl get pods -n argocd` shows every pod with status `Running` and all containers ready (the `READY` column shows `1/1` or `2/2` for each row)
- [ ] `argocd version --client` prints a version string without error, and `helm version` prints a version string without error, confirming the full toolchain is operational

---

## 🔐 Task 2: Secure Argo CD with TLS and Expose It via HTTPS Ingress

> ⚠️ This task is a **design challenge** — no starter code is provided. Use the requirements and acceptance criteria below to design and validate your own implementation.

### 📝 Requirement 1

Design and implement a TLS-secured Argo CD deployment.

**Requirements:**
- Generate a self-signed certificate authority (CA) and a server certificate signed by that CA
- The certificate's Subject Alternative Names must cover both `argocd.local` and the in-cluster service DNS name `argocd-server.argocd.svc.cluster.local`
- Store the certificate and key in a Kubernetes TLS Secret named `argocd-server-tls` in the `argocd` namespace — Argo CD automatically reads this secret name to serve HTTPS
- Configure the Argo CD server to run in secure mode (TLS enabled, not insecure mode) by patching the `argocd-cmd-params-cm` ConfigMap
- Add `127.0.0.1 argocd.local` to `/etc/hosts` so the hostname resolves locally

**Design decision:** No specific `openssl` command sequence is prescribed — design your certificate generation pipeline to satisfy the SAN requirements above. Confirm the server is serving your certificate by connecting to it and inspecting the returned certificate's subject and SANs.

**✅ Acceptance criteria:**

- [ ] `kubectl get secret argocd-server-tls -n argocd -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text` outputs a certificate whose Subject Alternative Name field includes both `argocd.local` and `argocd-server.argocd.svc.cluster.local`
- [ ] `curl -k -o /dev/null -w "%{http_code}" https://argocd.local` returns `200` or `307`, confirming the HTTPS endpoint is live and the Ingress is routing traffic to the Argo CD server

### 📝 Requirement 2

**Requirements:**
- Create an NGINX Ingress resource in the `argocd` namespace that terminates TLS using the `argocd-server-tls` secret
- Enforce HTTPS-only access (HTTP requests must redirect to HTTPS)
- Proxy traffic to the `argocd-server` Service on port 443 using the HTTPS backend protocol annotation required by NGINX Ingress when the upstream itself speaks TLS
- Retrieve the Argo CD initial admin password from the `argocd-initial-admin-secret` Secret
- Use the `argocd` CLI to log in against `https://argocd.local`, accepting the self-signed certificate, to confirm end-to-end HTTPS access works

**✅ Acceptance criteria:**

- [ ] `kubectl get ingress argocd-server-ingress -n argocd -o jsonpath='{.spec.tls[0].hosts[0]}'` returns `argocd.local`, confirming TLS is configured on the Ingress resource
- [ ] `argocd login argocd.local --username admin --password <retrieved-password> --insecure` exits with code `0` and prints `'admin' logged in successfully`, confirming authenticated HTTPS access through the Ingress

---

## 🔑 Task 3: Integrate Keycloak SSO and Enforce RBAC

> ⚠️ This task is a **design challenge** — no starter code is provided. Use the requirements and acceptance criteria below to design and validate your own implementation.

### 📝 Requirement 1

Design and deploy a Keycloak instance (OpenID Connect identity provider — a server that issues identity tokens after authenticating users) into a namespace named `keycloak` inside the same cluster. Expose it via an HTTP Ingress at `keycloak.local` (add this to `/etc/hosts`).

**Requirements — using Keycloak's REST Admin API, not the UI, programmatically:**
- Create a realm named `argocd`
- Create an OIDC client with client ID `argocd`, a fixed client secret of your choice, and a redirect URI pointing to `https://argocd.local/auth/callback`
- Create two groups: `argocd-admins` and `argocd-readonly`
- Create two users: one named `sso-admin` assigned to `argocd-admins`, and one named `sso-viewer` assigned to `argocd-readonly`; both must have non-temporary passwords set via the API
- Confirm the realm's OIDC discovery document is reachable and contains the correct issuer URL before proceeding

**✅ Acceptance criteria:**

- [ ] `curl -fsSL http://keycloak.local/realms/argocd/.well-known/openid-configuration | jq -r '.issuer'` returns `http://keycloak.local/realms/argocd` without error
- [ ] A direct Resource Owner Password Credentials grant request to Keycloak's token endpoint for `sso-admin` with the correct password returns a JSON response containing a non-null `access_token` field, confirming the user exists and can authenticate

### 📝 Requirement 2

**Requirements:**
- Configure Argo CD to delegate all authentication to Keycloak by updating the `argocd-cm` ConfigMap with a valid `oidc.config` block pointing to the `argocd` realm issuer, the client ID, and the client secret you set
- Configure the `argocd-rbac-cm` ConfigMap to define the following policy:
  - Members of `argocd-admins` receive the built-in `role:admin` permission set
  - Members of `argocd-readonly` receive the built-in `role:readonly` permission set
  - The default policy for any authenticated user not in either group is `role:readonly`
- Restart the Argo CD `server` and `dex-server` Deployments to apply the configuration
- Validate the RBAC mapping by using the `argocd` CLI's `account can-i` subcommand or by inspecting the decoded JWT `groups` claim from a token issued to `sso-admin` and confirming the group membership is present

**✅ Acceptance criteria:**

- [ ] `kubectl get configmap argocd-cm -n argocd -o jsonpath='{.data.oidc\.config}'` returns a non-empty string containing the Keycloak issuer URL, confirming OIDC is configured in Argo CD
- [ ] `kubectl get configmap argocd-rbac-cm -n argocd -o jsonpath='{.data.policy\.csv}'` returns a non-empty string containing both `argocd-admins` mapped to `role:admin` and `argocd-readonly` mapped to `role:readonly`, confirming RBAC policy is in place

---

## 🎯 Expected Outcomes

Argo CD is accessible exclusively over HTTPS at `argocd.local`, presenting a self-signed certificate with correct SANs, and all unauthenticated HTTP requests are redirected to HTTPS.

Authentication is handled by Keycloak SSO via OIDC, with group membership in Keycloak determining admin or read-only access in Argo CD according to the RBAC policy.

---

## 🛡️ MITRE ATT&CK Mapping

| Technique ID | Technique Name | Relevance to This Lab |
|---|---|---|
| T1557 | Adversary-in-the-Middle | TLS termination with a properly-SAN'd certificate and HTTPS-only redirect prevents interception/manipulation of traffic to the Argo CD API and UI |
| T1078 | Valid Accounts | SSO delegation to Keycloak centralizes account issuance and authentication, replacing locally-managed Argo CD credentials |
| T1556 | Modify Authentication Process | Configuring `argocd-cm` to delegate authentication to an external OIDC provider is itself a change to the authentication process that must be deliberate and auditable |
| T1552.001 | Unsecured Credentials: Credentials in Files | The OIDC client secret and initial admin password must be handled and stored (ConfigMap/Secret) carefully rather than left in plaintext files or shell history |
| T1098 | Account Manipulation | Group-to-role mapping in `argocd-rbac-cm` governs how access levels are granted or escalated as Keycloak group membership changes |

---

## 📚 Key Concepts

| Concept | Description |
|---------|-------------|
| **TLS Secret** | A Kubernetes Secret of type `kubernetes.io/tls` holding a certificate/key pair; Argo CD reads `argocd-server-tls` by convention to serve HTTPS |
| **Subject Alternative Name (SAN)** | Certificate field listing every hostname the certificate is valid for — both the external ingress hostname and the in-cluster service DNS name must be present |
| **NGINX Ingress TLS Termination** | The Ingress controller decrypts HTTPS traffic and can re-encrypt it to the backend when the backend-protocol annotation indicates the upstream itself speaks TLS |
| **OpenID Connect (OIDC)** | An identity layer on top of OAuth 2.0 that lets Argo CD delegate authentication to an external provider and receive identity/group claims in a token |
| **Keycloak Realm** | An isolated tenant in Keycloak containing its own clients, groups, users, and roles — configured here entirely through the REST Admin API |
| **RBAC policy.csv** | Argo CD's `argocd-rbac-cm` policy format mapping groups or subjects to built-in or custom roles (`role:admin`, `role:readonly`) |
| **Resource Owner Password Credentials Grant** | An OAuth 2.0 flow used here to directly validate that a Keycloak user was created correctly by exchanging a username/password for an access token |

---

## ✅ Conclusion

This lab required you to independently design a secure Argo CD deployment covering three distinct security layers: transport security via TLS, identity federation via OIDC, and authorization via RBAC. Each layer depends on the previous one being correctly configured, which reflects how production GitOps platforms are hardened. The skills applied here — certificate generation, OIDC client configuration, and policy-as-code RBAC — transfer directly to securing any Kubernetes-native platform that supports these standards.

---

<div align="center">

![Al Nafi](https://img.shields.io/badge/Al_Nafi-Cybersecurity_Training-blue?style=for-the-badge)

</div>
