# Kubernetes Deployment Strategies — Complete Study Guide
### All Topics · Detailed Q&A · Master Cheatsheet
**2025 Edition — Comprehensive coverage with examples, commands & cheatsheet**

---

## Table of Contents
- [Deployment Fundamentals](#deployment-fundamentals)
- [Recreate Strategy](#recreate)
- [Rolling Updates](#rolling-updates)
- [HPA Interaction with Deployments](#hpa)
- [Canary Deployments](#canary)
- [Blue/Green Deployments](#blue-green)
- [A/B Testing](#ab-testing)
- [Shadow / Traffic Mirroring](#shadow)
- [Feature Flags](#feature-flags)
- [Argo Rollouts](#argo-rollouts)
- [Zero-Downtime Deployments](#zero-downtime)
- [Multi-Region Deployment Strategies](#multi-region)
- [Master Cheatsheet](#master-cheatsheet)

---

## Deployment Fundamentals

### 🟢 Q1. What is a Kubernetes Deployment and what does it manage?

```yaml
# Full Deployment spec
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    version: v1
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubernetes.io/change-cause: "Release v1.2.3 - add login feature"
spec:
  replicas: 5
  revisionHistoryLimit: 5           # Keep last 5 ReplicaSets for rollback
  progressDeadlineSeconds: 600      # Fail deployment if not complete in 10 min
  selector:
    matchLabels:
      app: my-app                   # MUST match template labels
  strategy:
    type: RollingUpdate             # or Recreate
    rollingUpdate:
      maxSurge: 1                   # Extra pods during update
      maxUnavailable: 0             # Zero downtime (no pods unavailable)

  template:
    metadata:
      labels:
        app: my-app
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      # Graceful termination
      terminationGracePeriodSeconds: 60

      # Spread pods across nodes/zones
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app

      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: my-app
                topologyKey: kubernetes.io/hostname

      containers:
        - name: my-app
          image: registry.example.com/my-app:v1.2.3
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          env:
            - name: APP_ENV
              value: production
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]  # Drain in-flight requests
```

```bash
# Deployment commands
kubectl apply -f deployment.yaml
kubectl rollout status deployment/my-app -n production
kubectl rollout history deployment/my-app -n production
kubectl rollout undo deployment/my-app -n production
kubectl rollout undo deployment/my-app --to-revision=3 -n production
kubectl rollout pause deployment/my-app -n production     # Pause mid-rollout
kubectl rollout resume deployment/my-app -n production    # Resume
kubectl scale deployment/my-app --replicas=10 -n production
kubectl set image deployment/my-app my-app=myapp:v2 -n production
kubectl get deployment my-app -o wide
kubectl describe deployment my-app -n production
```

---

### 🟡 Q2. What are PodDisruptionBudgets and why are they important?

```yaml
# PDB ensures minimum availability during voluntary disruptions
# (node drain, cluster upgrades, Eviction API calls)

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: production
spec:
  # Choose ONE of:
  minAvailable: 3        # At least 3 pods must be available
  # OR
  maxUnavailable: 1      # At most 1 pod can be unavailable at a time
  # OR
  minAvailable: "75%"    # At least 75% of pods

  selector:
    matchLabels:
      app: my-app
```

```bash
# Check PDB status
kubectl get pdb -n production
# NAME        MIN AVAILABLE  MAX UNAVAILABLE  ALLOWED DISRUPTIONS  AGE
# my-app-pdb  3              N/A              2                    1d

# Drain node (respects PDB)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
# If PDB would be violated: error: cannot evict pod ...

# Force drain (ignores PDB — USE WITH CAUTION)
kubectl drain node-1 --ignore-daemonsets --force
```

---

## Recreate Strategy

### 🟢 Q3. What is the Recreate deployment strategy?

```yaml
# Recreate: stop ALL old pods, then start ALL new pods
# Results in DOWNTIME — use only for stateful apps that can't run two versions

spec:
  strategy:
    type: Recreate
    # No rollingUpdate block for Recreate
```

```
Timeline:
  t=0: v1 pods running (5 replicas)
  t=1: ALL v1 pods terminated (DOWNTIME STARTS)
  t=2: ALL v2 pods starting
  t=3: ALL v2 pods running (DOWNTIME ENDS)

Use cases:
  - Database schema migrations requiring single version
  - Apps with incompatible state between versions
  - Development environments
  - Apps that use shared file locks
```

---

## Rolling Updates

### 🟡 Q4. How do you perform and control rolling updates?

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Max extra pods (can be % or absolute)
      maxUnavailable: 0     # Never take pods offline during update
      # maxSurge: "25%"     # 25% extra pods
      # maxUnavailable: "25%" # 25% can be unavailable
```

```
Timeline (5 replicas, maxSurge=1, maxUnavailable=0):
  t=0: [v1,v1,v1,v1,v1] — 5 v1 pods
  t=1: [v1,v1,v1,v1,v1,v2] — 6 pods (surge)
  t=2: [v1,v1,v1,v1,v2] — old pod removed once new is ready
  t=3: [v1,v1,v1,v2,v2]
  t=4: [v1,v1,v2,v2,v2]
  t=5: [v1,v2,v2,v2,v2]
  t=6: [v2,v2,v2,v2,v2] — done
```

```bash
# Trigger rolling update
kubectl set image deployment/my-app my-app=myapp:v2 -n production

# Watch rollout progress
kubectl rollout status deployment/my-app -n production

# Pause rollout (canary check)
kubectl rollout pause deployment/my-app

# Check new pods' health manually
kubectl get pods -l app=my-app -n production

# Resume or abort
kubectl rollout resume deployment/my-app
kubectl rollout undo deployment/my-app    # Abort — go back

# Record change cause (for rollout history)
kubectl annotate deployment/my-app \
  kubernetes.io/change-cause="v2.0.0: add search feature" \
  -n production
```

---

## HPA Interaction with Deployments

### 🟡 Q5. How does HPA interact with deployments during updates?

```yaml
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
  minReplicas: 3
  maxReplicas: 20
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
          type: Utilization
          averageUtilization: 80
    # Custom metric
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300     # Wait 5 min before scale down
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60               # Remove max 2 pods per minute
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15              # Double pods every 15s if needed
        - type: Pods
          value: 5
          periodSeconds: 15
      selectPolicy: Max                  # Use the policy that allows more scale-up
```

```bash
# HPA + Deployment interaction during rollout:
# - HPA controls replicas based on metrics
# - Rolling update replaces pods while HPA manages total count
# - Set Deployment.spec.replicas = HPA.spec.minReplicas to avoid conflicts

# Check HPA status
kubectl get hpa -n production
kubectl describe hpa my-app-hpa -n production

# Temporarily disable HPA scaling during maintenance
kubectl patch hpa my-app-hpa -n production \
  -p '{"spec":{"minReplicas":5,"maxReplicas":5}}'

# KEDA — event-driven autoscaling (more advanced than HPA)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-app-keda
spec:
  scaleTargetRef:
    name: my-app
  minReplicaCount: 0           # Scale to zero when idle!
  maxReplicaCount: 50
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_requests_total
        threshold: "100"
        query: sum(rate(http_requests_total{app="my-app"}[2m]))
```

---

## Canary Deployments

### 🟡 Q6. How do you implement canary deployments in Kubernetes?

```yaml
# ===== BASIC CANARY (label-based traffic split) =====
# Step 1: Existing stable deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
spec:
  replicas: 9                          # 90% of traffic
  selector:
    matchLabels:
      app: my-app
      track: stable
  template:
    metadata:
      labels:
        app: my-app
        track: stable
    spec:
      containers:
        - name: my-app
          image: myapp:v1

---
# Step 2: Canary deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1                          # 10% of traffic (1/10 pods)
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
    spec:
      containers:
        - name: my-app
          image: myapp:v2              # NEW VERSION

---
# Service selects BOTH (uses app: my-app — no track label)
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app                        # Matches both stable + canary
  ports:
    - port: 80
      targetPort: 8080

# Traffic split is by pod count:
# 9 stable + 1 canary = 10% to canary
# To increase: kubectl scale deployment my-app-canary --replicas=3
# (30% canary)
```

```yaml
# ===== NGINX INGRESS CANARY =====
# Traffic split by percentage (not pod count)

# Stable ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-stable
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-stable
                port: { number: 80 }

---
# Canary ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"   # 20% of traffic
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-canary
                port: { number: 80 }
```

### 🔴 Q7. How do you implement header-based canary routing?

```yaml
# Route specific users to canary (based on header/cookie)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    # Route based on header
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
    # OR route based on cookie
    nginx.ingress.kubernetes.io/canary-by-cookie: "canary_user"
    # Cookie value: always = route to canary, never = route to stable
```

```bash
# Test canary with header
curl -H "X-Canary: true" https://app.example.com/api/status

# Gradually increase canary traffic
kubectl annotate ingress my-app-canary \
  nginx.ingress.kubernetes.io/canary-weight="50" --overwrite

# Promote canary to stable
# 1. Update stable deployment to new image
kubectl set image deployment/my-app-stable my-app=myapp:v2

# 2. Wait for rollout
kubectl rollout status deployment/my-app-stable

# 3. Remove canary
kubectl delete deployment my-app-canary
kubectl delete ingress my-app-canary
```

---

## Blue/Green Deployments

### 🟡 Q8. How do you implement blue/green deployments?

```yaml
# Blue = current production
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
  labels:
    app: my-app
    slot: blue
spec:
  replicas: 5
  selector:
    matchLabels:
      app: my-app
      slot: blue
  template:
    metadata:
      labels:
        app: my-app
        slot: blue
    spec:
      containers:
        - name: my-app
          image: myapp:v1

---
# Green = new version (already deployed, not receiving traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
  labels:
    app: my-app
    slot: green
spec:
  replicas: 5
  selector:
    matchLabels:
      app: my-app
      slot: green
  template:
    metadata:
      labels:
        app: my-app
        slot: green
    spec:
      containers:
        - name: my-app
          image: myapp:v2               # NEW VERSION — pre-warmed

---
# Service points to BLUE initially
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    slot: blue                          # Currently pointing to blue
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Switch traffic from blue to green (instant, zero downtime)
kubectl patch service my-app \
  -p '{"spec":{"selector":{"app":"my-app","slot":"green"}}}'

# Verify traffic is on green
kubectl get service my-app -o jsonpath='{.spec.selector}'

# Monitor green — check error rates, latency
# If green is good: delete blue
kubectl delete deployment my-app-blue

# If green has issues: instant rollback to blue
kubectl patch service my-app \
  -p '{"spec":{"selector":{"app":"my-app","slot":"blue"}}}'

# Automate with script
CURRENT=$(kubectl get service my-app -o jsonpath='{.spec.selector.slot}')
TARGET=$([[ $CURRENT == "blue" ]] && echo "green" || echo "blue")

echo "Switching from $CURRENT to $TARGET"

# Verify target deployment is ready
kubectl wait deployment/my-app-$TARGET \
  --for=condition=Available \
  --timeout=300s

# Switch
kubectl patch service my-app \
  -p "{\"spec\":{\"selector\":{\"app\":\"my-app\",\"slot\":\"$TARGET\"}}}"

echo "Traffic now on $TARGET"
```

---

## A/B Testing

### 🟡 Q9. How do you implement A/B testing in Kubernetes?

```yaml
# A/B testing = route users to different versions based on
# user attributes (not just percentage) — for product experiments

# Approach 1: Header/cookie based (Nginx Ingress)
# Version A = stable
# Version B = experimental (different feature, different UI)

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-b
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-cookie: "ab_test_group"
    # Users with cookie ab_test_group=always → version B
    # Users with cookie ab_test_group=never → version A
    # Others → version A

# Set cookie via feature flag service on first visit
# e.g. LaunchDarkly, Unleash, or custom

---
# Approach 2: Istio VirtualService (precise A/B)
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
    - my-app
  http:
    # Users in beta group
    - match:
        - headers:
            x-user-group:
              exact: beta
      route:
        - destination:
            host: my-app
            subset: v2
    # Premium users get version B
    - match:
        - headers:
            x-subscription:
              exact: premium
      route:
        - destination:
            host: my-app
            subset: v2
    # Everyone else gets v1
    - route:
        - destination:
            host: my-app
            subset: v1
```

---

## Shadow / Traffic Mirroring

### 🔴 Q10. What are shadow deployments and traffic mirroring?

```yaml
# Mirror production traffic to a shadow service for testing
# Shadow receives copy of all requests but responses are DISCARDED
# Users never see shadow responses

# Approach 1: Istio traffic mirroring
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
    - my-app
  http:
    - route:
        - destination:
            host: my-app-stable
            port:
              number: 80
          weight: 100
      mirror:
        host: my-app-shadow           # Mirror to shadow
        port:
          number: 80
      mirrorPercentage:
        value: 100.0                  # Mirror 100% of traffic
        # Or: value: 20.0 for 20% mirroring

---
# Shadow deployment (runs new version, receives mirrored traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-shadow
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
      track: shadow
  template:
    metadata:
      labels:
        app: my-app
        track: shadow
    spec:
      containers:
        - name: my-app
          image: myapp:v2-beta
          # Separate observability (don't mix metrics with prod)
          env:
            - name: APP_ENV
              value: shadow
```

```bash
# Nginx Ingress mirroring
kubectl annotate ingress my-app \
  nginx.ingress.kubernetes.io/mirror-target="http://my-app-shadow" \
  nginx.ingress.kubernetes.io/mirror-host="my-app-shadow.production.svc.cluster.local"

# Use cases:
# - Test v2 with real production load without risk
# - Verify new ML model with live traffic
# - Validate performance characteristics
# - Test database schema changes
```

---

## Feature Flags

### 🟡 Q11. How do feature flags integrate with Kubernetes deployments?

```yaml
# Approach 1: Feature flag via environment variable (simple)
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: my-app
          env:
            - name: FEATURE_NEW_CHECKOUT
              valueFrom:
                configMapKeyRef:
                  name: feature-flags
                  key: new_checkout_enabled

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
data:
  new_checkout_enabled: "false"
  new_search_enabled: "true"
  dark_mode_enabled: "false"
```

```bash
# Toggle feature without restart
kubectl patch configmap feature-flags \
  -p '{"data":{"new_checkout_enabled":"true"}}'
# Note: App must watch for configmap changes or pods must restart

# Force reload
kubectl rollout restart deployment/my-app
```

```yaml
# Approach 2: Unleash (open-source feature flag service)
# In-cluster Unleash deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unleash
spec:
  template:
    spec:
      containers:
        - name: unleash
          image: unleashorg/unleash-server:latest
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: unleash-db
                  key: url

# App SDK usage (Node.js example)
# const unleash = require('unleash-client');
# unleash.initialize({ url: 'http://unleash/api', appName: 'my-app' });
# if (unleash.isEnabled('new-checkout')) { ... }
```

---

## Argo Rollouts

### 🔴 Q12. What is Argo Rollouts and how does it improve deployments?

```bash
# Install Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install kubectl plugin
brew install argoproj/tap/kubectl-argo-rollouts
```

```yaml
# Rollout resource (replaces Deployment)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 5
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: myapp:v1
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: 200m, memory: 256Mi }
            limits: { cpu: 500m, memory: 512Mi }

  # ===== CANARY STRATEGY =====
  strategy:
    canary:
      canaryService: my-app-canary   # Points to canary pods
      stableService: my-app-stable   # Points to stable pods
      trafficRouting:
        nginx:
          stableIngress: my-app-ingress  # Nginx ingress to control weight
      steps:
        - setWeight: 5                # Step 1: 5% traffic to canary
        - pause: { duration: 5m }     # Wait 5 minutes
        - setWeight: 20
        - pause:
            duration: 10m
        - analysis:                   # Run automated analysis
            templates:
              - templateName: success-rate
            args:
              - name: service-name
                value: my-app-canary
        - setWeight: 50
        - pause: {}                   # Manual gate (indefinite pause)
        - setWeight: 100

      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}

---
# Analysis Template — automated promotion/rollback criteria
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      count: 5                       # Run 5 times
      successCondition: result[0] >= 0.95   # 95% success rate
      failureLimit: 2                # Fail if metric fails 2 times
      provider:
        prometheus:
          address: http://prometheus-server.monitoring:80
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))

    - name: latency-p99
      interval: 1m
      successCondition: result[0] <= 0.5    # p99 < 500ms
      provider:
        prometheus:
          address: http://prometheus-server.monitoring:80
          query: |
            histogram_quantile(0.99, sum(rate(
              http_request_duration_seconds_bucket{service="{{args.service-name}}"}[2m]
            )) by (le))
```

```yaml
# Blue/Green with Argo Rollouts
strategy:
  blueGreen:
    activeService: my-app-active       # Production traffic
    previewService: my-app-preview     # New version (no traffic)
    autoPromotionEnabled: false        # Manual promotion gate
    prePromotionAnalysis:
      templates:
        - templateName: success-rate
      args:
        - name: service-name
          value: my-app-preview
    postPromotionAnalysis:
      templates:
        - templateName: success-rate
      args:
        - name: service-name
          value: my-app-active
    scaleDownDelaySeconds: 600         # Keep old version 10 min after promotion
```

```bash
# Argo Rollouts commands
kubectl argo rollouts get rollout my-app -n production --watch

# Promote to next step (unblock manual pause)
kubectl argo rollouts promote my-app -n production

# Abort rollout (rollback)
kubectl argo rollouts abort my-app -n production

# Retry after abort
kubectl argo rollouts retry rollout my-app -n production

# Set image
kubectl argo rollouts set image my-app my-app=myapp:v2 -n production

# Dashboard (web UI)
kubectl argo rollouts dashboard -n production
# Open http://localhost:3100
```

---

## Zero-Downtime Deployments

### 🔴 Q13. How do you implement zero-downtime deployments with lifecycle hooks?

```yaml
# Zero-downtime checklist:
# 1. readinessProbe — pod not added to service until ready
# 2. preStop hook — drain in-flight requests before pod terminates
# 3. terminationGracePeriodSeconds — give app time to drain
# 4. maxUnavailable: 0 — no pods down during rollout
# 5. PodDisruptionBudget — protect during node drains
# 6. Anti-affinity — spread pods across nodes/zones

apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0            # Critical: no downtime
  template:
    spec:
      terminationGracePeriodSeconds: 90   # App has 90s to finish requests
      containers:
        - name: app
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 3
            successThreshold: 2        # Must pass twice before marking ready
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                # 1. Signal app to stop accepting new connections
                # 2. Sleep for load balancer propagation delay
                # 3. App finishes in-flight requests during sleep
                command:
                  - /bin/sh
                  - -c
                  - |
                    # Signal our app to start draining
                    kill -SIGTERM 1
                    # Wait for load balancer to stop routing to this pod
                    sleep 15
                    # App should finish remaining requests during terminationGracePeriodSeconds
```

```bash
# Test zero-downtime deployment
# During a rolling update, run continuous HTTP test

# Start rolling update
kubectl set image deployment/my-app my-app=myapp:v2 &

# Continuously check for errors
for i in $(seq 1 1000); do
  response=$(curl -s -o /dev/null -w "%{http_code}" https://app.example.com/api/ping)
  if [ "$response" != "200" ]; then
    echo "ERROR at iteration $i: HTTP $response"
  fi
  sleep 0.1
done
```

---

## Multi-Region Deployment Strategies

### 🔴 Q14. What are multi-region deployment strategies for Kubernetes?

```yaml
# ===== ACTIVE-ACTIVE (traffic split across regions) =====
# Use Global Load Balancer (AWS Route53, Google Cloud DNS, Cloudflare)

# Route53 weighted routing
# region-us: weight 50
# region-eu: weight 50

# ArgoCD ApplicationSet for multi-cluster deploy
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-multiregion
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
        repoURL: https://github.com/org/my-app
        targetRevision: HEAD
        path: k8s/production
        helm:
          values: |
            region: {{metadata.labels.region}}
            replicaCount: {{metadata.annotations.replicaCount}}
      destination:
        server: "{{server}}"
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true

---
# ===== ACTIVE-PASSIVE (failover) =====
# Primary region active, secondary on standby (0 replicas)
# Failover: increase replicas in secondary, cut DNS to primary

# Route53 failover routing
# Primary: region-us (active)
# Secondary: region-eu (passive — health check fails → failover)

---
# ===== PROGRESSIVE REGION ROLLOUT =====
# Deploy to one region, validate, then next region

# Step 1: Deploy to us-east-1
kubectl config use-context eks-us-east-1
helm upgrade --install my-app ./chart -f values-prod-us-east.yaml \
  --set image.tag=$NEW_TAG --atomic

# Step 2: Run smoke tests
./run-smoke-tests.sh https://us-east.app.example.com

# Step 3: Deploy to eu-west-1
kubectl config use-context eks-eu-west-1
helm upgrade --install my-app ./chart -f values-prod-eu-west.yaml \
  --set image.tag=$NEW_TAG --atomic

# Step 4: Deploy to ap-southeast-1
kubectl config use-context eks-ap-southeast-1
helm upgrade --install my-app ./chart -f values-prod-ap.yaml \
  --set image.tag=$NEW_TAG --atomic
```

---

## Master Cheatsheet

### Strategy Comparison
```
Recreate:
  Downtime: YES         Speed: Fast    Risk: Low rollback complexity
  Use: DB migrations, dev environments

Rolling Update (default):
  Downtime: NO          Speed: Gradual  Risk: Old+new coexist
  Use: Stateless apps, backward-compatible changes

Canary:
  Downtime: NO          Speed: Gradual  Risk: Limited blast radius
  Use: Test new version with subset of users

Blue/Green:
  Downtime: NO (fast)   Speed: Instant  Risk: Need 2x resources
  Use: Zero-downtime, instant rollback

A/B Testing:
  Downtime: NO          Speed: Gradual  Risk: Same as canary
  Use: Product experiments, user segmentation

Shadow:
  Downtime: NO          Speed: Parallel Risk: None (discarded responses)
  Use: Test new version with real traffic, no user impact
```

### Key Commands
```bash
# Rollout management
kubectl rollout status deployment/app -n ns
kubectl rollout history deployment/app -n ns
kubectl rollout undo deployment/app -n ns
kubectl rollout undo deployment/app --to-revision=3 -n ns
kubectl rollout pause/resume deployment/app -n ns

# Scale
kubectl scale deployment/app --replicas=5 -n ns
kubectl autoscale deployment/app --min=3 --max=10 --cpu-percent=70

# Image update
kubectl set image deployment/app app=image:v2 -n ns

# Argo Rollouts
kubectl argo rollouts get rollout app -n ns --watch
kubectl argo rollouts promote app -n ns
kubectl argo rollouts abort app -n ns

# PDB
kubectl get pdb -n ns
kubectl describe pdb app-pdb -n ns
```

### Question Coverage Index
| Q# | Topic | Difficulty |
|---|---|---|
| Q1 | Deployment fundamentals | 🟢 |
| Q2 | PodDisruptionBudget | 🟡 |
| Q3 | Recreate strategy | 🟢 |
| Q4 | Rolling updates | 🟡 |
| Q5 | HPA interaction | 🟡 |
| Q6 | Canary deployments | 🟡 |
| Q7 | Header-based canary routing | 🔴 |
| Q8 | Blue/Green deployments | 🟡 |
| Q9 | A/B testing | 🟡 |
| Q10 | Shadow / traffic mirroring | 🔴 |
| Q11 | Feature flags | 🟡 |
| Q12 | Argo Rollouts | 🔴 |
| Q13 | Zero-downtime lifecycle hooks | 🔴 |
| Q14 | Multi-region strategies | 🔴 |
