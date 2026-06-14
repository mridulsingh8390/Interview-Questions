# Azure Resources — Complete Interview Q&A Guide
> **All possible questions | June 2026 | Microsoft Learn aligned**
> 🟢 Basic | 🟡 Intermediate | 🔴 Advanced | Full CLI + Bicep + KQL examples

---

## TABLE OF CONTENTS
| Part | Category | Questions |
|------|---------|----------|
| [1](#part-1--compute) | Compute | Q1–Q25 |
| [2](#part-2--networking) | Networking | Q26–Q50 |
| [3](#part-3--storage) | Storage | Q51–Q70 |
| [4](#part-4--databases) | Databases | Q71–Q95 |
| [5](#part-5--monitoring) | Monitoring | Q96–Q120 |

---

# PART 1 — COMPUTE

---

### 🟢 Q1. What is Azure Virtual Machines and what problem does it solve?
**Answer:**
Azure Virtual Machines (VMs) is an Infrastructure-as-a-Service (IaaS) offering providing on-demand, scalable compute resources with full OS-level control. You choose the OS, size, and software — Azure handles the physical hardware, networking, and data center operations.

**Key reasons to use Azure VMs:**
- Lift-and-shift on-premises workloads without code changes
- Custom OS kernels, drivers, or software not supported by PaaS
- Specific licensing requirements (SQL Server, Oracle, SAP)
- GPU workloads for ML training and graphics rendering
- Full control over networking stack (custom routing, firewall rules)

---

### 🟢 Q2. What VM series exist and when do you choose each?
| Series | Optimised For | Max vCPUs | Max RAM | Best Use Case |
|--------|-------------|----------|--------|--------------|
| **B** | Burstable (CPU credits) | 20 | 80 GB | Dev/test, low-traffic web |
| **D v5** | General purpose | 96 | 384 GB | Most production workloads |
| **E v5** | Memory optimised | 104 | 672 GB | In-memory databases, SAP |
| **F v2** | Compute optimised | 72 | 144 GB | Batch, gaming, analytics |
| **L v3** | Storage optimised (NVMe) | 80 | 640 GB | NoSQL, data warehousing |
| **M** | Large memory | 128 | 3892 GB | SAP HANA, large SQL |
| **N / NC / ND** | GPU (NVIDIA A100/H100) | 96 | 900 GB | ML training, inferencing |
| **H / HB / HC** | HPC + InfiniBand | 120 | 480 GB | CFD, genomics, MPI jobs |
| **DC (v5)** | Confidential (AMD SEV-SNP) | 96 | 384 GB | Healthcare, finance, sovereign |

```bash
# List all sizes available in a region
az vm list-sizes --location eastus --output table

# Create VM — full production example
az vm create \
  --resource-group myRG \
  --name myVM \
  --image Ubuntu2204 \
  --size Standard_D4s_v5 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --vnet-name myVNet \
  --subnet appSubnet \
  --nsg myNSG \
  --public-ip-sku Standard \
  --zone 1 \
  --os-disk-size-gb 128 \
  --storage-sku Premium_LRS \
  --data-disk-sizes-gb 256 512 \
  --output table
```

---

### 🟢 Q3. What is the difference between Stop and Deallocate?
**Answer:**
This is one of the most common interview questions on Azure billing.

| Action | Compute Freed | Billed | Public IP | State |
|--------|------------|-------|----------|-------|
| `az vm stop` | ❌ No | ✅ **Yes** | Retained | Stopped |
| `az vm deallocate` | ✅ Yes | ❌ **No** | Retained (if static) | Stopped (deallocated) |
| `az vm delete` | ✅ Yes | ❌ No | Released | Deleted |

```bash
az vm stop       -g myRG -n myVM   # OS shutdown — still charged for compute
az vm deallocate -g myRG -n myVM   # releases hardware — compute billing stops
az vm start      -g myRG -n myVM
az vm restart    -g myRG -n myVM
az vm delete     -g myRG -n myVM --yes

# Auto-shutdown (save cost for dev VMs)
az vm auto-shutdown -g myRG --vm-name myVM --time 1900 --email dev@company.com
```

---

### 🟢 Q4. What are VM availability options in Azure?
```bash
# 1. Single VM + Premium SSD → 99.9% SLA
# 2. Availability Set  → 99.95% SLA (2+ VMs in same datacenter)
# 3. Availability Zones → 99.99% SLA (2+ VMs across separate datacenters)

# Availability Set
az vm availability-set create -g myRG -n myAS \
  --platform-fault-domain-count 3 \
  --platform-update-domain-count 20

az vm create -g myRG -n myVM \
  --availability-set myAS \
  --image Ubuntu2204 --size Standard_D4s_v5 --generate-ssh-keys

# Availability Zones (recommended — higher SLA)
az vm create -g myRG -n myVMzone1 --zone 1 \
  --image Ubuntu2204 --size Standard_D4s_v5 --generate-ssh-keys
az vm create -g myRG -n myVMzone2 --zone 2 \
  --image Ubuntu2204 --size Standard_D4s_v5 --generate-ssh-keys
az vm create -g myRG -n myVMzone3 --zone 3 \
  --image Ubuntu2204 --size Standard_D4s_v5 --generate-ssh-keys

# Fault domain = separate rack (power + network isolated)
# Update domain = VMs patched one group at a time (max 20 UDs)
```

---

### 🟢 Q5. What are VM image types and what is Shared Image Gallery?
```bash
# Marketplace images
az vm image list --publisher Canonical --output table
az vm image list --publisher MicrosoftWindowsServer --sku 2022-Datacenter --output table

# Create custom golden image from your configured VM
az vm deallocate -g myRG -n myVM
az vm generalize  -g myRG -n myVM
az image create -g myRG -n myGoldenImage --source myVM \
  --hyper-v-generation V2

# Shared Image Gallery (distribute images globally with versioning)
az sig create -g myRG --gallery-name myGallery --location eastus

az sig image-definition create -g myRG \
  --gallery-name myGallery \
  --gallery-image-definition myAppImage \
  --publisher MyCompany --offer MyApp --sku Production \
  --os-type Linux --os-state Generalized \
  --hyper-v-generation V2 \
  --features SecurityType=TrustedLaunch

az sig image-version create -g myRG \
  --gallery-name myGallery \
  --gallery-image-definition myAppImage \
  --gallery-image-version 1.0.0 \
  --target-regions eastus=3 westeurope=1 southeastasia=1 \
  --managed-image /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/images/myGoldenImage

# Create VM from gallery
az vm create -g myRG -n myVM \
  --image /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/galleries/myGallery/images/myAppImage/versions/latest \
  --generate-ssh-keys
```

---

### 🟡 Q6. What are Spot VMs and when should you use them?
```bash
# Spot VMs use spare Azure capacity at up to 90% discount
# Azure can evict with 30-second notice when capacity is needed

az vm create -g myRG -n mySpotVM \
  --image Ubuntu2204 --size Standard_D8s_v5 \
  --priority Spot \
  --eviction-policy Deallocate \   # Deallocate | Delete
  --max-price 0.10 \               # max $/hr; -1 = never evict on price only
  --generate-ssh-keys

# Use cases:
# ✅ Batch rendering, simulations, genomics
# ✅ Stateless, restartable workloads
# ✅ ML training with checkpointing
# ✅ Dev/test environments out of business hours
# ❌ Production databases
# ❌ Anything requiring guaranteed uptime

# Handle eviction gracefully (check metadata endpoint)
# curl -H Metadata:true http://169.254.169.254/metadata/scheduledevents?api-version=2020-07-01
```

---

### 🟡 Q7. How do you resize a VM?
```bash
# Check available sizes in current cluster (no downtime needed)
az vm list-vm-resize-options -g myRG -n myVM --output table

# Resize (same hardware family — usually no downtime)
az vm resize -g myRG -n myVM --size Standard_D8s_v5

# If target size is not in current cluster, must deallocate first
az vm deallocate -g myRG -n myVM
az vm resize     -g myRG -n myVM --size Standard_E8s_v5
az vm start      -g myRG -n myVM

# Resize considerations:
# - Moving D→E (memory optimised) always requires deallocate
# - Moving to GPU (N-series) requires deallocate
# - Data disks are NOT affected by resize
# - Static public IP is retained; dynamic IP may change
```

---

### 🟡 Q8. What are VM Extensions?
```bash
# Extensions: small programs that configure VMs post-deployment

# Azure Monitor Agent (collect metrics and logs)
az vm extension set -g myRG --vm-name myVM \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --version 1.0 --enable-auto-upgrade true

# Custom Script Extension (run scripts at deploy time)
az vm extension set -g myRG --vm-name myVM \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --settings '{"commandToExecute":"apt-get update && apt-get install -y nginx && systemctl enable nginx"}'

# Domain join (Windows VMs to Active Directory)
az vm extension set -g myRG --vm-name myWinVM \
  --name JsonADDomainExtension \
  --publisher Microsoft.Compute \
  --version 1.3 \
  --settings '{"Name":"corp.contoso.com","OUPath":"OU=AzureVMs,DC=corp,DC=contoso,DC=com","User":"joindomain@corp.contoso.com","Restart":"true","Options":"3"}' \
  --protected-settings '{"Password":"DomainP@ss!"}'

# Key Vault certificate auto-refresh
az vm extension set -g myRG --vm-name myVM \
  --name KeyVaultForLinux \
  --publisher Microsoft.Azure.KeyVault \
  --settings '{"secretsManagementSettings":{"pollingIntervalInS":"3600","certificateStoreName":"MY","certificateStoreLocation":"/etc/ssl/certs","observedCertificates":["https://myKV.vault.azure.net/secrets/mySSLCert"]}}'

# List and remove
az vm extension list -g myRG --vm-name myVM --output table
az vm extension delete -g myRG --vm-name myVM --name CustomScript
```

---

### 🟡 Q9. What is Azure Dedicated Host?
```bash
# Dedicated Host: physical server reserved exclusively for your VMs
# Use for: compliance (no multi-tenant), software licensing (BYOL per physical core)

az vm host group create -g myRG -n myHostGroup \
  --platform-fault-domain-count 2 \
  --location eastus --zone 1

az vm host create -g myRG \
  --host-group myHostGroup \
  --name myDedicatedHost \
  --sku DSv3-Type1 \        # host SKU determines VM families allowed
  --platform-fault-domain 0 \
  --auto-replace-on-failure true

# Deploy VM to dedicated host
az vm create -g myRG -n myVM \
  --host /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/hostGroups/myHostGroup/hosts/myDedicatedHost \
  --image Ubuntu2204 --size Standard_D4s_v3 --generate-ssh-keys

# Benefits:
# - Isolated hardware (compliance: HIPAA, FedRAMP, ISO 27001)
# - No other customer VMs on same physical server
# - Maintenance control window
# - Azure Hybrid Benefit per physical host (not per VM)
```

---

### 🔴 Q10. What is Trusted Launch and Confidential VMs?
```bash
# Trusted Launch: Secure Boot + vTPM + Boot integrity monitoring
# Prevents: rootkits, bootkit malware, firmware attacks, kernel exploits
# Available: Gen 2 VMs — no extra cost

az vm create -g myRG -n myTrustedVM \
  --image Ubuntu2204 --size Standard_D4s_v5 \
  --security-type TrustedLaunch \
  --enable-secure-boot true \
  --enable-vtpm true \
  --generate-ssh-keys

# Confidential VM: hardware-encrypted memory (AMD SEV-SNP or Intel TDX)
# Even Azure operators CANNOT read VM memory — true hardware isolation
az vm create -g myRG -n myConfidentialVM \
  --image Ubuntu2204 \
  --size Standard_DC4as_v5 \         # DC-series required
  --security-type ConfidentialVM \
  --os-disk-security-encryption-type VMGuestStateOnly \
  --enable-secure-boot true \
  --enable-vtpm true \
  --generate-ssh-keys

# Use cases for Confidential VMs:
# Healthcare (PHI), Finance (PCI), Government (FedRAMP High)
# Multi-party compute (process data you shouldn't see)
# Sovereign clouds and data residency requirements
```

---

### 🟡 Q11. What are Virtual Machine Scale Sets (VMSS)?
```bash
# VMSS: group of identical VMs that auto-scale
# Orchestration modes:
# Flexible (recommended 2024+): mix VM sizes, supports AZs, standalone VMs too
# Uniform (legacy): all identical, max 1000 instances, limited features

az vmss create -g myRG -n myVMSS \
  --orchestration-mode Flexible \
  --platform-fault-domain-count 3 \
  --image Ubuntu2204 \
  --vm-sku Standard_D4s_v5 \
  --instance-count 3 \
  --zones 1 2 3 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --upgrade-policy-mode Rolling \
  --max-batch-instance-percent 20 \
  --max-unhealthy-instance-percent 20 \
  --pause-time PT5M \
  --enable-automatic-os-upgrade true \
  --enable-auto-repair true \
  --lb myLB --backend-pool-name myPool

# Autoscale configuration
az monitor autoscale create -g myRG \
  --resource myVMSS \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name myAutoscale --min-count 2 --max-count 50 --count 3

az monitor autoscale rule create -g myRG --autoscale-name myAutoscale \
  --condition "Percentage CPU > 75 avg 5m" --scale out 3

az monitor autoscale rule create -g myRG --autoscale-name myAutoscale \
  --condition "Percentage CPU < 25 avg 10m" --scale in 1

# Schedule-based profile (known peak hours)
az monitor autoscale profile create -g myRG --autoscale-name myAutoscale \
  --name BusinessHours --count 15 --min-count 5 --max-count 50 \
  --recurrence week mon tue wed thu fri --timezone "UTC" \
  --start "09:00" --end "18:00"

# Manual scale
az vmss scale -g myRG -n myVMSS --new-capacity 20
az vmss update-instances -g myRG -n myVMSS --instance-ids "*"
```

---

### 🟡 Q12. What is Compute Fleet?
```bash
# Compute Fleet: next-gen VMSS that mixes multiple VM sizes and Spot+On-demand
# Automatically selects cheapest/most-available SKU from your allowed list

az fleet create -g myRG -n myFleet --location eastus \
  --vm-attributes-min-vcpu-count 4 \
  --vm-attributes-max-vcpu-count 16 \
  --vm-attributes-memory-in-gb-min 16 \
  --base-capacity 5 \
  --spot-priority-alloc-strategy PriceCapacityOptimized

# Fleet selects from D4s_v5, D8s_v5, E4s_v5, etc. automatically
# Reduces Spot interruption by diversifying across multiple SKUs
```

---

### 🟢 Q13. What is Azure App Service and what are the plan tiers?
```bash
# App Service: PaaS for web apps, REST APIs, mobile backends
# Supported runtimes: .NET, Java, Node.js, Python, PHP, Ruby, containers

# Plan tiers (key interview topic):
# F1 Free:     60 CPU-min/day, shared infra, no SLA, no custom domain
# D1 Shared:   240 CPU-min/day, shared infra, no SLA
# B1-B3 Basic: dedicated, no autoscale, no slots, no daily backup
# S1-S3 Std:   autoscale, 5 slots, daily backup, custom domain + TLS, SLA 99.95%
# P0V3-P3V3:   faster VMs, 30 slots, VNet integration, private endpoint
# P1MV3-P3MV3: memory-optimised premium
# I1V2-I3V2:   Isolated v2 — dedicated VNet, ASE, 100 slots, highest isolation

# Create App Service Plan + Web App
az appservice plan create -g myRG -n myPlan \
  --sku P2V3 --is-linux --number-of-workers 3

az webapp create -g myRG --plan myPlan \
  -n myWebApp --runtime "PYTHON:3.12"

# App settings
az webapp config appsettings set -g myRG -n myWebApp \
  --settings DB_HOST=mydb.postgres.database.azure.com \
             ENVIRONMENT=production \
             WEBSITE_RUN_FROM_PACKAGE=1

# Deploy ZIP
az webapp deploy -g myRG -n myWebApp --src-path ./dist.zip --type zip

# Stream logs
az webapp log tail -g myRG -n myWebApp
```

---

### 🟡 Q14. What are App Service deployment slots?
```bash
# Slots: separate environments (staging, qa) within same App Service Plan
# SLA-backed blue-green deployments with zero downtime

# Create staging slot
az webapp deployment slot create -g myRG -n myWebApp --slot staging

# Deploy to staging (old production still live)
az webapp deploy -g myRG -n myWebApp --slot staging \
  --src-path ./v2.zip --type zip

# Swap staging → production (zero downtime)
az webapp deployment slot swap -g myRG -n myWebApp \
  --slot staging --target-slot production

# Sticky settings (DON'T swap with slot)
az webapp config appsettings set -g myRG -n myWebApp --slot staging \
  --slot-settings ENVIRONMENT=staging DATABASE_URL=staging-db

# Canary — send 10% traffic to staging before full swap
az webapp traffic-routing set -g myRG -n myWebApp \
  --distribution staging=10

# Auto-swap on deploy
az webapp deployment slot auto-swap -g myRG -n myWebApp \
  --slot staging --auto-swap-slot production

# Available slots:
# B1-B3: 0 slots
# S1-S3: 5 slots
# P0V3-P3V3: 30 slots
# I1V2-I3V2: 100 slots
```

---

### 🟡 Q15. What is VNet integration and Private Endpoint in App Service?
```bash
# VNet Integration (OUTBOUND): App → private resources in VNet
# Regional VNet Integration: app calls private DB, Redis, Service Bus
az webapp vnet-integration add -g myRG -n myWebApp \
  --vnet myVNet --subnet appSubnet

# Route ALL traffic through VNet (access on-premises via ER/VPN)
az webapp config appsettings set -g myRG -n myWebApp \
  --settings WEBSITE_VNET_ROUTE_ALL=1

# Private Endpoint (INBOUND): no public internet access to app
az network private-endpoint create -g myRG -n webAppPE \
  --vnet-name myVNet --subnet privateSubnet \
  --private-connection-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Web/sites/myWebApp \
  --group-id sites --connection-name webConn

# Disable public access (force private endpoint only)
az webapp update -g myRG -n myWebApp --set publicNetworkAccess=Disabled

# App Service Environment (ASE) v3 — for maximum isolation
# ASE deploys App Service INTO your VNet (both inbound and outbound are private)
az appservice ase create -g myRG -n myASEv3 \
  --vnet-name myVNet --subnet myASESubnet \
  --kind asev3 --os-preference Linux --zone-redundant true
```

---

### 🟡 Q16. What is Azure Functions and what are the hosting plans?
```bash
# Hosting plans comparison:
# Consumption:       pay-per-execution, scale-to-zero, cold start, 1.5GB RAM max
# Flex Consumption:  per-instance concurrency, faster cold start, VNet, 4GB RAM
# Premium (EP1-3):   pre-warmed instances, NO cold start, VNet, unlimited exec
# Dedicated:         always-on, predictable cost, shares App Service Plan

# Create Consumption Function App
az functionapp create -g myRG \
  --consumption-plan-location eastus \
  --runtime python --runtime-version 3.12 \
  --functions-version 4 -n myFuncApp \
  --storage-account mystorageaccount \
  --os-type linux \
  --assign-identity '[{"type":"SystemAssigned"}]'

# Create Premium Function App (no cold starts)
az functionapp plan create -g myRG -n myPremiumPlan \
  --location eastus --sku EP1 --is-linux --number-of-workers 1

az functionapp create -g myRG --plan myPremiumPlan \
  --runtime python --runtime-version 3.12 \
  --functions-version 4 -n myPremiumFunc \
  --storage-account mystorageaccount

# Deploy
func azure functionapp publish myFuncApp

# All trigger types:
# HTTP Trigger     → REST API, webhook
# Timer Trigger    → CRON schedule
# Blob Trigger     → new/modified blob in Storage
# Queue Trigger    → Azure Storage Queue message
# Service Bus      → SB queue or topic subscription
# Event Hub        → event stream (IoT, telemetry)
# Cosmos DB        → change feed
# Event Grid       → any routed event
# Durable Entities → stateful entities
```

```python
# Python v2 — all key triggers in one file
import azure.functions as func, logging, json

app = func.FunctionApp()

@app.route(route="orders", methods=["GET","POST"], auth_level=func.AuthLevel.FUNCTION)
def http_trigger(req: func.HttpRequest) -> func.HttpResponse:
    if req.method == "POST":
        body = req.get_json()
        return func.HttpResponse(json.dumps({"id": body["id"], "status": "received"}),
                                 status_code=201, mimetype="application/json")
    return func.HttpResponse("OK", status_code=200)

@app.timer_trigger(schedule="0 */5 * * * *", arg_name="timer", run_on_startup=False)
def timer_job(timer: func.TimerRequest) -> None:
    logging.info(f"Timer fired. Past due: {timer.past_due}")

@app.blob_trigger(arg_name="blob", path="uploads/{name}",
                  connection="AzureWebJobsStorage")
def blob_trigger(blob: func.InputStream) -> None:
    logging.info(f"Blob: {blob.name}, size: {blob.length}")

@app.service_bus_queue_trigger(
    arg_name="msg", queue_name="orders",
    connection="SERVICEBUS_CONNECTION_STRING")
def sb_trigger(msg: func.ServiceBusMessage) -> None:
    order = json.loads(msg.get_body().decode())
    logging.info(f"Order ID: {order['id']}")

@app.cosmos_db_trigger(
    arg_name="docs", database_name="myDB", container_name="orders",
    connection="COSMOS_CONNECTION", lease_container_name="leases",
    create_lease_container_if_not_exists=True)
def cosmos_trigger(docs: func.DocumentList) -> None:
    for doc in docs:
        logging.info(f"Changed: {doc}")
```

---

### 🟡 Q17. What is Azure Container Apps?
```bash
# Container Apps: serverless containers — Kubernetes + KEDA + Dapr — no cluster mgmt

# Create environment
az containerapp env create -g myRG -n myCAEnv --location eastus \
  --infrastructure-subnet-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/infraSubnet \
  --logs-workspace-id <la-id> \
  --logs-workspace-key <la-key>

# HTTP-scaled app
az containerapp create -g myRG -n myApp --environment myCAEnv \
  --image myacr.azurecr.io/myapp:latest \
  --registry-server myacr.azurecr.io --registry-identity system \
  --target-port 8080 --ingress external \
  --min-replicas 1 --max-replicas 30 \
  --cpu 0.5 --memory 1Gi \
  --scale-rule-name http --scale-rule-http-concurrency 20

# KEDA scale (Service Bus queue depth → 0 to 100 workers)
az containerapp create -g myRG -n myWorker --environment myCAEnv \
  --image myacr.azurecr.io/worker:latest \
  --min-replicas 0 --max-replicas 100 \
  --secrets sb-conn="Endpoint=sb://..." \
  --scale-rule-name sb --scale-rule-type azure-servicebus \
  --scale-rule-metadata queueName=orders namespace=mySBNS messageCount=5 \
  --scale-rule-auth connection=sb-conn

# Blue-green with traffic splitting
az containerapp update -g myRG -n myApp \
  --image myacr.azurecr.io/myapp:v2.0

az containerapp ingress traffic set -g myRG -n myApp \
  --revision-weight myApp--abc123=90 myApp--def456=10

# Scheduled jobs
az containerapp job create -g myRG -n nightly-job --environment myCAEnv \
  --trigger-type Schedule --cron-expression "0 2 * * *" \
  --image myacr.azurecr.io/batchjob:latest \
  --cpu 2 --memory 4Gi --replica-timeout 3600

# Dapr sidecar
az containerapp update -g myRG -n myApp \
  --enable-dapr --dapr-app-id myapp \
  --dapr-app-port 8080 --dapr-app-protocol http
```

---

### 🟡 Q18. What is Azure Container Instances (ACI)?
```bash
# ACI: fastest way to run containers — no infra management, billed per second

# Public container
az container create -g myRG -n myACI \
  --image myacr.azurecr.io/myapp:latest \
  --registry-login-server myacr.azurecr.io \
  --registry-username myacr \
  --registry-password <acr-password> \
  --cpu 2 --memory 4 \
  --ip-address Public --ports 8080 \
  --environment-variables ENV=prod \
  --secure-environment-variables DB_PASS=secret \
  --restart-policy OnFailure --location eastus

# Private (VNet-integrated) container
az container create -g myRG -n myPrivateACI \
  --image myacr.azurecr.io/myapp:latest \
  --cpu 2 --memory 4 \
  --vnet myVNet --subnet aciSubnet --ip-address Private

# Logs and exec
az container logs -g myRG -n myACI --follow
az container exec -g myRG -n myACI --exec-command "/bin/sh"

# Use cases: CI/CD ephemeral build agents, batch jobs, one-off scripts
```

---

### 🟢 Q19. What is Azure Kubernetes Service (AKS)?
```bash
# AKS: managed Kubernetes — Microsoft runs the control plane at no cost

az aks create -g myRG -n myAKS \
  --location eastus \
  --kubernetes-version 1.30 \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --zones 1 2 3 \
  --enable-managed-identity \
  --enable-workload-identity \
  --enable-oidc-issuer \
  --enable-private-cluster \
  --network-plugin azure \
  --network-policy azure \
  --enable-azure-rbac \
  --enable-defender \
  --auto-upgrade-channel patch \
  --enable-cluster-autoscaler \
  --min-count 2 --max-count 20 \
  --generate-ssh-keys

# Get credentials and verify
az aks get-credentials -g myRG -n myAKS
kubectl get nodes

# Add GPU node pool
az aks nodepool add -g myRG --cluster-name myAKS -n gpupool \
  --node-vm-size Standard_NC6s_v3 \
  --node-count 0 \
  --enable-cluster-autoscaler --min-count 0 --max-count 10 \
  --node-taints "sku=gpu:NoSchedule" \
  --labels "accelerator=nvidia"

# Upgrade
az aks upgrade -g myRG -n myAKS --kubernetes-version 1.31 --yes
```

---

### 🟢 Q20. What is Azure Container Registry (ACR)?
```bash
# ACR: private Docker registry with geo-replication, build tasks, scanning

# SKUs:
# Basic:    10 GB, no geo-replication, no private link
# Standard: 100 GB, private link
# Premium:  500 GB, geo-replication, zone redundancy, content trust, dedicated EP

az acr create -g myRG -n myACR --sku Premium \
  --zone-redundancy Enabled \
  --public-network-enabled false

# Build in cloud (no local Docker required)
az acr build --registry myACR --image myapp:v1.0 .

# ACR Tasks (auto-rebuild on Git push)
az acr task create --registry myACR -n buildTask \
  --image myapp:{{.Run.ID}} \
  --context https://github.com/org/repo#main \
  --file Dockerfile --git-access-token <token>

# Geo-replicate (images pulled from nearest replica)
az acr replication create --registry myACR --location westeurope
az acr replication create --registry myACR --location southeastasia

# Import from Docker Hub (private mirror / air-gap)
az acr import --name myACR \
  --source docker.io/library/nginx:latest --image nginx:latest

# Lifecycle policy (auto-delete old untagged images)
az acr config retention update --registry myACR \
  --status Enabled --days 30 --type UntaggedManifests

# Private endpoint
az network private-endpoint create -g myRG -n acrPE \
  --vnet-name myVNet --subnet appSubnet \
  --private-connection-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.ContainerRegistry/registries/myACR \
  --group-id registry --connection-name acrConn
```

---

### 🟡 Q21. What is Azure Batch?
```bash
# Batch: managed HPC and parallel job execution at scale

az batch account create -g myRG -n myBatchAcct \
  --location eastus --storage-account mystorageaccount
az batch account login -g myRG -n myBatchAcct --shared-key-auth

# Create compute pool (auto-scale with Spot VMs)
az batch pool create --id myPool --vm-size Standard_D8s_v5 \
  --image publisher=microsoft-azure-batch offer=ubuntu-server-container sku=20-04-lts \
  --node-agent-sku-id "batch.node.ubuntu 20.04" \
  --target-dedicated-nodes 0 --target-low-priority-nodes 0 \
  --task-slots-per-node 8 \
  --auto-scale-formula '
    $TargetLowPriorityNodes = min(50,
      $PendingTasks.GetSamplePercent(60 * TimeInterval_Second) >= 70
        ? avg($PendingTasks.GetSample(60 * TimeInterval_Second)) * 1.5
        : 0);
    $NodeDeallocationOption = taskcompletion;'

# Submit job and 1000 parallel tasks
az batch job create --id myJob --pool-id myPool
for i in $(seq 1 1000); do
  az batch task create --job-id myJob --task-id task-$i \
    --command-line "/bin/bash -c 'echo Processing item $i'"
done

# Use cases: ML hyperparameter search, rendering, genomics, simulations
```

---

### 🟡 Q22. What is the difference between Container Apps, ACI, AKS, and App Service for containers?
| Service | Management | Scale-to-zero | Cluster control | Use case |
|---------|----------|-------------|--------------|---------|
| **App Service** | PaaS | No (min 1) | No | Web apps, APIs |
| **ACI** | Serverless | Yes | No | One-off jobs, sidecars |
| **Container Apps** | Serverless K8s | Yes | No | Event-driven microservices |
| **AKS** | Managed K8s | No (manual) | Full | Complex, stateful, GPU |

---

### 🟢 Q23. What is Azure Hybrid Benefit?
```bash
# Use your on-premises Windows Server / SQL Server / RHEL / SLES licences in Azure
# Up to 40% savings on Windows VMs, up to 55% on SQL

# Apply to VM
az vm update -g myRG -n myWinVM \
  --license-type Windows_Server   # or RHEL_BYOS | SLES_BYOS | Windows_Client

# Apply to VMSS
az vmss update -g myRG -n myVMSS --license-type Windows_Server

# Apply to SQL DB
az sql db update -g myRG -s mysqlserver -n mydb --license-type BasePrice

# Apply to AKS nodes (Windows node pools)
az aks nodepool add -g myRG --cluster-name myAKS -n winnodes \
  --os-type Windows --node-vm-size Standard_D4s_v5 \
  --windows-admin-password "WinP@ss!" \
  --enable-ahb   # Azure Hybrid Benefit for Windows licences
```

---

### 🟡 Q24. What are VM run commands and serial console?
```bash
# Run Command: execute scripts on VM WITHOUT SSH/RDP — uses VM agent

# Linux
az vm run-command invoke -g myRG -n myVM \
  --command-id RunShellScript \
  --scripts "df -h && free -m && ps aux --sort=-%mem | head -10 && systemctl status nginx"

# Windows
az vm run-command invoke -g myRG -n myWinVM \
  --command-id RunPowerShellScript \
  --scripts "Get-Process | Sort-Object CPU -Descending | Select-Object -First 10"

# Upload and run script file
az vm run-command invoke -g myRG -n myVM \
  --command-id RunShellScript \
  --scripts @./fix-disk-permissions.sh

# Serial Console: browser-based emergency console
# Access VM even if network/SSH is broken
# Available in Portal: VM → Help → Serial Console
# Requirements: Boot diagnostics enabled + VM Agent installed

# Enable boot diagnostics
az vm boot-diagnostics enable -g myRG -n myVM \
  --storage https://mystorageaccount.blob.core.windows.net
```

---

### 🟡 Q25. What is Azure Managed Disks and what types are available?
```bash
# Managed Disks: block storage for VMs — Azure manages storage accounts automatically

# Types (by performance):
# Standard HDD:  max 2,000 IOPS / 500 MB/s — backup, dev/test
# Standard SSD:  max 6,000 IOPS / 750 MB/s — web servers, lightly used apps
# Premium SSD:   max 20,000 IOPS / 900 MB/s — production DBs, Tier-1 apps
# Premium SSD v2: max 80,000 IOPS / 1,200 MB/s — tunable, no disk size tiers
# Ultra Disk:    max 160,000 IOPS / 4,000 MB/s — SAP HANA, SQL critical

az disk create -g myRG -n myDataDisk --size-gb 512 \
  --sku Premium_LRS --zone 1

# Premium SSD v2 (tune IOPS and throughput independently)
az disk create -g myRG -n myPremV2Disk --size-gb 1024 \
  --sku PremiumV2_LRS --zone 1 \
  --disk-iops-read-write 40000 \
  --disk-mbps-read-write 600

az vm disk attach -g myRG --vm-name myVM --name myDataDisk --lun 0

# Snapshot (crash-consistent backup)
az snapshot create -g myRG -n mySnap \
  --source /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/disks/myDataDisk \
  --incremental true   # incremental = only changed blocks, much faster + cheaper

# Customer-managed key encryption
az disk-encryption-set create -g myRG -n myDES \
  --key-url https://myKV.vault.azure.net/keys/myDiskKey/version \
  --source-vault myKV \
  --encryption-type EncryptionAtRestWithCustomerKey

---

# PART 2 — NETWORKING

---

### 🟢 Q26. What is Azure Virtual Network (VNet)?
```bash
# VNet: private network in Azure. Resources communicate without internet.
# Address spaces: RFC 1918 (10.x, 172.16-31.x, 192.168.x)
# Reserved IPs per subnet: first 4 (.0–.3) + last 1 (.255) = 5 total

az network vnet create -g myRG -n myVNet \
  --address-prefix 10.0.0.0/16 --location eastus

# Subnets for different tiers
az network vnet subnet create -g myRG --vnet-name myVNet -n webSubnet      --address-prefix 10.0.1.0/24
az network vnet subnet create -g myRG --vnet-name myVNet -n appSubnet      --address-prefix 10.0.2.0/24
az network vnet subnet create -g myRG --vnet-name myVNet -n dataSubnet     --address-prefix 10.0.3.0/24
az network vnet subnet create -g myRG --vnet-name myVNet -n GatewaySubnet  --address-prefix 10.0.255.0/27  # required name
az network vnet subnet create -g myRG --vnet-name myVNet -n AzureBastionSubnet  --address-prefix 10.0.250.0/26  # required /26+
az network vnet subnet create -g myRG --vnet-name myVNet -n AzureFirewallSubnet --address-prefix 10.0.248.0/26  # required name

# Service Endpoints: route PaaS traffic over Azure backbone (no internet)
az network vnet subnet update -g myRG --vnet-name myVNet -n dataSubnet \
  --service-endpoints Microsoft.Storage Microsoft.Sql Microsoft.KeyVault \
                       Microsoft.ServiceBus Microsoft.EventHub

# Subnet delegation (for PaaS service injection into VNet)
az network vnet subnet update -g myRG --vnet-name myVNet -n appSubnet \
  --delegations Microsoft.Web/serverFarms     # App Service VNet integration
```

---

### 🟢 Q27. What is VNet Peering and what are its limitations?
```bash
# Peering connects VNets — low latency, Microsoft backbone, no gateway needed

# Hub-and-spoke topology
az network vnet peering create -g hubRG --name hub-to-spoke1 \
  --vnet-name hubVNet \
  --remote-vnet /subscriptions/<sub>/resourceGroups/spoke1RG/providers/Microsoft.Network/virtualNetworks/spoke1VNet \
  --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit

az network vnet peering create -g spoke1RG --name spoke1-to-hub \
  --vnet-name spoke1VNet \
  --remote-vnet /subscriptions/<sub>/resourceGroups/hubRG/providers/Microsoft.Network/virtualNetworks/hubVNet \
  --allow-vnet-access --use-remote-gateways

# Key peering LIMITATIONS (interview focus):
# ❌ Non-transitive: A↔B and B↔C does NOT mean A↔C (need separate A↔C peering)
# ❌ No overlapping address spaces allowed
# ✅ Cross-subscription peering supported
# ✅ Cross-region (global) peering supported
# ✅ Can span tenants (with permissions)

# Fix transitive routing: use Azure Firewall or NVA as hub router
# Or use Azure Virtual WAN (automatic transitive routing)
```

---

### 🟢 Q28. What are User-Defined Routes (UDR)?
```bash
# UDR: override Azure default routing to control traffic flow
# Force all egress through Firewall/NVA for inspection

az network route-table create -g myRG -n myRT --location eastus
az network route-table route create -g myRG --route-table-name myRT \
  -n defaultRoute --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.248.4    # Azure Firewall private IP

az network route-table route create -g myRG --route-table-name myRT \
  -n azureInfraRoute --address-prefix 168.63.129.16/32 \
  --next-hop-type Internet             # Azure platform health probe — must go direct

az network vnet subnet update -g myRG --vnet-name myVNet \
  -n appSubnet --route-table myRT

# Next-hop types:
# VirtualAppliance:  NVA / Azure Firewall
# VirtualNetworkGateway: VPN / ER Gateway
# VnetLocal:         default (no change)
# Internet:          direct to internet
# None:              drop traffic (blackhole)
```

---

### 🟢 Q29. What are NSGs and how do they work?
```bash
# NSG: stateful L3/L4 firewall applied to subnets or NICs
# Rules evaluated by priority (100–4096), lower = higher priority, first match wins
# Stateful: return traffic automatically allowed

az network nsg create -g myRG -n myNSG

# Inbound rules
az network nsg rule create -g myRG --nsg-name myNSG \
  -n Allow-HTTPS --priority 100 --direction Inbound --access Allow \
  --protocol Tcp --source-address-prefix Internet --destination-port-range 443

az network nsg rule create -g myRG --nsg-name myNSG \
  -n Allow-HTTP --priority 110 --direction Inbound --access Allow \
  --protocol Tcp --source-address-prefix Internet --destination-port-range 80

az network nsg rule create -g myRG --nsg-name myNSG \
  -n Allow-SSH-Bastion --priority 120 --direction Inbound --access Allow \
  --protocol Tcp --source-address-prefix 10.0.250.0/26 --destination-port-range 22

az network nsg rule create -g myRG --nsg-name myNSG \
  -n Deny-All --priority 4000 --direction Inbound --access Deny \
  --protocol "*" --source-address-prefix "*" --destination-port-range "*"

# Associate with subnet
az network vnet subnet update -g myRG --vnet-name myVNet \
  -n webSubnet --network-security-group myNSG

# Default rules (always present, cannot delete):
# AllowVNetInBound (65000): VNet to VNet — allow
# AllowAzureLoadBalancerInBound (65001): LB health probes — allow
# DenyAllInBound (65500): everything else — deny
```

---

### 🟡 Q30. What are Application Security Groups (ASG)?
```bash
# ASG: tag VMs logically; reference ASGs in NSG rules instead of IP ranges
# Benefit: no IP management needed; auto-updates as VMs join/leave groups

az network asg create -g myRG -n webTier
az network asg create -g myRG -n appTier
az network asg create -g myRG -n dbTier

# Assign VM NIC to ASG
az network nic ip-config update -g myRG --nic-name myWebVMNic \
  -n ipconfig1 --application-security-groups webTier

az network nic ip-config update -g myRG --nic-name myAppVMNic \
  -n ipconfig1 --application-security-groups appTier

# NSG rules using ASGs
az network nsg rule create -g myRG --nsg-name myNSG \
  -n Allow-Web-to-App --priority 200 \
  --source-asgs webTier --destination-asgs appTier \
  --destination-port-range 8080 --protocol Tcp --access Allow

az network nsg rule create -g myRG --nsg-name myNSG \
  -n Allow-App-to-DB --priority 210 \
  --source-asgs appTier --destination-asgs dbTier \
  --destination-port-range 5432 --protocol Tcp --access Allow

# NSG Flow Logs (traffic analytics in LA workspace)
az network watcher flow-log create -g myRG -n myFlowLog \
  --nsg myNSG --enabled true --log-version 2 --retention 30 \
  --storage-account /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage \
  --traffic-analytics true \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA
```

---

### 🟢 Q31. What is Azure Standard Load Balancer?
```bash
# L4 TCP/UDP load balancer — zone-redundant, SLA 99.99%, no public IP on backend VMs needed

az network public-ip create -g myRG -n myLBIP \
  --sku Standard --zone 1 2 3 --allocation-method Static

az network lb create -g myRG -n myLB --sku Standard \
  --public-ip-address myLBIP \
  --frontend-ip-name myFrontend \
  --backend-pool-name myBackendPool

# Health probe (HTTP/TCP/HTTPS)
az network lb probe create -g myRG --lb-name myLB \
  -n httpProbe --protocol Http --port 80 --path "/health" \
  --interval 5 --threshold 2

# Load balancing rule
az network lb rule create -g myRG --lb-name myLB \
  -n httpRule --protocol Tcp \
  --frontend-port 80 --backend-port 80 \
  --frontend-ip-name myFrontend \
  --backend-pool-name myBackendPool \
  --probe-name httpProbe \
  --idle-timeout 15 --enable-tcp-reset true \
  --load-distribution Default    # Default | SourceIP | SourceIPProtocol

# HA Ports rule — all ports / all protocols (for NVA load balancing)
az network lb rule create -g myRG --lb-name myInternalLB \
  -n haPortsRule --protocol All \
  --frontend-port 0 --backend-port 0 \
  --frontend-ip-name myFrontend \
  --backend-pool-name myPool --probe-name myProbe

# Outbound rule (explicit SNAT)
az network lb outbound-rule create -g myRG --lb-name myLB \
  -n outboundRule --protocol All \
  --frontend-ip-configs myFrontend \
  --address-pool myBackendPool \
  --idle-timeout 15 --outbound-ports 10000

# Internal LB (private VIP)
az network lb create -g myRG -n myInternalLB --sku Standard \
  --vnet-name myVNet --subnet appSubnet \
  --frontend-ip-name myPrivateFE \
  --private-ip-address 10.0.2.100 \
  --backend-pool-name myInternalPool
```

---

### 🟡 Q32. What is Application Gateway?
```bash
# L7 HTTP/S load balancer — WAF, URL routing, SSL termination, autoscale, affinity

az network application-gateway create -g myRG -n myAppGW \
  --sku WAF_v2 \
  --capacity 2 --min-capacity 2 --max-capacity 10 \
  --vnet-name myVNet --subnet appGWSubnet \
  --public-ip-address myAppGWIP \
  --frontend-port 443 \
  --cert-file mycert.pfx --cert-password "CertP@ss" \
  --zones 1 2 3

# WAF Policy
az network application-gateway waf-policy create -g myRG -n myWAFPolicy \
  --type OWASP --version 3.2 --request-body-check true

# Backend pools
az network application-gateway address-pool create -g myRG \
  --gateway-name myAppGW -n webPool --servers 10.0.2.10 10.0.2.11 10.0.2.12

az network application-gateway address-pool create -g myRG \
  --gateway-name myAppGW -n apiPool --servers myapi.azurewebsites.net

# HTTP settings
az network application-gateway http-settings create -g myRG \
  --gateway-name myAppGW -n webSettings \
  --port 8080 --protocol Http \
  --cookie-based-affinity Enabled --timeout 30

# Health probe
az network application-gateway probe create -g myRG --gateway-name myAppGW \
  -n myProbe --protocol Https --host-name-from-http-settings true \
  --path "/health" --interval 10 --timeout 10 --threshold 3

# URL path-based routing /api/* → apiPool, /* → webPool
az network application-gateway url-path-map create -g myRG \
  --gateway-name myAppGW -n myPathMap \
  --paths "/api/*" --address-pool apiPool --http-settings apiSettings \
  --default-address-pool webPool --default-http-settings webSettings

# Rewrite rules (add security headers)
az network application-gateway rewrite-rule set create -g myRG \
  --gateway-name myAppGW -n myRewriteSet

az network application-gateway rewrite-rule create -g myRG \
  --gateway-name myAppGW --rule-set-name myRewriteSet \
  -n secHeaders \
  --response-headers \
    "Strict-Transport-Security=max-age=31536000; includeSubDomains" \
    "X-Content-Type-Options=nosniff" \
    "X-Frame-Options=DENY" \
    "Content-Security-Policy=default-src 'self'"

# HTTP to HTTPS redirect
az network application-gateway redirect-config create -g myRG \
  --gateway-name myAppGW -n http2https \
  --type Permanent --target-listener httpsListener \
  --include-path true --include-query-string true

# WAF custom rules
az network application-gateway waf-policy custom-rule create -g myRG \
  --policy-name myWAFPolicy -n GeoBlock --priority 5 \
  --rule-type MatchRule --action Block \
  --match-conditions '[{"matchVariables":[{"variableName":"RemoteAddr"}],"operator":"GeoMatch","matchValues":["RU","CN","KP"]}]'
```

---

### 🟡 Q33. What is Azure Front Door?
```bash
# Front Door: global L7 HTTP/S load balancer at edge PoPs
# Features: WAF, caching, acceleration, health probes, failover

az afd profile create -g myRG --profile-name myFD \
  --sku Premium_AzureFrontDoor

az afd endpoint create -g myRG --profile-name myFD \
  --endpoint-name myEP --enabled-state Enabled

az afd origin-group create -g myRG --profile-name myFD \
  --origin-group-name myOrigins \
  --probe-request-type HEAD --probe-protocol Https \
  --probe-interval-in-seconds 30 --probe-path "/health" \
  --sample-size 4 --successful-samples-required 3

az afd origin create -g myRG --profile-name myFD \
  --origin-group-name myOrigins --origin-name eastus \
  --host-name myapp-eastus.azurewebsites.net \
  --https-port 443 --priority 1 --weight 1000

az afd origin create -g myRG --profile-name myFD \
  --origin-group-name myOrigins --origin-name westeu \
  --host-name myapp-westeu.azurewebsites.net \
  --https-port 443 --priority 2 --weight 1000

az afd route create -g myRG --profile-name myFD \
  --endpoint-name myEP --origin-group myOrigins \
  --route-name myRoute --patterns-to-match "/*" \
  --forwarding-protocol MatchRequest --https-redirect Enabled

# WAF + Bot protection
az network front-door waf-policy create -g myRG -n myAFDWAF \
  --sku Premium_AzureFrontDoor --mode Prevention

az network front-door waf-policy managed-rules add -g myRG \
  --policy-name myAFDWAF \
  --type Microsoft_DefaultRuleSet --version 2.1

az network front-door waf-policy managed-rules add -g myRG \
  --policy-name myAFDWAF \
  --type Microsoft_BotManagerRuleSet --version 1.1

# Front Door vs Traffic Manager:
# Front Door: L7 proxy (sees HTTP headers, can WAF, cache, SSL terminate)
# Traffic Manager: DNS-based only (L3), no WAF, no caching, no SSL
```

---

### 🟡 Q34. What is Azure Traffic Manager?
```bash
# Traffic Manager: DNS-based global routing — NOT a proxy, NO WAF, NO caching
# Routing methods:
# Priority:     primary → secondary on failure
# Weighted:     split % across endpoints
# Performance:  route to lowest latency endpoint per user location
# Geographic:   users in EU → EU endpoint, US → US endpoint
# Multivalue:   return multiple healthy endpoints (client-side LB)
# Subnet:       route based on source IP subnet ranges

az network traffic-manager profile create -g myRG -n myTM \
  --routing-method Performance \
  --unique-dns-name mytm-unique-dns \
  --ttl 30 --protocol HTTPS --port 443 --path "/health"

az network traffic-manager endpoint create -g myRG --profile-name myTM \
  -n eastus-ep --type azureEndpoints \
  --target-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Web/sites/myapp-eastus \
  --priority 1 --weight 100

az network traffic-manager endpoint create -g myRG --profile-name myTM \
  -n westeu-ep --type azureEndpoints \
  --target-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Web/sites/myapp-westeu \
  --priority 2 --weight 100

az network traffic-manager endpoint create -g myRG --profile-name myTM \
  -n external-ep --type externalEndpoints \
  --target aws-backup.example.com \
  --priority 3 --endpoint-location eastus
```

---

### 🟡 Q35. What is Azure VPN Gateway?
```bash
# VPN Gateway: encrypted IPsec/IKEv2 tunnel over public internet

az network public-ip create -g myRG -n myVPNIP1 --sku Standard --zone 1 2 3
az network public-ip create -g myRG -n myVPNIP2 --sku Standard --zone 1 2 3

az network vnet-gateway create -g myRG -n myVPNGW \
  --public-ip-address myVPNIP1 myVPNIP2 \    # active-active for HA
  --vnet myVNet --gateway-type Vpn \
  --vpn-type RouteBased --sku VpnGw2AZ --no-wait

# SKUs and throughput:
# VpnGw1: 650 Mbps | VpnGw2: 1 Gbps | VpnGw3: 1.25 Gbps
# VpnGw4: 5 Gbps  | VpnGw5: 10 Gbps | AZ suffix = zone-redundant

# Site-to-Site (office to Azure)
az network local-gateway create -g myRG -n myOnPremGW \
  --gateway-ip-address 203.0.113.1 \
  --local-address-prefixes 192.168.0.0/16 10.10.0.0/24

az network vpn-connection create -g myRG -n myS2SConn \
  --vnet-gateway1 myVPNGW --local-gateway2 myOnPremGW \
  --shared-key "MySecretPreSharedKey!" --enable-bgp true

# Point-to-Site (developer laptop to Azure)
az network vnet-gateway update -g myRG -n myVPNGW \
  --client-protocol OpenVPN IkeV2 \
  --address-prefixes 172.16.0.0/24 \
  --vpn-auth-type AAD \
  --aad-tenant "https://login.microsoftonline.com/<tenant>/" \
  --aad-audience <azure-vpn-client-app-id> \
  --aad-issuer "https://sts.windows.net/<tenant>/"
```

---

### 🟡 Q36. What is Azure ExpressRoute?
```bash
# ExpressRoute: dedicated private fibre — no internet, up to 100 Gbps
# Peering types:
# Azure Private Peering: access VNets (VMs, internal services)
# Microsoft Peering:     access M365, Dynamics 365, Azure PaaS public IPs

az network express-route create -g myRG -n myERCircuit \
  --provider "Equinix" \
  --peering-location "Silicon Valley" \
  --bandwidth 1000 \
  --sku-tier Standard \      # Standard | Premium (global PoPs, 10+ circuits)
  --sku-family MeteredData   # MeteredData | UnlimitedData

# Get service key → share with ER provider to activate
az network express-route show -g myRG -n myERCircuit \
  --query serviceKey -o tsv

# ExpressRoute Gateway in GatewaySubnet
az network vnet-gateway create -g myRG -n myERGW \
  --gateway-type ExpressRoute --sku ErGw1AZ \
  --public-ip-address myERGWIP --vnet myVNet

# Global Reach: connect two on-prem sites via Azure backbone
az network express-route peering connection create -g myRG \
  --circuit-name myERCircuit1 \
  --peering-name AzurePrivatePeering \
  --connection-name GlobalReach \
  --peer-circuit /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/expressRouteCircuits/myERCircuit2 \
  --address-prefix 192.168.100.0/29

# ExpressRoute vs VPN:
# ExpressRoute: dedicated, private, up to 100Gbps, SLA 99.95%
# VPN Gateway:  encrypted over internet, up to 10Gbps, flexible
```

---

### 🟢 Q37. What is Azure Bastion?
```bash
# Bastion: browser-based RDP/SSH — NO public IP on VMs, NO open port 3389/22
# Required: subnet named "AzureBastionSubnet" with /26 or larger

az network public-ip create -g myRG -n myBastionIP --sku Standard --zone 1 2 3

az network bastion create -g myRG -n myBastion \
  --vnet-name myVNet --public-ip-address myBastionIP \
  --sku Standard \          # Basic | Standard
  --scale-units 10 \        # 1 unit = 25 concurrent sessions
  --enable-tunneling true \
  --enable-ip-connect true \
  --enable-shareable-link true

# SSH via CLI
az network bastion ssh -g myRG -n myBastion \
  --target-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --auth-type AAD

# RDP via CLI
az network bastion rdp -g myRG -n myBastion \
  --target-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myWinVM

# Native client tunnel (port-forward to use your own SSH client)
az network bastion tunnel -g myRG -n myBastion \
  --target-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --resource-port 22 --port 50022
# → ssh -p 50022 azureuser@127.0.0.1
```

---

### 🟡 Q38. What is Azure Firewall?
```bash
# Azure Firewall: fully managed, stateful L3-L7 network security service
# Premium tier: IDPS, TLS inspection, URL categories

az network public-ip create -g myRG -n myFWIP --sku Standard --zone 1 2 3

az network firewall create -g myRG -n myFirewall \
  --sku AZFW_VNet --tier Premium \
  --vnet-name myVNet --public-ip myFWIP \
  --enable-dns-proxy true      # required for FQDN rules

# Get private IP for UDR
FW_IP=$(az network firewall show -g myRG -n myFirewall \
  --query "ipConfigurations[0].privateIPAddress" -o tsv)

# Force all egress through Firewall
az network route-table create -g myRG -n myRT
az network route-table route create -g myRG --route-table-name myRT \
  -n toFirewall --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance --next-hop-ip-address $FW_IP
az network vnet subnet update -g myRG --vnet-name myVNet \
  -n appSubnet --route-table myRT

# Firewall Policy with rules
az network firewall policy create -g myRG -n myFWPolicy \
  --sku Premium --threat-intel-mode Deny --idps-mode Deny

az network firewall policy rule-collection-group create -g myRG \
  --policy-name myFWPolicy -n DefaultRules --priority 100

# Application rule (FQDN-based egress)
az network firewall policy rule-collection-group collection add-filter-collection \
  -g myRG --rule-collection-group-name DefaultRules \
  --policy-name myFWPolicy -n AllowInternet --collection-priority 100 \
  --action Allow --rule-type ApplicationRule --rule-name AllowMicrosoft \
  --source-addresses "10.0.0.0/8" --protocols Https=443 Http=80 \
  --target-fqdns "*.microsoft.com" "*.azure.com" "*.ubuntu.com" \
                 "github.com" "*.github.com" "*.pypi.org"

# DNAT rule (inbound NAT — expose internal service)
az network firewall policy rule-collection-group collection add-nat-collection \
  -g myRG --rule-collection-group-name DefaultRules \
  --policy-name myFWPolicy -n InboundDNAT --collection-priority 200 \
  --rule-type NatRule --rule-name expose-webapp \
  --source-addresses "*" --destination-addresses $FW_IP \
  --destination-ports 443 --ip-protocols TCP \
  --translated-address 10.0.2.10 --translated-port 443
```

---

### 🟢 Q39. What is Azure DNS?
```bash
# Public DNS zone (internet-facing)
az network dns zone create -g myRG -n example.com

az network dns record-set a add-record -g myRG --zone-name example.com \
  --record-set-name www --ipv4-address 20.1.2.3 --ttl 3600

az network dns record-set cname set-record -g myRG --zone-name example.com \
  --record-set-name api --cname myapp.azurewebsites.net --ttl 300

az network dns record-set mx add-record -g myRG --zone-name example.com \
  --record-set-name "@" --exchange mail.protection.outlook.com --preference 0

az network dns record-set txt add-record -g myRG --zone-name example.com \
  --record-set-name "@" --value "v=spf1 include:spf.protection.outlook.com -all"

# Alias record (CNAME-at-apex — avoids extra lookup)
az network dns record-set a create -g myRG --zone-name example.com \
  -n "@" --ttl 3600 --target-resource \
  /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/publicIPAddresses/myIP

# Private DNS zone (VNet-internal resolution)
az network private-dns zone create -g myRG -n "internal.example.com"

az network private-dns link vnet create -g myRG \
  --zone-name "internal.example.com" -n myVNetLink \
  --virtual-network myVNet --registration-enabled true

az network private-dns record-set a add-record -g myRG \
  --zone-name "internal.example.com" --record-set-name mydb \
  --ipv4-address 10.0.3.10

# Private DNS zones for Private Endpoints (must create one per service)
# Storage Blob:  privatelink.blob.core.windows.net
# Storage File:  privatelink.file.core.windows.net
# SQL Database:  privatelink.database.windows.net
# Key Vault:     privatelink.vaultcore.azure.net
# ACR:           privatelink.azurecr.io
# Service Bus:   privatelink.servicebus.windows.net
# Cosmos DB:     privatelink.documents.azure.com
# App Service:   privatelink.azurewebsites.net
```

---

### 🟢 Q40. What is Azure NAT Gateway?
```bash
# NAT Gateway: managed outbound SNAT — static outbound IPs, no port exhaustion
# 64,512 SNAT ports per public IP (vs 1,024 per VM with default SNAT)

az network public-ip create -g myRG -n myNATIP --sku Standard --zone 1 2 3

az network public-ip prefix create -g myRG -n myNATPrefix \
  --length 30 --zone 1 2 3    # /30 = 4 public IPs = 258,048 SNAT ports

az network nat gateway create -g myRG -n myNATGW \
  --public-ip-addresses myNATIP \
  --public-ip-prefixes myNATPrefix \
  --idle-timeout 10 --zone 1 2 3

az network vnet subnet update -g myRG --vnet-name myVNet \
  -n appSubnet --nat-gateway myNATGW

# NAT Gateway benefits:
# ✅ Static outbound IPs (good for IP allowlisting with partners)
# ✅ No SNAT port exhaustion at scale
# ✅ Zone-redundant
# ✅ Works without public IP on VMs
# ✅ Overrides default LB SNAT
```

---

### 🟡 Q41. What is Azure Private Link and Private Endpoints?
```bash
# Private Endpoint: private IP in your VNet for Azure PaaS service
# Traffic stays on Microsoft backbone — never touches internet

# Private endpoint for Azure SQL DB
az network private-endpoint create -g myRG -n sqlPE \
  --vnet-name myVNet --subnet dataSubnet \
  --private-connection-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Sql/servers/mysqlserver \
  --group-id sqlServer --connection-name sqlConn

# Create private DNS zone + link + zone group (auto DNS registration)
az network private-dns zone create -g myRG \
  -n "privatelink.database.windows.net"

az network private-dns link vnet create -g myRG \
  --zone-name "privatelink.database.windows.net" \
  -n sqlDNSLink --virtual-network myVNet --registration-enabled false

az network private-endpoint dns-zone-group create -g myRG \
  --endpoint-name sqlPE -n sqlDNSGroup \
  --private-dns-zone \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/privateDnsZones/privatelink.database.windows.net \
  --zone-name sqlZone

# Private Link Service (expose your own service to other tenants)
az network private-link-service create -g myRG -n myPLS \
  --vnet-name myVNet --subnet appSubnet \
  --lb-name myInternalLB \
  --lb-frontend-ip-configs myPrivateFE \
  --auto-approval subscriptions <trusted-sub-id>

# Service Endpoints vs Private Endpoints:
# Service Endpoints: subnet-level, PaaS firewall rule needed, stays in region
# Private Endpoints: specific resource, private IP, works from on-prem via ER/VPN
```

---

### 🟡 Q42. What is Azure CDN?
```bash
# CDN: cache and serve static content from edge PoPs globally

az cdn profile create -g myRG -n myCDNProfile --sku Standard_Microsoft
az cdn endpoint create -g myRG --profile-name myCDNProfile \
  -n myStaticAssets \
  --origin myapp.azurewebsites.net \
  --origin-host-header myapp.azurewebsites.net \
  --enable-compression \
  --content-types-to-compress "text/css" "text/javascript" "application/javascript" \
  --query-string-caching-behavior IgnoreQueryString

# Custom domain + HTTPS
az cdn custom-domain create -g myRG --profile-name myCDNProfile \
  --endpoint-name myStaticAssets -n cdn-domain --hostname cdn.example.com

az cdn custom-domain enable-https -g myRG --profile-name myCDNProfile \
  --endpoint-name myStaticAssets -n cdn-domain

# Purge cache
az cdn endpoint purge -g myRG --profile-name myCDNProfile \
  -n myStaticAssets --content-paths "/*"

# Rules engine (cache images for 30 days)
az cdn endpoint rule add -g myRG --profile-name myCDNProfile \
  -n myStaticAssets --order 1 --rule-name CacheImages \
  --match-variable RequestFilenameExtension \
  --operator Equal --match-values ".jpg" ".png" ".webp" ".gif" \
  --action-name CacheExpiration \
  --cache-behavior SetIfMissing --cache-duration "30.00:00:00"
```

---

### 🟡 Q43. What is Azure Network Watcher?
```bash
# Network Watcher: monitoring and diagnostics for networking issues

az network watcher configure -g NetworkWatcherRG \
  --locations eastus westeurope --enabled true

# IP Flow Verify (NSG allow or deny?)
az network watcher test-ip-flow -g myRG --vm myVM \
  --direction Inbound --protocol TCP \
  --local 10.0.1.10:80 --remote 203.0.113.5:12345

# Next Hop (routing path?)
az network watcher show-next-hop -g myRG --vm myVM \
  --source-ip 10.0.1.10 --dest-ip 10.0.3.10

# Effective NSG rules on NIC
az network watcher show-security-group-view -g myRG --vm myVM

# Effective routes on NIC
az network watcher show-effective-route-table -g myRG --vm myVM

# Connectivity check (can VM reach endpoint?)
az network watcher test-connectivity -g myRG \
  --source-resource myVM \
  --dest-address mydb.postgres.database.azure.com --dest-port 5432

# Connection Monitor (continuous probe with alerting)
az network watcher connection-monitor create -g myRG \
  -n VMtoDBMonitor --location eastus \
  --endpoints '[
    {"name":"myVM","type":"AzureVM","resourceId":"/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM"},
    {"name":"myDB","type":"ExternalAddress","address":"mydb.postgres.database.azure.com:5432"}
  ]' \
  --test-configurations '[{"name":"tcpTest","protocol":"Tcp","tcpConfiguration":{"port":5432},"testFrequencySec":60,"successThreshold":{"checksFailedPercent":5,"roundTripTimeMs":300}}]' \
  --test-groups '[{"name":"vmToDB","sources":["myVM"],"destinations":["myDB"],"testConfigurations":["tcpTest"]}]'

# Packet Capture
az network watcher packet-capture create -g myRG --vm myVM \
  -n myCapture --storage-account mystorage \
  --time-limit 300 --total-bytes-per-session 104857600 \
  --filters '[{"protocol":"TCP","localPort":"80;443"}]'
```

---

### 🟢 Q44. What is Service Endpoint vs Private Endpoint?
| Feature | Service Endpoint | Private Endpoint |
|---------|----------------|-----------------|
| IP type | Public IP of PaaS | Private IP in VNet |
| Traffic path | Azure backbone | Azure backbone |
| On-prem access | ❌ No | ✅ Via ER/VPN |
| Scope | Subnet-level | Specific resource |
| DNS change | No | Yes (private DNS) |
| Cost | Free | Per endpoint ($) |
| Works from | VNet only | VNet + on-prem |

---

### 🟡 Q45. What is Azure Virtual WAN?
```bash
# Virtual WAN: Microsoft-managed hub-and-spoke networking at scale
# Automatic transitive routing (spokes talk to each other through hub)
# Supported: VPN, ExpressRoute, SD-WAN, Firewall all in one hub

az network vwan create -g myRG -n myVWAN \
  --location eastus --branch-to-branch-traffic true

az network vhub create -g myRG -n myVHub \
  --vwan myVWAN --location eastus \
  --address-prefix 10.100.0.0/24 --sku Standard

# Add Firewall to hub (Secured Virtual Hub)
az network firewall create -g myRG -n myHubFW \
  --sku AZFW_Hub --virtual-hub myVHub \
  --tier Premium --location eastus

# Connect VNet to hub (auto-routes VNet traffic through hub)
az network vhub connection create -g myRG \
  --vhub-name myVHub -n myVNetConn \
  --remote-vnet /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet

# Benefits vs manual hub-spoke:
# ✅ Automatic transitive routing
# ✅ SD-WAN partner integration
# ✅ Global mesh with 70+ Azure PoPs
# ✅ No UDR management
```

---

### 🟡 Q46. What is Azure DDoS Protection?
```bash
# DDoS tiers:
# Network Protection: per-VNet, ~$2,944/month, full mitigation + Rapid Response
# IP Protection:      per-public-IP, ~$199/month

# Enable Network DDoS Protection
az network ddos-protection create -g myRG -n myDDoSPlan --location eastus

az network vnet update -g myRG -n myVNet \
  --ddos-protection-plan myDDoSPlan --ddos-protection true

# IP Protection on single public IP
az network public-ip update -g myRG -n myPublicIP \
  --ddos-protection-mode Enabled

# DDoS diagnostic logs
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/ddosProtectionPlans/myDDoSPlan \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA \
  --logs '[{"category":"DDoSProtectionNotifications","enabled":true},{"category":"DDoSMitigationFlowLogs","enabled":true}]' \
  -n ddosLogs

# Attack types protected:
# Volumetric: UDP floods, DNS amplification, NTP reflection
# Protocol:   SYN floods, fragmented packets
# Resource:   HTTP/S floods (use App Gateway WAF for L7)
```

---

### 🟡 Q47. What is Azure Load Balancer vs Application Gateway vs Front Door?
| Feature | Load Balancer | App Gateway | Front Door |
|---------|-------------|-----------|----------|
| Layer | L4 (TCP/UDP) | L7 (HTTP/S) | L7 (HTTP/S) |
| Scope | Regional | Regional | Global |
| WAF | ❌ | ✅ OWASP | ✅ OWASP + Bot |
| SSL termination | ❌ | ✅ | ✅ |
| URL routing | ❌ | ✅ | ✅ |
| Caching | ❌ | ❌ | ✅ |
| Multi-region | ❌ | ❌ | ✅ |
| AZ redundant | ✅ | ✅ | Built-in |
| Use case | VM/VMSS LB | App LB + WAF | Global apps |

---

### 🟡 Q48. What is Azure Private DNS Resolver?
```bash
# DNS Private Resolver: forward DNS queries between Azure and on-premises
# Replaces custom DNS VMs — fully managed, zone-redundant

az dns-resolver create -g myRG -n myDNSResolver \
  --location eastus \
  --id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet

# Inbound endpoint (on-prem queries Azure private zones)
az dns-resolver inbound-endpoint create -g myRG \
  --dns-resolver-name myDNSResolver -n myInboundEP \
  --ip-configurations '[{"privateIpAllocationMethod":"Static","privateIpAddress":"10.0.100.10","id":"/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/dnsSubnet"}]'

# Outbound endpoint + ruleset (Azure queries on-premises)
az dns-resolver outbound-endpoint create -g myRG \
  --dns-resolver-name myDNSResolver -n myOutboundEP \
  --subnet-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/dnsSubnet

az dns-resolver forwarding-ruleset create -g myRG -n myRuleset \
  --dns-resolver-outbound-endpoints \
    '[{"id":"/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/dnsResolvers/myDNSResolver/outboundEndpoints/myOutboundEP"}]'

az dns-resolver forwarding-rule create -g myRG \
  --ruleset-name myRuleset -n corpRule \
  --domain-name "corp.contoso.com." \
  --target-dns-servers '[{"ipAddress":"192.168.1.53","port":53}]'
```

---

### 🟡 Q49. What is Network Peering vs VPN Gateway transit?
```bash
# VNet Peering: connect VNets via Microsoft backbone (no gateway needed)
# Gateway Transit: spoke VNets can use the hub VPN/ER gateway

# Hub VNet (has VPN Gateway)
az network vnet peering create -g hubRG --name hub-to-spoke \
  --vnet-name hubVNet \
  --remote-vnet /subscriptions/<sub>/resourceGroups/spokeRG/providers/Microsoft.Network/virtualNetworks/spokeVNet \
  --allow-vnet-access --allow-gateway-transit   # allow hub GW to be used

# Spoke VNet
az network vnet peering create -g spokeRG --name spoke-to-hub \
  --vnet-name spokeVNet \
  --remote-vnet /subscriptions/<sub>/resourceGroups/hubRG/providers/Microsoft.Network/virtualNetworks/hubVNet \
  --allow-vnet-access --use-remote-gateways     # use hub GW for on-prem access

# Gateway transit use case:
# Spoke VNets don't need their own VPN Gateway
# All on-prem traffic goes: Spoke → Hub VPN GW → on-prem
# Saves cost: one gateway instead of N gateways
```

---

### 🟢 Q50. How do you troubleshoot Azure networking issues?
```bash
# Systematic troubleshooting checklist:

# 1. Check VM is running
az vm get-instance-view -g myRG -n myVM --query "instanceView.statuses[1].displayStatus"

# 2. IP Flow Verify (NSG blocking?)
az network watcher test-ip-flow -g myRG --vm myVM \
  --direction Inbound --protocol TCP --local 10.0.1.10:80 --remote 1.2.3.4:12345

# 3. Effective NSG rules
az network watcher show-security-group-view -g myRG --vm myVM

# 4. Next Hop (routing issue?)
az network watcher show-next-hop -g myRG --vm myVM \
  --source-ip 10.0.1.10 --dest-ip 10.0.3.10

# 5. Effective routes (UDR sending wrong way?)
az network watcher show-effective-route-table -g myRG --vm myVM

# 6. Connectivity test end-to-end
az network watcher test-connectivity -g myRG --source-resource myVM \
  --dest-address mydb.postgres.database.azure.com --dest-port 5432

# 7. Packet capture (deep inspection)
az network watcher packet-capture create -g myRG --vm myVM \
  -n debug --storage-account mystorage --time-limit 60 \
  --filters '[{"protocol":"TCP","remotePort":"5432"}]'

# 8. Check service health (is Azure having issues?)
az monitor activity-log list --query "[?category.value=='ServiceHealth']" \
  --start-time $(date -u -d '-24 hours' +%Y-%m-%dT%H:%M:%SZ) --output table

# Common root causes:
# NSG rule missing or wrong priority → IP Flow Verify
# UDR routing to wrong next-hop → Effective Routes
# Private endpoint DNS not resolving → nslookup, check private DNS zone
# Firewall blocking FQDN → check FW logs in Log Analytics
# SNAT port exhaustion → use NAT Gateway
# VNet peering non-transitive → add direct peering or use hub Firewall
```

---

# PART 3 — STORAGE

---

### 🟢 Q51. What are Azure Storage account types and kinds?
```bash
# Storage Account kinds:
# StorageV2 (GPv2):  all storage services, all tiers — RECOMMENDED
# Storage (GPv1):    legacy, no Cool tier, no lifecycle management
# BlobStorage:       blob-only, Hot/Cool tiers
# FileStorage:       Premium SMB/NFS file shares only (SSD)
# BlockBlobStorage:  Premium block blobs only (SSD)

az storage account create -g myRG -n mystorageaccount \
  --location eastus \
  --kind StorageV2 \
  --sku Standard_GZRS \
  --access-tier Hot \
  --https-only true \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --allow-cross-tenant-replication false \
  --default-action Deny \
  --bypass AzureServices Logging Metrics
```

---

### 🟢 Q52. What are storage redundancy options and which should you choose?
| SKU | Full Name | Copies | Scope | Durability | When to use |
|-----|----------|--------|-------|-----------|------------|
| **LRS** | Locally Redundant | 3 | Same datacenter | 11 nines | Dev/test, non-critical |
| **ZRS** | Zone Redundant | 3 | 3 AZs, same region | 12 nines | HA prod, zone failure |
| **GRS** | Geo-Redundant | 6 | LRS + paired region | 16 nines | DR, compliance |
| **GZRS** | Geo-Zone Redundant | 6 | ZRS + paired region | 16 nines | Max durability |
| **RA-GRS** | Read-Access GRS | 6 | GRS + read secondary | 16 nines | Read-heavy + DR |
| **RA-GZRS** | Read-Access GZRS | 6 | GZRS + read secondary | 16 nines | Best everything |

```bash
# Change redundancy
az storage account update -g myRG -n mystorageaccount --sku Standard_GZRS

# Initiate failover to secondary (GRS/RA-GRS only — during outage)
az storage account failover -g myRG -n mystorageaccount --yes

# Failover converts RA-GRS to LRS — need to reconfigure GRS after region recovers
```

---

### 🟢 Q53. What are Azure Blob Storage tiers?
| Tier | Storage Cost | Access Cost | Min Duration | Retrieval |
|------|------------|------------|-------------|---------|
| **Hot** | Highest | Lowest | None | Immediate |
| **Cool** | Lower | Higher | 30 days | Immediate |
| **Cold** | Lower still | Higher still | 90 days | Immediate |
| **Archive** | Lowest | Highest | 180 days | 1–15 hours |

```bash
# Create container
az storage container create \
  --account-name mystorageaccount -n mycontainer \
  --auth-mode login --public-access off

# Upload blob
az storage blob upload \
  --account-name mystorageaccount -c mycontainer \
  -n "data/2026/report.parquet" -f ./report.parquet \
  --auth-mode login

# Change blob tier
az storage blob set-tier \
  --account-name mystorageaccount -c mycontainer \
  -n "data/2025/old.parquet" --tier Cool

# Rehydrate from Archive
az storage blob set-tier \
  --account-name mystorageaccount -c mycontainer \
  -n "data/2024/ancient.parquet" --tier Hot \
  --rehydrate-priority High    # High (~1h) | Standard (~15h)

# Lifecycle management policy (auto-tier and delete)
az storage account management-policy create \
  --account-name mystorageaccount -g myRG --policy '{
  "rules":[{
    "name":"AutoTier",
    "enabled":true,
    "type":"Lifecycle",
    "definition":{
      "filters":{"blobTypes":["blockBlob"],"prefixMatch":["data/"]},
      "actions":{
        "baseBlob":{
          "tierToCool":   {"daysAfterModificationGreaterThan":30},
          "tierToCold":   {"daysAfterModificationGreaterThan":60},
          "tierToArchive":{"daysAfterModificationGreaterThan":90},
          "delete":       {"daysAfterModificationGreaterThan":365}
        },
        "snapshot":{"delete":{"daysAfterCreationGreaterThan":90}},
        "version":  {"delete":{"daysAfterCreationGreaterThan":180}}
      }
    }
  }]
}'
```

---

### 🟢 Q54. What are SAS tokens and what are the types?
```bash
# SAS = Shared Access Signature: time-limited access URL with scoped permissions

# Three SAS types:
# Account SAS:         multi-service, resource-type level permissions
# Service SAS:         single service (blob/file/queue/table)
# User Delegation SAS: signed with Entra ID — NO account key — MOST SECURE

# User Delegation SAS (recommended)
az storage blob generate-sas \
  --account-name mystorageaccount -c mycontainer \
  -n report.parquet --permissions r \
  --expiry $(date -u -d '+1 hour' +%Y-%m-%dT%H:%M:%SZ) \
  --https-only \
  --ip "203.0.113.0-203.0.113.255" \
  --auth-mode login --as-user

# Stored Access Policy (revocable SAS)
az storage container policy create \
  --account-name mystorageaccount -n mycontainer \
  --name myPolicy --permissions rl \
  --expiry $(date -u -d '+30 days' +%Y-%m-%dT%H:%M:%SZ)

# Generate SAS linked to policy
az storage container generate-sas \
  --account-name mystorageaccount -n mycontainer \
  --policy-name myPolicy --auth-mode key

# REVOKE all SAS using this policy instantly
az storage container policy delete \
  --account-name mystorageaccount -n mycontainer --name myPolicy

# SAS security best practices:
# 1. Always prefer User Delegation SAS
# 2. Use Stored Access Policy for revocability
# 3. Minimum permissions (r, not rwdlac)
# 4. Short expiry (1 hour, not 1 year)
# 5. --https-only always
# 6. Restrict IP range when known
# 7. Monitor SAS usage via Storage Analytics
# 8. Consider Private Endpoints instead for internal use
```

---

### 🟡 Q55. What is Azure Blob versioning and soft delete?
```bash
# Blob versioning: automatic version created on every write
# Soft delete: deleted blobs retained for retention period before permanent removal

az storage account blob-service-properties update \
  --account-name mystorageaccount \
  --enable-versioning true \
  --enable-change-feed true \
  --enable-delete-retention true \
  --delete-retention-days 30 \
  --enable-container-delete-retention true \
  --container-delete-retention-days 14

# List blob versions
az storage blob list \
  --account-name mystorageaccount -c mycontainer \
  --include v --query "[?isCurrentVersion==null].{name:name,version:versionId}" \
  --output table

# Restore specific version
az storage blob copy start \
  --account-name mystorageaccount \
  --source-container mycontainer --source-blob report.csv \
  --source-version-id "2026-06-10T10:00:00.0000000Z" \
  --destination-container mycontainer --destination-blob report.csv

# List soft-deleted blobs
az storage blob list \
  --account-name mystorageaccount -c mycontainer \
  --include d --query "[?deleted==true]" --output table

# Restore soft-deleted blob
az storage blob undelete \
  --account-name mystorageaccount -c mycontainer -n deleted-report.csv
```

---

### 🟡 Q56. What is Azure Data Lake Storage Gen2?
```bash
# ADLS Gen2 = StorageV2 + Hierarchical Namespace (HNS) enabled
# HNS enables: POSIX ACLs, atomic directory operations, Hive-compatible paths
# Use for: big data analytics (Synapse, Databricks, HDInsight)

az storage account create -g myRG -n mydatalake \
  --kind StorageV2 --sku Standard_GZRS \
  --enable-hierarchical-namespace true \    # KEY DIFFERENCE
  --location eastus

# File system operations (use az storage fs, not az storage blob)
az storage fs create -n rawzone --account-name mydatalake --auth-mode login
az storage fs create -n curatedzone --account-name mydatalake --auth-mode login
az storage fs create -n refinedzone --account-name mydatalake --auth-mode login

# Directory operations
az storage fs directory create -n "2026/06/13" \
  --file-system rawzone --account-name mydatalake

# Upload file
az storage fs file upload --source events.parquet \
  --path "2026/06/13/events.parquet" \
  --file-system rawzone --account-name mydatalake

# POSIX ACLs (fine-grained per-user/group access)
az storage fs access set \
  --acl "user::rwx,group::r-x,other::---,user:<etl-sp-object-id>:rwx" \
  --path "2026/06/13" --file-system rawzone --account-name mydatalake

# Default ACL (inherited by new files/dirs)
az storage fs access set \
  --acl "default:user::rwx,default:group::r-x,default:other::---" \
  --path "2026" --file-system rawzone --account-name mydatalake

# Data lake zone pattern:
# raw      → landing zone, original unmodified data
# curated  → cleaned and validated
# refined  → aggregated, analytics-ready
# sandbox  → data science exploration
```

---

### 🟢 Q57. What is Azure Files?
```bash
# Azure Files: fully managed SMB 3.x and NFS 4.1 file shares
# Use for: lift-and-shift file servers, shared config, home directories, AKS PVs

# Standard file shares (HDD)
az storage share-rm create -g myRG \
  --storage-account mystorageaccount \
  -n myshare --quota 5120 \    # GB
  --tier TransactionOptimized  # Hot | Cool | TransactionOptimized

# Premium file shares (SSD — lower latency)
az storage account create -g myRG -n mypremiumfiles \
  --kind FileStorage --sku Premium_ZRS --https-only false  # NFS requires this

az storage share-rm create -g myRG \
  --storage-account mypremiumfiles -n nfsshare \
  --quota 1024 --enabled-protocol NFS \
  --root-squash NoRootSquash

# Mount SMB on Linux
sudo mount -t cifs \
  //mystorageaccount.file.core.windows.net/myshare /mnt/share \
  -o vers=3.1.1,\
     username=mystorageaccount,\
     password=$(az storage account keys list -g myRG -n mystorageaccount --query [0].value -o tsv),\
     dir_mode=0777,file_mode=0777,seal

# Mount SMB on Windows
net use Z: \\mystorageaccount.file.core.windows.net\myshare <key> \
  /user:AZURE\mystorageaccount /persistent:yes

# Mount NFS on Linux (requires private endpoint or service endpoint)
sudo mount -t nfs \
  mypremiumfiles.file.core.windows.net:/mypremiumfiles/nfsshare /mnt/nfs \
  -o vers=4,minorversion=1,sec=sys

# Azure File Sync (sync on-prem file servers to Azure Files)
az storagesync create -g myRG -n myStorageSyncSvc --location eastus
az storagesync sync-group create -g myRG \
  --storage-sync-service-name myStorageSyncSvc -n mySyncGroup
az storagesync sync-group cloud-endpoint create -g myRG \
  --storage-sync-service-name myStorageSyncSvc \
  --sync-group-name mySyncGroup -n myCloudEP \
  --storage-account-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount \
  --azure-file-share-name myshare
```

---

### 🟡 Q58. What is Azure Queue Storage and Table Storage?
```bash
# Queue Storage: simple FIFO message queue (max 64KB per message, 7-day TTL)
# Table Storage: key-value NoSQL (row key + partition key model)

# Queue Storage
az storage queue create --account-name mystorageaccount -n myqueue

# Enqueue message
az storage message put \
  --account-name mystorageaccount --queue-name myqueue \
  --content '{"orderId":"12345","action":"process"}' \
  --time-to-live 3600   # 7 days max

# Peek messages
az storage message peek \
  --account-name mystorageaccount --queue-name myqueue --num-messages 10

# Get and delete (process-once pattern)
MSG=$(az storage message get \
  --account-name mystorageaccount --queue-name myqueue --query "[0]" -o json)
# process message...
az storage message delete \
  --account-name mystorageaccount --queue-name myqueue \
  --id $(echo $MSG | jq -r .id) \
  --pop-receipt $(echo $MSG | jq -r .popReceipt)

# Table Storage (NoSQL key-value)
az storage table create --account-name mystorageaccount -n orders

az storage entity insert \
  --account-name mystorageaccount --table-name orders \
  --entity PartitionKey=2026-06 RowKey=12345 \
            CustomerId=C001 Amount=299.99 Status=Pending

az storage entity query \
  --account-name mystorageaccount --table-name orders \
  --filter "PartitionKey eq '2026-06' and Status eq 'Pending'"

# Table Storage vs Cosmos DB Table API:
# Table Storage: simple, cheap, limited query (no indexes beyond PK+RK)
# Cosmos DB Table API: global distribution, SLA, better performance
```

---

### 🟡 Q59. What is Azure Managed Disks?
```bash
# Managed Disks: block storage for VMs, Kubernetes PersistentVolumes

# Performance tiers:
# Standard HDD:    2,000 IOPS / 500 MB/s — backup, archive
# Standard SSD:    6,000 IOPS / 750 MB/s — web, dev/test
# Premium SSD:    20,000 IOPS / 900 MB/s — production DBs
# Premium SSD v2: 80,000 IOPS / 1,200 MB/s — tunable (no disk tiers)
# Ultra Disk:    160,000 IOPS / 4,000 MB/s — SAP HANA, SQL critical

az disk create -g myRG -n myDisk --size-gb 512 \
  --sku Premium_LRS --zone 1

# Premium SSD v2 (tune IOPS independently of size)
az disk create -g myRG -n myPremV2 --size-gb 1024 \
  --sku PremiumV2_LRS --zone 1 \
  --disk-iops-read-write 50000 \
  --disk-mbps-read-write 800

az vm disk attach -g myRG --vm-name myVM --name myDisk --lun 0

# Online resize (no VM stop for Premium SSD)
az disk update -g myRG -n myDisk --size-gb 1024

# Incremental snapshot (only changed blocks)
az snapshot create -g myRG -n mySnap \
  --source /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/disks/myDisk \
  --incremental true

# Customer-managed key (CMK) encryption
az disk-encryption-set create -g myRG -n myDES \
  --key-url https://myKV.vault.azure.net/keys/myDiskKey/version \
  --source-vault myKV \
  --encryption-type EncryptionAtRestWithCustomerKey

# Disk bursting (Premium SSD — free burst credits)
az disk update -g myRG -n myDisk \
  --set properties.burstingEnabled=true

# Ultra Disk requirements:
# - Only in supported regions and AZs
# - VM must be UltraSSD_Enabled
az vm update -g myRG -n myVM \
  --ultra-ssd-enabled true
```

---

### 🟡 Q60. What is Azure Backup?
```bash
# Azure Backup: managed backup for VMs, SQL, Files, Cosmos DB, AKS, on-prem

# Create Recovery Services Vault
az backup vault create -g myRG -n myRSV --location eastus

# Set geo-redundant backup storage
az backup vault backup-properties set -g myRG -n myRSV \
  --backup-storage-redundancy GeoRedundant

# Enable soft delete on vault (protects from accidental deletion)
az backup vault backup-properties set -g myRG -n myRSV \
  --soft-delete-feature-state Enable

# Enable VM backup
az backup protection enable-for-vm -g myRG \
  --vault-name myRSV --vm myVM --policy-name DefaultPolicy

# Create custom policy (daily backup, 1yr retention, weekly 4wk, monthly 12m, yearly 5yr)
az backup policy set -g myRG --vault-name myRSV \
  --name myPolicy --backup-management-type AzureIaasVM \
  --policy '{
    "schedulePolicy": {
      "schedulePolicyType": "SimpleSchedulePolicy",
      "scheduleRunFrequency": "Daily",
      "scheduleRunTimes": ["2026-06-13T02:00:00Z"]
    },
    "retentionPolicy": {
      "retentionPolicyType": "LongTermRetentionPolicy",
      "dailySchedule":   {"retentionDuration":{"count":30, "durationType":"Days"}},
      "weeklySchedule":  {"daysOfTheWeek":["Sunday"],"retentionDuration":{"count":12,"durationType":"Weeks"}},
      "monthlySchedule": {"retentionDuration":{"count":12,"durationType":"Months"}},
      "yearlySchedule":  {"retentionDuration":{"count":5,"durationType":"Years"}}
    }
  }'

# On-demand backup
az backup protection backup-now -g myRG --vault-name myRSV \
  --container-name myVM --item-name myVM \
  --backup-management-type AzureIaasVM \
  --retain-until $(date -u -d '+30 days' +%Y-%m-%dT%H:%M:%SZ)

# List recovery points
az backup recoverypoint list -g myRG --vault-name myRSV \
  --container-name myVM --item-name myVM \
  --backup-management-type AzureIaasVM --output table

# Restore VM disks
az backup restore restore-disks -g myRG --vault-name myRSV \
  --container-name myVM --item-name myVM \
  --rp-name <recovery-point-name> \
  --storage-account mystorageaccount \
  --restore-to-staging-storage-account true
```

---

### 🟡 Q61. What is Azure Import/Export service?
```bash
# Import/Export: ship physical drives to/from Azure for large data transfers
# Use when: internet upload would take weeks (terabytes of data)

# Import job (data → Azure Blob/Files)
az storage import-export job create -g myRG \
  --name myImportJob \
  --type Import \
  --location "US East" \
  --storage-account mystorageaccount \
  --shipping-information '{"recipientName":"Azure Import Export","streetAddress1":"...","city":"...","stateOrProvince":"...","postalCode":"98052","countryOrRegion":"US","phone":"..."}' \
  --return-shipping '{"carrierName":"FedEx","carrierAccountNumber":"...","carrierName2":"FedEx","carrierAccountNumber2":"..."}' \
  --drive-list '[{"driveId":"9CA995BB","bitLockerKey":"...","manifestFile":"drive0\\ImportManifest.xml","manifestHash":"..."}]'

# Export job (Azure Blob → physical drive)
az storage import-export job create -g myRG \
  --name myExportJob \
  --type Export \
  --location "US East" \
  --storage-account mystorageaccount \
  --export '{"blobList":{"blobPath":["container1/"]}}' \
  --drive-list '[{"driveId":"9CA995BB","bitLockerKey":"...","manifestFile":"drive0\\ExportManifest.xml","manifestHash":"..."}]'

# Alternatives to Import/Export:
# Azure Data Box:    physical device (80TB–1PB) ordered from Microsoft
# Azure Data Box Disk: up to 40TB per order (SSDs shipped to you)
# Data Box Heavy:    up to 1 PB
# AzCopy:           up to ~100TB via internet (parallel transfers)
```

---

### 🟡 Q62. What is AzCopy and what are common operations?
```bash
# AzCopy: command-line tool optimised for large-scale Azure Storage transfers
# Supports: Blob, Files, Gen2, S3, GCS as source or destination

# Authenticate with Managed Identity
azcopy login --identity --identity-client-id <client-id>

# Upload file/directory
azcopy copy './localfolder/*' \
  'https://mystorageaccount.blob.core.windows.net/mycontainer/' \
  --recursive --put-md5 --check-md5 FailIfDifferent

# Download
azcopy copy \
  'https://mystorageaccount.blob.core.windows.net/mycontainer/data/' \
  './localfolder/' --recursive

# Sync (like rsync — only copy changed files)
azcopy sync './localfolder' \
  'https://mystorageaccount.blob.core.windows.net/mycontainer/backup/' \
  --recursive --delete-destination true

# Copy between storage accounts (server-side, no local download)
azcopy copy \
  'https://source.blob.core.windows.net/container/path' \
  'https://dest.blob.core.windows.net/container/path' \
  --recursive --s2s-preserve-access-tier

# Change blob tier in bulk
azcopy set-properties \
  'https://mystorageaccount.blob.core.windows.net/mycontainer/*' \
  --block-blob-tier Cool

# List jobs
azcopy jobs list
azcopy jobs resume <job-id>

# Benchmark
azcopy benchmark 'https://mystorageaccount.blob.core.windows.net/mycontainer' \
  --mode upload --file-count 100 --size-per-file 8M
```

---

### 🟡 Q63. What is Azure Storage firewall and network access control?
```bash
# Storage firewall: restrict access to specific VNets, IPs, or Azure services

az storage account update -g myRG -n mystorageaccount \
  --default-action Deny \                   # deny all by default
  --bypass AzureServices Logging Metrics    # allow Azure infrastructure

# Allow specific VNet subnet (service endpoint required on subnet)
az storage account network-rule add -g myRG -n mystorageaccount \
  --vnet-name myVNet --subnet dataSubnet

# Allow specific IP ranges
az storage account network-rule add -g myRG -n mystorageaccount \
  --ip-address 203.0.113.0/24

# Allow Azure services (trusted Microsoft services)
az storage account update -g myRG -n mystorageaccount \
  --bypass AzureServices

# Show current network rules
az storage account show -g myRG -n mystorageaccount \
  --query networkRuleSet --output json

# Private endpoint is preferred over firewall rules for security
# With private endpoint, disable public access entirely:
az storage account update -g myRG -n mystorageaccount \
  --public-network-access Disabled
```

---

### 🟡 Q64. What is Azure Storage encryption?
```bash
# Encryption at rest: AES 256-bit, always enabled — cannot disable
# Two key types:
# Microsoft-managed keys (MMK): default, zero management overhead
# Customer-managed keys (CMK): your key in Key Vault, compliance control

# Enable CMK on storage account
az keyvault key create --vault-name myKV -n myStorageKey \
  --kty RSA --size 4096 --protection hsm

# Assign Key Vault Crypto access to storage MSI
STORAGE_MSI=$(az storage account show -g myRG -n mystorageaccount \
  --query identity.principalId -o tsv)

az role assignment create \
  --assignee $STORAGE_MSI \
  --role "Key Vault Crypto Service Encryption User" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKV

# Enable CMK on storage account
az storage account update -g myRG -n mystorageaccount \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault https://myKV.vault.azure.net \
  --encryption-key-name myStorageKey \
  --encryption-key-version <key-version>

# Auto-rotate (remove version for auto-rotation)
az storage account update -g myRG -n mystorageaccount \
  --encryption-key-name myStorageKey \
  --encryption-key-version ""   # empty = auto-use latest version

# Encryption in transit: HTTPS enforced
az storage account update -g myRG -n mystorageaccount \
  --https-only true --min-tls-version TLS1_2
```

---

### 🟡 Q65. What is Azure Storage object replication?
```bash
# Object replication: async copy of block blobs between two storage accounts
# Use for: DR, analytics, content distribution, multi-region redundancy

# Enable versioning and change feed on BOTH accounts (required)
az storage account blob-service-properties update \
  --account-name mysourceaccount --enable-versioning true --enable-change-feed true

az storage account blob-service-properties update \
  --account-name mydestaccount --enable-versioning true

# Create replication policy
az storage account or-policy create \
  --account-name mysourceaccount \
  --source-account mysourceaccount \
  --destination-account mydestaccount \
  --rule source-container=rawdata destination-container=rawdata-backup \
         min-creation-time=2026-01-01T00:00:00Z \
         prefix-match=reports/

# List policies
az storage account or-policy list \
  --account-name mysourceaccount --output table

# Object replication vs GRS:
# GRS:               automatic, opaque, no filtering, whole-account
# Object replication: configurable, per-container, filter by prefix/time
```

---

### 🟡 Q66. What is Azure Blob static website hosting?
```bash
# Host SPAs and static sites directly from Blob Storage — no web server needed

az storage blob service-properties update \
  --account-name mystorageaccount \
  --static-website \
  --index-document index.html \
  --404-document 404.html

# Upload build output
az storage blob upload-batch \
  --account-name mystorageaccount \
  --source ./build --destination '$web' \
  --overwrite true \
  --content-cache-control "public, max-age=31536000" \    # 1 year for versioned assets
  --pattern "*.js" --content-type "application/javascript"

az storage blob upload-batch \
  --account-name mystorageaccount \
  --source ./build --destination '$web' \
  --overwrite true \
  --pattern "*.html" \
  --content-cache-control "no-cache"    # always revalidate HTML

# Get website URL
az storage account show -g myRG -n mystorageaccount \
  --query "primaryEndpoints.web" -o tsv
# → https://mystorageaccount.z13.web.core.windows.net/

# Add CDN for custom domain + HTTPS
az cdn endpoint create -g myRG --profile-name myCDNProfile \
  -n myStaticSite \
  --origin mystorageaccount.z13.web.core.windows.net \
  --origin-host-header mystorageaccount.z13.web.core.windows.net \
  --enable-compression
```

---

### 🟡 Q67. What is Azure Storage change feed?
```bash
# Change feed: ordered, immutable log of all create/modify/delete operations on blobs
# Use for: audit trail, data pipeline triggers, compliance logging

# Enable change feed
az storage account blob-service-properties update \
  --account-name mystorageaccount \
  --enable-change-feed true \
  --change-feed-retention-days 30   # 1–146000 days (7+ years for compliance)

# Change feed is stored at:
# $blobchangefeed/idx/segments/<year>/<month>/<day>/<hour>/<segment>.avro

# Consume change feed with SDK
from azure.storage.blob.changefeed import ChangeFeedClient
from azure.identity import DefaultAzureCredential

client = ChangeFeedClient("https://mystorageaccount.blob.core.windows.net",
                          credential=DefaultAzureCredential())

# Process all changes since a specific time
for event in client.list_changes(start_time=datetime(2026, 6, 1)):
    print(f"Event: {event['eventType']} on {event['subject']}")
    # eventType: BlobCreated | BlobDeleted | BlobPropertiesUpdated
```

---

### 🟢 Q68. What is Azure Storage monitoring?
```bash
# Storage metrics: capacity, transactions, latency, availability
# Storage logs: detailed request-level logging

# Enable diagnostic settings (Storage → Log Analytics)
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount/blobServices/default \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA \
  --logs '[
    {"category":"StorageRead","enabled":true},
    {"category":"StorageWrite","enabled":true},
    {"category":"StorageDelete","enabled":true}
  ]' \
  --metrics '[{"category":"Transaction","enabled":true}]' \
  -n storageLogs

# Alert on error rate
az monitor metrics alert create -g myRG -n StorageErrors \
  --scopes /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount \
  --condition "avg Transactions > 100 where ResponseType includes ServerOtherError" \
  --window-size 5m --evaluation-frequency 1m --severity 2 \
  --action /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/actionGroups/myAG
```

```kusto
// Storage errors by operation
StorageBlobLogs
| where TimeGenerated > ago(1h) and StatusCode >= 400
| summarize Count = count() by OperationName, StatusCode, bin(TimeGenerated, 5m)
| order by Count desc

// Top callers by IP
StorageBlobLogs
| where TimeGenerated > ago(24h)
| summarize Requests = count() by CallerIpAddress
| top 20 by Requests desc

// Latency analysis
StorageBlobLogs
| where TimeGenerated > ago(1h)
| where OperationName == "GetBlob"
| summarize p50 = percentile(DurationMs, 50),
            p95 = percentile(DurationMs, 95),
            p99 = percentile(DurationMs, 99)
            by bin(TimeGenerated, 5m)
```

---

### 🟢 Q69. What is AzCopy vs Storage Explorer vs portal for data management?
| Tool | Best For | Authentication | Platform |
|------|---------|--------------|---------|
| **Azure Portal** | Browsing, ad-hoc operations | Browser/AAD | Web |
| **Storage Explorer** | GUI browsing, manage, upload small files | AAD, key, SAS | Win/Mac/Linux |
| **AzCopy** | Large bulk transfers, scripting, automation | AAD, SAS, key | Win/Mac/Linux |
| **Azure CLI** | Scripting, integration with DevOps | AAD, key | Any |
| **SDK** | Application integration | AAD, key, SAS | Any language |

---

### 🟢 Q70. What is the difference between Blob, Files, Queue, and Table storage?
| Service | Type | Protocol | Use Case |
|---------|------|---------|---------|
| **Blob** | Object store | HTTPS/REST | Images, videos, backups, big data |
| **Files** | File share | SMB/NFS | Lift-and-shift file servers, shared config |
| **Queue** | Message queue | HTTP/REST | Simple async messaging, decoupling |
| **Table** | NoSQL key-value | HTTP/REST | Semi-structured data, lookup tables |
| **Disk** | Block storage | iSCSI (VM) | VM OS and data disks |


---

# PART 4 — DATABASES

---

### 🟢 Q71. What is Azure SQL Database and what deployment options exist?
```bash
# Deployment models:
# Single Database:    isolated DB, serverless or provisioned
# Elastic Pool:       multiple DBs share resources (cost-efficient for SaaS)
# Managed Instance:   full SQL Server in VNet (100% compat, lift-and-shift)

# Create logical server
az sql server create -g myRG -n mysqlserver --location eastus \
  --admin-user sqladmin --admin-password "P@ssw0rd!2026" \
  --enable-ad-only-auth \
  --external-admin-name mySQLAdmin \
  --external-admin-sid <aad-object-id>

# Serverless DB (auto-pause when idle, scale vCores dynamically)
az sql db create -g myRG -s mysqlserver -n myDevDB \
  --edition GeneralPurpose \
  --compute-model Serverless --family Gen5 \
  --min-capacity 0.5 --capacity 4 \
  --auto-pause-delay 60

# Business Critical DB (local SSD, Always On AG replica, read scale-out)
az sql db create -g myRG -s mysqlserver -n myProdDB \
  --service-objective BC_Gen5_8 \
  --zone-redundant true \
  --backup-storage-redundancy Zone \
  --read-replicas 1

# Service tiers comparison:
# General Purpose:    remote storage, 1–80 vCores, up to 4TB, ~5ms latency
# Business Critical:  local SSD, AlwaysOn AG, built-in read replica, <1ms
# Hyperscale:         up to 100TB, distributed storage, 0–30 read replicas
```

---

### 🟡 Q72. What is SQL Database Elastic Pool?
```bash
# Elastic Pool: multiple databases share a pool of resources (eDTUs or vCores)
# Cost-efficient when DBs have different peak times

az sql elastic-pool create -g myRG -s mysqlserver \
  -n myElasticPool \
  --edition GeneralPurpose --family Gen5 \
  --capacity 8 \              # vCores for the pool
  --db-min-capacity 0 \       # min vCores per DB (0 = can pause)
  --db-max-capacity 4 \       # max vCores per DB
  --zone-redundant true

# Add databases to pool
az sql db create -g myRG -s mysqlserver -n tenant1DB --elastic-pool myElasticPool
az sql db create -g myRG -s mysqlserver -n tenant2DB --elastic-pool myElasticPool
az sql db create -g myRG -s mysqlserver -n tenant3DB --elastic-pool myElasticPool

# Scale pool (increase capacity when needed)
az sql elastic-pool update -g myRG -s mysqlserver -n myElasticPool \
  --capacity 16

# When to use Elastic Pool:
# SaaS multi-tenant: each tenant has own DB, but usage is staggered
# Dev/test: multiple DBs rarely all busy simultaneously
# Microservices: each service has its own schema/DB
```

---

### 🟡 Q73. What is SQL Database geo-replication and failover groups?
```bash
# Geo-replication: manually managed secondary in another region
# Failover Group: transparent auto-failover with one connection string

# Failover Group (recommended for production)
az sql failover-group create -g myRG -s mysqlserver \
  -n myFOG \
  --partner-server mysqlserver-dr \
  --partner-resource-group myDRRG \
  --failover-policy Automatic \
  --grace-period 1 \           # hours before auto-failover
  --add-db myProdDB

# Connection strings use FOG endpoint (transparent failover)
# Primary:   mysqlserver.database.windows.net (current primary)
# Secondary: mysqlserver-dr.database.windows.net (current secondary)
# FOG:       myFOG.database.windows.net ← USE THIS (always points to primary)

# Manual failover (during DR drill)
az sql failover-group set-primary -g myDRRG -s mysqlserver-dr -n myFOG

# Point-in-time restore (restore to any point within retention)
az sql db restore -g myRG -s mysqlserver \
  -n myProdDB-restored \
  --source-database myProdDB \
  --time "2026-06-10T10:00:00Z"

# Long-term backup retention (LTR)
az sql db ltr-policy set -g myRG -s mysqlserver \
  --database myProdDB \
  --weekly-retention P4W \
  --monthly-retention P12M \
  --yearly-retention P5Y \
  --week-of-year 1

# List LTR backups
az sql db ltr-backup list -l eastus -s mysqlserver -d myProdDB

# Private endpoint
az network private-endpoint create -g myRG -n sqlPE \
  --vnet-name myVNet --subnet dataSubnet \
  --private-connection-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Sql/servers/mysqlserver \
  --group-id sqlServer --connection-name sqlConn
```

---

### 🟡 Q74. What is Azure SQL Managed Instance?
```bash
# SQL MI: full SQL Server engine deployed into your VNet
# 100% compatibility: SQL Agent, cross-DB queries, CLR, SSRS, linked servers
# Use for: lift-and-shift with no code changes

az sql mi create -g myRG -n mySQLMI --location eastus \
  --admin-user miAdmin --admin-password "P@ssw0rd!2026" \
  --vnet-name myVNet --subnet miSubnet \
  --sku-name GP_Gen5 --vcores 8 --storage-size 512 \
  --license-type LicenseIncluded \
  --proxy-override Redirect \     # lower latency than Proxy
  --timezone-id "Eastern Standard Time" \
  --collation SQL_Latin1_General_CP1_CI_AS \
  --public-data-endpoint-enabled false

# MI Link (replicate on-prem SQL Server to Azure MI for near-zero downtime migration)
az sql mi link create -g myRG --instance-name mySQLMI \
  -n myMILink \
  --primary-ag myAGName \
  --source-endpoint "TCP://onprem-server:5022" \
  --target-database myDB \
  --replication-mode Sync

# Failover group for MI
az sql instance-failover-group create -g myRG \
  --mi mySQLMI -n myMIFOG \
  --partner-resource-group myDRRG \
  --partner-mi mySQLMI-dr \
  --failover-policy Automatic --grace-period 1

# SQL MI deployment takes ~6 hours
# Subnet requirements: /27 minimum, dedicated to SQL MI
```

---

### 🟡 Q75. What is Azure Cosmos DB?
```bash
# Cosmos DB: globally distributed, multi-model NoSQL
# APIs: SQL (Core), MongoDB, Cassandra, Gremlin (graph), Table

# Consistency levels (strong → weak — lower consistency = lower latency):
# Strong:            reads always see latest write (linearisable)
# Bounded Staleness: lag bounded by K ops or T time
# Session:           consistent per client session — DEFAULT
# Consistent Prefix: no out-of-order reads
# Eventual:          lowest latency, no ordering guarantees

az cosmosdb create -g myRG -n myCosmosAcct \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
  --locations regionName=westeurope failoverPriority=1 isZoneRedundant=false \
  --default-consistency-level Session \
  --enable-automatic-failover true \
  --enable-multiple-write-locations true \  # active-active multi-master
  --kind GlobalDocumentDB \
  --enable-public-network false

# Create database and container (autoscale RU/s)
az cosmosdb sql database create -g myRG --account-name myCosmosAcct -n myDB

az cosmosdb sql container create -g myRG \
  --account-name myCosmosAcct --database-name myDB \
  -n orders --partition-key-path "/userId" \
  --max-throughput 10000 \          # autoscale: 1,000–10,000 RU/s
  --analytical-storage-ttl -1       # Synapse Link enabled

# Partition key selection (critical for performance):
# ✅ High cardinality (many unique values)
# ✅ Even data distribution
# ✅ Even request distribution
# ✅ Included in most queries (enables single-partition queries)
# ❌ Boolean, date, enum (too few values → hot partition)

# TTL (auto-expire sessions, cache entries)
az cosmosdb sql container update -g myRG \
  --account-name myCosmosAcct --database-name myDB \
  -n sessions --ttl 3600    # expire documents after 1 hour

# Continuous backup (restore to any point in 30 days)
az cosmosdb update -g myRG -n myCosmosAcct \
  --backup-policy-type Continuous \
  --continuous-tier Continuous30Days
```

---

### 🟡 Q76. What is the Cosmos DB Request Unit (RU) model?
```bash
# RU = Request Unit: normalised cost unit for all Cosmos DB operations
# 1 RU = read a 1KB document by its ID (point read)
# Writes cost ~5x more than reads
# Scans (no partition key) cost much more

# Throughput modes:
# Provisioned:  fixed RU/s (400–unlimited), good for predictable workloads
# Autoscale:    10-100% of max RU/s, good for variable workloads
# Serverless:   pay per RU consumed, no provisioning, great for dev/bursty

# Provisioned
az cosmosdb sql container create -g myRG \
  --account-name myCosmosAcct --database-name myDB \
  -n myContainer --partition-key-path "/id" \
  --throughput 1000

# Autoscale (scales between 100 RU - 10,000 RU automatically)
az cosmosdb sql container create -g myRG \
  --account-name myCosmosAcct --database-name myDB \
  -n myAutoContainer --partition-key-path "/id" \
  --max-throughput 10000

# Serverless Cosmos DB account
az cosmosdb create -g myRG -n myServerlessCosmosAcct \
  --locations regionName=eastus failoverPriority=0 \
  --capabilities EnableServerless \
  --kind GlobalDocumentDB

# Cost optimisation:
# - Use point reads (by ID + partition key) not cross-partition queries
# - Cache frequently read items in Redis
# - Use bulk mode SDK for high-volume writes
# - Enable time-to-live to auto-clean expired data
# - Use autoscale or serverless for variable workloads
```

---

### 🟡 Q77. What is Azure Database for PostgreSQL Flexible Server?
```bash
az postgres flexible-server create -g myRG -n mypostgres \
  --location eastus \
  --admin-user pgadmin --admin-password "P@ssw0rd!2026" \
  --sku-name Standard_D4s_v3 \
  --tier GeneralPurpose \     # Burstable | GeneralPurpose | MemoryOptimized
  --version 16 \
  --storage-size 128 --storage-auto-grow Enabled \
  --high-availability ZoneRedundant \   # Disabled | SameZone | ZoneRedundant
  --zone 1 --standby-zone 2 \
  --backup-retention 35 \
  --geo-redundant-backup Enabled \
  --public-access Disabled \
  --vnet myVNet --subnet dataSubnet

# PgBouncer connection pooler (reduce connection overhead)
az postgres flexible-server update -g myRG -n mypostgres \
  --pg-bouncer-enabled true

# Server parameters
az postgres flexible-server parameter set -g myRG -n mypostgres \
  --name max_connections --value 500

az postgres flexible-server parameter set -g myRG -n mypostgres \
  --name shared_buffers --value 1048576     # in 8KB pages

az postgres flexible-server parameter set -g myRG -n mypostgres \
  --name log_min_duration_statement --value 1000  # log queries > 1s

# Read replica (scale reads, reporting)
az postgres flexible-server replica create -g myRG \
  --replica-name mypostgres-replica \
  --source-server mypostgres --location westeurope

# Point-in-time restore
az postgres flexible-server restore -g myRG \
  -n mypostgres-restored \
  --source-server mypostgres \
  --restore-time "2026-06-10T10:00:00Z"

# Planned failover (test HA, standby becomes primary)
az postgres flexible-server restart -g myRG -n mypostgres \
  --failover Planned
```

---

### 🟡 Q78. What is Azure Database for MySQL Flexible Server?
```bash
az mysql flexible-server create -g myRG -n mymysql \
  --location eastus \
  --admin-user mysqladmin --admin-password "P@ssw0rd!2026" \
  --sku-name Standard_D4ds_v4 \
  --tier GeneralPurpose \
  --version 8.0.21 \
  --storage-size 128 --storage-auto-grow Enabled \
  --high-availability ZoneRedundant \
  --backup-retention 35 --geo-redundant-backup Enabled \
  --public-access Disabled \
  --vnet myVNet --subnet dataSubnet

# Key parameters
az mysql flexible-server parameter set -g myRG -n mymysql \
  --name slow_query_log --value ON

az mysql flexible-server parameter set -g myRG -n mymysql \
  --name long_query_time --value 2

az mysql flexible-server parameter set -g myRG -n mymysql \
  --name innodb_buffer_pool_size --value 4294967296  # 4GB

# Read replica
az mysql flexible-server replica create -g myRG \
  --replica-name mymysql-replica \
  --source-server mymysql --location westeurope

# Stop/start to save cost (for dev/test)
az mysql flexible-server stop  -g myRG -n mymysql
az mysql flexible-server start -g myRG -n mymysql
```

---

### 🟢 Q79. What is Azure Cache for Redis?
```bash
# Redis: in-memory data store for caching, sessions, pub/sub, leaderboards
# Tiers:
# Basic:      single node, no SLA, dev/test
# Standard:   primary + replica, SLA 99.9%
# Premium:    cluster, persistence, VNet, geo-replication, SLA 99.9%
# Enterprise: Redis Stack (Search, JSON, TimeSeries, BloomFilter)

az redis create -g myRG -n myredis \
  --location eastus --sku Premium --vm-size P3 \
  --shard-count 3 \              # clustering = 3 shards
  --replicas-per-master 2 \      # 2 replicas per shard
  --minimum-tls-version 1.2 \
  --zones 1 2 3

# Get connection info
REDIS_HOST=$(az redis show -g myRG -n myredis --query hostName -o tsv)
REDIS_KEY=$(az redis list-keys -g myRG -n myredis --query primaryKey -o tsv)
# Connect: $REDIS_HOST:6380,password=$REDIS_KEY,ssl=True,abortConnect=False

# RDB persistence (point-in-time snapshot)
az redis update -g myRG -n myredis \
  --set "properties.redisConfiguration.rdb-backup-enabled=true" \
  --set "properties.redisConfiguration.rdb-backup-frequency=60" \
  --set "properties.redisConfiguration.rdb-storage-connection-string=<conn>"

# AOF persistence (every write logged)
az redis update -g myRG -n myredis \
  --set "properties.redisConfiguration.aof-backup-enabled=true" \
  --set "properties.redisConfiguration.aof-storage-connection-string-0=<conn>"

# Geo-replication (Premium)
az redis geo-replication link -g myRG -n myredis \
  --server-to-link /subscriptions/<sub>/resourceGroups/myDRRG/providers/Microsoft.Cache/Redis/myredis-dr

# Export / Import (backup/restore)
az redis export -g myRG -n myredis \
  --prefix backup --file-format RDB \
  --container "https://mystorage.blob.core.windows.net/redis?sv=..."

az redis import -g myRG -n myredis \
  --files "https://mystorage.blob.core.windows.net/redis/backup0.rdb?sv=..."
```

---

### 🟡 Q80. What is Azure Synapse Analytics?
```bash
# Synapse: unified analytics — Serverless SQL + Dedicated SQL + Spark + Pipelines

az synapse workspace create -g myRG -n mySynapse \
  --storage-account mydatalake --file-system synapsecontainer \
  --sql-admin-login-user synapseadmin \
  --sql-admin-login-password "P@ssw0rd!2026" \
  --location eastus --enable-managed-virtual-network

# Dedicated SQL Pool (columnar MPP — traditional DW)
az synapse sql pool create -g myRG --workspace-name mySynapse \
  -n myDWPool --performance-level DW1000c

az synapse sql pool pause  -g myRG --workspace-name mySynapse -n myDWPool
az synapse sql pool resume -g myRG --workspace-name mySynapse -n myDWPool

# Spark Pool (distributed big data processing)
az synapse spark pool create -g myRG --workspace-name mySynapse \
  -n mySpark --spark-version 3.4 --node-size Large \
  --min-node-count 3 --max-node-count 30 --enable-auto-scale true

# Synapse Link (zero-ETL query Cosmos DB from Synapse)
az cosmosdb update -g myRG -n myCosmosAcct --enable-analytical-storage true
az cosmosdb sql container update -g myRG \
  --account-name myCosmosAcct --database-name myDB \
  -n orders --analytical-storage-ttl -1
```

```sql
-- Serverless SQL: query Parquet from ADLS Gen2
SELECT
    YEAR(OrderDate)  AS OrderYear,
    SUM(Amount)      AS TotalRevenue,
    COUNT(*)         AS OrderCount
FROM OPENROWSET(
    BULK 'https://mydatalake.dfs.core.windows.net/curated/orders/**',
    FORMAT = 'PARQUET'
) AS rows
GROUP BY YEAR(OrderDate)
ORDER BY OrderYear;

-- Dedicated SQL Pool: distributed table design
CREATE TABLE dbo.FactSales (
    SaleID      BIGINT        NOT NULL,
    CustomerID  INT           NOT NULL,
    ProductID   INT           NOT NULL,
    Amount      DECIMAL(18,2) NOT NULL,
    SaleDate    DATE          NOT NULL
)
WITH (
    DISTRIBUTION = HASH(CustomerID),       -- distribute on high-cardinality column
    CLUSTERED COLUMNSTORE INDEX,           -- columnar for analytics
    PARTITION (SaleDate RANGE RIGHT
               FOR VALUES ('2024-01-01','2025-01-01','2026-01-01'))
);

-- Small dimension table — replicate on all nodes
CREATE TABLE dbo.DimProduct (
    ProductID    INT          NOT NULL,
    ProductName  NVARCHAR(200),
    Category     NVARCHAR(100)
)
WITH (DISTRIBUTION = REPLICATE, CLUSTERED INDEX (ProductID));
```

---

### 🟡 Q81. What is Azure Databricks?
```bash
# Databricks: first-party managed Apache Spark — best for ML, Delta Lake, Unity Catalog

az databricks workspace create -g myRG -n myDBR \
  --location eastus --sku premium \
  --enable-no-public-ip \
  --vnet-name myVNet \
  --public-subnet-name databricks-public \
  --private-subnet-name databricks-private

# Databricks cluster (via databricks CLI)
databricks clusters create --json '{
  "cluster_name": "myProdCluster",
  "spark_version": "15.4.x-scala2.12",
  "node_type_id": "Standard_DS4_v2",
  "autoscale": {"min_workers": 2, "max_workers": 20},
  "auto_termination_minutes": 30,
  "azure_attributes": {
    "availability": "SPOT_WITH_FALLBACK_AZURE",
    "first_on_demand": 2
  }
}'

