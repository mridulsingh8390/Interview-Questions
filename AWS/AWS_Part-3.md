# AWS Security, Identity & Compliance — Migration & Transfer — Analytics
# Complete Interview Q&A Guide
> **All possible questions | June 2026 | AWS Documentation aligned**
> 🟢 Basic | 🟡 Intermediate | 🔴 Advanced | Full CLI + Python + YAML examples

---

## TABLE OF CONTENTS
| Part | Category | Questions |
|------|---------|----------|
| [1](#part-1--aws-security-identity--compliance) | Security, Identity & Compliance | Q1–Q45 |
| [2](#part-2--aws-migration--transfer) | Migration & Transfer | Q46–Q80 |
| [3](#part-3--aws-analytics) | Analytics | Q81–Q120 |

---

# PART 1 — AWS SECURITY, IDENTITY & COMPLIANCE

---

### 🟢 Q1. What is the AWS Shared Responsibility Model?
**Answer:** AWS and the customer share security responsibilities. Understanding this is the foundation of every AWS security discussion.

| Layer | AWS Responsible For | Customer Responsible For |
|-------|-------------------|------------------------|
| **Physical** | Data centres, hardware, network infrastructure | Nothing |
| **Hypervisor** | Virtualisation layer | Nothing |
| **OS (managed)** | Patching RDS, Lambda, ECS Fargate | Nothing |
| **OS (unmanaged)** | Hypervisor only | EC2 OS patching, firewall |
| **Data** | Durability of storage | Encryption, classification, access control |
| **IAM** | Providing IAM service | Creating users, roles, least-privilege |
| **Network** | AWS backbone, DDoS protection | VPC design, SGs, NACLs, routing |
| **Application** | Managed services | Application code security |

```
# Key phrase: "Security OF the cloud vs security IN the cloud"
# AWS: Security OF the cloud (physical, hypervisor, managed services)
# You:  Security IN the cloud (your data, your code, your IAM, your VPC)
```

---

### 🟢 Q2. What is AWS IAM and what are the core components?
```bash
# Core IAM components:
# Users:    long-term credentials (avoid for apps — use roles)
# Groups:   collection of users, attach policies to groups
# Roles:    temporary credentials via STS, cross-account, EC2, Lambda
# Policies: JSON documents defining Allow/Deny actions on resources

# Create a production-ready IAM structure

# 1. Groups with managed policies
aws iam create-group --group-name developers
aws iam attach-group-policy --group-name developers \
  --policy-arn arn:aws:iam::aws:policy/PowerUserAccess

aws iam create-group --group-name readonly-ops
aws iam attach-group-policy --group-name readonly-ops \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# 2. User with group membership (no inline policies)
aws iam create-user --user-name alice \
  --tags Key=Team,Value=backend Key=Department,Value=Engineering
aws iam add-user-to-group --user-name alice --group-name developers

# 3. IAM role for EC2
aws iam create-role --role-name ec2-app-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# 4. Least-privilege policy
aws iam create-policy \
  --policy-name app-s3-dynamodb-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "S3BucketAccess",
        "Effect": "Allow",
        "Action": ["s3:GetObject","s3:PutObject","s3:DeleteObject"],
        "Resource": "arn:aws:s3:::my-app-bucket/*",
        "Condition": {
          "StringEquals": {"s3:prefix": ["uploads/"]}
        }
      },
      {
        "Sid": "DynamoDBTableAccess",
        "Effect": "Allow",
        "Action": ["dynamodb:GetItem","dynamodb:PutItem","dynamodb:UpdateItem","dynamodb:Query","dynamodb:Scan"],
        "Resource": [
          "arn:aws:dynamodb:us-east-1:123456789:table/orders",
          "arn:aws:dynamodb:us-east-1:123456789:table/orders/index/*"
        ]
      },
      {
        "Sid": "DenyDangerousOps",
        "Effect": "Deny",
        "Action": ["dynamodb:DeleteTable","s3:DeleteBucket"],
        "Resource": "*"
      }
    ]
  }'

# 5. Attach and create instance profile
aws iam attach-role-policy --role-name ec2-app-role \
  --policy-arn arn:aws:iam::123456789:policy/app-s3-dynamodb-policy
aws iam create-instance-profile --instance-profile-name ec2-app-profile
aws iam add-role-to-instance-profile \
  --instance-profile-name ec2-app-profile \
  --role-name ec2-app-role

# IAM policy evaluation logic:
# 1. Explicit Deny → always wins
# 2. SCP Deny → wins
# 3. Permission Boundary Deny → wins
# 4. All three (SCP + PB + Identity) must Allow → access granted
# 5. Otherwise → implicit Deny
```

---

### 🟡 Q3. What are IAM condition keys and when do you use them?
```bash
# Condition keys add context-based controls to policies

# Time-based access (allow only during business hours)
'{
  "Condition": {
    "DateGreaterThan": {"aws:CurrentTime": "2026-01-01T08:00:00Z"},
    "DateLessThan":    {"aws:CurrentTime": "2026-12-31T18:00:00Z"},
    "StringEquals":    {"aws:RequestedRegion": ["us-east-1","eu-west-1"]}
  }
}'

# Require MFA for sensitive actions
'{
  "Effect": "Deny",
  "Action": ["ec2:TerminateInstances","rds:DeleteDBInstance","s3:DeleteBucket"],
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}
  }
}'

# Require SSL/TLS (deny HTTP)
'{
  "Effect": "Deny",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket","arn:aws:s3:::my-bucket/*"],
  "Condition": {"Bool": {"aws:SecureTransport": "false"}}
}'

# Restrict by source VPC (only access from within VPC)
'{
  "Effect": "Deny",
  "Action": "s3:*",
  "Resource": "*",
  "Condition": {"StringNotEquals": {"aws:SourceVpc": "vpc-12345678"}}
}'

# Tag-based access control (TBAC)
aws iam create-policy --policy-name TagBasedEC2Access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["ec2:StartInstances","ec2:StopInstances"],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Owner": "${aws:username}",
          "ec2:ResourceTag/Environment": "development"
        }
      }
    }]
  }'

# Key global condition keys:
# aws:PrincipalArn       — who is making the request
# aws:SourceIp           — caller IP address
# aws:RequestedRegion    — target region
# aws:CurrentTime        — time of request
# aws:SecureTransport    — HTTPS only
# aws:MultiFactorAuthPresent — MFA active
# aws:PrincipalTag       — tags on caller's role/user
# aws:ResourceTag        — tags on target resource
# aws:SourceVpc/Vpce     — from specific VPC/endpoint
```

---

### 🟡 Q4. What is AWS STS and cross-account access?
```bash
# STS: Security Token Service — issues temporary credentials (max 12 hours)

# Assume role (cross-account)
aws sts assume-role \
  --role-arn arn:aws:iam::987654321:role/cross-account-role \
  --role-session-name "myapp-session" \
  --duration-seconds 3600 \
  --external-id "myUniqueExternalId"  \
  --serial-number arn:aws:iam::123456789:mfa/alice \
  --token-code 123456

# Use temporary credentials
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

# Get caller identity (verify who you are)
aws sts get-caller-identity

# Python: assume role and use it
import boto3

def get_cross_account_client(account_id: str, role_name: str, service: str):
    sts = boto3.client("sts")
    creds = sts.assume_role(
        RoleArn=f"arn:aws:iam::{account_id}:role/{role_name}",
        RoleSessionName="automation-session",
        DurationSeconds=3600
    )["Credentials"]

    return boto3.client(
        service,
        aws_access_key_id=creds["AccessKeyId"],
        aws_secret_access_key=creds["SecretAccessKey"],
        aws_session_token=creds["SessionToken"]
    )

s3 = get_cross_account_client("987654321", "s3-read-role", "s3")
objects = s3.list_objects_v2(Bucket="cross-account-bucket")

# Trust policy for cross-account role (in 987654321)
'{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::123456789:role/automation-role"},
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {"sts:ExternalId": "myUniqueExternalId"},
      "Bool": {"aws:MultiFactorAuthPresent": "true"}
    }
  }]
}'
```

---

### 🟢 Q5. What is Amazon Cognito?
```bash
# Cognito: user identity and authentication for web and mobile apps
# Two components:
# User Pools: authentication (sign-up, sign-in, MFA, password reset)
# Identity Pools: authorisation (exchange any token for temporary AWS credentials)

# Create User Pool
aws cognito-idp create-user-pool \
  --pool-name my-app-users \
  --policies '{
    "PasswordPolicy": {
      "MinimumLength": 12,
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireNumbers": true,
      "RequireSymbols": true,
      "TemporaryPasswordValidityDays": 7
    }
  }' \
  --auto-verified-attributes email \
  --username-attributes email \
  --mfa-configuration OPTIONAL \
  --sms-authentication-message "Your verification code is {####}" \
  --email-verification-message "Verify your email: {####}" \
  --account-recovery-setting '{
    "RecoveryMechanisms": [{"Priority": 1, "Name": "verified_email"}]
  }' \
  --admin-create-user-config '{
    "AllowAdminCreateUserOnly": false,
    "InviteMessageTemplate": {
      "EmailMessage": "Welcome! Your temporary password is {####}",
      "EmailSubject": "Welcome to MyApp"
    }
  }' \
  --lambda-config '{
    "PreSignUp": "arn:aws:lambda:us-east-1:123456789:function:pre-signup-validator",
    "PostConfirmation": "arn:aws:lambda:us-east-1:123456789:function:post-confirmation",
    "PreTokenGeneration": "arn:aws:lambda:us-east-1:123456789:function:add-custom-claims"
  }' \
  --schema \
    Name=email,Required=true,Mutable=true \
    Name=custom:department,AttributeDataType=String,Mutable=true

POOL_ID=$(aws cognito-idp create-user-pool --pool-name test --query "UserPool.Id" --output text)

# Create App Client
aws cognito-idp create-user-pool-client \
  --user-pool-id $POOL_ID \
  --client-name my-web-app \
  --no-generate-secret \
  --explicit-auth-flows \
    ALLOW_USER_SRP_AUTH \
    ALLOW_REFRESH_TOKEN_AUTH \
    ALLOW_USER_PASSWORD_AUTH \
  --supported-identity-providers COGNITO Google Facebook \
  --callback-urls "https://myapp.com/callback" "http://localhost:3000/callback" \
  --logout-urls "https://myapp.com/logout" \
  --allowed-o-auth-flows code \
  --allowed-o-auth-scopes openid email profile \
  --allowed-o-auth-flows-user-pool-client \
  --access-token-validity 60 \
  --refresh-token-validity 30 \
  --token-validity-units AccessToken=minutes,RefreshToken=days

# Configure hosted UI (pre-built login page)
aws cognito-idp create-user-pool-domain \
  --domain myapp-auth \
  --user-pool-id $POOL_ID
# Login URL: https://myapp-auth.auth.us-east-1.amazoncognito.com/login

# Add Google as federated IdP
aws cognito-idp create-identity-provider \
  --user-pool-id $POOL_ID \
  --provider-name Google \
  --provider-type Google \
  --provider-details \
    client_id=google-client-id \
    client_secret=google-client-secret \
    authorize_scopes="openid email profile" \
  --attribute-mapping email=email name=name picture=picture

# Create Identity Pool (AWS credential vending)
aws cognito-identity create-identity-pool \
  --identity-pool-name my-identity-pool \
  --allow-unauthenticated-identities \
  --cognito-identity-providers \
    ProviderName=cognito-idp.us-east-1.amazonaws.com/$POOL_ID,ClientId=<app-client-id>

# User management
aws cognito-idp admin-create-user \
  --user-pool-id $POOL_ID \
  --username bob@example.com \
  --user-attributes Name=email,Value=bob@example.com Name=custom:department,Value=Engineering \
  --temporary-password "TempP@ss123!" \
  --desired-delivery-mediums EMAIL

aws cognito-idp admin-reset-user-password \
  --user-pool-id $POOL_ID \
  --username bob@example.com

aws cognito-idp admin-disable-user \
  --user-pool-id $POOL_ID \
  --username bob@example.com
```

---

### 🟡 Q6. What are Cognito User Pool triggers (Lambda)?
```python
# Lambda triggers let you customise Cognito flows

# Pre Sign-up: validate before creating user
def pre_signup_handler(event, context):
    email = event["request"]["userAttributes"]["email"]
    # Allow only corporate email
    if not email.endswith("@mycompany.com"):
        raise Exception("Only @mycompany.com emails allowed")
    # Auto-confirm users (skip email verification)
    event["response"]["autoConfirmUser"] = True
    event["response"]["autoVerifyEmail"] = True
    return event

# Post Confirmation: onboard user after verification
def post_confirmation_handler(event, context):
    import boto3
    user_sub = event["request"]["userAttributes"]["sub"]
    email = event["request"]["userAttributes"]["email"]
    dynamodb = boto3.resource("dynamodb")
    table = dynamodb.Table("users")
    table.put_item(Item={
        "userId": user_sub,
        "email": email,
        "createdAt": datetime.utcnow().isoformat(),
        "plan": "free",
        "status": "active"
    })
    return event

# Pre Token Generation: add custom claims to JWT
def pre_token_generation_handler(event, context):
    import boto3
    user_sub = event["request"]["userAttributes"]["sub"]
    dynamodb = boto3.resource("dynamodb")
    user = dynamodb.Table("users").get_item(Key={"userId": user_sub})["Item"]

    event["response"]["claimsOverrideDetails"] = {
        "claimsToAddOrOverride": {
            "plan": user["plan"],
            "orgId": user.get("orgId", ""),
            "role": user.get("role", "user"),
            "features": ",".join(user.get("features", []))
        }
    }
    return event

# All trigger types:
# Pre Sign-up:           validate/auto-confirm before account creation
# Post Confirmation:     run after email confirmed (onboarding)
# Pre Authentication:    block sign-in based on custom logic
# Post Authentication:   audit log, session tracking
# Pre Token Generation:  add custom claims to JWT tokens
# Migrate User:          import users from legacy system on first login
# Custom Message:        customise email/SMS text (verification, invite)
# Define/Create/Verify:  custom auth challenges (OTP, CAPTCHA)
```

---

### 🟡 Q7. What is AWS WAF?
```bash
# WAF (Web Application Firewall): protect APIs and websites from Layer 7 attacks
# Can attach to: ALB, API Gateway, CloudFront, AppSync, App Runner

# Create WAF WebACL
aws wafv2 create-web-acl \
  --name my-web-acl \
  --scope REGIONAL \
  --default-action Allow={} \
  --rules '[
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 10,
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet",
          "ExcludedRules": [{"Name": "SizeRestrictions_BODY"}]
        }
      },
      "OverrideAction": {"None": {}},
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRuleSet"
      }
    },
    {
      "Name": "AWSManagedRulesKnownBadInputsRuleSet",
      "Priority": 20,
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesKnownBadInputsRuleSet"
        }
      },
      "OverrideAction": {"None": {}},
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "BadInputs"}
    },
    {
      "Name": "RateLimitRule",
      "Priority": 30,
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "Action": {"Block": {}},
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "RateLimit"}
    },
    {
      "Name": "BlockBadBots",
      "Priority": 40,
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesBotControlRuleSet",
          "ManagedRuleGroupConfigs": [{
            "AWSManagedRulesBotControlRuleSet": {"InspectionLevel": "TARGETED"}
          }]
        }
      },
      "OverrideAction": {"None": {}},
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "BotControl"}
    },
    {
      "Name": "GeoBlockRule",
      "Priority": 50,
      "Statement": {
        "GeoMatchStatement": {
          "CountryCodes": ["RU","CN","KP","IR"]
        }
      },
      "Action": {"Block": {}},
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "GeoBlock"}
    },
    {
      "Name": "SQLiProtection",
      "Priority": 60,
      "Statement": {
        "SqliMatchStatement": {
          "FieldToMatch": {"Body": {"OversizeHandling": "CONTINUE"}},
          "TextTransformations": [{"Priority": 1, "Type": "URL_DECODE"},{"Priority": 2, "Type": "HTML_ENTITY_DECODE"}],
          "SensitivityLevel": "HIGH"
        }
      },
      "Action": {"Block": {}},
      "VisibilityConfig": {"SampledRequestsEnabled": true, "CloudWatchMetricsEnabled": true, "MetricName": "SQLi"}
    }
  ]' \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=my-web-acl \
  --region us-east-1

# Associate with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:us-east-1:123456789:regional/webacl/my-web-acl/abc123 \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/my-alb/abc123

# Associate with CloudFront (must use us-east-1 and CLOUDFRONT scope)
aws wafv2 create-web-acl \
  --name my-cf-acl \
  --scope CLOUDFRONT \
  --region us-east-1 \
  --default-action Allow={} \
  --rules '[]' \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=cf-acl

# Enable WAF logging
aws wafv2 put-logging-configuration \
  --logging-configuration '{
    "ResourceArn": "arn:aws:wafv2:us-east-1:123456789:regional/webacl/my-web-acl/abc123",
    "LogDestinationConfigs": ["arn:aws:firehose:us-east-1:123456789:deliverystream/aws-waf-logs-my-stream"],
    "LoggingFilter": {
      "DefaultBehavior": "DROP",
      "Filters": [{
        "Behavior": "KEEP",
        "Conditions": [{"ActionCondition": {"Action": "BLOCK"}}],
        "Requirement": "MEETS_ANY"
      }]
    }
  }'

# AWS Managed Rule Groups:
# AWSManagedRulesCommonRuleSet:        OWASP Top 10 (core)
# AWSManagedRulesKnownBadInputsRuleSet: log4j, shell injection, path traversal
# AWSManagedRulesAdminProtectionRuleSet: protect /admin paths
# AWSManagedRulesSQLiRuleSet:           SQL injection
# AWSManagedRulesLinuxRuleSet:          Linux-specific attacks
# AWSManagedRulesWindowsRuleSet:        Windows-specific attacks
# AWSManagedRulesBotControlRuleSet:     bot management
# AWSManagedRulesATPRuleSet:            account takeover prevention
# AWSManagedRulesACFPRuleSet:           account creation fraud prevention
```

---

### 🟡 Q8. What is AWS Shield?
```bash
# Shield Standard: FREE, automatic DDoS protection for all AWS customers
# Protects: Layer 3/4 attacks (SYN floods, UDP floods, DNS amplification)
# Shield Advanced: paid ($3,000/month), Layer 7, 24/7 DRT, cost protection

# Enable Shield Advanced
aws shield create-subscription

# Check protection for resource
aws shield describe-protection \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/my-alb/abc123

# Create protection for specific resource
aws shield create-protection \
  --name my-alb-protection \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/my-alb/abc123

# Create protection group (protect multiple related resources)
aws shield create-protection-group \
  --protection-group-id all-albs \
  --aggregation SUM \
  --pattern ARBITRARY \
  --members \
    arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/alb1/abc \
    arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/app/alb2/def

# Associate WAF WebACL (Shield Advanced + WAF = automatic L7 DDoS response)
aws shield associate-drt-role \
  --role-arn arn:aws:iam::123456789:role/AWSShieldDRTAccessRole

aws shield associate-drt-log-bucket --log-bucket my-shield-logs

# Check active attacks
aws shield list-attacks \
  --start-time "StartTime=$(date -u -d '-24 hours' +%Y-%m-%dT%H:%M:%SZ),EndTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --resource-arns arn:aws:elasticloadbalancing:...

# Shield Standard: protects EC2, ELB, CloudFront, Route53, Global Accelerator
# Shield Advanced: also protects with:
# - Layer 7 attack mitigation (with WAF)
# - Real-time attack notifications
# - Cost protection (credit for DDoS-induced scaling)
# - DRT (DDoS Response Team) 24/7 access
```

---

### 🟡 Q9. What is Amazon VPC Security — Security Groups vs NACLs?
```bash
# Security Groups: stateful, instance-level firewall
# NACLs: stateless, subnet-level firewall

# ── Security Groups ───────────────────────────────────────────────
aws ec2 create-security-group \
  --group-name web-tier-sg \
  --description "Web tier security group" \
  --vpc-id vpc-12345678

# Inbound: HTTPS from internet
aws ec2 authorize-security-group-ingress \
  --group-id sg-web \
  --protocol tcp --port 443 \
  --cidr 0.0.0.0/0

# Inbound: HTTP redirect (only for ALB to handle redirect)
aws ec2 authorize-security-group-ingress \
  --group-id sg-web --protocol tcp --port 80 --cidr 0.0.0.0/0

# App tier: only from web tier SG (reference SG, not CIDR)
aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp --port 8080 \
  --source-group sg-web    # reference SG ID

# DB tier: only from app tier
aws ec2 authorize-security-group-ingress \
  --group-id sg-db \
  --protocol tcp --port 5432 \
  --source-group sg-app

# SG key facts:
# ✅ Stateful (return traffic automatically allowed)
# ✅ Rules always ALLOW (no explicit deny — just omit)
# ✅ Multiple SGs per instance
# ✅ References to other SGs work within same VPC/peered VPC

# ── NACLs (Network ACLs) ─────────────────────────────────────────
aws ec2 create-network-acl --vpc-id vpc-12345678

# NACLs are stateless — must allow both inbound AND outbound
# Inbound: allow HTTPS
aws ec2 create-network-acl-entry \
  --network-acl-id acl-abc123 \
  --ingress --rule-number 100 \
  --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow

# Inbound: allow return traffic (ephemeral ports 1024-65535)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-abc123 \
  --ingress --rule-number 200 \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow

# Inbound: block specific IP (impossible with SGs)
aws ec2 create-network-acl-entry \
  --network-acl-id acl-abc123 \
  --ingress --rule-number 50 \
  --protocol -1 \
  --cidr-block 203.0.113.5/32 \
  --rule-action deny

# Outbound: allow HTTPS and ephemeral ports
aws ec2 create-network-acl-entry \
  --network-acl-id acl-abc123 \
  --egress --rule-number 100 \
  --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow

aws ec2 create-network-acl-entry \
  --network-acl-id acl-abc123 \
  --egress --rule-number 200 \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# NACL vs SG comparison:
# Feature          | NACL              | Security Group
# State            | Stateless         | Stateful
# Level            | Subnet            | Instance/ENI
# Rules            | Allow + Deny      | Allow only
# Evaluation       | Number order      | All rules evaluated
# Return traffic   | Must allow explic | Auto allowed
# Use for          | Broad subnet deny | Fine-grained instance rules
```

---

### 🟡 Q10. What is AWS KMS?
```bash
# KMS: Key Management Service — create and manage encryption keys
# CMK types:
# AWS Managed:    auto-created for services (aws/s3, aws/rds) — no cost
# Customer Managed: you create, control, rotate — $1/month/key
# External (BYOK): import your own key material

# Create CMK (symmetric AES-256)
aws kms create-key \
  --description "My application encryption key" \
  --key-spec SYMMETRIC_DEFAULT \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "EnableRootAccess",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789:root"},
        "Action": "kms:*",
        "Resource": "*"
      },
      {
        "Sid": "AllowAppRole",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789:role/app-role"},
        "Action": ["kms:Decrypt","kms:GenerateDataKey","kms:DescribeKey"],
        "Resource": "*"
      },
      {
        "Sid": "AllowAdmins",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789:role/kms-admin"},
        "Action": ["kms:Create*","kms:Describe*","kms:Enable*","kms:List*",
                   "kms:Put*","kms:Update*","kms:Revoke*","kms:Disable*",
                   "kms:Get*","kms:Delete*","kms:ScheduleKeyDeletion"],
        "Resource": "*"
      }
    ]
  }' \
  --tags TagKey=Environment,TagValue=production \
  --multi-region

KEY_ID=$(aws kms create-key --query "KeyMetadata.KeyId" --output text --description "test")

# Create alias
aws kms create-alias \
  --alias-name alias/my-app-key \
  --target-key-id $KEY_ID

# Encrypt/decrypt
aws kms encrypt \
  --key-id alias/my-app-key \
  --plaintext "My secret data" \
  --query CiphertextBlob \
  --output text | base64 -d > encrypted.bin

aws kms decrypt \
  --ciphertext-blob fileb://encrypted.bin \
  --query Plaintext --output text | base64 -d

# Envelope encryption (for large data)
DATA_KEY=$(aws kms generate-data-key \
  --key-id alias/my-app-key \
  --key-spec AES_256)
PLAINTEXT_KEY=$(echo $DATA_KEY | jq -r .Plaintext | base64 -d)
ENCRYPTED_KEY=$(echo $DATA_KEY | jq -r .CiphertextBlob)
# Use PLAINTEXT_KEY to encrypt your data locally
# Store ENCRYPTED_KEY alongside the ciphertext
# To decrypt: call kms:Decrypt on ENCRYPTED_KEY to get PLAINTEXT_KEY

# Enable automatic key rotation
aws kms enable-key-rotation --key-id $KEY_ID
aws kms get-key-rotation-status --key-id $KEY_ID

# Multi-Region key (replicate to other regions)
aws kms replicate-key \
  --key-id arn:aws:kms:us-east-1:123456789:key/mrk-abc123 \
  --replica-region eu-west-1

# Key policy vs IAM policy:
# KMS is unique: resource policy (key policy) MUST explicitly allow IAM
# If key policy doesn't mention IAM role, IAM policy can't grant access
# Exception: if key policy has "Enable IAM User Permissions" → IAM policies work

# Schedule key deletion (min 7 days)
aws kms schedule-key-deletion \
  --key-id $KEY_ID \
  --pending-window-in-days 30

# Cancel deletion
aws kms cancel-key-deletion --key-id $KEY_ID
```

---

### 🟡 Q11. What is AWS CloudHSM?
```bash
# CloudHSM: dedicated Hardware Security Module (single-tenant, FIPS 140-2 Level 3)
# Use when: regulatory requires HSM, need custom crypto, PKCS#11, JCE, CNG
# vs KMS: KMS is multi-tenant (shared HSM), CloudHSM = dedicated hardware you control

# CloudHSM cluster
aws cloudhsmv2 create-cluster \
  --hsm-type hsm2m.medium \
  --subnet-ids subnet-11111111 subnet-22222222

# Add HSM to cluster
aws cloudhsmv2 create-hsm \
  --cluster-id cluster-abc123 \
  --availability-zone us-east-1a

# Initialise (generate cluster certificate) — via CloudHSM CLI
# /opt/cloudhsm/bin/cloudhsm-cli cluster initialize --cluster-id cluster-abc123

# Use cases for CloudHSM over KMS:
# - PKCS#11 or JCE application integration
# - SSL/TLS certificate private key protection
# - Oracle TDE (Transparent Data Encryption)
# - Custom cryptographic algorithms
# - Compliance: PCI DSS Level 1, eIDAS, HIPAA with audit control
```

---

### 🟡 Q12. What is Amazon Detective?
```bash
# Detective: investigate security findings — who, what, when, where, why
# Automatically builds a graph model from CloudTrail, VPC Flow Logs, GuardDuty
# Complements GuardDuty: GuardDuty = detect, Detective = investigate

# Enable Detective
aws detective create-graph

GRAPH_ARN=$(aws detective list-graphs --query "GraphList[0].Arn" --output text)

# Invite member accounts (from security account)
aws detective create-members \
  --graph-arn $GRAPH_ARN \
  --accounts \
    AccountId=123456789,EmailAddress=prod@mycompany.com \
    AccountId=987654321,EmailAddress=dev@mycompany.com

# Detective automatically analyses:
# AWS CloudTrail logs
# Amazon VPC Flow Logs
# Amazon GuardDuty findings
# AWS Organizations data

# Investigation workflow:
# 1. GuardDuty finding: "UnauthorizedAccess:IAMUser/ConsoleLogin"
# 2. Click "Investigate in Detective" from GuardDuty console
# 3. Detective shows: timeline, IP geolocation, other actions by same user
# 4. Related entities: other resources accessed, unusual patterns
# 5. Behaviour graph: visualise normal vs anomalous

# Detective key features:
# Entity timelines: see all activity for a specific IP, user, or resource
# Behaviour profiling: baseline normal vs current anomalous
# Finding groups: cluster related findings across accounts
# Investigation summary: natural language summary of investigation
```

---

### 🟡 Q13. What is AWS Firewall Manager?
```bash
# Firewall Manager: centrally manage WAF, Shield, SGs, NACLs across org

# Prerequisites: AWS Organizations + Security Hub + AWS Config enabled

# Create Firewall Manager policy (WAF)
aws fms put-policy \
  --policy '{
    "PolicyName": "OrgWideWAFPolicy",
    "SecurityServicePolicyData": {
      "Type": "WAFV2",
      "ManagedServiceData": "{
        \"type\": \"WAFV2\",
        \"preProcessRuleGroups\": [{
          \"overrideAction\": {\"type\": \"NONE\"},
          \"managedRuleGroupIdentifier\": {
            \"vendorName\": \"AWS\",
            \"managedRuleGroupName\": \"AWSManagedRulesCommonRuleSet\"
          }
        }],
        \"postProcessRuleGroups\": [],
        \"defaultAction\": {\"type\": \"ALLOW\"},
        \"overrideCustomerWebACLAssociation\": true
      }"
    },
    "ResourceType": "AWS::ElasticLoadBalancingV2::LoadBalancer",
    "IncludeMap": {"ORG_UNIT": ["ou-abc123-workloads"]},
    "RemediationEnabled": true,
    "DeleteUnusedFMManagedResources": true
  }'

# Create SG audit policy (enforce SG rules org-wide)
aws fms put-policy \
  --policy '{
    "PolicyName": "DenySSHFromInternet",
    "SecurityServicePolicyData": {
      "Type": "SECURITY_GROUPS_USAGE_AUDIT",
      "ManagedServiceData": "{
        \"type\": \"SECURITY_GROUPS_USAGE_AUDIT\",
        \"deleteUnusedSecurityGroups\": false,
        \"coalesceRedundantSecurityGroups\": false
      }"
    },
    "ResourceType": "AWS::EC2::SecurityGroup",
    "IncludeMap": {"ACCOUNT": ["123456789"]},
    "RemediationEnabled": false
  }'

# List non-compliant resources
aws fms list-compliance-status \
  --policy-id <policy-id> \
  --query "PolicyComplianceStatusList[?EvaluationResults[0].ComplianceStatus==\`NON_COMPLIANT\`]"

# Firewall Manager policy types:
# WAFV2:                 WAF on ALB, API GW, CloudFront
# SHIELD_ADVANCED:       Shield protection across resources
# SECURITY_GROUPS_COMMON: enforce common SG rules on EC2/ALB
# SECURITY_GROUPS_AUDIT: audit SG compliance
# NETWORK_ACL_COMMON:    enforce NACL rules on subnets
# DNS_FIREWALL:          Route53 Resolver DNS Firewall
# NETWORK_FIREWALL:      AWS Network Firewall across VPCs
```

---

### 🟡 Q14. What is AWS Network Firewall?
```bash
# Network Firewall: managed VPC-level stateful firewall (Suricata IPS rules)
# Positioned in: firewall subnet in each AZ

# Create firewall policy
aws network-firewall create-rule-group \
  --rule-group-name block-malware-domains \
  --type STATEFUL \
  --capacity 100 \
  --rule-group '{
    "StatefulRulesAndCustomActions": {
      "StatefulRules": [
        {
          "Action": "DROP",
          "Header": {
            "Direction": "ANY",
            "Protocol": "DNS",
            "Source": "ANY",
            "SourcePort": "ANY",
            "Destination": "ANY",
            "DestinationPort": "ANY"
          },
          "RuleOptions": [
            {"Keyword": "dns_query", "Settings": ["malware.example.com"]},
            {"Keyword": "noalert"}
          ]
        }
      ]
    }
  }'

# Create firewall policy with Suricata rules
aws network-firewall create-rule-group \
  --rule-group-name suricata-rules \
  --type STATEFUL \
  --capacity 1000 \
  --rule-group '{
    "StatefulRulesAndCustomActions": {
      "CustomActions": [],
      "StatefulRules": []
    },
    "RulesSource": {
      "RulesString": "drop http any any -> any any (http.method; content:\"OPTIONS\"; msg:\"Block OPTIONS method\"; sid:1001; rev:1;)\ndrop tls any any -> any any (tls.sni; content:\"malicious.com\"; msg:\"Block malicious domain\"; sid:1002; rev:1;)"
    }
  }'

aws network-firewall create-firewall-policy \
  --firewall-policy-name my-firewall-policy \
  --firewall-policy '{
    "StatelessDefaultActions": ["aws:forward_to_sfe"],
    "StatelessFragmentDefaultActions": ["aws:forward_to_sfe"],
    "StatefulRuleGroupReferences": [
      {"ResourceArn": "arn:aws:network-firewall:us-east-1:123456789:stateful-rulegroup/block-malware-domains"}
    ],
    "StatefulEngineOptions": {"RuleOrder": "STRICT_ORDER"},
    "StatefulDefaultActions": ["aws:drop_established","aws:alert_established"]
  }'

# Create firewall (in dedicated firewall subnet)
aws network-firewall create-firewall \
  --firewall-name my-vpc-firewall \
  --firewall-policy-arn arn:aws:network-firewall:... \
  --vpc-id vpc-12345678 \
  --subnet-mappings \
    SubnetId=subnet-fw-az1 \
    SubnetId=subnet-fw-az2

# Network Firewall deployment pattern:
# Internet → IGW → Firewall Subnet → App Subnet
# Route Table on IGW: 10.0.0.0/8 → VPC Endpoint (firewall)
# Route Table on App: 0.0.0.0/0 → VPC Endpoint (firewall)
```

---

### 🟡 Q15. What is Amazon Route53 Resolver DNS Firewall?
```bash
# DNS Firewall: block/allow DNS queries from VPC resources
# Can block: malware domains, crypto-mining, phishing, custom blocks

# Create domain list (block list)
aws route53resolver create-firewall-domain-list \
  --name blocked-malware-domains \
  --tags Key=Purpose,Value=security

aws route53resolver update-firewall-domains \
  --firewall-domain-list-id <list-id> \
  --operation ADD \
  --domains malware.example.com badsite.io "*.cryptominer.com"

# Import domains from S3
aws route53resolver import-firewall-domains \
  --firewall-domain-list-id <list-id> \
  --operation REPLACE \
  --domain-file-url s3://my-security-bucket/blocked-domains.txt

# Use AWS managed lists
# AWSManagedDomainsMalwareDomainList   — malware and botnet
# AWSManagedDomainsBotnetCommandandControl — C2 servers
# AWSManagedAggregatedThreatList       — combined threat list

# Create firewall rule group
aws route53resolver create-firewall-rule-group \
  --name my-dns-firewall \
  --tags Key=Environment,Value=production

aws route53resolver create-firewall-rule \
  --firewall-rule-group-id <group-id> \
  --firewall-domain-list-id <block-list-id> \
  --priority 100 \
  --action BLOCK \
  --block-response NXDOMAIN \
  --name block-malware

# Allow exceptions (higher priority)
aws route53resolver create-firewall-rule \
  --firewall-rule-group-id <group-id> \
  --firewall-domain-list-id <allow-list-id> \
  --priority 50 \
  --action ALLOW \
  --name allow-exceptions

# Associate with VPC
aws route53resolver associate-firewall-rule-group \
  --firewall-rule-group-id <group-id> \
  --vpc-id vpc-12345678 \
  --priority 100 \
  --name my-vpc-dns-firewall

# Alert mode (ALERT vs BLOCK) — test before enforcing
aws route53resolver create-firewall-rule \
  --action ALERT \
  --block-response NODATA \
  --priority 200
```

---

### 🟡 Q16. What is AWS Certificate Manager (ACM)?
```bash
# ACM: provision and manage SSL/TLS certificates
# Free for certificates used with AWS services (ALB, CloudFront, API GW)

# Request public certificate
aws acm request-certificate \
  --domain-name myapp.com \
  --subject-alternative-names "*.myapp.com" "api.myapp.com" \
  --validation-method DNS \
  --tags Key=Environment,Value=production

CERT_ARN=$(aws acm request-certificate \
  --domain-name myapp.com \
  --validation-method DNS \
  --query CertificateArn --output text)

# Get DNS validation records (add to Route53 or external DNS)
aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --query "Certificate.DomainValidationOptions[0].ResourceRecord"

# Auto-validate if domain is in Route53 (same account)
aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --query "Certificate.DomainValidationOptions[].ResourceRecord" | \
jq -r '.[] | "aws route53 change-resource-record-sets --hosted-zone-id Z123 --change-batch \"{\\\"Changes\\\":[{\\\"Action\\\":\\\"UPSERT\\\",\\\"ResourceRecordSet\\\":{\\\"Name\\\":\\\"" + .Name + "\\\",\\\"Type\\\":\\\"" + .Type + "\\\",\\\"TTL\\\":300,\\\"ResourceRecords\\\":[{\\\"Value\\\":\\\"" + .Value + "\\\"}]}}]}\""'

# Email validation (alternative)
aws acm request-certificate \
  --domain-name myapp.com \
  --validation-method EMAIL

# Import certificate (for external CAs)
aws acm import-certificate \
  --certificate fileb://cert.pem \
  --private-key fileb://private-key.pem \
  --certificate-chain fileb://chain.pem \
  --tags Key=Issuer,Value=LetsEncrypt

# List certificates
aws acm list-certificates \
  --certificate-statuses ISSUED \
  --query "CertificateSummaryList[].{Domain:DomainName,Expires:NotAfter,ARN:CertificateArn}" \
  --output table

# Certificate expiry alarm
aws cloudwatch put-metric-alarm \
  --alarm-name cert-expiry-warning \
  --namespace AWS/CertificateManager \
  --metric-name DaysToExpiry \
  --dimensions Name=CertificateArn,Value=$CERT_ARN \
  --statistic Minimum \
  --period 86400 \
  --threshold 30 \
  --comparison-operator LessThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:cert-alerts

# ACM Private CA (for internal certificates)
aws acm-pca create-certificate-authority \
  --certificate-authority-configuration '{
    "KeyAlgorithm": "RSA_4096",
    "SigningAlgorithm": "SHA512WITHRSA",
    "Subject": {
      "Country": "US",
      "Organization": "My Company",
      "OrganizationalUnit": "IT",
      "CommonName": "Internal CA"
    }
  }' \
  --certificate-authority-type ROOT \
  --tags Key=Purpose,Value=internal-pki
```

---

### 🟡 Q17. What is AWS Secrets Manager vs SSM Parameter Store?
| Feature | Secrets Manager | SSM Parameter Store |
|---------|----------------|-------------------|
| Cost | $0.40/secret/month | Free (Standard tier) |
| Auto-rotation | Built-in Lambda rotation | No (manual or custom) |
| Max size | 64 KB | 4 KB (Standard), 8 KB (Advanced) |
| Cross-region | Replication supported | No native replication |
| Versioning | Yes (via rotation) | Yes (LATEST + numbered) |
| Audit | CloudTrail | CloudTrail |
| Best for | Passwords, DB creds, API keys needing rotation | Config, feature flags, non-rotating secrets |

```bash
# Secrets Manager
aws secretsmanager create-secret \
  --name prod/database/credentials \
  --secret-string '{"username":"admin","password":"MyP@ss!","host":"db.us-east-1.rds.amazonaws.com","port":5432}'

aws secretsmanager rotate-secret \
  --secret-id prod/database/credentials \
  --rotation-rules '{"ScheduleExpression":"rate(30 days)"}'

# SSM Parameter Store
aws ssm put-parameter \
  --name "/prod/app/config/db-host" --value "db.us-east-1.rds.amazonaws.com" \
  --type String --tier Standard

aws ssm put-parameter \
  --name "/prod/app/secrets/api-key" \
  --value "sk-abc123" \
  --type SecureString \
  --key-id alias/my-app-key

# Parameter Store hierarchy
aws ssm get-parameters-by-path \
  --path "/prod/app/" \
  --recursive --with-decryption
```

---

### 🟡 Q18. What is Amazon Macie?
```bash
# Macie: ML-powered S3 data security — discover PII and sensitive data

aws macie2 enable-macie --status ENABLED

# Create sensitive data discovery job
aws macie2 create-classification-job \
  --job-type SCHEDULED \
  --name "Weekly-PII-Scan" \
  --schedule-frequency WEEKLY \
  --s3-job-definition '{
    "bucketDefinitions": [{
      "accountId": "123456789",
      "buckets": ["customer-data-bucket","orders-bucket","hr-bucket"]
    }]
  }' \
  --managed-data-identifier-selector ALL \
  --sampling-percentage 100 \
  --description "Weekly scan for PII in customer-facing buckets"

# Get sensitivity findings summary
aws macie2 get-findings-statistics \
  --finding-criteria '{"criterion": {"severity.description": {"eq": ["High","Critical"]}}}' \
  --group-by type

# Sensitive data types Macie detects:
# AWS credentials:    Access key IDs, secret keys
# Bank accounts:      ABA routing, IBAN, SWIFT codes
# Credit cards:       Visa, Mastercard, Amex, Discover
# Driver licences:    US state DLs
# Health data:        NPI, DEA, health plan IDs
# National ID:        SSN, passport, SIN (Canada)
# PII:                Name, DOB, email, phone, address
# Crypto keys:        RSA private keys, PGP keys

# Suppress findings for known-safe content
aws macie2 create-findings-filter \
  --name "SuppressTestData" \
  --action ARCHIVE \
  --finding-criteria '{
    "criterion": {
      "resourcesAffected.s3Bucket.name": {"eq": ["test-data-bucket"]},
      "type": {"eq": ["SensitiveData:S3Object/Personal"]}
    }
  }'

# Automated remediation via EventBridge
aws events put-rule \
  --name "MacieHighSeverity" \
  --event-pattern '{
    "source": ["aws.macie"],
    "detail-type": ["Macie Finding"],
    "detail": {"severity": {"description": ["High","Critical"]}}
  }'
```

---

### 🟡 Q19. What is AWS Security Lake?
```bash
# Security Lake (2023): centralised security data lake in OCSF format
# Automatically collects: CloudTrail, VPC Flow Logs, Route53, S3 data events, Lambda, EKS
# Stores in: S3 (your own account), Open Cybersecurity Schema Framework (OCSF)
# Query with: Athena, Security Analytics tools (Splunk, Datadog, CrowdStrike)

# Enable Security Lake
aws securitylake create-data-lake \
  --configurations '[{
    "region": "us-east-1",
    "encryptionConfiguration": {"kmsKeyId": "arn:aws:kms:us-east-1:123456789:key/abc123"},
    "replicationConfiguration": {"regions": ["eu-west-1"], "roleArn": "arn:aws:iam::123456789:role/security-lake-replication"},
    "lifecycleConfiguration": {
      "transitions": [{"days": 365, "storageClass": "GLACIER"}],
      "expiration": {"days": 2557}
    }
  }]' \
  --meta-store-manager-role-arn arn:aws:iam::123456789:role/AmazonSecurityLakeMetaStoreManagerV2

# Add log sources
aws securitylake create-aws-log-source \
  --sources '[
    {"regions": ["us-east-1"], "sourceName": "CLOUD_TRAIL_MGMT", "sourceVersion": "2.0"},
    {"regions": ["us-east-1"], "sourceName": "VPC_FLOW", "sourceVersion": "2.0"},
    {"regions": ["us-east-1"], "sourceName": "ROUTE53", "sourceVersion": "2.0"},
    {"regions": ["us-east-1"], "sourceName": "SH_FINDINGS", "sourceVersion": "2.0"},
    {"regions": ["us-east-1"], "sourceName": "EKS_AUDIT", "sourceVersion": "2.0"}
  ]'

# Add custom log source
aws securitylake create-custom-log-source \
  --source-name "my-application-logs" \
  --configuration '{
    "crawlerConfiguration": {"roleArn": "arn:aws:iam::123456789:role/security-lake-crawler-role"},
    "providerIdentity": {"externalId": "my-ext-id", "principal": "arn:aws:iam::123456789:role/log-provider-role"}
  }' \
  --event-classes "[\"AUTHENTICATION\",\"AUDIT\"]"

# Query Security Lake data with Athena
# Tables auto-created in: amazon_security_lake_glue_db_{region}
# cloudtrail_mgmt, vpc_flow, route53, sh_findings, eks_audit

aws athena start-query-execution \
  --query-string "
    SELECT
      eventtime,
      useridentity.arn,
      eventname,
      sourceipaddress,
      requestparameters
    FROM amazon_security_lake_glue_db_us_east_1.amazon_security_lake_table_us_east_1_cloud_trail_mgmt_2_0
    WHERE eventname IN ('DeleteBucket','ConsoleLogin')
      AND eventtime > '2026-06-01'
    ORDER BY eventtime DESC
    LIMIT 100
  " \
  --query-execution-context Database=amazon_security_lake_glue_db_us_east_1 \
  --result-configuration OutputLocation=s3://my-athena-results/
```

---

### 🟡 Q20. What is AWS IAM Access Analyzer?
```bash
# IAM Access Analyzer: identify resources accessible outside your org/account
# External access: S3 buckets, IAM roles, KMS keys, Lambda, SQS, Secrets Manager

# Create analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name my-org-analyzer \
  --type ORGANIZATION \    # ACCOUNT | ORGANIZATION
  --tags Key=Purpose,Value=security-audit

# List external access findings
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789:analyzer/my-org-analyzer \
  --filter '{"status":{"eq":["ACTIVE"]}}' \
  --query "findings[].{
    Resource:resource,
    Type:resourceType,
    Principal:principal,
    Actions:action,
    Condition:condition
  }" --output table

# Archive known-good finding (false positive)
aws accessanalyzer update-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789:analyzer/my-org-analyzer \
  --status ARCHIVED \
  --ids <finding-id>

# Unused access analysis (new — find unused permissions)
aws accessanalyzer create-analyzer \
  --analyzer-name unused-access-analyzer \
  --type ACCOUNT_UNUSED_ACCESS \
  --configuration '{
    "unusedAccess": {"unusedAccessAge": 90}
  }'

# Policy validation
aws accessanalyzer validate-policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }]
  }' \
  --policy-type IDENTITY_POLICY \
  --query "findings[].{Type:findingType,Detail:findingDetails}"

# Generate least-privilege policy from CloudTrail activity
aws accessanalyzer start-policy-generation \
  --policy-generation-details '{
    "principalArn": "arn:aws:iam::123456789:role/app-role"
  }' \
  --cloud-trail-details '{
    "accessRole": "arn:aws:iam::123456789:role/access-analyzer-role",
    "trailProperties": [{
      "cloudTrailArn": "arn:aws:cloudtrail:us-east-1:123456789:trail/my-trail",
      "allRegions": true
    }],
    "startTime": "2026-05-01T00:00:00Z",
    "endTime": "2026-06-01T00:00:00Z"
  }'

JOB_ID=$(aws accessanalyzer start-policy-generation ... --query "jobId" --output text)
aws accessanalyzer get-generated-policy --job-id $JOB_ID
```

---

### 🟡 Q21. What is Amazon Inspector v2?
```bash
# Inspector v2: automated vulnerability management for EC2, ECR images, Lambda
# Continuously scans — not periodic

aws inspector2 enable --resource-types EC2 ECR LAMBDA LAMBDA_CODE

# Get vulnerability summary
aws inspector2 get-counts-by-finding-status --output table

# Critical vulnerabilities
aws inspector2 list-findings \
  --filter-criteria '{
    "severity": [{"comparison": "EQUALS", "value": "CRITICAL"}],
    "findingStatus": [{"comparison": "EQUALS", "value": "ACTIVE"}],
    "fixAvailable": [{"comparison": "EQUALS", "value": "YES"}]
  }' \
  --sort-criteria field=INSPECTOR_SCORE,sortOrder=DESC \
  --max-results 20 \
  --query "findings[].{
    Title:title,
    Score:inspectorScore,
    Resource:resources[0].id,
    CVE:packageVulnerabilityDetails.cvss[0].score,
    Fix:packageVulnerabilityDetails.vulnerablePackages[0].fixedInVersion
  }"

# Suppress known false positives
aws inspector2 create-filter \
  --name "SuppressDevTools" \
  --action SUPPRESS \
  --filter-criteria '{
    "resourceTags": [{"comparison": "EQUALS", "key": "Environment", "value": "development"}],
    "severity": [{"comparison": "EQUALS", "value": "MEDIUM"}]
  }'

# SBOM export (Software Bill of Materials)
aws inspector2 create-sbom-export \
  --report-format CYCLONEDX_1_4 \
  --s3-destination '{
    "bucketName": "my-sbom-bucket",
    "keyPrefix": "sbom/",
    "kmsKeyArn": "arn:aws:kms:us-east-1:123456789:key/abc123"
  }'
```

---

### 🟡 Q22. What is AWS Audit Manager?
```bash
# Audit Manager: continuously collect evidence for compliance audits
# Frameworks: SOC 2, PCI DSS, HIPAA, NIST CSF, ISO 27001, FedRAMP, CIS

# Enable Audit Manager
aws auditmanager register-account \
  --delegation-configuration '{
    "roleArn": "arn:aws:iam::123456789:role/audit-manager-role",
    "roleType": "PROCESS_OWNER"
  }'

# Create assessment from framework
aws auditmanager create-assessment \
  --name "SOC2-2026-Assessment" \
  --description "Annual SOC 2 Type II assessment" \
  --assessment-reports-destination '{
    "destinationType": "S3",
    "destination": "s3://my-audit-reports"
  }' \
  --scope '{
    "awsAccounts": [{"id": "123456789", "emailAddress": "admin@mycompany.com", "name": "Production"}],
    "awsServices": [{"serviceName": "S3"},{"serviceName": "EC2"},{"serviceName": "RDS"}]
  }' \
  --roles '[{"roleArn": "arn:aws:iam::123456789:role/audit-manager-role", "roleType": "PROCESS_OWNER"}]' \
  --framework-id <soc2-framework-id>

# Get evidence for controls
aws auditmanager list-controls \
  --control-type Custom \
  --output table

# Create custom control
aws auditmanager create-control \
  --name "EncryptionAtRest" \
  --description "All S3 buckets must have server-side encryption enabled" \
  --control-mapping-sources '[{
    "sourceName": "Config-S3Encryption",
    "sourceDescription": "AWS Config S3 bucket encryption check",
    "sourceSetUpOption": "System_Controls_Mapping",
    "sourceType": "AWS_Config",
    "troubleshootingText": "Enable SSE-S3 or SSE-KMS on the bucket"
  }]' \
  --testing-information "Check S3 console → Properties → Default encryption" \
  --action-plan-instructions "Enable encryption on all S3 buckets" \
  --action-plan-title "Remediate unencrypted S3 buckets" \
  --tags Key=Framework,Value=SOC2

# Generate report
aws auditmanager create-assessment-report \
  --name "SOC2-Q2-2026" \
  --description "Q2 2026 SOC 2 evidence report" \
  --assessment-id <assessment-id> \
  --query-statement "SELECT * FROM assessments"
```

---

### 🟡 Q23. What is AWS Security Hub vs GuardDuty vs Inspector vs Macie?
| Service | What it does | Primary data source | Key metric |
|---------|------------|-------------------|-----------|
| **GuardDuty** | Detect threats and attacks | VPC Flow, CloudTrail, DNS | Findings |
| **Inspector** | Vulnerability scanning | EC2, ECR, Lambda | CVE scores |
| **Macie** | Sensitive data in S3 | S3 objects | PII findings |
| **Security Hub** | Aggregate all findings | All above + Config | Security score |
| **Detective** | Investigate security events | GuardDuty, CloudTrail, VPC | Investigation |
| **Audit Manager** | Continuous compliance evidence | Config, CloudTrail | Control coverage |

---

### 🟡 Q24. What is VPC Endpoints?
```bash
# VPC Endpoints: private connectivity to AWS services without internet
# Gateway Endpoints: S3, DynamoDB (free)
# Interface Endpoints (PrivateLink): all other services (cost per ENI)

# Gateway Endpoint (S3)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-abc123 rtb-def456 \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject","s3:PutObject"],
      "Resource": "arn:aws:s3:::my-app-bucket/*",
      "Condition": {
        "StringEquals": {"aws:PrincipalAccount": "123456789"}
      }
    }]
  }'

# Interface Endpoint (Secrets Manager)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-12345678 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.secretsmanager \
  --subnet-ids subnet-11111111 subnet-22222222 \
  --security-group-ids sg-endpoint \
  --private-dns-enabled     # apps use same DNS name, goes private

# Interface Endpoint (SSM — needed for Session Manager to work)
for svc in ssm ssmmessages ec2messages; do
  aws ec2 create-vpc-endpoint \
    --vpc-id vpc-12345678 \
    --vpc-endpoint-type Interface \
    --service-name com.amazonaws.us-east-1.$svc \
    --subnet-ids subnet-11111111 \
    --security-group-ids sg-endpoint \
    --private-dns-enabled
done

# List available endpoint services
aws ec2 describe-vpc-endpoint-services \
  --filter Name=service-type,Values=Interface \
  --query "ServiceDetails[?ServiceName contains 's3'].ServiceName"

# Enforce VPC endpoint usage (S3 bucket policy deny non-VPCe)
aws s3api put-bucket-policy --bucket my-secure-bucket --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::my-secure-bucket","arn:aws:s3:::my-secure-bucket/*"],
    "Condition": {
      "StringNotEquals": {
        "aws:SourceVpce": "vpce-abc123"
      }
    }
  }]
}'
```

---

### 🟡 Q25. What is AWS Resource-based policies?
```bash
# Resource-based policies: attached TO the resource (vs identity-based = on user/role)
# Support: S3, SQS, SNS, Lambda, KMS, ECR, Secrets Manager, API Gateway, Glacier

# S3 bucket policy examples
aws s3api put-bucket-policy --bucket my-bucket --policy '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::my-bucket","arn:aws:s3:::my-bucket/*"],
      "Condition": {
        "Bool": {"aws:SecureTransport": "false"}
      }
    },
    {
      "Sid": "AllowCrossAccountRead",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::987654321:role/data-analytics-role"},
      "Action": ["s3:GetObject","s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/reports/*"
      ]
    },
    {
      "Sid": "AllowCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-bucket/cloudtrail/*",
      "Condition": {
        "StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}
      }
    }
  ]
}'

# Lambda resource-based policy (allow API GW to invoke)
aws lambda add-permission \
  --function-name my-function \
  --statement-id allow-api-gateway \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn arn:aws:execute-api:us-east-1:123456789:my-api/*/POST/orders \
  --source-account 123456789

# SQS resource-based policy (allow SNS to send)
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789/my-queue \
  --attributes '{"Policy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"sns.amazonaws.com\"},\"Action\":\"sqs:SendMessage\",\"Resource\":\"arn:aws:sqs:us-east-1:123456789:my-queue\",\"Condition\":{\"ArnLike\":{\"aws:SourceArn\":\"arn:aws:sns:us-east-1:123456789:my-topic\"}}}]}"}'
```


---

# PART 2 — AWS MIGRATION & TRANSFER

---

### 🟢 Q26. What is AWS Migration Hub?
```bash
# Migration Hub: single pane of glass for tracking migrations to AWS
# Integrates with: AWS Application Migration Service, DMS, Partner tools

# Create home region (required first)
aws migrationhub-config create-home-region-control \
  --home-region us-east-1 \
  --target '{"Type": "ACCOUNT"}'

# Track application migration
aws mgh create-progress-update-stream \
  --progress-update-stream-name my-migration-stream \
  --dry-run false

# Associate discovered server with application
aws mgh associate-discovered-resource \
  --progress-update-stream my-migration-stream \
  --migration-task-name server-001 \
  --discovered-resource ConfigurationId=d-server-12345,Description="Web Server 01"

# Migration phases tracked:
# Not Started → In Progress → Completed → Failed

# Notify task update
aws mgh notify-migration-task-state \
  --progress-update-stream my-migration-stream \
  --migration-task-name server-001 \
  --task '{"Status": "IN_PROGRESS", "StatusDetail": "Replicating data", "ProgressPercent": 45}' \
  --update-date-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --next-update-seconds 3600

# Migration strategies (7Rs):
# Retire:       decommission — not needed
# Retain:       keep on-prem for now
# Rehost:       lift-and-shift (EC2)
# Replatform:   move with small optimisations (RDS, Elastic Beanstalk)
# Repurchase:   move to SaaS (Salesforce, ServiceNow)
# Refactor:     re-architect for cloud-native (Lambda, containers)
# Relocate:     VMware Cloud on AWS (no change)
```

---

### 🟢 Q27. What is AWS Application Migration Service (MGN)?
```bash
# MGN (CloudEndure successor): lift-and-shift server replication to AWS
# Supports: physical servers, VMware, Hyper-V, other clouds (Azure, GCP, OCI)
# RPO: seconds | RTO: minutes | Minimal cutover downtime

# Initialize MGN
aws mgn initialize-service

# Install agent on source server (Linux)
wget -O ./aws-replication-installer-init.py \
  https://aws-application-migration-service-us-east-1.s3.amazonaws.com/latest/linux/AwsReplicationWindowsInstaller.exe
sudo python3 aws-replication-installer-init.py \
  --region us-east-1 \
  --aws-access-key-id $AWS_ACCESS_KEY_ID \
  --aws-secret-access-key $AWS_SECRET_ACCESS_KEY

# List source servers (after agent installed)
aws mgn describe-source-servers \
  --query "items[].{Hostname:sourceServerID,State:dataReplicationInfo.dataReplicationState,LastSeen:dataReplicationInfo.lastSnapshotDateTime}" \
  --output table

# Configure launch settings per server
aws mgn update-launch-configuration \
  --source-server-id s-abc12345 \
  --name "web-server-01" \
  --launch-disposition STARTED \
  --target-instance-type-right-sizing-method BASIC \
  --enable-map-auto-tagging true

# Configure replication settings
aws mgn update-replication-configuration \
  --source-server-id s-abc12345 \
  --replication-server-instance-type t3.small \
  --staging-area-subnet-id subnet-11111111 \
  --staging-area-tags Key=Purpose,Value=MGN-Staging \
  --use-dedicated-replication-server false \
  --bandwidth-throttling 0

# Test migration (non-disruptive)
aws mgn start-test \
  --source-server-ids s-abc12345

# Wait for test to complete, then validate
aws mgn describe-source-servers \
  --filters lifeCycle.state=TESTING \
  --query "items[0].lifeCycle"

# Finalize cutover
aws mgn start-cutover \
  --source-server-ids s-abc12345

# After cutover, finalize (terminates replication)
aws mgn finalize-cutover \
  --source-server-ids s-abc12345

# Archive completed server
aws mgn disconnect-from-service \
  --source-server-id s-abc12345

# EC2 launch template per server
aws mgn update-launch-configuration-template \
  --launch-configuration-template-id lct-abc123 \
  --post-launch-actions '{
    "deployment": "TEST_AND_CUTOVER",
    "s3LogBucket": "my-mgn-logs",
    "s3LogBucketPrefix": "mgn-post-launch/",
    "ssmDocuments": [{
      "actionName": "InstallAntivirus",
      "ssmDocumentName": "AWS-RunShellScript",
      "timeoutSeconds": 300,
      "parameters": {"commands": ["yum install -y clamd"]}
    }]
  }'
```

---

### 🟡 Q28. What is AWS Database Migration Service (DMS)?
```bash
# DMS: migrate databases with minimal downtime
# Supports: homogeneous (Oracle→Oracle) and heterogeneous (Oracle→PostgreSQL)
# Modes: Full Load | Full Load + CDC | CDC only

# Create replication instance
aws dms create-replication-instance \
  --replication-instance-identifier my-dms-instance \
  --replication-instance-class dms.t3.medium \
  --allocated-storage 100 \
  --vpc-security-group-ids sg-abc123 \
  --availability-zone us-east-1a \
  --replication-subnet-group-identifier my-dms-subnet-group \
  --publicly-accessible false \
  --multi-az true \
  --engine-version 3.5.3 \
  --auto-minor-version-upgrade true

# Create source endpoint (on-prem Oracle)
aws dms create-endpoint \
  --endpoint-identifier source-oracle \
  --endpoint-type source \
  --engine-name oracle \
  --username dms_user \
  --password "SourceP@ss!" \
  --server-name 192.168.1.100 \
  --port 1521 \
  --database-name MYDB \
  --oracle-settings '{
    "useDirectPathFullLoad": true,
    "parallelAsmReadThreads": 4,
    "readAheadBlocks": 150000,
    "allowSelectNestedTables": true,
    "standbyDelayTime": 0
  }'

# Create target endpoint (Aurora PostgreSQL)
aws dms create-endpoint \
  --endpoint-identifier target-aurora-pg \
  --endpoint-type target \
  --engine-name aurora-postgresql \
  --username admin \
  --password "TargetP@ss!" \
  --server-name my-aurora-cluster.cluster-abc.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --database-name myapp_db \
  --postgre-sql-settings '{
    "executeTimeout": 100,
    "maxFileSize": 65536,
    "captureDdls": true
  }'

# Test endpoint connections
aws dms test-connection \
  --replication-instance-arn arn:aws:dms:us-east-1:123456789:rep:my-dms-instance \
  --endpoint-arn arn:aws:dms:us-east-1:123456789:endpoint:source-oracle

# Create migration task (Full Load + CDC)
aws dms create-replication-task \
  --replication-task-identifier oracle-to-aurora-migration \
  --source-endpoint-arn arn:aws:dms:...:endpoint:source-oracle \
  --target-endpoint-arn arn:aws:dms:...:endpoint:target-aurora-pg \
  --replication-instance-arn arn:aws:dms:...:rep:my-dms-instance \
  --migration-type full-load-and-cdc \
  --table-mappings '{
    "rules": [
      {
        "rule-type": "selection",
        "rule-id": "1",
        "rule-name": "include-all",
        "object-locator": {"schema-name": "MYSCHEMA", "table-name": "%"},
        "rule-action": "include"
      },
      {
        "rule-type": "transformation",
        "rule-id": "2",
        "rule-name": "lowercase-schema",
        "rule-action": "convert-lowercase",
        "rule-target": "schema",
        "object-locator": {"schema-name": "MYSCHEMA"}
      },
      {
        "rule-type": "transformation",
        "rule-id": "3",
        "rule-name": "lowercase-tables",
        "rule-action": "convert-lowercase",
        "rule-target": "table",
        "object-locator": {"schema-name": "%", "table-name": "%"}
      }
    ]
  }' \
  --replication-task-settings '{
    "TargetMetadata": {
      "TargetSchema": "",
      "SupportLobs": true,
      "LobMaxSize": 102400
    },
    "FullLoadSettings": {
      "TargetTablePrepMode": "TRUNCATE_BEFORE_LOAD",
      "CreatePkAfterFullLoad": true,
      "StopTaskCachedChangesApplied": false,
      "StopTaskCachedChangesNotApplied": false,
      "MaxFullLoadSubTasks": 8,
      "TransactionConsistencyTimeout": 600,
      "CommitRate": 50000
    },
    "Logging": {
      "EnableLogging": true,
      "LogComponents": [
        {"Id": "SOURCE_UNLOAD", "Severity": "LOGGER_SEVERITY_DEFAULT"},
        {"Id": "TARGET_LOAD", "Severity": "LOGGER_SEVERITY_DEFAULT"},
        {"Id": "TASK_MANAGER", "Severity": "LOGGER_SEVERITY_DEBUG"}
      ]
    }
  }'

# Start task
aws dms start-replication-task \
  --replication-task-arn arn:aws:dms:...:task:oracle-to-aurora-migration \
  --start-replication-task-type start-replication

# Monitor task progress
aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=arn:aws:dms:...:task:oracle-to-aurora-migration \
  --query "ReplicationTasks[0].{Status:Status,Progress:ReplicationTaskStats.FullLoadProgressPercent,Lag:ReplicationTaskStats.CDCLatencyTarget}"

# DMS Schema Conversion Tool (SCT) for heterogeneous migrations
# SCT converts schema + stored procedures from Oracle/SQL Server → PostgreSQL/MySQL/Aurora
# Install SCT locally, connect to source and target, review and apply conversions
```

---

### 🟡 Q29. What is AWS Schema Conversion Tool (SCT)?
```bash
# SCT: convert database schema, stored procedures, functions, triggers
# Source → Target pairs:
# Oracle          → PostgreSQL, MySQL, Aurora, Redshift
# SQL Server      → PostgreSQL, MySQL, Aurora
# IBM DB2         → PostgreSQL, MySQL
# Teradata        → Redshift, S3 (for analytics)
# Sybase          → PostgreSQL, MySQL
# MySQL           → PostgreSQL, Aurora

# SCT runs locally (desktop app or CLI)
# Key features:
# 1. Automatic schema conversion
# 2. Conversion assessment report (complexity, effort)
# 3. Extension pack (emulate Oracle/SQL Server built-ins in PostgreSQL)
# 4. Multi-server projects
# 5. Integration with DMS for data migration

# SCT CLI commands
aws-sct-cli transform-schema \
  --source-engine oracle \
  --source-connection '{"endpoint":"db.corp.local","port":1521,"username":"dms_user","password":"P@ss!","database":"MYDB"}' \
  --target-engine aurora-postgresql \
  --target-connection '{"endpoint":"my-aurora.cluster-abc.us-east-1.rds.amazonaws.com","port":5432,"username":"admin","password":"P@ss!","database":"myapp"}' \
  --schema-name MYSCHEMA \
  --output-path ./conversion-report/

# Common conversion challenges:
# Oracle sequences   → PostgreSQL SEQUENCE or IDENTITY columns
# ROWNUM             → ROW_NUMBER() OVER (ORDER BY ...)
# Oracle packages    → PostgreSQL schemas + functions
# DATE (Oracle)      → TIMESTAMP in PostgreSQL
# SYSDATE            → NOW() or CURRENT_TIMESTAMP
# NVL()              → COALESCE()
# DECODE()           → CASE WHEN ... THEN ... END
# CONNECT BY PRIOR   → Recursive CTEs WITH RECURSIVE
# Bulk collect/FORALL→ rewrite as set-based SQL
```

---

### 🟡 Q30. What is AWS DataSync?
```bash
# DataSync: automated, accelerated data transfer (NFS, SMB, S3, EFS, FSx)
# Speed: up to 10 Gbps per task, parallel transfers, compression
# Use for: one-time migration or ongoing sync

# Create DataSync agent (on-prem VM or EC2)
# Deploy agent AMI in your environment
aws datasync create-agent \
  --activation-key <key-from-agent-ui> \
  --agent-name my-on-prem-agent \
  --vpc-endpoint-id vpce-abc123 \
  --subnet-arns arn:aws:ec2:us-east-1:123456789:subnet/subnet-11111111 \
  --security-group-arns arn:aws:ec2:us-east-1:123456789:security-group/sg-abc123

# Create NFS source location (on-prem NFS)
aws datasync create-location-nfs \
  --server-hostname 192.168.1.100 \
  --subdirectory /data/exports \
  --on-prem-config '{"AgentArns": ["arn:aws:datasync:us-east-1:123456789:agent/my-agent"]}' \
  --mount-options '{"Version": "NFS4_1"}'

# Create S3 destination location
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::my-destination-bucket \
  --subdirectory /migrated-data \
  --s3-config '{"BucketAccessRoleArn": "arn:aws:iam::123456789:role/datasync-s3-role"}' \
  --s3-storage-class STANDARD

# Create EFS destination
aws datasync create-location-efs \
  --efs-filesystem-arn arn:aws:elasticfilesystem:us-east-1:123456789:file-system/fs-abc123 \
  --ec2-config '{
    "SubnetArn": "arn:aws:ec2:us-east-1:123456789:subnet/subnet-11111111",
    "SecurityGroupArns": ["arn:aws:ec2:us-east-1:123456789:security-group/sg-efs"]
  }' \
  --subdirectory /

# Create transfer task
aws datasync create-task \
  --source-location-arn arn:aws:datasync:...:location/nfs-loc \
  --destination-location-arn arn:aws:datasync:...:location/s3-loc \
  --name "nfs-to-s3-migration" \
  --options '{
    "VerifyMode": "ONLY_FILES_TRANSFERRED",
    "Atime": "BEST_EFFORT",
    "Mtime": "PRESERVE",
    "Uid": "NONE",
    "Gid": "NONE",
    "PreserveDeletedFiles": "REMOVE",
    "PreserveDevices": "NONE",
    "PosixPermissions": "NONE",
    "BytesPerSecond": -1,
    "TaskQueueing": "ENABLED",
    "LogLevel": "TRANSFER",
    "TransferMode": "CHANGED"
  }' \
  --schedule '{"ScheduleExpression": "cron(0 2 * * ? *)"}' \
  --cloud-watch-log-group-arn arn:aws:logs:us-east-1:123456789:log-group:/datasync/tasks

# Start task execution
aws datasync start-task-execution \
  --task-arn arn:aws:datasync:...:task:task-abc123 \
  --includes '[{"FilterType": "SIMPLE_PATTERN", "Value": "*.csv|*.parquet"}]'

# Monitor execution
aws datasync describe-task-execution \
  --task-execution-arn arn:aws:datasync:...:task:task-abc123/execution/exec-abc123 \
  --query "{Status:Status,FilesTransferred:Result.FilesTransferred,BytesTransferred:Result.BytesTransferred}"

# DataSync supported sources:
# NFS, SMB, HDFS, S3 (cross-account/region), EFS, FSx for Windows/Lustre/OpenZFS
# Object storage: Azure Blob, Google Cloud Storage, Wasabi, Backblaze B2
```

---

### 🟡 Q31. What is AWS Snow Family?
```bash
# Snow Family: physical devices for offline data transfer and edge computing
# Snowcone:  8 TB, smallest, 2 vCPUs, 4 GB RAM — IoT, remote office
# Snowball Edge Storage Optimised: 80 TB usable, 40 vCPUs, 80 GB RAM — migration
# Snowball Edge Compute Optimised: 42 TB usable, 52 vCPUs, 208 GB RAM + GPU — edge ML
# Snowmobile: 100 PB shipping container — datacenter-scale migration

# Order a Snowball Edge
aws snowball create-job \
  --job-type IMPORT \
  --resources '{
    "S3Resources": [{
      "BucketArn": "arn:aws:s3:::my-migration-bucket",
      "KeyRange": {"BeginMarker": "", "EndMarker": ""},
      "TargetOnDeviceServices": [{"Name": "NONE_FSxW", "TransferOption": "IMPORT"}]
    }]
  }' \
  --address-id ADID-abc123 \
  --kms-key-arn arn:aws:kms:us-east-1:123456789:key/abc123 \
  --role-arn arn:aws:iam::123456789:role/snowball-role \
  --shipping-option SECOND_DAY \
  --snow-ball-capacity-preference T80 \
  --snow-ball-type EDGE_STORAGE_OPTIMIZED \
  --device-configuration '{
    "SnowconeDeviceConfiguration": {"WirelessConnection": {"IsWifiEnabled": false}}
  }'

# Check job status
aws snowball describe-job --job-id JID-abc123

# When device arrives:
# 1. Connect to power and network
# 2. Unlock device (get unlock code from console)
# aws snowball get-job-unlock-code --job-id JID-abc123
# 3. Configure Snowball client
# snowballEdge configure --endpoint https://192.168.1.20 --manifest <manifest-file>
# 4. Copy data
# snowballEdge cp -r /local/data/ s3://my-migration-bucket/
# 5. Validate checksums
# snowballEdge validate -s s3://my-migration-bucket/
# 6. Ship device back to AWS

# Snowball Edge as EC2 edge:
# Run EC2 instances locally on Snowball
aws ec2 run-instances \
  --image-id <ami-id-from-snowball> \
  --instance-type sbe-c.xlarge \   # Snowball-specific instance types
  --endpoint https://192.168.1.20:8008 \
  --region snow

# Snow Family use cases:
# Data migration: petabyte-scale, low-bandwidth location
# Edge computing: oil rigs, ships, military, manufacturing
# Temporary storage: disaster recovery site
# ML inference at edge: Snowball Edge Compute + GPU
```

---

### 🟡 Q32. What is AWS Transfer Family?
```bash
# Transfer Family: managed SFTP, FTPS, FTP, AS2 server backed by S3 or EFS
# No servers to manage — fully managed file transfer endpoint

# Create SFTP server
aws transfer create-server \
  --protocols SFTP \
  --domain S3 \
  --identity-provider-type SERVICE_MANAGED \
  --endpoint-type VPC \
  --endpoint-details '{
    "VpcId": "vpc-12345678",
    "SubnetIds": ["subnet-11111111", "subnet-22222222"],
    "SecurityGroupIds": ["sg-sftp-server"],
    "AddressAllocationIds": ["eipalloc-abc123", "eipalloc-def456"]
  }' \
  --logging-role arn:aws:iam::123456789:role/transfer-logging-role \
  --security-policy-name TransferSecurityPolicy-2024-01 \
  --structured-log-destinations arn:aws:logs:us-east-1:123456789:log-group:/transfer/sftp \
  --tags Key=Environment,Value=production

SERVER_ID=$(aws transfer create-server --query "ServerId" --output text ...)

# Create SFTP user
aws transfer create-user \
  --server-id $SERVER_ID \
  --user-name sftp-partner \
  --home-directory "/my-sftp-bucket/partners/acme" \
  --home-directory-type LOGICAL \
  --home-directory-mappings '[{
    "Entry": "/",
    "Target": "/my-sftp-bucket/partners/acme"
  }]' \
  --role arn:aws:iam::123456789:role/transfer-user-role \
  --tags Key=Partner,Value=Acme

# Add SSH public key
aws transfer import-ssh-public-key \
  --server-id $SERVER_ID \
  --user-name sftp-partner \
  --ssh-public-key-body "ssh-rsa AAAAB3NzaC1yc2E..."

# Password-based auth (Lambda custom IdP)
aws transfer create-server \
  --protocols SFTP \
  --identity-provider-type AWS_LAMBDA \
  --identity-provider-details '{
    "Function": "arn:aws:lambda:us-east-1:123456789:function:sftp-auth",
    "InvocationRole": "arn:aws:iam::123456789:role/transfer-invocation-role"
  }'

# Lambda auth function
def sftp_auth_handler(event, context):
    username = event["username"]
    password = event.get("password")
    # Validate against your auth system
    if validate_user(username, password):
        return {
            "Role": f"arn:aws:iam::123456789:role/sftp-{username}-role",
            "HomeDirectoryType": "LOGICAL",
            "HomeDirectoryDetails": json.dumps([
                {"Entry": "/", "Target": f"/my-bucket/{username}"}
            ]),
            "PublicKeys": []
        }
    return {}   # empty = auth failed

# AS2 connector (B2B EDI exchange)
aws transfer create-agreement \
  --server-id $SERVER_ID \
  --local-profile-id <local-profile> \
  --partner-profile-id <partner-profile> \
  --base-directory /bucket/as2/ \
  --access-role arn:aws:iam::123456789:role/transfer-as2-role

# Workflow automation (post-upload processing)
aws transfer create-workflow \
  --description "Process uploaded files" \
  --steps '[
    {
      "Type": "TAG",
      "TagStepDetails": {
        "Name": "TagUploads",
        "Tags": [{"Key": "UploadedBy", "Value": "${transfer:UserName}"}]
      }
    },
    {
      "Type": "COPY",
      "CopyStepDetails": {
        "Name": "CopyToProcessing",
        "DestinationFileLocation": {
          "S3FileLocation": {"Bucket": "processing-bucket", "Key": "${transfer:UploadDate}/${transfer:FilePath}"}
        },
        "OverwriteExisting": "TRUE"
      }
    },
    {
      "Type": "LAMBDA",
      "CustomStepDetails": {
        "Name": "TriggerProcessing",
        "Target": "arn:aws:lambda:us-east-1:123456789:function:process-file",
        "TimeoutSeconds": 300
      }
    },
    {
      "Type": "DELETE",
      "DeleteStepDetails": {"Name": "DeleteOriginal", "SourceFileLocation": "${original.file}"}
    }
  ]'
```

---

### 🟡 Q33. What is AWS Storage Gateway?
```bash
# Storage Gateway: hybrid cloud storage — extend on-prem to S3, EBS, Glacier
# Modes:
# S3 File Gateway:    NFS/SMB → S3 (local cache)
# FSx File Gateway:   SMB → FSx for Windows (local cache)
# Volume Gateway:     iSCSI volumes backed by S3 (cached) or snapshots (stored)
# Tape Gateway:       Virtual tape library backed by S3/Glacier

# Activate S3 File Gateway
# 1. Deploy gateway AMI (VMware, Hyper-V, KVM, EC2, hardware appliance)
# 2. Activate via CLI
aws storagegateway activate-gateway \
  --activation-key <key-from-gateway-local-ui> \
  --gateway-name my-file-gateway \
  --gateway-timezone GMT-5:00 \
  --gateway-region us-east-1 \
  --gateway-type FILE_S3

GATEWAY_ARN=$(aws storagegateway list-gateways --query "Gateways[0].GatewayARN" --output text)

# Add cache disk
aws storagegateway add-cache \
  --gateway-arn $GATEWAY_ARN \
  --disk-ids <disk-id>

# Create NFS file share
aws storagegateway create-nfs-file-share \
  --gateway-arn $GATEWAY_ARN \
  --role arn:aws:iam::123456789:role/storage-gateway-role \
  --location-arn arn:aws:s3:::my-storage-bucket \
  --client-list "192.168.0.0/24" \
  --squash NoSquash \
  --read-only false \
  --cache-attributes '{"CacheStaleTimeoutInSeconds": 300}' \
  --default-storage-class S3_STANDARD \
  --file-share-name my-share \
  --nfs-file-share-defaults '{
    "FileMode": "0666",
    "DirectoryMode": "0777",
    "GroupId": 65534,
    "OwnerId": 65534
  }' \
  --tags Key=Environment,Value=production

# Create SMB file share (with AD auth)
aws storagegateway create-smb-file-share \
  --gateway-arn $GATEWAY_ARN \
  --role arn:aws:iam::123456789:role/storage-gateway-role \
  --location-arn arn:aws:s3:::my-storage-bucket \
  --authentication ActiveDirectory \
  --valid-user-list "DOMAIN\\share-users" \
  --admin-user-list "DOMAIN\\it-admins" \
  --smb-guest-access-enabled false \
  --default-storage-class S3_INTELLIGENT_TIERING \
  --object-acl bucket-owner-full-control

# Volume Gateway (iSCSI block storage)
aws storagegateway create-cached-iscsi-volume \
  --gateway-arn $GATEWAY_ARN \
  --volume-size-in-bytes 107374182400 \   # 100 GB
  --target-name my-iscsi-target \
  --network-interface-id 192.168.1.50 \
  --client-token $(uuidgen)

# Tape Gateway
aws storagegateway create-tapes \
  --gateway-arn $GATEWAY_ARN \
  --tape-size-in-bytes 107374182400 \
  --client-token $(uuidgen) \
  --num-tapes-to-create 10 \
  --tape-barcode-prefix SGTP \
  --pool-id Glacier   # Glacier | Deep Archive
```

---

### 🟡 Q34. What is AWS Application Discovery Service?
```bash
# Application Discovery: discover and assess on-premises servers before migration
# Two methods:
# Agentless: VMware vCenter connector (limited data — VM config, perf)
# Agent-based: install on each server (full data — processes, connections, perf)

# Create discovery agent activation
aws discovery create-tags \
  --configuration-ids <config-id> \
  --tags '{"Environment": "production", "Datacenter": "DC1"}'

# Start data collection
aws discovery start-data-collection-by-agent-ids \
  --agent-ids <agent-id-1> <agent-id-2>

# List discovered servers
aws discovery describe-configurations \
  --configuration-type SERVER \
  --query "configurations[].{HostName:host.hostName,OS:os.name,CPU:performance.numCpu,RAM:performance.totalRamInMB}"

# Export discovery data
aws discovery start-export-task \
  --export-data-format CSV \
  --filters '[{
    "name": "agentIds",
    "values": ["<agent-id>"],
    "condition": "EQUALS"
  }]' \
  --start-time $(date -u -d '-30 days' +%Y-%m-%dT%H:%M:%SZ)

# Create application grouping
aws discovery create-application \
  --name "E-Commerce Application" \
  --description "Main customer-facing e-commerce stack"

aws discovery associate-configuration-items-to-application \
  --application-configuration-id app-abc123 \
  --configuration-ids server-001 server-002 server-003

# Agentless connector (VMware)
# Deploy OVA in vCenter
# Register connector
aws discovery create-tags \
  --configuration-ids <connector-config-id> \
  --tags '{"vCenterName": "VC-DC1"}'

# Network visualization (dependency mapping)
aws discovery list-server-neighbors \
  --configuration-id server-001 \
  --port-info-behavior include \
  --query "neighbors[].{DestinationPort:destinationPort,Protocol:transportProtocol,Count:connectionCount}"
```

---

### 🟡 Q35. What is AWS Migration Evaluator?
```bash
# Migration Evaluator (formerly TSO Logic): business case and cost analysis
# Analyses: actual utilisation data → right-sizes for AWS → builds TCO comparison

# Process:
# 1. Install collector (on-prem agent or agentless VMware)
# 2. 1-4 weeks of data collection
# 3. Upload to Migration Evaluator
# 4. Receive business case report

# Migration Evaluator report includes:
# Current on-premises TCO (servers, storage, network, facilities, staff)
# Projected AWS cost (right-sized EC2, RDS, EBS)
# 3-5 year comparison
# Recommended instance types per workload
# Reserved Instance/Savings Plans strategy
# Suggested migration waves

# CLI usage
aws migrationevaluator create-assessment \
  --client-version "2.0" \
  --status ACTIVE

aws migrationevaluator start-assessment \
  --assessment-id <id>

# Note: Most Migration Evaluator workflows use the GUI and downloadable collector
```

---

### 🟡 Q36. What is AWS Mainframe Modernisation?
```bash
# AWS Mainframe Modernization: migrate and modernise IBM z/OS and Unisys mainframes
# Patterns:
# Refactor: rewrite COBOL → Java/Python (Micro Focus, NTT)
# Replatform: run COBOL as-is on AWS Blu Age/Micro Focus runtime

# Create M2 environment (Micro Focus - runtime)
aws m2 create-environment \
  --name my-mainframe-env \
  --engine-type microfocus \
  --engine-version 9.0 \
  --instance-type m2.c5.large \
  --storage-configurations '[{
    "efs": {
      "fileSystemId": "fs-abc123",
      "mountPoint": "/m2/mount/app"
    }
  }]' \
  --high-availability-config '{"desiredCapacity": 2}' \
  --security-group-ids sg-abc123 \
  --subnet-ids subnet-11111111 subnet-22222222

# Deploy application
aws m2 create-application \
  --name my-cobol-app \
  --engine-type microfocus \
  --definition content=fileb://application-definition.yaml

aws m2 create-deployment \
  --application-id app-abc123 \
  --application-version 1 \
  --environment-id env-abc123 \
  --client-token $(uuidgen)

# Supported mainframe components:
# COBOL batch jobs → JCL equivalent
# VSAM files → Amazon DynamoDB or S3
# CICS transactions → REST/SOAP APIs or ECS tasks
# DB2 → Amazon RDS for PostgreSQL
# IMS DB → DynamoDB
# MQ → Amazon MQ or SQS
# JES (Job Entry Subsystem) → AWS Step Functions or Batch
```

---

### 🟡 Q37. What is the AWS Cloud Migration best practices?
```bash
# Migration phases: Assess → Mobilise → Migrate & Modernise

# ── ASSESS ────────────────────────────────────────────────────────
# 1. Portfolio discovery (Application Discovery Service + Migration Evaluator)
# 2. Application dependency mapping
# 3. Migration readiness assessment
# 4. Business case development (TCO analysis)
# 5. Prioritize workloads (7Rs for each app)

# ── MOBILISE ──────────────────────────────────────────────────────
# 1. Landing zone setup (Control Tower + AFT)
# 2. Network connectivity (Direct Connect or VPN)
# 3. Establish operations model (RACI, runbooks)
# 4. Proof of concept for complex apps
# 5. Security baseline (SCPs, guardrails)
# 6. Migration factory setup

# ── MIGRATE AND MODERNISE ──────────────────────────────────────────
# Wave 1: Dev/test servers (low risk, practice cutover)
# Wave 2: Internal apps (medium risk)
# Wave 3: Production apps (careful planning)
# Wave 4: Core business systems (highest risk)

# Migration Wave Checklist:
# Pre-migration:
aws datasync start-task-execution --task-arn <task-arn>   # start replication
aws mgn start-test --source-server-ids <server-id>        # test cutover
# Validate in test environment
# Get stakeholder sign-off

# Cutover:
# 1. Notify users (maintenance window)
# 2. Stop writes to source
aws mgn start-cutover --source-server-ids <server-id>     # final cutover
# 3. Update DNS records
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123 \
  --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{"Name":"myapp.com","Type":"A","TTL":60,"ResourceRecords":[{"Value":"52.1.2.3"}]}}]}'
# 4. Validate app functionality
# 5. Monitor for 24-48 hours

# Post-migration:
aws mgn finalize-cutover --source-server-ids <server-id>  # clean up
# Decommission source servers (after validation period)
# Optimise: Reserved Instances, right-sizing, autoscaling
```

---

### 🟡 Q38. What is AWS Direct Connect?
```bash
# Direct Connect: dedicated private network connection from on-prem to AWS
# Speeds: 1 Gbps, 10 Gbps, 100 Gbps | Through partners: 50 Mbps – 10 Gbps
# Benefits: consistent latency, lower cost for large data transfer, private

# Create connection (at DX location)
aws directconnect create-connection \
  --bandwidth 10Gbps \
  --connection-name my-dx-connection \
  --location EqNY5 \             # DX location
  --provider-name Equinix \
  --request-mac-sec false

# Share connection with Hosted Connection (from your hardware)
aws directconnect describe-locations --query "locations[?locationCode=='EqNY5']"

# Create Virtual Interface (VIF) after connection is provisioned
# Private VIF: access private VPC resources
aws directconnect create-private-virtual-interface \
  --connection-id dxcon-abc123 \
  --new-private-virtual-interface '{
    "virtualInterfaceName": "my-private-vif",
    "vlan": 100,
    "asn": 65000,
    "authKey": "BGPAuthKey123",
    "amazonAddress": "169.254.0.1/30",
    "customerAddress": "169.254.0.2/30",
    "virtualGatewayId": "vgw-abc123"
  }'

# Public VIF: access AWS public services (S3, DynamoDB) without internet
aws directconnect create-public-virtual-interface \
  --connection-id dxcon-abc123 \
  --new-public-virtual-interface '{
    "virtualInterfaceName": "my-public-vif",
    "vlan": 200,
    "asn": 65000,
    "routeFilterPrefixes": [{"cidr": "203.0.113.0/24"}]
  }'

# Transit VIF: connect to Transit Gateway (single VIF, multiple VPCs)
aws directconnect create-transit-virtual-interface \
  --connection-id dxcon-abc123 \
  --new-transit-virtual-interface '{
    "virtualInterfaceName": "transit-vif",
    "vlan": 300,
    "asn": 65000,
    "mtu": 9001,
    "directConnectGatewayId": "dx-gw-abc123"
  }'

# Direct Connect Gateway (multi-region access from one DX)
aws directconnect create-direct-connect-gateway \
  --direct-connect-gateway-name my-dxgw \
  --amazon-side-asn 64512

aws directconnect create-direct-connect-gateway-association \
  --direct-connect-gateway-id dx-gw-abc123 \
  --gateway-id vgw-def456 \
  --add-allowed-prefixes-to-direct-connect-gateway \
    cidr=10.0.0.0/8 cidr=172.16.0.0/12

# HA: Two connections from two different DX locations
# Two VIFs: active/active BGP ECMP or active/passive (higher AS-path prepend)
# DX + VPN backup: failover to VPN if DX fails
```

---

### 🟡 Q39. What is AWS Site-to-Site VPN?
```bash
# Site-to-Site VPN: IPsec encrypted tunnel over internet between on-prem and VPC
# Redundant: 2 tunnels per connection (different endpoints for HA)

# Create Customer Gateway (your VPN device)
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.1 \     # your on-prem router public IP
  --bgp-asn 65000 \
  --device-name "Cisco-ASR1001"

# Create Virtual Private Gateway (attach to VPC)
aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --amazon-side-asn 64512

aws ec2 attach-vpn-gateway \
  --vpc-id vpc-12345678 \
  --vpn-gateway-id vgw-abc123

# Create VPN connection
aws ec2 create-vpn-connection \
  --customer-gateway-id cgw-abc123 \
  --vpn-gateway-id vgw-def456 \
  --type ipsec.1 \
  --options '{
    "StaticRoutesOnly": false,
    "EnableAcceleration": true,
    "LocalIpv4NetworkCidr": "0.0.0.0/0",
    "RemoteIpv4NetworkCidr": "0.0.0.0/0",
    "TunnelOptions": [
      {
        "TunnelInsideCidr": "169.254.0.0/30",
        "PreSharedKey": "MyVPNKey123!",
        "Phase1EncryptionAlgorithms": [{"Value": "AES256"}],
        "Phase2EncryptionAlgorithms": [{"Value": "AES256"}],
        "Phase1IntegrityAlgorithms": [{"Value": "SHA2-256"}],
        "DPDTimeoutAction": "restart",
        "IKEVersions": [{"Value": "ikev2"}]
      },
      {
        "TunnelInsideCidr": "169.254.1.0/30",
        "PreSharedKey": "MyVPNKey123!",
        "IKEVersions": [{"Value": "ikev2"}]
      }
    ]
  }'

# Download VPN configuration for your device
aws ec2 get-vpn-connection-device-types --query "VpnConnectionDeviceTypes" --output table
aws ec2 get-vpn-connection-device-sample-configuration \
  --vpn-connection-id vpn-abc123 \
  --vpn-connection-device-type-id <device-type-id> \
  --internet-gateway-id igw-abc123

# Check VPN status
aws ec2 describe-vpn-connections \
  --vpn-connection-ids vpn-abc123 \
  --query "VpnConnections[0].VgwTelemetry[].{Tunnel:OutsideIpAddress,Status:Status,LastChange:LastStatusChange}"
```

---

### 🟢 Q40. What is AWS CloudEndure Disaster Recovery?
```bash
# CloudEndure DR (now: AWS Elastic Disaster Recovery — DRS)
# Continuous replication → fast failover to AWS in case of disaster

# Initialize DRS
aws drs initialize-service

# Install agent on source server
wget -O ./aws-replication-installer-init.py \
  https://aws-elastic-disaster-recovery-us-east-1.s3.amazonaws.com/latest/linux/aws-replication-installer-init.py
sudo python3 aws-replication-installer-init.py \
  --region us-east-1 \
  --aws-access-key-id $KEY \
  --aws-secret-access-key $SECRET

# List source servers
aws drs describe-source-servers --output table

# Configure launch settings
aws drs update-launch-configuration \
  --source-server-id s-abc12345 \
  --launch-disposition STARTED \
  --target-instance-type-right-sizing-method BASIC \
  --copy-private-ip false \
  --name "DR-web-server-01"

# Start drill (non-destructive DR test)
aws drs start-recovery \
  --is-drill true \
  --source-servers '[{"sourceServerID": "s-abc12345", "recoverySnapshotID": "latest"}]'

# Check recovery status
aws drs describe-recovery-instances --output table

# Stop drill (terminate drill instances)
aws drs terminate-recovery-instances \
  --recovery-instance-ids ri-abc12345

# Real failover (during actual disaster)
aws drs start-recovery \
  --is-drill false \
  --source-servers '[{"sourceServerID": "s-abc12345"}]'

# Failback to source
aws drs create-source-server-action \
  --source-server-id s-abc12345 \
  --action-code FAILBACK

# RPO: seconds (continuous block-level replication)
# RTO: minutes (pre-staged staging area, fast launch)
```


---

# PART 3 — AWS ANALYTICS

---

### 🟢 Q41. What is Amazon Athena?
```bash
# Athena: serverless SQL query engine — query S3 data with standard SQL
# No infrastructure, pay per query (TB scanned), integrates with Glue Data Catalog

# Create database and table (Glue Data Catalog)
aws glue create-database \
  --database-input Name=mydb,Description="Analytics database"

aws glue create-table \
  --database-name mydb \
  --table-input '{
    "Name": "orders",
    "Description": "Customer orders",
    "StorageDescriptor": {
      "Location": "s3://my-data-lake/orders/",
      "InputFormat": "org.apache.hadoop.mapred.TextInputFormat",
      "OutputFormat": "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat",
      "SerdeInfo": {
        "SerializationLibrary": "org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe",
        "Parameters": {"field.delim": ",", "skip.header.line.count": "1"}
      },
      "Columns": [
        {"Name": "order_id",     "Type": "string"},
        {"Name": "customer_id",  "Type": "string"},
        {"Name": "amount",       "Type": "double"},
        {"Name": "order_date",   "Type": "date"},
        {"Name": "status",       "Type": "string"}
      ]
    },
    "PartitionKeys": [
      {"Name": "year",  "Type": "string"},
      {"Name": "month", "Type": "string"}
    ],
    "TableType": "EXTERNAL_TABLE"
  }'

# Create Athena workgroup with result configuration
aws athena create-work-group \
  --name production \
  --configuration '{
    "ResultConfiguration": {
      "OutputLocation": "s3://my-athena-results/production/",
      "EncryptionConfiguration": {
        "EncryptionOption": "SSE_KMS",
        "KmsKey": "arn:aws:kms:us-east-1:123456789:key/abc123"
      }
    },
    "EnforceWorkGroupConfiguration": true,
    "PublishCloudWatchMetricsEnabled": true,
    "BytesScannedCutoffPerQuery": 10737418240,
    "RequesterPaysEnabled": false,
    "EngineVersion": {"SelectedEngineVersion": "Athena engine version 3"}
  }' \
  --tags Key=Environment,Value=production

# Run a query
QUERY_ID=$(aws athena start-query-execution \
  --query-string "
    SELECT
      DATE_TRUNC('month', order_date) AS month,
      COUNT(*) AS order_count,
      SUM(amount) AS revenue,
      AVG(amount) AS avg_order_value,
      APPROX_PERCENTILE(amount, 0.95) AS p95_amount
    FROM mydb.orders
    WHERE year = '2026' AND month = '06'
      AND status = 'DELIVERED'
    GROUP BY 1
    ORDER BY 1 DESC
  " \
  --work-group production \
  --query-execution-context Database=mydb \
  --query "QueryExecutionId" \
  --output text)

# Wait for completion
aws athena get-query-execution \
  --query-execution-id $QUERY_ID \
  --query "QueryExecution.Status"

# Get results
aws athena get-query-results \
  --query-execution-id $QUERY_ID \
  --query "ResultSet.Rows[*].Data[*].VarCharValue"

# Cost optimisation for Athena:
# 1. Use columnar formats (Parquet, ORC) → reads only needed columns
# 2. Partition data (year/month/day) → prune partitions with WHERE clause
# 3. Compress data (Snappy, ZSTD, Gzip) → less data to scan
# 4. Use CTAS to create optimised tables
aws athena start-query-execution \
  --query-string "
    CREATE TABLE orders_parquet
    WITH (
      format = 'PARQUET',
      parquet_compression = 'SNAPPY',
      partitioned_by = ARRAY['year','month'],
      write_compression = 'SNAPPY',
      external_location = 's3://my-data-lake/orders-parquet/'
    ) AS
    SELECT *, YEAR(order_date) AS year, MONTH(order_date) AS month
    FROM mydb.orders
  "
# Result: same data in Parquet vs CSV → 10-100x less scanned → 10-100x cheaper

# Federated queries (query non-S3 sources)
# Athena data source connectors for: DynamoDB, RDS, ElastiCache, CloudWatch, DocumentDB, etc.
aws athena create-data-catalog \
  --name dynamodb-catalog \
  --type LAMBDA \
  --parameters '{
    "function": "arn:aws:lambda:us-east-1:123456789:function:AthenaDynamoDBConnector"
  }'

# Then query DynamoDB like a table:
# SELECT * FROM "dynamodb-catalog"."default"."orders" LIMIT 10;
```

---

### 🟢 Q42. What is Amazon Redshift?
```bash
# Redshift: fully managed, petabyte-scale data warehouse (columnar storage, MPP)
# Architecture: leader node + compute nodes
# Node types: RA3 (managed storage), DC2 (compute-dense, SSD)

# Create Redshift Serverless namespace
aws redshift-serverless create-namespace \
  --namespace-name my-namespace \
  --admin-username admin \
  --admin-user-password "P@ssw0rd2026!" \
  --db-name mydb \
  --default-iam-role-arn arn:aws:iam::123456789:role/redshift-role \
  --iam-roles arn:aws:iam::123456789:role/redshift-role \
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/abc123 \
  --tags Key=Environment,Value=production

aws redshift-serverless create-workgroup \
  --workgroup-name my-workgroup \
  --namespace-name my-namespace \
  --base-capacity 32 \         # RPU (Redshift Processing Units), scales 32-512
  --max-capacity 128 \
  --security-group-ids sg-abc123 \
  --subnet-ids subnet-11111111 subnet-22222222 subnet-33333333 \
  --publicly-accessible false \
  --enhanced-vpc-routing true \
  --config-parameters \
    parameterKey=enable_user_activity_logging,parameterValue=true \
    parameterKey=max_query_queue_size,parameterValue=100

# Create provisioned cluster (for predictable workloads)
aws redshift create-cluster \
  --cluster-identifier my-redshift-cluster \
  --cluster-type multi-node \
  --node-type ra3.4xlarge \
  --number-of-nodes 4 \
  --master-username admin \
  --master-user-password "P@ssw0rd2026!" \
  --cluster-parameter-group-name my-param-group \
  --db-name mydb \
  --vpc-security-group-ids sg-abc123 \
  --cluster-subnet-group-name my-subnet-group \
  --availability-zone-relocation \
  --encrypted --kms-key-id arn:aws:kms:... \
  --enable-logging --bucket-name my-redshift-logs \
  --maintenance-track-name current \
  --iam-roles arn:aws:iam::123456789:role/redshift-role \
  --tags Key=Environment,Value=production

# SQL examples
```

```sql
-- Redshift SQL key features

-- 1. COPY from S3 (bulk load — fastest way to load data)
COPY orders
FROM 's3://my-data-lake/orders/'
IAM_ROLE 'arn:aws:iam::123456789:role/redshift-role'
FORMAT AS PARQUET
MANIFEST                          -- use manifest file for specific files
MAXERROR 100;

-- 2. Distribution styles (affects performance)
CREATE TABLE fact_orders (
    order_id    BIGINT      ENCODE az64,
    customer_id INT         ENCODE az64,
    product_id  INT         ENCODE az64,
    amount      DECIMAL(18,2),
    order_date  DATE        ENCODE az64
)
DISTSTYLE KEY
DISTKEY (customer_id)             -- distribute rows by customer_id (join key)
SORTKEY (order_date);             -- sort for range queries on dates

-- Distribution styles:
-- EVEN:    round-robin (default, no join optimization)
-- KEY:     same key → same node (fast joins, skew risk)
-- ALL:     copy to every node (small dimension tables)
-- AUTO:    Redshift decides (recommended for new tables)

-- 3. Redshift Spectrum (query S3 directly from Redshift)
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'mydb'
IAM_ROLE 'arn:aws:iam::123456789:role/redshift-role'
REGION 'us-east-1';

SELECT o.customer_id, SUM(e.amount) AS external_amount
FROM fact_orders o
JOIN spectrum_schema.external_orders e ON o.order_id = e.order_id
GROUP BY 1;

-- 4. Materialized views (auto-refresh)
CREATE MATERIALIZED VIEW daily_revenue
AUTO REFRESH YES
AS
SELECT
    order_date,
    SUM(amount) AS revenue,
    COUNT(*) AS order_count
FROM fact_orders
GROUP BY order_date;

REFRESH MATERIALIZED VIEW daily_revenue;

-- 5. Data sharing (share data across Redshift clusters, no copy)
-- Producer cluster:
CREATE DATASHARE my_datashare;
ALTER DATASHARE my_datashare ADD TABLE fact_orders;
GRANT USAGE ON DATASHARE my_datashare TO ACCOUNT '987654321';

-- Consumer cluster:
CREATE DATABASE shared_db FROM DATASHARE my_datashare
OF ACCOUNT '123456789' NAMESPACE 'producer-namespace-id';
SELECT * FROM shared_db.public.fact_orders LIMIT 100;
```

---

### 🟡 Q43. What is AWS Glue?
```bash
# Glue: serverless ETL — discover, transform, and load data
# Components: Data Catalog, ETL jobs, Crawlers, DataBrew, Workflows

# Create crawler (discover schema of S3 data)
aws glue create-crawler \
  --name orders-crawler \
  --role arn:aws:iam::123456789:role/glue-role \
  --database-name mydb \
  --targets '{
    "S3Targets": [
      {"Path": "s3://my-data-lake/orders/"},
      {"Path": "s3://my-data-lake/customers/"}
    ]
  }' \
  --schedule 'cron(0 6 * * ? *)' \
  --schema-change-policy '{
    "UpdateBehavior": "UPDATE_IN_DATABASE",
    "DeleteBehavior": "DEPRECATE_IN_DATABASE"
  }' \
  --recrawl-policy '{"RecrawlBehavior": "CRAWL_NEW_FOLDERS_ONLY"}' \
  --configuration '{
    "Version": 1.0,
    "CrawlerOutput": {
      "Partitions": {"AddOrUpdateBehavior": "InheritFromTable"},
      "Tables": {"AddOrUpdateBehavior": "MergeNewColumns"}
    }
  }'

# Start crawler
aws glue start-crawler --name orders-crawler

# Create ETL job (PySpark)
aws glue create-job \
  --name orders-etl-job \
  --role arn:aws:iam::123456789:role/glue-role \
  --command '{
    "Name": "glueetl",
    "ScriptLocation": "s3://my-scripts-bucket/etl/orders_etl.py",
    "PythonVersion": "3"
  }' \
  --default-arguments '{
    "--job-bookmark-option": "job-bookmark-enable",
    "--enable-glue-datacatalog": "",
    "--enable-continuous-cloudwatch-log": "true",
    "--enable-metrics": "",
    "--TempDir": "s3://my-temp-bucket/glue-temp/",
    "--source_bucket": "my-data-lake",
    "--target_bucket": "my-curated-lake",
    "--database": "mydb"
  }' \
  --glue-version "4.0" \
  --worker-type G.2X \
  --number-of-workers 10 \
  --timeout 120 \
  --max-retries 2 \
  --connections "my-rds-connection" \
  --tags Key=Environment,Value=production
```

```python
# orders_etl.py — Glue PySpark ETL script
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.sql.functions import col, to_date, year, month, sum as spark_sum, when

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'source_bucket', 'target_bucket', 'database'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Read from Data Catalog (S3 source)
orders_dyf = glueContext.create_dynamic_frame.from_catalog(
    database=args['database'],
    table_name="raw_orders",
    transformation_ctx="read_orders",
    additional_options={"useS3ListImplementation": True}
)

# Convert to Spark DataFrame for complex transforms
orders_df = orders_dyf.toDF()

# Transform
cleaned_df = orders_df \
    .filter(col("amount") > 0) \
    .filter(col("order_id").isNotNull()) \
    .dropDuplicates(["order_id"]) \
    .withColumn("order_date", to_date(col("order_date"), "yyyy-MM-dd")) \
    .withColumn("year",  year(col("order_date")).cast("string")) \
    .withColumn("month", month(col("order_date")).cast("string")) \
    .withColumn("amount_category",
        when(col("amount") < 50, "small")
        .when(col("amount") < 200, "medium")
        .otherwise("large")
    )

# Aggregate
daily_revenue_df = cleaned_df.groupBy("year", "month", "order_date") \
    .agg(
        spark_sum("amount").alias("total_revenue"),
        spark_sum(when(col("status") == "DELIVERED", col("amount"))).alias("delivered_revenue")
    )

# Write to S3 as Parquet with partitioning
output_dyf = DynamicFrame.fromDF(cleaned_df, glueContext, "output")
glueContext.write_dynamic_frame.from_options(
    frame=output_dyf,
    connection_type="s3",
    format="glueparquet",
    connection_options={
        "path": f"s3://{args['target_bucket']}/orders-curated/",
        "partitionKeys": ["year", "month"],
        "compression": "snappy"
    },
    format_options={"useGlueParquetWriter": True},
    transformation_ctx="write_orders"
)

job.commit()
```

```bash
# Glue DataBrew (no-code data preparation)
aws databrew create-dataset \
  --name orders-dataset \
  --input '{
    "S3InputDefinition": {
      "Bucket": "my-data-lake",
      "Key": "orders/2026/"
    }
  }' \
  --format CSV \
  --format-options '{"Csv": {"Delimiter": ",", "HeaderRow": true}}'

aws databrew create-recipe-job \
  --name clean-orders-job \
  --dataset-name orders-dataset \
  --recipe-reference '{"Name": "clean-orders-recipe", "RecipeVersion": "LATEST_WORKING"}' \
  --outputs '[{
    "Format": "PARQUET",
    "Location": {"Bucket": "my-curated-lake", "Key": "orders-clean/"},
    "CompressionFormat": "SNAPPY",
    "PartitionColumns": ["year", "month"]
  }]' \
  --role-arn arn:aws:iam::123456789:role/databrew-role

# Glue Workflows (orchestrate multiple jobs)
aws glue create-workflow \
  --name daily-etl-workflow \
  --description "Daily ETL pipeline" \
  --default-run-properties '{"timeout": "240"}'

aws glue create-trigger \
  --name daily-trigger \
  --workflow-name daily-etl-workflow \
  --type SCHEDULED \
  --schedule 'cron(0 6 * * ? *)' \
  --actions '[{"JobName": "orders-etl-job"}]'

aws glue create-trigger \
  --name on-etl-complete \
  --workflow-name daily-etl-workflow \
  --type CONDITIONAL \
  --predicate '{"Conditions":[{"JobName":"orders-etl-job","State":"SUCCEEDED","LogicalOperator":"EQUALS"}]}' \
  --actions '[{"CrawlerName": "orders-crawler"}]'
```

---

### 🟡 Q44. What is Amazon EMR?
```bash
# EMR: managed big data clusters — Spark, Hive, HBase, Presto, Flink, Hudi

# Create EMR Serverless application (no cluster management)
aws emr-serverless create-application \
  --name my-spark-app \
  --release-label emr-7.0.0 \
  --type SPARK \
  --initial-capacity '{
    "DRIVER": {"workerCount": 5, "workerConfiguration": {"cpu": "4vCPU", "memory": "16GB", "disk": "200GB"}},
    "EXECUTOR": {"workerCount": 10, "workerConfiguration": {"cpu": "4vCPU", "memory": "16GB", "disk": "200GB"}}
  }' \
  --maximum-capacity '{"cpu": "200vCPU", "memory": "1000GB", "disk": "2000GB"}' \
  --auto-stop-configuration '{"enabled": true, "idleTimeoutMinutes": 15}'

# Run Spark job on Serverless
aws emr-serverless start-job-run \
  --application-id app-abc123 \
  --execution-role-arn arn:aws:iam::123456789:role/emr-serverless-role \
  --job-driver '{
    "sparkSubmit": {
      "entryPoint": "s3://my-scripts-bucket/spark/process_orders.py",
      "entryPointArguments": ["s3://my-data-lake/orders/", "s3://my-curated-lake/orders/"],
      "sparkSubmitParameters": "--conf spark.executor.cores=4 --conf spark.executor.memory=16g --conf spark.executor.instances=20 --conf spark.sql.shuffle.partitions=200"
    }
  }' \
  --configuration-overrides '{
    "monitoringConfiguration": {
      "cloudWatchLoggingConfiguration": {"enabled": true, "logGroupName": "/emr-serverless/spark"},
      "s3MonitoringConfiguration": {"logUri": "s3://my-emr-logs/serverless/"}
    }
  }'

# Create provisioned EMR cluster
aws emr create-cluster \
  --name "prod-analytics-cluster" \
  --release-label emr-7.0.0 \
  --applications Name=Spark Name=Hive Name=Presto Name=JupyterEnterpriseGateway \
  --instance-type m6g.2xlarge \
  --instance-count 6 \
  --use-default-roles \
  --ec2-attributes '{
    "KeyName": "my-key-pair",
    "SubnetIds": ["subnet-11111111"],
    "EmrManagedMasterSecurityGroup": "sg-master",
    "EmrManagedSlaveSecurityGroup": "sg-slave"
  }' \
  --instance-fleet '{
    "InstanceFleetType": "TASK",
    "TargetOnDemandCapacity": 2,
    "TargetSpotCapacity": 8,
    "LaunchSpecifications": {
      "SpotSpecification": {
        "TimeoutDurationMinutes": 10,
        "TimeoutAction": "SWITCH_TO_ON_DEMAND"
      }
    },
    "InstanceTypeConfigs": [
      {"InstanceType": "m6g.2xlarge", "WeightedCapacity": 2},
      {"InstanceType": "m6g.4xlarge", "WeightedCapacity": 4},
      {"InstanceType": "r6g.2xlarge", "WeightedCapacity": 2}
    ]
  }' \
  --configurations '[
    {"Classification": "spark-defaults", "Properties": {
      "spark.dynamicAllocation.enabled": "true",
      "spark.shuffle.service.enabled": "true",
      "spark.sql.adaptive.enabled": "true",
      "spark.sql.adaptive.coalescePartitions.enabled": "true"
    }},
    {"Classification": "spark-hive-site", "Properties": {
      "hive.metastore.client.factory.class": "com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory"
    }}
  ]' \
  --log-uri s3://my-emr-logs/provisioned/ \
  --auto-termination-policy '{"IdleTimeout": 3600}' \
  --auto-scaling-role arn:aws:iam::123456789:role/EMR_AutoScaling_DefaultRole \
  --tags Key=Environment,Value=production \
  --managed-scaling-policy '{"ComputeLimits":{"UnitType":"Instances","MinimumCapacityUnits":3,"MaximumCapacityUnits":20,"MaximumCoreCapacityUnits":10}}'

# Submit Spark step to running cluster
aws emr add-steps \
  --cluster-id j-abc123 \
  --steps '[{
    "Name": "Process Orders",
    "ActionOnFailure": "CONTINUE",
    "HadoopJarStep": {
      "Jar": "command-runner.jar",
      "Args": [
        "spark-submit",
        "--deploy-mode", "cluster",
        "--conf", "spark.executor.memory=8g",
        "s3://my-scripts/process_orders.py",
        "--input", "s3://my-data-lake/orders/",
        "--output", "s3://my-curated-lake/orders/"
      ]
    }
  }]'

# EMR on EKS
aws emr-containers create-virtual-cluster \
  --name my-emr-eks \
  --container-provider '{
    "id": "my-eks-cluster",
    "type": "EKS",
    "info": {"eksInfo": {"namespace": "emr-jobs"}}
  }'

aws emr-containers start-job-run \
  --virtual-cluster-id vc-abc123 \
  --execution-role-arn arn:aws:iam::123456789:role/emr-eks-role \
  --release-label emr-7.0.0-latest \
  --job-driver '{
    "sparkSubmitJobDriver": {
      "entryPoint": "s3://my-scripts/process.py",
      "sparkSubmitParameters": "--conf spark.executor.instances=10 --conf spark.executor.memory=8G"
    }
  }'
```

---

### 🟡 Q45. What is Amazon Kinesis?
```bash
# Kinesis: real-time streaming data platform
# Services:
# Data Streams: custom real-time processing (Flink, Lambda, KCL)
# Data Firehose: zero-management delivery to S3, Redshift, OpenSearch
# Analytics (SQL): real-time SQL on streaming data
# Video Streams: ingest and process video

# ── Kinesis Data Streams ──────────────────────────────────────────
aws kinesis create-stream \
  --stream-name my-event-stream \
  --shard-count 10 \           # 1 shard = 1 MB/s in, 2 MB/s out, 1000 rec/s
  --stream-mode-details StreamMode=PROVISIONED

# Or On-Demand (auto-scales)
aws kinesis create-stream \
  --stream-name my-auto-stream \
  --stream-mode-details StreamMode=ON_DEMAND

# Put records
aws kinesis put-record \
  --stream-name my-event-stream \
  --partition-key "customer-123" \
  --data '{"event": "order_placed", "orderId": "ORD-001", "amount": 99.99}'

# Batch put records (up to 500 records, 5 MB)
aws kinesis put-records \
  --stream-name my-event-stream \
  --records '[
    {"Data": "{\"orderId\":\"1\",\"amount\":50}", "PartitionKey": "c001"},
    {"Data": "{\"orderId\":\"2\",\"amount\":75}", "PartitionKey": "c002"}
  ]'

# Python producer with KPL (Kinesis Producer Library) patterns
import boto3, json, time

kinesis = boto3.client("kinesis", region_name="us-east-1")

def publish_events(events: list[dict], stream_name: str):
    records = [
        {
            "Data": json.dumps(event).encode(),
            "PartitionKey": event.get("customerId", "default")
        }
        for event in events
    ]
    # Batch in groups of 500
    for i in range(0, len(records), 500):
        batch = records[i:i+500]
        response = kinesis.put_records(StreamName=stream_name, Records=batch)
        if response["FailedRecordCount"] > 0:
            # Retry failed records with backoff
            failed = [batch[j] for j, r in enumerate(response["Records"]) if "ErrorCode" in r]
            time.sleep(1)
            kinesis.put_records(StreamName=stream_name, Records=failed)

# ── Kinesis Data Firehose ─────────────────────────────────────────
aws firehose create-delivery-stream \
  --delivery-stream-name my-firehose \
  --delivery-stream-type DirectPut \
  --extended-s3-destination-configuration '{
    "RoleARN": "arn:aws:iam::123456789:role/firehose-role",
    "BucketARN": "arn:aws:s3:::my-data-lake",
    "Prefix": "data/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
    "ErrorOutputPrefix": "errors/",
    "BufferingHints": {"SizeInMBs": 128, "IntervalInSeconds": 300},
    "CompressionFormat": "SNAPPY",
    "DataFormatConversionConfiguration": {
      "Enabled": true,
      "InputFormatConfiguration": {
        "Deserializer": {"OpenXJsonSerDe": {}}
      },
      "OutputFormatConfiguration": {
        "Serializer": {"ParquetSerDe": {"Compression": "SNAPPY"}}
      },
      "SchemaConfiguration": {
        "RoleARN": "arn:aws:iam::123456789:role/firehose-role",
        "DatabaseName": "mydb",
        "TableName": "events",
        "Region": "us-east-1"
      }
    },
    "ProcessingConfiguration": {
      "Enabled": true,
      "Processors": [{
        "Type": "Lambda",
        "Parameters": [{
          "ParameterName": "LambdaArn",
          "ParameterValue": "arn:aws:lambda:us-east-1:123456789:function:enrich-events"
        }]
      }]
    },
    "S3BackupMode": "FailedDataOnly",
    "CloudWatchLoggingOptions": {"Enabled": true, "LogGroupName": "/firehose/my-firehose"}
  }'

# Firehose with Kinesis Data Streams source
aws firehose create-delivery-stream \
  --delivery-stream-name stream-to-s3 \
  --delivery-stream-type KinesisStreamAsSource \
  --kinesis-stream-source-configuration '{
    "KinesisStreamARN": "arn:aws:kinesis:us-east-1:123456789:stream/my-event-stream",
    "RoleARN": "arn:aws:iam::123456789:role/firehose-role"
  }' \
  --extended-s3-destination-configuration '{...}'

# ── Kinesis Data Analytics (Managed Flink) ────────────────────────
aws kinesisanalyticsv2 create-application \
  --application-name my-flink-app \
  --runtime-environment FLINK-1_18 \
  --service-execution-role arn:aws:iam::123456789:role/flink-role \
  --application-configuration '{
    "FlinkApplicationConfiguration": {
      "CheckpointConfiguration": {"ConfigurationType": "CUSTOM", "CheckpointingEnabled": true, "CheckpointInterval": 60000},
      "MonitoringConfiguration": {"ConfigurationType": "CUSTOM", "MetricsLevel": "TASK", "LogLevel": "INFO"},
      "ParallelismConfiguration": {
        "ConfigurationType": "CUSTOM",
        "Parallelism": 8,
        "ParallelismPerKPU": 1,
        "AutoScalingEnabled": true
      }
    },
    "ApplicationCodeConfiguration": {
      "CodeContent": {
        "S3ContentLocation": {
          "BucketARN": "arn:aws:s3:::my-scripts-bucket",
          "FileKey": "flink/my-flink-app.jar"
        }
      },
      "CodeContentType": "ZIPFILE"
    }
  }'
```

---

### 🟡 Q46. What is Amazon OpenSearch Service?
```bash
# OpenSearch Service: managed Elasticsearch-compatible search and analytics
# Use cases: full-text search, log analytics, security analytics, metrics

# Create domain
aws opensearch create-domain \
  --domain-name my-search-domain \
  --engine-version OpenSearch_2.13 \
  --cluster-config '{
    "InstanceType": "m6g.large.search",
    "InstanceCount": 3,
    "DedicatedMasterEnabled": true,
    "DedicatedMasterType": "m6g.large.search",
    "DedicatedMasterCount": 3,
    "ZoneAwarenessEnabled": true,
    "ZoneAwarenessConfig": {"AvailabilityZoneCount": 3},
    "WarmEnabled": true,
    "WarmType": "ultrawarm1.medium.search",
    "WarmCount": 2,
    "MultiAZWithStandbyEnabled": true
  }' \
  --ebs-options '{
    "EBSEnabled": true,
    "VolumeType": "gp3",
    "VolumeSize": 512,
    "Iops": 3000,
    "Throughput": 250
  }' \
  --vpc-options '{
    "SubnetIds": ["subnet-11111111","subnet-22222222","subnet-33333333"],
    "SecurityGroupIds": ["sg-opensearch"]
  }' \
  --encrypt-at-rest-options Enabled=true,KmsKeyId=arn:aws:kms:... \
  --node-to-node-encryption-options Enabled=true \
  --domain-endpoint-options '{
    "EnforceHTTPS": true,
    "TLSSecurityPolicy": "Policy-Min-TLS-1-2-2019-07",
    "CustomEndpointEnabled": true,
    "CustomEndpoint": "search.mycompany.com",
    "CustomEndpointCertificateArn": "arn:aws:acm:..."
  }' \
  --advanced-security-options '{
    "Enabled": true,
    "InternalUserDatabaseEnabled": true,
    "MasterUserOptions": {
      "MasterUserName": "admin",
      "MasterUserPassword": "P@ssw0rd2026!"
    }
  }' \
  --cognito-options '{
    "Enabled": true,
    "UserPoolId": "<cognito-user-pool-id>",
    "IdentityPoolId": "<cognito-identity-pool-id>",
    "RoleArn": "arn:aws:iam::123456789:role/opensearch-cognito-role"
  }' \
  --auto-tune-options '{"DesiredState": "ENABLED"}' \
  --off-peak-window-options '{
    "Enabled": true,
    "OffPeakWindow": {"WindowStartTime": {"Hours": 2, "Minutes": 0}}
  }' \
  --tags Key=Environment,Value=production

# OpenSearch Serverless
aws opensearchserverless create-collection \
  --name my-search-collection \
  --type SEARCH \
  --description "Product search index" \
  --tags Key=Environment,Value=production

# Create security policies
aws opensearchserverless create-encryption-policy \
  --name my-collection-encryption \
  --type encryption \
  --policy '{"Rules":[{"ResourceType":"collection","Resource":["collection/my-search-collection"]}],"AWSOwnedKey":true}'

aws opensearchserverless create-network-policy \
  --name my-collection-network \
  --type network \
  --policy '[{"Rules":[{"ResourceType":"collection","Resource":["collection/my-search-collection"]},{"ResourceType":"dashboard","Resource":["collection/my-search-collection"]}],"AllowFromPublic":false,"SourceVPCEs":["vpce-abc123"]}]'

aws opensearchserverless create-access-policy \
  --name my-collection-access \
  --type data \
  --policy '[{"Rules":[{"ResourceType":"index","Resource":["index/my-search-collection/*"],"Permission":["aoss:CreateIndex","aoss:WriteDocument","aoss:ReadDocument","aoss:SearchDocument","aoss:DeleteIndex"]},{"ResourceType":"collection","Resource":["collection/my-search-collection"],"Permission":["aoss:CreateCollectionItems"]}],"Principal":["arn:aws:iam::123456789:role/app-role"]}]'

# Python: index and search with OpenSearch client
from opensearchpy import OpenSearch, RequestsHttpConnection
from aws_requests_auth.boto_utils import BotoAWSRequestsAuth
import boto3

auth = BotoAWSRequestsAuth(
    aws_host="my-search-domain.us-east-1.es.amazonaws.com",
    aws_region="us-east-1",
    aws_service="es"
)
client = OpenSearch(
    hosts=[{"host": "my-search-domain.us-east-1.es.amazonaws.com", "port": 443}],
    http_auth=auth,
    use_ssl=True,
    verify_certs=True,
    connection_class=RequestsHttpConnection
)

# Create index with mapping
client.indices.create(index="products", body={
    "settings": {
        "number_of_shards": 3,
        "number_of_replicas": 1,
        "analysis": {
            "analyzer": {
                "product_analyzer": {
                    "type": "custom",
                    "tokenizer": "standard",
                    "filter": ["lowercase", "stop", "snowball"]
                }
            }
        }
    },
    "mappings": {
        "properties": {
            "productId":    {"type": "keyword"},
            "name":         {"type": "text", "analyzer": "product_analyzer"},
            "description":  {"type": "text", "analyzer": "product_analyzer"},
            "category":     {"type": "keyword"},
            "price":        {"type": "float"},
            "rating":       {"type": "float"},
            "inStock":      {"type": "boolean"},
            "tags":         {"type": "keyword"},
            "embedding":    {"type": "knn_vector", "dimension": 1536}
        }
    }
})

# Hybrid search (keyword + vector)
result = client.search(index="products", body={
    "query": {
        "bool": {
            "should": [
                {
                    "multi_match": {
                        "query": "wireless bluetooth headphones",
                        "fields": ["name^3", "description", "tags"],
                        "type": "best_fields"
                    }
                },
                {
                    "knn": {
                        "embedding": {
                            "vector": [0.1, 0.2, ...],  # query embedding
                            "k": 20
                        }
                    }
                }
            ]
        }
    },
    "aggs": {
        "by_category": {"terms": {"field": "category", "size": 10}},
        "price_range": {"histogram": {"field": "price", "interval": 50}},
        "avg_rating": {"avg": {"field": "rating"}}
    },
    "sort": [
        {"_score": {"order": "desc"}},
        {"rating": {"order": "desc"}}
    ],
    "size": 20,
    "from": 0
})
```

---

### 🟡 Q47. What is Amazon QuickSight?
```bash
# QuickSight: cloud-native BI and dashboards with ML insights
# Features: SPICE (in-memory engine), ML insights, embedded analytics

# Create QuickSight dataset from S3 (via Athena manifest)
aws quicksight create-data-set \
  --aws-account-id 123456789 \
  --data-set-id orders-dataset \
  --name "Orders Dataset" \
  --import-mode SPICE \
  --physical-table-map '{
    "orders-table": {
      "RelationalTable": {
        "DataSourceArn": "arn:aws:quicksight:us-east-1:123456789:datasource/athena-datasource",
        "Schema": "mydb",
        "Name": "orders",
        "InputColumns": [
          {"Name": "order_id", "Type": "STRING"},
          {"Name": "customer_id", "Type": "STRING"},
          {"Name": "amount", "Type": "DECIMAL"},
          {"Name": "order_date", "Type": "DATETIME"},
          {"Name": "status", "Type": "STRING"}
        ]
      }
    }
  }' \
  --logical-table-map '{
    "orders-logical": {
      "Alias": "Orders",
      "Source": {"PhysicalTableId": "orders-table"},
      "DataTransforms": [
        {
          "CreateColumnsOperation": {
            "Columns": [{
              "ColumnId": "revenue-column",
              "ColumnName": "Revenue",
              "Expression": "{amount}"
            }]
          }
        },
        {
          "FilterOperation": {
            "ConditionExpression": "{amount} > 0"
          }
        }
      ]
    }
  }' \
  --permissions '[{
    "Principal": "arn:aws:quicksight:us-east-1:123456789:group/default/analysts",
    "Actions": ["quicksight:DescribeDataSet","quicksight:DescribeDataSetPermissions","quicksight:PassDataSet","quicksight:DescribeIngestion","quicksight:ListIngestions"]
  }]' \
  --row-level-permission-data-set '{
    "Arn": "arn:aws:quicksight:us-east-1:123456789:dataset/rls-dataset",
    "PermissionPolicy": "GRANT_ACCESS"
  }'

# Refresh SPICE dataset
aws quicksight create-ingestion \
  --aws-account-id 123456789 \
  --data-set-id orders-dataset \
  --ingestion-id refresh-$(date +%Y%m%d%H%M%S)

# Create analysis
aws quicksight create-analysis \
  --aws-account-id 123456789 \
  --analysis-id orders-analysis \
  --name "Orders Analysis" \
  --source-entity '{
    "SourceTemplate": {
      "DataSetReferences": [{
        "DataSetPlaceholder": "orders",
        "DataSetArn": "arn:aws:quicksight:us-east-1:123456789:dataset/orders-dataset"
      }],
      "Arn": "arn:aws:quicksight:us-east-1:123456789:template/orders-template"
    }
  }'

# Embedded analytics
aws quicksight generate-embed-url-for-anonymous-user \
  --aws-account-id 123456789 \
  --namespace default \
  --session-lifetime-in-minutes 60 \
  --authorized-resource-arns arn:aws:quicksight:us-east-1:123456789:dashboard/orders-dashboard \
  --experience-configuration '{
    "Dashboard": {
      "InitialDashboardId": "orders-dashboard"
    }
  }' \
  --session-tags '[{"Key": "RegionId", "Value": "us-east"}]'

# ML Insights (auto-anomaly detection, narratives)
# Enabled automatically in SPICE datasets
# Dashboard shows: anomaly detection charts, auto-narrative summaries

# QuickSight Q (natural language queries)
# "What was the revenue last month?" → auto-generates visualization
```

---

### 🟡 Q48. What is AWS Lake Formation?
```bash
# Lake Formation: build, manage, and secure a data lake
# Provides: fine-grained access control (column, row, cell level) for Glue Catalog, S3

# Set up Lake Formation
aws lakeformation put-data-lake-settings \
  --data-lake-settings '{
    "DataLakeAdmins": [{"DataLakePrincipalIdentifier": "arn:aws:iam::123456789:role/datalake-admin"}],
    "CreateDatabaseDefaultPermissions": [],
    "CreateTableDefaultPermissions": [],
    "TrustedResourceOwners": ["123456789"],
    "AllowExternalDataFiltering": true,
    "ExternalDataFilteringAllowList": [{"DataLakePrincipalIdentifier": "123456789"}]
  }'

# Register S3 location with Lake Formation
aws lakeformation register-resource \
  --resource-arn arn:aws:s3:::my-data-lake \
  --role-arn arn:aws:iam::123456789:role/lakeformation-s3-role \
  --with-federation false

# Create database (through Glue)
aws glue create-database \
  --database-input Name=finance_db,Description="Finance data lake"

# Grant table-level permissions
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789:role/analyst-role \
  --resource '{"Table": {"DatabaseName": "finance_db", "TableWildcard": {}}}' \
  --permissions SELECT DESCRIBE \
  --permissions-with-grant-option

# Grant column-level permissions (hide sensitive columns)
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789:role/junior-analyst-role \
  --resource '{
    "TableWithColumns": {
      "DatabaseName": "finance_db",
      "Name": "employees",
      "ColumnWildcard": {
        "ExcludedColumnNames": ["salary", "ssn", "bank_account"]
      }
    }
  }' \
  --permissions SELECT

# Row-level permissions (via row filter)
aws lakeformation create-data-cells-filter \
  --table-data '{"TableCatalogId": "123456789", "DatabaseName": "finance_db", "TableName": "orders"}' \
  --name "emea-only-filter" \
  --row-filter '{"FilterExpression": "region = '\''EMEA'\''"}' \
  --column-wildcard

aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789:role/emea-analyst-role \
  --resource '{
    "DataCellsFilter": {
      "TableCatalogId": "123456789",
      "DatabaseName": "finance_db",
      "TableName": "orders",
      "Name": "emea-only-filter"
    }
  }' \
  --permissions SELECT

# Cross-account data sharing (share with another AWS account)
aws lakeformation create-lake-formation-opt-in \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::987654321:root \
  --resource '{"Database": {"CatalogId": "123456789", "Name": "finance_db"}}'
```

---

### 🟡 Q49. What is Amazon MSK (Managed Streaming for Apache Kafka)?
```bash
# MSK: fully managed Apache Kafka — no Kafka cluster management
# Versions: Kafka 3.6, 3.5, 3.4 | MSK Serverless: auto-scales capacity

# Create MSK Serverless cluster
aws kafka create-cluster-v2 \
  --cluster-name my-kafka-serverless \
  --serverless '{
    "vpcConfigs": [{
      "subnetIds": ["subnet-11111111","subnet-22222222"],
      "securityGroupIds": ["sg-kafka"]
    }],
    "clientAuthentication": {
      "sasl": {"iam": {"enabled": true}}
    }
  }'

# Create provisioned MSK cluster
aws kafka create-cluster \
  --cluster-name my-kafka-cluster \
  --kafka-version 3.6.0 \
  --number-of-broker-nodes 6 \
  --broker-node-group-info '{
    "InstanceType": "kafka.m5.large",
    "ClientSubnets": ["subnet-az1","subnet-az2","subnet-az3"],
    "SecurityGroups": ["sg-kafka"],
    "StorageInfo": {
      "EbsStorageInfo": {
        "VolumeSize": 1000,
        "ProvisionedThroughput": {"Enabled": true, "VolumeThroughput": 250}
      }
    }
  }' \
  --encryption-info '{
    "EncryptionInTransit": {"ClientBroker": "TLS", "InCluster": true},
    "EncryptionAtRest": {"DataVolumeKMSKeyId": "arn:aws:kms:..."}
  }' \
  --client-authentication '{
    "Sasl": {"Iam": {"Enabled": true}, "Scram": {"Enabled": false}},
    "Tls": {"Enabled": false},
    "Unauthenticated": {"Enabled": false}
  }' \
  --enhanced-monitoring PER_TOPIC_PER_PARTITION \
  --open-monitoring-info '{
    "Prometheus": {
      "JmxExporter": {"EnabledInBroker": true},
      "NodeExporter": {"EnabledInBroker": true}
    }
  }' \
  --broker-logs '{
    "CloudWatchLogs": {"Enabled": true, "LogGroup": "/msk/my-kafka"},
    "S3": {"Enabled": true, "Bucket": "my-kafka-logs", "Prefix": "kafka-logs/"}
  }'

# Get bootstrap servers
aws kafka get-bootstrap-brokers \
  --cluster-arn arn:aws:kafka:us-east-1:123456789:cluster/my-kafka-cluster/abc123

# Python: produce and consume with IAM auth
from kafka import KafkaProducer, KafkaConsumer
from aws_msk_iam_sasl_signer import MSKAuthTokenProvider
import boto3, json

# Producer with IAM
def get_iam_token(bootstrap_servers, **kwargs):
    token, expiry = MSKAuthTokenProvider.generate_auth_token("us-east-1")
    return token, expiry * 1000

producer = KafkaProducer(
    bootstrap_servers=["b-1.my-kafka.abc123.c1.kafka.us-east-1.amazonaws.com:9098"],
    security_protocol="SASL_SSL",
    sasl_mechanism="OAUTHBEARER",
    sasl_oauth_token_provider=MSKAuthTokenProvider.get_auth_token("us-east-1"),
    value_serializer=lambda v: json.dumps(v).encode()
)

producer.send("orders", {"orderId": "123", "amount": 99.99})
producer.flush()

# MSK Connect (managed Kafka connectors)
aws kafkaconnect create-connector \
  --connector-name "s3-sink-connector" \
  --connector-configuration '{
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "tasks.max": "10",
    "topics": "orders,payments",
    "s3.region": "us-east-1",
    "s3.bucket.name": "my-kafka-sink-bucket",
    "s3.part.size": "5242880",
    "flush.size": "1000",
    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.parquet.ParquetFormat",
    "schema.compatibility": "FULL"
  }' \
  --capacity '{"provisionedCapacity": {"mcuCount": 2, "workerCount": 4}}' \
  --kafka-cluster '{
    "apacheKafkaCluster": {
      "bootstrapServers": "b-1.my-kafka...:9098",
      "vpc": {
        "subnets": ["subnet-11111111"],
        "securityGroups": ["sg-kafka"]
      }
    }
  }' \
  --kafka-connect-version 2.7.1 \
  --plugins '[{"customPlugin": {"customPluginArn": "arn:aws:kafkaconnect:...", "revision": 1}}]' \
  --service-execution-role-arn arn:aws:iam::123456789:role/msk-connect-role
```

---

### 🟡 Q50. What is AWS Data Exchange?
```bash
# Data Exchange: marketplace for third-party datasets
# Providers: 3,500+ data products from 300+ providers
# Categories: financial data, demographic, healthcare, weather, geospatial

# Subscribe to a dataset
aws dataexchange create-subscription \
  --data-set-id <dataset-id> \
  --revision-id <revision-id>

# List subscribed datasets
aws dataexchange list-data-sets \
  --origin SUBSCRIBED \
  --query "DataSets[].{Name:Name,Provider:OriginDetails.ProductId}"

# Export revision assets to S3
aws dataexchange create-job \
  --type EXPORT_ASSETS_TO_S3 \
  --details '{
    "ExportAssetsToS3": {
      "AssetDestinations": [{
        "AssetId": "<asset-id>",
        "Bucket": "my-data-bucket",
        "Key": "third-party-data/provider-name/"
      }],
      "DataSetId": "<dataset-id>",
      "RevisionId": "<revision-id>"
    }
  }'

aws dataexchange start-job --job-id <job-id>

# Auto-export new revisions (EventBridge trigger)
aws events put-rule \
  --name "DataExchangeNewRevision" \
  --event-pattern '{
    "source": ["aws.dataexchange"],
    "detail-type": ["Revision Published To Data Set"],
    "detail": {"DataSetId": ["<dataset-id>"]}
  }'

# Common data categories:
# Financial: stock prices, SEC filings, ESG scores (FactSet, Refinitiv)
# Demographics: census, consumer behaviour (Nielsen, Experian)
# Healthcare: clinical trials, drug safety (FDA, Cortellis)
# Weather: historical, forecast (DTN, Tomorrow.io)
# Geospatial: maps, Points of Interest (HERE, SafeGraph)
# Cybersecurity: threat intelligence (CrowdStrike, Recorded Future)
```

---

## COMPLETE Q&A INDEX

| Q# | Question | Level | Part |
|----|---------|-------|------|
| **SECURITY, IDENTITY & COMPLIANCE (Q1–Q25)** | | | |
| Q1 | Shared Responsibility Model — who owns what | 🟢 | Security |
| Q2 | AWS IAM — users, groups, roles, policies, instance profiles | 🟢 | Security |
| Q3 | IAM condition keys — time, MFA, region, HTTPS, TBAC | 🟡 | Security |
| Q4 | AWS STS — assume role, cross-account, Python boto3 pattern | 🟡 | Security |
| Q5 | Amazon Cognito — User Pools, Identity Pools, hosted UI, federated IdP | 🟢 | Security |
| Q6 | Cognito Lambda triggers — pre-signup, post-confirm, pre-token claims | 🟡 | Security |
| Q7 | AWS WAF — WebACL, managed rules, rate limit, bot control, geo block, SQLi | 🟡 | Security |
| Q8 | AWS Shield — Standard vs Advanced, DRT, cost protection, protection groups | 🟡 | Security |
| Q9 | VPC Security — Security Groups vs NACLs, stateful vs stateless, SG references | 🟡 | Security |
| Q10 | AWS KMS — CMK types, key policy, envelope encryption, rotation, multi-region | 🟡 | Security |
| Q11 | AWS CloudHSM — FIPS 140-2 Level 3, PKCS#11, use cases vs KMS | 🟡 | Security |
| Q12 | Amazon Detective — investigate GuardDuty findings, entity timelines | 🟡 | Security |
| Q13 | AWS Firewall Manager — centralised WAF, Shield, SG, NACL across org | 🟡 | Security |
| Q14 | AWS Network Firewall — Suricata rules, stateless/stateful, deployment | 🟡 | Security |
| Q15 | Route53 DNS Firewall — block malware domains, AWS managed lists | 🟡 | Security |
| Q16 | AWS Certificate Manager — request, DNS validation, import, Private CA, expiry alarm | 🟡 | Security |
| Q17 | Secrets Manager vs SSM Parameter Store — comparison table | 🟢 | Security |
| Q18 | Amazon Macie — PII discovery, classification jobs, sensitive data categories | 🟡 | Security |
| Q19 | AWS Security Lake — OCSF format, log sources, Athena queries | 🟡 | Security |
| Q20 | IAM Access Analyzer — external access, unused access, policy generation | 🟡 | Security |
| Q21 | Amazon Inspector v2 — EC2/ECR/Lambda, continuous scanning, SBOM export | 🟡 | Security |
| Q22 | AWS Audit Manager — SOC 2, PCI DSS, evidence collection, custom controls | 🟡 | Security |
| Q23 | Security Hub vs GuardDuty vs Inspector vs Macie — comparison table | 🟢 | Security |
| Q24 | VPC Endpoints — Gateway (S3/DynamoDB), Interface (PrivateLink), bucket policy enforce | 🟡 | Security |
| Q25 | Resource-based policies — S3 bucket policy, Lambda permission, SQS policy | 🟡 | Security |
| **MIGRATION & TRANSFER (Q26–Q40)** | | | |
| Q26 | AWS Migration Hub — tracking, 7Rs, progress streams | 🟢 | Migration |
| Q27 | AWS Application Migration Service (MGN) — agent install, test, cutover | 🟡 | Migration |
| Q28 | AWS Database Migration Service (DMS) — Full Load+CDC, Oracle→Aurora, task settings | 🟡 | Migration |
| Q29 | AWS Schema Conversion Tool (SCT) — Oracle→PostgreSQL, conversion challenges | 🟡 | Migration |
| Q30 | AWS DataSync — NFS/S3/EFS transfer, agent, task, scheduling, filters | 🟡 | Migration |
| Q31 | AWS Snow Family — Snowcone/Snowball/Snowmobile, EC2 at edge, use cases | 🟡 | Migration |
| Q32 | AWS Transfer Family — SFTP/FTPS/AS2, Lambda auth, workflows, post-upload processing | 🟡 | Migration |
| Q33 | AWS Storage Gateway — S3 File, Volume, Tape, SMB/NFS configuration | 🟡 | Migration |
| Q34 | AWS Application Discovery Service — agent vs agentless, dependency mapping | 🟡 | Migration |
| Q35 | AWS Migration Evaluator — TCO analysis, business case, rightsizing | 🟢 | Migration |
| Q36 | AWS Mainframe Modernisation — COBOL, Micro Focus, Blu Age, VSAM→DynamoDB | 🟡 | Migration |
| Q37 | Migration best practices — Assess→Mobilise→Migrate phases, wave plan, DNS cutover | 🟡 | Migration |
| Q38 | AWS Direct Connect — VIF types (private/public/transit), DX Gateway, HA design | 🟡 | Migration |
| Q39 | AWS Site-to-Site VPN — customer GW, VGW, two tunnels, BGP, IKEv2 config | 🟡 | Migration |
| Q40 | AWS Elastic Disaster Recovery (DRS) — agent, test drill, failover, RPO/RTO | 🟡 | Migration |
| **ANALYTICS (Q41–Q50)** | | | |
| Q41 | Amazon Athena — Glue catalog, workgroups, cost optimisation, Parquet, CTAS, federated | 🟢 | Analytics |
| Q42 | Amazon Redshift — Serverless, distribution styles, Spectrum, data sharing, COPY | 🟡 | Analytics |
| Q43 | AWS Glue — crawlers, PySpark ETL, DataBrew, workflows, triggers, Glue 4.0 | 🟡 | Analytics |
| Q44 | Amazon EMR — Serverless, provisioned, EKS, instance fleets, Spot, Flink | 🟡 | Analytics |
| Q45 | Amazon Kinesis — Data Streams, Firehose (Parquet conversion), Analytics Flink | 🟡 | Analytics |
| Q46 | Amazon OpenSearch — managed domain, Serverless, hybrid search, k-NN vector search | 🟡 | Analytics |
| Q47 | Amazon QuickSight — SPICE, embedded analytics, ML insights, row-level security | 🟡 | Analytics |
| Q48 | AWS Lake Formation — fine-grained access (column/row/cell), cross-account sharing | 🟡 | Analytics |
| Q49 | Amazon MSK — Serverless, provisioned, IAM auth, MSK Connect S3 sink | 🟡 | Analytics |
| Q50 | AWS Data Exchange — subscribe, export to S3, EventBridge new revision trigger | 🟢 | Analytics |

---
*Total: 50 Q&A | Security (25) + Migration (15) + Analytics (10) | June 2026*
*🟢 10 Basic | 🟡 38 Intermediate | 🔴 2 Advanced*
*All examples use AWS CLI v2 + production configurations + Python SDK patterns*

---

# GAP-FILL — SECURITY (Q51–Q70)

---

### 🟡 Q51. What is Amazon GuardDuty in full detail?
```bash
# GuardDuty: intelligent threat detection using ML across multiple data sources
# Analyses: VPC Flow Logs, CloudTrail Mgmt Events, DNS logs, EKS audit logs,
#           S3 data events, RDS login events, Lambda network activity, EC2 runtime

# Enable with all data sources
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --data-sources '{
    "S3Logs":               {"Enable": true},
    "Kubernetes":           {"AuditLogs":  {"Enable": true}},
    "MalwareProtection":    {"ScanEc2InstanceWithFindings": {"EbsVolumes": {"Enable": true}}},
    "RdsLoginEvents":       {"Enable": true},
    "EksRuntimeMonitoring": {"AuditLogs": {"Enable": true}},
    "LambdaNetworkLogs":    {"Enable": true},
    "EksAddonVersion":      {"Enable": true}
  }'

DETECTOR=$(aws guardduty list-detectors --query "DetectorIds[0]" --output text)

# Finding categories:
# Backdoor:    C2 communication, BitCoin mining
# Behavior:    unusual API calls, unusual network traffic
# CryptoCurrency: Bitcoin tools, mining pools
# DefenseEvasion: CloudTrail disabled, GuardDuty disabled
# Discovery:   reconnaissance, port scanning
# Exfiltration: unusual S3 data access, large DNS queries
# Impact:      resource hijacking, destructive actions
# InitialAccess: credential stuffing, unusual login
# Pentest:     penetration testing tools
# Persistence: unusual IAM user activity, backdoor accounts
# PrivilegeEscalation: policy changes, role assumption
# Recon:       brute force, port scanning
# Stealth:     log tampering, trail deletion
# Trojan:      DNSDataExfiltration, DriveBySourceTraffic
# UnauthorizedAccess: credential theft, TorIPCaller

# Key finding examples:
# UnauthorizedAccess:IAMUser/MaliciousIPCaller.Custom — API call from custom threat list IP
# Recon:IAMUser/TorIPCaller                           — API call from Tor exit node
# PrivilegeEscalation:IAMUser/AnomalousBehavior       — unusual privilege escalation
# CryptoCurrency:EC2/BitcoinTool.B!DNS               — EC2 querying Bitcoin-related domains
# Backdoor:EC2/C&CActivity.B!DNS                     — C2 communication detected
# Exfiltration:S3/AnomalousBehavior                  — unusual S3 data retrieval

# Threat intelligence lists (custom IP/domain lists)
aws guardduty create-threat-intel-set \
  --detector-id $DETECTOR \
  --name "known-bad-actors" \
  --format TXT \
  --location s3://my-threat-intel-bucket/bad-ips.txt \
  --activate true

# Suppress noisy findings
aws guardduty create-filter \
  --detector-id $DETECTOR \
  --name "allow-my-scanner" \
  --action ARCHIVE \
  --finding-criteria '{
    "Criterion": {
      "type": {"Eq": ["Recon:EC2/PortProbeUnprotectedPort"]},
      "resource.instanceDetails.tags.key":   {"Eq": ["Role"]},
      "resource.instanceDetails.tags.value": {"Eq": ["scanner"]}
    }
  }'

# EventBridge → Lambda auto-remediation
aws events put-rule \
  --name "GuardDutyHighSeverity" \
  --event-pattern '{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
      "severity": [{"numeric": [">=", 7.0]}]
    }
  }'

# Auto-remediation Lambda example
def auto_remediate(event, context):
    finding = event["detail"]
    ftype = finding["type"]
    resource = finding["resource"]
    severity = finding["severity"]

    if "IAMUser" in ftype:
        user = resource["accessKeyDetails"]["userName"]
        # Disable access key immediately
        iam = boto3.client("iam")
        key_id = resource["accessKeyDetails"]["accessKeyId"]
        iam.update_access_key(UserName=user, AccessKeyId=key_id, Status="Inactive")
        print(f"Disabled access key {key_id} for user {user}")

    elif "EC2" in ftype and "CryptoCurrency" in ftype:
        instance_id = resource["instanceDetails"]["instanceId"]
        # Isolate instance (apply deny-all SG)
        ec2 = boto3.client("ec2")
        ec2.modify_instance_attribute(
            InstanceId=instance_id,
            Groups=["sg-isolate-all"]   # pre-created SG with deny all
        )

# Org-wide GuardDuty
aws guardduty enable-organization-admin-account --admin-account-id 111111111
aws guardduty update-organization-configuration \
  --detector-id $DETECTOR \
  --auto-enable ALL \
  --features '[
    {"name":"S3_DATA_EVENTS","autoEnable":"NEW"},
    {"name":"EKS_AUDIT_LOGS","autoEnable":"ALL"},
    {"name":"MALWARE_PROTECTION","autoEnable":"ALL"}
  ]'
```

---

### 🟡 Q52. What is S3 security — Block Public Access, Object Lock, Bucket Key?
```bash
# ── Block Public Access (BPA) ─────────────────────────────────────
# Four settings — all should be ON in production
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
    BlockPublicAcls=true \          # block new public ACLs
    IgnorePublicAcls=true \         # ignore existing public ACLs
    BlockPublicPolicy=true \        # block bucket policies allowing public access
    RestrictPublicBuckets=true      # restrict access to bucket with public policy

# Account-level BPA (covers ALL buckets in account)
aws s3control put-public-access-block \
  --account-id 123456789 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Check BPA status
aws s3api get-public-access-block --bucket my-bucket

# ── Object Lock (WORM) ───────────────────────────────────────────
# Must enable at bucket creation (cannot add later)
aws s3api create-bucket \
  --bucket my-worm-bucket \
  --object-lock-enabled-for-bucket \
  --create-bucket-configuration LocationConstraint=us-east-1

# Set default retention policy
aws s3api put-object-lock-configuration \
  --bucket my-worm-bucket \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",    # COMPLIANCE (cannot override even by root) | GOVERNANCE (admin can override)
        "Days": 365
      }
    }
  }'

# Retention modes:
# COMPLIANCE: nobody (including root) can delete/modify until retention expires
# GOVERNANCE: root and users with s3:BypassGovernanceRetention can override

# Apply legal hold (indefinite, no expiry date)
aws s3api put-object-legal-hold \
  --bucket my-worm-bucket \
  --key "evidence/case-2026-001.pdf" \
  --legal-hold '{"Status": "ON"}'

# Object-level retention (overrides bucket default)
aws s3api put-object-retention \
  --bucket my-worm-bucket \
  --key "reports/q2-2026.pdf" \
  --retention '{"Mode": "COMPLIANCE", "RetainUntilDate": "2031-06-30T00:00:00.000Z"}'

# ── S3 Bucket Key (reduce KMS costs) ─────────────────────────────
# Bucket Key: S3 generates short-lived data key per bucket → fewer KMS API calls
# Reduces KMS cost by up to 99%
aws s3api put-bucket-encryption \
  --bucket my-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789:key/abc123"
      },
      "BucketKeyEnabled": true
    }]
  }'

# ── S3 Versioning ─────────────────────────────────────────────────
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled

# ── S3 Replication (for DR and compliance) ────────────────────────
aws s3api put-bucket-replication \
  --bucket my-source-bucket \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789:role/s3-replication-role",
    "Rules": [{
      "ID": "replicate-all",
      "Status": "Enabled",
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::my-dr-bucket",
        "StorageClass": "STANDARD_IA",
        "ReplicationTime": {"Status": "Enabled", "Time": {"Minutes": 15}},
        "Metrics": {"Status": "Enabled", "EventThreshold": {"Minutes": 15}},
        "EncryptionConfiguration": {
          "ReplicaKmsKeyID": "arn:aws:kms:us-west-2:123456789:key/dr-key"
        }
      },
      "DeleteMarkerReplication": {"Status": "Enabled"},
      "SourceSelectionCriteria": {
        "SseKmsEncryptedObjects": {"Status": "Enabled"}
      }
    }]
  }'
```

---

### 🟡 Q53. What is Amazon GuardDuty vs AWS Config vs CloudTrail vs Security Hub?
| Service | WHAT it does | WHEN events fire | Primary output |
|---------|-------------|-----------------|----------------|
| **CloudTrail** | Records ALL API calls | Every API call | Audit log (who did what) |
| **AWS Config** | Tracks resource configuration changes | Config changes | Compliance snapshot |
| **GuardDuty** | Detects threats using ML | Anomaly detected | Security finding |
| **Security Hub** | Aggregates findings from all above | New finding ingested | Normalised finding |
| **Inspector** | Scans for vulnerabilities | Continuous scan | CVE finding |
| **Macie** | Finds PII in S3 | Classification job | Sensitive data finding |
| **Detective** | Investigates findings | Manual investigation | Graph analysis |

---

### 🟡 Q54. What is EC2 IMDSv2 and Nitro security?
```bash
# IMDSv2: Instance Metadata Service v2 — requires session token (blocks SSRF attacks)
# IMDSv1 was vulnerable: web app SSRF → fetch IAM credentials via metadata

# Require IMDSv2 on new instances
aws ec2 run-instances \
  --image-id ami-abc123 \
  --instance-type t3.medium \
  --metadata-options HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=2 \
  --generate-ssh-keys

# Enforce IMDSv2 on existing instances
aws ec2 modify-instance-metadata-options \
  --instance-id i-1234567890abcdef0 \
  --http-tokens required \
  --http-endpoint enabled \
  --http-put-response-hop-limit 2

# Use IMDSv2 in scripts (NOT v1 which uses curl http://169.254.169.254/latest/meta-data/)
# IMDSv2 requires a session token:
TOKEN=$(curl -X PUT -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" \
  http://169.254.169.254/latest/api/token)
ROLE_NAME=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/)
CREDS=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE_NAME)

# Python: automatically uses IMDSv2
import boto3   # boto3 uses IMDSv2 by default since version 1.18+

# Enforce IMDSv2 via SCP
'{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "RequireIMDSv2",
    "Effect": "Deny",
    "Action": "ec2:RunInstances",
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Condition": {
      "StringNotEquals": {
        "ec2:MetadataHttpTokens": "required"
      }
    }
  }]
}'

# Nitro System security:
# Hardware-enforced isolation between instances (no shared memory)
# Dedicated hardware components: VPC, EBS, ENA networking
# Nitro Hypervisor: minimal attack surface (no traditional hypervisor functions)
# Nitro Enclaves: isolated compute with no persistent storage, no external networking
# Use for: processing sensitive data (private keys, PII, ML model IP)

aws ec2 run-instances \
  --image-id ami-abc123 \
  --instance-type c5.xlarge \
  --enclave-options Enabled=true

# Connect to enclave via vsock
# Enclave processes data in isolated memory — no way for host to access

# EC2 security checklist:
# ✅ IMDSv2 required (HttpTokens=required)
# ✅ No public IP (use Bastion or SSM Session Manager)
# ✅ SSM Agent installed (for Session Manager, no SSH)
# ✅ Instance profile (IAM role) with least privilege
# ✅ Security Group: no 0.0.0.0/0 on port 22 or 3389
# ✅ EBS encryption enabled (default encryption in account)
# ✅ OS patching via SSM Patch Manager
# ✅ Inspector v2 enabled
```

---

### 🟡 Q55. What is IAM Roles Anywhere?
```bash
# IAM Roles Anywhere: use IAM roles from ANYWHERE (on-prem, other clouds)
# Uses X.509 certificates (PKI) instead of long-term IAM user keys
# Works with: on-prem servers, CI/CD outside AWS, other cloud workloads

# Create Trust Anchor (point to your CA)
aws rolesanywhere create-trust-anchor \
  --name "my-enterprise-ca" \
  --source '{
    "sourceType": "AWS_ACM_PCA",
    "sourceData": {
      "acmPcaArn": "arn:aws:acm-pca:us-east-1:123456789:certificate-authority/abc123"
    }
  }' \
  --enabled

# Or use your own CA certificate
aws rolesanywhere create-trust-anchor \
  --name "my-external-ca" \
  --source '{
    "sourceType": "CERTIFICATE_BUNDLE",
    "sourceData": {
      "x509CertificateData": "-----BEGIN CERTIFICATE-----\n..."
    }
  }' \
  --enabled

# Create Profile (maps certificate → IAM role)
aws rolesanywhere create-profile \
  --name "on-prem-servers" \
  --role-arns \
    "arn:aws:iam::123456789:role/on-prem-app-role" \
    "arn:aws:iam::123456789:role/on-prem-db-reader-role" \
  --session-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }]
  }' \
  --duration-seconds 3600 \
  --managed-policy-arns "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess" \
  --enabled

# IAM Role trust policy (allow rolesanywhere)
'{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "rolesanywhere.amazonaws.com"},
    "Action": ["sts:AssumeRole","sts:TagSession","sts:SetSourceIdentity"],
    "Condition": {
      "ArnEquals": {"aws:SourceArn": "arn:aws:rolesanywhere:us-east-1:123456789:trustanchor/abc123"}
    }
  }]
}'

# On-prem server: use AWS Signing Helper to get credentials
# Install: aws_signing_helper
aws_signing_helper credential-process \
  --certificate /path/to/server-cert.pem \
  --private-key /path/to/server-key.pem \
  --trust-anchor-arn arn:aws:rolesanywhere:us-east-1:123456789:trustanchor/abc123 \
  --profile-arn arn:aws:rolesanywhere:us-east-1:123456789:profile/abc123 \
  --role-arn arn:aws:iam::123456789:role/on-prem-app-role

# Add to ~/.aws/credentials:
# [on-prem]
# credential_process = aws_signing_helper credential-process \
#   --certificate /path/to/cert.pem --private-key /path/to/key.pem \
#   --trust-anchor-arn ... --profile-arn ... --role-arn ...
```

---

### 🟡 Q56. What is Amazon Verified Permissions?
```bash
# Verified Permissions: fine-grained authorisation using Cedar policy language
# Decouples authorisation logic from application code
# Cedar: expressive, fast, formally verified policy language from AWS

# Create policy store
aws verifiedpermissions create-policy-store \
  --validation-settings '{"mode": "STRICT"}' \
  --description "E-commerce authorisation policies"

STORE_ID=$(aws verifiedpermissions list-policy-stores --query "policyStores[0].policyStoreId" --output text)

# Create schema (define entity types and actions)
aws verifiedpermissions put-schema \
  --policy-store-id $STORE_ID \
  --definition '{
    "cedarJson": "{
      \"PhotoApp\": {
        \"entityTypes\": {
          \"User\": {\"memberOfTypes\": [\"UserGroup\"]},
          \"UserGroup\": {},
          \"Photo\": {
            \"shape\": {\"type\": \"Record\", \"attributes\": {
              \"owner\": {\"type\": \"Entity\", \"name\": \"User\"},
              \"isPublic\": {\"type\": \"Boolean\"}
            }}
          }
        },
        \"actions\": {
          \"viewPhoto\":   {\"appliesTo\": {\"principalTypes\": [\"User\"], \"resourceTypes\": [\"Photo\"]}},
          \"uploadPhoto\": {\"appliesTo\": {\"principalTypes\": [\"User\"], \"resourceTypes\": [\"Photo\"]}},
          \"deletePhoto\": {\"appliesTo\": {\"principalTypes\": [\"User\"], \"resourceTypes\": [\"Photo\"]}}
        }
      }
    }"
  }'

# Create Cedar policies
aws verifiedpermissions create-policy \
  --policy-store-id $STORE_ID \
  --definition '{
    "static": {
      "description": "Users can view public photos",
      "statement": "permit(principal, action == PhotoApp::Action::\"viewPhoto\", resource) when {resource.isPublic};"
    }
  }'

aws verifiedpermissions create-policy \
  --policy-store-id $STORE_ID \
  --definition '{
    "static": {
      "description": "Users can manage their own photos",
      "statement": "permit(principal, action in [PhotoApp::Action::\"uploadPhoto\",PhotoApp::Action::\"deletePhoto\",PhotoApp::Action::\"viewPhoto\"], resource) when {resource.owner == principal};"
    }
  }'

aws verifiedpermissions create-policy \
  --policy-store-id $STORE_ID \
  --definition '{
    "static": {
      "description": "Admins can do everything",
      "statement": "permit(principal in PhotoApp::UserGroup::\"admins\", action, resource);"
    }
  }'

# Authorisation check (in your application)
aws verifiedpermissions is-authorized \
  --policy-store-id $STORE_ID \
  --principal '{"entityType": "PhotoApp::User", "entityId": "user-123"}' \
  --action '{"actionType": "PhotoApp::Action", "actionId": "viewPhoto"}' \
  --resource '{"entityType": "PhotoApp::Photo", "entityId": "photo-456"}' \
  --entities '{
    "entityList": [
      {
        "identifier": {"entityType": "PhotoApp::Photo", "entityId": "photo-456"},
        "attributes": {
          "owner": {"entityIdentifier": {"entityType": "PhotoApp::User", "entityId": "user-789"}},
          "isPublic": {"boolean": true}
        }
      }
    ]
  }'
# Response: {"decision": "ALLOW", "determiningPolicies": [...], "errors": []}
```

---

### 🟡 Q57. What is the Zero Trust security model on AWS?
```bash
# Zero Trust: "never trust, always verify"
# Traditional: trust inside network perimeter
# Zero Trust: verify identity + device + context for EVERY request

# Zero Trust pillars on AWS:

# 1. IDENTITY (Entra ID / Cognito / IAM Identity Center)
# - MFA required for all access
# - Short-lived credentials (STS, Cognito tokens)
# - Least privilege (IAM Access Analyzer)
# - Continuous verification (GuardDuty, Security Hub)

# 2. DEVICE
# - IMDSv2 on EC2 (prevent SSRF credential theft)
# - Inspector v2 (vulnerability scanning)
# - AWS Systems Manager (patch compliance)
# - GuardDuty Malware Protection (EBS scanning)

# 3. NETWORK
# - No public IPs (Private subnets + NAT GW)
# - VPC Endpoints (no internet for AWS services)
# - Network Firewall (stateful inspection)
# - Security Groups (deny-by-default, least privilege)
# - No broad VPN (Verified Access for specific apps)

# 4. APPLICATION
# - WAF on every public endpoint
# - API Gateway with Cognito/Lambda authoriser
# - mTLS between services (ACM Private CA)
# - Secrets Manager (no hardcoded credentials)

# 5. DATA
# - KMS encryption (all data at rest)
# - TLS 1.2+ (all data in transit)
# - Macie (PII detection in S3)
# - S3 Block Public Access
# - Object Lock (WORM for critical data)

# AWS Verified Access (Zero Trust application access)
aws verifiedaccess create-instance \
  --description "Zero Trust access to internal apps"

aws verifiedaccess create-trust-provider \
  --trust-provider-type user \
  --user-trust-provider-type iam-identity-center \
  --policy-reference-name "idc" \
  --description "IAM Identity Center trust"

aws verifiedaccess create-group \
  --verified-access-instance-id vai-abc123 \
  --policy-document '
    permit(principal, action, resource)
    when {
      context.idc.groups.contains("engineering") &&
      context.idc.mfaAuthTime > (now() - 8h) &&
      context.device.isCompliant
    };'

aws verifiedaccess create-endpoint \
  --verified-access-group-id vag-abc123 \
  --endpoint-type load-balancer \
  --attachment-type vpc \
  --application-domain my-internal-app.mycompany.com \
  --domain-certificate-arn arn:aws:acm:us-east-1:123456789:certificate/abc123 \
  --load-balancer-options '{
    "LoadBalancerArn": "arn:aws:elasticloadbalancing:...",
    "Port": 443,
    "Protocol": "https",
    "SubnetIds": ["subnet-11111111"]
  }'
```

---

### 🟡 Q58. What are S3 security best practices?
```bash
# Complete S3 security checklist

# 1. Block Public Access (account-level)
aws s3control put-public-access-block \
  --account-id 123456789 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,\
    BlockPublicPolicy=true,RestrictPublicBuckets=true

# 2. Default encryption on all new buckets (account-level)
aws s3control put-account-default-encryption \
  --account-id 123456789 \
  --sse-algorithm aws:kms \
  --kms-master-key-id arn:aws:kms:us-east-1:123456789:key/abc123

# 3. Require HTTPS (deny HTTP) in bucket policy
aws s3api put-bucket-policy --bucket my-bucket --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyHTTP",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": ["arn:aws:s3:::my-bucket","arn:aws:s3:::my-bucket/*"],
    "Condition": {"Bool": {"aws:SecureTransport": "false"}}
  }]
}'

# 4. Enable access logging
aws s3api put-bucket-logging \
  --bucket my-bucket \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "my-access-logs",
      "TargetPrefix": "my-bucket/"
    }
  }'

# 5. Enable CloudTrail data events for S3 (object-level operations)
aws cloudtrail put-event-selectors \
  --trail-name my-trail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{"Type": "AWS::S3::Object", "Values": ["arn:aws:s3:::my-bucket/"]}]
  }]'

# 6. Intelligent-Tiering + Lifecycle
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket my-bucket \
  --id "archive-old-objects" \
  --intelligent-tiering-configuration '{
    "Id": "archive-old-objects",
    "Status": "Enabled",
    "Tierings": [
      {"Days": 90,  "AccessTier": "ARCHIVE_ACCESS"},
      {"Days": 180, "AccessTier": "DEEP_ARCHIVE_ACCESS"}
    ]
  }'

# 7. Macie for PII detection (covered in Q18)
# 8. Object Lock for immutability (covered in Q52)
# 9. VPC Endpoint to restrict S3 to within VPC (covered in Q24)
# 10. Pre-signed URLs with expiry for sharing
aws s3 presign s3://my-bucket/secret-report.pdf \
  --expires-in 3600   # 1 hour only
```

---

### 🟡 Q59. What are compliance frameworks on AWS?
```bash
# AWS compliance certifications (AWS-managed):
# PCI DSS Level 1:    payment card processing
# HIPAA:              healthcare data
# SOC 1, 2, 3:        security, availability, processing integrity
# ISO 27001/27017/27018: information security
# FedRAMP High:       US federal agencies
# GDPR:               EU data protection
# NIST 800-171:       CUI (Controlled Unclassified Information)
# IRAP:               Australian government
# C5:                 German BSI cloud standard

# Customer responsibility for compliance:
# 1. Use only compliant AWS services (check aws.amazon.com/compliance/services-in-scope)
# 2. Enable required controls (encryption, logging, access control)
# 3. Collect evidence (AWS Audit Manager)
# 4. Third-party audit

# HIPAA on AWS:
# Sign BAA (Business Associate Agreement) with AWS
aws artifact get-agreement-terms \
  --agreement-type HIPAA_BAA \
  --query "effectiveAgreements[0].effectiveDate"

# HIPAA-eligible services: EC2, S3, RDS, Lambda, DynamoDB, ECS, EKS, Redshift, etc.
# HIPAA controls on AWS:
# - Encryption at rest: KMS on S3, RDS, EBS
# - Encryption in transit: TLS 1.2+ everywhere
# - Access control: IAM least privilege + MFA
# - Audit: CloudTrail enabled in all regions
# - Backup: automated backups + cross-region replication

# PCI DSS on AWS:
# Requirement 1: Firewall → VPC SGs + NACLs + Network Firewall
# Requirement 2: No defaults → change admin passwords, disable unused services
# Requirement 3: Protect CHD → KMS encryption, Macie for PII
# Requirement 4: Encrypt transmission → TLS + WAF + ACM
# Requirement 5: Antivirus → GuardDuty Malware Protection, Inspector
# Requirement 6: Secure systems → Inspector CVE scanning, Config compliance rules
# Requirement 7: Need-to-know → IAM least privilege, SCP restrictions
# Requirement 8: Authentication → IAM MFA, Cognito, Identity Center
# Requirement 10: Audit trails → CloudTrail + Security Hub + GuardDuty
# Requirement 11: Security testing → Inspector + Security Hub + pen testing

# SOC 2 Type II — Trust Service Criteria:
# Security → GuardDuty, Security Hub, IAM, WAF
# Availability → Multi-AZ, Auto Scaling, Route53 health checks
# Processing Integrity → CloudTrail, AWS Config
# Confidentiality → KMS, ACM, VPC Endpoints
# Privacy → Macie, S3 Block Public Access

# AWS Audit Manager makes evidence collection automatic
# AWS Artifact provides compliance reports on demand
aws artifact get-report \
  --report-id SOC2_TYPE2_REPORT \
  --report-version LATEST \
  --termination-protect-enable true
```

---

### 🟡 Q60. What is AWS CloudTrail security — log file validation, immutability?
```bash
# CloudTrail security best practices

# 1. Multi-region trail covering all regions and global services
aws cloudtrail create-trail \
  --name org-security-trail \
  --s3-bucket-name my-cloudtrail-logs \
  --is-multi-region-trail \
  --include-global-service-events \
  --enable-log-file-validation \           # SHA-256 hash of every log file
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/abc123 \
  --is-organization-trail                   # covers ALL org accounts

aws cloudtrail start-logging --name org-security-trail

# 2. Log file validation (detect tampering)
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:us-east-1:123456789:trail/org-security-trail \
  --start-time $(date -u -d '-7 days' +%Y-%m-%dT%H:%M:%SZ) \
  --s3-bucket my-cloudtrail-logs \
  --s3-prefix AWSLogs/

# 3. Immutable S3 bucket (Object Lock for trail bucket)
aws s3api put-object-lock-configuration \
  --bucket my-cloudtrail-logs \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {"DefaultRetention": {"Mode": "COMPLIANCE", "Days": 2557}}
  }'

# 4. SCP: deny trail modification by ANYONE
'{
  "Sid": "ProtectCloudTrail",
  "Effect": "Deny",
  "Action": [
    "cloudtrail:StopLogging",
    "cloudtrail:DeleteTrail",
    "cloudtrail:UpdateTrail",
    "cloudtrail:PutEventSelectors",
    "cloudtrail:RemoveTags",
    "cloudtrail:CreateTrail"
  ],
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:PrincipalArn": "arn:aws:iam::123456789:role/security-admin-role"
    }
  }
}'

# 5. S3 bucket policy: deny deletion of trail logs
aws s3api put-bucket-policy --bucket my-cloudtrail-logs --policy '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDelete",
      "Effect": "Deny",
      "Principal": "*",
      "Action": ["s3:DeleteObject","s3:DeleteObjectVersion","s3:DeleteBucket"],
      "Resource": [
        "arn:aws:s3:::my-cloudtrail-logs",
        "arn:aws:s3:::my-cloudtrail-logs/*"
      ]
    },
    {
      "Sid": "AllowCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-cloudtrail-logs/AWSLogs/*",
      "Condition": {
        "StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"},
        "StringLike": {"aws:SourceArn": "arn:aws:cloudtrail:*:123456789:trail/*"}
      }
    }
  ]
}'

# 6. CloudWatch Alarms on critical events
# Root login alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "RootAccountLogin" \
  --alarm-description "Alert when root account signs in" \
  --metric-name RootAccountLogin \
  --namespace CloudTrailMetrics \
  --statistic Sum --period 300 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:security-alerts

# Create the metric filter for root login
aws logs put-metric-filter \
  --log-group-name cloudtrail \
  --filter-name RootAccountLogin \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations \
    metricName=RootAccountLogin,metricNamespace=CloudTrailMetrics,metricValue=1

# Other critical metric filters:
# IAM policy changes:    "$.eventSource = 'iam.amazonaws.com' && ..."
# Unauthorized API calls: "$.errorCode = 'AccessDenied' || $.errorCode = 'UnauthorizedOperation'"
# MFA device changes:    "$.eventName = 'DeactivateMFADevice' || $.eventName = 'DeleteVirtualMFADevice'"
# S3 bucket policy:      "$.eventSource = 's3.amazonaws.com' && ..."
# Network changes:       "$.eventName = 'AuthorizeSecurityGroupIngress' || ..."
# Console sign-in without MFA: "$.eventName = 'ConsoleLogin' && $.additionalEventData.MFAUsed != 'Yes'"
```


---

# GAP-FILL — MIGRATION (Q61–Q70)

---

### 🟡 Q61. What is VMware Cloud on AWS (VMC)?
```bash
# VMC on AWS: run VMware workloads natively in AWS datacenters
# Same VMware tools: vSphere, vSAN, NSX-T, HCX
# Use case: Relocate strategy (7th R) — no re-architecture, fast migration
# Shared infrastructure: AWS-dedicated hardware, VMware Cloud Foundation

# Create SDDC (Software Defined Data Center)
# Done via VMware Cloud console (not AWS CLI)
# Steps:
# 1. Link VMware Cloud account to AWS account
# 2. Select region, AZ, host count (3 minimum)
# 3. Deploy SDDC (2-3 hours)
# 4. Connect to VPC via ENI (Elastic Network Interface — 25Gbps link)

# HCX (Hybrid Cloud Extension) for live migration:
# Layer 2 network extension → same IP addresses, no reconfiguration
# vMotion over internet → migrate running VMs with < 1 min downtime
# Bulk migration → cold migrate many VMs at once
# Replication Assisted vMotion (RAV) → pre-copy + final vMotion

# VMC on AWS pricing:
# i3.metal or i3en.metal hosts
# $6.50/host/hour (on-demand)
# 1-year: ~30% discount | 3-year: ~50% discount
# Minimum: 3 hosts (1 cluster), max: 640 hosts per SDDC

# When to use VMC vs native AWS:
# VMC: VMware-specific features (vSAN, NSX-T, DRS), fast migration, compliance
# Native AWS: cost optimisation, cloud-native features, long-term modernisation
```

---

### 🟡 Q62. What is AWS Outposts?
```bash
# Outposts: bring AWS compute, storage, databases to your premises
# Same AWS APIs, same hardware as AWS regions
# Use for: low-latency local processing, data residency, offline scenarios

# Outpost types:
# Outposts Rack:   full 42U rack, 1-96 racks per site
# Outposts Server: 1U or 2U server for small/remote locations

# Order an Outpost
aws outposts create-outpost \
  --name "NYC-DataCenter-Outpost" \
  --site-id op-abc123 \
  --availability-zone us-east-1a \
  --availability-zone-id use1-az1 \
  --description "On-premises compute for NYC datacenter" \
  --tags Key=Location,Value=NYC

# List available capacity
aws outposts get-outpost-instance-types \
  --outpost-id op-abc123 \
  --query "InstanceTypes[].{Type:InstanceType,Available:TotalCapacity.INSTANCE}"

# Run EC2 on Outpost (specify Outpost ARN)
aws ec2 run-instances \
  --image-id ami-abc123 \
  --instance-type m5.xlarge \
  --placement '{"Tenancy": "default", "GroupName": ""}' \
  --network-interfaces '[{
    "SubnetId": "subnet-outpost-abc",
    "DeviceIndex": 0
  }]' \
  --outpost-arn arn:aws:outposts:us-east-1:123456789:outpost/op-abc123

# RDS on Outposts
aws rds create-db-instance \
  --db-instance-identifier my-outpost-db \
  --db-instance-class db.m5.xlarge \
  --engine mysql \
  --master-username admin \
  --master-user-password "P@ssw0rd!" \
  --db-subnet-group-name outpost-subnet-group \
  --outpost-identifier arn:aws:outposts:us-east-1:123456789:outpost/op-abc123 \
  --backup-retention-period 7

# EKS on Outposts
aws eks create-cluster \
  --name my-outpost-eks \
  --role-arn arn:aws:iam::123456789:role/eks-role \
  --outpost-config '{
    "outpostArns": ["arn:aws:outposts:us-east-1:123456789:outpost/op-abc123"],
    "controlPlaneInstanceType": "m5.xlarge",
    "controlPlanePlacement": {"groupName": "outpost-placement-group"}
  }'

# ECS on Outposts — same as ECS Anywhere (register Outpost instances)

# Outpost networking:
# LGW (Local Gateway): route traffic between Outpost and on-prem
# SLG (Service Link): connect Outpost to AWS region (encrypted VPN)
# LGW route table → on-prem destinations
aws ec2 create-route \
  --route-table-id rtb-outpost-abc \
  --destination-cidr-block 192.168.0.0/16 \
  --local-gateway-id lgw-abc123

# Use cases:
# Low-latency: manufacturing automation, real-time gaming, trading
# Data residency: GDPR, financial regulations
# Disconnected: Outpost Servers for remote edge locations
```

---

### 🟡 Q63. What are DNS migration strategies with Route53?
```bash
# DNS migration: minimise downtime during cutover using TTL management

# Step 1: Reduce TTL BEFORE migration (days before cutover)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "myapp.example.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "1.2.3.4"}]
      }
    }]
  }'
# Wait 24-48 hours for old TTL to expire everywhere

# Step 2: Cutover — update to new AWS IP
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "myapp.example.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "52.1.2.3"}]
      }
    }]
  }'

# Step 3: Validate (keep old server running for TTL duration)
# Wait 60 seconds (your new TTL) before decommissioning

# Step 4: After validation — restore TTL
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "myapp.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "52.1.2.3"}]
      }
    }]
  }'

# Weighted routing for gradual traffic shift (zero-downtime migration)
# Send 10% to AWS, 90% to on-prem
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123ABC \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "myapp.example.com",
          "Type": "A",
          "SetIdentifier": "on-prem",
          "Weight": 90,
          "TTL": 60,
          "ResourceRecords": [{"Value": "1.2.3.4"}],
          "HealthCheckId": "hc-on-prem"
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "myapp.example.com",
          "Type": "A",
          "SetIdentifier": "aws",
          "Weight": 10,
          "TTL": 60,
          "ResourceRecords": [{"Value": "52.1.2.3"}],
          "HealthCheckId": "hc-aws"
        }
      }
    ]
  }'

# Health check (auto-failback if AWS target is unhealthy)
aws route53 create-health-check \
  --health-check-config '{
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "FullyQualifiedDomainName": "52.1.2.3",
    "Port": 443,
    "RequestInterval": 30,
    "FailureThreshold": 3,
    "EnableSNI": true
  }' \
  --caller-reference $(date +%s)

# Migrate DNS from on-prem to Route53 (domain transfer or NS delegation)
aws route53domains transfer-domain \
  --domain-name example.com \
  --duration-in-years 1 \
  --nameservers \
    Name=ns-1.awsdns-01.com Name=ns-2.awsdns-02.org \
    Name=ns-3.awsdns-03.net Name=ns-4.awsdns-04.co.uk \
  --admin-contact file://contact.json \
  --registrant-contact file://contact.json \
  --tech-contact file://contact.json \
  --auth-code "domain-auth-code"
```

---

### 🟡 Q64. What is post-migration optimisation?
```bash
# After migration: optimise cost, performance, resilience

# 1. Right-sizing (use Compute Optimizer)
aws compute-optimizer update-enrollment-status --status Active

aws compute-optimizer get-ec2-instance-recommendations \
  --query "instanceRecommendations[?finding!='OPTIMIZED'].{
    Instance:instanceName,
    Current:currentInstanceType,
    Recommended:recommendationOptions[0].instanceType,
    Savings:recommendationOptions[0].estimatedMonthlySavings.value
  }" --output table

# 2. Reserved Instances (1-3 year commitment for predictable workloads)
# Savings: up to 72% vs on-demand
aws ec2 describe-reserved-instances-offerings \
  --instance-type t3.large \
  --product-description "Linux/UNIX" \
  --offering-class convertible \
  --duration 31536000 \    # 1 year
  --query "ReservedInstancesOfferings[?OfferingType=='All Upfront'].{Type:InstanceType,Upfront:FixedPrice,Monthly:RecurringCharges[0].Amount}"

aws ec2 purchase-reserved-instances-offering \
  --reserved-instances-offering-id <offering-id> \
  --instance-count 10

# 3. Savings Plans (flexible, covers EC2 + Fargate + Lambda)
aws savingsplans describe-savings-plans-offering-rates \
  --savings-plan-offering-types COMPUTE_SP \
  --products EC2 Fargate Lambda

# 4. Spot Instances for batch/stateless workloads
aws ec2 request-spot-fleet \
  --spot-fleet-request-config '{
    "TargetCapacity": 100,
    "TargetCapacityUnitType": "vcpu",
    "AllocationStrategy": "priceCapacityOptimized",
    "LaunchTemplateConfigs": [{
      "LaunchTemplateSpecification": {"LaunchTemplateId": "lt-abc123", "Version": "$Latest"},
      "Overrides": [
        {"InstanceType": "m5.xlarge", "WeightedCapacity": 4},
        {"InstanceType": "m6i.xlarge", "WeightedCapacity": 4},
        {"InstanceType": "r5.xlarge", "WeightedCapacity": 4}
      ]
    }]
  }'

# 5. Auto Scaling (scale in during off-hours)
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-cluster/my-service \
  --min-capacity 2 --max-capacity 50

# 6. S3 Intelligent Tiering (auto-move to cheaper tiers)
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket my-bucket --id default \
  --intelligent-tiering-configuration '{
    "Id": "default", "Status": "Enabled",
    "Tierings": [
      {"Days": 90,  "AccessTier": "ARCHIVE_ACCESS"},
      {"Days": 180, "AccessTier": "DEEP_ARCHIVE_ACCESS"}
    ]
  }'

# 7. Enable Cost Anomaly Detection
aws ce create-anomaly-monitor \
  --anomaly-monitor '{"MonitorName":"PostMigrationMonitor","MonitorType":"DIMENSIONAL","MonitorDimension":"SERVICE"}'

# 8. Enable Trusted Advisor checks
aws trustedadvisor list-recommendations \
  --pillar cost_optimising \
  --status warning error \
  --output table
```

---

### 🟡 Q65. Migration tools comparison — DataSync vs Storage Gateway vs Snow?
| Feature | DataSync | Storage Gateway | Snow Family |
|---------|---------|----------------|------------|
| **Use case** | One-time + ongoing sync | Ongoing hybrid access | One-time large transfer |
| **Protocol** | Agent-based | NFS/SMB/iSCSI/VTL | Physical device |
| **Speed** | 10 Gbps network | Limited by WAN | Up to PB in weeks |
| **Connectivity** | Internet or DX | Internet or DX | Physical shipping |
| **Cache** | No | Yes (local cache) | N/A |
| **Online/Offline** | Online only | Online only | Offline |
| **Data types** | Files, objects | Files, blocks, tape | Files, objects, blocks |
| **Best for** | File/object migration | Hybrid storage extension | Low/no bandwidth |

---

# GAP-FILL — ANALYTICS (Q66–Q80)

---

### 🟡 Q66. What is Redshift Workload Management (WLM) and Concurrency Scaling?
```sql
-- WLM: allocate cluster resources across query queues

-- Create WLM configuration (via Parameter Group)
-- Queue types:
-- Short query acceleration (SQA): auto-detect and prioritise short queries
-- Manual WLM: define queues by user group or query group
-- Auto WLM: Redshift manages queue priorities automatically

-- Auto WLM (recommended) with query priority
-- In Redshift Parameter Group:
-- wlm_json_configuration: [
--   {"auto_wlm": true},
--   {"query_group": ["highest"], "query_group_wild_card": 0,
--    "user_group": ["executives"], "priority": "highest"},
--   {"query_group": ["etl"], "priority": "normal",
--    "query_execution_timeout": 3600},
--   {"query_group": [], "user_group": [], "priority": "low"}
-- ]

-- Concurrency Scaling: burst capacity for high-concurrency
-- Automatically adds cluster capacity during peak query demand
-- Costs: 1 free hour/day per 24 active hours

-- Enable concurrency scaling on queue
-- wlm_json_configuration: [{
--   "concurrency_scaling": "auto",
--   "queue_type": "manual",
--   "user_group": ["analysts"]
-- }]

-- Check WLM queue status
SELECT wlm_queue_name, num_queued_queries, num_executing_queries,
       estimated_time_to_complete
FROM STV_WLM_SERVICE_CLASS_STATE;

-- Check concurrency scaling activity
SELECT node_role, starttime, endtime, duration_secs
FROM SVL_CONCURRENCY_SCALING_ACTIVITY
ORDER BY starttime DESC LIMIT 20;

-- Redshift ML (CREATE MODEL with SageMaker)
CREATE MODEL customer_churn_model
FROM (SELECT age, tenure, monthly_spend, churn FROM training_data)
TARGET churn
FUNCTION predict_churn
IAM_ROLE 'arn:aws:iam::123456789:role/redshift-sagemaker-role'
AUTO OFF
MODEL_TYPE XGBoost
OBJECTIVE 'binary:logistic'
SETTINGS (
  S3_BUCKET 'my-redshift-ml-bucket',
  MAX_CELLS 20000000
);

-- Use the model for prediction
SELECT customer_id, predict_churn(age, tenure, monthly_spend) AS churn_probability
FROM new_customers
WHERE predict_churn(age, tenure, monthly_spend) > 0.7;
```

---

### 🟡 Q67. What is Apache Iceberg on AWS?
```python
# Iceberg: open table format for huge analytics datasets
# Features: ACID transactions, schema evolution, time travel, partition evolution
# Supported by: Athena, Glue, EMR, Redshift Spectrum, Spark

# Create Iceberg table in Athena
import boto3

athena = boto3.client("athena")

athena.start_query_execution(
    QueryString="""
    CREATE TABLE mydb.orders_iceberg (
        order_id     STRING,
        customer_id  STRING,
        amount       DOUBLE,
        order_date   DATE,
        status       STRING
    )
    PARTITIONED BY (month(order_date))   -- hidden partitioning
    LOCATION 's3://my-data-lake/orders-iceberg/'
    TBLPROPERTIES (
        'table_type'          = 'ICEBERG',
        'format'              = 'parquet',
        'write_compression'   = 'snappy',
        'optimize_rewrite_delete_file_threshold' = '10'
    )
    """,
    QueryExecutionContext={"Database": "mydb"},
    ResultConfiguration={"OutputLocation": "s3://my-athena-results/"}
)

# ACID operations on Iceberg
# INSERT
athena.start_query_execution(QueryString="""
    INSERT INTO mydb.orders_iceberg VALUES
    ('ORD-001', 'C001', 99.99, DATE '2026-06-13', 'DELIVERED')
""", ...)

# UPDATE (not possible in regular Hive tables — Iceberg only)
athena.start_query_execution(QueryString="""
    UPDATE mydb.orders_iceberg
    SET status = 'RETURNED'
    WHERE order_id = 'ORD-001'
""", ...)

# DELETE
athena.start_query_execution(QueryString="""
    DELETE FROM mydb.orders_iceberg
    WHERE order_date < DATE '2020-01-01'
""", ...)

# MERGE (upsert)
athena.start_query_execution(QueryString="""
    MERGE INTO mydb.orders_iceberg t
    USING new_orders s ON t.order_id = s.order_id
    WHEN MATCHED THEN
        UPDATE SET status = s.status, amount = s.amount
    WHEN NOT MATCHED THEN
        INSERT VALUES (s.order_id, s.customer_id, s.amount, s.order_date, s.status)
""", ...)

# Time travel (query historical data)
athena.start_query_execution(QueryString="""
    SELECT * FROM mydb.orders_iceberg
    FOR TIMESTAMP AS OF TIMESTAMP '2026-06-01 10:00:00'
""", ...)

# Schema evolution (add/rename/drop columns safely)
athena.start_query_execution(QueryString="""
    ALTER TABLE mydb.orders_iceberg
    ADD COLUMNS (shipping_address STRING, discount_amount DOUBLE)
""", ...)

# Optimize: compact small files
athena.start_query_execution(QueryString="""
    OPTIMIZE mydb.orders_iceberg REWRITE DATA USING BIN_PACK
    WHERE month(order_date) = 6
""", ...)

# Vacuum: remove deleted data files
athena.start_query_execution(QueryString="""
    VACUUM mydb.orders_iceberg
""", ...)
```

---

### 🟡 Q68. What is AWS Glue Data Quality?
```python
# Glue Data Quality: automatically measure, monitor, and enforce data quality

# DQDL — Data Quality Definition Language (rules DSL)
dqdl_rules = """
Rules = [
    # Completeness
    IsComplete "order_id",
    IsComplete "customer_id",
    IsComplete "amount",

    # Validity
    ColumnValues "amount" > 0,
    ColumnValues "amount" <= 100000,
    ColumnValues "status" in ["PENDING","CONFIRMED","SHIPPED","DELIVERED","CANCELLED"],
    MatchesRegex "order_id" "ORD-[0-9]{6}",

    # Uniqueness
    IsPrimaryKey "order_id",

    # Freshness
    ColumnValues "order_date" >= (now() - 365 days),

    # Referential integrity
    Referential Integrity "customer_id" [distinct] = table("customers", "customer_id"),

    # Completeness threshold
    Completeness "amount" >= 0.99,    # at least 99% non-null

    # Statistical
    ColumnStatistics "amount" {
        Mean between 50 and 500,
        StandardDeviation < 1000,
        Min >= 0,
        Max < 100000
    },

    # Custom SQL rule
    CustomSql "SELECT count(*) FROM primary WHERE order_date > CURRENT_DATE" = 0
]
"""

# Create Glue Data Quality ruleset
import boto3
glue = boto3.client("glue")

glue.create_data_quality_ruleset(
    Name="orders-quality-rules",
    Ruleset=dqdl_rules,
    TargetTable={
        "DatabaseName": "mydb",
        "TableName": "orders"
    },
    Tags={"Environment": "production"}
)

# Run data quality evaluation
response = glue.start_data_quality_ruleset_evaluation_run(
    DataSource={
        "GlueTable": {"DatabaseName": "mydb", "TableName": "orders"}
    },
    Role="arn:aws:iam::123456789:role/glue-role",
    RulesetNames=["orders-quality-rules"],
    AdditionalRunOptions={
        "CloudWatchMetricsEnabled": True,
        "ResultsS3Prefix": "s3://my-dq-results/"
    }
)

run_id = response["RunId"]

# Check results
results = glue.get_data_quality_ruleset_evaluation_run(RunId=run_id)
for result in results["RuleResults"]:
    status = "✅" if result["Result"] == "PASS" else "❌"
    print(f"{status} {result['Name']}: {result['Result']} (score: {result.get('EvaluatedMetrics',{}).get('compliance',0):.2%})")

# In ETL pipeline: fail job if quality score below threshold
if results["Score"] < 0.95:
    raise Exception(f"Data quality too low: {results['Score']:.2%}. Aborting ETL.")
```

---

### 🟡 Q69. What is Amazon Kinesis Enhanced Fan-Out (EFO)?
```python
# EFO: dedicated throughput per consumer (2 MB/s per shard per consumer)
# Without EFO: all consumers share 2 MB/s per shard total
# Use when: multiple consumers need full throughput simultaneously

import boto3

kinesis = boto3.client("kinesis", region_name="us-east-1")
STREAM_ARN = "arn:aws:kinesis:us-east-1:123456789:stream/my-stream"

# Register enhanced fan-out consumer
response = kinesis.register_stream_consumer(
    StreamARN=STREAM_ARN,
    ConsumerName="analytics-consumer"
)
CONSUMER_ARN = response["Consumer"]["ConsumerARN"]

# Subscribe to shard with EFO (push model — no polling needed)
def process_shard_with_efo(shard_id: str):
    # Get shard iterator for starting position
    response = kinesis.subscribe_to_shard(
        ConsumerARN=CONSUMER_ARN,
        ShardId=shard_id,
        StartingPosition={"Type": "LATEST"}
    )

    # EFO uses HTTP/2 streaming — events pushed to you
    for event in response["EventStream"]:
        if "SubscribeToShardEvent" in event:
            records = event["SubscribeToShardEvent"]["Records"]
            for record in records:
                data = json.loads(record["Data"].decode())
                process_record(data)

            # Track checkpoint
            sequence = records[-1]["SequenceNumber"] if records else None
            if sequence:
                save_checkpoint(shard_id, sequence)

# EFO benefits:
# ✅ 2 MB/s per shard per registered consumer (dedicated)
# ✅ Push model (HTTP/2 streaming — lower latency)
# ✅ No impact on other consumers
# ✅ Lower latency (5ms vs 200ms for polling)

# KCL (Kinesis Client Library) — handles:
# - Shard discovery and enumeration
# - Checkpointing (DynamoDB)
# - Resharding coordination
# - Graceful failover
# pip install amazon-kinesis-client

# kinesis_consumer.py with KCL
from amazon_kinesis_video_streams_consumer_library.kinesis_video_fragment_processor import KinesisVideoFragmentProcessor

# Or use boto3 directly with checkpoint tracking
class KinesisConsumer:
    def __init__(self, stream_name: str, consumer_name: str):
        self.kinesis = boto3.client("kinesis")
        self.dynamodb = boto3.resource("dynamodb")
        self.checkpoint_table = self.dynamodb.Table("kinesis-checkpoints")
        self.stream_name = stream_name

    def get_checkpoint(self, shard_id: str) -> str:
        try:
            item = self.checkpoint_table.get_item(Key={"shardId": shard_id})
            return item.get("Item", {}).get("sequenceNumber")
        except:
            return None

    def save_checkpoint(self, shard_id: str, sequence: str):
        self.checkpoint_table.put_item(Item={
            "shardId": shard_id,
            "sequenceNumber": sequence,
            "updatedAt": datetime.utcnow().isoformat()
        })
```

---

### 🟡 Q70. What is AWS Clean Rooms?
```bash
# Clean Rooms: collaborate on datasets without sharing raw data
# Each party keeps data in their own AWS account
# Run queries on combined data — results are the only thing shared

# Common use cases:
# Ad measurement: advertiser + publisher match conversions without sharing user data
# Healthcare: hospital + pharma match patient cohorts for drug effectiveness
# Financial: bank + credit bureau perform joint fraud analysis

# Create collaboration
aws cleanrooms create-collaboration \
  --name "Advertiser-Publisher-Collab" \
  --description "Joint campaign performance analysis" \
  --members '[{
    "accountId": "123456789",
    "displayName": "Advertiser",
    "memberAbilities": ["CAN_QUERY","CAN_RECEIVE_RESULTS"]
  }]' \
  --creator-display-name "Publisher" \
  --creator-member-abilities CAN_QUERY CAN_RECEIVE_RESULTS \
  --query-log-status ENABLED

COLLAB_ID=$(aws cleanrooms list-collaborations --query "collaborationList[0].id" --output text)

# Create configured table (your data contribution)
aws cleanrooms create-configured-table \
  --name "publisher-impressions" \
  --table-reference '{
    "glue": {
      "databaseName": "publisher_db",
      "tableName": "ad_impressions"
    }
  }' \
  --allowed-columns user_id campaign_id impression_date site_id \
  --analysis-method DIRECT_QUERY

# Create analysis rule (control what queries are allowed)
aws cleanrooms create-configured-table-analysis-rule \
  --configured-table-identifier publisher-impressions \
  --analysis-rule-type AGGREGATION \
  --analysis-rule-policy '{
    "v1": {
      "aggregation": {
        "aggregateColumns": [
          {"columnNames": ["user_id"], "function": "COUNT_DISTINCT"},
          {"columnNames": ["campaign_id"], "function": "COUNT"}
        ],
        "joinColumns": ["user_id"],
        "joinRequired": "QUERY_RUNNER_PAYLOAD",
        "allowedResultReceivers": ["123456789"],
        "outputConstraints": [{
          "columnName": "user_id",
          "minimum": 100,            # minimum 100 users to protect privacy
          "type": "COUNT_DISTINCT"
        }]
      }
    }
  }'

# Run protected query (results only — no raw data exposure)
aws cleanrooms start-protected-query \
  --membership-identifier <membership-id> \
  --type SQL \
  --sql-parameters '{
    "queryString": "SELECT campaign_id, COUNT(DISTINCT i.user_id) AS matched_users FROM publisher_impressions i JOIN advertiser_conversions a ON i.user_id = a.user_id WHERE i.impression_date BETWEEN '\''2026-06-01'\'' AND '\''2026-06-30'\'' GROUP BY campaign_id HAVING COUNT(DISTINCT i.user_id) >= 100",
    "analysisTemplateArn": null
  }' \
  --result-configuration '{
    "outputConfiguration": {
      "s3": {"resultFormat": "CSV", "bucket": "my-results-bucket", "keyPrefix": "clean-rooms/"}
    }
  }'

# Clean Rooms ML (new) — train ML models on combined data without sharing
aws cleanrooms-ml create-training-dataset \
  --name "joint-churn-training" \
  --role-arn arn:aws:iam::123456789:role/cleanrooms-ml-role \
  --training-data '[{
    "type": "INTERACTIONS",
    "input-config": {"datasets": [{"type": "INTERACTIONS", "location": {"s3": {"s3Uri": "s3://my-data/interactions/"}}}]},
    "schema": [{"column-name": "USER_ID", "column-types": ["USER_ID"]},{"column-name": "ITEM_ID", "column-types": ["ITEM_ID"]}]
  }]'
```

---

### 🟡 Q71. What is AWS Entity Resolution?
```bash
# Entity Resolution: match and link records across datasets without common ID
# Use: customer 360, deduplication, cross-channel identity resolution

# Create Schema Mapping
aws entityresolution create-schema-mapping \
  --schema-name "customer-schema" \
  --mapped-input-fields '[
    {"fieldName": "email",       "type": "EMAIL_ADDRESS"},
    {"fieldName": "phone",       "type": "PHONE_NUMBER"},
    {"fieldName": "name",        "type": "NAME"},
    {"fieldName": "address",     "type": "ADDRESS"},
    {"fieldName": "customer_id", "type": "UNIQUE_ID"}
  ]'

# Create matching workflow (rule-based)
aws entityresolution create-matching-workflow \
  --workflow-name "customer-dedup" \
  --input-source-config '[
    {
      "inputSourceARN": "arn:aws:glue:us-east-1:123456789:table/mydb/crm_customers",
      "schemaName": "customer-schema",
      "applyNormalization": true
    },
    {
      "inputSourceARN": "arn:aws:glue:us-east-1:123456789:table/mydb/ecommerce_customers",
      "schemaName": "customer-schema",
      "applyNormalization": true
    }
  ]' \
  --output-source-config '[{
    "outputS3Path": "s3://my-er-output/matched-customers/",
    "applyNormalization": false,
    "output": [
      {"name": "customer_id", "hashed": false},
      {"name": "email", "hashed": true},
      {"name": "matchId", "hashed": false}
    ]
  }]' \
  --resolution-techniques '{
    "resolutionType": "RULE_MATCHING",
    "ruleBasedProperties": {
      "rules": [
        {
          "ruleName": "exact-email",
          "matchingKeys": ["email"]
        },
        {
          "ruleName": "phone-name",
          "matchingKeys": ["phone", "name"]
        }
      ],
      "attributeMatchingModel": "MANY_TO_MANY"
    }
  }' \
  --role-arn arn:aws:iam::123456789:role/entity-resolution-role

# Run matching job
aws entityresolution start-matching-job --workflow-name "customer-dedup"

# ML-based matching (probabilistic)
aws entityresolution create-matching-workflow \
  --workflow-name "ml-customer-match" \
  --resolution-techniques '{
    "resolutionType": "ML_MATCHING",
    "mlMatchingResolution": {"uniqueId": "customer_id"}
  }' ...
```

---

### 🟡 Q72. What is Amazon Comprehend?
```python
# Comprehend: NLP service — sentiment, entities, key phrases, classification, PII

import boto3

comprehend = boto3.client("comprehend", region_name="us-east-1")

# Detect sentiment
response = comprehend.detect_sentiment(
    Text="The product arrived on time and exceeded my expectations! Absolutely love it.",
    LanguageCode="en"
)
print(f"Sentiment: {response['Sentiment']}")
# Sentiment: POSITIVE
# SentimentScore: {'Positive': 0.98, 'Negative': 0.001, 'Neutral': 0.01, 'Mixed': 0.009}

# Detect entities (NER)
response = comprehend.detect_entities(
    Text="Amazon announced AWS re:Invent 2026 will be held in Las Vegas on December 1.",
    LanguageCode="en"
)
for entity in response["Entities"]:
    print(f"{entity['Type']}: {entity['Text']} (confidence: {entity['Score']:.2%})")
# ORGANIZATION: Amazon (99.5%)
# EVENT: AWS re:Invent 2026 (95.2%)
# LOCATION: Las Vegas (98.7%)
# DATE: December 1 (97.3%)

# Detect PII
response = comprehend.detect_pii_entities(
    Text="Please contact John Smith at john.smith@example.com or call 555-123-4567",
    LanguageCode="en"
)
for entity in response["Entities"]:
    print(f"PII: {entity['Type']} at positions {entity['BeginOffset']}-{entity['EndOffset']}")

# Redact PII
response = comprehend.contains_pii_entities(
    Text="SSN: 123-45-6789, Credit Card: 4111-1111-1111-1111",
    LanguageCode="en"
)

# Custom classification (train your own classifier)
response = comprehend.create_document_classifier(
    DocumentClassifierName="support-ticket-classifier",
    DataAccessRoleArn="arn:aws:iam::123456789:role/comprehend-role",
    InputDataConfig={
        "DataFormat": "COMPREHEND_CSV",
        "S3Uri": "s3://my-training-data/tickets/",
        "LabelDelimiter": "|"
    },
    OutputDataConfig={"S3Uri": "s3://my-comprehend-output/"},
    LanguageCode="en",
    Mode="MULTI_CLASS"
)

# Classify document with custom model
response = comprehend.classify_document(
    Text="My order hasn't arrived after 2 weeks. This is unacceptable!",
    EndpointArn="arn:aws:comprehend:us-east-1:123456789:document-classifier-endpoint/my-endpoint"
)
print(response["Classes"])
# [{"Name": "SHIPPING_ISSUE", "Score": 0.95}, {"Name": "COMPLAINT", "Score": 0.89}]

# Key phrases
response = comprehend.detect_key_phrases(
    Text="Machine learning models need large amounts of high-quality training data.",
    LanguageCode="en"
)
print([kp["Text"] for kp in response["KeyPhrases"]])
# ['Machine learning models', 'large amounts', 'high-quality training data']

# Targeted sentiment (aspect-based)
response = comprehend.detect_targeted_sentiment(
    Text="The camera is amazing but the battery life is terrible.",
    LanguageCode="en"
)
for entity in response["Entities"]:
    print(f"{entity['Text']}: {entity['Mentions'][0]['Sentiment']}")
# camera: POSITIVE, battery life: NEGATIVE
```

---

### 🟡 Q73. What is Amazon Rekognition?
```python
# Rekognition: image and video analysis — faces, objects, text, content moderation

import boto3

rekognition = boto3.client("rekognition", region_name="us-east-1")

# Detect labels (objects, scenes)
with open("warehouse.jpg", "rb") as img:
    response = rekognition.detect_labels(
        Image={"Bytes": img.read()},
        MinConfidence=75,
        MaxLabels=20
    )
for label in response["Labels"]:
    print(f"{label['Name']}: {label['Confidence']:.1f}%")
    for instance in label.get("Instances", []):
        box = instance["BoundingBox"]
        print(f"  Location: top={box['Top']:.2f}, left={box['Left']:.2f}")

# Content moderation (unsafe content detection)
response = rekognition.detect_moderation_labels(
    Image={"S3Object": {"Bucket": "my-uploads", "Name": "user-photo.jpg"}},
    MinConfidence=60
)
for label in response["ModerationLabels"]:
    print(f"Moderation: {label['ParentName']} > {label['Name']} ({label['Confidence']:.1f}%)")
if response["ModerationLabels"]:
    block_content(image_id)

# Face detection + attributes
response = rekognition.detect_faces(
    Image={"S3Object": {"Bucket": "my-bucket", "Name": "team-photo.jpg"}},
    Attributes=["ALL"]
)
for face in response["FaceDetails"]:
    print(f"Face — Age: {face['AgeRange']['Low']}-{face['AgeRange']['High']}")
    print(f"       Emotion: {max(face['Emotions'], key=lambda e: e['Confidence'])['Type']}")
    print(f"       Smile: {face['Smile']['Value']}")

# Face search (find a person in a collection)
# Build face collection first
rekognition.create_collection(CollectionId="employees")

rekognition.index_faces(
    CollectionId="employees",
    Image={"S3Object": {"Bucket": "hr-photos", "Name": "alice.jpg"}},
    ExternalImageId="alice-employee-001",
    DetectionAttributes=["ALL"]
)

# Search for face in image
response = rekognition.search_faces_by_image(
    CollectionId="employees",
    Image={"S3Object": {"Bucket": "entrance-camera", "Name": "visitor.jpg"}},
    MaxFaces=3,
    FaceMatchThreshold=85
)
for match in response["FaceMatches"]:
    print(f"Match: {match['Face']['ExternalImageId']} ({match['Similarity']:.1f}%)")

# Text detection in images (receipts, signs, documents)
response = rekognition.detect_text(
    Image={"S3Object": {"Bucket": "receipts", "Name": "receipt.jpg"}}
)
text = " ".join([t["DetectedText"] for t in response["TextDetections"] if t["Type"] == "LINE"])
print(f"Receipt text: {text}")

# Video analysis (asynchronous)
response = rekognition.start_label_detection(
    Video={"S3Object": {"Bucket": "surveillance", "Name": "parking-lot.mp4"}},
    MinConfidence=70,
    NotificationChannel={
        "SNSTopicArn": "arn:aws:sns:us-east-1:123456789:rekognition-results",
        "RoleArn": "arn:aws:iam::123456789:role/rekognition-role"
    },
    JobTag="parking-lot-analysis"
)
job_id = response["JobId"]

# Get video results (after SNS notification)
response = rekognition.get_label_detection(JobId=job_id, SortBy="TIMESTAMP")
for label in response["Labels"]:
    if label["Label"]["Name"] in ["Car", "Person"]:
        print(f"@{label['Timestamp']}ms: {label['Label']['Name']}")
```

---

### 🟢 Q74. What is AWS Forecast?
```python
# Forecast: time-series forecasting using ML (no ML expertise required)
# Algorithms: CNN-QR, DeepAR+, NPTS, Prophet, ARIMA, ETS, AutoML

import boto3

forecast = boto3.client("forecast", region_name="us-east-1")
forecastquery = boto3.client("forecastquery", region_name="us-east-1")

# 1. Create dataset group
forecast.create_dataset_group(
    DatasetGroupName="retail-sales",
    Domain="RETAIL"
)

# 2. Create dataset
forecast.create_dataset(
    DatasetName="daily-sales",
    Domain="RETAIL",
    DatasetType="TARGET_TIME_SERIES",
    DataFrequency="D",   # daily
    Schema={
        "Attributes": [
            {"AttributeName": "timestamp",  "AttributeType": "timestamp"},
            {"AttributeName": "target_value","AttributeType": "float"},
            {"AttributeName": "item_id",    "AttributeType": "string"}
        ]
    }
)

# 3. Import historical data (from S3)
forecast.create_dataset_import_job(
    DatasetImportJobName="import-2023-2025",
    DatasetArn="arn:aws:forecast:us-east-1:123456789:dataset/daily-sales",
    DataSource={
        "S3Config": {
            "Path": "s3://my-forecast-data/sales/",
            "RoleArn": "arn:aws:iam::123456789:role/forecast-role"
        }
    },
    TimestampFormat="yyyy-MM-dd"
)

# 4. Create predictor (AutoML selects best algorithm)
forecast.create_auto_predictor(
    PredictorName="sales-predictor-v1",
    ForecastHorizon=30,          # forecast 30 days ahead
    ForecastFrequency="D",
    DataConfig={
        "DatasetGroupArn": "arn:aws:forecast:us-east-1:123456789:datasetgroup/retail-sales",
        "AttributeConfigs": [
            {
                "AttributeName": "target_value",
                "Transformations": {"filling": "middlefill"}
            }
        ]
    },
    ExplainabilityConfig={
        "Mode": "CreateExplainability"
    },
    OptimizationMetric="WAPE"    # Weighted Absolute Percentage Error
)

# 5. Create forecast
forecast.create_forecast(
    ForecastName="q3-2026-forecast",
    PredictorArn="arn:aws:forecast:us-east-1:123456789:predictor/sales-predictor-v1",
    ForecastTypes=["0.1", "0.5", "0.9"]   # 10th, 50th, 90th percentiles
)

# 6. Query forecast (for specific item)
response = forecastquery.query_forecast(
    ForecastArn="arn:aws:forecast:us-east-1:123456789:forecast/q3-2026-forecast",
    Filters={"item_id": "PROD-001"},
    StartDate="2026-07-01",
    EndDate="2026-07-31"
)
for ts in response["Forecast"]["Predictions"]["p50"]:
    print(f"{ts['Timestamp']}: {ts['Value']:.0f} units (predicted)")

# 7. Export forecast to S3 for bulk access
forecast.create_forecast_export_job(
    ForecastExportJobName="q3-forecast-export",
    ForecastArn="arn:aws:forecast:us-east-1:123456789:forecast/q3-2026-forecast",
    Destination={
        "S3Config": {
            "Path": "s3://my-forecast-output/q3/",
            "RoleArn": "arn:aws:iam::123456789:role/forecast-role"
        }
    }
)
```

---

## FINAL COMPLETE INDEX (All 130 Q&A)

| Q# | Question | Level | Part |
|----|---------|-------|------|
| **SECURITY (Q1–Q60)** | | | |
| Q1–Q25 | (Original 25 questions) | 🟢🟡 | Security |
| Q51 | GuardDuty — all data sources, finding categories, auto-remediation, org setup | 🟡 | Security |
| Q52 | S3 security — Block Public Access (account-level), Object Lock COMPLIANCE/GOVERNANCE, Bucket Key, S3 Replication | 🟡 | Security |
| Q53 | CloudTrail vs Config vs GuardDuty vs Security Hub — what each does, comparison table | 🟢 | Security |
| Q54 | EC2 IMDSv2 — enforce via SCP, Python usage, Nitro System guarantees, Nitro Enclaves | 🟡 | Security |
| Q55 | IAM Roles Anywhere — PKI-based auth for on-prem, Trust Anchor, Profile, Signing Helper | 🟡 | Security |
| Q56 | Amazon Verified Permissions — Cedar policies, schema, permit/forbid, is-authorized API | 🔴 | Security |
| Q57 | Zero Trust on AWS — 5 pillars, Verified Access, implementation checklist | 🟡 | Security |
| Q58 | S3 security best practices — complete 10-point checklist with CLI commands | 🟢 | Security |
| Q59 | Compliance on AWS — HIPAA (BAA), PCI DSS (12 requirements mapped), SOC 2 (5 criteria) | 🟡 | Security |
| Q60 | CloudTrail security — immutable trail, log validation, Object Lock, SCP protection, metric filters for root/MFA/network | 🟡 | Security |
| **MIGRATION (Q26–Q65)** | | | |
| Q26–Q40 | (Original 15 questions) | 🟢🟡 | Migration |
| Q61 | VMware Cloud on AWS (VMC) — HCX live migration, L2 extension, pricing, when to use | 🟡 | Migration |
| Q62 | AWS Outposts — Rack vs Server, deploy EC2/RDS/EKS, Local Gateway, use cases | 🟡 | Migration |
| Q63 | DNS migration with Route53 — TTL reduction, weighted routing for gradual shift, health checks, domain transfer | 🟡 | Migration |
| Q64 | Post-migration optimisation — right-sizing, RIs, Savings Plans, Spot, S3 Intelligent-Tiering, anomaly detection | 🟡 | Migration |
| Q65 | DataSync vs Storage Gateway vs Snow — comparison table | 🟢 | Migration |
| **ANALYTICS (Q41–Q74)** | | | |
| Q41–Q50 | (Original 10 questions) | 🟢🟡 | Analytics |
| Q66 | Redshift WLM, Concurrency Scaling, Redshift ML (CREATE MODEL with SageMaker) | 🟡 | Analytics |
| Q67 | Apache Iceberg on AWS — ACID operations, time travel, schema evolution, OPTIMIZE, VACUUM | 🔴 | Analytics |
| Q68 | AWS Glue Data Quality — DQDL rules (completeness, validity, uniqueness, statistical), fail ETL on low score | 🟡 | Analytics |
| Q69 | Kinesis Enhanced Fan-Out (EFO) — dedicated 2 MB/s, push model, KCL checkpointing | 🟡 | Analytics |
| Q70 | AWS Clean Rooms — collaboration, analysis rules (min 100 users), protected queries, Clean Rooms ML | 🔴 | Analytics |
| Q71 | AWS Entity Resolution — match records, schema mapping, rule-based + ML matching, MANY_TO_MANY | 🟡 | Analytics |
| Q72 | Amazon Comprehend — sentiment, NER, PII detection/redaction, custom classification, targeted sentiment | 🟡 | Analytics |
| Q73 | Amazon Rekognition — labels, content moderation, face search, text detection, video analysis | 🟡 | Analytics |
| Q74 | AWS Forecast — time-series ML, AutoML, percentile forecasts, query + export | 🟡 | Analytics |

---
*Total: 130 Q&A after gap-fill | Security (35) + Migration (20) + Analytics (24) | June 2026*
*🟢 15 Basic | 🟡 105 Intermediate | 🔴 10 Advanced*

---

# FINAL GAP-FILL — SECURITY (Q75–Q87)

---

### 🟡 Q75. What is VPC Flow Logs — enable, format, Athena analysis?
```bash
# VPC Flow Logs: capture network traffic metadata for VPCs, subnets, or ENIs

# Enable on VPC (recommended: send to S3 for Athena + cost)
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-12345678 \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flow-logs-bucket/vpc-logs/ \
  --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${log-status} ${vpc-id} ${subnet-id} ${instance-id} ${tcp-flags} ${type} ${pkt-srcaddr} ${pkt-dstaddr} ${region} ${az-id} ${sublocation-type} ${sublocation-id} ${pkt-src-aws-service} ${pkt-dst-aws-service} ${flow-direction} ${traffic-path}' \
  --max-aggregation-interval 60 \
  --tags Key=Purpose,Value=security-monitoring

# Enable on subnet or individual ENI
aws ec2 create-flow-logs \
  --resource-type NetworkInterface \
  --resource-ids eni-abc12345 \
  --traffic-type REJECT \
  --log-destination-type cloud-watch-logs \
  --log-destination arn:aws:logs:us-east-1:123456789:log-group:/vpc/flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::123456789:role/flowlogs-role

# Flow log fields (default format):
# version account-id interface-id srcaddr dstaddr srcport dstport
# protocol packets bytes start end action log-status

# Create Athena table for Flow Logs (partition by date)
aws athena start-query-execution \
  --query-string "
    CREATE EXTERNAL TABLE IF NOT EXISTS vpc_flow_logs (
      version      INT,
      account_id   STRING,
      interface_id STRING,
      srcaddr      STRING,
      dstaddr      STRING,
      srcport      INT,
      dstport      INT,
      protocol     BIGINT,
      packets      BIGINT,
      bytes        BIGINT,
      start        BIGINT,
      end          BIGINT,
      action       STRING,
      log_status   STRING,
      vpc_id       STRING,
      subnet_id    STRING,
      instance_id  STRING,
      tcp_flags    INT,
      flow_direction STRING,
      traffic_path INT
    )
    PARTITIONED BY (
      dt STRING
    )
    ROW FORMAT DELIMITED FIELDS TERMINATED BY ' '
    LOCATION 's3://my-flow-logs-bucket/vpc-logs/'
    TBLPROPERTIES ('skip.header.line.count'='1')
  " \
  --query-execution-context Database=security_db \
  --result-configuration OutputLocation=s3://my-athena-results/

# Athena security queries
aws athena start-query-execution \
  --query-string "
    -- Top talkers sending data OUT (data exfiltration check)
    SELECT srcaddr, dstaddr, SUM(bytes) AS total_bytes,
           COUNT(*) AS connection_count
    FROM vpc_flow_logs
    WHERE action = 'ACCEPT'
      AND flow_direction = 'egress'
      AND dt = '2026/06/13'
    GROUP BY srcaddr, dstaddr
    HAVING SUM(bytes) > 100000000    -- 100 MB
    ORDER BY total_bytes DESC
    LIMIT 20
  " ...

aws athena start-query-execution \
  --query-string "
    -- Port scan detection (many ports from same source)
    SELECT srcaddr, COUNT(DISTINCT dstport) AS distinct_ports,
           COUNT(*) AS attempts
    FROM vpc_flow_logs
    WHERE action = 'REJECT' AND dt = '2026/06/13'
    GROUP BY srcaddr
    HAVING COUNT(DISTINCT dstport) > 50
    ORDER BY distinct_ports DESC
  " ...

aws athena start-query-execution \
  --query-string "
    -- Unexpected SSH/RDP access
    SELECT srcaddr, dstaddr, dstport, action, packets
    FROM vpc_flow_logs
    WHERE dstport IN (22, 3389)
      AND action = 'ACCEPT'
      AND srcaddr NOT LIKE '10.%'
      AND srcaddr NOT LIKE '172.16.%'
      AND dt = '2026/06/13'
    ORDER BY packets DESC
  " ...
```

---

### 🟡 Q76. What is RDS and Aurora security?
```bash
# RDS security layers: encryption, IAM auth, SSL, security groups, parameter groups

# Create encrypted RDS instance (KMS)
aws rds create-db-instance \
  --db-instance-identifier my-secure-rds \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 16.2 \
  --master-username admin \
  --master-user-password "SecureP@ss2026!" \
  --allocated-storage 100 \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/abc123 \
  --vpc-security-group-ids sg-db \
  --db-subnet-group-name my-private-subnet-group \
  --no-publicly-accessible \
  --deletion-protection \
  --backup-retention-period 35 \
  --enable-iam-database-authentication \
  --enable-cloudwatch-logs-exports postgresql upgrade \
  --copy-tags-to-snapshot \
  --ca-certificate-identifier rds-ca-rsa4096-g1 \
  --tags Key=Environment,Value=production

# Enable IAM database authentication (no password — use tokens)
aws rds modify-db-instance \
  --db-instance-identifier my-secure-rds \
  --enable-iam-database-authentication \
  --apply-immediately

# IAM policy to allow RDS connect
aws iam create-policy --policy-name rds-iam-auth \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": "rds-db:connect",
      "Resource": "arn:aws:rds-db:us-east-1:123456789:dbuser:db-ABCDEFGHIJK/mydbuser"
    }]
  }'

# Generate IAM auth token (valid 15 minutes)
TOKEN=$(aws rds generate-db-auth-token \
  --hostname my-secure-rds.abc123.us-east-1.rds.amazonaws.com \
  --port 5432 \
  --region us-east-1 \
  --username mydbuser)

# Connect with token (no password)
PGPASSWORD=$TOKEN psql \
  "host=my-secure-rds.abc123.us-east-1.rds.amazonaws.com \
   user=mydbuser dbname=myapp sslmode=require \
   sslrootcert=./rds-ca-2019-root.pem"

# Python: RDS IAM auth
import boto3, psycopg2

def get_rds_connection():
    rds = boto3.client("rds", region_name="us-east-1")
    token = rds.generate_db_auth_token(
        DBHostname="my-secure-rds.abc123.us-east-1.rds.amazonaws.com",
        Port=5432,
        DBUsername="mydbuser"
    )
    return psycopg2.connect(
        host="my-secure-rds.abc123.us-east-1.rds.amazonaws.com",
        user="mydbuser",
        password=token,
        database="myapp",
        sslmode="require",
        sslrootcert="./rds-ca-rsa4096-g1.pem"
    )

# Enforce SSL (deny non-SSL connections) — via parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name my-ssl-params \
  --db-parameter-group-family postgres16 \
  --description "Force SSL"

aws rds modify-db-parameter-group \
  --db-parameter-group-name my-ssl-params \
  --parameters ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate

# Secrets Manager rotation for RDS (automatic password rotation)
aws secretsmanager create-secret \
  --name prod/rds/credentials \
  --secret-string '{"username":"admin","password":"P@ss!","host":"my-rds.abc.us-east-1.rds.amazonaws.com","port":5432,"dbname":"myapp"}'

aws secretsmanager rotate-secret \
  --secret-id prod/rds/credentials \
  --rotation-rules ScheduleExpression="rate(30 days)" \
  --rotate-immediately

# Aurora security extras:
# Aurora Serverless v2: automatically encrypts at rest
# Aurora Global Database: encrypted replication across regions
# Database Activity Streams: real-time audit to Kinesis (PCI, SOX)
aws rds start-activity-stream \
  --resource-arn arn:aws:rds:us-east-1:123456789:cluster:my-aurora-cluster \
  --mode sync \
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/abc123 \
  --apply-immediately
```

---

### 🟡 Q77. What is Lambda security?
```bash
# Lambda security best practices

# 1. Least-privilege execution role
aws iam create-role --role-name my-lambda-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'

# Minimal managed policy (only CloudWatch Logs)
aws iam attach-role-policy --role-name my-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Add specific permissions (NOT AdministratorAccess)
aws iam put-role-policy --role-name my-lambda-role \
  --policy-name s3-read-specific \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Action":["s3:GetObject"],
      "Resource":"arn:aws:s3:::my-specific-bucket/*",
      "Condition":{"StringEquals":{"s3:prefix":["processed/"]}}
    }]
  }'

# 2. Resource-based policy (who can invoke this Lambda)
aws lambda add-permission \
  --function-name my-function \
  --statement-id allow-only-my-api \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn arn:aws:execute-api:us-east-1:123456789:abc123/*/POST/orders \
  --source-account 123456789

# 3. Environment variable encryption
aws lambda create-function \
  --function-name my-secure-func \
  --runtime python3.12 \
  --role arn:aws:iam::123456789:role/my-lambda-role \
  --handler app.handler \
  --zip-file fileb://function.zip \
  --kms-key-arn arn:aws:kms:us-east-1:123456789:key/abc123 \
  --environment Variables={DB_HOST=mydb.cluster.rds.amazonaws.com,ENV=production} \
  --vpc-config SubnetIds=subnet-11111111,subnet-22222222,SecurityGroupIds=sg-lambda \
  --tracing-config Mode=Active \
  --reserved-concurrent-executions 100

# 4. VPC Lambda (access private resources)
aws lambda update-function-configuration \
  --function-name my-function \
  --vpc-config SubnetIds=subnet-private-1,subnet-private-2,SecurityGroupIds=sg-lambda

# 5. Function URL (restrict to specific IPs or Cognito)
aws lambda create-function-url-config \
  --function-name my-function \
  --auth-type AWS_IAM \   # AWS_IAM | NONE
  --cors '{
    "AllowOrigins": ["https://myapp.com"],
    "AllowMethods": ["POST"],
    "AllowHeaders": ["Authorization","Content-Type"],
    "MaxAge": 3600
  }'

# 6. Lambda Layers security (validate layer source)
# Only use layers from trusted publishers or your own account
aws lambda get-layer-version \
  --layer-name my-deps-layer --version-number 1 \
  --query "Content.CodeSha256"   # verify SHA256 matches expected

# 7. Code signing (ensure only trusted code runs)
aws signer put-signing-profile \
  --profile-name my-signing-profile \
  --platform-id AWSLambda-SHA384-ECDSA

aws lambda create-code-signing-config \
  --description "Require signed Lambda code" \
  --allowed-publishers SigningProfileVersionArns=arn:aws:signer:...:my-signing-profile/abc \
  --code-signing-policies UntrustedArtifactOnDeployment=Enforce

aws lambda put-function-code-signing-config \
  --function-name my-function \
  --code-signing-config-arn arn:aws:lambda:us-east-1:123456789:code-signing-config:csc-abc123
```

---

### 🟡 Q78. What is EKS security in depth?
```bash
# EKS security: RBAC, pod security, secrets encryption, network policies

# 1. RBAC (Kubernetes RBAC)
kubectl apply -f - <<'RBAC'
# ClusterRole: read-only for cluster-wide resources
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
rules:
- apiGroups: [""]
  resources: ["pods","services","nodes","namespaces"]
  verbs: ["get","list","watch"]
- apiGroups: ["apps"]
  resources: ["deployments","replicasets"]
  verbs: ["get","list","watch"]
---
# Role: namespace-specific full access
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-namespace-admin
  namespace: development
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
# Map IAM role to Kubernetes role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developer-cluster-reader
subjects:
- kind: Group
  name: "arn:aws:iam::123456789:role/developer-role"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-reader
  apiGroup: rbac.authorization.k8s.io
RBAC

# Map IAM users/roles to Kubernetes (aws-auth ConfigMap)
kubectl edit configmap aws-auth -n kube-system
# Or use eksctl:
eksctl create iamidentitymapping \
  --cluster my-eks-cluster \
  --region us-east-1 \
  --arn arn:aws:iam::123456789:role/developer-role \
  --group system:masters \         # or custom group
  --username developer

# 2. Secrets encryption with KMS
aws eks associate-encryption-config \
  --cluster-name my-eks-cluster \
  --encryption-config '[{
    "resources": ["secrets"],
    "provider": {"keyArn": "arn:aws:kms:us-east-1:123456789:key/abc123"}
  }]'

# 3. Pod Security Admission (enforce restricted policy)
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.29 \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Restricted policy requires:
# - No privileged containers
# - No hostNetwork/hostPID/hostIPC
# - No hostPath volumes
# - Run as non-root user
# - Drop ALL capabilities, add only needed ones
# - No privilege escalation

# 4. Network policies (deny all, allow specific)
kubectl apply -f - <<'NETPOL'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: production
spec:
  podSelector:
    matchLabels: {app: postgres}
  ingress:
  - from:
    - podSelector:
        matchLabels: {app: myapp}
    ports:
    - port: 5432
  policyTypes: [Ingress]
NETPOL

# 5. Falco (runtime security monitoring)
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set falco.grpc.enabled=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/..."

# Falco rules example (built-in):
# "A shell was spawned in a container"
# "Write below binary dir (/bin, /sbin)"
# "Container running as root"
# "Sensitive file opened for reading (/etc/passwd, /etc/shadow)"

# 6. GuardDuty EKS Runtime Monitoring
aws eks update-addon \
  --cluster-name my-eks-cluster \
  --addon-name aws-guardduty-agent \
  --addon-version v1.6.1-eksbuild.1

# 7. Secrets from AWS Secrets Manager (not K8s secrets)
# Use External Secrets Operator or CSI Driver
kubectl apply -f - <<'ESO'
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef: {name: aws-secrets-manager, kind: SecretStore}
  target: {name: db-credentials, creationPolicy: Owner}
  data:
  - secretKey: password
    remoteRef:
      key: prod/database/credentials
      property: password
EOI
ESO
```

---

### 🟡 Q79. What is AWS PrivateLink?
```bash
# PrivateLink: expose your service privately to other VPCs/accounts without VPC peering
# Components: NLB (you) + VPC Endpoint Service + Interface Endpoint (consumer)
# No data traverses internet, no route table changes needed

# Provider side: create NLB + Endpoint Service
aws elbv2 create-load-balancer \
  --name my-nlb \
  --type network \
  --scheme internal \
  --subnets subnet-11111111 subnet-22222222

aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns arn:aws:elasticloadbalancing:us-east-1:123456789:loadbalancer/net/my-nlb/abc123 \
  --acceptance-required \
  --private-dns-name my-service.mycompany.com \
  --tags Key=Service,Value=my-api

SERVICE_NAME=$(aws ec2 describe-vpc-endpoint-service-configurations \
  --query "ServiceConfigurations[0].ServiceName" --output text)

# Whitelist consumer accounts
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-abc123 \
  --add-allowed-principals arn:aws:iam::987654321:root

# Accept pending connections
aws ec2 accept-vpc-endpoint-connections \
  --service-id vpce-svc-abc123 \
  --vpc-endpoint-ids vpce-def456

# Consumer side: create Interface Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-endpoint-type Interface \
  --vpc-id vpc-consumer \
  --service-name com.amazonaws.vpce.us-east-1.vpce-svc-abc123 \
  --subnet-ids subnet-consumer \
  --security-group-ids sg-consumer \
  --private-dns-enabled

# Consumer now connects to my-service.mycompany.com → private, no internet
# PrivateLink vs VPC Peering vs Transit Gateway:
# PrivateLink: expose single service, unidirectional, no route conflicts
# VPC Peering: full VPC-to-VPC, bidirectional, CIDR must not overlap
# Transit Gateway: hub-and-spoke, many VPCs, supports on-prem
```

---

### 🟡 Q80. What is AWS Config Security Conformance Packs?
```bash
# Conformance Packs: deploy 50+ Config rules as one package

# Deploy CIS AWS Foundations Benchmark
aws configservice put-conformance-pack \
  --conformance-pack-name "CIS-AWS-Foundations-Benchmark" \
  --template-s3-uri s3://aws-configservice-us-east-1/conformance-packs-for-core-guardrails/Operational-Best-Practices-for-CIS-AWS-v1.4-Level2.yaml \
  --delivery-s3-bucket my-config-results \
  --delivery-s3-key-prefix conformance-packs/

# Deploy PCI DSS conformance pack
aws configservice put-conformance-pack \
  --conformance-pack-name "PCI-DSS-3-2-1" \
  --template-s3-uri s3://aws-configservice-us-east-1/conformance-packs-for-core-guardrails/Operational-Best-Practices-for-PCI-DSS-3-2-1.yaml \
  --delivery-s3-bucket my-config-results

# Check conformance pack compliance
aws configservice get-conformance-pack-compliance-summary \
  --conformance-pack-names "CIS-AWS-Foundations-Benchmark" \
  --query "ConformancePackComplianceSummaryList[0].{Pack:ConformancePackName,Compliant:ConformancePackComplianceSummary.COMPLIANT_COUNT,NonCompliant:ConformancePackComplianceSummary.NON_COMPLIANT_COUNT}"

# Non-compliant resources in pack
aws configservice get-conformance-pack-compliance-details \
  --conformance-pack-name "CIS-AWS-Foundations-Benchmark" \
  --filters ComplianceType=NON_COMPLIANT \
  --query "ConformancePackRuleComplianceList[].{Rule:ConfigRuleName,Resources:NonCompliantCount}" \
  --output table

# Key CIS rules covered:
# MFA on root:                    root-account-mfa-enabled
# No root access keys:            iam-root-access-key-check
# Password policy:                iam-password-policy
# CloudTrail enabled:             cloud-trail-enabled
# S3 block public access:         s3-bucket-public-access-prohibited
# EBS encryption default:         ec2-ebs-encryption-by-default
# VPC default SG no rules:        vpc-default-security-group-closed
# GuardDuty enabled:              guardduty-enabled-centralized
# Security Hub enabled:           securityhub-enabled
```


---

# FINAL GAP-FILL — MIGRATION (Q81–Q88)

---

### 🟡 Q81. How do you migrate MongoDB to Amazon DocumentDB?
```bash
# DocumentDB: MongoDB 3.6/4.0/5.0-compatible managed database
# Migration options: mongodump/mongorestore, DMS, live migration with change streams

# Step 1: Assess compatibility
# Check unsupported features: $lookup with pipeline, transactions (partial), GridFS
aws docdb describe-db-engine-versions \
  --engine docdb \
  --query "DBEngineVersions[].{Version:EngineVersion,Features:SupportedFeatureNames}"

# Step 2: Export from MongoDB (offline migration)
mongodump \
  --host source-mongodb.corp.local:27017 \
  --username admin --password "P@ss!" \
  --authenticationDatabase admin \
  --db myDatabase \
  --out /tmp/mongo-dump/ \
  --gzip --numParallelCollections 4

# Step 3: Create DocumentDB cluster
aws docdb create-db-cluster \
  --db-cluster-identifier my-docdb-cluster \
  --engine docdb \
  --engine-version 5.0.0 \
  --master-username admin \
  --master-user-password "DocDBAzureP@ss!" \
  --db-subnet-group-name my-docdb-subnet-group \
  --vpc-security-group-ids sg-docdb \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789:key/abc123 \
  --backup-retention-period 35 \
  --deletion-protection \
  --tags Key=Environment,Value=production

aws docdb create-db-instance \
  --db-instance-identifier my-docdb-instance-1 \
  --db-instance-class db.r6g.large \
  --engine docdb \
  --db-cluster-identifier my-docdb-cluster

# Step 4: Restore to DocumentDB
# Get TLS certificate
wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

mongorestore \
  --host my-docdb-cluster.cluster-abc.us-east-1.docdb.amazonaws.com:27017 \
  --ssl --sslCAFile global-bundle.pem \
  --username admin --password "DocDBAzureP@ss!" \
  --authenticationDatabase admin \
  --db myDatabase \
  /tmp/mongo-dump/myDatabase/ \
  --gzip --numParallelCollections 4

# Step 5: Online migration with DMS (minimal downtime)
aws dms create-endpoint \
  --endpoint-identifier source-mongodb \
  --endpoint-type source \
  --engine-name mongodb \
  --username admin --password "P@ss!" \
  --server-name source-mongodb.corp.local \
  --port 27017 \
  --mongodb-settings '{
    "AuthSource": "admin",
    "AuthMechanism": "DEFAULT",
    "NestingLevel": "NONE",
    "ExtractDocId": "false",
    "DocsToInvestigate": "1000",
    "AuthType": "PASSWORD"
  }'

aws dms create-endpoint \
  --endpoint-identifier target-documentdb \
  --endpoint-type target \
  --engine-name docdb \
  --username admin --password "DocDBAzureP@ss!" \
  --server-name my-docdb-cluster.cluster-abc.us-east-1.docdb.amazonaws.com \
  --port 27017 \
  --ssl-mode require \
  --certificate-arn arn:aws:dms:us-east-1:123456789:cert:abc123

aws dms create-replication-task \
  --replication-task-identifier mongo-to-docdb \
  --source-endpoint-arn arn:aws:dms:...:endpoint:source-mongodb \
  --target-endpoint-arn arn:aws:dms:...:endpoint:target-documentdb \
  --replication-instance-arn arn:aws:dms:...:rep:my-dms-instance \
  --migration-type full-load-and-cdc \
  --table-mappings '{
    "rules": [{"rule-type":"selection","rule-id":"1","rule-name":"all","object-locator":{"schema-name":"myDatabase","table-name":"%"},"rule-action":"include"}]
  }'
```

---

### 🟡 Q82. How do you migrate Windows workloads to AWS?
```bash
# Windows migration: BYOL, Dedicated Hosts, Windows AMIs, SQL Server

# Option 1: BYOL (Bring Your Own Licence) on Dedicated Hosts
# Requires: Software Assurance with Licence Mobility

# Create Dedicated Host (physical server for your licences)
aws ec2 allocate-hosts \
  --availability-zone us-east-1a \
  --instance-type m5.xlarge \
  --quantity 2 \
  --auto-placement on \
  --host-recovery on \
  --instance-family m5 \
  --tags Key=Purpose,Value=windows-byol

HOST_ID=$(aws ec2 describe-hosts --query "Hosts[0].HostId" --output text)

# Launch Windows instance on Dedicated Host
aws ec2 run-instances \
  --image-id ami-windows-2022 \     # AWS Windows Server 2022 AMI
  --instance-type m5.xlarge \
  --placement "{\"HostId\": \"$HOST_ID\", \"Tenancy\": \"host\"}" \
  --key-name my-windows-key \
  --security-group-ids sg-windows \
  --subnet-id subnet-private \
  --iam-instance-profile Name=ec2-profile \
  --user-data "
    <powershell>
    # Enable SSM Agent (usually pre-installed on AWS AMIs)
    Start-Service AmazonSSMAgent
    Set-Service AmazonSSMAgent -StartupType Automatic
    # Set hostname
    Rename-Computer -NewName 'WEB-01' -Restart
    </powershell>
  "

# Option 2: Licence-included AMIs (no BYOL needed)
aws ec2 describe-images \
  --filters \
    Name=name,Values="Windows_Server-2022-English-Full-Base-*" \
    Name=owner-alias,Values=amazon \
  --query "Images | sort_by(@, &CreationDate) | [-1].{Name:Name,Id:ImageId}"

# SQL Server migration (EC2 vs RDS)
# SQL Server on EC2: full control, SSRS, SSIS, Agent, CLR
# RDS for SQL Server: managed, limited features

aws rds create-db-instance \
  --db-instance-identifier my-sqlserver \
  --db-instance-class db.m5.xlarge \
  --engine sqlserver-ee \
  --engine-version 15.00.4365.2.v1 \
  --master-username admin \
  --master-user-password "SqlP@ss2026!" \
  --allocated-storage 200 \
  --storage-type gp3 --storage-encrypted \
  --license-model license-included \  # or bring-your-own-license
  --no-publicly-accessible \
  --multi-az \
  --backup-retention-period 35 \
  --enable-iam-database-authentication

# Windows Active Directory integration
aws ds create-microsoft-ad \
  --name corp.mycompany.com \
  --short-name CORP \
  --password "DirP@ss2026!" \
  --description "AWS Managed Microsoft AD" \
  --vpc-settings VpcId=vpc-12345678,SubnetIds=subnet-11111111,subnet-22222222

# Join EC2 to domain automatically via SSM
aws ssm send-command \
  --document-name AWS-JoinDirectoryServiceDomain \
  --instance-ids i-abc12345 \
  --parameters '{"directoryId":["d-abc123"],"directoryName":["corp.mycompany.com"],"dnsIpAddresses":["10.0.0.10","10.0.0.11"]}'
```

---

### 🟡 Q83. What are Application Modernisation patterns?
```bash
# Modernisation: move beyond lift-and-shift to cloud-native architecture

# ── Strangler Fig Pattern ─────────────────────────────────────────
# Gradually replace monolith components with microservices
# Router (API GW/ALB) directs traffic to old or new system

# Step 1: Route ALL traffic to monolith (current state)
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/app/my-alb/abc/listener/def \
  --conditions Field=path-pattern,Values="/*" \
  --actions Type=forward,TargetGroupArn=arn:...:targetgroup/monolith/abc \
  --priority 100

# Step 2: Carve out Orders service (new microservice on ECS)
aws elbv2 create-rule \
  --listener-arn arn:...listener/... \
  --conditions Field=path-pattern,Values="/api/orders/*" \
  --actions Type=forward,TargetGroupArn=arn:...:targetgroup/orders-service/abc \
  --priority 10   # lower number = higher priority

# Step 3: Gradually extract more routes (payments, inventory, users)
# Step 4: Monolith shrinks — eventually decommission

# ── Anti-Corruption Layer (ACL) ───────────────────────────────────
# Translate between old domain model and new cloud model
# Lambda as ACL translating legacy SOAP to REST

# ── Database decomposition ────────────────────────────────────────
# Monolith DB → per-service DB (event-driven sync)
# Orders DB:   DynamoDB (high throughput writes)
# Inventory DB: Aurora PostgreSQL (complex joins)
# Analytics DB: Redshift (reporting)

# ── CQRS (Command Query Responsibility Segregation) ───────────────
# Commands (writes) → DynamoDB
# Queries (reads)   → ElastiCache Redis or DynamoDB Accelerator (DAX)

# ── Event Sourcing ────────────────────────────────────────────────
# Every state change = event → Kinesis Data Streams → EventBridge
# Replay events to rebuild state at any point

# ── Saga Pattern (distributed transactions) ───────────────────────
# Step Functions choreography for multi-service transactions
aws stepfunctions create-state-machine \
  --name order-saga \
  --definition '{
    "Comment": "Order processing saga",
    "StartAt": "ReserveInventory",
    "States": {
      "ReserveInventory": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:::function:reserve-inventory",
        "Catch": [{"ErrorEquals":["States.ALL"],"Next":"CompensateInventory"}],
        "Next": "ChargePayment"
      },
      "ChargePayment": {
        "Type": "Task",
        "Resource": "arn:aws:lambda:::function:charge-payment",
        "Catch": [{"ErrorEquals":["States.ALL"],"Next":"RefundPayment"}],
        "Next": "ShipOrder"
      },
      "ShipOrder": {"Type":"Task","Resource":"arn:aws:lambda:::function:ship-order","End":true},
      "CompensateInventory": {"Type":"Task","Resource":"arn:aws:lambda:::function:release-inventory","End":true},
      "RefundPayment": {"Type":"Task","Resource":"arn:aws:lambda:::function:refund","Next":"CompensateInventory"}
    }
  }' \
  --role-arn arn:aws:iam::123456789:role/step-functions-role

# ── Containerisation of VMs ───────────────────────────────────────
# 1. Analyse app: stateless? external state? port conflicts?
# 2. Create Dockerfile
# 3. Test locally with docker-compose
# 4. Push to ECR, deploy to ECS Fargate
# 5. Replace ASG-based VM deployment
```

---

### 🟡 Q84. What is AWS Managed Microsoft AD vs AD Connector?
```bash
# Three directory options on AWS:
# Simple AD:           standalone Samba-based (no trusts, basic LDAP, small orgs)
# Managed Microsoft AD: full AD DS in cloud (trusts, GPO, MFA, schema extensions)
# AD Connector:        proxy — redirects to your on-prem AD (no users stored in AWS)

# ── Managed Microsoft AD ──────────────────────────────────────────
# Best for: cloud-only or hybrid (establish trust with on-prem AD)
aws ds create-microsoft-ad \
  --name aws.corp.mycompany.com \
  --short-name AWSCORP \
  --password "DirAdminP@ss!" \
  --edition Standard \      # Standard (up to 5k objects) | Enterprise (500k objects)
  --vpc-settings VpcId=vpc-12345678,SubnetIds=subnet-az1,subnet-az2

DIR_ID=$(aws ds describe-directories --query "DirectoryDescriptions[0].DirectoryId" --output text)

# Wait for directory to be Active
aws ds describe-directories \
  --directory-ids $DIR_ID \
  --query "DirectoryDescriptions[0].Stage"

# Create trust relationship with on-prem AD
aws ds create-trust \
  --directory-id $DIR_ID \
  --remote-domain-name corp.mycompany.com \
  --trust-password "TrustP@ss!" \
  --trust-direction Two-Way \
  --trust-type Forest \
  --conditional-forwarder-ip-addrs 192.168.1.10 192.168.1.11

# Enable MFA for Managed AD (with RADIUS server)
aws ds enable-radius \
  --directory-id $DIR_ID \
  --radius-settings '{
    "RadiusServers": ["10.0.1.5"],
    "RadiusPort": 1812,
    "RadiusTimeout": 20,
    "RadiusRetries": 0,
    "SharedSecret": "RadiusS3cret!",
    "AuthenticationProtocol": "MS-CHAPv2",
    "DisplayLabel": "MFA Code",
    "UseSameUsername": true
  }'

# ── AD Connector ─────────────────────────────────────────────────
# Best for: keep existing on-prem AD, no sync to AWS
# Use for: SSO to AWS services using on-prem credentials
aws ds create-ad \
  --name corp.mycompany.com \
  --short-name CORP \
  --password "ConnectorP@ss!" \
  --connect-settings '{
    "VpcId": "vpc-12345678",
    "SubnetIds": ["subnet-az1", "subnet-az2"],
    "CustomerDnsIps": ["192.168.1.10", "192.168.1.11"],
    "CustomerUserName": "AWSConnectorUser"
  }'

# AD Connector use cases:
# IAM Identity Center with existing AD
# Amazon WorkSpaces with AD authentication
# RDS join to domain
# EC2 domain join without syncing users to AWS

# Directory services comparison:
# Simple AD:    50-5000 users, basic LDAP, no trust, Samba-based, cheapest
# Managed AD:   5000-500000 users, full GPO, trust, schema ext, MFA, $144/mo Standard
# AD Connector: redirect to on-prem, no user sync, requires VPN/DX, $72/mo
```

---

# FINAL GAP-FILL — ANALYTICS (Q85–Q95)

---

### 🟡 Q85. What is Redshift Zero-ETL and Streaming Ingestion?
```sql
-- Zero-ETL: Aurora automatically replicates to Redshift — no ETL job needed
-- Near real-time analytics on operational data

-- Setup Zero-ETL (Aurora PostgreSQL → Redshift)
-- 1. Enable Zero-ETL in Aurora cluster
-- aws rds create-integration \
--   --source-arn arn:aws:rds:us-east-1:123456789:cluster:my-aurora-cluster \
--   --target-arn arn:aws:redshift:us-east-1:123456789:namespace/my-redshift-namespace \
--   --integration-name aurora-to-redshift \
--   --kms-key-id arn:aws:kms:us-east-1:123456789:key/abc123

-- 2. In Redshift, create database from integration
CREATE DATABASE aurora_replica FROM INTEGRATION '<integration-id>';

-- 3. Query Aurora data directly in Redshift (seconds latency)
SELECT o.order_id, o.amount, c.customer_name, c.email
FROM aurora_replica.public.orders o
JOIN aurora_replica.public.customers c ON o.customer_id = c.id
WHERE o.created_at > GETDATE() - INTERVAL '1 hour';

-- Zero-ETL supported sources (2026):
-- Aurora MySQL, Aurora PostgreSQL, RDS MySQL
-- DynamoDB → Redshift (new 2025)
-- Salesforce → Redshift (via AppFlow)

-- Streaming Ingestion (Kinesis → Redshift directly)
-- No S3, no COPY — data lands in seconds

-- Create materialized view from Kinesis stream
CREATE MATERIALIZED VIEW kinesis_mv AS
SELECT
    approximate_arrival_timestamp,
    JSON_PARSE(kinesis_data) AS data,
    partition_key
FROM kinesis.'my-event-stream'
PARTITION_KEY (partition_key);
-- Refresh rate: ~5 seconds

-- Query streaming data
SELECT
    data.event_type,
    COUNT(*) AS event_count,
    SUM(CAST(data.amount AS FLOAT)) AS total_amount
FROM kinesis_mv
WHERE approximate_arrival_timestamp > GETDATE() - INTERVAL '5 minutes'
GROUP BY data.event_type;

-- Stream from MSK (Kafka) to Redshift
CREATE MATERIALIZED VIEW kafka_mv AS
SELECT * FROM kafka.'my-kafka-cluster'.'orders-topic';
```

---

### 🟡 Q86. What is AWS Glue Streaming ETL?
```python
# Glue Streaming: process Kinesis or Kafka data continuously
# Micro-batch: process every 100 seconds (default)

# glue_streaming_job.py
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from pyspark.sql.functions import col, from_json, to_timestamp, window
from pyspark.sql.types import StructType, StructField, StringType, DoubleType

args = getResolvedOptions(sys.argv, ["JOB_NAME", "stream_arn", "output_path"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

# Read from Kinesis Data Stream
kinesis_options = {
    "streamARN": args["stream_arn"],
    "startingPosition": "LATEST",
    "classification": "json",
    "inferSchema": "true"
}

kinesis_df = glueContext.create_data_frame_from_options(
    connection_type="kinesis",
    connection_options=kinesis_options
)

# Schema for our events
schema = StructType([
    StructField("orderId",    StringType(), True),
    StructField("customerId", StringType(), True),
    StructField("amount",     DoubleType(), True),
    StructField("timestamp",  StringType(), True),
    StructField("status",     StringType(), True)
])

def process_batch(micro_batch_df, batch_id):
    if micro_batch_df.count() == 0:
        return

    # Parse JSON payload
    parsed_df = micro_batch_df.select(
        from_json(col("data"), schema).alias("event"),
        col("approximateArrivalTimestamp")
    ).select("event.*", "approximateArrivalTimestamp") \
     .withColumn("order_timestamp", to_timestamp(col("timestamp")))

    # Aggregate per 5-minute tumbling window
    windowed_df = parsed_df \
        .withWatermark("order_timestamp", "5 minutes") \
        .groupBy(
            window(col("order_timestamp"), "5 minutes"),
            col("status")
        ).agg(
            {"amount": "sum", "orderId": "count"}
        ).withColumnRenamed("sum(amount)", "total_revenue") \
         .withColumnRenamed("count(orderId)", "order_count")

    # Write to S3 (partitioned)
    windowed_df.write \
        .mode("append") \
        .partitionBy("window") \
        .parquet(args["output_path"])

    # Also write to DynamoDB for real-time dashboard
    parsed_df.write \
        .format("dynamodb") \
        .option("tableName", "streaming-metrics") \
        .mode("append") \
        .save()

# Execute streaming with checkpointing
glueContext.forEachBatch(
    frame=kinesis_df,
    batch_function=process_batch,
    options={
        "windowSize": "100 seconds",
        "checkpointLocation": "s3://my-checkpoints/streaming-etl/"
    }
)

job.commit()
```

```bash
# Create Glue streaming job
aws glue create-job \
  --name kinesis-streaming-etl \
  --role arn:aws:iam::123456789:role/glue-role \
  --command '{
    "Name": "gluestreaming",
    "ScriptLocation": "s3://my-scripts/glue_streaming_job.py",
    "PythonVersion": "3"
  }' \
  --default-arguments '{
    "--stream_arn": "arn:aws:kinesis:us-east-1:123456789:stream/my-stream",
    "--output_path": "s3://my-data-lake/streaming-output/",
    "--enable-continuous-cloudwatch-log": "true",
    "--enable-metrics": "",
    "--TempDir": "s3://my-temp/glue/"
  }' \
  --glue-version "4.0" \
  --worker-type G.1X \
  --number-of-workers 5
```

---

### 🟡 Q87. What is Apache Hudi on AWS?
```python
# Hudi: open table format for data lakes — enables upserts, deletes, incremental queries
# COW (Copy-on-Write): better read performance, higher write latency
# MOR (Merge-on-Read): better write performance, higher read latency

# Hudi with Glue/Spark (EMR)

from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.hudi.catalog.HoodieCatalog") \
    .config("spark.sql.extensions", "org.apache.spark.sql.hudi.HoodieSparkSessionExtension") \
    .getOrCreate()

# Write to Hudi (upsert)
orders_df.write.format("hudi") \
    .option("hoodie.table.name", "orders") \
    .option("hoodie.datasource.write.recordkey.field", "order_id") \
    .option("hoodie.datasource.write.partitionpath.field", "year,month") \
    .option("hoodie.datasource.write.table.type", "COPY_ON_WRITE") \
    .option("hoodie.datasource.write.operation", "upsert") \
    .option("hoodie.datasource.write.precombine.field", "updated_at") \
    .option("hoodie.upsert.shuffle.parallelism", 200) \
    .option("hoodie.cleaner.commits.retained", 5) \
    .option("hoodie.keep.max.commits", 10) \
    .option("hoodie.datasource.write.hive_style_partitioning", "true") \
    .mode("append") \
    .save("s3://my-data-lake/orders-hudi/")

# Incremental query (only new/changed records since last checkpoint)
spark.read.format("hudi") \
    .option("hoodie.datasource.query.type", "incremental") \
    .option("hoodie.datasource.read.begin.instanttime", "20260613000000") \
    .option("hoodie.datasource.read.end.instanttime", "20260614000000") \
    .load("s3://my-data-lake/orders-hudi/")

# Time travel query
spark.read.format("hudi") \
    .option("as.of.instant", "20260601000000") \
    .load("s3://my-data-lake/orders-hudi/")

# Delete records
from pyspark.sql.functions import lit

deletes_df = spark.createDataFrame([("ORD-001",), ("ORD-002",)], ["order_id"]) \
    .withColumn("updated_at", lit("2026-06-13"))

deletes_df.write.format("hudi") \
    .option("hoodie.datasource.write.operation", "delete") \
    .option("hoodie.datasource.write.recordkey.field", "order_id") \
    .option("hoodie.datasource.write.precombine.field", "updated_at") \
    .mode("append") \
    .save("s3://my-data-lake/orders-hudi/")

# Hudi vs Iceberg vs Delta — comparison
```

| Feature | Apache Hudi | Apache Iceberg | Delta Lake |
|---------|------------|----------------|------------|
| **Upserts** | ✅ Native (COW/MOR) | ✅ MERGE INTO | ✅ MERGE INTO |
| **Time Travel** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Schema Evolution** | Partial | ✅ Full | ✅ Full |
| **ACID Transactions** | ✅ Yes | ✅ Yes | ✅ Yes |
| **AWS Service Support** | EMR, Glue, Athena | Athena, EMR, Glue, Redshift | EMR, Glue |
| **Incremental Queries** | ✅ Native | ✅ Via metadata | ✅ Via logs |
| **Streaming Ingest** | ✅ Excellent | ✅ Good | ✅ Good |
| **Best for** | CDC, streaming upserts | Analytics, schema evolution | Databricks shops |

---

### 🟡 Q88. What is OpenSearch Index State Management (ISM) and alerting?
```bash
# ISM: automate index lifecycle (rollover → close → delete)
# Alerting: notify when metric thresholds are breached

# ISM Policy (rollover when index reaches 50GB or 30 days)
curl -X PUT "https://my-search-domain.us-east-1.es.amazonaws.com/_plugins/_ism/policies/log-rotation" \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "description": "Log rotation policy",
      "default_state": "hot",
      "states": [
        {
          "name": "hot",
          "actions": [{
            "rollover": {
              "min_size": "50gb",
              "min_index_age": "30d",
              "min_doc_count": 10000000
            }
          }],
          "transitions": [{"state_name": "warm"}]
        },
        {
          "name": "warm",
          "actions": [{
            "replica_count": {"number_of_replicas": 1}
          }],
          "transitions": [{
            "state_name": "cold",
            "conditions": {"min_index_age": "90d"}
          }]
        },
        {
          "name": "cold",
          "actions": [{
            "index_priority": {"priority": 1}
          }],
          "transitions": [{
            "state_name": "delete",
            "conditions": {"min_index_age": "365d"}
          }]
        },
        {
          "name": "delete",
          "actions": [{"delete": {}}],
          "transitions": []
        }
      ],
      "ism_template": [{
        "index_patterns": ["logs-*", "app-*"],
        "priority": 100
      }]
    }
  }'

# OpenSearch Alerting (notify on anomalies)
curl -X POST "https://my-search-domain.us-east-1.es.amazonaws.com/_plugins/_alerting/monitors" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "monitor",
    "name": "High Error Rate Monitor",
    "enabled": true,
    "schedule": {"period": {"interval": 1, "unit": "MINUTES"}},
    "inputs": [{
      "search": {
        "indices": ["application-logs-*"],
        "query": {
          "size": 0,
          "query": {
            "bool": {
              "filter": [
                {"range": {"@timestamp": {"gte": "now-1m"}}},
                {"term": {"log_level": "ERROR"}}
              ]
            }
          },
          "aggs": {
            "error_count": {"value_count": {"field": "_id"}}
          }
        }
      }
    }],
    "triggers": [{
      "name": "Error count exceeds threshold",
      "severity": "1",
      "condition": {
        "script": {
          "source": "ctx.results[0].aggregations.error_count.value > 100",
          "lang": "painless"
        }
      },
      "actions": [{
        "name": "Send Slack Alert",
        "destination_id": "<slack-destination-id>",
        "message_template": {
          "source": "High error rate detected: {{ctx.results.0.aggregations.error_count.value}} errors in last minute"
        }
      }]
    }]
  }'

# OpenSearch anomaly detection
curl -X POST "https://my-search-domain.us-east-1.es.amazonaws.com/_plugins/_anomaly_detection/detectors" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "response-time-anomaly",
    "description": "Detect anomalies in API response time",
    "time_field": "@timestamp",
    "indices": ["api-logs-*"],
    "detection_interval": {"period": {"interval": 5, "unit": "Minutes"}},
    "window_delay": {"period": {"interval": 1, "unit": "Minutes"}},
    "feature_attributes": [{
      "feature_name": "avg_response_time",
      "feature_enabled": true,
      "aggregation_query": {
        "avg_response_time": {"avg": {"field": "response_time_ms"}}
      }
    }]
  }'
```

---

### 🟡 Q89. What is Amazon Personalize?
```python
# Personalize: ML-based recommendation engine — no ML expertise required
# Use cases: product recommendations, content recommendations, personalised search

import boto3

personalize = boto3.client("personalize", region_name="us-east-1")
personalize_runtime = boto3.client("personalize-runtime", region_name="us-east-1")

# 1. Create dataset group
personalize.create_dataset_group(
    name="retail-recommendations",
    tags=[{"tagKey": "Environment", "tagValue": "production"}]
)

# 2. Create schema for interactions
personalize.create_schema(
    name="interactions-schema",
    schema=json.dumps({
        "type": "record",
        "name": "Interactions",
        "namespace": "com.amazonaws.personalize.schema",
        "fields": [
            {"name": "USER_ID",    "type": "string"},
            {"name": "ITEM_ID",    "type": "string"},
            {"name": "TIMESTAMP",  "type": "long"},
            {"name": "EVENT_TYPE", "type": "string"},   # click, purchase, view
            {"name": "EVENT_VALUE","type": "float"}
        ],
        "version": "1.0"
    })
)

# 3. Import historical interactions from S3
personalize.create_dataset_import_job(
    jobName="import-interactions",
    datasetArn="arn:aws:personalize:us-east-1:123456789:dataset/retail-recommendations/INTERACTIONS",
    dataSource={"dataLocation": "s3://my-personalize-data/interactions/"},
    roleArn="arn:aws:iam::123456789:role/personalize-role"
)

# 4. Create solution (train model)
personalize.create_solution(
    name="user-personalisation",
    datasetGroupArn="arn:aws:personalize:us-east-1:123456789:dataset-group/retail-recommendations",
    recipeArn="arn:aws:personalize:::recipe/aws-user-personalization-v2",   # USER_PERSONALIZATION
    solutionConfig={
        "autoMLConfig": {"metricName": "coverage"},
        "hpoConfig": {
            "hpoObjective": {"type": "maximize", "metricName": "train:hrAt10"},
            "hpoResourceConfig": {"maxNumberOfTrainingJobs": "10", "maxParallelTrainingJobs": "2"}
        }
    }
)

# 5. Create solution version (training run)
personalize.create_solution_version(
    solutionArn="arn:aws:personalize:us-east-1:123456789:solution/user-personalisation"
)

# 6. Create campaign (serving endpoint)
personalize.create_campaign(
    name="product-recommendations",
    solutionVersionArn="arn:aws:personalize:us-east-1:123456789:solution/user-personalisation/abc123",
    minProvisionedTPS=10,         # transactions per second
    campaignConfig={"itemExplorationConfig": {"explorationWeight": "0.3", "explorationItemAgeCutOff": "30"}}
)

# 7. Get personalised recommendations
response = personalize_runtime.get_recommendations(
    campaignArn="arn:aws:personalize:us-east-1:123456789:campaign/product-recommendations",
    userId="user-123",
    numResults=10,
    context={"DEVICE": "mobile", "DAYOFWEEK": "Monday"},
    filterArn="arn:aws:personalize:us-east-1:123456789:filter/exclude-purchased"
)
for item in response["itemList"]:
    print(f"Recommend: {item['itemId']} (score: {item.get('score', 'N/A')})")

# 8. Real-time event tracking (update model with live events)
personalize_events = boto3.client("personalize-events", region_name="us-east-1")
personalize_events.put_events(
    trackingId="abc123",
    userId="user-123",
    sessionId="session-456",
    eventList=[{
        "sentAt": datetime.utcnow(),
        "eventType": "click",
        "itemId": "PROD-789",
        "eventValue": 1.0,
        "properties": json.dumps({"numRatings": 0})
    }]
)

# Personalize recipes:
# USER_PERSONALIZATION:     "customers who bought X also bought Y" style
# POPULAR_ITEMS:            most popular items globally or by segment
# PERSONALIZED_RANKING:     re-rank a given list of items for each user
# RELATED_ITEMS:            similar items (item-to-item)
# USER_SEGMENTATION:        group users into segments for targeting
```

---

### 🟢 Q90. What is Amazon Textract?
```python
# Textract: extract text and structured data from documents
# vs Rekognition DetectText: simple text → Textract for structured forms
# vs Document Intelligence (Azure): Textract is the AWS equivalent

import boto3

textract = boto3.client("textract", region_name="us-east-1")

# Detect document text (simple OCR)
with open("document.pdf", "rb") as f:
    response = textract.detect_document_text(
        Document={"Bytes": f.read()}
    )

for block in response["Blocks"]:
    if block["BlockType"] == "LINE":
        print(f"Line: {block['Text']}")

# Analyse document (forms, tables, signatures)
with open("form.pdf", "rb") as f:
    response = textract.analyze_document(
        Document={"Bytes": f.read()},
        FeatureTypes=["TABLES", "FORMS", "SIGNATURES", "LAYOUT"]
    )

# Extract key-value pairs from forms
key_map, value_map, block_map = {}, {}, {}
for block in response["Blocks"]:
    block_map[block["Id"]] = block
    if block["BlockType"] == "KEY_VALUE_SET":
        if "KEY" in block.get("EntityTypes", []):
            key_map[block["Id"]] = block
        else:
            value_map[block["Id"]] = block

def get_text(result, blocks_map):
    text = ""
    if "Relationships" in result:
        for rel in result["Relationships"]:
            if rel["Type"] == "CHILD":
                for child_id in rel["Ids"]:
                    word = blocks_map[child_id]
                    if word["BlockType"] == "WORD":
                        text += word["Text"] + " "
    return text.strip()

for key_id, key_block in key_map.items():
    key_text = get_text(key_block, block_map)
    for rel in key_block.get("Relationships", []):
        if rel["Type"] == "VALUE":
            for value_id in rel["Ids"]:
                if value_id in value_map:
                    value_text = get_text(value_map[value_id], block_map)
                    print(f"  {key_text}: {value_text}")

# Extract tables
for block in response["Blocks"]:
    if block["BlockType"] == "TABLE":
        print("Found table")
        # Navigate cells to build table matrix

# Async analysis for large PDFs (multi-page)
response = textract.start_document_analysis(
    DocumentLocation={"S3Object": {"Bucket": "my-docs", "Name": "multi-page.pdf"}},
    FeatureTypes=["TABLES", "FORMS"],
    NotificationChannel={
        "SNSTopicArn": "arn:aws:sns:us-east-1:123456789:textract-done",
        "RoleArn": "arn:aws:iam::123456789:role/textract-role"
    },
    OutputConfig={"S3Bucket": "my-textract-output", "S3Prefix": "results/"}
)
job_id = response["JobId"]

# Get async results
response = textract.get_document_analysis(JobId=job_id)
while response["JobStatus"] == "IN_PROGRESS":
    time.sleep(5)
    response = textract.get_document_analysis(JobId=job_id)

# Textract + Comprehend pipeline (invoice → extract → classify)
# Textract extracts invoice fields → Comprehend classifies line items
```

---

## FINAL COMPLETE MASTER INDEX (All 130+ Q&A)

| Q# | Question | Level | Part |
|----|---------|-------|------|
| **SECURITY (Q1–Q80)** | | | |
| Q1–Q25 | Original 25 questions | 🟢🟡 | Security |
| Q51–Q60 | First gap-fill: GuardDuty, S3 security, IMDSv2, IAM Roles Anywhere, Verified Permissions, Zero Trust, compliance, CloudTrail | 🟡 | Security |
| Q75 | VPC Flow Logs — enable, format, Athena queries (port scan, data exfil, SSH) | 🟡 | Security |
| Q76 | RDS/Aurora security — IAM auth token, SSL enforce, Secrets Manager rotation, Activity Streams | 🟡 | Security |
| Q77 | Lambda security — execution role, resource policy, env encryption, VPC, code signing | 🟡 | Security |
| Q78 | EKS security — RBAC, aws-auth, secrets KMS, Pod Security Admission, Network Policy, Falco, External Secrets | 🔴 | Security |
| Q79 | AWS PrivateLink — NLB + Endpoint Service (provider), Interface Endpoint (consumer), vs VPC Peering | 🟡 | Security |
| Q80 | AWS Config Conformance Packs — CIS, PCI DSS, check compliance, key rules | 🟢 | Security |
| **MIGRATION (Q26–Q84)** | | | |
| Q26–Q40 | Original 15 questions | 🟢🟡 | Migration |
| Q61–Q65 | First gap-fill: VMC, Outposts, DNS migration, post-migration optimisation, tools comparison | 🟡 | Migration |
| Q81 | MongoDB → DocumentDB — mongodump/restore, DMS CDC, endpoint configuration | 🟡 | Migration |
| Q82 | Windows workloads — Dedicated Hosts BYOL, SQL Server on RDS, Managed AD domain join | 🟡 | Migration |
| Q83 | Application Modernisation — strangler fig, ACL, DB decomposition, CQRS, saga, containerisation | 🔴 | Migration |
| Q84 | AWS Managed Microsoft AD vs AD Connector vs Simple AD — trust, MFA/RADIUS, BYOL | 🟡 | Migration |
| **ANALYTICS (Q41–Q90)** | | | |
| Q41–Q50 | Original 10 questions | 🟢🟡 | Analytics |
| Q66–Q74 | First gap-fill: Redshift WLM/ML, Iceberg, Glue DQ, EFO, Clean Rooms, Entity Resolution, Comprehend, Rekognition, Forecast | 🟡🔴 | Analytics |
| Q85 | Redshift Zero-ETL (Aurora→Redshift) + Streaming Ingestion (Kinesis→Redshift) | 🟡 | Analytics |
| Q86 | AWS Glue Streaming ETL — Kinesis source, micro-batch, windowed aggregation, checkpointing | 🟡 | Analytics |
| Q87 | Apache Hudi on AWS — COW/MOR, upserts, incremental queries, deletes, Hudi vs Iceberg vs Delta | 🔴 | Analytics |
| Q88 | OpenSearch ISM (rollover→warm→cold→delete) + Alerting + Anomaly Detection | 🟡 | Analytics |
| Q89 | Amazon Personalize — dataset, interactions, solution, campaign, real-time events, recipes | 🟡 | Analytics |
| Q90 | Amazon Textract — OCR, forms/tables/signatures, async multi-page, pipeline with Comprehend | 🟢 | Analytics |

---
*FINAL TOTAL: 90 core Q&A fully documented*
*Security: 35 Q&A | Migration: 24 Q&A | Analytics: 31 Q&A*
*🟢 15 Basic | 🟡 60 Intermediate | 🔴 15 Advanced | June 2026*
