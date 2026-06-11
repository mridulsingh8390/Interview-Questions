# Terraform — Complete Study Guide
### All Topics · Detailed Q&A · Master Cheatsheet
**2025 Edition — Comprehensive coverage with examples, commands & cheatsheet**

---

## Table of Contents
- [Terraform Basics](#terraform-basics)
- [Providers & Resources](#providers-resources)
- [Variables, Locals & Outputs](#variables)
- [Data Sources](#data-sources)
- [State Management](#state-management)
- [Modules](#modules)
- [Expressions: for_each, count, dynamic blocks](#expressions)
- [Functions](#functions)
- [Workspaces](#workspaces)
- [Import & Moved Blocks](#import)
- [Terraform Cloud & Enterprise](#tfc)
- [Terragrunt](#terragrunt)
- [Testing: Terratest & terraform test](#testing)
- [CI/CD Integration](#cicd)
- [Drift Detection & Sentinel](#drift)
- [Master Cheatsheet](#master-cheatsheet)

---

## Terraform Basics

### 🟢 Q1. What is Terraform and what is Infrastructure as Code?

**Explanation:**
Terraform is HashiCorp's Infrastructure as Code (IaC) tool. It allows you to define cloud infrastructure in declarative HCL (HashiCorp Configuration Language) files, which Terraform then creates, updates, or destroys to match the desired state. Terraform supports 1000+ providers (AWS, GCP, Azure, Kubernetes, Datadog, etc.).

```bash
# Install
brew install terraform               # macOS
# Or download from https://releases.hashicorp.com/terraform/

# Verify
terraform version

# Initialize a new working directory
terraform init

# Format code
terraform fmt
terraform fmt -recursive            # Format all .tf files in subdirectories
terraform fmt -check                # Check without writing (CI mode)

# Validate syntax
terraform validate

# Plan — preview changes
terraform plan
terraform plan -out=tfplan           # Save plan to file
terraform plan -var="env=prod"
terraform plan -var-file="prod.tfvars"
terraform plan -target=aws_instance.web  # Plan specific resource

# Apply
terraform apply
terraform apply tfplan               # Apply saved plan
terraform apply -auto-approve        # Skip confirmation
terraform apply -parallelism=20      # Parallel operations (default 10)

# Destroy
terraform destroy
terraform destroy -target=aws_instance.web
terraform destroy -auto-approve

# State
terraform show                       # Show state or plan
terraform state list                 # List resources in state
terraform output                     # Show outputs
terraform output -json               # JSON format
```

---

### 🟢 Q2. What is the Terraform workflow in detail?

```
Write → Init → Plan → Apply → (Manage)

terraform init:
  - Downloads provider plugins
  - Initializes backend (remote state)
  - Installs modules

terraform plan:
  - Reads current state
  - Calls provider APIs to read actual infrastructure
  - Computes diff (what to create/update/destroy)
  - Shows execution plan (+/-/~)

terraform apply:
  - Executes the plan
  - Creates/updates/destroys resources
  - Updates state file

terraform destroy:
  - Plans and applies destruction of all resources
```

---

## Providers & Resources

### 🟡 Q3. How do you configure providers?

```hcl
# versions.tf — pin provider versions
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"      # Allow 5.x but not 6.x
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.25.0"
    }
    datadog = {
      source  = "DataDog/datadog"
      version = "~> 3.0"
    }
  }

  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# provider.tf
provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile       # Use named AWS profile

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
      Team        = var.team
    }
  }
}

# Multiple provider instances (alias)
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

# Resource using aliased provider
resource "aws_instance" "europe" {
  provider = aws.eu_west
  ami           = "ami-..."
  instance_type = "t3.micro"
}
```

---

### 🟡 Q4. How do resource meta-arguments work?

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  # ===== depends_on =====
  # Explicit dependency (Terraform usually infers automatically)
  depends_on = [aws_iam_role_policy.policy]

  # ===== lifecycle =====
  lifecycle {
    create_before_destroy = true    # Create new before destroying old
    prevent_destroy       = true    # Block accidental destruction
    ignore_changes        = [       # Ignore drift in these attributes
      tags,
      user_data,
    ]
    replace_triggered_by = [        # Replace when this changes
      aws_security_group.web.id
    ]
    precondition {
      condition     = var.instance_type != "t2.micro"
      error_message = "t2.micro is not allowed in production."
    }
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance did not get a public IP."
    }
  }

  # ===== count =====
  count = var.enable_web ? 3 : 0

  # ===== provisioner (last resort — avoid if possible) =====
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]
    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ip_list.txt"
  }
}
```

---

## Variables, Locals & Outputs

### 🟢 Q5. How do variables, locals, and outputs work in Terraform?

```hcl
# ===== VARIABLES =====
# variables.tf
variable "environment" {
  type        = string
  description = "Deployment environment"
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Must be dev, staging, or production."
  }
}

variable "instance_count" {
  type    = number
  default = 1
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 20
    error_message = "Instance count must be between 1 and 20."
  }
}

variable "allowed_cidr_blocks" {
  type    = list(string)
  default = ["10.0.0.0/8"]
}

variable "tags" {
  type    = map(string)
  default = {}
}

variable "database_config" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    multi_az       = bool
  })
  default = {
    engine         = "postgres"
    engine_version = "15.4"
    instance_class = "db.t3.micro"
    multi_az       = false
  }
}

variable "db_password" {
  type      = string
  sensitive = true    # Masked in output and state
}

# Provide values via:
# 1. terraform.tfvars (auto-loaded)
# 2. prod.tfvars (explicit: -var-file=prod.tfvars)
# 3. TF_VAR_environment env var
# 4. -var="environment=prod" CLI flag

# prod.tfvars
environment     = "production"
instance_count  = 5
db_password     = "..."
database_config = {
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r6g.large"
  multi_az       = true
}

# ===== LOCALS =====
# Computed values (reduce repetition)
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = merge(var.tags, {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  })

  # Conditional
  instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

  # Derived from data source
  vpc_cidr_block = data.aws_vpc.main.cidr_block

  # Used to toggle features
  enable_monitoring = contains(["staging", "production"], var.environment)
}

# ===== OUTPUTS =====
# outputs.tf
output "vpc_id" {
  description = "The VPC ID"
  value       = aws_vpc.main.id
}

output "load_balancer_dns" {
  description = "DNS name of the load balancer"
  value       = aws_lb.main.dns_name
}

output "database_endpoint" {
  description = "Database connection endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true    # Mask in output, but accessible via terraform output
}

output "instance_ips" {
  value = aws_instance.web[*].public_ip    # All IPs as a list
}
```

---

## Data Sources

### 🟡 Q6. How do data sources work in Terraform?

**Explanation:**
Data sources allow Terraform to read existing infrastructure state without managing it. They query provider APIs and expose attributes for use in resources. Essential for referencing existing VPCs, AMIs, certificates, secrets, etc.

```hcl
# Read existing VPC (not managed by this Terraform state)
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# Or by ID
data "aws_vpc" "by_id" {
  id = "vpc-12345678"
}

# Latest Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]    # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Use the AMI
resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
  ...
}

# Read secret from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/db/password"
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
  ...
}

# Read state from another Terraform workspace (terraform_remote_state)
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "production/vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.vpc.outputs.private_subnet_ids[0]
  ...
}

# Read K8s resources
data "kubernetes_namespace" "production" {
  metadata {
    name = "production"
  }
}

# DNS zone
data "aws_route53_zone" "main" {
  name         = "example.com."
  private_zone = false
}

# SSL Certificate
data "aws_acm_certificate" "main" {
  domain      = "*.example.com"
  statuses    = ["ISSUED"]
  most_recent = true
}
```

---

## State Management

### 🟡 Q7. How does Terraform state work and how do you manage remote state?

```hcl
# Backend configuration — remote state on S3
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "services/api/production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"    # For state locking
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/abc-def"
  }
}

# Create S3 + DynamoDB for state storage (bootstrap)
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-company-terraform-state"
  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "state_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

```bash
# State commands
terraform state list                          # List all resources
terraform state show aws_instance.web         # Show resource attributes
terraform state show 'module.vpc.aws_vpc.main'

# Move resource (rename without destroy/recreate)
terraform state mv aws_instance.old aws_instance.new
terraform state mv 'module.old_name' 'module.new_name'

# Remove resource from state (without destroying)
terraform state rm aws_instance.web
terraform state rm 'module.vpc'

# Pull and push state manually
terraform state pull > backup.tfstate
terraform state push fixed.tfstate

# List workspaces
terraform workspace list
terraform workspace new staging
terraform workspace select production
terraform workspace show

# Backend migration
terraform init -migrate-state    # Migrate local → remote or between backends

# Unlock state (if locked by crashed process)
terraform force-unlock <lock-id>

# Refresh state (sync state with real infrastructure)
terraform apply -refresh-only    # Modern way
terraform refresh                # Old way (deprecated)
```

---

## Modules

### 🟡 Q8. How do you create and use Terraform modules?

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = merge(var.tags, { Name = var.name })
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = { Name = "${var.name}-private-${count.index + 1}" }
}

