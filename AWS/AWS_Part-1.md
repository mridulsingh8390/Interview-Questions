# AWS Interview Questions — Complete Guide
> **Compute | Networking | Storage | Databases | June 2026 | AWS Docs Aligned**
> 🟢 Basic | 🟡 Intermediate | 🔴 Advanced | Full CLI + Code examples

---

## TABLE OF CONTENTS
| Part | Category | Questions |
|------|---------|----------|
| [1](#part-1--compute) | Compute | Q1–Q45 |
| [2](#part-2--networking) | Networking | Q46–Q85 |
| [3](#part-3--storage) | Storage | Q86–Q115 |
| [4](#part-4--databases) | Databases | Q116–Q150 |

---

# PART 1 — COMPUTE

---

### 🟢 Q1. What is Amazon EC2 and what are the instance families?
**Answer:** EC2 (Elastic Compute Cloud) is AWS's IaaS service — virtual servers with full OS control.

| Family | Optimised For | Examples | Use Case |
|--------|-------------|---------|---------|
| **General Purpose** | Balanced CPU/memory | t3, t4g, m6i, m7g | Web servers, app servers |
| **Compute Optimised** | High vCPU | c6i, c7g, c7n | Batch, HPC, gaming |
| **Memory Optimised** | Large RAM | r6i, r7g, x2idn, u-* | In-memory DB, SAP HANA |
| **Storage Optimised** | NVMe I/O | i3, i4i, d3, h1 | NoSQL, data warehousing |
| **Accelerated (GPU)** | GPU/FPGA | p4, p5, g5, inf2, trn1 | ML training, rendering |
| **HPC Optimised** | InfiniBand fabric | hpc6a, hpc7g | Tightly-coupled MPI |

```bash
# Launch EC2 instance
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name my-key-pair \
  --security-group-ids sg-12345678 \
  --subnet-id subnet-12345678 \
  --iam-instance-profile Name=MyEC2Role \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyServer},{Key=Env,Value=prod}]' \
  --user-data '#!/bin/bash
yum update -y && yum install -y httpd
systemctl enable httpd && systemctl start httpd' \
  --count 1

# List running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,PrivateIpAddress,PublicIpAddress,State.Name]' \
  --output table

# Stop / Start / Terminate
aws ec2 stop-instances      --instance-ids i-1234567890abcdef0
aws ec2 start-instances     --instance-ids i-1234567890abcdef0
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0

# Resize instance
aws ec2 stop-instances --instance-ids i-1234567890abcdef0
aws ec2 wait instance-stopped --instance-ids i-1234567890abcdef0
aws ec2 modify-instance-attribute \
  --instance-id i-1234567890abcdef0 \
  --instance-type '{"Value":"m6i.large"}'
aws ec2 start-instances --instance-ids i-1234567890abcdef0
```

---

### 🟢 Q2. What is the difference between Stop, Hibernate, and Terminate?
| Action | State | RAM | EBS Root | Billed | Public IP |
|--------|-------|-----|---------|--------|----------|
| **Stop** | Stopped | Lost | Kept | EBS only | Released (unless Elastic IP) |
| **Hibernate** | Stopped | Saved to EBS | Kept | EBS only | Released (unless Elastic IP) |
| **Terminate** | Terminated | Lost | Deleted (default) | Nothing | Released |

```bash
# Enable hibernation (must be set at launch)
aws ec2 run-instances \
  --instance-type m5.large \
  --hibernation-options Configured=true \
  --block-device-mappings '[{
    "DeviceName": "/dev/xvda",
    "Ebs": {"Encrypted": true, "VolumeSize": 50, "VolumeType": "gp3"}
  }]' \
  --image-id ami-12345678

# Hibernate a running instance
aws ec2 stop-instances \
  --instance-ids i-1234567890abcdef0 \
  --hibernate

# Hibernate requirements:
# - EBS-backed root volume (encrypted)
# - Instance RAM < 150 GB
# - Hibernation enabled at launch time
# - Supported instance families: C3, C4, C5, M3, M4, M5, R3, R4, R5, T2, T3
```

---

### 🟢 Q3. What are EC2 purchasing options?
| Option | Discount vs On-Demand | Commitment | Best For |
|--------|---------------------|-----------|---------|
| **On-Demand** | 0% | None | Unpredictable, short-term |
| **Reserved (Standard)** | Up to 72% | 1 or 3 years | Steady-state, known workloads |
| **Reserved (Convertible)** | Up to 54% | 1 or 3 years | Flexible instance family/size |
| **Compute Savings Plans** | Up to 66% | 1 or 3 years | Flexible across EC2, Lambda, Fargate |
| **EC2 Instance Savings Plans** | Up to 72% | 1 or 3 years | Specific region + family |
| **Spot** | Up to 90% | None | Fault-tolerant batch, stateless |
| **Dedicated Host** | Variable | On-demand/reserved | BYOL, compliance |
| **Dedicated Instance** | ~10% premium | None | Hardware isolation |
| **Capacity Reservation** | 0% | None | Guaranteed AZ capacity |

```bash
# Purchase 5 Reserved Instances
aws ec2 describe-reserved-instances-offerings \
  --instance-type m6i.large \
  --product-description "Linux/UNIX" \
  --offering-type "Partial Upfront" \
  --duration 31536000 \
  --query 'ReservedInstancesOfferings[].[ReservedInstancesOfferingId,FixedPrice,UsagePrice]' \
  --output table

aws ec2 purchase-reserved-instances-offering \
  --reserved-instances-offering-id <offering-id> \
  --instance-count 5

# Request Spot instance
aws ec2 request-spot-instances \
  --instance-count 1 \
  --type persistent \
  --launch-specification '{
    "ImageId": "ami-12345678",
    "InstanceType": "m5.large",
    "KeyName": "my-key",
    "SecurityGroupIds": ["sg-12345678"],
    "SubnetId": "subnet-12345678",
    "IamInstanceProfile": {"Name": "MyEC2Role"}
  }' \
  --spot-price "0.05"

# Spot Fleet — diversify across instance types for availability
aws ec2 request-spot-fleet \
  --spot-fleet-request-config '{
    "IamFleetRole": "arn:aws:iam::123456789012:role/AmazonEC2SpotFleetRole",
    "TargetCapacity": 20,
    "OnDemandTargetCapacity": 4,
    "AllocationStrategy": "priceCapacityOptimized",
    "LaunchSpecifications": [
      {"InstanceType":"m5.large","ImageId":"ami-12345678","SubnetId":"subnet-11111111"},
      {"InstanceType":"m5a.large","ImageId":"ami-12345678","SubnetId":"subnet-22222222"},
      {"InstanceType":"m6i.large","ImageId":"ami-12345678","SubnetId":"subnet-33333333"}
    ]
  }'

# Capacity Reservation (ensure capacity without commitment)
aws ec2 create-capacity-reservation \
  --instance-type m6i.large \
  --instance-platform Linux/UNIX \
  --availability-zone us-east-1a \
  --instance-count 10 \
  --instance-match-criteria targeted \
  --end-date-type limited \
  --end-date 2026-12-31T00:00:00Z
```

---

### 🟢 Q4. What are AMIs and how do you manage them?
```bash
# AMI = Amazon Machine Image: OS + config template for EC2 instances
# Types:
# EBS-backed:            root on EBS (supports Stop/Hibernate/Snapshot)
# Instance store-backed: root on ephemeral storage (lost on stop/terminate)

# Create AMI from running or stopped instance
aws ec2 create-image \
  --instance-id i-1234567890abcdef0 \
  --name "golden-image-v2.1-$(date +%Y%m%d)" \
  --description "Production baseline — June 2026" \
  --block-device-mappings '[{
    "DeviceName": "/dev/xvda",
    "Ebs": {"VolumeType": "gp3", "DeleteOnTermination": true}
  }]' \
  --no-reboot    # skip reboot (may cause filesystem inconsistency for active DBs)

# Copy AMI to another region (cross-region DR)
aws ec2 copy-image \
  --source-image-id ami-12345678 \
  --source-region us-east-1 \
  --region eu-west-1 \
  --name "golden-image-v2.1-eu-copy" \
  --encrypted \
  --kms-key-id alias/my-key

# Share AMI with another AWS account
aws ec2 modify-image-attribute \
  --image-id ami-12345678 \
  --launch-permission "Add=[{UserId=123456789012},{UserId=987654321098}]"

# Make AMI public
aws ec2 modify-image-attribute \
  --image-id ami-12345678 \
  --launch-permission "Add=[{Group=all}]"

# Find latest Amazon Linux 2023 AMI
aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text

# List your AMIs
aws ec2 describe-images \
  --owners self \
  --query 'Images | sort_by(@, &CreationDate) | reverse(@) | [].[ImageId,Name,CreationDate,State]' \
  --output table

# Deprecate old AMI (doesn't delete, stops appearing in search)
aws ec2 enable-image-deprecation \
  --image-id ami-12345678 \
  --deprecate-at 2026-12-31T00:00:00.000Z

# Deregister + delete snapshots
SNAP_IDS=$(aws ec2 describe-images --image-ids ami-12345678 \
  --query 'Images[].BlockDeviceMappings[].Ebs.SnapshotId' --output text)
aws ec2 deregister-image --image-id ami-12345678
for snap in $SNAP_IDS; do
  aws ec2 delete-snapshot --snapshot-id $snap
done
```

---

### 🟡 Q5. What is EC2 Auto Scaling and what scaling policies exist?
```bash
# Auto Scaling Group: automatically maintain desired EC2 capacity

# Create Launch Template
aws ec2 create-launch-template \
  --launch-template-name prodLT \
  --version-description "v1-prod" \
  --launch-template-data '{
    "ImageId": "ami-12345678",
    "InstanceType": "t3.medium",
    "KeyName": "my-key",
    "SecurityGroupIds": ["sg-12345678"],
    "IamInstanceProfile": {"Name": "MyEC2Role"},
    "MetadataOptions": {"HttpTokens": "required"},
    "Monitoring": {"Enabled": true},
    "BlockDeviceMappings": [{
      "DeviceName": "/dev/xvda",
      "Ebs": {"VolumeSize": 30, "VolumeType": "gp3", "Encrypted": true}
    }],
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{"Key": "Env", "Value": "prod"}]
    }]
  }'

# Create Auto Scaling Group (across 3 AZs)
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name myASG \
  --launch-template LaunchTemplateId=lt-12345678,Version='$Latest' \
  --min-size 2 --max-size 20 --desired-capacity 4 \
  --vpc-zone-identifier "subnet-11111111,subnet-22222222,subnet-33333333" \
  --target-group-arns arn:aws:elasticloadbalancing:...:targetgroup/myTG/abc \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --default-instance-warmup 60 \
  --tags 'Key=Name,Value=asg-instance,PropagateAtLaunch=true'

# POLICY 1: Target Tracking (recommended — maintain metric at target value)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name myASG \
  --policy-name cpu-target-60 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ASGAverageCPUUtilization"},
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60,
    "DisableScaleIn": false
  }'

# Custom metric target tracking (ALB requests per target)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name myASG \
  --policy-name alb-req-target \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "TargetValue": 1000,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "app/myALB/abc123/targetgroup/myTG/def456"
    }
  }'

# POLICY 2: Step Scaling (scale in steps based on alarm breach size)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name myASG \
  --policy-name step-scale-out \
  --policy-type StepScaling \
  --adjustment-type ChangeInCapacity \
  --step-adjustments '[
    {"MetricIntervalLowerBound":0, "MetricIntervalUpperBound":20, "ScalingAdjustment":1},
    {"MetricIntervalLowerBound":20,"MetricIntervalUpperBound":40, "ScalingAdjustment":2},
    {"MetricIntervalLowerBound":40,"ScalingAdjustment":4}
  ]'

# POLICY 3: Scheduled Scaling (known traffic patterns)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name myASG \
  --scheduled-action-name scale-up-business-hours \
  --recurrence "0 8 * * MON-FRI" \
  --time-zone "America/New_York" \
  --min-size 4 --max-size 20 --desired-capacity 8

aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name myASG \
  --scheduled-action-name scale-down-night \
  --recurrence "0 22 * * MON-FRI" \
  --min-size 2 --desired-capacity 2

# POLICY 4: Predictive Scaling (ML-based, pre-emptively scales)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name myASG \
  --policy-name predictive \
  --policy-type PredictiveScaling \
  --predictive-scaling-configuration '{
    "MetricSpecifications": [{
      "TargetValue": 60.0,
      "PredefinedMetricPairSpecification": {"PredefinedMetricType": "ASGCPUUtilization"}
    }],
    "Mode": "ForecastAndScale",
    "SchedulingBufferTime": 300,
    "MaxCapacityBreachBehavior": "IncreaseMaxCapacity"
  }'

# Warm pool (pre-initialised instances for fast scale-out)
aws autoscaling put-warm-pool \
  --auto-scaling-group-name myASG \
  --min-size 2 \
  --pool-state Stopped   # Stopped = no billing; Running = billed but instant
```

---

### 🟡 Q6. What is the EC2 Instance Metadata Service (IMDS)?
```bash
# IMDS: provides instance info at link-local address 169.254.169.254
# IMDSv1: simple GET — vulnerable to SSRF (attacker tricks app to fetch creds)
# IMDSv2: session-oriented with PUT + token — SSRF-resistant

# IMDSv2 usage from within an EC2 instance
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Instance identity
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-type
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone

# IAM temporary credentials (from instance role)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/MyEC2Role
# Returns: AccessKeyId, SecretAccessKey, Token, Expiration

# User data
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/user-data

# Instance identity document (cryptographically signed)
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/dynamic/instance-identity/document

# Enforce IMDSv2 at launch
aws ec2 run-instances \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=1 \
  --image-id ami-12345678 --instance-type t3.micro

# Enforce IMDSv2 on existing instance
aws ec2 modify-instance-metadata-options \
  --instance-id i-1234567890abcdef0 \
  --http-tokens required \
  --http-put-response-hop-limit 1

# SCP to deny launching with IMDSv1
# Deny ec2:RunInstances if ec2:MetadataHttpTokens != required
```

---

### 🟡 Q7. What are EC2 Placement Groups?
```bash
# Cluster:   all instances same rack, same AZ — highest network throughput (100Gbps)
# Spread:    each instance on different hardware — max HA (7 instances per AZ per group)
# Partition: groups of instances on separate racks — Hadoop/Kafka (up to 7 partitions/AZ)

# Create placement groups
aws ec2 create-placement-group --group-name cluster-pg   --strategy cluster
aws ec2 create-placement-group --group-name spread-pg    --strategy spread
aws ec2 create-placement-group --group-name partition-pg  --strategy partition --partition-count 7

# Cluster group — HPC, low latency networking
aws ec2 run-instances \
  --placement 'GroupName=cluster-pg' \
  --instance-type c5n.18xlarge \
  --image-id ami-12345678 --count 10

# Spread group — critical HA (primary + replicas on different hardware)
aws ec2 run-instances \
  --placement 'GroupName=spread-pg' \
  --instance-type m5.xlarge \
  --image-id ami-12345678 --count 7

# Partition group — Kafka brokers, rack-aware placement
aws ec2 run-instances \
  --placement 'GroupName=partition-pg,PartitionNumber=0' \
  --instance-type r5.2xlarge \
  --image-id ami-12345678 --count 3

# Describe partitions and their instances
aws ec2 describe-instances \
  --filters "Name=placement-group-name,Values=partition-pg" \
  --query 'Reservations[].Instances[].[InstanceId,Placement.PartitionNumber]' \
  --output table
```

---

### 🟡 Q8. What is AWS Lambda and what are its limits?
```bash
# Lambda: serverless functions — no servers to manage, pay per invocation

# Limits:
# Max execution time:  15 minutes
# Memory:              128MB – 10,240MB (10GB)
# Ephemeral storage:   512MB – 10,240MB (/tmp)
# Deployment package:  50MB zip (250MB unzipped), 10GB container image
# Concurrency:         1,000 default (soft limit, can increase)
# Payload:             6MB sync, 256KB async

# Create Lambda function
aws lambda create-function \
  --function-name myFunction \
  --runtime python3.12 \
  --role arn:aws:iam::123456789012:role/lambda-execution-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 512 \
  --ephemeral-storage Size=1024 \
  --environment Variables='{DB_HOST=mydb.rds.amazonaws.com}' \
  --vpc-config SubnetIds=subnet-11111111,subnet-22222222,SecurityGroupIds=sg-12345678 \
  --tracing-config Mode=Active \
  --architectures arm64

# Invoke synchronously
aws lambda invoke \
  --function-name myFunction \
  --payload '{"action":"process","id":"12345"}' \
  --cli-binary-format raw-in-base64-out \
  --log-type Tail \
  response.json

# Invoke asynchronously (fire and forget)
aws lambda invoke \
  --function-name myFunction \
  --invocation-type Event \
  --payload '{"action":"batch"}' \
  --cli-binary-format raw-in-base64-out \
  /dev/null

# Publish version + alias (blue-green)
aws lambda publish-version --function-name myFunction
aws lambda create-alias \
  --function-name myFunction \
  --name production \
  --function-version 7 \
  --routing-config AdditionalVersionWeights={"6":0.05}  # 5% canary to v6

# Function URL (built-in HTTPS endpoint, no API Gateway needed)
aws lambda create-function-url-config \
  --function-name myFunction \
  --auth-type AWS_IAM \
  --cors '{
    "AllowOrigins": ["https://myapp.com"],
    "AllowMethods": ["GET","POST"],
    "AllowHeaders": ["*"],
    "MaxAge": 86400
  }'

# Event Source Mappings (triggers)
# SQS trigger
aws lambda create-event-source-mapping \
  --function-name myFunction \
  --event-source-arn arn:aws:sqs:us-east-1:123456789012:myQueue \
  --batch-size 100 \
  --maximum-batching-window-in-seconds 5 \
  --function-response-types ReportBatchItemFailures \
  --bisect-batch-on-function-error true

# DynamoDB Streams trigger
aws lambda create-event-source-mapping \
  --function-name myFunction \
  --event-source-arn arn:aws:dynamodb:us-east-1:123456789012:table/myTable/stream/2026-01-01 \
  --starting-position LATEST \
  --batch-size 100 \
  --filter-criteria '{"Filters":[{"Pattern":"{\"eventName\":[\"INSERT\",\"MODIFY\"]}"}]}'

# Kinesis trigger with enhanced fan-out
aws lambda create-event-source-mapping \
  --function-name myFunction \
  --event-source-arn arn:aws:kinesis:us-east-1:123456789012:stream/myStream \
  --starting-position TRIM_HORIZON \
  --batch-size 100 \
  --parallelization-factor 10
```

```python
# Lambda handler patterns

import json, boto3, os
from typing import Any

# API Gateway HTTP trigger
def api_handler(event: dict, context: Any) -> dict:
    method = event.get("requestContext", {}).get("http", {}).get("method")
    path   = event.get("rawPath")
    body   = json.loads(event.get("body") or "{}")

    if method == "POST" and path == "/orders":
        return {
            "statusCode": 201,
            "headers": {"Content-Type": "application/json",
                        "X-Request-Id": context.aws_request_id},
            "body": json.dumps({"orderId": process_order(body)})
        }
    return {"statusCode": 404, "body": json.dumps({"error": "Not found"})}

# SQS batch trigger (partial failure support)
def sqs_handler(event: dict, context: Any) -> dict:
    failures = []
    for record in event["Records"]:
        try:
            payload = json.loads(record["body"])
            process_message(payload)
        except Exception as e:
            print(f"Failed {record['messageId']}: {e}")
            failures.append({"itemIdentifier": record["messageId"]})
    return {"batchItemFailures": failures}

# S3 event trigger
def s3_handler(event: dict, context: Any):
    s3 = boto3.client("s3")
    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key    = record["s3"]["object"]["key"]
        size   = record["s3"]["object"]["size"]
        print(f"New object: s3://{bucket}/{key} ({size} bytes)")
        obj = s3.get_object(Bucket=bucket, Key=key)
        process_file(obj["Body"].read())

# EventBridge scheduled trigger
def scheduled_handler(event: dict, context: Any):
    print(f"Scheduled event: {event['time']}")
    # Time remaining
    print(f"Time remaining: {context.get_remaining_time_in_millis()}ms")
    cleanup_old_records()

# Lambda SnapStart (Java) — pre-initialise JVM (not Python/Node)
# Use with: java21 runtime, published versions only
# Eliminates cold start for Java lambdas
```

---

### 🟡 Q9. What are Lambda concurrency, throttling, and cold starts?
```bash
# Concurrency types:
# Unreserved:   shared pool (default 1000/account)
# Reserved:     dedicated allocation; also CAPS the function
# Provisioned:  pre-warmed, eliminates cold starts

# Set reserved concurrency (caps at 100 concurrent invocations)
aws lambda put-function-concurrency \
  --function-name myFunction \
  --reserved-concurrent-executions 100

# Remove concurrency limit (back to unreserved pool)
aws lambda delete-function-concurrency --function-name myFunction

# Provisioned concurrency (pre-warmed — eliminates cold start)
aws lambda put-provisioned-concurrency-config \
  --function-name myFunction \
  --qualifier production \
  --provisioned-concurrent-executions 20

# Auto-scale provisioned concurrency
aws application-autoscaling register-scalable-target \
  --service-namespace lambda \
  --resource-id function:myFunction:production \
  --scalable-dimension lambda:function:ProvisionedConcurrency \
  --min-capacity 5 --max-capacity 200

aws application-autoscaling put-scaling-policy \
  --service-namespace lambda \
  --resource-id function:myFunction:production \
  --scalable-dimension lambda:function:ProvisionedConcurrency \
  --policy-name lambda-pc-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 0.7,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "LambdaProvisionedConcurrencyUtilization"
    }
  }'

# Cold start mitigation strategies:
# 1. Use arm64 + Python/Node (faster cold start than Java)
# 2. Reduce package size (remove unused dependencies)
# 3. Lazy imports (import inside function body, not module level)
# 4. Use Provisioned Concurrency
# 5. Lambda SnapStart for Java (pre-snapshotted JVM)
# 6. Keep functions warm with EventBridge scheduled pings

# Cold start factors:
# Runtime:      Python/Node ~100ms, Java/.NET ~1-5s
# VPC:          adds ~100-200ms (Hyperplane ENIs reduced this)
# Memory:       more memory = faster init
# Package size: smaller = faster
# Layers:       each layer adds overhead
```

---

### 🟡 Q10. What is Amazon ECS (Elastic Container Service)?
```bash
# ECS: managed container orchestration (Docker)
# Launch types:
# EC2:      you manage EC2 hosts in cluster
# Fargate:  serverless — AWS manages infrastructure, you define CPU/memory

# Create cluster
aws ecs create-cluster \
  --cluster-name myCluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=2 \
    capacityProvider=FARGATE_SPOT,weight=4 \
  --settings name=containerInsights,value=enabled

# Register task definition
aws ecs register-task-definition \
  --family myapp \
  --requires-compatibilities FARGATE \
  --network-mode awsvpc \
  --cpu 512 --memory 1024 \
  --execution-role-arn arn:aws:iam::123456789012:role/ecsTaskExecutionRole \
  --task-role-arn arn:aws:iam::123456789012:role/myAppTaskRole \
  --container-definitions '[{
    "name": "myapp",
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest",
    "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
    "environment": [{"name": "ENV", "value": "prod"}],
    "secrets": [
      {"name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:db-password-abc123"}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/myapp",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs",
        "awslogs-create-group": "true"
      }
    },
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
      "interval": 30, "timeout": 5, "retries": 3, "startPeriod": 60
    },
    "essential": true,
    "readonlyRootFilesystem": true
  }]'

# Create service
aws ecs create-service \
  --cluster myCluster \
  --service-name myapp \
  --task-definition myapp:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --platform-version LATEST \
  --network-configuration 'awsvpcConfiguration={
    subnets=["subnet-11111111","subnet-22222222"],
    securityGroups=["sg-12345678"],
    assignPublicIp=DISABLED
  }' \
  --load-balancers '[{
    "targetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/myTG/abc",
    "containerName": "myapp",
    "containerPort": 8080
  }]' \
  --deployment-configuration 'maximumPercent=200,minimumHealthyPercent=100,deploymentCircuitBreaker={enable=true,rollback=true}' \
  --enable-execute-command \
  --service-connect-configuration enabled=true,namespace=myNamespace

# ECS Exec (connect to running container without SSH)
aws ecs execute-command \
  --cluster myCluster \
  --task <task-arn> \
  --container myapp \
  --interactive \
  --command "/bin/sh"

# Rolling update
aws ecs update-service \
  --cluster myCluster \
  --service myapp \
  --task-definition myapp:2 \
  --force-new-deployment

# Auto-scale ECS service
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/myCluster/myapp \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 50

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/myCluster/myapp \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ECSServiceAverageCPUUtilization"},
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

---

### 🟡 Q11. What is Amazon EKS?
```bash
# EKS: managed Kubernetes — AWS manages control plane (free), you pay for nodes

# Create cluster
aws eks create-cluster \
  --name myEKSCluster \
  --kubernetes-version 1.30 \
  --role-arn arn:aws:iam::123456789012:role/eks-cluster-role \
  --resources-vpc-config \
    subnetIds=subnet-11111111,subnet-22222222,subnet-33333333,\
    securityGroupIds=sg-12345678,\
    endpointPublicAccess=true,\
    endpointPrivateAccess=true,\
    publicAccessCidrs=10.0.0.0/8 \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}' \
  --tags Env=prod,Project=myapp

aws eks wait cluster-active --name myEKSCluster

# Create managed node group
aws eks create-nodegroup \
  --cluster-name myEKSCluster \
  --nodegroup-name app-nodes \
  --subnets subnet-11111111 subnet-22222222 \
  --node-role arn:aws:iam::123456789012:role/eks-node-role \
  --instance-types t3.medium m5.large \
  --capacity-type ON_DEMAND \
  --scaling-config minSize=2,maxSize=20,desiredSize=4 \
  --ami-type AL2_x86_64 \
  --disk-size 50 \
  --labels Environment=production,tier=app \
  --taints 'key=dedicated,value=app,effect=NO_SCHEDULE' \
  --update-config maxUnavailable=1

# Fargate profile
aws eks create-fargate-profile \
  --cluster-name myEKSCluster \
  --fargate-profile-name fargate-profile \
  --pod-execution-role-arn arn:aws:iam::123456789012:role/eks-fargate-role \
  --subnets subnet-11111111 subnet-22222222 \
  --selectors namespace=fargate-workloads namespace=kube-system

# Get kubeconfig
aws eks update-kubeconfig --region us-east-1 --name myEKSCluster
kubectl get nodes

# Upgrade cluster and node group
aws eks update-cluster-version \
  --name myEKSCluster --kubernetes-version 1.31
aws eks update-nodegroup-version \
  --cluster-name myEKSCluster --nodegroup-name app-nodes
```

---

### 🟡 Q12. What is AWS Elastic Beanstalk?
```bash
# Elastic Beanstalk: PaaS — deploy apps; AWS manages EC2, ASG, ALB, RDS
# Platforms: Python, Node.js, Java, .NET, PHP, Ruby, Go, Docker

pip install awsebcli

# Init and create environment
eb init myapp \
  --platform "Python 3.12 running on 64bit Amazon Linux 2023" \
  --region us-east-1

eb create myapp-prod \
  --instance-type t3.medium \
  --min-instances 2 --max-instances 10 \
  --elb-type application \
  --envvars DB_HOST=mydb.rds.amazonaws.com,ENV=production

eb deploy           # deploy new version
eb status           # check health
eb logs             # tail logs
eb open             # open URL in browser
eb config           # view/edit config
eb scale 5          # set desired capacity

# .ebextensions/app-config.config
option_settings:
  aws:autoscaling:launchconfiguration:
    InstanceType: t3.medium
    IamInstanceProfile: aws-elasticbeanstalk-ec2-role
  aws:autoscaling:asg:
    MinSize: 2
    MaxSize: 10
  aws:elasticbeanstalk:environment:
    EnvironmentType: LoadBalanced
    LoadBalancerType: application
  aws:elasticbeanstalk:healthreporting:system:
    SystemType: enhanced
  aws:elasticbeanstalk:cloudwatch:logs:
    StreamLogs: true
    RetentionInDays: 30

packages:
  yum:
    git: []
    postgresql-devel: []
```

---

### 🟡 Q13. What is AWS Fargate?
```bash
# Fargate: serverless compute for ECS and EKS containers
# No EC2 instances to manage — define CPU/memory, AWS handles the rest

# Fargate CPU/Memory combinations:
# 0.25 vCPU: 0.5, 1, 2 GB
# 0.5 vCPU:  1–4 GB
# 1 vCPU:    2–8 GB
# 2 vCPU:    4–16 GB
# 4 vCPU:    8–30 GB
# 8 vCPU:    16–60 GB
# 16 vCPU:   32–120 GB

# Fargate task with persistent storage (EFS mount)
aws ecs register-task-definition \
  --family myFargateTask \
  --requires-compatibilities FARGATE \
  --network-mode awsvpc \
  --cpu 1024 --memory 2048 \
  --execution-role-arn arn:aws:iam::123456789012:role/ecsTaskExecutionRole \
  --container-definitions '[{
    "name": "myapp",
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest",
    "portMappings": [{"containerPort": 8080}],
    "mountPoints": [{"sourceVolume": "shared-data", "containerPath": "/data"}]
  }]' \
  --volumes '[{
    "name": "shared-data",
    "efsVolumeConfiguration": {
      "fileSystemId": "fs-12345678",
      "rootDirectory": "/myapp",
      "transitEncryption": "ENABLED",
      "authorizationConfig": {"accessPointId": "fsap-12345678", "iam": "ENABLED"}
    }
  }]' \
  --ephemeral-storage size=50

# Fargate Spot (70% cheaper, can be interrupted)
aws ecs create-service \
  --cluster myCluster --service-name myapp-spot \
  --task-definition myFargateTask:1 --desired-count 10 \
  --capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=2 \
    capacityProvider=FARGATE_SPOT,weight=9 \
  --network-configuration 'awsvpcConfiguration={
    subnets=["subnet-11111111","subnet-22222222"],
    securityGroups=["sg-12345678"],assignPublicIp=DISABLED
  }'
