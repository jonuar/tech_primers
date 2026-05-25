# GCP Associate Cloud Engineer — Certification Cheatsheet

## Mental Model

The Associate Cloud Engineer (ACE) exam tests whether you can **deploy, manage, and monitor** workloads on Google Cloud. It's operations-heavy — not architecture or design (that's Professional Cloud Architect). The exam assumes you know how to use `gcloud`, navigate the console, and understand the core compute, storage, networking, and identity services. Focus on knowing *when to use which service* and *what the CLI commands look like*.

**Exam facts:** 50 questions · 2 hours · Multiple choice / multiple select · ~$200 · Passing score ~70%

---

## 1. gcloud CLI Essentials

```bash
# Auth & config — always start here
gcloud auth login
gcloud auth application-default login      # for SDKs and local dev
gcloud config set project PROJECT_ID
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Manage configurations (like kubectl contexts)
gcloud config configurations create staging
gcloud config configurations activate staging
gcloud config configurations list

# Useful flags (work on almost every command)
--project=PROJECT_ID
--region=REGION
--zone=ZONE
--format=json          # or yaml, table, value
--filter="status=RUNNING"
--quiet                # skip confirmation prompts

# Get help
gcloud compute instances --help
gcloud help compute instances create
```

---

## 2. Compute

### Compute Engine (VMs)

```bash
# Create a VM
gcloud compute instances create my-vm \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --zone=us-central1-a \
  --tags=http-server \
  --metadata=startup-script='#!/bin/bash
    apt-get update && apt-get install -y nginx'

# Common operations
gcloud compute instances list
gcloud compute instances start my-vm --zone=us-central1-a
gcloud compute instances stop my-vm --zone=us-central1-a
gcloud compute instances delete my-vm --zone=us-central1-a
gcloud compute instances describe my-vm --zone=us-central1-a

# SSH into instance
gcloud compute ssh my-vm --zone=us-central1-a

# Copy files
gcloud compute scp local-file.txt my-vm:~/remote-file.txt --zone=us-central1-a
```

| Machine family | Use case |
|---|---|
| `e2-*` | Cost-optimized, general workloads |
| `n2-*` | Balanced performance |
| `c2-*` | Compute-intensive (ML training) |
| `m1-*` / `m2-*` | Memory-intensive (SAP, large DBs) |
| `a2-*` | GPU workloads (NVIDIA A100) |

### Instance Groups

```bash
# Managed Instance Group (MIG) — autoscaling, rolling updates
gcloud compute instance-groups managed create my-mig \
  --template=my-template \
  --size=3 \
  --zone=us-central1-a

# Set autoscaling
gcloud compute instance-groups managed set-autoscaling my-mig \
  --max-num-replicas=10 \
  --min-num-replicas=2 \
  --target-cpu-utilization=0.75 \
  --zone=us-central1-a
```

### GKE (Google Kubernetes Engine)

```bash
# Create cluster
gcloud container clusters create my-cluster \
  --num-nodes=3 \
  --machine-type=e2-standard-2 \
  --region=us-central1

# Get credentials (updates ~/.kube/config)
gcloud container clusters get-credentials my-cluster --region=us-central1

# Node pools
gcloud container node-pools create gpu-pool \
  --cluster=my-cluster \
  --machine-type=a2-highgpu-1g \
  --num-nodes=1 \
  --region=us-central1

# Upgrade cluster
gcloud container clusters upgrade my-cluster --master --cluster-version=1.28
```

### Cloud Run (Serverless Containers)

```bash
# Deploy a container
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-app:latest \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated \
  --memory=512Mi \
  --cpu=1 \
  --min-instances=0 \
  --max-instances=10 \
  --set-env-vars="ENV=production,DB_URL=postgresql://..."

gcloud run services list --region=us-central1
gcloud run services describe my-service --region=us-central1
```

### App Engine

```bash
# Deploy (app.yaml defines runtime, scaling)
gcloud app deploy

# Standard environment (auto-scales to 0, limited runtimes)
# Flexible environment (Docker, scales to 0 is slower)
gcloud app browse
gcloud app logs tail
gcloud app versions list
gcloud app versions stop v1
```

### Cloud Functions