# modules/vpc/variables.tf
variable "name" { type = string }
variable "cidr" { type = string }
variable "private_subnet_cidrs" { type = list(string) }
variable "tags" { type = map(string); default = {} }

# modules/vpc/outputs.tf
output "vpc_id" { value = aws_vpc.this.id }
output "private_subnet_ids" { value = aws_subnet.private[*].id }

# ===== CALLING A MODULE =====
# main.tf (root module)
module "vpc" {
  source  = "./modules/vpc"           # Local path
  # source = "terraform-aws-modules/vpc/aws"  # Terraform Registry
  # source = "git::https://github.com/org/tf-modules.git//vpc?ref=v2.0.0"  # Git
  # source = "github.com/org/tf-modules//vpc"

  version = "~> 3.0"                  # Only for registry modules

  name                 = "production-vpc"
  cidr                 = "10.0.0.0/16"
  private_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  tags                 = local.common_tags
}

# Access module outputs
resource "aws_instance" "app" {
  subnet_id = module.vpc.private_subnet_ids[0]
}
```

---

## Expressions

### 🟡 Q9. How do for_each, count, and dynamic blocks work?

```hcl
# ===== COUNT =====
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index}" }
}

# Reference: aws_instance.web[0], aws_instance.web[1], aws_instance.web[2]
# All IPs: aws_instance.web[*].public_ip

