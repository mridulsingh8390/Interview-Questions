# AWS Containers, Developer Tools & Management/Governance — Complete Interview Q&A
> **All possible questions | June 2026 | AWS Documentation aligned**
> 🟢 Basic | 🟡 Intermediate | 🔴 Advanced | Full CLI + YAML + code examples

---

## TABLE OF CONTENTS
| Part | Category | Questions |
|------|---------|----------|
| [1](#part-1--aws-containers) | AWS Containers | Q1–Q40 |
| [2](#part-2--aws-developer-tools) | AWS Developer Tools | Q41–Q80 |
| [3](#part-3--aws-management--governance) | Management & Governance | Q81–Q120 |

---

# PART 1 — AWS CONTAINERS

---

### 🟢 Q1. What is Amazon ECS and what problem does it solve?
**Answer:**
Amazon ECS (Elastic Container Service) is a fully managed container orchestration service. It lets you run, stop, and manage Docker containers on a cluster of EC2 instances or using AWS Fargate (serverless).

| Feature | ECS on EC2 | ECS on Fargate |
|---------|-----------|----------------|
| Server management | You manage EC2 | AWS manages infra |
| Pricing | Pay for EC2 instances | Pay per vCPU/memory per second |
| Customisation | Full control | Limited (no SSH) |
| Use case | Cost-optimised, GPU, compliance | Simplicity, variable load |

```bash
# Create ECS cluster
aws ecs create-cluster \
  --cluster-name my-production-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1 \
    capacityProvider=FARGATE_SPOT,weight=4 \
  --settings name=containerInsights,value=enabled \
  --tags key=Environment,value=production

# List clusters
aws ecs list-clusters --output table

# Describe cluster
aws ecs describe-clusters \
  --clusters my-production-cluster \
  --include ATTACHMENTS SETTINGS STATISTICS TAGS

# Delete cluster
aws ecs delete-cluster --cluster my-production-cluster
```

---

### 🟢 Q2. What is an ECS Task Definition?
```bash
# Task Definition: blueprint for your container(s)
# Defines: image, CPU, memory, ports, env vars, volumes, IAM role, logging

aws ecs register-task-definition --cli-input-json '{
  "family": "my-web-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn":      "arn:aws:iam::123456789:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
      "essential": true,
      "cpu": 256,
      "memory": 512,
      "memoryReservation": 256,
      "environment": [
        {"name": "ENVIRONMENT", "value": "production"},
        {"name": "PORT",        "value": "8080"}
      ],
      "secrets": [
        {"name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:db-password"},
        {"name": "API_KEY",     "valueFrom": "arn:aws:ssm:us-east-1:123456789:parameter/prod/api-key"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group":         "/ecs/my-web-app",
          "awslogs-region":        "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command":     ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval":    30,
        "timeout":     5,
        "retries":     3,
        "startPeriod": 60
      },
      "readonlyRootFilesystem": true,
      "user": "1000:1000"
    },
    {
      "name": "sidecar-logger",
      "image": "amazon/aws-for-fluent-bit:latest",
      "essential": false,
      "cpu": 64,
      "memory": 128,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/sidecar-logger",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ],
  "volumes": [
    {
      "name": "app-data",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-abc12345",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED"
      }
    }
  ]
}'

# List task definition revisions
aws ecs list-task-definitions --family-prefix my-web-app --output table

# Deregister old revision
aws ecs deregister-task-definition --task-definition my-web-app:1
```

---

### 🟢 Q3. What is an ECS Service?
```bash
# ECS Service: keeps N tasks running, integrates with ALB, enables rolling updates

aws ecs create-service \
  --cluster my-production-cluster \
  --service-name my-web-service \
  --task-definition my-web-app:2 \
  --desired-count 3 \
  --launch-type FARGATE \
  --platform-version LATEST \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-11111111,subnet-22222222],
    securityGroups=[sg-33333333],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789:targetgroup/my-tg/abc123,containerName=web,containerPort=8080" \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100,deploymentCircuitBreaker={enable=true,rollback=true}" \
  --deployment-controller "type=ECS" \
  --enable-execute-command \
  --enable-ecs-managed-tags \
  --propagate-tags SERVICE \
  --tags key=Environment,value=production

# Update service (rolling deployment)
aws ecs update-service \
  --cluster my-production-cluster \
  --service my-web-service \
  --task-definition my-web-app:3 \
  --force-new-deployment

# Scale service
aws ecs update-service \
  --cluster my-production-cluster \
  --service my-web-service \
  --desired-count 10

# View service events (for debugging)
aws ecs describe-services \
  --cluster my-production-cluster \
  --services my-web-service \
  --query "services[0].events[:5]"

# Stop all tasks in service
aws ecs update-service \
  --cluster my-production-cluster \
  --service my-web-service \
  --desired-count 0
```

---

### 🟡 Q4. How does ECS auto scaling work?
```bash
# ECS Service Auto Scaling uses Application Auto Scaling
# Policies: Target Tracking | Step Scaling | Scheduled

# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-production-cluster/my-web-service \
  --min-capacity 2 \
  --max-capacity 50

# Target Tracking policy (recommended) — keep CPU at 70%
aws application-autoscaling put-scaling-policy \
  --policy-name cpu-target-tracking \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-production-cluster/my-web-service \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'

# Target tracking on ALB request count per target
aws application-autoscaling put-scaling-policy \
  --policy-name alb-request-count \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-production-cluster/my-web-service \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 1000.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "app/my-alb/50dc6c495c0c9188/targetgroup/my-tg/73e2d6bc24d8a067"
    }
  }'

# Step scaling (scale-out aggressively on sudden spikes)
aws application-autoscaling put-scaling-policy \
  --policy-name step-scale-out \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-production-cluster/my-web-service \
  --policy-type StepScaling \
  --step-scaling-policy-configuration '{
    "AdjustmentType": "PercentChangeInCapacity",
    "StepAdjustments": [
      {"MetricIntervalLowerBound": 0,  "MetricIntervalUpperBound": 20, "ScalingAdjustment": 25},
      {"MetricIntervalLowerBound": 20, "MetricIntervalUpperBound": 40, "ScalingAdjustment": 50},
      {"MetricIntervalLowerBound": 40, "ScalingAdjustment": 100}
    ],
    "Cooldown": 60
  }'

# Scheduled scaling (known peak times)
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-production-cluster/my-web-service \
  --scheduled-action-name scale-out-business-hours \
  --schedule "cron(0 8 * * ? *)" \
  --scalable-target-action MinCapacity=10,MaxCapacity=50

aws application-autoscaling put-scheduled-action \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-production-cluster/my-web-service \
  --scheduled-action-name scale-in-evenings \
  --schedule "cron(0 20 * * ? *)" \
  --scalable-target-action MinCapacity=2,MaxCapacity=10
```

---

### 🟡 Q5. What are ECS deployment strategies?
```bash
# 1. Rolling update (default ECS)
aws ecs update-service \
  --cluster my-cluster --service my-service \
  --task-definition my-app:2 \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=50,deploymentCircuitBreaker={enable=true,rollback=true}"

# Circuit breaker: if 10% of tasks fail in 1 hour → auto rollback

# 2. Blue/Green deployment (CodeDeploy)
aws ecs create-service \
  --cluster my-cluster --service-name my-bg-service \
  --task-definition my-app:1 --desired-count 3 \
  --deployment-controller type=CODE_DEPLOY \
  --load-balancers "targetGroupArn=arn:aws:...blue-tg...,containerName=web,containerPort=8080"

# appspec.yaml for CodeDeploy B/G
cat > appspec.yaml << 'APPSPEC'
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: web
          ContainerPort: 8080
        PlatformVersion: LATEST
Hooks:
  - BeforeInstall: "arn:aws:lambda:us-east-1:123456789:function:pre-deploy-check"
  - AfterInstall:  "arn:aws:lambda:us-east-1:123456789:function:post-deploy-health"
  - AfterAllowTestTraffic: "arn:aws:lambda:us-east-1:123456789:function:run-integration-tests"
  - BeforeAllowTraffic: "arn:aws:lambda:us-east-1:123456789:function:final-checks"
  - AfterAllowTraffic: "arn:aws:lambda:us-east-1:123456789:function:cleanup"
APPSPEC

# 3. External deployment (full custom control)
aws ecs create-service \
  --deployment-controller type=EXTERNAL
# Use AWS SDK to manage task sets manually
```

---

### 🟡 Q6. What is ECS Exec (interactive shell into containers)?
```bash
# ECS Exec: run commands in running Fargate/EC2 containers — like kubectl exec
# Uses SSM Session Manager (no inbound ports needed)

# Prerequisites:
# 1. Task IAM role must have SSM permissions
# 2. Service must have --enable-execute-command
# 3. SSM Agent running in container (included in AL2-based images)

# IAM policy for task role:
aws iam create-policy --policy-name ecs-exec-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "ssmmessages:CreateControlChannel",
        "ssmmessages:CreateDataChannel",
        "ssmmessages:OpenControlChannel",
        "ssmmessages:OpenDataChannel"
      ],
      "Resource": "*"
    }]
  }'

# Enable execute command on service
aws ecs update-service \
  --cluster my-cluster --service my-service \
  --enable-execute-command

# Execute interactive shell
aws ecs execute-command \
  --cluster my-cluster \
  --task <task-arn> \
  --container web \
  --command "/bin/bash" \
  --interactive

# Execute a one-off command
aws ecs execute-command \
  --cluster my-cluster \
  --task <task-arn> \
  --container web \
  --command "cat /etc/hosts" \
  --interactive
```

---

### 🟡 Q7. What is Amazon ECR?
```bash
# ECR: fully managed Docker container registry (like Docker Hub, but AWS-native)
# Features: image scanning, lifecycle policies, replication, pull-through cache, OCI support

# Create private repository
aws ecr create-repository \
  --repository-name my-app \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS,kmsKey=arn:aws:kms:... \
  --image-tag-mutability IMMUTABLE \
  --tags Key=Environment,Value=production

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# Build, tag, push
docker build -t my-app:latest .
docker tag my-app:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.2.3

# Lifecycle policy (auto-delete old images to save cost)
aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 10 tagged releases",
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
        "description": "Delete untagged images after 7 days",
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

# Image scanning results
aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=v1.2.3 \
  --query "imageScanFindings.findings[?severity=='CRITICAL']"

# Pull-through cache (mirror Docker Hub without rate limiting)
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix dockerhub \
  --upstream-registry-url registry-1.docker.io

# ECR Public (share images publicly)
aws ecr-public create-repository \
  --repository-name my-public-app \
  --region us-east-1   # ECR Public only in us-east-1

# Cross-account pull (add registry policy)
aws ecr set-repository-policy \
  --repository-name my-app \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::987654321:root"},
      "Action": ["ecr:GetDownloadUrlForLayer","ecr:BatchGetImage","ecr:BatchCheckLayerAvailability"]
    }]
  }'
```

---

### 🟢 Q8. What is Amazon EKS?
```bash
# EKS: managed Kubernetes — AWS manages control plane (free), you pay for nodes
# EKS Auto Mode (2024): AWS manages nodes too (like GKE Autopilot)

# Create EKS cluster
aws eks create-cluster \
  --name my-eks-cluster \
  --kubernetes-version 1.31 \
  --role-arn arn:aws:iam::123456789:role/eks-cluster-role \
  --resources-vpc-config \
    subnetIds=subnet-11111111,subnet-22222222,subnet-33333333,\
    securityGroupIds=sg-44444444,\
    endpointPublicAccess=false,\
    endpointPrivateAccess=true \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}' \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:..."}}]' \
  --tags Environment=production

# Wait for cluster to be active
aws eks wait cluster-active --name my-eks-cluster

# Update kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name my-eks-cluster \
  --kubeconfig ~/.kube/config

# Add managed node group
aws eks create-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name general-purpose \
  --node-role arn:aws:iam::123456789:role/eks-node-role \
  --subnets subnet-11111111 subnet-22222222 \
  --instance-types t3.xlarge \
  --ami-type AL2023_x86_64_STANDARD \
  --capacity-type ON_DEMAND \
  --scaling-config minSize=2,maxSize=20,desiredSize=3 \
  --disk-size 50 \
  --labels Environment=production,tier=app \
  --taints key=dedicated,value=app,effect=NO_SCHEDULE \
  --update-config maxUnavailable=1

# Add Spot node group (cost savings)
aws eks create-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name spot-workers \
  --node-role arn:aws:iam::123456789:role/eks-node-role \
  --subnets subnet-11111111 subnet-22222222 \
  --instance-types t3.large t3.xlarge m5.large m5.xlarge \
  --capacity-type SPOT \
  --scaling-config minSize=0,maxSize=50,desiredSize=3 \
  --disk-size 50

# EKS Auto Mode (managed nodes — simplest)
aws eks create-cluster \
  --name my-automode-cluster \
  --kubernetes-version 1.31 \
  --role-arn arn:aws:iam::123456789:role/eks-cluster-role \
  --compute-config '{"enabled":true,"nodePools":["general-purpose","system"]}' \
  --kubernetes-network-config '{"elasticLoadBalancing":{"enabled":true}}' \
  --storage-config '{"blockStorage":{"enabled":true}}'

# Upgrade cluster version
aws eks update-cluster-version \
  --name my-eks-cluster \
  --kubernetes-version 1.32
```

---

### 🟡 Q9. What are EKS add-ons?
```bash
# Add-ons: AWS-managed components — auto-updated, security-patched

# List available add-ons
aws eks describe-addon-versions \
  --kubernetes-version 1.31 \
  --query "addons[].addonName" --output table

# Key add-ons:
# vpc-cni:               AWS VPC CNI — pod networking with VPC IPs
# coredns:               DNS for service discovery
# kube-proxy:            network rules on nodes
# aws-ebs-csi-driver:    EBS persistent volumes
# aws-efs-csi-driver:    EFS persistent volumes
# amazon-cloudwatch-observability: Container Insights + Prometheus
# aws-load-balancer-controller: ALB/NLB from Kubernetes Ingress/Service
# adot:                  AWS Distro for OpenTelemetry (metrics/traces)
# aws-guardduty-agent:   GuardDuty threat detection on nodes

# Install EBS CSI driver add-on
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name aws-ebs-csi-driver \
  --addon-version v1.35.0-eksbuild.1 \
  --service-account-role-arn arn:aws:iam::123456789:role/eks-ebs-csi-role \
  --resolve-conflicts OVERWRITE \
  --configuration-values '{"defaultStorageClass":{"enabled":true}}'

# Install ALB controller add-on
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name aws-load-balancer-controller \
  --service-account-role-arn arn:aws:iam::123456789:role/eks-alb-controller-role

# Update add-on to latest version
aws eks update-addon \
  --cluster-name my-eks-cluster \
  --addon-name vpc-cni \
  --addon-version v1.19.0-eksbuild.1 \
  --resolve-conflicts OVERWRITE
```

---

### 🟡 Q10. What is EKS Pod Identity (IRSA replacement)?
```bash
# EKS Pod Identity (2023): pods get AWS credentials via service account
# Replaces IRSA (IAM Roles for Service Accounts) — simpler, no OIDC configuration

# Enable Pod Identity add-on
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name eks-pod-identity-agent

# Create IAM role for pod
aws iam create-role \
  --role-name my-app-pod-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "pods.eks.amazonaws.com"},
      "Action": ["sts:AssumeRole","sts:TagSession"],
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "123456789",
          "aws:SourceArn": "arn:aws:eks:us-east-1:123456789:cluster/my-eks-cluster"
        }
      }
    }]
  }'

# Attach permissions to role
aws iam attach-role-policy \
  --role-name my-app-pod-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create Pod Identity association (link namespace+SA → IAM role)
aws eks create-pod-identity-association \
  --cluster-name my-eks-cluster \
  --namespace my-app \
  --service-account my-app-sa \
  --role-arn arn:aws:iam::123456789:role/my-app-pod-role
```

```yaml
# Kubernetes: service account (no annotations needed with Pod Identity)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 3
  template:
    spec:
      serviceAccountName: my-app-sa   # automatically gets AWS credentials
      containers:
      - name: app
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
        # boto3 / AWS SDK automatically uses pod credentials
```

---

### 🟡 Q11. What is the EKS Cluster Autoscaler vs Karpenter?
```bash
# Cluster Autoscaler: scale node groups based on pending pods
# Karpenter (recommended): provision RIGHT-SIZED nodes directly — faster, more efficient

# ── KARPENTER ──────────────────────────────────────────────────────
# Install Karpenter via Helm
helm repo add karpenter https://charts.karpenter.sh/
helm install karpenter karpenter/karpenter \
  --namespace kube-system \
  --set "settings.clusterName=my-eks-cluster" \
  --set "settings.interruptionQueue=my-karpenter-queue" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi
```

```yaml
# Karpenter NodePool (replaces Cluster Autoscaler node groups)
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general
spec:
  template:
    spec:
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: [amd64, arm64]
      - key: karpenter.sh/capacity-type
        operator: In
        values: [spot, on-demand]     # prefer Spot, fall back to On-Demand
      - key: karpenter.k8s.aws/instance-category
        operator: In
        values: [c, m, r]            # compute, memory, general
      - key: karpenter.k8s.aws/instance-size
        operator: NotIn
        values: [nano, micro, small]  # exclude tiny instances
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000                          # max 1000 vCPUs in this pool
    memory: 2000Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m               # consolidate underutilised nodes after 1 min
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: eks-node-role
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-eks-cluster
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: my-eks-cluster
  blockDeviceMappings:
  - deviceName: /dev/xvda
    ebs:
      volumeSize: 50Gi
      volumeType: gp3
      encrypted: true
```

---

### 🟡 Q12. What is AWS Fargate?
```bash
# Fargate: serverless compute for containers — no EC2 to manage
# Works with: ECS and EKS
# Pricing: per vCPU/hour + per GB memory/hour

# Fargate task sizes (vCPU : Memory combinations):
# 0.25 vCPU: 0.5, 1, 2 GB
# 0.5 vCPU:  1, 2, 3, 4 GB
# 1 vCPU:    2, 3, 4, 5, 6, 7, 8 GB
# 2 vCPU:    4–16 GB (1 GB increments)
# 4 vCPU:    8–30 GB (1 GB increments)
# 8 vCPU:    16–60 GB (4 GB increments)
# 16 vCPU:   32–120 GB (8 GB increments)

# EKS Fargate profile (run specific pods on Fargate)
aws eks create-fargate-profile \
  --cluster-name my-eks-cluster \
  --fargate-profile-name app-profile \
  --pod-execution-role-arn arn:aws:iam::123456789:role/eks-fargate-role \
  --subnets subnet-11111111 subnet-22222222 \
  --selectors '[
    {"namespace": "production", "labels": {"fargate": "true"}},
    {"namespace": "kube-system"}
  ]'

# Fargate Spot (for interruption-tolerant workloads — up to 70% savings)
# In ECS task definition:
# --capacity-provider-strategy capacityProvider=FARGATE_SPOT,weight=4,base=0

# Fargate benefits:
# ✅ No node management, patching, or scaling of EC2
# ✅ Per-task isolation (separate microVM per task)
# ✅ No Kubernetes node scheduling concerns
# ❌ No privileged containers
# ❌ No DaemonSets on Fargate
# ❌ No GPU support
# ❌ Slightly higher cost vs equivalently-sized EC2
```

---

### 🟡 Q13. What is Amazon ECS Anywhere and EKS Anywhere?
```bash
# ECS Anywhere: run ECS tasks on your on-prem or other-cloud servers
# EKS Anywhere: run EKS Kubernetes clusters on your own hardware

# ── ECS Anywhere ─────────────────────────────────────────────────
# Register external instance with ECS cluster
aws ecs register-external-instance \
  --cluster my-production-cluster

# Get SSM activation for on-prem server
aws ecs create-service \
  --cluster my-production-cluster \
  --launch-type EXTERNAL \   # key difference
  --service-name on-prem-service \
  --task-definition my-on-prem-app:1 \
  --desired-count 2

# ── EKS Anywhere ─────────────────────────────────────────────────
# Supports: bare metal, VMware vSphere, Nutanix, Snow Family, CloudStack

# Generate cluster config
eksctl anywhere generate clusterconfig my-cluster \
  --provider vsphere > my-cluster.yaml

# Create cluster
eksctl anywhere create cluster -f my-cluster.yaml

# EKS Distro (EKS-D): same Kubernetes distribution AWS uses in EKS
# Allows self-managed Kubernetes with same patches and security updates as EKS
```

---

### 🟡 Q14. What is AWS App Runner?
```bash
# App Runner: simplest way to run containerised web apps and APIs
# Fully managed: build, deploy, scale, load balancing, TLS all automatic

# Create service from ECR image
aws apprunner create-service \
  --service-name my-web-service \
  --source-configuration '{
    "ImageRepository": {
      "ImageIdentifier": "123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": {
        "Port": "8080",
        "RuntimeEnvironmentVariables": {
          "ENVIRONMENT": "production"
        },
        "RuntimeEnvironmentSecrets": {
          "DB_PASSWORD": "arn:aws:secretsmanager:us-east-1:123456789:secret:db-password"
        }
      }
    },
    "AutoDeploymentsEnabled": true
  }' \
  --instance-configuration '{
    "Cpu": "1 vCPU",
    "Memory": "2 GB",
    "InstanceRoleArn": "arn:aws:iam::123456789:role/app-runner-role"
  }' \
  --health-check-configuration '{
    "Protocol": "HTTP",
    "Path": "/health",
    "Interval": 10,
    "Timeout": 5,
    "HealthyThreshold": 1,
    "UnhealthyThreshold": 5
  }' \
  --auto-scaling-configuration-arn arn:aws:apprunner:...

# Create service from GitHub (with auto-build)
aws apprunner create-service \
  --service-name my-github-service \
  --source-configuration '{
    "CodeRepository": {
      "RepositoryUrl": "https://github.com/myOrg/myApp",
      "SourceCodeVersion": {"Type": "BRANCH", "Value": "main"},
      "CodeConfiguration": {
        "ConfigurationSource": "API",
        "CodeConfigurationValues": {
          "Runtime": "PYTHON_3",
          "BuildCommand": "pip install -r requirements.txt",
          "StartCommand": "python app.py",
          "Port": "8080"
        }
      }
    },
    "AutoDeploymentsEnabled": true,
    "AuthenticationConfiguration": {
      "ConnectionArn": "arn:aws:apprunner:us-east-1:123456789:connection/github/abc"
    }
  }'

# VNet association (access private VPC resources)
aws apprunner create-vpc-connector \
  --vpc-connector-name my-vpc-connector \
  --subnets subnet-11111111 subnet-22222222 \
  --security-groups sg-33333333

aws apprunner update-service \
  --service-arn arn:aws:apprunner:... \
  --network-configuration '{
    "EgressConfiguration": {
      "EgressType": "VPC",
      "VpcConnectorArn": "arn:aws:apprunner:..."
    }
  }'

# App Runner vs ECS Fargate vs Lambda:
# App Runner:  simplest, opinionated, auto-scale, great for web apps
# ECS Fargate: more control, VPC from start, long-running, complex routing
# Lambda:      true serverless, per-request pricing, max 15 min
```

---

### 🔴 Q15. What are EKS best practices for production?
```bash
# 1. Multi-AZ node groups (resilience)
aws eks create-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name prod-nodes \
  --subnets subnet-az1 subnet-az2 subnet-az3 \   # one per AZ
  --scaling-config minSize=3,maxSize=30,desiredSize=3

# 2. Private cluster endpoint (security)
aws eks update-cluster-config \
  --name my-eks-cluster \
  --resources-vpc-config \
    endpointPublicAccess=false,endpointPrivateAccess=true

# 3. Enable audit logging
aws eks update-cluster-config \
  --name my-eks-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# 4. Encrypt secrets with KMS
aws eks associate-encryption-config \
  --cluster-name my-eks-cluster \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:..."}}]'

# 5. Enable GuardDuty EKS protection
aws guardduty update-detector \
  --detector-id <detector-id> \
  --data-sources '{"Kubernetes":{"AuditLogs":{"Enable":true}},"MalwareProtection":{"ScanEc2InstanceWithFindings":{"EbsVolumes":true}}}'

# 6. Network policies (restrict pod-to-pod traffic)
kubectl apply -f - <<'NETPOL'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp
    ports:
    - port: 5432
NETPOL

# 7. Pod Security Admission (replace PSP)
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# 8. OPA/Gatekeeper policies
kubectl apply -f - <<'CONSTRAINT'
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds: [{apiGroups: ["apps"], kinds: ["Deployment"]}]
  parameters:
    labels: ["team", "environment"]
CONSTRAINT
```

---

### 🟡 Q16. What are ECS vs EKS — when to choose each?
| Factor | ECS | EKS |
|--------|-----|-----|
| Complexity | Low — AWS-native | High — Kubernetes |
| Kubernetes compatibility | ❌ | ✅ Full |
| Multi-cloud portability | ❌ | ✅ |
| Learning curve | Low | High |
| AWS integration | Excellent | Very good |
| Cost | Lower management overhead | Cluster ~$0.10/hr + nodes |
| Ecosystem | AWS-centric | Huge (Helm, Flux, Istio, Keda) |
| Best for | Pure AWS, simple, fast | Multi-cloud, K8s ecosystem, complex |

---

### 🟡 Q17. What is AWS Copilot CLI?
```bash
# Copilot: developer-friendly CLI to build, release, and operate ECS apps
# Abstracts away: task definitions, services, VPC, ALB, CloudFormation

# Install
brew install aws/tap/copilot-cli   # macOS
# or: curl -Lo copilot ... && chmod +x copilot && sudo mv copilot /usr/local/bin/

# Initialize app
copilot app init my-app

# Create environment
copilot env init --name production --profile prod-profile --default-config

# Deploy environment infra
copilot env deploy --name production

# Create and deploy a service
copilot svc init \
  --name api \
  --svc-type "Load Balanced Web Service" \
  --image 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest \
  --port 8080

copilot svc deploy --name api --env production

# Create a worker service (processes SQS messages)
copilot svc init \
  --name order-processor \
  --svc-type "Worker Service"
# copilot automatically creates SQS queue + subscribes to SNS

# Check status
copilot svc status --name api --env production
copilot svc logs --name api --env production --follow

# Create scheduled job
copilot job init --name db-cleanup --job-type "Scheduled Job" \
  --schedule "@midnight"
copilot job deploy --name db-cleanup --env production

# Copilot manifest (copilot/api/manifest.yml)
cat copilot/api/manifest.yml
```

```yaml
# copilot/api/manifest.yml
name: api
type: Load Balanced Web Service

image:
  build: Dockerfile
  port: 8080

http:
  path: '/'
  healthcheck:
    path: '/health'
    interval: 10s
    retries: 2
    timeout: 5s
    start_period: 60s

cpu: 512
memory: 1024
count:
  range: 2-50
  cooldown:
    in: 120s
    out: 30s
  cpu_percentage: 70
  requests: 1000

env_file: .env

secrets:
  DB_PASSWORD: /my-app/production/db-password

variables:
  LOG_LEVEL: INFO

environments:
  production:
    count:
      range: 5-100
      cpu_percentage: 75
```

---

### 🟡 Q18. What are container networking modes in ECS?
```bash
# Network modes:
# awsvpc:  each task gets its own ENI + private IP — REQUIRED for Fargate
# bridge:  Docker bridge network, port mapping (legacy, EC2 only)
# host:    container shares EC2 host network namespace (EC2 only)
# none:    no networking

# awsvpc mode (recommended):
# - Task gets dedicated ENI with VPC IP
# - Security groups applied per-task (not per-host)
# - Enables service discovery via Cloud Map

# awsvpc considerations:
# - ENI limit per EC2 instance (check aws ec2 describe-instance-types)
# - Use ENI Trunking for higher density on EC2
aws ecs put-account-setting \
  --name awsvpcTrunking \
  --value enabled

# Service Discovery with Cloud Map
aws servicediscovery create-private-dns-namespace \
  --name my-app.local \
  --vpc vpc-12345678

aws servicediscovery create-service \
  --name api \
  --dns-config "NamespaceId=ns-abc123,RoutingPolicy=MULTIVALUE,DnsRecords=[{Type=A,TTL=10}]" \
  --health-check-custom-config FailureThreshold=1

# Then in ECS service:
# --service-registries "registryArn=arn:aws:servicediscovery:...,port=8080"
# Other services can call: api.my-app.local:8080
```

---

### 🟡 Q19. What is ECS Capacity Providers?
```bash
# Capacity Providers: manage the capacity backend for ECS tasks

# Types:
# FARGATE:       managed serverless (pay per task)
# FARGATE_SPOT:  managed serverless with interruptions (70% savings)
# Auto Scaling Group (ASG): your EC2 instances with automatic scaling

# Create ASG Capacity Provider
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name ecs-asg \
  --launch-template "LaunchTemplateId=lt-abc123,Version=\$Latest" \
  --min-size 2 --max-size 20 --desired-capacity 3 \
  --vpc-zone-identifier "subnet-11111111,subnet-22222222" \
  --capacity-rebalance

aws ecs create-capacity-provider \
  --name my-ec2-capacity-provider \
  --auto-scaling-group-provider '{
    "autoScalingGroupArn": "arn:aws:autoscaling:us-east-1:123456789:autoScalingGroup:...",
    "managedScaling": {
      "status": "ENABLED",
      "targetCapacity": 80,
      "minimumScalingStepSize": 1,
      "maximumScalingStepSize": 100
    },
    "managedTerminationProtection": "ENABLED",
    "managedDraining": "ENABLED"
  }'

# Update cluster with capacity providers
aws ecs put-cluster-capacity-providers \
  --cluster my-cluster \
  --capacity-providers FARGATE FARGATE_SPOT my-ec2-capacity-provider \
  --default-capacity-provider-strategy \
    capacityProvider=my-ec2-capacity-provider,weight=3,base=2 \
    capacityProvider=FARGATE_SPOT,weight=1

# Mixed strategy: base 2 tasks on EC2, then FARGATE_SPOT for burst
# Saves cost while maintaining reliability
```

---

### 🟢 Q20. What are the IAM roles needed for ECS?
```bash
# Three distinct IAM roles for ECS:

# 1. ECS Task Execution Role (ecsTaskExecutionRole)
#    Used BY ECS AGENT to pull images, write logs, get secrets
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Additional: allow getting secrets
aws iam put-role-policy --role-name ecsTaskExecutionRole \
  --policy-name secrets-access \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["secretsmanager:GetSecretValue","ssm:GetParameters","kms:Decrypt"],"Resource":"*"}]}'

# 2. ECS Task Role
#    Used BY YOUR APPLICATION to access AWS services
aws iam create-role \
  --role-name ecsTaskRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy \
  --role-name ecsTaskRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 3. ECS Service-Linked Role (auto-created)
#    AWS creates automatically: AWSServiceRoleForECS
#    Used BY ECS SERVICE to register targets with ALB, manage ENIs

# Summary:
# executionRoleArn → pull image + logs + secrets (ECS agent)
# taskRoleArn      → your app calls S3, DynamoDB, SQS, etc. (your code)
```


---

# PART 2 — AWS DEVELOPER TOOLS

---

### 🟢 Q21. What is AWS CodeCommit?
```bash
# CodeCommit: fully managed private Git repositories
# NOTE: AWS announced CodeCommit will no longer onboard new customers (July 2024)
# Existing users continue; new projects should use GitHub/GitLab/Bitbucket

# Create repository
aws codecommit create-repository \
  --repository-name my-app \
  --repository-description "Main application repository" \
  --tags Environment=production

# Clone repository
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-app
# Or with SSH:
git clone ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-app

# Configure Git credentials helper
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true

# Create branch protection (approval rule)
aws codecommit create-approval-rule-template \
  --approval-rule-template-name require-two-approvers \
  --approval-rule-template-content '{
    "Version": "2018-11-08",
    "Statements": [{
      "Type": "Approvers",
      "NumberOfApprovalsNeeded": 2,
      "ApprovalPoolMembers": ["arn:aws:iam::123456789:group/senior-devs"]
    }]
  }'

aws codecommit associate-approval-rule-template-with-repository \
  --approval-rule-template-name require-two-approvers \
  --repository-name my-app

# Create pull request
aws codecommit create-pull-request \
  --title "Add payment feature" \
  --description "Adds Apple Pay and Google Pay support" \
  --targets "repositoryName=my-app,sourceReference=feature/payment,destinationReference=main"

# Trigger notifications on events
aws codecommit put-repository-triggers \
  --repository-name my-app \
  --triggers '[{
    "name": "AllEvents",
    "destinationArn": "arn:aws:sns:us-east-1:123456789:codecommit-notifications",
    "events": ["all"]
  }]'
```

---

### 🟢 Q22. What is AWS CodeBuild?
```bash
# CodeBuild: fully managed build service — compiles, tests, packages code
# No servers to manage; pay per build minute

# Create build project
aws codebuild create-project \
  --name my-app-build \
  --source '{
    "type": "GITHUB",
    "location": "https://github.com/myOrg/my-app",
    "buildspec": "buildspec.yml",
    "reportBuildStatus": true,
    "gitCloneDepth": 1
  }' \
  --artifacts '{
    "type": "S3",
    "location": "my-artifacts-bucket",
    "name": "build-output",
    "packaging": "ZIP"
  }' \
  --environment '{
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/standard:7.0",
    "computeType": "BUILD_GENERAL1_MEDIUM",
    "privilegedMode": true,
    "environmentVariables": [
      {"name": "AWS_ACCOUNT_ID",  "value": "123456789", "type": "PLAINTEXT"},
      {"name": "IMAGE_REPO_NAME", "value": "my-app",    "type": "PLAINTEXT"},
      {"name": "DB_PASSWORD",     "value": "/prod/db-password", "type": "PARAMETER_STORE"}
    ]
  }' \
  --service-role arn:aws:iam::123456789:role/codebuild-role \
  --logs-config '{
    "cloudWatchLogs": {"status": "ENABLED", "groupName": "/codebuild/my-app-build"},
    "s3Logs": {"status": "ENABLED", "location": "my-logs-bucket/build-log"}
  }' \
  --vpc-config '{
    "vpcId": "vpc-12345678",
    "subnets": ["subnet-11111111"],
    "securityGroupIds": ["sg-22222222"]
  }' \
  --cache '{"type": "LOCAL", "modes": ["LOCAL_DOCKER_LAYER_CACHE","LOCAL_SOURCE_CACHE","LOCAL_CUSTOM_CACHE"]}'

# buildspec.yml — complete example
cat > buildspec.yml << 'BUILDSPEC'
version: 0.2

env:
  variables:
    PYTHON_VERSION: "3.12"
  parameter-store:
    DB_PASSWORD: "/prod/db-password"
  secrets-manager:
    API_KEY: "prod/api-key:apiKey"

phases:
  install:
    runtime-versions:
      python: 3.12
    commands:
      - pip install --upgrade pip
      - pip install -r requirements.txt
      - pip install pytest pytest-cov safety bandit

  pre_build:
    commands:
      - echo "Running security scans..."
      - safety check --full-report
      - bandit -r src/ -f json -o bandit-report.json || true
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - IMAGE_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}

  build:
    commands:
      - echo "Running tests..."
      - pytest tests/ --junitxml=test-results.xml --cov=src --cov-report=xml
      - echo "Building Docker image..."
      - docker build -t $IMAGE_URI:$IMAGE_TAG
          --build-arg BUILD_ID=$CODEBUILD_BUILD_ID
          --build-arg COMMIT=$CODEBUILD_RESOLVED_SOURCE_VERSION
          --cache-from $IMAGE_URI:latest .
      - docker tag $IMAGE_URI:$IMAGE_TAG $IMAGE_URI:latest

  post_build:
    commands:
      - echo "Pushing Docker images..."
      - docker push $IMAGE_URI:$IMAGE_TAG
      - docker push $IMAGE_URI:latest
      - echo "Generating image definitions file..."
      - printf '[{"name":"web","imageUri":"%s"}]' $IMAGE_URI:$IMAGE_TAG > imagedefinitions.json

reports:
  test-results:
    files: [test-results.xml]
    file-format: JUNITXML
  coverage-report:
    files: [coverage.xml]
    file-format: COBERTURAXML
  security-reports:
    files: [bandit-report.json]

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yml
    - taskdef.json

cache:
  paths:
    - '/root/.cache/pip/**/*'
    - '/var/lib/docker/**/*'
BUILDSPEC

# Start a build
aws codebuild start-build \
  --project-name my-app-build \
  --environment-variables-override \
    name=IMAGE_TAG,value=v1.2.3,type=PLAINTEXT

# Check build status
aws codebuild batch-get-builds \
  --ids "my-app-build:abc123" \
  --query "builds[0].{Status:buildStatus,Phase:currentPhase,Duration:buildComplete}"
```

---

### 🟢 Q23. What is AWS CodeDeploy?
```bash
# CodeDeploy: automates application deployments to EC2, Lambda, ECS, on-prem
# Supports: rolling, blue/green, canary, linear deployment strategies

# Create application
aws codedeploy create-application \
  --application-name my-app \
  --compute-platform ECS    # Server | Lambda | ECS

# Create deployment group (ECS)
aws codedeploy create-deployment-group \
  --application-name my-app \
  --deployment-group-name production \
  --service-role-arn arn:aws:iam::123456789:role/codedeploy-role \
  --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
  --ecs-services '[{"clusterName":"my-cluster","serviceName":"my-service"}]' \
  --load-balancer-info '{
    "targetGroupPairInfoList": [{
      "targetGroups": [
        {"name": "my-tg-blue"},
        {"name": "my-tg-green"}
      ],
      "prodTrafficRoute": {"listenerArns": ["arn:aws:elasticloadbalancing:...prod-listener..."]},
      "testTrafficRoute": {"listenerArns": ["arn:aws:elasticloadbalancing:...test-listener..."]}
    }]
  }' \
  --blue-green-deployment-configuration '{
    "terminateBlueInstancesOnDeploymentSuccess": {
      "action": "TERMINATE",
      "terminationWaitTimeInMinutes": 60
    },
    "deploymentReadyOption": {
      "actionOnTimeout": "STOP_DEPLOYMENT",
      "waitTimeInMinutes": 30
    }
  }' \
  --auto-rollback-configuration '{
    "enabled": true,
    "events": ["DEPLOYMENT_FAILURE","DEPLOYMENT_STOP_ON_ALARM","DEPLOYMENT_STOP_ON_REQUEST"]
  }' \
  --alarm-configuration '{
    "enabled": true,
    "alarms": [{"name": "high-error-rate"}, {"name": "high-latency"}]
  }'

# EC2 deployment group
aws codedeploy create-deployment-group \
  --application-name my-ec2-app \
  --deployment-group-name production \
  --service-role-arn arn:aws:iam::123456789:role/codedeploy-role \
  --deployment-config-name CodeDeployDefault.HalfAtATime \
  --ec2-tag-set '{
    "ec2TagSetList": [[
      {"Key": "Environment", "Value": "production", "Type": "KEY_AND_VALUE"},
      {"Key": "App",         "Value": "myapp",       "Type": "KEY_AND_VALUE"}
    ]]
  }' \
  --auto-scaling-groups my-ec2-asg \
  --deployment-style '{
    "deploymentType": "BLUE_GREEN",
    "deploymentOption": "WITH_TRAFFIC_CONTROL"
  }'

# Lambda deployment (canary)
aws codedeploy create-deployment-group \
  --application-name my-lambda-app \
  --deployment-group-name production \
  --service-role-arn arn:aws:iam::123456789:role/codedeploy-role \
  --deployment-config-name CodeDeployDefault.LambdaCanary10Percent5Minutes

# Built-in deployment configs:
# AllAtOnce:                    all at once (fastest, no rollback granularity)
# HalfAtATime:                  50% then 50%
# OneAtATime:                   one instance at a time (safest)
# ECSAllAtOnce:                 ECS full cutover
# LambdaLinear10PercentEvery1Minute:   10% more each minute
# LambdaCanary10Percent5Minutes:        10% for 5 min, then 90%
```

---

### 🟢 Q24. What is AWS CodePipeline?
```bash
# CodePipeline: fully managed CI/CD orchestration
# Chains together: Source → Build → Test → Deploy stages

# Create pipeline
aws codepipeline create-pipeline --pipeline '{
  "name": "my-app-pipeline",
  "roleArn": "arn:aws:iam::123456789:role/codepipeline-role",
  "artifactStore": {
    "type": "S3",
    "location": "my-pipeline-artifacts-bucket"
  },
  "stages": [
    {
      "name": "Source",
      "actions": [{
        "name": "Source",
        "actionTypeId": {
          "category": "Source",
          "owner": "AWS",
          "provider": "CodeCommit",
          "version": "1"
        },
        "configuration": {
          "RepositoryName": "my-app",
          "BranchName": "main",
          "DetectChanges": "true",
          "OutputArtifactFormat": "CODEBUILD_CLONE_REF"
        },
        "outputArtifacts": [{"name": "SourceOutput"}]
      }]
    },
    {
      "name": "Build",
      "actions": [{
        "name": "Build",
        "actionTypeId": {
          "category": "Build",
          "owner": "AWS",
          "provider": "CodeBuild",
          "version": "1"
        },
        "configuration": {
          "ProjectName": "my-app-build"
        },
        "inputArtifacts":  [{"name": "SourceOutput"}],
        "outputArtifacts": [{"name": "BuildOutput"}]
      }]
    },
    {
      "name": "Test",
      "actions": [
        {
          "name": "IntegrationTests",
          "actionTypeId": {"category": "Build", "owner": "AWS", "provider": "CodeBuild", "version": "1"},
          "configuration": {"ProjectName": "my-app-integration-tests"},
          "inputArtifacts": [{"name": "BuildOutput"}],
          "runOrder": 1
        },
        {
          "name": "SecurityScan",
          "actionTypeId": {"category": "Build", "owner": "AWS", "provider": "CodeBuild", "version": "1"},
          "configuration": {"ProjectName": "my-app-security-scan"},
          "inputArtifacts": [{"name": "SourceOutput"}],
          "runOrder": 1
        }
      ]
    },
    {
      "name": "ApproveProduction",
      "actions": [{
        "name": "ManualApproval",
        "actionTypeId": {
          "category": "Approval",
          "owner": "AWS",
          "provider": "Manual",
          "version": "1"
        },
        "configuration": {
          "NotificationArn": "arn:aws:sns:us-east-1:123456789:pipeline-approvals",
          "CustomData": "Production deployment requires CTO approval. Review: https://staging.myapp.com",
          "ExternalEntityLink": "https://staging.myapp.com/test-report"
        }
      }]
    },
    {
      "name": "DeployProduction",
      "actions": [{
        "name": "Deploy",
        "actionTypeId": {
          "category": "Deploy",
          "owner": "AWS",
          "provider": "CodeDeployToECS",
          "version": "1"
        },
        "configuration": {
          "ApplicationName":     "my-app",
          "DeploymentGroupName": "production",
          "TaskDefinitionTemplateArtifact": "BuildOutput",
          "TaskDefinitionTemplatePath": "taskdef.json",
          "AppSpecTemplateArtifact": "BuildOutput",
          "AppSpecTemplatePath": "appspec.yml",
          "Image1ArtifactName": "BuildOutput",
          "Image1ContainerName": "IMAGE1_NAME"
        },
        "inputArtifacts": [{"name": "BuildOutput"}]
      }]
    }
  ]
}'

# Trigger pipeline manually
aws codepipeline start-pipeline-execution --name my-app-pipeline

# Check pipeline status
aws codepipeline get-pipeline-state --name my-app-pipeline \
  --query "stageStates[].{Stage:stageName,Status:latestExecution.status}"

# Approve/reject manual approval
TOKEN=$(aws codepipeline get-pipeline-state --name my-app-pipeline \
  --query "stageStates[?stageName=='ApproveProduction'].actionStates[0].latestExecution.token" \
  --output text)

aws codepipeline put-approval-result \
  --pipeline-name my-app-pipeline \
  --stage-name ApproveProduction \
  --action-name ManualApproval \
  --result Summary="Approved for production",Status=Approved \
  --token $TOKEN

# V2 pipelines (2024) — GitHub triggers, variables between stages
aws codepipeline create-pipeline --pipeline '{
  "name": "my-v2-pipeline",
  "pipelineType": "V2",
  "triggers": [{
    "providerType": "CodeStarSourceConnection",
    "gitConfiguration": {
      "sourceActionName": "Source",
      "push": [{"branches": {"includes": ["main"]}, "filePaths": {"includes": ["src/*"]}}]
    }
  }],
  "variables": [
    {"name": "ENVIRONMENT", "defaultValue": "production"},
    {"name": "IMAGE_TAG"}
  ]
}'
```

---

### 🟡 Q25. What is AWS CodeStar Connections?
```bash
# CodeStar Connections: link AWS to third-party repos (GitHub, GitLab, Bitbucket)

# Create connection (then complete in browser)
aws codestar-connections create-connection \
  --provider-type GitHub \
  --connection-name github-my-org

# Complete in browser:
# AWS Console → Settings → Connections → Pending → Update pending connection

# List connections
aws codestar-connections list-connections --output table

# Use connection in CodePipeline source action:
# "provider": "CodeStarSourceConnection"
# "configuration": {"ConnectionArn": "arn:aws:codestar-connections:..."}

# GitHub Actions integration:
# Add AWS credentials to GitHub Secrets → trigger CodePipeline from GHA
```

---

### 🟡 Q26. What is AWS CDK (Cloud Development Kit)?
```bash
# CDK: define AWS infrastructure using real programming languages
# Languages: TypeScript, JavaScript, Python, Java, C#, Go
# Compiles to CloudFormation templates

# Install
npm install -g aws-cdk

# Initialise project
mkdir my-infra && cd my-infra
cdk init app --language typescript

# Deploy
cdk bootstrap aws://123456789/us-east-1   # one-time per account/region
cdk deploy MyStack
cdk diff    # show changes before deploying
cdk destroy MyStack

# Python CDK example — ECS Fargate service
```

```python
from aws_cdk import (
    Stack, App, Duration,
    aws_ec2 as ec2,
    aws_ecs as ecs,
    aws_ecs_patterns as ecs_patterns,
    aws_ecr as ecr,
    aws_iam as iam,
    aws_logs as logs,
    aws_secretsmanager as sm,
    aws_elasticloadbalancingv2 as elbv2
)
from constructs import Construct

class MyAppStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)

        # VPC
        vpc = ec2.Vpc(self, "MyVpc",
            max_azs=3,
            nat_gateways=1,
            subnet_configuration=[
                ec2.SubnetConfiguration(name="Public",  subnet_type=ec2.SubnetType.PUBLIC,  cidr_mask=24),
                ec2.SubnetConfiguration(name="Private", subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS, cidr_mask=24),
            ]
        )

        # ECS Cluster
        cluster = ecs.Cluster(self, "MyCluster",
            vpc=vpc,
            container_insights=True,
            enable_fargate_capacity_providers=True
        )

        # Secret
        db_secret = sm.Secret.from_secret_name_v2(self, "DbSecret", "prod/db-password")

        # ALB + Fargate Service pattern (opinionated, handles ALB + SG + Task Def)
        fargate_service = ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "MyService",
            cluster=cluster,
            cpu=512,
            memory_limit_mib=1024,
            desired_count=3,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_ecr_repository(
                    ecr.Repository.from_repository_name(self, "MyRepo", "my-app"),
                    tag="latest"
                ),
                container_port=8080,
                environment={"ENVIRONMENT": "production"},
                secrets={"DB_PASSWORD": ecs.Secret.from_secrets_manager(db_secret)},
                log_driver=ecs.LogDrivers.aws_logs(
                    stream_prefix="ecs",
                    log_retention=logs.RetentionDays.ONE_MONTH
                )
            ),
            public_load_balancer=True,
            redirect_http=True,
            protocol=elbv2.ApplicationProtocol.HTTPS,
            certificate=elbv2.Certificate.from_certificate_arn(self, "Cert", "arn:aws:acm:...")
        )

        # Auto Scaling
        scaling = fargate_service.service.auto_scale_task_count(min_capacity=2, max_capacity=50)
        scaling.scale_on_cpu_utilization("CpuScaling", target_utilization_percent=70,
                                         scale_in_cooldown=Duration.minutes(5),
                                         scale_out_cooldown=Duration.seconds(60))
        scaling.scale_on_request_count("RequestScaling",
                                        requests_per_target=1000,
                                        target_group=fargate_service.target_group)

app = App()
MyAppStack(app, "MyAppStack", env={"account": "123456789", "region": "us-east-1"})
app.synth()
```

---

### 🟡 Q27. What is AWS CloudFormation?
```bash
# CloudFormation: Infrastructure as Code using JSON or YAML templates
# Manages: resources as stacks, handles dependencies, enables rollback

# Create stack
aws cloudformation create-stack \
  --stack-name my-app-stack \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=Environment,ParameterValue=production \
    ParameterKey=InstanceType,ParameterValue=t3.medium \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --tags Key=Project,Value=my-app

# Wait for completion
aws cloudformation wait stack-create-complete --stack-name my-app-stack

# Update stack
aws cloudformation update-stack \
  --stack-name my-app-stack \
  --template-body file://template-v2.yaml \
  --parameters ParameterKey=Environment,ParameterValue=production

# Change sets (preview changes before applying)
aws cloudformation create-change-set \
  --stack-name my-app-stack \
  --change-set-name my-changes \
  --template-body file://template-v2.yaml \
  --parameters ParameterKey=Environment,ParameterValue=production

aws cloudformation describe-change-set \
  --stack-name my-app-stack \
  --change-set-name my-changes \
  --query "Changes[].{Action:ResourceChange.Action,Resource:ResourceChange.LogicalResourceId,Type:ResourceChange.ResourceType}"

aws cloudformation execute-change-set \
  --stack-name my-app-stack \
  --change-set-name my-changes

# CloudFormation template structure
cat > template.yaml << 'CFN'
AWSTemplateFormatVersion: '2010-09-09'
Description: ECS Fargate Application Stack

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, production]
    Default: dev
  ImageTag:
    Type: String
    Default: latest

Mappings:
  EnvironmentConfig:
    dev:        {Cpu: 256,  Memory: 512,  DesiredCount: 1}
    staging:    {Cpu: 512,  Memory: 1024, DesiredCount: 2}
    production: {Cpu: 1024, Memory: 2048, DesiredCount: 5}

Conditions:
  IsProduction: !Equals [!Ref Environment, production]

Resources:
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub '${AWS::StackName}-cluster'
      ClusterSettings:
        - Name: containerInsights
          Value: !If [IsProduction, enabled, disabled]

  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Sub '${AWS::StackName}-task'
      NetworkMode: awsvpc
      RequiresCompatibilities: [FARGATE]
      Cpu: !FindInMap [EnvironmentConfig, !Ref Environment, Cpu]
      Memory: !FindInMap [EnvironmentConfig, !Ref Environment, Memory]
      ExecutionRoleArn: !GetAtt TaskExecutionRole.Arn
      ContainerDefinitions:
        - Name: web
          Image: !Sub '${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/my-app:${ImageTag}'
          PortMappings:
            - ContainerPort: 8080
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref LogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: ecs

  LogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/ecs/${AWS::StackName}'
      RetentionInDays: !If [IsProduction, 90, 7]

  TaskExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: {Service: ecs-tasks.amazonaws.com}
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

Outputs:
  ClusterName:
    Value: !Ref ECSCluster
    Export:
      Name: !Sub '${AWS::StackName}-cluster'
  TaskDefinitionArn:
    Value: !Ref TaskDefinition
CFN
```

---

### 🟡 Q28. What is AWS SAM (Serverless Application Model)?
```bash
# SAM: CloudFormation extension for serverless apps (Lambda, API GW, DynamoDB)
# Simplifies: Lambda deployment, API Gateway, event source mappings

# Install SAM CLI
pip install aws-sam-cli

# Initialize project
sam init --runtime python3.12 --name my-serverless-app

# SAM template
cat > template.yaml << 'SAM'
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Serverless API Application

Globals:
  Function:
    Timeout: 30
    MemorySize: 256
    Runtime: python3.12
    Tracing: Active
    Environment:
      Variables:
        POWERTOOLS_SERVICE_NAME: my-api
        LOG_LEVEL: INFO
    Layers:
      - !Sub arn:aws:lambda:${AWS::Region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:7
    VpcConfig:
      SecurityGroupIds: [!Ref LambdaSG]
      SubnetIds: !Ref PrivateSubnets

Parameters:
  Environment:
    Type: String
    Default: production

Resources:
  # REST API
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: !Ref Environment
      Auth:
        DefaultAuthorizer: MyCognitoAuthorizer
        Authorizers:
          MyCognitoAuthorizer:
            UserPoolArn: !GetAtt UserPool.Arn
      AccessLogSetting:
        DestinationArn: !GetAtt ApiAccessLogs.Arn
      TracingEnabled: true
      Cors:
        AllowOrigin: "'https://myapp.com'"
        AllowMethods: "'GET,POST,PUT,DELETE,OPTIONS'"
        AllowHeaders: "'Content-Type,Authorization'"

  # Lambda functions
  GetOrdersFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: orders.get_orders
      CodeUri: src/orders/
      Events:
        GetOrders:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /orders
            Method: get
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref OrdersTable

  CreateOrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: orders.create_order
      CodeUri: src/orders/
      ReservedConcurrentExecutions: 100
      Events:
        CreateOrder:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /orders
            Method: post
        SQSOrders:
          Type: SQS
          Properties:
            Queue: !GetAtt OrderQueue.Arn
            BatchSize: 10
            FunctionResponseTypes: [ReportBatchItemFailures]
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
        - SQSSendMessagePolicy:
            QueueName: !GetAtt ProcessingQueue.QueueName

  ScheduledCleanupFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: cleanup.handler
      CodeUri: src/cleanup/
      Events:
        Nightly:
          Type: Schedule
          Properties:
            Schedule: cron(0 2 * * ? *)
            Description: Nightly database cleanup

  # DynamoDB
  OrdersTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      TableName: !Sub '${AWS::StackName}-orders'
      PrimaryKey: {Name: orderId, Type: String}
      ProvisionedThroughput: {ReadCapacityUnits: 5, WriteCapacityUnits: 5}

  # SQS
  OrderQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 300
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt OrderDLQ.Arn
        maxReceiveCount: 3

  OrderDLQ:
    Type: AWS::SQS::Queue

SAM

# Build
sam build --use-container

# Test locally
sam local invoke GetOrdersFunction --event events/get-orders.json
sam local start-api --port 3000   # local API Gateway

# Deploy
sam deploy \
  --stack-name my-serverless-app \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides Environment=production \
  --resolve-s3 \
  --confirm-changeset

# Delete
sam delete --stack-name my-serverless-app
```

---

### 🟡 Q29. What is AWS X-Ray?
```bash
# X-Ray: distributed tracing for microservices, Lambda, ECS, EKS

# Instrument Python Lambda function
pip install aws-xray-sdk
```

```python
from aws_xray_sdk.core import xray_recorder, patch_all
from aws_xray_sdk.core import patch

# Patch all supported libraries (boto3, requests, mysql, etc.)
patch_all()

@xray_recorder.capture("process_order")
def process_order(order_id: str) -> dict:
    # Subsegment for database call
    with xray_recorder.in_subsegment("database") as subsegment:
        subsegment.put_metadata("order_id", order_id)
        result = db.query(f"SELECT * FROM orders WHERE id = '{order_id}'")
        subsegment.put_annotation("db_records", len(result))

    # Subsegment for external API call
    with xray_recorder.in_subsegment("external-api") as subsegment:
        subsegment.put_annotation("service", "payment-gateway")
        response = requests.post("https://payment.example.com/charge", json=order)
        subsegment.put_http_meta("response_code", response.status_code)

    return result

def lambda_handler(event, context):
    # X-Ray automatically captures Lambda invocations
    xray_recorder.put_annotation("environment", os.environ.get("ENVIRONMENT"))
    return process_order(event["orderId"])
```

```bash
# Enable X-Ray on ECS (sidecar daemon)
# Add to task definition container:
# {
#   "name": "xray-daemon",
#   "image": "amazon/aws-xray-daemon:3.3.13",
#   "portMappings": [{"containerPort": 2000, "protocol": "udp"}],
#   "cpu": 32, "memory": 256, "essential": false
# }

# Enable X-Ray on EKS (DaemonSet)
kubectl apply -f https://github.com/aws/aws-distro-for-opentelemetry-operator/releases/latest/download/aws-otel-operator.yaml

# X-Ray + CloudWatch ServiceLens = unified observability
aws xray get-service-graph \
  --start-time $(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)

aws xray get-trace-summaries \
  --start-time $(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --filter-expression "service(\"my-api\") AND responsetime > 3"
```

---

### 🟡 Q30. What is AWS Elastic Beanstalk?
```bash
# Elastic Beanstalk: PaaS — deploy and scale web apps without managing infra
# Supports: Node.js, Python, Ruby, Java, .NET, PHP, Go, Docker

# Install EB CLI
pip install awsebcli

# Initialise application
eb init my-application \
  --platform "python-3.12" \
  --region us-east-1

# Create environment
eb create production \
  --cname my-app-prod \
  --elb-type application \
  --instance-type t3.medium \
  --min-instances 2 \
  --max-instances 20 \
  --database.engine mysql \
  --database.version 8.0 \
  --database.size db.t3.medium

# Deploy
eb deploy production

# .ebextensions — customise environment
mkdir .ebextensions
cat > .ebextensions/01_setup.config << 'EB'
option_settings:
  aws:autoscaling:launchconfiguration:
    InstanceType: t3.medium
    SecurityGroups: sg-12345678
  aws:autoscaling:asg:
    MinSize: 2
    MaxSize: 20
  aws:elasticbeanstalk:environment:
    EnvironmentType: LoadBalanced
  aws:elasticbeanstalk:application:environment:
    ENVIRONMENT: production
    DB_HOST: mydb.us-east-1.rds.amazonaws.com
  aws:elb:loadbalancer:
    CrossZone: true
  aws:elasticbeanstalk:cloudwatch:logs:
    StreamLogs: true
    RetentionInDays: 30

container_commands:
  01_migrate:
    command: "python manage.py migrate --no-input"
    leader_only: true
EB

# Environment configuration
cat > Procfile << 'PROC'
web: gunicorn myapp.wsgi:application --bind 0.0.0.0:8080 --workers 4 --timeout 120
PROC

# Blue/Green deployment
eb clone production --clone_name production-v2
eb deploy production-v2
eb swap production --destination_name production-v2

# Check status
eb status production
eb logs production --all
eb health production
```

---

### 🟡 Q31. What is AWS Amplify?
```bash
# Amplify: build and host full-stack web and mobile apps
# Backend: Auth (Cognito), API (AppSync/REST), Storage (S3), Functions (Lambda)
# Hosting: CDN delivery, CI/CD from Git, PR previews

# Install Amplify CLI
npm install -g @aws-amplify/cli
amplify configure   # set up IAM user

# Initialize project
amplify init

# Add authentication (Cognito)
amplify add auth
amplify push

# Add GraphQL API (AppSync)
amplify add api
# Choose GraphQL, create schema

# schema.graphql
cat > amplify/backend/api/myapp/schema.graphql << 'SCHEMA'
type Order @model @auth(rules: [{allow: owner}]) {
  id: ID!
  userId: String! @index(name: "byUserId")
  items: [OrderItem]
  total: Float!
  status: OrderStatus!
  createdAt: AWSDateTime!
}
type OrderItem {
  productId: String!
  quantity: Int!
  price: Float!
}
enum OrderStatus { PENDING CONFIRMED SHIPPED DELIVERED }
SCHEMA

amplify push --yes   # creates DynamoDB tables + AppSync resolvers automatically

# Add hosting (CI/CD)
amplify add hosting
# Choose: Amplify Console (Git-based deployment)
amplify publish

# React frontend
```

```javascript
// src/App.js — Amplify frontend
import { Amplify }  from 'aws-amplify';
import { generateClient } from 'aws-amplify/api';
import { createOrder, listOrders } from './graphql/mutations';
import { withAuthenticator } from '@aws-amplify/ui-react';
import awsconfig from './aws-exports';

Amplify.configure(awsconfig);
const client = generateClient();

async function createNewOrder(items, total) {
  const result = await client.graphql({
    query: createOrder,
    variables: { input: { items, total, status: 'PENDING' } }
  });
  return result.data.createOrder;
}

export default withAuthenticator(App);
// withAuthenticator: auto adds login/signup UI (hosted by Amplify)
```

---

### 🟡 Q32. What is AWS CodeArtifact?
```bash
# CodeArtifact: managed package repository (npm, PyPI, Maven, NuGet, Cargo, Swift)
# Benefits: store internal packages, proxy public repos, audit package usage

# Create domain and repository
aws codeartifact create-domain --domain my-company

aws codeartifact create-repository \
  --domain my-company \
  --repository my-packages \
  --description "Internal packages"

# Create repository with upstream (proxy PyPI)
aws codeartifact create-repository \
  --domain my-company \
  --repository pypi-store \
  --upstreams repositoryName=pypi-proxy

aws codeartifact associate-external-connection \
  --domain my-company \
  --repository pypi-store \
  --external-connection public:pypi

# Get authentication token
aws codeartifact get-authorization-token \
  --domain my-company \
  --query authorizationToken \
  --output text

# Configure pip to use CodeArtifact
aws codeartifact login \
  --tool pip \
  --domain my-company \
  --repository my-packages
# Sets pip.conf automatically

# Publish a package
pip install build twine
python -m build
twine upload \
  --repository-url https://my-company-123456789.d.codeartifact.us-east-1.amazonaws.com/pypi/my-packages/ \
  --username aws \
  --password $(aws codeartifact get-authorization-token --domain my-company --query authorizationToken --output text) \
  dist/*

# Configure npm
aws codeartifact login \
  --tool npm \
  --domain my-company \
  --repository my-packages

# List packages
aws codeartifact list-packages \
  --domain my-company \
  --repository my-packages \
  --format pypi --output table
```

---

### 🟡 Q33. What is AWS CodeGuru?
```bash
# CodeGuru Reviewer: AI-powered code review (security, performance, best practices)
# CodeGuru Profiler: continuous performance profiling of production apps

# ── CodeGuru Reviewer ─────────────────────────────────────────────
# Associate repository
aws codeguru-reviewer associate-repository \
  --repository '{"CodeCommit": {"Name": "my-app"}}'

# Or GitHub
aws codeguru-reviewer associate-repository \
  --repository '{"GitHubEnterpriseServer": {"ConnectionArn": "arn:aws:codestar-connections:...", "Name": "myOrg/my-app", "Owner": "myOrg"}}'

# Create code review (on demand)
aws codeguru-reviewer create-code-review \
  --name "PR-123-review" \
  --repository-association-arn arn:aws:codeguru-reviewer:... \
  --type '{"RepositoryAnalysis": {"RepositoryHead": {"BranchName": "main"}}}'

# CodeGuru detects:
# Security: AWS credentials in code, SQL injection, XSS, path traversal
# Performance: inefficient DB calls, N+1 queries, unnecessary large objects
# Best practices: proper error handling, thread safety, resource leaks

# ── CodeGuru Profiler ─────────────────────────────────────────────
# Create profiling group
aws codeguruprofiler create-profiling-group \
  --profiling-group-name my-app-profiler \
  --compute-platform Default   # Default | AWSLambda

# Instrument Python application
pip install codeguru-profiler-python-agent
```

```python
# In your application:
from codeguru_profiler_python_agent import Profiler

profiler = Profiler(profiling_group_name="my-app-profiler", region_name="us-east-1")
profiler.start()

# Your application code runs here...
# Profiler sends flame graphs to CodeGuru Profiler every 5 minutes

# For Lambda:
from codeguru_profiler_python_agent import with_lambda_profiler

@with_lambda_profiler(profiling_group_name="my-lambda-profiler")
def handler(event, context):
    return do_work()
```

---

### 🟡 Q34. What is AWS Cloud9?
```bash
# Cloud9: cloud-based IDE in browser — pre-configured with AWS CLI, SDKs
# Features: code, run, debug, direct AWS resource access, pair programming

# Create environment
aws cloud9 create-environment-ec2 \
  --name my-dev-env \
  --instance-type t3.medium \
  --image-id amazonlinux-2023-x86_64 \
  --subnet-id subnet-11111111 \
  --connection-type CONNECT_SSM \   # no inbound SSH needed
  --automatic-stop-time-minutes 60

# List environments
aws cloud9 list-environments --output table

# Share with collaborator
aws cloud9 create-environment-membership \
  --environment-id <env-id> \
  --user-arn arn:aws:iam::123456789:user/colleague \
  --permissions read-write

# Cloud9 benefits for containers:
# Pre-installed: Docker, kubectl, eksctl, aws-cli, sam-cli
# IAM role attached to Cloud9 instance — no credential management
# Direct access to VPC resources
```

---

### 🟡 Q35. What is AWS CodeWhisperer (Amazon Q Developer)?
```bash
# Amazon Q Developer (formerly CodeWhisperer): AI coding assistant
# Integrated in: VS Code, JetBrains, AWS Cloud9, CLI

# Features:
# Code generation: generate functions from comments
# Code completion: AI-powered suggestions in real time
# Security scanning: detect vulnerabilities as you code
# /transform command: upgrade Java 8/11 → Java 17/21 automatically
# /dev command: multi-file code generation
# Explain: explain complex code
# Fix: auto-fix security issues and bugs

# Amazon Q in CLI (q CLI):
q chat "What is the AWS CLI command to list all S3 buckets?"
q chat "Write a boto3 script to copy all objects from one S3 bucket to another"

# In terminal — explain a command
q translate "list all running EC2 instances in us-east-1"
# Output: aws ec2 describe-instances --region us-east-1 --filters "Name=instance-state-name,Values=running"

# Security scanning finds:
# Hardcoded credentials
# SQL injection vulnerabilities
# Cross-site scripting (XSS)
# Insecure cryptography
# Clear-text password logging
# Path traversal vulnerabilities
```

---

### 🟡 Q36. What is AWS Systems Manager (SSM)?
```bash
# SSM: operational management for EC2, on-prem, EKS nodes
# Key capabilities: Session Manager, Parameter Store, Patch Manager, Automation, Run Command

# ── Session Manager (secure SSH without opening port 22) ─────────
aws ssm start-session --target i-1234567890abcdef0

# Port forwarding via SSM (access private RDS without bastion)
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.us-east-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'
# Now: psql -h localhost -p 5432 -U admin mydb

# ── Parameter Store (secrets and config) ─────────────────────────
aws ssm put-parameter \
  --name "/myapp/production/db-password" \
  --value "MySecretP@ss!" \
  --type SecureString \
  --key-id arn:aws:kms:us-east-1:123456789:key/abc123 \
  --tier Advanced \     # Standard (4KB) | Advanced (8KB, expiration, policies)
  --tags Key=Environment,Value=production

aws ssm get-parameter \
  --name "/myapp/production/db-password" \
  --with-decryption \
  --query "Parameter.Value" \
  --output text

# Get all parameters by path
aws ssm get-parameters-by-path \
  --path "/myapp/production/" \
  --recursive --with-decryption \
  --query "Parameters[].{Name:Name,Value:Value}"

# ── Run Command (run scripts on multiple EC2 instances) ────────────
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=production" \
  --parameters "commands=['systemctl restart nginx','nginx -t && echo OK']" \
  --timeout-seconds 60 \
  --max-concurrency "50%" \
  --max-errors "10%"

# ── Patch Manager ─────────────────────────────────────────────────
aws ssm create-patch-baseline \
  --name "AmazonLinux2023-Security" \
  --operating-system AMAZON_LINUX_2023 \
  --approval-rules 'PatchRules=[{PatchFilterGroup:{PatchFilters:[{Key=SEVERITY,Values=[Critical,Important]}]},ApproveAfterDays=7,EnableNonSecurity=false}]'

aws ssm create-maintenance-window \
  --name "Saturday-Patching" \
  --schedule "cron(0 2 ? * SAT *)" \
  --duration 4 \
  --cutoff 1 \
  --allow-unassociated-targets false

# ── Automation (runbooks) ─────────────────────────────────────────
aws ssm start-automation-execution \
  --document-name "AWS-RestartEC2Instance" \
  --parameters "InstanceId=i-1234567890abcdef0"
```

---

### 🟡 Q37. What is AWS Secrets Manager?
```bash
# Secrets Manager: store, rotate, and retrieve secrets (DB passwords, API keys)
# vs SSM Parameter Store: Secrets Manager auto-rotates, more features, higher cost

# Create secret
aws secretsmanager create-secret \
  --name prod/myapp/db-credentials \
  --description "Production DB credentials" \
  --secret-string '{
    "username": "admin",
    "password": "MySecretP@ss!",
    "host": "mydb.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "dbname": "myapp"
  }' \
  --tags Key=Environment,Value=production

# Enable automatic rotation (Lambda)
aws secretsmanager rotate-secret \
  --secret-id prod/myapp/db-credentials \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789:function:SecretsManagerRDSRotation \
  --rotation-rules AutomaticallyAfterDays=30

# AWS-managed rotation templates (no custom Lambda needed for supported DBs):
# SecretsManagerRDSMySQLRotationSingleUser
# SecretsManagerRDSPostgreSQLRotationSingleUser
# SecretsManagerRDSOracleRotationSingleUser

aws secretsmanager rotate-secret \
  --secret-id prod/myapp/db-credentials \
  --rotation-rules '{"ScheduleExpression":"rate(30 days)","Duration":"2h"}' \
  --rotate-immediately

# Retrieve secret in Python
import boto3, json
from functools import lru_cache

@lru_cache(maxsize=None)
def get_secret(secret_name: str) -> dict:
    client = boto3.client("secretsmanager", region_name="us-east-1")
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

creds = get_secret("prod/myapp/db-credentials")
# Connect to DB using creds["host"], creds["username"], etc.

# Resource policy (cross-account access)
aws secretsmanager put-resource-policy \
  --secret-id prod/myapp/db-credentials \
  --resource-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::987654321:role/app-role"},
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "*"
    }]
  }'
```

---

### 🟡 Q38. What is AWS CloudShell?
```bash
# CloudShell: browser-based shell pre-authenticated with your AWS credentials
# Persistent storage: 1 GB per region
# Includes: AWS CLI, Python, Node.js, git, jq, Docker (no daemon), kubectl, helm

# Access: AWS Console → CloudShell icon (top right)
# Or: https://console.aws.amazon.com/cloudshell/

# CloudShell features:
# ✅ No credentials to configure — uses console session
# ✅ 1 GB persistent home directory per region
# ✅ Available in all regions
# ✅ Tabs: run multiple shells in same window
# ✅ File upload/download to/from shell environment
# ❌ No Docker daemon (can build with Finch or Buildah)
# ❌ No privileged containers

# Quick tasks in CloudShell:
aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,State.Name,Tags[?Key=='Name'].Value|[0]]" --output table
kubectl get pods --all-namespaces
helm list --all-namespaces
aws s3 ls s3://my-bucket/
```

---

### 🟡 Q39. What is AWS CloudWatch (developer perspective)?
```bash
# CloudWatch: metrics, logs, alarms, dashboards, insights for AWS services

# ── Custom Metrics ────────────────────────────────────────────────
aws cloudwatch put-metric-data \
  --namespace "MyApp/Performance" \
  --metric-name "OrderProcessingTime" \
  --dimensions Name=Environment,Value=production Name=Service,Value=orders \
  --value 245 \
  --unit Milliseconds \
  --timestamp $(date -u +%Y-%m-%dT%H:%M:%SZ)

# ── Alarms ───────────────────────────────────────────────────────
aws cloudwatch put-metric-alarm \
  --alarm-name HighErrorRate \
  --alarm-description "Error rate > 5% for 5 minutes" \
  --metric-name 5XXError \
  --namespace AWS/ApiGateway \
  --dimensions Name=ApiName,Value=my-api Name=Stage,Value=production \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:123456789:pagerduty-alerts \
  --ok-actions arn:aws:sns:us-east-1:123456789:recovery-alerts

# ── Log Insights queries ──────────────────────────────────────────
aws logs start-query \
  --log-group-name "/ecs/my-app" \
  --start-time $(date -u -d '-1 hour' +%s) \
  --end-time $(date -u +%s) \
  --query-string "
    fields @timestamp, @message
    | filter @message like /ERROR/
    | parse @message 'ERROR [*] *' as orderId, errorMsg
    | stats count(*) as errorCount by orderId
    | sort errorCount desc
    | limit 20
  "

# ── Container Insights for ECS/EKS ──────────────────────────────
aws ecs update-cluster-settings \
  --cluster my-cluster \
  --settings name=containerInsights,value=enabled

# ── CloudWatch Synthetics (canary monitoring) ────────────────────
aws synthetics create-canary \
  --name my-app-canary \
  --code S3Bucket=my-canary-bucket,S3Key=canary.zip,Handler=index.handler \
  --artifact-s3-location s3://my-canary-artifacts/ \
  --execution-role-arn arn:aws:iam::123456789:role/synthetics-role \
  --runtime-version syn-python-selenium-4.0 \
  --schedule Expression="rate(5 minutes)"

# ── Embedded Metrics Format (EMF) — structured metrics from Lambda/ECS ──
import json
from datetime import datetime

def put_metric_emf(metric_name: str, value: float, unit: str = "Count"):
    """Write metric using EMF — no separate PutMetricData API call needed."""
    print(json.dumps({
        "_aws": {
            "Timestamp": int(datetime.utcnow().timestamp() * 1000),
            "CloudWatchMetrics": [{
                "Namespace": "MyApp/Business",
                "Dimensions": [["Environment", "Service"]],
                "Metrics": [{"Name": metric_name, "Unit": unit}]
            }]
        },
        "Environment": "production",
        "Service": "orders",
        metric_name: value
    }))

# Logged to CloudWatch Logs → automatically extracted as metrics
put_metric_emf("OrdersProcessed", 1)
put_metric_emf("OrderValue", 99.99, "None")
```

---

### 🟢 Q40. What is AWS EventBridge (developer perspective)?
```bash
# EventBridge: serverless event bus — connect AWS services and your apps
# Event sources: 200+ AWS services, your apps, SaaS partners

# Create custom event bus
aws events create-event-bus --name my-app-events

# Put events
aws events put-events --entries '[
  {
    "Source": "com.myapp.orders",
    "DetailType": "OrderCreated",
    "Detail": "{\"orderId\": \"12345\", \"customerId\": \"C001\", \"amount\": 99.99}",
    "EventBusName": "my-app-events"
  }
]'

# Create rule to route to Lambda
aws events put-rule \
  --name "ProcessNewOrders" \
  --event-bus-name my-app-events \
  --event-pattern '{
    "source": ["com.myapp.orders"],
    "detail-type": ["OrderCreated"],
    "detail": {"amount": [{"numeric": [">", 100]}]}
  }' \
  --state ENABLED

aws events put-targets \
  --rule ProcessNewOrders \
  --event-bus-name my-app-events \
  --targets '[
    {
      "Id": "process-order-lambda",
      "Arn": "arn:aws:lambda:us-east-1:123456789:function:process-order"
    },
    {
      "Id": "archive-to-sqs",
      "Arn": "arn:aws:sqs:us-east-1:123456789:orders-queue",
      "SqsParameters": {"MessageGroupId": "orders"}
    }
  ]'

# Scheduled rules (replaces CloudWatch Events cron)
aws events put-rule \
  --name "DailyReport" \
  --schedule-expression "cron(0 8 * * ? *)" \
  --state ENABLED

# EventBridge Pipes (point-to-point event processing)
aws pipes create-pipe \
  --name sqs-to-lambda \
  --role-arn arn:aws:iam::123456789:role/pipes-role \
  --source arn:aws:sqs:us-east-1:123456789:orders-queue \
  --target arn:aws:lambda:us-east-1:123456789:function:process-order \
  --enrichment arn:aws:lambda:us-east-1:123456789:function:enrich-order \
  --filter '{"Filters": [{"Pattern": "{\"body\": {\"amount\": [{\"numeric\": [\">\", 100]}]}}"}]}'

# EventBridge Scheduler (improved cron replacement)
aws scheduler create-schedule \
  --name daily-cleanup \
  --schedule-expression "cron(0 2 * * ? *)" \
  --flexible-time-window '{"Mode":"FLEXIBLE","MaximumWindowInMinutes":60}' \
  --target '{
    "Arn": "arn:aws:lambda:us-east-1:123456789:function:cleanup",
    "RoleArn": "arn:aws:iam::123456789:role/scheduler-role",
    "RetryPolicy": {"MaximumRetryAttempts": 3, "MaximumEventAgeInSeconds": 3600}
  }'
```


---

# PART 3 — AWS MANAGEMENT & GOVERNANCE

---

### 🟢 Q41. What is AWS Organizations?
```bash
# Organizations: manage multiple AWS accounts centrally
# Features: consolidated billing, SCPs, AWS Control Tower, account vending

# Create organization
aws organizations create-organization --feature-set ALL

# Create Organizational Units (OUs) — hierarchical grouping
ROOT_ID=$(aws organizations list-roots --query "Roots[0].Id" --output text)

aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Workloads"

aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Sandbox"

aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Infrastructure"

# Create account in org
aws organizations create-account \
  --email prod-account@mycompany.com \
  --account-name "Production Account" \
  --role-name OrganizationAccountAccessRole \
  --iam-user-access-to-billing ALLOW \
  --tags Key=Environment,Value=production

# Move account to OU
aws organizations move-account \
  --account-id 123456789 \
  --source-parent-id $ROOT_ID \
  --destination-parent-id ou-abc123-def456

# OU structure (best practice):
# Root
# ├── Security (log archive, security tooling accounts)
# ├── Infrastructure (network, shared services)
# ├── Workloads
# │   ├── Production
# │   ├── Development
# │   └── Testing
# └── Sandbox (developer playground, loose restrictions)

# Enable all features (required for SCPs)
aws organizations enable-all-features

# Delegate administration (allow member account to manage a service)
aws organizations register-delegated-administrator \
  --account-id 123456789 \
  --service-principal config.amazonaws.com

aws organizations register-delegated-administrator \
  --account-id 123456789 \
  --service-principal guardduty.amazonaws.com
```

---

### 🟡 Q42. What are Service Control Policies (SCPs)?
```bash
# SCPs: guardrails on what actions can be performed in member accounts
# Key: SCPs limit what IAM policies can grant — they don't grant permissions
# Even Account Root cannot do something blocked by SCP

# SCP evaluation:
# Identity-based policy AND SCP must both allow → access granted
# Either denies (explicit or implicit) → access denied

# Create SCP — deny deleting CloudTrail
aws organizations create-policy \
  --name "DenyCloudTrailDeletion" \
  --description "Prevent anyone from deleting CloudTrail trails" \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyCloudTrailModification",
        "Effect": "Deny",
        "Action": [
          "cloudtrail:StopLogging",
          "cloudtrail:DeleteTrail",
          "cloudtrail:UpdateTrail",
          "cloudtrail:PutEventSelectors"
        ],
        "Resource": "*"
      }
    ]
  }'

# Create SCP — deny leaving org + deleting GuardDuty
aws organizations create-policy \
  --name "SecurityBaseline" \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyLeaveOrg",
        "Effect": "Deny",
        "Action": "organizations:LeaveOrganization",
        "Resource": "*"
      },
      {
        "Sid": "DenyDisableGuardDuty",
        "Effect": "Deny",
        "Action": [
          "guardduty:DeleteDetector",
          "guardduty:DisassociateFromMasterAccount",
          "guardduty:StopMonitoringMembers",
          "guardduty:UpdateDetector"
        ],
        "Resource": "*"
      },
      {
        "Sid": "DenyDisableSecurityHub",
        "Effect": "Deny",
        "Action": [
          "securityhub:DeleteHub",
          "securityhub:DisableSecurityHub",
          "securityhub:DisassociateFromMasterAccount"
        ],
        "Resource": "*"
      }
    ]
  }'

# Region restriction SCP (only allow specific regions)
aws organizations create-policy \
  --name "AllowedRegions" \
  --type SERVICE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "DenyNonApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "cloudfront:*", "iam:*", "route53:*", "sts:*", "support:*",
        "globalaccelerator:*", "organizations:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2", "eu-west-1"]
        }
      }
    }]
  }'

# Attach SCP to OU
aws organizations attach-policy \
  --policy-id p-abc123 \
  --target-id ou-abc123-def456

# Attach SCP to specific account
aws organizations attach-policy \
  --policy-id p-abc123 \
  --target-id 123456789

# FullAWSAccess SCP (default, attached to all)
# Must keep this attached or NO actions are allowed at all
```

---

### 🟡 Q43. What is AWS Control Tower?
```bash
# Control Tower: landing zone setup and governance — automates org structure
# Creates: multi-account environment with security baseline, guardrails, Account Factory

# Set up Control Tower (via Console — landing zone)
# 1. Launches management account
# 2. Creates Log Archive account (all accounts send logs here)
# 3. Creates Audit account (security team access)
# 4. Creates OU structure
# 5. Enables CloudTrail, Config, GuardDuty, Security Hub across all accounts
# 6. Applies baseline SCPs (guardrails)

# Guardrail types:
# Preventive: SCP-based, stops non-compliant actions (e.g., deny public S3 buckets)
# Detective: Config rule-based, detects violations (e.g., detect unencrypted EBS)
# Proactive: CloudFormation hooks — prevent non-compliant resources from being created

# Account Factory (provision new accounts via self-service)
aws servicecatalog describe-product \
  --name "AWS Control Tower Account Factory"

# Account Factory via CLI (or Service Catalog)
aws servicecatalog provision-product \
  --product-name "AWS Control Tower Account Factory" \
  --provisioning-artifact-name "AWS Control Tower Account Factory" \
  --provisioned-product-name "new-dev-account" \
  --provisioning-parameters \
    Key=AccountEmail,Value=dev-team@mycompany.com \
    Key=AccountName,Value="Dev Team Account" \
    Key=SSOUserEmail,Value=developer@mycompany.com \
    Key=SSOUserFirstName,Value=John \
    Key=SSOUserLastName,Value=Smith \
    Key=ManagedOrganizationalUnit,Value="Workloads/Development"

# Control Tower Customizations (CfCT) — apply custom SCPs and CloudFormation
# to all accounts in OU automatically
# Uses: CodePipeline + CloudFormation StackSets

# Account Factory for Terraform (AFT)
# Provision AWS accounts using Terraform
# GitHub repo → CodePipeline → Terraform → new account
```

---

### 🟢 Q44. What is AWS CloudTrail?
```bash
# CloudTrail: audit log of all API calls in your AWS account
# Who did what, when, from where — essential for security and compliance

# Create trail
aws cloudtrail create-trail \
  --name my-audit-trail \
  --s3-bucket-name my-cloudtrail-bucket \
  --s3-key-prefix "cloudtrail" \
  --include-global-service-events \
  --is-multi-region-trail \       # capture all regions
  --enable-log-file-validation   # detect log tampering

aws cloudtrail start-logging --name my-audit-trail

# Enable CloudWatch Logs integration
aws cloudtrail update-trail \
  --name my-audit-trail \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789:log-group:cloudtrail \
  --cloud-watch-logs-role-arn arn:aws:iam::123456789:role/cloudtrail-cloudwatch-role

# Enable Data Events (S3 object-level, Lambda invocations — additional cost)
aws cloudtrail put-event-selectors \
  --trail-name my-audit-trail \
  --event-selectors '[
    {
      "ReadWriteType": "WriteOnly",
      "IncludeManagementEvents": true,
      "DataResources": [
        {"Type": "AWS::S3::Object", "Values": ["arn:aws:s3:::my-sensitive-bucket/"]},
        {"Type": "AWS::Lambda::Function", "Values": ["arn:aws:lambda:::function:my-critical-func"]}
      ]
    }
  ]'

# Enable Insights (detect unusual API call rates)
aws cloudtrail put-insight-selectors \
  --trail-name my-audit-trail \
  --insight-selectors '[
    {"InsightType": "ApiCallRateInsight"},
    {"InsightType": "ApiErrorRateInsight"}
  ]'

# CloudTrail Lake (query events with SQL — no S3 needed)
aws cloudtrail create-event-data-store \
  --name my-event-store \
  --retention-period 2190 \         # 6 years
  --multi-region-enabled \
  --organization-enabled \          # all accounts in org
  --kms-key-id arn:aws:kms:...

# Query CloudTrail Lake with SQL
aws cloudtrail start-query \
  --query-statement "
    SELECT eventTime, userIdentity.arn, eventName, sourceIPAddress, requestParameters
    FROM my-event-store-id
    WHERE eventName IN ('DeleteBucket', 'PutBucketPublicAccessBlock', 'DeleteObject')
    AND eventTime > '2026-06-01 00:00:00'
    ORDER BY eventTime DESC
    LIMIT 100
  "

# Query using CloudWatch Logs Insights
aws logs start-query \
  --log-group-name "cloudtrail" \
  --start-time $(date -u -d '-24 hours' +%s) \
  --end-time $(date -u +%s) \
  --query-string "
    fields @timestamp, userIdentity.arn, eventName, sourceIPAddress
    | filter eventName in ['ConsoleLogin'] and responseElements.ConsoleLogin = 'Failure'
    | stats count(*) as FailedLogins by userIdentity.arn
    | sort FailedLogins desc
  "
```

---

### 🟢 Q45. What is AWS Config?
```bash
# AWS Config: continuous resource configuration tracking + compliance checking
# Records: when resources change, what changed, who changed, relationships

# Enable Config recording
aws configservice put-configuration-recorder \
  --configuration-recorder '{
    "name": "default",
    "roleARN": "arn:aws:iam::123456789:role/config-role",
    "recordingGroup": {
      "allSupported": true,
      "includeGlobalResourceTypes": true,
      "recordingStrategy": {
        "useOnly": "ALL_SUPPORTED_RESOURCE_TYPES"
      }
    }
  }'

aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "s3BucketName": "my-config-bucket",
    "snsTopicARN": "arn:aws:sns:us-east-1:123456789:config-notifications",
    "configSnapshotDeliveryProperties": {"deliveryFrequency": "TwentyFour_Hours"}
  }'

aws configservice start-configuration-recorder --configuration-recorder-name default

# Create managed rule (pre-built compliance rules)
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "require-encrypted-volumes",
    "Source": {"Owner": "AWS", "SourceIdentifier": "ENCRYPTED_VOLUMES"},
    "Description": "All EBS volumes must be encrypted",
    "ConfigRuleState": "ACTIVE"
  }'

# Custom Config rule (Lambda-based)
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "require-tags",
    "Source": {
      "Owner": "CUSTOM_LAMBDA",
      "SourceIdentifier": "arn:aws:lambda:us-east-1:123456789:function:require-tags-check",
      "SourceDetails": [{
        "EventSource": "aws.config",
        "MessageType": "ConfigurationItemChangeNotification"
      }]
    },
    "InputParameters": "{\"requiredTags\": \"Environment,Owner,CostCenter\"}",
    "Scope": {"ComplianceResourceTypes": ["AWS::EC2::Instance", "AWS::RDS::DBInstance"]}
  }'

# Check compliance
aws configservice get-compliance-summary-by-config-rule --output table

aws configservice get-compliance-details-by-config-rule \
  --config-rule-name require-encrypted-volumes \
  --compliance-types NON_COMPLIANT \
  --query "EvaluationResults[].EvaluationResultIdentifier.EvaluationResultQualifier.ResourceId"

# Config Aggregator (view compliance across all accounts/regions)
aws configservice put-configuration-aggregator \
  --configuration-aggregator-name org-aggregator \
  --organization-aggregation-source '{
    "RoleArn": "arn:aws:iam::123456789:role/config-org-role",
    "AllAwsRegions": true
  }'

# Conformance Packs (deploy multiple rules as one package)
aws configservice put-conformance-pack \
  --conformance-pack-name "operational-best-practices-for-cis" \
  --template-s3-uri s3://my-config-bucket/CISAWSFoundationsBenchmark.yaml \
  --delivery-s3-bucket my-config-bucket
```

---

### 🟡 Q46. What is AWS IAM (comprehensive)?
```bash
# IAM: Identity and Access Management — who can do what in AWS

# Users, Groups, Roles, Policies — core IAM components

# Create user
aws iam create-user --user-name alice \
  --tags Key=Department,Value=Engineering

# Add to group
aws iam add-user-to-group --user-name alice --group-name developers

# Create inline policy on user
aws iam put-user-policy \
  --user-name alice \
  --policy-name s3-access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject","s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "StringEquals": {"s3:prefix": ["uploads/alice/"]}
      }
    }]
  }'

# Create customer managed policy
aws iam create-policy \
  --policy-name ECSReadOnly \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "ecs:Describe*", "ecs:List*",
        "ecr:Describe*", "ecr:List*", "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "*"
    }]
  }'

aws iam attach-user-policy \
  --user-name alice \
  --policy-arn arn:aws:iam::123456789:policy/ECSReadOnly

# IAM role (for services/cross-account)
aws iam create-role \
  --role-name lambda-execution-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Cross-account role
aws iam create-role \
  --role-name cross-account-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::987654321:root"},
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {"aws:MultiFactorAuthPresent": "true"},
        "StringEquals": {"sts:ExternalId": "myUniqueExternalId"}
      }
    }]
  }'

# Permission boundary (limit max permissions a user/role can have)
aws iam create-policy \
  --policy-name DeveloperBoundary \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {"Effect": "Allow", "Action": ["s3:*", "ec2:*", "lambda:*", "logs:*"], "Resource": "*"},
      {"Effect": "Deny", "Action": ["iam:CreateUser", "iam:DeleteUser", "organizations:*"], "Resource": "*"}
    ]
  }'

aws iam put-user-permissions-boundary \
  --user-name alice \
  --permissions-boundary arn:aws:iam::123456789:policy/DeveloperBoundary

# Simulate policy
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789:user/alice \
  --action-names s3:GetObject ec2:RunInstances iam:DeleteUser \
  --resource-arns "arn:aws:s3:::my-bucket/test.txt" \
  --query "EvaluationResults[].{Action:EvalActionName,Decision:EvalDecision}"

# MFA enforcement policy (deny if no MFA)
# (Applied to user via group policy to force MFA setup)
'{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowViewAccountInfo",
      "Effect": "Allow",
      "Action": ["iam:GetAccountPasswordPolicy", "iam:ListVirtualMFADevices"],
      "Resource": "*"
    },
    {
      "Sid": "AllowManageOwnMFA",
      "Effect": "Allow",
      "Action": ["iam:CreateVirtualMFADevice", "iam:EnableMFADevice",
                 "iam:GetUser", "iam:ListMFADevices"],
      "Resource": [
        "arn:aws:iam::*:mfa/${aws:username}",
        "arn:aws:iam::*:user/${aws:username}"
      ]
    },
    {
      "Sid": "DenyAllExceptListedIfNoMFA",
      "Effect": "Deny",
      "NotAction": ["iam:CreateVirtualMFADevice", "iam:EnableMFADevice",
                    "iam:GetUser", "iam:ListMFADevices", "iam:ListVirtualMFADevices",
                    "iam:ResyncMFADevice", "sts:GetSessionToken"],
      "Resource": "*",
      "Condition": {"BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}}
    }
  ]
}'
```

---

### 🟡 Q47. What is AWS IAM Identity Center (SSO)?
```bash
# IAM Identity Center (formerly SSO): centralised access for multiple AWS accounts
# Users sign in once → access all assigned accounts/apps with same identity
# Integrates with: Active Directory, Okta, Azure AD (Entra ID), Ping

# Enable IAM Identity Center (console only for initial setup)
# aws sso-admin ... (configure via CLI after enabling in console)

# Create permission sets
aws sso-admin create-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123 \
  --name "AdministratorAccess" \
  --description "Full admin for production accounts" \
  --session-duration PT8H

aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123 \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-abc123/ps-abc123 \
  --managed-policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Assign user to account with permission set
aws sso-admin create-account-assignment \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123 \
  --target-id 123456789 \
  --target-type AWS_ACCOUNT \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-abc123/ps-abc123 \
  --principal-type USER \
  --principal-id <user-id-in-identity-store>

# Assign group to account
aws sso-admin create-account-assignment \
  --instance-arn arn:aws:sso:::instance/ssoins-abc123 \
  --target-id 123456789 \
  --target-type AWS_ACCOUNT \
  --permission-set-arn arn:aws:sso:::permissionSet/ssoins-abc123/ps-abc123 \
  --principal-type GROUP \
  --principal-id <group-id-in-identity-store>

# CLI: assume role via SSO
aws configure sso \
  --sso-start-url https://mycompany.awsapps.com/start \
  --sso-region us-east-1 \
  --sso-account-id 123456789 \
  --sso-role-name AdministratorAccess \
  --region us-east-1 \
  --profile prod-admin

aws sso login --profile prod-admin
aws s3 ls --profile prod-admin
```

---

### 🟡 Q48. What is AWS Trusted Advisor?
```bash
# Trusted Advisor: automated best practice recommendations
# Categories: Cost Optimisation, Performance, Security, Fault Tolerance, Service Limits

# Check all recommendations
aws trustedadvisor list-recommendations \
  --pillar cost_optimising \
  --status warning error \
  --output table

aws trustedadvisor list-recommendations \
  --pillar security \
  --status warning error \
  --output table

aws trustedadvisor list-recommendations \
  --pillar fault_tolerance \
  --output table

# Get specific check details
aws trustedadvisor get-recommendation \
  --recommendation-identifier <id>

# Priority: Investigate Immediately | Medium | Low
# Exclude specific resources from check
aws trustedadvisor update-recommendation-resource \
  --recommendation-identifier <id> \
  --resource-identifier <resource-arn> \
  --exclusion-status excluded

# Key checks:
# COST: unused EBS volumes, unassociated EIPs, idle LBs, RIs utilisation
# SECURITY: unrestricted SGs, MFA on root, public S3 buckets, CloudTrail not enabled
# PERFORMANCE: high-utilisation EC2, CloudFront caching, EBS throughput optimisation
# FAULT TOLERANCE: no Multi-AZ RDS, missing backups, VPN tunnel redundancy
# SERVICE LIMITS: approaching EC2/VPC/IAM limits

# Note: some checks require Business/Enterprise Support tier
# Basic: ~6 security checks only
# Business/Enterprise: all checks + API access
```

---

### 🟡 Q49. What is AWS Well-Architected Tool?
```bash
# Well-Architected Tool: review workloads against AWS best practices
# 5+1 Pillars: Operational Excellence, Security, Reliability, Performance, Cost, Sustainability

# Create workload
aws wellarchitected create-workload \
  --workload-name "My E-commerce Platform" \
  --description "Customer-facing e-commerce application" \
  --environment PRODUCTION \
  --aws-regions us-east-1 eu-west-1 \
  --pillar-priorities COST_OPTIMISATION SECURITY RELIABILITY PERFORMANCE_EFFICIENCY OPERATIONAL_EXCELLENCE SUSTAINABILITY \
  --review-owner "architecture-team@mycompany.com" \
  --lenses wellarchitected

# List questions for a pillar
aws wellarchitected list-lens-review-improvements \
  --workload-id <workload-id> \
  --lens-alias wellarchitected \
  --pillar-id security

# Answer questions
aws wellarchitected update-answer \
  --workload-id <workload-id> \
  --lens-alias wellarchitected \
  --question-id "ops_workload_documentation" \
  --selected-choices ops_q1_a1 ops_q1_a2 \
  --notes "We use Confluence for documentation with quarterly reviews"

# Get risk summary
aws wellarchitected get-lens-review \
  --workload-id <workload-id> \
  --lens-alias wellarchitected \
  --query "LensReview.{RiskCounts:RiskCounts,Score:LensScore}"

# Generate report (PDF)
aws wellarchitected get-lens-review-report \
  --workload-id <workload-id> \
  --lens-alias wellarchitected \
  --milestone-number 1

# Custom lenses (your own best practices)
aws wellarchitected create-lens-version \
  --lens-alias my-security-lens \
  --lens-version "1.0" \
  --json-string file://my-lens.json
```

---

### 🟡 Q50. What is AWS Cost Explorer and Cost Management?
```bash
# Cost Explorer: visualise and analyse AWS spending
# Budgets: alert when spending approaches or exceeds threshold
# Savings Plans / Reserved Instances: commitment discounts

# Create budget
aws budgets create-budget \
  --account-id 123456789 \
  --budget '{
    "BudgetName": "monthly-cost-budget",
    "BudgetType": "COST",
    "TimeUnit": "MONTHLY",
    "BudgetLimit": {"Amount": "10000", "Unit": "USD"},
    "CostFilters": {
      "Service": ["Amazon Elastic Compute Cloud"],
      "TagKeyValue": ["user:Environment$production"]
    },
    "CostTypes": {
      "IncludeTax": true,
      "IncludeSubscription": true,
      "UseBlended": false
    }
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "finance@mycompany.com"}]
    },
    {
      "Notification": {
        "NotificationType": "FORECASTED",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 100,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        {"SubscriptionType": "EMAIL",   "Address": "cto@mycompany.com"},
        {"SubscriptionType": "SNS",     "Address": "arn:aws:sns:us-east-1:123456789:cost-alerts"}
      ]
    }
  ]'

# Get cost and usage
aws ce get-cost-and-usage \
  --time-period Start=2026-06-01,End=2026-06-30 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query "ResultsByTime[0].Groups[*].{Service:Keys[0],Cost:Metrics.BlendedCost.Amount}" \
  --output table | sort -k2 -rn | head -20

# Top 10 most expensive resources
aws ce get-cost-and-usage \
  --time-period Start=2026-06-01,End=2026-06-30 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=TAG,Key=Environment Type=DIMENSION,Key=SERVICE

# Cost anomaly detection
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "ServiceMonitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "DailyAnomalyAlert",
    "MonitorArnList": ["arn:aws:ce::123456789:anomalymonitor/abc123"],
    "Subscribers": [{"Address": "finops@mycompany.com", "Type": "EMAIL"}],
    "Threshold": 100,
    "Frequency": "DAILY"
  }'

# Rightsizing recommendations
aws ce get-rightsizing-recommendation \
  --service EC2 \
  --configuration LookbackPeriodInDays=30,BenefitsConsidered=NONE \
  --query "RightsizingRecommendations[].{Instance:CurrentInstance.InstanceName,Action:RightsizingType,Savings:RightsizingType}" \
  --output table

# Savings Plans purchase recommendation
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days THIRTY_DAYS
```

---

### 🟡 Q51. What is AWS Security Hub?
```bash
# Security Hub: centralised security findings across AWS accounts
# Aggregates findings from: GuardDuty, Inspector, Macie, Config, IAM AA, Firewall Manager
# Standards: CIS AWS Foundations, AWS Foundational Security Best Practices, PCI DSS, NIST

# Enable Security Hub
aws securityhub enable-security-hub \
  --enable-default-standards \
  --control-finding-generator SECURITY_CONTROL

# Enable standards
aws securityhub batch-enable-standards \
  --standards-subscription-requests \
    StandardsArn=arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0 \
    StandardsArn=arn:aws:securityhub:us-east-1::standards/cis-aws-foundations-benchmark/v/1.4.0

# Get security score
aws securityhub describe-standards-controls \
  --standards-subscription-arn arn:aws:securityhub:us-east-1:123456789:subscription/aws-foundational-security-best-practices/v/1.0.0 \
  --query "Controls[?ControlStatus=='FAILED'].[ControlId,Title,SeverityRating]" \
  --output table

# Get high severity findings
aws securityhub get-findings \
  --filters '{
    "SeverityLabel": [{"Value": "CRITICAL", "Comparison": "EQUALS"}, {"Value": "HIGH", "Comparison": "EQUALS"}],
    "RecordState": [{"Value": "ACTIVE", "Comparison": "EQUALS"}],
    "WorkflowStatus": [{"Value": "NEW", "Comparison": "EQUALS"}]
  }' \
  --sort-criteria Field=SeverityNormalized,SortOrder=desc \
  --max-items 20 \
  --query "Findings[].{Title:Title,Severity:Severity.Label,Resource:Resources[0].Id,Account:AwsAccountId}"

# Create custom insight
aws securityhub create-insight \
  --name "PublicS3Buckets" \
  --filters '{
    "Type": [{"Value": "Software and Configuration Checks/AWS Security Best Practices", "Comparison": "PREFIX"}],
    "ResourceType": [{"Value": "AwsS3Bucket", "Comparison": "EQUALS"}]
  }' \
  --group-by-attribute ResourceId

# Automate response with EventBridge
# EventBridge rule → trigger Lambda on CRITICAL finding
aws events put-rule \
  --name "SecurityHubCriticalFinding" \
  --event-pattern '{
    "source": ["aws.securityhub"],
    "detail-type": ["Security Hub Findings - Imported"],
    "detail": {
      "findings": {
        "Severity": {"Label": ["CRITICAL"]},
        "Workflow": {"Status": ["NEW"]},
        "RecordState": ["ACTIVE"]
      }
    }
  }'
```

---

### 🟡 Q52. What is Amazon GuardDuty?
```bash
# GuardDuty: intelligent threat detection using ML
# Analyses: VPC Flow Logs, CloudTrail, DNS logs, EKS audit logs, S3 data events

# Enable GuardDuty
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --data-sources '{
    "S3Logs": {"Enable": true},
    "Kubernetes": {"AuditLogs": {"Enable": true}},
    "MalwareProtection": {"ScanEc2InstanceWithFindings": {"EbsVolumes": {"Enable": true}}},
    "RdsLoginEvents": {"Enable": true},
    "RuntimeMonitoring": {
      "RuntimeConfigurationUpdate": {"AutoEnable": "ALL"}
    }
  }'

DETECTOR_ID=$(aws guardduty list-detectors --query "DetectorIds[0]" --output text)

# Get findings
aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --finding-criteria '{
    "Criterion": {
      "severity": {"Gte": 7},
      "service.archived": {"Eq": ["false"]}
    }
  }' \
  --sort-criteria AttributeName=severity,OrderBy=DESC

aws guardduty get-findings \
  --detector-id $DETECTOR_ID \
  --finding-ids <finding-id> \
  --query "Findings[].{Type:Type,Severity:Severity,Title:Title,Resource:Resource.ResourceType}"

# GuardDuty finding types:
# CryptoCurrency:BitcoinTool              (crypto mining)
# Backdoor:EC2/C&CActivity               (command and control)
# UnauthorizedAccess:IAMUser/ConsoleLogin (suspicious login)
# Recon:IAMUser/MaliciousIPCaller        (reconnaissance)
# PrivilegeEscalation:Kubernetes/PrivilegedContainer
# Discovery:S3/MaliciousIPCaller
# Exfiltration:S3/ObjectRead
# Impact:EC2/PortScanOnEC2               (lateral movement)

# Suppress low-severity noisy findings
aws guardduty create-filter \
  --detector-id $DETECTOR_ID \
  --name SuppressSSHBruteForceKnownIPs \
  --action ARCHIVE \
  --finding-criteria '{
    "Criterion": {
      "type": {"Eq": ["UnauthorizedAccess:EC2/SSHBruteForce"]},
      "resource.instanceDetails.networkInterfaces[0].publicIp": {
        "Eq": ["203.0.113.5"]
      }
    }
  }'

# Export findings to S3 (for SIEM)
aws guardduty create-publishing-destination \
  --detector-id $DETECTOR_ID \
  --destination-type S3 \
  --destination-properties '{
    "DestinationArn": "arn:aws:s3:::my-security-findings",
    "KmsKeyArn": "arn:aws:kms:us-east-1:123456789:key/abc123"
  }'

# Multi-account GuardDuty (Security account as delegated admin)
aws guardduty enable-organization-admin-account \
  --admin-account-id 123456789

aws guardduty update-organization-configuration \
  --detector-id $DETECTOR_ID \
  --auto-enable ALL \
  --data-sources '{"S3Logs":{"AutoEnable":true},"Kubernetes":{"AuditLogs":{"AutoEnable":true}}}'
```

---

### 🟡 Q53. What is AWS Inspector?
```bash
# Inspector v2: automated vulnerability scanning for EC2, ECR images, Lambda functions
# Continuously scans — not just on-demand

# Enable Inspector
aws inspector2 enable \
  --resource-types EC2 ECR LAMBDA LAMBDA_CODE

# Get findings
aws inspector2 list-findings \
  --filter-criteria '{
    "severity": [{"comparison": "EQUALS", "value": "CRITICAL"}],
    "findingStatus": [{"comparison": "EQUALS", "value": "ACTIVE"}]
  }' \
  --sort-criteria field=SEVERITY,sortOrder=DESC \
  --max-results 20 \
  --query "findings[].{Title:title,Severity:severity,Resource:resources[0].id,Score:inspectorScore}"

# ECR enhanced scanning (Inspector scans images automatically)
aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{
    "repositoryFilters": [{"filter": "*", "filterType": "WILDCARD"}],
    "scanFrequency": "CONTINUOUS_SCAN"
  }]'

# Check ECR image findings
aws inspector2 list-findings \
  --filter-criteria '{
    "resourceType": [{"comparison": "EQUALS", "value": "AWS_ECR_CONTAINER_IMAGE"}],
    "ecrImageRepositoryName": [{"comparison": "EQUALS", "value": "my-app"}]
  }'

# Lambda code scanning (new — scans Lambda function code)
aws inspector2 update-configuration \
  --ec2-configuration '{"scanMode": "EC2_HYBRID"}' \
  --ecr-configuration '{"rescanDuration": "DAYS_30"}' \
  --lambda-scan-mode ENABLED

# Suppress false positives
aws inspector2 create-filter \
  --name "SuppressExpiredCVE" \
  --action SUPPRESS \
  --filter-criteria '{
    "fixAvailable": [{"comparison": "EQUALS", "value": "NO"}],
    "severity": [{"comparison": "EQUALS", "value": "MEDIUM"}]
  }'

# Inspector + Security Hub integration (findings appear in Security Hub)
# Enabled automatically when both services are active
```

---

### 🟡 Q54. What is AWS Macie?
```bash
# Macie: ML-powered S3 data security — discover, classify, protect sensitive data
# Detects: PII, financial data, credentials, health data in S3

# Enable Macie
aws macie2 enable-macie \
  --status ENABLED \
  --finding-publishing-frequency FIFTEEN_MINUTES

# Create classification job
aws macie2 create-classification-job \
  --job-type ONE_TIME \
  --name "PII-Scan-All-Buckets" \
  --s3-job-definition '{
    "bucketDefinitions": [{"accountId": "123456789", "buckets": ["*"]}],
    "scoping": {
      "includes": {
        "and": [{
          "simpleScopeTerm": {
            "comparator": "EQ",
            "key": "OBJECT_EXTENSION",
            "values": [".csv", ".json", ".txt", ".xlsx", ".pdf"]
          }
        }]
      }
    }
  }' \
  --managed-data-identifier-selector ALL \
  --sampling-percentage 100

# Get findings
aws macie2 list-findings \
  --finding-criteria '{
    "criterion": {
      "severity.description": {"eq": ["High", "Critical"]},
      "archived": {"eq": ["false"]}
    }
  }'

aws macie2 get-findings \
  --finding-ids <finding-id> \
  --query "findings[].{Bucket:resourcesAffected.s3Bucket.name,DataType:classificationDetails.result.sensitiveData[0].category,Occurrences:classificationDetails.result.sensitiveData[0].totalCount}"

# Sensitive data categories:
# CREDENTIALS:    access keys, passwords, API tokens
# FINANCIAL_INFORMATION: credit cards, bank accounts
# PERSONAL_HEALTH_INFORMATION: medical records
# PERSONALLY_IDENTIFIABLE_INFORMATION: SSN, passport, email, phone
# CUSTOM_IDENTIFIER: your own regex patterns

# Custom data identifier (find company-specific patterns)
aws macie2 create-custom-data-identifier \
  --name "EmployeeID" \
  --regex "EMP-[0-9]{6}" \
  --description "Employee ID numbers" \
  --keywords EMP EmployeeID

# Automated findings to S3
aws macie2 put-findings-publication-configuration \
  --s3-destination '{
    "bucketName": "my-macie-findings",
    "kmsKeyArn": "arn:aws:kms:us-east-1:123456789:key/abc123"
  }'
```

---

### 🟡 Q55. What is AWS Resource Access Manager (RAM)?
```bash
# RAM: share AWS resources across accounts WITHOUT creating duplicate resources
# Shared resources: subnets, Transit Gateway, Route53 Resolver rules, Prefix Lists, etc.

# Share VPC subnets with another account (most common use case)
aws ram create-resource-share \
  --name "SharedSubnets" \
  --resource-arns \
    arn:aws:ec2:us-east-1:123456789:subnet/subnet-11111111 \
    arn:aws:ec2:us-east-1:123456789:subnet/subnet-22222222 \
  --principals \
    arn:aws:organizations::123456789:organization/o-abc123 \  # whole org
    987654321                                                   # or specific account
  --allow-external-principals false

# In the member account — accept the resource share
aws ram accept-resource-share-invitation \
  --resource-share-invitation-arn arn:aws:ram:us-east-1:123456789:resource-share-invitation/abc123

# List resources shared WITH me
aws ram list-resources \
  --resource-owner OTHER-ACCOUNTS \
  --query "resources[].{Type:resourceType,ARN:arn,ShareArn:resourceShareArn}"

# Common shared resources:
# VPC Subnets:           central VPC, spoke accounts deploy into shared subnets
# Transit Gateway:       network hub, share with all accounts
# Route53 Resolver Rules: DNS forwarding across accounts
# License Manager:       software licences
# Aurora DB clusters:    share DB with multiple accounts
# Capacity Reservations: EC2 capacity reserved and shared
# AWS Glue Data Catalog: share data catalog across accounts

# RAM with Organizations (auto-accept)
aws ram enable-sharing-with-aws-organization
```

---

### 🟡 Q56. What is AWS Service Catalog?
```bash
# Service Catalog: create and manage approved product portfolios
# Enables: self-service provisioning of approved AWS resources

# Create portfolio
aws servicecatalog create-portfolio \
  --display-name "Approved Data Science Tools" \
  --description "Pre-approved ML and data products" \
  --provider-name "Cloud Platform Team" \
  --tags Key=Department,Value=CloudPlatform

# Create product (CloudFormation template)
aws servicecatalog create-product \
  --name "SageMaker Studio Domain" \
  --owner "Cloud Platform Team" \
  --type CLOUD_FORMATION_TEMPLATE \
  --provisioning-artifact-parameters '{
    "Name": "v1.0",
    "Description": "SageMaker Studio Domain",
    "Type": "CLOUD_FORMATION_TEMPLATE",
    "Info": {"LoadTemplateFromURL": "https://s3.amazonaws.com/my-templates/sagemaker-studio.yaml"}
  }'

# Associate product with portfolio
aws servicecatalog associate-product-with-portfolio \
  --product-id prod-abc123 \
  --portfolio-id port-xyz789

# Grant access to portfolio
aws servicecatalog associate-principal-with-portfolio \
  --portfolio-id port-xyz789 \
  --principal-arn arn:aws:iam::123456789:group/data-scientists \
  --principal-type IAM

# Provision product (end user deploys pre-approved resource)
aws servicecatalog provision-product \
  --product-id prod-abc123 \
  --provisioning-artifact-id pa-abc123 \
  --provisioned-product-name my-sagemaker-studio \
  --provisioning-parameters Key=UserProfileName,Value=alice Key=DomainName,Value=ds-domain

# AWS Service Catalog AppRegistry (application portfolio)
aws servicecatalog-appregistry create-application \
  --name "E-commerce Platform" \
  --description "All resources for e-commerce"

aws servicecatalog-appregistry associate-resource \
  --application my-ecommerce-app \
  --resource-type CFN_STACK \
  --resource my-app-cloudformation-stack
```

---

### 🟡 Q57. What is AWS License Manager?
```bash
# License Manager: manage software licences (Windows Server, SQL Server, Oracle, RHEL)
# Prevents: licence violations, tracks usage, enforces limits

# Create licence configuration
aws license-manager create-license-configuration \
  --name "SQLServer2022-Standard" \
  --license-counting-type Socket \
  --license-count 100 \
  --license-count-hard-limit \
  --description "SQL Server 2022 Standard Edition" \
  --product-information-list '[{
    "ResourceType": "SSM_MANAGED",
    "ProductInformationFilterList": [{
      "ProductInformationFilterName": "Application Name",
      "ProductInformationFilterValue": ["SQL Server"],
      "ProductInformationFilterComparator": "Contains"
    }]
  }]'

# Associate with launch template
aws license-manager update-license-specifications-for-resource \
  --resource-arn arn:aws:ec2:us-east-1:123456789:launch-template/lt-abc123 \
  --add-license-specifications LicenseConfigurationArn=arn:aws:license-manager:...

# Track licence usage
aws license-manager list-usage-for-license-configuration \
  --license-configuration-arn arn:aws:license-manager:...

# BYOL (Bring Your Own Licence) report
aws license-manager list-license-specifications-for-resource \
  --resource-arn arn:aws:ec2:us-east-1:123456789:instance/i-1234567890
```

---

### 🟢 Q58. What is AWS Tagging Strategy?
```bash
# Tags: key-value metadata on AWS resources — essential for cost, security, ops

# Mandatory tags (enforce via AWS Config or SCP):
# Environment: production | staging | development | sandbox
# Owner:       team or individual email
# Project:     business project name
# CostCenter:  finance department code
# Application: application name

# Apply tags in bulk (Tag Editor in console or CLI)
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list \
    arn:aws:ec2:us-east-1:123456789:instance/i-1111111 \
    arn:aws:ec2:us-east-1:123456789:instance/i-2222222 \
    arn:aws:rds:us-east-1:123456789:db:mydb \
  --tags Environment=production,Owner=backend-team,CostCenter=CC-2026

# Find untagged resources
aws resourcegroupstaggingapi get-resources \
  --tag-filters 'Key=Environment' \
  --query "length(ResourceTagMappingList)" \
  --output text

# Find resources missing required tags
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters ec2:instance \
  --query "ResourceTagMappingList[?!contains(Tags[].Key,'Owner')].ResourceARN"

# Resource Groups (group resources by tags for operations)
aws resource-groups create-group \
  --name "production-backend" \
  --resource-query '{
    "Type": "TAG_FILTERS_1_0",
    "Query": "{\"ResourceTypeFilters\":[\"AWS::AllSupported\"],\"TagFilters\":[{\"Key\":\"Environment\",\"Values\":[\"production\"]},{\"Key\":\"Tier\",\"Values\":[\"backend\"]}]}"
  }'

# List resources in group
aws resource-groups list-group-resources \
  --group-name production-backend

# AWS Config rule for tag compliance
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "required-tags",
    "Source": {"Owner": "AWS", "SourceIdentifier": "REQUIRED_TAGS"},
    "InputParameters": "{\"tag1Key\":\"Environment\",\"tag2Key\":\"Owner\",\"tag3Key\":\"CostCenter\"}",
    "Scope": {"ComplianceResourceTypes": ["AWS::EC2::Instance","AWS::RDS::DBInstance","AWS::S3::Bucket"]}
  }'
```

---

### 🟡 Q59. What is AWS Compute Optimizer?
```bash
# Compute Optimizer: ML-based rightsizing recommendations
# Analyses: CloudWatch metrics for 14 days → recommends optimal size

# Enable Compute Optimizer
aws compute-optimizer update-enrollment-status \
  --status Active

# For Organization (all accounts)
aws compute-optimizer update-enrollment-status \
  --status Active \
  --include-member-accounts

# Get EC2 recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --instance-arns arn:aws:ec2:us-east-1:123456789:instance/i-1234567890 \
  --query "instanceRecommendations[].{
    Current:currentInstanceType,
    Recommended:recommendationOptions[0].instanceType,
    Savings:recommendationOptions[0].estimatedMonthlySavings.value,
    PerfRisk:recommendationOptions[0].performanceRisk
  }"

# ECS Fargate recommendations (right-size task CPU/memory)
aws compute-optimizer get-ecs-service-recommendations \
  --service-arns arn:aws:ecs:us-east-1:123456789:service/my-cluster/my-service \
  --query "ecsServiceRecommendations[].{
    Current:currentServiceConfiguration,
    Recommended:serviceRecommendationOptions[0].containerRecommendations
  }"

# Lambda recommendations
aws compute-optimizer get-lambda-function-recommendations \
  --function-arns arn:aws:lambda:us-east-1:123456789:function:my-function \
  --query "lambdaFunctionRecommendations[].{
    MemorySize:functionVersion,
    Recommended:memorySizeRecommendationOptions[0].memorySize,
    Savings:memorySizeRecommendationOptions[0].projectedUtilizationMetrics
  }"

# EBS recommendations
aws compute-optimizer get-ebs-volume-recommendations \
  --volume-arns arn:aws:ec2:us-east-1:123456789:volume/vol-12345678 \
  --query "volumeRecommendations[].{
    CurrentType:currentConfiguration.volumeType,
    RecommendedType:volumeRecommendationOptions[0].configuration.volumeType,
    Savings:volumeRecommendationOptions[0].estimatedMonthlySavings.value
  }"
```

---

### 🟡 Q60. What are AWS Service Quotas and Service Limits?
```bash
# Service Quotas: manage AWS service limits per account/region

# View current quotas
aws service-quotas list-service-quotas \
  --service-code ec2 \
  --query "Quotas[?Adjustable==\`true\`].[QuotaName,Value]" \
  --output table

# Check quota usage
aws service-quotas get-service-quota \
  --service-code ec2 \
  --quota-code L-1216C47A \   # Running On-Demand Standard Instances
  --query "Quota.{Name:QuotaName,Limit:Value,Adjustable:Adjustable}"

# Request quota increase
aws service-quotas request-service-quota-increase \
  --service-code ec2 \
  --quota-code L-1216C47A \
  --desired-value 500

# Track request status
aws service-quotas get-requested-service-quota-change \
  --request-id abc123

# Proactive quota monitoring
aws service-quotas put-service-quota-increase-request-into-template \
  --service-code ec2 \
  --quota-code L-1216C47A \
  --aws-region us-east-1 \
  --desired-value 500

# CloudWatch alarm on quota usage (for critical services)
aws cloudwatch put-metric-alarm \
  --alarm-name "EC2InstanceLimitWarning" \
  --metric-name ResourceCount \
  --namespace AWS/Usage \
  --dimensions \
    Name=Type,Value=Resource \
    Name=Resource,Value=vCPU \
    Name=Service,Value=EC2 \
    Name=Class,Value=Standard/OnDemand \
  --statistic Maximum \
  --period 300 \
  --threshold 400 \     # alert at 80% of 500 vCPU limit
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:quota-alerts

# Key limits to monitor:
# EC2: vCPU limits per instance family
# VPC: VPCs per region (5), subnets per VPC (200), SGs per VPC (500)
# Lambda: concurrent executions (1,000), functions per region (75,000)
# ECS: tasks per cluster (5,000), services per cluster (2,000)
# EKS: clusters per region (100), node groups per cluster (30)
```

---

## COMPLETE Q&A INDEX

| Q# | Question | Level | Part |
|----|---------|-------|------|
| **CONTAINERS (Q1–Q20)** | | | |
| Q1 | What is Amazon ECS — EC2 vs Fargate | 🟢 | Containers |
| Q2 | ECS Task Definition — containers, secrets, health checks, logging | 🟢 | Containers |
| Q3 | ECS Service — create, update, scale, events | 🟢 | Containers |
| Q4 | ECS Auto Scaling — target tracking, step, scheduled | 🟡 | Containers |
| Q5 | ECS deployment strategies — rolling, blue/green, circuit breaker | 🟡 | Containers |
| Q6 | ECS Exec — interactive shell, SSM Session Manager | 🟡 | Containers |
| Q7 | Amazon ECR — lifecycle policies, scanning, pull-through cache, cross-account | 🟢 | Containers |
| Q8 | Amazon EKS — create cluster, node groups, Spot, EKS Auto Mode | 🟢 | Containers |
| Q9 | EKS add-ons — VPC CNI, EBS CSI, ALB controller, GuardDuty | 🟡 | Containers |
| Q10 | EKS Pod Identity — IAM roles for pods without OIDC | 🟡 | Containers |
| Q11 | Cluster Autoscaler vs Karpenter — NodePool, EC2NodeClass | 🟡 | Containers |
| Q12 | AWS Fargate — task sizes, Fargate Spot, EKS profiles | 🟡 | Containers |
| Q13 | ECS Anywhere and EKS Anywhere — on-prem containers | 🟡 | Containers |
| Q14 | AWS App Runner — simplest container hosting, VPC connector | 🟢 | Containers |
| Q15 | EKS production best practices — multi-AZ, encryption, network policies, OPA | 🔴 | Containers |
| Q16 | ECS vs EKS — when to choose each | 🟢 | Containers |
| Q17 | AWS Copilot CLI — svc init, deploy, worker service, scheduled job | 🟡 | Containers |
| Q18 | Container networking modes — awsvpc, bridge, host, Service Discovery | 🟡 | Containers |
| Q19 | ECS Capacity Providers — Fargate, Fargate Spot, ASG CP | 🟡 | Containers |
| Q20 | ECS IAM roles — execution role vs task role vs service-linked | 🟢 | Containers |
| **DEVELOPER TOOLS (Q21–Q40)** | | | |
| Q21 | AWS CodeCommit — repos, PRs, approval rules, triggers | 🟢 | DevTools |
| Q22 | AWS CodeBuild — project, buildspec.yml phases, ECR push, reports | 🟢 | DevTools |
| Q23 | AWS CodeDeploy — ECS B/G, EC2 rolling, Lambda canary, appspec | 🟢 | DevTools |
| Q24 | AWS CodePipeline — source/build/test/approve/deploy stages, V2 | 🟢 | DevTools |
| Q25 | CodeStar Connections — link GitHub/GitLab/Bitbucket to AWS | 🟢 | DevTools |
| Q26 | AWS CDK — Python example, ECS Fargate pattern, auto scaling | 🟡 | DevTools |
| Q27 | AWS CloudFormation — create/update, change sets, SAM, mappings, conditions | 🟡 | DevTools |
| Q28 | AWS SAM — template, Lambda functions, API Gateway, DynamoDB, SQS | 🟡 | DevTools |
| Q29 | AWS X-Ray — tracing, subsegments, ECS sidecar, ADOT, Service Lens | 🟡 | DevTools |
| Q30 | AWS Elastic Beanstalk — EB CLI, .ebextensions, B/G swap, Procfile | 🟡 | DevTools |
| Q31 | AWS Amplify — Auth, GraphQL API, schema, hosting, CI/CD | 🟡 | DevTools |
| Q32 | AWS CodeArtifact — npm, PyPI, Maven, NuGet, upstream proxy | 🟡 | DevTools |
| Q33 | AWS CodeGuru — Reviewer (AI code review), Profiler (flame charts) | 🟡 | DevTools |
| Q34 | AWS Cloud9 — browser IDE, SSM connect, pair programming | 🟢 | DevTools |
| Q35 | Amazon Q Developer — code generation, security scan, /transform | 🟢 | DevTools |
| Q36 | AWS Systems Manager — Session Manager, Parameter Store, Run Command, Patch | 🟡 | DevTools |
| Q37 | AWS Secrets Manager — create, rotate, cross-account, Python retrieval | 🟡 | DevTools |
| Q38 | AWS CloudShell — pre-auth shell, persistent storage, included tools | 🟢 | DevTools |
| Q39 | AWS CloudWatch — custom metrics, alarms, Log Insights, EMF, Synthetics | 🟡 | DevTools |
| Q40 | AWS EventBridge — event bus, rules, targets, Pipes, Scheduler | 🟡 | DevTools |
| **MANAGEMENT & GOVERNANCE (Q41–Q60)** | | | |
| Q41 | AWS Organizations — create org, OUs, accounts, delegated admin | 🟢 | Mgmt |
| Q42 | Service Control Policies (SCPs) — deny CloudTrail deletion, region restriction | 🟡 | Mgmt |
| Q43 | AWS Control Tower — landing zone, guardrails, Account Factory | 🟡 | Mgmt |
| Q44 | AWS CloudTrail — multi-region trail, data events, Insights, Lake SQL | 🟢 | Mgmt |
| Q45 | AWS Config — recording, managed rules, custom rules, Aggregator, Conformance Packs | 🟢 | Mgmt |
| Q46 | AWS IAM — users, roles, policies, permission boundaries, MFA enforcement, simulate | 🟡 | Mgmt |
| Q47 | AWS IAM Identity Center (SSO) — permission sets, account assignment, CLI SSO | 🟡 | Mgmt |
| Q48 | AWS Trusted Advisor — 5 categories, support tiers, suppress checks | 🟢 | Mgmt |
| Q49 | AWS Well-Architected Tool — 5+1 pillars, workloads, risk counts, custom lenses | 🟡 | Mgmt |
| Q50 | AWS Cost Explorer — budgets, anomaly detection, rightsizing, Savings Plans | 🟡 | Mgmt |
| Q51 | AWS Security Hub — standards, findings, insights, EventBridge automation | 🟡 | Mgmt |
| Q52 | Amazon GuardDuty — threat detection, finding types, suppress filters, org setup | 🟡 | Mgmt |
| Q53 | AWS Inspector v2 — EC2/ECR/Lambda scanning, enhanced scanning, suppress | 🟡 | Mgmt |
| Q54 | Amazon Macie — S3 PII detection, classification jobs, custom identifiers | 🟡 | Mgmt |
| Q55 | AWS Resource Access Manager (RAM) — share subnets, Transit GW, org sharing | 🟡 | Mgmt |
| Q56 | AWS Service Catalog — portfolios, products, AppRegistry | 🟡 | Mgmt |
| Q57 | AWS License Manager — track Windows/SQL Server/Oracle licences | 🟡 | Mgmt |
| Q58 | AWS Tagging Strategy — mandatory tags, bulk apply, Config rule enforcement | 🟢 | Mgmt |
| Q59 | AWS Compute Optimizer — EC2/ECS/Lambda/EBS rightsizing recommendations | 🟡 | Mgmt |
| Q60 | AWS Service Quotas — view, increase, proactive monitoring | 🟢 | Mgmt |

---
*Total: 60 Q&A | Containers (20) + Developer Tools (20) + Management (20) | June 2026*
*🟢 22 Basic | 🟡 35 Intermediate | 🔴 3 Advanced*
*All examples use AWS CLI v2 + real ARN patterns + production-ready configurations*

---

# GAP-FILL PART 1 — CONTAINERS (Q61–Q75)

---

### 🟡 Q61. What are ECS task placement strategies and constraints?
```bash
# Placement strategies: how tasks are distributed across container instances (EC2 launch type)
# Placement constraints: rules that must be satisfied before placing a task

# Placement strategies:
# binpack:  fill one instance before using next (minimise instances — cost)
# spread:   spread evenly across instances/AZs (availability)
# random:   place randomly

# Create service with placement strategy (EC2 launch type)
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-service \
  --task-definition my-app:1 \
  --desired-count 10 \
  --launch-type EC2 \
  --placement-strategy \
    type=spread,field=attribute:ecs.availability-zone \
    type=binpack,field=cpu \
  --placement-constraints \
    type=memberOf,expression="attribute:ecs.instance-type =~ t3.*" \
    type=distinctInstance

# Placement strategy types:
# spread by AZ then binpack by CPU:
# → distribute evenly across AZs, then fill each host by CPU

# Placement constraint types:
# distinctInstance: each task on a different instance
# memberOf: expression using cluster query language
#   attribute:ecs.instance-type == "t3.medium"
#   attribute:ecs.availability-zone in [us-east-1a, us-east-1b]
#   attribute:stack == production (custom attributes)
#   runningTasksCount < 5  (max 5 tasks per instance)

# Custom attribute on container instance
aws ecs put-attributes \
  --cluster my-cluster \
  --attributes name=stack,value=production,targetType=container-instance,targetId=<container-instance-arn>

# Then use in constraint:
# type=memberOf,expression="attribute:stack == production"

# Task spread across AZs using Fargate (awsvpc):
# Fargate automatically spreads across AZs if you specify multi-AZ subnets
```

---

### 🟡 Q62. What is ECS Service Connect?
```bash
# Service Connect: service mesh for ECS — simpler than App Mesh, no sidecar injection
# Replaces: Cloud Map service discovery for most ECS use cases
# Features: automatic service discovery, traffic metrics, load balancing, retries

# Enable Service Connect on cluster
aws ecs create-cluster \
  --cluster-name my-cluster \
  --service-connect-defaults namespace=my-app.local

# Create service with Service Connect enabled
aws ecs create-service \
  --cluster my-cluster \
  --service-name orders-service \
  --task-definition orders:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-1,subnet-2],securityGroups=[sg-1]}" \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "my-app.local",
    "services": [
      {
        "portName": "orders-http",
        "clientAliases": [{"port": 8080, "dnsName": "orders"}],
        "discoveryName": "orders",
        "ingressPortOverride": 8080
      }
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/service-connect",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "orders"
      }
    }
  }'

# Other services call: http://orders:8080/api/orders
# Service Connect handles: discovery, load balancing, circuit breaking, metrics

# Service Connect vs Cloud Map vs App Mesh:
# Service Connect: ECS-native, managed Envoy proxy, no extra infra, simple
# Cloud Map:       AWS service discovery via DNS/API, works with ECS/EC2/Lambda
# App Mesh:       Full service mesh (Envoy), multi-service/multi-account, complex
```

---

### 🟡 Q63. What are EKS Access Entries (replacement for aws-auth ConfigMap)?
```bash
# Old way: aws-auth ConfigMap — manual YAML, error-prone, no audit trail
# New way: EKS Access Entries API (2024) — IAM-native, audited via CloudTrail

# Enable access entries (new clusters use this by default)
aws eks update-cluster-config \
  --name my-eks-cluster \
  --access-config authenticationMode=API_AND_CONFIG_MAP
# Or: authenticationMode=API (fully migrate away from ConfigMap)

# Create access entry for IAM user/role
aws eks create-access-entry \
  --cluster-name my-eks-cluster \
  --principal-arn arn:aws:iam::123456789:user/developer \
  --type STANDARD \
  --kubernetes-groups developers

aws eks create-access-entry \
  --cluster-name my-eks-cluster \
  --principal-arn arn:aws:iam::123456789:role/admin-role \
  --type STANDARD

# Associate access policy (cluster admin)
aws eks associate-access-policy \
  --cluster-name my-eks-cluster \
  --principal-arn arn:aws:iam::123456789:role/admin-role \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster

# Associate namespace-scoped policy (view-only in specific namespace)
aws eks associate-access-policy \
  --cluster-name my-eks-cluster \
  --principal-arn arn:aws:iam::123456789:user/developer \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=production,staging

# Built-in access policies:
# AmazonEKSClusterAdminPolicy: cluster-admin (full access)
# AmazonEKSAdminPolicy:        admin (most resources, no cluster-level)
# AmazonEKSEditPolicy:         edit (read/write most resources)
# AmazonEKSViewPolicy:         view (read-only)

# For node groups (EC2 nodes joining cluster)
aws eks create-access-entry \
  --cluster-name my-eks-cluster \
  --principal-arn arn:aws:iam::123456789:role/eks-node-role \
  --type EC2_LINUX   # automatically assigned required node permissions

# List access entries
aws eks list-access-entries --cluster-name my-eks-cluster --output table
```

---

### 🔴 Q64. What is EKS observability with CloudWatch Container Insights and Managed Prometheus?
```bash
# Container Insights: CPU, memory, disk, network for pods/nodes/clusters
# Managed Prometheus (AMP): scrape Kubernetes metrics at scale
# Managed Grafana (AMG): visualise Prometheus + CloudWatch metrics

# Enable Container Insights with enhanced observability
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name amazon-cloudwatch-observability \
  --service-account-role-arn arn:aws:iam::123456789:role/eks-cloudwatch-role

# Creates automatically:
# CloudWatch agent DaemonSet (metrics + logs)
# Fluent Bit DaemonSet (log forwarding)
# Log groups: /aws/containerinsights/my-eks-cluster/{application,host,dataplane,performance}

# Managed Prometheus workspace
aws amp create-workspace \
  --alias my-eks-metrics \
  --logging-configuration '{"logGroupArn":"arn:aws:logs:us-east-1:123456789:log-group:/aws/prometheus:*"}'

WORKSPACE_ID=$(aws amp list-workspaces --query "workspaces[0].workspaceId" --output text)
REMOTE_WRITE_URL="https://aps-workspaces.us-east-1.amazonaws.com/workspaces/${WORKSPACE_ID}/api/v1/remote_write"

# Enable Prometheus metrics collection on EKS
aws eks update-addon \
  --cluster-name my-eks-cluster \
  --addon-name amazon-cloudwatch-observability \
  --configuration-values "{\"containerLogs\":{\"enabled\":true},\"metrics\":{\"prometheusEnabled\":true,\"prometheusRemoteWriteEndpoint\":\"${REMOTE_WRITE_URL}\"}}"

# Create Managed Grafana workspace
aws grafana create-workspace \
  --account-access-type CURRENT_ACCOUNT \
  --authentication-providers AWS_SSO \
  --permission-type SERVICE_MANAGED \
  --workspace-name my-grafana \
  --workspace-data-sources PROMETHEUS CLOUDWATCH XRAY

# Key CloudWatch Insights queries for EKS:
```

```
# CloudWatch Logs Insights — EKS performance
fields @timestamp, NodeName, pod_name, Namespace
| filter Type == "Pod" and node_memory_utilization > 80
| stats max(node_memory_utilization) as MaxMem by pod_name
| sort MaxMem desc

# Container OOMKill detection
fields @timestamp, ContainerName, Namespace, pod_name
| filter @logStream like /performance/
| filter oom_killed == 1
| stats count() as OOMCount by ContainerName, Namespace
```

---

### 🟡 Q65. What are Kubernetes HPA vs VPA vs KEDA?
```yaml
# HPA (Horizontal Pod Autoscaler): scale pod count based on CPU/memory/custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
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
        averageValue: 512Mi
  - type: External
    external:
      metric:
        name: sqs_queue_depth
        selector:
          matchLabels:
            queue: orders
      target:
        type: AverageValue
        averageValue: "30"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
---
# VPA (Vertical Pod Autoscaler): right-size CPU/memory requests per pod
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"   # Auto | Recreate | Initial | Off
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: 4
        memory: 4Gi
      controlledResources: [cpu, memory]
---
# KEDA ScaledObject: scale on external events (SQS, Kafka, Redis, etc.)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-keda
  namespace: production
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 0      # scale to zero!
  maxReplicaCount: 100
  pollingInterval: 15
  cooldownPeriod: 300
  triggers:
  - type: aws-sqs-queue
    authenticationRef:
      name: keda-aws-auth
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789/orders
      queueLength: "5"   # 1 pod per 5 messages
      awsRegion: us-east-1
      identityOwner: pod  # use pod's IAM role (IRSA/Pod Identity)
```

```bash
# HPA vs VPA vs KEDA:
# HPA:   scale pods based on CPU/memory/custom metrics — most common
# VPA:   right-size resource requests — good for variable workloads
# KEDA:  event-driven scaling — scale to zero on queues, streams, databases
# Note: HPA and VPA conflict on CPU — use KEDA + VPA or HPA alone
```

---

### 🟡 Q66. What are EKS storage classes and persistent volumes?
```bash
# Install storage add-ons
aws eks create-addon --cluster-name my-eks-cluster --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789:role/eks-ebs-csi-role
aws eks create-addon --cluster-name my-eks-cluster --addon-name aws-efs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789:role/eks-efs-csi-role
```

```yaml
# EBS gp3 StorageClass (default for most workloads)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer  # wait for pod scheduling
reclaimPolicy: Retain
allowVolumeExpansion: true
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/abc123
---
# EFS StorageClass (ReadWriteMany — shared across pods)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-abc12345
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/dynamic_provisioning"
  encrypted: "true"
---
# PVC using EBS
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 100Gi
---
# PVC using EFS (shared)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
spec:
  accessModes: [ReadWriteMany]
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

---

### 🟡 Q67. What are container image best practices?
```dockerfile
# Multi-stage build — small, secure final image
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM gcr.io/distroless/python3-debian12 AS production
# Distroless: no shell, no package manager, no extra tools — minimal attack surface
WORKDIR /app
COPY --from=builder /install /usr/local
COPY src/ .
USER 65534:65534    # nobody:nogroup — never run as root
EXPOSE 8080
ENTRYPOINT ["python", "app.py"]

# Security best practices:
# 1. Use specific image tags (not :latest) for reproducibility
# 2. Use distroless or Alpine for smallest attack surface
# 3. Never run as root (USER nonroot)
# 4. Read-only root filesystem (readOnlyRootFilesystem: true in task def)
# 5. No privileged containers
# 6. Scan with Trivy or ECR Enhanced Scanning
# 7. Sign images with AWS Signer or Notation
# 8. .dockerignore to exclude tests, .git, secrets
```

```bash
# .dockerignore
cat > .dockerignore << 'IGNORE'
.git
.gitignore
**/__pycache__
**/*.pyc
**/*.pyo
tests/
docs/
*.md
.env
.env.*
secrets/
local.settings.json
IGNORE

# Scan with Trivy before push
docker run --rm aquasec/trivy image \
  --exit-code 1 \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# Sign image with AWS Signer
aws signer put-signing-profile \
  --profile-name myECRSigningProfile \
  --signing-material certificateArn=arn:aws:acm:... \
  --platform-id Notation-OCI-SHA384-ECDSA

notation sign 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest \
  --plugin com.amazonaws.signer.notation.plugin \
  --id arn:aws:signer:us-east-1:123456789:/signing-profiles/myECRSigningProfile
```

---

### 🟡 Q68. What is AWS Fault Injection Service (FIS)?
```bash
# FIS: managed chaos engineering — inject failures to test resilience
# Target: EC2, ECS, EKS, RDS, Lambda, network, SSM

# Create experiment template
aws fis create-experiment-template \
  --description "Terminate 20% of ECS tasks" \
  --actions '{
    "terminateTasks": {
      "actionId": "aws:ecs:stop-task",
      "parameters": {"count": "20%"},
      "targets": {"Tasks": "ecsTaskTargets"}
    }
  }' \
  --targets '{
    "ecsTaskTargets": {
      "resourceType": "aws:ecs:task",
      "resourceArns": [],
      "selectionMode": "PERCENT(20)",
      "filters": [
        {"path": "cluster", "values": ["arn:aws:ecs:us-east-1:123456789:cluster/my-cluster"]},
        {"path": "service", "values": ["my-service"]}
      ]
    }
  }' \
  --stop-conditions '[{"source": "aws:cloudwatch:alarm", "value": "arn:aws:cloudwatch:us-east-1:123456789:alarm:HighErrorRate"}]' \
  --role-arn arn:aws:iam::123456789:role/fis-role

# Common FIS actions:
# aws:ec2:stop-instances             Stop EC2 instances
# aws:ec2:terminate-instances        Terminate EC2 instances
# aws:ec2:reboot-instances           Reboot EC2 instances
# aws:ecs:stop-task                  Stop ECS tasks
# aws:eks:inject-kubernetes-custom-object  Inject K8s resource
# aws:rds:failover-db-cluster        Trigger RDS Aurora failover
# aws:rds:reboot-db-instances        Reboot RDS instance
# aws:fis:inject-api-throttle-errors Inject API throttle errors
# aws:fis:inject-api-unavailable-error Inject 503 errors on any AWS API
# aws:network:disrupt-connectivity   Block network (packet loss, latency)
# aws:ssm:send-command               Run SSM stress test on instances

# Start experiment
aws fis start-experiment \
  --experiment-template-id EXT123456 \
  --tags Key=Purpose,Value=GameDay

# Monitor and stop
aws fis get-experiment --id EXP123456
aws fis stop-experiment --id EXP123456

# AWS Resilience Hub (measure resilience score)
aws resiliencehub create-app \
  --name my-ecommerce-app \
  --description "E-commerce platform" \
  --policy-arn arn:aws:resiliencehub:us-east-1:123456789:resiliency-policy/my-policy

aws resiliencehub import-resources-to-draft-app-version \
  --app-arn arn:aws:resiliencehub:us-east-1:123456789:app/abc123 \
  --terraform-sources '[{"s3StateFileUrl":"s3://my-terraform-state/prod.tfstate"}]'

aws resiliencehub start-app-assessment \
  --app-arn arn:aws:resiliencehub:us-east-1:123456789:app/abc123 \
  --app-version release \
  --assessment-name quarterly-review
```

---

# GAP-FILL PART 2 — DEVELOPER TOOLS (Q69–Q80)

---

### 🔴 Q69. What is AWS Step Functions?
```bash
# Step Functions: serverless orchestration of Lambda, ECS, DynamoDB, etc.
# Two types:
# Standard: exactly-once, long-running (up to 1 year), audit history
# Express:  at-least-once, high-throughput (up to 5 min), cheaper

# Create state machine
aws stepfunctions create-state-machine \
  --name OrderProcessingWorkflow \
  --type STANDARD \
  --role-arn arn:aws:iam::123456789:role/sfn-role \
  --definition '{
    "Comment": "Order processing workflow",
    "StartAt": "ValidateOrder",
    "States": {
      "ValidateOrder": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:123456789:function:ValidateOrder",
        "Retry": [{"ErrorEquals": ["Lambda.ServiceException"], "IntervalSeconds": 2, "MaxAttempts": 3}],
        "Catch": [{"ErrorEquals": ["ValidationError"], "Next": "InvalidOrderNotification"}],
        "Next": "ChargePayment"
      },
      "ChargePayment": {
        "Type": "Task",
        "Resource": "arn:aws:states:::dynamodb:putItem",
        "Parameters": {
          "TableName": "PaymentRecords",
          "Item": {
            "orderId": {"S.$": "$.orderId"},
            "amount": {"N.$": "States.JsonToString($.amount)"}
          }
        },
        "Next": "ProcessInParallel"
      },
      "ProcessInParallel": {
        "Type": "Parallel",
        "Branches": [
          {
            "StartAt": "UpdateInventory",
            "States": {
              "UpdateInventory": {
                "Type": "Task",
                "Resource": "arn:aws:lambda:us-east-1:123456789:function:UpdateInventory",
                "End": true
              }
            }
          },
          {
            "StartAt": "SendConfirmationEmail",
            "States": {
              "SendConfirmationEmail": {
                "Type": "Task",
                "Resource": "arn:aws:states:::sns:publish",
                "Parameters": {
                  "TopicArn": "arn:aws:sns:us-east-1:123456789:order-notifications",
                  "Message.$": "States.Format(\"Order {} confirmed\", $.orderId)"
                },
                "End": true
              }
            }
          }
        ],
        "Next": "WaitForShipment"
      },
      "WaitForShipment": {
        "Type": "Wait",
        "Seconds": 86400,
        "Next": "CheckShipmentStatus"
      },
      "CheckShipmentStatus": {
        "Type": "Choice",
        "Choices": [
          {
            "Variable": "$.shipmentStatus",
            "StringEquals": "SHIPPED",
            "Next": "OrderComplete"
          },
          {
            "Variable": "$.shipmentStatus",
            "StringEquals": "DELAYED",
            "Next": "SendDelayNotification"
          }
        ],
        "Default": "WaitForShipment"
      },
      "OrderComplete": {"Type": "Succeed"},
      "InvalidOrderNotification": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:123456789:function:NotifyInvalidOrder",
        "End": true
      },
      "SendDelayNotification": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:123456789:function:SendDelay",
        "Next": "WaitForShipment"
      }
    }
  }'

