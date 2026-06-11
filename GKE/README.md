# Google Kubernetes Engine (GKE) — Complete Study Guide
> **Source-aligned with Google Cloud official documentation (June 2026)**  
> Format: 🟢 Basic | 🟡 Intermediate | 🔴 Advanced | YAML/CLI examples | Master Cheatsheet

---

## Table of Contents
1. [GKE Core Concepts & Architecture](#1-gke-core-concepts--architecture)
2. [Autopilot vs Standard Mode](#2-autopilot-vs-standard-mode)
3. [Cluster Creation (gcloud, Terraform)](#3-cluster-creation)
4. [Node Pools & Machine Types](#4-node-pools--machine-types)
5. [Networking (VPC-native, Services, Ingress, Gateway API)](#5-networking)
6. [Storage (Persistent Disks, Filestore, GCS Fuse)](#6-storage)
7. [Identity & Security (Workload Identity, RBAC, Binary Authorization)](#7-identity--security)
8. [Scaling (Cluster Autoscaler, NAP, HPA, VPA, KEDA)](#8-scaling)
9. [Upgrades & Release Channels](#9-upgrades--release-channels)
10. [Monitoring & Observability (Cloud Monitoring, Managed Prometheus, Logging)](#10-monitoring--observability)
11. [DevOps & CI/CD (Cloud Build, Cloud Deploy, Skaffold)](#11-devops--cicd)
12. [Cost Optimization](#12-cost-optimization)
13. [Multi-Cluster (Fleet, Anthos, Config Sync)](#13-multi-cluster--anthos)
14. [Advanced Topics (GKE Enterprise, Confidential Nodes, GPU, TPU)](#14-advanced-topics)
15. [Troubleshooting Scenarios](#15-troubleshooting-scenarios)
16. [Master Cheatsheet](#master-cheatsheet)

---

## 1. GKE Core Concepts & Architecture

### 🟢 Q1. What is GKE and what does Google Cloud manage?

**GKE** is Google Cloud's fully managed Kubernetes service. In both Autopilot and Standard mode, Google manages the control plane at no cost (free). Autopilot additionally manages nodes and system components.

```
GKE Architecture:
┌──────────────────────────────────────────────────────┐
│              Google Managed (Free)                    │
│  kube-apiserver │ etcd │ scheduler │ controller-mgr  │
│  (Regional: 3 AZ HA | Zonal: single AZ)               │
└──────────────────────────────────────────────────────┘
         ↕ Private/Public endpoint
┌──────────────────────────────────────────────────────┐
│         Data Plane                                    │
│  Autopilot: Google manages nodes (fully managed)      │
│  Standard: Customer manages node pools                │
└──────────────────────────────────────────────────────┘
```

| Feature | Autopilot | Standard |
|---------|-----------|----------|
| Control plane | Google managed | Google managed |
| Nodes | Google managed | Customer managed |
| Node OS | Container-Optimized OS | COS, Ubuntu, Windows |
| Cost model | Per pod vCPU/mem | Per node (VM) |
| Security defaults | Hardened (Shielded, Workload Identity) | You configure |
| Recommended for | Most workloads | Custom kernels, special configs |

---

### 🟢 Q2. What are zonal vs regional GKE clusters?

```bash
# Zonal cluster (single zone, control plane in 1 AZ - dev only)
gcloud container clusters create myCluster \
  --zone us-central1-a \
  --num-nodes 3

# Regional cluster (control plane in 3 AZs, HA - recommended for prod)
gcloud container clusters create myCluster \
  --region us-central1 \
  --num-nodes 1 \          # per zone (3 zones × 1 = 3 total nodes)
  --node-locations us-central1-a,us-central1-b,us-central1-c
```

| Cluster Type | Control Plane AZs | Node Distribution | Downtime on Zone Failure |
|-------------|------------------|------------------|-------------------------|
| Zonal | 1 | Single zone | Yes |
| Regional | 3 (same region) | Multiple zones | No |
| Autopilot | Regional (default) | Managed | No |

---

### 🟡 Q3. How does GKE use Container-Optimized OS (COS)?

**Container-Optimized OS** (COS) is the default node OS for GKE. It is minimal, read-only root filesystem, includes containerd, and is auto-hardened with:
- Locked-down kernel (no kernel modules loaded at runtime)
- Verified boot (dm-verity)
- Built-in Docker credential helper for Artifact Registry
- Automatic node auto-upgrades

```bash
# COS is default; you can also use Ubuntu or Windows
gcloud container node-pools create ubuntu-pool \
  --cluster myCluster \
  --region us-central1 \
  --image-type UBUNTU_CONTAINERD \
  --num-nodes 2

# Check node image type
kubectl get nodes -o custom-columns=NAME:.metadata.name,IMAGE:.status.nodeInfo.osImage
```

---

## 2. Autopilot vs Standard Mode

### 🟡 Q4. When should you choose Autopilot over Standard?

```bash
# Create Autopilot cluster (recommended default)
gcloud container clusters create-auto myAutopilotCluster \
  --region us-central1 \
  --release-channel regular \
  --network my-vpc \
  --subnetwork my-subnet

# Autopilot specifics:
# - Billing: per pod CPU/memory (not per node)
# - Minimum resources: 250m CPU, 512Mi memory per container
# - Workload Identity: always enabled
# - Shielded nodes: always enabled
# - Network policy: always enabled (VPC firewall)
# - Node auto-upgrades: always on (can't disable)
# - Cannot SSH into nodes, no DaemonSets (except system ones)

# Create Standard cluster
gcloud container clusters create myCluster \
  --region us-central1 \
  --num-nodes 3 \
  --machine-type e2-standard-4 \
  --release-channel regular \
  --workload-pool=myproject.svc.id.goog \   # workload identity
  --enable-ip-alias \                        # VPC-native
  --network my-vpc \
  --subnetwork my-subnet
```

```yaml
# Autopilot compute class annotation (controls node hardware)
apiVersion: v1
kind: Pod
metadata:
  name: high-memory-app
  annotations:
    cloud.google.com/compute-class: "Balanced"   # Balanced | Performance | Accelerator | Scale-Out
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        cpu: "4"
        memory: "16Gi"
      limits:
        cpu: "4"
        memory: "16Gi"
---
# Autopilot GPU workload
apiVersion: v1
kind: Pod
metadata:
  name: gpu-inference
  annotations:
    cloud.google.com/compute-class: "Accelerator"
spec:
  containers:
  - name: inference
    image: myapp:latest
    resources:
      requests:
        nvidia.com/gpu: "1"
      limits:
        nvidia.com/gpu: "1"
```

---

### 🟡 Q5. What are GKE Node Auto-Provisioning (NAP) and Autopilot-managed node pools?

```bash
# Enable Node Auto-Provisioning on Standard cluster
gcloud container clusters update myCluster \
  --region us-central1 \
  --enable-autoprovisioning \
  --max-cpu 100 \
  --max-memory 1000 \
  --min-cpu 0 \
  --min-memory 0 \
  --autoprovisioning-locations us-central1-a,us-central1-b,us-central1-c

# NAP automatically creates/deletes node pools based on pending pods
# Uses least-waste bin-packing algorithm

# NAP resource limits
gcloud container clusters update myCluster \
  --region us-central1 \
  --autoprovisioning-max-surge-upgrade 1 \
  --autoprovisioning-max-unavailable-upgrade 0
```

---

## 3. Cluster Creation

### 🟢 Q6. What are the key parameters for creating a production GKE cluster?

```bash
gcloud container clusters create myProdCluster \
  --region us-central1 \
  # Networking
  --enable-ip-alias \                         # VPC-native (required for most features)
  --network my-vpc \
  --subnetwork my-subnet \
  --cluster-secondary-range-name pods \       # secondary range for pods
  --services-secondary-range-name services \  # secondary range for services
  # Node pools
  --machine-type n2-standard-4 \
  --num-nodes 2 \                             # per zone
  --disk-type pd-ssd \
  --disk-size 100GB \
  --image-type COS_CONTAINERD \
  # Availability
  --node-locations us-central1-a,us-central1-b,us-central1-c \
  # Security
  --enable-shielded-nodes \
  --shielded-secure-boot \
  --shielded-integrity-monitoring \
  --workload-pool=myproject.svc.id.goog \     # Workload Identity
  --enable-private-nodes \                    # nodes no public IP
  --enable-private-endpoint \                 # API server private only
  --master-authorized-networks 10.0.0.0/8 \
  # Upgrades
  --release-channel regular \
  --enable-autoupgrade \
  --enable-autorepair \
  # Monitoring
  --enable-managed-prometheus \
  --logging=SYSTEM,WORKLOAD \
  --monitoring=SYSTEM,WORKLOAD \
  # Autoscaling
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 10 \
  --enable-autoprovisioning
```

---

### 🔴 Q7. How do you create a GKE cluster with Terraform?

```hcl
# main.tf - GKE cluster with Google provider
terraform {
  required_providers {
    google = { source = "hashicorp/google", version = "~> 5.0" }
  }
}

resource "google_container_cluster" "gke" {
  name     = "my-gke-cluster"
  location = "us-central1"          # regional cluster

  # Autopilot mode
  enable_autopilot = false           # set true for Autopilot

  # Remove default node pool (use separate resource)
  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  release_channel {
    channel = "REGULAR"             # RAPID | REGULAR | STABLE | EXTENDED
  }

  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Private cluster
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }

  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "10.0.0.0/8"
      display_name = "Internal"
    }
  }

  # Security
  enable_shielded_nodes = true

  # Logging and Monitoring
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }

  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
    managed_prometheus { enabled = true }
  }

  # Addons
  addons_config {
    http_load_balancing { disabled = false }
    horizontal_pod_autoscaling { disabled = false }
    gce_persistent_disk_csi_driver_config { enabled = true }
    gcs_fuse_csi_driver_config { enabled = true }
    gke_backup_agent_config { enabled = true }
  }
}

resource "google_container_node_pool" "primary" {
  name     = "primary-pool"
  cluster  = google_container_cluster.gke.name
  location = "us-central1"
  node_count = 2                    # per zone

  autoscaling {
    min_node_count       = 1
    max_node_count       = 10
    location_policy      = "BALANCED"
  }

  management {
    auto_repair  = true
    auto_upgrade = true
  }

  upgrade_settings {
    max_surge       = 1
    max_unavailable = 0
    strategy        = "SURGE"       # SURGE | BLUE_GREEN
  }

  node_config {
    machine_type = "n2-standard-4"
    disk_type    = "pd-ssd"
    disk_size_gb = 100
    image_type   = "COS_CONTAINERD"
    spot         = false            # set true for Spot VMs

    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }

    workload_metadata_config {
      mode = "GKE_METADATA"        # required for Workload Identity
    }

    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]

    labels = { environment = "production" }
    tags   = ["gke-node"]
  }
}
```

---

## 4. Node Pools & Machine Types

### 🟢 Q8. How do you add and manage node pools in GKE?

```bash
# Add node pool
gcloud container node-pools create gpu-pool \
  --cluster myCluster \
  --region us-central1 \
  --machine-type a2-highgpu-1g \   # A100 GPU
  --accelerator type=nvidia-tesla-a100,count=1,gpu-driver-version=default \
  --num-nodes 2 \
  --min-nodes 0 \
  --max-nodes 10 \
  --enable-autoscaling \
  --node-taints nvidia.com/gpu=present:NoSchedule

# Add Spot VM node pool (preemptible successor - 60-90% discount)
gcloud container node-pools create spot-pool \
  --cluster myCluster \
  --region us-central1 \
  --machine-type e2-standard-4 \
  --spot \
  --num-nodes 3 \
  --min-nodes 0 \
  --max-nodes 50 \
  --enable-autoscaling

# Add ARM (Tau T2A) node pool (Google Axion / Arm-based)
gcloud container node-pools create arm-pool \
  --cluster myCluster \
  --region us-central1 \
  --machine-type t2a-standard-4 \
  --image-type COS_CONTAINERD \
  --num-nodes 2

# List node pools
gcloud container node-pools list \
  --cluster myCluster \
  --region us-central1

# Resize node pool
gcloud container clusters resize myCluster \
  --region us-central1 \
  --node-pool primary-pool \
  --num-nodes 5
```

---

### 🟡 Q9. How do you use Spot VMs (preemptible) for GKE cost savings?

```bash
# Spot VMs: up to 91% cheaper, can be reclaimed with 30s notice
gcloud container node-pools create spot-pool \
  --cluster myCluster \
  --region us-central1 \
  --spot \
  --machine-type e2-standard-4 \
  --enable-autoscaling \
  --min-nodes 0 \
  --max-nodes 100

# Check node is spot
kubectl get node -l cloud.google.com/gke-spot=true
```

```yaml
# Workload tolerating Spot preemption
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-app
spec:
  replicas: 20
  template:
    spec:
      nodeSelector:
        cloud.google.com/gke-spot: "true"
      tolerations:
      - key: cloud.google.com/gke-spot
        operator: Exists
        effect: NoSchedule
      terminationGracePeriodSeconds: 25    # < 30s (spot eviction notice)
      containers:
      - name: app
        image: myapp:latest
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "save-progress.sh && sleep 20"]
```

---

### 🟡 Q10. How do you run GPU and TPU workloads on GKE?

```bash
# GPU node pool with auto-driver installation
gcloud container node-pools create gpu-pool \
  --cluster myCluster \
  --region us-central1 \
  --machine-type n1-standard-4 \
  --accelerator type=nvidia-tesla-t4,count=1,gpu-driver-version=latest \
  --num-nodes 2 \
  --node-taints nvidia.com/gpu=present:NoSchedule

# TPU node pool (v4, v5)
gcloud container node-pools create tpu-pool \
  --cluster myCluster \
  --region us-central1 \
  --tpu-topology 2x2x2 \
  --machine-type ct5p-hightpu-4t
```

```yaml
# GPU Pod (NVIDIA)
apiVersion: v1
kind: Pod
metadata:
  name: gpu-inference
spec:
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  containers:
  - name: inference
    image: gcr.io/myproject/inference:latest
    resources:
      limits:
        nvidia.com/gpu: "1"         # request 1 GPU
    env:
    - name: NVIDIA_VISIBLE_DEVICES
      value: all
---
# TPU Pod
apiVersion: v1
kind: Pod
metadata:
  name: tpu-training
spec:
  containers:
  - name: trainer
    image: gcr.io/myproject/tpu-trainer:latest
    resources:
      limits:
        google.com/tpu: "4"         # TPU chips
```

---

## 5. Networking

### 🟢 Q11. How does VPC-native networking work in GKE?

**VPC-native** (alias IP) is the recommended networking mode in GKE. Each pod gets an alias IP from a secondary range in the VPC subnet — pods are real VPC citizens.

```bash
# Create VPC with secondary ranges for GKE
gcloud compute networks create my-vpc --subnet-mode custom

gcloud compute networks subnets create gke-subnet \
  --network my-vpc \
  --region us-central1 \
  --range 10.0.0.0/24 \
  --secondary-range pods=10.4.0.0/14,services=10.0.16.0/20

# Create VPC-native cluster using these ranges
gcloud container clusters create myCluster \
  --region us-central1 \
  --enable-ip-alias \
  --network my-vpc \
  --subnetwork gke-subnet \
  --cluster-secondary-range-name pods \
  --services-secondary-range-name services

# Benefits of VPC-native:
# - Pods reachable from other VPCs via VPC peering
# - No double NAT (unlike routes-based)
# - Firewall rules apply to pods directly
# - Compatible with private Google access
```

---

### 🟡 Q12. How do you configure GKE Ingress and Gateway API?

```bash
# Enable HTTP Load Balancing add-on (required for Ingress)
gcloud container clusters update myCluster \
  --region us-central1 \
  --update-addons=HttpLoadBalancing=ENABLED

# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

```yaml
# GKE Ingress (uses Google Cloud Load Balancer - Layer 7)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"                      # external GCLB
    # kubernetes.io/ingress.class: "gce-internal"           # internal GCLB
    kubernetes.io/ingress.global-static-ip-name: "myapp-ip" # pre-reserved static IP
    networking.gke.io/managed-certificates: "myapp-ssl"     # Google-managed cert
    kubernetes.io/ingress.allow-http: "false"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-svc
            port:
              number: 8080
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: myapi-svc
            port:
              number: 8080
---
# Google Managed SSL Certificate
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: myapp-ssl
spec:
  domains:
  - myapp.example.com
---
# Gateway API (modern, preferred over Ingress)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: external-http
  annotations:
    networking.gke.io/certmap: myapp-certmap
spec:
  gatewayClassName: gke-l7-global-external-managed    # GKE-managed GCLB
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: myapp-route
spec:
  parentRefs:
  - name: external-http
  hostnames:
  - "myapp.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: myapp-svc
      port: 8080
      weight: 90
    - name: myapp-svc-canary
      port: 8080
      weight: 10
```

---

### 🟡 Q13. How do you configure Private GKE clusters?

```bash
# Private cluster: nodes have no external IPs, control plane on private endpoint
gcloud container clusters create myPrivateCluster \
  --region us-central1 \
  --enable-private-nodes \                    # nodes no public IP
  --enable-private-endpoint \                 # API server only accessible via private IP
  --master-ipv4-cidr 172.16.0.0/28 \         # control plane VPC peering CIDR
  --master-authorized-networks 10.0.0.0/8 \  # IPs that can reach API server
  --network my-vpc \
  --subnetwork gke-subnet \
  --enable-ip-alias

# Access private cluster from Cloud Shell / VM in VPC
gcloud container clusters get-credentials myPrivateCluster \
  --region us-central1

# Set up Cloud NAT for internet egress (nodes need to pull images)
gcloud compute routers create my-router \
  --network my-vpc \
  --region us-central1

gcloud compute routers nats create my-nat \
  --router my-router \
  --region us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

---

### 🟡 Q14. How do you implement Network Policies in GKE?

```bash
# Enable network policy (Standard clusters)
gcloud container clusters update myCluster \
  --region us-central1 \
  --update-addons=NetworkPolicy=ENABLED

# Enable DatapathV2 (eBPF-based, Cilium - recommended for new clusters)
gcloud container clusters create myCluster \
  --region us-central1 \
  --enable-dataplane-v2              # Cilium-based eBPF dataplane

# In Autopilot: network policy always enabled via VPC firewall
```

```yaml
# GKE network policy with Dataplane V2 (supports FQDN egress)
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-google-apis
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
  - toFQDNs:
    - matchName: "storage.googleapis.com"
    - matchName: "*.googleapis.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
---
# Standard K8s NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-namespace
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}                # allow intra-namespace
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
  egress:
  - to:
    - podSelector: {}
  - to:
    - namespaceSelector: {}
    ports:
    - port: 53
      protocol: UDP
```

---

## 6. Storage

### 🟢 Q15. What are the GKE storage options and default StorageClasses?

```bash
# List GKE default storage classes
kubectl get storageclass

# GKE default storage classes:
# standard          → PD HDD (Regional or Zonal)
# standard-rwo      → PD balanced (ReadWriteOnce)
# premium-rwo       → PD SSD (ReadWriteOnce) ← recommended default
# standard-rwx      → Filestore (ReadWriteMany) - requires Filestore CSI
```

```yaml
# Custom StorageClass - Regional PD (zone redundant)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd      # replicated across 2 zones
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain
allowedTopologies:
- matchLabelExpressions:
  - key: topology.gke.io/zone
    values:
    - us-central1-a
    - us-central1-b
---
# PVC for a database (regional SSD)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: regional-ssd
  resources:
    requests:
      storage: 200Gi
```

---

### 🟡 Q16. How do you use Filestore (NFS) and GCSFuse for shared storage in GKE?

```bash
# Enable Filestore CSI driver
gcloud container clusters update myCluster \
  --update-addons=GcpFilestoreCsiDriver=ENABLED \
  --region us-central1

# Enable GCSFuse CSI driver (mount GCS buckets as volumes)
gcloud container clusters update myCluster \
  --update-addons=GcsFuseCsiDriver=ENABLED \
  --region us-central1
```

```yaml
# Filestore StorageClass (dynamic provisioning)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-standard
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: standard                     # standard | premium | enterprise
  network: my-vpc
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# PVC for shared storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
spec:
  accessModes:
  - ReadWriteMany                    # Filestore supports RWX
  storageClassName: filestore-standard
  resources:
    requests:
      storage: 1Ti
---
# GCSFuse - mount GCS bucket as volume (ML workloads)
apiVersion: v1
kind: Pod
metadata:
  name: ml-trainer
  annotations:
    gke-gcsfuse/volumes: "true"      # enable GCSFuse sidecar injection
spec:
  serviceAccountName: ml-sa          # needs storage.objectViewer on bucket
  containers:
  - name: trainer
    image: gcr.io/myproject/trainer:latest
    volumeMounts:
    - name: training-data
      mountPath: /data
      readOnly: true
    - name: model-output
      mountPath: /output
    resources:
      requests:
        cpu: "4"
        memory: "16Gi"
  volumes:
  - name: training-data
    csi:
      driver: gcsfuse.csi.storage.gke.io
      volumeAttributes:
        bucketName: my-training-data
        mountOptions: "implicit-dirs"
  - name: model-output
    csi:
      driver: gcsfuse.csi.storage.gke.io
      volumeAttributes:
        bucketName: my-model-output
```

---

## 7. Identity & Security

### 🟢 Q17. What is GKE Workload Identity and how do you configure it?

**Workload Identity** lets pods authenticate as Google service accounts (GSA) using Kubernetes service accounts — no JSON key files.

```bash
# Enable Workload Identity on existing cluster
gcloud container clusters update myCluster \
  --region us-central1 \
  --workload-pool=myproject.svc.id.goog

# Enable on node pool
gcloud container node-pools update primary-pool \
  --cluster myCluster \
  --region us-central1 \
  --workload-metadata=GKE_METADATA

# Create Google Service Account
gcloud iam service-accounts create myapp-gsa \
  --display-name "MyApp GSA"

# Grant GSA permission to GCS bucket
gsutil iam ch serviceAccount:myapp-gsa@myproject.iam.gserviceaccount.com:roles/storage.objectViewer \
  gs://my-bucket

# Bind K8s SA to Google SA (the federation link)
gcloud iam service-accounts add-iam-policy-binding \
  myapp-gsa@myproject.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:myproject.svc.id.goog[production/myapp-ksa]"
```

```yaml
# Kubernetes Service Account annotated with Google SA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-ksa
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: myapp-gsa@myproject.iam.gserviceaccount.com
---
# Pod uses the service account - GCP SDK auto-authenticates
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-ksa
  containers:
  - name: app
    image: gcr.io/myproject/myapp:latest
    # GCP credentials auto-injected via metadata server
    env:
    - name: GOOGLE_CLOUD_PROJECT
      value: myproject
```

---

### 🟡 Q18. How do you implement Binary Authorization in GKE?

**Binary Authorization** enforces container image signing policies — only signed images can be deployed.

```bash
# Enable Binary Authorization on cluster
gcloud container clusters update myCluster \
  --region us-central1 \
  --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE

# Create attestor (entity that signs images)
gcloud container binauthz attestors create qa-attestor \
  --attestation-authority-note-project myproject \
  --attestation-authority-note my-attestor-note \
  --description "QA team attestor"

# Create policy (require attestation from qa-attestor)
cat > binauthz-policy.yaml << EOF
globalPolicyEvaluationMode: ENABLE
defaultAdmissionRule:
  evaluationMode: REQUIRE_ATTESTATION
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  requireAttestationsBy:
  - projects/myproject/attestors/qa-attestor
clusterAdmissionRules:
  us-central1.myCluster:
    evaluationMode: REQUIRE_ATTESTATION
    enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
    requireAttestationsBy:
    - projects/myproject/attestors/qa-attestor
EOF

gcloud container binauthz policy import binauthz-policy.yaml

# Sign image after QA passes
gcloud container binauthz attestations sign-and-create \
  --artifact-url gcr.io/myproject/myapp@sha256:abc123 \
  --attestor qa-attestor \
  --attestor-project myproject \
  --keyversion projects/myproject/locations/global/keyRings/binauthz/cryptoKeyVersions/1
```

---

### 🟡 Q19. How do you use GKE Secrets with Secret Manager?

```bash
# Enable Secret Manager add-on
gcloud container clusters update myCluster \
  --region us-central1 \
  --update-addons=SecretManagerPlugin=ENABLED

# Or install Secrets Store CSI Driver with GCP provider
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true

gcloud container clusters get-credentials myCluster --region us-central1
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/secrets-store-csi-driver-provider-gcp/main/deploy/provider-gcp-plugin.yaml
```

```yaml
# SecretProviderClass for Google Secret Manager
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: gcp-secrets
  namespace: production
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/myproject/secrets/db-password/versions/latest"
        path: "db-password"
      - resourceName: "projects/myproject/secrets/api-key/versions/1"
        path: "api-key"
  secretObjects:
  - secretName: app-secrets
    type: Opaque
    data:
    - objectName: db-password
      key: password
```

---

### 🔴 Q20. How do you implement GKE security hardening (Shielded nodes, RBAC, Policy Controller)?

```bash
# Enable Shielded nodes (Secure Boot + vTPM + Integrity Monitoring)
gcloud container node-pools update primary-pool \
  --cluster myCluster \
  --region us-central1 \
  --shielded-secure-boot \
  --shielded-integrity-monitoring

# Enable Policy Controller (GKE Enterprise - OPA Gatekeeper)
gcloud container fleet policycontroller enable \
  --project myproject

# Apply CIS GKE Benchmark constraints bundle
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/gke-policy-library/main/bundles/cis-gke-v1.5.0/bundle.yaml

# Audit cluster security posture
gcloud container clusters describe myCluster \
  --region us-central1 \
  --format="value(securityPostureConfig)"
```

```yaml
# GKE Security Posture (built-in vulnerability scanning)
# Enabled via gcloud or Terraform
# securityPostureConfig:
#   mode: BASIC           # workload vulnerability scanning
#   vulnerabilityMode: VULNERABILITY_ENTERPRISE  # deep scanning

# Custom OPA constraint (Policy Controller)
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredAnnotations
metadata:
  name: require-team-annotation
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
  parameters:
    annotations:
    - key: team
    - key: cost-center
```

---

## 8. Scaling

### 🟢 Q21. How does Cluster Autoscaler work in GKE?

```bash
# Enable CA on existing node pool
gcloud container node-pools update primary-pool \
  --cluster myCluster \
  --region us-central1 \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 20

# Fine-tune CA profile
gcloud container clusters update myCluster \
  --region us-central1 \
  --autoscaling-profile optimize-utilization  # optimize-utilization | balanced

# Check CA activity
kubectl get events -n kube-system | grep cluster-autoscaler
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml

# Scale-down safety: nodes not removed if PDB prevents it
```

---

### 🟡 Q22. How do you configure HPA with custom metrics in GKE?

```bash
# Enable Custom Metrics Stackdriver Adapter (for Cloud Monitoring metrics in HPA)
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter_new_resource_model.yaml
```

```yaml
# HPA on Cloud Monitoring custom metric
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: External
    external:
      metric:
        name: pubsub.googleapis.com|subscription|num_undelivered_messages
        selector:
          matchLabels:
            resource.labels.subscription_id: myapp-subscription
      target:
        type: AverageValue
        averageValue: "30"            # scale out when >30 pending messages/pod
```

---

### 🟡 Q23. How do you configure VPA in GKE?

```bash
# VPA is available in GKE Standard and Autopilot (limited)
# Enable VPA in Standard cluster
gcloud container clusters update myCluster \
  --region us-central1 \
  --enable-vertical-pod-autoscaling
```

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 8
        memory: 16Gi
      controlledResources: ["cpu", "memory"]
```

---

## 9. Upgrades & Release Channels

### 🟢 Q24. What are GKE release channels and how do upgrades work?

| Channel | Kubernetes Version | Update Frequency | Best For |
|---------|------------------|-----------------|----------|
| **Rapid** | Latest GA | Weekly | Testing new features |
| **Regular** | N-1 | Monthly (recommended) | Most production |
| **Stable** | N-2 | Quarterly | Stability-critical |
| **Extended** | LTS versions | 24-month support | Long-term compliance |
| **None** | Manual | You control | Special cases only |

```bash
# Enroll cluster in release channel
gcloud container clusters update myCluster \
  --region us-central1 \
  --release-channel regular

# Manual upgrade (if not on channel)
# Upgrade control plane
gcloud container clusters upgrade myCluster \
  --region us-central1 \
  --master

# Upgrade node pool
gcloud container clusters upgrade myCluster \
  --region us-central1 \
  --node-pool primary-pool \
  --cluster-version 1.30.5-gke.1234

# Blue-green node pool upgrade (zero disruption)
gcloud container node-pools update primary-pool \
  --cluster myCluster \
  --region us-central1 \
  --node-pool-soak-duration 10m \
  --strategy BLUE_GREEN \
  --standard-rollout-policy batch-soak-duration=5m,batch-percentage=0.33

# Set maintenance window
gcloud container clusters update myCluster \
  --region us-central1 \
  --maintenance-window-start 2024-01-01T22:00:00Z \
  --maintenance-window-end 2024-01-02T06:00:00Z \
  --maintenance-window-recurrence "FREQ=WEEKLY;BYDAY=SU"
```

---

## 10. Monitoring & Observability

### 🟢 Q25. How do you enable and use Google Cloud Monitoring for GKE?

```bash
# Logging + Monitoring enabled by default in new GKE clusters
# Verify:
gcloud container clusters describe myCluster \
  --region us-central1 \
  --format="value(loggingConfig,monitoringConfig)"

# Enable all monitoring components
gcloud container clusters update myCluster \
  --region us-central1 \
  --logging=SYSTEM,WORKLOAD,API_SERVER,SCHEDULER,CONTROLLER_MANAGER,DATAPLANE_OBSERVABILITY \
  --monitoring=SYSTEM,WORKLOAD,API_SERVER,SCHEDULER,CONTROLLER_MANAGER,DAEMONSET,DEPLOYMENT,HPA
```

```python
# Cloud Logging query for pod errors (Python SDK)
from google.cloud import logging

client = logging.Client()
logger = client.logger("projects/myproject/logs/stdout")

# Equivalent Log Explorer query:
# resource.type="k8s_container"
# resource.labels.cluster_name="myCluster"
# severity>=ERROR
# labels."k8s-pod/app"="myapp"
```

---

### 🟡 Q26. How do you set up GKE Managed Prometheus?

```bash
# Enable Managed Prometheus (GMP)
gcloud container clusters update myCluster \
  --region us-central1 \
  --enable-managed-prometheus

# Verify: GMP operator and collector are running
kubectl get pods -n gmp-system
```

```yaml
# PodMonitoring CRD - scrape your app metrics
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: myapp-monitoring
  namespace: production
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
---
# ClusterPodMonitoring - cluster-wide scraping
apiVersion: monitoring.googleapis.com/v1
kind: ClusterPodMonitoring
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  endpoints:
  - port: metrics
    interval: 30s
---
# Rules (alert) via GMP
apiVersion: monitoring.googleapis.com/v1
kind: Rules
metadata:
  name: pod-alerts
  namespace: production
spec:
  groups:
  - name: pod-health
    interval: 1m
    rules:
    - alert: HighPodCPU
      expr: rate(container_cpu_usage_seconds_total{container!=""}[5m]) > 0.9
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} CPU > 90% for 10 minutes"
```

---

### 🔴 Q27. How do you implement distributed tracing with Cloud Trace on GKE?

```yaml
# Deploy OpenTelemetry Collector for Cloud Trace
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: gke-otel
  namespace: monitoring
spec:
  mode: sidecar
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch: {}
      memory_limiter:
        limit_mib: 200
    exporters:
      googlecloud:
        project: myproject
        log:
          default_log_name: opentelemetry.io/collector-exported-log
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [googlecloud]
        metrics:
          receivers: [otlp]
          processors: [batch]
          exporters: [googlecloud]
```

---

## 11. DevOps & CI/CD

### 🟢 Q28. How do you use Cloud Build to build and deploy to GKE?

```yaml
# cloudbuild.yaml
steps:
# Build image
- name: gcr.io/cloud-builders/docker
  args:
  - build
  - -t
  - gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA
  - -t
  - gcr.io/$PROJECT_ID/myapp:latest
  - .

# Push to GCR / Artifact Registry
- name: gcr.io/cloud-builders/docker
  args: [push, gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA]

# Deploy to GKE
- name: gcr.io/google.com/cloudsdktool/cloud-sdk
  entrypoint: gcloud
  args:
  - container
  - clusters
  - get-credentials
  - myCluster
  - --region
  - us-central1
  - --project
  - $PROJECT_ID

- name: gcr.io/cloud-builders/kubectl
  args:
  - set
  - image
  - deployment/myapp
  - myapp=gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA
  - -n
  - production

# Or Helm upgrade
- name: gcr.io/cloud-builders/helm
  args:
  - upgrade
  - --install
  - myapp
  - ./helm/myapp
  - --set
  - image.tag=$COMMIT_SHA
  - --namespace
  - production
  - --wait

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: E2_HIGHCPU_8

timeout: 600s
```

---

### 🟡 Q29. How do you use Cloud Deploy for progressive delivery to GKE?

```yaml
# clouddeploy.yaml - delivery pipeline definition
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: myapp-pipeline
  location: us-central1
description: MyApp delivery pipeline
serialPipeline:
  stages:
  - targetId: staging
    profiles: [staging]
    strategy:
      canary:
        runtimeConfig:
          kubernetes:
            serviceNetworking:
              service: myapp-svc
              deployment: myapp
        canaryDeployment:
          percentages: [25, 50, 75]
          verify: true
  - targetId: production
    profiles: [production]
    strategy:
      standard:
        verify: true
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: staging
  location: us-central1
gke:
  cluster: projects/myproject/locations/us-central1/clusters/staging-cluster
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: production
  location: us-central1
requireApproval: true              # manual approval gate
gke:
  cluster: projects/myproject/locations/us-central1/clusters/prod-cluster
```

```bash
# Create release (triggers pipeline)
gcloud deploy releases create myapp-v1-0-0 \
  --delivery-pipeline myapp-pipeline \
  --region us-central1 \
  --images myapp=gcr.io/myproject/myapp:v1.0.0

# Promote to next stage manually
gcloud deploy rollouts promote \
  --delivery-pipeline myapp-pipeline \
  --release myapp-v1-0-0 \
  --region us-central1

# Check rollout status
gcloud deploy rollouts list \
  --delivery-pipeline myapp-pipeline \
  --release myapp-v1-0-0 \
  --region us-central1
```

---

### 🟡 Q30. How does Config Sync (GitOps) work with GKE?

```bash
# Enable Config Sync (part of GKE Enterprise / Fleet)
gcloud beta container fleet config-management enable

gcloud beta container fleet config-management apply \
  --membership=myCluster-us-central1 \
  --config=config-sync.yaml

# Or via kubectl (standalone Config Sync)
kubectl apply -f https://github.com/GoogleContainerTools/kpt/releases/download/configsync/config-sync-operator.yaml
```

```yaml
# config-sync.yaml
applySpecVersion: 1
spec:
  configSync:
    enabled: true
    sourceFormat: unstructured       # unstructured | hierarchy
    syncRepo: https://github.com/myorg/fleet-config
    syncBranch: main
    secretType: gcpserviceaccount   # or none | token | ssh
    gcpServiceAccountEmail: config-sync-sa@myproject.iam.gserviceaccount.com
    policyDir: clusters/prod
    syncWait: 15
  policyController:
    enabled: true
    auditIntervalSeconds: 60
    referentialRulesEnabled: true
```

---

## 12. Cost Optimization

### 🟡 Q31. What are the key GKE cost optimization strategies?

```bash
# 1. Use Spot VMs (up to 91% savings for preemptible workloads)
gcloud container node-pools create spot-pool --spot ...

# 2. Autopilot: pay per pod (no idle node charges)
gcloud container clusters create-auto myCluster ...

# 3. Committed Use Discounts (CUD) for baseline nodes
# Purchase 1yr/3yr CUD for Compute Engine in GCP console

# 4. Node Auto-Provisioning (NAP) - right-sized nodes
gcloud container clusters update myCluster --enable-autoprovisioning ...

# 5. Scale to zero with KEDA + Spot for batch
# minReplicaCount: 0 in ScaledObject

# 6. GKE Cost Breakdown (via GCP Billing)
# Labels are key: propagate namespace/team labels to cost center

# 7. Autopilot Scale-Out compute class (cheapest for stateless)
# annotation: cloud.google.com/compute-class: "Scale-Out"

# 8. ARM/Tau nodes (15-20% cheaper)
gcloud container node-pools create arm-pool \
  --machine-type t2a-standard-4

# 9. Enable cluster cost monitoring
gcloud container clusters update myCluster \
  --enable-cost-management-config
```

---

## 13. Multi-Cluster & Anthos

### 🟡 Q32. How do you manage multiple GKE clusters with Fleet (formerly Anthos)?

```bash
# Register clusters to Fleet
gcloud container fleet memberships register myCluster \
  --gke-cluster us-central1/myCluster \
  --enable-workload-identity

# View all fleet clusters
gcloud container fleet memberships list

# Multi-cluster Ingress (traffic across clusters)
gcloud container fleet ingress enable \
  --config-membership projects/myproject/locations/global/memberships/myCluster

# Multi-cluster Services (service discovery across clusters)
gcloud container fleet multi-cluster-services enable

# Config Sync across all fleet clusters
gcloud beta container fleet config-management apply \
  --membership=myCluster \
  --config=policy.yaml

# Anthos Service Mesh (managed Istio across clusters)
gcloud container fleet mesh enable
gcloud container fleet mesh update \
  --management automatic \
  --memberships myCluster
```

---

## 14. Advanced Topics

### 🔴 Q33. How do you enable Confidential Nodes in GKE?

**Confidential GKE Nodes** use AMD SEV (Secure Encrypted Virtualization) to encrypt VM memory in hardware.

```bash
# Create cluster with Confidential Nodes
gcloud container clusters create myConfidentialCluster \
  --zone us-central1-a \
  --machine-type n2d-standard-2 \   # requires N2D (AMD EPYC)
  --enable-confidential-nodes

# Add confidential node pool to existing cluster
gcloud container node-pools create confidential-pool \
  --cluster myCluster \
  --zone us-central1-a \
  --machine-type n2d-standard-4 \
  --enable-confidential-nodes \
  --enable-shielded-nodes \
  --shielded-secure-boot
```

---

### 🔴 Q34. What is GKE Dataplane V2 (eBPF/Cilium) and when to use it?

```bash
# GKE Dataplane V2: Cilium-based eBPF dataplane
# Features over standard kube-proxy:
# - L7 network policy (HTTP, gRPC, DNS)
# - FQDN-based egress policies
# - Built-in network observability (Hubble)
# - Better performance (bypass iptables)

# Enable at cluster creation (cannot be changed later)
gcloud container clusters create myCluster \
  --region us-central1 \
  --enable-dataplane-v2 \
  --enable-network-policy \
  --enable-ip-alias

# View Hubble network flow observability
kubectl exec -n kube-system daemonset/cilium -c cilium-agent -- \
  hubble observe --type drop --last 50
```

---

### 🔴 Q35. How do you run large-scale AI/ML workloads on GKE with A3 nodes?

```bash
# A3 Mega nodes: 8x H100 80GB GPUs, 200 Gbps RDMA networking
gcloud container node-pools create a3-mega-pool \
  --cluster myCluster \
  --zone us-central1-a \
  --machine-type a3-megagpu-8g \
  --accelerator type=nvidia-h100-mega-80gb,count=8,gpu-driver-version=latest \
  --num-nodes 4 \
  --placement-policy myRDMAPlacementPolicy \  # compact placement for RDMA
  --node-taints nvidia.com/gpu=present:NoSchedule
```

```yaml
# Distributed training with topology-aware scheduling
apiVersion: batch/v1
kind: Job
metadata:
  name: llm-training
spec:
  completions: 32
  parallelism: 32
  template:
    spec:
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: cloud.google.com/gke-nodepool
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            job-name: llm-training
      containers:
      - name: trainer
        image: gcr.io/myproject/llm-trainer:latest
        resources:
          limits:
            nvidia.com/gpu: "8"
        env:
        - name: NCCL_DEBUG
          value: INFO
        - name: NCCL_IB_DISABLE
          value: "0"              # enable InfiniBand (RDMA)
```

---

## 15. Troubleshooting Scenarios

### 🟡 Q36. How do you troubleshoot pod scheduling failures in GKE?

```bash
# Step 1: Describe pod
kubectl describe pod <pod> -n <ns>
# Look for Events section - "FailedScheduling"

# Common reasons:
# "Insufficient cpu/memory" → Cluster Autoscaler should add nodes
# "node(s) had taint" → pod needs toleration
# "0/3 nodes are available: 3 Insufficient nvidia.com/gpu" → wrong node pool

# Step 2: Check if CA is working
kubectl get events -n kube-system --field-selector reason=TriggeredScaleUp

# Step 3: Check CA status
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml

# Step 4: Force scale-up (manual test)
kubectl run test --image=nginx --requests=cpu=100m --limits=cpu=200m

# GKE specific: check node auto-provisioning
gcloud container operations list --filter="operationType=CREATE_NODE_POOL"

# Step 5: Spot preemption causing issues?
kubectl get events -A --field-selector reason=SpotPreemption
```

---

### 🔴 Q37. How do you troubleshoot GKE networking and Cloud Load Balancer issues?

```bash
# Test Service DNS
kubectl run test --image=busybox --rm -it -- \
  nslookup myapp-svc.production.svc.cluster.local

# Check Ingress backend health
kubectl describe ingress myapp-ingress -n production
# Look for: "Backends" section - healthy/unhealthy count

# GCP Load Balancer health checks
gcloud compute backend-services list
gcloud compute backend-services get-health <backend-service> --global

# Check GCP firewall rules (common issue: health check blocked)
gcloud compute firewall-rules list \
  --filter="network=my-vpc" \
  --format="table(name,direction,sourceRanges,targetTags,allowed)"

# GKE Dataplane V2: use Hubble for flow analysis
kubectl exec -n kube-system daemonset/cilium -- \
  hubble observe --namespace production --type drop

# Check VPC flow logs
gcloud logging read \
  'resource.type="gce_subnetwork" AND jsonPayload.disposition="DENIED"' \
  --limit 50 \
  --format json
```

---

## Master Cheatsheet

### Cluster Operations
```bash
# Create
gcloud container clusters create myCluster --region us-central1 --num-nodes 3
gcloud container clusters create-auto myAutopilot --region us-central1    # Autopilot

# Credentials
gcloud container clusters get-credentials myCluster --region us-central1

# Info
gcloud container clusters describe myCluster --region us-central1
gcloud container clusters list

# Upgrade
gcloud container clusters upgrade myCluster --region us-central1 --master
gcloud container clusters upgrade myCluster --region us-central1 --node-pool primary-pool

# Delete
gcloud container clusters delete myCluster --region us-central1
```

### Node Pools
```bash
gcloud container node-pools create spot-pool --cluster myCluster --region us-central1 \
  --spot --machine-type e2-standard-4 --num-nodes 2 --enable-autoscaling --min-nodes 0 --max-nodes 20
gcloud container node-pools delete pool-name --cluster myCluster --region us-central1
gcloud container node-pools resize pool-name --cluster myCluster --region us-central1 --num-nodes 5
gcloud container node-pools list --cluster myCluster --region us-central1
```

### Identity (Workload Identity)
```bash
# Enable
gcloud container clusters update myCluster --region us-central1 --workload-pool=PROJECT.svc.id.goog
gcloud container node-pools update pool --cluster myCluster --region us-central1 --workload-metadata=GKE_METADATA

# Bind K8s SA to Google SA
gcloud iam service-accounts add-iam-policy-binding GSA@PROJECT.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT.svc.id.goog[NAMESPACE/KSA]"

# Annotate K8s SA
kubectl annotate serviceaccount KSA -n NAMESPACE \
  iam.gke.io/gcp-service-account=GSA@PROJECT.iam.gserviceaccount.com
```

### Add-ons
```bash
gcloud container clusters update myCluster --region us-central1 \
  --update-addons=GcePersistentDiskCsiDriver=ENABLED,GcsFuseCsiDriver=ENABLED,HttpLoadBalancing=ENABLED
```

### Scaling
```bash
kubectl autoscale deployment myapp --cpu-percent=70 --min=2 --max=20
gcloud container node-pools update pool --cluster myCluster --region us-central1 \
  --enable-autoscaling --min-nodes 1 --max-nodes 20
gcloud container clusters update myCluster --region us-central1 \
  --enable-autoprovisioning --max-cpu 500 --max-memory 2000
```

### Monitoring
```bash
gcloud container clusters update myCluster --region us-central1 \
  --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM,WORKLOAD
gcloud container clusters update myCluster --region us-central1 --enable-managed-prometheus
kubectl get pods -n gmp-system
```

### Storage Quick Reference
```bash
# Storage classes
kubectl get storageclass
# standard-rwo    → Balanced PD (default)
# premium-rwo     → SSD PD
# standard-rwx    → Filestore NFS (ReadWriteMany)

# GCSFuse volumes
# annotation: gke-gcsfuse/volumes: "true"
# CSI driver: gcsfuse.csi.storage.gke.io
```

### Security
```bash
# Binary Authorization
gcloud container clusters update myCluster --region us-central1 \
  --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE

# Security posture
gcloud container clusters describe myCluster --region us-central1 \
  --format="value(securityPostureConfig)"

# Shielded nodes
gcloud container node-pools update pool --cluster myCluster --region us-central1 \
  --shielded-secure-boot --shielded-integrity-monitoring
```

---

## Question Coverage Index

| # | Question | Level | Topic |
|---|----------|-------|-------|
| Q1 | GKE architecture & Autopilot vs Standard overview | 🟢 | Architecture |
| Q2 | Zonal vs Regional clusters | 🟢 | Architecture |
| Q3 | Container-Optimized OS (COS) | 🟡 | Node OS |
| Q4 | Autopilot compute classes + when to use | 🟡 | Autopilot |
| Q5 | Node Auto-Provisioning (NAP) | 🟡 | Scaling |
| Q6 | Production cluster creation parameters | 🟢 | Cluster Config |
| Q7 | Terraform for GKE | 🔴 | IaC |
| Q8 | Node pool management | 🟢 | Node Pools |
| Q9 | Spot VMs for cost savings | 🟡 | Cost |
| Q10 | GPU and TPU node pools | 🟡 | Accelerators |
| Q11 | VPC-native networking (alias IP) | 🟢 | Networking |
| Q12 | GKE Ingress + Gateway API | 🟡 | Networking |
| Q13 | Private GKE clusters + Cloud NAT | 🟡 | Networking |
| Q14 | Network Policies + Dataplane V2 | 🟡 | Networking |
| Q15 | Storage options + StorageClasses | 🟢 | Storage |
| Q16 | Filestore (NFS) + GCSFuse | 🟡 | Storage |
| Q17 | Workload Identity (GSA + KSA binding) | 🟢 | Identity |
| Q18 | Binary Authorization | 🟡 | Security |
| Q19 | Secret Manager + CSI driver | 🟡 | Security |
| Q20 | Shielded nodes + Policy Controller | 🔴 | Security |
| Q21 | Cluster Autoscaler | 🟢 | Scaling |
| Q22 | HPA with Cloud Monitoring custom metrics | 🟡 | Scaling |
| Q23 | VPA | 🟡 | Scaling |
| Q24 | Release channels + upgrade strategies | 🟢 | Upgrades |
| Q25 | Cloud Monitoring + Cloud Logging | 🟢 | Monitoring |
| Q26 | GKE Managed Prometheus (GMP) | 🟡 | Monitoring |
| Q27 | Cloud Trace + OpenTelemetry | 🔴 | Monitoring |
| Q28 | Cloud Build CI/CD | 🟢 | DevOps |
| Q29 | Cloud Deploy progressive delivery | 🟡 | DevOps |
| Q30 | Config Sync (GitOps) + Fleet | 🟡 | DevOps |
| Q31 | Cost optimization strategies | 🟡 | Cost |
| Q32 | Fleet + Multi-cluster Services + ASM | 🟡 | Multi-cluster |
| Q33 | Confidential Nodes (AMD SEV) | 🔴 | Security |
| Q34 | GKE Dataplane V2 (eBPF/Cilium) | 🔴 | Networking |
| Q35 | A3 GPU nodes for AI/ML workloads | 🔴 | Advanced |
| Q36 | Troubleshoot pod scheduling failures | 🟡 | Troubleshooting |
| Q37 | Troubleshoot networking + Cloud LB | 🔴 | Troubleshooting |

---
*Generated from Google Cloud official GKE documentation — June 2026*  
*Key refs: cloud.google.com/kubernetes-engine/docs | cloud.google.com/kubernetes-engine/networking | cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview*

---

# PART 2 — Additional Topics (Gap Fill from Official GKE Docs)

---

## 16. GKE Sandbox (gVisor)

### 🔴 Q38. What is GKE Sandbox and when should you use it?

**GKE Sandbox** uses **gVisor** — a user-space kernel that intercepts syscalls — to provide a second layer of isolation for untrusted or multi-tenant workloads.

```bash
# Enable GKE Sandbox on a node pool
gcloud container node-pools create sandbox-pool \
  --cluster myCluster \
  --region us-central1 \
  --machine-type n2-standard-4 \
  --sandbox type=gvisor \
  --num-nodes 2 \
  --node-taints sandbox.gke.io/runtime=gvisor:NoSchedule
```

```yaml
# Pod using gVisor sandbox runtime
apiVersion: v1
kind: Pod
metadata:
  name: untrusted-app
spec:
  runtimeClassName: gvisor          # use gVisor runtime
  tolerations:
  - key: sandbox.gke.io/runtime
    value: gvisor
    effect: NoSchedule
  containers:
  - name: app
    image: gcr.io/myproject/untrusted-app:latest
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
---
# RuntimeClass (auto-created by GKE when sandbox enabled)
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc             # gVisor's OCI runtime
scheduling:
  nodeClassification:
    tolerations:
    - key: sandbox.gke.io/runtime
      operator: Equal
      value: gvisor
      effect: NoSchedule
```

---

## 17. Windows Nodes on GKE

### 🟡 Q39. How do you add Windows Server node pools to GKE?

```bash
# GKE supports Windows Server 2019 and 2022
# Requirements: VPC-native cluster, no Dataplane V2 (limited support)

# Add Windows node pool
gcloud container node-pools create windows-pool \
  --cluster myCluster \
  --region us-central1 \
  --image-type WINDOWS_LTSC_CONTAINERD \    # Windows Server 2019 LTSC
  # --image-type WINDOWS_SAC_CONTAINERD     # Windows Server 2022 SAC
  --machine-type n2-standard-4 \
  --num-nodes 2 \
  --disk-size 128GB \
  --no-enable-autoupgrade                   # Windows pool upgrade is manual
```

```yaml
# Windows pod spec
apiVersion: apps/v1
kind: Deployment
metadata:
  name: win-iis
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows
      tolerations:
      - key: node.kubernetes.io/os
        operator: Equal
        value: windows
        effect: NoSchedule
      containers:
      - name: iis
        image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2019
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
```

---

## 18. Cloud Armor WAF with GKE

### 🟡 Q40. How do you integrate Google Cloud Armor WAF with GKE Ingress?

```bash
# Create Cloud Armor security policy
gcloud compute security-policies create myapp-waf-policy \
  --description "WAF policy for myapp"

# Add OWASP Top 10 preconfigured rules
gcloud compute security-policies rules create 1000 \
  --security-policy myapp-waf-policy \
  --expression "evaluatePreconfiguredExpr('xss-v33-stable')" \
  --action deny-403 \
  --description "Block XSS attacks"

gcloud compute security-policies rules create 1001 \
  --security-policy myapp-waf-policy \
  --expression "evaluatePreconfiguredExpr('sqli-v33-stable')" \
  --action deny-403 \
  --description "Block SQL injection"

# Rate limiting rule
gcloud compute security-policies rules create 2000 \
  --security-policy myapp-waf-policy \
  --expression "true" \
  --action throttle \
  --rate-limit-threshold-count 100 \
  --rate-limit-threshold-interval-sec 60 \
  --conform-action allow \
  --exceed-action deny-429 \
  --enforce-on-key IP
```

```yaml
# BackendConfig CRD: attach Cloud Armor to GKE service
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: myapp-backend-config
  namespace: production
spec:
  securityPolicy:
    name: "myapp-waf-policy"         # Cloud Armor policy name
  timeoutSec: 30
  connectionDraining:
    drainingTimeoutSec: 60
  healthCheck:
    checkIntervalSec: 10
    timeoutSec: 5
    healthyThreshold: 1
    unhealthyThreshold: 3
    type: HTTP
    requestPath: /health
    port: 8080
  sessionAffinity:
    affinityType: "GENERATED_COOKIE"
    affinityCookieTtlSec: 3600
---
# Service with BackendConfig annotation
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  namespace: production
  annotations:
    cloud.google.com/backend-config: '{"default": "myapp-backend-config"}'
    cloud.google.com/neg: '{"ingress": true}'         # enable NEGs
spec:
  type: NodePort                    # use NodePort with NEG (not LoadBalancer)
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

---

## 19. Network Endpoint Groups (NEGs) — Container-Native Load Balancing

### 🟡 Q41. What are NEGs and why are they better than NodePort in GKE?

**NEGs** (Network Endpoint Groups) allow the Google Cloud Load Balancer to route traffic **directly to pod IPs** — bypassing kube-proxy and NodePort. This gives lower latency and better health checking.

```yaml
# Enable NEG on service (container-native load balancing)
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  annotations:
    cloud.google.com/neg: '{"ingress": true}'         # auto-create NEG for Ingress
    # cloud.google.com/neg: '{"exposed_ports":{"80":{}}}'  # manual NEG
spec:
  type: ClusterIP                   # ClusterIP + NEG = container-native LB
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Verify NEG was created
gcloud compute network-endpoint-groups list
# NAME                   LOCATION        ENDPOINT-TYPE   SIZE
# k8s1-xxxxx-prod-myapp  us-central1-a   GCE_VM_IP_PORT  3

# Inspect NEG endpoints (pod IPs)
gcloud compute network-endpoint-groups list-network-endpoints \
  k8s1-xxxxx-prod-myapp \
  --zone us-central1-a
```

---

## 20. Config Connector

### 🟡 Q42. How do you use Config Connector to manage GCP resources from Kubernetes?

**Config Connector** is the GKE equivalent of AWS ACK — manage GCP resources (Cloud SQL, PubSub, BigQuery) using Kubernetes CRDs.

```bash
# Enable Config Connector add-on
gcloud container clusters update myCluster \
  --region us-central1 \
  --update-addons ConfigConnector=ENABLED

# Create namespace with Config Connector annotation
kubectl annotate namespace production \
  cnrm.cloud.google.com/project-id=myproject
```

```yaml
# Create a Cloud SQL instance via Kubernetes CRD
apiVersion: sql.cnrm.cloud.google.com/v1beta1
kind: SQLInstance
metadata:
  name: myapp-postgres
  namespace: production
spec:
  region: us-central1
  databaseVersion: POSTGRES_15
  settings:
    tier: db-n1-standard-2
    availabilityType: REGIONAL       # HA with failover replica
    backupConfiguration:
      enabled: true
      startTime: "02:00"
      transactionLogRetentionDays: 7
    ipConfiguration:
      ipv4Enabled: false
      privateNetworkRef:
        name: my-vpc
---
# Create a Pub/Sub topic
apiVersion: pubsub.cnrm.cloud.google.com/v1beta1
kind: PubSubTopic
metadata:
  name: orders-topic
  namespace: production
spec:
  resourceID: orders
---
# Create a GCS bucket
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  name: my-app-data
  namespace: production
spec:
  location: US
  uniformBucketLevelAccess: true
  versioning:
    enabled: true
```

---

## 21. Backup for GKE

### 🟡 Q43. How do you back up and restore GKE workloads with Backup for GKE?

```bash
# Enable Backup for GKE add-on
gcloud container clusters update myCluster \
  --region us-central1 \
  --update-addons=BackupRestore=ENABLED

# Create backup plan
gcloud beta container backup-restore backup-plans create daily-backup \
  --project myproject \
  --location us-central1 \
  --cluster projects/myproject/locations/us-central1/clusters/myCluster \
  --all-namespaces \
  --include-secrets \
  --include-volume-data \
  --cron-schedule "0 2 * * *" \
  --backup-retain-days 7 \
  --description "Daily cluster backup"

# On-demand backup
gcloud beta container backup-restore backups create manual-backup-$(date +%Y%m%d) \
  --project myproject \
  --location us-central1 \
  --backup-plan daily-backup

# List backups
gcloud beta container backup-restore backups list \
  --project myproject \
  --location us-central1 \
  --backup-plan daily-backup

# Restore to a different cluster (disaster recovery)
gcloud beta container backup-restore restores create my-restore \
  --project myproject \
  --location us-central1 \
  --restore-plan my-restore-plan \
  --backup projects/myproject/locations/us-central1/backupPlans/daily-backup/backups/manual-backup-20260611
```

---

## 22. GKE Multi-Tenancy Deep Dive

### 🔴 Q44. How do you implement enterprise multi-tenancy on GKE?

```bash
# GKE best practice: separate projects + clusters per team (hard isolation)
# OR: namespace isolation within shared cluster (soft isolation)

# Create per-team namespace with hierarchy (using Hierarchical Namespace Controller)
kubectl hns create team-payments -n root
kubectl hns create team-orders -n root
```

```yaml
# Full multi-tenant namespace setup for GKE
---
# Namespace with Workload Identity
apiVersion: v1
kind: Namespace
metadata:
  name: team-payments
  labels:
    pod-security.kubernetes.io/enforce: restricted
    team: payments
---
# Team-scoped service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-app-sa
  namespace: team-payments
  annotations:
    iam.gke.io/gcp-service-account: payments-sa@myproject.iam.gserviceaccount.com
---
# ResourceQuota per team
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: team-payments
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services.loadbalancers: "2"
    count/deployments.apps: "30"
    persistentvolumeclaims: "20"
---
# RBAC: team gets admin on their namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-team-admin
  namespace: team-payments
subjects:
- kind: Group
  name: payments-team@myproject.iam.gserviceaccount.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
---
# NetworkPolicy isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: team-payments
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector: {}              # intra-namespace
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
  egress:
  - to: [{podSelector: {}}]
  - to: [{}]
    ports:
    - port: 53
      protocol: UDP
    - port: 443
      protocol: TCP
```

---

## 23. GKE Shared VPC

### 🔴 Q45. How do you deploy GKE in a Shared VPC?

```bash
# Shared VPC: host project owns VPC, service projects run GKE
# Use case: centralized networking, separate billing per team

# In host project: grant GKE service account permissions on Shared VPC
gcloud projects add-iam-policy-binding HOST_PROJECT_ID \
  --member "serviceAccount:service-SERVICE_PROJECT_NUMBER@container-engine-robot.iam.gserviceaccount.com" \
  --role roles/container.hostServiceAgentUser

gcloud projects add-iam-policy-binding HOST_PROJECT_ID \
  --member "serviceAccount:SERVICE_PROJECT_NUMBER@cloudservices.gserviceaccount.com" \
  --role roles/compute.networkUser

# Create GKE cluster in service project using host VPC
gcloud container clusters create sharedvpc-cluster \
  --region us-central1 \
  --network projects/HOST_PROJECT_ID/global/networks/shared-vpc \
  --subnetwork projects/HOST_PROJECT_ID/regions/us-central1/subnetworks/gke-subnet \
  --cluster-secondary-range-name pods \
  --services-secondary-range-name services \
  --enable-ip-alias \
  --project SERVICE_PROJECT_ID
```

---

## 24. Artifact Registry Integration

### 🟢 Q46. How do you use Artifact Registry (not GCR) with GKE?

```bash
# Artifact Registry replaced Container Registry (GCR) - recommended
# GKE nodes auto-authenticate to Artifact Registry in same project

# Create Artifact Registry repository
gcloud artifacts repositories create myapp-images \
  --repository-format docker \
  --location us-central1 \
  --description "App container images"

# Build and push image
gcloud builds submit \
  --tag us-central1-docker.pkg.dev/myproject/myapp-images/myapp:latest .

# Configure Docker authentication for Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev

# For cross-project access: grant Artifact Registry reader to GKE node SA
gcloud projects add-iam-policy-binding SOURCE_PROJECT \
  --member "serviceAccount:GKE_NODE_SA@TARGET_PROJECT.iam.gserviceaccount.com" \
  --role roles/artifactregistry.reader

# Scan images for vulnerabilities
gcloud artifacts docker images scan \
  us-central1-docker.pkg.dev/myproject/myapp-images/myapp:latest \
  --location us-central1

# Set up Artifact Registry remote repository (mirror Docker Hub)
gcloud artifacts repositories create docker-hub-mirror \
  --repository-format docker \
  --location us-central1 \
  --mode remote-repository \
  --remote-repo-config-desc "Docker Hub" \
  --disable-vulnerability-scanning
```

---

## 25. GKE Node Auto-Repair

### 🟢 Q47. How does GKE node auto-repair work?

```bash
# Node auto-repair: automatically repairs/recreates unhealthy nodes
# Enabled by default on Standard clusters

# Enable explicitly on node pool
gcloud container node-pools update primary-pool \
  --cluster myCluster \
  --region us-central1 \
  --enable-autorepair

# Auto-repair triggers when node is NotReady >10 min or unreachable >10 min

# Check auto-repair events
gcloud container operations list \
  --filter="operationType=REPAIR_CLUSTER" \
  --format="table(name,operationType,status,targetLink)"

# Node pool repair settings
gcloud container node-pools describe primary-pool \
  --cluster myCluster \
  --region us-central1 \
  --format="value(management.autoRepair,management.autoUpgrade)"
```

---

## 26. Image Streaming

### 🟡 Q48. What is GKE Image Streaming and how does it speed up pod startup?

```bash
# Image Streaming: start containers before full image download completes
# Requires Artifact Registry (not Docker Hub or GCR)
# Supported on COS_CONTAINERD image type

# Enable Image Streaming on cluster
gcloud container clusters update myCluster \
  --region us-central1 \
  --enable-image-streaming

# Enable on node pool
gcloud container node-pools update primary-pool \
  --cluster myCluster \
  --region us-central1 \
  --enable-image-streaming

# Image Streaming reduces pod startup time from minutes → seconds
# Especially impactful for large ML/AI images (5-20GB+)
# Monitor streaming metrics
gcloud logging read \
  'resource.type=k8s_node AND jsonPayload.MESSAGE=~"imagestreaming"' \
  --limit 20
```

---

## 27. Cloud Logging Advanced Queries

### 🟡 Q49. How do you use Cloud Logging and structured queries for GKE troubleshooting?

```bash
# GKE sends all pod stdout/stderr to Cloud Logging automatically
# Log Explorer queries

# All ERROR logs from a specific namespace
gcloud logging read \
  'resource.type="k8s_container"
   resource.labels.cluster_name="myCluster"
   resource.labels.namespace_name="production"
   severity>=ERROR' \
  --limit 100 \
  --format json

# OOMKilled events
gcloud logging read \
  'resource.type="k8s_node"
   jsonPayload.reason="OOMKilling"' \
  --limit 50
```

```
# Log Explorer query examples:

# Find pod restart events
resource.type="k8s_pod"
resource.labels.cluster_name="myCluster"
jsonPayload.reason="BackOff"

# Control plane audit log: who deleted a deployment
resource.type="k8s_cluster"
protoPayload.methodName="io.k8s.apps.v1.deployments.delete"
protoPayload.authenticationInfo.principalEmail!=""

# Slow HTTP requests via Istio
resource.type="k8s_container"
resource.labels.container_name="istio-proxy"
jsonPayload.response_code="503"
jsonPayload.duration>1000

# GKE Audit log: privileged pod creation attempts
resource.type="k8s_cluster"
protoPayload.request.spec.containers.securityContext.privileged=true
```

---

## 28. Troubleshoot OOMKilled & Disk Pressure

### 🟡 Q50. How do you troubleshoot OOMKilled and disk pressure in GKE?

```bash
# Find OOMKilled pods
kubectl get pods -A --field-selector=status.phase=Failed | grep OOMKilled
kubectl get pods -A -o json | jq '.items[] | select(.status.containerStatuses[]?.lastState.terminated.reason=="OOMKilled") | .metadata.name'

# Get memory usage trend
kubectl top pods -n production --containers --sort-by=memory

# View OOM events in Cloud Logging
gcloud logging read \
  'resource.type="k8s_node" jsonPayload.reason="OOMKilling"' \
  --limit 20 --format="value(jsonPayload.MESSAGE)"

# Fix: increase memory limit
kubectl patch deployment myapp -n production \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"limits":{"memory":"2Gi"}}}]}}}}'

# Or use VPA to auto-right-size
kubectl get vpa myapp-vpa -n production -o jsonpath='{.status.recommendation}'

# Disk Pressure: node running out of disk space
kubectl describe node <node> | grep -A5 "Conditions:"
# DiskPressure = True → node evicting pods

# Check disk usage on node (via debug pod)
kubectl debug node/<node> -it --image=busybox -- df -h

# Fix disk pressure:
# 1. Enable Image Cleaner (removes unused images)
gcloud container clusters update myCluster \
  --region us-central1 \
  --enable-image-streaming   # images stay in AR, not fully pulled to disk

# 2. Increase node disk size
gcloud container node-pools update primary-pool \
  --cluster myCluster \
  --region us-central1 \
  --disk-size 200GB           # will recreate nodes
```

---

## 29. GKE Workload Separation

### 🔴 Q51. How do you implement workload separation in GKE Autopilot?

```yaml
# Workload separation: guarantee batch-job pods never share nodes with web-server pods
# Use case: compliance isolation, noisy-neighbor prevention

# Deployment with workload separation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sensitive-app
spec:
  template:
    metadata:
      labels:
        app: sensitive-app
      annotations:
        # Autopilot: guarantee exclusive node
        cloud.google.com/gke-isolation-mode: "dedicated"
    spec:
      # Use podAntiAffinity to separate from other apps
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: security-tier
                operator: NotIn
                values: ["sensitive"]
            topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: gcr.io/myproject/sensitive-app:latest
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
```

---

## 30. GKE IPv6 Dual-Stack

### 🔴 Q52. How do you enable IPv6 dual-stack on GKE?

```bash
# IPv6 dual-stack: pods get both IPv4 and IPv6 addresses
# Requires GKE 1.27+ and VPC with IPv6 enabled

# Create dual-stack subnet
gcloud compute networks subnets create gke-dualstack-subnet \
  --network my-vpc \
  --region us-central1 \
  --range 10.0.0.0/24 \
  --ipv6-access-type INTERNAL \
  --stack-type IPV4_IPV6 \
  --secondary-range pods=10.4.0.0/14,services=10.0.16.0/20

# Create dual-stack GKE cluster
gcloud container clusters create dualstack-cluster \
  --region us-central1 \
  --network my-vpc \
  --subnetwork gke-dualstack-subnet \
  --stack-type IPV4_IPV6 \
  --enable-ip-alias \
  --cluster-secondary-range-name pods \
  --services-secondary-range-name services

# Verify dual-stack is working
kubectl get nodes -o jsonpath='{.items[*].status.addresses}'
kubectl get pods -n production myapp -o jsonpath='{.status.podIPs}'
# Should show both IPv4 and IPv6 addresses
```

---

## Updated GKE Master Cheatsheet — Additional Commands

```bash
## SANDBOX (gVisor)
gcloud container node-pools create sandbox-pool --cluster myCluster --region us-central1 \
  --sandbox type=gvisor --machine-type n2-standard-4

## WINDOWS NODES
gcloud container node-pools create win-pool --cluster myCluster --region us-central1 \
  --image-type WINDOWS_LTSC_CONTAINERD --machine-type n2-standard-4

## CLOUD ARMOR
gcloud compute security-policies create myapp-waf-policy
gcloud compute security-policies rules create 1000 --security-policy myapp-waf-policy \
  --expression "evaluatePreconfiguredExpr('xss-v33-stable')" --action deny-403

## CONFIG CONNECTOR
gcloud container clusters update myCluster --region us-central1 --update-addons ConfigConnector=ENABLED
kubectl get sqlinstances.sql.cnrm.cloud.google.com -A
kubectl get storagebuckets.storage.cnrm.cloud.google.com -A

## BACKUP FOR GKE
gcloud beta container backup-restore backup-plans create daily-backup --project myproject \
  --location us-central1 --cluster ... --all-namespaces --cron-schedule "0 2 * * *"
gcloud beta container backup-restore backups create manual-$(date +%Y%m%d) ...

## IMAGE STREAMING
gcloud container clusters update myCluster --region us-central1 --enable-image-streaming

## AUTO-REPAIR
gcloud container node-pools update pool --cluster myCluster --region us-central1 --enable-autorepair

## ARTIFACT REGISTRY
gcloud artifacts repositories create myapp-images --repository-format docker --location us-central1
gcloud builds submit --tag us-central1-docker.pkg.dev/myproject/myapp-images/myapp:latest .

## SECURITY
gcloud container clusters update myCluster --region us-central1 \
  --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE
# Shielded nodes
gcloud container node-pools update pool --cluster myCluster --region us-central1 \
  --shielded-secure-boot --shielded-integrity-monitoring
```

---

## Additional Question Coverage Index (Part 2)

| # | Question | Level | Topic |
|---|----------|-------|-------|
| Q38 | GKE Sandbox (gVisor) workload isolation | 🔴 | Security |
| Q39 | Windows Server node pools | 🟡 | Compute |
| Q40 | Cloud Armor WAF + BackendConfig CRD | 🟡 | Security/Networking |
| Q41 | NEGs (container-native load balancing) | 🟡 | Networking |
| Q42 | Config Connector (manage GCP resources via K8s) | 🟡 | Operations |
| Q43 | Backup for GKE (native backup solution) | 🟡 | Operations |
| Q44 | Multi-tenancy deep dive (Shared VPC + Quotas) | 🔴 | Multi-tenancy |
| Q45 | Shared VPC GKE deployment | 🔴 | Networking |
| Q46 | Artifact Registry integration (replace GCR) | 🟢 | DevOps |
| Q47 | Node auto-repair behavior | 🟢 | Operations |
| Q48 | Image Streaming (faster pod startup) | 🟡 | Performance |
| Q49 | Cloud Logging advanced queries | 🟡 | Monitoring |
| Q50 | Troubleshoot OOMKilled + Disk Pressure | 🟡 | Troubleshooting |
| Q51 | Workload Separation (Autopilot) | 🔴 | Security |
| Q52 | IPv6 dual-stack clusters | 🔴 | Networking |

---
*Part 2 added June 2026 — gap-filled against Google Cloud official GKE documentation*
*Key refs: cloud.google.com/kubernetes-engine/docs | cloud.google.com/kubernetes-engine/networking*