# ===== FOR_EACH (preferred over count for maps/sets) =====
variable "services" {
  default = {
    api     = { port = 8080, cpu = "256m", memory = "512Mi" }
    worker  = { port = 0,    cpu = "512m", memory = "1Gi" }
    cron    = { port = 0,    cpu = "100m", memory = "256Mi" }
  }
}

resource "kubernetes_deployment" "services" {
  for_each = var.services

  metadata {
    name = each.key                      # "api", "worker", "cron"
  }
  spec {
    template {
      spec {
        container {
          name   = each.key
          resources {
            requests = {
              cpu    = each.value.cpu
              memory = each.value.memory
            }
          }
        }
      }
    }
  }
}

# for_each with a set
resource "aws_security_group_rule" "ingress" {
  for_each    = toset(["80", "443", "8080"])
  type        = "ingress"
  from_port   = tonumber(each.value)
  to_port     = tonumber(each.value)
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
  security_group_id = aws_security_group.main.id
}

# ===== DYNAMIC BLOCKS =====
variable "ingress_rules" {
  default = [
    { port = 80,  cidr = "0.0.0.0/0",    description = "HTTP" },
    { port = 443, cidr = "0.0.0.0/0",    description = "HTTPS" },
    { port = 22,  cidr = "10.0.0.0/8",   description = "SSH internal" },
  ]
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# ===== FOR EXPRESSIONS =====
# Transform a list
locals {
  upper_names = [for name in var.names : upper(name)]
  long_names  = [for name in var.names : name if length(name) > 5]

  # List → map
  name_map    = { for name in var.names : name => upper(name) }

  # Map → filtered map
  prod_tags   = { for k, v in var.tags : k => v if k != "debug" }
}

# ===== CONDITIONAL EXPRESSIONS =====
locals {
  instance_type = var.environment == "production" ? "t3.large" : "t3.micro"
  subnet_ids    = var.use_private_subnets ? module.vpc.private_subnet_ids : module.vpc.public_subnet_ids
}
```

---

## Functions

### 🔴 Q10. What are the most important Terraform functions?

```hcl
# ===== STRING FUNCTIONS =====
locals {
  # Format
  name    = format("%s-%s", var.project, var.environment)
  padded  = formatlist("item-%02d", range(5))    # ["item-00","item-01",...]

  # Case
  upper   = upper("hello")            # HELLO
  lower   = lower("HELLO")            # hello

  # Split / join
  parts   = split(",", "a,b,c")       # ["a","b","c"]
  joined  = join("-", ["a", "b"])     # "a-b"

  # Trim / replace
  trimmed = trimspace("  hello  ")    # "hello"
  replaced = replace("hello world", " ", "-")  # "hello-world"
  stripped = trimprefix("tf-vpc", "tf-")       # "vpc"

  # Regex
  matches = regexall("[0-9]+", "abc123def456")  # ["123","456"]
  is_ip   = can(regex("^[0-9.]+$", var.ip))

  # String interpolation with template
  rendered = templatefile("${path.module}/user_data.tpl", {
    hostname = var.hostname
    packages = var.packages
  })
}

# ===== NUMERIC FUNCTIONS =====
locals {
  maxval  = max(1, 2, 3)            # 3
  minval  = min(1, 2, 3)            # 1
  floors  = floor(3.7)              # 3
  ceils   = ceil(3.2)               # 4
  absval  = abs(-5)                 # 5
  logval  = log(100, 10)            # 2
  powered = pow(2, 10)              # 1024
}

# ===== COLLECTION FUNCTIONS =====
locals {
  # List
  merged   = concat(["a", "b"], ["c", "d"])
  unique   = distinct(["a", "b", "a", "c"])      # ["a","b","c"]
  flat     = flatten([["a","b"], ["c","d"]])      # ["a","b","c","d"]
  sorted   = sort(["c", "a", "b"])
  reversed = reverse(["a", "b", "c"])
  sliced   = slice(["a","b","c","d"], 1, 3)       # ["b","c"]
  indexed  = element(["a","b","c"], 1)            # "b"
  length   = length(["a","b","c"])                # 3
  contains_val = contains(["a","b"], "a")         # true
  indexed_key  = index(["a","b","c"], "b")        # 1

  # Map
  keys_list   = keys({ a = 1, b = 2 })           # ["a","b"]
  values_list = values({ a = 1, b = 2 })          # [1,2]
  lookup_val  = lookup(var.map, "key", "default")
  merged_map  = merge({ a = 1 }, { b = 2 })      # { a=1, b=2 }
  has_key     = contains(keys(var.map), "mykey")

  # toset / tolist / tomap
  my_set    = toset(["a", "b", "a"])              # Set removes duplicates
  my_list   = tolist(["a", "b"])
}

# ===== TYPE / ENCODING FUNCTIONS =====
locals {
  jsonenc   = jsonencode({ key = "value" })
  jsondec   = jsondecode(var.json_string)
  yamlenc   = yamlencode({ key = "value" })
  yamldec   = yamldecode(var.yaml_string)
  b64enc    = base64encode("hello world")
  b64dec    = base64decode(var.encoded)
  sha256    = sha256("hello")
  md5val    = md5("content")
}

# ===== PATH FUNCTIONS =====
locals {
  module_path = path.module          # Module directory
  root_path   = path.root            # Root module directory
  cwd_path    = path.cwd             # Current working directory
  abs_path    = abspath("./files")
  dir_name    = dirname("/a/b/c.txt")    # /a/b
  base_name   = basename("/a/b/c.txt")   # c.txt
}

# ===== FILESYSTEM FUNCTIONS =====
locals {
  script_content = file("${path.module}/scripts/setup.sh")
  all_files      = fileset("${path.module}/policies", "*.json")
  file_sha256    = filesha256("${path.module}/files/config.zip")
  file_b64       = filebase64("${path.module}/files/binary.zip")
}
```

---

## Workspaces

### 🟡 Q11. How do Terraform workspaces work?

```hcl
# Workspace = isolated state file for same config
# Use for: multiple environments in same config

# workspace-aware config
locals {
  env_config = {
    dev = {
      instance_type = "t3.micro"
      replicas      = 1
    }
    staging = {
      instance_type = "t3.small"
      replicas      = 2
    }
    production = {
      instance_type = "t3.large"
      replicas      = 5
    }
  }

  config = local.env_config[terraform.workspace]
}

resource "aws_instance" "app" {
  instance_type = local.config.instance_type
  count         = local.config.replicas
}

# State files created per workspace:
# s3://bucket/key                   # default workspace
# s3://bucket/env:/staging/key      # staging workspace
# s3://bucket/env:/production/key   # production workspace
```

```bash
# Workspace commands
terraform workspace list
terraform workspace new staging
terraform workspace new production
terraform workspace select staging
terraform workspace show          # Show current
terraform workspace delete staging  # (must switch off it first)

# Use workspace in CI/CD
export WORKSPACE=staging
terraform workspace select $WORKSPACE || terraform workspace new $WORKSPACE
terraform plan -var-file="${WORKSPACE}.tfvars"
terraform apply -auto-approve
```

---

## Import & Moved Blocks

### 🟡 Q12. How do you import existing resources and use moved blocks?

```hcl
# ===== IMPORT BLOCK (Terraform 1.5+) =====
# Declarative import — add to config, then terraform plan/apply
import {
  to = aws_security_group.main
  id = "sg-12345678"
}

import {
  to = aws_instance.legacy
  id = "i-1234567890abcdef0"
}

# After import, resource config is auto-generated by:
terraform plan -generate-config-out=generated.tf

# Then move the generated config to your .tf files

# OLD WAY (still works)
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_s3_bucket.logs my-logs-bucket
terraform import 'module.vpc.aws_vpc.main' vpc-12345678
terraform import 'aws_security_group_rule.ingress["443"]' sg-id_ingress_tcp_443_443_0.0.0.0/0

# ===== MOVED BLOCK =====
# Rename a resource in state without destroy/recreate
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}