```

---

### 🟡 Q14. What is AWS Step Functions?
```bash
# Step Functions: serverless workflow orchestration (state machines)
# Standard: exactly-once, up to 1 year, pay per state transition
# Express:  at-least-once, up to 5 min, high-throughput, pay per duration

aws stepfunctions create-state-machine \
  --name OrderProcessingWorkflow \
  --type STANDARD \
  --role-arn arn:aws:iam::123456789012:role/StepFunctionsRole \
  --definition '{
    "Comment": "Order Processing",
    "StartAt": "ValidateOrder",
    "States": {
      "ValidateOrder": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ValidateOrder",
        "Next": "CheckInventory",
        "Retry": [{"ErrorEquals":["Lambda.ServiceException","Lambda.AWSLambdaException"],"IntervalSeconds":2,"MaxAttempts":3,"BackoffRate":2}],
        "Catch": [{"ErrorEquals":["ValidationError"],"Next":"OrderFailed","ResultPath":"$.error"}]
      },
      "CheckInventory": {
        "Type": "Task",
        "Resource": "arn:aws:states:::dynamodb:getItem",
        "Parameters": {
          "TableName": "Inventory",
          "Key": {"ProductId": {"S.$": "$.productId"}}
        },
        "Next": "IsInStock?"
      },
      "IsInStock?": {
        "Type": "Choice",
        "Choices": [
          {"Variable": "$.Item.Stock.N", "NumericGreaterThan": 0, "Next": "ProcessPayment"}
        ],
        "Default": "Backorder"
      },
      "ProcessPayment": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:::function:ProcessPayment",
        "Next": "FulfillOrder"
      },
      "FulfillOrder": {
        "Type": "Parallel",
        "Branches": [
          {
            "StartAt": "ShipOrder",
            "States": {"ShipOrder":{"Type":"Task","Resource":"arn:aws:lambda:::function:ShipOrder","End":true}}
          },
          {
            "StartAt": "SendEmail",
            "States": {"SendEmail":{"Type":"Task","Resource":"arn:aws:states:::sns:publish","Parameters":{"TopicArn":"arn:aws:sns:us-east-1:123456789012:orders","Message.$":"$.orderId"},"End":true}}
          }
        ],
        "Next": "Success"
      },
      "WaitForHumanApproval": {
        "Type": "Task",
        "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
        "Parameters": {
          "FunctionName": "SendApprovalEmail",
          "Payload": {"orderId.$":"$.orderId","taskToken.$":"$$.Task.Token"}
        },
        "HeartbeatSeconds": 86400,
        "Next": "ProcessPayment"
      },
      "Backorder": {"Type":"Task","Resource":"arn:aws:lambda:::function:Backorder","End":true},
      "Success":   {"Type":"Succeed"},
      "OrderFailed":{"Type":"Fail","Error":"OrderFailed","Cause":"Validation failed"}
    }
  }'

# Start execution
aws stepfunctions start-execution \
  --state-machine-arn arn:aws:states:us-east-1:123456789012:stateMachine:OrderProcessingWorkflow \
  --name "order-12345-$(date +%s)" \
  --input '{"orderId":"12345","productId":"P001","quantity":2}'
```

---

### 🟢 Q15. What is Amazon ECR?
```bash
# ECR: managed Docker/OCI container registry

# Create repository
aws ecr create-repository \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS \
  --image-tag-mutability IMMUTABLE

# Authenticate and push
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

docker build -t myapp:latest .
docker tag myapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0

# Scan results
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=v1.0 \
  --query 'imageScanFindings.findings[?severity==`CRITICAL`].[name,description]' \
  --output table

# Lifecycle policy (auto-clean old images)
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 10 tagged images",
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
        "description": "Remove untagged after 24 hours",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "hours",
          "countNumber": 24
        },
        "action": {"type": "expire"}
      }
    ]
  }'

# Cross-account pull permission
aws ecr set-repository-policy \
  --repository-name myapp \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "AllowCrossAccount",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::987654321098:root"},
      "Action": ["ecr:GetDownloadUrlForLayer","ecr:BatchGetImage","ecr:BatchCheckLayerAvailability","ecr:GetAuthorizationToken"]
    }]
  }'

# Replication to another region
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [{
      "destinations": [
        {"region": "eu-west-1", "registryId": "123456789012"},
        {"region": "ap-southeast-1", "registryId": "123456789012"}
      ],
      "repositoryFilters": [{"filter": "myapp", "filterType": "PREFIX_MATCH"}]
    }]
  }'
```

---

### 🟡 Q16. What is AWS CloudFormation?
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Production Web Application Stack

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev
  InstanceType:
    Type: String
    Default: t3.medium
    AllowedValues: [t3.small, t3.medium, t3.large, m5.large, m5.xlarge]
  VpcId:
    Type: AWS::EC2::VPC::Id
  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>

Mappings:
  EnvConfig:
    dev:     {MinSize: 1, MaxSize: 3,  Desired: 1, InstanceType: t3.small}
    staging: {MinSize: 2, MaxSize: 6,  Desired: 2, InstanceType: t3.medium}
    prod:    {MinSize: 3, MaxSize: 30, Desired: 6, InstanceType: m5.large}

Conditions:
  IsProd: !Equals [!Ref Environment, prod]
  IsNotDev: !Not [!Equals [!Ref Environment, dev]]

Resources:
  AppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: App security group
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 8080
          ToPort: 8080
          SourceSecurityGroupId: !Ref ALBSecurityGroup

  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: !Sub "{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64}}"
        InstanceType: !FindInMap [EnvConfig, !Ref Environment, InstanceType]
        IamInstanceProfile:
          Arn: !GetAtt InstanceProfile.Arn
        SecurityGroupIds: [!Ref AppSecurityGroup]
        MetadataOptions:
          HttpTokens: required
          HttpEndpoint: enabled
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            set -e
            yum update -y
            aws s3 cp s3://my-artifacts/${Environment}/myapp.tar.gz /opt/
            tar -xzf /opt/myapp.tar.gz -C /opt/
            systemctl enable myapp && systemctl start myapp
            /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource AppASG --region ${AWS::Region}

  AppASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    CreationPolicy:
      ResourceSignal:
        Count: !FindInMap [EnvConfig, !Ref Environment, Desired]
        Timeout: PT10M
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MinInstancesInService: 1
        MaxBatchSize: 2
        PauseTime: PT5M
        WaitOnResourceSignals: true
    Properties:
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      MinSize: !FindInMap [EnvConfig, !Ref Environment, MinSize]
      MaxSize: !FindInMap [EnvConfig, !Ref Environment, MaxSize]
      DesiredCapacity: !FindInMap [EnvConfig, !Ref Environment, Desired]
      VPCZoneIdentifier: !Ref SubnetIds
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300
      TargetGroupARNs: [!Ref AppTargetGroup]
      Tags:
        - Key: Env
          Value: !Ref Environment
          PropagateAtLaunch: true

  CPUScalingPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref AppASG
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 65.0

  EnhancedMonitoringAlarm:
    Type: AWS::CloudWatch::Alarm
    Condition: IsProd
    Properties:
      AlarmName: !Sub "${Environment}-high-cpu"
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 3
      Threshold: 90
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref AppASG
      AlarmActions: [!Ref AlertTopic]

Outputs:
  LoadBalancerDNS:
    Value: !GetAtt AppLoadBalancer.DNSName
    Export:
      Name: !Sub "${Environment}-alb-dns"
  ASGName:
    Value: !Ref AppASG
    Export:
      Name: !Sub "${Environment}-asg-name"
```

```bash
# Deploy
aws cloudformation deploy \
  --stack-name myWebStack \
  --template-file template.yaml \
  --parameter-overrides Environment=prod VpcId=vpc-12345678 \
  --capabilities CAPABILITY_IAM \
  --tags Env=prod Project=WebApp

# Change set preview
aws cloudformation create-change-set \
  --stack-name myWebStack \
  --change-set-name my-changes \
  --template-file template-v2.yaml
aws cloudformation describe-change-set \
  --stack-name myWebStack --change-set-name my-changes
aws cloudformation execute-change-set \
  --stack-name myWebStack --change-set-name my-changes

# Stack drift detection
aws cloudformation detect-stack-drift --stack-name myWebStack
```

---

### 🟡 Q17. What is AWS Batch?
```bash
# Batch: managed HPC — schedules and runs batch jobs on EC2 or Fargate

# Compute environment (Spot for cost savings)
aws batch create-compute-environment \
  --compute-environment-name myComputeEnv \
  --type MANAGED --state ENABLED \
  --compute-resources '{
    "type": "SPOT",
    "allocationStrategy": "SPOT_CAPACITY_OPTIMIZED",
    "minvCpus": 0, "maxvCpus": 256, "desiredvCpus": 0,
    "instanceTypes": ["optimal"],
    "subnets": ["subnet-11111111","subnet-22222222"],
    "securityGroupIds": ["sg-12345678"],
    "instanceRole": "arn:aws:iam::123456789012:instance-profile/ecsInstanceRole",
    "bidPercentage": 60,
    "spotIamFleetRole": "arn:aws:iam::123456789012:role/AmazonEC2SpotFleetRole",
    "ec2Configuration": [{"imageType": "ECS_AL2023"}]
  }' \
  --service-role arn:aws:iam::123456789012:role/AWSBatchServiceRole

# Job queue
aws batch create-job-queue \
  --job-queue-name myJobQueue \
  --state ENABLED --priority 100 \
  --compute-environment-order order=1,computeEnvironment=myComputeEnv

# Job definition
aws batch register-job-definition \
  --job-definition-name imageProcessor \
  --type container \
  --container-properties '{
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/imgproc:latest",
    "vcpus": 4, "memory": 8192,
    "jobRoleArn": "arn:aws:iam::123456789012:role/BatchJobRole",
    "environment": [{"name":"S3_BUCKET","value":"my-images"}],
    "resourceRequirements": [{"type":"GPU","value":"1"}]
  }' \
  --retry-strategy attempts=3,evaluateOnExit=[{onStatusReason="Host EC2*terminated",action="RETRY"}] \
  --timeout attemptDurationSeconds=7200

# Submit array job (1000 parallel tasks)
aws batch submit-job \
  --job-name processImages-$(date +%s) \
  --job-queue myJobQueue \
  --job-definition imageProcessor \
  --array-properties size=1000 \
  --container-overrides environment=[{name=BATCH_SIZE,value=10}]

# Monitor
aws batch describe-jobs \
  --jobs $(aws batch list-jobs --job-queue myJobQueue --job-status RUNNING \
    --query 'jobSummaryList[].jobId' --output text)
```

---

### 🟡 Q18. What is AWS Elastic Load Balancing?
```bash
# ALB: L7 HTTP/HTTPS — URL/host routing, WAF, gRPC, Lambda targets
# NLB: L4 TCP/UDP/TLS — ultra-low latency, static IPs, 100M+ connections/s
# GWLB: L3 — for 3rd party firewalls (Palo Alto, Fortinet) inline inspection

# Create ALB
aws elbv2 create-load-balancer \
  --name myALB \
  --subnets subnet-11111111 subnet-22222222 subnet-33333333 \
  --security-groups sg-12345678 \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4

ALB_ARN=$(aws elbv2 describe-load-balancers --names myALB \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

# Target group
aws elbv2 create-target-group \
  --name myAppTG \
  --protocol HTTP --port 8080 \
  --vpc-id vpc-12345678 \
  --target-type ip \
  --health-check-path "/health" \
  --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --matcher HttpCode=200,301

TG_ARN=$(aws elbv2 describe-target-groups --names myAppTG \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# HTTPS listener
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTPS --port 443 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/abc123 \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

# HTTP→HTTPS redirect
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions 'Type=redirect,RedirectConfig={Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'

# Path-based routing
aws elbv2 create-rule \
  --listener-arn <https-listener-arn> \
  --priority 10 \
  --conditions Field=path-pattern,Values=/api/* \
  --actions Type=forward,TargetGroupArn=<api-tg-arn>

# Host-based routing
aws elbv2 create-rule \
  --listener-arn <https-listener-arn> \
  --priority 20 \
  --conditions Field=host-header,Values=api.example.com \
  --actions Type=forward,TargetGroupArn=<api-tg-arn>

# Header-based routing (canary/version routing)
aws elbv2 create-rule \
  --listener-arn <https-listener-arn> \
  --priority 5 \
  --conditions '[{"Field":"http-header","HttpHeaderConfig":{"HttpHeaderName":"X-Version","Values":["v2"]}}]' \
  --actions Type=forward,TargetGroupArn=<v2-tg-arn>

# NLB with static IPs
aws elbv2 create-load-balancer \
  --name myNLB --type network \
  --subnets subnet-11111111 subnet-22222222 \
  --scheme internet-facing

# Register targets
aws elbv2 register-targets \
  --target-group-arn $TG_ARN \
  --targets Id=i-1234567890abcdef0,Port=8080 Id=i-abcdef1234567890,Port=8080
```

---

### 🟢 Q19. What is the difference between ECS, EKS, Lambda, and Fargate?
| Feature | ECS | EKS | Lambda | Fargate |
|---------|-----|-----|--------|---------|
| **Type** | Container orchestration | Kubernetes | Serverless functions | Serverless compute |
| **Management** | AWS-native | Kubernetes API | None | None (used with ECS/EKS) |
| **Execution** | Long-running | Long-running | Short (max 15 min) | Both |
| **Scaling** | Manual + ASG | HPA + Cluster Autoscaler | Automatic | Automatic |
| **Cold start** | Minimal | Minimal | Yes (100ms–5s) | Minimal |
| **Pricing** | EC2 + Fargate | $0.10/hr + nodes | Per invocation | vCPU + memory/second |
| **Use case** | Microservices | Complex K8s workloads | Event-driven tasks | Serverless containers |

---

### 🟡 Q20. What is AWS Auto Scaling for multiple services?
```bash
# Application Auto Scaling: scale beyond EC2
# Supports: ECS, DynamoDB, Aurora, Kinesis, SageMaker, Lambda

# Scale ECS service
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/myCluster/myapp \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 100

# Scale DynamoDB read capacity
aws application-autoscaling register-scalable-target \
  --service-namespace dynamodb \
  --resource-id table/MyTable \
  --scalable-dimension dynamodb:table:ReadCapacityUnits \
  --min-capacity 5 --max-capacity 1000

aws application-autoscaling put-scaling-policy \
  --service-namespace dynamodb \
  --resource-id table/MyTable \
  --scalable-dimension dynamodb:table:ReadCapacityUnits \
  --policy-name dynamo-read-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "DynamoDBReadCapacityUtilization"
    }
  }'

# Scale Aurora read replicas
aws application-autoscaling register-scalable-target \
  --service-namespace rds \
  --resource-id cluster:my-aurora-cluster \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --min-capacity 1 --max-capacity 15

aws application-autoscaling put-scaling-policy \
  --service-namespace rds \
  --resource-id cluster:my-aurora-cluster \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --policy-name aurora-reader-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 60.0,
    "PredefinedMetricSpecification": {"PredefinedMetricType": "RDSReaderAverageCPUUtilization"}
  }'

# AWS Compute Optimizer recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --filters name=Finding,values=OVER_PROVISIONED \
  --query 'instanceRecommendations[].[instanceArn,finding,recommendationOptions[0].instanceType,recommendationOptions[0].estimatedMonthlySavings.value]' \
  --output table
```


---

# PART 2 — NETWORKING

---

### 🟢 Q21. What is Amazon VPC?
```bash
# VPC: logically isolated virtual network in AWS
# Default VPC: created per region, /16 CIDR, one /20 subnet per AZ
# Custom VPC: you define CIDR, subnets, routing, gateways

# Create VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --enable-dns-hostnames \
  --enable-dns-support \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=myVPC},{Key=Env,Value=prod}]'

VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=myVPC" \
  --query 'Vpcs[0].VpcId' --output text)

# Add secondary CIDR (for IPv6 or additional IP space)
aws ec2 associate-vpc-cidr-block --vpc-id $VPC_ID --cidr-block 10.1.0.0/16

# Enable IPv6
aws ec2 associate-vpc-cidr-block \
  --vpc-id $VPC_ID \
  --amazon-provided-ipv6-cidr-block

# Create subnets (public + private across 3 AZs)
for az in a b c; do
  # Public subnet
  aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.$((${#az}*10)).0/24 \
    --availability-zone us-east-1$az \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=public-us-east-1$az},{Key=Type,Value=public}]"

  # Private subnet
  aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.$((${#az}*10+1)).0/24 \
    --availability-zone us-east-1$az \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=private-us-east-1$az},{Key=Type,Value=private}]"
done

# Internet Gateway (for public subnet internet access)
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=myIGW}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# Route table for public subnets
aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=public-rt}]'
aws ec2 create-route --route-table-id <rt-id> --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id <rt-id> --subnet-id <public-subnet-id>

# Enable auto-assign public IP for public subnets
aws ec2 modify-subnet-attribute \
  --subnet-id <public-subnet-id> \
  --map-public-ip-on-launch
```

---

### 🟢 Q22. What are Security Groups vs NACLs?
| Feature | Security Group | NACL |
|---------|--------------|------|
| **Level** | Instance (ENI) | Subnet |
| **Statefulness** | Stateful (return traffic auto-allowed) | Stateless (both directions need rules) |
| **Default** | Deny all in, allow all out | Allow all in and out |
| **Rules** | Allow only | Allow AND Deny |
| **Evaluation** | All rules checked | Rules by number order (lowest first) |
| **Applies to** | Instances in group | All instances in subnet |

```bash
# Security Group
aws ec2 create-security-group \
  --group-name web-sg \
  --description "Web servers security group" \
  --vpc-id $VPC_ID

SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=web-sg" \
  --query 'SecurityGroups[0].GroupId' --output text)

# Inbound rules
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --ip-permissions '[
    {"IpProtocol":"tcp","FromPort":443,"ToPort":443,"IpRanges":[{"CidrIp":"0.0.0.0/0"}]},
    {"IpProtocol":"tcp","FromPort":80,"ToPort":80,"IpRanges":[{"CidrIp":"0.0.0.0/0"}]},
    {"IpProtocol":"tcp","FromPort":22,"ToPort":22,"IpRanges":[{"CidrIp":"10.0.0.0/8","Description":"Bastion access"}]}
  ]'

# Allow from another security group (app→database)
aws ec2 authorize-security-group-ingress --group-id <db-sg-id> \
  --ip-permissions '[{
    "IpProtocol":"tcp","FromPort":5432,"ToPort":5432,
    "UserIdGroupPairs":[{"GroupId":"<app-sg-id>","Description":"App servers"}]
  }]'

# NACL (stateless — need both inbound and outbound rules)
aws ec2 create-network-acl --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=web-nacl}]'

NACL_ID=$(aws ec2 describe-network-acls \
  --filters "Name=tag:Name,Values=web-nacl" \
  --query 'NetworkAcls[0].NetworkAclId' --output text)

# INBOUND rules (lower number = higher priority)
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 100 --protocol tcp --rule-action allow --ingress \
  --cidr-block 0.0.0.0/0 --port-range From=443,To=443

aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 200 --protocol tcp --rule-action allow --ingress \
  --cidr-block 0.0.0.0/0 --port-range From=1024,To=65535   # ephemeral ports

aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 32766 --protocol all --rule-action deny --ingress \
  --cidr-block 0.0.0.0/0   # deny all (implicit, but explicit is safer)

# OUTBOUND rules (stateless — must explicitly allow response traffic)
aws ec2 create-network-acl-entry --network-acl-id $NACL_ID \
  --rule-number 100 --protocol tcp --rule-action allow --egress \
  --cidr-block 0.0.0.0/0 --port-range From=1024,To=65535
```

---

### 🟢 Q23. What are NAT Gateway and NAT Instance?
```bash
# NAT Gateway: managed, highly available, outbound internet for private subnets
# NAT Instance: EC2 instance with NAT — self-managed, cheaper, not HA

# Create Elastic IP for NAT Gateway
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)

# Create NAT Gateway in PUBLIC subnet
NGW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id <public-subnet-id> \
  --allocation-id $EIP_ALLOC \
  --connectivity-type public \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=myNATGW}]' \
  --query NatGateway.NatGatewayId --output text)

aws ec2 wait nat-gateway-available --nat-gateway-ids $NGW_ID

# Private subnet route table → NAT Gateway
aws ec2 create-route \
  --route-table-id <private-rt-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NGW_ID

# Private NAT Gateway (for VPC-to-VPC routing without internet)
aws ec2 create-nat-gateway \
  --subnet-id <private-subnet-id> \
  --connectivity-type private \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=privateNATGW}]'

# NAT Gateway vs NAT Instance:
# NAT GW: Managed, HA, scales to 45 Gbps, $0.045/hr + data, no security group
# NAT Instance: EC2 t3.nano ~$4/mo, manually HA, security group support, port forwarding
```

---

### 🟡 Q24. What is VPC Peering?
```bash
# VPC Peering: private connection between two VPCs (same or different accounts/regions)
# Non-transitive: A↔B and B↔C does NOT mean A↔C (must peer A↔C separately)
# No overlapping CIDR blocks allowed

# Request peering (same account)
PEERING_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-11111111 \
  --peer-vpc-id vpc-22222222 \
  --peer-region us-west-2 \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=prod-dev-peering}]' \
  --query VpcPeeringConnection.VpcPeeringConnectionId --output text)

# Accept peering (in peer account/region)
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID \
  --region us-west-2

# Add routes (must add in BOTH VPCs)
# VPC1 → VPC2
aws ec2 create-route \
  --route-table-id <vpc1-rt-id> \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID

# VPC2 → VPC1
aws ec2 create-route \
  --route-table-id <vpc2-rt-id> \
  --destination-cidr-block 10.0.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID

# Modify security groups to allow traffic from peer VPC
aws ec2 authorize-security-group-ingress --group-id <sg-id> \
  --ip-permissions '[{
    "IpProtocol":"tcp","FromPort":5432,"ToPort":5432,
    "IpRanges":[{"CidrIp":"10.1.0.0/16","Description":"Peer VPC"}]
  }]'

# Peering limitations:
# ❌ Non-transitive
# ❌ No overlapping CIDRs
# ❌ No edge-to-edge routing (VPN, Direct Connect, IGW through peering)
# ✅ Cross-account, cross-region
# ✅ Encrypted (AES-256 across regions)
```

---