# Delta Lake upsert (MERGE)
from delta.tables import DeltaTable
from pyspark.sql.functions import col

delta_table = DeltaTable.forPath(spark,
    "abfss://curated@mydatalake.dfs.core.windows.net/orders")

delta_table.alias("target").merge(
    new_data.alias("source"),
    "target.OrderID = source.OrderID"
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .execute()

# Optimize and compact
spark.sql("OPTIMIZE delta.`abfss://curated@mydatalake.dfs.core.windows.net/orders` ZORDER BY (CustomerID)")
spark.sql("VACUUM delta.`abfss://curated@mydatalake.dfs.core.windows.net/orders` RETAIN 168 HOURS")
```

---

### 🟡 Q82. What is Azure Stream Analytics?
```bash
# Stream Analytics: real-time event processing using SAQL (SQL-like)
# Sources: Event Hub, IoT Hub, Blob | Sinks: Cosmos DB, SQL, Event Hub, Power BI

az stream-analytics job create -g myRG -n myASAJob \
  --location eastus --sku Standard \
  --events-outoforder-policy Adjust \
  --events-outoforder-max-delay-in-seconds 5 \
  --events-late-arrival-max-delay-in-seconds 60

az stream-analytics job start -g myRG -n myASAJob \
  --output-start-mode JobStartTime
```

```sql
-- Stream Analytics SAQL window functions

-- Tumbling Window: non-overlapping 30-second windows
SELECT deviceId, AVG(temperature) AS AvgTemp, MAX(temperature) AS MaxTemp
FROM input TIMESTAMP BY EventTime
GROUP BY deviceId, TumblingWindow(second, 30)
HAVING AVG(temperature) > 80

-- Hopping Window: overlapping (compute every 10s over last 30s)
SELECT deviceId, AVG(temperature) AS AvgTemp
FROM input TIMESTAMP BY EventTime
GROUP BY deviceId, HoppingWindow(second, 30, 10)

-- Sliding Window: fires on every event within window
SELECT deviceId, AVG(temperature) AS AvgTemp
FROM input TIMESTAMP BY EventTime
GROUP BY deviceId, SlidingWindow(second, 60)

-- Session Window: groups events by inactivity gap
SELECT userId, COUNT(*) AS PageViews
FROM clickstream TIMESTAMP BY EventTime
GROUP BY userId, SessionWindow(second, 5, 120)

-- JOIN two streams within 10-second window
SELECT o.orderId, p.productName, o.quantity
FROM orders o TIMESTAMP BY orderTime
JOIN products p TIMESTAMP BY updateTime
ON o.productId = p.productId
AND DATEDIFF(second, o, p) BETWEEN 0 AND 10

-- Anomaly detection (built-in ML)
SELECT deviceId, temperature,
    AnomalyDetection_SpikeAndDip(temperature, 95, 120, 'spikesanddips')
    OVER(PARTITION BY deviceId LIMIT DURATION(second, 120)) AS SpikeScore
FROM sensors TIMESTAMP BY EventTime
```

---

### 🟡 Q83. What is Azure SQL Database security features?
```bash
# Threat Detection (Advanced Threat Protection)
az sql db threat-policy update -g myRG -s mysqlserver -n myDB \
  --state Enabled \
  --email-account-admins Enabled \
  --storage-account mystorageaccount

# Auditing (write audit logs to Storage / LA / Event Hub)
az sql server audit-policy update -g myRG -n mysqlserver \
  --state Enabled \
  --storage-account mystorageaccount \
  --retention-days 90

# TDE with Customer-Managed Key
az sql server key create -g myRG -s mysqlserver \
  --kid https://myKV.vault.azure.net/keys/myTDEKey/version

az sql server tde-key set -g myRG -s mysqlserver \
  --server-key-type AzureKeyVault \
  --kid https://myKV.vault.azure.net/keys/myTDEKey/version

# Row-Level Security (filter rows per user)
-- CREATE FUNCTION dbo.fn_RLS_filter(@TenantId int)
-- RETURNS TABLE WITH SCHEMABINDING
-- AS RETURN (SELECT 1 AS fn_result WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS int));
-- CREATE SECURITY POLICY TenantPolicy
-- ADD FILTER PREDICATE dbo.fn_RLS_filter(TenantId) ON dbo.Orders;