# Rename a module
moved {
  from = module.old_module
  to   = module.new_module
}

# Move resource into a module
moved {
  from = aws_security_group.main
  to   = module.security.aws_security_group.main
}

# Move from count to for_each
moved {
  from = aws_instance.web[0]
  to   = aws_instance.web["primary"]
}
```

---

## Terraform Cloud & Enterprise

### 🔴 Q13. How do you use Terraform Cloud and Enterprise?

```hcl
# Configure TFC backend
terraform {
  cloud {
    organization = "my-org"

    workspaces {
      name = "production-api"
      # Or use tags to dynamically select workspaces:
      # tags = ["production", "api"]
    }
  }
}
```

```bash
# Login to Terraform Cloud
terraform login

# Create workspace via CLI
terraform workspace new production-api

# TFC CLI workflow
terraform plan   # Runs in TFC, streams output locally
terraform apply  # Runs in TFC with approval (if configured)

# TFC API usage
curl -H "Authorization: Bearer $TFC_TOKEN" \
  https://app.terraform.io/api/v2/organizations/my-org/workspaces

# Trigger run via API
curl -X POST \
  -H "Authorization: Bearer $TFC_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  -d '{"data":{"type":"runs","attributes":{"message":"Triggered by CI"},"relationships":{"workspace":{"data":{"type":"workspaces","id":"ws-xxxxx"}}}}}' \
  https://app.terraform.io/api/v2/runs