### 🟡 Q25. What is AWS Transit Gateway?
```bash
# Transit Gateway: hub-and-spoke network hub (transitive routing!)
# Replaces: multiple VPC peering connections
# Supports: VPCs, VPNs, Direct Connect, peered TGWs

# Create Transit Gateway
TGW_ID=$(aws ec2 create-transit-gateway \
  --description "Central network hub" \
  --options '{
    "AmazonSideAsn": 64512,
    "AutoAcceptSharedAttachments": "enable",
    "DefaultRouteTableAssociation": "enable",
    "DefaultRouteTablePropagation": "enable",
    "VpnEcmpSupport": "enable",
    "DnsSupport": "enable",
    "MulticastSupport": "enable"
  }' \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=myTGW}]' \
  --query TransitGateway.TransitGatewayId --output text)

# Attach VPCs to TGW
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id vpc-11111111 \
  --subnet-ids subnet-11111111 subnet-22222222 \
  --options ApplianceModeSupport=disable,DnsSupport=enable \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=prod-vpc-attach}]'

# Create custom route tables for isolation (spoke VPCs can't talk to each other)
TGW_RT=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id $TGW_ID \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=spoke-rt}]' \
  --query TransitGatewayRouteTable.TransitGatewayRouteTableId --output text)

# Static route to firewall VPC for inspection
aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id $TGW_RT \
  --destination-cidr-block 0.0.0.0/0 \
  --transit-gateway-attachment-id <firewall-vpc-attach-id>

# Inter-region peering
aws ec2 create-transit-gateway-peering-attachment \
  --transit-gateway-id $TGW_ID \
  --peer-transit-gateway-id <eu-tgw-id> \
  --peer-account-id 123456789012 \
  --peer-region eu-west-1

# Share TGW with other accounts (RAM)
aws ram create-resource-share \
  --name TGWShare \
  --resource-arns arn:aws:ec2:us-east-1:123456789012:transit-gateway/$TGW_ID \
  --principals 987654321098
```

---

### 🟡 Q26. What is VPC Endpoints?
```bash
# VPC Endpoints: private access to AWS services without internet/NAT
# Gateway Endpoint: S3 and DynamoDB only (free, route table based)
# Interface Endpoint: other AWS services (PrivateLink, ENI, $0.01/hr)

# Gateway Endpoint for S3
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids <private-rt-id> \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject","s3:PutObject","s3:DeleteObject"],
      "Resource": ["arn:aws:s3:::my-bucket","arn:aws:s3:::my-bucket/*"]
    }]
  }'

# Gateway Endpoint for DynamoDB
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --vpc-endpoint-type Gateway \
  --route-table-ids <private-rt-id>

# Interface Endpoint (PrivateLink) for Secrets Manager
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-11111111 subnet-22222222 \
  --security-group-ids $SG_ID \
  --private-dns-enabled

# Interface Endpoint for ECR
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-11111111 subnet-22222222 \
  --security-group-ids $SG_ID \
  --private-dns-enabled

# Other commonly needed interface endpoints for private EKS/ECS:
# ecr.api, ecr.dkr, s3, logs, monitoring, secretsmanager, sts, ec2, elasticloadbalancing

# List available services
aws ec2 describe-vpc-endpoint-services \
  --query 'ServiceNames' --output text | tr '\t' '\n' | grep amazonaws
```

---

### 🟡 Q27. What is AWS VPN (Site-to-Site and Client)?
```bash
# Site-to-Site VPN: connect on-premises to VPC over IPsec tunnels
# Client VPN: remote user access via OpenVPN

# Create Customer Gateway (your on-prem router)
CGW_ID=$(aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.1 \
  --bgp-asn 65000 \
  --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=myOnPremGW}]' \
  --query CustomerGateway.CustomerGatewayId --output text)

# Create Virtual Private Gateway (AWS side)
VGW_ID=$(aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --amazon-side-asn 64512 \
  --tag-specifications 'ResourceType=vpn-gateway,Tags=[{Key=Name,Value=myVGW}]' \
  --query VpnGateway.VpnGatewayId --output text)

aws ec2 attach-vpn-gateway --vpn-gateway-id $VGW_ID --vpc-id $VPC_ID

# Create VPN connection (2 tunnels for HA)
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id $CGW_ID \
  --vpn-gateway-id $VGW_ID \
  --options '{
    "StaticRoutesOnly": false,
    "TunnelOptions": [
      {"TunnelInsideCidr": "169.254.100.0/30", "PreSharedKey": "MySecret1!"},
      {"TunnelInsideCidr": "169.254.101.0/30", "PreSharedKey": "MySecret2!"}
    ]
  }'

# Enable route propagation
aws ec2 enable-vgw-route-propagation \
  --route-table-id <private-rt-id> \
  --gateway-id $VGW_ID

# Accelerated VPN (uses AWS Global Accelerator for better performance)
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id $CGW_ID \
  --transit-gateway-id $TGW_ID \
  --options '{"EnableAcceleration": true, "StaticRoutesOnly": false}'

# Client VPN
aws ec2 create-client-vpn-endpoint \
  --client-cidr-block 172.16.0.0/22 \
  --server-certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/abc123 \
  --authentication-options '[{
    "Type": "federated-authentication",
    "FederatedAuthentication": {"SAMLProviderArn": "arn:aws:iam::123456789012:saml-provider/myIDP"}
  }]' \
  --connection-log-options EnabledClientVpnLogs=true,CloudwatchLogGroup=/aws/clientvpn \
  --split-tunnel false \
  --dns-servers 10.0.0.2
```

---

### 🟡 Q28. What is AWS Direct Connect?
```bash
# Direct Connect: dedicated private fiber connection (1Gbps, 10Gbps, 100Gbps)
# Benefits: consistent bandwidth, lower latency, reduced data transfer costs

# Connection types:
# Dedicated:  1Gbps / 10Gbps / 100Gbps direct port to AWS
# Hosted:     sub-1Gbps (50Mbps–10Gbps) via AWS partner

# Create Direct Connect connection (after ordering physical cross-connect)
aws directconnect create-connection \
  --location EqDC2 \                    # location code for equinix DC
  --bandwidth 10Gbps \
  --connection-name myDXConnection \
  --request-mac-sec false

# Create Virtual Interface (VIF)
# Private VIF: access VPC via VGW
aws directconnect create-private-virtual-interface \
  --connection-id dxcon-12345678 \
  --new-private-virtual-interface '{
    "virtualInterfaceName": "myPrivateVIF",
    "vlan": 100,
    "asn": 65000,
    "amazonSideAsn": 64512,
    "authKey": "myBGPauthKey",
    "amazonAddress": "169.254.1.1/30",
    "customerAddress": "169.254.1.2/30",
    "virtualGatewayId": "vgw-12345678",
    "addressFamily": "ipv4"
  }'

# Public VIF: access AWS public services (S3, DynamoDB) without internet
aws directconnect create-public-virtual-interface \
  --connection-id dxcon-12345678 \
  --new-public-virtual-interface '{
    "virtualInterfaceName": "myPublicVIF",
    "vlan": 200,
    "asn": 65000,
    "amazonAddress": "169.254.2.1/30",
    "customerAddress": "169.254.2.2/30",
    "routeFilterPrefixes": [{"cidr": "0.0.0.0/0"}]
  }'

# Transit VIF: connect to Transit Gateway for multiple VPCs
aws directconnect create-transit-virtual-interface \
  --connection-id dxcon-12345678 \
  --new-transit-virtual-interface '{
    "virtualInterfaceName": "myTransitVIF",
    "vlan": 300, "asn": 65000,
    "directConnectGatewayId": "dxgw-12345678"
  }'

# Direct Connect Gateway (access multiple VPCs/regions from single DX connection)
aws directconnect create-direct-connect-gateway \
  --direct-connect-gateway-name myDXGW \
  --amazon-side-asn 64512

# Link resiliency: use 2 connections at 2 different locations for HA
# Backup with Site-to-Site VPN (failover if DX goes down)
```

---

### 🟡 Q29. What are Route 53 routing policies?
```bash
# Route 53: DNS service with health checks and traffic management

# Create hosted zone
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference $(date +%s) \
  --hosted-zone-config Comment="Main zone",PrivateZone=false

ZONE_ID=$(aws route53 list-hosted-zones-by-name \
  --dns-name example.com --query 'HostedZones[0].Id' --output text)

# Simple routing (single record)
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [{"Action": "UPSERT", "ResourceRecordSet": {
      "Name": "www.example.com",
      "Type": "A",
      "TTL": 300,
      "ResourceRecords": [{"Value": "203.0.113.1"}]
    }}]
  }'

# Alias record (for AWS resources — free, root domain supported)
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [{"Action": "UPSERT", "ResourceRecordSet": {
      "Name": "example.com",
      "Type": "A",
      "AliasTarget": {
        "HostedZoneId": "Z35SXDOTRQ7X7K",
        "DNSName": "myalb-1234567890.us-east-1.elb.amazonaws.com",
        "EvaluateTargetHealth": true
      }
    }}]
  }'

# Weighted routing (A/B testing: 90% prod, 10% new)
for i in 1 2; do
  aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
    --change-batch "{
      \"Changes\": [{\"Action\": \"UPSERT\", \"ResourceRecordSet\": {
        \"Name\": \"api.example.com\",
        \"Type\": \"A\",
        \"SetIdentifier\": \"v${i}\",
        \"Weight\": $([[ $i -eq 1 ]] && echo 90 || echo 10),
        \"TTL\": 60,
        \"ResourceRecords\": [{\"Value\": \"10.0.${i}.1\"}]
      }}]
    }"
done

# Latency-based routing (route to lowest-latency region)
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [{"Action": "UPSERT", "ResourceRecordSet": {
      "Name": "api.example.com", "Type": "A",
      "SetIdentifier": "us-east-1",
      "Region": "us-east-1",
      "TTL": 60,
      "ResourceRecords": [{"Value": "10.0.1.1"}]
    }}]
  }'

# Failover routing (primary + secondary)
# Create health check
HC_ID=$(aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config '{
    "IPAddress": "203.0.113.1",
    "Port": 443,
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FullyQualifiedDomainName": "primary.example.com",
    "RequestInterval": 30,
    "FailureThreshold": 3,
    "MeasureLatency": true,
    "EnableSNI": true
  }' \
  --query HealthCheck.Id --output text)

aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
  --change-batch "{
    \"Changes\": [{\"Action\": \"UPSERT\", \"ResourceRecordSet\": {
      \"Name\": \"api.example.com\", \"Type\": \"A\",
      \"SetIdentifier\": \"primary\",
      \"Failover\": \"PRIMARY\",
      \"TTL\": 60,
      \"ResourceRecords\": [{\"Value\": \"203.0.113.1\"}],
      \"HealthCheckId\": \"$HC_ID\"
    }}]
  }"

# Geolocation routing (EU users → EU endpoint)
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [{"Action": "UPSERT", "ResourceRecordSet": {
      "Name": "api.example.com", "Type": "A",
      "SetIdentifier": "EU-record",
      "GeoLocation": {"ContinentCode": "EU"},
      "TTL": 60,
      "ResourceRecords": [{"Value": "10.1.1.1"}]
    }}]
  }'

# Geoproximity routing (bias-based geographic routing via Traffic Policies)
# IP-based routing (route by source IP ranges)
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [{"Action": "UPSERT", "ResourceRecordSet": {
      "Name": "api.example.com", "Type": "A",
      "SetIdentifier": "corporate-network",
      "CidrRoutingConfig": {"CollectionId": "cidr-collection-id", "LocationName": "corporate"},
      "TTL": 60,
      "ResourceRecords": [{"Value": "10.0.1.1"}]
    }}]
  }'

# Private hosted zone (internal DNS in VPC)
aws route53 create-hosted-zone \
  --name internal.example.com \
  --caller-reference $(date +%s) \
  --hosted-zone-config PrivateZone=true \
  --vpc VPCRegion=us-east-1,VPCId=$VPC_ID
```

---

### 🟡 Q30. What is Amazon CloudFront?
```bash
# CloudFront: CDN with 450+ edge locations
# Features: caching, WAF, Lambda@Edge, CloudFront Functions, HTTPS, custom headers

# Create distribution
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "'$(date +%s)'",
    "Origins": {
      "Quantity": 2,
      "Items": [
        {
          "Id": "S3-static",
          "DomainName": "my-bucket.s3.us-east-1.amazonaws.com",
          "S3OriginConfig": {"OriginAccessIdentity": ""},
          "OriginAccessControlId": "EXXXXXXXXXX"
        },
        {
          "Id": "ALB-api",
          "DomainName": "myalb.us-east-1.elb.amazonaws.com",
          "CustomOriginConfig": {
            "HTTPSPort": 443,
            "OriginProtocolPolicy": "https-only",
            "OriginSSLProtocols": {"Quantity": 1, "Items": ["TLSv1.2"]}
          }
        }
      ]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "S3-static",
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
      "Compress": true,
      "AllowedMethods": {"Quantity":2,"Items":["GET","HEAD"],"CachedMethods":{"Quantity":2,"Items":["GET","HEAD"]}}
    },
    "CacheBehaviors": {
      "Quantity": 1,
      "Items": [{
        "PathPattern": "/api/*",
        "TargetOriginId": "ALB-api",
        "ViewerProtocolPolicy": "https-only",
        "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
        "AllowedMethods": {"Quantity":7,"Items":["DELETE","GET","HEAD","OPTIONS","PATCH","POST","PUT"],"CachedMethods":{"Quantity":2,"Items":["GET","HEAD"]}}
      }]
    },
    "Aliases": {"Quantity": 1, "Items": ["www.example.com"]},
    "ViewerCertificate": {
      "ACMCertificateArn": "arn:aws:acm:us-east-1:123456789012:certificate/abc123",
      "SSLSupportMethod": "sni-only",
      "MinimumProtocolVersion": "TLSv1.2_2021"
    },
    "HttpVersion": "http2and3",
    "IsIPV6Enabled": true,
    "Enabled": true,
    "PriceClass": "PriceClass_All",
    "DefaultRootObject": "index.html",
    "Comment": "My CloudFront Distribution"
  }'

# Invalidate cache (purge)
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/*"                     # all objects
# or specific: "/js/*" "/index.html"

# CloudFront Functions (lightweight JS, runs at edge for header manipulation)
aws cloudfront create-function \
  --name addSecurityHeaders \
  --function-config Comment="Add security headers",Runtime=cloudfront-js-2.0 \
  --function-code fileb://security-headers.js

# security-headers.js
# function handler(event) {
#   var response = event.response;
#   var headers = response.headers;
#   headers['strict-transport-security'] = {value: 'max-age=31536000; includeSubdomains'};
#   headers['x-content-type-options'] = {value: 'nosniff'};
#   headers['x-frame-options'] = {value: 'DENY'};
#   headers['x-xss-protection'] = {value: '1; mode=block'};
#   return response;
# }
```

---

### 🟡 Q31. What is AWS WAF?
```bash
# WAF: web application firewall — filter HTTP/S traffic for ALB, CloudFront, API GW

# Create WebACL
aws wafv2 create-web-acl \
  --name myWebACL \
  --scope REGIONAL \
  --default-action Allow={} \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=myWebACLMetric \
  --rules '[
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "OverrideAction": {"None":{}},
      "VisibilityConfig": {"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"CommonRules"},
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      }
    },
    {
      "Name": "AWSManagedRulesKnownBadInputsRuleSet",
      "Priority": 2,
      "OverrideAction": {"None":{}},
      "VisibilityConfig": {"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"BadInputs"},
      "Statement": {
        "ManagedRuleGroupStatement": {"VendorName":"AWS","Name":"AWSManagedRulesKnownBadInputsRuleSet"}
      }
    },
    {
      "Name": "RateLimitRule",
      "Priority": 3,
      "Action": {"Block":{}},
      "VisibilityConfig": {"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"RateLimit"},
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP",
          "EvaluationWindowSec": 300
        }
      }
    },
    {
      "Name": "GeoBlockRule",
      "Priority": 4,
      "Action": {"Block":{}},
      "VisibilityConfig": {"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"GeoBlock"},
      "Statement": {
        "GeoMatchStatement": {"CountryCodes": ["KP","IR","CU"]}
      }
    }
  ]' \
  --region us-east-1

# Associate WebACL with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:us-east-1:123456789012:regional/webacl/myWebACL/abc123 \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/myALB/abc123

# Enable logging to S3 or Kinesis
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/myWebACL/abc123",
    "LogDestinationConfigs": ["arn:aws:firehose:us-east-1:123456789012:deliverystream/aws-waf-logs-mystream"],
    "RedactedFields": [{"SingleHeader":{"Name":"authorization"}}]
  }'
```

---

### 🟡 Q32. What is AWS PrivateLink?
```bash
# PrivateLink: expose services from one VPC to others privately (no peering, no internet)
# Use: expose your SaaS service / consume partner services
# Components: NLB (service side) + VPC Endpoint (consumer side)

# Service provider side: create endpoint service from NLB
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/net/myNLB/abc123 \
  --acceptance-required false \
  --tag-specifications 'ResourceType=vpc-endpoint-service,Tags=[{Key=Name,Value=myPrivateLinkService}]'

# Allow specific accounts
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-12345678 \
  --add-allowed-principals arn:aws:iam::987654321098:root

# Consumer side: create interface endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-consumer \
  --service-name com.amazonaws.vpce.us-east-1.vpce-svc-12345678 \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-consumer1 subnet-consumer2 \
  --security-group-ids sg-consumer \
  --private-dns-enabled
```

---

### 🟡 Q33. What is AWS Global Accelerator?
```bash
# Global Accelerator: static anycast IPs that route to optimal AWS region
# Uses AWS backbone (not public internet) — low latency, fast failover
# vs CloudFront: GA works at L4 (TCP/UDP), CloudFront at L7 (HTTP/S only)

# Create accelerator
aws globalaccelerator create-accelerator \
  --name myAccelerator \
  --ip-address-type IPV4 \
  --enabled \
  --tags Key=Env,Value=prod

ACC_ARN=$(aws globalaccelerator describe-accelerator \
  --accelerator-arn arn:aws:globalaccelerator::123456789012:accelerator/abc123 \
  --query Accelerator.AcceleratorArn --output text)

# Create listener
aws globalaccelerator create-listener \
  --accelerator-arn $ACC_ARN \
  --protocol TCP \
  --port-ranges '[{"FromPort":80},{"FromPort":443}]' \
  --client-affinity SOURCE_IP

# Create endpoint groups (one per region)
aws globalaccelerator create-endpoint-group \
  --listener-arn <listener-arn> \
  --endpoint-group-region us-east-1 \
  --endpoint-configurations '[{
    "EndpointId": "arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/myALB/abc",
    "Weight": 100,
    "ClientIPPreservationEnabled": true
  }]' \
  --traffic-dial-percentage 100 \
  --health-check-path /health \
  --health-check-protocol HTTP

aws globalaccelerator create-endpoint-group \
  --listener-arn <listener-arn> \
  --endpoint-group-region eu-west-1 \
  --endpoint-configurations '[{
    "EndpointId": "arn:aws:elasticloadbalancing:eu-west-1:123456789012:loadbalancer/app/myEUALB/def",
    "Weight": 100
  }]' \
  --traffic-dial-percentage 0   # start at 0, increase for DR drill
```

---

### 🟢 Q34. What is Amazon API Gateway?
```bash
# API Gateway: create, publish, maintain REST/HTTP/WebSocket APIs
# Types: REST API (full features), HTTP API (lower cost, simpler), WebSocket

# Create HTTP API (recommended for most use cases)
aws apigatewayv2 create-api \
  --name myHttpAPI \
  --protocol-type HTTP \
  --cors-configuration '{
    "AllowOrigins": ["https://myapp.com"],
    "AllowMethods": ["GET","POST","PUT","DELETE","OPTIONS"],
    "AllowHeaders": ["Content-Type","Authorization","X-Api-Key"],
    "MaxAge": 86400
  }'

API_ID=$(aws apigatewayv2 describe-api --api-id <id> --query ApiId --output text)

# Lambda integration
aws apigatewayv2 create-integration \
  --api-id $API_ID \
  --integration-type AWS_PROXY \
  --integration-uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:123456789012:function:myFunction/invocations \
  --payload-format-version 2.0 \
  --timeout-in-millis 29000

# Route
aws apigatewayv2 create-route \
  --api-id $API_ID \
  --route-key "ANY /orders/{orderId}" \
  --target integrations/<integration-id>

# Deploy stage
aws apigatewayv2 create-stage \
  --api-id $API_ID \
  --stage-name prod \
  --auto-deploy true \
  --default-route-settings ThrottlingBurstLimit=5000,ThrottlingRateLimit=10000

# Authorizer (JWT — Cognito or custom)
aws apigatewayv2 create-authorizer \
  --api-id $API_ID \
  --name CognitoAuthorizer \
  --authorizer-type JWT \
  --identity-source '$request.header.Authorization' \
  --jwt-configuration '{
    "Audience": ["my-app-client-id"],
    "Issuer": "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXXXX"
  }'

# API key + usage plan (REST API)
aws apigateway create-api-key --name myAPIKey --enabled

aws apigateway create-usage-plan \
  --name BasicPlan \
  --throttle burstLimit=1000,rateLimit=500 \
  --quota limit=1000000,period=MONTH
```

---

### 🟡 Q35. What is the difference between ALB, NLB, and GWLB?
| Feature | ALB | NLB | GWLB |
|---------|-----|-----|------|
| **OSI Layer** | L7 (HTTP/S) | L4 (TCP/UDP/TLS) | L3 (IP) |
| **Routing** | URL, host, header | IP, port | Flow hash |
| **Use case** | Web apps, microservices | Gaming, IoT, low-latency | 3rd party firewalls |
| **Static IP** | ❌ (dynamic) | ✅ (per AZ) | ✅ |
| **TLS offload** | ✅ | ✅ (passthrough too) | ❌ |
| **WebSocket** | ✅ | ✅ | ❌ |
| **gRPC** | ✅ | ❌ | ❌ |
| **Lambda target** | ✅ | ❌ | ❌ |
| **Throughput** | ~60 Gbps | 100M+ req/sec | High |

---

### 🟡 Q36. What are VPC Flow Logs?
```bash
# Flow Logs: capture IP traffic info for VPC, subnets, or ENIs
# Destinations: CloudWatch Logs, S3, Kinesis Data Firehose
# Use for: security analysis, troubleshooting, compliance

# Enable flow logs for entire VPC to CloudWatch
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-destination arn:aws:logs:us-east-1:123456789012:log-group:/vpc/flow-logs \
  --iam-role-arn arn:aws:iam::123456789012:role/VPCFlowLogsRole \
  --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${log-status} ${vpc-id} ${subnet-id} ${instance-id}'

# Enable flow logs to S3 (cheaper, queryable with Athena)
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-vpc-flow-logs/flowlogs/ \
  --log-format '${version} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${action} ${bytes}' \
  --destination-options FileFormat=parquet,HiveCompatiblePartitions=true,PerHourPartition=true

# Query flow logs with Athena
# CREATE EXTERNAL TABLE vpc_flow_logs (
#   version int, account_id string, interface_id string,
#   srcaddr string, dstaddr string, srcport int, dstport int,
#   protocol bigint, packets bigint, bytes bigint,
#   start bigint, end bigint, action string, logstatus string
# ) PARTITIONED BY (year string, month string, day string)
# STORED AS PARQUET LOCATION 's3://my-vpc-flow-logs/flowlogs/'

# CloudWatch Insights query
# stats sum(bytes) as bytesTransferred by srcaddr, dstaddr
# | sort bytesTransferred desc | limit 20
```

---

### 🟡 Q37. What is AWS Network Firewall?
```bash
# Network Firewall: managed stateful firewall for VPC traffic inspection
# Supports: IPS/IDS, deep packet inspection, domain filtering, TLS inspection

aws network-firewall create-firewall-policy \
  --firewall-policy-name myFirewallPolicy \
  --firewall-policy '{
    "StatelessDefaultActions": ["aws:forward_to_sfe"],
    "StatelessFragmentDefaultActions": ["aws:forward_to_sfe"],
    "StatefulDefaultActions": ["aws:drop_established"],
    "StatefulRuleGroupReferences": [
      {"ResourceArn": "arn:aws:network-firewall:us-east-1:123456789012:stateful-rulegroup/myRules"}
    ],
    "StatefulEngineOptions": {"RuleOrder": "STRICT_ORDER"}
  }'

# Create stateful rule group (Suricata-compatible IPS rules)
aws network-firewall create-rule-group \
  --rule-group-name domainFilterRules \
  --type STATEFUL \
  --capacity 100 \
  --rule-group '{
    "RulesSource": {
      "RulesSourceList": {
        "Targets": [".malware.example.com", ".phishing.example.com"],
        "TargetTypes": ["TLS_SNI", "HTTP_HOST"],
        "GeneratedRulesType": "DENYLIST"
      }
    }
  }'

# Create firewall in VPC
aws network-firewall create-firewall \
  --firewall-name myNetworkFirewall \
  --firewall-policy-arn arn:aws:network-firewall:us-east-1:123456789012:firewall-policy/myFirewallPolicy \
  --vpc-id $VPC_ID \
  --subnet-mappings SubnetId=subnet-firewall-az1 SubnetId=subnet-firewall-az2 \
  --tags Key=Env,Value=prod

# Route traffic through firewall (spoke subnet → TGW → firewall subnet → internet)
# Architecture: Internet → IGW → Firewall Subnets → TGW → Spoke Subnets
```

---

### 🟢 Q38. What is Elastic IP Address?
```bash
# Elastic IP: static public IPv4 address for your AWS account (not tied to instance)
# Use: static IP for NAT GW, public EC2, NLB, application that needs consistent IP

# Allocate EIP
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=myStaticIP}]' \
  --query AllocationId --output text)

# Associate with EC2 instance
aws ec2 associate-address \
  --instance-id i-1234567890abcdef0 \
  --allocation-id $EIP_ALLOC

# Associate with ENI
aws ec2 associate-address \
  --network-interface-id eni-12345678 \
  --allocation-id $EIP_ALLOC \
  --private-ip-address 10.0.1.5   # specific private IP if multiple

# Disassociate and release (stop paying)
aws ec2 disassociate-address --association-id eipassoc-12345678
aws ec2 release-address --allocation-id $EIP_ALLOC

# EIP pricing:
# Running instance with EIP: FREE
# EIP not associated: $0.005/hr
# More than 1 EIP per instance: $0.005/hr each additional
```

---

### 🟡 Q39. What is AWS PrivateLink vs VPC Peering vs VPN?
| Feature | VPC Peering | VPN | PrivateLink |
|---------|------------|-----|------------|
| **Transitive** | ❌ No | ❌ No | N/A |
| **Internet** | No (backbone) | Yes (encrypted) | No (backbone) |
| **Bandwidth** | Unlimited | Limited (~1.25Gbps) | Up to 100Gbps |
| **Latency** | Very low | Higher | Very low |
| **Direction** | Bidirectional | Bidirectional | One-way (service→consumer) |
| **Cost** | Data transfer only | $0.05/hr + data | $0.01/hr + data |
| **Use case** | VPC-to-VPC comms | On-prem to cloud | Service sharing |
| **Overlapping CIDR** | ❌ | ✅ | ✅ |