# Dynamic Data Masking (mask sensitive columns from non-privileged users)
-- ALTER TABLE Customers ALTER COLUMN Email
-- ADD MASKED WITH (FUNCTION = 'email()');
-- ALTER TABLE Customers ALTER COLUMN SSN
-- ADD MASKED WITH (FUNCTION = 'partial(0,"xxx-xx-",4)');

# Vulnerability Assessment
az sql db va scan execute -g myRG -s mysqlserver -n myDB \
  --storage-account mystorageaccount

az sql db va scan list -g myRG -s mysqlserver -n myDB --output table
```

---

### 🟢 Q84. What is Azure Table Storage vs Cosmos DB?
| Feature | Azure Table Storage | Cosmos DB Table API |
|---------|-------------------|-------------------|
| Latency | Variable | <10ms guaranteed |
| Throughput | Low, shared | Configurable RU/s |
| SLA | 99.9% | 99.99% |
| Distribution | Single region | Global |
| Pricing | Cheap (GB) | Per RU + GB |
| Migration | Source | Target |
| Use when | Simple, cheap, non-critical | HA, global, SLA-bound |

---

### 🟡 Q85. What are Azure database backup and restore options?
```bash
# SQL Database backups:
# Full: weekly | Differential: 12–24 hours | Log: 5–10 minutes
# PITR retention: 1–35 days (default 7)
# LTR: weekly/monthly/yearly backups up to 10 years