```

```hcl
# ===== SENTINEL POLICIES (TFE/TFC+) =====
# policy.sentinel
import "tfplan/v2" as tfplan

# Deny any EC2 instance type not in allowed list
allowed_types = ["t3.micro", "t3.small", "t3.medium"]

deny_unapproved_instance_types = rule {
  all tfplan.resource_changes as _, changes {
    changes.type is "aws_instance" and
    changes.change.actions contains "create" implies
    changes.change.after.instance_type in allowed_types
  }
}

# Require tags on all AWS resources
required_tags = ["Environment", "ManagedBy", "Team"]

all_resources_tagged = rule {
  all tfplan.resource_changes as _, changes {
    changes.type matches "^aws_" and
    changes.change.actions contains "create" implies
    all required_tags as tag {
      tag in keys(changes.change.after.tags)
    }
  }
}

main = rule {
  deny_unapproved_instance_types and all_resources_tagged
}
```

---

## Terragrunt

### 🔴 Q14. What is Terragrunt and how does it make Terraform DRY?

```hcl
# Terragrunt wraps Terraform — adds DRY config inheritance, remote state auto-config,
# dependency management across modules

# Directory structure:
# infra/
# ├── terragrunt.hcl              # Root config (shared)
# ├── prod/
# │   ├── account.hcl
# │   ├── vpc/terragrunt.hcl
# │   ├── eks/terragrunt.hcl
# │   └── rds/terragrunt.hcl
# └── staging/
#     ├── account.hcl
#     ├── vpc/terragrunt.hcl
#     └── eks/terragrunt.hcl