# Start execution
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456789:stateMachine:OrderProcessingWorkflow \
  --input '{"orderId": "12345", "customerId": "C001", "amount": 99.99}'

# Check status
aws stepfunctions describe-execution \
  --execution-arn arn:aws:states:us-east-1:123456789:execution:OrderProcessingWorkflow:abc123

# State types summary:
# Task:     call Lambda, ECS, DynamoDB, SNS, SQS, HTTP (SDK integrations)
# Choice:   conditional branching
# Wait:     pause (seconds, timestamp, or WaitForTaskToken)
# Parallel: run multiple branches simultaneously
# Map:      iterate over array (like forEach)
# Pass:     pass input to output (transform)
# Succeed:  success terminal
# Fail:     failure terminal
```

---

### 🟡 Q70. What is AWS Lambda Powertools?
```python
# Lambda Powertools: best practices library for Lambda (Python, TypeScript, Java, .NET)
# Pillars: Logging, Tracing, Metrics, Event Source Utilities

from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.event_handler import APIGatewayRestResolver
from aws_lambda_powertools.utilities.typing import LambdaContext
from aws_lambda_powertools.utilities.data_classes import (
    SQSEvent, S3Event, DynamoDBStreamEvent
)
from aws_lambda_powertools.utilities.batch import (
    BatchProcessor, EventType, process_partial_response
)
from aws_lambda_powertools.utilities.idempotency import (
    idempotent, DynamoDBPersistenceLayer
)

