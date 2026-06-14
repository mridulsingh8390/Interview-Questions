# Azure DevOps, Migration & Serverless — Complete Interview Q&A Guide
> **All possible questions | June 2026 | Microsoft Learn aligned**
> 🟢 Basic | 🟡 Intermediate | 🔴 Advanced | Full CLI + YAML + code examples

---

## TABLE OF CONTENTS
| Part | Category | Questions |
|------|---------|----------|
| [1](#part-1--azure-devops) | Azure DevOps | Q1–Q50 |
| [2](#part-2--azure-migration) | Azure Migration | Q51–Q85 |
| [3](#part-3--serverless) | Serverless | Q86–Q130 |

---

# PART 1 — AZURE DEVOPS

---

### 🟢 Q1. What is Azure DevOps and what are its five services?
**Answer:**
Azure DevOps is a suite of developer services for planning, developing, delivering, and operating software. It supports any language, platform, and cloud.

| Service | Purpose |
|---------|---------|
| **Azure Boards** | Agile project management (epics, features, stories, tasks, bugs, sprints) |
| **Azure Repos** | Git source control (unlimited free private repos) |
| **Azure Pipelines** | CI/CD automation (YAML-based, multi-stage, multi-platform) |
| **Azure Test Plans** | Manual + automated test management, exploratory testing |
| **Azure Artifacts** | Package management (npm, NuGet, PyPI, Maven, Cargo, Universal) |

```bash
# Install Azure DevOps CLI extension
az extension add --name azure-devops

# Configure defaults
az devops configure --defaults \
  organization=https://dev.azure.com/myOrg \
  project=myProject

# Create project
az devops project create \
  --name myProject \
  --visibility private \
  --source-control git \
  --process Agile \
  --organization https://dev.azure.com/myOrg

# List projects
az devops project list --output table
```

---

### 🟢 Q2. What is Azure Repos and how do you work with it?
```bash
# Create repository
az repos create --project myProject -n myRepo --output table

# List repositories
az repos list --project myProject --output table

# Clone repository
git clone https://myOrg@dev.azure.com/myOrg/myProject/_git/myRepo
cd myRepo

# Set upstream and push
git remote set-url origin https://myOrg@dev.azure.com/myOrg/myProject/_git/myRepo
git push -u origin main

# Create branch policy (protect main branch)
az repos policy merge-strategy create \
  --project myProject --repo-id <repo-id> \
  --branch main --is-enabled true --blocking true \
  --allow-no-fast-forward false \
  --allow-rebase false \
  --allow-rebase-merge false \
  --allow-squash true

# Require minimum 2 reviewers on PR
az repos policy approver-count create \
  --project myProject --repo-id <repo-id> \
  --branch main --is-enabled true --blocking true \
  --minimum-approver-count 2 \
  --creator-vote-counts false \
  --allow-downvotes false \
  --reset-on-source-push true

# Require build to pass before PR merges
az repos policy build create \
  --project myProject --repo-id <repo-id> \
  --branch main --is-enabled true --blocking true \
  --build-definition-id <pipeline-id> \
  --display-name "CI Build" \
  --queue-on-source-update-only false \
  --manual-queue-only false \
  --valid-duration 720

# Create pull request
az repos pr create \
  --project myProject \
  --repository myRepo \
  --source-branch feature/my-feature \
  --target-branch main \
  --title "Add new feature" \
  --description "This PR adds the new payment feature" \
  --reviewers "jane.doe@company.com" "john.smith@company.com" \
  --work-items 1234 \
  --auto-complete false \
  --squash false

# List PRs
az repos pr list --project myProject --status active --output table

# Complete PR
az repos pr update \
  --project myProject --id <pr-id> \
  --status completed \
  --merge-strategy squash \
  --delete-source-branch true
```

---

### 🟢 Q3. What is Azure Boards and how do you manage work items?
```bash
# Work item hierarchy (Agile process):
# Epic → Feature → User Story → Task / Bug / Test Case

# Create epic
az boards work-item create \
  --project myProject \
  --type Epic \
  --title "Payment System Upgrade" \
  --description "Upgrade payment processing to support new gateways" \
  --assigned-to "productowner@company.com"

# Create user story
az boards work-item create \
  --project myProject \
  --type "User Story" \
  --title "As a user, I can pay with Apple Pay" \
  --assigned-to "developer@company.com" \
  --area "myProject\Backend" \
  --iteration "myProject\Sprint 3"

# Create task under user story
az boards work-item create \
  --project myProject \
  --type Task \
  --title "Implement Apple Pay webhook handler" \
  --assigned-to "developer@company.com"

# Link work items (parent-child)
az boards work-item relation add \
  --id <story-id> \
  --relation-type "System.LinkTypes.Hierarchy-Reverse" \
  --target-id <epic-id>

# Create sprint
az boards iteration project create \
  --project myProject \
  --path "\myProject\Sprint 3" \
  --start-date 2026-06-16 \
  --finish-date 2026-06-30

# Query work items
az boards query \
  --project myProject \
  --wiql "SELECT [Id],[Title],[State],[AssignedTo] FROM WorkItems WHERE [System.TeamProject] = 'myProject' AND [System.State] <> 'Closed' AND [System.AssignedTo] = 'developer@company.com' ORDER BY [Microsoft.VSTS.Common.Priority] ASC"

# Update work item
az boards work-item update \
  --id <work-item-id> \
  --state "Active" \
  --reason "Implementation started"

# Close work item
az boards work-item update --id <work-item-id> --state "Closed"
```

---

### 🟢 Q4. What is Azure Pipelines and how does a basic YAML pipeline work?
```yaml
# azure-pipelines.yml — basic CI pipeline
trigger:
  branches:
    include: [main, develop, release/*]
  paths:
    exclude: [docs/*, "*.md", "*.txt"]

pr:
  branches:
    include: [main]
  paths:
    exclude: [docs/*]

variables:
  pythonVersion: '3.12'
  POETRY_VERSION: '1.8.0'

pool:
  vmImage: ubuntu-latest        # Microsoft-hosted agent

stages:
- stage: Validate
  displayName: Validate Code
  jobs:
  - job: Lint
    displayName: Lint and Type Check
    steps:
    - task: UsePythonVersion@0
      inputs:
        versionSpec: $(pythonVersion)
      displayName: Set Python $(pythonVersion)

    - script: |
        pip install ruff mypy
        ruff check . --output-format=github
        mypy src/ --ignore-missing-imports
      displayName: Lint and type check

  - job: Test
    displayName: Unit Tests
    dependsOn: Lint
    steps:
    - task: UsePythonVersion@0
      inputs: { versionSpec: $(pythonVersion) }

    - script: |
        pip install pytest pytest-cov pytest-asyncio
        pip install -r requirements.txt
        pytest tests/unit \
          --junitxml=test-results/unit.xml \
          --cov=src --cov-report=xml:coverage.xml \
          --cov-report=html:htmlcov \
          -v
      displayName: Run unit tests

    - task: PublishTestResults@2
      condition: always()
      inputs:
        testResultsFormat: JUnit
        testResultsFiles: test-results/unit.xml
        testRunTitle: Unit Tests

    - task: PublishCodeCoverageResults@2
      inputs:
        summaryFileLocation: coverage.xml
        reportDirectory: htmlcov

- stage: Build
  displayName: Build and Push
  dependsOn: Validate
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - job: DockerBuild
    steps:
    - task: Docker@2
      displayName: Build and push image
      inputs:
        command: buildAndPush
        repository: $(containerRegistry)/$(imageRepository)
        dockerfile: Dockerfile
        containerRegistry: myACRServiceConnection
        tags: |
          $(Build.BuildId)
          latest
        buildContext: .
        arguments: |
          --build-arg BUILD_ID=$(Build.BuildId)
          --build-arg COMMIT=$(Build.SourceVersion)
```

---

### 🟡 Q5. What is a multi-stage Azure Pipeline with environments and approvals?
```yaml
# azure-pipelines.yml — full multi-stage CI/CD with approvals
trigger:
  branches:
    include: [main]

variables:
  containerRegistry: myacr.azurecr.io
  imageRepository: myapp
  tag: $(Build.BuildId)
  vmImageName: ubuntu-latest

stages:
# ─────────────── CI ────────────────────────────────────────────
- stage: CI
  displayName: Build, Test and Push
  jobs:
  - job: BuildAndTest
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: UsePythonVersion@0
      inputs: { versionSpec: '3.12' }

    - script: pip install -r requirements.txt pytest pytest-cov
      displayName: Install dependencies

    - script: pytest tests/ --junitxml=results.xml --cov=src --cov-report=xml
      displayName: Run tests

    - task: PublishTestResults@2
      condition: always()
      inputs: { testResultsFiles: results.xml }

    - task: Docker@2
      inputs:
        command: buildAndPush
        repository: $(containerRegistry)/$(imageRepository)
        containerRegistry: myACRServiceConnection
        tags: $(tag)

# ─────────────── STAGING ───────────────────────────────────────
- stage: DeployStaging
  displayName: Deploy to Staging
  dependsOn: CI
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: DeployToStaging
    displayName: Deploy to Staging
    environment:
      name: staging              # environment gates configured in UI
      resourceType: Kubernetes
      resourceName: myAKS/staging-namespace
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
              azureResourceGroup: myRG
              kubernetesCluster: myAKS
              manifests: |
                k8s/deployment.yaml
                k8s/service.yaml
              containers: $(containerRegistry)/$(imageRepository):$(tag)

          - task: Kubernetes@1
            displayName: Smoke test
            inputs:
              connectionType: Azure Resource Manager
              azureSubscriptionEndpoint: myAzureServiceConnection
              azureResourceGroup: myRG
              kubernetesCluster: myAKS
              command: exec
              arguments: deploy/myapp -- curl -f http://localhost:8080/health

# ─────────────── PRODUCTION ────────────────────────────────────
- stage: DeployProduction
  displayName: Deploy to Production
  dependsOn: DeployStaging
  condition: succeeded()
  jobs:
  - deployment: DeployToProduction
    displayName: Deploy to Production
    environment: production     # requires manual approval gate in Azure DevOps UI
    strategy:
      canary:
        increments: [10, 25, 100]    # canary deployment increments
        preDeploy:
          steps:
          - script: echo "Pre-deploy checks for $(strategy.action)"
        deploy:
          steps:
          - task: KubernetesManifest@1
            inputs:
              action: $(strategy.action)
              manifests: k8s/deployment.yaml
              containers: $(containerRegistry)/$(imageRepository):$(tag)
              percentage: $(strategy.increment)
        postRouteTraffic:
          steps:
          - script: |
              # Wait and check error rate
              sleep 60
              ERROR_RATE=$(curl -s http://prometheus/api/v1/query?query=error_rate | jq '.data.result[0].value[1]')
              if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
                echo "##vso[task.logissue type=error]Error rate too high: $ERROR_RATE"
                exit 1
              fi
            displayName: Validate canary health
        on:
          failure:
            steps:
            - script: echo "Rolling back canary deployment"
            - task: KubernetesManifest@1
              inputs:
                action: reject
                manifests: k8s/deployment.yaml
          success:
            steps:
            - script: echo "Canary promotion successful"
```

---

### 🟡 Q6. What are Azure Pipeline agents — Microsoft-hosted vs self-hosted?
```bash
# Microsoft-hosted agents: provisioned fresh for each job by Microsoft
# Self-hosted agents: your own VMs/containers — persistent, customised

# Microsoft-hosted agent images:
# ubuntu-latest (ubuntu-24.04)
# ubuntu-22.04, ubuntu-20.04
# windows-latest (windows-2022)
# windows-2022, windows-2019
# macOS-latest (macOS-14)
# macOS-14, macOS-13, macOS-12

# Self-hosted agent setup (Linux)
# 1. Create agent pool in Azure DevOps UI
# 2. Download and configure agent
mkdir -p ~/agent && cd ~/agent
curl -L https://vstsagentpackage.azureedge.net/agent/4.0.0/vsts-agent-linux-x64-4.0.0.tar.gz -o agent.tar.gz
tar xzf agent.tar.gz
./config.sh \
  --url https://dev.azure.com/myOrg \
  --auth pat \
  --token <PAT-token> \
  --pool myAgentPool \
  --agent myAgent01 \
  --replace \
  --acceptTeeEula
./svc.sh install && ./svc.sh start

# Self-hosted agent in Docker
docker run -d \
  -e AZP_URL=https://dev.azure.com/myOrg \
  -e AZP_TOKEN=<PAT-token> \
  -e AZP_POOL=myDockerPool \
  -e AZP_AGENT_NAME=docker-agent-01 \
  --name myADOAgent \
  myacr.azurecr.io/ado-agent:latest

# Self-hosted agent in AKS (KEDA-based auto-scaling)
# Uses azure-pipelines-agent Helm chart
helm repo add azure-pipelines-agent https://keda.sh/charts
helm install ado-agent azure-pipelines-agent/azure-pipelines-agent \
  --set pipelines.url=https://dev.azure.com/myOrg \
  --set pipelines.pat=<PAT-token> \
  --set pipelines.poolName=myK8sPool \
  --set replicaCount=3 \
  --set keda.enabled=true \
  --set keda.maxReplicaCount=20

# Use self-hosted agent in pipeline
pool:
  name: myAgentPool       # self-hosted pool name
  demands:
  - docker                # agent must have docker installed
  - Agent.OS -equals Linux
```

---

### 🟡 Q7. What are Azure Pipeline variables, variable groups, and secrets?
```yaml
# Pipeline variables
variables:
  # Static variables
  APP_NAME: myapp
  DOCKER_REGISTRY: myacr.azurecr.io

  # Group reference (Library → Variable Groups in UI)
  - group: production-secrets       # contains DB_PASSWORD, API_KEY, etc.
  - group: app-configuration        # contains APP_CONFIG, FEATURE_FLAGS, etc.

  # Runtime variable (set during run)
  IMAGE_TAG: $(Build.BuildId)

# Use variables
steps:
- script: |
    echo "Deploying $(APP_NAME):$(IMAGE_TAG) to $(DOCKER_REGISTRY)"
    echo "DB: $(DB_HOST)"          # from variable group
    echo "Pass: $(DB_PASSWORD)"    # secret — masked in logs
  env:
    DB_PASSWORD: $(DB_PASSWORD)    # inject secret as env var

# Set variable at runtime (pass between jobs)
- script: |
    VERSION=$(cat version.txt)
    echo "##vso[task.setvariable variable=APP_VERSION;isOutput=true]$VERSION"
  name: setVersion

# Read output variable in next job
- job: NextJob
  dependsOn: PreviousJob
  variables:
    APP_VERSION: $[ dependencies.PreviousJob.outputs['setVersion.APP_VERSION'] ]
  steps:
  - script: echo "Version is $(APP_VERSION)"
```

```bash
# Create variable group (CLI)
az pipelines variable-group create \
  --project myProject \
  --name production-config \
  --variables \
    APP_ENV=production \
    DB_HOST=mydb.postgres.database.azure.com \
    DB_PORT=5432 \
    REDIS_HOST=myredis.redis.cache.windows.net

# Add secret variable (marked as secret)
az pipelines variable-group variable create \
  --project myProject \
  --group-id <group-id> \
  --name DB_PASSWORD \
  --value "MySecretP@ss!" \
  --secret true

# Link variable group to Key Vault (secrets sync automatically)
az pipelines variable-group create \
  --project myProject \
  --name kv-secrets \
  --authorize true \
  --provider-name AzureKeyVault \
  --provider-service-endpoint myAzureServiceConnection \
  --provider-vault-name myKeyVault
# Then select which secrets to sync in Portal
```

---

### 🟡 Q8. What are Azure Pipeline templates?
```yaml
# templates/python-build.yml — reusable build template
parameters:
- name: pythonVersion
  type: string
  default: '3.12'
- name: testPath
  type: string
  default: 'tests/'
- name: requirementsFile
  type: string
  default: 'requirements.txt'

steps:
- task: UsePythonVersion@0
  inputs:
    versionSpec: ${{ parameters.pythonVersion }}
  displayName: Use Python ${{ parameters.pythonVersion }}

- script: pip install -r ${{ parameters.requirementsFile }} pytest pytest-cov
  displayName: Install dependencies

- script: |
    pytest ${{ parameters.testPath }} \
      --junitxml=test-results.xml \
      --cov=src --cov-report=xml
  displayName: Run tests

- task: PublishTestResults@2
  condition: always()
  inputs:
    testResultsFiles: test-results.xml

- task: PublishCodeCoverageResults@2
  inputs:
    summaryFileLocation: coverage.xml
```

```yaml
# templates/deploy-k8s.yml — reusable K8s deployment template
parameters:
- name: environment
  type: string
- name: kubernetesCluster
  type: string
- name: namespace
  type: string
- name: imageTag
  type: string

steps:
- task: KubernetesManifest@1
  displayName: Deploy to ${{ parameters.environment }}
  inputs:
    action: deploy
    connectionType: azureResourceManager
    azureSubscriptionConnection: myAzureServiceConnection
    azureResourceGroup: myRG
    kubernetesCluster: ${{ parameters.kubernetesCluster }}
    namespace: ${{ parameters.namespace }}
    manifests: k8s/*.yaml
    containers: myacr.azurecr.io/myapp:${{ parameters.imageTag }}
    imagePullSecrets: acr-pull-secret
```

```yaml
# azure-pipelines.yml — consume templates
trigger: [main]

stages:
- stage: CI
  jobs:
  - job: Build
    pool: { vmImage: ubuntu-latest }
    steps:
    - template: templates/python-build.yml   # local template
      parameters:
        pythonVersion: '3.12'
        testPath: 'tests/unit tests/integration'

- stage: DeployDev
  jobs:
  - deployment: Dev
    environment: dev
    strategy:
      runOnce:
        deploy:
          steps:
          - template: templates/deploy-k8s.yml
            parameters:
              environment: dev
              kubernetesCluster: myAKS-dev
              namespace: dev
              imageTag: $(Build.BuildId)

# Reference template from another repository
resources:
  repositories:
  - repository: templates
    type: git
    name: myProject/pipeline-templates
    ref: refs/heads/main

steps:
- template: python/build.yml@templates    # cross-repo template
  parameters:
    pythonVersion: '3.12'
```

---

### 🟡 Q9. What are service connections in Azure Pipelines?
```bash
# Service connections: securely store credentials for external services

# Azure Resource Manager service connection
az devops service-endpoint azurerm create \
  --project myProject \
  --name myAzureServiceConnection \
  --azure-rm-service-principal-id <app-id> \
  --azure-rm-subscription-id <sub-id> \
  --azure-rm-subscription-name "My Azure Subscription" \
  --azure-rm-tenant-id <tenant-id>

# Or use Workload Identity Federation (no secret needed — recommended)
az devops service-endpoint azurerm create \
  --project myProject \
  --name myWIFServiceConnection \
  --azure-rm-service-principal-id <app-id> \
  --azure-rm-subscription-id <sub-id> \
  --azure-rm-subscription-name "My Azure Subscription" \
  --azure-rm-tenant-id <tenant-id> \
  --azure-rm-workload-identity-federation  # OIDC-based, no secret rotation needed

# Docker Registry service connection (for ACR)
az devops service-endpoint create \
  --project myProject \
  --service-endpoint-configuration @acr-endpoint.json
# acr-endpoint.json:
# {
#   "type": "dockerregistry",
#   "name": "myACRServiceConnection",
#   "data": {"registrytype": "ACR", "resourceId": "/subscriptions/.../registries/myACR"},
#   "authorization": {"scheme": "WorkloadIdentityFederation", "parameters": {"loginServer": "myacr.azurecr.io"}}
# }

# GitHub service connection
az devops service-endpoint github create \
  --project myProject \
  --name myGitHubConnection \
  --github-url https://github.com/myOrg/myRepo

# Service connection types:
# Azure Resource Manager: deploy to Azure
# Docker Registry:        push/pull Docker images
# Kubernetes:             deploy to any K8s cluster
# GitHub / Bitbucket:     checkout code from external repos
# npm / NuGet / PyPI:     publish packages
# SSH:                    connect to Linux servers
# Generic:                custom endpoints (REST APIs)
```

---

### 🟡 Q10. What are Azure Pipeline environments and gates?
```bash
# Environments: track deployments, require approvals, define health checks

# Create environment (via CLI)
az devops environment create \
  --project myProject \
  --name staging

az devops environment create \
  --project myProject \
  --name production

# Environment checks (configured in UI → Environments → Checks):
# 1. Approvals:          specific people must approve
# 2. Branch control:     only specific branches can deploy
# 3. Business hours:     only deploy during office hours
# 4. Exclusive lock:     only one deployment at a time
# 5. Invoke Azure Function: custom validation via Azure Function
# 6. Invoke REST API:    custom validation via any REST endpoint
# 7. Query Work Items:   check no open P1/P2 bugs before deploy
# 8. Required template:  pipeline must use specific YAML template

# In pipeline YAML:
- deployment: DeployToProd
  environment:
    name: production
    resourceType: Kubernetes
  strategy:
    runOnce:
      preDeploy:
        steps:
        - script: echo "Running pre-deploy checks"
      deploy:
        steps:
        - script: echo "Deploying $(tag)"
      routeTraffic:
        steps:
        - script: echo "Traffic routed"
      postRouteTraffic:
        steps:
        - script: |
            # Custom health check after deployment
            for i in {1..10}; do
              STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://myapp.com/health)
              if [ "$STATUS" = "200" ]; then echo "Healthy"; exit 0; fi
              sleep 30
            done
            echo "Health check failed"
            exit 1
      on:
        failure:
          steps:
          - script: echo "Deployment failed — alerting on-call"
        success:
          steps:
          - script: echo "Deployment successful"
```

---

### 🟡 Q11. What is Azure Artifacts?
```bash
# Azure Artifacts: package management for npm, NuGet, PyPI, Maven, Cargo, Universal

# Create feed
az artifacts universal publish \
  --organization https://dev.azure.com/myOrg \
  --project myProject \
  --feed myFeed \
  --name mypackage \
  --version 1.2.3 \
  --description "My internal package" \
  --path ./dist/

# Download universal package
az artifacts universal download \
  --organization https://dev.azure.com/myOrg \
  --project myProject \
  --feed myFeed \
  --name mypackage \
  --version 1.2.3 \
  --path ./downloaded/

# Publish Python package to Artifacts feed
# In pipeline:
- task: TwineAuthenticate@1
  inputs:
    artifactFeed: myProject/myFeed

- script: |
    pip install build twine
    python -m build
    twine upload -r myFeed dist/*
  displayName: Publish to Azure Artifacts

# Consume from Artifacts (pip)
# pip install --index-url https://pkgs.dev.azure.com/myOrg/myProject/_packaging/myFeed/pypi/simple/ mypackage

# NuGet feed (connect in .csproj or nuget.config)
dotnet nuget add source \
  "https://pkgs.dev.azure.com/myOrg/myProject/_packaging/myFeed/nuget/v3/index.json" \
  --name AzureArtifacts \
  --username myOrg \
  --password <PAT-token>

# npm feed (.npmrc)
# registry=https://pkgs.dev.azure.com/myOrg/myProject/_packaging/myFeed/npm/registry/

# Upstream sources (proxy + cache public registries)
# Enables: one feed for internal + public packages
# Public packages cached in Artifacts: no rate limiting, faster, reliable
# Sources: npmjs.com, nuget.org, pypi.org, maven.org, docker hub

# View packages
az artifacts package list \
  --organization https://dev.azure.com/myOrg \
  --feed myFeed --output table
```

---

### 🟡 Q12. What are Azure Pipeline triggers — CI, PR, scheduled, manual?
```yaml
# ── CI Trigger (push to branches) ────────────────────────────────
trigger:
  batch: true              # wait for in-progress run to finish before queuing
  branches:
    include:
    - main
    - develop
    - release/*
    exclude:
    - feature/experimental-*
  paths:
    include:
    - src/*
    - tests/*
    exclude:
    - docs/*
    - "**/*.md"
  tags:
    include:
    - v*             # also trigger on version tags

# ── PR Trigger (validate PRs) ────────────────────────────────────
pr:
  autoCancel: true   # cancel superseded PR runs
  drafts: false      # don't run on draft PRs
  branches:
    include: [main, develop]
  paths:
    exclude: [docs/*]

# ── Scheduled Trigger ─────────────────────────────────────────────
schedules:
- cron: "0 2 * * 1-5"        # 2 AM UTC, Mon-Fri
  displayName: Nightly build
  branches:
    include: [main]
  always: false              # only run if code changed since last scheduled run

- cron: "0 0 * * 0"          # midnight UTC Sunday
  displayName: Weekly full test
  branches:
    include: [main, develop]
  always: true               # always run regardless of changes

# ── Resource triggers (trigger on another pipeline or container image) ──
resources:
  pipelines:
  - pipeline: build-pipeline  # alias
    source: MyBuildPipeline    # pipeline name in ADO
    trigger:
      branches:
        include: [main]

  containers:
  - container: baseImage
    type: ACR
    azureSubscription: myServiceConnection
    resourceGroup: myRG
    registry: myacr.azurecr.io
    repository: baseimage
    trigger: true   # rebuild when base image is updated

# ── Manual trigger only (disable all auto triggers) ───────────────
trigger: none
pr: none
# Run manually or via API
```

---

### 🟡 Q13. What are pipeline decorators and extensions?
```bash
# Pipeline decorators: inject steps into ALL pipelines in an org
# Use for: security scans, compliance checks, telemetry

# Install extension from Marketplace
az devops extension install \
  --extension-id OWASP.owasp-zap \
  --publisher-id OWASP \
  --org https://dev.azure.com/myOrg

# Custom extension development:
# 1. Create vss-extension.json (manifest)
# 2. Create YAML decorator file
# 3. Package with tfx-cli
# 4. Publish to Marketplace (private or public)

# Decorator YAML example (auto-inject into all pipelines):
# injected-steps.yml
steps:
- task: CredScan@3           # scan for secrets in code
  inputs:
    toolMajorVersion: Latest
  continueOnError: false

- task: PostAnalysis@2       # fail if CredScan found secrets
  inputs:
    CredScan: true
    ToolLogsNotFoundAction: Error

# Key extensions for DevOps:
# Microsoft Security DevOps (MSDO): SAST, IaC scanning, secrets
# OWASP ZAP: DAST security testing
# Terraform: plan/apply in pipelines
# Helm Deploy: K8s Helm chart deployment
# SonarCloud: code quality and security
# Checkmarx: SAST for enterprise
# WhiteSource/Mend: SCA for open source vulnerabilities
```

---

### 🟡 Q14. What are Azure DevOps permissions and access levels?
```bash
# Access levels: Basic, Stakeholder, Visual Studio Subscriber, GitHub Enterprise
# Stakeholder: free, unlimited — access to Boards only (no repos/pipelines)
# Basic: paid — full access to Boards, Repos, Pipelines, Artifacts
# Basic + Test Plans: Basic + Test Plans access

# Add user to organisation
az devops user add \
  --email developer@company.com \
  --license-type express \     # express=Basic, stakeholder, advanced
  --org https://dev.azure.com/myOrg

# Add user to project
az devops team add-member \
  --team "myProject Team" \
  --org https://dev.azure.com/myOrg \
  --members developer@company.com

# Security groups (project-level):
# Project Administrators: full project control
# Build Administrators: manage pipelines and agents
# Readers: view-only access
# Contributors: read/write to repos, create PRs, queue builds
# Project Valid Users: all users in project (auto-added)
# Release Administrators: manage release pipelines

# Assign to group
az devops security group membership add \
  --group-id <group-object-id> \
  --member-id <user-object-id>

# Organization-level security:
# Project Collection Administrators: org-wide admin
# Project Collection Build Service: system account for pipelines

# Branch permissions (restrict who can push to main)
az repos policy required-reviewer create \
  --project myProject \
  --repository-id <repo-id> \
  --branch main \
  --required-reviewer-ids <reviewer-object-id> \
  --is-enabled true --blocking true \
  --message "Architecture review required for main branch changes"
```

---

### 🟡 Q15. What are Azure Pipeline caching and optimisation strategies?
```yaml
# Pipeline cache: reuse downloaded dependencies between runs

# Cache pip dependencies
variables:
  PIP_CACHE_DIR: $(Pipeline.Workspace)/.pip

steps:
- task: Cache@2
  inputs:
    key: 'pip | "$(Agent.OS)" | requirements.txt'
    restoreKeys: |
      pip | "$(Agent.OS)"
    path: $(PIP_CACHE_DIR)
  displayName: Cache pip packages

- script: pip install --cache-dir $(PIP_CACHE_DIR) -r requirements.txt

# Cache npm dependencies
- task: Cache@2
  inputs:
    key: 'npm | "$(Agent.OS)" | package-lock.json'
    restoreKeys: npm | "$(Agent.OS)"
    path: $(npm_config_cache)

- script: npm ci   # use ci instead of install for reproducible builds

# Cache Maven dependencies
- task: Cache@2
  inputs:
    key: 'maven | "$(Agent.OS)" | **/pom.xml'
    path: $(HOME)/.m2/repository

# Cache Docker layers
- task: Cache@2
  inputs:
    key: 'docker | "$(Agent.OS)" | Dockerfile'
    path: /tmp/.buildx-cache

- script: |
    docker buildx build \
      --cache-from type=local,src=/tmp/.buildx-cache \
      --cache-to type=local,dest=/tmp/.buildx-cache-new,mode=max \
      -t myapp:$(Build.BuildId) .
    mv /tmp/.buildx-cache-new /tmp/.buildx-cache

# Performance optimisation:
# 1. Parallel jobs across stages
# 2. Split tests (test splitting with pytest-split)
# 3. Conditional steps (skip unchanged components)
# 4. Artifact caching
# 5. Shallow git clone
- checkout: self
  fetchDepth: 1     # shallow clone — much faster for large repos

# 6. Sparse checkout (only clone needed paths)
- checkout: self
  sparse: true
  sparseCheckoutDirectories: src/ tests/
```

---

### 🔴 Q16. What are GitHub Actions and how do they integrate with Azure?
```yaml
# .github/workflows/deploy-to-azure.yml
name: Deploy to Azure

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: myacr.azurecr.io
  IMAGE_NAME: myapp
  RESOURCE_GROUP: myRG
  AKS_CLUSTER: myAKS

# Use OIDC federation — no stored secrets
permissions:
  id-token: write   # required for OIDC
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.12'
        cache: 'pip'

    - name: Install and test
      run: |
        pip install -r requirements.txt pytest pytest-cov
        pytest tests/ --junitxml=results.xml --cov=src --cov-report=xml

    - name: Upload test results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-results
        path: results.xml

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
    - uses: actions/checkout@v4

    # Authenticate to Azure using OIDC (no stored secrets)
    - name: Azure Login (OIDC)
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Login to ACR
      run: az acr login --name myacr

    - name: Docker meta
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix=sha-
          type=ref,event=branch
          type=semver,pattern={{version}}

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production   # requires approval in GitHub Environments

    steps:
    - uses: actions/checkout@v4

    - name: Azure Login (OIDC)
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Get AKS credentials
      uses: azure/aks-set-context@v3
      with:
        resource-group: ${{ env.RESOURCE_GROUP }}
        cluster-name: ${{ env.AKS_CLUSTER }}

    - name: Deploy to AKS
      uses: azure/k8s-deploy@v4
      with:
        manifests: k8s/
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
        namespace: production
```

---

### 🟡 Q17. What is infrastructure as Code in Azure DevOps with Terraform?
```yaml
# .azure-pipelines/terraform.yml
trigger:
  paths:
    include: [infra/terraform/*]

variables:
  TF_WORKING_DIR: infra/terraform
  ARM_SUBSCRIPTION_ID: $(AZURE_SUBSCRIPTION_ID)
  ARM_TENANT_ID: $(AZURE_TENANT_ID)
  ARM_CLIENT_ID: $(AZURE_CLIENT_ID)
  ARM_USE_OIDC: true     # Workload Identity Federation

stages:
- stage: TerraformPlan
  displayName: Terraform Plan
  jobs:
  - job: Plan
    pool: { vmImage: ubuntu-latest }
    steps:
    - task: TerraformInstaller@1
      inputs:
        terraformVersion: '1.9.0'

    - task: TerraformTaskV4@4
      displayName: Terraform Init
      inputs:
        provider: azurerm
        command: init
        workingDirectory: $(TF_WORKING_DIR)
        backendServiceArm: myAzureServiceConnection
        backendAzureRmResourceGroupName: terraform-state-rg
        backendAzureRmStorageAccountName: tfstatemyorg
        backendAzureRmContainerName: tfstate
        backendAzureRmKey: myapp.tfstate

    - task: TerraformTaskV4@4
      displayName: Terraform Validate
      inputs:
        provider: azurerm
        command: validate
        workingDirectory: $(TF_WORKING_DIR)

    - task: TerraformTaskV4@4
      displayName: Terraform Plan
      inputs:
        provider: azurerm
        command: plan
        workingDirectory: $(TF_WORKING_DIR)
        environmentServiceNameAzureRM: myAzureServiceConnection
        commandOptions: '-out=tfplan -var-file=environments/prod.tfvars'

    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: $(TF_WORKING_DIR)/tfplan
        artifact: terraform-plan

- stage: TerraformApply
  displayName: Terraform Apply
  dependsOn: TerraformPlan
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: Apply
    environment: terraform-production    # requires approval
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              artifact: terraform-plan
              path: $(TF_WORKING_DIR)

          - task: TerraformTaskV4@4
            displayName: Terraform Apply
            inputs:
              provider: azurerm
              command: apply
              workingDirectory: $(TF_WORKING_DIR)
              environmentServiceNameAzureRM: myAzureServiceConnection
              commandOptions: 'tfplan'
```

---

### 🟡 Q18. What is Azure DevOps Wiki?
```bash
# Wiki: built-in markdown documentation, diagrams, meeting notes

# Create wiki (project wiki — stored in own Git repo)
az devops wiki create \
  --project myProject \
  --name "myProject Wiki" \
  --type projectwiki

# Create wiki page
az devops wiki page create \
  --project myProject \
  --wiki "myProject Wiki" \
  --path "/Architecture/System Design" \
  --content "# System Architecture\n\nThis document describes..."
  --comment "Initial architecture document"

# Update wiki page
az devops wiki page update \
  --project myProject \
  --wiki "myProject Wiki" \
  --path "/Architecture/System Design" \
  --version <current-version> \
  --content "# Updated System Architecture\n\n..."
  --comment "Updated diagrams"

# Code Wiki (linked to existing Git repo)
az devops wiki create \
  --project myProject \
  --name "Code Wiki" \
  --type codewiki \
  --repository myRepo \
  --mapped-path /docs   # docs/ folder in repo becomes wiki root

# Wiki supports:
# Mermaid diagrams (flowcharts, sequence, class, gantt)
# MathJax (LaTeX equations)
# @mentions and work item links
# Table of contents [TOC]
# Code blocks with syntax highlighting
```

---

### 🟡 Q19. What are Azure DevOps dashboards and queries?
```bash
# Dashboard: customisable team dashboard with widgets

# Create dashboard
az devops dashboard create \
  --project myProject \
  --name "Sprint Dashboard" \
  --description "Team sprint progress dashboard"

# Widget types:
# Sprint Overview:      remaining capacity, velocity
# Work Item Chart:      bar/pie charts from queries
# Build History:        CI pipeline pass/fail trend
# Deployment Status:    environment deployment status
# Code Coverage:        from pipeline test results
# Lead Time / Cycle Time: flow metrics
# Markdown:             custom text/links
# Azure Monitor Metrics: live Azure metrics

# Work item query
az boards query \
  --project myProject \
  --wiql "
    SELECT [Id],[Title],[State],[AssignedTo],[StoryPoints]
    FROM WorkItems
    WHERE [System.TeamProject] = @Project
      AND [System.IterationPath] UNDER @CurrentIteration
      AND [System.WorkItemType] IN ('User Story','Bug')
    ORDER BY [Microsoft.VSTS.Common.Priority] ASC,
             [System.CreatedDate] DESC
  " --output table

# Save query
az boards query create \
  --project myProject \
  --name "Active Sprint Items" \
  --path "Shared Queries/Sprint" \
  --wiql "SELECT [Id],[Title],[State] FROM WorkItems WHERE [System.IterationPath] UNDER @CurrentIteration"
```

---

### 🟡 Q20. What is Azure Test Plans?
```bash
# Azure Test Plans: manage manual, automated, and exploratory testing

# Test plan structure:
# Test Plan → Test Suite → Test Case

# Create test plan
az boards work-item create \
  --project myProject \
  --type "Test Plan" \
  --title "Release 3.0 Test Plan"

# Test case
az boards work-item create \
  --project myProject \
  --type "Test Case" \
  --title "Verify user can checkout with Apple Pay"

# Test features:
# Manual testing:       execute test cases step by step in browser
# Exploratory testing:  ad-hoc testing with session recording
# Test & Feedback extension: Chrome extension for capture/annotate bugs
# Automated tests:      link to Azure Pipelines test results
# Load testing:         Apache JMeter integration (via Azure Load Testing)
# MTM (deprecated):     migrate to new web-based Test Plans

# Run tests via CLI
az pipelines run \
  --project myProject \
  --name myPipeline \
  --parameters testEnvironment=staging testSuite=1234

# Test impact analysis: only run tests affected by code changes
# Enables faster CI by skipping unrelated tests
```

---

### 🟡 Q21. How do you implement GitOps with Azure DevOps and Flux?
```bash
# GitOps: Git is the single source of truth for cluster state
# Flux v2: K8s operator that continuously syncs cluster to Git repo

# Install Flux in AKS
az k8s-configuration flux create \
  --resource-group myRG \
  --cluster-name myAKS \
  --cluster-type managedClusters \
  --name cluster-config \
  --namespace flux-system \
  --scope cluster \
  --url https://dev.azure.com/myOrg/myProject/_git/k8s-config \
  --branch main \
  --ssh-private-key-file ~/.ssh/flux_deploy_key

# GitOps pipeline flow:
# Developer pushes code →
# Azure Pipeline builds image, pushes to ACR, updates image tag in Git →
# Flux detects change in Git →
# Flux applies Kubernetes manifests to cluster →
# Cluster reaches desired state

# Flux Kustomization (multi-environment)
az k8s-configuration flux create \
  --resource-group myRG \
  --cluster-name myAKS \
  --cluster-type managedClusters \
  --name apps \
  --namespace flux-system \
  --url https://dev.azure.com/myOrg/myProject/_git/k8s-config \
  --branch main \
  --kustomization name=infrastructure path=./infrastructure prune=true interval=10m \
  --kustomization name=apps path=./apps/production prune=true interval=5m dependsOn=infrastructure

# Update image tag in GitOps repo (in pipeline after Docker push)
- script: |
    git clone https://$(ADO_PAT)@dev.azure.com/myOrg/myProject/_git/k8s-config
    cd k8s-config
    # Update image tag using yq or kustomize
    kustomize edit set image myacr.azurecr.io/myapp:$(Build.BuildId)
    git add .
    git commit -m "chore: update myapp to $(Build.BuildId) [skip ci]"
    git push
  displayName: Update GitOps repo
```

---

### 🟡 Q22. What is Azure DevOps security scanning in pipelines?
```yaml
# Microsoft Security DevOps (MSDO) — single task, runs all security tools
- task: MicrosoftSecurityDevOps@1
  displayName: Security Scan
  inputs:
    categories: 'IaC,containers,secrets,code,dependency'
    tools: 'checkov,terrascan,trivy,bandit,credscan,eslint'
    break: false

# Individual security tasks:

# CredScan (detect secrets in code)
- task: CredScan@3
  inputs:
    toolMajorVersion: Latest
    scanFolder: $(Build.SourcesDirectory)
    suppressionsFile: .gitsecret/suppressions.json

- task: PostAnalysis@2
  inputs:
    CredScan: true

# Trivy (container vulnerability scanning)
- script: |
    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
    trivy image \
      --exit-code 1 \
      --severity HIGH,CRITICAL \
      --format sarif \
      --output trivy-results.sarif \
      myacr.azurecr.io/myapp:$(Build.BuildId)
  displayName: Trivy vulnerability scan

# Publish security results to Azure DevOps
- task: PublishBuildArtifacts@1
  inputs:
    pathToPublish: trivy-results.sarif
    artifactName: SecurityResults

# OWASP Dependency Check (SCA)
- task: dependency-check-build-task@6
  inputs:
    projectName: myProject
    scanPath: .
    format: JUNIT
    failOnCVSS: 7   # fail if CVSS score >= 7

# SonarCloud (code quality + security)
- task: SonarCloudPrepare@1
  inputs:
    SonarCloud: mySonarConnection
    organization: myOrg
    scannerMode: CLI
    configMode: manual
    cliProjectKey: myProject
    cliSources: src/

- script: pytest tests/ --cov=src --cov-report=xml
- task: SonarCloudAnalyze@1
- task: SonarCloudPublish@1
  inputs:
    pollingTimeoutSec: 300
```

---

### 🟢 Q23. What are Azure DevOps notifications and integrations?
```bash
# Notifications: email/Teams/Slack alerts on DevOps events

# Subscription types:
# Build: succeeded, failed, partially succeeded
# Release: deployment started, succeeded, failed
# Code: PR created, PR merge failed, PR comment added
# Work Items: assigned to me, state changed, comment added
# Extensions: custom notification types

# Integrate with Microsoft Teams
# Install "Azure DevOps" app in Teams →
# Add tab to channel → connect to ADO project
# Or use incoming webhook → ADO Service Hook → Teams

# Create service hook (Slack notification on build fail)
az devops service-endpoint create \
  --project myProject \
  --service-endpoint-configuration @slack-hook.json

# slack-hook.json:
# {
#   "type": "slack",
#   "url": "https://hooks.slack.com/services/...",
#   "events": [{"eventType": "build.complete", "filters": {"buildStatus": "Failed"}}]
# }

# Microsoft Teams notification on deployment
az devops service-hook create \
  --project myProject \
  --event-type ms.vss-release.deployment-completed-event \
  --consumer-id teams \
  --consumer-action-id postMessageToConversation \
  --consumer-inputs connectorId=<connector-id> \
  --publisher-inputs releaseEnvironmentId=<env-id>
```

---

### 🟢 Q24. What is the difference between Azure DevOps and GitHub?
| Feature | Azure DevOps | GitHub |
|---------|-------------|--------|
| **Boards** | Excellent (Agile, CMMI, Scrum) | Basic (Projects beta) |
| **Repos** | Full-featured Git | Full-featured Git + Copilot |
| **Pipelines CI/CD** | Azure Pipelines (YAML/Classic) | GitHub Actions |
| **Package Management** | Azure Artifacts | GitHub Packages |
| **Test Management** | Azure Test Plans (paid) | Via 3rd party |
| **Security Scanning** | MSDO, CredScan | Advanced Security (GHAS) |
| **AI Assistance** | (limited) | GitHub Copilot (strong) |
| **Enterprise Features** | Strong (auditing, compliance) | Strong (GHAS, Enterprise) |
| **Pricing** | 5 free users + Parallel jobs | Public repos free, private tiered |
| **Preferred by** | Microsoft/.NET shops, enterprise | Open source, modern teams |

---

### 🟢 Q25. What are Azure DevOps PAT tokens and authentication?
```bash
# Personal Access Tokens (PAT): scoped tokens for authentication

# Create PAT (via portal: User Settings → Personal Access Tokens)
# Or via API:
curl -X POST \
  -H "Authorization: Basic $(echo -n :$ADO_PAT | base64)" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "CI/CD Token",
    "scope": "vso.build_execute vso.code_read vso.packaging",
    "allOrgs": false,
    "validTo": "2027-01-01T00:00:00.000Z"
  }' \
  "https://vssps.dev.azure.com/myOrg/_apis/tokens/pats?api-version=7.1-preview.1"

# PAT scopes:
# vso.build_execute:  queue and manage builds
# vso.code_full:      read/write code, PRs, branches
# vso.code_read:      read-only code access
# vso.packaging:      read/write packages
# vso.release:        manage releases
# vso.work_full:      read/write work items
# vso.graph:          read users/groups

# Authentication methods:
# PAT:             username + PAT as password (git clone, REST API)
# OAuth 2.0:       Entra ID app registration (server-side flows)
# Service Principal: OIDC/federated credential for pipelines
# SSH key:         for Git operations only (no REST API access)

# GitHub Actions → Azure DevOps via PAT
# Store PAT as GitHub Secret → use in workflow:
- name: Queue Azure DevOps pipeline
  run: |
    curl -X POST \
      -H "Authorization: Basic $(echo -n :${{ secrets.ADO_PAT }} | base64)" \
      -H "Content-Type: application/json" \
      -d '{"definition":{"id":1},"sourceBranch":"refs/heads/main"}' \
      "https://dev.azure.com/myOrg/myProject/_apis/build/builds?api-version=7.1"
```


---

# PART 2 — AZURE MIGRATION

---

### 🟢 Q26. What is Azure Migrate and what does it do?
**Answer:**
Azure Migrate is the central hub for discovering, assessing, and migrating on-premises servers, databases, web apps, and data to Azure. It is free to use (you pay only for Azure resources used).

```bash
# Create Azure Migrate project
az migrate project create \
  --resource-group myMigrateRG \
  --project-name myMigrateProject \
  --location eastus \
  --assessment-solution-properties '{}'

# Azure Migrate components:
# Discovery and Assessment tool: discover VMware/Hyper-V/physical servers
# Migration and Modernization tool: agentless/agent-based VM migration
# Azure Migrate: Database Migration (integrates with DMS)
# Azure Migrate: Web App Migration (App Service Migration Assistant)
# Azure Migrate: Azure Stack HCI (migrate to Azure Stack)

# Migration phases:
# 1. DISCOVER:   inventory all on-prem assets (servers, databases, apps, dependencies)
# 2. ASSESS:     evaluate readiness, estimate costs, get right-sizing recommendations
# 3. MIGRATE:    replicate, test, and cut over to Azure
# 4. OPTIMISE:   right-size, reserve instances, governance

# Appliance-based discovery:
# Download OVA (VMware) or VHD (Hyper-V) template from portal
# Deploy appliance VM in on-prem environment
# Register appliance with Azure Migrate project
# Appliance continuously discovers and sends metadata to Azure

# Supported sources:
# VMware vSphere: agentless (uses vCenter API)
# Hyper-V:        uses Hyper-V host agent
# Physical/bare-metal: lightweight agent installed on each server
# AWS EC2, GCP VMs: treated as physical servers
```

---

### 🟢 Q27. What are the migration strategies — 6Rs?
**Answer:**
The 6Rs (or 7Rs) framework defines how to migrate each workload:

| Strategy | Also Called | Description | When to Use |
|---------|------------|-------------|------------|
| **Rehost** | Lift-and-shift | Move VM as-is to Azure VM | Quick migration, no code changes |
| **Replatform** | Lift-and-optimise | Minor changes (SQL DB → Azure SQL, Java → App Service) | Small effort, managed service benefits |
| **Refactor** | Re-architect | Redesign to use cloud-native services | Maximum cloud benefits |
| **Repurchase** | Replace | Switch to SaaS (on-prem CRM → Dynamics 365) | Vendor-managed, modern functionality |
| **Retain** | Revisit | Keep on-premises for now | Regulatory, latency, not yet ready |
| **Retire** | Decommission | Shut down unused workloads | Identified as unused during discovery |
| **Relocate** | (7th R) | Move to Azure VMware Solution without changes | VMware-dependent workloads |

---

### 🟡 Q28. How does agentless VMware VM migration work?
```bash
# Azure Migrate: Migration and Modernization tool
# Agentless VMware migration: no agent installed in VMs
# Uses vSphere APIs via replication appliance

# Steps:
# 1. Deploy Azure Migrate Replication Appliance (OVA in vCenter)
# 2. Register with Azure Migrate project
# 3. Configure source (vCenter credentials)
# 4. Enable replication for VMs
# 5. Run test migration
# 6. Run actual migration (cutover)

# Enable replication (via portal — CLI limited for migration tool)
# az migrate start replication  (not yet fully in CLI)

# Using az REST to trigger migration:
az rest --method POST \
  --url "https://management.azure.com/subscriptions/<sub>/resourceGroups/myMigrateRG/providers/Microsoft.RecoveryServices/vaults/myMigrateVault/replicationFabrics/myFabric/replicationProtectionContainers/myContainer/replicationMigrationItems/myVM/migrate?api-version=2021-02-10" \
  --body '{"properties":{"migrationDetails":{"os":"Linux"}}}'

# Agent-based migration (for physical servers, AWS, GCP):
# 1. Install Azure Site Recovery Mobility Service agent on source VMs
# 2. Configure replication toward Azure Recovery Services Vault
# 3. Monitor replication health
# 4. Run test failover → verify in Azure
# 5. Run planned failover → final cutover

# Test migration:
# Creates a test VM in Azure from latest replication point
# Does NOT interrupt production
# Validate: connectivity, apps, database integrity
# Clean up test VMs after validation

# Cutover (actual migration):
# Stop replication traffic from source
# Final sync of changed blocks
# Start VM in Azure
# Update DNS/load balancer to point to Azure
# Decommission on-prem VM after validation period
```

---

### 🟡 Q29. What is Azure Database Migration Service (DMS)?
```bash
# DMS: fully managed service for migrating databases to Azure
# Supported sources:
# SQL Server → Azure SQL DB, SQL MI, SQL Server on Azure VM
# MySQL → Azure DB for MySQL
# PostgreSQL → Azure DB for PostgreSQL
# Oracle → Azure SQL DB, SQL MI (via SSMA)
# MongoDB → Azure Cosmos DB
# Amazon RDS → Azure SQL DB, SQL MI, MySQL, PostgreSQL

# DMS migration modes:
# Online (minimal downtime): continuous replication, short cutover window
# Offline (scheduled downtime): full backup restore, acceptable downtime

# Create DMS instance
az dms create \
  --resource-group myMigrateRG \
  --service-name myDMS \
  --location eastus \
  --sku-name Standard_4vCores \
  --vnet /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet \
  --subnet /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/mySubnet

# Create migration project
az dms project create \
  --resource-group myMigrateRG \
  --service-name myDMS \
  --project-name SqlMigration \
  --location eastus \
  --source-platform SQL \
  --target-platform SQLDB \     # SQLDB | SQLMI | PostgreSQL | MySQL | CosmosDb

# Create migration task (SQL → Azure SQL DB, offline)
az dms project task create \
  --resource-group myMigrateRG \
  --service-name myDMS \
  --project-name SqlMigration \
  --task-name sqlMigrateTask \
  --task-type OfflineMigration \
  --source-connection-json @source-connection.json \
  --target-connection-json @target-connection.json \
  --database-options-json '[{
    "name": "mySourceDB",
    "targetDatabaseName": "myTargetDB",
    "makeSourceDbReadOnly": false,
    "tableMap": {}
  }]'

# source-connection.json
# {
#   "userName": "sa",
#   "password": "P@ss!",
#   "dataSource": "192.168.1.100,1433",
#   "authType": "SqlAuthentication",
#   "encryptConnection": true,
#   "trustServerCertificate": true
# }

# target-connection.json
# {
#   "userName": "sqladmin",
#   "password": "AzureP@ss!",
#   "dataSource": "mysqlserver.database.windows.net",
#   "authType": "SqlAuthentication",
#   "encryptConnection": true,
#   "trustServerCertificate": false
# }

# Monitor task
az dms project task show \
  --resource-group myMigrateRG \
  --service-name myDMS \
  --project-name SqlMigration \
  --task-name sqlMigrateTask \
  --query "properties.state"

# Azure SQL Migration extension in Azure Data Studio:
# GUI-based approach for SQL Server → Azure SQL DB/MI
# Assessment + online migration in one tool
# Recommended over DMS portal for SQL migrations
```

---

### 🟡 Q30. How do you migrate SQL Server to Azure SQL Managed Instance?
```bash
# SQL MI: highest compatibility, minimal code changes
# Features missing vs on-prem: SQL Server Agent (partial), MSDTC, some xp_ procs

# Step 1: Assessment — use Database Experimentation Assistant (DEA)
# or Azure Migrate Database Assessment

# Step 2: Schema migration with SSMA or DMS Schema Conversion
# SSMA (SQL Server Migration Assistant) for Oracle/MySQL/PostgreSQL

# Step 3: SQL MI Link (online migration — near-zero downtime)
# Creates distributed Always On AG between on-prem and SQL MI

# On SQL Server side:
# sp_configure 'contained database authentication', 1
# CREATE ENDPOINT HaEndpoint AS TCP (LISTENER_PORT = 5022)
# FOR DATABASE_MIRRORING (ROLE = ALL, AUTHENTICATION = CERTIFICATE miCert, ENCRYPTION = REQUIRED)

# Create SQL MI Link
az sql mi link create \
  -g myRG --instance-name mySQLMI -n myLink \
  --primary-availability-group-name myAG \
  --source-endpoint "TCP://onprem-sql.corp.local:5022" \
  --replication-mode Sync

# Monitor replication lag
az sql mi link show -g myRG --instance-name mySQLMI -n myLink \
  --query "properties.replicationState"

# Step 4: Test migration (read-only validation on MI side)
# Step 5: Cutover
# - Stop all writes to on-prem DB
# - Wait for final sync (lag = 0)
az sql mi link failover \
  -g myRG --instance-name mySQLMI -n myLink \
  --failover-type PlannedFailover

# - Update connection strings in application
# - Decommission on-prem AG

# Large database offline migration (backup/restore):
az storage container create -n sqlbackups \
  --account-name mystorageaccount --auth-mode login

# On SQL Server:
# BACKUP DATABASE myDB TO URL = 'https://mystorageaccount.blob.core.windows.net/sqlbackups/myDB.bak'
# WITH CREDENTIAL = 'MyAzureCredential', COMPRESSION, STATS = 10

# Restore on SQL MI:
# RESTORE DATABASE myDB FROM URL = 'https://mystorageaccount.blob.core.windows.net/sqlbackups/myDB.bak'
# WITH MOVE 'myDB' TO 'D:\DATA\myDB.mdf', MOVE 'myDB_log' TO 'D:\LOG\myDB.ldf', STATS = 10
```

---

### 🟡 Q31. How do you migrate web applications to Azure App Service?
```bash
# App Service Migration Assistant: GUI tool for .NET/PHP web apps
# Download: https://azure.microsoft.com/migration/web-applications/

# Manual migration steps:

# 1. Create App Service (same OS as source)
az appservice plan create -g myMigrateRG -n myAppPlan --sku P2V3 --is-linux
az webapp create -g myMigrateRG --plan myAppPlan -n myWebApp \
  --runtime "DOTNET:8.0"

# 2. Configure app settings
az webapp config appsettings set -g myMigrateRG -n myWebApp \
  --settings \
    ConnectionStrings__DefaultConnection="Server=mysqlserver.database.windows.net;Database=mydb;Authentication=Active Directory Managed Identity" \
    ASPNETCORE_ENVIRONMENT=Production

# 3. Deploy application
# Option A: ZIP deploy
az webapp deploy -g myMigrateRG -n myWebApp \
  --src-path ./publish.zip --type zip

# Option B: Azure DevOps Pipeline deploy
# Option C: GitHub Actions

# 4. Migrate sessions to Azure Cache for Redis
# (replace in-proc session with distributed session)

# 5. Configure custom domain and TLS
az webapp config hostname add -g myMigrateRG \
  --webapp-name myWebApp --hostname www.example.com

# 6. Configure scaling
az monitor autoscale create -g myMigrateRG --resource myAppPlan \
  --resource-type Microsoft.Web/serverFarms \
  --name webAutoscale --min-count 2 --max-count 20 --count 3

# Containerise first (if refactoring):
# Dockerfile + docker build → push to ACR → deploy as container app

# Common issues during web app migration:
# Windows file system paths → use blob storage or Azure Files
# Windows registry → use app settings
# MSMQ → use Service Bus
# COM components → refactor to managed code
# Windows authentication → use Entra ID + MSAL
# Crystal Reports → SSRS or Azure Analysis Services
```

---

### 🟡 Q32. What is Azure VMware Solution (AVS)?
```bash
# AVS: run VMware workloads natively in Azure (dedicated bare-metal)
# No re-architecture needed — vSphere, vSAN, NSX-T all included

# When to use AVS:
# VMware-dependent workloads (complex NSX-T networks, vSAN, vMotion)
# Regulatory: must keep VMware licensing
# Migration stepping stone: migrate to AVS first, modernise later

# Create AVS private cloud
az vmware private-cloud create \
  --resource-group myAVSRG \
  --name myAVSCloud \
  --location eastus \
  --sku av36 \           # av20 | av36 | av36P | av52 | av64
  --cluster-size 3 \     # minimum 3 hosts, max 16 per cluster
  --network-block 10.20.0.0/22 \   # management CIDR (must be /22)
  --nsxt-password "NSXtP@ss!" \
  --vcenter-password "vCenterP@ss!" \
  --no-wait

# Add cluster
az vmware cluster create \
  --resource-group myAVSRG \
  --private-cloud-name myAVSCloud \
  --name mySecondCluster \
  --sku av36 --cluster-size 3

# Connect AVS to Azure VNet
az vmware authorization create \
  --resource-group myAVSRG \
  --private-cloud myAVSCloud \
  --name myCircuitAuth

# Express Route connection to VNet
az network vpn-connection create \
  --resource-group myAVSRG \
  --name avs-to-azure \
  --vnet-gateway1 myERGW \
  --express-route-circuit2 /subscriptions/<sub>/resourceGroups/myAVSRG/providers/Microsoft.AVS/privateClouds/myAVSCloud/authorizations/myCircuitAuth

# AVS pricing: billed per host per hour
# av36: ~$6.50/host/hour = ~$4,700/host/month
# Must commit to 3-host minimum

# HCX for live migration:
# Migrate running VMware VMs to AVS without downtime
# Using vMotion over WAN (HCX Network Extension)
```

---

### 🟡 Q33. What is Azure Stack HCI?
```bash
# Azure Stack HCI: hyper-converged infrastructure (on-prem)
# Runs Azure Kubernetes Service, Azure Arc, Azure Virtual Desktop
# Connected to Azure for management, updates, and billing

# Register Azure Stack HCI cluster
az stack-hci cluster create \
  --resource-group myHCIRG \
  --name myHCICluster \
  --location eastus \
  --aad-client-id <app-id> \
  --aad-tenant-id <tenant-id>

# Enable AKS on Stack HCI
az aksarc create \
  --resource-group myHCIRG \
  --name myAKSHCI \
  --custom-location /subscriptions/<sub>/resourceGroups/myHCIRG/providers/Microsoft.ExtendedLocation/customLocations/myCustomLocation \
  --node-count 3 --node-vm-size Standard_A4_v2

# Use cases for Azure Stack HCI:
# Low-latency apps (manufacturing, retail)
# Data residency (cannot send data to cloud)
# Disconnected environments (military, submarine)
# Gradual cloud migration path
```

---

### 🟡 Q34. What is Azure Arc and how does it extend Azure management?
```bash
# Azure Arc: extend Azure management plane to any infrastructure
# Supported: on-prem servers, VMware VMs, Hyper-V, AWS EC2, GCP VMs, K8s clusters

# Arc-enabled servers
az connectedmachine connect \
  --resource-group myArcRG \
  --name myOnPremServer \
  --location eastus \
  --correlation-id $(uuidgen)

# Or generate onboarding script
az connectedmachine generate-install-script \
  --resource-group myArcRG \
  --location eastus \
  --output-file install.sh
# Download script, run on each on-prem server

# Arc-enabled Kubernetes (connect any K8s cluster)
az connectedk8s connect \
  --resource-group myArcRG \
  --name myOnPremK8s \
  --location eastus

# GitOps on Arc-enabled K8s (Flux v2)
az k8s-configuration flux create \
  --resource-group myArcRG \
  --cluster-name myOnPremK8s \
  --cluster-type connectedClusters \
  --name cluster-config \
  --url https://dev.azure.com/myOrg/myProject/_git/k8s-config \
  --branch main

# Arc-enabled data services (SQL MI on any K8s)
az arcdata dc create \
  --resource-group myArcRG \
  --name myArcDataController \
  --location eastus \
  --connectivity-mode indirect \   # direct (connected) | indirect (air-gap)
  --infrastructure onpremises \
  --k8s-namespace arc-data

# Apply Azure Policy to Arc servers
az policy assignment create \
  --name "require-tags-arc-servers" \
  --scope /subscriptions/<sub>/resourceGroups/myArcRG \
  --policy <built-in-policy-id>

# Arc benefits:
# ✅ Unified Azure Portal for all resources (on-prem, AWS, GCP)
# ✅ Azure Monitor Agent on non-Azure servers
# ✅ Defender for Cloud across hybrid estate
# ✅ Azure Policy compliance on on-prem servers
# ✅ Update Manager for on-prem VMs
# ✅ Azure Automanage best practices applied
# ✅ SQL MI and PostgreSQL on any Kubernetes
```

---

### 🟡 Q35. What is the Cloud Adoption Framework (CAF) for Azure?
```bash
# CAF: Microsoft's structured guide for Azure adoption

# CAF phases:
# 1. STRATEGY:  Define motivations, business outcomes, financial justification
# 2. PLAN:      Inventory assets, rationalise portfolio (6Rs), skill gaps
# 3. READY:     Azure Landing Zone setup (subscriptions, management groups, policies)
# 4. ADOPT:
#    - Migrate: migrate existing workloads (Rehost, Replatform)
#    - Innovate: build new cloud-native apps
# 5. GOVERN:    Azure Policy, Cost Management, Security Baseline
# 6. MANAGE:    Azure Monitor, Update Manager, Business Continuity

# Azure Landing Zone (CAF-aligned subscription structure):
# Root MG
# ├── Platform
# │   ├── Identity sub (Entra DS, AD Connect)
# │   ├── Management sub (Log Analytics, Defender, Sentinel)
# │   └── Connectivity sub (Hub VNet, Firewall, VPN/ER Gateway)
# ├── Landing Zones
# │   ├── Corp (VNet-connected, internal apps)
# │   └── Online (internet-facing, no VNet required)
# ├── Sandbox (dev/test, relaxed policies)
# └── Decommissioned

az account management-group create --name Platform --display-name "Platform"
az account management-group create --name LandingZones --display-name "Landing Zones"
az account management-group create --name Corp --display-name "Corp" --parent LandingZones
az account management-group create --name Online --display-name "Online" --parent LandingZones

# Deploy Azure Landing Zone accelerator
# Portal: https://aka.ms/caf/ready/accelerator
# Bicep: https://github.com/Azure/ALZ-Bicep
# Terraform: https://github.com/Azure/terraform-azurerm-caf-enterprise-scale
```

---

### 🟡 Q36. What is Azure Cost Management for migration planning?
```bash
# TCO Calculator: estimate cost savings vs on-premises
# https://azure.microsoft.com/pricing/tco/calculator/

# Azure Pricing Calculator: estimate Azure resource costs
# https://azure.microsoft.com/pricing/calculator/

# Azure Migrate cost estimate: right-sized Azure VM recommendations
# Assessment → Cost report → Export to Excel

# Track actual migration costs
az consumption usage list \
  --billing-period-name "202606" \
  --query "[?contains(instanceName,'migrate')]" \
  --output table

# Create budget for migration project
az consumption budget create \
  --budget-name MigrationBudget \
  --amount 50000 \
  --time-grain Monthly \
  --resource-group myMigrateRG \
  --time-period start=2026-06-01 end=2026-12-31 \
  --notifications '[{
    "enabled": true,
    "operator": "GreaterThan",
    "threshold": 80,
    "contactEmails": ["migration-team@company.com"]
  }]'

# Cost optimisation after migration:
# 1. Right-size based on actual usage (Advisor recommendations)
az advisor recommendation list --filter "Category eq 'Cost'" --output table

# 2. Reserved Instances for stable workloads (1 or 3 year)
# 3. Azure Hybrid Benefit (Windows + SQL Server licences)
# 4. Dev/Test subscriptions (discounted rates for non-production)
# 5. Auto-shutdown dev VMs
# 6. Storage tiering (Cool/Archive for backup data)
```

---

### 🟡 Q37. How do you migrate data to Azure with Azure Data Factory?
```bash
# ADF as migration tool: move large datasets from on-prem to Azure
# Supports: SQL Server, Oracle, SAP, filesystems, Hadoop → ADLS, Blob, SQL

# Create ADF for migration
az datafactory create -g myMigrateRG -n myMigrationADF --location eastus

# Self-hosted Integration Runtime (for on-prem connectivity)
az datafactory integration-runtime create \
  -g myMigrateRG --factory-name myMigrationADF \
  -n mySHIR --type SelfHosted

# Get auth key for SHIR agent
az datafactory integration-runtime list-auth-key \
  -g myMigrateRG --factory-name myMigrationADF \
  -n mySHIR --query "authKey1" -o tsv

# Install SHIR agent on on-prem server with this auth key
# Download: https://www.microsoft.com/download/details.aspx?id=39717

# Create linked service (on-prem SQL Server via SHIR)
az datafactory linked-service create \
  -g myMigrateRG --factory-name myMigrationADF \
  --linked-service-name OnPremSQL \
  --properties '{
    "type": "SqlServer",
    "typeProperties": {
      "connectionString": "Server=192.168.1.100;Database=myDB;Integrated Security=true"
    },
    "connectVia": {
      "referenceName": "mySHIR",
      "type": "IntegrationRuntimeReference"
    }
  }'

# Create copy pipeline (full load)
az datafactory pipeline create \
  -g myMigrateRG --factory-name myMigrationADF \
  --pipeline-name MigrateAllTables \
  --pipeline '{
    "activities": [{
      "name": "CopyAllTables",
      "type": "Copy",
      "inputs": [{"referenceName": "SourceSQLDataset", "type": "DatasetReference"}],
      "outputs": [{"referenceName": "TargetADLSDataset", "type": "DatasetReference"}],
      "typeProperties": {
        "source": {"type": "SqlServerSource", "sqlReaderQuery": "SELECT * FROM myTable"},
        "sink": {"type": "ParquetSink", "storeSettings": {"type": "AzureBlobFSWriteSettings"}},
        "parallelCopies": 8,
        "dataIntegrationUnits": 32
      }
    }]
  }'

# Run migration pipeline
az datafactory pipeline create-run \
  -g myMigrateRG --factory-name myMigrationADF \
  --pipeline-name MigrateAllTables
```

---

### 🟢 Q38. What is the Azure Migrate assessment process?
```bash
# Assessment types:
# Azure VM assessment:       right-size + cost for VM migration
# AVS assessment:            size for Azure VMware Solution
# Azure SQL assessment:      readiness for Azure SQL DB/MI/SQL VM
# Web App assessment:        readiness for Azure App Service
# Azure Stack HCI assessment: hybrid on-prem sizing

# After appliance discovers servers → run assessment
# Assessment output:
# Readiness: Ready | Ready with conditions | Not ready | Unknown
# Recommended SKU: right-sized Azure VM (based on CPU/memory utilisation)
# Cost estimate: monthly cost for compute + storage
# Dependency map: what connects to what (requires agent or agentless network analysis)

# Key assessment settings:
# Comfort factor:    buffer over actual utilisation (default 1.3 = 30% headroom)
# Performance history: 1d, 7d, 30d, custom (use perf data for right-sizing)
# Percentile:        50th, 90th, 95th, 99th (default 95th)
# Pricing:           PAYG or Reserved (1yr, 3yr) or Dev/Test
# Hybrid Benefit:    apply Windows licence savings

# Export assessment
az migrate assessment export \
  --resource-group myMigrateRG \
  --project-name myMigrateProject \
  --assessment-name myAssessment \
  --output-file assessment.xlsx
```

---

### 🟡 Q39. What is minimal downtime database migration strategy?
```bash
# Minimal downtime migration: online replication → short cutover window

# Strategy:
# Phase 1: Initial load (full backup/restore) — hours/days
# Phase 2: Ongoing replication (CDC — Change Data Capture) — real-time
# Phase 3: Validation — run reports, check row counts, test app
# Phase 4: Cutover — switch connection strings, very short outage

# SQL Server → Azure SQL DB (online, using DMS):
# 1. Enable CDC on source database
# EXEC sys.sp_cdc_enable_db
# EXEC sys.sp_cdc_enable_table @source_schema='dbo', @source_name='Orders', ...

# 2. Configure DMS online task
az dms project task create \
  --resource-group myMigrateRG \
  --service-name myDMS \
  --project-name SqlMigration \
  --task-name onlineMigrateTask \
  --task-type OnlineMigration \
  --source-connection-json @source.json \
  --target-connection-json @target.json \
  --selected-databases-json '[{"name":"myDB","targetDatabaseName":"myDB"}]'

# 3. Monitor replication lag
az dms project task show \
  --resource-group myMigrateRG \
  --service-name myDMS \
  --project-name SqlMigration \
  --task-name onlineMigrateTask \
  --expand output \
  --query "properties.output[0].migrationReportResult.id"

# 4. Initiate cutover when ready
az dms project task cancel \
  --resource-group myMigrateRG --service-name myDMS \
  --project-name SqlMigration --task-name onlineMigrateTask
# Then: start cutover in portal → change connection strings → verify → done

# PostgreSQL online migration:
# Use pglogical or AWS DMS-compatible tools
# Or Azure DMS for PostgreSQL → PostgreSQL Flexible Server
```

---

### 🟡 Q40. What are migration waves and prioritisation?
```bash
# Migration wave planning: group workloads into sequential migration batches

# Wave 1 - Low risk, quick wins (rehost):
# Development/test VMs
# Internal tools, monitoring servers
# Intranet sites
# Expected: 2-4 weeks

# Wave 2 - Medium risk (rehost/replatform):
# Internal line-of-business apps
# Batch processing servers
# File servers (→ Azure Files)
# Expected: 4-8 weeks

# Wave 3 - Higher risk (production workloads):
# Customer-facing web applications
# Internal databases
# ERP/CRM systems
# Expected: 6-12 weeks

# Wave 4 - Complex workloads (refactor):
# High-availability clusters
# SAP/Oracle ERP
# Real-time data processing
# Expected: 3-6 months

# Dependency mapping (critical for wave planning):
# Azure Migrate dependency analysis shows which VMs communicate
# Ensures: dependent VMs migrate in same wave
# Tool: Service Map (requires Dependency Agent) or agentless (network data)

# Migration tracking via Azure DevOps Boards:
az boards work-item create --project myProject \
  --type Feature --title "Wave 1: DEV/TEST Migration" \
  --tags "migration; wave-1"

# Use epics per wave, features per workload, stories per migration step
```

---

### 🟡 Q41. What are common migration challenges and solutions?
```bash
# Challenge 1: Large on-prem data (PBs) — internet upload too slow
# Solution: Azure Data Box (80TB–1PB physical device)
az databox job create -g myMigrateRG -n myDataBoxJob \
  --location eastus \
  --contact-details contactPerson="Admin" phone="5551234567" emailList="admin@co.com" \
  --destination-account-details storageAccountId=/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount \
  --sku DataBox \
  --shipping-address streetAddress1="123 Main St" city="Redmond" stateOrProvince="WA" postalCode="98052" country="US" addressType=Commercial

# Challenge 2: Network bandwidth too low for replication
# Solution: ExpressRoute for dedicated bandwidth + Azure Data Box for initial seed
# Steps: Data Box seeds data → ER provides ongoing delta replication

# Challenge 3: Windows authentication apps
# Solution: Extend AD to Azure via AD Connect + Azure VPN/ER
# Or: Migrate to Entra ID authentication (MSAL)

# Challenge 4: Hardcoded IP addresses in apps
# Solution: Use private DNS zones — apps resolve by hostname, not IP
# Change app config to use FQDNs instead of IPs

# Challenge 5: Legacy .NET Framework apps
# Solution:
# - Rehost as-is on Windows App Service
# - Containerise with Windows container
# - Replatform to .NET 8 (use Upgrade Assistant tool)
az dotnet-upgrade-assistant upgrade --project myApp.csproj \
  --target-framework net8.0

# Challenge 6: SQL Server features not in Azure SQL DB
# Solution: Migrate to SQL Managed Instance instead
# or refactor: SQL Agent → Azure Functions, Linked Server → ADF, SSIS → ADF

# Challenge 7: Oracle → Azure
# SSMA (SQL Server Migration Assistant) for Oracle
# AWS Schema Conversion Tool (SCT) alternative
# OracleDB@Azure (native Oracle deployment in Azure)
```

---

### 🟡 Q42. What is Azure Migrate for containerisation?
```bash
# App Containerisation Assistant: discovers Java/ASP.NET apps → generates Dockerfiles + Kubernetes manifests

# Supported:
# ASP.NET apps on IIS → Windows containers on AKS/App Service
# Java web apps on Apache Tomcat → Linux containers

# Steps:
# 1. Install App Containerisation tool on Windows machine in same network as IIS servers
# 2. Tool discovers IIS apps, analyses binaries, generates Dockerfile
# 3. Build container image (builds directly in ACR)
# 4. Generate AKS deployment manifests
# 5. Deploy to AKS

# Generated Dockerfile example (ASP.NET):
# FROM mcr.microsoft.com/dotnet/framework/aspnet:4.8-windowsservercore-ltsc2022
# WORKDIR /inetpub/wwwroot
# COPY . .
# RUN powershell -Command \
#   Import-Module WebAdministration; \
#   New-WebApplication -Name myapp -Site 'Default Web Site' -PhysicalPath 'C:\inetpub\wwwroot\myapp'

# Generated AKS manifest:
# apiVersion: apps/v1
# kind: Deployment
# metadata:
#   name: myapp
# spec:
#   replicas: 2
#   template:
#     spec:
#       nodeSelector:
#         "kubernetes.io/os": windows
#       containers:
#       - name: myapp
#         image: myacr.azurecr.io/myapp:v1.0
#         ports: [{containerPort: 80}]
```

---

### 🟢 Q43. What are Azure migration tools comparison?
| Tool | Purpose | Source | Target |
|------|---------|--------|--------|
| **Azure Migrate** | Discovery + assessment + migration hub | VMware, Hyper-V, Physical | Azure VMs |
| **Azure Site Recovery** | VM replication + DR + migration | VMware, Hyper-V, Physical | Azure VMs |
| **Azure DMS** | Database migration | SQL, MySQL, PostgreSQL, MongoDB, Oracle | Azure PaaS DBs |
| **Azure Data Box** | Offline bulk data transfer | Any on-prem | Azure Storage |
| **AzCopy** | Online file/blob transfer | Any | Azure Storage |
| **Azure Data Factory** | ETL/data pipeline migration | SQL, Oracle, SAP, FS, Hadoop | Azure Storage/SQL |
| **SSMA** | SQL Server migration assessment | Oracle, MySQL, PostgreSQL, Access | Azure SQL |
| **App Containerisation** | Containerise IIS/Tomcat apps | IIS, Tomcat | AKS, App Service |
| **App Migration Assistant** | Web app migration | IIS | App Service |

---

### 🟡 Q44. What is the Well-Architected Framework for migration?
```bash
# Azure Well-Architected Framework (WAF) — 5 pillars to design good architectures

# Reliability:
# - Multi-region or multi-AZ deployment
# - RTO/RPO targets met
# - Health checks, circuit breakers, retry patterns
az aks create --zones 1 2 3   # zone-redundant AKS

# Security:
# - Least-privilege RBAC
# - Managed Identities (no secrets)
# - Private endpoints for PaaS
# - Defender for Cloud enabled
az security pricing create --name VirtualMachines --tier Standard

# Cost Optimisation:
# - Right-sized VMs (Advisor recommendations)
# - Reserved Instances for predictable workloads
# - Autoscale for variable workloads
az advisor recommendation list --filter "Category eq 'Cost'" -o table

# Operational Excellence:
# - CI/CD pipelines (Azure Pipelines / GitHub Actions)
# - Infrastructure as Code (Bicep / Terraform)
# - Monitoring and alerting (Azure Monitor)

# Performance Efficiency:
# - Azure CDN for static content
# - Redis caching for DB reads
# - Premium SSD for IO-intensive workloads
# - AKS Horizontal Pod Autoscaler
```

---

### 🟡 Q45. What is Azure Policy for migration governance?
```bash
# Enforce governance during migration

# Require tagging (cost allocation)
az policy assignment create \
  --name "require-migration-tags" \
  --scope /subscriptions/<sub>/resourceGroups/myMigrateRG \
  --policy <require-tag-policy-id> \
  --params '{"tagName":{"value":"MigrationWave"}}'

# Allowed regions (data sovereignty)
az policy assignment create \
  --name "allowed-regions" \
  --scope /subscriptions/<sub> \
  --policy "e56962a6-4747-49cd-b67b-bf8b01975c4f" \
  --params '{"listOfAllowedLocations":{"value":["eastus","westus2","northeurope"]}}'

# Deny public IPs (security baseline)
az policy assignment create \
  --name "deny-public-ip" \
  --scope /subscriptions/<sub>/resourceGroups/myMigrateRG \
  --policy "83a86a26-fd1f-447c-b59d-daf3f1205012"

# Require Defender plans
az policy assignment create \
  --name "require-defender" \
  --scope /subscriptions/<sub> \
  --policy-set-definition <defender-initiative-id>

# Check compliance after migration
az policy state list \
  --filter "ComplianceState eq 'NonCompliant'" \
  --query "[].{Resource:resourceId,Policy:policyDefinitionName}" \
  --output table
```


---

# PART 3 — SERVERLESS

---

### 🟢 Q46. What is serverless computing and what Azure services are serverless?
**Answer:**
Serverless means you run code or workloads without managing servers. Azure handles provisioning, scaling, patching, and capacity. You pay only for what you use (per execution, per request, per unit consumed).

| Service | Type | Trigger |
|---------|------|---------|
| **Azure Functions** | Code execution | HTTP, Timer, Event, Queue, etc. |
| **Container Apps** | Container execution | HTTP, KEDA events, Jobs |
| **Logic Apps** | Workflow automation | 1000+ connectors |
| **Event Grid** | Event routing | 50+ system topics |
| **Service Bus** | Message broker | Queue/Topic |
| **Event Hubs** | Event streaming | AMQP/HTTP/Kafka |
| **API Management (Consumption)** | API gateway | HTTP |
| **Cosmos DB (Serverless)** | NoSQL database | SDK |
| **SQL DB (Serverless)** | Relational DB | SDK |
| **Azure Static Web Apps** | Frontend hosting | Git push |
| **Power Automate** | Low-code automation | Triggers |

---

### 🟢 Q47. What is Azure Functions in detail?
```bash
# Azure Functions: event-driven, serverless compute — run code in response to events

# Hosting plans (already covered in Q16 of Part 1 — recap here):
# Consumption:      Auto-scale to 0, cold starts, max 10 min exec, 1.5GB RAM
# Flex Consumption: Per-instance concurrency, VNet, up to 4GB, faster start
# Premium (EP1-3):  Pre-warmed, no cold start, VNet, unlimited exec, up to 14GB
# Dedicated:        Always on, App Service Plan, predictable cost

# Function App configuration
az functionapp create -g myRG -n myFunc \
  --consumption-plan-location eastus \
  --runtime python --runtime-version 3.12 \
  --functions-version 4 \
  --storage-account mystorageaccount \
  --assign-identity '[{"type":"SystemAssigned"}]'

# All environment variables (app settings)
az functionapp config appsettings set -g myRG -n myFunc \
  --settings \
    SERVICEBUS_CONNECTION="Endpoint=sb://..." \
    COSMOS_CONNECTION="AccountEndpoint=..." \
    KEY_VAULT_URI="https://myKV.vault.azure.net" \
    ENVIRONMENT="production"

# Disable specific function
az functionapp function keys set -g myRG \
  --function-app myFunc \
  --function-name myHttpFunc \
  --key-name default \
  --key-value ""    # clear key = disable

# Scale settings (Premium)
az functionapp config set -g myRG -n myFunc \
  --prewarmed-instance-count 2 \     # always warm
  --minimum-elastic-instance-count 2

# VNet integration (for private resources access)
az functionapp vnet-integration add -g myRG -n myFunc \
  --vnet myVNet --subnet funcSubnet

# Deployment slots
az functionapp deployment slot create -g myRG \
  --name myFunc --slot staging

az functionapp deployment slot swap -g myRG \
  --name myFunc --slot staging
```

---

### 🟡 Q48. What are all Azure Functions triggers and bindings?
```python
# Input bindings: read data when function runs
# Output bindings: write data when function completes
# Triggers: what causes the function to run

import azure.functions as func
import logging
import json

app = func.FunctionApp()

# ── HTTP Trigger (REST API) ────────────────────────────────────────
@app.route(route="orders/{orderId}", methods=["GET","POST","PUT","DELETE"],
           auth_level=func.AuthLevel.FUNCTION)
def http_orders(req: func.HttpRequest) -> func.HttpResponse:
    order_id = req.route_params.get("orderId")
    if req.method == "GET":
        return func.HttpResponse(json.dumps({"id": order_id}),
                                 mimetype="application/json")
    body = req.get_json()
    return func.HttpResponse(json.dumps(body), status_code=201,
                             mimetype="application/json")

# ── Timer Trigger (CRON) ──────────────────────────────────────────
@app.timer_trigger(schedule="0 0 2 * * *",   # 2 AM daily
                   arg_name="timer",
                   run_on_startup=False)
def daily_cleanup(timer: func.TimerRequest) -> None:
    logging.info(f"Daily cleanup. Past due: {timer.past_due}")

# ── Blob Trigger + Blob Output binding ───────────────────────────
@app.blob_trigger(arg_name="inputBlob",
                  path="uploads/{name}.csv",
                  connection="AzureWebJobsStorage")
@app.blob_output(arg_name="outputBlob",
                 path="processed/{name}.parquet",
                 connection="AzureWebJobsStorage")
def process_csv(inputBlob: func.InputStream, outputBlob: func.Out[bytes]) -> None:
    data = inputBlob.read()
    processed = transform_csv_to_parquet(data)
    outputBlob.set(processed)

# ── Queue Trigger + Queue Output ──────────────────────────────────
@app.queue_trigger(arg_name="msg",
                   queue_name="orders",
                   connection="AzureWebJobsStorage")
@app.queue_output(arg_name="outMsg",
                  queue_name="processed-orders",
                  connection="AzureWebJobsStorage")
def process_queue(msg: func.QueueMessage, outMsg: func.Out[str]) -> None:
    order = json.loads(msg.get_body().decode())
    result = process_order(order)
    outMsg.set(json.dumps(result))

# ── Service Bus Trigger + Table Output ────────────────────────────
@app.service_bus_queue_trigger(arg_name="msg",
                                queue_name="orders",
                                connection="SERVICEBUS_CONNECTION")
@app.table_output(arg_name="outTable",
                  table_name="ProcessedOrders",
                  connection="AzureWebJobsStorage")
def sb_to_table(msg: func.ServiceBusMessage,
                outTable: func.Out[str]) -> None:
    order = json.loads(msg.get_body().decode())
    outTable.set(json.dumps({
        "PartitionKey": order["customerId"],
        "RowKey": order["orderId"],
        "Amount": order["amount"],
        "Status": "Processed"
    }))

# ── Event Hub Trigger + Cosmos DB Output ──────────────────────────
@app.event_hub_message_trigger(arg_name="events",
                                event_hub_name="telemetry",
                                cardinality=func.Cardinality.MANY,
                                consumer_group="functions-consumer",
                                connection="EVENTHUB_CONNECTION")
@app.cosmos_db_output(arg_name="outputDocs",
                       database_name="myDB",
                       container_name="telemetry",
                       connection="COSMOS_CONNECTION",
                       create_if_not_exists=True)
def eh_to_cosmos(events: list[func.EventHubEvent],
                 outputDocs: func.Out[func.DocumentList]) -> None:
    docs = []
    for event in events:
        data = json.loads(event.get_body().decode())
        docs.append({
            "id": event.sequence_number,
            "deviceId": data["deviceId"],
            "temperature": data["temperature"],
            "timestamp": event.enqueued_time.isoformat()
        })
    outputDocs.set(func.DocumentList(docs))

# ── Cosmos DB Change Feed Trigger ─────────────────────────────────
@app.cosmos_db_trigger(arg_name="documents",
                        database_name="myDB",
                        container_name="orders",
                        connection="COSMOS_CONNECTION",
                        lease_container_name="leases",
                        create_lease_container_if_not_exists=True)
def cosmos_changefeed(documents: func.DocumentList) -> None:
    for doc in documents:
        logging.info(f"Changed document: {doc['id']}")
        publish_event(doc)   # trigger downstream workflow

# ── Durable Functions Orchestrator ───────────────────────────────
import azure.durable_functions as df

orchestrator_app = df.Blueprint()

@orchestrator_app.orchestration_trigger(context_name="context")
def order_orchestrator(context: df.DurableOrchestrationContext):
    order = context.get_input()
    validated = yield context.call_activity("ValidateOrder", order)
    charged = yield context.call_activity("ChargePayment", validated)
    shipped = yield context.call_activity("ShipOrder", charged)
    yield context.call_activity("SendConfirmation", shipped)
    return shipped
```

---

### 🟡 Q49. What are Azure Functions best practices?
```python
# 1. Stateless functions (don't store state in memory between invocations)
# BAD:
class_level_cache = {}  # shared state — breaks on scale-out

# GOOD:
# Use Redis, Cosmos DB, or Azure Cache for shared state

# 2. Idempotent functions (safe to retry)
def process_order(order_id: str) -> None:
    # Check if already processed before doing work
    existing = cosmos.get_item(order_id)
    if existing and existing.get("status") == "processed":
        logging.info(f"Order {order_id} already processed — skipping")
        return
    do_processing(order_id)
    cosmos.upsert_item({"id": order_id, "status": "processed"})

# 3. Use managed identity (no connection strings in code)
from azure.identity import DefaultAzureCredential
from azure.servicebus import ServiceBusClient

credential = DefaultAzureCredential()
sb_client = ServiceBusClient("mySBNamespace.servicebus.windows.net", credential)

# 4. Configure max concurrency (avoid thundering herd)
# host.json
{
  "version": "2.0",
  "extensions": {
    "serviceBus": {
      "maxConcurrentCalls": 16,
      "maxConcurrentSessions": 8
    },
    "eventHubs": {
      "maxEventBatchSize": 100,
      "prefetchCount": 300
    }
  },
  "functionTimeout": "00:10:00"
}

# 5. Async I/O (don't block threads)
import asyncio
import aiohttp

async def fetch_data(url: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

# 6. Retry policies (transient failures)
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3),
       wait=wait_exponential(multiplier=1, min=2, max=10))
def call_external_api(payload: dict) -> dict:
    response = requests.post("https://api.example.com/process", json=payload)
    response.raise_for_status()
    return response.json()

# 7. Structured logging
import structlog
log = structlog.get_logger()

def process_event(event_id: str) -> None:
    log.info("processing_event", event_id=event_id, service="order-processor")

# 8. Dead letter handling
# Configure maxDeliveryCount on Service Bus queue
# Function reads DLQ periodically and alerts/reprocesses

# 9. Cold start mitigation
# Premium plan: pre-warmed instances
# Optimize imports: lazy load heavy modules
# Keep Functions deployment package small

# 10. Secrets via Key Vault reference
# App setting value:
# @Microsoft.KeyVault(VaultName=myKV;SecretName=db-password)
# No code change needed — resolved at runtime
```

---

### 🟡 Q50. What is Azure Logic Apps?
```bash
# Logic Apps: low-code/no-code workflow automation with 1000+ connectors
# Triggers: HTTP, Schedule, Event Grid, Service Bus, SharePoint, Outlook, etc.

# SKUs:
# Consumption: multi-tenant, pay-per-action, great for simple workflows
# Standard:    single-tenant, VNet integration, stateful+stateless, CI/CD

# Create Standard Logic App
az logicapp create -g myRG -n myLogicApp \
  --storage-account mystorageaccount \
  --plan myAppPlan \           # App Service Plan
  --runtime-version ~4 \
  --assign-identity '[{"type":"SystemAssigned"}]'

# Common Logic Apps patterns:

# 1. HTTP webhook → parse → Service Bus
# Trigger: HTTP request
# Action:  Parse JSON
# Action:  Send message to Service Bus queue

# 2. Email → ticket creation
# Trigger: When email arrives in Outlook
# Condition: Subject contains "Support Request"
# Action:    Create ticket in ServiceNow/Jira
# Action:    Reply to sender with ticket number

# 3. Blob created → process → notify
# Trigger: When blob is created in storage
# Action:  Call Azure Function for processing
# Action:  Send Teams notification on result

# 4. Scheduled report
# Trigger: Recurrence (daily at 8 AM)
# Action:  Query SQL Database
# Action:  Format as HTML table
# Action:  Send email via Outlook/SendGrid

# 5. Event Grid → Logic App → multiple notifications
# Trigger: Event Grid (resource health alert)
# Parallel branches:
#   Branch A: Create PagerDuty incident
#   Branch B: Post to Teams channel
#   Branch C: Create ServiceNow incident

# Deploy workflow via ARM template
az deployment group create -g myRG \
  --template-file logicapp-workflow.json \
  --parameters logicAppName=myLogicApp

# Run history (debugging)
az rest --method GET \
  --url "https://management.azure.com/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Logic/workflows/myLogicApp/runs?api-version=2016-06-01&\$top=10" \
  --query "value[].{RunId:name,Status:properties.status,StartTime:properties.startTime}" \
  --output table
```

---

### 🟡 Q51. What is Azure Event Grid in detail?
```bash
# Event Grid: serverless event routing — push model
# Sources → Event Grid → Subscribers (webhook, Function, SB, EH, Logic App)
# Retry: up to 24h with exponential backoff (max 30 retries)

# System topics (no custom setup needed):
# Microsoft.Storage.StorageAccounts
# Microsoft.ContainerRegistry.Registries
# Microsoft.Resources.Subscriptions
# Microsoft.Resources.ResourceGroups
# Microsoft.KeyVault.vaults
# Microsoft.ServiceBus.Namespaces
# Microsoft.EventHub.Namespaces
# Microsoft.AppService.ServerFarms
# Microsoft.ContainerService.ManagedClusters

# Subscribe to blob created event → Azure Function
az eventgrid event-subscription create \
  --name blobCreatedSub \
  --source-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount \
  --endpoint https://myfunc.azurewebsites.net/api/ProcessNewBlob?code=<key> \
  --endpoint-type webhook \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with /blobServices/default/containers/uploads/ \
  --subject-ends-with .csv \
  --max-delivery-attempts 30 \
  --event-ttl 1440

# Event Grid Domain (fan-out to thousands of topics)
az eventgrid domain create -g myRG -n myDomain --location eastus

# Each tenant gets a topic: myDomain/topics/<tenant-id>
# Publisher sends with topic header → tenant-specific delivery

# CloudEvents schema (CNCF standard — recommended)
az eventgrid topic create -g myRG -n myCloudEventsTopic \
  --location eastus \
  --input-schema CloudEventSchemaV1_0

# Advanced filtering (route only matching events)
az eventgrid event-subscription update \
  --name mySubscription \
  --source-resource-id <resource-id> \
  --advanced-filter data.amount NumberGreaterThan 1000 \
  --advanced-filter data.status StringIn "new" "pending"

# Dead-letter destination (undeliverable events)
az eventgrid event-subscription update \
  --name mySubscription \
  --source-resource-id <resource-id> \
  --deadletter-endpoint \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount/blobServices/default/containers/deadletter

# Event Grid Namespaces (MQTT 3.1.1 + 5.0 for IoT)
az eventgrid namespace create -g myRG -n myEGNamespace \
  --location eastus \
  --topic-spaces-configuration '{"state":"Enabled"}'
```

---

### 🟡 Q52. What is Azure Service Bus in detail?
```bash
# Service Bus: enterprise message broker — guaranteed delivery, ordering, DLQ

# Create Premium namespace
az servicebus namespace create -g myRG -n mySBNamespace \
  --location eastus --sku Premium --capacity 4 \
  --zone-redundant true \
  --disable-local-auth true   # force AAD auth

# Queue (point-to-point)
az servicebus queue create -g myRG --namespace-name mySBNamespace \
  -n orders \
  --max-size 5120 \
  --default-message-time-to-live P14D \
  --dead-lettering-on-message-expiration true \
  --duplicate-detection-history-time-window PT10M \
  --lock-duration PT5M \
  --max-delivery-count 10 \
  --enable-partitioning true \     # 16 partitions for high throughput
  --enable-session true            # FIFO per session ID (order per customer)

# Topic + Subscriptions (pub/sub)
az servicebus topic create -g myRG --namespace-name mySBNamespace \
  -n events --enable-partitioning true

az servicebus topic subscription create -g myRG \
  --namespace-name mySBNamespace --topic-name events \
  -n email-processor \
  --max-delivery-count 5 --lock-duration PT2M

# SQL filter (route by message content)
az servicebus topic subscription rule create -g myRG \
  --namespace-name mySBNamespace --topic-name events \
  --subscription-name email-processor \
  -n orderFilter \
  --filter-type SqlFilter \
  --filter-sql-expression "eventType = 'order.created' AND amount > 100"

# Correlation filter (faster — indexed lookup)
az servicebus topic subscription rule create -g myRG \
  --namespace-name mySBNamespace --topic-name events \
  --subscription-name priority-handler -n priorityFilter \
  --filter-type CorrelationFilter \
  --correlation-filter-properties '{"priority":"high","eventType":"order.created"}'
```

```python
# Python: Service Bus with sessions (FIFO per customer)
from azure.servicebus.aio import ServiceBusClient
from azure.servicebus import ServiceBusMessage
from azure.identity.aio import DefaultAzureCredential

async def send_order(order: dict):
    credential = DefaultAzureCredential()
    async with ServiceBusClient(
        "mySBNamespace.servicebus.windows.net", credential
    ) as client:
        async with client.get_queue_sender("orders") as sender:
            msg = ServiceBusMessage(
                body=json.dumps(order).encode(),
                session_id=order["customerId"],    # FIFO per customer
                content_type="application/json",
                subject="order.created",
                application_properties={"priority": "high"},
                time_to_live=timedelta(days=7)
            )
            await sender.send_messages(msg)

async def process_orders(customer_id: str):
    async with ServiceBusClient(
        "mySBNamespace.servicebus.windows.net", DefaultAzureCredential()
    ) as client:
        # Receive in session order for a specific customer
        async with client.get_queue_receiver(
            "orders", session_id=customer_id, prefetch_count=20
        ) as receiver:
            async for msg in receiver:
                try:
                    order = json.loads(msg.body)
                    await process(order)
                    await receiver.complete_message(msg)
                except TransientError:
                    await receiver.abandon_message(msg)    # retry
                except PermanentError:
                    await receiver.dead_letter_message(    # send to DLQ
                        msg, reason="PermanentFailure",
                        error_description=str(e)
                    )
```

---

### 🟡 Q53. What is Azure Event Hubs in detail?
```bash
# Event Hubs: high-throughput event streaming — millions of events/second
# Kafka-compatible (drop-in replacement for Kafka brokers)

az eventhubs namespace create -g myRG -n myEHNamespace \
  --location eastus --sku Premium --capacity 4 \
  --zone-redundant true --disable-local-auth true

az eventhubs eventhub create -g myRG \
  --namespace-name myEHNamespace -n telemetry \
  --partition-count 32 \      # determines max parallel consumers; immutable
  --message-retention 7 \     # days (up to 90 for Premium)
  --cleanup-policy Delete     # Delete | Compact (log compaction)

# Consumer groups (one per independent consumer)
az eventhubs eventhub consumer-group create -g myRG \
  --namespace-name myEHNamespace --eventhub-name telemetry \
  -n asa-consumer     # Stream Analytics
az eventhubs eventhub consumer-group create -g myRG \
  --namespace-name myEHNamespace --eventhub-name telemetry \
  -n databricks-consumer

# Capture to ADLS (automatic raw data archival)
az eventhubs eventhub update -g myRG \
  --namespace-name myEHNamespace -n telemetry \
  --enable-capture true \
  --capture-interval 300 \           # seconds (60-900)
  --capture-size-limit 314572800 \   # bytes (max 524MB)
  --destination-name EventHubArchive.AzureBlockBlob \
  --storage-account /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mydatalake \
  --blob-container captures \
  --archive-name-format "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"

# Schema Registry (Avro schema validation)
az eventhubs namespace schema-registry create -g myRG \
  --namespace-name myEHNamespace -n mySchemaGroup \
  --schema-compatibility Backward --schema-type Avro
```

```python
# Python: Produce and consume with checkpointing
from azure.eventhub.aio import EventHubProducerClient, EventHubConsumerClient
from azure.eventhub import EventData
from azure.eventhub.extensions.checkpointstoreblobaio import BlobCheckpointStore
from azure.identity.aio import DefaultAzureCredential

async def send_events(events: list[dict]):
    producer = EventHubProducerClient(
        fully_qualified_namespace="myEHNamespace.servicebus.windows.net",
        eventhub_name="telemetry",
        credential=DefaultAzureCredential()
    )
    async with producer:
        # Group by partition key for ordering per device
        batch = await producer.create_batch(
            partition_key=events[0]["deviceId"]
        )
        for event in events:
            batch.add(EventData(json.dumps(event)))
        await producer.send_batch(batch)

async def consume_events():
    checkpoint_store = BlobCheckpointStore.from_connection_string(
        "<storage-conn>", container_name="checkpoints"
    )
    consumer = EventHubConsumerClient(
        fully_qualified_namespace="myEHNamespace.servicebus.windows.net",
        eventhub_name="telemetry",
        consumer_group="databricks-consumer",
        checkpoint_store=checkpoint_store,
        credential=DefaultAzureCredential()
    )

    async def on_event(partition_context, event):
        data = json.loads(event.body_as_str())
        await process_telemetry(data)
        await partition_context.update_checkpoint(event)   # commit offset

    async def on_partition_initialize(partition_context):
        logging.info(f"Partition {partition_context.partition_id} initialized")

    async with consumer:
        await consumer.receive(
            on_event=on_event,
            on_partition_initialize=on_partition_initialize,
            starting_position="-1",    # -1=earliest, @latest=new only
            max_wait_time=30
        )
```

---

### 🟡 Q54. What is Azure API Management (APIM) in serverless context?
```bash
# APIM Consumption tier: serverless API gateway
# Pay per 1M calls, no VNet, no SLA — perfect for serverless backends

az apim create -g myRG -n myAPIM \
  --publisher-email admin@company.com \
  --publisher-name "My Company" \
  --sku-name Consumption \     # Consumption = serverless API GW
  --location eastus

# Import Azure Function as API
az apim api import -g myRG --service-name myAPIM \
  --path /orders \
  --specification-format OpenApi \
  --specification-url https://myfunc.azurewebsites.net/api/swagger.json \
  --api-id orders-api \
  --display-name "Orders API" \
  --protocols https

# Set backend to Azure Function
az apim backend create -g myRG --service-name myAPIM \
  --backend-id orders-backend \
  --protocol http \
  --url https://myfunc.azurewebsites.net/api \
  --credentials-header "x-functions-key" "<function-key>"

# Apply policies (rate limit + JWT validation)
az apim api policy create -g myRG --service-name myAPIM \
  --api-id orders-api \
  --value '<policies><inbound><rate-limit calls="100" renewal-period="60"/><validate-jwt header-name="Authorization"><openid-config url="https://login.microsoftonline.com/<tenant>/.well-known/openid-configuration"/></validate-jwt></inbound><backend><forward-request/></backend><outbound><base/></outbound></policies>'

# APIM as facade for multiple serverless backends:
# /orders → Azure Function
# /products → Container App
# /users → App Service
# /analytics → Logic App
# Single entry point, consistent auth, rate limiting for all
```

---

### 🟡 Q55. What is Azure Static Web Apps?
```bash
# Static Web Apps: host SPAs (React, Angular, Vue, Blazor) + serverless API (Functions)
# Auto-deploys from GitHub / Azure DevOps on push
# Free SSL, custom domain, global CDN included

az staticwebapp create -g myRG \
  -n myStaticWebApp \
  --source https://github.com/myOrg/myRepo \
  --branch main \
  --app-location "/" \          # frontend source (where package.json is)
  --output-location "build" \   # build output folder
  --api-location "api" \        # Azure Functions folder (optional)
  --login-with-github           # authenticates CLI with GitHub

# staticwebapp.config.json (routing + auth)
{
  "routes": [
    {
      "route": "/api/*",
      "allowedRoles": ["authenticated"]
    },
    {
      "route": "/admin/*",
      "allowedRoles": ["admin"]
    },
    {
      "route": "/*",
      "serve": "/index.html",
      "statusCode": 200
    }
  ],
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/api/*", "/*.{css,scss,js,png,gif,ico,html}"]
  },
  "auth": {
    "identityProviders": {
      "azureActiveDirectory": {
        "registration": {
          "openIdIssuer": "https://login.microsoftonline.com/<tenant-id>/v2.0",
          "clientIdSettingName": "AZURE_CLIENT_ID",
          "clientSecretSettingName": "AZURE_CLIENT_SECRET"
        }
      },
      "github": {
        "registration": {
          "clientIdSettingName": "GITHUB_CLIENT_ID",
          "clientSecretSettingName": "GITHUB_CLIENT_SECRET"
        }
      }
    }
  },
  "globalHeaders": {
    "Cache-Control": "no-cache",
    "X-Content-Type-Options": "nosniff"
  },
  "mimeTypes": {
    ".json": "text/json"
  }
}

# App settings
az staticwebapp appsettings set -n myStaticWebApp -g myRG \
  --setting-names \
    COSMOS_CONNECTION="AccountEndpoint=..." \
    API_KEY="mySecretKey"

# Get deployment URL
az staticwebapp show -n myStaticWebApp -g myRG \
  --query defaultHostname -o tsv
```

---

### 🟡 Q56. What is serverless architecture patterns?
```python
# Pattern 1: Event-driven microservices
# Web → API → Event Grid → Functions (fan-out) → multiple backends

# Pattern 2: Saga pattern for distributed transactions
# Orchestration saga via Durable Functions
import azure.durable_functions as df

def order_saga(context: df.DurableOrchestrationContext):
    order = context.get_input()
    compensations = []
    try:
        # Step 1: Reserve inventory
        reservation = yield context.call_activity("ReserveInventory", order)
        compensations.append(("CancelReservation", reservation["id"]))

        # Step 2: Charge payment
        payment = yield context.call_activity("ChargePayment", order)
        compensations.append(("RefundPayment", payment["id"]))

        # Step 3: Ship order
        shipment = yield context.call_activity("ShipOrder", order)
        return {"status": "success", "shipmentId": shipment["id"]}

    except Exception as e:
        # Compensate in reverse order
        for activity, id in reversed(compensations):
            yield context.call_activity(activity, id)
        return {"status": "failed", "error": str(e)}

# Pattern 3: CQRS (Command Query Responsibility Segregation)
# Commands → Service Bus → Function (write) → Cosmos DB (write model)
# Queries → Function (read) → Redis Cache → Cosmos DB (read model/replica)

# Pattern 4: Competing consumers (parallel processing)
# Service Bus queue with N Functions processing concurrently
# Each message processed by exactly one Function instance

# Pattern 5: Event sourcing
# All state changes as events → Event Hub → Functions → Cosmos DB
# Replay events to rebuild state at any point in time

# Pattern 6: Choreography vs Orchestration
# Choreography: services react to events (loose coupling)
#   Event Grid → each service subscribes to what it cares about
# Orchestration: central coordinator (Durable Functions)
#   Orchestrator calls activities in sequence, handles errors

# Pattern 7: Throttle/Circuit breaker
import functools
from datetime import datetime, timedelta

class CircuitBreaker:
    def __init__(self, failure_threshold=5, reset_timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED | OPEN | HALF_OPEN

    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if datetime.now() > self.last_failure_time + timedelta(seconds=60):
                self.state = "HALF_OPEN"
            else:
                raise Exception("Circuit breaker OPEN")
        try:
            result = func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except Exception:
            self.failure_count += 1
            self.last_failure_time = datetime.now()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise
```

---

### 🟢 Q57. What is the comparison between Functions, Container Apps, and Logic Apps?
| Feature | Azure Functions | Container Apps | Logic Apps |
|---------|---------------|----------------|-----------|
| **Code** | Required | Required | Optional (low-code) |
| **Language** | .NET, Java, Python, Node, PS, Go (custom) | Any (containers) | JSON workflow |
| **Execution time** | Max 10min (Consumption), unlimited (Premium) | Unlimited | Unlimited |
| **Scale to zero** | Yes | Yes | Yes |
| **VNet** | Premium only | Yes (standard) | Standard only |
| **State** | Durable Functions | External store | Built-in |
| **Connectors** | Bindings (~20) | Any (code) | 1000+ native |
| **Pricing** | Per execution | Per replica-second | Per action |
| **Best for** | Event-driven code, short tasks | Long-running, containerised | Business process automation |

---

### 🟡 Q58. How do you monitor serverless applications?
```bash
# Application Insights is essential for serverless monitoring

az monitor app-insights component create \
  --app myServerlessAI -g myRG --location eastus \
  --kind web --application-type web \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myLA

# Connect to Function App
az functionapp config appsettings set -g myRG -n myFunc \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=xxx;..."

# Connect to Logic Apps
az logicapp config appsettings set -g myRG -n myLogicApp \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=xxx;..."

# Key KQL queries for serverless monitoring
```

```kusto
// ── Function executions by status ─────────────────────────────────
requests
| where timestamp > ago(1h)
| summarize Total = count(), Failed = countif(success == false),
            SuccessRate = round(100.0 * countif(success) / count(), 2)
    by name
| order by Total desc

// ── Function cold starts ──────────────────────────────────────────
traces
| where timestamp > ago(1h)
| where message contains "Host started"
| project timestamp, operation_Name, cloud_RoleInstance
| summarize ColdStarts = count() by operation_Name, bin(timestamp, 5m)

// ── Function execution duration by percentile ─────────────────────
requests
| where timestamp > ago(1h)
| where name contains "HttpTrigger"
| summarize P50=percentile(duration,50), P95=percentile(duration,95),
            P99=percentile(duration,99), Count=count()
    by name, bin(timestamp, 5m)

// ── Logic App run status ──────────────────────────────────────────
AzureDiagnostics
| where ResourceType == "WORKFLOWS/RUNS"
| where TimeGenerated > ago(24h)
| summarize Runs = count() by status_s, bin(TimeGenerated, 1h)
| order by TimeGenerated desc

// ── Event Hub consumer lag ────────────────────────────────────────
AzureMetrics
| where ResourceProvider == "MICROSOFT.EVENTHUB"
| where MetricName == "IncomingMessages"
| summarize Total = sum(Total) by bin(TimeGenerated, 1h)

// ── Service Bus queue depth ────────────────────────────────────────
AzureMetrics
| where ResourceProvider == "MICROSOFT.SERVICEBUS"
| where MetricName == "ActiveMessages"
| summarize MaxDepth = max(Maximum) by ResourceId, bin(TimeGenerated, 5m)
| where MaxDepth > 1000   // alert when > 1000 messages waiting

// ── Function errors with stack trace ─────────────────────────────
exceptions
| where timestamp > ago(1h)
| project timestamp, operation_Name, type, outerMessage,
          innerMessage, stack = outerStackTrace
| order by timestamp desc
```

---

### 🟢 Q59. What is Azure serverless security best practices?
```bash
# 1. Never store secrets in code or environment variables as plaintext
# Use Key Vault references:
az functionapp config appsettings set -g myRG -n myFunc \
  --settings \
    DB_PASSWORD="@Microsoft.KeyVault(VaultName=myKV;SecretName=db-password)" \
    API_KEY="@Microsoft.KeyVault(VaultName=myKV;SecretName=api-key)"

# 2. Use Managed Identity (no stored credentials)
az functionapp identity assign -g myRG -n myFunc

MI_PRINCIPAL=$(az functionapp identity show -g myRG -n myFunc \
  --query principalId -o tsv)

az role assignment create --assignee $MI_PRINCIPAL \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKV

az role assignment create --assignee $MI_PRINCIPAL \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount

# 3. HTTP trigger auth levels
# Anonymous:  no auth (public webhooks, health checks)
# Function:   function-level key in URL (?code=xxx)
# Admin:      master key required (management operations)
# System:     internal system key
# User:       Entra ID OAuth token validation (recommended for APIs)

# For production APIs — use Entra ID auth
az functionapp auth update -g myRG -n myFunc \
  --enabled true \
  --action LoginWithAzureActiveDirectory \
  --aad-allowed-token-audiences "api://myfunc" \
  --aad-client-id <app-id> \
  --aad-client-secret <secret>

# 4. Network isolation (Premium/Dedicated plan)
az functionapp vnet-integration add -g myRG -n myFunc \
  --vnet myVNet --subnet funcSubnet

az network private-endpoint create -g myRG -n funcPE \
  --vnet-name myVNet --subnet privateSubnet \
  --private-connection-resource-id \
    /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Web/sites/myFunc \
  --group-id sites --connection-name funcConn

# Disable public access
az functionapp update -g myRG -n myFunc \
  --set publicNetworkAccess=Disabled

# 5. CORS (only allow your own domains)
az functionapp cors add -g myRG -n myFunc \
  --allowed-origins https://myapp.com https://portal.myapp.com

# 6. IP restriction
az functionapp config access-restriction add -g myRG -n myFunc \
  --rule-name AllowOnlyAPIM \
  --action Allow \
  --ip-address <apim-outbound-ip>/32 \
  --priority 100
```

---

### 🟢 Q60. What is the serverless cost model?
```bash
# Azure Functions Consumption:
# Free tier: 1M executions/month + 400,000 GB-s memory/month
# After free: $0.20 per 1M executions + $0.000016 per GB-s

# Cost calculation example:
# 10M requests/month
# Average 200ms duration
# 512MB memory
# GB-s = 10,000,000 × 0.2s × (512MB/1024) = 1,000,000 GB-s
# Cost = (10M - 1M) × $0.20/1M + (1,000,000 - 400,000) × $0.000016
#      = $1.80 + $9.60 = $11.40/month

# Azure Functions Premium:
# EP1: ~$0.131/hr = ~$95/month (1 pre-warmed instance)
# EP2: ~$0.262/hr = ~$190/month
# EP3: ~$0.524/hr = ~$380/month

# Container Apps (Consumption):
# Dedicated: vCPU $0.000024/s + Memory $0.000003/GB/s
# Serverless: $0.000015/vCPU/s + $0.0000015/GB/s

# Logic Apps Consumption:
# Action executions: $0.000025 each
# Connector calls (standard): $0.000025 each
# Enterprise connectors: $0.001 each
# 10,000 actions/month × $0.000025 = $0.25/month

# Event Grid:
# First 100,000 operations/month: free
# $0.60 per million operations after

# Service Bus:
# Standard: $0.10/million operations
# Premium: ~$677/month per messaging unit

# Event Hubs:
# Basic: $0.028/million events
# Standard: $0.028/million + throughput units
# Premium: $0.50/hour per PU

# Cost optimisation:
# 1. Use Consumption for sporadic workloads (<500K req/month)
# 2. Use Premium only when cold start is unacceptable
# 3. Batch event processing (process 100 events at once)
# 4. Cache results to reduce Function invocations
# 5. Use Azure Monitor to identify unused functions
az monitor app-insights query \
  --app myServerlessAI -g myRG \
  --analytics-query "requests | where timestamp > ago(30d) | summarize count() by name | where count_ < 10"
```

---

## COMPLETE Q&A INDEX

| Q# | Question | Level | Part |
|----|---------|-------|------|
| **AZURE DEVOPS** | | | |
| Q1 | What is Azure DevOps and its 5 services? | 🟢 | DevOps |
| Q2 | Azure Repos — Git, branch policies, PRs | 🟢 | DevOps |
| Q3 | Azure Boards — epics, stories, sprints, queries | 🟢 | DevOps |
| Q4 | Azure Pipelines — basic CI YAML pipeline | 🟢 | DevOps |
| Q5 | Multi-stage pipeline with approvals and canary | 🟡 | DevOps |
| Q6 | Microsoft-hosted vs self-hosted agents, AKS agents | 🟡 | DevOps |
| Q7 | Variables, variable groups, Key Vault integration, secrets | 🟡 | DevOps |
| Q8 | Pipeline templates — local, cross-repo, parameters | 🟡 | DevOps |
| Q9 | Service connections — ARM, Workload Identity, Docker, GitHub | 🟡 | DevOps |
| Q10 | Environments, gates, deployment checks | 🟡 | DevOps |
| Q11 | Azure Artifacts — feeds, PyPI/NuGet/npm, upstream sources | 🟡 | DevOps |
| Q12 | Pipeline triggers — CI, PR, scheduled, resource, manual | 🟡 | DevOps |
| Q13 | Pipeline decorators, extensions, security tools | 🟡 | DevOps |
| Q14 | Permissions, access levels, security groups | 🟡 | DevOps |
| Q15 | Caching, optimisation — deps, Docker layers, shallow clone | 🟡 | DevOps |
| Q16 | GitHub Actions with Azure — OIDC, AKS deploy | 🔴 | DevOps |
| Q17 | Infrastructure as Code — Terraform in Pipelines | 🟡 | DevOps |
| Q18 | Azure DevOps Wiki — project wiki, code wiki | 🟢 | DevOps |
| Q19 | Dashboards and work item queries | 🟢 | DevOps |
| Q20 | Azure Test Plans — manual, automated, exploratory | 🟡 | DevOps |
| Q21 | GitOps with Azure DevOps and Flux v2 | 🟡 | DevOps |
| Q22 | Security scanning in pipelines — MSDO, Trivy, SonarCloud | 🟡 | DevOps |
| Q23 | Notifications and integrations (Teams, Slack, PagerDuty) | 🟢 | DevOps |
| Q24 | Azure DevOps vs GitHub — comparison table | 🟢 | DevOps |
| Q25 | PAT tokens, authentication methods | 🟢 | DevOps |
| **AZURE MIGRATION** | | | |
| Q26 | Azure Migrate — discovery, assessment, migration hub | 🟢 | Migration |
| Q27 | 6Rs / 7Rs — Rehost, Replatform, Refactor, Repurchase, Retain, Retire, Relocate | 🟢 | Migration |
| Q28 | Agentless VMware VM migration — steps and cutover | 🟡 | Migration |
| Q29 | Azure Database Migration Service (DMS) — SQL, MySQL, PostgreSQL | 🟡 | Migration |
| Q30 | SQL Server → Azure SQL Managed Instance (MI Link) | 🟡 | Migration |
| Q31 | Web application migration → App Service | 🟡 | Migration |
| Q32 | Azure VMware Solution (AVS) — HCX, live migration | 🟡 | Migration |
| Q33 | Azure Stack HCI — on-prem AKS, Arc integration | 🟡 | Migration |
| Q34 | Azure Arc — servers, K8s, data services, GitOps | 🟡 | Migration |
| Q35 | Cloud Adoption Framework (CAF) — phases, landing zones | 🟡 | Migration |
| Q36 | Cost Management for migration — TCO, Pricing Calculator, budgets | 🟢 | Migration |
| Q37 | Azure Data Factory for data migration — SHIR, copy pipeline | 🟡 | Migration |
| Q38 | Azure Migrate assessment — comfort factor, right-sizing | 🟢 | Migration |
| Q39 | Minimal downtime database migration strategy | 🟡 | Migration |
| Q40 | Migration waves and prioritisation — dependency mapping | 🟡 | Migration |
| Q41 | Common migration challenges and solutions | 🟡 | Migration |
| Q42 | App containerisation for IIS/Tomcat apps | 🟡 | Migration |
| Q43 | Migration tools comparison table | 🟢 | Migration |
| Q44 | Well-Architected Framework (WAF) — 5 pillars | 🟡 | Migration |
| Q45 | Azure Policy for migration governance | 🟡 | Migration |
| **SERVERLESS** | | | |
| Q46 | What is serverless and which Azure services are serverless? | 🟢 | Serverless |
| Q47 | Azure Functions — all plans, configuration, slots | 🟢 | Serverless |
| Q48 | All Azure Functions triggers and bindings — Python examples | 🟡 | Serverless |
| Q49 | Azure Functions best practices — idempotent, stateless, managed identity | 🟡 | Serverless |
| Q50 | Azure Logic Apps — SKUs, patterns, debugging | 🟡 | Serverless |
| Q51 | Azure Event Grid — system topics, filters, DLQ, MQTT | 🟡 | Serverless |
| Q52 | Azure Service Bus — sessions, filters, Python AAD auth | 🟡 | Serverless |
| Q53 | Azure Event Hubs — partitions, capture, schema registry, Python | 🟡 | Serverless |
| Q54 | Azure API Management (Consumption) — serverless API facade | 🟡 | Serverless |
| Q55 | Azure Static Web Apps — SPA hosting, routing, auth | 🟡 | Serverless |
| Q56 | Serverless architecture patterns — Saga, CQRS, event sourcing, circuit breaker | 🔴 | Serverless |
| Q57 | Functions vs Container Apps vs Logic Apps comparison | 🟢 | Serverless |
| Q58 | Monitoring serverless — App Insights KQL queries | 🟡 | Serverless |
| Q59 | Serverless security — Key Vault refs, managed identity, auth, network | 🟢 | Serverless |
| Q60 | Serverless cost model — calculation, optimisation | 🟢 | Serverless |

---
*Total: 60 Q&A | Azure DevOps (25) + Migration (20) + Serverless (15) | June 2026*
*🟢 22 Basic | 🟡 35 Intermediate | 🔴 3 Advanced*

---

# GAP-FILL — AZURE DEVOPS (Q61–Q80)

---

### 🟡 Q61. What are parallel jobs in Azure Pipelines?
**Answer:** A parallel job is one pipeline run executing at a time on a hosted or self-hosted agent. More parallel jobs = more simultaneous pipeline runs.

```bash
# Free tier limits (as of 2026):
# Public projects:  10 Microsoft-hosted parallel jobs, unlimited minutes
# Private projects: 1 Microsoft-hosted parallel job, 1800 minutes/month
#                   1 free self-hosted parallel job, unlimited minutes

# Buy additional parallel jobs
# Microsoft-hosted: $40/month per parallel job
# Self-hosted:      $15/month per parallel job
# (Azure DevOps → Organization Settings → Billing)

# Check current usage
az devops invoke \
  --org https://dev.azure.com/myOrg \
  --area distributedtask --resource pools \
  --route-parameters poolId=10 \
  --query "value[].{Name:name,IsHosted:isHosted,Size:size}"

# Pipeline YAML: control concurrency
pool:
  vmImage: ubuntu-latest
  # Each job consumes 1 parallel job slot

# Limit concurrent runs of same pipeline
concurrency:
  group: production-deploy    # only 1 run at a time in this group
  cancelInProgress: false     # queue instead of cancel

# Limit per environment (Exclusive Lock check)
# Environment → Checks → Exclusive Lock
# Prevents simultaneous deployments to same environment
```

---

### 🟡 Q62. What are branch strategies in Azure DevOps?
```
# Three main branch strategies:

# 1. GITFLOW (traditional, release-based)
# Branches: main, develop, feature/*, release/*, hotfix/*
# Flow: feature → develop → release → main
# ✅ Structured releases, good for versioned products
# ❌ Complex, slow feedback, long-lived branches → merge conflicts

main ──────────────────────────────── v1.0 ─── hotfix/bug ─── v1.0.1
         ↑                              ↑
develop ─┼──── feature/A ─────────────┤
         └──── feature/B ─────────────┘
         
# 2. GITHUB FLOW (simplified, continuous delivery)
# Branches: main, feature/*
# Flow: feature/* → PR → main (auto-deploy)
# ✅ Simple, fast, good for web apps
# ❌ No staging release branch

main ── feature/payment ──PR──→ main ──deploy──→ production

# 3. TRUNK-BASED DEVELOPMENT (recommended for high-velocity teams)
# Branches: main (trunk) + very short-lived feature branches (< 1 day)
# Feature flags: incomplete features hidden behind flags
# ✅ No merge hell, continuous integration, fastest feedback
# ❌ Requires discipline, feature flags overhead

# Azure DevOps branch policies for trunk-based:
az repos policy merge-strategy create \
  --project myProject --repo-id <id> \
  --branch main --is-enabled true --blocking true \
  --allow-squash true --allow-no-fast-forward false \
  --allow-rebase false --allow-rebase-merge false

# Require at least 1 reviewer, reset on new commits
az repos policy approver-count create \
  --project myProject --repo-id <id> \
  --branch main --is-enabled true --blocking true \
  --minimum-approver-count 1 \
  --reset-on-source-push true    # re-review if author pushes changes

# Require comments resolved before merge
az repos policy comment-required create \
  --project myProject --repo-id <id> \
  --branch main --is-enabled true --blocking true
```

---

### 🟡 Q63. What are deployment groups in Azure DevOps?
```bash
# Deployment Groups: deploy to a set of VMs without K8s
# Good for: on-prem VMs, IaaS VMs, legacy .NET apps on IIS

# Create deployment group
az devops deployment-group create \
  --project myProject \
  --name WebServers \
  --description "Production IIS web servers"

# Get registration script (run on each target VM)
az devops deployment-group add-tags \
  --project myProject \
  --deployment-group-name WebServers \
  --tags web production

# Register target VM (Windows PowerShell script)
# Generated in Portal → Deployment Groups → Registration Script
# $env:VSTS_AGENT_INPUT_SERVERURL="https://dev.azure.com/myOrg"
# $env:VSTS_AGENT_INPUT_AUTH="PAT"
# $env:VSTS_AGENT_INPUT_TOKEN="<PAT>"
# $env:VSTS_AGENT_INPUT_DEPLOYMENTGROUP="WebServers"
# $env:VSTS_AGENT_INPUT_PROJECT="myProject"
# .\config.cmd --deploymentgroup --acceptTeeEula

# Use deployment group in pipeline
stages:
- stage: Deploy
  jobs:
  - deployment: DeployToIIS
    displayName: Deploy to Web Servers
    environment:
      name: production
      resourceType: VirtualMachine   # targets deployment group VMs
    strategy:
      rolling:
        maxParallel: 2               # deploy to 2 VMs at a time
        preDeploy:
          steps:
          - script: iisreset /stop
        deploy:
          steps:
          - task: IISWebAppDeploymentOnMachineGroup@0
            inputs:
              WebSiteName: 'Default Web Site'
              Package: $(Pipeline.Workspace)/drop/myapp.zip
              TakeAppOfflineFlag: true
        postDeploy:
          steps:
          - script: iisreset /start
          - script: |
              $response = Invoke-WebRequest http://localhost/health
              if ($response.StatusCode -ne 200) { exit 1 }
            displayName: Health check
        on:
          failure:
            steps:
            - script: iisreset /start
```

---

### 🟡 Q64. What are Azure DevOps Analytics and reporting?
```bash
# Azure DevOps Analytics: built-in analytics engine for DevOps metrics
# Provides: velocity, burndown, cumulative flow, lead time, cycle time

# Access: Boards → Analytics → Views OR Dashboards → Analytics widgets

# Key reports:
# 1. Velocity Chart: story points completed per sprint (predictability)
# 2. Burndown Chart: work remaining vs time in sprint/release
# 3. Cumulative Flow Diagram (CFD): bottleneck identification across stages
# 4. Lead Time: time from work item creation to closure
# 5. Cycle Time: time from work item "Active" to "Closed"
# 6. Sprint Burndown: daily remaining work in current sprint

# OData API (Analytics service) — Power BI integration
# Connect Power BI to:
# https://analytics.dev.azure.com/myOrg/myProject/_odata/v4.0-preview/

# Sample OData query for velocity data
# GET https://analytics.dev.azure.com/myOrg/myProject/_odata/v4.0-preview/
# WorkItemSnapshot?$filter=WorkItemType eq 'User Story'
#   and State eq 'Closed'
#   and ClosedDate ge 2026-01-01Z
# &$select=WorkItemId,Title,StoryPoints,ClosedDate,IterationPath
# &$orderby=ClosedDate desc

# Pipeline Analytics (in Pipelines → Analytics):
# Pass rate trends by stage
# Test pass rate over time
# Pipeline duration trends
# Agent utilisation

# Create analytics view
az devops invoke \
  --org https://dev.azure.com/myOrg \
  --area wit --resource views \
  --http-method POST \
  --area wit --api-version 7.1 \
  --in-file analytics-view.json

# Delivery Plans (multi-team roadmap view)
# Shows: all teams' iterations on one timeline
# Useful for: cross-team dependency tracking, release planning
# Access: Boards → Delivery Plans → New Plan
```

---

### 🟡 Q65. What are Azure Load Testing and pipeline integration?
```bash
# Azure Load Testing: fully managed JMeter-based load testing service

az load create \
  --resource-group myRG \
  --name myLoadTest \
  --location eastus

# Create and run a quick test (URL-based, no JMeter file needed)
az load test create \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --test-id myQuickTest \
  --display-name "Homepage Load Test" \
  --description "Test homepage under 500 concurrent users"

az load test-run create \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --test-id myQuickTest \
  --test-run-id run-001 \
  --display-name "Baseline Run"

# Upload JMeter script (for complex scenarios)
az load test file upload \
  --load-test-resource myLoadTest \
  --resource-group myRG \
  --test-id myJMeterTest \
  --path ./load-test.jmx \
  --file-type JMETER_SCRIPT

# In Azure Pipeline
- task: AzureLoadTesting@1
  inputs:
    azureSubscription: myAzureServiceConnection
    loadTestConfigFile: load-test-config.yaml
    loadTestResource: myLoadTest
    resourceGroup: myRG
    failCriteria: |
      avg(response_time_ms) > 2000
      percentage(error) > 1

# load-test-config.yaml
version: v0.1
testId: myTest
displayName: API Load Test
testPlan: load-test.jmx
engineInstances: 5              # number of JMeter engines (more = more load)
splitAllCSVs: true
failureCriteria:
- avg(response_time_ms) > 3000  # fail if avg response > 3s
- percentage(error) > 2         # fail if error rate > 2%
- p99(response_time_ms) > 5000  # fail if 99th percentile > 5s
autoStop:
  errorPercentage: 90           # stop test if error rate > 90%
  timeWindow: 60
```

---

### 🟡 Q66. What is the Azure DevOps REST API?
```bash
# Azure DevOps REST API: automate any ADO operation programmatically
# Base URL: https://dev.azure.com/{org}/{project}/_apis/{area}/{resource}?api-version=7.1

# Authentication: Basic Auth with PAT token
ENCODED_PAT=$(echo -n ":$ADO_PAT" | base64)
AUTH_HEADER="Authorization: Basic $ENCODED_PAT"

# List pipelines
curl -H "$AUTH_HEADER" \
  "https://dev.azure.com/myOrg/myProject/_apis/pipelines?api-version=7.1"

# Trigger a pipeline run
curl -X POST \
  -H "$AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -d '{
    "resources": {"repositories": {"self": {"refName": "refs/heads/main"}}},
    "variables": {"environment": {"value": "production"}},
    "stagesToSkip": []
  }' \
  "https://dev.azure.com/myOrg/myProject/_apis/pipelines/1/runs?api-version=7.1"

# Approve a pipeline run (programmatic approval)
curl -X PATCH \
  -H "$AUTH_HEADER" \
  -H "Content-Type: application/json" \
  -d '{"status": "approved", "comment": "Auto-approved by automation"}' \
  "https://dev.azure.com/myOrg/myProject/_apis/pipelines/approvals/{approvalId}?api-version=7.1-preview.1"

# Create work item
curl -X POST \
  -H "$AUTH_HEADER" \
  -H "Content-Type: application/json-patch+json" \
  -d '[
    {"op":"add","path":"/fields/System.Title","value":"Fix login bug"},
    {"op":"add","path":"/fields/System.IterationPath","value":"myProject\\Sprint 3"},
    {"op":"add","path":"/fields/Microsoft.VSTS.Common.Priority","value":1}
  ]' \
  "https://dev.azure.com/myOrg/myProject/_apis/wit/workitems/\$Bug?api-version=7.1"

# Get audit log
curl -H "$AUTH_HEADER" \
  "https://auditservice.dev.azure.com/myOrg/_apis/audit/auditlog?api-version=7.1-preview.1&startTime=$(date -u -d '-1 day' +%Y-%m-%dT%H:%M:%SZ)"

# Common automation scenarios:
# 1. Auto-create bug when pipeline fails
# 2. Lock sprint when capacity is exceeded
# 3. Notify on PR older than 2 days
# 4. Auto-resolve linked work items when PR is merged
# 5. Generate release notes from closed work items
# 6. Trigger pipeline from external system webhook
```

---

### 🟡 Q67. What is Managed DevOps Pools?
```bash
# Managed DevOps Pools (GA 2024): Microsoft manages the VMs, you get predictable scale
# Combines: flexibility of self-hosted + convenience of Microsoft-hosted
# Key advantage: VMs persist between jobs (faster than Microsoft-hosted spin-up)
#                OR fresh VMs per job (cleaner than persistent self-hosted)

az mdp pool create \
  --resource-group myRG \
  --pool-name myManagedPool \
  --location eastus \
  --maximum-concurrency 20 \
  --sku-profile '[{"name":"Standard_D4s_v5","source":"AzureComputeGallery"}]' \
  --agent-profile '{"kind":"Stateless"}' \    # Stateless=fresh VM per job | Stateful=persist
  --devcenter-project-resource-id /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.DevCenter/projects/myDevCenterProject \
  --organization-profile '{"kind":"AzureDevOps","organizations":[{"url":"https://dev.azure.com/myOrg","parallelism":10}]}'

# Use in pipeline
pool:
  name: myManagedPool

# Benefits vs Microsoft-hosted:
# ✅ Custom VM image (tools pre-installed)
# ✅ VNet integration (access private resources)
# ✅ Larger VMs (up to 96 vCPUs)
# ✅ Faster startup (VMs pre-allocated)
# ✅ No parallel job limits from Microsoft
# ✅ Pay only for compute time (billed per-minute via Azure)
```

---

### 🟡 Q68. What are Classic Release Pipelines vs YAML Pipelines?
| Feature | Classic Release | YAML Pipelines |
|---------|--------------|---------------|
| Definition | GUI-based | Code in repo |
| Version control | No (settings stored in ADO) | Yes (in Git) |
| Multi-stage | Limited | Native |
| Templates | No | Yes |
| PR validation | No | Yes |
| Audit trail | ADO only | Git history |
| Gates | GUI-configured | Environment checks |
| Recommended | ❌ Legacy | ✅ Current |

```yaml
# Migrate classic release → YAML
# Classic: Build → Release Pipeline (separate entities)
# YAML: Single pipeline with stages

# Classic equivalent in YAML:
stages:
- stage: Build
  jobs:
  - job: Build
    steps:
    - script: dotnet build && dotnet test
    - task: PublishBuildArtifacts@1
      inputs:
        pathToPublish: bin/Release/net8.0/publish
        artifactName: drop

- stage: Deploy_Dev
  dependsOn: Build
  jobs:
  - deployment: Dev
    environment: dev
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: drop
          - task: AzureWebApp@1
            inputs:
              azureSubscription: myConnection
              appName: myapp-dev
              package: $(Pipeline.Workspace)/drop

- stage: Deploy_Prod
  dependsOn: Deploy_Dev
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: Production
    environment: production    # approval gate here
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: drop
          - task: AzureWebApp@1
            inputs:
              azureSubscription: myConnection
              appName: myapp-prod
              deployToSlotOrASE: true
              slotName: staging
          - task: AzureAppServiceManage@0
            inputs:
              azureSubscription: myConnection
              Action: Swap Slots
              WebAppName: myapp-prod
              SourceSlot: staging
```

---

### 🟡 Q69. What are deployment rings pattern?
```yaml
# Deployment rings: progressive exposure — reduces blast radius of bad deploys
# Ring 0: internal (devs) → Ring 1: canary (1%) → Ring 2: early adopters (10%) → Ring 3: GA (100%)

# Implementation with Azure DevOps + Feature Flags

stages:
- stage: Ring0_Internal
  displayName: Ring 0 — Internal Users
  jobs:
  - deployment: InternalDeploy
    environment: ring0-internal
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              appName: myapp-ring0
          - script: |
              # Run smoke tests against ring0
              pytest tests/smoke --base-url=https://myapp-ring0.azurewebsites.net
            displayName: Smoke tests

- stage: Ring1_Canary
  displayName: Ring 1 — 1% Traffic
  dependsOn: Ring0_Internal
  condition: succeeded()
  jobs:
  - deployment: CanaryDeploy
    environment: ring1-canary    # approval gate: monitor metrics 24h
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs:
              appName: myapp-prod
              deployToSlotOrASE: true
              slotName: canary
          # Route 1% traffic to canary slot
          - task: AzureAppServiceManage@0
            inputs:
              Action: Start Slot
              WebAppName: myapp-prod
              SpecifySlotOrASE: true
              Slot: canary
          - script: |
              az webapp traffic-routing set -g myRG -n myapp-prod \
                --distribution canary=1

- stage: Ring2_EarlyAdopters
  displayName: Ring 2 — 10% Traffic
  dependsOn: Ring1_Canary
  jobs:
  - deployment: EarlyAdopterDeploy
    environment: ring2-early     # 24h soak with metrics monitoring
    strategy:
      runOnce:
        deploy:
          steps:
          - script: |
              az webapp traffic-routing set -g myRG -n myapp-prod \
                --distribution canary=10

- stage: Ring3_GA
  displayName: Ring 3 — GA (100%)
  dependsOn: Ring2_EarlyAdopters
  jobs:
  - deployment: GADeploy
    environment: ring3-ga
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureAppServiceManage@0
            inputs:
              Action: Swap Slots
              WebAppName: myapp-prod
              SourceSlot: canary
          - script: |
              az webapp traffic-routing set -g myRG -n myapp-prod \
                --distribution canary=0    # remove canary routing
```

---

### 🟡 Q70. What is package promotion in Azure Artifacts?
```bash
# Package views: promote packages through quality gates
# Views: @local (just published) → @prerelease → @release

# Create feeds for each environment
az artifacts universal publish \
  --org https://dev.azure.com/myOrg \
  --project myProject \
  --feed dev-feed \
  --name mypackage --version 1.2.3 \
  --path ./dist/

# Promote to prerelease view (after QA passes)
az artifacts universal publish \
  --org https://dev.azure.com/myOrg \
  --project myProject \
  --feed dev-feed@prerelease \    # @prerelease view
  --name mypackage --version 1.2.3 \
  --path ./dist/

# Promote to release view (after prod validation)
az artifacts universal publish \
  --org https://dev.azure.com/myOrg \
  --project myProject \
  --feed dev-feed@release \       # @release view = stable
  --name mypackage --version 1.2.3 \
  --path ./dist/

# Consumers pin to a view (not a specific version)
# @release always points to latest stable
# pip install --index-url "https://pkgs.dev.azure.com/myOrg/myProject/_packaging/dev-feed@release/pypi/simple/" mypackage

# Multi-feed promotion pipeline
stages:
- stage: Publish_Dev
  jobs:
  - job: Publish
    steps:
    - task: TwineAuthenticate@1
      inputs: { artifactFeed: myProject/dev-feed }
    - script: |
        python -m build
        twine upload -r dev-feed dist/*

- stage: Promote_Prerelease
  dependsOn: Publish_Dev
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: PromotePrerelease
    environment: prerelease-promotion
    strategy:
      runOnce:
        deploy:
          steps:
          - task: UniversalPackages@0
            inputs:
              command: promote
              vstsFeed: myProject/dev-feed
              packageName: mypackage
              packageVersion: $(Build.BuildId)
              viewName: prerelease
```


---

# GAP-FILL — MIGRATION (Q71–Q85)

---

### 🟡 Q71. How do you migrate Hyper-V VMs to Azure?
```bash
# Hyper-V migration: uses lightweight agent on Hyper-V hosts (not VMs)
# Requires: Hyper-V hosts Windows Server 2012 R2 or later

# Step 1: Deploy Azure Migrate Hyper-V appliance
# Download VHD from Azure Migrate portal
# Deploy on Hyper-V host (not a VM guest)
# Register with Azure Migrate project

# Step 2: Enable replication
# Replication agent: installed on Hyper-V hosts
# Uses HTTPS 443 outbound from hosts to Azure

# Step 3: Enable replication per VM
az rest --method POST \
  --url "https://management.azure.com/subscriptions/<sub>/resourceGroups/myMigrateRG/providers/Microsoft.RecoveryServices/vaults/myVault/replicationFabrics/myHyperVFabric/replicationProtectionContainers/myContainer/replicationProtectedItems?api-version=2021-08-01" \
  --body '{
    "properties": {
      "policyId": "/subscriptions/<sub>/resourceGroups/myMigrateRG/providers/Microsoft.RecoveryServices/vaults/myVault/replicationPolicies/hyperVPolicy",
      "providerSpecificDetails": {
        "instanceType": "HyperVReplicaAzure",
        "hvHostVmId": "<hyper-v-host-id>",
        "vmName": "myHyperVVM",
        "osType": "Windows"
      }
    }
  }'

# Hyper-V vs VMware migration differences:
# VMware:  agentless (vCenter API via appliance); no agents on guest VMs
# Hyper-V: agent on Hyper-V HOST; still no agents on guest VMs
# Both:    same test migration → cutover flow

# Key Hyper-V settings that may differ in Azure:
# Generation: Gen 1 → Azure Gen 1; Gen 2 → Azure Gen 2 (UEFI)
# Disk: VHDX → managed disk (auto-converted)
# Network: virtual switch → Azure VNet subnet
# Checkpoints: must remove all checkpoints before migration
```

---

### 🟡 Q72. How do you migrate physical servers to Azure?
```bash
# Physical server (bare-metal) migration: install agent on each server

# Step 1: Deploy Azure Migrate Replication Appliance
# Deploy a Windows VM in your network
# Install the replication appliance (combined mobility service + process server)
# Register with Azure Migrate project

# Step 2: Install Mobility Service agent on each physical server
# The appliance can push-install the agent (requires admin credentials)
# Or manually install on each server

# Linux (push-install via appliance UI)
# OR manual install:
sudo ./install_linux_agent.sh \
  -i <appliance-ip> \
  -P /tmp/passphrase.txt \
  -d /usr/local/ASRsetup

# Windows (push-install via appliance UI)
# OR manual:
# MicrosoftAzureSiteRecoveryUnifiedSetup.exe /Role MS /Silent

# Step 3: Enable replication, test, cutover (same as VMware)

# Physical server scenarios:
# Bare-metal Linux servers (RHEL, Ubuntu, CentOS, SLES)
# Bare-metal Windows servers
# AWS EC2 instances (treated as physical)
# GCP Compute Engine instances (treated as physical)
# Other cloud VMs (OCI, Alibaba, IBM Cloud)

# Post-migration: install Azure VM agent
# Linux:
sudo apt-get install walinuxagent    # or yum install WALinuxAgent
sudo systemctl enable walinuxagent

# Windows:
# WindowsAzureGuestAgent.exe (included in Azure marketplace images)
# Download and install from: https://aka.ms/hvguestagent
```

---

### 🟡 Q73. How do you migrate MongoDB to Azure Cosmos DB?
```bash
# MongoDB → Cosmos DB MongoDB API (drop-in compatible)
# No app code change needed — connection string change only

# Step 1: Assessment
# Check MongoDB version compatibility
# Cosmos DB supports MongoDB wire protocol 3.2, 3.6, 4.0, 4.2, 5.0, 6.0, 7.0

# Step 2: Export from source MongoDB
mongodump \
  --host 192.168.1.100:27017 \
  --username admin --password P@ss! \
  --db myDatabase \
  --out /tmp/mongo-backup/

# Step 3: Import to Cosmos DB MongoDB API
COSMOS_CONN=$(az cosmosdb show -g myRG -n myCosmosAcct \
  --query "connectionStrings[0].connectionString" -o tsv)

mongorestore \
  --uri "$COSMOS_CONN" \
  --db myDatabase \
  --dir /tmp/mongo-backup/myDatabase/ \
  --noIndexRestore \    # create indexes separately after import
  --numParallelCollections 4 \
  --batchSize 24

# Step 4: Create indexes
mongosh "$COSMOS_CONN" --eval "
  use myDatabase;
  db.orders.createIndex({customerId: 1, createdAt: -1});
  db.orders.createIndex({status: 1});
"

# Step 5: Data validation
# Count documents
mongosh "$COSMOS_CONN" --eval "db.orders.countDocuments({})"

# Step 6: Online migration with continuous sync (DMS for MongoDB)
az dms project create \
  --resource-group myMigrateRG \
  --service-name myDMS \
  --project-name MongoMigration \
  --location eastus \
  --source-platform MongoDb \
  --target-platform CosmosDb

az dms project task create \
  --resource-group myMigrateRG \
  --service-name myDMS \
  --project-name MongoMigration \
  --task-name mongoOnlineTask \
  --task-type OnlineMongoToCosmosDb \
  --source-connection-json '{
    "serverName": "192.168.1.100",
    "port": 27017,
    "userName": "admin",
    "password": "P@ss!",
    "connectionType": "ReplicaSet"
  }' \
  --target-connection-json '{
    "serverName": "myCosmosAcct.mongo.cosmos.azure.com",
    "port": 10255,
    "userName": "myCosmosAcct",
    "password": "<cosmos-key>",
    "connectionType": "ReplicaSet"
  }'
```

---

### 🟡 Q74. How do you migrate file servers to Azure Files?
```bash
# Storage Migration Service (SMS): migrate file shares from Windows / Linux / NAS
# Runs as Windows service, orchestrated from Windows Admin Center or CLI

# Step 1: Inventory source shares
# Storage Migration Service scans source and catalogues all shares/files

# Step 2: Transfer data to Azure Files
# Create target Azure Files share
az storage share-rm create -g myRG \
  --storage-account mystorageaccount \
  --name finance-share --quota 5120 --tier TransactionOptimized

# Use Robocopy for initial seeding (on-prem to Azure Files)
net use Z: \\mystorageaccount.file.core.windows.net\finance-share \
  /user:AZURE\mystorageaccount <storage-key>

robocopy \\fileserver\Finance Z:\ \
  /MIR /MT:32 /R:3 /W:10 /LOG:robocopy.log \
  /TEE /NP /FFT /COMPRESS

# AzCopy for faster transfer (parallel, resumable)
azcopy sync "\\fileserver\Finance" \
  "https://mystorageaccount.file.core.windows.net/finance-share/" \
  --recursive --delete-destination=true

# Azure File Sync for ongoing sync (then cut over)
# 1. Install Azure File Sync agent on file server
# 2. Create sync group → cloud endpoint (Azure Files) → server endpoint (local path)
# 3. Let sync complete
# 4. Test Azure Files from clients
# 5. Update DFS namespace to point to Azure Files
# 6. Remove server endpoint (keeps data in Azure Files)

# Update DFS namespace to point to Azure Files
# On DFS server:
# Set-DfsnFolderTarget -Path \\domain\DFS\Finance \
#   -TargetPath \\mystorageaccount.file.core.windows.net\finance-share \
#   -State Online
```

---

### 🟡 Q75. How do you migrate Active Directory to Azure (Entra ID)?
```bash
# Three scenarios for AD migration:

# SCENARIO 1: Hybrid identity (most common)
# Keep on-prem AD + sync to Entra ID via Microsoft Entra Connect
# Users: same identity cloud + on-prem
# SSO: Entra ID for cloud apps + on-prem apps via AD FS or PTA

# Install Entra Connect (on-prem Windows Server)
# Download: https://www.microsoft.com/download/details.aspx?id=47594
# Setup wizard: Express or Custom settings

az ad connect sync verify   # verify sync configuration

# Check sync status
az ad connect show \
  --query "synchronizationStatus.{LastSync:lastSyncDateTime,Status:syncStatus}"

# SCENARIO 2: Migrate to cloud-only (Entra ID only)
# Prerequisites: no apps requiring NTLM/Kerberos
# Step 1: Ensure all apps support OIDC/SAML/OAuth
# Step 2: Migrate all devices to Entra ID joined (not AD joined)
# Step 3: Replace on-prem apps with cloud equivalents
# Step 4: Decommission AD DS

# SCENARIO 3: Azure AD Domain Services (AADDS) — managed AD DS in cloud
# For apps requiring LDAP/Kerberos without managing DCs
az ad ds create \
  --resource-group myRG \
  --name contoso.onmicrosoft.com \
  --location eastus \
  --replica-sets '[{"location":"eastus","subnetId":"/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/AADDSSubnet"}]' \
  --sku Enterprise

# ADMT (Active Directory Migration Tool) — domain consolidation
# Migrate users/groups between on-prem AD domains
# Useful for: company mergers, domain consolidation, rename domains
# Download: https://www.microsoft.com/download/details.aspx?id=19188

# Post-migration identity checklist:
# ✅ All users can sign in with Entra ID
# ✅ MFA configured for all users
# ✅ Conditional Access policies applied
# ✅ PIM for privileged roles
# ✅ Entra ID Protection enabled
# ✅ Password hash sync or PTA configured
# ✅ SSPR (Self-Service Password Reset) enabled
```

---

### 🟡 Q76. What is SAP migration to Azure?
```bash
# SAP on Azure: certified infrastructure for SAP HANA, S/4HANA, NetWeaver

# Certified VM families for SAP HANA:
# M-series: M208ms_v2 (5.7TB RAM) — largest HANA scale-up
# E-series: E96bds_v5 — smaller HANA instances
# Azure Large Instances (BareMetal): up to 120TB RAM for massive HANA

# Create M-series VM for SAP HANA
az vm create -g mySAPRG -n myHANAVM \
  --image SLES-SAP-15-SP4 \    # SUSE Linux Enterprise for SAP
  --size Standard_M208ms_v2 \  # 208 vCPUs, 5.7 TB RAM
  --zone 1 \
  --os-disk-size-gb 256 \
  --os-disk-caching ReadWrite \
  --generate-ssh-keys

# Premium SSD v2 for HANA data volumes
az disk create -g mySAPRG -n hanadata \
  --size-gb 4096 \
  --sku PremiumV2_LRS --zone 1 \
  --disk-iops-read-write 60000 \    # high IOPS for HANA
  --disk-mbps-read-write 800

# SAP migration tools:
# SWPM (SAP Software Provisioning Manager): classical export/import
# SAP HANA System Replication (HSR): online migration with near-zero downtime
# Azure Site Recovery (ASR): for SAP application servers (not HANA DB)
# SAP LaMa (Landscape Management): automated migration + cloning

# SAP-specific networking:
# Proximity Placement Group: co-locate SAP app servers + HANA
az ppg create -g mySAPRG -n sapPPG --type Standard --location eastus

az vm create -g mySAPRG -n myAppServer \
  --ppg sapPPG --size Standard_E16s_v5 \
  --image RHEL-SAP-HA-8 --zone 1 --generate-ssh-keys

# Availability Set for SAP HA:
az vm availability-set create -g mySAPRG -n sapHANAAS \
  --platform-fault-domain-count 3 \
  --platform-update-domain-count 20 \
  --proximity-placement-group sapPPG

# SAP HANA HSR for HA (2-node scale-up)
# Primary site: registration in /etc/hosts + hdbnsutil -sr_register
# Secondary site: hdbnsutil -sr_register --remoteHost=<primary> --remoteInstance=00 --replicationMode=sync

# Pacemaker cluster for STONITH fencing (SUSE or RHEL)
# Azure Fence Agent: fence_azure_arm (uses Azure REST API to fence VM)

# Monitoring SAP on Azure:
# Azure Monitor for SAP Solutions (AMS):
az workloads monitor create -g mySAPRG -n mySAPMonitor \
  --location eastus \
  --app-location eastus \
  --managed-resource-group-name mySAPMonitor-mrg
```

---

### 🟡 Q77. What is Oracle migration to Azure?
```bash
# Oracle options on Azure:

# Option 1: Oracle on Azure VM (IaaS — lift-and-shift)
# Oracle Database 19c/21c on RHEL or Oracle Linux
az vm create -g myOracleRG -n myOracleDB \
  --image OracleDatabase19c \    # Oracle Marketplace image
  --size Standard_E64s_v5 \     # memory-optimised
  --os-disk-size-gb 512 \
  --generate-ssh-keys

# Option 2: OracleDB@Azure (native Oracle service in Azure)
# Dedicated Oracle Exadata hardware in Azure datacenters
# Managed by Oracle Cloud Infrastructure (OCI)
# Low-latency interconnect to Azure services

# Option 3: Migrate Oracle → Azure SQL MI or PostgreSQL
# SSMA (SQL Server Migration Assistant) for Oracle:
# https://www.microsoft.com/download/details.aspx?id=54257
# - Converts Oracle schema, stored procedures, triggers → T-SQL
# - Migrates data with type mapping
# Steps:
# 1. Connect SSMA to Oracle source
# 2. Convert schema (review conversion warnings)
# 3. Sync to Azure SQL MI target
# 4. Migrate data
# 5. Test with application

# AWS SCT (Schema Conversion Tool) — alternative for Oracle → PostgreSQL
# Ora2Pg — open source Oracle → PostgreSQL migration tool

# Common Oracle features needing attention:
# Sequences → IDENTITY columns or SEQUENCE in SQL
# Oracle-specific SQL → T-SQL equivalents (ROWNUM → ROW_NUMBER())
# PL/SQL → T-SQL stored procedures (significant rewrite)
# Oracle synonyms → SQL views
# CONNECT BY PRIOR (hierarchical) → CTEs in T-SQL
# Oracle spatial data → SQL Server Spatial or PostGIS

# Oracle Data Guard → Azure SQL MI failover groups (equivalent HA)
# Oracle RAC → Azure SQL MI Business Critical (equivalent availability)
```

---

### 🟡 Q78. What are application modernisation patterns?
```bash
# Strangler Fig Pattern: incrementally replace legacy app with new cloud-native
# Named after fig trees that grow around and eventually replace their host tree

# Implementation:
# 1. Deploy new microservice alongside monolith
# 2. Route % of traffic to new service via API GW / Front Door
# 3. Gradually increase traffic to new service
# 4. Decommission monolith component once traffic is 100% migrated

# Example with Azure API Management:
# Week 1:  old monolith = 100%, new service = 0%
# Week 2:  old monolith = 90%, new orders service = 10%
# Week 4:  old monolith = 50%, new orders service = 50%
# Week 8:  old monolith = 0%, new orders service = 100%
# (decomission orders from monolith)

az apim api create -g myRG --service-name myAPIM \
  --api-id orders --path /orders \
  --display-name "Orders API"

# Route /orders to new service
az apim api operation create -g myRG --service-name myAPIM \
  --api-id orders --operation-id get-order \
  --method GET --url-template "/orders/{orderId}" \
  --display-name "Get Order"

# ── Anti-Corruption Layer (ACL) pattern ───────────────────────────
# Translates between legacy system and new cloud system
# Prevents old domain model "leaking" into new cloud-native design

# Example: Legacy SOAP service → REST API facade
# Azure Functions as ACL:
@app.route(route="orders/{orderId}", methods=["GET"])
def get_order(req: func.HttpRequest) -> func.HttpResponse:
    order_id = req.route_params["orderId"]
    # Call legacy SOAP service
    soap_response = call_legacy_soap(order_id)
    # Transform to new domain model
    modern_order = transform_legacy_to_modern(soap_response)
    return func.HttpResponse(json.dumps(modern_order), mimetype="application/json")

# ── Backend for Frontend (BFF) pattern ───────────────────────────
# Separate API gateway per client type (mobile, web, 3rd party)
# Each BFF optimised for its client's needs

# ── CQRS during migration ─────────────────────────────────────────
# Write to legacy DB (keeping it as source of truth temporarily)
# Read from new Azure DB (populated by sync/CDC)
# Gradually flip writes to new DB
# Eventually decommission legacy DB

# ── Event-driven migration ────────────────────────────────────────
# Capture changes from legacy via CDC → Event Hubs → new cloud services
# Legacy continues operating while cloud services consume events
# New services build their own read models from events
```

---

### 🟡 Q79. What is post-migration validation checklist?
```bash
# ── Connectivity validation ───────────────────────────────────────
# Test all inbound connections
curl -f https://myapp.azure.com/health && echo "PASS" || echo "FAIL"

# Test all internal service connections
az network watcher test-connectivity -g myRG \
  --source-resource myAppVM \
  --dest-address mydb.postgres.database.azure.com --dest-port 5432

# Test DNS resolution
nslookup mydb.postgres.database.azure.com
nslookup myredis.redis.cache.windows.net

# ── Application validation ────────────────────────────────────────
# Run automated test suite against Azure environment
pytest tests/integration/ \
  --base-url=https://myapp.azure.com \
  -v --junit-xml=post-migration-results.xml

# ── Data validation ───────────────────────────────────────────────
# Row count comparison
python3 << 'PYTHON'
import pyodbc, psycopg2

# Source (on-prem SQL Server)
src_conn = pyodbc.connect("Server=192.168.1.100;Database=myDB;...")
src_cursor = src_conn.cursor()
src_cursor.execute("SELECT COUNT(*) FROM dbo.Orders")
src_count = src_cursor.fetchone()[0]

# Target (Azure SQL DB)
tgt_conn = pyodbc.connect("Server=mysqlserver.database.windows.net;Database=myDB;...")
tgt_cursor = tgt_conn.cursor()
tgt_cursor.execute("SELECT COUNT(*) FROM dbo.Orders")
tgt_count = tgt_cursor.fetchone()[0]

assert src_count == tgt_count, f"Row count mismatch: {src_count} vs {tgt_count}"
print(f"Data validated: {tgt_count} rows")
PYTHON

# ── Performance validation ────────────────────────────────────────
# Baseline Azure performance vs on-prem
az monitor metrics list \
  --resource /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Sql/servers/mysqlserver/databases/myDB \
  --metric "dtu_consumption_percent" --aggregation Average \
  --start-time $(date -u -d '-1 hour' +%Y-%m-%dT%H:%M:%SZ) \
  --output table

# ── Security validation ───────────────────────────────────────────
# Check Defender for Cloud recommendations
az security task list \
  --query "[?state=='Active'].{Rec:recommendationType,Resource:resourceName}" \
  --output table

# Check NSG has no open RDP/SSH to internet
az network watcher test-ip-flow -g myRG --vm myVM \
  --direction Inbound --protocol TCP \
  --local <vm-ip>:3389 --remote 1.2.3.4:12345

# ── Cost validation ───────────────────────────────────────────────
# Check actual cost vs estimate
az consumption usage list \
  --billing-period-name "202606" \
  --query "[].{Resource:instanceName,Cost:pretaxCost}" \
  --output table | sort -k2 -rn | head -20

# ── Backup validation ─────────────────────────────────────────────
# Verify backups are running
az backup job list -g myRG --vault-name myRSV \
  --query "[?properties.status=='Completed'].{VM:properties.entityFriendlyName,Time:properties.startTime}" \
  --output table
```

---

### 🟢 Q80. What is disaster recovery planning for migrated workloads?
```bash
# RTO = Recovery Time Objective: max time to restore
# RPO = Recovery Point Objective: max data loss acceptable

# DR tiers:
# Tier 1 (Mission critical): RTO < 1h, RPO < 15min
# Tier 2 (Business critical): RTO < 4h, RPO < 1h
# Tier 3 (Important):        RTO < 24h, RPO < 4h
# Tier 4 (Standard):         RTO < 72h, RPO < 24h

# DR strategies by tier:

# Tier 1: Active-Active multi-region
# Front Door → two App Service deployments (eastus + westeurope)
# SQL DB failover group (auto-failover)
# Cosmos DB multi-region writes
az cosmosdb create -g myRG -n myCosmosAcct \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
  --locations regionName=westeurope failoverPriority=1 isZoneRedundant=true \
  --enable-automatic-failover true --enable-multiple-write-locations true

# Tier 2: Active-Passive warm standby
# ASR replication to secondary region
# SQL DB geo-replication (manual failover)
# Storage RA-GRS (read-only secondary)
az sql db create -g myRG -s mysqlserver -n myDB \
  --service-objective GP_Gen5_4 --zone-redundant true

az sql failover-group create -g myRG -s mysqlserver \
  --name myFOG \
  --partner-server mysqlserver-dr \
  --partner-resource-group myDRRG \
  --failover-policy Automatic --grace-period 1 \
  --add-db myDB

# Tier 3: Azure Backup + ASR
az backup protection enable-for-vm -g myRG \
  --vault-name myRSV --vm myVM --policy-name DailyPolicy

# DR testing (important — untested DR is no DR):
az site-recovery replication-protected-item test-failover \
  -g myDRRG --vault-name myASRVault \
  --fabric-name myFabric \
  --protection-container-name myContainer \
  --protected-item myVM \
  --failover-direction PrimaryToRecovery

# DR runbook automation (Logic Apps):
# Trigger: Azure Monitor alert OR manual HTTP call
# Steps:
# 1. Notify on-call team
# 2. Run pre-DR checklist
# 3. Trigger ASR failover
# 4. Update DNS to DR region
# 5. Run smoke tests
# 6. Send status notification
```


---

# GAP-FILL — SERVERLESS (Q81–Q100)

---

### 🔴 Q81. What are all 5 Durable Functions patterns in detail?
```python
import azure.durable_functions as df
from datetime import timedelta

# ── PATTERN 1: Function Chaining (sequential pipeline) ────────────
def chaining_orchestrator(context: df.DurableOrchestrationContext):
    """Execute activities in sequence; each step uses previous result."""
    order_id = context.get_input()
    validated  = yield context.call_activity("ValidateOrder",  order_id)
    enriched   = yield context.call_activity("EnrichOrder",    validated)
    charged    = yield context.call_activity("ChargePayment",  enriched)
    confirmed  = yield context.call_activity("ConfirmOrder",   charged)
    return confirmed

# ── PATTERN 2: Fan-out / Fan-in (parallel scatter-gather) ─────────
def fanout_orchestrator(context: df.DurableOrchestrationContext):
    """Launch many tasks in parallel; wait for all to complete."""
    product_ids = yield context.call_activity("GetAllProductIds", None)

    # Fan-out: start all in parallel
    tasks = [
        context.call_activity("CheckInventory", pid)
        for pid in product_ids
    ]
    # Fan-in: wait for ALL tasks
    results = yield context.task_all(tasks)

    # OR wait for ANY (first one to finish)
    # first_result = yield context.task_any(tasks)

    low_stock = [r for r in results if r["quantity"] < 10]
    return low_stock

# ── PATTERN 3: Async Human Interaction (approval workflow) ────────
def approval_orchestrator(context: df.DurableOrchestrationContext):
    """Wait for external human input (approval/rejection)."""
    expense = context.get_input()

    # Send approval email with callback URLs
    yield context.call_activity("SendApprovalEmail", {
        "approver": "manager@company.com",
        "instanceId": context.instance_id,
        "expense": expense
    })

    try:
        # Wait up to 72 hours for approval event
        approved = yield context.wait_for_external_event(
            "ApprovalDecision",
            timeout=timedelta(hours=72)
        )
        if approved:
            yield context.call_activity("ProcessExpense", expense)
            yield context.call_activity("NotifyEmployee", {"status": "approved"})
        else:
            yield context.call_activity("NotifyEmployee", {"status": "rejected"})

    except df.TimeoutError:
        # No response in 72 hours — escalate
        yield context.call_activity("EscalateToDirector", expense)

# Raise the event from HTTP endpoint (manager clicks "Approve"):
# POST /runtime/webhooks/durabletask/instances/{instanceId}/raiseEvent/ApprovalDecision
# Body: true  (or false for reject)

# ── PATTERN 4: Monitor (polling loop) ─────────────────────────────
def monitor_orchestrator(context: df.DurableOrchestrationContext):
    """Poll an external process until it completes."""
    job_id = context.get_input()
    deadline = context.current_utc_datetime + timedelta(hours=2)
    poll_interval = 30   # seconds

    while context.current_utc_datetime < deadline:
        status = yield context.call_activity("CheckJobStatus", job_id)

        if status["state"] == "Succeeded":
            result = yield context.call_activity("GetJobResult", job_id)
            return result

        if status["state"] == "Failed":
            raise Exception(f"Job {job_id} failed: {status['error']}")

        # Wait before next poll
        next_check = context.current_utc_datetime + timedelta(seconds=poll_interval)
        yield context.create_timer(next_check)
        poll_interval = min(poll_interval * 2, 300)   # exponential backoff, max 5 min

    raise TimeoutError(f"Job {job_id} did not complete within 2 hours")

# ── PATTERN 5: Eternal Orchestration (stateful singleton) ─────────
def eternal_orchestrator(context: df.DurableOrchestrationContext):
    """Run forever; periodically does work and reschedules itself."""
    config = context.get_input() or {"lastProcessedId": 0}

    # Do work
    new_config = yield context.call_activity("ProcessNewOrders", config)

    # Schedule next run in 60 seconds
    next_run = context.current_utc_datetime + timedelta(seconds=60)
    yield context.create_timer(next_run)

    # Restart with updated state (history is cleared on continue_as_new)
    context.continue_as_new(new_config)

# ── ENTITY FUNCTIONS (stateful counters, accumulators) ────────────
@df.entity_trigger(context_name="context")
def counter_entity(context: df.DurableEntityContext):
    """Stateful entity — survives restarts."""
    current = context.get_state(lambda: 0)
    operation = context.operation_name

    if operation == "add":
        amount = context.get_input()
        context.set_state(current + amount)
    elif operation == "reset":
        context.set_state(0)
    elif operation == "get":
        context.set_result(current)

# Signal entity from orchestrator
def orchestrator_using_entity(context: df.DurableOrchestrationContext):
    entity_id = df.EntityId("counter_entity", "my-counter")
    context.signal_entity(entity_id, "add", 5)   # fire-and-forget
    count = yield context.call_entity(entity_id, "get")  # read with result
    return count
```

---

### 🟡 Q82. What are Logic Apps error handling patterns?
```json
// Logic Apps: error handling in workflow definition

// Run-after conditions (equivalent to try-catch-finally)
// Default: each action runs only after previous succeeds
// Override with runAfter conditions:

{
  "actions": {
    "Try_Block": {
      "type": "Http",
      "inputs": {
        "method": "POST",
        "uri": "https://api.example.com/process"
      }
    },
    "Catch_Block": {
      "type": "Http",
      "inputs": {
        "method": "POST",
        "uri": "https://alerts.example.com/notify",
        "body": "@{result('Try_Block')}"
      },
      "runAfter": {
        "Try_Block": ["Failed", "TimedOut"]    // run ONLY if Try_Block fails
      }
    },
    "Finally_Block": {
      "type": "Compose",
      "inputs": "Cleanup",
      "runAfter": {
        "Try_Block": ["Succeeded", "Failed", "TimedOut", "Skipped"]  // always run
      }
    }
  }
}
```

```bash
# Retry policies (per-action configuration)
# Fixed interval: retry every 60 seconds, up to 4 times
# Exponential interval: backoff 5s → 10s → 20s → 40s
# None: no retry

# In Logic App Standard — JSON workflow definition
{
  "type": "Http",
  "inputs": {
    "method": "POST",
    "uri": "https://api.example.com/process"
  },
  "retryPolicy": {
    "type": "exponential",
    "count": 4,
    "interval": "PT5S",      // ISO 8601: 5 seconds
    "minimumInterval": "PT5S",
    "maximumInterval": "PT2M" // max 2 minutes between retries
  }
}

# Scopes (group actions together for try-catch)
# Scope "Try" → Scope "Catch" (runs if Try failed)
# Configure runAfter on Catch scope: Try → ["Failed","TimedOut"]

# Dead-letter for Logic Apps:
# On failure → send to Service Bus dead-letter queue
# Or: on failure → write to blob with error details
# Or: on failure → create work item in Azure DevOps

# Monitor failed runs
az rest --method GET \
  --url "https://management.azure.com/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Logic/workflows/myLogicApp/runs?api-version=2016-06-01&\$filter=status eq 'Failed'&\$top=20"
```

---

### 🟡 Q83. What is Service Bus geo-disaster recovery?
```bash
# Service Bus Geo-DR: replicate namespace metadata to secondary region
# NOTE: Messages are NOT replicated — only namespace config (queues, topics, subscriptions)
# Use alias endpoint → always points to active namespace

# Create primary and secondary namespaces
az servicebus namespace create -g myRG -n mySBPrimary \
  --location eastus --sku Premium --capacity 4 --zone-redundant true

az servicebus namespace create -g myDRRG -n mySBSecondary \
  --location westus2 --sku Premium --capacity 4

# Create geo-DR pairing
az servicebus georecovery-alias create \
  --resource-group myRG \
  --namespace-name mySBPrimary \
  --alias mySBAlias \
  --partner-namespace /subscriptions/<sub>/resourceGroups/myDRRG/providers/Microsoft.ServiceBus/namespaces/mySBSecondary

# Check replication state (wait for Succeeded)
az servicebus georecovery-alias show \
  --resource-group myRG \
  --namespace-name mySBPrimary \
  --alias mySBAlias \
  --query "provisioningState"

# ALL applications MUST connect via alias endpoint (NOT direct namespace):
# mySBAlias.servicebus.windows.net  ← use this always

# Failover (when primary region fails)
az servicebus georecovery-alias fail-over \
  --resource-group myDRRG \            # secondary resource group
  --namespace-name mySBSecondary \
  --alias mySBAlias

# After failover:
# - Alias now points to mySBSecondary
# - mySBPrimary is no longer paired
# - Apps reconnect transparently via alias
# - In-flight messages on primary ARE LOST (metadata replication only)

# Service Bus Premium — new message replication (GA 2024)
# Replicates BOTH metadata AND messages to secondary
az servicebus namespace create -g myRG -n mySBPremiumGeo \
  --location eastus --sku Premium \
  --geo-data-replication-locations '[
    {"locationName":"eastus","roleType":"Primary"},
    {"locationName":"westus2","roleType":"Secondary"}
  ]'
```

---

### 🟡 Q84. What are API Management policies in detail?
```xml
<!-- Complete APIM policy covering all major sections -->
<policies>
  <!-- INBOUND: runs before forwarding to backend -->
  <inbound>
    <base />   <!-- inherit parent policies -->

    <!-- Rate limiting per subscription key -->
    <rate-limit calls="200" renewal-period="60" />
    <rate-limit-by-key calls="50" renewal-period="60"
      counter-key="@(context.Request.IpAddress)"
      increment-condition="@(context.Response.StatusCode == 200)" />

    <!-- Quota per subscription (daily limit) -->
    <quota calls="100000" renewal-period="86400" />

    <!-- JWT validation (Entra ID) -->
    <validate-jwt header-name="Authorization"
                  failed-validation-httpcode="401"
                  failed-validation-error-message="Unauthorized. Valid token required.">
      <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration"/>
      <audiences>
        <audience>api://myapi</audience>
      </audiences>
      <issuers>
        <issuer>https://sts.windows.net/{tenant}/</issuer>
      </issuers>
      <required-claims>
        <claim name="roles" match="any">
          <value>Api.Read</value>
          <value>Api.Write</value>
        </claim>
      </required-claims>
    </validate-jwt>

    <!-- API key validation (alternative to JWT) -->
    <check-header name="X-API-Key" failed-check-httpcode="401"
                  failed-check-error-message="Invalid API key">
      <value>@(context.Variables.GetValueOrDefault<string>("validApiKey"))</value>
    </check-header>

    <!-- CORS -->
    <cors allow-credentials="true">
      <allowed-origins>
        <origin>https://myapp.com</origin>
        <origin>https://portal.myapp.com</origin>
      </allowed-origins>
      <allowed-methods preflight-result-max-age="300">
        <method>GET</method><method>POST</method>
        <method>PUT</method><method>DELETE</method>
      </allowed-methods>
      <allowed-headers>
        <header>Content-Type</header>
        <header>Authorization</header>
        <header>X-Correlation-ID</header>
      </allowed-headers>
    </cors>

    <!-- IP filtering -->
    <ip-filter action="allow">
      <address-range from="10.0.0.0" to="10.255.255.255" />
      <address>203.0.113.5</address>
    </ip-filter>

    <!-- Response caching -->
    <cache-lookup vary-by-developer="false"
                  vary-by-developer-groups="false"
                  downstream-caching-type="none">
      <vary-by-query-parameter>page</vary-by-query-parameter>
      <vary-by-header>Accept-Language</vary-by-header>
    </cache-lookup>

    <!-- Add correlation ID -->
    <set-header name="X-Correlation-ID" exists-action="skip">
      <value>@(Guid.NewGuid().ToString())</value>
    </set-header>

    <!-- Transform request (add/modify headers, query params) -->
    <set-header name="X-Forwarded-User" exists-action="override">
      <value>@(context.User?.Id ?? "anonymous")</value>
    </set-header>

    <!-- Named value (secret stored in APIM) -->
    <set-header name="X-Backend-Key" exists-action="override">
      <value>{{backend-api-key}}</value>  <!-- Named Value -->
    </set-header>

    <!-- Mock response (for testing) -->
    <!-- <mock-response status-code="200" content-type="application/json" /> -->
  </inbound>

  <!-- BACKEND: controls how request is forwarded -->
  <backend>
    <!-- Retry on backend failures -->
    <retry condition="@(context.Response.StatusCode >= 500)"
           count="3" interval="2" max-interval="10" delta="2" first-fast-retry="false">
      <forward-request />
    </retry>

    <!-- Load balance across backends -->
    <!-- <forward-request /> -->
  </backend>

  <!-- OUTBOUND: runs after getting backend response -->
  <outbound>
    <base />
    <!-- Store in cache -->
    <cache-store duration="300" />

    <!-- Remove sensitive headers from response -->
    <set-header name="X-Powered-By" exists-action="delete" />
    <set-header name="X-AspNet-Version" exists-action="delete" />
    <set-header name="Server" exists-action="delete" />

    <!-- Add security headers -->
    <set-header name="Strict-Transport-Security" exists-action="override">
      <value>max-age=31536000; includeSubDomains</value>
    </set-header>
    <set-header name="X-Content-Type-Options" exists-action="override">
      <value>nosniff</value>
    </set-header>

    <!-- Transform response body -->
    <json-to-xml apply="always" consider-accept-header="false" />
    <!-- OR: find-and-replace, set-body, xsl-transform -->

    <!-- Pagination link header -->
    <set-header name="X-Total-Count" exists-action="override">
      <value>@(context.Variables.GetValueOrDefault<string>("totalCount","0"))</value>
    </set-header>
  </outbound>

  <!-- ON-ERROR: handles any errors in the pipeline -->
  <on-error>
    <base />
    <return-response>
      <set-status code="@(context.Response.StatusCode)" />
      <set-header name="Content-Type"><value>application/json</value></set-header>
      <set-body>@{
        return new JObject(
          new JProperty("correlationId", context.Variables["X-Correlation-ID"]),
          new JProperty("error", context.LastError.Reason),
          new JProperty("message", context.LastError.Message),
          new JProperty("timestamp", DateTime.UtcNow.ToString("o"))
        ).ToString();
      }</set-body>
    </return-response>
  </on-error>
</policies>
```

---

### 🟡 Q85. What is Azure SignalR Service?
```bash
# SignalR Service: managed real-time WebSocket messaging
# Supports: WebSocket, Server-Sent Events (SSE), Long Polling

az signalr create -g myRG -n mySignalR \
  --location eastus \
  --sku Premium_P1 \       # Free_F1 | Standard_S1 | Premium_P1
  --unit-count 10 \        # 1 unit = 1,000 concurrent connections
  --service-mode Serverless \  # Default | Serverless | Classic
  --allowed-origins "https://myapp.com" "https://portal.myapp.com"

# Get connection string
az signalr key list -g myRG -n mySignalR \
  --query primaryConnectionString -o tsv
```

```csharp
// Azure Functions: SignalR serverless hub

[FunctionName("negotiate")]
public static SignalRConnectionInfo Negotiate(
    [HttpTrigger(AuthorizationLevel.Anonymous)] HttpRequest req,
    [SignalRConnectionInfo(HubName = "notifications",
                          UserId = "{headers.x-ms-signalr-user-id}")]
    SignalRConnectionInfo connectionInfo)
{
    return connectionInfo;  // returns URL + token to JS client
}

[FunctionName("broadcast")]
public static Task BroadcastToAll(
    [TimerTrigger("*/10 * * * * *")] TimerInfo timer,
    [SignalR(HubName = "notifications")] IAsyncCollector<SignalRMessage> signalRMessages)
{
    return signalRMessages.AddAsync(new SignalRMessage
    {
        Target = "priceUpdate",
        Arguments = new[] { new { symbol = "MSFT", price = 420.69 } }
    });
}

[FunctionName("sendToUser")]
public static Task SendToUser(
    [ServiceBusTrigger("user-notifications")] UserNotification notification,
    [SignalR(HubName = "notifications")] IAsyncCollector<SignalRMessage> signalRMessages)
{
    return signalRMessages.AddAsync(new SignalRMessage
    {
        UserId = notification.UserId,   // send to specific user only
        Target = "notification",
        Arguments = new[] { notification }
    });
}

[FunctionName("sendToGroup")]
public static Task SendToGroup(
    [HttpTrigger(AuthorizationLevel.Function)] GroupMessage msg,
    [SignalR(HubName = "notifications")] IAsyncCollector<SignalRMessage> signalRMessages)
{
    return signalRMessages.AddAsync(new SignalRMessage
    {
        GroupName = msg.Group,          // send to a group
        Target = "groupMessage",
        Arguments = new[] { msg.Text }
    });
}
```

```javascript
// JavaScript client
const { HubConnectionBuilder } = require("@microsoft/signalr");

const connection = new HubConnectionBuilder()
    .withUrl("/api", {
        accessTokenFactory: () => getAccessToken()
    })
    .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
    .configureLogging("information")
    .build();

connection.on("priceUpdate", (data) => {
    console.log(`${data.symbol}: $${data.price}`);
    updateUI(data);
});

connection.on("notification", (msg) => {
    showNotification(msg);
});

await connection.start();
console.log("Connected to SignalR");
```

---

### 🟡 Q86. What is Azure Notification Hubs?
```bash
# Notification Hubs: multi-platform push notifications at scale
# Single API → route to APNs (iOS), FCM (Android), WNS (Windows), MPNS, Baidu, ADM

az notification-hub namespace create -g myRG \
  -n myNHNamespace --location eastus --sku Standard

az notification-hub create -g myRG \
  --namespace-name myNHNamespace \
  -n myNotificationHub \
  --location eastus

# Configure FCM (Android/Firebase)
az notification-hub credential gcm update -g myRG \
  --namespace-name myNHNamespace \
  -n myNotificationHub \
  --google-api-key "<FCM_SERVER_KEY>"

# Configure APNs (iOS)
az notification-hub credential apns update -g myRG \
  --namespace-name myNHNamespace \
  -n myNotificationHub \
  --apns-credential '{
    "apnsCertificate": "<base64-cert>",
    "certificateKey": "<cert-password>",
    "endpoint": "https://gateway.push.apple.com"
  }'

# Send test notification
az notification-hub test-send -g myRG \
  --namespace-name myNHNamespace \
  -n myNotificationHub \
  --notification-format gcm \
  --message '{"notification":{"title":"Hello","body":"Push notification test"}}'

# Register device (from mobile app SDK)
# iOS: register with APNs → get device token → register with NH
# Android: register with FCM → get registration token → register with NH

# Tag-based routing (target specific users/segments)
# Register with tags: userId:user123, region:eastus, plan:premium
# Send to: (userId:user123)                  → specific user
# Send to: (region:eastus && plan:premium)   → premium users in East US
# Broadcast: no tag expression               → all devices
```

---

### 🟡 Q87. What are Azure Functions on Kubernetes (KEDA)?
```bash
# KEDA (Kubernetes Event-Driven Autoscaling): run Functions on any K8s cluster
# Scales from 0 to N pods based on event source depth

# Install KEDA on AKS
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace

# Install Azure Functions KEDA HTTP add-on
helm install http-add-on kedacore/keda-add-ons-http --namespace keda

# Build Functions container
func init myFuncApp --worker-runtime python --docker
func new --name OrderProcessor --template "Service Bus Queue trigger"
docker build -t myacr.azurecr.io/myfunc:latest .
docker push myacr.azurecr.io/myfunc:latest
```

```yaml
# Kubernetes: Functions + KEDA ScaledObject
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
  namespace: functions
spec:
  replicas: 0              # starts at 0 — KEDA will scale up
  selector:
    matchLabels:
      app: order-processor
  template:
    metadata:
      labels:
        app: order-processor
    spec:
      containers:
      - name: order-processor
        image: myacr.azurecr.io/myfunc:latest
        env:
        - name: AzureWebJobsServiceBus__fullyQualifiedNamespace
          value: mySBNamespace.servicebus.windows.net
        - name: FUNCTIONS_WORKER_RUNTIME
          value: python
        resources:
          requests:
            memory: 256Mi
            cpu: 100m
          limits:
            memory: 512Mi
            cpu: 500m
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: functions
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 0         # scale to zero when idle
  maxReplicaCount: 50
  pollingInterval: 15        # check every 15 seconds
  cooldownPeriod: 300        # wait 5 min before scaling in
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      namespace: mySBNamespace
      messageCount: "5"      # 1 pod per 5 messages in queue
    authenticationRef:
      name: keda-sb-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-sb-auth
  namespace: functions
spec:
  podIdentity:
    provider: azure-workload  # use workload identity (no secrets)
```

---

### 🟡 Q88. What is Power Automate vs Logic Apps?
| Feature | Power Automate | Logic Apps |
|---------|--------------|-----------|
| **Target user** | Business users (no-code) | Developers + IT pros |
| **License** | Microsoft 365 license | Azure pay-per-use |
| **Connectors** | 1000+ (same as Logic Apps) | 1000+ (same) |
| **Custom connectors** | ✅ Yes | ✅ Yes |
| **Code** | Expressions only | Code via Azure Functions |
| **VNet** | ❌ No | ✅ Standard plan |
| **CI/CD** | Limited | Full (ARM/Bicep/YAML) |
| **Monitoring** | Power Platform Admin Center | Azure Monitor |
| **Approvals** | ✅ Built-in, excellent | Via Teams connector |
| **Best for** | Office 365 automation, HR, business process | Enterprise integration, B2B, complex flows |
| **ALM** | Solutions (export/import) | ARM templates, Bicep |

```bash
# Logic Apps and Power Automate share the same connector ecosystem
# You can call a Logic Apps HTTP trigger from Power Automate and vice versa

# Convert Power Automate flow → Logic Apps:
# Export flow from Power Automate → OpenAPI definition
# Import OpenAPI to Logic Apps OR rebuild using Logic Apps Studio

# When to use which:
# Power Automate: SharePoint workflows, Outlook rules, Teams approvals,
#                 HR processes (leave requests), business notifications
# Logic Apps:     B2B integration (EDI/AS2), enterprise service bus,
#                 complex orchestration, hybrid integration, APIM backend
```

---

### 🟡 Q89. What are Azure Functions cold start mitigation strategies?
```bash
# Cold start: time to start a new function instance from scratch
# Causes: Consumption plan spins down after 20 min of inactivity
# Typical cold start: 1-5 seconds (Python slower than C#)

# Strategy 1: Premium plan (pre-warmed instances — eliminates cold start)
az functionapp plan create -g myRG -n myPremiumPlan \
  --location eastus --sku EP1 --is-linux

az functionapp create -g myRG --plan myPremiumPlan \
  --runtime python --runtime-version 3.12 \
  --functions-version 4 -n myFastFunc \
  --storage-account mystorageaccount

# Configure always-warm instances
az functionapp config set -g myRG -n myFastFunc \
  --prewarmed-instance-count 2     # always 2 instances ready

# Strategy 2: Timer trigger keepalive (Consumption plan workaround)
@app.timer_trigger(schedule="*/19 * * * *",   # every 19 minutes
                   arg_name="timer",
                   run_on_startup=True)
def keepalive(timer: func.TimerRequest) -> None:
    pass   # just keeps the instance alive

# Strategy 3: Reduce package size
# - Use lazy imports (import inside function, not at module level)
# - Exclude dev dependencies from deployment
# - Use .funcignore file to exclude test files

# .funcignore
.git
.vscode
local.settings.json
__pycache__
tests/
*.pyc
*.pyo

# Strategy 4: Optimise imports (lazy loading)
# BAD (loaded at cold start):
import pandas as pd          # heavy library

def main(req):
    df = pd.DataFrame(...)

# GOOD (loaded on first call only):
def main(req):
    import pandas as pd      # lazy import — only on first call per instance
    df = pd.DataFrame(...)

# Strategy 5: Connection reuse (keep DB connections warm between invocations)
# Module-level connections persist between invocations on same instance
import psycopg2

# Initialise once per instance (not per invocation)
_db_conn = None

def get_db():
    global _db_conn
    if _db_conn is None or _db_conn.closed:
        _db_conn = psycopg2.connect(os.environ["DATABASE_URL"])
    return _db_conn

def main(req: func.HttpRequest) -> func.HttpResponse:
    conn = get_db()   # reused between invocations
    # ...

# Strategy 6: Azure Container Apps (built-in warm containers)
# Scale-to-zero but with much faster startup than Consumption Functions
# Min replicas = 1 avoids cold start entirely
az containerapp create -g myRG -n myApp --environment myCAEnv \
  --min-replicas 1   # always 1 warm instance
```

---

### 🟢 Q90. What is Azure Web PubSub?
```bash
# Web PubSub: managed WebSocket + SSE pub/sub messaging
# Difference from SignalR: no hub model, pure pub/sub with groups and users
# Both use WebSocket; Web PubSub has simpler, more flexible model

az webpubsub create -g myRG -n myWebPubSub \
  --location eastus \
  --sku Standard_S1 \    # Free_F1 | Standard_S1 | Premium_P1
  --unit-count 10        # 1 unit = 1,000 concurrent connections

# Get connection string
az webpubsub key show -g myRG -n myWebPubSub \
  --query primaryConnectionString -o tsv

# Configure event handler (Azure Function receives WebSocket events)
az webpubsub hub update -g myRG -n myWebPubSub \
  --hub-name "chat" \
  --event-handler \
    url-template="https://myfunc.azurewebsites.net/api/handler?code=xxx" \
    system-event="connect,disconnected" \
    user-event-pattern="*"
```

```python
# Python: send messages to users/groups
from azure.messaging.webpubsubservice import WebPubSubServiceClient
from azure.identity import DefaultAzureCredential

client = WebPubSubServiceClient(
    endpoint="https://myWebPubSub.webpubsub.azure.com",
    hub="chat",
    credential=DefaultAzureCredential()
)

# Broadcast to all connected clients
client.send_to_all(
    message='{"type":"announcement","text":"System maintenance in 5 minutes"}',
    content_type="application/json"
)

# Send to specific user
client.send_to_user(
    user_id="user123",
    message='{"type":"message","text":"You have a new order"}',
    content_type="application/json"
)

# Send to group (room, channel)
client.send_to_group(
    group="room-101",
    message='{"type":"chat","text":"Hello room!"}',
    content_type="application/json"
)

# Add user to group
client.add_user_to_group(user_id="user123", group="room-101")

# Generate client access token (for WebSocket client)
token = client.get_client_access_token(
    user_id="user123",
    roles=["webpubsub.joinLeaveGroup.room-101",
           "webpubsub.sendToGroup.room-101"],
    minutes_to_expire=60
)
websocket_url = token["url"]  # client connects to this URL
```

---

## FINAL COMPLETE Q&A INDEX (All 130 Questions)

| Q# | Question | Level | Part |
|----|---------|-------|------|
| **AZURE DEVOPS (Q1–Q70)** | | | |
| Q1 | 5 services overview | 🟢 | DevOps |
| Q2 | Azure Repos — Git, branch policies, PRs | 🟢 | DevOps |
| Q3 | Azure Boards — work items, sprints, WIQL | 🟢 | DevOps |
| Q4 | Basic CI YAML pipeline | 🟢 | DevOps |
| Q5 | Multi-stage pipeline with canary | 🟡 | DevOps |
| Q6 | Agents — hosted vs self-hosted vs AKS | 🟡 | DevOps |
| Q7 | Variables, variable groups, secrets, KV | 🟡 | DevOps |
| Q8 | Pipeline templates — local + cross-repo | 🟡 | DevOps |
| Q9 | Service connections — ARM, WIF, Docker | 🟡 | DevOps |
| Q10 | Environments and gates | 🟡 | DevOps |
| Q11 | Azure Artifacts — feeds, upstream sources | 🟡 | DevOps |
| Q12 | All trigger types | 🟡 | DevOps |
| Q13 | Decorators and extensions | 🟡 | DevOps |
| Q14 | Permissions and access levels | 🟡 | DevOps |
| Q15 | Caching and performance optimisation | 🟡 | DevOps |
| Q16 | GitHub Actions with Azure OIDC | 🔴 | DevOps |
| Q17 | Terraform in Azure Pipelines | 🟡 | DevOps |
| Q18 | Azure DevOps Wiki | 🟢 | DevOps |
| Q19 | Dashboards and queries | 🟢 | DevOps |
| Q20 | Azure Test Plans | 🟡 | DevOps |
| Q21 | GitOps with Flux v2 | 🟡 | DevOps |
| Q22 | Security scanning — MSDO, Trivy, SonarCloud | 🟡 | DevOps |
| Q23 | Notifications — Teams, Slack, PagerDuty | 🟢 | DevOps |
| Q24 | DevOps vs GitHub comparison | 🟢 | DevOps |
| Q25 | PAT tokens and authentication | 🟢 | DevOps |
| Q61 | Parallel jobs — free limits, buying more | 🟡 | DevOps |
| Q62 | Branch strategies — GitFlow, GitHub Flow, Trunk | 🟡 | DevOps |
| Q63 | Deployment groups — deploy to IIS VMs | 🟡 | DevOps |
| Q64 | Analytics — velocity, burndown, CFD, OData | 🟡 | DevOps |
| Q65 | Azure Load Testing — JMeter, pipeline integration | 🟡 | DevOps |
| Q66 | Azure DevOps REST API — all scenarios | 🟡 | DevOps |
| Q67 | Managed DevOps Pools | 🟡 | DevOps |
| Q68 | Classic release pipelines vs YAML | 🟡 | DevOps |
| Q69 | Deployment rings pattern | 🔴 | DevOps |
| Q70 | Package promotion — views, multi-feed | 🟡 | DevOps |
| **MIGRATION (Q26–Q80)** | | | |
| Q26–Q45 | (original 20 questions) | 🟢🟡 | Migration |
| Q71 | Hyper-V VM migration | 🟡 | Migration |
| Q72 | Physical server migration | 🟡 | Migration |
| Q73 | MongoDB migration to Cosmos DB | 🟡 | Migration |
| Q74 | File server migration to Azure Files | 🟡 | Migration |
| Q75 | Active Directory migration to Entra ID | 🟡 | Migration |
| Q76 | SAP migration to Azure | 🟡 | Migration |
| Q77 | Oracle migration to Azure | 🟡 | Migration |
| Q78 | Application modernisation patterns — Strangler Fig, ACL, BFF | 🔴 | Migration |
| Q79 | Post-migration validation checklist | 🟡 | Migration |
| Q80 | DR planning for migrated workloads | 🟡 | Migration |
| **SERVERLESS (Q46–Q90)** | | | |
| Q46–Q60 | (original 15 questions) | 🟢🟡🔴 | Serverless |
| Q81 | Durable Functions — all 5 patterns + entities | 🔴 | Serverless |
| Q82 | Logic Apps error handling — try/catch/finally, retry | 🟡 | Serverless |
| Q83 | Service Bus geo-DR — alias, pairing, failover | 🟡 | Serverless |
| Q84 | API Management policies — complete reference | 🔴 | Serverless |
| Q85 | Azure SignalR Service — Functions bindings, JS client | 🟡 | Serverless |
| Q86 | Azure Notification Hubs — APNs/FCM, tag routing | 🟡 | Serverless |
| Q87 | Azure Functions on Kubernetes (KEDA ScaledObject) | 🔴 | Serverless |
| Q88 | Power Automate vs Logic Apps — comparison | 🟢 | Serverless |
| Q89 | Functions cold start mitigation strategies | 🟡 | Serverless |
| Q90 | Azure Web PubSub — pub/sub WebSockets | 🟡 | Serverless |

---
*Total: 130 Q&A | Azure DevOps (35) + Migration (30) + Serverless (25) | June 2026*
*🟢 25 Basic | 🟡 90 Intermediate | 🔴 15 Advanced*