# PostgreSQL backups:
# Full + WAL continuous log shipping
# PITR: up to 35 days (default 7)
# Geo-redundant backup: enabled at creation only

# Cosmos DB backups:
# Periodic: every 1-24h, retained 2-30 copies, no PITR
# Continuous 7-day: restore to any second in last 7 days
# Continuous 30-day: restore to any second in last 30 days

# Redis backups:
# RDB: snapshot at interval (Premium)
# AOF: log every write (Premium)
# Export to Blob then import to restore

# Trigger SQL DB restore
az sql db restore -g myRG -s mysqlserver \
  -n myProdDB-restored \
  --source-database myProdDB \
  --time "2026-06-10T10:00:00Z"

# Trigger PostgreSQL restore
az postgres flexible-server restore -g myRG \
  -n mypostgres-restored \
  --source-server mypostgres \
  --restore-time "2026-06-10T10:00:00Z"
```

---

# PART 5 — MONITORING

---

### 🟢 Q86. What is Azure Monitor and what are its components?
```
Azure Monitor — Unified Observability Platform
═══════════════════════════════════════════════════════════════════
Sources                     Collection              Destinations
───────────────────────────────────────────────────────────────────
Azure Resources             Platform metrics (auto)  Metrics Explorer
VMs / Arc servers           Azure Monitor Agent      Log Analytics WS
Applications (SDK)          App Insights SDK/OTel    App Insights
AKS containers              Container Insights       Log Analytics + Grafana
Activity log                Auto-collected           Activity Log Store
Service/resource logs       Diagnostic Settings      Log Analytics / Storage / EH
═══════════════════════════════════════════════════════════════════
```

```bash
# Create Log Analytics Workspace (central log store)
az monitor log-analytics workspace create -g myRG -n myLA \
  --location eastus --sku PerGB2018 --retention-time 90

