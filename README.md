# Kubernetes GitOps Backup & Restore (Minikube → GKE)

## Project Overview

### Problem Statement

Organizations often run workloads in local or on-premises Kubernetes clusters during development or early-stage deployments. However, **local clusters are not resilient**: hardware failure, data loss, or environment corruption can result in complete service outages. Without a tested disaster recovery strategy, restoring applications to the cloud can be slow, manual, and error-prone.

### Solution

This project demonstrates a **disaster recovery and migration strategy from a local Kubernetes cluster (Minikube) to a cloud-managed cluster (GKE)**. The solution uses:

* **GitHub Actions** to build and publish application images
* **Argo CD (GitOps)** to ensure declarative, version-controlled deployments
* **Velero** to back up the full Kubernetes cluster state
* **Google Cloud Storage (GCS)** as an external, durable backup location
* **Terraform** to provision a new cloud environment

In a simulated failure scenario, the local cluster is assumed to be lost. The backed-up workloads are then **restored into a freshly created GKE cluster**, proving that the application can be recovered in the cloud **without manual redeployment**.

This mirrors a real-world disaster recovery use case where workloads must be quickly restored from on-premises or local environments to the cloud.

---

## Architecture Overview

### Design Goals

* **Automation first**: No manual deployments to Kubernetes
* **Environment parity**: Same manifests used across local and cloud clusters
* **Disaster recovery readiness**: Backups stored externally and restorable to a new cluster
* **Reproducibility**: Entire setup can be recreated using code and documented steps

