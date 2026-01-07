# Spacelift Self-Hosted Installation Checklist

> [!NOTE]
> **Link Legend:**
> 🚀 **Spacelift Documentation** (Official guides)
> 🔗 **External Documentation** (AWS, GCP, Azure, etc.)

---

## 📋 Installation Tracker

Use this checklist to track your progress. Click the links to jump to detailed instructions for each step.

### **Phase 1: Pre-requisites**
- [ ] [**Obtain License Key & Materials**](#obtain-license-key-and-installation-materials)
- [ ] [**Choose Hostnames**](#choose-hostnames)
- [ ] [**Prepare TLS Certificate**](#prepare-tls-certificate)

### **Phase 2: External Dependencies**
- [ ] [**PostgreSQL Database**](#postgresql-database-required) *(Required)*
- [ ] [**Object Storage**](#object-storage-required) *(Required)*
- [ ] [**Encryption**](#encryption-configurable) *(Configurable)*
- [ ] [**Message Queue**](#message-queue-configurable) *(Configurable)*
- [ ] [**MQTT Broker**](#mqtt-broker-configurable) *(Configurable)*

### **Phase 3: Infrastructure Deployment**
- [ ] [**Configure Networking**](#configure-networking)
- [ ] [**Deploy Infrastructure via Terraform**](#deploy-infrastructure-via-terraform)
- [ ] [**Setup Container Registry & Push Images**](#setup-container-registry-and-push-images)

### **Phase 4: Deploy Spacelift Application**
- [ ] [**Kubernetes Deployments**](#kubernetes-deployments-eksgkeakson-prem) (EKS / GKE / AKS / On-Prem)
  - [ ] Configure `kubectl` credentials
  - [ ] Create Kubernetes namespace
  - [ ] Install Ingress Controller
  - [ ] Install cert-manager *(if using Let's Encrypt)*
  - [ ] Create ClusterIssuer *(for Let's Encrypt)*
  - [ ] Create Kubernetes secrets
  - [ ] Deploy via Helm chart
- [ ] [**ECS Deployment**](#ecs-deployment) *(if applicable)*

### **Phase 5: DNS Configuration**
- [ ] [**Configure DNS Records**](#phase-5-dns-configuration)

### **Phase 6: First Setup**
- [ ] [**Login with Admin Credentials**](#login-with-admin-credentials)
- [ ] [**Create Account**](#create-account)
- [ ] [**Configure SSO/IdP**](#configure-ssoidp-optional) *(Optional)*
- [ ] [**Setup VCS Integration**](#setup-vcs-integration)
- [ ] [**Create Worker Pool**](#create-worker-pool)
- [ ] [**Disable Admin Login**](#disable-admin-login-highly-recommended) *(Recommended)*

### **Phase 7: Validation**
- [ ] [**Verify Spacelift UI Loads**](#phase-7-validation)
- [ ] [**Verify Workers Connect**](#phase-7-validation)
- [ ] [**Create Test Stack**](#phase-7-validation)
- [ ] [**Trigger a Run**](#phase-7-validation)

### **Phase 8: Optional Integrations**
<details>
<summary>Click to expand optional integrations</summary>

- [ ] [Observability](#observability)
- [ ] [Telemetry](#telemetry)
- [ ] [Slack Integration](#slack-integration)
- [ ] [Cloud Integrations (OIDC)](#cloud-integrations-oidc)
- [ ] [Usage Reporting](#usage-reporting)
- [ ] [Disaster Recovery](#disaster-recovery)
</details>

---

## Phase 1: Pre-requisites

### Obtain License Key and Installation Materials

Contact your Spacelift sales representative to receive:

1.  **License token**
    Required environment variable (`LICENSE_TOKEN`) for Spacelift to start.
2.  **Installation archive** (e.g., `self-hosted-v3.0.0.tar.gz`)
    Contains:
    *   `spacelift-backend` container image
    *   `spacelift-launcher` container image
    *   `spacelift-launcher` binary (for non-Kubernetes workers)

[⬆️ Back to Checklist](#-installation-tracker)

---

### Choose Hostnames

<a href="https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/environment-requirements#choose-your-hostnames">
  <img src="https://img.shields.io/badge/Docs-Environment_Requirements-blue?style=flat-square&logo=spacelift" alt="Docs" />
</a>

Spacelift requires two hostnames to operate:

| Hostname | Example | Purpose | Who Needs Access |
| :--- | :--- | :--- | :--- |
| **Server hostname** | `spacelift.example.com` | Web UI, API (GraphQL/REST), VCS webhooks | Users, VCS systems, Workers |
| **MQTT broker hostname** | `mqtt.spacelift.example.com` | Worker-to-server communication | Workers only |

> [!TIP]
> **MQTT Hostname Notes**
> *   Only required if running workers **outside** the Spacelift cluster.
> *   If workers run in the same Kubernetes cluster, use internal K8s DNS (e.g., `spacelift-mqtt.spacelift.svc.cluster.local`).
> *   Default port: `1984` (configurable).

[⬆️ Back to Checklist](#-installation-tracker)

---

### Prepare TLS Certificate

<a href="https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks#server-certificate">
  <img src="https://img.shields.io/badge/EKS-Server_Certificate-orange?style=flat-square&logo=amazon-aws" />
</a>
<a href="https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-ecs#server-certificate">
  <img src="https://img.shields.io/badge/ECS-Server_Certificate-orange?style=flat-square&logo=amazon-aws" />
</a>

The TLS certificate secures HTTPS access to the Spacelift server and must be valid for your **server hostname**.

**Protects:**
*   User access to Web UI
*   API endpoints
*   Inbound VCS webhooks
*   Worker HTTP requests

**Requirements:**
*   Valid for server hostname
*   **Issued** status before deployment (ACM)
*   Trusted by browsers and VCS systems

<details open>
<summary><strong>Platform Recommendations</strong></summary>

| Platform | Recommended Approach | Docs |
| :--- | :--- | :--- |
| **AWS (EKS/ECS)** | [AWS Certificate Manager (ACM)](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html) 🔗<br>*(Provision before deployment)* | [EKS guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks#server-certificate) 🚀 |
| **GKE** | [Let's Encrypt via cert-manager](https://cert-manager.io/docs/configuration/acme/) 🔗<br>*(Auto-provisioned)* | [GKE setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke#cert-manager) 🚀 |
| **AKS** | [Let's Encrypt via cert-manager](https://cert-manager.io/docs/configuration/acme/) 🔗<br>*(Auto-provisioned)* | [AKS setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks#cert-manager) 🚀 |
| **On-prem** | Bring your own certificate OR [cert-manager](https://cert-manager.io/) 🔗 | [On-prem guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-onprem) 🚀 |

</details>

[⬆️ Back to Checklist](#-installation-tracker)

---

## Phase 2: External Dependencies

[External Dependencies Overview](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/external-dependencies) 🚀

### PostgreSQL Database (Required)

Spacelift stores all application data (accounts, stacks, history, secrets) in PostgreSQL.

**Requirements:**
*   **Version:** 14, 15, or 16 (v17+ not officially tested)
*   **Access:** Dedicated DB with full admin permissions
*   **Network:** Accessible from Server, Drain, and Scheduler components
*   **Port:** `5432` (default)

<details>
<summary><strong>📚 Supported Versions Table</strong></summary>

| Version | Status |
| :--- | :--- |
| PostgreSQL <= 13 | ❌ Not supported |
| PostgreSQL 14.x | ✅ Supported |
| PostgreSQL 15.x | ✅ Supported |
| PostgreSQL 16.x | ✅ Supported |
| PostgreSQL >= 17 | ⚠️ Not officially tested |

</details>

<details open>
<summary><strong>Managed Database Options</strong></summary>

| Platform | Managed Service | Docs |
| :--- | :--- | :--- |
| **AWS** | [Amazon RDS for PostgreSQL](https://aws.amazon.com/rds/postgresql/) 🔗 | [RDS Docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html) 🔗 |
| **AWS** | [Amazon Aurora PostgreSQL](https://aws.amazon.com/rds/aurora/) 🔗 | [Aurora Docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html) 🔗 |
| **GCP** | [Google Cloud SQL for PostgreSQL](https://cloud.google.com/sql/docs/postgres) 🔗 | [Cloud SQL Docs](https://cloud.google.com/sql/docs/postgres/create-instance) 🔗 |
| **Azure** | [Azure DB for PostgreSQL Flexible Server](https://azure.microsoft.com/en-us/products/postgresql) 🔗 | [Azure Docs](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview) 🔗 |

</details>

> [!NOTE]
> **Terraform Module Defaults**
> *   **EKS/ECS:** Creates Aurora Serverless (`create_database = true`)
> *   **GKE:** Creates Cloud SQL (`enable_database = true`)
> *   **AKS:** Creates PostgreSQL Flexible Server
> *   *Set to `false` to bring your own database.*

[⬆️ Back to Checklist](#-installation-tracker)

---

### Object Storage (Required)

[Object Storage Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage) 🚀

Used to persist logs, state files, policy inputs, workspace data, artifacts, and webhooks. **Multiple buckets are created automatically by the Terraform modules.**

| Provider | Authentication Method | Docs |
| :--- | :--- | :--- |
| **Amazon S3** | IAM roles, access keys, instance profiles | [Configuration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage#amazon-s3) 🚀 |
| **GCS** | Workload Identity, service account keys | [Configuration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage#google-cloud-storage) 🚀 |
| **Azure Blob** | Workload Identity, managed identity | [Configuration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage#azure-blob-storage) 🚀 |
| **MinIO** | S3-compatible API with access keys | [Configuration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage#minio) 🚀 |

<details>
<summary><strong>View created buckets list</strong></summary>

*   `deliveries` (Webhooks)
*   `large-queue-messages`
*   `metadata` (Run metadata)
*   `modules` (Private registry)
*   `policy-inputs`
*   `run-logs`
*   `states` (Terraform state)
*   `uploads`
*   `user-uploaded-workspaces`
*   `workspace`
</details>

[⬆️ Back to Checklist](#-installation-tracker)

---

### Encryption (Configurable)

[Encryption Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/encryption) 🚀

Encrypts sensitive data (secrets, tokens, keys) in the database and signs JWT tokens.

> [!WARNING]
> **CRITICAL DECISION**
> Once you select an encryption method and deploy, **you cannot migrate** to a different method without re-encrypting all data manually (not supported). Choose carefully.

| Option | Best For | How It Works |
| :--- | :--- | :--- |
| **AWS KMS** | AWS (EKS/ECS) | Uses KMS keys; keys never leave AWS. Terraform creates 2 keys (encrypt & sign). |
| **RSA (built-in)** | GKE, AKS, On-prem | You generate a 4096-bit RSA private key and pass it via env var. |

**Setup Details:**

*   **AWS KMS:** Requires `kms:Encrypt`, `kms:Decrypt`, `kms:Sign`, `kms:Verify` permissions.
*   **RSA:** Generate with:
    ```bash
    openssl genrsa -out spacelift.key 4096 && base64 -w0 spacelift.key
    ```
    Pass as `ENCRYPTION_RSA_PRIVATE_KEY`. **Keep this key secure.**

[⬆️ Back to Checklist](#-installation-tracker)

---

### Message Queue (Configurable)

[Message Queues Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/message-queues) 🚀

Handles async job processing (webhooks, scheduling, policies).

| Option | Best For | Configuration |
| :--- | :--- | :--- |
| **Built-in (PostgreSQL)** | ✅ **Recommended** for new installs. No extra infra. | `MESSAGE_QUEUE_TYPE=builtin` (default) |
| **AWS SQS** | Existing installs or managed preference. Required for IoT Core. | `MESSAGE_QUEUE_TYPE=sqs` + queue URLs |

[⬆️ Back to Checklist](#-installation-tracker)

---

### MQTT Broker (Configurable)

[MQTT Broker Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/mqtt-broker) 🚀

Enables "push" communication from server to workers (scheduling, cancellations, heartbeats).

| Option | Best For | Configuration |
| :--- | :--- | :--- |
| **Built-in** | ✅ **Recommended**. Runs on port `1984` inside server. | `MQTT_BROKER_TYPE=builtin` (default) |
| **AWS IoT Core** | Large-scale AWS deployments. **Requires SQS**. | `MQTT_BROKER_TYPE=iot-core` + endpoint |

[⬆️ Back to Checklist](#-installation-tracker)

---

## Phase 3: Infrastructure Deployment

### Configure Networking

[Networking Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/networking) 🚀

Spacelift is composed of **Server**, **Drain**, **Scheduler**, and **Workers**.

<details open>
<summary><strong>Port Requirements (Server Ingress)</strong></summary>

| Port | Protocol | Purpose |
| :--- | :--- | :--- |
| **1983** | TCP/HTTP | Web UI, API, Webhooks |
| **1984** | TCP/MQTT | Worker communication (Built-in MQTT) |

</details>

<details>
<summary><strong>Egress Requirements (Outbound)</strong></summary>

**Server Egress:**
*   **PostgreSQL:** 5432 (Required)
*   **Object Storage:** 443 (Required)
*   **VCS System:** 443 (Required)
*   **AWS Services (SQS/IoT/KMS):** 443 (If used)

**Worker Egress:**
*   **Spacelift Server:** 443 (Required)
*   **MQTT Broker:** 1984 or 443 (Required)
*   **Object Storage & VCS:** 443 (Required)
*   **Infra APIs:** 443 (AWS/GCP/Azure)

</details>

**Checklist for Deployment:**
- [ ] VPC/VNet with public & private subnets
- [ ] Internet Gateway (Ingress to LB)
- [ ] NAT Gateway (Egress from private subnets)
- [ ] Application Load Balancer (HTTPS Ingress)
- [ ] Network Load Balancer (MQTT, if external workers)
- [ ] Security Groups / Firewall Rules

[⬆️ Back to Checklist](#-installation-tracker)

---

### Deploy Infrastructure via Terraform

Use the official Terraform modules to deploy clusters, databases, storage, keys, and networking.

| Platform | Terraform Module | Deployment Guide |
| :--- | :--- | :--- |
| **AWS EKS** | [terraform-aws-eks-spacelift-selfhosted](https://github.com/spacelift-io/terraform-aws-eks-spacelift-selfhosted) | [EKS Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks) 🚀 |
| **AWS ECS** | [terraform-aws-spacelift-selfhosted](https://github.com/spacelift-io/terraform-aws-spacelift-selfhosted) | [ECS Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-ecs) 🚀 |
| **GKE** | [terraform-google-spacelift-selfhosted](https://github.com/spacelift-io/terraform-google-spacelift-selfhosted) | [GKE Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke) 🚀 |
| **AKS** | [terraform-azure-spacelift-selfhosted](https://github.com/spacelift-io/terraform-azure-spacelift-selfhosted) | [AKS Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks) 🚀 |

**Look for these Terraform Outputs:**
*   `shell`: Env vars to source for next steps
*   `kubernetes_secrets`: YAML for K8s secrets
*   `helm_values`: `values.yaml` for Helm
*   Connection strings, bucket names, ARNs

[⬆️ Back to Checklist](#-installation-tracker)

---

### Setup Container Registry and Push Images

[Container Registries Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/networking#container-registries) 🚀

You need to push two images from your installation archive to your private registry.

**Images:**
1.  `spacelift-backend`: Runs server/drain/scheduler.
2.  `spacelift-launcher`: Runs worker pods.

**Process:**
```bash
# 1. Extract archive
tar -xzf self-hosted-v3.0.0.tar.gz

# 2. Load & Tag
docker image load --input=container-images/spacelift-backend.tar
docker tag spacelift-backend:v3.0.0 your-registry/spacelift-backend:v3.0.0

# 3. Push
docker push your-registry/spacelift-backend:v3.0.0
# Repeat for spacelift-launcher
```

<details>
<summary><strong>Registry Login Commands</strong></summary>

*   **AWS ECR:** `aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URL`
*   **GCP Artifact Registry:** `gcloud auth configure-docker $REGION-docker.pkg.dev`
*   **Azure ACR:** `az acr login --name $REGISTRY_NAME`

</details>

[⬆️ Back to Checklist](#-installation-tracker)

---

## Phase 4: Deploy Spacelift Application

### Kubernetes Deployments (EKS/GKE/AKS/On-Prem)

Follow these steps sequentially.

#### 1. Configure kubectl credentials

<details>
<summary>Show commands</summary>

```bash
# EKS
aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION

# GKE
gcloud container clusters get-credentials $GKE_CLUSTER_NAME --location $GCP_LOCATION --project $GCP_PROJECT

# AKS
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME
```
</details>

#### 2. Create Namespace
```bash
kubectl create namespace spacelift
```

#### 3. Install Ingress Controller
*   **EKS:** AWS Load Balancer Controller (often installed by EKS Auto).
*   **GKE:** Built-in Gateway API.
*   **AKS/On-Prem:** NGINX Ingress Controller.

#### 4. Install cert-manager (If using Let's Encrypt)
*Required for GKE, AKS, and On-prem.*

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.16.2 --set crds.enabled=true
```

#### 5. Create ClusterIssuer
*Create if using cert-manager.*

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: prod-issuer-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

#### 6. Create Kubernetes Secrets
Apply the secrets generated by Terraform.

```bash
tofu output -raw kubernetes_secrets | kubectl apply -f -
```

#### 7. Deploy via Helm
[Spacelift Helm Chart](https://downloads.spacelift.io/helm) 🚀

```bash
# Generate values
tofu output -raw helm_values > spacelift-values.yaml

# Deploy
helm upgrade --repo https://downloads.spacelift.io/helm \
  spacelift spacelift-self-hosted \
  --install --wait --timeout 20m \
  --namespace spacelift \
  --values spacelift-values.yaml

# Monitor
kubectl logs -n spacelift deployments/spacelift-server -f
```

[⬆️ Back to Checklist](#-installation-tracker)

---

### ECS Deployment

[Deploying to ECS Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-ecs) 🚀

ECS services are deployed via Terraform, not Helm.

1.  Deploy **Infra** module (`terraform-aws-spacelift-selfhosted`).
2.  Push images to ECR.
3.  Deploy **Services** module (`terraform-aws-ecs-spacelift-selfhosted`).

```bash
# Enable service deployment
export TF_VAR_deploy_services=true
tofu apply
```

[⬆️ Back to Checklist](#-installation-tracker)

---

## Phase 5: DNS Configuration

Create DNS records pointing to your new Load Balancers.

**1. Get Load Balancer Address:**
```bash
# Server (HTTP)
kubectl get ingress -n spacelift

# MQTT (if external)
kubectl get service spacelift-mqtt -n spacelift
```

**2. Create Records:**

| Record | Type | Value | Purpose |
| :--- | :--- | :--- | :--- |
| `spacelift.example.com` | CNAME | *(LB Hostname)* | Server (UI/API) |
| `mqtt.spacelift.example.com` | CNAME | *(LB Hostname)* | MQTT (External workers) |

> **Note for GKE:** You may need A (IPv4) and AAAA (IPv6) records if provided an IP instead of a hostname.

[⬆️ Back to Checklist](#-installation-tracker)

---

## Phase 6: First Setup

[First Setup Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/first-setup) 🚀

### Login with Admin Credentials
Navigate to `https://your-server-hostname`. You will be redirected to login.
*   **Username:** `ADMIN_USERNAME` (from Terraform)
*   **Password:** `ADMIN_PASSWORD` (from Terraform)

### Create Account
Choose your account name (Subdomain/Organization name).
*   **Constraint:** 2-38 chars, letters/numbers/`_`/`-`.
*   **Warning:** Changing this later is difficult (breaks integrations).

### Configure SSO/IdP (Optional)
**Settings > Single Sign-On**.
Recommended: Use **User Management** for UI-driven setup (SAML, OIDC, GitHub/GitLab OAuth). Use **Login Policy** for advanced code-based rules.

### Setup VCS Integration
Connect to GitHub, GitLab, Bitbucket, or Azure DevOps to allow Spacelift to access your infrastructure code.
[View Integration Guides 🚀](https://docs.spacelift.io/self-hosted/latest/integrations/source-control)

### Create Worker Pool
You need at least one worker pool to run stacks.
1.  **K8s:** Create `spacelift-workers` namespace.
2.  **UI:** Create Worker Pool, get credentials.
3.  **K8s:** Deploy WorkerPool CRD & Secret.

### Disable Admin Login (Highly Recommended)
Once SSO is working, **remove** `ADMIN_USERNAME` and `ADMIN_PASSWORD` variables from your deployment to close this security vector.

[⬆️ Back to Checklist](#-installation-tracker)

---

## Phase 7: Validation

- [ ] **Verify Spacelift UI loads** (`https://your-server-hostname`)
- [ ] **Verify workers connect** (UI shows "Idle" workers)
- [ ] **Create test stack** ([Guide](https://docs.spacelift.io/self-hosted/latest/getting-started/create-stack) 🚀)
- [ ] **Trigger a run** (Verify success)

[⬆️ Back to Checklist](#-installation-tracker)

---

## Phase 8: Optional Integrations

<details>
<summary><strong>Observability (Datadog / Prometheus)</strong></summary>

[Observability Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/observability) 🚀
Export metrics, traces, and logs to Datadog or Prometheus.
</details>

<details>
<summary><strong>Telemetry (Tracing)</strong></summary>

[Telemetry Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/telemetry) 🚀
Debug with distributed tracing via Datadog APM, AWS X-Ray, or OpenTelemetry.
</details>

<details>
<summary><strong>Cloud Integrations (OIDC)</strong></summary>

[Cloud Integrations](https://docs.spacelift.io/self-hosted/latest/integrations/cloud-providers) 🚀
Allow Spacelift to assume roles (AWS/GCP/Azure/Vault) without static credentials.
</details>

<details>
<summary><strong>Other Integrations</strong></summary>

*   [Slack Integration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/slack) 🚀
*   [Usage Reporting](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/usage-reporting) 🚀
*   [Disaster Recovery](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/disaster-recovery) 🚀
</details>

[⬆️ Back to Checklist](#-installation-tracker)

---

## Quick Reference Tables

<details>
<summary><strong>Platform-Specific Guides</strong></summary>

| Platform | Deployment Guide | Terraform Module |
| :--- | :--- | :--- |
| **AWS EKS** | [Deploying to EKS](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks) 🚀 | [terraform-aws-eks-spacelift-selfhosted](https://github.com/spacelift-io/terraform-aws-eks-spacelift-selfhosted) |
| **AWS ECS** | [Deploying to ECS](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-ecs) 🚀 | [terraform-aws-ecs-spacelift-selfhosted](https://github.com/spacelift-io/terraform-aws-ecs-spacelift-selfhosted) |
| **Google GKE** | [Deploying to GKE](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke) 🚀 | [terraform-google-spacelift-selfhosted](https://github.com/spacelift-io/terraform-google-spacelift-selfhosted) |
| **Azure AKS** | [Deploying to AKS](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks) 🚀 | [terraform-azure-spacelift-selfhosted](https://github.com/spacelift-io/terraform-azure-spacelift-selfhosted) |

</details>

<details>
<summary><strong>Configuration Reference</strong></summary>

| Topic | Reference |
| :--- | :--- |
| General configuration | [Docs](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/general-configuration) 🚀 |
| Encryption | [Docs](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/encryption) 🚀 |
| Message queues | [Docs](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/message-queues) 🚀 |
| Networking | [Docs](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/networking) 🚀 |
| Object storage | [Docs](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage) 🚀 |
</details>