# Install Azure Monitor Agent on VM
az vm extension set -g myRG --vm-name myVM \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor --version 1.0 \
  --enable-auto-upgrade true

# Data Collection Rule (what to collect and where to send)
az monitor data-collection rule create -g myRG -n myDCR --location eastus \
  --data-flows '[{"streams":["Microsoft-Perf","Microsoft-Syslog"],"destinations":["myLA"]}]' \
  --performance-counters '[{
    "name":"cpuMemDisk","streams":["Microsoft-Perf"],
    "samplingFrequencyInSeconds":60,
    "counterSpecifiers":[
      "\\Processor(_Total)\\% Processor Time",
      "\\Memory\\Available MBytes",
      "\\LogicalDisk(_Total)\\% Free Space",
      "\\LogicalDisk(_Total)\\Disk Transfers/sec"
    ]
  }]' \
  --syslog '[{"name":"syslog","streams":["Microsoft-Syslog"],
    "facilityNames":["auth","kern","daemon","cron"],"logLevels":["Warning","Error","Critical","Alert","Emergency"]}]' \
  --log-analytics '[{"name":"myLA","workspaceResourceId":"/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA"}]'

az monitor data-collection rule association create -g myRG \
  --association-name myVMDCR \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --rule-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/dataCollectionRules/myDCR

# Enable diagnostic settings on resources
az monitor diagnostic-settings create \
  --name AllLogsToLA \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKV \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA \
  --logs '[{"categoryGroup":"allLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

---