# Root terragrunt.hcl — inherited by all
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "my-terraform-state"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.region}"
}
EOF
}

locals {
  account_vars    = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region          = local.account_vars.locals.region
  account_id      = local.account_vars.locals.account_id
}
```

```hcl
# prod/eks/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()   # Inherit root config
}

terraform {
  source = "git::https://github.com/terraform-aws-modules/terraform-aws-eks.git//?ref=v20.0.0"
}

dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = {
    vpc_id             = "vpc-00000000"
    private_subnets    = ["subnet-00000000"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  cluster_name    = "prod-eks"
  cluster_version = "1.29"
  vpc_id          = dependency.vpc.outputs.vpc_id
  subnet_ids      = dependency.vpc.outputs.private_subnets
}
```

```bash
# Terragrunt commands (mirror Terraform)
terragrunt plan
terragrunt apply
terragrunt destroy

# Run across ALL modules in directory tree
terragrunt run-all plan
terragrunt run-all apply --terragrunt-non-interactive

# Only changed modules
terragrunt run-all plan --terragrunt-include-module-prefix prod/eks

# Init all
terragrunt run-all init
```

---

## Testing

### 🔴 Q15. How do you test Terraform with Terratest and terraform test?

```go
// Terratest — Go-based integration testing
// tests/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestVpcModule(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "name":                 "test-vpc",
            "cidr":                 "10.99.0.0/16",
            "private_subnet_cidrs": []string{"10.99.1.0/24", "10.99.2.0/24"},
            "environment":          "test",
        },
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": "us-east-1",
        },
    })

    // Clean up after test
    defer terraform.Destroy(t, terraformOptions)

    // Deploy infrastructure
    terraform.InitAndApply(t, terraformOptions)

    // Get outputs
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    subnetIds := terraform.OutputList(t, terraformOptions, "private_subnet_ids")

    // Assertions
    assert.NotEmpty(t, vpcId)
    assert.Len(t, subnetIds, 2)

    // Verify with AWS SDK
    vpc := aws.GetVpcById(t, vpcId, "us-east-1")
    assert.Equal(t, "10.99.0.0/16", aws.GetTagValue(vpc.Tags, "CIDR"))
}
```

```hcl
# terraform test (native — Terraform 1.6+)
# tests/vpc.tftest.hcl

run "creates_vpc" {
  command = plan     # or apply

  variables {
    name = "test-vpc"
    cidr = "10.99.0.0/16"
  }

  assert {
    condition     = aws_vpc.this.cidr_block == "10.99.0.0/16"
    error_message = "VPC CIDR block should be 10.99.0.0/16"
  }

  assert {
    condition     = aws_vpc.this.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled"
  }
}

run "applies_correct_tags" {
  command = apply

  variables {
    name        = "test-vpc"
    environment = "test"
  }

  assert {
    condition     = aws_vpc.this.tags["Environment"] == "test"
    error_message = "Environment tag should be 'test'"
  }
}
```

```bash
# Run native tests
terraform test                      # Run all .tftest.hcl files
terraform test -filter=tests/vpc.tftest.hcl

# Run Terratest
cd tests && go test -v -timeout 30m ./...
go test -v -run TestVpcModule -timeout 20m .
```

---

## CI/CD Integration

### 🔴 Q16. How do you integrate Terraform in CI/CD?

```yaml
# GitHub Actions — Terraform CI/CD
name: Terraform

on:
  push:
    branches: [main]
    paths: ["infrastructure/**"]
  pull_request:
    paths: ["infrastructure/**"]

permissions:
  contents: read
  pull-requests: write
  id-token: write               # For OIDC auth to AWS

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infrastructure/production

    steps:
      - uses: actions/checkout@v4

      # Authenticate to AWS via OIDC (no long-lived keys)
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GithubActionsTerraform
          aws-region: us-east-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0
          terraform_wrapper: true

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: |
          terraform plan -no-color -out=tfplan \
            -var="environment=production" 2>&1 | tee plan_output.txt
          echo "exit_code=$?" >> $GITHUB_OUTPUT

      # Post plan as PR comment
      - name: Post Plan to PR
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('infrastructure/production/plan_output.txt', 'utf8');
            const maxLength = 65000;
            const truncated = plan.length > maxLength ? plan.substring(0, maxLength) + '\n...(truncated)' : plan;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Plan\n\`\`\`terraform\n${truncated}\n\`\`\``
            });

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