---

### 🟡 Q40. What is Amazon VPC Lattice?
```bash
# VPC Lattice: application networking for microservices across VPCs and accounts
# Replaces: complex VPC peering + NLB + PrivateLink setup for service mesh
# Handles: service discovery, traffic routing, auth, observability

# Create service network (logical grouping)
aws vpc-lattice create-service-network \
  --name myServiceNetwork \
  --auth-type AWS_IAM

# Create service (represents your microservice)
aws vpc-lattice create-service \
  --name orderService \
  --auth-type AWS_IAM

# Create target group
aws vpc-lattice create-target-group \
  --name orderServiceTG \
  --type INSTANCE \
  --config '{
    "port": 8080,
    "protocol": "HTTP",
    "vpcIdentifier": "vpc-12345678",
    "healthCheck": {"enabled": true, "path": "/health", "protocol": "HTTP"}
  }'

# Register targets
aws vpc-lattice register-targets \
  --target-group-identifier <tg-id> \
  --targets id=i-1234567890abcdef0,port=8080

# Create listener and rules
aws vpc-lattice create-listener \
  --service-identifier <service-id> \
  --protocol HTTP --port 80 \
  --default-action '{
    "forward": {"targetGroups": [{"targetGroupIdentifier": "<tg-id>", "weight": 100}]}
  }'

# Associate service network with VPC
aws vpc-lattice create-service-network-vpc-association \
  --service-network-identifier <sn-id> \
  --vpc-identifier vpc-12345678
```


---

# PART 3 — STORAGE

---

### 🟢 Q41. What is Amazon S3?
```bash
# S3: object storage — unlimited objects, 5TB max per object
# Storage classes: Standard, Intelligent-Tiering, Standard-IA, One Zone-IA,
#                  Glacier Instant, Glacier Flexible, Glacier Deep Archive

# Create bucket
aws s3api create-bucket \
  --bucket my-production-bucket \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1

# Block all public access (best practice)
aws s3api put-public-access-block \
  --bucket my-production-bucket \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,\
    BlockPublicPolicy=true,RestrictPublicBuckets=true

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-production-bucket \
  --versioning-configuration Status=Enabled

# Enable server-side encryption (SSE-KMS)
aws s3api put-bucket-encryption \
  --bucket my-production-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/abc123"
      },
      "BucketKeyEnabled": true
    }]
  }'

# Upload objects
aws s3 cp localfile.txt s3://my-production-bucket/path/file.txt
aws s3 cp s3://my-production-bucket/file.txt ./downloaded.txt
aws s3 sync ./local-folder/ s3://my-production-bucket/backup/ --delete
aws s3 mv s3://my-production-bucket/old.txt s3://my-production-bucket/new.txt

# Multipart upload (for files > 100MB)
aws s3api create-multipart-upload \
  --bucket my-production-bucket --key large-file.zip
# Upload parts, then complete:
aws s3api complete-multipart-upload \
  --bucket my-production-bucket --key large-file.zip \
  --upload-id <upload-id> \
  --multipart-upload file://parts.json

# Generate presigned URL (time-limited access without AWS credentials)
aws s3 presign s3://my-production-bucket/report.pdf \
  --expires-in 3600   # 1 hour

# List versions
aws s3api list-object-versions \
  --bucket my-production-bucket \
  --prefix reports/

# Delete specific version
aws s3api delete-object \
  --bucket my-production-bucket \
  --key reports/q4.pdf \
  --version-id <version-id>
```

---

### 🟢 Q42. What are S3 storage classes?
| Class | Availability | Retrieval | Min Duration | Cost Approx |
|-------|------------|---------|-------------|------------|
| **Standard** | 99.99% | Instant | None | $0.023/GB |
| **Intelligent-Tiering** | 99.9% | Instant/Archive | None | $0.023+monitoring |
| **Standard-IA** | 99.9% | Instant | 30 days | $0.0125/GB |
| **One Zone-IA** | 99.5% | Instant | 30 days | $0.01/GB |
| **Glacier Instant** | 99.9% | Instant | 90 days | $0.004/GB |
| **Glacier Flexible** | 99.99% | 1min–12hr | 90 days | $0.0036/GB |
| **Glacier Deep Archive** | 99.99% | 12–48hr | 180 days | $0.00099/GB |

```bash
# Lifecycle policy (auto-transition between classes)
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-production-bucket \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "TransitionLogsToIA",
        "Status": "Enabled",
        "Filter": {"Prefix": "logs/"},
        "Transitions": [
          {"Days": 30,  "StorageClass": "STANDARD_IA"},
          {"Days": 90,  "StorageClass": "GLACIER_IR"},
          {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
        ],
        "NoncurrentVersionTransitions": [
          {"NoncurrentDays": 30, "StorageClass": "STANDARD_IA"}
        ],
        "NoncurrentVersionExpiration": {"NoncurrentDays": 365},
        "Expiration": {"Days": 2555},
        "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
      }
    ]
  }'

# Change storage class of existing object
aws s3 cp s3://bucket/file.txt s3://bucket/file.txt \
  --storage-class STANDARD_IA \
  --metadata-directive COPY

# Glacier restore (make temporarily accessible)
aws s3api restore-object \
  --bucket my-production-bucket \
  --key archived/report.pdf \
  --restore-request '{
    "Days": 7,
    "GlacierJobParameters": {"Tier": "Expedited"}
  }'
# Tiers: Expedited (1-5 min), Standard (3-5 hrs), Bulk (5-12 hrs) for Glacier Flexible
```

---

### 🟡 Q43. What is S3 Intelligent-Tiering?
```bash
# Intelligent-Tiering: ML-based auto tiering — no retrieval fees
# Tiers (automatic):
# Frequent Access: immediately accessible
# Infrequent Access (30 days): 40% cheaper
# Archive Instant Access (90 days): 68% cheaper
# Optional archive tiers:
# Archive Access (90 days+): 71% cheaper, 3-5hr retrieval
# Deep Archive Access (180 days+): 95% cheaper, 12hr retrieval

# Enable Intelligent-Tiering on bucket
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket my-production-bucket \
  --id myITConfig \
  --intelligent-tiering-configuration '{
    "Id": "myITConfig",
    "Status": "Enabled",
    "Tierings": [
      {"Days": 90,  "AccessTier": "ARCHIVE_ACCESS"},
      {"Days": 180, "AccessTier": "DEEP_ARCHIVE_ACCESS"}
    ]
  }'

# Set default storage class to Intelligent-Tiering for all new objects
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-production-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "DefaultToIT",
      "Status": "Enabled",
      "Filter": {},
      "Transitions": [{"Days": 0, "StorageClass": "INTELLIGENT_TIERING"}]
    }]
  }'
```

---

### 🟡 Q44. What is S3 Replication?
```bash
# Cross-Region Replication (CRR): replicate to different region (DR, compliance)
# Same-Region Replication (SRR): replicate within same region (log aggregation)
# Requires: versioning enabled on both source and destination

# Create replication role
aws iam create-role \
  --role-name S3ReplicationRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{"Effect":"Allow","Principal":{"Service":"s3.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'

# Configure CRR
aws s3api put-bucket-replication \
  --bucket source-bucket \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
    "Rules": [
      {
        "ID": "ReplicateEverything",
        "Status": "Enabled",
        "Filter": {},
        "Destination": {
          "Bucket": "arn:aws:s3:::destination-bucket-eu-west-1",
          "StorageClass": "STANDARD_IA",
          "ReplicationTime": {"Status": "Enabled", "Time": {"Minutes": 15}},
          "Metrics": {"Status": "Enabled", "EventThreshold": {"Minutes": 15}},
          "EncryptionConfiguration": {
            "ReplicaKmsKeyID": "arn:aws:kms:eu-west-1:123456789012:key/eu-key"
          }
        },
        "DeleteMarkerReplication": {"Status": "Enabled"},
        "SourceSelectionCriteria": {
          "SseKmsEncryptedObjects": {"Status": "Enabled"},
          "ReplicaModifications": {"Status": "Enabled"}
        }
      }
    ]
  }'

# Replication Time Control (RTC): 99.99% objects replicated within 15 minutes
# S3 Batch Replication: replicate existing objects (not just new ones)
aws s3control create-job \
  --account-id 123456789012 \
  --operation '{"S3ReplicateObject":{}}' \
  --manifest '{"Spec":{"Format":"S3BatchOperations_CSV_20180820","Fields":["Bucket","Key"]},"Location":{"ObjectArn":"arn:aws:s3:::manifest-bucket/manifest.csv","ETag":"abc123"}}' \
  --report '{"Bucket":"arn:aws:s3:::report-bucket","Prefix":"batch-report","Format":"Report_CSV_20180820","Enabled":true,"ReportScope":"AllTasks"}' \
  --priority 10 \
  --role-arn arn:aws:iam::123456789012:role/S3BatchRole
```

---

### 🟡 Q45. What is S3 Security — bucket policies, ACLs, and access points?
```bash
# Bucket Policy (resource-based IAM policy on bucket)
aws s3api put-bucket-policy \
  --bucket my-production-bucket \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyNonSSL",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": [
          "arn:aws:s3:::my-production-bucket",
          "arn:aws:s3:::my-production-bucket/*"
        ],
        "Condition": {"Bool": {"aws:SecureTransport": "false"}}
      },
      {
        "Sid": "AllowSpecificRoles",
        "Effect": "Allow",
        "Principal": {
          "AWS": [
            "arn:aws:iam::123456789012:role/AppRole",
            "arn:aws:iam::123456789012:role/AnalyticsRole"
          ]
        },
        "Action": ["s3:GetObject","s3:PutObject"],
        "Resource": "arn:aws:s3:::my-production-bucket/*"
      },
      {
        "Sid": "DenyDeleteToEveryone",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:DeleteObject",
        "Resource": "arn:aws:s3:::my-production-bucket/*",
        "Condition": {
          "StringNotLike": {"aws:PrincipalArn": "arn:aws:iam::123456789012:role/AdminRole"}
        }
      },
      {
        "Sid": "AllowCrossAccount",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::987654321098:role/PartnerRole"},
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::my-production-bucket/public-data/*"
      }
    ]
  }'

# S3 Access Point (fine-grained access per application)
aws s3control create-access-point \
  --account-id 123456789012 \
  --name analytics-access-point \
  --bucket my-production-bucket \
  --vpc-configuration VpcId=$VPC_ID   # restrict to VPC only

aws s3control put-access-point-policy \
  --account-id 123456789012 \
  --name analytics-access-point \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:role/AnalyticsRole"},
      "Action": ["s3:GetObject","s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:us-east-1:123456789012:accesspoint/analytics-access-point",
        "arn:aws:s3:us-east-1:123456789012:accesspoint/analytics-access-point/object/analytics/*"
      ]
    }]
  }'

# S3 Object Lock (WORM — Write Once Read Many)
aws s3api put-object-lock-configuration \
  --bucket my-compliance-bucket \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Days": 365
      }
    }
  }'
# Modes: COMPLIANCE (cannot delete even by root) | GOVERNANCE (admin can override)
```

---

### 🟡 Q46. What is S3 Event Notifications?
```bash
# S3 Events: trigger actions when objects are created, deleted, replicated

# Notify SNS on object creation
aws s3api put-bucket-notification-configuration \
  --bucket my-production-bucket \
  --notification-configuration '{
    "TopicConfigurations": [{
      "TopicArn": "arn:aws:sns:us-east-1:123456789012:myTopic",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {"Name": "prefix", "Value": "uploads/"},
            {"Name": "suffix", "Value": ".jpg"}
          ]
        }
      }
    }],
    "LambdaFunctionConfigurations": [{
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:processImage",
      "Events": ["s3:ObjectCreated:Put","s3:ObjectCreated:CompleteMultipartUpload"],
      "Filter": {"Key": {"FilterRules": [{"Name": "prefix", "Value": "raw/"}]}}
    }],
    "QueueConfigurations": [{
      "QueueArn": "arn:aws:sqs:us-east-1:123456789012:imageProcessingQueue",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {"Key": {"FilterRules": [{"Name": "suffix", "Value": ".csv"}]}}
    }]
  }'

# S3 Event Bridge (richer filtering, archive, replay)
aws s3api put-bucket-notification-configuration \
  --bucket my-production-bucket \
  --notification-configuration '{"EventBridgeConfiguration": {}}'

# EventBridge rule
aws events put-rule \
  --name S3ObjectUploaded \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Created"],
    "detail": {
      "bucket": {"name": ["my-production-bucket"]},
      "object": {"key": [{"prefix": "invoices/"}]}
    }
  }'
```

---

### 🟢 Q47. What is Amazon EBS?
```bash
# EBS: block storage volumes for EC2 — like a hard drive attached to an instance
# Types:
# gp3: General Purpose SSD — 3000 IOPS baseline, 125MB/s, up to 16000 IOPS tunable
# gp2: General Purpose SSD — 3 IOPS/GB, burstable, legacy
# io2 Block Express: Provisioned IOPS — up to 256,000 IOPS, sub-ms latency
# io1: Provisioned IOPS — up to 64,000 IOPS
# st1: Throughput Optimized HDD — 500 MB/s, big data, log processing
# sc1: Cold HDD — 250 MB/s, cheapest, infrequent access

# Create gp3 volume
aws ec2 create-volume \
  --volume-type gp3 \
  --size 100 \
  --iops 6000 \
  --throughput 250 \
  --availability-zone us-east-1a \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=myDataVolume}]'

# Create io2 Block Express for databases
aws ec2 create-volume \
  --volume-type io2 \
  --size 500 \
  --iops 50000 \
  --multi-attach-enabled \   # attach to multiple instances (io1/io2 only)
  --availability-zone us-east-1a \
  --encrypted

# Attach to instance
aws ec2 attach-volume \
  --volume-id vol-12345678 \
  --instance-id i-1234567890abcdef0 \
  --device /dev/sdf

# On the instance (format and mount)
# sudo mkfs.ext4 /dev/nvme1n1
# sudo mkdir /data
# sudo mount /dev/nvme1n1 /data
# echo '/dev/nvme1n1 /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab

# Resize volume (online — no detach needed for most types)
aws ec2 modify-volume \
  --volume-id vol-12345678 \
  --size 200 \
  --iops 8000 \
  --throughput 500

# On instance after resize:
# sudo growpart /dev/nvme1n1 1
# sudo resize2fs /dev/nvme1n1p1  # ext4
# sudo xfs_growfs /data           # xfs

# Create snapshot
aws ec2 create-snapshot \
  --volume-id vol-12345678 \
  --description "Daily backup $(date +%Y-%m-%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=daily-backup}]'

# Fast Snapshot Restore (instant access from snapshot)
aws ec2 enable-fast-snapshot-restores \
  --availability-zones us-east-1a us-east-1b \
  --source-snapshot-ids snap-12345678
```

---

### 🟡 Q48. What is Amazon EFS?
```bash
# EFS: shared NFS file system — multiple EC2 instances read/write simultaneously
# Performance modes:
# General Purpose: low latency, max 35,000 IOPS (default)
# Max I/O:         higher aggregate throughput, higher latency
# Throughput modes:
# Bursting:        throughput scales with size (1 MB/s per GB)
# Provisioned:     fixed throughput (for consistent high-throughput workloads)
# Elastic:         automatically adjusts (recommended)
# Storage classes:
# Standard:        frequently accessed
# Standard-IA:     infrequent access (40% cheaper)
# One Zone:        single AZ (cheaper, lower durability)

# Create EFS
aws efs create-file-system \
  --creation-token myEFS-$(date +%s) \
  --performance-mode generalPurpose \
  --throughput-mode elastic \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123 \
  --tags Key=Name,Value=mySharedEFS

EFS_ID=$(aws efs describe-file-systems \
  --query 'FileSystems[-1].FileSystemId' --output text)

# Create mount targets (one per AZ for HA)
for subnet in subnet-11111111 subnet-22222222 subnet-33333333; do
  aws efs create-mount-target \
    --file-system-id $EFS_ID \
    --subnet-id $subnet \
    --security-groups sg-12345678
done

# Enable lifecycle (move to IA after 30 days)
aws efs put-lifecycle-configuration \
  --file-system-id $EFS_ID \
  --lifecycle-policies '[
    {"TransitionToIA": "AFTER_30_DAYS"},
    {"TransitionToPrimaryStorageClass": "AFTER_1_ACCESS"}
  ]'

# Mount from EC2 (requires amazon-efs-utils)
# sudo mount -t efs -o tls,iam $EFS_ID:/ /mnt/efs
# sudo mount -t efs -o tls,accesspoint=fsap-12345678 $EFS_ID:/ /mnt/efs-ap

# /etc/fstab entry
# fs-12345678.efs.us-east-1.amazonaws.com:/ /mnt/efs efs defaults,tls,iam 0 0

# EFS Access Point (application-specific root with POSIX identity)
aws efs create-access-point \
  --file-system-id $EFS_ID \
  --posix-user Uid=1000,Gid=1000 \
  --root-directory Path=/myapp,CreationInfo='{OwnerUid=1000,OwnerGid=1000,Permissions=755}' \
  --tags Key=App,Value=myapp
```

---

### 🟡 Q49. What is Amazon FSx?
```bash
# FSx family — managed file systems:
# FSx for Windows File Server: SMB, Windows ACLs, DFS
# FSx for Lustre:              HPC, ML, high-throughput parallel
# FSx for NetApp ONTAP:        NFS/SMB/iSCSI, enterprise features
# FSx for OpenZFS:             NFS, ZFS snapshots, clones

# FSx for Windows File Server
aws fsx create-file-system \
  --file-system-type WINDOWS \
  --storage-capacity 300 \
  --storage-type SSD \
  --subnet-ids subnet-11111111 subnet-22222222 \
  --security-group-ids sg-12345678 \
  --windows-configuration '{
    "ActiveDirectoryId": "d-1234567890",
    "ThroughputCapacity": 512,
    "DeploymentType": "MULTI_AZ_1",
    "PreferredSubnetId": "subnet-11111111",
    "AutomaticBackupRetentionDays": 30,
    "DailyAutomaticBackupStartTime": "02:00",
    "WeeklyMaintenanceStartTime": "1:02:30",
    "AuditLogConfiguration": {"FileAccessAuditLogLevel":"SUCCESS_AND_FAILURE","FileShareAccessAuditLogLevel":"SUCCESS_AND_FAILURE"}
  }'

# FSx for Lustre (ML training, HPC)
aws fsx create-file-system \
  --file-system-type LUSTRE \
  --storage-capacity 1200 \     # must be multiple of 1200 for persistent
  --storage-type SSD \
  --subnet-ids subnet-11111111 \
  --security-group-ids sg-12345678 \
  --lustre-configuration '{
    "DeploymentType": "PERSISTENT_2",
    "PerUnitStorageThroughput": 500,
    "DataCompressionType": "LZ4",
    "ImportPath": "s3://my-training-data/",
    "ExportPath": "s3://my-training-data/lustre-exports/",
    "AutoImportPolicy": "NEW_CHANGED_DELETED"
  }'
# Mount: sudo mount -t lustre -o relatime,flock fs-12345678.fsx.us-east-1.amazonaws.com@tcp:/fsx /mnt/fsx

# FSx for NetApp ONTAP (most feature-rich — NFS, SMB, iSCSI)
aws fsx create-file-system \
  --file-system-type ONTAP \
  --storage-capacity 1024 \
  --storage-type SSD \
  --subnet-ids subnet-11111111 subnet-22222222 \
  --security-group-ids sg-12345678 \
  --ontap-configuration '{
    "DeploymentType": "MULTI_AZ_1",
    "ThroughputCapacity": 512,
    "PreferredSubnetId": "subnet-11111111",
    "RouteTableIds": ["rtb-11111111"],
    "AutomaticBackupRetentionDays": 30,
    "DiskIopsConfiguration": {"Mode":"USER_PROVISIONED","Iops":20000}
  }'
```

---

### 🟢 Q50. What is AWS S3 Transfer Acceleration?
```bash
# Transfer Acceleration: upload to S3 via CloudFront edge locations (faster long-distance)
# Best for: users uploading to S3 from far geographic regions

# Enable on bucket
aws s3api put-bucket-accelerate-configuration \
  --bucket my-production-bucket \
  --accelerate-configuration Status=Enabled

# Use accelerated endpoint
aws s3 cp largefile.zip \
  s3://my-production-bucket/uploads/largefile.zip \
  --endpoint-url https://my-production-bucket.s3-accelerate.amazonaws.com

# Test if acceleration is beneficial
# Use speed comparison tool: http://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html

# AWS DataSync (for large migrations — on-prem to S3/EFS/FSx)
aws datasync create-agent \
  --activation-key <activation-key> \
  --agent-name myDataSyncAgent

# Create NFS location (on-prem source)
aws datasync create-location-nfs \
  --server-hostname 192.168.1.100 \
  --subdirectory /data/exports \
  --on-prem-config AgentArns=arn:aws:datasync:us-east-1:123456789012:agent/agent-abc123

# Create S3 location (target)
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::my-production-bucket \
  --s3-config BucketAccessRoleArn=arn:aws:iam::123456789012:role/DataSyncRole \
  --subdirectory /migrated-data

# Create and run task
aws datasync create-task \
  --source-location-arn <nfs-location-arn> \
  --destination-location-arn <s3-location-arn> \
  --options OverwriteMode=ALWAYS,PreserveDeletedFiles=PRESERVE,TransferMode=CHANGED \
  --schedule ScheduleExpression="cron(0 2 * * ? *)"
```

---

### 🟡 Q51. What is Amazon Storage Gateway?
```bash
# Storage Gateway: hybrid cloud storage bridge (on-prem to AWS)
# Types:
# S3 File Gateway:       NFS/SMB → S3 (file share backed by S3)
# FSx File Gateway:      SMB → FSx for Windows (low latency cache on-prem)
# Volume Gateway Stored: iSCSI volumes stored on-prem, backed up to S3
# Volume Gateway Cached: iSCSI volumes in S3, frequently accessed cached on-prem
# Tape Gateway:          Virtual tape library (VTL) → S3 Glacier

# Activate S3 File Gateway (after deploying VM/hardware appliance)
aws storagegateway activate-gateway \
  --activation-key <activation-key> \
  --gateway-name myS3FileGateway \
  --gateway-timezone GMT-5:00 \
  --gateway-region us-east-1 \
  --gateway-type FILE_S3

# Create NFS file share (S3 File Gateway)
aws storagegateway create-nfs-file-share \
  --client-token $(date +%s) \
  --gateway-arn arn:aws:storagegateway:us-east-1:123456789012:gateway/sgw-12345678 \
  --location-arn arn:aws:s3:::my-production-bucket \
  --role arn:aws:iam::123456789012:role/StorageGatewayRole \
  --client-list 10.0.0.0/8 \
  --default-storage-class S3_INTELLIGENT_TIERING \
  --guess-mime-type-enabled true

# Create SMB file share
aws storagegateway create-smb-file-share \
  --client-token $(date +%s) \
  --gateway-arn arn:aws:storagegateway:us-east-1:123456789012:gateway/sgw-12345678 \
  --location-arn arn:aws:s3:::my-production-bucket \
  --role arn:aws:iam::123456789012:role/StorageGatewayRole \
  --authentication ActiveDirectory \
  --access-based-enumeration true \
  --valid-user-list domain\\finance-team
```

---

### 🟢 Q52. What is AWS Backup?
```bash
# AWS Backup: centralised backup across EC2, EBS, RDS, DynamoDB, EFS, FSx, S3

# Create backup vault
aws backup create-backup-vault \
  --backup-vault-name myBackupVault \
  --encryption-key-arn arn:aws:kms:us-east-1:123456789012:key/abc123

# Create backup plan
aws backup create-backup-plan \
  --backup-plan '{
    "BackupPlanName": "DailyWeeklyMonthly",
    "Rules": [
      {
        "RuleName": "DailyBackup",
        "TargetBackupVaultName": "myBackupVault",
        "ScheduleExpression": "cron(0 2 * * ? *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 360,
        "Lifecycle": {"DeleteAfterDays": 30},
        "EnableContinuousBackup": true
      },
      {
        "RuleName": "WeeklyBackup",
        "TargetBackupVaultName": "myBackupVault",
        "ScheduleExpression": "cron(0 1 ? * SUN *)",
        "Lifecycle": {"DeleteAfterDays": 90}
      },
      {
        "RuleName": "MonthlyBackup",
        "TargetBackupVaultName": "myBackupVault",
        "ScheduleExpression": "cron(0 0 1 * ? *)",
        "CopyActions": [{
          "DestinationBackupVaultArn": "arn:aws:backup:eu-west-1:123456789012:backup-vault:DR-Vault",
          "Lifecycle": {"DeleteAfterDays": 365}
        }],
        "Lifecycle": {"DeleteAfterDays": 365}
      }
    ]
  }'

PLAN_ID=$(aws backup list-backup-plans \
  --query 'BackupPlansList[-1].BackupPlanId' --output text)

# Assign resources to backup plan
aws backup create-backup-selection \
  --backup-plan-id $PLAN_ID \
  --backup-selection '{
    "SelectionName": "AllTaggedResources",
    "IamRoleArn": "arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole",
    "ListOfTags": [
      {"ConditionType":"STRINGEQUALS","ConditionKey":"Backup","ConditionValue":"true"}
    ]
  }'

# On-demand backup
aws backup start-backup-job \
  --backup-vault-name myBackupVault \
  --resource-arn arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0 \
  --iam-role-arn arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole \
  --lifecycle DeleteAfterDays=30

# Restore from backup
aws backup start-restore-job \
  --recovery-point-arn arn:aws:backup:us-east-1:123456789012:recovery-point:rp-12345678 \
  --iam-role-arn arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole \
  --resource-type EC2 \
  --metadata '{
    "InstanceType": "t3.medium",
    "SubnetId": "subnet-12345678",
    "SecurityGroupIds": "sg-12345678"
  }'
```

---