### 🟢 Q87. What is KQL and what are the essential queries?
```kusto
// ═══════════════════════════════════════════════════════════════
// ESSENTIAL KQL QUERIES FOR INTERVIEWS AND PRODUCTION USE
// ═══════════════════════════════════════════════════════════════

// 1. CPU > 90% — which VMs are hot?
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where TimeGenerated > ago(1h) and CounterValue > 90
| summarize AvgCPU = avg(CounterValue), MaxCPU = max(CounterValue)
    by Computer, bin(TimeGenerated, 5m)
| order by MaxCPU desc

// 2. Memory pressure — available memory < 512MB
Perf
| where ObjectName == "Memory" and CounterName == "Available MBytes"
| where TimeGenerated > ago(1h)
| summarize AvailMB = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| where AvailMB < 512
| order by AvailMB asc

// 3. Disk space < 20% free
Perf
| where CounterName == "% Free Space" and InstanceName != "_Total"
| where TimeGenerated > ago(30m)
| summarize FreeSpace = avg(CounterValue) by Computer, InstanceName
| where FreeSpace < 20
| project Computer, Drive = InstanceName, FreeSpacePct = FreeSpace

// 4. Offline VMs (no heartbeat for 15+ minutes)
Heartbeat
| summarize LastSeen = max(TimeGenerated) by Computer, ComputerIP
| where LastSeen < ago(15m)
| extend MinutesOffline = datetime_diff('minute', now(), LastSeen)
| project Computer, ComputerIP, LastSeen, MinutesOffline
| order by MinutesOffline desc

// 5. HTTP 5xx errors on App Service
AppServiceHTTPLogs
| where TimeGenerated > ago(1h) and ScStatus >= 500
| summarize Errors = count() by CsUriStem, ScStatus, bin(TimeGenerated, 5m)
| order by Errors desc

// 6. App Service slow responses (> 3 seconds)
AppServiceHTTPLogs
| where TimeGenerated > ago(1h)
| where TimeTaken > 3000
| summarize Count = count(), AvgMs = avg(TimeTaken), P95 = percentile(TimeTaken, 95)
    by CsUriStem
| order by P95 desc

// 7. Failed sign-ins (Entra ID) — brute force detection
SigninLogs
| where TimeGenerated > ago(1h) and ResultType != 0
| summarize Failures = count() by UserPrincipalName, AppDisplayName, ResultDescription
| where Failures > 5
| order by Failures desc

// 8. Who deleted Azure resources?
AzureActivity
| where OperationNameValue endswith "delete"
| where ActivityStatusValue == "Success"
| where TimeGenerated > ago(24h)
| project TimeGenerated, Caller, OperationNameValue, ResourceGroup, _ResourceId
| order by TimeGenerated desc

// 9. Key Vault secret access audit
AzureDiagnostics
| where ResourceType == "VAULTS" and OperationName == "SecretGet"
| project TimeGenerated, CallerIPAddress,
          User = identity_claim_upn_s, SecretName = id_s
| order by TimeGenerated desc

// 10. SQL DB high CPU/DTU usage
AzureMetrics
| where ResourceProvider == "MICROSOFT.SQL"
| where MetricName == "dtu_consumption_percent"
| summarize AvgDTU = avg(Average), MaxDTU = max(Maximum)
    by Resource, bin(TimeGenerated, 5m)
| where MaxDTU > 90

// 11. App Insights: slow requests
requests
| where timestamp > ago(1h)
| where duration > 3000   // > 3 seconds
| summarize Count = count(), AvgDuration = avg(duration),
            P95 = percentile(duration, 95) by name, resultCode
| order by P95 desc

// 12. App Insights: exceptions by type
exceptions
| where timestamp > ago(1h)
| summarize Count = count() by type, outerMessage, problemId
| order by Count desc

// 13. AKS pod restarts
KubePodInventory
| where TimeGenerated > ago(1h) and ContainerRestartCount > 0
| summarize MaxRestarts = max(ContainerRestartCount) by Name, Namespace, Computer
| order by MaxRestarts desc

// 14. AKS pending pods
KubePodInventory
| where TimeGenerated > ago(10m) and PodStatus == "Pending"
| project PodName = Name, Namespace, PodStatus, Reason = PodStatusReason
| distinct PodName, Namespace, PodStatus, Reason

// 15. NSG flow logs — top source IPs
AzureNetworkAnalytics_CL
| where SubType_s == "FlowLog"
| where TimeGenerated > ago(1h)
| summarize Connections = count(), TotalBytes = sum(InboundBytes_d + OutboundBytes_d)
    by SrcIP_s, DestIP_s, L7Protocol_s
| top 20 by TotalBytes desc

// 16. Storage account errors
StorageBlobLogs
| where TimeGenerated > ago(1h) and StatusCode >= 400
| summarize Count = count() by OperationName, StatusCode, bin(TimeGenerated, 5m)
| order by Count desc

// 17. Cosmos DB request unit consumption
AzureDiagnostics
| where ResourceType == "DATABASEACCOUNTS"
| where Category == "DataPlaneRequests"
| summarize AvgRU = avg(requestCharge_s), MaxRU = max(requestCharge_s)
    by OperationName, statusCode_s
| order by MaxRU desc

// 18. Cost by resource group
AzureActivity
| where TimeGenerated > ago(7d)
| where OperationNameValue endswith "write" and ActivityStatusValue == "Success"
| summarize ResourceCreations = count() by ResourceGroup, Caller
| order by ResourceCreations desc

// 19. Role assignment changes (security audit)
AzureActivity
| where OperationNameValue contains "roleAssignments"
| where ActivityStatusValue == "Success"
| where TimeGenerated > ago(7d)
| project TimeGenerated, Caller, OperationNameValue,
          Target = tostring(parse_json(Properties).requestbody)
| order by TimeGenerated desc

// 20. Container CPU throttling (AKS)
Perf
| where ObjectName == "K8SContainer" and CounterName == "cpuThrottledNanoCores"
| where TimeGenerated > ago(1h)
| summarize ThrottledCPU = sum(CounterValue)
    by InstanceName, Computer, bin(TimeGenerated, 5m)
| where ThrottledCPU > 0
| order by ThrottledCPU desc
```

---

### 🟢 Q88. What is Application Insights?
```bash
# App Insights: APM service tracking requests, dependencies, exceptions, users

az monitor app-insights component create \
  --app myAppInsights -g myRG --location eastus \
  --kind web --application-type web \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA

# Connection string
az monitor app-insights component show -g myRG --app myAppInsights \
  --query connectionString -o tsv

# Add to App Service
az webapp config appsettings set -g myRG -n myWebApp \
  --settings \
    APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=xxx;..." \
    ApplicationInsightsAgent_EXTENSION_VERSION="~3"

# Sampling (reduce volume and cost)
az monitor app-insights component update -g myRG --app myAppInsights \
  --sampling-percentage 20   # sample 20%, infer full traffic

# Availability test (synthetic monitoring from 5 world locations)
az monitor app-insights web-test create -g myRG \
  --app-insights-name myAppInsights -n HealthCheck \
  --defined-web-test-kind ping \
  --url https://myapp.com/health \
  --frequency 300 --timeout 30 --retry-enabled true
```

```python
# OpenTelemetry instrumentation (Python)
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace, metrics
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
import logging

configure_azure_monitor(
    connection_string="InstrumentationKey=xxx;IngestionEndpoint=..."
)
RequestsInstrumentor().instrument()    # auto-track HTTP calls
SQLAlchemyInstrumentor().instrument()  # auto-track DB queries

tracer = trace.get_tracer(__name__)
logger = logging.getLogger(__name__)

def process_order(order_id: str):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("service.name", "OrderService")
        try:
            result = do_processing(order_id)
            span.set_attribute("order.status", "success")
            return result
        except Exception as e:
            span.record_exception(e)
            span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
            logger.exception(f"Failed processing {order_id}", extra={"order_id": order_id})
            raise

# Custom metrics
meter = metrics.get_meter(__name__)
order_counter = meter.create_counter("orders.processed")
order_latency = meter.create_histogram("orders.duration_ms")

order_counter.add(1, {"status": "success", "region": "eastus"})
order_latency.record(245, {"endpoint": "/api/orders"})
```

---

### 🟢 Q89. How do you create Azure Monitor Alerts?
```bash
# Action Group (notification channels)
az monitor action-group create -g myRG -n myAG --short-name myAG \
  --action email ops ops@company.com \
  --action sms oncall 1 5551234567 \
  --action webhook pagerduty https://events.pagerduty.com/integration/<key>/enqueue

# Metric Alert (single condition)
az monitor metrics alert create -g myRG -n HighCPUAlert \
  --scopes /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --condition "avg Percentage CPU > 90" \
  --window-size 5m --evaluation-frequency 1m --severity 2 \
  --action /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/actionGroups/myAG \
  --auto-mitigate true

# Multi-condition metric alert (CPU AND memory)
az monitor metrics alert create -g myRG -n HighCPUAndMemAlert \
  --scopes /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --condition "avg Percentage CPU > 90" \
  --condition "avg Available Memory Bytes < 536870912" \
  --window-size 5m --evaluation-frequency 1m --severity 1 \
  --action /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/actionGroups/myAG

# Log Search Alert (KQL-based)
az monitor scheduled-query create -g myRG -n HttpErrorsAlert \
  --scopes /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA \
  --condition-query "AppServiceHTTPLogs | where ScStatus >= 500 | summarize Count=count() | where Count > 50" \
  --condition "count Count > 0" \
  --evaluation-frequency 5m --window-duration 15m --severity 1 \
  --action-groups /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/actionGroups/myAG

# Activity Log Alert (config/security changes)
az monitor activity-log alert create -g myRG -n NSGRuleChanged \
  --scopes /subscriptions/<sub> \
  --condition \
    category=Administrative \
    operationName=Microsoft.Network/networkSecurityGroups/securityRules/write \
    status=Succeeded \
  --action-group /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/actionGroups/myAG

# Service Health Alert
az monitor activity-log alert create -g myRG -n ServiceIncident \
  --scopes /subscriptions/<sub> \
  --condition category=ServiceHealth properties.incidentType=Incident \
  --action-group /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/actionGroups/myAG

# Alert Severity levels:
# 0=Critical | 1=Error | 2=Warning | 3=Informational | 4=Verbose

# Alert Processing Rule (suppress during maintenance)
az monitor alert-processing-rule create -g myRG -n WeekendSuppression \
  --scopes /subscriptions/<sub>/resourceGroups/myRG \
  --rule-type Suppression \
  --schedule-recurrence-type Weekly \
  --schedule-recurrence "daysOfWeek=Saturday,Sunday" \
  --schedule-start-datetime "2026-06-13T22:00:00" \
  --schedule-end-datetime "2026-06-14T06:00:00"
```

---

### 🟢 Q90. What is Azure Advisor?
```bash
# Advisor: AI-powered personalised cloud consultant — 5 recommendation categories

az advisor recommendation list --filter "Category eq 'Cost'" --output table
az advisor recommendation list --filter "Category eq 'Security'" --output table
az advisor recommendation list --filter "Category eq 'HighAvailability'" --output table
az advisor recommendation list --filter "Category eq 'Performance'" --output table
az advisor recommendation list --filter "Category eq 'OperationalExcellence'" --output table
az advisor score list --output table

# Top recommendations per category:
# COST:     right-size underused VMs, buy RIs, delete orphaned disks, stop idle GW
# SECURITY: enable MFA, enable Defender plans, rotate keys, disable public blobs
# HA:       add Availability Zones, enable geo-redundant backup, autoscale
# PERF:     Premium SSD for IO workloads, Redis caching, CDN for static assets
# OPS:      enable diagnostics, use managed identity, tag all resources

# Dismiss or snooze
az advisor recommendation suppress -g myRG \
  --recommendation-id /subscriptions/<sub>/.../recommendations/<id>
```

---

### 🟡 Q91. What is Service Health vs Resource Health?
```bash
# Azure Status:     global Azure platform status (status.azure.com) — public
# Service Health:   incidents impacting YOUR subscription and regions
# Resource Health:  health status of a SPECIFIC resource

# Resource Health statuses:
# Available:   resource healthy and working normally
# Unavailable: platform issue (outage or maintenance)
# Unknown:     no health data received for > 10 min
# Degraded:    working but with reduced performance

# Resource Health check
az resource-health show -g myRG -n myVM \
  --resource-type Microsoft.Compute/virtualMachines \
  --query "{Status:availabilityStatus.availabilityState,Summary:availabilityStatus.summary}"

# Service Health alert (notify on incidents)
az monitor activity-log alert create -g myRG -n ServiceHealthAlert \
  --scopes /subscriptions/<sub> \
  --condition category=ServiceHealth properties.incidentType=Incident \
  --action-group /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/actionGroups/myAG

# Resource Health alert (notify when VM goes Unavailable)
az monitor activity-log alert create -g myRG -n VMUnavailableAlert \
  --scopes /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --condition category=ResourceHealth properties.currentHealthStatus=Unavailable \
  --action-group /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/actionGroups/myAG
```

---

### 🟡 Q92. What is Azure Monitor Workbooks?
```bash
# Workbooks: interactive reports combining KQL queries, metrics, text, parameters
# Use for: custom dashboards, incident reports, operational reviews

# Pre-built workbook templates for:
# VM Performance, App Insights failure analysis, Azure AD sign-in analysis,
# AKS health, SQL Insights, Cost analysis, Security overview, Network monitor

# Create workbook via REST/Bicep
az resource create -g myRG \
  --resource-type Microsoft.Insights/workbooks \
  --api-version 2022-04-01 -n $(uuidgen) \
  --properties '{
    "displayName":"VM Performance Overview",
    "category":"workbook",
    "sourceId":"/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA",
    "serializedData":"<workbook-json-string>"
  }'

# Workbooks support:
# Parameters: dropdowns, time range, resource pickers
# Query tiles: KQL queries rendered as tables, charts, grids, tiles
# Metrics tiles: Azure Monitor metrics charts
# Text tiles: markdown documentation inline with data
# Links: drill-down to other workbooks or Azure resources
# Export to PDF or share via link
```

---

### 🟡 Q93. What is Azure Monitor Managed Prometheus and Grafana?
```bash
# Managed Prometheus: scrape Kubernetes metrics without managing Prometheus infra
# Managed Grafana: fully hosted Grafana with Azure AD integration and pre-built dashboards

# Create Azure Monitor Workspace (stores Prometheus metrics)
az monitor account create -g myRG -n myAMW --location eastus

# Create Managed Grafana
az grafana create -g myRG -n myGrafana \
  --location eastus --sku Standard \
  --deterministic-outbound-ip Enabled

# Grant Grafana access to Monitor workspace
az role assignment create \
  --assignee $(az grafana show -g myRG -n myGrafana --query identity.principalId -o tsv) \
  --role "Monitoring Reader" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/microsoft.monitor/accounts/myAMW

# Enable on AKS (installs prometheus-collector daemonset automatically)
az aks update -g myRG -n myAKS \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/microsoft.monitor/accounts/myAMW \
  --grafana-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Dashboard/grafana/myGrafana

# Get Grafana URL
az grafana show -g myRG -n myGrafana --query properties.endpoint -o tsv

# Add Grafana admin
az grafana user create -g myRG -n myGrafana \
  --login analyst@company.com --role Viewer

# Pre-built dashboards available:
# Kubernetes / Nodes / Workloads / Namespaces / Deployments / StatefulSets
# Azure Data Explorer, Azure SQL, Storage, CosmosDB, App Service
```

---

### 🟡 Q94. What is Change Analysis?
```bash
# Change Analysis: tracks all changes that may cause incidents
# Sources: ARM resource changes, App Service file config changes, guest OS (via AMA)

# Enable
az rest --method put \
  --url "https://management.azure.com/subscriptions/<sub>/providers/Microsoft.ChangeAnalysis/profile/default?api-version=2020-04-01-preview" \
  --body '{"properties":{"state":"enabled"}}'

# View in Portal: Monitor → Change Analysis → select time range
# Shows: property changes, new resources, deleted resources, config drift

# KQL query for resource changes
ResourceChanges
| where TimeGenerated > ago(24h)
| where resourceGroup =~ "myRG"
| project TimeGenerated,
          ResourceType = type,
          ResourceName = name,
          ChangeType = properties.changeType,
          OldValue = properties.changes[0].previousValue,
          NewValue = properties.changes[0].newValue
| order by TimeGenerated desc

# Use cases:
# - Root cause analysis: what changed before the incident?
# - Compliance: detect unauthorised config changes
# - Release validation: confirm expected changes were applied
```

---

### 🟡 Q95. What is Log Analytics workspace design?
```bash
# Design considerations:
# Single workspace:    simplest, easy cross-service correlation — recommended default
# Multiple workspaces: data residency (EU data in EU), security isolation (SOC team only)

# Pricing tiers:
# PerGB2018:            pay per GB ingested (most common)
# CapacityReservation:  commit to 100–50000 GB/day (up to 25% discount)

# Retention:
# Default: 30 days hot (free) + 0 days archive
# Extended: up to 730 days hot at extra cost
# Archive:  730 days – 12 years (very low cost, async query only)

az monitor log-analytics workspace create -g myRG -n myLA \
  --sku PerGB2018 --retention-time 90 --location eastus

# Update retention
az monitor log-analytics workspace update -g myRG -n myLA \
  --retention-time 365

# Per-table retention (different tables need different retention)
az monitor log-analytics workspace table update -g myRG \
  --workspace-name myLA --name SecurityEvent \
  --retention-time 365 --total-retention-time 730

# Data export (to Storage or Event Hub for archiving / SIEM)
az monitor log-analytics workspace data-export create -g myRG \
  --workspace-name myLA -n exportSecurity \
  --tables SecurityEvent AzureActivity SigninLogs AuditLogs \
  --destination /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount \
  --enable true

# Cost optimisation:
# 1. DCR filtering: only collect relevant events (exclude verbose Debug logs)
# 2. Sampling App Insights at 10–20% for high-traffic apps
# 3. Commitment tiers for >100 GB/day ingestion
# 4. Archive old data vs retaining as hot
# 5. Table-level retention (SecurityEvent needs longer than AppLogs)
```

---

## COMPLETE Q&A INDEX

| Q# | Question | Level | Topic |
|----|---------|-------|-------|
| Q1 | What is Azure VM and what problem does it solve? | 🟢 | Compute |
| Q2 | VM series — B/D/E/F/L/M/N/H/DC use cases | 🟢 | Compute |
| Q3 | Stop vs Deallocate — billing impact | 🟢 | Compute |
| Q4 | VM availability: Zones vs Availability Sets | 🟢 | Compute |
| Q5 | VM image types and Shared Image Gallery | 🟢 | Compute |
| Q6 | Spot VMs — discount, eviction, use cases | 🟡 | Compute |
| Q7 | How do you resize a VM? | 🟡 | Compute |
| Q8 | VM Extensions — AMA, CustomScript, KeyVault | 🟡 | Compute |
| Q9 | Azure Dedicated Host | 🟡 | Compute |
| Q10 | Trusted Launch and Confidential VMs | 🔴 | Compute |
| Q11 | VMSS — Flexible orchestration, autoscale | 🟡 | Compute |
| Q12 | Compute Fleet — multi-SKU Spot pool | 🟡 | Compute |
| Q13 | App Service — plans/tiers explained | 🟢 | Compute |
| Q14 | Deployment slots — swap, canary, sticky settings | 🟡 | Compute |
| Q15 | VNet integration and Private Endpoint in App Service | 🟡 | Compute |
| Q16 | Azure Functions — plans, triggers, Python v2 | 🟡 | Compute |
| Q17 | Azure Container Apps — KEDA, Dapr, jobs | 🟡 | Compute |
| Q18 | Azure Container Instances | 🟡 | Compute |
| Q19 | AKS — production cluster, node pools | 🟢 | Compute |
| Q20 | ACR — tasks, geo-replication, lifecycle | 🟢 | Compute |
| Q21 | Azure Batch — HPC, autoscale formula | 🟡 | Compute |
| Q22 | Container Apps vs ACI vs AKS vs App Service | 🟡 | Compute |
| Q23 | Azure Hybrid Benefit | 🟢 | Compute |
| Q24 | VM run commands and serial console | 🟡 | Compute |
| Q25 | Managed Disks — all types, snapshot, CMK | 🟡 | Compute |
| Q26 | VNet — subnets, service endpoints, delegation | 🟢 | Networking |
| Q27 | VNet peering — transitive limitations | 🟢 | Networking |
| Q28 | User-Defined Routes (UDR) | 🟢 | Networking |
| Q29 | NSG — rules, priorities, defaults | 🟢 | Networking |
| Q30 | Application Security Groups (ASG) + flow logs | 🟡 | Networking |
| Q31 | Standard Load Balancer — HA ports, outbound | 🟢 | Networking |
| Q32 | Application Gateway — WAF, URL routing, rewrite | 🟡 | Networking |
| Q33 | Azure Front Door — global L7, WAF, caching | 🟡 | Networking |
| Q34 | Traffic Manager — routing methods | 🟡 | Networking |
| Q35 | VPN Gateway — S2S, P2S, active-active | 🟡 | Networking |
| Q36 | ExpressRoute — peering types, Global Reach | 🟡 | Networking |
| Q37 | Azure Bastion — tunneling, native client | 🟢 | Networking |
| Q38 | Azure Firewall Premium — IDPS, DNAT, FQDN | 🟡 | Networking |
| Q39 | Azure DNS — public, private, alias, PE zones | 🟢 | Networking |
| Q40 | NAT Gateway — SNAT ports, IP prefix | 🟢 | Networking |
| Q41 | Private Link vs Service Endpoints | 🟡 | Networking |
| Q42 | Azure CDN — rules engine, custom domain | 🟡 | Networking |
| Q43 | Network Watcher — IP flow, packet capture | 🟡 | Networking |
| Q44 | Service Endpoint vs Private Endpoint table | 🟢 | Networking |
| Q45 | Azure Virtual WAN | 🟡 | Networking |
| Q46 | Azure DDoS Protection — tiers, attack types | 🟡 | Networking |
| Q47 | LB vs App GW vs Front Door comparison | 🟢 | Networking |
| Q48 | Azure DNS Private Resolver | 🟡 | Networking |
| Q49 | VNet peering vs VPN Gateway transit | 🟡 | Networking |
| Q50 | Networking troubleshooting systematic approach | 🟢 | Networking |
| Q51 | Storage account types and kinds | 🟢 | Storage |
| Q52 | Storage redundancy — LRS/ZRS/GRS/GZRS | 🟢 | Storage |
| Q53 | Blob tiers — Hot/Cool/Cold/Archive, lifecycle | 🟢 | Storage |
| Q54 | SAS tokens — types, stored access policy | 🟡 | Storage |
| Q55 | Blob versioning and soft delete | 🟡 | Storage |
| Q56 | ADLS Gen2 — HNS, POSIX ACLs | 🟡 | Storage |
| Q57 | Azure Files — SMB/NFS, File Sync | 🟢 | Storage |
| Q58 | Queue Storage and Table Storage | 🟢 | Storage |
| Q59 | Managed Disks — types, v2, Ultra, CMK | 🟡 | Storage |
| Q60 | Azure Backup — policies, restore | 🟡 | Storage |
| Q61 | Azure Import/Export and Data Box | 🟡 | Storage |
| Q62 | AzCopy — bulk transfer, sync, benchmark | 🟡 | Storage |
| Q63 | Storage firewall and network rules | 🟡 | Storage |
| Q64 | Storage encryption — MMK vs CMK | 🟡 | Storage |
| Q65 | Object replication — async cross-account | 🟡 | Storage |
| Q66 | Static website hosting | 🟢 | Storage |
| Q67 | Change feed | 🟡 | Storage |
| Q68 | Storage monitoring and KQL | 🟢 | Storage |
| Q69 | AzCopy vs Storage Explorer vs Portal | 🟢 | Storage |
| Q70 | Blob vs Files vs Queue vs Table | 🟢 | Storage |
| Q71 | Azure SQL DB — deployment models, serverless, BC | 🟢 | Databases |
| Q72 | SQL Elastic Pool — SaaS multi-tenant pattern | 🟡 | Databases |
| Q73 | Geo-replication and failover groups | 🟡 | Databases |
| Q74 | SQL Managed Instance — lift-and-shift | 🟡 | Databases |
| Q75 | Cosmos DB — consistency levels, partition key | 🟡 | Databases |
| Q76 | Cosmos DB RU model — provisioned vs autoscale vs serverless | 🟡 | Databases |
| Q77 | PostgreSQL Flexible Server — HA, PgBouncer, replicas | 🟡 | Databases |
| Q78 | MySQL Flexible Server | 🟡 | Databases |
| Q79 | Azure Cache for Redis — tiers, cluster, persistence | 🟢 | Databases |
| Q80 | Synapse Analytics — SQL pool, Spark, distributed tables | 🟡 | Databases |
| Q81 | Azure Databricks — Delta Lake, ML | 🟡 | Databases |
| Q82 | Stream Analytics — SAQL window functions | 🟡 | Databases |
| Q83 | SQL DB security — TDE, RLS, DDM, VA | 🟡 | Databases |
| Q84 | Table Storage vs Cosmos DB Table API | 🟢 | Databases |
| Q85 | Database backup and restore options | 🟡 | Databases |
| Q86 | Azure Monitor — architecture and components | 🟢 | Monitoring |
| Q87 | KQL — 20 essential production queries | 🟢 | Monitoring |
| Q88 | Application Insights — OTel, custom metrics | 🟢 | Monitoring |
| Q89 | Monitor Alerts — metric, log, activity, SH | 🟢 | Monitoring |
| Q90 | Azure Advisor — 5 categories | 🟢 | Monitoring |
| Q91 | Service Health vs Resource Health | 🟢 | Monitoring |
| Q92 | Azure Monitor Workbooks | 🟡 | Monitoring |
| Q93 | Managed Prometheus and Grafana for AKS | 🟡 | Monitoring |
| Q94 | Change Analysis | 🟡 | Monitoring |
| Q95 | Log Analytics workspace design and cost | 🟡 | Monitoring |

---
*Generated June 2026 | Microsoft Learn aligned | 95 Q&A across 5 topics*
*🟢 25 Basic | 🟡 58 Intermediate | 🔴 2 Advanced*