---

## Drift Detection

### 🔴 Q17. How do you detect and handle Terraform drift?

```bash
# Drift = real infrastructure differs from Terraform state

# Method 1: terraform plan (will show drift as changes)
terraform plan -detailed-exitcode
# Exit code 0 = no changes, 1 = error, 2 = changes present (drift!)

# Method 2: refresh-only plan (just sync state, don't make changes)
terraform plan -refresh-only
terraform apply -refresh-only      # Update state to match reality

# Method 3: Scheduled drift detection in CI
# .github/workflows/drift-detection.yml
name: Drift Detection
on:
  schedule:
    - cron: '0 8 * * 1-5'        # Every weekday at 8am

jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check for drift
        run: |
          terraform init
          terraform plan -detailed-exitcode -no-color 2>&1
        continue-on-error: true
      - name: Alert on drift
        if: steps.*.outcome == 'failure'
        run: |
          curl -X POST $SLACK_WEBHOOK -d \
            '{"text":"⚠️ Terraform drift detected in production!"}'

# Driftctl — dedicated drift detection tool
driftctl scan
driftctl scan --from tfstate://s3://my-state/prod/terraform.tfstate
driftctl gen-driftignore    # Generate .driftignore for accepted drift
```

---

## Master Cheatsheet

### Core Commands
```bash
terraform init                    # Initialize, download providers
terraform init -upgrade           # Update providers to latest allowed
terraform fmt -recursive          # Format all files
terraform validate                # Check syntax/config
terraform plan -out=tfplan        # Save plan
terraform apply tfplan            # Apply saved plan
terraform apply -auto-approve     # Skip confirmation
terraform destroy -auto-approve   # Destroy all
terraform output -json            # Show outputs as JSON
terraform state list              # List managed resources
terraform state show <resource>   # Show resource details
terraform state mv <src> <dst>    # Move/rename in state
terraform state rm <resource>     # Remove from state (don't destroy)
terraform import <resource> <id>  # Import existing resource
terraform force-unlock <lock-id>  # Release stuck lock
```

### HCL Quickref
```hcl
# variable types: string, number, bool, list, map, set, object, tuple, any
variable "x" { type = string; default = "y"; sensitive = true }
locals { name = "${var.prefix}-${var.env}" }
output "id" { value = resource.type.name.id; sensitive = true }
resource "type" "name" { attr = value; depends_on = [other.resource] }
data "type" "name" { filter { name = "x"; values = ["y"] } }
module "name" { source = "./path"; var1 = "value" }

# Meta-arguments
count = var.enabled ? 1 : 0
for_each = var.map
lifecycle { prevent_destroy = true; ignore_changes = [tags] }

# Expressions
[for x in list : upper(x)]
{for k, v in map : k => upper(v)}
condition ? true_val : false_val
coalesce(var.x, var.y, "default")
```

### Question Coverage Index
| Q# | Topic | Difficulty |
|---|---|---|
| Q1 | Introduction & CLI basics | 🟢 |
| Q2 | Workflow: init/plan/apply/destroy | 🟢 |
| Q3 | Provider configuration | 🟡 |
| Q4 | Resource meta-arguments & lifecycle | 🟡 |
| Q5 | Variables, locals, outputs | 🟢 |
| Q6 | Data sources & terraform_remote_state | 🟡 |
| Q7 | State management & remote backend | 🟡 |
| Q8 | Modules | 🟡 |
| Q9 | for_each, count, dynamic blocks, for expressions | 🟡 |
| Q10 | Built-in functions | 🔴 |
| Q11 | Workspaces | 🟡 |
| Q12 | Import & moved blocks | 🟡 |
| Q13 | Terraform Cloud, Enterprise & Sentinel | 🔴 |
| Q14 | Terragrunt | 🔴 |
| Q15 | Testing: Terratest & terraform test | 🔴 |
| Q16 | CI/CD integration | 🔴 |
| Q17 | Drift detection | 🔴 |