# Initialize (reads from env vars POWERTOOLS_SERVICE_NAME, POWERTOOLS_LOG_LEVEL)
logger  = Logger(service="orders-api")
tracer  = Tracer(service="orders-api")
metrics = Metrics(namespace="MyApp", service="orders-api")
app     = APIGatewayRestResolver()

# ── Structured Logging ────────────────────────────────────────────
logger.info("Processing order", extra={"orderId": "12345", "amount": 99.99})
logger.warning("High order value", extra={"orderId": "12345", "flag": "REVIEW"})
# Output: {"level":"INFO","message":"Processing order","orderId":"12345","service":"orders-api","timestamp":"..."}

# ── Tracing ───────────────────────────────────────────────────────
@tracer.capture_method
def charge_payment(order_id: str, amount: float) -> dict:
    # Auto-creates X-Ray subsegment
    tracer.put_annotation(key="order_id", value=order_id)
    tracer.put_metadata(key="payment_details", value={"amount": amount})
    return {"status": "charged", "transaction_id": "TXN-001"}

# ── Custom Metrics ────────────────────────────────────────────────
metrics.add_metric(name="OrdersProcessed", unit=MetricUnit.Count, value=1)
metrics.add_metric(name="OrderValue", unit=MetricUnit.None_, value=99.99)
metrics.add_dimension(name="Environment", value="production")
# Uses EMF — pushed to CloudWatch via stdout automatically