---

# GAP-FILL PART 1 — COMPUTE (Q96–Q115)

---

### 🟡 Q96. What is App Service Environment (ASE) v3?
**Answer:** ASE v3 is a fully isolated, single-tenant App Service deployment inside your own VNet. Both inbound and outbound traffic stay within your VNet — no shared infrastructure.

```bash
# Requirements: /24 or larger subnet, delegated to Microsoft.Web/hostingEnvironments
az appservice ase create -g myRG -n myASEv3 \
  --vnet-name myVNet --subnet myASESubnet \
  --kind asev3 --os-preference Linux --zone-redundant true

# Create Isolated v2 plan inside ASE
az appservice plan create -g myRG -n myIsolatedPlan \
  --app-service-environment myASEv3 --sku I2V2 --is-linux

az webapp create -g myRG --plan myIsolatedPlan -n myIsolatedApp \
  --runtime "PYTHON:3.12"

# ASE types:
# External ASE: internet-facing, public IP on ILB
# ILB ASE: internal load balancer only — no internet exposure at all

# Zone-redundant ASE: minimum 9 Isolated v2 instances across 3 AZs
# Benefits: dedicated hardware, no noisy neighbours, HIPAA/PCI compliance
```

---

### 🟡 Q97. What is App Service backup, custom domain, TLS, and health check?
```bash
# Backup (requires Standard tier or above)
az webapp config backup create -g myRG --webapp-name myWebApp \
  --backup-name myBackup \
  --container-url "https://mystorageaccount.blob.core.windows.net/backups?sv=..."

# Scheduled backups
az webapp config backup update -g myRG --webapp-name myWebApp \
  --container-url "https://mystorageaccount.blob.core.windows.net/backups?sv=..." \
  --frequency 1d --retain-one true --retention-period-in-days 30

# Restore from backup
az webapp config backup restore -g myRG --webapp-name myWebApp-restored \
  --container-url "https://mystorageaccount.blob.core.windows.net/backups?sv=..." \
  --backup-name myBackup --db-type None

# Custom domain
az webapp config hostname add -g myRG --webapp-name myWebApp \
  --hostname www.example.com

# TLS binding (App Service Managed Certificate — free)
az webapp config ssl create -g myRG -n myWebApp \
  --hostname www.example.com

az webapp config ssl bind -g myRG -n myWebApp \
  --certificate-thumbprint <thumb> --ssl-type SNI

# Force HTTPS
az webapp update -g myRG -n myWebApp --https-only true

# Health check (unhealthy instances removed from LB after 2 failed probes)
az webapp config set -g myRG -n myWebApp --health-check-path "/api/health"

# App Service autoscaling
az monitor autoscale create -g myRG --resource myPlan \
  --resource-type Microsoft.Web/serverFarms \
  --name webAutoscale --min-count 2 --max-count 20 --count 3

az monitor autoscale rule create -g myRG --autoscale-name webAutoscale \
  --condition "HttpQueueLength > 100 avg 5m" --scale out 2

az monitor autoscale rule create -g myRG --autoscale-name webAutoscale \
  --condition "CpuPercentage < 25 avg 10m" --scale in 1
```

---

### 🔴 Q98. What are Azure Durable Functions?
**Answer:** Durable Functions is an extension of Azure Functions for writing stateful, serverless orchestration workflows in code. No infrastructure or state management needed.

```python
# Durable Function patterns:

# 1. Function Chaining (sequential steps)
import azure.durable_functions as df

def orchestrator_function(context: df.DurableOrchestrationContext):
    # Each activity runs in sequence
    result1 = yield context.call_activity("StepOne", "input")
    result2 = yield context.call_activity("StepTwo", result1)
    result3 = yield context.call_activity("StepThree", result2)
    return result3

main = df.Orchestrator.create(orchestrator_function)

# 2. Fan-out / Fan-in (parallel execution)
def orchestrator_fanout(context: df.DurableOrchestrationContext):
    items = yield context.call_activity("GetItems", None)
    # Launch all in parallel
    tasks = [context.call_activity("ProcessItem", item) for item in items]
    results = yield context.task_all(tasks)   # wait for ALL
    return results

# 3. Human interaction / approval workflow
def orchestrator_approval(context: df.DurableOrchestrationContext):
    yield context.call_activity("SendApprovalEmail", "approver@company.com")
    try:
        # Wait up to 48 hours for approval event
        approval = yield context.wait_for_external_event("Approval",
                         timeout=timedelta(hours=48))
        if approval:
            yield context.call_activity("GrantAccess", None)
    except df.TimeoutError:
        yield context.call_activity("EscalateToManager", None)

# 4. Monitor (poll until done)
def orchestrator_monitor(context: df.DurableOrchestrationContext):
    while True:
        status = yield context.call_activity("CheckJobStatus", job_id)
        if status == "Completed":
            return yield context.call_activity("GetJobResult", job_id)
        # Wait 30 seconds before next poll
        next_check = context.current_utc_datetime + timedelta(seconds=30)
        yield context.create_timer(next_check)

# 5. Eternal Orchestration (stateful singleton — runs forever)
def eternal_orchestrator(context: df.DurableOrchestrationContext):
    yield context.call_activity("DailyCleanup", None)
    # Schedule next run in 24 hours
    next_run = context.current_utc_datetime + timedelta(hours=24)
    yield context.create_timer(next_run)
    context.continue_as_new(None)  # restart with cleared history

# Durable entities (stateful objects)
class Counter:
    def __init__(self):
        self.value = 0
    def add(self, amount):
        self.value += amount
    def reset(self):
        self.value = 0
    def get(self):
        return self.value
```

---

### 🔴 Q99. What are AKS node pools, taints, and tolerations?
```bash
# System node pool: runs kube-system pods (CoreDNS, metrics-server, etc.)
# User node pool:   runs application workloads

# Create AKS with separate system and user pools
az aks create -g myRG -n myAKS \
  --node-count 2 \
  --node-vm-size Standard_D4s_v5 \
  --nodepool-name system \
  --nodepool-taints CriticalAddonsOnly=true:NoSchedule \  # no app pods here
  --generate-ssh-keys

# Add user node pools
az aks nodepool add -g myRG --cluster-name myAKS \
  -n apppool \
  --node-vm-size Standard_D8s_v5 \
  --node-count 3 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 2 --max-count 20 \
  --mode User

az aks nodepool add -g myRG --cluster-name myAKS \
  -n gpupool \
  --node-vm-size Standard_NC6s_v3 \
  --node-count 0 \
  --enable-cluster-autoscaler --min-count 0 --max-count 10 \
  --node-taints "sku=gpu:NoSchedule" \
  --labels "accelerator=nvidia" \
  --mode User

az aks nodepool add -g myRG --cluster-name myAKS \
  -n spotpool \
  --node-vm-size Standard_D4s_v5 \
  --priority Spot --eviction-policy Delete \
  --spot-max-price -1 \
  --node-taints "kubernetes.azure.com/scalesetpriority=spot:NoSchedule" \
  --mode User
```

```yaml
# Kubernetes: pod scheduling with nodeSelector, taints, tolerations
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-ml-training
spec:
  template:
    spec:
      nodeSelector:
        accelerator: nvidia           # only schedule on GPU nodes
      tolerations:
      - key: "sku"
        operator: "Equal"
        value: "gpu"
        effect: "NoSchedule"          # tolerate the GPU taint
      - key: "kubernetes.azure.com/scalesetpriority"
        operator: "Equal"
        value: "spot"
        effect: "NoSchedule"
      containers:
      - name: trainer
        image: myacr.azurecr.io/ml-trainer:latest
        resources:
          requests:
            nvidia.com/gpu: 1
          limits:
            nvidia.com/gpu: 1
```

---

### 🔴 Q100. What is AKS Workload Identity?
```bash
# Workload Identity: pods get Entra ID tokens without managing secrets
# Replaces: pod identity, manually mounted service principal credentials

# Enable on AKS cluster
az aks create -g myRG -n myAKS \
  --enable-workload-identity \
  --enable-oidc-issuer \
  --generate-ssh-keys

# Get OIDC issuer URL
OIDC_ISSUER=$(az aks show -g myRG -n myAKS \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

# Create User-Assigned Managed Identity
az identity create -g myRG -n myPodIdentity --location eastus
CLIENT_ID=$(az identity show -g myRG -n myPodIdentity --query clientId -o tsv)
PRINCIPAL_ID=$(az identity show -g myRG -n myPodIdentity --query principalId -o tsv)

# Grant identity access to Key Vault
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKV

# Create federated credential (link K8s service account to Managed Identity)
az identity federated-credential create \
  --identity-name myPodIdentity -g myRG \
  --name myFederatedCred \
  --issuer $OIDC_ISSUER \
  --subject "system:serviceaccount:default:myapp-sa" \
  --audience "api://AzureADTokenExchange"
```

```yaml
# Create ServiceAccount annotated with MI client ID
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: default
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID>"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      labels:
        azure.workload.identity/use: "true"   # inject OIDC token
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: myapp
        image: myacr.azurecr.io/myapp:latest
        # DefaultAzureCredential auto-detects the OIDC token — no secrets needed
```

```python
# App code — no credentials, just DefaultAzureCredential
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()  # automatically uses OIDC token from pod
client = SecretClient(vault_url="https://myKV.vault.azure.net", credential=credential)
secret = client.get_secret("db-password").value
```

---

### 🟡 Q101. What are AKS persistent volumes with Azure CSI drivers?
```yaml
# Azure Disk CSI (ReadWriteOnce — single pod)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium-retain
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
  kind: managed
  cachingMode: ReadOnly
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-disk-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: managed-premium-retain
  resources:
    requests:
      storage: 100Gi
---
# Azure Files CSI (ReadWriteMany — multiple pods, shared)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-files-premium
provisioner: file.csi.azure.com
parameters:
  skuName: Premium_LRS
  protocol: nfs        # NFS 4.1 for Linux | smb for Windows
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - nconnect=8         # parallel NFS connections
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-files-pvc
spec:
  accessModes: [ReadWriteMany]
  storageClassName: azure-files-premium
  resources:
    requests:
      storage: 100Gi
```

---

### 🟡 Q102. What is AKS networking — CNI, Overlay, Cilium?
```bash
# AKS network plugins:
# kubenet:       basic, pod IPs from overlay (not VNet), no VNet integration
# azure (CNI):   pod IPs from VNet subnet — enables direct VNet routing
# azure overlay: pod IPs from overlay, but node IPs in VNet — saves IP space
# cilium (CNI):  eBPF-based, advanced network policies, Hubble observability

# Azure CNI Overlay (recommended — saves VNet IP addresses)
az aks create -g myRG -n myAKS \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16 \       # overlay pod IP space (not in VNet)
  --generate-ssh-keys

# Azure CNI + Cilium (advanced observability + Kubernetes network policies)
az aks create -g myRG -n myAKS \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium \       # cilium instead of azure dataplane
  --generate-ssh-keys

# Network Policy (restrict pod-to-pod communication)
az aks create -g myRG -n myAKS \
  --network-plugin azure \
  --network-policy azure             # azure | calico | cilium
```

```yaml
# Kubernetes NetworkPolicy — deny all, allow only specific
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp       # only myapp pods can reach postgres
    ports:
    - protocol: TCP
      port: 5432
  egress: []               # no egress from postgres pods
```

---

### 🟡 Q103. What is VM disk encryption — ADE vs SSE?
```bash
# SSE (Server-Side Encryption): encrypts data on Azure storage
#   Always ON, transparent, protects data at rest at storage level
#   Key options: Platform-managed (default) or Customer-managed (CMK)

# ADE (Azure Disk Encryption): OS-level encryption (BitLocker/DM-Crypt)
#   Encrypts the VHD using OS-native tools + Key Vault
#   Required for: FIPS 140-2, compliance requiring OS-level encryption

# Enable ADE on Linux VM
az vm encryption enable -g myRG -n myVM \
  --disk-encryption-keyvault myKV \
  --key-encryption-key myKEK \         # optional key encryption key
  --volume-type All                    # OS | Data | All

# Check encryption status
az vm encryption show -g myRG -n myVM

# SSE with CMK (Customer-Managed Key)
az disk-encryption-set create -g myRG -n myDES \
  --key-url https://myKV.vault.azure.net/keys/myDiskKey/version \
  --source-vault myKV \
  --encryption-type EncryptionAtRestWithCustomerKey

# Apply DES to new disk
az disk create -g myRG -n myEncryptedDisk \
  --size-gb 512 --sku Premium_LRS --zone 1 \
  --encryption-type EncryptionAtRestWithCustomerKey \
  --disk-encryption-set myDES

# SSE vs ADE:
# SSE: storage-level, always on, zero performance impact, managed by Azure
# ADE: OS-level, BitLocker (Win)/DM-Crypt (Linux), needed for boot disk compliance
# For most compliance: SSE with CMK is sufficient
# For highest compliance (FIPS 140-2, boot encryption): ADE + SSE
```

---

### 🟡 Q104. What is VM proximity placement group?
```bash
# Proximity Placement Group (PPG): co-locate VMs in same Azure datacenter
# Use for: latency-sensitive apps (HPC, trading, SAP), InfiniBand clusters

az ppg create -g myRG -n myPPG --type Standard --location eastus

az vm create -g myRG -n myVM1 \
  --ppg myPPG --zone 1 \
  --image Ubuntu2204 --size Standard_D4s_v5 --generate-ssh-keys

az vm create -g myRG -n myVM2 \
  --ppg myPPG --zone 1 \
  --image Ubuntu2204 --size Standard_D4s_v5 --generate-ssh-keys

# PPG with VMSS
az vmss create -g myRG -n myVMSS \
  --ppg myPPG \
  --image Ubuntu2204 --vm-sku Standard_D4s_v5 --instance-count 3

# Verify co-location
az ppg show -g myRG -n myPPG \
  --query "virtualMachines[].id" --output table

# PPG constraint:
# - All VMs must be same region and zone
# - First VM allocated defines physical location — subsequent VMs must fit
# - Remove from PPG before resizing to a different VM family
```

---

### 🟡 Q105. What are VMSS upgrade policies?
```bash
# Upgrade policies:
# Manual:      instances NOT upgraded until you explicitly update them
# Automatic:   Azure upgrades instances as soon as model changes (no surge, risky)
# Rolling:     upgrades in configurable batches — RECOMMENDED for production

# Rolling upgrade (most control)
az vmss update -g myRG -n myVMSS \
  --set upgradePolicy.mode=Rolling \
  --set upgradePolicy.rollingUpgradePolicy.maxBatchInstancePercent=20 \
  --set upgradePolicy.rollingUpgradePolicy.maxUnhealthyInstancePercent=20 \
  --set upgradePolicy.rollingUpgradePolicy.maxUnhealthyUpgradedInstancePercent=20 \
  --set upgradePolicy.rollingUpgradePolicy.pauseTimeBetweenBatches=PT5M

# Trigger rolling upgrade after model change
az vmss rolling-upgrade start -g myRG -n myVMSS

# Monitor rolling upgrade
az vmss rolling-upgrade get-latest -g myRG -n myVMSS

# Cancel rolling upgrade (if issues)
az vmss rolling-upgrade cancel -g myRG -n myVMSS

# Manual upgrade (safe — you control when instances update)
az vmss update -g myRG -n myVMSS --set upgradePolicy.mode=Manual

# After changing VMSS model:
az vmss update-instances -g myRG -n myVMSS --instance-ids "0 1 2"  # specific
az vmss update-instances -g myRG -n myVMSS --instance-ids "*"       # all
```


---

# GAP-FILL PART 2 — NETWORKING (Q106–Q115)

---

### 🟡 Q106. What are Service Tags in NSG rules?
```bash
# Service Tags: named groups of IP prefixes for Azure services
# Updated automatically by Microsoft — no manual IP management

# Common service tags:
# Internet:           all public internet IPs
# VirtualNetwork:     all VNet address spaces (including peered)
# AzureLoadBalancer:  Azure infrastructure IPs for health probes
# AzureCloud:         all Azure datacenter IPs
# AzureMonitor:       Log Analytics, App Insights ingestion endpoints
# AzureActiveDirectory: Entra ID endpoints
# AppService:         App Service outbound IPs
# Sql:                Azure SQL / SQL MI endpoints
# Storage:            Azure Storage endpoints
# EventHub:           Event Hub endpoints
# ServiceBus:         Service Bus endpoints
# KeyVault:           Key Vault endpoints
# GatewayManager:     Azure Gateway Manager for VPN/App GW

# Allow AzureMonitor agent to send data to Azure Monitor
az network nsg rule create -g myRG --nsg-name myNSG \
  -n Allow-AzureMonitor-Out --priority 100 --direction Outbound --access Allow \
  --protocol Tcp \
  --source-address-prefix VirtualNetwork \
  --destination-address-prefix AzureMonitor \
  --destination-port-range 443

# Block all internet outbound except to Storage and SQL
az network nsg rule create -g myRG --nsg-name myNSG \
  -n Allow-Storage-Out --priority 110 --direction Outbound --access Allow \
  --protocol Tcp --destination-address-prefix Storage \
  --destination-port-range 443

az network nsg rule create -g myRG --nsg-name myNSG \
  -n Allow-SQL-Out --priority 120 --direction Outbound --access Allow \
  --protocol Tcp --destination-address-prefix Sql \
  --destination-port-range 1433

az network nsg rule create -g myRG --nsg-name myNSG \
  -n Deny-Internet-Out --priority 4000 --direction Outbound --access Deny \
  --protocol "*" --destination-address-prefix Internet \
  --destination-port-range "*"
```

---

### 🟡 Q107. What is Azure Route Server?
```bash
# Route Server: enables dynamic routing (BGP) between NVAs and Azure VNet
# Without Route Server: NVA must use static UDRs — hard to manage at scale

az network routeserver create -g myRG -n myRouteServer \
  --location eastus \
  --hosted-subnet /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/RouteServerSubnet \
  --public-ip-address myRouteServerIP

# Peer with NVA (Palo Alto, Fortinet, Cisco CSR, etc.)
az network routeserver peering create -g myRG \
  --routeserver myRouteServer \
  --name myNVAPeer \
  --peer-ip 10.0.0.4 \    # NVA BGP speaker IP
  --peer-asn 65001

# Enable branch-to-branch (for SD-WAN / VWAN-like topology)
az network routeserver update -g myRG -n myRouteServer \
  --allow-b2b-traffic true

# Route Server use cases:
# - SD-WAN integration: NVA advertises on-prem routes dynamically
# - BGP-based failover between active/passive NVAs
# - Multi-NVA active/active with ECMP routing
```

---

### 🟡 Q108. Hub-spoke vs Virtual WAN vs Mesh — when to use each?
| Topology | Management | Transitive | Max Scale | Best For |
|---------|----------|----------|---------|---------|
| **Hub-Spoke** | Manual UDR, FW | Via Firewall | 500 VNets | Small–medium enterprise |
| **Virtual WAN** | Automatic | Native | 1000s VNets | Large enterprise, SD-WAN |
| **Mesh (full)** | N×(N-1) peerings | Native | ~50 VNets | Small, low-latency |

```bash
# Hub-Spoke: one hub VNet with Firewall/VPN, spokes peer to hub
# Spokes can reach on-prem via hub gateway (gateway transit)
# Spoke-to-spoke: traffic goes Spoke → Firewall → Spoke (inspection)

# Virtual WAN: Microsoft manages hub, routing, Firewall, VPN at global scale
# Automatic transitive: SpokA → Hub → SpokB without UDRs
az network vwan create -g myRG -n myVWAN --location eastus
az network vhub create -g myRG -n myVHub --vwan myVWAN \
  --location eastus --address-prefix 10.100.0.0/24 --sku Standard

# Connect VNet to hub (auto routes)
az network vhub connection create -g myRG --vhub-name myVHub \
  -n myVNetConn \
  --remote-vnet /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet

# Full Mesh: every VNet peers with every other VNet
# Fine for < 10 VNets; grows quadratically (10 VNets = 45 peerings)
# No Firewall inspection by default between VNets
```

---

### 🟡 Q109. What is ExpressRoute Local, Standard, and Premium?
```bash
# ExpressRoute circuit tiers:
# Local:    connect to single Azure region, data transfer FREE, limited PoPs
# Standard: connect to all regions in same geopolitical area (e.g., all of Europe)
# Premium:  connect to all regions globally, 10+ ER circuits, M365 peering

az network express-route create -g myRG -n myERLocal \
  --provider "Equinix" \
  --peering-location "Singapore" \
  --bandwidth 1000 \
  --sku-tier Local \            # Local = free data transfer, limited to local AZ
  --sku-family MeteredData      # MeteredData | UnlimitedData (flat fee)

# ExpressRoute FastPath: bypasses ExpressRoute Gateway for direct flows
# Reduces latency for ultra-high-bandwidth flows (10–100 Gbps)
# Requires: UltraPerformance or ErGw3AZ gateway SKU
az network vnet-gateway update -g myRG -n myERGW \
  --set properties.enablePrivateIpAddress=true    # required for FastPath

# Per circuit cost breakdown:
# Local:     circuit fee only (no data transfer charge)
# Standard:  circuit fee + data transfer (MeteredData) or flat (UnlimitedData)
# Premium:   higher circuit fee + global routing + more PoPs
```

---

### 🟡 Q110. What is IPv6 dual-stack in Azure VNet?
```bash
# Azure supports IPv4 + IPv6 dual-stack in VNets, subnets, NICs, LBs

# Create dual-stack VNet
az network vnet create -g myRG -n myDualStackVNet \
  --address-prefix 10.0.0.0/16 "fd00::/48" \
  --location eastus

# Create dual-stack subnet
az network vnet subnet create -g myRG --vnet-name myDualStackVNet \
  -n mySubnet \
  --address-prefixes 10.0.1.0/24 "fd00::1:0/112"

# Create dual-stack public IPs
az network public-ip create -g myRG -n myIPv4IP \
  --sku Standard --version IPv4 --zone 1 2 3
az network public-ip create -g myRG -n myIPv6IP \
  --sku Standard --version IPv6 --zone 1 2 3

# Create VM with dual-stack NIC
az network nic create -g myRG -n myDualNIC \
  --vnet-name myDualStackVNet --subnet mySubnet \
  --private-ip-address 10.0.1.10 \
  --private-ip-address-version IPv4

az network nic ip-config create -g myRG --nic-name myDualNIC \
  -n IPv6Config \
  --private-ip-address-version IPv6 \
  --subnet /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myDualStackVNet/subnets/mySubnet

az vm create -g myRG -n myDualVM --nics myDualNIC \
  --image Ubuntu2204 --generate-ssh-keys
```

---

# GAP-FILL PART 3 — STORAGE (Q111–Q115)

---

### 🟡 Q111. What are Immutable Blobs (WORM policies)?
```bash
# WORM = Write Once Read Many: blobs cannot be modified or deleted
# Two policy types:
# Time-based retention: data locked for N days
# Legal hold:           data locked until explicitly cleared (no time limit)

# Enable versioning (required for account-level immutability)
az storage account blob-service-properties update \
  --account-name mystorageaccount --enable-versioning true

# Container-level time-based retention
az storage container immutability-policy create \
  --account-name mystorageaccount --container-name mycontainer \
  --period 365 \       # retain for 365 days
  --allow-protected-append-writes true   # allows appending to append blobs only

# Lock the policy (IRREVERSIBLE — cannot change retention period down)
az storage container immutability-policy lock \
  --account-name mystorageaccount --container-name mycontainer \
  --if-match <etag>    # etag from immutability-policy show

# Legal hold (compliance — block deletion until cleared)
az storage container legal-hold set \
  --account-name mystorageaccount --container-name mycontainer \
  --tags investigation2026 regulator-hold

az storage container legal-hold clear \
  --account-name mystorageaccount --container-name mycontainer \
  --tags investigation2026    # clear specific tag to allow deletion

# WORM use cases:
# Financial records (SEC Rule 17a-4, FINRA)
# Healthcare records (HIPAA)
# Legal discovery / litigation hold
# Audit log preservation
```

---

### 🟡 Q112. What is Azure Files identity-based authentication?
```bash
# Azure Files supports Kerberos auth via:
# 1. On-prem AD DS (most common for lift-and-shift)
# 2. Microsoft Entra Domain Services (cloud-managed AD)
# 3. Microsoft Entra Kerberos (hybrid identity — no DC needed)

# Enable Azure AD Kerberos (hybrid workers — Entra joined devices)
az storage account update -g myRG -n mystorageaccount \
  --enable-files-aad-kerberos true

# Assign share-level permission (RBAC)
az role assignment create \
  --assignee user@company.com \
  --role "Storage File Data SMB Share Contributor" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount/fileServices/default/fileshares/myshare

# Available roles:
# Storage File Data SMB Share Reader:         read-only
# Storage File Data SMB Share Contributor:    read/write
# Storage File Data SMB Share Elevated Contributor: read/write + change permissions
# Storage File Data SMB Share Owner:          full control

# Mount with identity auth (no storage key needed)
# Windows: net use Z: \\mystorageaccount.file.core.windows.net\myshare
#          /user:company.com\username  (password: Entra ID password)
# Linux: kinit user@COMPANY.COM && mount -t cifs ... -o sec=krb5i
```

