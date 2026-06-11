# Amazon EKS (Elastic Kubernetes Service) — Complete Study Guide
> **Source-aligned with AWS official documentation (June 2026)**  
> Format: 🟢 Basic | 🟡 Intermediate | 🔴 Advanced | YAML/CLI examples | Master Cheatsheet

---

## Table of Contents
1. [EKS Core Concepts & Architecture](#1-eks-core-concepts--architecture)
2. [Cluster Creation (eksctl, CLI, Terraform)](#2-cluster-creation)
3. [Compute Options (Managed Node Groups, Fargate, Self-Managed, Auto Mode)](#3-compute-options)
4. [Networking (VPC CNI, Load Balancers, Ingress, VPC Lattice)](#4-networking)
5. [Storage (EBS, EFS, FSx CSI Drivers)](#5-storage)
6. [Identity & Security (IAM, IRSA, Pod Identity, RBAC, Secrets)](#6-identity--security)
7. [Scaling (Karpenter, Cluster Autoscaler, HPA, KEDA)](#7-scaling)
8. [EKS Add-ons & Managed Add-ons](#8-eks-add-ons)
9. [Upgrades & Cluster Lifecycle](#9-upgrades--cluster-lifecycle)
10. [Monitoring & Observability (CloudWatch, Prometheus, OpenTelemetry)](#10-monitoring--observability)
11. [DevOps & CI/CD](#11-devops--cicd)
12. [EKS Auto Mode](#12-eks-auto-mode)
13. [Multi-Cluster & Hybrid (EKS Anywhere, Outposts)](#13-multi-cluster--hybrid)
14. [Cost Optimization](#14-cost-optimization)
15. [Troubleshooting Scenarios](#15-troubleshooting-scenarios)
16. [Master Cheatsheet](#master-cheatsheet)

---

## 1. EKS Core Concepts & Architecture

### 🟢 Q1. What is Amazon EKS and what does AWS manage?

**Amazon EKS** is a fully managed Kubernetes service. AWS manages the Kubernetes control plane — API server, etcd, scheduler, and controller manager — across **3 Availability Zones** for high availability. You manage your data plane (worker nodes).

```
EKS Architecture:
┌─────────────────────────────────────────────────────┐
│                  AWS Managed                         │
│  kube-apiserver  │  etcd  │  scheduler  │  controller│
│  (3 AZ HA)       │  (3 AZ)│             │  -manager  │
└─────────────────────────────────────────────────────┘
              ↕ ENI / private endpoint
┌─────────────────────────────────────────────────────┐
│               Customer Managed (Data Plane)          │
│  Managed Node Groups │ Fargate │ Self-managed EC2    │
│  EKS Auto Mode (Karpenter-based)                     │
└─────────────────────────────────────────────────────┘
```

**Shared responsibility model:**
| Component | AWS Responsibility | Customer Responsibility |
|-----------|-------------------|------------------------|
| Control plane | Full (HA, patching, etcd backup) | None |
| Managed node groups | OS patching, AMI | Config, IAM, taints |
| Fargate | Full infrastructure | Pod spec, IAM, namespace |
| Self-managed nodes | None | Everything |
| EKS Auto Mode | Full (like Fargate for nodes) | Pod specs, NodePools |

```bash
# EKS control plane pricing: $0.10/hr per cluster (~$72/month)
# Nodes billed at standard EC2/Fargate rates
```

---

### 🟢 Q2. What are the EKS API endpoint access modes?

```bash
# Public endpoint (default) — kubectl from internet
# Public + Private — recommended for production
# Private only — kubectl only from within VPC

# Create cluster with public + private endpoint
eksctl create cluster \
  --name myCluster \
  --region us-east-1 \
  --endpoint-public-access true \
  --endpoint-private-access true \
  --public-access-cidrs "10.0.0.0/8,203.0.113.0/32"

# Update existing cluster endpoint access
aws eks update-cluster-config \
  --region us-east-1 \
  --name myCluster \
  --resources-vpc-config \
    endpointPublicAccess=true,\
    endpointPrivateAccess=true,\
    publicAccessCidrs="10.0.0.0/8"
```

---

### 🟡 Q3. What is the aws-auth ConfigMap and EKS Access Entries?

The `aws-auth` ConfigMap (legacy) or **EKS Access Entries** (recommended, GA 2024) map IAM principals to Kubernetes RBAC.

```yaml
# LEGACY: aws-auth ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/NodeGroupRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
      - system:bootstrappers
      - system:nodes
    - rolearn: arn:aws:iam::123456789012:role/DevTeamRole
      username: dev-user
      groups:
      - dev-group
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/alice
      username: alice
      groups:
      - system:masters
```

```bash
# MODERN: EKS Access Entries (replaces aws-auth ConfigMap)
# Enable Access Entries on cluster
aws eks update-cluster-config \
  --name myCluster \
  --access-config authenticationMode=API_AND_CONFIG_MAP  # or API (entries only)

# Create access entry for an IAM role
aws eks create-access-entry \
  --cluster-name myCluster \
  --principal-arn arn:aws:iam::123456789012:role/DevRole \
  --kubernetes-groups dev-group \
  --type STANDARD

# Associate with built-in policy
aws eks associate-access-policy \
  --cluster-name myCluster \
  --principal-arn arn:aws:iam::123456789012:role/DevRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=production

# Built-in EKS access policies:
# AmazonEKSClusterAdminPolicy   → cluster-admin
# AmazonEKSAdminPolicy          → admin (namespace-scoped)
# AmazonEKSEditPolicy           → edit
# AmazonEKSViewPolicy           → view
```

---

### 🟡 Q4. What are EKS Security Groups and how do they protect the cluster?

```bash
# EKS creates a cluster security group automatically
# It allows all traffic between control plane and managed node group nodes

# View cluster security group
aws eks describe-cluster \
  --name myCluster \
  --query "cluster.resourcesVpcConfig.clusterSecurityGroupId"

# Additional node security group rules (minimum required):
# Control plane → nodes: TCP 10250 (kubelet)
# Nodes → control plane: TCP 443 (API server)
# Nodes ↔ nodes: all (for pod-to-pod)

# Security Groups for Pods (SGP) - assign SG directly to pods
# Requires VPC CNI with ENABLE_POD_ENI=true + Nitro instances
```

```yaml
# SecurityGroupPolicy CRD - assign SG to matching pods
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: my-security-group-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  securityGroups:
    groupIds:
    - sg-0123456789abcdef0        # specific SG for these pods
```

---

## 2. Cluster Creation

### 🟢 Q5. How do you create an EKS cluster with eksctl?

```bash
# Install eksctl
curl --silent --location \
  "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp && sudo mv /tmp/eksctl /usr/local/bin

# Create cluster with managed node group (quick)
eksctl create cluster \
  --name myCluster \
  --region us-east-1 \
  --version 1.30 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed

# Create cluster from config file (recommended for prod)
eksctl create cluster -f cluster.yaml
```

```yaml
# cluster.yaml - production-grade eksctl config
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: myCluster
  region: us-east-1
  version: "1.30"
  tags:
    Environment: production
    Team: platform

vpc:
  cidr: 10.0.0.0/16
  clusterEndpoints:
    publicAccess: true
    privateAccess: true
  publicAccessCIDRs:
  - "10.0.0.0/8"

iam:
  withOIDC: true                  # enables IRSA
  serviceAccounts:
  - metadata:
      name: aws-load-balancer-controller
      namespace: kube-system
    wellKnownPolicies:
      awsLoadBalancerController: true
  - metadata:
      name: cluster-autoscaler
      namespace: kube-system
    wellKnownPolicies:
      autoScaler: true
  - metadata:
      name: ebs-csi-controller-sa
      namespace: kube-system
    wellKnownPolicies:
      ebsCSIController: true

managedNodeGroups:
- name: system
  instanceType: m5.large
  minSize: 2
  maxSize: 6
  desiredCapacity: 3
  availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"]
  amiFamily: AmazonLinux2023       # AmazonLinux2023 | Bottlerocket | Ubuntu2204
  iam:
    withAddonPolicies:
      cloudWatch: true
      imageBuilder: true
  labels:
    role: system
  taints:
  - key: CriticalAddonsOnly
    value: "true"
    effect: NoSchedule
  tags:
    k8s.io/cluster-autoscaler/enabled: "true"
    k8s.io/cluster-autoscaler/myCluster: "owned"
  updateConfig:
    maxUnavailable: 1

- name: workers
  instanceTypes: ["m5.xlarge", "m5a.xlarge", "m5n.xlarge"]   # multiple types for Spot
  spot: true
  minSize: 0
  maxSize: 20
  desiredCapacity: 3
  availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"]

addons:
- name: vpc-cni
  version: latest
  attachPolicyARNs:
  - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
- name: coredns
  version: latest
- name: kube-proxy
  version: latest
- name: aws-ebs-csi-driver
  version: latest
  wellKnownPolicies:
    ebsCSIController: true

cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]
```

---

### 🔴 Q6. How do you create an EKS cluster with Terraform?

```hcl
# main.tf - EKS cluster with Terraform aws provider
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  version         = "~> 20.0"

  cluster_name    = "myCluster"
  cluster_version = "1.30"

  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access_cidrs = ["10.0.0.0/8"]

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # Enable IRSA (OIDC provider)
  enable_irsa = true

  # EKS managed add-ons
  cluster_addons = {
    coredns                = { most_recent = true }
    kube-proxy             = { most_recent = true }
    vpc-cni = {
      most_recent              = true
      service_account_role_arn = module.vpc_cni_irsa_role.iam_role_arn
    }
    aws-ebs-csi-driver = {
      most_recent              = true
      service_account_role_arn = module.ebs_csi_irsa_role.iam_role_arn
    }
    aws-efs-csi-driver = { most_recent = true }
  }

  # Managed node groups
  eks_managed_node_groups = {
    system = {
      name           = "system"
      instance_types = ["m5.large"]
      min_size       = 2
      max_size       = 6
      desired_size   = 3
      ami_type       = "AL2023_x86_64_STANDARD"   # AmazonLinux2023
      capacity_type  = "ON_DEMAND"
      subnet_ids     = module.vpc.private_subnets

      taints = {
        CriticalAddonsOnly = {
          key    = "CriticalAddonsOnly"
          value  = "true"
          effect = "NO_SCHEDULE"
        }
      }
    }

    workers = {
      name           = "workers"
      instance_types = ["m5.xlarge", "m5a.xlarge", "m5n.xlarge"]
      min_size       = 0
      max_size       = 50
      desired_size   = 3
      capacity_type  = "SPOT"
    }
  }

  # Access entries (modern auth)
  authentication_mode = "API_AND_CONFIG_MAP"

  access_entries = {
    admin = {
      principal_arn = "arn:aws:iam::123456789012:role/AdminRole"
      policy_associations = {
        admin = {
          policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
          access_scope  = { type = "cluster" }
        }
      }
    }
  }

  # Enable control plane logging
  cluster_enabled_log_types = ["api", "audit", "authenticator", "controllerManager", "scheduler"]

  tags = { Environment = "production" }
}
```

---

## 3. Compute Options

### 🟢 Q7. What are the four EKS compute options and when to use each?

| Option | Management | Isolation | Cost | Best For |
|--------|-----------|-----------|------|----------|
| **Managed Node Groups** | AWS patches AMI | Shared node | EC2 price | General workloads |
| **Self-managed Nodes** | You manage | Shared node | EC2 price | Custom AMI/kernels |
| **Fargate** | AWS full infra | Pod-level (micro-VM) | vCPU+mem/sec | Batch, bursty |
| **EKS Auto Mode** | AWS managed Karpenter | Shared (Bottlerocket) | EC2+12-15% premium | Simplest ops |

```bash
# Managed Node Group - best for most workloads
eksctl create nodegroup \
  --cluster myCluster \
  --name workers \
  --node-type m5.xlarge \
  --nodes 3 --nodes-min 1 --nodes-max 10 \
  --managed \
  --asg-access                     # allows cluster autoscaler

# Fargate Profile - for specific namespaces/labels
eksctl create fargateprofile \
  --cluster myCluster \
  --name batch-profile \
  --namespace batch \
  --labels workload-type=batch

# Verify Fargate profile
aws eks describe-fargate-profile \
  --cluster-name myCluster \
  --fargate-profile-name batch-profile
```

```yaml
# Fargate profile via CloudFormation / manifest
# Pod automatically lands on Fargate if it matches namespace+labels
apiVersion: v1
kind: Namespace
metadata:
  name: batch
---
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
  namespace: batch              # matches Fargate profile namespace
spec:
  template:
    metadata:
      labels:
        workload-type: batch    # matches Fargate profile selector
    spec:
      restartPolicy: Never
      containers:
      - name: processor
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/processor:latest
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"       # Fargate bills on requests (round up to nearest config)
```

---

### 🟡 Q8. What is Bottlerocket and when should you use it as the node OS?

**Bottlerocket** is AWS's purpose-built, minimal Linux OS for containers. It uses an immutable filesystem, SELinux, and dm-verity for integrity.

```bash
# Create node group with Bottlerocket
eksctl create nodegroup \
  --cluster myCluster \
  --name bottlerocket-ng \
  --node-type m5.xlarge \
  --node-ami-family Bottlerocket \
  --managed

# Or via eksctl config
# amiFamily: Bottlerocket
# In Bottlerocket:
# - No SSH by default (use SSM Session Manager)
# - Read-only root filesystem
# - Smaller attack surface
# - Container-optimized kernel
# - Supports FIPS 140-2

# SSM into Bottlerocket node (no SSH needed)
aws ssm start-session \
  --target i-0123456789abcdef0
```

---

## 4. Networking

### 🟢 Q9. How does the VPC CNI plugin work in EKS?

The **Amazon VPC CNI** plugin gives every pod a real VPC IP address from the node's subnet. Pods are first-class VPC citizens — no NAT, no overlay.

```bash
# VPC CNI architecture:
# - Each node pre-allocates ENIs (Elastic Network Interfaces)
# - Each ENI has multiple IP addresses (secondary IPs)
# - Each pod gets one secondary IP
# - Max pods per node = (ENIs × IPs per ENI) - 1

# View max pods for an instance type
aws ec2 describe-instance-types \
  --instance-types m5.xlarge \
  --query "InstanceTypes[].NetworkInfo.MaximumNetworkInterfaces"

# Enable prefix delegation to increase pod density (doubles pod count)
kubectl set env daemonset aws-node \
  ENABLE_PREFIX_DELEGATION=true \
  -n kube-system

# Warm pool tuning (pre-allocated IPs)
kubectl set env daemonset aws-node \
  WARM_PREFIX_TARGET=1 \
  WARM_IP_TARGET=5 \
  MINIMUM_IP_TARGET=2 \
  -n kube-system

# Custom networking (pods on different subnet from nodes)
kubectl set env daemonset aws-node \
  AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true \
  ENI_CONFIG_LABEL_DEF=topology.kubernetes.io/zone \
  -n kube-system
```

```yaml
# ENIConfig for custom networking (different subnet per AZ)
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1a
spec:
  securityGroups:
  - sg-0123456789abcdef0
  subnet: subnet-0123456789abcdef0   # pod subnet in us-east-1a
```

---

### 🟡 Q10. How do you set up the AWS Load Balancer Controller for ALB/NLB?

```bash
# Install AWS Load Balancer Controller (required for modern EKS ingress)
# Step 1: Create IAM policy + IRSA
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster myCluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

# Step 2: Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=myCluster \
  --set serviceAccountName=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-0123456789abcdef0
```

```yaml
# ALB Ingress (Application Load Balancer - Layer 7)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing      # or internal
    alb.ingress.kubernetes.io/target-type: ip              # ip | instance
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/xxx
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/group.name: shared-alb       # share ALB across Ingresses
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
---
# NLB Service (Network Load Balancer - Layer 4)
apiVersion: v1
kind: Service
metadata:
  name: myapp-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 443
    targetPort: 8443
```

---

### 🟡 Q11. How does Network Policy work in EKS?

EKS VPC CNI supports two network policy engines: **Calico** (self-managed) and the native **VPC CNI Network Policy** (AWS-managed, recommended).

```bash
# Enable VPC CNI native network policy (built into VPC CNI v1.14+)
kubectl set env daemonset aws-node \
  ENABLE_NETWORK_POLICY=true \
  -n kube-system

# Verify policy agent is running
kubectl get pods -n kube-system | grep network-policy
```

```yaml
# EKS network policy with Security Groups for Pods
# Deny all by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow app tier to call database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: application
    ports:
    - protocol: TCP
      port: 5432
  egress: []
```

---

### 🔴 Q12. How do you configure private EKS clusters with VPC endpoints?

```bash
# Private cluster: API endpoint private-only
# Requires VPC endpoints for all AWS services the nodes use

aws eks update-cluster-config \
  --name myCluster \
  --resources-vpc-config \
    endpointPublicAccess=false,\
    endpointPrivateAccess=true

# Required VPC endpoints for private EKS:
REGION=us-east-1
VPC_ID=vpc-0123456789abcdef0
SUBNET_IDS="subnet-abc,subnet-def"
SG_ID=sg-0123456789abcdef0

for SERVICE in \
  com.amazonaws.$REGION.ec2 \
  com.amazonaws.$REGION.ecr.api \
  com.amazonaws.$REGION.ecr.dkr \
  com.amazonaws.$REGION.s3 \
  com.amazonaws.$REGION.sts \
  com.amazonaws.$REGION.elasticloadbalancing \
  com.amazonaws.$REGION.autoscaling \
  com.amazonaws.$REGION.logs; do
  aws ec2 create-vpc-endpoint \
    --vpc-id $VPC_ID \
    --service-name $SERVICE \
    --vpc-endpoint-type Interface \
    --subnet-ids $SUBNET_IDS \
    --security-group-ids $SG_ID \
    --private-dns-enabled
done

# S3 gateway endpoint (free, no interface endpoint needed)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.$REGION.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-0123456789abcdef0
```

---

### 🟡 Q13. What is Amazon VPC Lattice and how does it work with EKS?

**VPC Lattice** is a managed application networking service enabling secure cross-cluster/cross-account service communication via the Gateway API.

```bash
# Install AWS Gateway API Controller for VPC Lattice
helm install gateway-api-controller \
  oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart \
  --version v1.0.0 \
  --namespace aws-application-networking-system \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123456789012:role/LatticeRole
```

```yaml
# VPC Lattice Gateway + HTTPRoute
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: my-vpc-lattice-gateway
spec:
  gatewayClassName: amazon-vpc-lattice
  listeners:
  - name: http
    protocol: HTTP
    port: 80
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: orders-route
spec:
  parentRefs:
  - name: my-vpc-lattice-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /orders
    backendRefs:
    - name: orders-svc
      port: 80
      weight: 90
    - name: orders-svc-v2
      port: 80
      weight: 10
```

---

## 5. Storage

### 🟢 Q14. What are the EKS CSI drivers and storage options?

| Storage | CSI Driver | Access Mode | Best For |
|---------|-----------|-------------|----------|
| Amazon EBS | aws-ebs-csi-driver | RWO | Databases, stateful apps |
| Amazon EFS | aws-efs-csi-driver | RWX | Shared, multi-pod |
| Amazon FSx for Lustre | aws-fsx-csi-driver | RWX | HPC, ML training |
| FSx for NetApp ONTAP | aws-fsx-ontap-csi | RWO/RWX | Enterprise NFS/SMB |
| Amazon S3 (Mountpoint) | mountpoint-s3-csi | RWO (read) | Object storage access |

```bash
# Install EBS CSI driver as managed add-on (recommended)
aws eks create-addon \
  --cluster-name myCluster \
  --addon-name aws-ebs-csi-driver \
  --addon-version v1.28.0-eksbuild.1 \
  --service-account-role-arn arn:aws:iam::123456789012:role/EBSCSIRole

# Install EFS CSI driver
aws eks create-addon \
  --cluster-name myCluster \
  --addon-name aws-efs-csi-driver \
  --addon-version v1.7.0-eksbuild.1
```

```yaml
# EBS StorageClass (gp3 recommended)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  throughput: "250"               # MB/s (gp3 default 125)
  iops: "6000"                    # gp3 max 16000
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789012:key/xxx
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# EFS StorageClass (dynamic provisioning)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap          # creates an EFS Access Point
  fileSystemId: fs-0123456789abcdef0
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"
  subPathPattern: "${.PVC.namespace}/${.PVC.name}"
---
# PVC using EFS (RWX - shared across pods)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 100Gi              # EFS is elastic, size is a hint only
```

---

### 🔴 Q15. How do you use Amazon S3 Mountpoint CSI driver for ML/data workloads?

```bash
# Install Mountpoint for Amazon S3 CSI driver
helm repo add aws-mountpoint-s3-csi-driver \
  https://awslabs.github.io/mountpoint-s3-csi-driver
helm install aws-mountpoint-s3-csi-driver \
  aws-mountpoint-s3-csi-driver/aws-mountpoint-s3-csi-driver \
  --namespace kube-system
```

```yaml
# PersistentVolume for S3 bucket
apiVersion: v1
kind: PersistentVolume
metadata:
  name: s3-pv
spec:
  capacity:
    storage: 1200Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  csi:
    driver: s3.csi.aws.com
    volumeHandle: s3-csi-driver-volume
    volumeAttributes:
      bucketName: my-ml-data-bucket
      region: us-east-1
      mountOptions: "--allow-delete --region us-east-1"
---
# ML training pod reading from S3
apiVersion: batch/v1
kind: Job
metadata:
  name: ml-training
spec:
  template:
    spec:
      serviceAccountName: ml-sa      # needs s3:GetObject policy via Pod Identity
      containers:
      - name: trainer
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/trainer:latest
        volumeMounts:
        - name: training-data
          mountPath: /data
          readOnly: true
        - name: model-output
          mountPath: /output
      volumes:
      - name: training-data
        persistentVolumeClaim:
          claimName: s3-training-pvc
      - name: model-output
        emptyDir: {}
```

---

## 6. Identity & Security

### 🟢 Q16. What is IRSA (IAM Roles for Service Accounts)?

**IRSA** lets pods assume AWS IAM roles without node-level IAM credentials, using OIDC federation.

```bash
# Step 1: Enable OIDC provider for cluster
eksctl utils associate-iam-oidc-provider \
  --cluster myCluster \
  --approve

# Get OIDC issuer URL
aws eks describe-cluster \
  --name myCluster \
  --query "cluster.identity.oidc.issuer" \
  --output text

# Step 2: Create IAM role with trust policy for SA
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_PROVIDER=$(aws eks describe-cluster --name myCluster \
  --query "cluster.identity.oidc.issuer" --output text | sed s@https://@@)

cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "${OIDC_PROVIDER}:sub": "system:serviceaccount:production:myapp-sa",
        "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name MyAppIRSARole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name MyAppIRSARole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

```yaml
# Service Account annotated with IAM role
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/MyAppIRSARole
    eks.amazonaws.com/token-expiration: "86400"   # 24h token
---
# Pod using IRSA - AWS SDK auto-detects the token
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: AWS_REGION
      value: us-east-1
    # AWS_WEB_IDENTITY_TOKEN_FILE and AWS_ROLE_ARN are auto-injected
```

---

### 🟡 Q17. What is EKS Pod Identity and how does it differ from IRSA?

**EKS Pod Identity** (GA 2024) is the newer, simpler way to grant AWS IAM access to pods. No OIDC provider management, no trust policy per cluster.

```bash
# Install Pod Identity Agent add-on (required)
aws eks create-addon \
  --cluster-name myCluster \
  --addon-name eks-pod-identity-agent \
  --addon-version v1.3.0-eksbuild.1

# Create IAM role for Pod Identity (simpler trust policy)
cat > pod-identity-trust.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "pods.eks.amazonaws.com"
    },
    "Action": ["sts:AssumeRole","sts:TagSession"]
  }]
}
EOF

aws iam create-role \
  --role-name MyAppPodIdentityRole \
  --assume-role-policy-document file://pod-identity-trust.json

aws iam attach-role-policy \
  --role-name MyAppPodIdentityRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create Pod Identity Association (links K8s SA to IAM role)
aws eks create-pod-identity-association \
  --cluster-name myCluster \
  --namespace production \
  --service-account myapp-sa \
  --role-arn arn:aws:iam::123456789012:role/MyAppPodIdentityRole
```

| Feature | IRSA | Pod Identity |
|---------|------|-------------|
| OIDC setup | Per cluster | Not needed |
| Trust policy | Per cluster/SA | Reusable across clusters |
| Agent required | No | Yes (add-on) |
| Cross-account | Needs trust policy update | Simpler |
| Recommended | Legacy but works | ✅ New standard |

---

### 🟡 Q18. How do you manage Kubernetes Secrets with AWS Secrets Manager and CSI driver?

```bash
# Install Secrets Store CSI Driver + AWS provider
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true

helm repo add aws-secrets-manager \
  https://aws.github.io/secrets-store-csi-driver-provider-aws
helm install secrets-provider-aws \
  aws-secrets-manager/secrets-store-csi-driver-provider-aws \
  --namespace kube-system
```

```yaml
# SecretProviderClass for AWS Secrets Manager
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
  namespace: production
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "myapp/production/db-password"
        objectType: "secretsmanager"
        objectAlias: "db-password"       # filename in volume
      - objectName: "myapp/production/api-key"
        objectType: "secretsmanager"
        jmesPath:                        # extract specific JSON key
        - path: "apiKey"
          objectAlias: "api-key"
      - objectName: "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/config"
        objectType: "ssmparameter"
        objectAlias: "app-config"
  secretObjects:
  - secretName: app-db-secret
    type: Opaque
    data:
    - objectName: db-password
      key: password
---
# Pod mounting the secrets
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa    # needs secretsmanager:GetSecretValue via IRSA/Pod Identity
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secrets
      mountPath: /mnt/secrets
      readOnly: true
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-db-secret
          key: password
  volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: aws-secrets
```

---

### 🔴 Q19. How do you enable etcd envelope encryption with AWS KMS in EKS?

```bash
# Create KMS key
aws kms create-key \
  --description "EKS etcd encryption key" \
  --key-usage ENCRYPT_DECRYPT

KMS_KEY_ARN=$(aws kms describe-key --key-id alias/eks-etcd \
  --query "KeyMetadata.Arn" --output text)

# Enable encryption at cluster creation
aws eks create-cluster \
  --name myCluster \
  --encryption-config \
    "[{\"provider\":{\"keyArn\":\"${KMS_KEY_ARN}\"},\"resources\":[\"secrets\"]}]"

# Enable on existing cluster (triggers re-encryption of all Secrets)
aws eks associate-encryption-config \
  --cluster-name myCluster \
  --encryption-config \
    "[{\"provider\":{\"keyArn\":\"${KMS_KEY_ARN}\"},\"resources\":[\"secrets\"]}]"

# Verify encryption config
aws eks describe-cluster \
  --name myCluster \
  --query "cluster.encryptionConfig"
```

---

## 7. Scaling

### 🟢 Q20. How does the EKS Cluster Autoscaler work?

```bash
# Deploy Cluster Autoscaler (requires node group tags)
# Node group must have tags:
# k8s.io/cluster-autoscaler/enabled = "true"
# k8s.io/cluster-autoscaler/<cluster-name> = "owned"

kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Patch with cluster name
kubectl patch deployment cluster-autoscaler \
  -n kube-system \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"cluster-autoscaler","command":["./cluster-autoscaler","--v=4","--stderrthreshold=info","--cloud-provider=aws","--skip-nodes-with-local-storage=false","--expander=least-waste","--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/myCluster","--balance-similar-node-groups","--skip-nodes-with-system-pods=false"]}]}}}}'

# Annotate with IAM role (IRSA)
kubectl annotate serviceaccount cluster-autoscaler \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/ClusterAutoscalerRole
```

---

### 🟡 Q21. How do you set up Karpenter on EKS?

```bash
# Install Karpenter (IRSA-based)
export KARPENTER_VERSION=v0.37.0
export CLUSTER_NAME=myCluster
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace karpenter \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterControllerRole \
  --set settings.clusterName=${CLUSTER_NAME} \
  --set settings.interruptionQueue=${CLUSTER_NAME} \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi
```

```yaml
# Karpenter NodePool (v1 API)
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64", "arm64"]
      - key: karpenter.k8s.aws/instance-category
        operator: In
        values: ["c", "m", "r"]
      - key: karpenter.k8s.aws/instance-generation
        operator: Gt
        values: ["2"]
      - key: karpenter.k8s.aws/instance-cpu
        operator: In
        values: ["4", "8", "16", "32"]
  limits:
    cpu: 1000
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    budgets:
    - nodes: "10%"               # max 10% nodes disrupted at once
---
# EC2NodeClass - defines node configuration
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023              # AL2023 | Bottlerocket | Ubuntu | Windows2022
  role: KarpenterNodeRole        # IAM role for nodes
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: myCluster
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: myCluster
  amiSelectorTerms:
  - alias: al2023@latest         # always latest AL2023 AMI
  blockDeviceMappings:
  - deviceName: /dev/xvda
    ebs:
      volumeSize: 100Gi
      volumeType: gp3
      encrypted: true
  instanceStorePolicy: RAID0     # NVMe instance store RAID
  metadataOptions:
    httpTokens: required         # IMDSv2 required
    httpPutResponseHopLimit: 2
```

---

## 8. EKS Add-ons

### 🟢 Q22. What are EKS managed add-ons and how do you manage them?

```bash
# List available add-ons
aws eks describe-addon-versions \
  --kubernetes-version 1.30 \
  --query "addons[].addonName" \
  --output text

# Core managed add-ons:
# vpc-cni           - Pod networking
# coredns           - Cluster DNS
# kube-proxy        - Network rules
# aws-ebs-csi-driver - Block storage
# aws-efs-csi-driver - Shared storage
# eks-pod-identity-agent - Pod Identity
# amazon-cloudwatch-observability - CW metrics/logs
# aws-guardduty-agent - Runtime threat detection

# Install add-on
aws eks create-addon \
  --cluster-name myCluster \
  --addon-name amazon-cloudwatch-observability \
  --addon-version v1.7.0-eksbuild.1 \
  --service-account-role-arn arn:aws:iam::123456789012:role/CloudWatchRole

# Update add-on (with conflict resolution)
aws eks update-addon \
  --cluster-name myCluster \
  --addon-name coredns \
  --addon-version v1.11.1-eksbuild.4 \
  --resolve-conflicts OVERWRITE   # OVERWRITE | PRESERVE | NONE

# Check add-on health
aws eks describe-addon \
  --cluster-name myCluster \
  --addon-name coredns \
  --query "addon.status"
```

---

## 9. Upgrades & Cluster Lifecycle

### 🟢 Q23. How do you upgrade an EKS cluster?

```bash
# Check available upgrade versions
aws eks describe-cluster \
  --name myCluster \
  --query "cluster.version"

aws eks describe-update --name myCluster --update-id <update-id>

# Step 1: Upgrade control plane (one minor version at a time)
aws eks update-cluster-version \
  --name myCluster \
  --kubernetes-version 1.30

# Monitor upgrade status
aws eks describe-update \
  --name myCluster \
  --update-id $(aws eks list-updates --name myCluster --query "updateIds[0]" --output text)

# Step 2: Update add-ons to compatible versions
aws eks update-addon \
  --cluster-name myCluster \
  --addon-name vpc-cni \
  --addon-version v1.18.0-eksbuild.1 \
  --resolve-conflicts OVERWRITE

aws eks update-addon --cluster-name myCluster --addon-name coredns --addon-version v1.11.1-eksbuild.4

# Step 3: Upgrade managed node groups
aws eks update-nodegroup-version \
  --cluster-name myCluster \
  --nodegroup-name workers \
  --kubernetes-version 1.30 \
  --update-config maxUnavailable=1    # or maxUnavailablePercentage=33

# Step 4: Update Karpenter (if used)
helm upgrade karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version v0.37.0 \
  --namespace karpenter
```

---

### 🟡 Q24. How do you implement blue-green node upgrades with managed node groups?

```bash
# Blue-green node group upgrade strategy
# 1. Create new node group with updated version (GREEN)
aws eks create-nodegroup \
  --cluster-name myCluster \
  --nodegroup-name workers-v2 \
  --kubernetes-version 1.30 \
  --scaling-config minSize=3,maxSize=10,desiredSize=3 \
  --instance-types m5.xlarge \
  --subnets subnet-abc subnet-def subnet-ghi \
  --node-role arn:aws:iam::123456789012:role/NodeGroupRole

# 2. Wait for GREEN to be Active
aws eks wait nodegroup-active \
  --cluster-name myCluster \
  --nodegroup-name workers-v2

# 3. Drain and delete BLUE node group
kubectl drain --ignore-daemonsets --delete-emptydir-data \
  $(kubectl get nodes -l eks.amazonaws.com/nodegroup=workers-v1 -o name)

aws eks delete-nodegroup \
  --cluster-name myCluster \
  --nodegroup-name workers-v1
```

---

## 10. Monitoring & Observability

### 🟢 Q25. How do you set up CloudWatch Container Insights for EKS?

```bash
# Install CloudWatch Observability add-on (recommended)
aws eks create-addon \
  --cluster-name myCluster \
  --addon-name amazon-cloudwatch-observability \
  --addon-version v1.7.0-eksbuild.1 \
  --service-account-role-arn arn:aws:iam::123456789012:role/CloudWatchRole

# The add-on installs:
# - CloudWatch Agent (metrics + Container Insights)
# - Fluent Bit (log forwarding to CloudWatch Logs)

# View logs in CloudWatch
# Log groups:
# /aws/containerinsights/<cluster>/application  - pod stdout/stderr
# /aws/containerinsights/<cluster>/host         - node logs
# /aws/containerinsights/<cluster>/dataplane    - kube-proxy logs
# /aws/eks/<cluster>/cluster                   - control plane logs
```

```python
# CloudWatch Logs Insights query - pod errors
import boto3

client = boto3.client('logs', region_name='us-east-1')
response = client.start_query(
    logGroupName='/aws/containerinsights/myCluster/application',
    startTime=int((datetime.now() - timedelta(hours=1)).timestamp()),
    endTime=int(datetime.now().timestamp()),
    queryString='''
        fields @timestamp, kubernetes.pod_name, kubernetes.namespace_name, log
        | filter log like /ERROR/
        | sort @timestamp desc
        | limit 100
    '''
)
```

---

### 🟡 Q26. How do you set up managed Prometheus and Grafana for EKS?

```bash
# Create Amazon Managed Service for Prometheus (AMP) workspace
aws amp create-workspace \
  --alias myEKSWorkspace

WORKSPACE_ID=$(aws amp list-workspaces \
  --alias myEKSWorkspace \
  --query "workspaces[0].workspaceId" \
  --output text)

# Install ADOT (AWS Distro for OpenTelemetry) Collector for AMP
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: adot-col-amp
  namespace: monitoring
spec:
  mode: daemonset
  serviceAccount: adot-collector-sa
  config: |
    receivers:
      prometheus:
        config:
          scrape_configs:
          - job_name: 'kubernetes-pods'
            kubernetes_sd_configs:
            - role: pod
    exporters:
      prometheusremotewrite:
        endpoint: "https://aps-workspaces.us-east-1.amazonaws.com/workspaces/${WORKSPACE_ID}/api/v1/remote_write"
        auth:
          authenticator: sigv4auth
    extensions:
      sigv4auth:
        region: "us-east-1"
        service: "aps"
    service:
      extensions: [sigv4auth]
      pipelines:
        metrics:
          receivers: [prometheus]
          exporters: [prometheusremotewrite]
EOF

# Create Amazon Managed Grafana workspace and link to AMP
aws grafana create-workspace \
  --workspace-name myGrafana \
  --account-access-type CURRENT_ACCOUNT \
  --authentication-providers AWS_SSO \
  --permission-type SERVICE_MANAGED
```

---

### 🔴 Q27. How do you enable GuardDuty EKS Runtime Monitoring?

```bash
# Enable GuardDuty EKS Protection
aws guardduty update-detector \
  --detector-id $(aws guardduty list-detectors --query "DetectorIds[0]" --output text) \
  --features '[{
    "Name": "EKS_AUDIT_LOGS",
    "Status": "ENABLED"
  },{
    "Name": "EKS_RUNTIME_MONITORING",
    "Status": "ENABLED",
    "AdditionalConfiguration": [{
      "Name": "EKS_ADDON_MANAGEMENT",
      "Status": "ENABLED"
    }]
  }]'

# Verify GuardDuty agent add-on was auto-installed
kubectl get daemonset aws-guardduty-agent -n amazon-guardduty

# GuardDuty detects:
# - Crypto mining
# - Container escapes
# - Privilege escalation
# - Lateral movement
# - Backdoor persistence
# - Malicious file execution
```

---

## 11. DevOps & CI/CD

### 🟡 Q28. How do you deploy to EKS from GitHub Actions?

```yaml
# .github/workflows/eks-deploy.yml
name: Deploy to EKS

on:
  push:
    branches: [main]

permissions:
  id-token: write       # OIDC auth
  contents: read

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: myapp
  EKS_CLUSTER: myCluster

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials (OIDC - no static keys)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build, tag, push to ECR
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

    - name: Update kubeconfig for EKS
      run: |
        aws eks update-kubeconfig \
          --region $AWS_REGION \
          --name $EKS_CLUSTER

    - name: Deploy with Helm
      run: |
        helm upgrade --install myapp ./helm/myapp \
          --namespace production \
          --set image.tag=${{ github.sha }} \
          --set image.repository=${{ steps.login-ecr.outputs.registry }}/$ECR_REPOSITORY \
          --wait --timeout 5m
```

---

### 🟡 Q29. How do you implement GitOps on EKS with Flux v2?

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux on EKS (GitHub)
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/myCluster \
  --personal

# Verify Flux components
kubectl get pods -n flux-system
```

```yaml
# Flux HelmRelease for a workload
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: myapp
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: myapp
      version: ">=1.0.0 <2.0.0"
      sourceRef:
        kind: HelmRepository
        name: myapp-charts
  values:
    replicaCount: 3
    image:
      repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp
      tag: stable
  upgrade:
    remediation:
      remediateLastFailure: true
```

---

## 12. EKS Auto Mode

### 🟡 Q30. What is EKS Auto Mode and how do you use it?

**EKS Auto Mode** (GA December 2024) is a fully-managed data plane where AWS runs Karpenter, EBS CSI, and Load Balancer Controller on your behalf. Nodes run on **Bottlerocket** AMIs and are automatically replaced every 21 days.

```bash
# Create EKS Auto Mode cluster
aws eks create-cluster \
  --name autoModeCluster \
  --kubernetes-version 1.30 \
  --role-arn arn:aws:iam::123456789012:role/EKSClusterRole \
  --resources-vpc-config subnetIds=subnet-abc,subnet-def,securityGroupIds=sg-xxx \
  --compute-config enabled=true,nodeRoleArn=arn:aws:iam::123456789012:role/EKSNodeRole,nodePools=["general-purpose","system"] \
  --kubernetes-network-config elasticLoadBalancing={enabled=true} \
  --storage-config blockStorage={enabled=true}

# Or with eksctl
eksctl create cluster \
  --name autoModeCluster \
  --enable-auto-mode
```

```yaml
# Custom NodePool for Auto Mode
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu-pool
spec:
  template:
    spec:
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand"]
      - key: eks.amazonaws.com/instance-category
        operator: In
        values: ["g"]           # GPU instances
  limits:
    cpu: 100
  disruption:
    consolidationPolicy: WhenEmpty
---
# EKS Auto Mode NodeClass (AWS managed)
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: default
spec:
  role: EKSNodeRole
  subnetSelectorTerms:
  - tags:
      kubernetes.io/cluster/autoModeCluster: shared
  securityGroupSelectorTerms:
  - tags:
      kubernetes.io/cluster/autoModeCluster: owned
  ephemeralStorage:
    size: 80Gi
    iops: 3000
    throughput: 125
```

| Feature | Auto Mode | Traditional EKS |
|---------|-----------|-----------------|
| Karpenter | AWS managed | Self-managed |
| EBS CSI | AWS managed | Managed add-on |
| LB Controller | AWS managed | Helm install |
| Node OS | Bottlerocket (immutable) | Choice |
| Node max age | 21 days auto-replace | Indefinite |
| Cost | EC2 + ~12-15% premium | EC2 only |

---

## 13. Multi-Cluster & Hybrid

### 🟡 Q31. What is EKS Anywhere and when should you use it?

```bash
# EKS Anywhere: run EKS on your own infrastructure (vSphere, bare-metal, Nutanix, etc.)

# Install eksctl-anywhere
export EKSA_RELEASE="v0.20.0"
curl -sLO "https://anywhere-assets.eks.amazonaws.com/releases/eks-a/prod/executables/eksctl-anywhere/${EKSA_RELEASE}/eksctl-anywhere-$(uname -s | tr A-Z a-z)-amd64.tar.gz" | tar xz

# Create vsphere cluster
eksctl anywhere create cluster \
  -f vsphere-cluster.yaml

# Connect to AWS for hybrid management (EKS Connector)
aws eks register-cluster \
  --name myOnPremCluster \
  --connector-config roleArn=arn:aws:iam::123456789012:role/AmazonEKSConnectorRole,provider=EKS_ANYWHERE
```

---

## 14. Cost Optimization

### 🟡 Q32. What are the key EKS cost optimization strategies?

```bash
# 1. Use Spot instances (60-90% savings)
eksctl create nodegroup \
  --cluster myCluster \
  --spot \
  --instance-types m5.xlarge,m5a.xlarge,m5n.xlarge,m4.xlarge

# 2. Karpenter consolidation (removes underutilized nodes)
# consolidationPolicy: WhenUnderutilized

# 3. Scale to zero with KEDA + Fargate for batch
# minReplicaCount: 0 with KEDA ScaledObject

# 4. Savings Plans / Reserved Instances for baseline
# Purchase Compute Savings Plan for 1 or 3 years

# 5. Use Graviton (ARM) nodes (up to 20% cheaper + 40% better perf)
eksctl create nodegroup \
  --instance-types m7g.xlarge,m7g.2xlarge \   # Graviton3
  --asg-access

# 6. Enable Cost Allocation Tags
aws eks tag-resource \
  --resource-arn arn:aws:eks:us-east-1:123456789012:cluster/myCluster \
  --tags Team=payments,Environment=production

# 7. AWS Compute Optimizer for right-sizing
aws compute-optimizer get-ecs-service-recommendations   # also works for K8s via Container Insights
```

```yaml
# Karpenter - prefer Spot, fall back to On-Demand
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: cost-optimized
spec:
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]    # tries spot first
      - key: karpenter.k8s.aws/instance-family
        operator: In
        values: ["m5", "m5a", "m5n", "m6i", "m6a", "m7i"]
      - key: karpenter.k8s.aws/instance-cpu
        operator: Gt
        values: ["1"]
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    budgets:
    - schedule: "0 9 * * MON-FRI"   # business hours: no disruption
      duration: 9h
      nodes: "0"
    - nodes: "10%"                   # off-hours: up to 10% nodes
```

---

## 15. Troubleshooting Scenarios

### 🟡 Q33. How do you troubleshoot node NotReady in EKS?

```bash
# Step 1: Check node status
kubectl describe node <node-name>
# Look for: conditions, events, kubelet version

# Step 2: Check node group in EC2 console
aws ec2 describe-instances \
  --filters Name=tag:eks:nodegroup-name,Values=workers \
  --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name}"

# Step 3: Check system logs (via SSM for Bottlerocket/AL2023)
aws ssm start-session --target i-0123456789abcdef0

# Step 4: Common causes & fixes
# Cause 1: VPC IP exhaustion (VPC CNI)
kubectl describe node <node> | grep "Insufficient"
# Fix: enable prefix delegation or add new subnet

# Cause 2: Bootstrap failure (cloud-init)
aws ec2 get-console-output --instance-id i-xxx --output text | tail -50

# Cause 3: IAM role issue (nodes can't join cluster)
aws eks describe-cluster --name myCluster \
  --query "cluster.resourcesVpcConfig"

# Cause 4: Security group blocking control plane → node 10250
aws ec2 describe-security-group-rules \
  --filters Name=group-id,Values=sg-xxx
```

---

### 🔴 Q34. How do you debug pod networking issues (DNS, connectivity) in EKS?

```bash
# Test DNS resolution
kubectl run dns-test --image=busybox --rm -it -- \
  nslookup kubernetes.default.svc.cluster.local

# Check CoreDNS health
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check VPC CNI health
kubectl get pods -n kube-system -l k8s-app=aws-node
kubectl logs -n kube-system -l k8s-app=aws-node | grep -i error

# Check available IPs per node (VPC CNI)
kubectl describe node <node> | grep "vpc.amazonaws.com/pod-eni"

# Pod-to-pod connectivity test
kubectl run nettest \
  --image=nicolaka/netshoot \
  --rm -it -- bash
# Inside: ping <pod-ip>, curl http://<service>:<port>/health

# Capture packets (requires node access)
# kubectl debug node/<node> -it --image=nicolaka/netshoot -- tcpdump -i eth0

# Check Security Group rules
aws ec2 describe-security-group-rules \
  --filters Name=group-id,Values=<sg-id> \
  --query "SecurityGroupRules[?!IsEgress]"

# VPC Flow Logs for dropped traffic
aws logs start-query \
  --log-group-name /vpc/flowlogs \
  --query-string 'fields @timestamp, srcAddr, dstAddr, action | filter action="REJECT"'
```

---

## Master Cheatsheet

### Cluster Operations
```bash
# Create / Get Credentials
eksctl create cluster -f cluster.yaml
aws eks update-kubeconfig --name myCluster --region us-east-1
aws eks update-kubeconfig --name myCluster --region us-east-1 --role-arn arn:aws:iam::xxx:role/AdminRole

# Info
aws eks describe-cluster --name myCluster
aws eks list-clusters
eksctl get cluster

# Upgrade
aws eks update-cluster-version --name myCluster --kubernetes-version 1.30
aws eks update-nodegroup-version --cluster-name myCluster --nodegroup-name workers --kubernetes-version 1.30
```

### Node Groups
```bash
# Managed node group
eksctl create nodegroup --cluster myCluster --name ng2 --node-type m5.xlarge --nodes 3 --managed
eksctl scale nodegroup --cluster myCluster --name ng2 --nodes 5
eksctl delete nodegroup --cluster myCluster --name ng2 --drain

# Spot + multi-type
eksctl create nodegroup --cluster myCluster --name spot-ng --spot \
  --instance-types m5.xlarge,m5a.xlarge,m5n.xlarge --nodes-min 0 --nodes-max 20
```

### IAM / Identity
```bash
# IRSA setup
eksctl utils associate-iam-oidc-provider --cluster myCluster --approve
eksctl create iamserviceaccount --cluster myCluster --name mysa --namespace prod \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess --approve

# Pod Identity (new)
aws eks create-pod-identity-association --cluster-name myCluster \
  --namespace prod --service-account mysa --role-arn arn:aws:iam::xxx:role/myRole

# Access Entries
aws eks create-access-entry --cluster-name myCluster \
  --principal-arn arn:aws:iam::xxx:role/DevRole --type STANDARD
aws eks associate-access-policy --cluster-name myCluster \
  --principal-arn arn:aws:iam::xxx:role/DevRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=prod
```

### Add-ons
```bash
aws eks create-addon --cluster-name myCluster --addon-name aws-ebs-csi-driver --addon-version v1.28.0-eksbuild.1
aws eks update-addon --cluster-name myCluster --addon-name coredns --resolve-conflicts OVERWRITE
aws eks list-addons --cluster-name myCluster
aws eks describe-addon --cluster-name myCluster --addon-name vpc-cni
```

### Karpenter
```bash
kubectl get nodepools                     # list NodePools
kubectl get nodeclaims                    # list provisioned nodes
kubectl get ec2nodeclasses                # list NodeClasses
kubectl annotate nodepool default karpenter.sh/do-not-disrupt=true   # protect nodepool
```

### Storage
```bash
# EBS default gp3 StorageClass — set as default
kubectl patch storageclass ebs-gp3 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### Monitoring
```bash
# CloudWatch logs
aws logs get-log-events --log-group-name /aws/eks/myCluster/cluster --log-stream-name kube-apiserver-xxx
# Control plane log types: api | audit | authenticator | controllerManager | scheduler
eksctl utils update-cluster-logging --cluster myCluster --enable-types all
```

---

## Question Coverage Index

| # | Question | Level | Topic |
|---|----------|-------|-------|
| Q1 | EKS architecture & shared responsibility | 🟢 | Architecture |
| Q2 | API endpoint access modes | 🟢 | Security |
| Q3 | aws-auth ConfigMap vs Access Entries | 🟡 | Identity |
| Q4 | Security Groups for Pods | 🟡 | Networking |
| Q5 | Create cluster with eksctl | 🟢 | Cluster Creation |
| Q6 | Create cluster with Terraform | 🔴 | IaC |
| Q7 | Four compute options (MNG/Fargate/Self/Auto) | 🟢 | Compute |
| Q8 | Bottlerocket OS | 🟡 | Compute |
| Q9 | VPC CNI plugin (prefix delegation, custom networking) | 🟡 | Networking |
| Q10 | AWS LB Controller (ALB/NLB) | 🟡 | Networking |
| Q11 | Network Policy (VPC CNI native) | 🟡 | Networking |
| Q12 | Private cluster + VPC endpoints | 🔴 | Networking |
| Q13 | VPC Lattice + Gateway API | 🔴 | Networking |
| Q14 | CSI drivers (EBS/EFS/FSx/S3) + StorageClass | 🟢 | Storage |
| Q15 | S3 Mountpoint CSI for ML workloads | 🔴 | Storage |
| Q16 | IRSA (IAM Roles for Service Accounts) | 🟡 | Identity |
| Q17 | EKS Pod Identity vs IRSA | 🟡 | Identity |
| Q18 | Secrets Manager CSI driver | 🟡 | Security |
| Q19 | KMS etcd encryption | 🔴 | Security |
| Q20 | Cluster Autoscaler | 🟢 | Scaling |
| Q21 | Karpenter (NodePool + EC2NodeClass) | 🟡 | Scaling |
| Q22 | Managed add-ons lifecycle | 🟢 | Add-ons |
| Q23 | Cluster upgrade process | 🟢 | Upgrades |
| Q24 | Blue-green node group upgrade | 🟡 | Upgrades |
| Q25 | CloudWatch Container Insights | 🟢 | Monitoring |
| Q26 | AMP + AMG (managed Prometheus/Grafana) | 🟡 | Monitoring |
| Q27 | GuardDuty EKS Runtime Monitoring | 🔴 | Security |
| Q28 | GitHub Actions → EKS | 🟡 | DevOps |
| Q29 | GitOps with Flux v2 | 🟡 | DevOps |
| Q30 | EKS Auto Mode | 🟡 | Auto Mode |
| Q31 | EKS Anywhere | 🟡 | Hybrid |
| Q32 | Cost optimization (Spot, Graviton, Savings Plans) | 🟡 | Cost |
| Q33 | Troubleshoot NotReady nodes | 🟡 | Troubleshooting |
| Q34 | Debug pod networking + DNS | 🔴 | Troubleshooting |

---
*Generated from AWS official EKS documentation and best practices — June 2026*  
*Key refs: docs.aws.amazon.com/eks | docs.aws.amazon.com/eks/latest/best-practices | karpenter.sh*

---

# PART 2 — Additional Topics (Gap Fill from Official EKS Docs)

---

## 16. Custom Launch Templates & Custom AMIs

### 🟡 Q35. How do you use custom EC2 launch templates with EKS managed node groups?

Custom launch templates let you specify custom AMIs, EBS volume config, user data, instance metadata options, and security groups — while still benefiting from managed node group lifecycle.

```bash
# Create custom launch template
aws ec2 create-launch-template \
  --launch-template-name eks-custom-nodes \
  --version-description "Custom EKS nodes" \
  --launch-template-data file://lt-data.json
```

```json
{
  "imageId": "ami-0123456789abcdef0",
  "instanceType": "m5.xlarge",
  "blockDeviceMappings": [{
    "deviceName": "/dev/xvda",
    "ebs": {
      "volumeSize": 100,
      "volumeType": "gp3",
      "iops": 3000,
      "throughput": 125,
      "encrypted": true,
      "kmsKeyId": "arn:aws:kms:us-east-1:123456789012:key/xxx",
      "deleteOnTermination": true
    }
  }],
  "metadataOptions": {
    "httpTokens": "required",
    "httpPutResponseHopLimit": 2,
    "httpEndpoint": "enabled"
  },
  "monitoring": { "enabled": true },
  "tagSpecifications": [{
    "resourceType": "instance",
    "tags": [{"key": "Environment", "value": "production"}]
  }]
}
```

```bash
# Create managed node group referencing the launch template
aws eks create-nodegroup \
  --cluster-name myCluster \
  --nodegroup-name custom-ng \
  --launch-template id=lt-0123456789abcdef0,version=1 \
  --scaling-config minSize=2,maxSize=10,desiredSize=3 \
  --node-role arn:aws:iam::123456789012:role/NodeGroupRole \
  --subnets subnet-abc subnet-def subnet-ghi

# User data for AL2023 (nodeadm format - NEW)
cat > userdata.yaml << 'USERDATA'
---
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: myCluster
    apiServerEndpoint: https://abc123.gr7.us-east-1.eks.amazonaws.com
    certificateAuthority: BASE64_CA_DATA
    cidr: 172.20.0.0/16
  kubelet:
    config:
      maxPods: 110
      clusterDNS:
      - "172.20.0.10"
    flags:
    - "--node-labels=team=platform,env=production"
USERDATA
```

---

## 17. Windows Nodes on EKS

### 🟡 Q36. How do you add Windows node pools to EKS?

```bash
# EKS supports Windows Server 2019, 2022, and 2025 (K8s 1.35+)
# Requirements: VPC CNI, CoreDNS, kube-proxy add-ons must be updated

# Enable Windows support (creates aws-auth mapping for Windows nodes)
eksctl utils install-vpc-controllers \
  --cluster myCluster \
  --approve

# Create Windows managed node group
eksctl create nodegroup \
  --cluster myCluster \
  --name windows-ng \
  --node-ami-family WindowsServer2022FullContainer \
  --node-type m5.xlarge \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 5 \
  --managed
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
      - key: "os"
        operator: "Equal"
        value: "windows"
        effect: "NoSchedule"
      containers:
      - name: iis
        image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: "2Gi"
            cpu: "1"
```

---

## 18. EKS Version Lifecycle & Extended Support

### 🟡 Q37. What is EKS Extended Support and how do you enable it?

```bash
# Standard support: 14 months (K8s version stays supported)
# Extended support: additional 12 months (26 months total) = $0.60/hr surcharge per cluster

# Kubernetes version lifecycle in EKS:
# - AWS releases new minor version ~2 months after upstream
# - Standard support: 14 months
# - Extended support: additional 12 months (auto-enrolled unless you upgrade)
# - Total max: 26 months

# Check if cluster is on extended support
aws eks describe-cluster \
  --name myCluster \
  --query "cluster.upgradePolicy"

# Configure upgrade policy (opt out of extended support → force upgrade)
aws eks update-cluster-config \
  --name myCluster \
  --upgrade-policy supportType=STANDARD   # STANDARD | EXTENDED
```

---

## 19. IPv6 Dual-Stack Clusters

### 🔴 Q38. How do you create an IPv6 EKS cluster?

```bash
# IPv6 EKS cluster: pods get IPv6 addresses from VPC
# Requirements: IPv6-enabled VPC, VPC CNI with IPv6 mode

# Create IPv6 VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --amazon-provided-ipv6-cidr-block

# Create EKS cluster with IPv6
cat > ipv6-cluster.yaml << 'EOF'
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ipv6Cluster
  region: us-east-1
  version: "1.30"
kubernetesNetworkConfig:
  ipFamily: IPv6                  # IPv4 | IPv6
vpc:
  id: vpc-0123456789abcdef0
  subnets:
    private:
      us-east-1a:
        id: subnet-abc
      us-east-1b:
        id: subnet-def
managedNodeGroups:
- name: workers
  instanceType: m5.xlarge
  minSize: 2
  maxSize: 10
EOF

eksctl create cluster -f ipv6-cluster.yaml
```

---

## 20. IMDSv2 Hardening

### 🟡 Q39. How do you enforce IMDSv2 on EKS nodes?

```bash
# IMDSv2 requires a session token — prevents SSRF-based metadata theft

# Enforce via launch template (recommended)
# metadataOptions.httpTokens = "required"
# metadataOptions.httpPutResponseHopLimit = 2  (blocks pods from calling IMDS directly)

# Enforce at account level
aws ec2 modify-instance-metadata-defaults \
  --http-tokens required \
  --http-put-response-hop-limit 2 \
  --region us-east-1

# Verify on running node
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

```yaml
# Block pod access to IMDS via NetworkPolicy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-imds
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32    # block IMDS endpoint
```

---

## 21. Policy Enforcement (OPA Gatekeeper / Kyverno)

### 🟡 Q40. How do you implement policy-as-code with Kyverno on EKS?

```bash
# Install Kyverno (simpler than Gatekeeper for EKS)
helm repo add kyverno https://kyverno.github.io/kyverno
helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --set admissionController.replicas=3 \
  --set backgroundController.replicas=2
```

```yaml
# Kyverno ClusterPolicy: require labels on all Deployments
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce    # Enforce | Audit
  background: true
  rules:
  - name: check-team-label
    match:
      any:
      - resources:
          kinds: ["Deployment"]
    validate:
      message: "Deployment must have 'team' and 'env' labels."
      pattern:
        metadata:
          labels:
            team: "?*"
            env: "?*"
---
# Kyverno: auto-add resource limits if missing
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-limits
spec:
  rules:
  - name: add-limits
    match:
      any:
      - resources:
          kinds: ["Pod"]
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - (name): "*"
            resources:
              limits:
                +(cpu): "500m"
                +(memory): "512Mi"
---
# Kyverno: block latest image tag
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-image-tag
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "Using 'latest' image tag is not allowed."
      foreach:
      - list: "request.object.spec.containers"
        deny:
          conditions:
            any:
            - key: "{{element.image}}"
              operator: Equals
              value: "*:latest"
```

---

## 22. ECR Image Security

### 🟡 Q41. How do you configure ECR image scanning and lifecycle policies?

```bash
# Enable ECR Enhanced Scanning (powered by Amazon Inspector)
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{
    "repositoryFilters": [{"filter": "*", "filterType": "WILDCARD"}],
    "scanFrequency": "CONTINUOUS_SCAN"
  }]'

# View scan findings for an image
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=latest \
  --query "imageScanFindings.findings[?severity=='CRITICAL']"

# ECR Lifecycle Policy (auto-delete old images)
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep only last 10 tagged images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["v"],
          "countType": "imageCountMoreThan",
          "countNumber": 10
        },
        "action": {"type": "expire"}
      },
      {
        "rulePriority": 2,
        "description": "Delete untagged images older than 7 days",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 7
        },
        "action": {"type": "expire"}
      }
    ]
  }'

# ECR Pull Through Cache (cache from public ECR/DockerHub)
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix "docker-hub" \
  --upstream-registry-url "registry-1.docker.io" \
  --credential-arn arn:aws:secretsmanager:us-east-1:123456789012:secret/dockerhub-creds
```

---

## 23. EKS Multi-Tenancy

### 🔴 Q42. How do you implement multi-tenancy on EKS?

```yaml
# Full namespace isolation pattern for multiple teams
---
# Team namespace with resource quota
apiVersion: v1
kind: Namespace
metadata:
  name: team-payments
  labels:
    team: payments
    pod-security.kubernetes.io/enforce: restricted
---
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
    persistentvolumeclaims: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: payments-limits
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
# RBAC: team gets admin in their namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-admin
  namespace: team-payments
subjects:
- kind: Group
  name: "payments-team"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
---
# Network isolation
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
    - podSelector: {}                  # intra-namespace only
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
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

## 24. Velero Backup & Restore

### 🟡 Q43. How do you backup and restore EKS workloads with Velero?

```bash
# Install Velero with S3 backend
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --set serviceAccount.server.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123456789012:role/VeleroRole \
  --set configuration.provider=aws \
  --set configuration.backupStorageLocation.bucket=my-velero-backups \
  --set configuration.backupStorageLocation.config.region=us-east-1 \
  --set configuration.volumeSnapshotLocation.config.region=us-east-1 \
  --set initContainers[0].name=velero-plugin-for-aws \
  --set initContainers[0].image=velero/velero-plugin-for-aws:v1.9.0 \
  --set initContainers[0].volumeMounts[0].mountPath=/target \
  --set initContainers[0].volumeMounts[0].name=plugins

# Create backup schedule (daily at 2 AM UTC)
velero schedule create daily-backup \
  --schedule "0 2 * * *" \
  --ttl 168h \                        # 7 days retention
  --include-namespaces production,staging \
  --snapshot-volumes

# On-demand backup
velero backup create mybackup-$(date +%Y%m%d) \
  --include-namespaces production \
  --snapshot-volumes

# Restore
velero restore create --from-backup mybackup-20260611 \
  --include-namespaces production \
  --namespace-mappings production:production-restored
```

---

## 25. AWS X-Ray Distributed Tracing

### 🟡 Q44. How do you enable distributed tracing with AWS X-Ray on EKS?

```bash
# Deploy X-Ray DaemonSet
kubectl apply -f https://eksworkshop.com/intermediate/245_x-ray/daemonset.files/xray-k8s-daemonset.yaml

# Or use ADOT (recommended - OpenTelemetry compatible)
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: adot-xray
  namespace: monitoring
spec:
  serviceAccount: adot-collector
  config: |
    receivers:
      otlp:
        protocols:
          grpc: {}
          http: {}
      awsxray:
        transport: udp
    exporters:
      awsxray:
        region: "us-east-1"
      awscloudwatch:
        region: us-east-1
    service:
      pipelines:
        traces:
          receivers: [otlp, awsxray]
          exporters: [awsxray]
EOF
```

```yaml
# App pod with X-Ray sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      serviceAccountName: myapp-sa    # needs xray:PutTraceSegments, xray:PutTelemetryRecords
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: AWS_XRAY_DAEMON_ADDRESS
          value: "xray-service:2000"
      - name: xray-daemon
        image: public.ecr.aws/xray/aws-xray-daemon:3.x
        ports:
        - containerPort: 2000
          protocol: UDP
        resources:
          limits:
            cpu: "32m"
            memory: "24Mi"
```

---

## 26. EKS Hybrid Nodes

### 🔴 Q45. What are EKS Hybrid Nodes and how do you use them?

**EKS Hybrid Nodes** (GA 2025) let you connect on-premises VMs or bare-metal servers as worker nodes in your EKS cluster — using your existing hardware with EKS control plane.

```bash
# Install nodeadm on on-prem server
curl -OL 'https://hybrid-assets.eks.amazonaws.com/releases/latest/bin/linux/amd64/nodeadm'
chmod +x nodeadm

# Create hybrid node IAM role
aws iam create-role \
  --role-name HybridNodesRole \
  --assume-role-policy-document file://hybrid-trust-policy.json

aws iam attach-role-policy \
  --role-name HybridNodesRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodeMinimalPolicy

# Configure nodeadm on the on-prem server
cat > nodeconfig.yaml << 'EOF'
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: myCluster
    region: us-east-1
  hybrid:
    ssm:
      activationCode: XXXXXXXXXX
      activationId: XXXXXXXXXX
EOF

sudo nodeadm init --config-source file://nodeconfig.yaml

# Verify hybrid node joined cluster
kubectl get nodes -l eks.amazonaws.com/compute-type=hybrid
```

---

## 27. EKS Add-on: AWS Controllers for Kubernetes (ACK)

### 🟡 Q46. How do you use ACK to manage AWS resources from Kubernetes?

```bash
# ACK lets you create AWS resources (S3, RDS, DynamoDB) using K8s CRDs

# Install ACK for S3
aws eks create-addon \
  --cluster-name myCluster \
  --addon-name ack-s3-controller \
  --addon-version v1.0.14-eksbuild.1 \
  --service-account-role-arn arn:aws:iam::123456789012:role/ACK-S3-Role

# Or via Helm
helm install ack-s3-controller \
  oci://public.ecr.aws/aws-controllers-k8s/s3-chart \
  --version v1.0.14 \
  --namespace ack-system \
  --create-namespace \
  --set aws.region=us-east-1 \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123456789012:role/ACK-S3-Role
```

```yaml
# Create an S3 bucket using Kubernetes CRD
apiVersion: s3.services.k8s.aws/v1alpha1
kind: Bucket
metadata:
  name: my-app-bucket
  namespace: production
spec:
  name: my-app-prod-bucket-123
  versioning:
    status: Enabled
  publicAccessBlock:
    blockPublicAcls: true
    blockPublicPolicy: true
    ignorePublicAcls: true
    restrictPublicBuckets: true
---
# Create an RDS database using Kubernetes CRD
apiVersion: rds.services.k8s.aws/v1alpha1
kind: DBInstance
metadata:
  name: myapp-postgres
  namespace: production
spec:
  dbInstanceClass: db.t3.medium
  dbInstanceIdentifier: myapp-postgres
  engine: postgres
  engineVersion: "15"
  masterUsername: admin
  masterUserPassword:
    namespace: production
    name: rds-password
    key: password
  allocatedStorage: 100
  multiAZ: true
```

---

## 28. PDB + Topology Spread Constraints

### 🟡 Q47. How do PDB and Topology Spread Constraints protect EKS workloads?

```yaml
# PDB: protect during node drains and upgrades
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: "60%"              # always 60% up during disruptions
  selector:
    matchLabels:
      app: myapp
---
# Topology Spread across AZs (EKS multi-AZ pattern)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: myapp
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: myapp
```

---

## 29. CoreDNS Customization

### 🟡 Q48. How do you customize CoreDNS in EKS?

```yaml
# Customize CoreDNS ConfigMap
kubectl edit configmap coredns -n kube-system

# Or apply patch
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
    # Custom: forward corp.internal to on-prem DNS
    corp.internal:53 {
        errors
        cache 30
        forward . 10.0.0.53 10.0.1.53
    }
    # Custom: stub zone for EKS private API
    eks.amazonaws.com:53 {
        errors
        cache 30
        forward . 169.254.169.253
    }
```

---

## 30. EKS Platform Versions & Kubernetes Version Policy

### 🟢 Q49. What are EKS platform versions and what is the Kubernetes version policy?

```bash
# EKS platform versions: eks.1, eks.2, etc.
# New patch versions and security fixes delivered via platform version updates
# Format: kubernetes_version-eksPlatformVersion (e.g., 1.30.5-eks.3)

# View cluster platform version
aws eks describe-cluster \
  --name myCluster \
  --query "cluster.platformVersion"

# EKS Kubernetes version lifecycle:
# - New minor version: ~2 months after upstream release
# - Standard support: 14 months
# - Extended support: +12 months (total 26) at $0.60/cluster/hr
# - Auto-enrolled in extended support if not upgraded in time

# Check all version end-of-support dates
aws eks describe-addon-versions \
  --kubernetes-version 1.28 \
  --query "addons[0].marketplaceVersion"

# EKS currently supports (June 2026):
# 1.30, 1.31, 1.32, 1.33, 1.34 (standard)
# 1.28, 1.29 (extended support)
```

---

## Updated EKS Master Cheatsheet — Additional Commands

```bash
## LAUNCH TEMPLATES
aws ec2 create-launch-template --launch-template-name eks-custom --launch-template-data file://lt.json
aws eks create-nodegroup --launch-template id=lt-xxx,version=1 ...

## WINDOWS NODES
eksctl create nodegroup --node-ami-family WindowsServer2022FullContainer ...

## ECR
aws ecr put-registry-scanning-configuration --scan-type ENHANCED ...
aws ecr describe-image-scan-findings --repository-name myapp --image-id imageTag=v1.0
aws ecr put-lifecycle-policy --repository-name myapp --lifecycle-policy-text file://lp.json

## SECURITY
# IMDSv2 enforcement
aws ec2 modify-instance-metadata-defaults --http-tokens required --http-put-response-hop-limit 2
# KMS encryption
aws eks associate-encryption-config --cluster-name myCluster --encryption-config '[{"provider":{"keyArn":"arn:..."},"resources":["secrets"]}]'

## BACKUP
velero backup create mybackup --include-namespaces production --snapshot-volumes
velero restore create --from-backup mybackup-xxx

## EXTENDED SUPPORT
aws eks update-cluster-config --name myCluster --upgrade-policy supportType=STANDARD

## HYBRID NODES
kubectl get nodes -l eks.amazonaws.com/compute-type=hybrid

## ACK
helm install ack-s3-controller oci://public.ecr.aws/aws-controllers-k8s/s3-chart ...
kubectl get buckets.s3.services.k8s.aws
kubectl get dbinstances.rds.services.k8s.aws
```

---

## Additional Question Coverage Index (Part 2)

| # | Question | Level | Topic |
|---|----------|-------|-------|
| Q35 | Custom launch templates + AL2023 nodeadm | 🟡 | Compute |
| Q36 | Windows node pools | 🟡 | Compute |
| Q37 | Extended Support (14→26 months) | 🟡 | Versions |
| Q38 | IPv6 dual-stack clusters | 🔴 | Networking |
| Q39 | IMDSv2 hardening + block pods from IMDS | 🟡 | Security |
| Q40 | Kyverno policy-as-code | 🟡 | Security |
| Q41 | ECR enhanced scanning + lifecycle policies | 🟡 | Security |
| Q42 | Multi-tenancy (NS + RBAC + NP + Quota) | 🔴 | Multi-tenancy |
| Q43 | Velero backup & restore | 🟡 | Operations |
| Q44 | AWS X-Ray distributed tracing | 🟡 | Monitoring |
| Q45 | EKS Hybrid Nodes (on-prem as EKS workers) | 🔴 | Hybrid |
| Q46 | ACK (AWS Controllers for Kubernetes) | 🟡 | Add-ons |
| Q47 | PDB + Topology Spread Constraints | 🟡 | Reliability |
| Q48 | CoreDNS customization | 🟡 | Networking |
| Q49 | Platform versions & K8s version policy | 🟢 | Versions |

---
*Part 2 added June 2026 — gap-filled against AWS official EKS documentation*
*Key refs: docs.aws.amazon.com/eks/latest/userguide | docs.aws.amazon.com/eks/latest/best-practices*