# ── Lambda Handler with all decorators ───────────────────────────
@app.get("/orders/<order_id>")
@tracer.capture_method
def get_order(order_id: str):
    logger.append_keys(order_id=order_id)
    order = fetch_order(order_id)
    return {"statusCode": 200, "body": order}

@app.post("/orders")
@tracer.capture_method
def create_order():
    body = app.current_event.json_body
    order = process_order(body)
    metrics.add_metric(name="OrdersCreated", unit=MetricUnit.Count, value=1)
    return {"statusCode": 201, "body": order}

@logger.inject_lambda_context(correlation_id_path="requestContext.requestId")
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True)
def lambda_handler(event: dict, context: LambdaContext):
    return app.resolve(event, context)

# ── Idempotency (prevent duplicate processing) ────────────────────
persistence_layer = DynamoDBPersistenceLayer(table_name="IdempotencyTable")

@idempotent(persistence_store=persistence_layer)
def process_payment(event: dict, context: LambdaContext) -> dict:
    # Safe to retry — idempotency key derived from event
    charge_card(event["card_token"], event["amount"])
    return {"status": "charged"}

# ── SQS Batch Processing with partial failures ────────────────────
processor = BatchProcessor(event_type=EventType.SQS)

def record_handler(record):
    payload = json.loads(record.body)
    process_order(payload["orderId"])    # if this raises, record fails gracefully

