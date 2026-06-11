# Azure Kubernetes Service (AKS) — Complete Study Guide
> **Source-aligned with Microsoft Learn official docs (June 2026)**  
> Format: 🟢 Basic | 🟡 Intermediate | 🔴 Advanced | YAML/CLI examples | Master Cheatsheet

---

## Table of Contents
1. [AKS Core Concepts & Architecture](#1-aks-core-concepts--architecture)
2. [Cluster Creation & Configuration](#2-cluster-creation--configuration)
3. [Node Pools & VM SKUs](#3-node-pools--vm-skus)
4. [Networking (CNI, Ingress, DNS, Private Clusters)](#4-networking)
5. [Storage (Persistent Volumes, CSI Drivers)](#5-storage)
6. [Identity & Security (AAD, RBAC, Workload Identity)](#6-identity--security)
7. [Scaling (HPA, VPA, KEDA, Cluster Autoscaler, NAP)](#7-scaling)
8. [Upgrades & Node Image Updates](#8-upgrades--node-image-updates)
9. [Monitoring & Observability (Azure Monitor, Prometheus, Grafana)](#9-monitoring--observability)
10. [DevOps & CI/CD Integration](#10-devops--cicd-integration)
11. [Cost Management & Best Practices](#11-cost-management--best-practices)
12. [AKS Automatic vs Standard](#12-aks-automatic-vs-standard)
13. [Advanced Topics (Workload Identity, KEDA, KAITO, Dapr, Arc)](#13-advanced-topics)
14. [Troubleshooting Scenarios](#14-troubleshooting-scenarios)
15. [Master Cheatsheet](#master-cheatsheet)

---

## 1. AKS Core Concepts & Architecture

### 🟢 Q1. What is Azure Kubernetes Service (AKS) and what does Azure manage for you?

AKS is a **managed Kubernetes service** on Azure that offloads control plane management to Microsoft. You only manage your application workloads and node pools. Azure handles the Kubernetes API server, etcd, scheduler, controller manager, and cloud-controller-manager at no extra cost (free tier) or with an uptime SLA (Standard/Premium tier).

**Two cluster modes (GA 2025):**
- **AKS Standard** — maximum flexibility, you own node pool design, networking, scaling config
- **AKS Automatic** — production-ready defaults, preconfigured node management, security guardrails, auto-upgrades

```bash
# Create a Standard AKS cluster (az CLI)
az group create --name myRG --location eastus

az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --node-count 3 \
  --enable-managed-identity \
  --enable-addons monitoring \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group myRG --name myAKSCluster

# Verify nodes
kubectl get nodes -o wide
```

---

### 🟢 Q2. What are the main components of an AKS cluster?

AKS clusters have two planes:

| Plane | Components | Managed by |
|-------|-----------|-----------|
| **Control Plane** | kube-apiserver, etcd, kube-scheduler, kube-controller-manager, cloud-controller-manager | Azure (free) |
| **Data Plane (Nodes)** | kubelet, kube-proxy, containerd runtime | Customer |

**Two resource groups are created:**
1. Customer RG → holds the AKS resource object
2. **Node Resource Group** (MC_<rg>_<cluster>_<region>) → holds VMs, VMSS, NICs, disks, LBs

```bash
# View node resource group
az aks show \
  --resource-group myRG \
  --name myAKSCluster \
  --query nodeResourceGroup \
  --output tsv
# Output: MC_myRG_myAKSCluster_eastus

# List all resources in node RG
az resource list \
  --resource-group MC_myRG_myAKSCluster_eastus \
  --output table
```

---

### 🟢 Q3. What Kubernetes versions does AKS support and what is the support policy?

AKS follows a **12-month support policy** for GA Kubernetes versions. AKS supports N-2 minor versions (latest + 2 older). Starting 2024, **Long-Term Support (LTS)** is available for select versions (e.g., 1.27 LTS = 2 years).

```bash
# List all supported versions in a region
az aks get-versions --location eastus --output table

# Check current cluster version
az aks show \
  --resource-group myRG \
  --name myAKSCluster \
  --query kubernetesVersion

# Use alias minor version (auto-selects latest patch)
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --kubernetes-version 1.30   # alias — picks latest 1.30.x
```

**Key version facts (2025-2026):**
- Azure Linux 2.0 → EOL Nov 30, 2025; migrate to Azure Linux 3.0
- containerd is default runtime since K8s 1.19 (Linux) and 1.23 (Windows)
- LTS requires explicit enablement via `--tier premium`

---

### 🟡 Q4. Explain the AKS node resource group and why you must not modify it manually.

The **node resource group** (MC_*) is auto-created and fully managed by AKS. It contains VMSS, NICs, public IPs, load balancers, and disks. Manually modifying resources in this RG can cause **cluster breakage** because AKS reconciles the state and may overwrite or delete your changes.

```bash
# CORRECT: Customize node RG name at creation time
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --node-resource-group myCustomNodeRG \
  --node-count 2

# WRONG: Don't manually modify VMs or VMSS in MC_* RG
# Use AKS APIs/node pools instead

# Lock node RG from manual changes (best practice)
az lock create \
  --name "AKSNodeRGLock" \
  --lock-type ReadOnly \
  --resource-group MC_myRG_myAKSCluster_eastus
```

---

### 🟡 Q5. What is the AKS pricing tier model (Free, Standard, Premium)?

| Tier | SLA | Use Case | Cost |
|------|-----|----------|------|
| **Free** | None | Dev/test | Free control plane |
| **Standard** | 99.9% (LZ), 99.95% (AZ) | Production | ~$0.10/cluster/hr |
| **Premium** | 99.95% + LTS | Enterprise, compliance | Higher cost |

```bash
# Create cluster with Standard tier
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --tier standard \
  --node-count 3

# Upgrade existing cluster to Premium (for LTS)
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --tier premium \
  --k8s-support-plan AKSLongTermSupport
```

---

## 2. Cluster Creation & Configuration

### 🟢 Q6. What are the key parameters when creating an AKS cluster?

```bash
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  # Node configuration
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --os-sku AzureLinux \              # Ubuntu | AzureLinux | Windows2022
  # Networking
  --network-plugin azure \           # azure (CNI) | kubenet | none
  --network-policy calico \          # calico | azure | cilium
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
  --service-cidr 10.0.0.0/16 \
  --dns-service-ip 10.0.0.10 \
  # Identity & Security
  --enable-managed-identity \
  --enable-aad \
  --aad-admin-group-object-ids <group-id> \
  --enable-azure-rbac \
  # Add-ons
  --enable-addons monitoring,azure-policy \
  # Availability
  --zones 1 2 3 \                    # Availability Zones
  --tier standard \
  --generate-ssh-keys
```

---

### 🟡 Q7. How does AKS handle SSH access to nodes and what are alternatives?

Direct SSH to nodes is not recommended. AKS provides several secure alternatives:

```bash
# Option 1: kubectl debug (node shell - ephemeral container)
kubectl debug node/aks-nodepool1-12345-vmss000000 \
  -it \
  --image=mcr.microsoft.com/cbl-mariner/busybox:latest

# Option 2: az aks command invoke (run kubectl without direct cluster access)
az aks command invoke \
  --resource-group myRG \
  --name myAKSCluster \
  --command "kubectl get pods -A"

# Option 3: SSH with az aks nodepool update (enable SSH)
# First create a debug pod on target node
kubectl run -it --rm debug \
  --image=mcr.microsoft.com/aks/fundamental/base-ubuntu:v0.0.11 \
  --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"aks-node-1"}}}'

# Option 4: Azure Bastion for direct node access
```

---

### 🟡 Q8. What is the AKS stop/start feature and when would you use it?

AKS allows **stopping a cluster** to pause billing for node VMs while preserving cluster state. The control plane continues to incur charges (Standard tier). Use for non-production clusters outside business hours.

```bash
# Stop cluster (saves ~80% of costs for dev/test)
az aks stop \
  --resource-group myRG \
  --name myAKSCluster

# Check cluster state
az aks show \
  --resource-group myRG \
  --name myAKSCluster \
  --query powerState

# Start cluster (restores all workloads)
az aks start \
  --resource-group myRG \
  --name myAKSCluster

# Automate with Azure Automation or Logic Apps for scheduled stop/start
```

---

### 🔴 Q9. How do you enable and configure a private AKS cluster?

A **private cluster** ensures the Kubernetes API server is only accessible via a private endpoint within your VNet — no public internet exposure.

```bash
# Create private cluster
az aks create \
  --resource-group myRG \
  --name privateAKS \
  --enable-private-cluster \
  --private-dns-zone system \        # system | none | custom BYO zone resource ID
  --disable-public-fqdn \            # removes public FQDN entirely
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet

# Access private cluster (requires being on same VNet or via VPN/ExpressRoute)
az aks get-credentials \
  --resource-group myRG \
  --name privateAKS

# Use az aks command invoke to run kubectl without VPN
az aks command invoke \
  --resource-group myRG \
  --name privateAKS \
  --command "kubectl get nodes"

# Connect via Azure Bastion jump host
# Or configure VNet peering + private DNS zone linking
```

**Private DNS Zone options:**
| Option | Description |
|--------|-------------|
| `system` | AKS creates and manages the private DNS zone |
| `none` | No private DNS zone (you handle DNS) |
| Custom resource ID | BYO private DNS zone |

---

## 3. Node Pools & VM SKUs

### 🟢 Q10. What are system and user node pools in AKS?

AKS has two node pool types:

| Type | Purpose | Taint | Min Nodes |
|------|---------|-------|-----------|
| **System** | Hosts critical system pods (CoreDNS, konnectivity, metrics-server) | `CriticalAddonsOnly=true:NoSchedule` | 1 (prod: 3) |
| **User** | Hosts application workloads | None (default) | 0 (can scale to zero) |

```bash
# Add a user node pool
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name gpupool \
  --node-count 2 \
  --node-vm-size Standard_NC6s_v3 \  # GPU SKU
  --mode User \
  --node-taints sku=gpu:NoSchedule \  # dedicated GPU pool
  --labels hardware=gpu \
  --zones 1 2 3

# List all node pools
az aks nodepool list \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --output table

# Scale a node pool
az aks nodepool scale \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name gpupool \
  --node-count 5
```

---

### 🟡 Q11. How do you use node taints, labels, and affinity to control pod scheduling?

```yaml
# Node pool taint at creation (via CLI)
# az aks nodepool add --node-taints "dedicated=frontend:NoSchedule"

# Pod tolerates the taint
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "frontend"
    effect: "NoSchedule"
  nodeSelector:
    hardware: gpu                  # match node label

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: agentpool
            operator: In
            values:
            - gpupool
    podAntiAffinity:               # spread pods across nodes
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchLabels:
              app: frontend
  containers:
  - name: frontend
    image: myapp:latest
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1"
```

---

### 🟡 Q12. What is Node Auto-Provisioning (NAP) in AKS and how does it differ from Cluster Autoscaler?

**NAP** (powered by Karpenter) is the next-generation node provisioning system for AKS. It automatically creates right-sized node pools on demand based on pod requirements, instead of scaling within a pre-defined VMSS.

| Feature | Cluster Autoscaler (CA) | Node Auto-Provisioning (NAP) |
|---------|------------------------|------------------------------|
| Approach | Scale existing node pools | Create new nodes on demand |
| VM size | Fixed per node pool | Dynamic, best-fit selection |
| Speed | Minutes | Seconds |
| Config | Min/max per pool | NodePool CRD |
| AKS Automatic | Default | Default |

```bash
# Enable NAP on existing cluster
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-node-auto-provisioning

# NAP NodePool CRD
kubectl get nodepools
kubectl describe nodepool default
```

```yaml
# Custom NodePool resource (NAP/Karpenter-style)
apiVersion: karpenter.azure.com/v1alpha2
kind: AKSNodeClass
metadata:
  name: default
spec:
  osSKU: AzureLinux
  imageVersion: latest
---
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: gpu-nodepool
spec:
  template:
    spec:
      nodeClassRef:
        name: default
      requirements:
      - key: karpenter.azure.com/sku-family
        operator: In
        values: ["N"]              # GPU SKU family
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
```

---

### 🔴 Q13. How do you configure Spot node pools in AKS for cost savings?

**Spot VMs** use unused Azure capacity at up to 90% discount but can be evicted with 30-second notice.

```bash
# Add spot node pool
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \        # Delete | Deallocate
  --spot-max-price -1 \             # -1 = current spot price (max = on-demand price)
  --node-vm-size Standard_D4s_v5 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 10 \
  --node-taints "kubernetes.azure.com/scalesetpriority=spot:NoSchedule"
```

```yaml
# Workload that tolerates spot eviction
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-job
spec:
  replicas: 10
  template:
    spec:
      tolerations:
      - key: "kubernetes.azure.com/scalesetpriority"
        operator: "Equal"
        value: "spot"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: "kubernetes.azure.com/scalesetpriority"
                operator: In
                values: ["spot"]
      # Handle graceful shutdown on spot eviction
      terminationGracePeriodSeconds: 30
      containers:
      - name: batch
        image: myapp:latest
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 20 && save-checkpoint.sh"]
```

---

## 4. Networking

### 🟢 Q14. What are the AKS CNI network plugin options and when to use each?

| Plugin | Model | IP Allocation | Use Case |
|--------|-------|--------------|----------|
| **kubenet** | Overlay (NAT) | Node IPs from VNet, pod IPs from separate CIDR | IP-constrained envs |
| **Azure CNI** | Flat | Every pod gets a VNet IP | Direct routing, peering |
| **Azure CNI Overlay** | Overlay (no NAT) | Pods use private overlay IPs | Scale + VNet efficiency |
| **Azure CNI Powered by Cilium** | eBPF | VNet IPs + eBPF dataplane | High-perf, network policy |
| **Bring Your Own CNI** | Custom | Per CNI | Advanced/custom |

```bash
# Azure CNI (recommended for production)
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/.../subnets/aks \
  --service-cidr 172.16.0.0/16 \
  --dns-service-ip 172.16.0.10

# Azure CNI Overlay (AKS 1.26+) - best of both worlds
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16

# Cilium eBPF dataplane
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium
```

---

### 🟡 Q15. How do you configure Ingress in AKS (Application Gateway, NGINX, AGIC)?

AKS supports multiple ingress approaches:

```bash
# Option 1: Application Gateway Ingress Controller (AGIC) - recommended for Azure-native
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-addons ingress-appgw \
  --appgw-name myAppGW \
  --appgw-subnet-cidr 10.2.0.0/16

# Option 2: NGINX via Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

```yaml
# NGINX Ingress with TLS + cert-manager
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
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
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: myapi-svc
            port:
              number: 8080
---
# Application Gateway Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: appgw-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/backend-path-prefix: "/"
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
              number: 80
```

---

### 🟡 Q16. How do Network Policies work in AKS and what providers are supported?

```yaml
# Block all ingress by default, allow only from same namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}                    # applies to all pods
  policyTypes:
  - Ingress
---
# Allow frontend → backend traffic only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
---
# Cilium network policy (extended - FQDN egress)
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-external-api
spec:
  endpointSelector:
    matchLabels:
      app: myapp
  egress:
  - toFQDNs:
    - matchName: "api.example.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

```bash
# Enable network policy at cluster creation
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --network-plugin azure \
  --network-policy calico     # calico | azure | cilium

# Verify network policy provider
kubectl describe daemonset -n kube-system | grep -i calico
```

---

### 🔴 Q17. Explain AKS DNS architecture (CoreDNS, custom DNS, Private DNS Zones).

```yaml
# View CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Customize CoreDNS (add custom upstream DNS)
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  custom.server: |
    corp.example.com:53 {                # forward corp DNS to on-prem resolver
      forward . 10.0.0.5 10.0.0.6
    }
  log.override: |
    log                                  # enable DNS query logging
```

```bash
# Private AKS with custom DNS server
az aks create \
  --resource-group myRG \
  --name privateAKS \
  --enable-private-cluster \
  --private-dns-zone /subscriptions/.../privateDnsZones/privatelink.eastus.azmk8s.io \
  --dns-name-prefix myaks

# Link private DNS zone to additional VNets
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "privatelink.eastus.azmk8s.io" \
  --name appvnet-link \
  --virtual-network /subscriptions/.../vnets/app-vnet \
  --registration-enabled false
```

---

## 5. Storage

### 🟢 Q18. What storage options are available in AKS?

| Storage Type | Use Case | Access Modes | CSI Driver |
|-------------|----------|-------------|-----------|
| Azure Disk | Single-pod RW (databases) | RWO | disk.csi.azure.com |
| Azure Files | Shared storage, multi-pod | RWO, RWX, ROX | file.csi.azure.com |
| Azure Blob | Object storage, big data | RWO, RWX | blob.csi.azure.com |
| Azure NetApp Files | High-perf NFS | RWO, RWX | Custom ANF CSI |
| NFS Server | On-prem NFS | RWO, RWX | nfs.csi.azure.com |

```bash
# List default storage classes
kubectl get storageclass

# Output (typical AKS defaults):
# NAME                    PROVISIONER               RECLAIMPOLICY
# azurefile               file.csi.azure.com        Delete
# azurefile-csi           file.csi.azure.com        Delete
# azurefile-csi-premium   file.csi.azure.com        Delete
# azuredisk-csi           disk.csi.azure.com        Delete  
# managed                 disk.csi.azure.com        Delete
# managed-csi (default)   disk.csi.azure.com        Delete
# managed-csi-premium     disk.csi.azure.com        Delete
```

---

### 🟡 Q19. How do you create and use Persistent Volumes with Azure Disk and Azure Files?

```yaml
# Azure Disk - dynamic provisioning via StorageClass
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes:
  - ReadWriteOnce                    # Azure Disk = RWO only
  storageClassName: managed-csi-premium
  resources:
    requests:
      storage: 100Gi
---
# Use PVC in Pod
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 1
  serviceName: postgres
  selector:
    matchLabels:
      app: postgres
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: managed-csi-premium
      resources:
        requests:
          storage: 100Gi
---
# Azure Files - shared storage (RWX) - for multi-pod access
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
  - ReadWriteMany                    # Azure Files supports RWX
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 50Gi
---
# Custom StorageClass with parameters
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azuredisk-retain
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS              # Standard_LRS | StandardSSD_LRS | Premium_LRS | UltraSSD_LRS
  kind: Managed
  cachingMode: ReadOnly             # None | ReadOnly | ReadWrite
reclaimPolicy: Retain               # Retain data when PVC deleted
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

---

### 🔴 Q20. How do you handle CSI driver secrets for Azure storage authentication (Workload Identity)?

```yaml
# SecretProviderClass for Azure Key Vault CSI driver
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname-wi
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "<workload-identity-client-id>"
    keyvaultName: myKeyVault
    cloudName: ""
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: tls-cert
          objectType: cert
          objectVersion: ""
    tenantId: "<tenant-id>"
  secretObjects:                     # sync to K8s secret
  - secretName: db-credentials
    type: Opaque
    data:
    - objectName: db-password
      key: password
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    azure.workload.identity/use: "true"   # required for Workload Identity
spec:
  serviceAccountName: myapp-sa
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: azure-kvname-wi
```

---

## 6. Identity & Security

### 🟢 Q21. What is the difference between Kubernetes RBAC and Azure RBAC in AKS?

| Feature | Kubernetes RBAC | Azure RBAC for K8s |
|---------|----------------|-------------------|
| Auth source | K8s ClusterRole/RoleBinding | Azure IAM roles |
| Identity | K8s users/groups/SA | AAD users/groups/MSI |
| Scope | Namespace / Cluster | Azure resource scope |
| Management | kubectl / YAML | Azure portal / CLI |
| Audit | K8s audit logs | Azure Activity Log |

```bash
# Enable both Azure RBAC and AAD on cluster
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-aad \
  --enable-azure-rbac \
  --aad-admin-group-object-ids <aad-group-id>

# Assign Azure RBAC role (Azure Kubernetes Service RBAC Admin)
az role assignment create \
  --assignee <user-or-group-object-id> \
  --role "Azure Kubernetes Service RBAC Admin" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.ContainerService/managedClusters/myAKSCluster/namespaces/production

# Built-in AKS Azure RBAC roles:
# - Azure Kubernetes Service RBAC Cluster Admin
# - Azure Kubernetes Service RBAC Admin
# - Azure Kubernetes Service RBAC Writer
# - Azure Kubernetes Service RBAC Reader
```

```yaml
# Kubernetes RBAC: create role + binding
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: production
subjects:
- kind: Group
  name: <aad-group-object-id>       # AAD group maps here
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

### 🟡 Q22. What is Workload Identity in AKS and how does it replace Pod Identity (AAD Pod Identity)?

**AKS Workload Identity** (GA 2023) uses Kubernetes Service Account Token Volume Projection + Azure AD federated credentials to grant pods access to Azure resources — without any secrets or node-level MSIs.

```bash
# Enable Workload Identity on cluster
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-workload-identity \
  --enable-oidc-issuer

# Get OIDC issuer URL
AKS_OIDC_ISSUER=$(az aks show \
  --resource-group myRG \
  --name myAKSCluster \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

# Create managed identity
az identity create \
  --name myWorkloadIdentity \
  --resource-group myRG

CLIENT_ID=$(az identity show \
  --name myWorkloadIdentity \
  --resource-group myRG \
  --query clientId -o tsv)

# Create federated credential (link K8s SA → Azure MSI)
az identity federated-credential create \
  --name myFederatedCred \
  --identity-name myWorkloadIdentity \
  --resource-group myRG \
  --issuer "${AKS_OIDC_ISSUER}" \
  --subject "system:serviceaccount:default:myapp-sa" \
  --audience api://AzureADTokenExchange

# Assign Azure role to identity
az role assignment create \
  --assignee "${CLIENT_ID}" \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/.../vaults/myKeyVault
```

```yaml
# Kubernetes service account with Workload Identity annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
  annotations:
    azure.workload.identity/client-id: "<client-id>"    # MSI client ID
---
# Pod using Workload Identity
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    azure.workload.identity/use: "true"                  # required label
spec:
  serviceAccountName: myapp-sa
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: AZURE_CLIENT_ID
      value: "<client-id>"
    - name: AZURE_TENANT_ID
      value: "<tenant-id>"
    # AZURE_FEDERATED_TOKEN_FILE is auto-injected by the webhook
```

---

### 🟡 Q23. How do you implement Pod Security with AKS (Pod Security Admission, Azure Policy)?

```yaml
# Pod Security Admission (PSA) - K8s 1.25+, replaces PSP
# Apply namespace-level security standards
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted    # enforce restricted profile
    pod-security.kubernetes.io/warn: restricted       # warn on violations
    pod-security.kubernetes.io/audit: restricted      # audit violations
---
# Restricted-compliant pod spec
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: production
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp             # writable temp dir
  volumes:
  - name: tmp
    emptyDir: {}
```

```bash
# Azure Policy add-on for AKS (Gatekeeper/OPA)
az aks enable-addons \
  --resource-group myRG \
  --name myAKSCluster \
  --addons azure-policy

# View policy compliance
kubectl get constrainttemplate
kubectl get k8sazurecontainerallowedimages

# Assign built-in Azure Policy initiative
az policy assignment create \
  --name "AKS-Baseline-Security" \
  --scope /subscriptions/<sub>/resourceGroups/myRG \
  --policy-set-definition "a8eff44f-8c92-45c3-a3fb-9880802d67a7"  # Kubernetes cluster pod security baseline
```

---

### 🔴 Q24. How do you secure the AKS API server with authorized IP ranges and private endpoints?

```bash
# Restrict API server access to specific IP ranges
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --api-server-authorized-ip-ranges "10.0.0.0/8,203.0.113.5/32"

# Update authorized IP ranges on existing cluster
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --api-server-authorized-ip-ranges "10.0.0.0/8,your-new-ip/32"

# Remove all restrictions (not recommended for prod)
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --api-server-authorized-ip-ranges ""

# Fully private cluster (API via private endpoint only)
az aks create \
  --resource-group myRG \
  --name privateAKS \
  --enable-private-cluster \
  --disable-public-fqdn

# Defender for Containers (runtime threat detection)
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-defender
```

---

## 7. Scaling

### 🟢 Q25. How does the Cluster Autoscaler work in AKS?

The **Cluster Autoscaler (CA)** watches for pending pods and scales node pools up/down within configured min/max boundaries.

```bash
# Enable CA on a node pool
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name apppool \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 10 \
  --node-vm-size Standard_D4s_v5

# Enable CA on existing node pool
az aks nodepool update \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name apppool \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 20

# View CA activity
kubectl get events -n kube-system | grep cluster-autoscaler

# Check CA status
kubectl get cm cluster-autoscaler-status -n kube-system -o yaml
```

```yaml
# Cluster Autoscaler profile (fine-tuning)
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --cluster-autoscaler-profile \
    scan-interval=10s \
    scale-down-delay-after-add=10m \
    scale-down-unneeded-time=10m \
    scale-down-unready-time=20m \
    max-graceful-termination-sec=600 \
    balance-similar-node-groups=true \
    skip-nodes-with-system-pods=true
```

---

### 🟡 Q26. How do HPA and VPA work in AKS, and what are their limitations?

```yaml
# HPA - scale deployment based on CPU/memory
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
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 500Mi
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # fast scale-up
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # slow scale-down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
---
# VPA - right-size pod resource requests automatically
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
    updateMode: "Auto"               # Off | Initial | Recreate | Auto
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
```

**HPA + CA interaction:** HPA scales pods → if nodes full, CA adds nodes. HPA `minReplicas: 0` requires KEDA.

---

### 🟡 Q27. How does KEDA work with AKS for event-driven scaling?

```bash
# Install KEDA via AKS add-on (managed)
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-keda
```

```yaml
# KEDA ScaledObject - scale on Azure Service Bus queue depth
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: servicebus-scaler
spec:
  scaleTargetRef:
    name: order-processor
  pollingInterval: 15               # check every 15 seconds
  cooldownPeriod: 300               # wait 5min before scaling down
  minReplicaCount: 0                # scale to zero!
  maxReplicaCount: 50
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      namespace: myservicebus
      messageCount: "5"             # scale out when >5 msgs/replica
    authenticationRef:
      name: keda-servicebus-auth
---
# TriggerAuthentication using Workload Identity
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-servicebus-auth
spec:
  podIdentity:
    provider: azure-workload
---
# KEDA with Azure Storage Queue
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-scaler
spec:
  scaleTargetRef:
    name: queue-worker
  minReplicaCount: 0
  maxReplicaCount: 100
  triggers:
  - type: azure-queue
    metadata:
      queueName: myqueue
      queueLength: "10"
      accountName: mystorageaccount
    authenticationRef:
      name: keda-storage-auth
```

---

## 8. Upgrades & Node Image Updates

### 🟢 Q28. How do you upgrade an AKS cluster and what is the recommended approach?

AKS supports **in-place** cluster upgrades. Node surges are used to minimize disruption.

```bash
# Check available upgrade versions
az aks get-upgrades \
  --resource-group myRG \
  --name myAKSCluster \
  --output table

# Upgrade control plane only first
az aks upgrade \
  --resource-group myRG \
  --name myAKSCluster \
  --kubernetes-version 1.30.5 \
  --control-plane-only

# Then upgrade each node pool
az aks nodepool upgrade \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name nodepool1 \
  --kubernetes-version 1.30.5 \
  --max-surge 33%         # number of extra nodes during upgrade

# Full upgrade (control plane + all node pools)
az aks upgrade \
  --resource-group myRG \
  --name myAKSCluster \
  --kubernetes-version 1.30.5
```

---

### 🟡 Q29. What are Auto-Upgrade Channels in AKS and how do you configure them?

| Channel | Behavior |
|---------|----------|
| `none` | No automatic upgrades |
| `patch` | Auto-patch within current minor version |
| `stable` | Auto-upgrade to N-1 stable minor version |
| `rapid` | Auto-upgrade to latest GA minor version |
| `node-image` | Only node OS images (not K8s version) |

```bash
# Set auto-upgrade channel
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --auto-upgrade-channel patch

# Combine with maintenance window to control when upgrades happen
az aks maintenanceconfiguration add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name default \
  --weekday Sunday \
  --start-hour 2                    # 2 AM Sunday window

# Node OS auto-upgrade channel
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --node-os-upgrade-channel NodeImage   # NodeImage | SecurityPatch | None | Unmanaged

# Check upgrade history
az aks show \
  --resource-group myRG \
  --name myAKSCluster \
  --query "upgradeSettings"
```

---

### 🔴 Q30. How do you implement blue-green cluster upgrades with no downtime?

```bash
# Blue-Green cluster upgrade strategy:
# 1. Create GREEN cluster with new version
az aks create \
  --resource-group myRG \
  --name myAKSCluster-green \
  --kubernetes-version 1.30.5 \
  --node-count 5

# 2. Deploy application to GREEN cluster
az aks get-credentials --name myAKSCluster-green -g myRG
kubectl apply -f manifests/

# 3. Run smoke tests on GREEN
./run-smoke-tests.sh green-endpoint

# 4. Switch traffic (update Azure Traffic Manager or Front Door)
az network traffic-manager endpoint update \
  --resource-group myRG \
  --profile-name myTM \
  --name aks-endpoint \
  --type AzureEndpoints \
  --target-resource-id /subscriptions/.../loadBalancers/green-lb

# 5. Monitor → delete BLUE after validation period
az aks delete --name myAKSCluster-blue --resource-group myRG
```

---

## 9. Monitoring & Observability

### 🟢 Q31. How do you enable Azure Monitor / Container Insights for AKS?

```bash
# Enable during cluster creation
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-addons monitoring \
  --workspace-resource-id /subscriptions/.../workspaces/myLogAnalytics

# Enable on existing cluster
az aks enable-addons \
  --resource-group myRG \
  --name myAKSCluster \
  --addons monitoring \
  --workspace-resource-id /subscriptions/.../workspaces/myLA

# Verify agent is running
kubectl get pods -n kube-system | grep ama-logs

# View metrics in Azure portal → AKS → Insights
# Or query via Log Analytics
```

```kusto
// KQL: Top CPU-consuming pods in last hour
KubePodInventory
| where TimeGenerated > ago(1h)
| where ClusterName == "myAKSCluster"
| join kind=inner (
    Perf
    | where ObjectName == "K8SContainer"
    | where CounterName == "cpuUsageNanoCores"
) on InstanceName
| summarize avg(CounterValue) by PodName, Namespace
| top 20 by avg_CounterValue desc
```

---

### 🟡 Q32. How do you set up managed Prometheus and Grafana for AKS?

```bash
# Enable Azure Monitor managed Prometheus
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id /subscriptions/.../azureMonitorWorkspaces/myAMW

# Create managed Grafana and link to AKS
az grafana create \
  --name myGrafana \
  --resource-group myRG

GRAFANA_ID=$(az grafana show \
  --name myGrafana \
  --resource-group myRG \
  --query id -o tsv)

az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-azure-monitor-metrics \
  --grafana-resource-id "${GRAFANA_ID}"

# Deploy custom PrometheusRule
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: aks-custom-rules
  namespace: monitoring
spec:
  groups:
  - name: aks.rules
    rules:
    - alert: HighPodMemory
      expr: container_memory_usage_bytes{container!=""} > 900000000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ \$labels.pod }} memory > 900MB"
EOF
```

---

### 🔴 Q33. How do you implement distributed tracing with OpenTelemetry on AKS?

```yaml
# Deploy OTel Collector as DaemonSet on AKS
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  mode: daemonset
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      prometheus:
        config:
          scrape_configs:
          - job_name: 'kubernetes-pods'
            kubernetes_sd_configs:
            - role: pod
    processors:
      batch:
        timeout: 1s
        send_batch_size: 1024
      memory_limiter:
        limit_mib: 400
    exporters:
      azuremonitor:
        connection_string: "${APPLICATIONINSIGHTS_CONNECTION_STRING}"
      otlp/tempo:
        endpoint: "tempo:4317"
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [azuremonitor, otlp/tempo]
        metrics:
          receivers: [otlp, prometheus]
          processors: [batch]
          exporters: [azuremonitor]
```

---

## 10. DevOps & CI/CD Integration

### 🟢 Q34. How do you deploy to AKS from Azure DevOps Pipelines?

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
    - main

variables:
  containerRegistry: myacr.azurecr.io
  imageRepository: myapp
  tag: $(Build.BuildId)
  kubernetesCluster: myAKSCluster
  resourceGroup: myRG

stages:
- stage: Build
  jobs:
  - job: BuildAndPush
    pool:
      vmImage: ubuntu-latest
    steps:
    - task: Docker@2
      displayName: Build and push image
      inputs:
        command: buildAndPush
        repository: $(containerRegistry)/$(imageRepository)
        dockerfile: Dockerfile
        containerRegistry: myACRServiceConnection
        tags: |
          $(tag)
          latest

- stage: Deploy
  dependsOn: Build
  jobs:
  - deployment: DeployToAKS
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@1
            displayName: Deploy to AKS
            inputs:
              action: deploy
              connectionType: azureResourceManager
              azureSubscriptionConnection: myAzureServiceConnection
              azureResourceGroup: $(resourceGroup)
              kubernetesCluster: $(kubernetesCluster)
              namespace: production
              manifests: |
                k8s/deployment.yaml
                k8s/service.yaml
              containers: |
                $(containerRegistry)/$(imageRepository):$(tag)
```

---

### 🟡 Q35. How do you integrate ACR (Azure Container Registry) with AKS?

```bash
# Attach ACR to AKS (grants AcrPull role to kubelet managed identity)
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --attach-acr myACR

# Verify the role assignment
az role assignment list \
  --scope /subscriptions/.../registries/myACR \
  --query "[?roleDefinitionName=='AcrPull']"

# Build image in ACR (no local Docker needed)
az acr build \
  --registry myACR \
  --image myapp:v1.0 \
  --file Dockerfile .

# ACR Tasks - auto-rebuild on base image update
az acr task create \
  --registry myACR \
  --name buildOnBaseUpdate \
  --image myapp:{{.Run.ID}} \
  --arg REGISTRY=myacr.azurecr.io \
  --file acr-task.yaml \
  --context https://github.com/org/repo#main
```

```yaml
# Pod using ACR image (no imagePullSecrets needed with attach-acr)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myacr.azurecr.io/myapp:v1.0    # direct ACR reference
        imagePullPolicy: Always
```

---

### 🔴 Q36. How do you implement GitOps on AKS using Flux v2?

```bash
# Enable Flux extension on AKS
az k8s-configuration flux create \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --cluster-type managedClusters \
  --name cluster-config \
  --namespace flux-system \
  --scope cluster \
  --url https://github.com/org/fleet-infra \
  --branch main \
  --kustomization name=infra path=./clusters/production prune=true
```

```yaml
# Flux Kustomization for AKS workloads
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  path: ./apps/production
  prune: true                        # delete resources removed from Git
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: myapp
    namespace: production
  timeout: 5m
  postBuild:
    substitute:
      cluster_name: myAKSCluster
      cluster_env: production
---
# Flux HelmRelease for managed Helm deployments
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: myapp
  namespace: production
spec:
  interval: 30m
  chart:
    spec:
      chart: myapp
      version: ">=1.0.0 <2.0.0"
      sourceRef:
        kind: HelmRepository
        name: myapp-charts
      interval: 12h
  values:
    replicaCount: 3
    image:
      repository: myacr.azurecr.io/myapp
      tag: stable
  upgrade:
    remediation:
      remediateLastFailure: true
```

---

## 11. Cost Management & Best Practices

### 🟡 Q37. What are the key cost optimization strategies for AKS?

```bash
# 1. Right-size nodes — use VPA recommendations
kubectl describe vpa myapp-vpa | grep -A10 "Recommendation"

# 2. Use Spot node pools for non-critical workloads
az aks nodepool add --priority Spot --spot-max-price -1 ...

# 3. Scale to zero with KEDA + user node pools (minCount: 0)
az aks nodepool update --min-count 0 ...

# 4. Use cluster stop for dev/test environments
az aks stop --name devAKS --resource-group devRG

# 5. Use Azure Reservations for predictable baseline nodes
# Purchase 1yr/3yr reservations for system node pool VMs via portal

# 6. Enable cost analysis add-on
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-cost-analysis

# View per-namespace costs in Azure Cost Management
```

```yaml
# LimitRange - prevent resource waste
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: production
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "4"
      memory: "8Gi"
    type: Container
---
# ResourceQuota - cap namespace spending
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services.loadbalancers: "5"
    persistentvolumeclaims: "20"
```

---

## 12. AKS Automatic vs Standard

### 🟡 Q38. What are the key differences between AKS Automatic and AKS Standard?

| Feature | AKS Automatic | AKS Standard |
|---------|--------------|-------------|
| Node provisioning | NAP (Karpenter) | Manual pools or CA |
| OS upgrades | Auto (node-image channel) | Manual or configured |
| Security defaults | Pre-hardened (restricted PSA) | You configure |
| RBAC | Azure RBAC only (no kubeconfig with static creds) | Both Azure RBAC & K8s RBAC |
| Uptime SLA | Included by default | Standard/Premium tier |
| Network policy | Cilium (default) | Choose your own |
| Node pool management | Fully managed | Full control |
| Best for | App teams, simplified ops | Platform teams, customization |

```bash
# Create AKS Automatic cluster
az aks create \
  --resource-group myRG \
  --name autoAKS \
  --mode Automatic               # the key flag

# AKS Automatic uses Azure RBAC exclusively
# Cannot download static kubeconfig — use az aks get-credentials with --overwrite-existing
az aks get-credentials \
  --resource-group myRG \
  --name autoAKS

# Configure Helm with AKS Automatic (Azure RBAC — no static token)
# Option 1: Use az aks command invoke
az aks command invoke \
  --resource-group myRG \
  --name autoAKS \
  --command "helm upgrade --install myapp ./chart -n default"

# Option 2: Use kubelogin with service principal
kubelogin convert-kubeconfig -l spn
```

---

## 13. Advanced Topics

### 🔴 Q39. How do you deploy AI/ML workloads on AKS using KAITO?

**KAITO** (Kubernetes AI Toolchain Operator) simplifies deploying large AI/ML models on AKS with GPU node auto-provisioning.

```bash
# Install KAITO via AKS add-on
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-kaito

# Verify KAITO controller
kubectl get pods -n kaito-system
```

```yaml
# Deploy LLaMA model with KAITO Workspace CRD
apiVersion: kaito.sh/v1alpha1
kind: Workspace
metadata:
  name: llama-inference
spec:
  resource:
    instanceType: Standard_NC6s_v3  # GPU VM
    labelSelector:
      matchLabels:
        apps: llama
  inference:
    preset:
      name: llama-2-7b-chat        # KAITO preset models
---
# KAITO auto-provisions GPU node pool, downloads model, deploys endpoint
# Access inference endpoint
kubectl get service llama-inference -n default
# curl http://<service-ip>:80/v1/completions -d '{"prompt":"Hello"}'
```

---

### 🔴 Q40. How do you configure Dapr on AKS for microservices communication?

```bash
# Enable Dapr extension on AKS
az k8s-extension create \
  --cluster-type managedClusters \
  --cluster-name myAKSCluster \
  --resource-group myRG \
  --name dapr \
  --extension-type Microsoft.Dapr \
  --auto-upgrade-minor-version true \
  --configuration-settings \
    "global.ha.enabled=true"
```

```yaml
# Dapr-enabled deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "8080"
        dapr.io/log-level: "info"
        dapr.io/config: "appconfig"    # Dapr configuration
    spec:
      containers:
      - name: order-service
        image: myacr.azurecr.io/order-service:v1
---
# Dapr Component - Azure Service Bus pub/sub
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: orders-pubsub
spec:
  type: pubsub.azure.servicebus
  version: v1
  metadata:
  - name: connectionString
    secretKeyRef:
      name: servicebus-secret
      key: connectionString
---
# Dapr Component - Azure Blob State Store
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.azure.blobstorage
  version: v1
  metadata:
  - name: accountName
    value: myStorageAccount
  - name: accountKey
    secretKeyRef:
      name: storage-secret
      key: accountKey
  - name: containerName
    value: dapr-state
```

---

### 🔴 Q41. How do you connect AKS to on-premises or multi-cloud with Azure Arc?

```bash
# Connect non-Azure K8s cluster to Azure Arc
az connectedk8s connect \
  --name myOnPremCluster \
  --resource-group myRG \
  --location eastus

# Deploy configuration to Arc-connected cluster via GitOps
az k8s-configuration flux create \
  --resource-group myRG \
  --cluster-name myOnPremCluster \
  --cluster-type connectedClusters \  # Arc cluster type
  --name cluster-config \
  --url https://github.com/org/fleet-infra \
  --branch main \
  --kustomization name=apps path=./apps prune=true

# Azure Arc extensions (same as AKS extensions)
az k8s-extension create \
  --cluster-name myOnPremCluster \
  --cluster-type connectedClusters \
  --resource-group myRG \
  --name dapr \
  --extension-type Microsoft.Dapr

# Unified policy management across AKS + Arc clusters
az policy assignment create \
  --policy "Azure Arc Kubernetes cluster should have Kubernetes Configuration extension installed" \
  --scope /subscriptions/<sub>
```

---

## 14. Troubleshooting Scenarios

### 🟡 Q42. How do you troubleshoot a pod stuck in Pending state on AKS?

```bash
# Step 1: Describe the pod for events
kubectl describe pod <pod-name> -n <namespace>
# Look for: Insufficient cpu/memory, No nodes available, Taints not tolerated

# Step 2: Check node capacity
kubectl describe nodes | grep -A5 "Allocated resources"

# Step 3: Check if CA is working
kubectl get events -n kube-system | grep -i scale

# Step 4: Check node pool min/max
az aks nodepool show \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name nodepool1 \
  --query "enableAutoScaling"

# Step 5: Check for resource quota blocking
kubectl describe resourcequota -n <namespace>

# Step 6: Check taints on nodes
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Step 7: Verify PVC bound (if pod has volumes)
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name>

# Common fixes:
# - Increase max-count on node pool
# - Add tolerations to pod spec
# - Reduce resource requests
# - Delete stuck finalizers on PVC
```

---

### 🟡 Q43. How do you diagnose and fix OOMKilled pods in AKS?

```bash
# Identify OOMKilled pods
kubectl get pods -A | grep OOMKilled

# Get exit code (137 = OOMKilled)
kubectl describe pod <pod> -n <ns> | grep -A3 "Last State"

# Check memory usage trend
kubectl top pod <pod> -n <ns> --containers

# View container memory limits
kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.containers[*].resources}'

# AKS Container Insights query for OOM
```

```kusto
// KQL: OOMKilled events in last 24h
KubeEvents
| where TimeGenerated > ago(24h)
| where Reason == "OOMKilling"
| project TimeGenerated, Name, Namespace, Message
| order by TimeGenerated desc
```

```yaml
# Fix: increase memory limit or use VPA
spec:
  containers:
  - name: app
    resources:
      requests:
        memory: "512Mi"
      limits:
        memory: "2Gi"           # increase limit
```

---

### 🔴 Q44. How do you troubleshoot AKS networking issues (pod-to-pod, service DNS, egress)?

```bash
# Test pod-to-pod connectivity
kubectl run test-pod --image=mcr.microsoft.com/cbl-mariner/busybox:latest --rm -it -- sh
# Inside pod:
nslookup kubernetes.default.svc.cluster.local    # test DNS
wget -qO- http://myapp-svc.production.svc.cluster.local:8080/health

# Test service DNS resolution
kubectl run dnstest --image=busybox --rm -it -- nslookup myapp-svc.production

# Check CoreDNS health
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Trace network policy blocking traffic
kubectl run nettest --image=nicolaka/netshoot --rm -it -- bash
# tcpdump -i eth0 port 8080

# Check Azure CNI IP allocation
az network nic list \
  --resource-group MC_myRG_myAKSCluster_eastus \
  --query "[].ipConfigurations[].privateIPAddress" -o tsv

# Verify no IP exhaustion in subnet
az network vnet subnet show \
  --resource-group myRG \
  --vnet-name myVNet \
  --name aks-subnet \
  --query "ipConfigurations | length(@)"

# Test egress (outbound internet)
kubectl run egress-test --image=curlimages/curl --rm -it -- \
  curl -v https://api.example.com

# Check UDR / firewall blocking egress
az network route-table show --resource-group myRG --name myRouteTable
```

---

## Master Cheatsheet

### AKS Cluster Commands
```bash
# Create / Get Credentials
az aks create -g myRG -n myCluster --node-count 3 --enable-managed-identity --generate-ssh-keys
az aks get-credentials -g myRG -n myCluster [--admin]

# Status / Info
az aks show -g myRG -n myCluster
az aks list -g myRG -o table
az aks get-upgrades -g myRG -n myCluster -o table

# Lifecycle
az aks stop/start -g myRG -n myCluster
az aks delete -g myRG -n myCluster --no-wait

# Update features
az aks update -g myRG -n myCluster \
  --enable-workload-identity \
  --enable-oidc-issuer \
  --auto-upgrade-channel patch \
  --enable-keda \
  --enable-azure-monitor-metrics
```

### Node Pool Commands
```bash
az aks nodepool add    -g myRG --cluster-name myCluster -n pool2 --node-count 3 --node-vm-size Standard_D4s_v5
az aks nodepool scale  -g myRG --cluster-name myCluster -n pool2 --node-count 5
az aks nodepool upgrade -g myRG --cluster-name myCluster -n pool2 --kubernetes-version 1.30.5
az aks nodepool delete -g myRG --cluster-name myCluster -n pool2
az aks nodepool list   -g myRG --cluster-name myCluster -o table

# CA control
az aks nodepool update -g myRG --cluster-name myCluster -n pool2 \
  --enable-cluster-autoscaler --min-count 2 --max-count 20
```

### Kubectl Quick Reference
```bash
# Debug
kubectl describe pod <pod>           # events + spec
kubectl logs <pod> -c <container> --previous    # crashed container logs
kubectl debug node/<node> -it --image=busybox   # node shell
kubectl exec -it <pod> -- /bin/sh   # shell into pod

# Resource info
kubectl top nodes / pods
kubectl get events --sort-by='.lastTimestamp' -n <ns>
kubectl get pods -o wide -A         # all pods with node info

# Context switching
kubectl config get-contexts
kubectl config use-context myAKSCluster
kubelogin convert-kubeconfig -l azurecli  # for AAD auth
```

### Identity Quick Reference
```bash
# Workload Identity setup
AKS_OIDC=$(az aks show -g myRG -n myCluster --query oidcIssuerProfile.issuerUrl -o tsv)
CLIENT_ID=$(az identity show -g myRG -n myMSI --query clientId -o tsv)

az identity federated-credential create \
  --name myCred --identity-name myMSI -g myRG \
  --issuer "${AKS_OIDC}" \
  --subject "system:serviceaccount:default:myapp-sa" \
  --audience api://AzureADTokenExchange
```

### Networking Quick Reference
```bash
# Network plugin options
--network-plugin azure                           # Azure CNI (VNet IPs)
--network-plugin azure --network-plugin-mode overlay  # Azure CNI Overlay
--network-plugin azure --network-dataplane cilium     # Cilium eBPF

# Ingress
helm install nginx-ingress ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
az aks enable-addons --addons ingress-appgw --appgw-name myGW ...

# Private cluster
az aks create --enable-private-cluster --private-dns-zone system --disable-public-fqdn
az aks command invoke -g myRG -n myCluster --command "kubectl get nodes"
```

### Storage Quick Reference
```bash
kubectl get storageclass                       # list storage classes
kubectl get pvc -A                             # list all PVCs
kubectl describe pvc <name>                    # debug binding issues

# Storage classes
managed-csi          → Azure Disk Standard SSD (default)
managed-csi-premium  → Azure Disk Premium SSD
azurefile-csi        → Azure Files Standard
azurefile-csi-premium→ Azure Files Premium
```

### Autoscaling Quick Reference
```bash
# HPA
kubectl autoscale deployment myapp --cpu-percent=70 --min=2 --max=20
kubectl get hpa

# CA Profile
az aks update -g myRG -n myCluster \
  --cluster-autoscaler-profile scan-interval=10s scale-down-unneeded-time=10m

# KEDA
az aks update -g myRG -n myCluster --enable-keda
kubectl get scaledobject
```

### Monitoring Quick Reference
```bash
# Enable
az aks enable-addons --addons monitoring --workspace-resource-id <la-id>
az aks update --enable-azure-monitor-metrics --azure-monitor-workspace-resource-id <amw-id>

# Verify
kubectl get pods -n kube-system | grep ama
kubectl get pods -n monitoring | grep prometheus

# Useful KQL
ContainerLog | where LogEntry contains "ERROR" | top 50 by TimeGenerated
KubePodInventory | where PodStatus == "Failed" | summarize count() by Namespace
```

---

## Question Coverage Index

| # | Question | Level | Topic |
|---|----------|-------|-------|
| Q1 | What is AKS and what does Azure manage? | 🟢 | Architecture |
| Q2 | Main components of AKS cluster | 🟢 | Architecture |
| Q3 | Kubernetes version support policy | 🟢 | Versions |
| Q4 | Node resource group and why not modify | 🟡 | Architecture |
| Q5 | AKS pricing tier model | 🟡 | Cost |
| Q6 | Key cluster creation parameters | 🟢 | Cluster Config |
| Q7 | SSH access to nodes | 🟡 | Operations |
| Q8 | Cluster stop/start feature | 🟡 | Cost/Operations |
| Q9 | Private AKS cluster | 🔴 | Networking/Security |
| Q10 | System vs user node pools | 🟢 | Node Pools |
| Q11 | Taints, labels, affinity | 🟡 | Scheduling |
| Q12 | Node Auto-Provisioning (NAP) vs CA | 🟡 | Scaling |
| Q13 | Spot node pools | 🔴 | Cost/Scaling |
| Q14 | CNI network plugin options | 🟢 | Networking |
| Q15 | Ingress (AGIC, NGINX) | 🟡 | Networking |
| Q16 | Network Policies | 🟡 | Networking/Security |
| Q17 | AKS DNS architecture (CoreDNS) | 🔴 | Networking |
| Q18 | Storage options in AKS | 🟢 | Storage |
| Q19 | Azure Disk and Azure Files PVCs | 🟡 | Storage |
| Q20 | CSI driver secrets + Workload Identity | 🔴 | Storage/Security |
| Q21 | Kubernetes RBAC vs Azure RBAC | 🟢 | Security |
| Q22 | Workload Identity (replaces AAD Pod Identity) | 🟡 | Security |
| Q23 | Pod Security Admission + Azure Policy | 🟡 | Security |
| Q24 | API server authorized IP + private endpoint | 🔴 | Security |
| Q25 | Cluster Autoscaler | 🟢 | Scaling |
| Q26 | HPA and VPA | 🟡 | Scaling |
| Q27 | KEDA event-driven scaling | 🟡 | Scaling |
| Q28 | Cluster upgrade process | 🟢 | Upgrades |
| Q29 | Auto-upgrade channels + maintenance windows | 🟡 | Upgrades |
| Q30 | Blue-green cluster upgrades | 🔴 | Upgrades |
| Q31 | Container Insights / Azure Monitor | 🟢 | Monitoring |
| Q32 | Managed Prometheus + Grafana | 🟡 | Monitoring |
| Q33 | OpenTelemetry distributed tracing | 🔴 | Monitoring |
| Q34 | Azure DevOps CI/CD pipeline to AKS | 🟢 | DevOps |
| Q35 | ACR integration with AKS | 🟡 | DevOps |
| Q36 | GitOps with Flux v2 on AKS | 🔴 | DevOps |
| Q37 | Cost optimization strategies | 🟡 | Cost |
| Q38 | AKS Automatic vs Standard deep dive | 🟡 | Architecture |
| Q39 | AI/ML with KAITO | 🔴 | Advanced |
| Q40 | Dapr on AKS | 🔴 | Advanced |
| Q41 | Azure Arc multi-cluster | 🔴 | Advanced |
| Q42 | Troubleshoot Pending pods | 🟡 | Troubleshooting |
| Q43 | OOMKilled diagnosis and fix | 🟡 | Troubleshooting |
| Q44 | Networking troubleshooting | 🔴 | Troubleshooting |

---
*Generated from Microsoft Learn official AKS documentation — June 2026*  
*Key refs: learn.microsoft.com/en-us/azure/aks/ | blog.aks.azure.com | github.com/MicrosoftDocs/azure-aks-docs*

---

# PART 2 — Additional Topics (Gap Fill from Official Docs)

---

## 15. Networking Deep-Dive

### 🟢 Q45. What are the AKS Service types and when do you use each?

| Service Type | Access | Azure Resource Created | Use Case |
|-------------|--------|----------------------|----------|
| `ClusterIP` | Cluster-internal only | None | Pod-to-pod communication |
| `NodePort` | Via node IP:port (30000–32767) | None | Dev/test, non-Azure LB |
| `LoadBalancer` | Public or internal VIP | Azure Load Balancer | Internet-facing / internal services |
| `ExternalName` | CNAME to external DNS | None | Off-cluster service alias |

```yaml
# Internal LoadBalancer (private IP only)
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "internal-subnet"
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
---
# External LoadBalancer with static IP
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  annotations:
    service.beta.kubernetes.io/azure-dns-label-name: "myapp"      # creates DNS label
spec:
  type: LoadBalancer
  loadBalancerIP: 20.1.2.3          # pre-allocated static public IP
  selector:
    app: frontend
  ports:
  - port: 443
    targetPort: 8443
```

---

### 🟡 Q46. How does AKS egress traffic work? (NAT Gateway, Load Balancer outbound rules, UDR)

AKS supports four **outboundType** options controlling how pods reach the internet:

| outboundType | Description | Best For |
|-------------|-------------|----------|
| `loadBalancer` | Default, SNAT via Azure LB outbound rules | Simple clusters |
| `managedNATGateway` | AKS provisions NAT GW automatically | High-SNAT, scalable egress |
| `userAssignedNATGateway` | You bring your own NAT GW | Custom egress IPs |
| `userDefinedRouting` | All egress via custom UDR (e.g., Azure Firewall) | Enterprise, security compliance |

```bash
# Create cluster with managed NAT Gateway (recommended for prod)
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --outbound-type managedNATGateway \
  --nat-gateway-managed-outbound-ip-count 2 \   # 2 public IPs on NAT GW
  --nat-gateway-idle-timeout 30                  # 30 second idle timeout

# Create cluster with Azure Firewall (UDR) egress
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --outbound-type userDefinedRouting \
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
  --network-plugin azure

# Required: UDR pointing 0.0.0.0/0 to Azure Firewall private IP
az network route-table route create \
  --resource-group myRG \
  --route-table-name aks-udr \
  --name internet \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.0.4    # Azure Firewall private IP
```

```yaml
# Required Azure Firewall rules for AKS (minimum)
# Application rules:
# - *.hcp.<region>.azmk8s.io:443   (AKS API server)
# - mcr.microsoft.com:443          (MCR images)
# - *.data.mcr.microsoft.com:443   (MCR CDN)
# - management.azure.com:443       (Azure management)
# - login.microsoftonline.com:443  (AAD auth)
# - packages.microsoft.com:443     (OS packages)
# - acs-mirror.azureedge.net:443   (AKS bootstrap)
```

---

### 🟡 Q47. How does the AKS managed Istio service mesh add-on work?

AKS provides a **managed Istio add-on** so you don't manage Istio upgrades yourself.

```bash
# Enable Istio add-on at cluster creation
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-istio-service-mesh

# Enable on existing cluster
az aks mesh enable \
  --resource-group myRG \
  --name myAKSCluster

# Enable external ingress gateway
az aks mesh enable-ingress-gateway \
  --resource-group myRG \
  --name myAKSCluster \
  --ingress-gateway-type external

# Verify Istio components
kubectl get pods -n aks-istio-system
kubectl get pods -n aks-istio-ingress
```

```yaml
# Enable sidecar injection per namespace
kubectl label namespace production istio.io/rev=asm-1-20  # revision-based label for AKS managed Istio

# Istio VirtualService for traffic splitting (canary)
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: myapp
        subset: v2
  - route:
    - destination:
        host: myapp
        subset: v1
      weight: 90
    - destination:
        host: myapp
        subset: v2
      weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

---

### 🔴 Q48. How do you set up Virtual Nodes (ACI burst) for serverless pod scaling?

**Virtual Nodes** use the Azure Container Instances (ACI) virtual kubelet to burst pods onto serverless infrastructure instantly — no VM provisioning wait.

```bash
# Enable Virtual Nodes add-on (requires Azure CNI + specific VNet setup)
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
  --enable-addons virtual-node \
  --aci-subnet-name aci-subnet         # separate subnet for ACI pods

# Verify virtual node
kubectl get nodes | grep virtual
# NAME                   STATUS   ROLES   AGE
# virtual-node-aci-linux Ready    agent   1m
```

```yaml
# Deploy burst workload to ACI virtual node
apiVersion: apps/v1
kind: Deployment
metadata:
  name: burst-app
spec:
  replicas: 1
  template:
    spec:
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet            # target virtual node
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Exists
      containers:
      - name: app
        image: mcr.microsoft.com/oss/nginx/nginx:1.25
        resources:
          requests:
            cpu: 0.5
            memory: 1Gi
          limits:
            cpu: 2
            memory: 4Gi
---
# HPA that bursts onto virtual node
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: burst-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: burst-app
  minReplicas: 1
  maxReplicas: 100               # spills onto ACI beyond node capacity
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

---

## 16. Storage Deep-Dive

### 🟡 Q49. What are Ephemeral OS disks and when should you use them in AKS?

**Ephemeral OS disks** store the OS disk in local VM cache/temp storage rather than remote Azure-managed disks. This gives significantly lower latency for OS I/O and reduced cost (no managed disk charge).

```bash
# Create node pool with ephemeral OS disk
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name ephemeralnp \
  --node-vm-size Standard_D8s_v5 \    # must have adequate local cache
  --node-osdisk-type Ephemeral \
  --node-osdisk-size 100              # must fit in VM cache size

# Check VM SKU cache size before choosing
az vm list-skus \
  --location eastus \
  --size Standard_D8s_v5 \
  --query "[].capabilities[?name=='CachedDiskBytes']"
```

| Disk Type | Location | Cost | Latency | Persist on deallocate |
|-----------|---------|------|---------|----------------------|
| Managed OS Disk | Azure Storage | Charged | ~ms | Yes |
| Ephemeral OS Disk | VM local cache | Free | ~µs | No |

---

### 🔴 Q50. What is Azure Container Storage and how does it integrate with AKS?

**Azure Container Storage** (ACS, GA 2024) is a volume management service built on top of Azure Block Storage that provides high-performance, low-latency volumes for stateful workloads with a StoragePool abstraction.

```bash
# Enable Azure Container Storage on AKS
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-azure-container-storage azureDisk   # azureDisk | ephemeralDisk | elasticSanVolume

# Verify storage pool
kubectl get storagepool -n acstor
kubectl get storagepoolclass
```

```yaml
# StoragePool backed by Azure Disk
apiVersion: containerstorage.azure.com/v1
kind: StoragePool
metadata:
  name: azuredisk-pool
  namespace: acstor
spec:
  poolType:
    azureDisk:
      skuName: Premium_LRS
  resources:
    requests:
      storage: 1Ti                   # total pool size
---
# StorageClass referencing the pool
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: acstor-azuredisk
provisioner: containerstorage.azure.com
parameters:
  pool: azuredisk-pool
  namespace: acstor
volumeBindingMode: WaitForFirstConsumer
---
# PVC using Azure Container Storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fast-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: acstor-azuredisk
  resources:
    requests:
      storage: 100Gi
```

---

## 17. Security Deep-Dive

### 🟡 Q51. How do you enable and use Microsoft Defender for Containers with AKS?

```bash
# Enable Defender for Containers (via Defender for Cloud)
az security pricing create \
  --name Containers \
  --tier Standard

# Or via AKS directly
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-defender

# Verify Defender sensor DaemonSet
kubectl get daemonset microsoft-defender-collector-ds -n kube-system
kubectl get daemonset microsoft-defender-publisher-ds -n kube-system
```

**Defender for Containers capabilities:**
- ACR image scanning (push-triggered + periodic)
- Runtime threat detection (process anomalies, network, crypto mining)
- K8s audit log analysis (API server anomalies)
- Security posture recommendations (CIS K8s benchmark)
- Agentless vulnerability assessment (no DaemonSet needed for posture)

```kusto
// KQL: View Defender security alerts for AKS
SecurityAlert
| where ProviderName == "Azure Defender for Kubernetes"
| where TimeGenerated > ago(24h)
| project TimeGenerated, AlertName, AlertSeverity, Entities, ExtendedProperties
| order by AlertSeverity asc, TimeGenerated desc
```

---

### 🟡 Q52. How do you manage Key Vault secrets in AKS with the Secrets Store CSI Driver?

```bash
# Install Secrets Store CSI Driver + Azure Key Vault provider (via add-on)
az aks enable-addons \
  --resource-group myRG \
  --name myAKSCluster \
  --addons azure-keyvault-secrets-provider

# Verify
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=csi-secrets-store-provider-azure

# Enable secret auto-rotation (poll KV for changes)
az aks addon update \
  --resource-group myRG \
  --name myAKSCluster \
  --addon azure-keyvault-secrets-provider \
  --enable-secret-rotation \
  --rotation-poll-interval 2m
```

```yaml
# SecretProviderClass - fetch from Key Vault + sync as K8s Secret
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: kv-secrets
  namespace: production
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: "<workload-identity-client-id>"
    keyvaultName: "myKeyVault"
    tenantId: "<tenant-id>"
    objects: |
      array:
        - |
          objectName: db-conn-string
          objectType: secret
        - |
          objectName: api-key
          objectType: secret
        - |
          objectName: tls-cert
          objectType: cert
        - |
          objectName: tls-key
          objectType: key
  secretObjects:
  - secretName: app-secrets
    type: Opaque
    data:
    - objectName: db-conn-string
      key: DB_CONNECTION_STRING
    - objectName: api-key
      key: API_KEY
  - secretName: tls-secret
    type: kubernetes.io/tls
    data:
    - objectName: tls-cert
      key: tls.crt
    - objectName: tls-key
      key: tls.key
```

---

### 🟡 Q53. How do you implement OPA Gatekeeper constraint policies in AKS?

```bash
# Azure Policy add-on uses Gatekeeper under the hood
az aks enable-addons \
  --addons azure-policy \
  --resource-group myRG \
  --name myAKSCluster

# View Gatekeeper pods
kubectl get pods -n gatekeeper-system

# List installed ConstraintTemplates
kubectl get constrainttemplates
```

```yaml
# Custom ConstraintTemplate: require labels on all deployments
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      violation[{"msg": msg, "details": {"missing_labels": missing}}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("You must provide labels: %v", [missing])
      }
---
# Constraint: enforce required labels in production namespace
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-owner-labels
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
    namespaces: ["production"]
  parameters:
    labels: ["app", "owner", "team", "version"]
```

---

### 🔴 Q54. How do you enable etcd encryption at rest with KMS in AKS?

**KMS (Key Management Service)** integration encrypts etcd data using customer-managed keys in Azure Key Vault.

```bash
# Enable KMS etcd encryption at cluster creation
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-azure-keyvault-kms \
  --azure-keyvault-kms-key-id https://myKeyVault.vault.azure.net/keys/myKey/version \
  --azure-keyvault-kms-key-vault-network-access Public

# Enable on existing cluster
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-azure-keyvault-kms \
  --azure-keyvault-kms-key-id https://myKeyVault.vault.azure.net/keys/myKey/version

# Rotate KMS key (trigger re-encryption of etcd)
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --azure-keyvault-kms-key-id https://myKeyVault.vault.azure.net/keys/myKey/newVersion
```

---

### 🔴 Q55. What are Confidential Computing nodes in AKS?

AKS supports **Intel SGX** (DCsv2/DCsv3) and **AMD SEV-SNP Confidential VMs** (DCasv5/ECasv5) for sensitive workloads requiring hardware-level isolation.

```bash
# Add Intel SGX confidential computing node pool
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name sgxpool \
  --node-vm-size Standard_DC4s_v3 \   # Intel SGX SKU
  --node-count 2 \
  --enable-addons confcom              # confcom add-on for EPC memory device plugin

# Add AMD SEV-SNP Confidential VM pool
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name cvmpool \
  --node-vm-size Standard_DC4as_v5 \  # AMD SEV-SNP CVM SKU
  --os-sku AzureLinux

# Verify confcom add-on
kubectl get pods -n kube-system | grep confcom
```

```yaml
# SGX workload requesting EPC memory
apiVersion: v1
kind: Pod
metadata:
  name: sgx-app
spec:
  nodeSelector:
    kubernetes.azure.com/sgx_epc_mem_in_MiB: "8"
  containers:
  - name: sgx-app
    image: myacr.azurecr.io/sgx-app:latest
    resources:
      limits:
        sgx.intel.com/epc: "8Mi"       # request EPC memory
```

---

## 18. Cluster Reliability

### 🟡 Q56. How do Pod Disruption Budgets (PDB) protect workloads during AKS upgrades?

PDBs define the minimum number of pods that must remain available during **voluntary disruptions** (upgrades, node drains, CA scale-down).

```yaml
# PDB - allow max 1 pod disruption at a time
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2                      # at least 2 pods always up
  # OR: maxUnavailable: 1             # at most 1 down at a time
  selector:
    matchLabels:
      app: myapp
---
# For StatefulSets - protect quorum
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: redis-pdb
spec:
  minAvailable: "67%"                  # maintain quorum for 3-node cluster
  selector:
    matchLabels:
      app: redis
```

```bash
# WARNING: PDB misconfiguration blocks AKS upgrades
# If minAvailable == replicas, drain is impossible
# Best practice: minAvailable should be < replicas

# Check if PDB is blocking a node drain
kubectl get pdb -A
kubectl describe pdb myapp-pdb -n production

# Force drain if PDB is blocking (break glass - use carefully)
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --disable-eviction=true             # bypasses PDB
```

---

### 🟡 Q57. How do Topology Spread Constraints work for zone-aware scheduling?

```yaml
# Spread pods evenly across Availability Zones
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1                     # max difference between zones
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule   # or ScheduleAnyway
        labelSelector:
          matchLabels:
            app: myapp
      - maxSkew: 1                     # also spread across nodes
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: myapp
      containers:
      - name: app
        image: myapp:latest
```

```bash
# Create zone-redundant node pool (spans all 3 AZs)
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name zonepool \
  --zones 1 2 3 \                     # spread VMSS across AZs
  --node-count 3                      # 1 node per AZ

# Zone-redundant storage for StatefulSets
# Use ZRS storage class (replicates across zones)
kubectl get storageclass | grep zrs
# managed-csi-premium-zrs available in AKS 1.25+
```

---

### 🟡 Q58. What are liveness, readiness, and startup probes and why are they critical in AKS?

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080

        # Startup probe: give slow-starting apps time to initialize
        # Checked first — liveness/readiness don't run until startup succeeds
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          failureThreshold: 30         # 30 * 10s = 5 min max startup time
          periodSeconds: 10

        # Readiness probe: pod removed from Service endpoints when failing
        # Use for: app init, downstream dependency checks
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
          successThreshold: 1

        # Liveness probe: pod restarted when failing
        # Use for: deadlock detection, hung processes
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 15      # after startup succeeds
          periodSeconds: 20
          failureThreshold: 3
          timeoutSeconds: 5
```

---

## 19. Windows & Specialized Node Pools

### 🟡 Q59. How do you add Windows Server node pools to AKS?

```bash
# Create cluster with Windows node pool support
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --network-plugin azure \            # Windows requires Azure CNI
  --windows-admin-username azureuser \
  --windows-admin-password "P@ssword1234!" \
  --node-count 2

# Add Windows Server node pool
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name winpool \
  --os-type Windows \
  --os-sku Windows2022 \             # Windows2019 | Windows2022 | WindowsAnnual
  --node-vm-size Standard_D4s_v5 \
  --node-count 2

# Check Windows nodes
kubectl get nodes -l kubernetes.io/os=windows
```

```yaml
# Windows-specific pod spec
apiVersion: apps/v1
kind: Deployment
metadata:
  name: win-app
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: windows     # schedule only on Windows nodes
      containers:
      - name: win-app
        image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022
        ports:
        - containerPort: 80
      tolerations:
      - key: "os"
        operator: "Equal"
        value: "windows"
        effect: "NoSchedule"
```

---

### 🟡 Q60. How do you add GPU node pools and manage GPU workloads in AKS?

```bash
# Add GPU node pool (NVIDIA)
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name gpupool \
  --node-vm-size Standard_NC6s_v3 \   # or Standard_NC4as_T4_v3 (T4 GPU)
  --node-count 2 \
  --node-taints sku=gpu:NoSchedule

# AKS auto-installs NVIDIA device plugin
# Verify GPU nodes
kubectl get nodes -l accelerator=nvidia

# Check GPU allocation
kubectl describe node <gpu-node> | grep -A10 "Allocatable"
# nvidia.com/gpu: 1
```

```yaml
# GPU workload
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-app
spec:
  template:
    spec:
      tolerations:
      - key: sku
        value: gpu
        effect: NoSchedule
      containers:
      - name: gpu-app
        image: nvcr.io/nvidia/cuda:12.0-base-ubuntu20.04
        command: ["nvidia-smi"]
        resources:
          limits:
            nvidia.com/gpu: 1         # request 1 GPU
---
# Time-sliced GPU sharing (multiple pods share 1 GPU)
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  any: |
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        replicas: 4                   # 4 pods can share 1 GPU
```

---

## 20. Multi-Tenancy

### 🔴 Q61. How do you implement multi-tenancy on AKS for multiple teams?

Multi-tenancy in AKS uses a layered approach: **namespace isolation + RBAC + Network Policies + Resource Quotas + Pod Security**.

```bash
# Create isolated namespace for a team
kubectl create namespace team-payments

# Apply PSA restricted profile
kubectl label namespace team-payments \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted
```

```yaml
# Full multi-tenant namespace setup
---
# 1. ResourceQuota - cap compute per team
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-payments
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services.loadbalancers: "2"
    persistentvolumeclaims: "10"
---
# 2. LimitRange - defaults per container
apiVersion: v1
kind: LimitRange
metadata:
  name: team-limits
  namespace: team-payments
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
---
# 3. Network Policy - namespace isolation
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: team-payments
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector: {}               # only from same namespace
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx       # allow from ingress controller NS
  egress:
  - to:
    - podSelector: {}               # same namespace
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - port: 53                      # allow DNS
      protocol: UDP
---
# 4. RBAC - team gets admin in their namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-payments-admin
  namespace: team-payments
subjects:
- kind: Group
  name: "<AAD-group-object-id-payments-team>"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin                       # K8s built-in admin role (namespace-scoped)
  apiGroup: rbac.authorization.k8s.io
```

---

## 21. AKS Operations

### 🟡 Q62. How do you configure and use AKS maintenance windows?

```bash
# Create planned maintenance window (Sunday 2–6 AM)
az aks maintenanceconfiguration add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name default \
  --weekday Sunday \
  --start-hour 2

# Add maintenance configuration for node image upgrades (separate from cluster upgrades)
az aks maintenanceconfiguration add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name aksManagedNodeOSUpgradeSchedule \
  --schedule-type Weekly \
  --day-of-week Sunday \
  --start-hour 3 \
  --duration 4 \                    # 4-hour window
  --utc-offset +05:30               # IST (Mumbai)

# List maintenance configs
az aks maintenanceconfiguration list \
  --resource-group myRG \
  --cluster-name myAKSCluster
```

---

### 🟡 Q63. How does AKS node auto-repair work?

AKS **node auto-repair** continuously monitors node health and automatically repairs or reimages unhealthy nodes.

A node is considered unhealthy when:
- `NotReady` status for >10 minutes
- No status report for >10 minutes

```bash
# Node auto-repair is ENABLED by default in AKS
# No configuration needed — Azure monitors and reimages nodes automatically

# Check node conditions
kubectl describe node <node-name> | grep -A10 "Conditions:"

# View auto-repair events in Azure Activity Log
az monitor activity-log list \
  --resource-group MC_myRG_myAKSCluster_eastus \
  --query "[?operationName.value=='Microsoft.Compute/virtualMachineScaleSets/reimage/action']"

# Force cordon/drain (manual repair)
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
# After reimage:
kubectl uncordon <node-name>
```

---

### 🟡 Q64. How do you access AKS diagnostic logs and resource logs?

```bash
# Enable diagnostic settings (send K8s API server logs to Log Analytics)
az monitor diagnostic-settings create \
  --name aks-diagnostics \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.ContainerService/managedClusters/myAKSCluster \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA \
  --logs '[
    {"category":"kube-apiserver","enabled":true},
    {"category":"kube-controller-manager","enabled":true},
    {"category":"kube-scheduler","enabled":true},
    {"category":"kube-audit","enabled":true},
    {"category":"kube-audit-admin","enabled":true},
    {"category":"guard","enabled":true},
    {"category":"cluster-autoscaler","enabled":true}
  ]'
```

```kusto
// KQL: API server audit log - who deleted what
AzureDiagnostics
| where Category == "kube-audit"
| where TimeGenerated > ago(1h)
| extend log = parse_json(log_s)
| where log.verb == "delete"
| project TimeGenerated, user=log.user.username, resource=log.objectRef.resource, 
          name=log.objectRef.name, namespace=log.objectRef.namespace
| order by TimeGenerated desc

// KQL: Failed authentication attempts
AzureDiagnostics
| where Category == "guard"
| where TimeGenerated > ago(24h)
| where log_s contains "Unauthorized"
| summarize count() by bin(TimeGenerated, 1h)
```

---

## 22. Image Security & Supply Chain

### 🟡 Q65. How do you secure the container image supply chain for AKS?

```bash
# 1. Enable Content Trust / Image Signing with Notation
# Install notation CLI
az acr supply-chain policy create \
  --registry myACR \
  --policy-name block-unsigned \
  --image-filters "myacr.azurecr.io/*:*" \
  --policy-type SoftwareSupplyChain

# 2. Sign image with Notation + Azure Key Vault
notation sign \
  --key "myKeyVault/keys/mySigningKey" \
  myacr.azurecr.io/myapp:v1.0

# 3. Enable Image Cleaner (remove unused/vulnerable images from nodes)
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-image-cleaner \
  --image-cleaner-interval-hours 48     # scan every 48 hours

# 4. ACR Geo-replication (resilience)
az acr replication create \
  --registry myACR \
  --location westus2

# 5. ACR Private Endpoint (no public internet for image pulls)
az network private-endpoint create \
  --name acrPrivateEndpoint \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet aks-subnet \
  --private-connection-resource-id /subscriptions/.../registries/myACR \
  --group-id registry
```

```yaml
# OPA policy: only allow images from approved ACR
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allow-only-myacr
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    repos:
    - "myacr.azurecr.io/"           # only images from our ACR
```

---

## 23. DevOps Additions

### 🟡 Q66. How do you deploy AKS infrastructure with Terraform?

```hcl
# main.tf - AKS cluster with Terraform AzureRM provider
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.90"
    }
  }
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstate12345"
    container_name       = "tfstate"
    key                  = "aks.tfstate"
  }
}

resource "azurerm_resource_group" "aks" {
  name     = "aks-rg"
  location = "East US"
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "myAKSCluster"
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  dns_prefix          = "myakscluster"
  kubernetes_version  = "1.30"

  default_node_pool {
    name                = "system"
    node_count          = 3
    vm_size             = "Standard_D4s_v5"
    os_sku              = "AzureLinux"
    zones               = ["1", "2", "3"]
    enable_auto_scaling = true
    min_count           = 2
    max_count           = 10
    vnet_subnet_id      = azurerm_subnet.aks.id
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    network_policy    = "calico"
    outbound_type     = "managedNATGateway"
    load_balancer_sku = "standard"
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.aks.id
  }

  azure_active_directory_role_based_access_control {
    managed            = true
    azure_rbac_enabled = true
  }

  workload_identity_enabled = true
  oidc_issuer_enabled       = true

  auto_upgrade_channel = "patch"

  maintenance_window_auto_upgrade {
    frequency   = "Weekly"
    interval    = 1
    day_of_week = "Sunday"
    start_hour  = 2
    utc_offset  = "+05:30"
    duration    = 4
  }

  tags = {
    Environment = "Production"
    Team        = "Platform"
  }
}

resource "azurerm_kubernetes_cluster_node_pool" "user" {
  name                  = "userpool"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id
  vm_size               = "Standard_D4s_v5"
  zones                 = ["1", "2", "3"]
  enable_auto_scaling   = true
  min_count             = 2
  max_count             = 20
  mode                  = "User"
  vnet_subnet_id        = azurerm_subnet.aks.id

  node_labels = {
    "workload-type" = "application"
  }
}

output "kube_config" {
  value     = azurerm_kubernetes_cluster.aks.kube_config_raw
  sensitive = true
}
```

---

### 🟡 Q67. How do you set up GitHub Actions to deploy to AKS?

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy to AKS

on:
  push:
    branches: [main]

env:
  ACR_NAME: myacr
  AKS_CLUSTER: myAKSCluster
  AKS_RG: myRG
  IMAGE: myapp

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write       # required for OIDC auth to Azure
      contents: read

    steps:
    - uses: actions/checkout@v4

    - name: Login to Azure (OIDC - no secrets needed)
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Login to ACR
      run: az acr login --name ${{ env.ACR_NAME }}

    - name: Build and push image
      run: |
        IMAGE_TAG=${{ env.ACR_NAME }}.azurecr.io/${{ env.IMAGE }}:${{ github.sha }}
        docker build -t $IMAGE_TAG .
        docker push $IMAGE_TAG
        echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_ENV

    - name: Set AKS context
      uses: azure/aks-set-context@v4
      with:
        resource-group: ${{ env.AKS_RG }}
        cluster-name: ${{ env.AKS_CLUSTER }}

    - name: Deploy to AKS
      uses: azure/k8s-deploy@v5
      with:
        namespace: production
        manifests: |
          k8s/deployment.yaml
          k8s/service.yaml
        images: |
          ${{ env.IMAGE_TAG }}
        strategy: canary
        percentage: 25             # deploy to 25% first
```

---

## 24. Bicep for AKS

### 🟡 Q68. How do you define AKS with Bicep (Infrastructure as Code)?

```bicep
// aks.bicep
param clusterName string = 'myAKSCluster'
param location string = resourceGroup().location
param kubernetesVersion string = '1.30'
param nodeCount int = 3

resource aks 'Microsoft.ContainerService/managedClusters@2024-01-01' = {
  name: clusterName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    kubernetesVersion: kubernetesVersion
    dnsPrefix: clusterName
    agentPoolProfiles: [
      {
        name: 'system'
        count: nodeCount
        vmSize: 'Standard_D4s_v5'
        osType: 'Linux'
        osSKU: 'AzureLinux'
        mode: 'System'
        availabilityZones: ['1', '2', '3']
        enableAutoScaling: true
        minCount: 2
        maxCount: 10
      }
    ]
    networkProfile: {
      networkPlugin: 'azure'
      networkPolicy: 'calico'
      outboundType: 'managedNATGateway'
      loadBalancerSku: 'standard'
    }
    aadProfile: {
      managed: true
      enableAzureRBAC: true
    }
    workloadAutoScalerProfile: {
      keda: {
        enabled: true
      }
    }
    oidcIssuerProfile: {
      enabled: true
    }
    securityProfile: {
      workloadIdentity: {
        enabled: true
      }
      defender: {
        securityMonitoring: {
          enabled: true
        }
      }
    }
    autoUpgradeProfile: {
      upgradeChannel: 'patch'
    }
  }
}

output controlPlaneFQDN string = aks.properties.fqdn
output kubeletIdentityObjectId string = aks.properties.identityProfile.kubeletidentity.objectId
```

---

## 25. AKS Long-Term Support (LTS)

### 🟡 Q69. What is AKS Long-Term Support and when should you use it?

```bash
# LTS extends Kubernetes version support from 1 year to 2 years
# Requires Premium tier

# Enable LTS on cluster creation
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --kubernetes-version 1.27 \
  --tier premium \
  --k8s-support-plan AKSLongTermSupport

# Check LTS support plan
az aks show \
  --resource-group myRG \
  --name myAKSCluster \
  --query "supportPlan"
# Output: "AKSLongTermSupport"

# Upgrade existing cluster to premium + LTS
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --tier premium \
  --k8s-support-plan AKSLongTermSupport
```

| Plan | Support Duration | Tier Required |
|------|-----------------|--------------|
| `KubernetesOfficial` | 1 year (N-2) | Free / Standard / Premium |
| `AKSLongTermSupport` | 2 years | Premium only |

---

## 26. FIPS & Trusted Launch

### 🔴 Q70. How do you enable FIPS 140-2 and Trusted Launch on AKS nodes?

```bash
# FIPS 140-2 compliant node pool (for compliance workloads)
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name fipspool \
  --enable-fips-image \
  --os-sku AzureLinux \
  --node-vm-size Standard_D4s_v5

# Trusted Launch (Secure Boot + vTPM) - requires Gen2 VMs
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name trustedpool \
  --node-vm-size Standard_D4s_v5 \     # Gen2 SKU
  --enable-secure-boot \
  --enable-vtpm

# Check node FIPS mode
kubectl get node <node> -o jsonpath='{.metadata.labels.kubernetes\.azure\.com/fips_enabled}'
```

---

## Updated Master Cheatsheet — Additional Commands

```bash
## EGRESS
az aks create --outbound-type managedNATGateway --nat-gateway-managed-outbound-ip-count 2
az aks create --outbound-type userDefinedRouting   # for Azure Firewall

## ISTIO
az aks mesh enable -g myRG -n myCluster
az aks mesh enable-ingress-gateway --ingress-gateway-type external
kubectl get pods -n aks-istio-system

## WINDOWS NODES
az aks nodepool add --os-type Windows --os-sku Windows2022

## GPU NODES
az aks nodepool add --node-vm-size Standard_NC6s_v3 --node-taints sku=gpu:NoSchedule

## SECURITY
az aks update --enable-defender                          # Defender for Containers
az aks enable-addons --addons azure-keyvault-secrets-provider --enable-secret-rotation
az aks enable-addons --addons azure-policy               # OPA Gatekeeper
az aks update --enable-azure-keyvault-kms --azure-keyvault-kms-key-id <key-id>

## IMAGES
az aks update --enable-image-cleaner --image-cleaner-interval-hours 48
az aks update --attach-acr myACR                        # attach ACR (AcrPull role)

## FEATURES
az aks update --enable-keda
az aks update --enable-node-auto-provisioning
az aks update --enable-workload-identity --enable-oidc-issuer
az aks update --enable-azure-container-storage azureDisk

## DIAGNOSTICS
az aks get-credentials -g myRG -n myCluster --admin    # admin kubeconfig
az aks command invoke -g myRG -n myCluster --command "kubectl get nodes"
kubectl debug node/<node> -it --image=busybox
kubectl get events --sort-by='.lastTimestamp' -A
```

---

## Complete Question Coverage Index (All 70 Questions)

| # | Question | Level | Topic |
|---|----------|-------|-------|
| Q1–Q44 | (See Part 1) | Various | Core, Networking, Storage, Security, Scaling, Upgrades, Monitoring, DevOps |
| Q45 | Service types (ClusterIP/NodePort/LB/ExternalName) | 🟢 | Networking |
| Q46 | Egress: NAT GW, LB outbound, UDR/Firewall | 🟡 | Networking |
| Q47 | AKS managed Istio add-on | 🟡 | Service Mesh |
| Q48 | Virtual Nodes (ACI burst) | 🔴 | Scaling |
| Q49 | Ephemeral OS disks | 🟡 | Storage |
| Q50 | Azure Container Storage (ACS) | 🔴 | Storage |
| Q51 | Microsoft Defender for Containers | 🟡 | Security |
| Q52 | Key Vault CSI driver (secret rotation) | 🟡 | Security |
| Q53 | OPA Gatekeeper constraint templates | 🟡 | Security |
| Q54 | KMS etcd encryption | 🔴 | Security |
| Q55 | Confidential computing (SGX, CVM) | 🔴 | Security |
| Q56 | Pod Disruption Budgets (PDB) | 🟡 | Reliability |
| Q57 | Topology Spread Constraints + AZ scheduling | 🟡 | Reliability |
| Q58 | Liveness/Readiness/Startup probes | 🟡 | Reliability |
| Q59 | Windows node pools | 🟡 | Node Pools |
| Q60 | GPU node pools | 🟡 | Node Pools |
| Q61 | Multi-tenancy (NS + RBAC + NP + Quota) | 🔴 | Multi-tenancy |
| Q62 | Maintenance windows | 🟡 | Operations |
| Q63 | Node auto-repair | 🟡 | Operations |
| Q64 | Diagnostic logs + KQL queries | 🟡 | Operations |
| Q65 | Image supply chain security | 🟡 | Security |
| Q66 | Terraform for AKS | 🟡 | IaC |
| Q67 | GitHub Actions → AKS | 🟡 | DevOps |
| Q68 | Bicep for AKS | 🟡 | IaC |
| Q69 | Long-Term Support (LTS) | 🟡 | Versions |
| Q70 | FIPS 140-2 + Trusted Launch | 🔴 | Security |

---
*Part 2 added June 2026 — gap-filled against official Microsoft Learn AKS docs*