### 🟡 Q53. What is Amazon S3 analytics and querying?
```bash
# S3 Select: query CSV/JSON/Parquet content with SQL without downloading
aws s3api select-object-content \
  --bucket my-production-bucket \
  --key data/sales.csv \
  --expression "SELECT s._1, s._2, CAST(s._3 AS DECIMAL) FROM S3Object s WHERE CAST(s._3 AS DECIMAL) > 1000" \
  --expression-type SQL \
  --input-serialization '{"CSV":{"FileHeaderInfo":"USE","RecordDelimiter":"\n","FieldDelimiter":","}}' \
  --output-serialization '{"CSV":{"RecordDelimiter":"\n","FieldDelimiter":","}}' \
  output.csv

# Athena (SQL queries on S3 — serverless)
aws athena start-query-execution \
  --query-string "SELECT customer_id, SUM(amount) AS total_spend, COUNT(*) AS orders
                  FROM sales_db.transactions
                  WHERE year='2026' AND month='06'
                  GROUP BY customer_id
                  HAVING SUM(amount) > 1000
                  ORDER BY total_spend DESC
                  LIMIT 100" \
  --query-execution-context Database=sales_db \
  --result-configuration OutputLocation=s3://my-athena-results/ \
  --work-group myWorkGroup

# Get results
aws athena get-query-results \
  --query-execution-id <execution-id>

# S3 Storage Lens (storage analytics across organisation)
aws s3control put-storage-lens-configuration \
  --account-id 123456789012 \
  --config-id myLensConfig \
  --storage-lens-configuration '{
    "Id": "myLensConfig",
    "IsEnabled": true,
    "DataExport": {
      "S3BucketDestination": {
        "Format": "Parquet",
        "OutputSchemaVersion": "V_1",
        "AccountId": "123456789012",
        "Arn": "arn:aws:s3:::my-lens-results"
      }
    },
    "AccountLevel": {
      "ActivityMetrics": {"IsEnabled": true},
      "BucketLevel": {
        "ActivityMetrics": {"IsEnabled": true},
        "PrefixLevel": {"StorageMetrics": {"IsEnabled": true, "SelectionCriteria": {"MaxDepth": 5}}}
      }
    }
  }'
```


---

# PART 4 — DATABASES

---

### 🟢 Q54. What is Amazon RDS?
```bash
# RDS: managed relational DB — MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2
# AWS handles: backups, patching, HA, failover, replication

# Create RDS PostgreSQL instance
aws rds create-db-instance \
  --db-instance-identifier myProdDB \
  --db-instance-class db.r6g.xlarge \
  --engine postgres \
  --engine-version 16.3 \
  --master-username dbadmin \
  --master-user-password "MySecureP@ss!" \
  --allocated-storage 100 \
  --storage-type gp3 \
  --iops 3000 \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123 \
  --vpc-security-group-ids sg-12345678 \
  --db-subnet-group-name myDBSubnetGroup \
  --multi-az \
  --auto-minor-version-upgrade true \
  --backup-retention-period 35 \
  --preferred-backup-window "02:00-03:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --deletion-protection \
  --enable-cloudwatch-logs-exports postgresql upgrade \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role \
  --enable-performance-insights \
  --performance-insights-retention-period 731 \
  --tags Key=Env,Value=prod

# Create read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier myProdDB-replica \
  --source-db-instance-identifier myProdDB \
  --db-instance-class db.r6g.large \
  --availability-zone us-east-1b \
  --publicly-accessible false \
  --auto-minor-version-upgrade true \
  --replica-mode open-read-only

# Cross-region read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier myProdDB-eu-replica \
  --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:myProdDB \
  --db-instance-class db.r6g.large \
  --region eu-west-1 \
  --source-region us-east-1 \
  --kms-key-id arn:aws:kms:eu-west-1:123456789012:key/eu-key

# Point-in-time restore (any second in backup window)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier myProdDB \
  --target-db-instance-identifier myProdDB-restored \
  --restore-time "2026-06-10T10:00:00Z" \
  --db-instance-class db.r6g.large \
  --vpc-security-group-ids sg-12345678 \
  --db-subnet-group-name myDBSubnetGroup

# Failover to standby (Multi-AZ)
aws rds reboot-db-instance \
  --db-instance-identifier myProdDB \
  --force-failover

# Modify instance (upgrade instance class)
aws rds modify-db-instance \
  --db-instance-identifier myProdDB \
  --db-instance-class db.r6g.2xlarge \
  --apply-immediately false   # apply during maintenance window

# RDS Proxy (connection pooling for Lambda/serverless)
aws rds create-db-proxy \
  --db-proxy-name myRDSProxy \
  --engine-family POSTGRESQL \
  --auth '[{
    "Description": "Use Secrets Manager",
    "AuthScheme": "SECRETS",
    "SecretArn": "arn:aws:secretsmanager:us-east-1:123456789012:secret:rds-creds",
    "IAMAuth": "REQUIRED"
  }]' \
  --role-arn arn:aws:iam::123456789012:role/rds-proxy-role \
  --vpc-subnet-ids subnet-11111111 subnet-22222222 \
  --vpc-security-group-ids sg-12345678 \
  --require-tls

# Associate proxy with DB
aws rds register-db-proxy-targets \
  --db-proxy-name myRDSProxy \
  --db-instance-identifiers myProdDB
```

---

### 🟡 Q55. What is Amazon Aurora?
```bash
# Aurora: AWS-native compatible MySQL/PostgreSQL
# 5x faster than MySQL, 3x faster than PostgreSQL
# Storage: auto-scales 10GB–128TB, 6 copies across 3 AZs
# Replication: sub-10ms to read replicas (vs standard RDS async)

# Create Aurora Cluster
aws rds create-db-cluster \
  --db-cluster-identifier myAuroraCluster \
  --engine aurora-postgresql \
  --engine-version 16.3 \
  --master-username dbadmin \
  --master-user-password "MySecureP@ss!" \
  --database-name myapp \
  --vpc-security-group-ids sg-12345678 \
  --db-subnet-group-name myDBSubnetGroup \
  --backup-retention-period 35 \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123 \
  --deletion-protection \
  --enable-cloudwatch-logs-exports postgresql \
  --enable-http-endpoint \   # Data API
  --serverlessv2-scaling-configuration MinCapacity=0.5,MaxCapacity=128

# Create writer instance
aws rds create-db-instance \
  --db-cluster-identifier myAuroraCluster \
  --db-instance-identifier myAuroraCluster-writer \
  --db-instance-class db.serverless \
  --engine aurora-postgresql

# Add reader instances
aws rds create-db-instance \
  --db-cluster-identifier myAuroraCluster \
  --db-instance-identifier myAuroraCluster-reader-1 \
  --db-instance-class db.serverless \
  --engine aurora-postgresql

# Aurora connection endpoints:
# Cluster endpoint:  Writer (primary)
# Reader endpoint:   Load-balanced across readers
# Instance endpoint: Specific instance

# Aurora Global Database (< 1 second cross-region replication)
aws rds create-global-cluster \
  --global-cluster-identifier myGlobalCluster \
  --source-db-cluster-identifier arn:aws:rds:us-east-1:123456789012:cluster:myAuroraCluster \
  --deletion-protection

# Add secondary region
aws rds create-db-cluster \
  --global-cluster-identifier myGlobalCluster \
  --db-cluster-identifier myAuroraCluster-eu \
  --engine aurora-postgresql \
  --db-subnet-group-name myEUDBSubnetGroup \
  --vpc-security-group-ids sg-eu-12345678 \
  --region eu-west-1

# Managed failover to secondary (RTO < 1 min)
aws rds failover-global-cluster \
  --global-cluster-identifier myGlobalCluster \
  --target-db-cluster-identifier arn:aws:rds:eu-west-1:123456789012:cluster:myAuroraCluster-eu

# Aurora Serverless v2: scales per 0.5 ACU, instant (no cold start)
# 1 ACU = ~2GB RAM
# Scales from 0.5 to 128 ACU based on load
# Ideal for: variable workloads, dev/test, microservices

# Aurora Zero-ETL integration with Redshift
aws rds create-integration \
  --source-arn arn:aws:rds:us-east-1:123456789012:cluster:myAuroraCluster \
  --target-arn arn:aws:redshift:us-east-1:123456789012:namespace:my-namespace \
  --integration-name aurora-to-redshift \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123
```

---

### 🟡 Q56. What is Amazon DynamoDB?
```bash
# DynamoDB: fully managed, serverless NoSQL — key-value and document model
# Performance: single-digit millisecond at any scale
# Primary key: Partition key (hash) + optional Sort key (range)

# Create table
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
    AttributeName=OrderId,AttributeType=S \
    AttributeName=CustomerId,AttributeType=S \
    AttributeName=CreatedAt,AttributeType=S \
  --key-schema \
    AttributeName=OrderId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=arn:aws:kms:... \
  --deletion-protection-enabled \
  --tags Key=Env,Value=prod \
  --global-secondary-indexes '[
    {
      "IndexName": "CustomerIdIndex",
      "KeySchema": [
        {"AttributeName": "CustomerId", "KeyType": "HASH"},
        {"AttributeName": "CreatedAt", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }
  ]'

# Table modes:
# On-demand (PAY_PER_REQUEST): no capacity planning, auto-scales, pay per request
# Provisioned: specify RCU/WCU, cheaper for predictable loads

# CRUD operations
aws dynamodb put-item \
  --table-name Orders \
  --item '{
    "OrderId":    {"S": "ORD-12345"},
    "CustomerId": {"S": "C001"},
    "Amount":     {"N": "299.99"},
    "Status":     {"S": "PENDING"},
    "Items":      {"L": [{"M": {"Name": {"S": "Widget"}, "Qty": {"N": "2"}}}]},
    "CreatedAt":  {"S": "2026-06-13T10:00:00Z"},
    "TTL":        {"N": "1765641600"}
  }' \
  --condition-expression "attribute_not_exists(OrderId)"

aws dynamodb get-item \
  --table-name Orders \
  --key '{"OrderId": {"S": "ORD-12345"}}' \
  --consistent-read

aws dynamodb update-item \
  --table-name Orders \
  --key '{"OrderId": {"S": "ORD-12345"}}' \
  --update-expression "SET #s = :new_status, UpdatedAt = :ts ADD Version :inc" \
  --expression-attribute-names '{"#s": "Status"}' \
  --expression-attribute-values '{
    ":new_status": {"S": "SHIPPED"},
    ":ts": {"S": "2026-06-13T12:00:00Z"},
    ":inc": {"N": "1"}
  }' \
  --condition-expression "#s = :expected_status" \
  --expression-attribute-values ':expected_status: {"S": "PENDING"}' \
  --return-values ALL_NEW

aws dynamodb delete-item \
  --table-name Orders \
  --key '{"OrderId": {"S": "ORD-12345"}}'

# Query (uses index — efficient)
aws dynamodb query \
  --table-name Orders \
  --index-name CustomerIdIndex \
  --key-condition-expression "CustomerId = :cid AND CreatedAt BETWEEN :start AND :end" \
  --filter-expression "#s = :status" \
  --expression-attribute-names '{"#s": "Status"}' \
  --expression-attribute-values '{
    ":cid": {"S": "C001"},
    ":start": {"S": "2026-06-01"},
    ":end": {"S": "2026-06-30"},
    ":status": {"S": "SHIPPED"}
  }' \
  --limit 100 \
  --scan-index-forward false   # newest first

# Batch operations
aws dynamodb batch-write-item \
  --request-items '{
    "Orders": [
      {"PutRequest": {"Item": {"OrderId": {"S": "ORD-1"}, "Status": {"S": "NEW"}}}},
      {"PutRequest": {"Item": {"OrderId": {"S": "ORD-2"}, "Status": {"S": "NEW"}}}},
      {"DeleteRequest": {"Key": {"OrderId": {"S": "ORD-OLD"}}}}
    ]
  }'

# Transactional writes (ACID — all-or-nothing)
aws dynamodb transact-write-items \
  --transact-items '[
    {
      "Put": {
        "TableName": "Orders",
        "Item": {"OrderId":{"S":"ORD-NEW"},"Status":{"S":"NEW"}},
        "ConditionExpression": "attribute_not_exists(OrderId)"
      }
    },
    {
      "Update": {
        "TableName": "Inventory",
        "Key": {"ProductId":{"S":"P001"}},
        "UpdateExpression": "ADD Stock :dec",
        "ExpressionAttributeValues": {":dec":{"N":"-1"},":min":{"N":"0"}},
        "ConditionExpression": "Stock > :min"
      }
    }
  ]'

# DynamoDB Streams + Lambda trigger
aws lambda create-event-source-mapping \
  --function-name processOrderChanges \
  --event-source-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders/stream/2026-01-01 \
  --starting-position LATEST \
  --batch-size 100 \
  --filter-criteria '{"Filters":[{"Pattern":"{\"eventName\":[\"INSERT\"]}"}]}' \
  --bisect-batch-on-function-error true
```

---

### 🟡 Q57. What are DynamoDB Accelerator (DAX) and Global Tables?
```bash
# DAX: in-memory cache for DynamoDB — microsecond reads (vs millisecond)
# Compatible with DynamoDB API — just change endpoint
# Not suitable for: strongly consistent reads, write-heavy without reads

# Create DAX cluster
aws dax create-cluster \
  --cluster-name myDAXCluster \
  --node-type dax.r5.large \
  --replication-factor 3 \
  --iam-role-arn arn:aws:iam::123456789012:role/DAXRole \
  --subnet-group-name myDAXSubnetGroup \
  --security-group-ids sg-12345678 \
  --sse-specification Enabled=true \
  --cluster-endpoint-encryption-type TLS

# Use DAX in application (just change endpoint)
import amazondax, boto3
dax_endpoint = "mycluster.abc123.dax-clusters.us-east-1.amazonaws.com"
dax = amazondax.AmazonDaxClient.resource(
    endpoints=[dax_endpoint], region_name="us-east-1"
)
table = dax.Table("Orders")
response = table.get_item(Key={"OrderId": "ORD-12345"})

# DynamoDB Global Tables (multi-region, active-active)
aws dynamodb update-table \
  --table-name Orders \
  --replica-updates '[
    {"Create": {"RegionName": "eu-west-1"}},
    {"Create": {"RegionName": "ap-southeast-1"}}
  ]'

# Global Tables features:
# Active-active: write to any region
# Last-writer-wins conflict resolution
# < 1 second replication between regions
# Region-level DynamoDB Streams in each replica

# DynamoDB TTL (auto-delete expired items)
aws dynamodb update-time-to-live \
  --table-name Orders \
  --time-to-live-specification Enabled=true,AttributeName=TTL

# Set TTL when creating item
# "TTL": {"N": "1765641600"}  # Unix timestamp of expiration
```

---

### 🟢 Q58. What is Amazon ElastiCache?
```bash
# ElastiCache: managed in-memory caching
# Redis (recommended): data structures, persistence, pub/sub, Lua scripting
# Memcached: simple multi-threaded, no persistence, no replication

# Create Redis cluster (Serverless — auto-scales, simplest)
aws elasticache create-serverless-cache \
  --serverless-cache-name myRedisServerless \
  --engine redis \
  --major-engine-version 7 \
  --cache-usage-limits DataStorage={Maximum=10,Unit=GB},ECPUPerSecond={Maximum=5000} \
  --subnet-ids subnet-11111111 subnet-22222222 \
  --security-group-ids sg-12345678 \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123

# Create Redis cluster (provisioned — more control)
aws elasticache create-replication-group \
  --replication-group-id myRedisCluster \
  --description "Production Redis Cache" \
  --cache-node-type cache.r7g.large \
  --engine redis \
  --engine-version 7.1 \
  --num-cache-clusters 3 \               # 1 primary + 2 replicas
  --automatic-failover-enabled \
  --multi-az-enabled \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled \
  --auth-token "MyRedisPassword123!" \
  --cache-subnet-group-name myRedisSubnetGroup \
  --security-group-ids sg-12345678 \
  --snapshot-retention-limit 7 \
  --tags Key=Env,Value=prod

# Redis Cluster Mode (sharding — horizontal scale)
aws elasticache create-replication-group \
  --replication-group-id myRedisSharded \
  --description "Sharded Redis" \
  --cache-node-type cache.r7g.large \
  --num-node-groups 3 \                  # 3 shards
  --replicas-per-node-group 2 \          # 2 replicas per shard
  --automatic-failover-enabled \
  --cluster-mode Enabled

# ElastiCache use patterns (Python redis-py)
import redis

r = redis.Redis(
    host='myrediscluster.abc123.ng.0001.use1.cache.amazonaws.com',
    port=6379,
    ssl=True,
    password='MyRedisPassword123!',
    decode_responses=True
)

# Cache-aside pattern
def get_user(user_id: str) -> dict:
    cached = r.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    user = db.query_user(user_id)
    r.setex(f"user:{user_id}", 3600, json.dumps(user))  # 1hr TTL
    return user

# Session storage
r.hset(f"session:{session_id}", mapping={
    "user_id": "U001", "role": "admin", "created": "2026-06-13"
})
r.expire(f"session:{session_id}", 86400)  # 24hr TTL

# Rate limiting
def rate_limit(client_ip: str, limit: int = 100) -> bool:
    key = f"ratelimit:{client_ip}:{int(time.time() / 60)}"  # per minute
    count = r.incr(key)
    if count == 1:
        r.expire(key, 60)
    return count <= limit

# Pub/Sub
r.publish("orders:new", json.dumps({"orderId": "ORD-12345"}))
```

---

### 🟡 Q59. What is Amazon Redshift?
```bash
# Redshift: managed data warehouse — petabyte-scale analytics with SQL
# Columnar storage + MPP (Massively Parallel Processing)

# Create Redshift Serverless (no cluster management)
aws redshift-serverless create-namespace \
  --namespace-name myNamespace \
  --db-name analytics \
  --admin-username rsadmin \
  --admin-user-password "MyRedshiftP@ss!" \
  --iam-roles arn:aws:iam::123456789012:role/RedshiftRole \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123

aws redshift-serverless create-workgroup \
  --workgroup-name myWorkgroup \
  --namespace-name myNamespace \
  --base-capacity 32 \          # RPU (Redshift Processing Units) — can scale to 512
  --publicly-accessible false \
  --subnet-ids subnet-11111111 subnet-22222222 \
  --security-group-ids sg-12345678

# Create provisioned cluster
aws redshift create-cluster \
  --cluster-identifier myRedshiftCluster \
  --node-type ra3.4xlarge \
  --number-of-nodes 4 \
  --master-username rsadmin \
  --master-user-password "MyRedshiftP@ss!" \
  --db-name analytics \
  --vpc-security-group-ids sg-12345678 \
  --cluster-subnet-group-name myRedshiftSubnetGroup \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123 \
  --enhanced-vpc-routing \
  --automated-snapshot-retention-period 35 \
  --enable-logging BucketName=my-redshift-logs,S3KeyPrefix=audit

# Load data from S3 (COPY command)
# COPY orders FROM 's3://my-data-bucket/orders/'
# IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftRole'
# FORMAT AS PARQUET;

# Redshift Spectrum (query S3 directly without loading)
# CREATE EXTERNAL SCHEMA spectrum FROM DATA CATALOG
# DATABASE 'my_glue_database'
# IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftRole';
#
# SELECT o.order_id, c.customer_name, o.amount
# FROM spectrum.orders o
# JOIN local_customers c ON o.customer_id = c.customer_id
# WHERE o.order_date >= '2026-01-01';

# Redshift ML (CREATE MODEL with SageMaker integration)
# CREATE MODEL churn_model
# FROM (SELECT * FROM customer_features WHERE split = 'train')
# TARGET churned
# FUNCTION predict_churn
# IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftMLRole';
#
# SELECT customer_id, predict_churn(age, tenure, monthly_spend) AS churn_prob
# FROM customers WHERE split = 'test';

# Pause/resume cluster (save cost for dev/analytics)
aws redshift pause-cluster --cluster-identifier myRedshiftCluster
aws redshift resume-cluster --cluster-identifier myRedshiftCluster
```

---

### 🟡 Q60. What is Amazon DocumentDB?
```bash
# DocumentDB: MongoDB-compatible managed document database
# Compatible with MongoDB 3.6, 4.0, 5.0 drivers
# Scales storage automatically up to 128TB

# Create cluster
aws docdb create-db-cluster \
  --db-cluster-identifier myDocDBCluster \
  --engine docdb \
  --engine-version 5.0.0 \
  --master-username docadmin \
  --master-user-password "MyDocDBP@ss!" \
  --vpc-security-group-ids sg-12345678 \
  --db-subnet-group-name myDocDBSubnetGroup \
  --backup-retention-period 35 \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123 \
  --deletion-protection \
  --enable-cloudwatch-logs-exports profiler audit

# Create instances
aws docdb create-db-instance \
  --db-cluster-identifier myDocDBCluster \
  --db-instance-identifier myDocDBCluster-primary \
  --db-instance-class db.r6g.xlarge \
  --engine docdb

aws docdb create-db-instance \
  --db-cluster-identifier myDocDBCluster \
  --db-instance-identifier myDocDBCluster-replica \
  --db-instance-class db.r6g.large \
  --engine docdb

# Use with pymongo
import pymongo
client = pymongo.MongoClient(
    "mongodb://docadmin:MyDocDBP@ss@myDocDBCluster.cluster-abc123.us-east-1.docdb.amazonaws.com:27017",
    tls=True,
    tlsCAFile="global-bundle.pem",
    retryWrites=False   # DocumentDB doesn't support retryable writes
)
db = client.myapp
collection = db.products

# CRUD
collection.insert_one({"name":"Widget","price":29.99,"stock":100,"tags":["electronics","gadget"]})
product = collection.find_one({"name": "Widget"})
collection.update_one({"name":"Widget"},{"$set":{"price":24.99},"$inc":{"stock":-1}})
collection.create_index([("name",1)], unique=True)
cursor = collection.find({"price":{"$lt":50},"tags":"electronics"}).sort("price",1).limit(10)
```

---

### 🟡 Q61. What is Amazon Keyspaces (Cassandra)?
```bash
# Keyspaces: serverless Apache Cassandra-compatible managed DB
# CQL (Cassandra Query Language) compatible
# Auto-scales, multi-AZ, serverless

# Create keyspace
aws keyspaces create-keyspace --keyspace-name myKeyspace

# Create table (CQL via cqlsh or API)
# CREATE TABLE myKeyspace.sensor_data (
#   device_id TEXT,
#   timestamp TIMESTAMP,
#   temperature DECIMAL,
#   humidity DECIMAL,
#   location TEXT,
#   PRIMARY KEY (device_id, timestamp)
# ) WITH CLUSTERING ORDER BY (timestamp DESC)
#   AND CUSTOM_PROPERTIES = {'capacity_mode':{'throughput_mode':'PAY_PER_REQUEST'}};

# Create table via API
aws keyspaces create-table \
  --keyspace-name myKeyspace \
  --table-name sensor_data \
  --schema-definition '{
    "allColumns": [
      {"name":"device_id","type":"text"},
      {"name":"timestamp","type":"timestamp"},
      {"name":"temperature","type":"decimal"},
      {"name":"humidity","type":"decimal"}
    ],
    "partitionKeys": [{"name":"device_id"}],
    "clusteringKeys": [{"name":"timestamp","orderBy":"DESC"}]
  }' \
  --capacity-specification throughputMode=PAY_PER_REQUEST \
  --encryption-specification type=CUSTOMER_MANAGED_KMS_KEY,kmsKeyIdentifier=arn:aws:kms:... \
  --point-in-time-recovery status=ENABLED

# Keyspaces use cases:
# IoT sensor time-series data
# User activity tracking
# Recommendation data
# Gaming leaderboards with partition key = game_id
```

---

### 🟡 Q62. What is Amazon Neptune?
```bash
# Neptune: managed graph database — property graphs and RDF/SPARQL
# Query languages: Gremlin, SPARQL, openCypher
# Use cases: social networks, fraud detection, knowledge graphs, recommendations

# Create Neptune cluster
aws neptune create-db-cluster \
  --db-cluster-identifier myNeptuneCluster \
  --engine neptune \
  --engine-version 1.3.0.0 \
  --vpc-security-group-ids sg-12345678 \
  --db-subnet-group-name myNeptuneSubnetGroup \
  --backup-retention-period 35 \
  --storage-encrypted \
  --deletion-protection \
  --enable-cloudwatch-logs-exports audit

aws neptune create-db-instance \
  --db-cluster-identifier myNeptuneCluster \
  --db-instance-identifier myNeptuneCluster-primary \
  --db-instance-class db.r6g.xlarge \
  --engine neptune

# Gremlin queries (property graph traversal)
# g.V().hasLabel('Person').has('name','Alice')   -- find Alice
#   .out('KNOWS')                                 -- people she knows
#   .out('KNOWS')                                 -- 2nd degree connections
#   .has('age',gte(25))                           -- adults only
#   .dedup()                                      -- no duplicates
#   .values('name')                               -- return names
#   .order().by()
#   .limit(10)

# openCypher (Neo4j-compatible)
# MATCH (p:Person {name: 'Alice'})-[:KNOWS*1..3]-(friend:Person)
# WHERE friend.age >= 25 AND NOT friend = p
# RETURN DISTINCT friend.name, friend.age
# ORDER BY friend.name LIMIT 10

# Neptune ML (graph ML predictions)
# Install NeptuneML → SageMaker trains GNN (Graph Neural Network)
# Predict: link prediction, node classification, graph classification
```

---

### 🟢 Q63. What is Amazon MemoryDB for Redis?
```bash
# MemoryDB: Redis-compatible in-memory DB with full durability
# vs ElastiCache Redis: MemoryDB = primary database; ElastiCache = cache
# Multi-AZ, microsecond reads, single-digit ms writes
# Durability: all data written to Multi-AZ transaction log

# Create MemoryDB cluster
aws memorydb create-cluster \
  --cluster-name myMemoryDBCluster \
  --node-type db.r7g.xlarge \
  --acl-name myACL \
  --subnet-group-name myMemoryDBSubnetGroup \
  --security-group-ids sg-12345678 \
  --num-shards 3 \
  --num-replicas-per-shard 2 \
  --tls-enabled \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123 \
  --snapshot-retention-limit 35 \
  --tags Key=Env,Value=prod \
  --engine-version 7.1

# MemoryDB use cases:
# Real-time leaderboards (sorted sets)
# Session storage (primary store, not just cache)
# Real-time fraud detection (sub-millisecond)
# Gaming state management
# Shopping cart (durable, low latency)
```

---