@logger.inject_lambda_context
def sqs_handler(event: SQSEvent, context: LambdaContext):
    return process_partial_response(
        event=event,
        record_handler=record_handler,
        processor=processor,
        context=context
    )
# Returns: {"batchItemFailures": [...]} — only failed items returned to SQS
```

---

### 🟡 Q71. What is CloudFormation StackSets?
```bash
# StackSets: deploy CloudFormation stacks to multiple accounts/regions simultaneously
# Use cases: security baseline, CloudTrail, Config, GuardDuty across all accounts

# Create StackSet (organisation-managed)
aws cloudformation create-stack-set \
  --stack-set-name SecurityBaseline \
  --description "Security baseline for all accounts" \
  --template-url https://s3.amazonaws.com/my-templates/security-baseline.yaml \
  --parameters \
    ParameterKey=AlertEmail,ParameterValue=security@mycompany.com \
    ParameterKey=CloudTrailBucket,ParameterValue=my-cloudtrail-bucket \
  --capabilities CAPABILITY_NAMED_IAM \
  --permission-model SERVICE_MANAGED \     # AWS manages role creation
  --auto-deployment '{"Enabled":true,"RetainStacksOnAccountRemoval":false}'

# Deploy to all accounts in specific OUs
aws cloudformation create-stack-instances \
  --stack-set-name SecurityBaseline \
  --deployment-targets '{"OrganizationalUnitIds":["ou-abc123-def456","ou-abc123-ghi789"]}' \
  --regions us-east-1 eu-west-1 ap-southeast-1 \
  --operation-preferences \
    FailureTolerancePercentage=20 \
    MaxConcurrentPercentage=50 \
    RegionConcurrencyType=PARALLEL