---

### 🟡 Q113. What is Azure Site Recovery (ASR)?
```bash
# ASR: VM disaster recovery — replicate VMs to another Azure region
# RPO: 15 seconds | RTO: < 1 hour

# Create Recovery Services Vault (in DR region)
az backup vault create -g myDRRG -n myASRVault --location westus2

# Create replication policy
az site-recovery replication-policy create -g myDRRG \
  --vault-name myASRVault -n myRepPolicy \
  --recovery-point-threshold-in-minutes 60 \
  --recovery-point-history 24 \
  --app-consistent-frequency-in-minutes 60

# Enable VM replication (in primary region)
# az site-recovery replication-protected-item create ...
# (complex — typically done via Portal or PowerShell)

# Test failover (validate DR without impacting production)
az site-recovery replication-protected-item test-failover \
  -g myDRRG --vault-name myASRVault \
  --fabric-name myFabric \
  --protection-container-name myContainer \
  --protected-item myVM \
  --failover-direction PrimaryToRecovery

# Clean up test failover
az site-recovery replication-protected-item test-failover-cleanup \
  -g myDRRG --vault-name myASRVault \
  --fabric-name myFabric \
  --protection-container-name myContainer \
  --protected-item myVM

# Planned failover (graceful — zero data loss)
az site-recovery replication-protected-item failover-commit \
  -g myDRRG --vault-name myASRVault \
  --fabric-name myFabric \
  --protection-container-name myContainer \
  --protected-item myVM

# ASR use cases:
# Azure region → Azure region (most common)
# On-prem VMware/Hyper-V → Azure (migration + DR)
# Physical servers → Azure
```

---

# GAP-FILL PART 4 — DATABASES (Q114–Q120)

---

### 🟡 Q114. What is Azure SQL DB DTU vs vCore model?
| Feature | DTU Model | vCore Model |
|---------|----------|------------|
| Compute unit | Blended (CPU+IO+Memory) | vCPUs (separately configurable) |
| Transparency | Hidden | Visible |
| Serverless | No | Yes |
| Hybrid Benefit | No | Yes |
| Max cores | 3000 DTU (S12) | 80 vCores |
| Max storage | 4 TB (Premium) | 100 TB (Hyperscale) |
| Cross-region restore | Yes | Yes |
| Recommended | Legacy/simple | New deployments |

```bash
# DTU model (Basic/Standard/Premium)
az sql db create -g myRG -s mysqlserver -n myDTUdb \
  --service-objective S3     # S0(10 DTU) S1 S2 S3 S4 S6 S7 S9 S12

# vCore model (GeneralPurpose/BusinessCritical/Hyperscale)
az sql db create -g myRG -s mysqlserver -n myVCoredb \
  --service-objective GP_Gen5_4   # GP=GeneralPurpose, 4 vCores
```

---

### 🔴 Q115. What is Cosmos DB change feed and conflict resolution?
```bash
# Change Feed: ordered log of all inserts and updates in a container
# Use for: event sourcing, CDC, real-time analytics, cache invalidation

# Python: consume change feed
from azure.cosmos import CosmosClient
from azure.identity import DefaultAzureCredential

client = CosmosClient(
    url="https://mycosmosaccnt.documents.azure.com",
    credential=DefaultAzureCredential()
)
container = client.get_database_client("myDB").get_container_client("orders")

# Read change feed (latest mode — only new changes)
for page in container.query_items_change_feed(is_start_from_beginning=False):
    print(f"Changed: {page['id']}, type: {page.get('_lsn')}")

# Full fidelity change feed (includes deletes — preview)
# Requires: EnableFullFidelityChangeFeed capability on account

# Multi-region write conflict resolution:
# Last Write Wins (LWW): highest _ts wins (DEFAULT)
az cosmosdb update -g myRG -n myCosmosAcct \
  --conflict-resolution-policy '{"mode":"LastWriterWins","conflictResolutionPath":"/_ts"}'

# Custom conflict resolution (stored procedure)
az cosmosdb update -g myRG -n myCosmosAcct \
  --conflict-resolution-policy '{
    "mode":"Custom",
    "conflictResolutionProcedure":"dbs/myDB/colls/orders/sprocs/resolveConflict"
  }'

# Async conflict feed (manual resolution)
az cosmosdb update -g myRG -n myCosmosAcct \
  --conflict-resolution-policy '{"mode":"Custom"}'
# Conflicts appear in _conflicts feed — your app reads and resolves

# Synapse Link (analytical store — zero-ETL HTAP)
az cosmosdb update -g myRG -n myCosmosAcct \
  --enable-analytical-storage true
az cosmosdb sql container update -g myRG \
  --account-name myCosmosAcct --database-name myDB -n orders \
  --analytical-storage-ttl -1   # enable analytical store

# Query from Synapse without impacting transactional performance:
# SELECT * FROM OPENROWSET('CosmosDB',
#   'account=mycosmosaccnt;database=myDB;collection=orders;region=eastus;...',
#   orders) AS [orders]
```

---

### 🟡 Q116. What is Azure SQL Always Encrypted?
```sql
-- Always Encrypted: encrypt specific columns in client — SQL Server never sees plaintext
-- Key hierarchy: Column Master Key (CMK) → Column Encryption Key (CEK) → Column

-- CMK in Azure Key Vault
CREATE COLUMN MASTER KEY MyCMK
WITH (
    KEY_STORE_PROVIDER_NAME = 'AZURE_KEY_VAULT',
    KEY_PATH = 'https://myKV.vault.azure.net/keys/AlwaysEncryptedCMK/version'
);

CREATE COLUMN ENCRYPTION KEY MyCEK
WITH VALUES (
    COLUMN_MASTER_KEY = MyCMK,
    ALGORITHM = 'RSA_OAEP',
    ENCRYPTED_VALUE = 0x01700000...  -- generated by SSMS/SDK
);

-- Table with encrypted columns
CREATE TABLE Patients (
    PatientId   INT          NOT NULL PRIMARY KEY,
    FirstName   NVARCHAR(50)
                ENCRYPTED WITH (
                    ENCRYPTION_TYPE = DETERMINISTIC,   -- allows equality search
                    ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256',
                    COLUMN_ENCRYPTION_KEY = MyCEK),
    Diagnosis   NVARCHAR(200)
                ENCRYPTED WITH (
                    ENCRYPTION_TYPE = RANDOMIZED,      -- more secure, no search
                    ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256',
                    COLUMN_ENCRYPTION_KEY = MyCEK),
    SSN         CHAR(11)
                ENCRYPTED WITH (
                    ENCRYPTION_TYPE = DETERMINISTIC,
                    ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256',
                    COLUMN_ENCRYPTION_KEY = MyCEK)
);

-- Encryption types:
-- DETERMINISTIC: same plaintext → same ciphertext → supports =, IN, JOIN, GROUP BY
-- RANDOMIZED:    same plaintext → different ciphertext → no search, but more secure

-- Secure Enclaves (Always Encrypted with Secure Enclaves)
-- Allows: LIKE, range comparisons, ORDER BY on encrypted columns
-- Requires: VBS or SGX enclave + attestation service
```

---

### 🟡 Q117. What is Azure SQL Database Ledger?
```sql
-- SQL Ledger: tamper-evident, append-only tables with cryptographic verification
-- Use for: financial records, compliance audit, supply chain, legal records

-- Updatable ledger table (DML allowed, history retained)
CREATE TABLE dbo.Transactions (
    TransactionId  BIGINT          NOT NULL PRIMARY KEY,
    AccountId      INT             NOT NULL,
    Amount         DECIMAL(18,2)   NOT NULL,
    TransactionDate DATE           NOT NULL
)
WITH (SYSTEM_VERSIONING = ON, LEDGER = ON);

-- Each update creates a new row — nothing is truly deleted
-- History stored in: dbo.MSSQL_LedgerHistoryFor_<tableid>

-- Append-only ledger table (INSERT only — no UPDATE/DELETE)
CREATE TABLE dbo.AuditLog (
    LogId      BIGINT       NOT NULL PRIMARY KEY,
    UserId     INT          NOT NULL,
    Action     NVARCHAR(100),
    Timestamp  DATETIME2    NOT NULL DEFAULT SYSUTCDATETIME()
)
WITH (LEDGER = ON (APPEND_ONLY = ON));

-- Verify ledger integrity (detect tampering)
EXECUTE sp_verify_database_ledger_from_digest_storage;

-- Export digest to Azure Storage (immutable proof)
EXECUTE sys.sp_generate_database_ledger_digest;
-- Returns: {"block_id":1,"hash":"0x...","treeRootHash":"0x..."}
-- Store in Azure Blob with immutability policy for court-admissible evidence
```

---

# GAP-FILL PART 5 — MONITORING (Q118–Q130)

---

### 🟡 Q118. What is VM Insights and Container Insights?
```bash
# VM Insights: pre-built performance monitoring + dependency/service maps for VMs

# Enable VM Insights
az vm extension set -g myRG --vm-name myVM \
  --name DependencyAgentLinux \
  --publisher Microsoft.Azure.Monitoring.DependencyAgent \
  --version 9.10 --enable-auto-upgrade true

# Also requires Azure Monitor Agent (AMA) — already covered
# VM Insights tables in Log Analytics:
# InsightsMetrics: CPU, memory, disk, network performance
# VMConnection:    network connections (source IP, dest IP, port, direction)
# VMProcess:       running processes with CPU/memory per process
# VMComputer:      VM inventory
```

```kusto
// VM Insights: top processes by CPU
VMProcess
| where TimeGenerated > ago(15m)
| where Computer == "myVM"
| summarize AvgCPU = avg(CpuPercentage), MaxCPU = max(CpuPercentage)
    by ProcessName, Computer
| where AvgCPU > 5
| order by AvgCPU desc

// VM Insights: network connections from VM
VMConnection
| where TimeGenerated > ago(1h)
| where Computer == "myVM"
| summarize ConnectionCount = count()
    by DestinationIp, DestinationPort, Protocol, Direction
| order by ConnectionCount desc

// Container Insights: full pod health
KubePodInventory
| where TimeGenerated > ago(5m)
| summarize arg_max(TimeGenerated, *) by ContainerId
| project PodName = Name, Namespace, Node = Computer,
          Status = PodStatus, Restarts = ContainerRestartCount,
          Image = ContainerImage
| order by Restarts desc
```

```bash
# Container Insights — enable on existing AKS cluster
az aks enable-addons -g myRG -n myAKS --addons monitoring \
  --workspace-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA

# Container Insights tables:
# KubePodInventory:     all pods with status, restarts, images
# KubeNodeInventory:    node status, capacity, conditions
# KubeEvents:           all K8s events (OOMKilled, FailedScheduling, etc.)
# ContainerLog:         stdout/stderr from all containers
# ContainerInventory:   container inventory with image info
# InsightsMetrics:      CPU, memory metrics per pod and node
# KubeServices:         service inventory
# KubeDeployment:       deployment status and history
```

---

### 🟡 Q119. What is Application Insights distributed tracing and Application Map?
```bash
# Distributed tracing: track requests across microservices
# Application Map: visual topology of your app's dependency graph

# Every request gets a correlation ID (traceparent header W3C standard)
# Propagated automatically across HTTP calls when using OpenTelemetry SDK

# Application Map shows:
# - Each microservice as a node
# - Call counts and failure rates on edges
# - Dependency call performance (SQL, Redis, Service Bus, etc.)
# - Health traffic light (green/amber/red)

# Live Metrics Stream: real-time telemetry with <1s latency
# Shows: incoming request rate, failure rate, CPU, memory, dependencies

# Performance Profiler: flame chart of CPU usage during request
# - Identifies which function is the bottleneck
# - On-demand or scheduled profiling

# Snapshot Debugger: capture memory snapshot when exception is thrown
# - See local variables, call stack at exact moment of exception
# - No need to repro the bug
```

```python
# Distributed tracing propagation example
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace
from opentelemetry.instrumentation.aiohttp_client import AioHttpClientInstrumentor
import aiohttp

configure_azure_monitor(connection_string="InstrumentationKey=xxx;...")
AioHttpClientInstrumentor().instrument()   # auto-propagates traceparent header

tracer = trace.get_tracer(__name__)

async def process_order(order_id: str):
    with tracer.start_as_current_span("process-order") as span:
        span.set_attribute("order.id", order_id)
        # This HTTP call automatically carries the trace context
        async with aiohttp.ClientSession() as session:
            # inventory-service receives traceparent header → linked trace
            async with session.get(f"http://inventory-service/check/{order_id}") as resp:
                inventory = await resp.json()

        # SQL calls also auto-traced by SQLAlchemyInstrumentor
        result = await save_to_db(order_id)
        span.set_attribute("order.saved", True)
        return result
```

---

### 🟡 Q120. What is Azure Monitor Private Link Scope (AMPLS)?
```bash
# AMPLS: route Azure Monitor data (logs, metrics) through private endpoints
# No telemetry goes over public internet

# Create Azure Monitor Private Link Scope
az monitor private-link-scope create -g myRG -n myAMPLS --location global

# Associate Log Analytics workspace with AMPLS
az monitor private-link-scope scoped-resource create -g myRG \
  --scope-name myAMPLS -n myLALink \
  --linked-resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA

# Associate Application Insights with AMPLS
az monitor private-link-scope scoped-resource create -g myRG \
  --scope-name myAMPLS -n myAILink \
  --linked-resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/components/myAppInsights

# Create private endpoint for AMPLS
az network private-endpoint create -g myRG -n ampls-PE \
  --vnet-name myVNet --subnet appSubnet \
  --private-connection-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/privateLinkScopes/myAMPLS \
  --group-id azuremonitor --connection-name amplsConn

# Create private DNS zones for Monitor (5 zones required)
for ZONE in "privatelink.monitor.azure.com" \
            "privatelink.oms.opinsights.azure.com" \
            "privatelink.ods.opinsights.azure.com" \
            "privatelink.agentsvc.azure-automation.net" \
            "privatelink.blob.core.windows.net"; do
  az network private-dns zone create -g myRG -n $ZONE
  az network private-dns link vnet create -g myRG \
    --zone-name $ZONE -n amplsDNSLink-$ZONE \
    --virtual-network myVNet --registration-enabled false
done

# AMPLS ingestion modes:
# Open:   VNet agents use private link; public internet agents also allowed
# PrivateOnly: only agents with private link can send data (most secure)
az monitor private-link-scope update -g myRG -n myAMPLS \
  --access-mode-settings '{"ingestionAccessMode":"PrivateOnly","queryAccessMode":"PrivateOnly"}'
```

---

### 🟡 Q121. What is Azure Monitor Metrics Explorer?
```bash
# Metrics Explorer: visualise and analyse Azure Monitor platform metrics
# Platform metrics: automatically collected from all Azure resources (free)

# Query metrics via CLI
az monitor metrics list \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --metric "Percentage CPU" \
  --interval PT5M \
  --start-time $(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --aggregation Average Maximum \
  --output table

# Multi-dimensional metrics (split by dimension)
az monitor metrics list \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount \
  --metric "Transactions" \
  --interval PT1H \
  --aggregation Total \
  --filter "ApiName eq 'GetBlob'" \   # filter by dimension
  --output table

# Available dimensions vary by resource and metric:
# VM:       Percentage CPU, Available Memory Bytes, Disk Read/Write Bytes
# App GW:   Throughput, Failed Requests, Healthy/Unhealthy Host Count
# SQL DB:   DTU Consumption %, Sessions %, CPU Percentage
# Cosmos:   Total Requests, Total Request Units, Server Side Latency
# AKS:      Node CPU %, Node Memory %, Pod Count, Pending Pod Count
# Storage:  Transactions (by ApiName, StatusCode), Ingress, Egress

# Pin metric chart to shared dashboard
# Portal: Metrics → select resource → select metric → pin to dashboard
```

---

### 🟡 Q122. What is Azure Monitor action groups — all action types?
```bash
# Action Group: defines WHO gets notified and HOW when alert fires

az monitor action-group create -g myRG -n myCompleteAG \
  --short-name myAG \
  --action email    email-ops   ops@company.com \
  --action sms      sms-oncall  1 5551234567 \
  --action voice    voice-oncall 1 5551234567 \
  --action webhook  pagerduty   https://events.pagerduty.com/integration/<key>/enqueue \
  --action webhook  slack       https://hooks.slack.com/services/<key> \
  --action azureapp mobile-app /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Insights/components/myAppInsights \
  --action itsm     servicenow  <workspace-id> <connection-id> <region> <ticket-config-id> \
  --action automationrunbook  myRunbook \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Automation/automationAccounts/myAutoAcct/runbooks/myRunbook \
    "https://myAutoAcct.webhook.azure.com/..." true \
  --action logicapp  myLogicApp \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Logic/workflows/myLogicApp \
    "https://prod-14.eastus.logic.azure.com/workflows/..."

# Action types:
# Email:               direct email notification
# SMS:                 text message
# Voice:               automated phone call
# Webhook:             HTTP POST to any endpoint (PagerDuty, Slack, Teams)
# Azure App:           push notification to Azure mobile app
# ITSM:                create ticket in ServiceNow, JIRA, etc.
# Automation Runbook:  trigger PowerShell runbook
# Logic App:           trigger Logic Apps workflow
# Function App:        trigger Azure Function
# Event Hub:           publish event to Event Hub
```

---

### 🟢 Q123. What is Azure Monitor diagnostic settings for all resource types?
```bash
# Enable diagnostic settings on key resource types

# Key Vault
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKV \
  --workspace <la-id> -n kvDiag \
  --logs '[{"category":"AuditEvent","enabled":true},{"category":"AzurePolicyEvaluationDetails","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# NSG (flow logs are separate from diagnostic settings)
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/networkSecurityGroups/myNSG \
  --workspace <la-id> -n nsgDiag \
  --logs '[{"category":"NetworkSecurityGroupEvent","enabled":true},{"category":"NetworkSecurityGroupRuleCounter","enabled":true}]'

# SQL DB
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Sql/servers/mysqlserver/databases/myDB \
  --workspace <la-id> -n sqlDiag \
  --logs '[{"category":"SQLInsights","enabled":true},{"category":"AutomaticTuning","enabled":true},{"category":"QueryStoreRuntimeStatistics","enabled":true},{"category":"Errors","enabled":true},{"category":"DatabaseWaitStatistics","enabled":true}]' \
  --metrics '[{"category":"Basic","enabled":true},{"category":"InstanceAndAppAdvanced","enabled":true}]'

# App Service
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Web/sites/myWebApp \
  --workspace <la-id> -n appDiag \
  --logs '[{"category":"AppServiceHTTPLogs","enabled":true},{"category":"AppServiceConsoleLogs","enabled":true},{"category":"AppServiceAppLogs","enabled":true},{"category":"AppServiceAuditLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# AKS
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.ContainerService/managedClusters/myAKS \
  --workspace <la-id> -n aksDiag \
  --logs '[{"category":"kube-apiserver","enabled":true},{"category":"kube-controller-manager","enabled":true},{"category":"kube-scheduler","enabled":true},{"category":"guard","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Cosmos DB
az monitor diagnostic-settings create \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.DocumentDB/databaseAccounts/myCosmosAcct \
  --workspace <la-id> -n cosmosDiag \
  --logs '[{"category":"DataPlaneRequests","enabled":true},{"category":"QueryRuntimeStatistics","enabled":true},{"category":"PartitionKeyStatistics","enabled":true},{"category":"ControlPlaneRequests","enabled":true}]' \
  --metrics '[{"category":"Requests","enabled":true}]'
```

---

### 🟡 Q124. What is Azure Monitor Agent (AMA) migration from MMA/OMS?
```bash
# Legacy agents (being retired):
# MMA = Microsoft Monitoring Agent (Log Analytics agent for Windows)
# OMS = Operations Management Suite agent (Log Analytics agent for Linux)
# MMA/OMS retirement date: August 31, 2024

# New agent:
# AMA = Azure Monitor Agent — replaces both MMA and OMS
# Benefits: uses DCR (Data Collection Rules) — granular, per-VM filtering

# Migration steps:
# 1. Install AMA on all VMs
# 2. Create DCRs to replace workspace configuration
# 3. Verify data flowing in Log Analytics
# 4. Uninstall MMA/OMS

# 1. Install AMA on Linux VM
az vm extension set -g myRG --vm-name myVM \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --version 1.0 --enable-auto-upgrade true

# 2. Install AMA on Windows VM
az vm extension set -g myRG --vm-name myWinVM \
  --name AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor \
  --version 1.0 --enable-auto-upgrade true

# 3. Create DCR (defines what to collect and where to send)
# See Q86 for full DCR creation example

# 4. Remove MMA (after validating AMA is working)
az vm extension delete -g myRG --vm-name myVM \
  --name OmsAgentForLinux

az vm extension delete -g myRG --vm-name myWinVM \
  --name MicrosoftMonitoringAgent

# AMA supports: Azure VMs, Arc-enabled servers, VMSS, AKS nodes
# AMA does NOT replace Diagnostics Extension (for metrics)
```

---

### FINAL COMPREHENSIVE Q&A INDEX (All 130 Questions)

| Q# | Question | Level | Part |
|----|---------|-------|------|
| Q1–Q25 | Core Compute (VM, VMSS, App Service, Functions, AKS, ACR, Batch) | 🟢🟡🔴 | Compute |
| Q26–Q50 | Core Networking (VNet, NSG, LB, App GW, Front Door, VPN, ER, Bastion, Firewall, DNS, NAT, Private Link) | 🟢🟡 | Networking |
| Q51–Q70 | Core Storage (Blob tiers, SAS, ADLS, Files, Queue, Table, Backup, AzCopy, encryption) | 🟢🟡 | Storage |
| Q71–Q85 | Core Databases (SQL DB, Cosmos DB, PostgreSQL, MySQL, Redis, Synapse, Databricks, ASA) | 🟢🟡 | Databases |
| Q86–Q95 | Core Monitoring (Azure Monitor, KQL, App Insights, Alerts, Advisor) | 🟢🟡 | Monitoring |
| **GAP-FILL ADDITIONS:** | | | |
| Q96 | App Service Environment (ASE) v3 | 🟡 | Compute |
| Q97 | App Service backup, custom domain, TLS, health check, autoscale | 🟡 | Compute |
| Q98 | Durable Functions — all 5 patterns | 🔴 | Compute |
| Q99 | AKS node pools, taints, tolerations, GPU/Spot pools | 🔴 | Compute |
| Q100 | AKS Workload Identity — OIDC, federated credentials | 🔴 | Compute |
| Q101 | AKS persistent volumes — Azure Disk + Azure Files CSI | 🟡 | Compute |
| Q102 | AKS networking — CNI Overlay, Cilium, NetworkPolicy | 🟡 | Compute |
| Q103 | VM disk encryption — ADE vs SSE with CMK | 🟡 | Compute |
| Q104 | VM proximity placement groups | 🟡 | Compute |
| Q105 | VMSS upgrade policies — rolling vs automatic vs manual | 🟡 | Compute |
| Q106 | Service Tags in NSG rules | 🟡 | Networking |
| Q107 | Azure Route Server — BGP with NVAs | 🟡 | Networking |
| Q108 | Hub-Spoke vs Virtual WAN vs Mesh comparison | 🟡 | Networking |
| Q109 | ExpressRoute Local vs Standard vs Premium + FastPath | 🟡 | Networking |
| Q110 | IPv6 dual-stack in Azure VNet | 🟡 | Networking |
| Q111 | Immutable Blobs (WORM) — time-based + legal hold | 🟡 | Storage |
| Q112 | Azure Files identity-based auth (Kerberos, Entra ID) | 🟡 | Storage |
| Q113 | Azure Site Recovery (ASR) — cross-region VM DR | 🟡 | Storage |
| Q114 | Azure SQL DB DTU vs vCore model | 🟡 | Databases |
| Q115 | Cosmos DB change feed + conflict resolution | 🔴 | Databases |
| Q116 | SQL Always Encrypted — CMK, CEK, deterministic vs randomized | 🔴 | Databases |
| Q117 | SQL Database Ledger — tamper-proof audit tables | 🟡 | Databases |
| Q118 | VM Insights + Container Insights — KQL queries | 🟡 | Monitoring |
| Q119 | App Insights distributed tracing, Application Map, Profiler | 🟡 | Monitoring |
| Q120 | Azure Monitor Private Link Scope (AMPLS) | 🔴 | Monitoring |
| Q121 | Metrics Explorer — dimensions, splitting, filtering | 🟡 | Monitoring |
| Q122 | Action groups — all action types (ITSM, Runbook, Logic App) | 🟡 | Monitoring |
| Q123 | Diagnostic settings for all key resource types | 🟢 | Monitoring |
| Q124 | AMA vs MMA/OMS migration | 🟡 | Monitoring |

---
*Total: 130 Q&A | June 2026 | Microsoft Learn aligned*
*🟢 30 Basic | 🟡 88 Intermediate | 🔴 12 Advanced*