```bash
# Gen 1
gcloud functions deploy my-function \
  --runtime=python311 \
  --trigger-http \
  --entry-point=handler \
  --allow-unauthenticated

# Gen 2 (Cloud Run based — preferred)
gcloud functions deploy my-function \
  --gen2 \
  --runtime=python311 \
  --trigger-http \
  --entry-point=handler \
  --region=us-central1
```

---

## 3. Storage

| Service | Use case |
|---|---|
| **Cloud Storage (GCS)** | Object storage — blobs, backups, static assets, data lake |
| **Cloud SQL** | Managed relational DB (PostgreSQL, MySQL, SQL Server) |
| **Cloud Spanner** | Globally distributed relational DB (very expensive) |
| **Firestore** | Serverless NoSQL document DB |
| **Bigtable** | NoSQL wide-column, high-throughput (IoT, time series) |
| **BigQuery** | Serverless data warehouse — analytical queries at scale |
| **Memorystore** | Managed Redis / Memcached |
| **AlloyDB** | PostgreSQL-compatible, ML-optimized (pgvector support) |

### Cloud Storage (GCS)

```bash
# Bucket operations
gcloud storage buckets create gs://my-bucket --location=us-central1
gcloud storage buckets list
gcloud storage buckets describe gs://my-bucket

# Object operations
gcloud storage cp local-file.txt gs://my-bucket/path/
gcloud storage cp gs://my-bucket/path/file.txt ./local/
gcloud storage ls gs://my-bucket/
gcloud storage rm gs://my-bucket/path/file.txt
gcloud storage rsync ./local-dir gs://my-bucket/prefix/ --recursive

# Signed URL (temporary access)
gcloud storage sign-url gs://my-bucket/file.txt \
  --duration=1h \
  --private-key-file=key.json
```

| Storage class | Use case | Retrieval cost |
|---|---|---|
| `STANDARD` | Frequently accessed | None |
| `NEARLINE` | ~Once/month | Yes |
| `COLDLINE` | ~Once/quarter | Yes |
| `ARCHIVE` | < Once/year | Yes |

### Cloud SQL

```bash
# Create instance
gcloud sql instances create my-db \
  --database-version=POSTGRES_15 \
  --tier=db-g1-small \
  --region=us-central1

# Connect via Cloud SQL Proxy (recommended)
gcloud sql connect my-db --user=postgres

# Create database and user
gcloud sql databases create myapp --instance=my-db
gcloud sql users create myuser --instance=my-db --password=secret
```

---

## 4. Networking

### VPC

```bash
# Create VPC and subnet
gcloud compute networks create my-vpc --subnet-mode=custom
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.0.0/24

# Firewall rules
gcloud compute firewall-rules create allow-http \
  --network=my-vpc \
  --allow=tcp:80,tcp:443 \
  --target-tags=http-server \
  --source-ranges=0.0.0.0/0

gcloud compute firewall-rules list --filter="network=my-vpc"
```

### Load Balancing

| Type | Layer | Use case |
|---|---|---|
| HTTP(S) LB | 7 | Global, path-based routing, Cloud CDN |
| TCP Proxy | 4 | Global TCP |
| Network LB | 4 | Regional, preserves client IP |
| Internal HTTP(S) LB | 7 | Internal services only |

### Cloud DNS

```bash
gcloud dns managed-zones create my-zone \
  --dns-name=myapp.com. \
  --description="Production zone"

gcloud dns record-sets create api.myapp.com. \
  --zone=my-zone \
  --type=A \
  --ttl=300 \
  --rrdatas=1.2.3.4
```

---

## 5. IAM & Security

```bash
# View IAM policy for a project
gcloud projects get-iam-policy PROJECT_ID

# Add a binding
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:email@example.com" \
  --role="roles/compute.viewer"

# Remove a binding
gcloud projects remove-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:sa@project.iam.gserviceaccount.com" \
  --role="roles/storage.admin"

# Service accounts
gcloud iam service-accounts create my-sa \
  --display-name="My Service Account"

gcloud iam service-accounts keys create key.json \
  --iam-account=my-sa@PROJECT_ID.iam.gserviceaccount.com

# Impersonate a service account (for testing permissions)
gcloud auth print-access-token --impersonate-service-account=my-sa@PROJECT.iam.gserviceaccount.com
```

### Key IAM Roles to Know