# Update all instances (rolling update)
aws cloudformation update-stack-set \
  --stack-set-name SecurityBaseline \
  --template-url https://s3.amazonaws.com/my-templates/security-baseline-v2.yaml \
  --operation-preferences \
    FailureTolerancePercentage=10 \
    MaxConcurrentPercentage=25

# Check operation status
aws cloudformation list-stack-set-operations \
  --stack-set-name SecurityBaseline \
  --query "Summaries[0].{Status:Status,Completed:OperationPreferences}" \
  --output table

# Delete specific instances
aws cloudformation delete-stack-instances \
  --stack-set-name SecurityBaseline \
  --deployment-targets OrganizationalUnitIds=ou-abc123-def456 \
  --regions us-east-1 \
  --no-retain-stacks \
  --operation-preferences FailureTolerancePercentage=0
```

---

### 🟡 Q72. What is CloudFormation drift detection?
```bash
# Drift: difference between actual resource state and CloudFormation template state
# Causes: manual console changes, AWS auto-updates, external automation

# Detect drift on stack
aws cloudformation detect-stack-drift \
  --stack-name my-app-stack

# Get drift detection status
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id abc123

# List drifted resources
aws cloudformation list-stack-resource-drifts \
  --stack-name my-app-stack \
  --stack-resource-drift-status-filters MODIFIED DELETED \
  --query "StackResourceDrifts[].{Resource:LogicalResourceId,Status:StackResourceDriftStatus,Expected:ExpectedProperties,Actual:ActualProperties}"

