# Spacelift Self-Hosted Installation Checklist

## Table of Contents / Checklist

Use this table of contents to track your progress through the installation. Check off items as you complete them.

### Phase 1: Pre-requisites
- [ ] [Obtain License Key and Installation Materials](#obtain-license-key-and-installation-materials)
- [ ] [Choose Hostnames](#choose-hostnames)
- [ ] [Prepare TLS Certificate](#prepare-tls-certificate)

### Phase 2: External Dependencies
- [ ] [PostgreSQL Database (Required)](#postgresql-database-required)
- [ ] [Object Storage (Required)](#object-storage-required)
- [ ] [Encryption (Configurable)](#encryption-configurable)
- [ ] [Message Queue (Configurable)](#message-queue-configurable)
- [ ] [MQTT Broker (Configurable)](#mqtt-broker-configurable)

### Phase 3: Infrastructure Deployment
- [ ] [Configure Networking](#configure-networking)
- [ ] [Deploy Infrastructure via Terraform](#deploy-infrastructure-via-terraform)
- [ ] [Setup Container Registry and Push Images](#setup-container-registry-and-push-images)

### Phase 4: Deploy Spacelift Application
- [ ] [Kubernetes Deployments (EKS/GKE/AKS/On-Prem)](#kubernetes-deployments-eksgkeakson-prem)
  - [ ] [Configure kubectl credentials](#configure-kubectl-credentials)
  - [ ] [Create Kubernetes namespace](#create-kubernetes-namespace)
  - [ ] [Install ingress controller](#install-ingress-controller)
  - [ ] [Install cert-manager (if using Let's Encrypt)](#install-cert-manager-if-using-lets-encrypt)
  - [ ] [Create ClusterIssuer (for Let's Encrypt)](#create-clusterissuer-for-lets-encrypt)
  - [ ] [Create Kubernetes secrets](#create-kubernetes-secrets)
  - [ ] [Deploy via Helm chart](#deploy-via-helm-chart)
- [ ] [ECS Deployment](#ecs-deployment) *(if applicable)*

### Phase 5: DNS Configuration
- [ ] [Configure DNS Records](#phase-5-dns-configuration)

### Phase 6: First Setup
- [ ] [Login with admin credentials](#login-with-admin-credentials)
- [ ] [Create account](#create-account)
- [ ] [Configure SSO/IdP (optional)](#configure-ssoidp-optional)
- [ ] [Setup VCS integration](#setup-vcs-integration)
- [ ] [Create worker pool](#create-worker-pool)
- [ ] [Disable admin login (optional)](#disable-admin-login-optional)

### Phase 7: Validation
- [ ] [Verify Spacelift UI loads](#phase-7-validation)
- [ ] [Verify workers connect](#phase-7-validation)
- [ ] [Create test stack](#phase-7-validation)
- [ ] [Trigger a run](#phase-7-validation)

### Phase 8: Optional Integrations
- [ ] [Observability](#observability)
- [ ] [Telemetry](#telemetry)
- [ ] [Slack Integration](#slack-integration)
- [ ] [Cloud Integrations (OIDC)](#cloud-integrations-oidc)
- [ ] [Usage Reporting](#usage-reporting)
- [ ] [Disaster Recovery](#disaster-recovery)

---

## Phase 1: Pre-requisites

### Obtain License Key and Installation Materials

Contact your Spacelift sales representative to receive:

- **License token** - Required environment variable (`LICENSE_TOKEN`) for Spacelift to start
- **Installation archive** (e.g., `self-hosted-v3.0.0.tar.gz`) - Contains:
  - `spacelift-backend` container image
  - `spacelift-launcher` container image
  - `spacelift-launcher` binary (for non-Kubernetes workers)

---

### Choose Hostnames

[Environment Requirements: Choose Your Hostnames](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/environment-requirements#choose-your-hostnames)

Spacelift requires two hostnames to operate:

| Hostname | Example | Purpose | Who Needs Access |
|----------|---------|---------|------------------|
| **Server hostname** | `spacelift.example.com` | Spacelift web UI, GraphQL API, REST API, VCS webhooks | Users (browsers), VCS systems (GitHub/GitLab), workers |
| **MQTT broker hostname** | `mqtt.spacelift.example.com` | Worker-to-server communication for job scheduling | Workers only |

**MQTT hostname notes:**

- Only required if running workers **outside** the Spacelift cluster
- If workers run in the same Kubernetes cluster, you can use internal K8s DNS (e.g., `spacelift-mqtt.spacelift.svc.cluster.local`)
- The MQTT broker runs on a configurable port (default: `1984`)

---

### Prepare TLS Certificate

- [EKS: Server Certificate](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks#server-certificate)
- [ECS: Server Certificate](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-ecs#server-certificate)

The TLS certificate secures HTTPS access to the Spacelift server. It must be valid for your **server hostname** (e.g., `spacelift.example.com`).

**What the certificate protects:**

- User access to the Spacelift web UI
- GraphQL and REST API endpoints
- Inbound webhooks from VCS systems (e.g., GitHub push events)
- Worker HTTP requests to the server (e.g., state changes, log URL retrieval)

**Certificate requirements:**

- Must be valid for your server hostname
- Must be in *Issued* status before deployment (for ACM)
- Must be trusted by browsers and VCS systems

| Platform | Recommended Approach | Docs |
|----------|---------------------|------|
| AWS (EKS/ECS) | [AWS Certificate Manager (ACM)](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html) - provision before deployment, must be *Issued* | [EKS guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks#server-certificate) |
| GKE | [Let's Encrypt via cert-manager](https://cert-manager.io/docs/configuration/acme/) - auto-provisioned during deployment | [GKE cert-manager setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke#cert-manager) |
| AKS | [Let's Encrypt via cert-manager](https://cert-manager.io/docs/configuration/acme/) - auto-provisioned during deployment | [AKS cert-manager setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks#cert-manager) |
| On-prem | Bring your own certificate or use [cert-manager](https://cert-manager.io/) | [On-prem guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-onprem) |

---

## Phase 2: External Dependencies

[External Dependencies Overview](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/external-dependencies)

### PostgreSQL Database (Required)

[Database Requirements](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/external-dependencies#database)

Spacelift stores all application data in PostgreSQL, including:

- Account and user information
- Stack, context, and policy configurations
- Run history and state
- Encrypted secrets and environment variables

**Requirements:**

- PostgreSQL version 14, 15, or 16 (v17+ not officially supported)
- Dedicated database with full administrative permissions
- Network accessible from Spacelift server, drain, and scheduler components
- Default port: `5432` (configurable)

**Supported versions:**

| Version | Status |
|---------|--------|
| PostgreSQL <= 13 | Not supported |
| PostgreSQL 14.x | Supported |
| PostgreSQL 15.x | Supported |
| PostgreSQL 16.x | Supported |
| PostgreSQL >= 17 | Not officially tested |

**Managed database options by platform:**

| Platform | Managed Service | Docs |
|----------|----------------|------|
| AWS | [Amazon RDS for PostgreSQL](https://aws.amazon.com/rds/postgresql/) | [RDS PostgreSQL docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html) |
| AWS | [Amazon Aurora PostgreSQL](https://aws.amazon.com/rds/aurora/) (Serverless v2 supported) | [Aurora PostgreSQL docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html) |
| GCP | [Google Cloud SQL for PostgreSQL](https://cloud.google.com/sql/docs/postgres) | [Cloud SQL docs](https://cloud.google.com/sql/docs/postgres/create-instance) |
| Azure | [Azure Database for PostgreSQL Flexible Server](https://azure.microsoft.com/en-us/products/postgresql) | [Azure PostgreSQL docs](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview) |
| On-prem | Self-managed PostgreSQL | Not recommended to run in Kubernetes - use persistent volumes if you must |

**Terraform module behavior:**

- EKS/ECS modules create Aurora Serverless by default (`create_database = true`)
- GKE module creates Cloud SQL instance by default (`enable_database = true`)
- AKS module creates PostgreSQL Flexible Server by default
- Set to `false` to bring your own database

---

### Object Storage (Required)

[Object Storage Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage)

Spacelift requires object storage to persist files including:

- **Run logs** - stdout/stderr from Terraform, OpenTofu, Pulumi, etc.
- **Terraform state files** - when using Spacelift as a state backend
- **Policy inputs** - cached inputs for policy evaluation
- **Workspace data** - uploaded workspaces for runs
- **Module artifacts** - for the private module registry
- **Webhook deliveries** - stored for debugging/replay

**Multiple buckets are created** - the Terraform modules handle this automatically.

**Supported providers:**

| Provider | Authentication | Docs |
|----------|---------------|------|
| Amazon S3 | IAM roles (EKS/ECS), access keys, instance profiles | [S3 Configuration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage#amazon-s3) |
| Google Cloud Storage | Workload Identity, service account keys, Application Default Credentials | [GCS Configuration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage#google-cloud-storage) |
| Azure Blob Storage | Workload Identity, managed identity, connection strings | [Azure Blob Configuration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage#azure-blob-storage) |
| MinIO | S3-compatible API with access keys | [MinIO Configuration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage#minio) |

**Buckets created by Terraform modules:**

- `deliveries` - webhook delivery storage
- `large-queue-messages` - oversized queue messages
- `metadata` - run metadata
- `modules` - private module registry
- `policy-inputs` - policy evaluation inputs
- `run-logs` - run output logs
- `states` - Terraform state files
- `uploads` - user uploads
- `user-uploaded-workspaces` - workspace uploads
- `workspace` - workspace data

---

### Encryption (Configurable)

[Encryption Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/encryption)

Spacelift encrypts sensitive data stored in the database to prevent plaintext storage of secrets. Encryption is also used for signing JWT tokens for user sessions and API authentication.

**What gets encrypted:**

- Environment variables marked as secret
- Context values marked as secret
- Cloud integration credentials
- VCS integration tokens
- API keys

> **Critical decision - cannot be changed later:**
> Once you select an encryption method and deploy Spacelift, you **cannot migrate** to a different method. This would require re-encrypting all database entries, which is not supported.

| Option | Best For | How It Works | Docs |
|--------|----------|--------------|------|
| **AWS KMS** | AWS deployments (EKS/ECS) | Uses KMS keys for encryption and signing; keys never leave AWS | [KMS setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/encryption#kms) |
| **RSA (built-in)** | GKE, AKS, on-prem | You generate an RSA private key and pass it via environment variable | [RSA key generation](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/encryption#rsa) |

**KMS (AWS) details:**

- Two KMS keys are created: one for encryption, one for signing
- Terraform modules create these automatically
- Keys are referenced by ARN in configuration
- Spacelift components need `kms:Encrypt`, `kms:Decrypt`, `kms:Sign`, `kms:Verify` permissions

**RSA (built-in) details:**

- You generate a 4096-bit RSA private key
- Key is base64-encoded and passed as `ENCRYPTION_RSA_PRIVATE_KEY` environment variable
- Key must be kept secure - if compromised, all secrets are exposed
- Generate with: `openssl genrsa -out spacelift.key 4096 && base64 -w0 spacelift.key`

---

### Message Queue (Configurable)

[Message Queues Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/message-queues)

Spacelift uses message queues for asynchronous job processing:

- Processing inbound VCS webhooks (e.g., GitHub push events)
- Scheduling runs
- Triggering policy evaluations
- Background cleanup tasks

| Option | Best For | How It Works | Docs |
|--------|----------|--------------|------|
| **Built-in (PostgreSQL-based)** | **Recommended for all new installations** | Uses PostgreSQL tables as a queue; no additional infrastructure | [Built-in setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/message-queues#built-in) |
| **AWS SQS** | Existing installations, or if you prefer managed queues | Creates SQS queues for each message type | [SQS setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/message-queues#sqs) |

**Built-in queue details:**

- Default for new installations
- No additional AWS/GCP/Azure resources needed
- Uses PostgreSQL for durability
- Configuration: `MESSAGE_QUEUE_TYPE=builtin` (or omit - it's the default)

**SQS queue details:**

- Multiple queues are created automatically by Terraform modules
- Required if using AWS IoT Core for MQTT
- Configuration: `MESSAGE_QUEUE_TYPE=sqs` plus queue URL environment variables
- IAM permissions needed: `sqs:SendMessage`, `sqs:ReceiveMessage`, `sqs:DeleteMessage`

---

### MQTT Broker (Configurable)

[MQTT Broker Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/mqtt-broker)

The MQTT broker enables server-to-worker communication. It's used for:

- Broadcasting job scheduling messages to workers
- Notifying workers of run cancellations
- Worker heartbeat monitoring

Workers connect to the MQTT broker to receive real-time messages from the server. This is why MQTT uses a "push" model rather than polling.

| Option | Best For | How It Works | Docs |
|--------|----------|--------------|------|
| **Built-in** | **Recommended for most deployments** | MQTT broker runs embedded in the Spacelift server process | [Built-in MQTT setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/mqtt-broker#built-in) |
| **AWS IoT Core** | Large-scale AWS deployments needing managed MQTT | Uses AWS IoT Core as external broker | [IoT Core setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/mqtt-broker#iot-core) |

**Built-in broker details:**

- Runs on port `1984` by default (configurable)
- Exposed via Network Load Balancer (for external workers) or ClusterIP Service (for in-cluster workers)
- No TLS required for in-cluster communication; TLS recommended for external workers
- Configuration: `MQTT_BROKER_TYPE=builtin` (or omit - it's the default)

**IoT Core details:**

- **Requires AWS SQS** - cannot use built-in message queue with IoT Core
- Terraform modules create IoT Core resources automatically
- Workers authenticate using IoT credentials
- Configuration: `MQTT_BROKER_TYPE=iot-core` plus IoT endpoint configuration

---

## Phase 3: Infrastructure Deployment

[Environment Requirements](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/environment-requirements)

### Configure Networking

[Networking Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/networking)

Spacelift consists of three main backend services plus workers. Each has specific networking requirements.

**Spacelift Backend Components:**

| Component | Role | Ingress Required | Egress Required |
|-----------|------|------------------|-----------------|
| **Server** | Web UI, API, MQTT broker | Yes - HTTP (1983), MQTT (1984) | Database, object storage, VCS, KMS (if used) |
| **Drain** | Async job processing | No | Database, object storage, VCS, KMS (if used) |
| **Scheduler** | Cron job triggering | No | Database, message queue |
| **Workers** | Run execution | No | Server, MQTT broker, object storage, VCS, infrastructure APIs |

**Server ingress** - [Server Networking](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/networking#server):

| Port | Protocol | Purpose |
|------|----------|---------|
| 1983 | TCP/HTTP | Web UI, GraphQL API, REST API, VCS webhooks |
| 1984 | TCP/MQTT | Worker communication (built-in MQTT broker) |

**Server egress** - [Server Networking](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/networking#server):

| Destination | Port | Required |
|-------------|------|----------|
| PostgreSQL database | 5432 (default) | Required |
| Object storage (S3/GCS/Azure) | 443 | Required |
| VCS system (GitHub/GitLab/etc.) | 443 | Required |
| AWS SQS | 443 | Only if using SQS |
| AWS IoT Core | 443 | Only if using IoT Core |
| AWS KMS | 443 | Only if using KMS encryption |

**Worker egress** - [Worker Networking](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/networking#workers):

| Destination | Port | Required |
|-------------|------|----------|
| Spacelift server | 443 | Required |
| MQTT broker | 1984 (or 443 for IoT Core) | Required |
| Object storage | 443 | Required |
| VCS system | 443 | Required |
| Container registries | 443 | Required |
| Infrastructure APIs (AWS/GCP/Azure) | 443 | Required (for managed resources) |

**Network infrastructure to deploy:**

- [ ] VPC/VNet with public and private subnets
- [ ] Internet Gateway for inbound access to load balancer
- [ ] NAT Gateway for egress from private subnets
- [ ] Application Load Balancer for HTTPS ingress to server
- [ ] Network Load Balancer for MQTT (if external workers)
- [ ] Security groups/firewall rules per above requirements

---

### Deploy Infrastructure via Terraform

Each platform has a Terraform module that deploys:

- Container/Kubernetes cluster
- Database (optional - can bring your own)
- Object storage buckets
- Encryption keys (KMS or outputs for RSA)
- Networking (VPC, subnets, load balancers)
- IAM roles and permissions
- Container registry

| Platform | Terraform Module | What It Creates | Deployment Guide |
|----------|-----------------|-----------------|------------------|
| AWS EKS | [terraform-aws-eks-spacelift-selfhosted](https://github.com/spacelift-io/terraform-aws-eks-spacelift-selfhosted) | EKS cluster, Aurora PostgreSQL, S3 buckets, KMS keys, ECR, VPC, ALB/NLB | [EKS Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks) |
| AWS ECS | [terraform-aws-spacelift-selfhosted](https://github.com/spacelift-io/terraform-aws-spacelift-selfhosted) (infra) + [terraform-aws-ecs-spacelift-selfhosted](https://github.com/spacelift-io/terraform-aws-ecs-spacelift-selfhosted) (services) | ECS Fargate cluster, Aurora PostgreSQL, S3 buckets, KMS keys, ECR, VPC, ALB/NLB | [ECS Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-ecs) |
| Google GKE | [terraform-google-spacelift-selfhosted](https://github.com/spacelift-io/terraform-google-spacelift-selfhosted) | GKE Autopilot cluster, Cloud SQL PostgreSQL, GCS buckets, Artifact Registry, VPC | [GKE Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke) |
| Azure AKS | [terraform-azure-spacelift-selfhosted](https://github.com/spacelift-io/terraform-azure-spacelift-selfhosted) | AKS cluster, PostgreSQL Flexible Server, Azure Blob containers, Container Registry, VNet | [AKS Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks) |

**Common Terraform module outputs:**

- `shell` - environment variables to source for subsequent steps
- `kubernetes_secrets` - YAML to create K8s secrets
- `helm_values` - values.yaml for Helm deployment
- Database connection strings, bucket names, key ARNs, etc.

---

### Setup Container Registry and Push Images

[Container Registries](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/networking#container-registries)

Spacelift requires two container images from the installation archive:

| Image | Purpose | Required By |
|-------|---------|-------------|
| `spacelift-backend` | Runs server, drain, and scheduler services | Kubernetes/ECS cluster |
| `spacelift-launcher` | Worker container for Kubernetes workers | Kubernetes worker nodes |

**Steps:**

1. Extract the installation archive: `tar -xzf self-hosted-v3.0.0.tar.gz`
2. Load images into Docker: `docker image load --input=container-images/spacelift-backend.tar`
3. Tag with your registry URL: `docker tag spacelift-backend:v3.0.0 your-registry/spacelift-backend:v3.0.0`
4. Push to registry: `docker push your-registry/spacelift-backend:v3.0.0`
5. Repeat for `spacelift-launcher`

**Platform-specific registry setup:**

| Platform | Registry | Login Command | Push Docs |
|----------|----------|---------------|-----------|
| AWS | Amazon ECR | `aws ecr get-login-password --region $REGION \| docker login --username AWS --password-stdin $ECR_URL` | [EKS: Push Images](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks#push-images-to-elastic-container-registry) |
| GCP | Artifact Registry | `gcloud auth configure-docker $REGION-docker.pkg.dev` | [GKE: Push Images](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke#push-images-to-artifact-registry) |
| Azure | Azure Container Registry | `az acr login --name $REGISTRY_NAME` | [AKS: Push Images](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks#push-images-to-container-registry) |

---

## Phase 4: Deploy Spacelift Application

### Kubernetes Deployments (EKS/GKE/AKS/On-Prem)

#### Configure kubectl credentials

```bash
# EKS
aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION

# GKE
gcloud container clusters get-credentials $GKE_CLUSTER_NAME --location $GCP_LOCATION --project $GCP_PROJECT

# AKS
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME
```

#### Create Kubernetes namespace

```bash
kubectl create namespace spacelift
```

- [EKS namespace setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks#create-kubernetes-namespace)
- [GKE namespace setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke#create-kubernetes-namespace)
- [AKS namespace setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks#create-kubernetes-namespace)

#### Install ingress controller

| Platform | Ingress Controller | Docs |
|----------|-------------------|------|
| EKS | AWS Load Balancer Controller (installed by EKS Auto) | [EKS IngressClass setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks#create-ingressclass) |
| GKE | GKE built-in ingress (Gateway API) | [GKE ingress setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke) |
| AKS | NGINX Ingress Controller | [AKS NGINX setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks#nginx-controller) |
| On-prem | NGINX Ingress Controller or Traefik | [On-prem guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-onprem) |

#### Install cert-manager (if using Let's Encrypt)

Required for GKE, AKS, and on-prem deployments using Let's Encrypt. Not needed for AWS deployments using ACM.

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.16.2 --set crds.enabled=true
```

- [cert-manager installation](https://cert-manager.io/docs/installation/)
- [GKE cert-manager setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke#cert-manager)
- [AKS cert-manager setup](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks#cert-manager)

#### Create ClusterIssuer (for Let's Encrypt)

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
            ingressClassName: nginx  # or appropriate class
```

[ACME configuration](https://cert-manager.io/docs/configuration/acme/)

#### Create Kubernetes secrets

The Terraform module outputs all required secrets:

```bash
tofu output -raw kubernetes_secrets | kubectl apply -f -
```

This creates secrets containing:

- Database connection strings
- License token
- Encryption keys (RSA private key or KMS ARNs)
- Admin credentials
- Object storage configuration

- [EKS secrets](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks#create-secrets)
- [GKE secrets](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke#create-secrets)
- [AKS secrets](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks#create-secrets)

#### Deploy via Helm chart

[Spacelift Helm chart](https://downloads.spacelift.io/helm)

```bash
# Generate values file from Terraform output
tofu output -raw helm_values > spacelift-values.yaml

# Deploy
helm upgrade --repo https://downloads.spacelift.io/helm \
  spacelift spacelift-self-hosted \
  --install --wait --timeout 20m \
  --namespace spacelift \
  --values spacelift-values.yaml
```

**Monitor deployment:**

```bash
kubectl logs -n spacelift deployments/spacelift-server -f
```

- [EKS Helm deploy](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks#deploy-application)
- [GKE Helm deploy](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke#deploy-application)
- [AKS Helm deploy](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks#deploy-application)

---

### ECS Deployment

[Deploying to ECS](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-ecs)

For ECS, the services are deployed via the Terraform module rather than Helm:

1. Deploy infrastructure module first (`terraform-aws-spacelift-selfhosted`)
2. Push container images to ECR
3. Deploy services module (`terraform-aws-ecs-spacelift-selfhosted`)

```bash
# After deploying infra and pushing images, enable services
export TF_VAR_deploy_services=true
tofu apply
```

[ECS deployment guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-ecs#deploy-spacelift)

---

## Phase 5: DNS Configuration

- [EKS DNS](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks#configure-your-dns-zone)
- [GKE DNS](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke#configure-your-dns-zone)
- [AKS DNS](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks#configure-your-dns-zone)

After deployment, create DNS records pointing to your load balancers.

**Get load balancer addresses:**

```bash
# Server (HTTP) - Kubernetes
kubectl get ingress -n spacelift

# MQTT broker - Kubernetes (if using external workers)
kubectl get service spacelift-mqtt -n spacelift
```

**Create DNS records:**

| Record | Type | Value | Purpose |
|--------|------|-------|---------|
| `spacelift.example.com` | CNAME | `k8s-spacelif-serverv4-xxx.region.elb.amazonaws.com` | Server (UI/API) |
| `mqtt.spacelift.example.com` | CNAME | `k8s-spacelif-mqtt-xxx.elb.region.amazonaws.com` | MQTT broker (external workers only) |

**GKE note:** GKE may require both A (IPv4) and AAAA (IPv6) records:

```
spacelift.example.com    A     ${PUBLIC_IP_ADDRESS}
spacelift.example.com    AAAA  ${PUBLIC_IPV6_ADDRESS}
```

---

## Phase 6: First Setup

[First Setup Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/first-setup)

### Login with admin credentials

[Admin Login](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/first-setup#logging-in-using-admin-credentials)

Navigate to `https://your-server-hostname` (e.g., `https://spacelift.example.com`).

You'll be redirected to the admin login screen. Use the credentials configured during deployment:

- `ADMIN_USERNAME` - from Terraform variables
- `ADMIN_PASSWORD` - from Terraform variables

**Direct admin login URL:** `https://your-server-hostname/admin-login`

[Admin login configuration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/general-configuration#admin-login)

---

### Create account

[Creating an Account](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/first-setup#creating-an-account)

After logging in, choose an account name. This is used for:

- Display in the Spacelift UI
- AWS external ID and session names (cloud integration)
- Private registry URLs (modules/providers)

**Requirements:**

- 2-38 characters
- Must start with letter or number
- Only letters, numbers, `-`, and `_`

> **Choose carefully** - changing account names after deployment can break integrations (AWS trust policies, webhooks, etc.)

---

### Configure SSO/IdP (optional)

[Login Policy](https://docs.spacelift.io/self-hosted/latest/concepts/policy/login-policy)

Spacelift supports SSO via:

- SAML 2.0
- OpenID Connect (OIDC)
- GitHub OAuth
- GitLab OAuth

Configure in **Settings > Single Sign-On** after initial setup.

---

### Setup VCS integration

[Source Control Integrations](https://docs.spacelift.io/self-hosted/latest/integrations/source-control)

Connect Spacelift to your source control system to:

- Create stacks from repositories
- Receive push webhooks to trigger runs
- Post PR comments with plan output
- Access source code during runs

| VCS Provider | Setup Guide |
|--------------|-------------|
| GitHub (Cloud or Enterprise) | [GitHub Integration](https://docs.spacelift.io/self-hosted/latest/integrations/source-control/github) |
| GitLab (Cloud or Self-Managed) | [GitLab Integration](https://docs.spacelift.io/self-hosted/latest/integrations/source-control/gitlab) |
| Bitbucket Cloud | [Bitbucket Cloud Integration](https://docs.spacelift.io/self-hosted/latest/integrations/source-control/bitbucket-cloud) |
| Bitbucket Datacenter/Server | [Bitbucket Datacenter Integration](https://docs.spacelift.io/self-hosted/latest/integrations/source-control/bitbucket-datacenter-server) |
| Azure DevOps | [Azure DevOps Integration](https://docs.spacelift.io/self-hosted/latest/integrations/source-control/azure-devops) |
| Raw Git (any Git server) | [Raw Git Integration](https://docs.spacelift.io/self-hosted/latest/integrations/source-control/raw-git) |

---

### Create worker pool

[Worker Pools Overview](https://docs.spacelift.io/self-hosted/latest/concepts/worker-pools)

Workers execute your infrastructure-as-code runs. You need at least one worker pool before creating stacks.

| Worker Type | Best For | Setup Guide |
|-------------|----------|-------------|
| **Kubernetes Workers** | Kubernetes deployments (EKS/GKE/AKS) - workers run as pods | [Kubernetes Workers](https://docs.spacelift.io/self-hosted/latest/concepts/worker-pools/kubernetes-workers) |
| **Docker-based Workers** | EC2/VM deployments - workers run in Docker on instances | [Docker Workers](https://docs.spacelift.io/self-hosted/latest/concepts/worker-pools/docker-based-workers) |

**Kubernetes worker setup:**

1. Create a dedicated namespace: `kubectl create namespace spacelift-workers`
2. Create a WorkerPool in Spacelift UI
3. Create K8s secret with worker pool credentials
4. Deploy the WorkerPool CRD

> **Important:** Configure resource limits on WorkerPools to prevent unbounded resource requests. See [Run Pods configuration](https://docs.spacelift.io/self-hosted/latest/concepts/worker-pools/kubernetes-workers#run-pods).

---

### Disable admin login (optional)

[Admin Login Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/general-configuration#admin-login)

After configuring SSO, you can disable admin login by removing the `ADMIN_USERNAME` and `ADMIN_PASSWORD` environment variables from your deployment.

---

## Phase 7: Validation

- [ ] **Verify Spacelift UI loads** at `https://your-server-hostname`
- [ ] **Verify workers connect** - Check worker pool in Spacelift UI shows workers as "idle"
- [ ] **Create test stack** - [Create a Stack](https://docs.spacelift.io/self-hosted/latest/getting-started/create-stack)
- [ ] **Trigger a run** - Verify run completes successfully

---

## Phase 8: Optional Integrations

### Observability

[Observability Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/observability)

| Integration | What It Provides | Docs |
|-------------|-----------------|------|
| Datadog | Metrics, traces, logs | [Datadog setup](https://docs.spacelift.io/self-hosted/latest/integrations/observability/datadog) |
| Prometheus | Metrics scraping | [Prometheus setup](https://docs.spacelift.io/self-hosted/latest/integrations/observability/prometheus) |

### Telemetry

[Telemetry Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/telemetry)

Spacelift supports distributed tracing for debugging:

| Vendor | Docs |
|--------|------|
| Datadog APM | [Telemetry Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/telemetry) |
| AWS X-Ray | [Telemetry Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/telemetry) |
| OpenTelemetry | [Telemetry Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/telemetry) |

### Slack Integration

[Slack Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/slack)

Enable Slack notifications for runs, approvals, and drift detection.

### Cloud Integrations (OIDC)

[Cloud Integrations](https://docs.spacelift.io/self-hosted/latest/integrations/cloud-providers)

Allow Spacelift to assume roles in your cloud accounts without static credentials:

| Cloud | Docs |
|-------|------|
| AWS | [AWS OIDC Integration](https://docs.spacelift.io/self-hosted/latest/integrations/cloud-providers/oidc/aws-oidc) |
| GCP | [GCP OIDC Integration](https://docs.spacelift.io/self-hosted/latest/integrations/cloud-providers/oidc/gcp-oidc) |
| Azure | [Azure OIDC Integration](https://docs.spacelift.io/self-hosted/latest/integrations/cloud-providers/oidc/azure-oidc) |
| HashiCorp Vault | [Vault OIDC Integration](https://docs.spacelift.io/self-hosted/latest/integrations/cloud-providers/oidc/vault-oidc) |

### Usage Reporting

[Usage Reporting Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/usage-reporting)

Report usage data to Spacelift for license compliance. Can be automatic or manual export.

### Disaster Recovery

[Disaster Recovery Guide](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/disaster-recovery)

Plan for database backups, state recovery, and failover procedures.

---

## Quick Reference Tables

### Platform-Specific Guides

| Platform | Deployment Guide | Terraform Module |
|----------|-----------------|------------------|
| AWS EKS | [Deploying to EKS](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-eks) | [terraform-aws-eks-spacelift-selfhosted](https://github.com/spacelift-io/terraform-aws-eks-spacelift-selfhosted) |
| AWS ECS | [Deploying to ECS](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-ecs) | [terraform-aws-ecs-spacelift-selfhosted](https://github.com/spacelift-io/terraform-aws-ecs-spacelift-selfhosted) |
| Google GKE | [Deploying to GKE](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-gke) | [terraform-google-spacelift-selfhosted](https://github.com/spacelift-io/terraform-google-spacelift-selfhosted) |
| Azure AKS | [Deploying to AKS](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-aks) | [terraform-azure-spacelift-selfhosted](https://github.com/spacelift-io/terraform-azure-spacelift-selfhosted) |
| On-Prem K8s | [Deploying to on-prem](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/deploying-to-onprem) | - |
| Air-gapped | [Air-gapped environments](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/guides/air-gapped) | - |

### Configuration Reference

| Topic | Reference |
|-------|-----------|
| General configuration | [General Configuration](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/general-configuration) |
| Encryption | [Encryption Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/encryption) |
| Message queues | [Message Queues Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/message-queues) |
| MQTT broker | [MQTT Broker Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/mqtt-broker) |
| Networking | [Networking Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/networking) |
| Object storage | [Object Storage Reference](https://docs.spacelift.io/self-hosted/latest/installing-spacelift/reference-architecture/reference/object-storage) |