| Role | Scope |
|---|---|
| `roles/viewer` | Read-only access to all resources |
| `roles/editor` | Read + write, no IAM changes |
| `roles/owner` | Full access including IAM |
| `roles/compute.admin` | Full Compute Engine |
| `roles/storage.objectViewer` | Read GCS objects |
| `roles/storage.objectAdmin` | Full GCS object management |
| `roles/container.developer` | Deploy to GKE |
| `roles/run.invoker` | Invoke Cloud Run services |
| `roles/iam.serviceAccountUser` | Act as a service account |
| `roles/bigquery.dataViewer` | Query BigQuery tables |

---

## 6. Monitoring & Operations (Cloud Operations Suite)

```bash
# View logs
gcloud logging read "resource.type=gce_instance" --limit=50
gcloud logging read "severity>=ERROR" --freshness=1h

# Create log sink (export logs to GCS, BigQuery, Pub/Sub)
gcloud logging sinks create my-sink \
  storage.googleapis.com/my-bucket \
  --log-filter="severity>=ERROR"
```

| Tool | Purpose |
|---|---|
| **Cloud Monitoring** | Metrics, dashboards, alerting |
| **Cloud Logging** | Centralized log storage and querying |
| **Cloud Trace** | Distributed tracing |
| **Cloud Profiler** | CPU/memory profiling in production |
| **Error Reporting** | Aggregates and alerts on exceptions |

---

## 7. Key Topics for the Exam

### Billing & Quotas

```bash
gcloud billing accounts list
gcloud billing projects link PROJECT_ID --billing-account=ACCOUNT_ID
gcloud compute project-info describe --project=PROJECT_ID  # quotas
```

- **Committed use discounts** — 1 or 3 year commitment, up to 57% off
- **Sustained use discounts** — automatic discount for running VMs > 25% of the month
- **Preemptible / Spot VMs** — up to 91% cheaper, can be terminated anytime (max 24h for preemptible)

### Deployment Manager / Cloud Deployment

```bash
# Deployment Manager — GCP's Terraform equivalent (YAML/Python/Jinja2)
gcloud deployment-manager deployments create my-deployment \
  --config=config.yaml

gcloud deployment-manager deployments update my-deployment \
  --config=config.yaml
```

### Cloud Build (CI/CD)

```bash
# Trigger a build
gcloud builds submit --tag=gcr.io/PROJECT_ID/my-app:latest .

# cloudbuild.yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/my-app', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/my-app']
images:
  - 'gcr.io/$PROJECT_ID/my-app'
```

---

## Exam Tips

- **Know the decision tree for compute** — Compute Engine (full control) → GKE (containers) → Cloud Run (serverless containers, scales to 0) → App Engine (PaaS) → Cloud Functions (event-driven, small functions)
- **Storage decision tree** — structured relational? Cloud SQL. Global scale relational? Spanner. NoSQL document? Firestore. Analytical? BigQuery. Object/files? GCS.
- **IAM principle of least privilege** — exam favors more granular, predefined roles over primitive roles (Owner/Editor/Viewer)
- **`gcloud` vs console** — know both; the exam describes scenarios where you use either
- **Managed vs unmanaged instance groups** — MIGs support autoscaling and rolling updates; unmanaged do not
- **Regions vs zones** — regions are geographic areas (us-central1), zones are data centers within a region (us-central1-a). Put VMs in multiple zones for HA.
- **Cloud NAT** — allows VMs without external IPs to reach the internet. Frequently tested.
- **Shared VPC** — one host project's VPC shared across multiple service projects. Used in organizations.

---

## Quick Links

- [GCP ACE Exam Guide (official)](https://cloud.google.com/learn/certification/cloud-engineer)
- [GCP Documentation](https://cloud.google.com/docs)
- [gcloud Reference](https://cloud.google.com/sdk/gcloud/reference)
- [Google Cloud Skills Boost](https://cloudskillsboost.google) — official hands-on labs
- [Associate Cloud Engineer Study Guide (book)](https://www.wiley.com/en-us/Official+Google+Cloud+Certified+Associate+Cloud+Engineer+Study+Guide-p-9781119564416)
- [ExamTopics GCP ACE](https://www.examtopics.com/exams/google/associate-cloud-engineer/) — community practice questions