# Drift statuses:
# DRIFTED:         resource differs from expected config
# IN_SYNC:         resource matches expected config
# NOT_CHECKED:     drift detection not yet run
# MODIFIED:        resource exists but properties changed
# DELETED:         resource was manually deleted
# ADDED:           resource was manually added (not in template)

# Remediate drift — reimport resource or update stack
# Option A: update template to match reality, then update stack
# Option B: import drifted resource back under CloudFormation control
aws cloudformation create-change-set \
  --stack-name my-app-stack \
  --change-set-name reimport-bucket \
  --change-set-type IMPORT \
  --resources-to-import '[{
    "ResourceType": "AWS::S3::Bucket",
    "LogicalResourceId": "MyBucket",
    "ResourceIdentifier": {"BucketName": "my-bucket-that-was-modified"}
  }]' \
  --template-body file://template.yaml
```

---

### 🟡 Q73. What is CloudFormation custom resources?
```python
# Custom resources: extend CloudFormation with Lambda-backed logic
# Use for: resources CFN doesn't support, external API calls, complex logic

import json, boto3, urllib3
from botocore.exceptions import ClientError

http = urllib3.PoolManager()

def lambda_handler(event, context):
    """CloudFormation custom resource handler."""
    print(f"Received event: {json.dumps(event)}")

    response_url    = event["ResponseURL"]
    stack_id        = event["StackId"]
    request_id      = event["RequestId"]
    logical_id      = event["LogicalResourceId"]
    request_type    = event["RequestType"]
    resource_props  = event["ResourceProperties"]

    # Remove ServiceToken from props (CloudFormation adds it)
    resource_props.pop("ServiceToken", None)

    physical_id = event.get("PhysicalResourceId", "custom-resource-id")
    data = {}
    status = "SUCCESS"
    reason = ""

    try:
        if request_type == "Create":
            # Your creation logic
            result = create_resource(resource_props)
            physical_id = result["id"]
            data = {"Arn": result["arn"], "Endpoint": result["endpoint"]}

        elif request_type == "Update":
            # Your update logic
            result = update_resource(physical_id, resource_props,
                                      event.get("OldResourceProperties", {}))
            data = {"Arn": result["arn"]}

        elif request_type == "Delete":
            # Your deletion logic
            delete_resource(physical_id)

    except Exception as e:
        status = "FAILED"
        reason = str(e)
        print(f"Error: {e}")

    # MUST send response to presigned S3 URL
    send_response(response_url, stack_id, request_id, logical_id,
                  physical_id, status, reason, data)

def send_response(url, stack_id, req_id, logical_id, physical_id, status, reason, data):
    body = json.dumps({
        "Status":             status,
        "Reason":             reason or f"See logs: {context.log_stream_name}",
        "PhysicalResourceId": physical_id,
        "StackId":            stack_id,
        "RequestId":          req_id,
        "LogicalResourceId":  logical_id,
        "NoEcho":             False,
        "Data":               data
    }).encode("utf-8")

    http.request("PUT", url, body=body,
                 headers={"Content-Type": "", "Content-Length": str(len(body))})
```

```yaml
# CloudFormation template using custom resource
Resources:
  MyCustomResource:
    Type: Custom::MyResourceType
    Properties:
      ServiceToken: !GetAtt CustomResourceFunction.Arn
      BucketPrefix: "my-app"
      Region: !Ref AWS::Region
      Tags:
        Environment: production

  CustomResourceFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: cfn-custom-resource
      Handler: index.lambda_handler
      Role: !GetAtt CustomResourceRole.Arn
      Code:
        S3Bucket: my-lambda-code
        S3Key: custom-resource.zip
      Runtime: python3.12
      Timeout: 300

Outputs:
  CustomEndpoint:
    Value: !GetAtt MyCustomResource.Endpoint
```

---

### 🟡 Q74. What is Terraform on AWS?
```bash
# Terraform: HashiCorp IaC tool — popular alternative to CDK/CloudFormation

# Backend in S3 (remote state with DynamoDB locking)
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/app/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/abc123"
    dynamodb_table = "terraform-state-lock"  # prevents concurrent applies
  }
}

# Create S3 + DynamoDB for state backend
aws s3api create-bucket \
  --bucket my-terraform-state \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket my-terraform-state \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket my-terraform-state \
  --server-side-encryption-configuration \
    '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms"}}]}'

aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Terraform commands
terraform init       # download providers, configure backend
terraform plan       # preview changes
terraform apply      # apply changes
terraform destroy    # destroy all resources
terraform workspace new staging    # isolated state per environment

# Terraform with OIDC (no stored AWS credentials)
# GitHub Actions → OIDC → AssumeRoleWithWebIdentity
provider "aws" {
  region = "us-east-1"
  # Credentials from environment (OIDC in CI, profile locally)
}

# CDK vs CloudFormation vs Terraform:
# CloudFormation: AWS-native, no external dependencies, free
# CDK:            CloudFormation with real code, best AWS experience
# Terraform:      multi-cloud, large ecosystem, state management complexity
```

---

### 🟡 Q75. What is CDK Pipelines (self-mutating CI/CD)?
```python
# CDK Pipelines: self-mutating pipeline that updates itself when infra code changes

from aws_cdk import (
    Stack, App, Environment,
    aws_codepipeline_actions as actions,
    pipelines
)
from constructs import Construct

class AppStage(pipelines.Stage):
    def __init__(self, scope, id, **kwargs):
        super().__init__(scope, id, **kwargs)
        # Your application stacks
        MyAppStack(self, "App")

class PipelineStack(Stack):
    def __init__(self, scope, id, **kwargs):
        super().__init__(scope, id, **kwargs)

        # CodePipeline backed pipeline
        pipeline = pipelines.CodePipeline(
            self, "Pipeline",
            pipeline_name="MyAppPipeline",
            synth=pipelines.ShellStep(
                "Synth",
                input=pipelines.CodePipelineSource.connection(
                    "myOrg/my-app",
                    "main",
                    connection_arn="arn:aws:codestar-connections:..."
                ),
                commands=[
                    "npm ci",
                    "npm run build",
                    "npx cdk synth"
                ]
            ),
            code_build_defaults=pipelines.CodeBuildOptions(
                role_policy=[iam.PolicyStatement(
                    actions=["sts:AssumeRole"],
                    resources=["arn:aws:iam::*:role/cdk-*"]
                )]
            ),
            docker_enabled_for_synth=True
        )

        # Add stages with pre/post steps
        pipeline.add_stage(
            AppStage(self, "Staging",
                     env=Environment(account="123456789", region="us-east-1")),
            pre=[
                pipelines.ShellStep("RunUnitTests",
                    commands=["npm test"])
            ],
            post=[
                pipelines.ShellStep("IntegrationTests",
                    env_from_cfn_outputs={
                        "API_URL": my_app_stack.api_url
                    },
                    commands=["pytest tests/integration/"])
            ]
        )

        pipeline.add_stage(
            AppStage(self, "Production",
                     env=Environment(account="987654321", region="us-east-1")),
            pre=[
                pipelines.ManualApprovalStep("ApproveProduction",
                    comment="Approve deployment to production")
            ]
        )

app = App()
PipelineStack(app, "PipelineStack",
              env=Environment(account="111111111", region="us-east-1"))
app.synth()
```

---

# GAP-FILL PART 3 — MANAGEMENT & GOVERNANCE (Q76–Q100)

---

### 🟡 Q76. What is IAM policy evaluation logic?
```bash
# Full policy evaluation order (most restrictive wins):
# 1. DENY in SCP           → DENY (can't override)
# 2. DENY in RCP           → DENY (can't override) [new 2024]
# 3. ALLOW in SCP          → continue
# 4. DENY in Permission Boundary → DENY
# 5. ALLOW in Permission Boundary → continue
# 6. DENY in Identity Policy → DENY
# 7. ALLOW in Identity Policy → check resource policy
# 8. DENY in Resource Policy → DENY
# 9. ALLOW in Resource Policy → ALLOW (or cross-account: need both)
# 10. DENY in Session Policy  → DENY
# 11. ALLOW in Session Policy → ALLOW

# Key rules:
# Explicit DENY anywhere = DENY (no override)
# Default = DENY (implicit deny)
# Cross-account: BOTH identity + resource policy must allow
# Same account: identity OR resource policy sufficient (with some exceptions)

# Simulate policy evaluation
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789:role/my-role \
  --action-names s3:GetObject s3:DeleteObject ec2:RunInstances iam:DeleteUser \
  --resource-arns "arn:aws:s3:::my-bucket/*" \
  --context-entries \
    ContextKeyName=aws:MultiFactorAuthPresent,ContextKeyType=boolean,ContextKeyValues=true \
    ContextKeyName=aws:RequestedRegion,ContextKeyType=string,ContextKeyValues=us-east-1 \
  --query "EvaluationResults[].{Action:EvalActionName,Decision:EvalDecision,MatchedStatement:MatchedStatements[0].SourcePolicyId}"
```

---

### 🟡 Q77. What is AWS IAM Access Analyzer?
```bash
# Access Analyzer: find resources accessible from outside your account/zone of trust
# Analyzes: S3 buckets, IAM roles, KMS keys, Lambda functions, SQS queues, Secrets Manager, SNS

# Enable Access Analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name my-org-analyzer \
  --type ORGANIZATION \   # ACCOUNT | ORGANIZATION
  --tags Key=Environment,Value=production

# List findings (externally accessible resources)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789:analyzer/my-org-analyzer \
  --filter '{"status":{"eq":["ACTIVE"]}}' \
  --query "findings[].{ResourceType:resourceType,Resource:resource,ExternalAccess:principal,Status:status}"

