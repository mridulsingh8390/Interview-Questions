# Helm & Helmfile — Complete Study Guide
### All Topics · Detailed Q&A · Master Cheatsheet
**2025 Edition — Comprehensive coverage with examples, commands & cheatsheet**

---

## Table of Contents
- [Helm Basics](#helm-basics)
- [Chart Structure & Templating](#chart-structure)
- [Helm Repositories & Dependencies](#helm-repositories)
- [Helm Hooks & Tests](#helm-hooks)
- [Helm Rollback & Lifecycle](#helm-rollback)
- [Helm Plugins, OCI & Publishing](#helm-plugins)
- [Post-Renderer & Kustomize Integration](#post-renderer)
- [Helmfile](#helmfile)
- [Helmfile Environments & Secrets](#helmfile-secrets)
- [Master Cheatsheet](#master-cheatsheet)

---

## Helm Basics

### 🟢 Q1. What is Helm and why do we need it?

**Explanation:**
Helm is the package manager for Kubernetes. Without Helm, deploying an application to Kubernetes means writing and maintaining many raw YAML files — Deployments, Services, ConfigMaps, Ingress, RBAC, etc. — and hand-editing them for each environment. Helm solves this by introducing **Charts**: reusable, parameterised packages of Kubernetes manifests. A single `helm install` command deploys everything, and values can be overridden per environment without duplicating YAML.

Key concepts:
- **Chart** — a package of Kubernetes manifests with templates and default values
- **Release** — a running instance of a chart in a cluster (one chart can have multiple releases)
- **Revision** — a numbered snapshot of a release (used for rollbacks)
- **Repository** — a collection of charts hosted on an HTTP server or OCI registry

```bash
# Install helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version

# Basic workflow
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/nginx
helm install my-nginx bitnami/nginx          # Install with defaults
helm install my-nginx bitnami/nginx \
  --namespace production \
  --create-namespace \
  --set replicaCount=3 \
  --set service.type=LoadBalancer
helm list -A                                  # List all releases
helm status my-nginx                          # Status of a release
helm uninstall my-nginx                       # Remove release
```

---

### 🟢 Q2. What is the difference between a Chart, Release, and Revision?

**Explanation:**
- **Chart**: the template/blueprint. Lives in a repo or local directory. Version-controlled independently.
- **Release**: a deployment of a chart with specific values. Named, namespaced, stored as a Secret in the cluster.
- **Revision**: every `helm install` creates revision 1. Every `helm upgrade` increments the revision. Every `helm rollback` creates a new revision pointing to old manifests.

```bash
# See revisions of a release
helm history my-app -n production
# REVISION  UPDATED                  STATUS     CHART         APP VERSION  DESCRIPTION
# 1         Mon Jan 01 10:00:00 2024  superseded  my-app-1.0.0  1.0.0        Install complete
# 2         Mon Jan 01 11:00:00 2024  superseded  my-app-1.1.0  1.1.0        Upgrade complete
# 3         Mon Jan 01 12:00:00 2024  deployed    my-app-1.1.0  1.1.0        Rollback to 2

# Release metadata is stored as k8s secrets
kubectl get secrets -n production | grep helm
# sh.helm.release.v1.my-app.v1
# sh.helm.release.v1.my-app.v2
# sh.helm.release.v1.my-app.v3
```

---

### 🟢 Q3. How do you install, upgrade, rollback, and uninstall a Helm release?

**Explanation:**
These four operations form the Helm lifecycle. `upgrade --install` is idempotent and useful in CI/CD pipelines. `--atomic` rolls back automatically on failure. `--dry-run` previews without applying. `rollback` is essential for quick recovery.

```bash
# INSTALL
helm install my-app ./my-chart \
  --namespace production \
  --create-namespace \
  -f values-prod.yaml \
  --set image.tag=v1.2.3 \
  --timeout 5m \
  --wait                           # Wait for pods to be Ready
  --atomic                         # Auto-rollback on failure

# Idempotent install/upgrade (great for CI/CD)
helm upgrade --install my-app ./my-chart \
  --namespace production \
  -f values-prod.yaml \
  --set image.tag=$IMAGE_TAG \
  --atomic \
  --cleanup-on-fail               # Delete new resources on failure

# UPGRADE
helm upgrade my-app ./my-chart \
  --namespace production \
  -f values-prod.yaml \
  --set image.tag=v1.3.0 \
  --reuse-values                  # Keep all previous values, only override specified

# DRY RUN — preview rendered manifests without applying
helm upgrade --install my-app ./my-chart \
  -f values-prod.yaml \
  --dry-run

# Render locally without a cluster
helm template my-app ./my-chart \
  -f values-prod.yaml \
  --set image.tag=v1.3.0

# ROLLBACK
helm rollback my-app -n production         # Rollback to previous revision
helm rollback my-app 2 -n production       # Rollback to specific revision
helm rollback my-app 0 -n production       # 0 = previous revision
helm rollback my-app --wait --timeout 3m   # Wait for rollback to complete

# UNINSTALL
helm uninstall my-app -n production
helm uninstall my-app -n production --keep-history  # Keep release history

# DIFF (requires helm-diff plugin)
helm diff upgrade my-app ./my-chart -f values-prod.yaml
```

---

### 🟡 Q4. How do Values work and what is values precedence?

**Explanation:**
Values are the configuration layer of Helm. They flow from multiple sources with a defined precedence (highest wins):

1. `--set` flags on CLI (highest)
2. `--set-string`, `--set-file`, `--set-json`
3. `-f values.yaml` files (last file wins if multiple)
4. `values.yaml` in the chart (lowest)

```yaml
# chart/values.yaml — chart defaults
replicaCount: 1
image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  host: ""

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

config:
  logLevel: info
  features:
    featureA: false
    featureB: false
```

```yaml
# values-prod.yaml — environment override
replicaCount: 3
image:
  tag: "1.25.3"
service:
  type: LoadBalancer
ingress:
  enabled: true
  host: "app.example.com"
config:
  logLevel: warn
  features:
    featureA: true
```

```bash
# Precedence demo — --set wins over -f file, -f file wins over chart defaults
helm upgrade --install my-app ./chart \
  -f values-prod.yaml \
  --set replicaCount=5          # This wins — 5 replicas

# Set nested values
--set config.features.featureB=true
--set "ingress.annotations.nginx\.ingress\.kubernetes\.io/rewrite-target=/"

# Set list values
--set "tolerations[0].key=node-role,tolerations[0].operator=Exists"

# Set from file (e.g. TLS cert)
--set-file "tls.cert=./certs/tls.crt"

# Set as explicit string (prevents type coercion)
--set-string "image.tag=1.0"     # Prevents 1.0 being treated as float

# Set as JSON
--set-json 'config={"logLevel":"debug","maxConn":100}'

# Inspect rendered values for a release
helm get values my-app -n production         # User-supplied values
helm get values my-app -n production --all   # All values including defaults
```

---

## Chart Structure & Templating

### 🟢 Q5. What is the structure of a Helm chart?

**Explanation:**
A Helm chart is a directory with a specific structure. Understanding each file's role is fundamental for creating and debugging charts.

```
my-chart/
├── Chart.yaml              # Chart metadata (name, version, description, dependencies)
├── values.yaml             # Default configuration values
├── values.schema.json      # JSON Schema for values validation (optional)
├── charts/                 # Dependency charts (populated by helm dependency update)
├── crds/                   # CRDs applied before templates (not templated)
├── templates/              # Kubernetes manifest templates (Go templating)
│   ├── NOTES.txt           # Post-install notes printed to user
│   ├── _helpers.tpl        # Named templates / helper partials
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── serviceaccount.yaml
│   ├── hpa.yaml
│   └── tests/
│       └── test-connection.yaml
└── .helmignore             # Files to exclude when packaging
```

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
description: A Helm chart for my application
type: application          # or 'library' for shared templates
version: 1.2.3             # Chart version (semver)
appVersion: "2.0.0"        # App version (string)
icon: https://example.com/icon.png
keywords:
  - web
  - api
maintainers:
  - name: Platform Team
    email: platform@example.com

# Dependencies
dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled    # Only if values.postgresql.enabled=true
  - name: redis
    version: "18.x.x"
    repository: https://charts.bitnami.com/bitnami
    alias: cache                     # Reference as .Values.cache.*
    condition: cache.enabled
```

---

### 🟡 Q6. How does Helm templating work?

**Explanation:**
Helm uses Go's `text/template` package plus Sprig functions. Templates have access to built-in objects: `.Values`, `.Release`, `.Chart`, `.Files`, `.Capabilities`, `.Template`.

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}   # Use named template
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}   # nindent adds newline + indent
  annotations:
    helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort | default 8080 }}
          env:
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}    # quote wraps in ""
            {{- range $key, $val := .Values.config.extraEnv }}
            - name: {{ $key }}
              value: {{ $val | quote }}
            {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- if .Values.probes.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path | default "/healthz" }}
              port: {{ .Values.service.targetPort | default 8080 }}
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds | default 30 }}
          {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

---

### 🟡 Q7. What are Helm template functions and pipelines?

**Explanation:**
Helm provides Sprig (a Go template function library) plus custom Helm functions. Pipelines pass output from one function to the next using `|`. Key functions every Helm author must know:

```yaml
# ===== STRING FUNCTIONS =====
{{ .Values.name | upper }}               # MYAPP
{{ .Values.name | lower }}               # myapp
{{ .Values.name | title }}               # My App
{{ .Values.name | trim }}                # remove whitespace
{{ .Values.name | trimAll "-" }}         # trim specific chars
{{ .Values.name | replace " " "-" }}     # replace spaces with hyphens
{{ .Values.name | trunc 63 | trimSuffix "-" }}  # k8s name limit
{{ printf "%s-%s" .Release.Name .Chart.Name }}  # string formatting
{{ .Values.name | contains "prod" }}     # boolean contains check
{{ .Values.name | hasPrefix "app-" }}

# ===== TYPE CONVERSION =====
{{ .Values.port | toString }}
{{ "8080" | int }}
{{ .Values.flag | toString | lower | eq "true" }}
{{ .Values.name | quote }}               # wrap in double quotes
{{ .Values.name | squote }}             # wrap in single quotes

# ===== DEFAULTS AND CONDITIONALS =====
{{ .Values.tag | default "latest" }}     # fallback value
{{ .Values.replicas | default 1 }}
{{ if .Values.ingress.enabled }}...{{ end }}
{{ if and .Values.ingress.enabled .Values.ingress.tls }}...{{ end }}
{{ if or .Values.debug .Values.verbose }}...{{ end }}
{{ if not .Values.readonly }}...{{ end }}

# ===== MATH =====
{{ add 1 .Values.replicas }}
{{ mul .Values.replicas 2 }}
{{ div .Values.memory 1024 }}
{{ max 1 .Values.minReplicas }}

# ===== LIST AND DICT FUNCTIONS =====
{{ .Values.tags | join "," }}
{{ list "a" "b" "c" | join "-" }}
{{ .Values.hosts | first }}
{{ .Values.hosts | last }}
{{ .Values.hosts | len }}
{{ hasKey .Values.annotations "team" }}
{{ keys .Values.labels | sortAlpha | join "," }}
{{ merge .Values.defaults .Values.overrides }}  # merge dicts
{{ pick .Values.config "key1" "key2" }}          # pick specific keys
{{ omit .Values.config "password" }}             # omit specific keys

# ===== YAML / JSON ENCODING =====
{{ toYaml .Values.resources | nindent 12 }}
{{ toJson .Values.config }}
{{ fromYaml .Values.rawYaml }}
{{ .Values.config | toJson | b64enc }}    # base64 encode JSON

# ===== CRYPTO =====
{{ .Values.data | sha256sum }}
{{ randAlphaNum 32 }}                           # random password
{{ genSelfSignedCert "example.com" nil nil 365 }}  # TLS cert

# ===== DATES =====
{{ now | htmlDate }}                     # 2024-01-15
{{ now | unixEpoch }}                    # Unix timestamp
{{ dateInZone "2006-01-02" (now) "UTC" }}

# ===== PIPELINES =====
# Multiple pipes — output of left becomes input to right
{{ .Values.name | lower | trunc 63 | trimSuffix "-" | quote }}

# nindent vs indent — nindent adds leading newline, indent does not
{{- toYaml .Values.resources | nindent 12 }}
# generates:
#             cpu: 100m
#             memory: 128Mi

# Whitespace control — {{- trims preceding whitespace, -}} trims following
{{- if .Values.debug }}
  debug: true
{{- end }}
```

---

### 🔴 Q8. What are named templates and how do you use include vs template?

**Explanation:**
Named templates (defined in `_helpers.tpl` using `{{- define ... }}`) allow reuse across chart templates. `include` is preferred over `template` because `include` returns a string that can be piped to functions like `nindent`, while `template` outputs directly and cannot be piped.

```yaml
# templates/_helpers.tpl

{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels — applied to all resources
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- with .Values.commonLabels }}
{{ toYaml . }}
{{- end }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Service account name
*/}}
{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
Image tag — accepts context and a dict with overrides
Usage: {{ include "my-app.image" (dict "Values" .Values "imageKey" "sidecar") }}
*/}}
{{- define "my-app.image" -}}
{{- $img := index .Values .imageKey | default .Values.image -}}
{{- printf "%s:%s" $img.repository ($img.tag | default "latest") -}}
{{- end }}
```

```yaml
# Using named templates in deployment.yaml

metadata:
  name: {{ include "my-app.fullname" . }}    # include returns string → pipeable
  labels:
    {{- include "my-app.labels" . | nindent 4 }}   # pipe to nindent ✅

# WRONG — template cannot be piped
  labels:
    {{- template "my-app.labels" . | nindent 4 }}   # ❌ syntax error

# Passing custom context to a named template
{{- include "my-app.image" (dict "Values" .Values "imageKey" "worker") }}

# Calling with modified scope
{{- include "my-app.labels" (dict "Chart" .Chart "Release" .Release "Values" .Values) | nindent 4 }}
```

---

## Helm Repositories & Dependencies

### 🟢 Q9. How do you manage Helm repositories?

```bash
# Add repos
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update all repos
helm repo update

# List configured repos
helm repo list

# Search across all repos
helm search repo nginx
helm search repo nginx --versions       # All versions
helm search repo nginx --version ">=1.0.0 <2.0.0"

# Search Artifact Hub
helm search hub nginx

# Remove a repo
helm repo remove stable

# OCI registry (no repo add needed)
helm install my-chart oci://registry.example.com/charts/my-chart --version 1.0.0
helm pull oci://registry.example.com/charts/my-chart --version 1.0.0
```

---

### 🟡 Q10. How do you manage Chart dependencies?

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "13.2.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
    tags:
      - database

  - name: redis
    version: ">=18.0.0"
    repository: https://charts.bitnami.com/bitnami
    alias: cache
    condition: cache.enabled
    tags:
      - cache

  - name: common
    version: "2.x.x"
    repository: https://charts.bitnami.com/bitnami
    # Library chart — only provides named templates
```

```bash
# Download dependencies into charts/ directory
helm dependency update ./my-chart      # Fetches & creates Chart.lock

# List dependencies
helm dependency list ./my-chart

# Build from Chart.lock (exact versions)
helm dependency build ./my-chart
```

```yaml
# values.yaml — control dependency with condition
postgresql:
  enabled: true
  auth:
    postgresPassword: secret
    database: myapp
  primary:
    persistence:
      enabled: true
      size: 20Gi

cache:
  enabled: false      # redis disabled

# Access sub-chart values from parent
# Sub-chart values are namespaced under the chart name (or alias)
```

---

## Helm Hooks & Tests

### 🟡 Q11. What are Helm hooks and how do you use them?

**Explanation:**
Hooks are templates with the annotation `helm.sh/hook` that run at specific points in a release lifecycle — before/after install, upgrade, delete, or rollback. Common uses: database migrations, pre-flight checks, cleanup jobs.

```yaml
# templates/migrations.yaml — pre-upgrade job
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "my-app.fullname" . }}-migrations
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install    # Runs before install AND upgrade
    "helm.sh/hook-weight": "-5"               # Execution order (lower = first)
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
    # before-hook-creation: delete old job before creating new
    # hook-succeeded: delete after success
    # hook-failed: delete after failure (for debugging keep it — omit this)
spec:
  backoffLimit: 0          # Don't retry
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrations
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["./run-migrations.sh"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-app.fullname" . }}-secrets
                  key: database-url
---
# Hook types and when they run:
# pre-install   — before any resources are installed
# post-install  — after all resources are installed
# pre-upgrade   — before upgrade
# post-upgrade  — after upgrade
# pre-rollback  — before rollback
# post-rollback — after rollback
# pre-delete    — before delete
# post-delete   — after delete
# test          — when "helm test" is run
```

---

### 🟡 Q12. How do you write and run Helm tests?

**Explanation:**
Helm tests are Pods with the `helm.sh/hook: test` annotation. They validate that a release works correctly after installation. Tests run `helm test <release>` and pass if the Pod exits 0, fail otherwise.

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "my-app.fullname" . }}-test-connection
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args:
        - '--spider'
        - '--timeout=5'
        - 'http://{{ include "my-app.fullname" . }}:{{ .Values.service.port }}/health'
---
# More thorough test
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "my-app.fullname" . }}-test-api
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: test
      image: curlimages/curl:latest
      command:
        - /bin/sh
        - -c
        - |
          set -e
          echo "Testing health endpoint..."
          curl -sf http://{{ include "my-app.fullname" . }}:{{ .Values.service.port }}/health
          echo "Testing API endpoint..."
          response=$(curl -sf http://{{ include "my-app.fullname" . }}:{{ .Values.service.port }}/api/v1/status)
          echo $response | grep -q '"status":"ok"'
          echo "All tests passed!"
```

```bash
# Run tests
helm test my-app -n production

# See test output
helm test my-app -n production --logs

# Test with filter
helm test my-app --filter name=test-connection
```

---

## Helm Rollback & Lifecycle

### 🟡 Q13. How do Helm rollbacks work in detail?

**Explanation:**
`helm rollback` creates a new revision that re-applies the manifests from a previous revision. It does NOT undo secrets or persistent data changes. The `--recreate-pods` flag forces pod restart even if the manifest didn't change.

```bash
# View release history
helm history my-app -n production
# REVISION  UPDATED              STATUS      CHART         DESCRIPTION
# 1         2024-01-10 10:00:00  superseded  my-app-1.0.0  Install complete
# 2         2024-01-15 14:00:00  superseded  my-app-1.1.0  Upgrade complete
# 3         2024-01-20 09:00:00  failed      my-app-1.2.0  Upgrade "my-app" failed
# 4         2024-01-20 09:05:00  deployed    my-app-1.1.0  Rollback to 2

# Rollback to previous revision
helm rollback my-app -n production

# Rollback to specific revision
helm rollback my-app 2 -n production

# Rollback options
helm rollback my-app 2 \
  --wait \
  --timeout 5m \
  --recreate-pods \        # Force pod restart
  --cleanup-on-fail        # Clean up on failed rollback

# Get manifests that were applied in a revision
helm get manifest my-app --revision 2 -n production

# Get values from a specific revision
helm get values my-app --revision 2 -n production

# What changed between revisions (requires helm-diff plugin)
helm diff revision my-app 2 3 -n production

# Auto-rollback: use --atomic flag on upgrade
helm upgrade my-app ./chart -f values.yaml --atomic
# If upgrade fails → automatic rollback to last successful revision
```

---

## Helm Plugins, OCI & Publishing

### 🟡 Q14. What are important Helm plugins?

```bash
# helm-diff — show diff before applying upgrade
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade my-app ./chart -f values.yaml
helm diff revision my-app 2 3

# helm-secrets — encrypted values with SOPS or Vault
helm plugin install https://github.com/jkroepke/helm-secrets
# With SOPS + AWS KMS
helm secrets encrypt values-secrets.yaml
helm secrets decrypt values-secrets.yaml
helm secrets upgrade my-app ./chart \
  -f values.yaml \
  -f secrets://values-secrets.yaml.enc

# helm-unittest — unit testing for templates
helm plugin install https://github.com/helm-unittest/helm-unittest
helm unittest ./my-chart

# helm-docs — auto-generate README from values.yaml
helm plugin install https://github.com/norwoodj/helm-docs
helm-docs                   # generates README.md from chart

# helm-mapkubeapis — fix deprecated API versions
helm plugin install https://github.com/helm/helm-mapkubeapis
helm mapkubeapis my-release -n production

# List installed plugins
helm plugin list

# Update plugin
helm plugin update diff
```

---

### 🔴 Q15. How do you create and publish your own Helm chart?

```bash
# Create a new chart scaffold
helm create my-app

# Lint the chart
helm lint ./my-app
helm lint ./my-app -f values-prod.yaml    # Lint with specific values
helm lint ./my-app --strict               # Fail on warnings too

# Package the chart
helm package ./my-app                     # Creates my-app-0.1.0.tgz
helm package ./my-app --destination ./dist

# === HOST ON HTTP REPO ===
# Create index file
helm repo index ./dist --url https://charts.example.com
# Serve with any web server (nginx, GitHub Pages, S3)

# === PUSH TO OCI REGISTRY ===
# Login
helm registry login registry.example.com -u myuser -p mypassword
helm registry login ghcr.io -u github-user --password-stdin < token.txt

# Push chart
helm push my-app-0.1.0.tgz oci://registry.example.com/charts

# Pull chart
helm pull oci://registry.example.com/charts/my-app --version 0.1.0

# Install from OCI
helm install my-release oci://registry.example.com/charts/my-app --version 0.1.0

# === GITHUB PAGES (popular free chart hosting) ===
# 1. Package charts
helm package ./charts/my-app -d ./docs
# 2. Generate index
helm repo index ./docs --url https://myorg.github.io/helm-charts
# 3. Push docs/ to GitHub, enable Pages
# 4. Users: helm repo add myorg https://myorg.github.io/helm-charts
```

---

### 🔴 Q16. What is values.schema.json and how do you validate values?

**Explanation:**
`values.schema.json` is a JSON Schema file that validates user-supplied values at install/upgrade time. Helm rejects any values that fail validation — catching misconfiguration before resources are applied.

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "my-app values",
  "type": "object",
  "required": ["image", "service"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 50,
      "default": 1,
      "description": "Number of pod replicas"
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "minLength": 1,
          "description": "Container image repository"
        },
        "tag": {
          "type": "string",
          "default": "latest"
        },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "IfNotPresent", "Never"],
          "default": "IfNotPresent"
        }
      },
      "additionalProperties": false
    },
    "service": {
      "type": "object",
      "required": ["type", "port"],
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer", "ExternalName"]
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    },
    "resources": {
      "type": "object",
      "properties": {
        "requests": {
          "type": "object",
          "properties": {
            "cpu": { "type": "string", "pattern": "^[0-9]+(m|\\.[0-9]+)?$" },
            "memory": { "type": "string", "pattern": "^[0-9]+(Ki|Mi|Gi|Ti)?$" }
          }
        }
      }
    },
    "config": {
      "type": "object",
      "properties": {
        "logLevel": {
          "type": "string",
          "enum": ["debug", "info", "warn", "error"],
          "default": "info"
        }
      }
    }
  },
  "additionalProperties": true
}
```

```bash
# Validation happens automatically on install/upgrade
helm install my-app ./chart --set replicaCount=200
# Error: values don't meet the specifications of the schema(s) in the following chart(s):
# my-app:
# - replicaCount: Must be less than or equal to 50

helm install my-app ./chart --set service.type=InvalidType
# Error: service.type must be one of the following: "ClusterIP", "NodePort", ...
```

---

## Post-Renderer & Kustomize Integration

### 🔴 Q17. What is a post-renderer and how do you use it with Kustomize?

**Explanation:**
A post-renderer is an executable that Helm pipes its rendered YAML through before applying to the cluster. This allows arbitrary modifications to chart output without forking the chart — patching values the chart doesn't expose, adding custom labels/annotations, transforming resources. Kustomize is the most common post-renderer.

```bash
# Create a kustomize overlay directory
mkdir -p kustomize-overlay

cat > kustomize-overlay/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - all.yaml              # Helm piped output goes here

patches:
  - patch: |-
      - op: add
        path: /metadata/annotations/custom-team
        value: platform
    target:
      kind: Deployment

  - patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: 512Mi
    target:
      kind: Deployment
      name: my-app

commonLabels:
  cost-center: "12345"
  managed-by-custom: "true"

commonAnnotations:
  custom.example.com/version: "v2"
EOF

# Create wrapper script
cat > kustomize-post-renderer.sh << 'EOF'
#!/bin/bash
cat > kustomize-overlay/all.yaml
kustomize build kustomize-overlay
EOF
chmod +x kustomize-post-renderer.sh

# Use with helm
helm install my-app ./chart \
  -f values.yaml \
  --post-renderer ./kustomize-post-renderer.sh

# Or upgrade
helm upgrade my-app ./chart \
  -f values.yaml \
  --post-renderer ./kustomize-post-renderer.sh

# You can also use post-renderer with --post-renderer-args for custom scripts
helm install my-app ./chart \
  --post-renderer ./my-transformer.py \
  --post-renderer-args "--env=production" \
  --post-renderer-args "--team=platform"
```

---

## Helmfile

### 🟢 Q18. What is Helmfile and why use it?

**Explanation:**
Helmfile is a declarative spec for deploying multiple Helm charts. It solves the problem of orchestrating many releases across many clusters/environments. Instead of maintaining many `helm upgrade` scripts, you describe desired state in `helmfile.yaml`. Helmfile handles ordering (needs), diff, sync, and destroy.

```bash
# Install helmfile
brew install helmfile         # macOS
# Or download binary from https://github.com/helmfile/helmfile/releases

# Helmfile commands
helmfile sync          # Apply all releases
helmfile diff          # Show what would change
helmfile apply         # Diff + sync if changes detected (safe CI/CD command)
helmfile lint          # Lint all charts
helmfile destroy       # Uninstall all releases
helmfile template      # Render all manifests
helmfile test          # Run helm test on all releases
helmfile status        # Show status of all releases

# Target specific releases
helmfile -l app=nginx sync
helmfile -l tier=database apply
helmfile --selector name=my-app diff
```

---

### 🟡 Q19. What is the structure of a helmfile.yaml?

```yaml
# helmfile.yaml — complete structure

# Repositories
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx
  - name: cert-manager
    url: https://charts.jetstack.io
  - name: private-oci
    url: oci://registry.example.com/charts
    oci: true

# Helm settings
helmDefaults:
  wait: true
  timeout: 600
  atomic: false
  cleanupOnFail: true
  createNamespace: true
  historyMax: 5
  tillerNamespace: kube-system
  kubeContext: production-cluster

# Environments
environments:
  staging:
    values:
      - environments/staging/values.yaml
    secrets:
      - environments/staging/secrets.yaml.enc
  production:
    values:
      - environments/production/values.yaml
    secrets:
      - environments/production/secrets.yaml.enc
    kubeContext: prod-eks-cluster

# Releases
releases:
  # Infrastructure layer
  - name: cert-manager
    namespace: cert-manager
    chart: cert-manager/cert-manager
    version: v1.14.0
    labels:
      tier: infrastructure
    values:
      - installCRDs: true
    wait: true

  - name: ingress-nginx
    namespace: ingress-nginx
    chart: ingress-nginx/ingress-nginx
    version: 4.9.0
    labels:
      tier: infrastructure
    needs:
      - cert-manager/cert-manager      # Deploy after cert-manager
    values:
      - controller:
          replicaCount: 2
          service:
            type: LoadBalancer

  # Application layer
  - name: my-app
    namespace: production
    chart: ./charts/my-app
    version: ~1.2.0              # semver range
    labels:
      app: my-app
      tier: application
    needs:
      - ingress-nginx/ingress-nginx
    values:
      - charts/my-app/values.yaml
      - environments/{{ .Environment.Name }}/my-app.yaml  # per-env override
    set:
      - name: image.tag
        value: {{ requiredEnv "IMAGE_TAG" }}              # from env var
      - name: replicaCount
        value: {{ .Environment.Values.replicaCount | default 2 }}

  - name: postgresql
    namespace: production
    chart: bitnami/postgresql
    version: 13.2.x
    labels:
      tier: database
    values:
      - auth:
          postgresPassword: {{ requiredEnv "POSTGRES_PASSWORD" }}
          database: myapp
      - primary:
          persistence:
            size: {{ .Environment.Values.dbSize | default "20Gi" }}

# Hooks
hooks:
  - events: ["prepare"]
    showlogs: true
    command: "./scripts/pre-deploy.sh"
    args:
      - "{{ .Environment.Name }}"
  - events: ["cleanup"]
    command: "./scripts/post-deploy.sh"
```

---

### 🟡 Q20. What are Helmfile selectors, diff, and lock?

```bash
# ===== SELECTORS — target specific releases =====
# Labels defined in releases[].labels
helmfile -l tier=infrastructure sync
helmfile -l app=my-app,tier=application apply
helmfile -l name=postgresql diff
helmfile --selector "tier!=infrastructure" destroy   # NOT selector

# ===== DIFF — preview changes =====
helmfile diff
# Shows:
# - Resources that will be added (green +)
# - Resources that will be removed (red -)
# - Resources that will be modified (yellow ~)

helmfile diff --context 5          # Show 5 lines of context around changes
helmfile diff --suppress-secrets   # Don't show secret values

# ===== APPLY vs SYNC =====
# apply = diff + sync only if there are changes (preferred for CI/CD)
# sync = always apply regardless (always does a helm upgrade)
helmfile apply                     # Safe — only deploys if needed
helmfile sync                      # Always deploys
helmfile sync --concurrency 4      # Deploy 4 releases in parallel

# ===== HELMFILE.LOCK =====
# Pin exact chart versions (like package-lock.json)
helmfile deps                      # Resolves & writes helmfile.lock
# helmfile.lock created:
# dependencies:
# - name: cert-manager
#   repository: https://charts.jetstack.io
#   version: v1.14.2               # Exact version pinned

helmfile sync --skip-deps          # Use versions from lock file
```

---

## Helmfile Environments & Secrets

### 🔴 Q21. How do you manage environments and secrets in Helmfile?

**Explanation:**
Helmfile supports multiple environments, each with their own values and secrets files. Secrets are encrypted using **helm-secrets** (backed by SOPS or Vault). The environment is selected with `--environment` flag.

```
helmfile-project/
├── helmfile.yaml
├── environments/
│   ├── staging/
│   │   ├── values.yaml           # Plain values
│   │   └── secrets.yaml.enc      # SOPS-encrypted secrets
│   └── production/
│       ├── values.yaml
│       └── secrets.yaml.enc
└── charts/
    └── my-app/
```

```yaml
# environments/production/values.yaml
replicaCount: 5
dbSize: 100Gi
ingress:
  host: app.example.com
  tls: true
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2
    memory: 2Gi
```

```yaml
# environments/production/secrets.yaml — BEFORE encryption
databasePassword: supersecretpassword
apiKey: sk-prod-xxxxx
tlsCert: |
  -----BEGIN CERTIFICATE-----
  ...
```

```bash
# Encrypt with SOPS + AWS KMS
export SOPS_KMS_ARN="arn:aws:kms:us-east-1:123456789:key/abc-def"
sops --encrypt environments/production/secrets.yaml > environments/production/secrets.yaml.enc

# Or with age encryption
export SOPS_AGE_RECIPIENTS="age1xxxxxxxxx"
sops --encrypt --age $SOPS_AGE_RECIPIENTS environments/production/secrets.yaml \
  > environments/production/secrets.yaml.enc

# helmfile.yaml — use encrypted secrets
environments:
  production:
    values:
      - environments/production/values.yaml
    secrets:
      - environments/production/secrets.yaml.enc   # auto-decrypted by helm-secrets

# Deploy to production
helmfile -e production apply

# Deploy to staging
helmfile -e staging apply

# Using secrets in releases
releases:
  - name: my-app
    values:
      - db:
          password: {{ .Environment.Values.databasePassword }}
      - api:
          key: {{ .Environment.Values.apiKey }}
```

```yaml
# helmfile.yaml — advanced environment patterns
environments:
  production:
    values:
      - environments/production/values.yaml
      - environments/production/secrets.yaml.enc
    missingFileHandler: Error     # Fail if file missing

# Conditional release per environment
releases:
  - name: debug-tools
    chart: ./charts/debug
    condition: debug.enabled      # Only deploy if .Values.debug.enabled = true
    # In staging values: debug.enabled: true
    # In production values: debug.enabled: false (or omit)

  - name: my-app
    values:
      - image:
          tag: {{ env "IMAGE_TAG" | default "latest" }}
          pullPolicy: {{ if eq .Environment.Name "production" }}IfNotPresent{{ else }}Always{{ end }}
      - replicaCount: >-
          {{ if eq .Environment.Name "production" }}
          {{ .Environment.Values.replicaCount }}
          {{ else }}
          1
          {{ end }}
```

---

## Master Cheatsheet

### Helm Commands
```bash
# Install / Upgrade
helm install <name> <chart> -n <ns> -f values.yaml --create-namespace
helm upgrade --install <name> <chart> -n <ns> -f values.yaml --atomic
helm upgrade <name> <chart> --reuse-values --set image.tag=v2

# Inspect
helm list -A                         # All releases all namespaces
helm status <name> -n <ns>
helm history <name> -n <ns>
helm get values <name> -n <ns>
helm get manifest <name> -n <ns>
helm get all <name> -n <ns>

# Rollback
helm rollback <name> -n <ns>         # to previous
helm rollback <name> <rev> -n <ns>   # to specific rev

# Chart Development
helm create <chart>
helm lint <chart>
helm template <name> <chart> -f values.yaml
helm package <chart>
helm repo index .

# Repos
helm repo add <name> <url>
helm repo update
helm repo list
helm search repo <keyword>
helm search hub <keyword>

# Plugins
helm plugin install <url>
helm plugin list
helm plugin update <name>
helm diff upgrade <name> <chart>     # requires helm-diff

# OCI
helm registry login <registry>
helm push <chart.tgz> oci://<registry>/<org>
helm pull oci://<registry>/<org>/<chart> --version <ver>

# Test
helm test <name> -n <ns>
helm test <name> -n <ns> --logs
```

### Helmfile Commands
```bash
helmfile apply              # diff + sync if changes
helmfile sync               # always sync
helmfile diff               # preview changes
helmfile lint               # lint all charts
helmfile template           # render all manifests
helmfile destroy            # uninstall all
helmfile test               # run helm tests
helmfile deps               # update lock file
helmfile -e production apply    # target environment
helmfile -l app=nginx sync      # target by label selector
```

### Template Quickref
```yaml
{{ .Values.key }}                      # Value access
{{ .Values.key | default "fallback" }} # Default
{{ .Values.key | quote }}              # Wrap in quotes
{{ .Values.key | upper }}              # Uppercase
{{ .Values.key | trunc 63 }}           # Truncate
{{ toYaml .Values.obj | nindent 4 }}  # Render YAML with indent
{{ include "chart.fullname" . }}       # Named template
{{- if .Values.enabled }}...{{- end }} # Conditional (trim whitespace)
{{- range .Values.list }}{{ . }}{{- end }} # Loop list
{{- range $k, $v := .Values.map }}{{ $k }}: {{ $v }}{{- end }} # Loop map
{{ required "msg" .Values.key }}       # Fail if not set
{{ .Release.Name }}                    # Release name
{{ .Release.Namespace }}               # Namespace
{{ .Chart.Name }}                      # Chart name
{{ .Chart.Version }}                   # Chart version
{{ .Capabilities.KubeVersion.Major }}  # K8s version
```

### Question Coverage Index
| Q# | Topic | Difficulty |
|---|---|---|
| Q1 | Helm introduction & workflow | 🟢 |
| Q2 | Chart, Release, Revision concepts | 🟢 |
| Q3 | Install, upgrade, rollback, uninstall | 🟢 |
| Q4 | Values and precedence | 🟡 |
| Q5 | Chart structure | 🟢 |
| Q6 | Go templating | 🟡 |
| Q7 | Template functions & pipelines | 🟡 |
| Q8 | Named templates: include vs template | 🔴 |
| Q9 | Repository management | 🟢 |
| Q10 | Chart dependencies | 🟡 |
| Q11 | Helm hooks | 🟡 |
| Q12 | Helm tests | 🟡 |
| Q13 | Rollback deep dive | 🟡 |
| Q14 | Essential plugins | 🟡 |
| Q15 | Create and publish charts + OCI | 🔴 |
| Q16 | values.schema.json validation | 🔴 |
| Q17 | Post-renderer & Kustomize | 🔴 |
| Q18 | Helmfile introduction | 🟢 |
| Q19 | helmfile.yaml full structure | 🟡 |
| Q20 | Selectors, diff, lock | 🟡 |
| Q21 | Environments & secrets (SOPS) | 🔴 |