```
Developer Commit
   │
   ▼
GitHub Repository
   │
   ├── GitHub Actions (CI)
   │      ├── Build Docker Image
   │      └── Push Image to Docker Hub
   │
   └── Kubernetes Manifests
           │
           ▼
        Argo CD (GitOps CD)
           │
           ▼
      Minikube Cluster
           │
           ▼
       Velero Backup
           │
           ▼
 Google Cloud Storage (GCS)
           │
           ▼
       Velero Restore
           │
           ▼
        GKE Cluster
```
![diagram flow](https://github.com/smogalloyubio/04-Googlecloud-gke-Gitops-pipeline/blob/main/picture/Untitled-2025-12-26-1916.png)
---

## Technologies Used

> All tools selected are widely used in production DevOps environments.

| Category                   | Tool                        |
| -------------------------- | --------------------------- |
| Cloud Provider             | Google Cloud Platform (GCP) |
| Containerization           | Docker                      |
| Container Registry         | Docker Hub                  |
| Container Orchestration    | Kubernetes                  |
| Local Kubernetes           | Minikube                    |
| Continuous Integration     | GitHub Actions              |
| Continuous Delivery        | Argo CD (GitOps)            |
| Infrastructure as Code     | Terraform                   |
| Backup & Disaster Recovery | Velero                      |
| Object Storage             | Google Cloud Storage (GCS)  |
| Version Control            | GitHub                      |

---

## Tooling Overview & Rationale

Before describing the implementation steps, this section explains **why each tool was chosen and how it is used within the project**.

### GitHub Actions (CI)

**Purpose:** Continuous Integration

GitHub Actions is used to automate the build process of the application. Every change pushed to the repository triggers a workflow that builds a Docker image and pushes it to Docker Hub. This ensures that application images are **versioned, reproducible, and automatically generated**.

---

### Docker

**Purpose:** Application Containerization

Docker packages the application and its dependencies into a container image. This guarantees that the application runs consistently across local and cloud Kubernetes environments.
```

 # Build stage
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy application files
COPY . .

# Build the application
RUN npm run build

# Runtime stage
FROM node:20-alpine

WORKDIR /app

# Install a simple HTTP server to serve the static files
RUN npm install -g serve

# Copy built application from builder stage
COPY --from=builder /app/dist ./dist

# Expose port
EXPOSE 3000

# Start the application
CMD ["serve", "-s", "dist", "-l", "3000"]

```
---

### Docker Hub

**Purpose:** Container Image Registry

Docker Hub stores the built container images and acts as a central registry accessible by both Minikube and GKE clusters.

---

### Kubernetes

**Purpose:** Container Orchestration

Kubernetes manages application deployment, scaling, and networking. The same Kubernetes manifests are used across all environments to maintain consistency.

---

### Argo CD (GitOps)

**Purpose:** Continuous Delivery

Argo CD implements the GitOps model by continuously reconciling the Kubernetes cluster state with declarative manifests stored in Git. This eliminates manual deployments and ensures traceability and rollback capability.

---

### Velero

**Purpose:** Backup and Disaster Recovery

Velero is used to back up Kubernetes resources and persistent data from the local Minikube cluster. These backups are stored externally and later restored into a cloud-based GKE cluster during disaster recovery.

---

### Google Cloud Storage (GCS)

**Purpose:** External Backup Storage

GCS serves as a durable, cloud-hosted storage backend for Velero backups, enabling recovery even if the local cluster is completely lost.

---

### Terraform

**Purpose:** Infrastructure as Code

Terraform provisions the cloud infrastructure, including the VPC and GKE cluster. This ensures the disaster recovery environment is **fully reproducible and version-controlled**.

---

## CI/CD Pipeline Design

### Continuous Integration – GitHub Actions

The CI pipeline is triggered on every push to the `main` branch. It is responsible for building the application Docker image and publishing it to Docker Hub.

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: ["main"]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source code
        uses: actions/checkout@v3

      - name: Authenticate to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: <docker-username>/<image-name>:latest
```
![Gitaction pipeline](https://github.com/smogalloyubio/04-Googlecloud-gke-Gitops-pipeline/blob/main/picture/Screenshot%202026-01-24%20at%2022.38.55.png)

### Continuous Delivery – GitOps with Argo CD

Argo CD continuously monitors the Kubernetes manifests stored in Git and ensures that the desired state is applied to the cluster. Application updates are deployed automatically without manual `kubectl` commands.

```yaml
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd

Argo CD generates an initial admin password stored as a Kubernetes secret.
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode

kubectl port-forward svc/argocd-server -n argocd 8080:443

kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort"}}'


```

![argocd pipeline](https://github.com/smogalloyubio/04-Googlecloud-gke-Gitops-pipeline/blob/main/picture/Screenshot%202026-01-24%20at%2022.38.17.png)
---

## Prerequisites

Ensure the following tools are installed locally:

* Docker
* kubectl
* Minikube
* Terraform
* gcloud CLI
* Velero CLI
* A Google Cloud project with billing enabled
* A Docker Hub account

---

## Environment Setup & Deployment

### 1. Start Local Kubernetes Cluster

```bash
minikube start
kubectl get nodes
kubectl  config get-context
kuebctl  config  current-context
kubectl create namespace  dev,argocd, demo
```
![minikube cluster](https://github.com/smogalloyubio/04-Googlecloud-gke-Gitops-pipeline/blob/main/picture/Screenshot%202026-01-24%20at%2019.44.24.png)
---

### 2. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

Argo CD generates an initial admin password stored as a Kubernetes secret.
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode

kubectl port-forward svc/argocd-server -n argocd 8080:443

kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort"}}'


```

Access the Argo CD UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

---

### 3. Configure GCS Bucket and Service Account

```bash
gsutil mb gs://ubioworo_netflix-backup

gcloud iam service-accounts create velero

gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:velero@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/storage.admin"
```

---

### 4. Install Velero on Minikube

```bash
velero install \
  --provider gcp \
  --plugins velero/velero-plugin-for-gcp:v1.7.0 \
  --bucket <BUCKET_NAME> \
  --secret-file ./credentials-velero
```
![install  velero](https://github.com/smogalloyubio/04-Googlecloud-gke-Gitops-pipeline/blob/main/picture/Screenshot%202026-01-24%20at%2020.47.54.png)
---

### 5. Backup the Minikube Cluster

```bash
velero backup create minikube-backup --include-namespaces '*'
velero backup get
velero backup-location get
velero describe  backup minikube-backup

```
![velero backup localcluster](https://github.com/smogalloyubio/04-Googlecloud-gke-Gitops-pipeline/blob/main/picture/Screenshot%202026-01-24%20at%2022.22.35.png)
---

### 6. Provision GKE Infrastructure Using Terraform

```bash
terraform init
terraform plan
terraform apply
```
![terraform  provison](https://github.com/smogalloyubio/04-Googlecloud-gke-Gitops-pipeline/blob/main/picture/Screenshot%202026-01-24%20at%2019.41.43.png)
Configure cluster access:

```bash
gcloud container clusters get-credentials <CLUSTER_NAME> \
  --region <REGION> --project <PROJECT_ID>
```

---

### 7. Restore Backup to GKE Cluster

```bash
velero install \
  --provider gcp \
  --plugins velero/velero-plugin-for-gcp:v1.7.0 \
  --bucket <BUCKET_NAME> \
  --secret-file ./credentials-velero

velero restore create --from-backup minikube-backup
```
![gke recovery cluster](https://github.com/smogalloyubio/04-Googlecloud-gke-Gitops-pipeline/blob/main/picture/Screenshot%202026-01-24%20at%2022.25.34.png)
---

## Backup & Restore Strategy

* Full cluster backups are created using Velero
* Backups are stored remotely in GCS
* Restore operations are tested on a clean GKE cluster
* No application redeployment is required after restore

This approach simulates a **real disaster recovery or cluster migration scenario**.

---

## Validation & Results

```bash
kubectl get all --all-namespaces
velero restore get
```
![velero restore on gke](https://github.com/smogalloyubio/04-Googlecloud-gke-Gitops-pipeline/blob/main/picture/Screenshot%202026-01-24%20at%2022.26.39.png)

* Application workloads are successfully restored on GKE
* Kubernetes resources match the original Minikube state
* GitOps reconciliation resumes automatically via Argo CD

---

## Key DevOps Concepts Demonstrated

* CI/CD automation and pipeline design
* Docker image lifecycle management
* GitOps-based continuous delivery
* Kubernetes backup and disaster recovery
* Infrastructure as Code (Terraform)
* Cloud migration and environment portability