### 🟡 Q64. What is Amazon Timestream?
```bash
# Timestream: serverless time-series database (IoT, DevOps, application metrics)
# Automatic tiering: recent data in memory, historical on SSD
# 1000x faster and 1/10th cost of relational DBs for time-series

# Create database
aws timestream-write create-database \
  --database-name myTimeseriesDB \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/abc123

# Create table with retention
aws timestream-write create-table \
  --database-name myTimeseriesDB \
  --table-name deviceMetrics \
  --retention-properties MemoryStoreRetentionPeriodInHours=24,MagneticStoreRetentionPeriodInDays=365 \
  --magnetic-store-write-properties '{"EnableMagneticStoreWrites":true}'

# Write records
aws timestream-write write-records \
  --database-name myTimeseriesDB \
  --table-name deviceMetrics \
  --common-attributes '{"Dimensions":[{"Name":"region","Value":"us-east-1"}],"MeasureValueType":"DOUBLE","Time":"'$(date +%s000)'","TimeUnit":"MILLISECONDS"}' \
  --records '[
    {"Dimensions":[{"Name":"deviceId","Value":"sensor-001"}],"MeasureName":"temperature","MeasureValue":"23.5"},
    {"Dimensions":[{"Name":"deviceId","Value":"sensor-001"}],"MeasureName":"humidity","MeasureValue":"65.2"},
    {"Dimensions":[{"Name":"deviceId","Value":"sensor-002"}],"MeasureName":"temperature","MeasureValue":"24.1"}
  ]'

# Query (SQL-like with time functions)
aws timestream-query query \
  --query-string "
    SELECT device_id,
           BIN(time, 5m) AS time_bin,
           AVG(temperature) AS avg_temp,
           MAX(temperature) AS max_temp,
           STDDEV(temperature) AS temp_stddev
    FROM myTimeseriesDB.deviceMetrics
    WHERE measure_name = 'temperature'
      AND device_id = 'sensor-001'
      AND time BETWEEN ago(1h) AND now()
    GROUP BY device_id, BIN(time, 5m)
    ORDER BY time_bin DESC"
```

---

### 🟡 Q65. What is Amazon QLDB?
```bash
# QLDB: Quantum Ledger DB — immutable, cryptographically verifiable transaction log
# No deletes — append-only journal (but tables can delete/update, history preserved)
# Use: financial records, supply chain, healthcare audit trail

# Create ledger
aws qldb create-ledger \
  --name myLedger \
  --permissions-mode STANDARD \
  --deletion-protection \
  --kms-key "arn:aws:kms:us-east-1:123456789012:key/abc123"

# QLDB uses PartiQL (SQL-compatible)
# CREATE TABLE BankTransactions
# INSERT INTO BankTransactions VALUE {
#   'TransactionId': 'TXN-001', 'Amount': 500.00,
#   'FromAccount': 'ACC-A', 'ToAccount': 'ACC-B',
#   'Timestamp': `2026-06-13T10:00:00Z`
# }

# Verify history (cryptographic proof)
# SELECT * FROM history(BankTransactions)
# WHERE metadata.id = 'document-id-here'

# Export journal to S3
aws qldb export-journal-to-s3 \
  --name myLedger \
  --inclusive-start-time "2026-01-01T00:00:00Z" \
  --exclusive-end-time "2026-06-13T00:00:00Z" \
  --s3-export-configuration '{
    "Bucket": "my-qldb-exports",
    "Prefix": "exports/",
    "EncryptionConfiguration": {"ObjectEncryptionType": "SSE_KMS", "KmsKeyArn": "arn:aws:kms:..."}
  }' \
  --role-arn arn:aws:iam::123456789012:role/QLDBExportRole
```

---

## COMPLETE Q&A INDEX

| Q# | Question | Level | Part |
|----|---------|-------|------|
| **COMPUTE (Q1–Q20)** | | | |
| Q1 | EC2 instance families — all types, use cases | 🟢 | Compute |
| Q2 | Stop vs Hibernate vs Terminate | 🟢 | Compute |
| Q3 | EC2 purchasing options — On-Demand/Reserved/Spot/Dedicated | 🟢 | Compute |
| Q4 | AMIs — create, copy, share, deprecate, deregister | 🟢 | Compute |
| Q5 | EC2 Auto Scaling — LT, ASG, all 4 scaling policies, warm pools | 🟡 | Compute |
| Q6 | IMDSv1 vs IMDSv2 — SSRF protection, tokens | 🟡 | Compute |
| Q7 | Placement groups — cluster, spread, partition | 🟡 | Compute |
| Q8 | AWS Lambda — limits, triggers, aliases, function URLs | 🟡 | Compute |
| Q9 | Lambda concurrency — reserved, provisioned, cold starts | 🟡 | Compute |
| Q10 | Amazon ECS — task definitions, services, ECS Exec | 🟡 | Compute |
| Q11 | Amazon EKS — cluster, node groups, Fargate profiles | 🟡 | Compute |
| Q12 | AWS Elastic Beanstalk — PaaS, .ebextensions | 🟡 | Compute |
| Q13 | AWS Fargate — CPU/memory, Fargate Spot, EFS mount | 🟡 | Compute |
| Q14 | AWS Step Functions — all state types, WaitForTaskToken | 🟡 | Compute |
| Q15 | Amazon ECR — scan, lifecycle, replication, cross-account | 🟢 | Compute |
| Q16 | AWS CloudFormation — full template, change sets, drift | 🟡 | Compute |
| Q17 | AWS Batch — compute env, job queue, array jobs | 🟡 | Compute |
| Q18 | Elastic Load Balancing — ALB/NLB, listeners, rules | 🟡 | Compute |
| Q19 | ECS vs EKS vs Lambda vs Fargate comparison | 🟢 | Compute |
| Q20 | Application Auto Scaling — ECS, DynamoDB, Aurora | 🟡 | Compute |
| **NETWORKING (Q21–Q40)** | | | |
| Q21 | Amazon VPC — subnets, IGW, route tables | 🟢 | Network |
| Q22 | Security Groups vs NACLs — stateful vs stateless | 🟢 | Network |
| Q23 | NAT Gateway vs NAT Instance | 🟢 | Network |
| Q24 | VPC Peering — non-transitive, cross-account, cross-region | 🟡 | Network |
| Q25 | AWS Transit Gateway — hub-spoke, route tables, peering | 🟡 | Network |
| Q26 | VPC Endpoints — Gateway (S3/DynamoDB), Interface (PrivateLink) | 🟡 | Network |
| Q27 | AWS VPN — Site-to-Site, Client VPN, Accelerated VPN | 🟡 | Network |
| Q28 | AWS Direct Connect — VIFs, DXGW, Global Reach | 🟡 | Network |
| Q29 | Route 53 routing policies — all 7 types | 🟡 | Network |
| Q30 | Amazon CloudFront — distributions, origins, CF Functions | 🟡 | Network |
| Q31 | AWS WAF — managed rules, rate limiting, geo-blocking | 🟡 | Network |
| Q32 | AWS PrivateLink — endpoint services, NLB | 🟡 | Network |
| Q33 | AWS Global Accelerator — anycast, endpoint groups | 🟡 | Network |
| Q34 | Amazon API Gateway — HTTP API, JWT auth, throttling | 🟢 | Network |
| Q35 | ALB vs NLB vs GWLB comparison | 🟢 | Network |
| Q36 | VPC Flow Logs — CloudWatch, S3, Athena queries | 🟡 | Network |
| Q37 | AWS Network Firewall — stateful rules, Suricata IPS | 🟡 | Network |
| Q38 | Elastic IP Address — allocation, association, cost | 🟢 | Network |
| Q39 | PrivateLink vs VPC Peering vs VPN comparison | 🟡 | Network |
| Q40 | VPC Lattice — service mesh for microservices | 🟡 | Network |
| **STORAGE (Q41–Q53)** | | | |
| Q41 | Amazon S3 — create, upload, versioning, presigned URLs | 🟢 | Storage |
| Q42 | S3 storage classes — all 7 tiers, lifecycle policies | 🟢 | Storage |
| Q43 | S3 Intelligent-Tiering — auto tiering, archive tiers | 🟡 | Storage |
| Q44 | S3 Replication — CRR, SRR, RTC, Batch Replication | 🟡 | Storage |
| Q45 | S3 Security — bucket policies, access points, Object Lock | 🟡 | Storage |
| Q46 | S3 Event Notifications — SNS, Lambda, SQS, EventBridge | 🟡 | Storage |
| Q47 | Amazon EBS — types, snapshots, online resize, FSR | 🟢 | Storage |
| Q48 | Amazon EFS — performance modes, lifecycle, access points | 🟡 | Storage |
| Q49 | Amazon FSx — Windows, Lustre, ONTAP, OpenZFS | 🟡 | Storage |
| Q50 | S3 Transfer Acceleration + AWS DataSync | 🟡 | Storage |
| Q51 | AWS Storage Gateway — S3 File, Volume, Tape | 🟡 | Storage |
| Q52 | AWS Backup — vaults, plans, cross-region copy | 🟡 | Storage |
| Q53 | S3 Analytics — S3 Select, Athena, Storage Lens | 🟡 | Storage |
| **DATABASES (Q54–Q65)** | | | |
| Q54 | Amazon RDS — create, read replicas, PITR, Proxy | 🟢 | Database |
| Q55 | Amazon Aurora — serverless v2, Global DB, zero-ETL | 🟡 | Database |
| Q56 | Amazon DynamoDB — CRUD, GSI, transactions, Streams | 🟡 | Database |
| Q57 | DAX + Global Tables — caching, active-active multi-region | 🟡 | Database |
| Q58 | Amazon ElastiCache — Redis serverless, cluster mode | 🟢 | Database |
| Q59 | Amazon Redshift — serverless, Spectrum, ML | 🟡 | Database |
| Q60 | Amazon DocumentDB — MongoDB-compatible | 🟡 | Database |
| Q61 | Amazon Keyspaces — Cassandra-compatible | 🟡 | Database |
| Q62 | Amazon Neptune — graph DB, Gremlin, openCypher | 🟡 | Database |
| Q63 | Amazon MemoryDB for Redis — durable in-memory DB | 🟢 | Database |
| Q64 | Amazon Timestream — serverless time-series | 🟡 | Database |
| Q65 | Amazon QLDB — immutable ledger, cryptographic verification | 🟡 | Database |

---
*Total: 65 Q&A | Compute (20) + Networking (20) + Storage (13) + Databases (12)*
*June 2026 | AWS Documentation Aligned*
*🟢 22 Basic | 🟡 43 Intermediate*

---

# GAP-FILL — COMPUTE (Q66–Q90)

---

### 🟡 Q66. What is AWS Systems Manager (SSM)?
```bash
# SSM: manage EC2 and on-prem servers at scale — no SSH/RDP needed
# Key capabilities: Session Manager, Patch Manager, Parameter Store, Run Command, Automation

# Session Manager — browser/CLI SSH without opening port 22
aws ssm start-session --target i-1234567890abcdef0

# SSH over Session Manager (port forwarding)
aws ssm start-session --target i-1234567890abcdef0 \
  --document-name AWS-StartSSHSession \
  --parameters portNumber=22
# ssh -o ProxyCommand='aws ssm start-session --target %h --document AWS-StartSSHSession --parameters portNumber=%p' ec2-user@i-1234567890abcdef0

# Run Command — run scripts on multiple instances
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --targets Key=tag:Env,Values=prod \
  --parameters commands=["df -h","free -m","systemctl status myapp"] \
  --output-s3-bucket-name my-ssm-logs \
  --output-s3-key-prefix run-command-output/ \
  --timeout-seconds 300

# Check command status
aws ssm list-command-invocations \
  --command-id <command-id> \
  --details \
  --query 'CommandInvocations[*].[InstanceId,Status,CommandPlugins[0].Output]' \
  --output table

# Parameter Store — secure config/secrets storage
# Standard: free, 4KB limit | Advanced: paid, 8KB, policies, cross-account
aws ssm put-parameter \
  --name /myapp/prod/db-password \
  --value "MySecretP@ss!" \
  --type SecureString \
  --key-id arn:aws:kms:us-east-1:123456789012:key/abc123 \
  --description "Production database password" \
  --tags Key=App,Value=myapp Key=Env,Value=prod

aws ssm put-parameter \
  --name /myapp/prod/config \
  --value '{"maxConnections":100,"timeout":30}' \
  --type String

# Get parameter
aws ssm get-parameter \
  --name /myapp/prod/db-password \
  --with-decryption \
  --query Parameter.Value --output text

# Get by path (all params under prefix)
aws ssm get-parameters-by-path \
  --path /myapp/prod/ \
  --recursive \
  --with-decryption \
  --query 'Parameters[].[Name,Value]' --output table

# In EC2 user data / Lambda (no credentials needed with IAM role)
# PARAM=$(aws ssm get-parameter --name /myapp/prod/db-password --with-decryption --query Parameter.Value --output text)

# Patch Manager — automated OS patching
aws ssm create-patch-baseline \
  --name MyLinuxBaseline \
  --operating-system AMAZON_LINUX_2023 \
  --approval-rules '{
    "PatchRules": [{
      "PatchFilterGroup": {
        "PatchFilters": [
          {"Key":"SEVERITY","Values":["Critical","Important"]},
          {"Key":"CLASSIFICATION","Values":["Security"]}
        ]
      },
      "ApproveAfterDays": 7,
      "ComplianceLevel": "HIGH"
    }]
  }' \
  --approved-patches "CVE-2026-12345"

aws ssm create-maintenance-window \
  --name MyPatchWindow \
  --schedule "cron(0 2 ? * SUN *)" \
  --duration 4 --cutoff 1 --allow-unassociated-targets false

# SSM Automation — complex workflows
aws ssm start-automation-execution \
  --document-name AWS-UpdateLinuxAmi \
  --parameters SourceAmiId=ami-12345678,InstanceIamRole=EC2RoleForSSM
```

---

### 🟡 Q67. What are EC2 Network Interfaces (ENI, ENA, EFA)?
```bash
# ENI (Elastic Network Interface): virtual NIC — can attach/detach from instances
# ENA (Elastic Network Adapter): high-performance NIC driver (up to 100 Gbps)
# EFA (Elastic Fabric Adapter): HPC — bypass OS networking for MPI (low latency)

# Create ENI
ENI_ID=$(aws ec2 create-network-interface \
  --subnet-id subnet-12345678 \
  --groups sg-12345678 \
  --description "Secondary NIC for myapp" \
  --private-ip-address 10.0.1.100 \
  --tag-specifications 'ResourceType=network-interface,Tags=[{Key=Name,Value=mySecondaryNIC}]' \
  --query NetworkInterface.NetworkInterfaceId --output text)

# Attach to instance (hot attach — no reboot needed)
aws ec2 attach-network-interface \
  --network-interface-id $ENI_ID \
  --instance-id i-1234567890abcdef0 \
  --device-index 1

# Detach and move to another instance (useful for failover)
aws ec2 detach-network-interface \
  --attachment-id eni-attach-12345678 --force

aws ec2 attach-network-interface \
  --network-interface-id $ENI_ID \
  --instance-id i-abcdef1234567890 \
  --device-index 1

# ENI use cases:
# - Management NIC on separate subnet (dual-homed instances)
# - Move static IP between instances on failover
# - Multiple IPs on one instance (hosting multiple sites)
# - Security appliances (traffic inspection)

# Check ENA support
aws ec2 describe-instances \
  --instance-ids i-1234567890abcdef0 \
  --query 'Reservations[].Instances[].EnaSupport'

# EFA — high-performance for ML/HPC (must use efa instance types)
aws ec2 create-network-interface \
  --subnet-id subnet-12345678 \
  --interface-type efa \    # specify EFA type
  --groups sg-12345678

# EFA instance types: p3dn.24xlarge, p4d.24xlarge, p5.48xlarge, hpc6a.48xlarge
# EFA benefits: bypass TCP/IP stack, OS-bypass for RDMA and MPI
# Required software: aws-ofi-nccl (for ML), libfabric, Open MPI
```

---

### 🟡 Q68. What is EC2 Image Builder?
```bash
# Image Builder: automated pipeline for creating, testing, and distributing AMIs
# Components: Recipe → Pipeline → Trigger → Test → Distribute

# Create component (software to install)
aws imagebuilder create-component \
  --name InstallNodeJS \
  --semantic-version 1.0.0 \
  --platform Linux \
  --data '{
    "schemaVersion": "1.0",
    "phases": [
      {
        "name": "build",
        "steps": [
          {
            "name": "InstallNVM",
            "action": "ExecuteBash",
            "inputs": {
              "commands": [
                "curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash",
                "source ~/.bashrc",
                "nvm install 20 && nvm use 20",
                "node --version"
              ]
            }
          }
        ]
      },
      {
        "name": "test",
        "steps": [
          {
            "name": "TestNodeJS",
            "action": "ExecuteBash",
            "inputs": {
              "commands": ["node --version | grep v20"]
            }
          }
        ]
      }
    ]
  }'

# Create image recipe
aws imagebuilder create-image-recipe \
  --name MyAppGoldenImage \
  --semantic-version 1.0.0 \
  --parent-image arn:aws:imagebuilder:us-east-1:aws:image/amazon-linux-2023-x86/x.x.x \
  --components '[
    {"componentArn": "arn:aws:imagebuilder:us-east-1:aws:component/aws-cli-version-2-linux/x.x.x"},
    {"componentArn": "arn:aws:imagebuilder:us-east-1:123456789012:component/InstallNodeJS/1.0.0"},
    {"componentArn": "arn:aws:imagebuilder:us-east-1:aws:component/simple-boot-test-linux/x.x.x"}
  ]' \
  --block-device-mappings '[{
    "deviceName": "/dev/xvda",
    "ebs": {"volumeSize": 30, "volumeType": "gp3", "encrypted": true}
  }]'

# Create pipeline
aws imagebuilder create-image-pipeline \
  --name MyGoldenImagePipeline \
  --image-recipe-arn arn:aws:imagebuilder:us-east-1:123456789012:image-recipe/MyAppGoldenImage/1.0.0 \
  --infrastructure-configuration-arn <infra-config-arn> \
  --distribution-configuration-arn <dist-config-arn> \
  --schedule '{"scheduleExpression":"cron(0 2 ? * MON *)","pipelineExecutionStartCondition":"EXPRESSION_MATCH_AND_DEPENDENCY_UPDATES_AVAILABLE"}' \
  --status ENABLED

# Start pipeline manually
aws imagebuilder start-image-pipeline-execution \
  --image-pipeline-arn arn:aws:imagebuilder:us-east-1:123456789012:image-pipeline/MyGoldenImagePipeline
```

---

### 🟡 Q69. What are Lambda Layers and container images?
```bash
# Lambda Layers: shared code/libraries packaged separately from function
# Max: 5 layers per function, 250MB total unzipped

# Create layer
mkdir -p python/lib/python3.12/site-packages
pip install pandas numpy requests -t python/lib/python3.12/site-packages/
zip -r pandas-layer.zip python/

aws lambda publish-layer-version \
  --layer-name pandas-numpy \
  --description "pandas 2.2 + numpy 1.26" \
  --zip-file fileb://pandas-layer.zip \
  --compatible-runtimes python3.12 \
  --compatible-architectures x86_64 arm64

LAYER_ARN=$(aws lambda list-layer-versions --layer-name pandas-numpy \
  --query 'LayerVersions[0].LayerVersionArn' --output text)

# Attach layer to function
aws lambda update-function-configuration \
  --function-name myFunction \
  --layers $LAYER_ARN \
    arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:7

# Lambda Powertools — structured logging, tracing, metrics (recommended)
# pip install aws-lambda-powertools
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit

logger  = Logger(service="OrderService")
tracer  = Tracer(service="OrderService")
metrics = Metrics(namespace="MyApp", service="OrderService")

@logger.inject_lambda_context(log_event=True)
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True)
def lambda_handler(event, context):
    logger.info("Processing order", extra={"orderId": event.get("orderId")})
    metrics.add_metric(name="OrdersProcessed", unit=MetricUnit.Count, value=1)
    return {"statusCode": 200}

# Lambda container image (up to 10GB — for ML models, large dependencies)
aws lambda create-function \
  --function-name myMLFunction \
  --package-type Image \
  --code ImageUri=123456789012.dkr.ecr.us-east-1.amazonaws.com/ml-lambda:latest \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --timeout 900 --memory-size 10240 \
  --architectures arm64

# Dockerfile for Lambda container image
# FROM public.ecr.aws/lambda/python:3.12
# COPY requirements.txt .
# RUN pip install -r requirements.txt
# COPY lambda_function.py .
# CMD ["lambda_function.lambda_handler"]

# Lambda destinations (async invoke routing)
aws lambda put-function-event-invoke-config \
  --function-name myFunction \
  --maximum-retry-attempts 2 \
  --maximum-event-age-in-seconds 3600 \
  --destination-config '{
    "OnSuccess": {"Destination": "arn:aws:sqs:us-east-1:123456789012:success-queue"},
    "OnFailure": {"Destination": "arn:aws:sqs:us-east-1:123456789012:failed-queue"}
  }'
# OnSuccess → SQS, SNS, EventBridge, or another Lambda
# OnFailure → SQS, SNS, EventBridge, or another Lambda
```

---

### 🟡 Q70. What is AWS App Runner?
```bash
# App Runner: fully managed service for containerised web apps — simplest option
# No clusters, no load balancers, no auto-scaling to configure
# Deploy from: ECR image or GitHub source code (App Runner builds it)

# Deploy from ECR image
aws apprunner create-service \
  --service-name myWebApp \
  --source-configuration '{
    "ImageRepository": {
      "ImageIdentifier": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest",
      "ImageRepositoryType": "ECR",
      "ImageConfiguration": {
        "Port": "8080",
        "RuntimeEnvironmentVariables": {
          "DB_HOST": "mydb.rds.amazonaws.com",
          "ENV": "production"
        }
      }
    },
    "AutoDeploymentsEnabled": true
  }' \
  --instance-configuration '{
    "Cpu": "1 vCPU",
    "Memory": "2 GB",
    "InstanceRoleArn": "arn:aws:iam::123456789012:role/AppRunnerRole"
  }' \
  --health-check-configuration '{
    "Protocol": "HTTP",
    "Path": "/health",
    "Interval": 10,
    "Timeout": 5,
    "HealthyThreshold": 1,
    "UnhealthyThreshold": 5
  }' \
  --auto-scaling-configuration-arn <autoscaling-config-arn>

# Deploy from GitHub source code
aws apprunner create-service \
  --service-name myGitHubApp \
  --source-configuration '{
    "CodeRepository": {
      "RepositoryUrl": "https://github.com/myOrg/myRepo",
      "SourceCodeVersion": {"Type": "BRANCH", "Value": "main"},
      "CodeConfiguration": {
        "ConfigurationSource": "REPOSITORY",
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
      "ConnectionArn": "arn:aws:apprunner:us-east-1:123456789012:connection/github-conn/abc123"
    }
  }' \
  --instance-configuration '{"Cpu":"0.25 vCPU","Memory":"0.5 GB"}'

# App Runner vs ECS Fargate vs Lambda:
# App Runner:    simplest, web-only, auto-scale (0→N), no infra
# ECS Fargate:   more control, any container, background jobs, custom networking
# Lambda:        event-driven, max 15min, no persistent connections
```

---

### 🟡 Q71. What are CloudFormation StackSets and nested stacks?
```bash
# StackSets: deploy one CloudFormation template to multiple accounts/regions simultaneously

# Create StackSet (deploy to entire AWS Organization)
aws cloudformation create-stack-set \
  --stack-set-name NetworkBaseline \
  --template-body file://vpc-baseline.yaml \
  --parameters ParameterKey=VpcCidr,ParameterValue=10.0.0.0/16 \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM

# Deploy to specific OUs
aws cloudformation create-stack-instances \
  --stack-set-name NetworkBaseline \
  --deployment-targets OrganizationalUnitIds=ou-root-abc123 \
  --regions us-east-1 eu-west-1 ap-southeast-1 \
  --operation-preferences '{
    "RegionConcurrencyType": "PARALLEL",
    "MaxConcurrentPercentage": 25,
    "FailureTolerancePercentage": 10
  }'

# Nested stacks — reuse common templates
# Parent template references child templates via AWS::CloudFormation::Stack
```

```yaml
# parent-stack.yaml
Resources:
  VPCStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/vpc.yaml
      Parameters:
        VpcCidr: 10.0.0.0/16
        Environment: !Ref Environment
      TimeoutInMinutes: 10

  SecurityStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: VPCStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/security.yaml
      Parameters:
        VpcId: !GetAtt VPCStack.Outputs.VpcId

  AppStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: SecurityStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/app.yaml
      Parameters:
        VpcId: !GetAtt VPCStack.Outputs.VpcId
        AppSG: !GetAtt SecurityStack.Outputs.AppSecurityGroupId
```

---

### 🟡 Q72. What is AWS SAM (Serverless Application Model)?
```yaml
# SAM: CloudFormation extension specifically for serverless
# Simpler syntax, local testing, faster deployments

# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.12
    Timeout: 30
    MemorySize: 256
    Environment:
      Variables:
        TABLE_NAME: !Ref OrdersTable
        LOG_LEVEL: INFO
    Tracing: Active
    Layers:
      - !Sub arn:aws:lambda:${AWS::Region}:017000801446:layer:AWSLambdaPowertoolsPythonV3-python312-x86_64:7

Resources:
  OrdersApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: CognitoAuthorizer
        Authorizers:
          CognitoAuthorizer:
            UserPoolArn: !GetAtt UserPool.Arn
      Cors:
        AllowOrigin: "'https://myapp.com'"
        AllowHeaders: "'Content-Type,Authorization'"
      TracingEnabled: true

  CreateOrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: orders.create
      CodeUri: src/orders/
      Events:
        CreateOrder:
          Type: Api
          Properties:
            RestApiId: !Ref OrdersApi
            Path: /orders
            Method: POST
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref OrdersTable
        - SQSSendMessagePolicy:
            QueueName: !GetAtt OrderQueue.QueueName

  ProcessOrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: processor.handler
      CodeUri: src/processor/
      Events:
        OrderQueue:
          Type: SQS
          Properties:
            Queue: !GetAtt OrderQueue.Arn
            BatchSize: 10
            FunctionResponseTypes: [ReportBatchItemFailures]

  OrdersTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey:
        Name: OrderId
        Type: String
      ProvisionedThroughput:
        ReadCapacityUnits: 5
        WriteCapacityUnits: 5

  OrderQueue:
    Type: AWS::SQS::Queue
    Properties:
      VisibilityTimeout: 180
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt OrderDLQ.Arn
        maxReceiveCount: 3

  OrderDLQ:
    Type: AWS::SQS::Queue
```

```bash
# SAM CLI commands
sam build                                    # build artifacts
sam local invoke CreateOrderFunction \       # local invoke
  --event events/create-order.json

sam local start-api --port 3000             # local API Gateway

sam deploy \
  --stack-name myServerlessApp \
  --s3-bucket my-sam-artifacts \
  --parameter-overrides Env=prod \
  --capabilities CAPABILITY_IAM \
  --confirm-changeset

sam logs -n CreateOrderFunction --tail --filter "ERROR"
sam traces --tracing --start-time "-1h"
```