# Archive a finding (known-good external access)
aws accessanalyzer update-findings \
  --analyzer-arn arn:aws:access-analyzer:... \
  --ids <finding-id> \
  --status ARCHIVED

# Policy validation (check your policy before applying)
aws accessanalyzer validate-policy \
  --policy-type IDENTITY_POLICY \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    }]
  }' \
  --query "findings[].{Type:findingType,Details:findingDetails,Severity:issueCode}"

# Policy generation (generate least-privilege policy from CloudTrail logs)
aws accessanalyzer start-policy-generation \
  --policy-generation-details '{"principalArn": "arn:aws:iam::123456789:role/my-app-role"}' \
  --cloud-trail-details '{
    "trails": [{"cloudTrailArn": "arn:aws:cloudtrail:us-east-1:123456789:trail/my-trail", "allRegions": true}],
    "accessRole": "arn:aws:iam::123456789:role/access-analyzer-role",
    "startTime": "2026-05-01T00:00:00Z",
    "endTime": "2026-06-01T00:00:00Z"
  }'

# Get generated policy
aws accessanalyzer get-generated-policy \
  --job-id <job-id> \
  --include-resource-placeholders \
  --query "generatedPolicyResult.generatedPolicies[0].policy"
```

---

### 🟡 Q78. What are AWS Resource Control Policies (RCPs)?
```bash
# RCPs (2024): new policy type attached to OUs/accounts
# Restricts what RESOURCES can be accessed, regardless of who is calling
# Complements SCPs (which restrict what principals can DO)

# Key difference:
# SCP: limits what the PRINCIPAL (IAM role/user) in that account can do
# RCP: limits who can access RESOURCES in that account

# Example: prevent ANY access to S3 buckets from outside the org
aws organizations create-policy \
  --name "RestrictS3ToOrganization" \
  --type RESOURCE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "RequireOrgIdentity",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": {
          "aws:PrincipalOrgID": "o-myorgid123"
        },
        "Bool": {
          "aws:PrincipalIsAWSService": "false"
        }
      }
    }]
  }'

# Example: require TLS for all S3 access
aws organizations create-policy \
  --name "RequireS3TLS" \
  --type RESOURCE_CONTROL_POLICY \
  --content '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "DenyNonTLS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {"Bool": {"aws:SecureTransport": "false"}}
    }]
  }'

# Attach to root (all accounts in org)
aws organizations attach-policy \
  --policy-id p-rcp123 \
  --target-id r-abc1       # root ID
```

---

### 🟡 Q79. What is AWS Network Firewall?
```bash
# Network Firewall: stateful, managed network firewall for VPCs
# Use: inspect inbound/outbound traffic, block domains, detect intrusions (IPS)

# Create firewall
aws network-firewall create-firewall \
  --firewall-name my-vpc-firewall \
  --firewall-policy-arn arn:aws:network-firewall:us-east-1:123456789:firewall-policy/my-policy \
  --vpc-id vpc-12345678 \
  --subnet-mappings SubnetId=subnet-11111111 SubnetId=subnet-22222222 \
  --tags Key=Environment,Value=production

# Create firewall policy
aws network-firewall create-firewall-policy \
  --firewall-policy-name my-policy \
  --firewall-policy '{
    "StatelessDefaultActions": ["aws:forward_to_sfe"],
    "StatelessFragmentDefaultActions": ["aws:forward_to_sfe"],
    "StatefulEngineOptions": {
      "RuleOrder": "STRICT_ORDER",
      "StreamExceptionPolicy": "DROP"
    },
    "StatefulRuleGroupReferences": [
      {"ResourceArn": "arn:aws:network-firewall:us-east-1:123456789:stateful-rulegroup/block-malware"},
      {"ResourceArn": "arn:aws:network-firewall:us-east-1:aws-managed:stateful-rulegroup/AbusedLegitMalwareDomainsActionOrder", "Priority": 100}
    ]
  }'

# Create stateful rule group (block malware domains)
aws network-firewall create-rule-group \
  --rule-group-name block-malware \
  --type STATEFUL \
  --capacity 100 \
  --rule-group '{
    "RulesSource": {
      "RulesString": "alert dns $HOME_NET any -> any 53 (dns.query; content:\"malware.example.com\"; msg:\"Malware domain\"; sid:1000001; rev:1;)\ndrop dns $HOME_NET any -> any 53 (dns.query; content:\"badactor.com\"; msg:\"Known bad actor\"; sid:1000002; rev:1;)"
    },
    "StatefulRuleOptions": {"RuleOrder": "STRICT_ORDER"}
  }'

# Domain list rule (block categories of domains)
aws network-firewall create-rule-group \
  --rule-group-name block-bad-domains \
  --type STATEFUL --capacity 100 \
  --rule-group '{
    "RulesSource": {
      "RulesSourceList": {
        "Targets": ["badactor.com", "malware.example.com"],
        "TargetTypes": ["HTTP_HOST", "TLS_SNI"],
        "GeneratedRulesType": "DENYLIST"
      }
    }
  }'

# Route traffic through firewall (VPC routing)
# Add route: 0.0.0.0/0 → Firewall endpoint in each AZ
```

---

### 🟡 Q80. What is AWS WAF?
```bash
# WAF: web application firewall — protect HTTP/S applications
# Attach to: ALB, CloudFront, API Gateway, AppSync, Cognito

# Create WAF Web ACL
aws wafv2 create-web-acl \
  --name my-web-acl \
  --scope REGIONAL \    # REGIONAL (ALB/API GW) | CLOUDFRONT (must be us-east-1)
  --default-action Allow={} \
  --rules '[
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 10,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet",
          "ExcludedRules": []
        }
      },
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "AWSCommonRules"}
    },
    {
      "Name": "AWSManagedRulesKnownBadInputsRuleSet",
      "Priority": 20,
      "OverrideAction": {"None": {}},
      "Statement": {
        "ManagedRuleGroupStatement": {"VendorName": "AWS", "Name": "AWSManagedRulesKnownBadInputsRuleSet"}
      },
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "BadInputs"}
    },
    {
      "Name": "RateLimitRule",
      "Priority": 5,
      "Action": {"Block": {}},
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "RateLimit"}
    },
    {
      "Name": "GeoBlock",
      "Priority": 1,
      "Action": {"Block": {}},
      "Statement": {
        "GeoMatchStatement": {"CountryCodes": ["RU","CN","KP","IR"]}
      },
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "GeoBlock"}
    }
  ]' \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=my-web-acl \
  --region us-east-1

# Get Web ACL ARN
WEB_ACL_ARN=$(aws wafv2 list-web-acls --scope REGIONAL --query "WebACLs[?Name=='my-web-acl'].ARN" --output text)

# Associate with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn $WEB_ACL_ARN \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/my-alb/abc123

# AWS managed rule groups:
# AWSManagedRulesCommonRuleSet:          OWASP Top 10, common attacks
# AWSManagedRulesKnownBadInputsRuleSet:  Log4j, XXE, Spring4Shell
# AWSManagedRulesSQLiRuleSet:            SQL injection
# AWSManagedRulesLinuxRuleSet:           Linux-specific threats
# AWSManagedRulesAdminProtectionRuleSet: Admin panel protection
# AWSManagedRulesAmazonIpReputationList: Known bad IP list
# AWSManagedRulesBotControlRuleSet:      Bot detection (paid)

# Enable WAF logging
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "'$WEB_ACL_ARN'",
    "LogDestinationConfigs": ["arn:aws:firehose:us-east-1:123456789:deliverystream/waf-logs"],
    "RedactedFields": [{"SingleHeader": {"Name": "authorization"}}]
  }'
```

---

### 🟡 Q81. What is AWS Shield Standard vs Advanced?
```bash
# Shield Standard: FREE, automatic, on for all AWS customers
# Protects: against common L3/L4 DDoS attacks (SYN floods, UDP reflection)
# Coverage: CloudFront, Route 53, ELB, EC2

# Shield Advanced: $3,000/month per org (covers unlimited resources)
# Additional protections:
# ✅ L7 DDoS protection (with WAF)
# ✅ Real-time attack visibility in console
# ✅ 24/7 access to AWS Shield Response Team (SRT)
# ✅ Cost protection (credit for DDoS-related billing spikes)
# ✅ Global threat environment dashboard
# ✅ Proactive engagement (SRT contacts you when under attack)
# ✅ Application Layer (L7) DDoS detection and mitigation

# Enable Shield Advanced
aws shield create-subscription   # enables Shield Advanced org-wide

# Create protection for specific resource
aws shield create-protection \
  --name my-alb-protection \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/my-alb/abc123

# Enable proactive engagement (SRT contacts you)
aws shield update-proactive-engagement --proactive-engagement-status ENABLED

# Associate health check with protection (SRT uses this)
aws shield associate-health-check \
  --protection-id <protection-id> \
  --health-check-arn arn:aws:route53:::healthcheck/abc123

# Subscribe SRT to SNS
aws shield update-emergency-contact-settings \
  --emergency-contact-list \
    EmailAddress=security@mycompany.com,PhoneNumber=+15551234567,ContactNotes="Primary security contact"
```

---

### 🟡 Q82. What is AWS Audit Manager?
```bash
# Audit Manager: automate evidence collection for compliance frameworks
# Frameworks: SOC2, ISO 27001, NIST CSF, HIPAA, FedRAMP, PCI DSS, GDPR

# Enable Audit Manager
aws auditmanager register-account \
  --kms-key arn:aws:kms:us-east-1:123456789:key/abc123

# List available frameworks
aws auditmanager list-assessment-frameworks \
  --framework-type Standard \
  --query "frameworkMetadataList[].{Name:name,Description:description}" \
  --output table

# Create assessment
aws auditmanager create-assessment \
  --name "SOC2-2026-Q2" \
  --assessment-reports-destination destinationType=S3,destination=s3://my-audit-reports \
  --scope '{"awsAccounts": [{"id": "123456789"}], "awsServices": [{"serviceName": "S3"}, {"serviceName": "EC2"}, {"serviceName": "IAM"}]}' \
  --roles '[{"roleArn": "arn:aws:iam::123456789:role/audit-manager-role", "roleType": "PROCESS_OWNER"}]' \
  --framework-id <soc2-framework-id>

# Get assessment status
aws auditmanager get-assessment \
  --assessment-id <assessment-id> \
  --query "assessment.{Name:metadata.name,Status:metadata.status,Compliance:metadata.complianceType}"

# Evidence is automatically collected from:
# Config Rules, CloudTrail, Security Hub, GuardDuty, IAM, CloudWatch
# Manual evidence: upload screenshots, attestations

# Generate assessment report (PDF)
aws auditmanager create-assessment-report \
  --name "SOC2-Report-Q2-2026" \
  --assessment-id <assessment-id>

# Delegate control to another team member (auditor)
aws auditmanager batch-create-delegation-by-assessment \
  --assessment-id <assessment-id> \
  --create-delegation-requests '[{
    "controlSetId": "SOC2",
    "roleArn": "arn:aws:iam::123456789:role/external-auditor",
    "roleType": "RESOURCE_OWNER",
    "comment": "Delegated to external SOC2 auditor"
  }]'
```

---

### 🟡 Q83. What is AWS Security Lake?
```bash
# Security Lake: centralise security logs in OCSF (Open Cybersecurity Schema Framework)
# Automatically normalises logs from 50+ AWS sources + third-party tools
# Stores in: Amazon S3 (Parquet format in subscriber's account)

# Enable Security Lake
aws securitylake create-data-lake \
  --configurations '[{
    "region": "us-east-1",
    "encryptionConfiguration": {"kmsKeyId": "arn:aws:kms:..."},
    "lifecycleConfiguration": {
      "transitions": [{"days": 30, "storageClass": "ONEZONE_IA"}],
      "expiration": {"days": 365}
    }
  }]' \
  --meta-store-manager-role-arn arn:aws:iam::123456789:role/security-lake-role

# Add log sources (automatically collected)
aws securitylake create-aws-log-source \
  --sources '[
    {"accounts": ["123456789"], "regions": ["us-east-1"], "sourceName": "CLOUD_TRAIL_MGMT", "sourceVersion": "2.0"},
    {"accounts": ["123456789"], "regions": ["us-east-1"], "sourceName": "VPC_FLOW",         "sourceVersion": "2.0"},
    {"accounts": ["123456789"], "regions": ["us-east-1"], "sourceName": "ROUTE53",          "sourceVersion": "2.0"},
    {"accounts": ["123456789"], "regions": ["us-east-1"], "sourceName": "SH_FINDINGS",      "sourceVersion": "2.0"},
    {"accounts": ["123456789"], "regions": ["us-east-1"], "sourceName": "LAMBDA_EXECUTION", "sourceVersion": "2.0"},
    {"accounts": ["123456789"], "regions": ["us-east-1"], "sourceName": "S3_DATA",          "sourceVersion": "2.0"}
  ]'

# Create subscriber (SIEM, analytics tool)
aws securitylake create-subscriber \
  --subscriber-name Splunk \
  --sources '[{"awsLogSource": {"sourceName": "CLOUD_TRAIL_MGMT", "sourceVersion": "2.0"}}]' \
  --subscriber-identity '{
    "externalId": "splunk-external-id",
    "principal": "arn:aws:iam::splunk-account:root"
  }' \
  --access-types S3    # S3 (pull) | LAKEFORMATION (push via Kinesis)

# Query Security Lake with Athena
# Tables auto-created in AWS Glue Data Catalog
aws athena start-query-execution \
  --query-string "
    SELECT eventTime, userIdentity.arn, eventName, sourceIPAddress
    FROM amazon_security_lake_glue_db_us_east_1.amazon_security_lake_table_us_east_1_cloud_trail_mgmt_2_0
    WHERE eventName = 'ConsoleLogin'
    AND unmappedAttributes.responseElements.ConsoleLogin = 'Failure'
    AND CAST(eventTime AS TIMESTAMP) > now() - interval '24' hour
    ORDER BY eventTime DESC
    LIMIT 100
  " \
  --query-execution-context Database=amazon_security_lake_glue_db_us_east_1 \
  --result-configuration OutputLocation=s3://my-athena-results/
```

---

### 🟢 Q84. What is AWS Backup?
```bash
# AWS Backup: centralised backup across AWS services
# Supports: EC2, RDS, Aurora, DynamoDB, EFS, S3, FSx, DocumentDB, Neptune, VMware

# Create backup vault
aws backup create-backup-vault \
  --backup-vault-name my-production-vault \
  --encryption-key-arn arn:aws:kms:us-east-1:123456789:key/abc123

# Create backup plan
aws backup create-backup-plan \
  --backup-plan '{
    "BackupPlanName": "production-backup-plan",
    "Rules": [
      {
        "RuleName": "daily-backups",
        "TargetBackupVaultName": "my-production-vault",
        "ScheduleExpression": "cron(0 5 * * ? *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 180,
        "Lifecycle": {"DeleteAfterDays": 35},
        "RecoveryPointTags": {"BackupType": "Daily"}
      },
      {
        "RuleName": "monthly-backups",
        "TargetBackupVaultName": "my-production-vault",
        "ScheduleExpression": "cron(0 5 1 * ? *)",
        "Lifecycle": {"DeleteAfterDays": 365},
        "CopyActions": [{
          "DestinationBackupVaultArn": "arn:aws:backup:us-west-2:123456789:backup-vault:dr-vault",
          "Lifecycle": {"DeleteAfterDays": 365}
        }]
      }
    ]
  }'

# Assign resources to backup plan
aws backup create-backup-selection \
  --backup-plan-id <plan-id> \
  --backup-selection '{
    "SelectionName": "all-production-resources",
    "IamRoleArn": "arn:aws:iam::123456789:role/aws-backup-role",
    "ListOfTags": [
      {"ConditionType": "STRINGEQUALS", "ConditionKey": "Environment", "ConditionValue": "production"},
      {"ConditionType": "STRINGEQUALS", "ConditionKey": "Backup", "ConditionValue": "true"}
    ]
  }'

# Cross-account backup (send to DR account)
aws backup create-backup-vault \
  --backup-vault-name cross-account-vault \
  --backup-vault-tags BackupType=CrossAccount

# Vault access policy (allow source account to put recovery points)
aws backup put-backup-vault-access-policy \
  --backup-vault-name cross-account-vault \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789:root"},
      "Action": ["backup:CopyIntoBackupVault"],
      "Resource": "*"
    }]
  }'

# List recovery points
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name my-production-vault \
  --query "RecoveryPoints[?Status=='COMPLETED'].{Resource:ResourceArn,Created:CreationDate,Expiry:CalculatedLifecycle.DeleteAt}" \
  --output table
```

---

### 🟡 Q85. What are AWS Disaster Recovery strategies?
```bash
# 4 DR strategies (RTO/RPO from slowest-cheapest to fastest-expensive):

# 1. BACKUP AND RESTORE (RTO: hours, RPO: hours)
#    Store backups in S3/Glacier, restore when needed
#    Cost: $ (cheapest — just storage)
aws backup start-backup-job \
  --backup-vault-name my-dr-vault \
  --resource-arn arn:aws:rds:us-east-1:123456789:db:mydb \
  --iam-role-arn arn:aws:iam::123456789:role/aws-backup-role

# 2. PILOT LIGHT (RTO: 10s mins, RPO: minutes)
#    Core infrastructure running at minimum capacity in DR region
#    Scale up only when disaster occurs
# DR region: RDS read replica running, EC2 AMIs ready, infrastructure defined
aws rds create-db-instance-read-replica \
  --db-instance-identifier mydb-dr-replica \
  --source-db-instance-identifier arn:aws:rds:us-east-1:123456789:db:mydb \
  --db-instance-class db.t3.medium \
  --region us-west-2

# Failover to DR (promote replica, update DNS)
aws rds promote-read-replica \
  --db-instance-identifier mydb-dr-replica \
  --region us-west-2

# 3. WARM STANDBY (RTO: minutes, RPO: seconds)
#    Scaled-down version running in DR — scale up on failover
#    Traffic can be routed immediately (small capacity)
# Route 53 health check → failover routing
aws route53 create-health-check \
  --caller-reference "primary-hc-$(date +%s)" \
  --health-check-config Type=HTTP,FullyQualifiedDomainName=my-app.us-east-1.example.com,Port=80,ResourcePath=/health

aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "my-app.example.com",
          "Type": "CNAME",
          "Failover": "PRIMARY",
          "HealthCheckId": "primary-hc-id",
          "TTL": 30,
          "ResourceRecords": [{"Value": "my-app.us-east-1.example.com"}]
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "my-app.example.com",
          "Type": "CNAME",
          "Failover": "SECONDARY",
          "TTL": 30,
          "ResourceRecords": [{"Value": "my-app.us-west-2.example.com"}]
        }
      }
    ]
  }'

# 4. MULTI-SITE ACTIVE/ACTIVE (RTO: ~0, RPO: ~0)
#    Full capacity in 2+ regions, traffic load balanced
#    Most expensive — used for critical applications
# Route 53 weighted routing or latency-based routing
# DynamoDB Global Tables, Aurora Global Database, S3 Cross-Region Replication

# DR strategy comparison:
# Backup/Restore: low cost, high RTO/RPO — good for dev/test
# Pilot Light:    moderate cost, medium RTO — good for non-critical prod
# Warm Standby:   higher cost, low RTO — good for business-critical
# Multi-site A/A: highest cost, zero RTO — good for mission-critical

# Test DR regularly with FIS (Fault Injection Service)
aws fis create-experiment-template \
  --description "DR test — terminate primary region resources" \
  --actions '{"failover":{"actionId":"aws:rds:failover-db-cluster","targets":{"Clusters":"primaryAurora"}}}'
```

---

## FINAL COMPLETE INDEX (All 125 Q&A)

| Q# | Question | Level | Part |
|----|---------|-------|------|
| **CONTAINERS (Q1–Q68)** | | | |
| Q1–Q20 | (Original 20 questions) | 🟢🟡🔴 | Containers |
| Q61 | ECS task placement strategies and constraints | 🟡 | Containers |
| Q62 | ECS Service Connect vs Cloud Map | 🟡 | Containers |
| Q63 | EKS Access Entries — replacement for aws-auth ConfigMap | 🟡 | Containers |
| Q64 | EKS observability — Container Insights, AMP, AMG | 🔴 | Containers |
| Q65 | Kubernetes HPA vs VPA vs KEDA — YAML examples | 🟡 | Containers |
| Q66 | EKS storage classes — EBS gp3, EFS, PVCs | 🟡 | Containers |
| Q67 | Container image best practices — distroless, signing, scanning | 🟡 | Containers |
| Q68 | AWS Fault Injection Service — chaos engineering, Resilience Hub | 🟡 | Containers |
| **DEVELOPER TOOLS (Q21–Q75)** | | | |
| Q21–Q40 | (Original 20 questions) | 🟢🟡 | DevTools |
| Q69 | AWS Step Functions — state types, Standard vs Express | 🔴 | DevTools |
| Q70 | AWS Lambda Powertools — Logger, Tracer, Metrics, Idempotency, Batch | 🟡 | DevTools |
| Q71 | CloudFormation StackSets — org-managed, deploy across regions | 🟡 | DevTools |
| Q72 | CloudFormation drift detection — detect/remediate/import | 🟡 | DevTools |
| Q73 | CloudFormation custom resources — Lambda-backed, send response | 🟡 | DevTools |
| Q74 | Terraform on AWS — S3 backend, DynamoDB locking, OIDC | 🟡 | DevTools |
| Q75 | CDK Pipelines — self-mutating, multi-stage, cross-account | 🔴 | DevTools |
| **MANAGEMENT (Q41–Q85)** | | | |
| Q41–Q60 | (Original 20 questions) | 🟢🟡 | Mgmt |
| Q76 | IAM policy evaluation logic — full order, simulate | 🟡 | Mgmt |
| Q77 | AWS IAM Access Analyzer — findings, policy validation, generate | 🟡 | Mgmt |
| Q78 | Resource Control Policies (RCPs) — restrict resource access org-wide | 🟡 | Mgmt |
| Q79 | AWS Network Firewall — stateful rules, domain lists, Suricata | 🟡 | Mgmt |
| Q80 | AWS WAF — managed rule groups, rate limit, geo-block, logging | 🟡 | Mgmt |
| Q81 | AWS Shield Standard vs Advanced — SRT, cost protection | 🟢 | Mgmt |
| Q82 | AWS Audit Manager — frameworks, evidence collection, SOC2 | 🟡 | Mgmt |
| Q83 | AWS Security Lake — OCSF, log sources, Athena queries | 🟡 | Mgmt |
| Q84 | AWS Backup — vault, plan, cross-account, cross-region copy | 🟡 | Mgmt |
| Q85 | DR strategies — Backup/Restore, Pilot Light, Warm Standby, Active/Active | 🟡 | Mgmt |

---
*Total: 125 Q&A | Containers (28) + Developer Tools (35) + Management (35) | June 2026*
*🟢 22 Basic | 🟡 90 Intermediate | 🔴 13 Advanced*
