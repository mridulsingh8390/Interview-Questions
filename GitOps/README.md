# GitOps: ArgoCD & Flux — Complete Study Guide
### All Topics · Detailed Q&A · Master Cheatsheet
**2025 Edition — Comprehensive coverage with examples, commands & cheatsheet**

---

## Table of Contents
- [GitOps Principles](#gitops-principles)
- [GitOps vs Push-Based CD](#gitops-vs-push)
- [ArgoCD Architecture](#argocd-arch)
- [ArgoCD Applications](#argocd-apps)
- [AppProject & Multi-Tenancy](#appproject)
- [ApplicationSet (Fleet Management)](#applicationset)
- [ArgoCD RBAC](#argocd-rbac)
- [ArgoCD SSO / Dex](#argocd-sso)
- [ArgoCD Notifications](#argocd-notifications)
- [ArgoCD Image Updater](#argocd-image-updater)
- [Flux Architecture & CRDs](#flux)
- [Flux Multi-Tenancy](#flux-multitenancy)
- [Flux Notifications](#flux-notifications)
- [Secrets in GitOps](#secrets)
- [Master Cheatsheet](#master-cheatsheet)

---

## GitOps Principles

### 🟢 Q1. What is GitOps and what are its four principles?

```
GitOps (coined by Weaveworks, 2017):
Operate infrastructure and applications by using Git as the single source of truth.

Four Principles (OpenGitOps v1.0):

1. DECLARATIVE
   All desired state is expressed declaratively (YAML, HCL, etc.)
   No imperative scripts — describe WHAT, not HOW

2. VERSIONED & IMMUTABLE
   Desired state stored in Git with full history
   Previous states retrievable via git history

3. PULLED AUTOMATICALLY
   Software agents (ArgoCD, Flux) continuously pull from Git
   Agents detect drift and reconcile automatically

4. CONTINUOUSLY RECONCILED
   Agents monitor actual state vs desired state
   Automatically correct drift without human intervention
```

---

## GitOps vs Push-Based CD

### 🟡 Q2. How does GitOps differ from push-based CD?

```
PUSH-BASED (traditional CI/CD):
  Git → CI/CD tool (Jenkins/GitHub Actions) → kubectl apply → Cluster
  
  Pros:
    - Simple mental model
    - CI tool has full control
    - Works without cluster access from Git
  Cons:
    - CI tool needs cluster credentials (security risk)
    - No automatic drift detection
    - Cluster state can diverge from Git
    - Hard to reproduce state

PULL-BASED (GitOps):
  Git → [ArgoCD/Flux watches] → Cluster
  
  Pros:
    - No external access to cluster needed (agent runs inside)
    - Continuous reconciliation (drift auto-fixed)
    - Full auditability (every change is a git commit)
    - Easy rollback (git revert)
    - Cluster state ALWAYS matches Git
  Cons:
    - Agent must be running in cluster
    - Learning curve
    - Secret management complexity (secrets can't live in Git)

Common hybrid pattern:
  CI (build image, run tests, update image tag in Git) → GitOps agent deploys
```

---

## ArgoCD Architecture

### 🟢 Q3. What is ArgoCD and how does it work?

```
ArgoCD components:
  argocd-server          — API server + Web UI (port 443/8080)
  argocd-repo-server     — Clones repos, renders manifests (Helm, Kustomize, etc.)
  argocd-application-controller — Reconciliation loop, monitors cluster vs Git
  argocd-dex-server      — OIDC SSO provider (optional)
  argocd-redis           — Cache for app state
  argocd-notifications   — Notifications (Slack, email, etc.)
  argocd-applicationset-controller — ApplicationSet controller

Reconciliation loop:
  1. Watch Git repo for changes (webhook or polling every 3 min)
  2. Render manifests (helm template, kustomize build, etc.)
  3. Compare with live Kubernetes resources
  4. If diff found → sync (apply manifests)
  5. Continuous: if someone does kubectl edit → auto-revert (selfHeal=true)
```

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=300s

# Get initial admin password
kubectl argocd admin initial-password -n argocd
# Or:
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# Port forward to UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login via CLI
argocd login localhost:8080 --username admin --password <password> --insecure

# Change password
argocd account update-password

# argocd CLI — add repo
argocd repo add https://github.com/org/my-app.git \
  --username bot \
  --password $GITHUB_TOKEN

# Add cluster (to manage remote cluster)
argocd cluster add eks-prod --name production
```

---

## ArgoCD Applications

### 🟢 Q4. How do you create and manage ArgoCD Applications?

```yaml
# Application CRD — the core GitOps unit
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io    # Delete app resources when app is deleted
spec:
  project: production                            # AppProject (default = default)

  source:
    repoURL: https://github.com/myorg/my-app.git
    targetRevision: main                         # Branch, tag, or commit SHA
    path: k8s/production                         # Path within repo

    # For Helm charts:
    helm:
      valueFiles:
        - values-prod.yaml
      values: |
        replicaCount: 5
        image:
          tag: v1.2.3
      parameters:
        - name: image.tag
          value: "v1.2.3"

    # For Kustomize:
    kustomize:
      version: v5.0.0
      images:
        - myapp=registry.example.com/myapp:v1.2.3

  destination:
    server: https://kubernetes.default.svc       # In-cluster
    # server: https://prod-cluster.example.com   # Remote cluster
    namespace: production

  syncPolicy:
    automated:
      prune: true                               # Delete resources removed from Git
      selfHeal: true                            # Fix manual changes (drift correction)
      allowEmpty: false                         # Don't delete all resources if Git is empty
    syncOptions:
      - CreateNamespace=true                    # Auto-create destination namespace
      - PrunePropagationPolicy=foreground       # Wait for dependent resources to delete
      - ApplyOutOfSyncOnly=true                 # Only sync resources that differ
      - ServerSideApply=true                    # Use server-side apply (handles conflicts better)
    retry:
      limit: 5                                  # Retry failed sync 5 times
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas                        # Ignore HPA-managed replicas
    - group: ""
      kind: ConfigMap
      name: my-app-config
      jsonPointers:
        - /data/last-updated                    # Ignore auto-updated field
```

```bash
# ArgoCD CLI
argocd app create my-app \
  --repo https://github.com/org/my-app.git \
  --path k8s/production \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy automated \
  --auto-prune \
  --self-heal

argocd app list
argocd app get my-app
argocd app sync my-app                      # Manual sync
argocd app sync my-app --force              # Force sync (replaces, doesn't patch)
argocd app sync my-app --dry-run            # Preview
argocd app diff my-app                      # Show diff
argocd app rollback my-app 3                # Rollback to revision 3
argocd app history my-app
argocd app delete my-app                    # Delete app (and resources if finalizer set)
argocd app delete my-app --cascade          # Delete app + all Kubernetes resources
argocd app set my-app --sync-policy automated
argocd app wait my-app --sync --health      # Wait for healthy+synced
```

---

## AppProject & Multi-Tenancy

### 🟡 Q5. What is an AppProject and how do you use it for multi-tenancy?

```yaml
# AppProject restricts what teams can deploy where
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production workloads

  # Source repos allowed (restrict which repos can be deployed from)
  sourceRepos:
    - https://github.com/myorg/my-app.git
    - https://github.com/myorg/platform.git
    # - "*"    # Allow all (use with caution)

  # Destination clusters and namespaces
  destinations:
    - server: https://kubernetes.default.svc
      namespace: production
    - server: https://kubernetes.default.svc
      namespace: production-*             # Wildcards supported

  # Deny specific cluster-scoped resources
  clusterResourceBlacklist:
    - group: ""
      kind: Namespace                     # Can't create Namespaces

  # Only allow specific namespace-scoped resources
  namespaceResourceWhitelist:
    - group: "apps"
      kind: Deployment
    - group: "apps"
      kind: StatefulSet
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap
    - group: ""
      kind: Secret
    - group: "networking.k8s.io"
      kind: Ingress
    - group: "autoscaling"
      kind: HorizontalPodAutoscaler
    - group: "policy"
      kind: PodDisruptionBudget

  # Deny specific resources
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
    - group: ""
      kind: LimitRange

  # Roles within this project
  roles:
    - name: developer
      description: Can sync and view apps
      policies:
        - p, proj:production:developer, applications, get, production/*, allow
        - p, proj:production:developer, applications, sync, production/*, allow
      groups:
        - myorg:developers

    - name: deployer
      description: Full app management
      policies:
        - p, proj:production:deployer, applications, *, production/*, allow
      groups:
        - myorg:devops

  # Orphaned resources monitoring
  orphanedResources:
    warn: true                           # Warn if resources exist not managed by ArgoCD
```

---

## ApplicationSet

### 🔴 Q6. What is ApplicationSet and how does it enable fleet management?

```yaml
# ApplicationSet generates multiple Applications from templates
# Use cases:
# - Deploy to multiple clusters
# - Deploy feature branches automatically
# - Per-environment deployments

# ===== CLUSTER GENERATOR =====
# Deploy to all registered clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-all-clusters
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  template:
    metadata:
      name: "my-app-{{name}}"
    spec:
      project: production
      source:
        repoURL: https://github.com/org/my-app.git
        targetRevision: HEAD
        path: k8s/production
        helm:
          values: |
            clusterName: {{name}}
            region: {{metadata.labels.region}}
      destination:
        server: "{{server}}"
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true

---
# ===== LIST GENERATOR =====
# Explicit list of clusters/environments
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-environments
spec:
  generators:
    - list:
        elements:
          - env: staging
            cluster: https://staging-cluster.example.com
            replicas: "2"
          - env: production
            cluster: https://prod-cluster.example.com
            replicas: "5"
  template:
    metadata:
      name: "my-app-{{env}}"
    spec:
      project: "{{env}}"
      source:
        repoURL: https://github.com/org/my-app.git
        path: "k8s/{{env}}"
      destination:
        server: "{{cluster}}"
        namespace: "{{env}}"

---
# ===== GIT GENERATOR =====
# Create an Application per directory in the repo
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: services-appset
spec:
  generators:
    - git:
        repoURL: https://github.com/org/platform.git
        revision: HEAD
        directories:
          - path: services/*            # One app per service directory
          - path: services/deprecated   # Exclude
            exclude: true
  template:
    metadata:
      name: "{{path.basename}}"        # Service name from directory
    spec:
      project: platform
      source:
        repoURL: https://github.com/org/platform.git
        targetRevision: HEAD
        path: "{{path}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{path.basename}}"
      syncPolicy:
        automated: { prune: true, selfHeal: true }
        syncOptions: [CreateNamespace=true]

---
# ===== PULL REQUEST GENERATOR =====
# Create ephemeral apps for each open PR (preview environments!)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-prs
spec:
  generators:
    - pullRequest:
        github:
          owner: myorg
          repo: my-app
          tokenRef:
            secretName: github-token
            key: token
          labels:
            - preview                   # Only PRs with 'preview' label
  template:
    metadata:
      name: "my-app-pr-{{number}}"
    spec:
      project: staging
      source:
        repoURL: https://github.com/myorg/my-app.git
        targetRevision: "{{head_sha}}"
        path: k8s/preview
        helm:
          values: |
            image:
              tag: "pr-{{number}}-{{head_short_sha}}"
            ingress:
              host: "pr-{{number}}.preview.example.com"
      destination:
        server: https://kubernetes.default.svc
        namespace: "preview-pr-{{number}}"
      syncPolicy:
        automated: { prune: true }
        syncOptions: [CreateNamespace=true]
  syncPolicy:
    preserveResourcesOnDeletion: false  # Delete all resources when PR is closed
```

---

## ArgoCD RBAC

### 🟡 Q7. How do you configure ArgoCD RBAC?

```yaml
# argocd-rbac-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly       # Default role for authenticated users

  policy.csv: |
    # Format: p, <subject>, <resource>, <action>, <object>, <effect>
    # Resources: clusters, projects, applications, repositories, certificates, accounts, gpgkeys, logs, exec
    # Actions: get, create, update, delete, sync, override, action

    # Readonly role (built-in)
    p, role:readonly, applications, get, */*, allow
    p, role:readonly, projects, get, *, allow
    p, role:readonly, repositories, get, *, allow

    # Developer role — can sync but not delete
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    p, role:developer, applications, override, */*, allow
    p, role:developer, logs, get, */*, allow

    # DevOps role — full app management
    p, role:devops, applications, *, */*, allow
    p, role:devops, projects, get, *, allow
    p, role:devops, repositories, get, *, allow
    p, role:devops, clusters, get, *, allow

    # Admin role (built-in admin = full access)

    # Restrict production to senior devops only
    p, role:devops, applications, delete, production/*, deny
    p, role:senior-devops, applications, *, */*, allow

    # Group to role mapping (SSO groups)
    g, myorg:developers, role:developer
    g, myorg:devops, role:devops
    g, myorg:senior-devops, role:senior-devops
    g, myorg:platform-admins, role:admin

    # User to role mapping (local users)
    g, ci-bot, role:devops
    g, readonly-bot, role:readonly
```

```bash
# Create local ArgoCD user
kubectl patch configmap argocd-cm -n argocd \
  --patch '{"data":{"accounts.ci-bot":"apiKey,login"}}'

# Generate API key for CI bot
argocd account generate-token --account ci-bot
# Use this token in CI:
argocd app sync my-app --auth-token $ARGOCD_AUTH_TOKEN --server argocd.example.com
```

---

## ArgoCD SSO / Dex

### 🟡 Q8. How do you configure ArgoCD SSO with Dex/OIDC?

```yaml
# argocd-cm — configure SSO
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.example.com

  # Dex config (built-in OIDC proxy)
  dex.config: |
    connectors:
      # GitHub SSO
      - type: github
        id: github
        name: GitHub
        config:
          clientID: $GITHUB_CLIENT_ID
          clientSecret: $GITHUB_CLIENT_SECRET
          redirectURI: https://argocd.example.com/api/dex/callback
          orgs:
            - name: myorg
          loadAllGroups: false
          teamNameField: slug
          useLoginAsID: false

      # GitLab SSO
      - type: gitlab
        id: gitlab
        name: GitLab
        config:
          clientID: $GITLAB_CLIENT_ID
          clientSecret: $GITLAB_CLIENT_SECRET
          redirectURI: https://argocd.example.com/api/dex/callback
          baseURL: https://gitlab.com
          groups:
            - myorg

      # Google / Okta OIDC
      - type: oidc
        id: okta
        name: Okta
        config:
          issuer: https://myorg.okta.com
          clientID: $OKTA_CLIENT_ID
          clientSecret: $OKTA_CLIENT_SECRET
          redirectURI: https://argocd.example.com/api/dex/callback
          insecureEnableGroups: true
          scopes:
            - openid
            - profile
            - email
            - groups
          userIDKey: email
          userNameKey: email
          claimsToGroupsMappings:
            - key: groups

  # OR: Direct OIDC (skip Dex)
  oidc.config: |
    name: Okta
    issuer: https://myorg.okta.com
    clientID: $OIDC_CLIENT_ID
    clientSecret: $OIDC_CLIENT_SECRET
    requestedScopes: [openid, profile, email, groups]
    requestedIDTokenClaims:
      groups:
        essential: true
```

---

## ArgoCD Notifications

### 🟡 Q9. How do you configure ArgoCD notifications?

```yaml
# argocd-notifications-cm
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Slack integration
  service.slack: |
    token: $slack-token
    username: ArgoCD
    icon: ":argo:"

  # Email integration
  service.email: |
    host: smtp.gmail.com
    port: 587
    from: argocd@example.com
    username: argocd@example.com
    password: $email-password

  # Webhook (generic)
  service.webhook.pagerduty: |
    url: https://events.pagerduty.com/v2/enqueue
    headers:
      - name: Content-Type
        value: application/json

  # Notification templates
  template.app-deployed: |
    slack:
      attachments: |
        [{
          "title": "{{.app.metadata.name}}",
          "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [
            {"title":"Sync Status","value":"{{.app.status.sync.status}}","short":true},
            {"title":"Health Status","value":"{{.app.status.health.status}}","short":true},
            {"title":"Revision","value":"{{.app.status.sync.revision | truncate 7 \"\"}}","short":true}
          ]
        }]
    email:
      subject: "✅ {{.app.metadata.name}} deployed successfully"
      body: |
        Application {{.app.metadata.name}} has been deployed.
        Sync Status: {{.app.status.sync.status}}
        Health Status: {{.app.status.health.status}}

  template.app-sync-failed: |
    slack:
      attachments: |
        [{
          "title": "{{.app.metadata.name}} sync failed!",
          "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#E96D76",
          "fields": [
            {"title":"Error","value":"{{.app.status.operationState.message}}","short":false}
          ]
        }]

  # Triggers
  trigger.on-deployed: |
    - description: App is healthy and synced
      send:
        - app-deployed
      when: app.status.health.status == 'Healthy' and app.status.sync.status == 'Synced' and app.status.operationState != nil and app.status.operationState.phase in ['Succeeded']

  trigger.on-sync-failed: |
    - description: Sync failed
      send:
        - app-sync-failed
      when: app.status.operationState.phase in ['Error', 'Failed']

  trigger.on-health-degraded: |
    - description: Health degraded
      send:
        - app-sync-failed
      when: app.status.health.status == 'Degraded'

  # Default subscriptions
  defaultTriggers: |
    - on-sync-failed
    - on-health-degraded
```

```yaml
# Annotate app to enable notifications
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: "#deployments"
    notifications.argoproj.io/subscribe.on-sync-failed.slack: "#alerts"
    notifications.argoproj.io/subscribe.on-health-degraded.email: "devops@example.com"
```

---

## ArgoCD Image Updater

### 🟡 Q10. How does ArgoCD Image Updater work?

```yaml
# Watches container registry for new image tags
# Automatically updates Application with new image

# Install
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

# Annotate Application to enable image updates
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    # Which images to watch
    argocd-image-updater.argoproj.io/image-list: "myapp=registry.example.com/myapp"

    # Update strategy
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    # semver: latest semver tag
    # latest: newest by creation date
    # digest: use digest pinning
    # name: alphabetically latest tag

    # Version constraint
    argocd-image-updater.argoproj.io/myapp.allow-tags: "regexp:^v[0-9]+\\.[0-9]+\\.[0-9]+$"

    # How to write back the new tag
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main

    # For Helm values
    argocd-image-updater.argoproj.io/myapp.helm.image-name: image.repository
    argocd-image-updater.argoproj.io/myapp.helm.image-tag: image.tag
```

---

## Flux Architecture & CRDs

### 🟡 Q11. What is Flux and how does it differ from ArgoCD?

```
Flux v2 (GitOps Toolkit):
  - More modular (each controller independent)
  - Better multi-tenancy (by design)
  - More Kubernetes-native (CRDs for everything)
  - Better Helm support (HelmRelease CRD)
  - No built-in UI (use Weave GitOps, Grafana, etc.)
  - No ApplicationSet equivalent (use Kustomize overlays or Helm + values)

ArgoCD:
  - Built-in Web UI
  - SSO + RBAC out-of-box
  - ApplicationSet for fleet management
  - Multi-source applications
  - More opinionated

Both tools are CNCF graduated projects.
```

```bash
# Install Flux CLI
brew install fluxcd/tap/flux

# Bootstrap (installs Flux into cluster, creates Git repo structure)
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/production \
  --personal
```

```yaml
# ===== GITREPOSITORY — defines the source =====
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m                           # Poll every 1 minute
  url: https://github.com/org/my-app.git
  ref:
    branch: main
  secretRef:
    name: github-credentials             # PAT or deploy key

---
# ===== KUSTOMIZATION — applies manifests from GitRepository =====
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m                           # Reconcile every 5 min
  path: ./k8s/production                 # Path in the repo
  prune: true                            # Delete resources removed from Git
  sourceRef:
    kind: GitRepository
    name: my-app
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-app
      namespace: production
  postBuild:
    substitute:
      cluster_name: production
      region: us-east-1
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars

---
# ===== HELMREPOSITORY =====
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.bitnami.com/bitnami

---
# ===== HELMRELEASE — Helm chart management =====
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: postgresql
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: postgresql
      version: ">=13.0.0 <14.0.0"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
      interval: 1h
  values:
    auth:
      postgresPassword: "${DB_PASSWORD}"  # From Kubernetes secret via substitution
      database: myapp
    primary:
      persistence:
        size: 20Gi
  valuesFrom:
    - kind: Secret
      name: postgresql-values
      valuesKey: values.yaml
  upgrade:
    remediation:
      retries: 3
  rollback:
    timeout: 5m

---
# ===== OCIREPOSITORY (Flux OCI — pull from container registry) =====
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: my-app-manifests
  namespace: flux-system
spec:
  interval: 5m
  url: oci://registry.example.com/my-app-manifests
  ref:
    semver: ">=1.0.0"
  secretRef:
    name: registry-credentials
```

---

## Flux Multi-Tenancy

### 🔴 Q12. How does Flux handle multi-tenancy?

```yaml
# Flux multi-tenancy model:
# - Each tenant has its own namespace
# - Flux impersonates a ServiceAccount per tenant
# - Tenants cannot affect other namespaces

# 1. Platform team creates tenant namespace and ServiceAccount
apiVersion: v1
kind: Namespace
metadata:
  name: team-alpha
  labels:
    toolkit.fluxcd.io/tenant: team-alpha
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flux-reconciler
  namespace: team-alpha
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: flux-reconciler
  namespace: team-alpha
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin           # Scoped to namespace only via namespace isolation
subjects:
  - kind: ServiceAccount
    name: flux-reconciler
    namespace: team-alpha

---
# 2. Kustomization for team-alpha (runs as their ServiceAccount)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team-alpha-apps
  namespace: flux-system
spec:
  serviceAccountName: flux-reconciler   # Impersonate this SA
  targetNamespace: team-alpha           # Force all resources into this namespace
  sourceRef:
    kind: GitRepository
    name: team-alpha-repo
  path: ./apps
  prune: true
  interval: 5m

---
# 3. Tenant Kustomization creates their own sub-Kustomizations
# But those sub-Kustomizations inherit the namespace restriction
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: frontend
  namespace: team-alpha          # Lives in tenant namespace
spec:
  sourceRef:
    kind: GitRepository
    name: team-alpha-repo
    namespace: team-alpha        # Can only use sources in their namespace
  path: ./frontend
  prune: true
  interval: 5m
```

---

## Flux Notifications

### 🟡 Q13. How do you configure Flux notifications?

```yaml
# Flux notification-controller

# Provider — where to send notifications
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: "#gitops-alerts"
  secretRef:
    name: slack-webhook-url        # Secret with "address" key = webhook URL

---
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: github-status
  namespace: flux-system
spec:
  type: github
  address: https://github.com/org/repo
  secretRef:
    name: github-token

---
# Alert — what events trigger notifications
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: on-error
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: error             # info, error
  eventSources:
    - kind: Kustomization
      name: "*"                    # All Kustomizations
    - kind: HelmRelease
      name: "*"
  exclusionList:
    - "^dependencies.*"            # Ignore dependency error messages
  summary: "Cluster production has issues"
```

---

## Secrets in GitOps

### 🔴 Q14. How do you manage secrets in GitOps (Sealed Secrets, SOPS, External Secrets)?

```bash
# ===== SEALED SECRETS =====
# Encrypt Kubernetes Secrets with a cluster-specific key
# Encrypted secret (SealedSecret) is SAFE to commit to Git

# Install controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Install kubeseal CLI
brew install kubeseal

# Seal a secret
kubectl create secret generic db-creds \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > sealed-db-creds.yaml

# Commit sealed-db-creds.yaml to Git (safe!)
cat sealed-db-creds.yaml
# apiVersion: bitnami.com/v1alpha1
# kind: SealedSecret
# metadata:
#   name: db-creds
# spec:
#   encryptedData:
#     password: AgA3xYZ...

# Controller decrypts in-cluster → creates regular Kubernetes Secret

# ===== SOPS (with ArgoCD/Flux) =====
# Encrypt secrets with AWS KMS, GCP KMS, age, PGP

# Encrypt
sops --kms arn:aws:kms:us-east-1:123:key/abc-def secrets.yaml > secrets.enc.yaml

# Flux SOPS integration (via kustomize-controller)
# flux-system/gotk-sync.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age             # Contains AGE_KEY or AWS credentials

---
# ===== EXTERNAL SECRETS OPERATOR (ESO) =====
# Sync secrets from AWS Secrets Manager, Vault, GCP Secret Manager, etc.
# ESO creates and syncs Kubernetes Secrets automatically

# Install ESO
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets

# SecretStore — defines the secret backend
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key

---
# ExternalSecret — syncs a specific secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h              # Re-sync every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials           # Creates this Kubernetes Secret
    creationPolicy: Owner
  data:
    - secretKey: password          # Key in Kubernetes Secret
      remoteRef:
        key: production/db/credentials   # Path in Secrets Manager
        property: password               # JSON key within secret

---
# Vault integration
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault
spec:
  provider:
    vault:
      server: https://vault.example.com
      path: secret
      version: v2
      auth:
        kubernetes:
          mountPath: kubernetes
          role: external-secrets
```

---

## Master Cheatsheet

### ArgoCD Commands
```bash
argocd login <server> --username admin --password <pwd>
argocd app list
argocd app get <app>
argocd app sync <app>
argocd app diff <app>
argocd app history <app>
argocd app rollback <app> <rev>
argocd app delete <app> --cascade
argocd app set <app> --sync-policy automated
argocd app wait <app> --health --sync
argocd proj list
argocd repo list
argocd cluster list
argocd account generate-token --account ci-bot
```

### Flux Commands
```bash
flux check                          # Check Flux health
flux get all                        # Show all Flux resources
flux get kustomizations             # List Kustomizations
flux get helmreleases               # List HelmReleases
flux reconcile kustomization my-app # Force reconcile
flux reconcile helmrelease postgresql
flux suspend kustomization my-app   # Pause reconciliation
flux resume kustomization my-app
flux logs --follow                  # Stream Flux logs
flux events                         # Show recent events
flux diff kustomization my-app      # Preview changes
flux export kustomization my-app    # Export to YAML
flux bootstrap github ...           # Bootstrap Flux
```

### Question Coverage Index
| Q# | Topic | Difficulty |
|---|---|---|
| Q1 | GitOps four principles | 🟢 |
| Q2 | GitOps vs push-based CD | 🟡 |
| Q3 | ArgoCD architecture | 🟢 |
| Q4 | ArgoCD Applications | 🟢 |
| Q5 | AppProject & multi-tenancy | 🟡 |
| Q6 | ApplicationSet fleet management | 🔴 |
| Q7 | ArgoCD RBAC | 🟡 |
| Q8 | ArgoCD SSO with Dex/OIDC | 🟡 |
| Q9 | ArgoCD Notifications | 🟡 |
| Q10 | ArgoCD Image Updater | 🟡 |
| Q11 | Flux CRDs: GitRepository, Kustomization, HelmRelease | 🟡 |
| Q12 | Flux multi-tenancy | 🔴 |
| Q13 | Flux notifications | 🟡 |
| Q14 | Secrets: Sealed Secrets, SOPS, ESO | 🔴 |