---

### 🟡 Q73. What is ECS Service Discovery?
```bash
# Service Discovery: ECS services find each other via DNS (AWS Cloud Map)
# Each service gets a DNS record: myservice.myapp.local → IP addresses

# Create namespace (private DNS zone)
NS_ID=$(aws servicediscovery create-private-dns-namespace \
  --name myapp.local \
  --vpc $VPC_ID \
  --query OperationId --output text)

# Wait for namespace creation
aws servicediscovery get-operation --operation-id $NS_ID

NAMESPACE_ID=$(aws servicediscovery list-namespaces \
  --query 'Namespaces[?Name==`myapp.local`].Id' --output text)

# Create service (DNS record per task)
aws servicediscovery create-service \
  --name orders-api \
  --namespace-id $NAMESPACE_ID \
  --dns-config 'NamespaceId='"$NAMESPACE_ID"',DnsRecords=[{Type=A,TTL=10}]' \
  --health-check-custom-config FailureThreshold=1

DISCOVERY_SRN=$(aws servicediscovery list-services \
  --query 'Services[?Name==`orders-api`].Arn' --output text)

# Create ECS service with service discovery
aws ecs create-service \
  --cluster myCluster \
  --service-name orders-api \
  --task-definition orders-api:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration 'awsvpcConfiguration={subnets=["subnet-11111111"],securityGroups=["sg-12345678"],assignPublicIp=DISABLED}' \
  --service-registries '[{
    "registryArn": "'"$DISCOVERY_SRN"'",
    "containerName": "orders-api",
    "containerPort": 8080
  }]'

# Now: orders-api.myapp.local → resolves to all running task IPs
# Other ECS services can call http://orders-api.myapp.local:8080
```

---

### 🟡 Q74. What is EKS security with IRSA?
```bash
# IRSA (IAM Roles for Service Accounts): pods get AWS permissions via OIDC
# No static credentials, no kiam/kube2iam needed

# Enable OIDC provider on cluster
OIDC_URL=$(aws eks describe-cluster --name myEKSCluster \
  --query "cluster.identity.oidc.issuer" --output text | sed 's|https://||')

aws iam create-open-id-connect-provider \
  --url "https://${OIDC_URL}" \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list <thumbprint>

# Create IAM role for service account
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
NAMESPACE=myapp
SA_NAME=orders-service-account

cat > trust-policy.json << JSON
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_URL}"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "${OIDC_URL}:sub": "system:serviceaccount:${NAMESPACE}:${SA_NAME}",
        "${OIDC_URL}:aud": "sts.amazonaws.com"
      }
    }
  }]
}
JSON

aws iam create-role --role-name eks-orders-role \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name eks-orders-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
```

```yaml
# Create Kubernetes ServiceAccount with IAM role annotation
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-service-account
  namespace: myapp
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eks-orders-role
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
  namespace: myapp
spec:
  template:
    spec:
      serviceAccountName: orders-service-account   # pod gets OIDC token
      containers:
      - name: orders-api
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/orders-api:latest
        # boto3/AWS SDK auto-detects credentials from OIDC token via SDK
```

---

### 🟡 Q75. What is EC2 Nitro System?
```bash
# Nitro: AWS custom hypervisor replacing Xen
# Benefits: near bare-metal performance, better security, more innovation

# Nitro components:
# Nitro Hypervisor:  lightweight hypervisor (< 2% overhead)
# Nitro Cards:       dedicated cards for EBS, networking, security
# Nitro Security Chip: hardware security for cryptographic operations
# Nitro Enclaves:    isolated compute for sensitive data (credit card, medical)

# Nitro benefits:
# ✅ Higher network bandwidth (100 Gbps on compatible types)
# ✅ NVMe SSD support (local NVMe and EBS NVMe)
# ✅ Better security isolation (hardware-enforced)
# ✅ Dedicated hardware for VPC networking
# ✅ Enables Elastic Fabric Adapter (EFA)
# ✅ Required for: metal instances, C5, M5, R5, etc.

# Nitro Enclaves — isolated compute environment within EC2
aws ec2 run-instances \
  --instance-type m5.xlarge \
  --enclave-options Enabled=true \
  --image-id ami-12345678

# On the instance:
# nitro-cli build-enclave --docker-uri 123456789012.dkr.ecr.us-east-1.amazonaws.com/enclave-app:latest --output-file myenclave.eif
# nitro-cli run-enclave --eif-path myenclave.eif --memory 512 --cpu-count 2

# Nitro Enclave use cases:
# Process credit card data without exposing to instance OS
# Medical record processing
# Cryptographic key management
# Machine learning on sensitive data
```


---

# GAP-FILL — NETWORKING (Q76–Q90)

---

### 🟢 Q76. What is AWS Shield?
```bash
# Shield Standard: automatic DDoS protection for all AWS customers (FREE)
# Shield Advanced: enhanced protection with 24/7 DDoS Response Team ($3,000/month)

# Shield Standard protects:
# EC2, ELB, CloudFront, Route 53, Global Accelerator
# Protection: L3/L4 attacks (SYN floods, UDP reflection, volumetric)

# Shield Advanced adds:
# L7 protection with WAF (HTTP floods)
# Real-time attack visibility
# DDoS cost protection (credits for scaling costs during attack)
# AWS DDoS Response Team (DRT) 24/7 access
# Attack diagnostics and post-event analysis
# Proactive engagement (DRT contacts you during attack)

# Subscribe to Shield Advanced
aws shield create-subscription   # one-time per account

# Add protected resource
aws shield create-protection \
  --name "My ALB Protection" \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/myALB/abc123

# Enable proactive engagement (DRT contacts you)
aws shield update-proactive-engagement --proactive-engagement-status ENABLED

aws shield update-emergency-contact-settings \
  --emergency-contact-list '[
    {"EmailAddress":"security@company.com","PhoneNumber":"+15551234567","ContactNotes":"Primary security contact"},
    {"EmailAddress":"cto@company.com","PhoneNumber":"+15559876543","ContactNotes":"CTO escalation"}
  ]'

# Associate WAF WebACL with ALB (for L7 protection)
aws shield associate-drt-log-bucket --log-bucket my-shield-logs
aws shield associate-drt-role --role-arn arn:aws:iam::123456789012:role/ShieldDRTRole

# View attack events
aws shield list-attacks \
  --start-time StartTime="$(date -u -d '-7 days' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time EndTime="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --output table
```

---

### 🟡 Q77. What is Route 53 Resolver?
```bash
# Route 53 Resolver: bidirectional DNS resolution between VPC and on-premises
# Inbound endpoint:  on-prem resolves AWS private DNS
# Outbound endpoint: AWS instances resolve on-prem DNS

# Create inbound endpoint (on-prem → AWS)
aws route53resolver create-resolver-endpoint \
  --creator-request-id myInboundEP-$(date +%s) \
  --name MyInboundEndpoint \
  --security-group-ids sg-12345678 \
  --direction INBOUND \
  --ip-addresses \
    SubnetId=subnet-11111111,Ip=10.0.1.10 \
    SubnetId=subnet-22222222,Ip=10.0.2.10
# On-prem DNS servers forward *.myapp.local to these IPs

# Create outbound endpoint (AWS → on-prem)
aws route53resolver create-resolver-endpoint \
  --creator-request-id myOutboundEP-$(date +%s) \
  --name MyOutboundEndpoint \
  --security-group-ids sg-12345678 \
  --direction OUTBOUND \
  --ip-addresses \
    SubnetId=subnet-11111111 \
    SubnetId=subnet-22222222

# Create forwarding rule (forward corp.contoso.com queries to on-prem DNS)
OUTBOUND_EP=$(aws route53resolver list-resolver-endpoints \
  --filters Name=Direction,Values=OUTBOUND \
  --query 'ResolverEndpoints[0].Id' --output text)

aws route53resolver create-resolver-rule \
  --creator-request-id myRule-$(date +%s) \
  --name ForwardToCorp \
  --rule-type FORWARD \
  --domain-name corp.contoso.com \
  --target-ips Ip=192.168.1.53,Port=53 Ip=192.168.1.54,Port=53 \
  --resolver-endpoint-id $OUTBOUND_EP

# Associate rule with VPC
aws route53resolver associate-resolver-rule \
  --resolver-rule-id <rule-id> \
  --vpc-id $VPC_ID

# DNS Firewall (block malicious domains in VPC)
aws route53resolver create-firewall-domain-list \
  --creator-request-id myBlockList-$(date +%s) \
  --name MaliciousDomains

aws route53resolver update-firewall-domains \
  --firewall-domain-list-id <list-id> \
  --operation ADD \
  --domains malware.example.com phishing.example.com ransomware.example.net

aws route53resolver create-firewall-rule-group \
  --creator-request-id myRuleGroup-$(date +%s) \
  --name MyDNSFirewallRules

aws route53resolver create-firewall-rule \
  --creator-request-id myRule-$(date +%s) \
  --firewall-rule-group-id <rg-id> \
  --firewall-domain-list-id <list-id> \
  --priority 100 \
  --action BLOCK \
  --block-response NXDOMAIN

aws route53resolver associate-firewall-rule-group \
  --creator-request-id myAssoc-$(date +%s) \
  --firewall-rule-group-id <rg-id> \
  --vpc-id $VPC_ID \
  --priority 100 \
  --name MyDNSFirewall
```

---

### 🟡 Q78. What are CloudFront signed URLs vs signed cookies?
```bash
# Signed URLs:    restrict access to a SINGLE object (PDF, video file)
# Signed Cookies: restrict access to MULTIPLE objects (entire site area)

# When to use Signed URL:
# - Restrict single file (invoice.pdf, video.mp4)
# - RTMP streaming (CloudFront signed URLs)
# - Clients can't use cookies (mobile apps, wget, curl)

# When to use Signed Cookies:
# - Restrict entire premium area (/premium/*)
# - Web browser clients
# - Many files without changing URLs

# Create key pair (CloudFront uses RSA-SHA1)
openssl genrsa -out cloudfront-private.pem 2048
openssl rsa -pubout -in cloudfront-private.pem -out cloudfront-public.pem

# Upload public key to CloudFront
KEY_ID=$(aws cloudfront create-public-key \
  --public-key-config '{
    "CallerReference": "mykey-'$(date +%s)'",
    "Name": "MySigningKey",
    "EncodedKey": "'"$(cat cloudfront-public.pem)"'"
  }' \
  --query PublicKey.Id --output text)

# Create key group
aws cloudfront create-key-group \
  --key-group-config '{
    "Name": "MyKeyGroup",
    "Items": ["'"$KEY_ID"'"]
  }'

# Generate signed URL (Python)
import boto3, datetime, rsa
from botocore.signers import CloudFrontSigner

def rsa_signer(message):
    with open('cloudfront-private.pem', 'rb') as f:
        private_key = rsa.PrivateKey.load_pkcs1(f.read())
    return rsa.sign(message, private_key, 'SHA-1')

signer = CloudFrontSigner(KEY_ID, rsa_signer)
signed_url = signer.generate_presigned_url(
    url='https://d1234567890.cloudfront.net/premium/video.mp4',
    date_less_than=datetime.datetime.utcnow() + datetime.timedelta(hours=2),
    ip_address='192.0.2.0/24'   # optional IP restriction
)

# Generate signed cookies
cookies = signer.generate_presigned_cookies(
    url='https://d1234567890.cloudfront.net/premium/*',
    date_less_than=datetime.datetime.utcnow() + datetime.timedelta(days=1)
)
# Set cookies: CloudFront-Policy, CloudFront-Signature, CloudFront-Key-Pair-Id
```

---

### 🟡 Q79. What is CloudFront Origin Access Control (OAC)?
```bash
# OAC: secure S3 origin access — only CloudFront can access S3 (not public)
# OAC replaces OAI (Origin Access Identity) — supports KMS encrypted buckets, POST/PUT/DELETE

# Create OAC
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "MyOAC",
    "Description": "OAC for my S3 bucket",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }' \
  --query OriginAccessControl.Id --output text)

# Update S3 origin in distribution to use OAC
# (update-distribution with OriginAccessControlId set)

# Update S3 bucket policy to allow OAC
aws s3api put-bucket-policy \
  --bucket my-cloudfront-origin-bucket \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "AllowCloudFrontOAC",
      "Effect": "Allow",
      "Principal": {"Service": "cloudfront.amazonaws.com"},
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-cloudfront-origin-bucket/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EXXXXXXXXXX"
        }
      }
    }]
  }'

# Block all public access on bucket (CloudFront OAC handles access)
aws s3api put-public-access-block \
  --bucket my-cloudfront-origin-bucket \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,\
    BlockPublicPolicy=true,RestrictPublicBuckets=true
```

---

### 🟡 Q80. What are API Gateway types and throttling?
```bash
# API Gateway types:
# REST API:      full-featured, request/response transformation, caching, 10MB payload
# HTTP API:      simpler, 60% cheaper, JWT auth, Lambda/HTTP, 10MB payload
# WebSocket API: persistent bidirectional, route based on $request.body.action

# Throttling hierarchy (lower limit wins):
# Account: 10,000 requests/second burst, 5,000 RPS steady (default, per region)
# Stage:   set per stage (e.g., prod=5000, dev=100)
# Method:  override per HTTP method + resource
# Usage Plan + API Key: throttle specific clients

# Create usage plan with API key throttling
aws apigateway create-usage-plan \
  --name PremiumPlan \
  --throttle burstLimit=500,rateLimit=200 \
  --quota limit=1000000,period=MONTH \
  --api-stages apiId=abc123,stage=prod

# Create API key
KEY_ID=$(aws apigateway create-api-key \
  --name PremiumClientKey \
  --enabled \
  --query id --output text)

# Associate key with usage plan
aws apigateway create-usage-plan-key \
  --usage-plan-id <plan-id> \
  --key-id $KEY_ID \
  --key-type API_KEY

# Enable caching (REST API only)
aws apigateway update-stage \
  --rest-api-id abc123 \
  --stage-name prod \
  --patch-operations '[
    {"op":"replace","path":"/cacheClusterEnabled","value":"true"},
    {"op":"replace","path":"/cacheClusterSize","value":"0.5"},
    {"op":"replace","path":"/*/*/caching/enabled","value":"true"},
    {"op":"replace","path":"/*/*/caching/ttlInSeconds","value":"300"}
  ]'

# WebSocket API — real-time bidirectional
aws apigatewayv2 create-api \
  --name myWebSocketAPI \
  --protocol-type WEBSOCKET \
  --route-selection-expression '$request.body.action'

# Routes: $connect, $disconnect, $default, plus custom routes
aws apigatewayv2 create-route \
  --api-id <ws-api-id> \
  --route-key '$connect' \
  --authorization-type AWS_IAM \
  --target integrations/<integration-id>

aws apigatewayv2 create-route \
  --api-id <ws-api-id> \
  --route-key sendMessage \
  --target integrations/<integration-id>

# Lambda: send message to WebSocket client
import boto3
apigw = boto3.client('apigatewaymanagementapi',
    endpoint_url='https://abc123.execute-api.us-east-1.amazonaws.com/prod')
apigw.post_to_connection(
    ConnectionId=connection_id,
    Data=json.dumps({"type": "message", "text": "Hello!"}).encode()
)
```

---

### 🟡 Q81. What is AWS Firewall Manager?
```bash
# Firewall Manager: centrally manage WAF, Shield, Security Groups, Network Firewall across org
# Requires: AWS Organizations, AWS Config enabled, Firewall Manager admin account

# Designate Firewall Manager admin
aws fms associate-admin-account --admin-account 123456789012

# Create WAF policy (apply same WebACL to all ALBs in org)
aws fms put-policy \
  --policy '{
    "PolicyName": "OrgWAFPolicy",
    "SecurityServicePolicyData": {
      "Type": "WAFV2",
      "ManagedServiceData": "{\"type\":\"WAFV2\",\"defaultAction\":{\"type\":\"ALLOW\"},\"overrideCustomer DefaultAction\":false,\"preProcessRuleGroups\":[{\"managedRuleGroupIdentifier\":{\"vendorName\":\"AWS\",\"managedRuleGroupName\":\"AWSManagedRulesCommonRuleSet\"},\"overrideAction\":{\"type\":\"NONE\"},\"ruleGroupArn\":null,\"excludeRules\":[],\"ruleGroupType\":\"ManagedRuleGroup\"}],\"postProcessRuleGroups\":[],\"ruleGroups\":[]}"
    },
    "ResourceType": "AWS::ElasticLoadBalancingV2::LoadBalancer",
    "ResourceTags": [],
    "ExcludeResourceTags": false,
    "RemediationEnabled": true,
    "IncludeMap": {"ACCOUNT": []},
    "ExcludeMap": {"ACCOUNT": ["123456789012"]}
  }'

# Shield Advanced policy (protect all resources in org)
aws fms put-policy \
  --policy '{
    "PolicyName": "OrgShieldPolicy",
    "SecurityServicePolicyData": {"Type": "SHIELD_ADVANCED"},
    "ResourceType": "AWS::ElasticLoadBalancingV2::LoadBalancer",
    "RemediationEnabled": true,
    "IncludeMap": {}
  }'

# VPC Security Group policy (ensure all EC2 have specific SG)
aws fms put-policy \
  --policy '{
    "PolicyName": "RequiredSGPolicy",
    "SecurityServicePolicyData": {
      "Type": "SECURITY_GROUPS_USAGE_AUDIT"
    },
    "ResourceType": "AWS::EC2::Instance",
    "RemediationEnabled": true,
    "IncludeMap": {}
  }'
```

---

### 🟡 Q82. What is VPC Sharing (Resource Access Manager)?
```bash
# VPC Sharing: share subnets with other accounts in same AWS Organization
# One account owns the VPC/subnets; others deploy resources into shared subnets
# Benefits: shared networking cost, central network team manages VPC

# In VPC owner account: share subnet
aws ram create-resource-share \
  --name SharedSubnets \
  --resource-arns \
    arn:aws:ec2:us-east-1:123456789012:subnet/subnet-11111111 \
    arn:aws:ec2:us-east-1:123456789012:subnet/subnet-22222222 \
  --principals arn:aws:organizations::123456789012:ou/o-root/ou-abc123 \
  --allow-external-principals false

# In participant account: accept share + deploy resources
aws ram accept-resource-share-invitation \
  --resource-share-invitation-arn arn:aws:ram:us-east-1:123456789012:resource-share-invitation/abc123

# Now participant can deploy EC2 into shared subnet
aws ec2 run-instances \
  --subnet-id subnet-11111111 \   # subnet owned by central account
  --image-id ami-12345678 \
  --instance-type t3.micro

# VPC Sharing best practices:
# - Central networking account owns all VPCs and subnets
# - Business unit accounts deploy apps into shared subnets
# - NACLs and route tables controlled by owner (central team)
# - Security groups managed by participant account
# - One NAT Gateway, one VPN Gateway — shared cost
```

---

# GAP-FILL — STORAGE (Q83–Q95)

---

### 🟡 Q83. What are S3 multipart upload and S3 Batch Operations?
```bash
# Multipart upload: upload large files in parts (required > 5GB, recommended > 100MB)
# Benefits: retry failed parts, parallel uploads, resume interrupted uploads

# Manual multipart upload
UPLOAD_ID=$(aws s3api create-multipart-upload \
  --bucket my-bucket --key large-file.bin \
  --server-side-encryption aws:kms \
  --query UploadId --output text)

# Upload parts (in parallel with & )
aws s3api upload-part \
  --bucket my-bucket --key large-file.bin \
  --upload-id $UPLOAD_ID --part-number 1 \
  --body part1.bin &

aws s3api upload-part \
  --bucket my-bucket --key large-file.bin \
  --upload-id $UPLOAD_ID --part-number 2 \
  --body part2.bin &

wait   # wait for all parallel uploads

# Complete multipart upload
aws s3api complete-multipart-upload \
  --bucket my-bucket --key large-file.bin \
  --upload-id $UPLOAD_ID \
  --multipart-upload '{
    "Parts": [
      {"PartNumber": 1, "ETag": "etag1"},
      {"PartNumber": 2, "ETag": "etag2"}
    ]
  }'

# Abort incomplete multipart uploads (clean up storage)
aws s3api list-multipart-uploads --bucket my-bucket
aws s3api abort-multipart-upload \
  --bucket my-bucket --key large-file.bin --upload-id $UPLOAD_ID

# Lifecycle rule to auto-abort incomplete uploads
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "AbortIncompleteUploads",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }]
  }'

# ── S3 Batch Operations ────────────────────────────────────────────
# Process millions of S3 objects in batch — copy, tag, invoke Lambda, restore, ACL

# Create batch job to copy objects between buckets
aws s3control create-job \
  --account-id 123456789012 \
  --manifest '{
    "Spec": {"Format": "S3BatchOperations_CSV_20180820","Fields":["Bucket","Key"]},
    "Location": {
      "ObjectArn": "arn:aws:s3:::manifest-bucket/manifest.csv",
      "ETag": "abc123"
    }
  }' \
  --operation '{
    "S3PutObjectCopy": {
      "TargetResource": "arn:aws:s3:::destination-bucket",
      "StorageClass": "STANDARD_IA",
      "MetadataDirective": "COPY",
      "NewObjectTagging": [{"Key":"Archived","Value":"true"}],
      "CannedAccessControlList": "bucket-owner-full-control"
    }
  }' \
  --report '{
    "Bucket": "arn:aws:s3:::report-bucket",
    "Format": "Report_CSV_20180820",
    "Enabled": true,
    "ReportScope": "FailedTasksOnly"
  }' \
  --priority 10 \
  --role-arn arn:aws:iam::123456789012:role/S3BatchRole \
  --description "Migrate objects to destination bucket"

# Batch invoke Lambda (process each object)
aws s3control create-job \
  --account-id 123456789012 \
  --manifest '...' \
  --operation '{
    "LambdaInvoke": {
      "FunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:processObject"
    }
  }' \
  --priority 10 \
  --role-arn arn:aws:iam::123456789012:role/S3BatchRole

# Other batch operations:
# S3InitiateRestoreObject: restore from Glacier
# S3PutObjectTagging:      add/update tags on millions of objects
# S3DeleteObjectTagging:   remove tags
# S3PutObjectAcl:          update ACLs
# S3ReplicateObject:       replicate existing objects
```

---

### 🟡 Q84. What is EBS vs Instance Store?
| Feature | EBS | Instance Store |
|---------|-----|---------------|
| **Persistence** | Survives stop/reboot | Lost on stop/terminate |
| **Attachment** | Network-attached (any instance) | Physically attached (fixed) |
| **Performance** | gp3: 16K IOPS; io2: 256K IOPS | NVMe SSDs: millions IOPS |
| **Snapshot** | Yes (S3-backed) | No |
| **Encryption** | Yes (KMS) | No (encrypt in software) |
| **Resize** | Yes (online) | No |
| **Cost** | Per GB provisioned | Included in instance price |
| **Use case** | Databases, OS volumes | Temp data, buffer, cache |

```bash
# Instance store volumes — appear at /dev/nvme*n1 on Nitro instances
# View instance store volumes
aws ec2 describe-instances \
  --instance-ids i-1234567890abcdef0 \
  --query 'Reservations[].Instances[].BlockDeviceMappings'

# On instance:
# lsblk                    # see all volumes
# sudo mkfs.xfs /dev/nvme1n1
# sudo mkdir /tmp-fast && sudo mount /dev/nvme1n1 /tmp-fast

# Instance store use cases:
# ✅ Swap space
# ✅ Temporary files, scratch space
# ✅ Buffer/cache for distributed storage (ElastiCache, Hadoop HDFS)
# ✅ Replicated data (multiple copies across instances)
# ❌ Boot volume
# ❌ Primary database storage
# ❌ Anything that must survive instance stop
```

---

### 🟡 Q85. What is EBS Data Lifecycle Manager (DLM)?
```bash
# DLM: automate EBS snapshot and AMI creation, retention, and deletion

aws dlm create-lifecycle-policy \
  --description "Daily EBS snapshots with 30-day retention" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::123456789012:role/AWSDataLifecycleManagerDefaultRole \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["VOLUME"],
    "ResourceLocations": ["CLOUD"],
    "TargetTags": [{"Key": "Backup", "Value": "true"}],
    "Schedules": [
      {
        "Name": "DailySnapshots",
        "CreateRule": {
          "Interval": 24,
          "IntervalUnit": "HOURS",
          "Times": ["02:00"]
        },
        "RetainRule": {
          "Interval": 30,
          "IntervalUnit": "DAYS"
        },
        "CopyTags": true,
        "CrossRegionCopyRules": [{
          "TargetRegion": "eu-west-1",
          "Encrypted": true,
          "CmkArn": "arn:aws:kms:eu-west-1:123456789012:key/eu-key",
          "RetainRule": {"Interval": 7, "IntervalUnit": "DAYS"},
          "CopyTags": true
        }]
      },
      {
        "Name": "WeeklySnapshots",
        "CreateRule": {
          "CronExpression": "cron(0 2 ? * SUN *)"
        },
        "RetainRule": {
          "Interval": 12,
          "IntervalUnit": "WEEKS"
        },
        "CopyTags": true
      }
    ]
  }'

# EBS Recycle Bin (recover accidentally deleted snapshots)
aws rbin create-rule \
  --retention-period RetentionPeriodValue=30,RetentionPeriodUnit=DAYS \
  --resource-type EBS_SNAPSHOT \
  --tags '[{"Key":"Env","Value":"prod"}]'

# Recover snapshot from Recycle Bin
aws rbin list-resources-in-recycle-bin --resource-type EBS_SNAPSHOT
aws rbin restore-resource --resource-id snap-12345678 --resource-type EBS_SNAPSHOT
```

---

### 🟡 Q86. What is the AWS Snow family?
| Service | Capacity | Network | Use Case |
|---------|---------|---------|---------|
| **Snowcone** | 8TB HDD / 14TB SSD | WiFi / USB | Remote/edge, small transfers |
| **Snowcone with DataSync** | 8TB | Direct Connect | Continuous edge sync |
| **Snowball Edge Storage** | 80TB usable | 10 GbE | Large migration (TB scale) |
| **Snowball Edge Compute** | 42TB + 52 vCPU + GPU | 10/25 GbE | Edge compute + storage |
| **Snowmobile** | 100 PB | 10 Gbps | Exabyte migration, data center |

```bash
# Order Snowball Edge
aws snowball create-job \
  --job-type IMPORT \
  --resources '{
    "S3Resources": [{
      "BucketArn": "arn:aws:s3:::my-migration-bucket",
      "KeyRange": {}
    }]
  }' \
  --address-id <address-id> \
  --kms-key-arn arn:aws:kms:us-east-1:123456789012:key/abc123 \
  --role-arn arn:aws:iam::123456789012:role/SnowballRole \
  --snowball-type EDGE_STORAGE_OPTIMIZED \
  --shipping-option NEXT_DAY \
  --description "Datacenter migration batch 1"

# Transfer process:
# 1. AWS ships device to your location (1-2 weeks)
# 2. Connect device to your network (10 GbE)
# 3. Use AWS OpsHub or Snowball client to transfer data
# 4. Ship device back to AWS
# 5. AWS uploads data to S3 (within a week)

# Snowball Edge — edge computing (run Lambda, EC2 instances offline)
# Install EC2-compatible AMIs on the device
# Run workloads in manufacturing plants, ships, remote locations

# Decision guide:
# < 10 TB and fast internet: Use S3 Transfer Acceleration
# 10-80 TB per week:         Snowball Edge Storage
# >80 TB one-time:           Multiple Snowball Edge or Snowmobile
# Need to compute at edge:   Snowball Edge Compute
# < 8 TB remote collection:  Snowcone
```

---

### 🟡 Q87. What is AWS Transfer Family?
```bash
# Transfer Family: managed SFTP/FTPS/FTP/AS2 gateway in front of S3 or EFS
# Lift-and-shift legacy file transfer workflows without changing client-side

# Create SFTP server
aws transfer create-server \
  --protocols SFTP \
  --endpoint-type PUBLIC \
  --identity-provider-type SERVICE_MANAGED \
  --logging-role arn:aws:iam::123456789012:role/TransferLoggingRole \
  --structured-log-destinations arn:aws:logs:us-east-1:123456789012:log-group:/aws/transfer \
  --security-policy-name TransferSecurityPolicy-2024-01

SERVER_ID=$(aws transfer list-servers \
  --query 'Servers[0].ServerId' --output text)

# Create user
aws transfer create-user \
  --server-id $SERVER_ID \
  --user-name sftp-user \
  --role arn:aws:iam::123456789012:role/TransferUserRole \
  --home-directory-type LOGICAL \
  --home-directory-mappings '[
    {"Entry": "/", "Target": "/my-sftp-bucket/users/sftp-user"}
  ]' \
  --ssh-public-key-body "$(cat ~/.ssh/sftp_user_key.pub)"

# Custom identity provider (API Gateway + Lambda)
aws transfer create-server \
  --protocols SFTP FTPS \
  --endpoint-type PUBLIC \
  --identity-provider-type API_GATEWAY \
  --identity-provider-details '{
    "Url": "https://abc123.execute-api.us-east-1.amazonaws.com/prod/servers/",
    "InvocationRole": "arn:aws:iam::123456789012:role/TransferGatewayRole"
  }'

# AS2 (Electronic data interchange — EDI)
aws transfer create-connector \
  --url "https://partner.example.com/as2" \
  --as2-config '{
    "LocalProfileId": "local-profile-id",
    "PartnerProfileId": "partner-profile-id",
    "MessageSubject": "AS2 Transfer",
    "Compression": "ZLIB",
    "EncryptionAlgorithm": "AES256_CBC",
    "SigningAlgorithm": "SHA256",
    "MdnSigningAlgorithm": "SHA256",
    "MdnResponse": "SYNC"
  }' \
  --role arn:aws:iam::123456789012:role/TransferConnectorRole
```

# GAP-FILL — DATABASES (Q88–Q100)

---

### 🟡 Q88. What is RDS Multi-AZ vs Read Replicas?
| Feature | Multi-AZ | Read Replica |
|---------|---------|-------------|
| **Purpose** | High availability | Read scaling |
| **Replication** | Synchronous | Asynchronous |
| **Replica type** | Standby (no reads) | Active (read traffic) |
| **Failover** | Automatic (60-120s) | Manual promote |
| **Region** | Same region | Same or cross-region |
| **Endpoints** | One (auto-switches) | Separate endpoint each |
| **Use** | HA, failover | Scale reads, analytics |

```bash
# Multi-AZ: automatic failover when primary fails
# Triggers: loss of AZ, network failure, DB crash, maintenance
# Failover time: 60-120 seconds (DNS TTL change)
# During failover: DNS name switches to standby (same endpoint)

# Create Multi-AZ RDS
aws rds create-db-instance \
  --db-instance-identifier myProdDB \
  --multi-az \
  --engine postgres --engine-version 16.3 \
  --db-instance-class db.r6g.xlarge \
  --master-username admin --master-user-password "P@ss!" \
  --allocated-storage 100 --storage-type gp3

# Convert single-AZ to Multi-AZ (brief outage for RDS, zero for Aurora)
aws rds modify-db-instance \
  --db-instance-identifier myProdDB \
  --multi-az \
  --apply-immediately false   # applies during maintenance window

# Create read replica (for read scaling)
aws rds create-db-instance-read-replica \
  --db-instance-identifier myProdDB-replica \
  --source-db-instance-identifier myProdDB \
  --db-instance-class db.r6g.large \
  --availability-zone us-east-1b

# Promote read replica to standalone (DR scenario)
aws rds promote-read-replica \
  --db-instance-identifier myProdDB-replica \
  --backup-retention-period 7

# Monitor replication lag
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=myProdDB-replica \
  --start-time $(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Average --output table

# RDS Blue/Green deployment (zero-downtime major version upgrades)
aws rds create-blue-green-deployment \
  --blue-green-deployment-name myBlueGreenDeploy \
  --source arn:aws:rds:us-east-1:123456789012:db:myProdDB \
  --target-engine-version 16.3 \
  --target-db-instance-class db.r6g.2xlarge \
  --upgrade-target-storage-config

# After testing green environment:
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier bgd-12345678 \
  --switchover-timeout 300
```

---

### 🟡 Q89. What is DynamoDB capacity planning and partition key design?
```bash
# RCU (Read Capacity Unit) = 1 strongly consistent read of 4KB/s
#                           = 2 eventually consistent reads of 4KB/s
# WCU (Write Capacity Unit) = 1 write of 1KB/s

# Calculate needed capacity:
# 1000 reads/sec of 3KB items (eventually consistent) = 1000 * ceil(3/4) / 2 = 500 RCUs
# 500 writes/sec of 2KB items                         = 500 * ceil(2/1)     = 1000 WCUs

# Provision capacity
aws dynamodb update-table \
  --table-name Orders \
  --billing-mode PROVISIONED \
  --provisioned-throughput ReadCapacityUnits=500,WriteCapacityUnits=1000

# Partition key design — CRITICAL for performance:
# ✅ Good partition keys (high cardinality, even distribution):
# userId (millions of users), orderId (UUID), deviceId, timestamp+suffix
# ❌ Bad partition keys (hot partition):
# status (only PENDING/SHIPPED/DELIVERED — 3 values)
# boolean, date (all writes go to same partition)
# country (US gets most traffic)
# createdAt (all new items go to latest partition)

# Fix hot partition — add suffix to spread writes
import random
partition_key = f"ORDER#{random.randint(0, 9)}"    # 10 logical partitions

# Read all shards back:
# for i in range(10):
#     results += table.query(KeyConditionExpression=Key('pk').eq(f'ORDER#{i}'))

# Single Table Design — all entities in one table
# Use overloaded keys: PK=USER#123, SK=PROFILE
#                      PK=USER#123, SK=ORDER#ORD-001
#                      PK=USER#123, SK=ORDER#ORD-002
# Benefits: fewer tables, atomic transactions across entity types

# GSI overloading — reuse GSI for multiple access patterns
aws dynamodb create-table \
  --table-name SingleTable \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
    AttributeName=GSI1PK,AttributeType=S \
    AttributeName=GSI1SK,AttributeType=S \
  --key-schema AttributeName=PK,KeyType=HASH AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes '[{
    "IndexName": "GSI1",
    "KeySchema": [{"AttributeName":"GSI1PK","KeyType":"HASH"},{"AttributeName":"GSI1SK","KeyType":"RANGE"}],
    "Projection": {"ProjectionType":"ALL"}
  }]' \
  --billing-mode PAY_PER_REQUEST

# DynamoDB PartiQL (SQL-like queries)
aws dynamodb execute-statement \
  --statement "SELECT OrderId, Amount, Status FROM Orders WHERE CustomerId = 'C001'"

# DynamoDB Export to S3 (point-in-time export for analytics)
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --s3-bucket my-analytics-bucket \
  --s3-prefix dynamo-exports/ \
  --export-format DYNAMODB_JSON \   # DYNAMODB_JSON | ION
  --export-time "$(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%SZ)"
```

---

### 🟡 Q90. What is ElastiCache Redis vs Memcached?
| Feature | Redis | Memcached |
|---------|-------|-----------|
| **Data structures** | Strings, lists, sets, sorted sets, hashes, streams, HyperLogLog | Strings only |
| **Persistence** | RDB + AOF | No |
| **Replication** | Yes (primary + replicas) | No |
| **Failover** | Automatic | No |
| **Pub/Sub** | Yes | No |
| **Lua scripting** | Yes | No |
| **Cluster/sharding** | Yes (cluster mode) | Yes (multi-threaded) |
| **Atomic operations** | Yes | Yes |
| **Use case** | Full-featured cache, session store, leaderboards, pub/sub | Simple cache, multi-thread |

```bash
# Redis cluster mode OFF (single shard — replication only)
aws elasticache create-replication-group \
  --replication-group-id redis-no-cluster \
  --num-cache-clusters 3 \      # 1 primary + 2 replicas, same dataset
  --automatic-failover-enabled

# Redis cluster mode ON (multiple shards — horizontal scaling)
aws elasticache create-replication-group \
  --replication-group-id redis-cluster \
  --num-node-groups 6 \         # 6 shards
  --replicas-per-node-group 2   # 2 replicas per shard = 18 nodes total

# Memcached (simple, multithreaded, auto-discovery)
aws elasticache create-cache-cluster \
  --cache-cluster-id myMemcached \
  --engine memcached \
  --cache-node-type cache.m6g.large \
  --num-cache-nodes 5 \
  --az-mode cross-az \
  --auto-minor-version-upgrade true

# Memcached auto-discovery (client connects to cluster endpoint)
# Client: ElastiCache.Client with auto-discovery → distributes across all nodes
```

---

### 🟡 Q91. What is Redshift distribution styles and sort keys?
```sql
-- Distribution styles: control how data is spread across nodes

-- KEY distribution: rows with same key go to same node (good for JOIN)
CREATE TABLE orders (
    order_id BIGINT,
    customer_id INT,       -- JOIN key with customers table
    amount DECIMAL(18,2),
    order_date DATE
)
DISTKEY (customer_id);     -- same customer_id → same node

-- EVEN distribution: round-robin across nodes (good for tables without frequent JOINs)
CREATE TABLE audit_log (
    log_id BIGINT,
    message VARCHAR(4000)
)
DISTSTYLE EVEN;

-- ALL distribution: full copy on every node (good for small dimension tables)
CREATE TABLE products (
    product_id INT,
    name VARCHAR(200),
    category VARCHAR(100)
)
DISTSTYLE ALL;             -- 5MB table copied to all 10 nodes = 50MB total

-- AUTO distribution: Redshift chooses (small=ALL, large=EVEN, then optionally KEY)

-- Sort keys: pre-sort data for faster range queries and JOIN performance
-- Compound sort key: multiple columns, prefix must be used
CREATE TABLE daily_metrics (
    metric_date DATE,       -- first sort column
    region VARCHAR(50),     -- second sort column
    metric_name VARCHAR(100),
    value DECIMAL(18,2)
)
COMPOUND SORTKEY (metric_date, region);
-- Fast: WHERE metric_date >= '2026-01-01'
-- Fast: WHERE metric_date >= '2026-01-01' AND region = 'us-east'
-- Slow: WHERE region = 'us-east' (first key not used)

-- Interleaved sort key: equal weight to all columns
CREATE TABLE orders (
    order_id BIGINT,
    customer_id INT,
    order_date DATE,
    status VARCHAR(20)
)
INTERLEAVED SORTKEY (customer_id, order_date, status);
-- All column combinations are fast (but VACUUM takes longer)

-- VACUUM: reclaim space after deletes/updates, re-sort data
VACUUM orders;                    -- full vacuum + sort
VACUUM REINDEX orders;            -- rebuild interleaved sort key stats
VACUUM DELETE ONLY orders;        -- only reclaim deleted space
VACUUM SORT ONLY orders;          -- only re-sort

-- ANALYZE: update statistics for query planner
ANALYZE orders;
ANALYZE PREDICATE COLUMNS orders;  -- only analyze queried columns
```

---

### 🟡 Q92. What is AWS Database Migration Service (DMS)?
```bash
# DMS: migrate databases to AWS with minimal downtime
# Supports: Oracle, SQL Server, MySQL, PostgreSQL, MariaDB, MongoDB, SAP, DB2 → RDS/Aurora/DynamoDB

# Create replication instance
aws dms create-replication-instance \
  --replication-instance-identifier myDMSInstance \
  --replication-instance-class dms.r5.xlarge \
  --allocated-storage 100 \
  --multi-az \
  --vpc-security-group-ids sg-12345678 \
  --replication-subnet-group-identifier myDMSSubnetGroup \
  --auto-minor-version-upgrade true

# Create source endpoint (on-prem Oracle)
aws dms create-endpoint \
  --endpoint-identifier myOracleSource \
  --endpoint-type source \
  --engine-name oracle \
  --server-name 192.168.1.100 \
  --port 1521 \
  --database-name MYDB \
  --username dmsuser \
  --password "MyPassword!" \
  --oracle-settings '{
    "NumberDatatypeScale": -1,
    "AsmServer": "192.168.1.101",
    "AsmUser": "asmuser",
    "AsmPassword": "AsmP@ss!",
    "SecurityDbEncryptionName": "mySecKey"
  }'

# Create target endpoint (Aurora PostgreSQL)
aws dms create-endpoint \
  --endpoint-identifier myAuroraTarget \
  --endpoint-type target \
  --engine-name aurora-postgresql \
  --server-name myAuroraCluster.cluster-abc123.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --database-name mydb \
  --username masteruser \
  --password "MyPassword!"

# Create migration task
aws dms create-replication-task \
  --replication-task-identifier myOracleToAurora \
  --source-endpoint-arn <source-arn> \
  --target-endpoint-arn <target-arn> \
  --replication-instance-arn <replication-instance-arn> \
  --migration-type full-load-and-cdc \
  --table-mappings '{
    "rules": [{
      "rule-type": "selection",
      "rule-id": "1",
      "rule-action": "include",
      "object-locator": {"schema-name": "myschema", "table-name": "%"},
      "rule-name": "include-all"
    }]
  }' \
  --replication-task-settings '{
    "TargetMetadata": {
      "TargetSchema": "public",
      "SupportLobs": true,
      "FullLobMode": false,
      "LobChunkSize": 64
    },
    "FullLoadSettings": {"TargetTablePrepMode": "TRUNCATE_BEFORE_LOAD"},
    "Logging": {"EnableLogging": true}
  }'

# Start task
aws dms start-replication-task \
  --replication-task-arn <task-arn> \
  --start-replication-task-type start-replication

# Monitor progress
aws dms describe-replication-tasks \
  --query 'ReplicationTasks[].[ReplicationTaskIdentifier,Status,ReplicationTaskStats.FullLoadProgressPercent]' \
  --output table
```

---

### 🟡 Q93. What is Amazon OpenSearch Service?
```bash
# OpenSearch: managed Elasticsearch/OpenSearch — search, log analytics, APM
# Use cases: application search, log analysis (ELK stack), real-time monitoring

# Create domain
aws opensearch create-domain \
  --domain-name my-opensearch \
  --engine-version OpenSearch_2.13 \
  --cluster-config '{
    "InstanceType": "r6g.large.search",
    "InstanceCount": 3,
    "DedicatedMasterEnabled": true,
    "DedicatedMasterType": "r6g.large.search",
    "DedicatedMasterCount": 3,
    "ZoneAwarenessEnabled": true,
    "ZoneAwarenessConfig": {"AvailabilityZoneCount": 3},
    "MultiAZWithStandbyEnabled": true
  }' \
  --ebs-options 'EBSEnabled=true,VolumeType=gp3,VolumeSize=100,Iops=3000,Throughput=250' \
  --vpc-options 'SubnetIds=subnet-11111111,subnet-22222222,subnet-33333333,SecurityGroupIds=sg-12345678' \
  --node-to-node-encryption-options 'Enabled=true' \
  --encryption-at-rest-options 'Enabled=true,KmsKeyId=arn:aws:kms:us-east-1:123456789012:key/abc123' \
  --advanced-security-options 'Enabled=true,InternalUserDatabaseEnabled=true,MasterUserOptions={MasterUserName=admin,MasterUserPassword=P@ss!}' \
  --domain-endpoint-options 'EnforceHTTPS=true,TLSSecurityPolicy=Policy-Min-TLS-1-2-2019-07' \
  --log-publishing-options '{
    "INDEX_SLOW_LOGS": {"CloudWatchLogsLogGroupArn":"arn:aws:logs:us-east-1:123456789012:log-group:/aws/opensearch/slow-logs","Enabled":true},
    "SEARCH_SLOW_LOGS": {"CloudWatchLogsLogGroupArn":"arn:aws:logs:us-east-1:123456789012:log-group:/aws/opensearch/slow-logs","Enabled":true}
  }'

# Index documents
curl -XPOST "https://my-opensearch.us-east-1.es.amazonaws.com/orders/_doc" \
  --aws-sigv4 "aws:amz:us-east-1:es" \
  -H "Content-Type: application/json" \
  -d '{"orderId":"ORD-001","customer":"John Smith","amount":299.99,"status":"shipped","timestamp":"2026-06-13T10:00:00Z"}'

# Search
curl "https://my-opensearch.us-east-1.es.amazonaws.com/orders/_search" \
  --aws-sigv4 "aws:amz:us-east-1:es" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must": [{"match": {"status": "shipped"}}],
        "filter": [{"range": {"amount": {"gte": 100}}}]
      }
    },
    "aggs": {
      "avg_amount": {"avg": {"field": "amount"}},
      "status_counts": {"terms": {"field": "status.keyword"}}
    },
    "sort": [{"timestamp": {"order": "desc"}}],
    "size": 20
  }'

# OpenSearch Ingestion (managed data pipeline, replaces Logstash)
aws osis create-pipeline \
  --pipeline-name my-s3-to-opensearch \
  --min-units 1 --max-units 4 \
  --pipeline-configuration-body '
    version: "2"
    s3-source:
      s3:
        notification_type: sqs
        codec:
          json:
        sqs:
          queue_url: https://sqs.us-east-1.amazonaws.com/123456789012/my-queue
        aws:
          region: us-east-1
          sts_role_arn: arn:aws:iam::123456789012:role/OsisPipelineRole
    opensearch-sink:
      opensearch:
        hosts: ["https://my-opensearch.us-east-1.es.amazonaws.com"]
        index: orders
        aws:
          sts_role_arn: arn:aws:iam::123456789012:role/OsisPipelineRole
          region: us-east-1
          serverless: false
  '
```

---

### 🟢 Q94. Which AWS database should you choose?
| Use Case | AWS Service | Why |
|---------|------------|-----|
| **Relational, OLTP, existing SQL app** | RDS (MySQL/PostgreSQL) | Familiar SQL, managed |
| **Relational, high scale, HA** | Aurora | 5x MySQL perf, 3x PG, auto-scale storage |
| **Relational, horizontal write scale** | Aurora Limitless | Sharded Aurora, PB-scale |
| **NoSQL, serverless, any scale** | DynamoDB | Single-digit ms, auto-scale |
| **NoSQL, MongoDB apps** | DocumentDB | MongoDB compatible |
| **Graph relationships** | Neptune | Graph traversal, fraud detection |
| **In-memory cache** | ElastiCache (Redis) | Microsecond, data structures |
| **In-memory primary DB** | MemoryDB for Redis | Durable Redis |
| **Column analytics, DW** | Redshift | SQL at petabyte scale |
| **Search, log analytics** | OpenSearch | Full-text search, Kibana |
| **Cassandra apps** | Keyspaces | CQL compatible, serverless |
| **Time-series, IoT** | Timestream | Purpose-built time functions |
| **Ledger, audit trail** | QLDB | Immutable, cryptographic |
| **Mixed media + SQL** | RDS PostgreSQL (jsonb) | JSON + relational |
| **Simple key-value** | DynamoDB | Or ElastiCache if cache-only |

---

### 🟡 Q95. What is Aurora Backtrack and RDS IAM authentication?
```bash
# Aurora Backtrack: rewind database to specific time WITHOUT restore
# Near-instant (seconds) vs PITR restore (30+ minutes)
# Requires: enabled at cluster creation (MySQL only, not PostgreSQL)

aws rds create-db-cluster \
  --db-cluster-identifier myAuroraCluster \
  --engine aurora-mysql \
  --backtrack-window 86400 \   # enable backtrack up to 24 hours
  --master-username admin \
  --master-user-password "P@ss!"

# Backtrack to specific time
aws rds backtrack-db-cluster \
  --db-cluster-identifier myAuroraCluster \
  --backtrack-to "2026-06-13T09:00:00Z" \
  --force-backtrack-db-cluster \
  --use-earliest-time-on-point-in-time-unavailable

# Monitor backtrack status
aws rds describe-db-cluster-backtracks \
  --db-cluster-identifier myAuroraCluster \
  --query 'DBClusterBacktracks[].[BacktrackIdentifier,BacktrackTo,Status]' \
  --output table

# RDS IAM database authentication (no passwords — use IAM tokens)
# Enable on RDS instance
aws rds modify-db-instance \
  --db-instance-identifier myProdDB \
  --enable-iam-database-authentication

# Create IAM policy for RDS authentication
aws iam create-policy \
  --policy-name RDSIAMAuthPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "rds-db:connect",
      "Resource": "arn:aws:rds-db:us-east-1:123456789012:dbuser:myProdDB/myapp_user"
    }]
  }'

# Create DB user mapped to IAM
# psql> CREATE USER myapp_user;
# psql> GRANT rds_iam TO myapp_user;

# Connect using IAM token (Python)
import boto3, psycopg2

client = boto3.client('rds', region_name='us-east-1')
token = client.generate_db_auth_token(
    DBHostname='myProdDB.abc123.us-east-1.rds.amazonaws.com',
    Port=5432,
    DBUsername='myapp_user'
)

conn = psycopg2.connect(
    host='myProdDB.abc123.us-east-1.rds.amazonaws.com',
    port=5432,
    database='myapp',
    user='myapp_user',
    password=token,
    sslmode='require'
)
# Token valid for 15 minutes — refresh before expiry
```

---

## COMPLETE FINAL INDEX (All 150+ Q&A)

| Q# | Question | Level | Part |
|----|---------|-------|------|
| **COMPUTE (Q1–Q75)** | | | |
| Q1–Q20 | Core compute topics (EC2, Lambda, ECS, EKS, etc.) | 🟢🟡 | Compute |
| Q66 | AWS Systems Manager — Session Manager, Parameter Store, Patch | 🟡 | Compute |
| Q67 | EC2 Network Interfaces — ENI, ENA, EFA | 🟡 | Compute |
| Q68 | EC2 Image Builder — components, recipes, pipelines | 🟡 | Compute |
| Q69 | Lambda Layers, container images, destinations, Powertools | 🟡 | Compute |
| Q70 | AWS App Runner — container and GitHub source deployment | 🟡 | Compute |
| Q71 | CloudFormation StackSets + nested stacks | 🟡 | Compute |
| Q72 | AWS SAM — template, local testing, deployment | 🟡 | Compute |
| Q73 | ECS Service Discovery (Cloud Map) | 🟡 | Compute |
| Q74 | EKS IRSA — IAM roles for Kubernetes service accounts | 🔴 | Compute |
| Q75 | EC2 Nitro System — hypervisor, Nitro Enclaves | 🟡 | Compute |
| **NETWORKING (Q21–Q82)** | | | |
| Q21–Q40 | Core networking topics | 🟢🟡 | Network |
| Q76 | AWS Shield — Standard vs Advanced, DRT, attack logs | 🟢 | Network |
| Q77 | Route 53 Resolver — inbound/outbound, DNS Firewall | 🟡 | Network |
| Q78 | CloudFront signed URLs vs signed cookies | 🟡 | Network |
| Q79 | CloudFront Origin Access Control (OAC) | 🟡 | Network |
| Q80 | API Gateway types — REST vs HTTP vs WebSocket, throttling | 🟡 | Network |
| Q81 | AWS Firewall Manager — org-wide WAF, Shield, SG policies | 🟡 | Network |
| Q82 | VPC Sharing via Resource Access Manager | 🟡 | Network |
| **STORAGE (Q41–Q87)** | | | |
| Q41–Q53 | Core storage topics | 🟢🟡 | Storage |
| Q83 | S3 multipart upload + S3 Batch Operations | 🟡 | Storage |
| Q84 | EBS vs Instance Store — comparison, use cases | 🟢 | Storage |
| Q85 | EBS Data Lifecycle Manager + Recycle Bin | 🟡 | Storage |
| Q86 | AWS Snow family — Snowcone/Snowball/Snowmobile decision guide | 🟢 | Storage |
| Q87 | AWS Transfer Family — SFTP/FTPS/AS2 to S3/EFS | 🟡 | Storage |
| **DATABASES (Q54–Q95)** | | | |
| Q54–Q65 | Core database topics | 🟢🟡 | Database |
| Q88 | RDS Multi-AZ vs Read Replicas + Blue/Green deployments | 🟢 | Database |
| Q89 | DynamoDB capacity planning, partition key design, Single Table | 🔴 | Database |
| Q90 | ElastiCache Redis vs Memcached, cluster vs non-cluster mode | 🟢 | Database |
| Q91 | Redshift distribution styles, sort keys, VACUUM/ANALYZE | 🔴 | Database |
| Q92 | AWS DMS — migrate Oracle → Aurora, full-load-and-CDC | 🟡 | Database |
| Q93 | Amazon OpenSearch — domain, search, ingestion pipeline | 🟡 | Database |
| Q94 | Database selection guide — all 15 scenarios | 🟢 | Database |
| Q95 | Aurora Backtrack + RDS IAM authentication | 🟡 | Database |

---
*Total: 95+ core questions + 30 gap-fill questions = 125+ Q&A*
*Compute (35) + Networking (27) + Storage (20) + Databases (23)*
*June 2026 | AWS Documentation Aligned*
*🟢 30 Basic | 🟡 80 Intermediate | 🔴 5 Advanced*
