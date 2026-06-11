# Kubernetes Troubleshooting — Complete Guide
> Covers: Vanilla K8s · AKS · EKS · GKE  
> Scope: Cluster Level → Control Plane → Nodes → Pods → Deployments → StatefulSets → DaemonSets → Jobs → Services → Ingress → NetworkPolicy → Storage → RBAC → HPA/VPA → Admission Webhooks → etcd/Certificates  
> Format: Symptom → Diagnosis → Fix → Prevention

---

## 🗺️ Troubleshooting Decision Tree

```
Something is broken
        │
        ├─► Is the cluster reachable? (kubectl get nodes)
        │         │
        │         ├─ NO  → Section 1: Control Plane / API Server
        │         └─ YES → Continue
        │
        ├─► Are nodes healthy? (kubectl get nodes)
        │         │
        │         ├─ NotReady  → Section 2: Node Troubleshooting
        │         └─ Ready     → Continue
        │
        ├─► Is the pod running? (kubectl get pods)
        │         │
        │         ├─ Pending        → Section 3A: Pending Pods
        │         ├─ CrashLoopBackOff → Section 3B: CrashLoopBackOff
        │         ├─ ImagePullBackOff → Section 3C: Image Issues
        │         ├─ OOMKilled      → Section 3D: OOMKilled
        │         ├─ Error/Failed   → Section 3E: Error States
        │         └─ Running but broken → Continue
        │
        ├─► Can traffic reach the pod? (curl, wget)
        │         │
        │         ├─ Service not reachable → Section 4: Services
        │         ├─ Ingress not working   → Section 5: Ingress
        │         ├─ DNS failing           → Section 6: DNS/CoreDNS
        │         └─ Pod-to-pod broken     → Section 7: NetworkPolicy
        │
        ├─► Is storage working?
        │         └─ PVC stuck / mount fails → Section 8: Storage
        │
        ├─► Is autoscaling working?
        │         └─ HPA/VPA/CA not scaling → Section 9: Autoscaling
        │
        ├─► RBAC / Permission denied?
        │         └─ Forbidden errors → Section 10: RBAC
        │
        └─► Workload-specific issues
                  ├─ StatefulSet   → Section 11
                  ├─ DaemonSet    → Section 12
                  ├─ Jobs/CronJobs → Section 13
                  ├─ Helm         → Section 14
                  └─ Admission Webhook → Section 15
```

---

## 🔧 Universal First-Response Commands

```bash
# ── Always start here ──────────────────────────────────────────
kubectl get nodes                                   # node health
kubectl get pods -A --field-selector=status.phase!=Running   # all non-running pods
kubectl get events -A --sort-by='.lastTimestamp' | tail -50  # recent events
kubectl top nodes                                   # node resource usage
kubectl top pods -A --containers                    # pod resource usage

# Describe any object
kubectl describe <resource> <name> -n <namespace>

# Logs
kubectl logs <pod> -n <ns> --previous              # last crashed container
kubectl logs <pod> -n <ns> -c <container> -f       # follow specific container
kubectl logs <pod> -n <ns> --tail=100              # last 100 lines

# Quick shell into a running pod
kubectl exec -it <pod> -n <ns> -- /bin/sh

# Ephemeral debug container (non-destructive)
kubectl debug -it <pod> -n <ns> \
  --image=nicolaka/netshoot \
  --target=<container-name>

# Debug a node directly
kubectl debug node/<node-name> -it --image=busybox
```

---

## 1. Control Plane / API Server Issues

### 1.1 `kubectl` — Cannot connect / timeout

```
Error: The connection to the server was refused
Error: Unable to connect to the server: dial tcp: i/o timeout
```

**Diagnosis:**
```bash
# Check kubeconfig context
kubectl config current-context
kubectl config get-contexts

# Verify API server endpoint is reachable
curl -k https://<api-server-endpoint>/healthz

# Check certificate expiry
kubectl get csr
openssl s_client -connect <api-server>:6443 2>/dev/null | openssl x509 -noout -dates
```

**AKS fix:**
```bash
az aks get-credentials --resource-group myRG --name myCluster --overwrite-existing
# For private cluster use command invoke
az aks command invoke -g myRG -n myCluster --command "kubectl get nodes"
```

**EKS fix:**
```bash
aws eks update-kubeconfig --name myCluster --region us-east-1
# Check endpoint access mode
aws eks describe-cluster --name myCluster \
  --query "cluster.resourcesVpcConfig.{pub:endpointPublicAccess,priv:endpointPrivateAccess}"
# If public=false, you must be inside VPC
```

**GKE fix:**
```bash
gcloud container clusters get-credentials myCluster --region us-central1
# Enable authorized networks temporarily
gcloud container clusters update myCluster --region us-central1 \
  --master-authorized-networks $(curl -s ifconfig.me)/32
```

---

### 1.2 API Server — Slow / 429 Too Many Requests

```bash
# Check API server latency
kubectl get --raw /metrics | grep apiserver_request_duration

# Find who is hammering the API
kubectl get --raw /metrics | grep apiserver_request_total \
  | sort -t'"' -k4 -rn | head -20

# Check audit logs (AKS)
az monitor diagnostic-settings list --resource <aks-resource-id>
# Look for high-frequency callers in kube-audit

# Reduce API server load
# 1. Add rate limits to controllers
# 2. Reduce controller resync periods
# 3. Scale up API server replicas (AKS/EKS/GKE do this automatically)
```

---

### 1.3 etcd — Health Issues (self-managed K8s)

```bash
# Check etcd health
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check etcd member list
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# etcd disk latency (should be <10ms)
ETCDCTL_API=3 etcdctl check perf \
  --endpoints=https://127.0.0.1:2379 ...

# Compact and defrag etcd (if DB size is large)
ETCDCTL_API=3 etcdctl compact $(etcdctl endpoint status --write-out="json" | \
  egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')
ETCDCTL_API=3 etcdctl defrag --endpoints=https://127.0.0.1:2379 ...

# AKS/EKS/GKE: etcd is fully managed — you don't directly access it
```

---

### 1.4 Certificate Expiry (self-managed / kubeadm)

```bash
# Check all certificate expiry dates
kubeadm certs check-expiration

# Renew all certificates (requires kubeadm)
kubeadm certs renew all

# Restart control-plane components after renewal
systemctl restart kubelet

# AKS/EKS/GKE: certificates managed automatically — no action needed
# AKS: certificates auto-rotated; check via:
az aks show --name myCluster -g myRG --query "enableRBAC"
```

---

## 2. Node Troubleshooting

### 2.1 Node — `NotReady`

**Diagnosis flow:**
```bash
# Step 1: Get node conditions
kubectl describe node <node-name>
# Look for: MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable, Ready

# Step 2: SSH into node (if possible) and check kubelet
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager

# Step 3: Common root causes and fixes:
```

| Condition | Root Cause | Fix |
|-----------|-----------|-----|
| `MemoryPressure` | Node RAM exhausted | Evict low-priority pods; add nodes |
| `DiskPressure` | Node disk full | Remove unused images; increase disk |
| `PIDPressure` | Too many processes | Check for fork bombs; increase pid limit |
| `NetworkUnavailable` | CNI plugin down | Restart CNI DaemonSet |
| `NotReady` (no conditions) | kubelet dead | `systemctl restart kubelet` |

```bash
# Fix: kubelet not running
ssh <node>
sudo systemctl start kubelet
sudo systemctl enable kubelet

# Fix: disk pressure — clean up images
ssh <node>
sudo crictl rmi --prune          # remove unused container images
sudo journalctl --vacuum-size=1G # clean old journal logs
df -h                            # verify space freed

# Fix: network plugin down
kubectl rollout restart daemonset/aws-node -n kube-system          # EKS VPC CNI
kubectl rollout restart daemonset/cilium -n kube-system             # Cilium
kubectl rollout restart daemonset/calico-node -n kube-system        # Calico

# Cordon + drain + delete stuck node
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force
kubectl delete node <node>
```

**AKS:**
```bash
# AKS auto-repair: if node is NotReady >10 min, Azure reimages it
# Check auto-repair events
az aks nodepool show -g myRG --cluster-name myCluster -n nodepool1 \
  --query "upgradeSettings"

# Manually trigger node reimage
az aks nodepool upgrade -g myRG --cluster-name myCluster \
  -n nodepool1 --node-image-only
```

**EKS:**
```bash
# Check EC2 instance health
aws ec2 describe-instance-status --instance-ids i-0123456789abcdef0

# Terminate + replace node (ASG will replace)
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0

# Check bootstrap logs (cloud-init)
aws ec2 get-console-output --instance-id i-xxx | tail -50
```

**GKE:**
```bash
# GKE auto-repairs NotReady nodes automatically
# View repair operations
gcloud container operations list --filter="operationType=REPAIR_CLUSTER"

# Manually repair a node pool
gcloud container node-pools update primary-pool \
  --cluster myCluster --region us-central1 --enable-autorepair
```

---

### 2.2 Node — Resource Pressure / Eviction

```bash
# See which pods were evicted
kubectl get events -A | grep Evicted
kubectl get pods -A --field-selector=status.phase=Failed | grep Evicted

# Check eviction thresholds
kubectl describe node <node> | grep -A20 "Conditions:"
# memory.available < 100Mi → MemoryPressure
# nodefs.available < 10%   → DiskPressure

# Find largest images on node (via debug pod)
kubectl debug node/<node> -it --image=busybox -- sh
# df -h
# du -sh /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/

# Prevention: set proper requests/limits on all pods
kubectl get pods -A -o json | jq -r \
  '.items[] | select(.spec.containers[].resources.requests == null) | 
   "\(.metadata.namespace)/\(.metadata.name)"'
```

---

### 2.3 Node — Too Many Pods / IP Exhaustion

```bash
# Check pod count per node
kubectl get pods -A -o wide | awk '{print $8}' | sort | uniq -c | sort -rn

# Check max pods setting
kubectl describe node <node> | grep "Allocatable" -A10
# Capacity.pods vs Allocatable.pods

# EKS: VPC IP exhaustion
kubectl describe node <node> | grep vpc.amazonaws.com
# Fix: enable prefix delegation
kubectl set env daemonset aws-node \
  ENABLE_PREFIX_DELEGATION=true \
  WARM_PREFIX_TARGET=1 \
  -n kube-system

# AKS: increase max pods per node (only at node pool creation)
az aks nodepool add --max-pods 110 ...

# GKE: increase max pods per node
gcloud container node-pools create bigger-pool \
  --max-pods-per-node 110 ...
```

---

## 3. Pod Troubleshooting

### 3A. Pod — `Pending`

**Decision tree:**
```bash
kubectl describe pod <pod> -n <ns>
# Read the Events section carefully
```

| Event Message | Root Cause | Fix |
|--------------|-----------|-----|
| `Insufficient cpu/memory` | No node has capacity | Add nodes / reduce requests |
| `0/N nodes available: N Insufficient nvidia.com/gpu` | No GPU nodes | Add GPU node pool |
| `node(s) had taint ... pod did not tolerate` | Missing toleration | Add toleration to pod spec |
| `0/N nodes matched pod topology spread constraints` | Topology constraint unsatisfiable | Relax `maxSkew` or add nodes |
| `persistentvolumeclaim "x" not found` | PVC doesn't exist | Create PVC first |
| `pod has unbound PersistentVolumeClaims` | PVC not bound to PV | Check StorageClass / PVC status |
| `exceeded quota` | ResourceQuota hit | Increase quota or reduce requests |
| `No preemption victims found` | All pods same priority | Set PriorityClass |

```bash
# Detailed scheduling failure analysis
kubectl get events -n <ns> --field-selector involvedObject.name=<pod>

# Check if resource quota is blocking
kubectl describe resourcequota -n <ns>
kubectl describe limitrange -n <ns>

# Check node taints
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Force-schedule for debugging (never in prod)
kubectl patch pod <pod> -n <ns> \
  -p '{"spec":{"tolerations":[{"operator":"Exists"}]}}'

# AKS: Cluster Autoscaler not scaling up?
kubectl get events -n kube-system | grep cluster-autoscaler
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml

# EKS: Karpenter not provisioning?
kubectl get nodeclaims
kubectl logs -n karpenter deployment/karpenter | grep -i error

# GKE: Node Auto-Provisioning not working?
gcloud container operations list --filter="operationType=CREATE_NODE_POOL"
```

---

### 3B. Pod — `CrashLoopBackOff`

```bash
# Step 1: Read the last crash logs
kubectl logs <pod> -n <ns> --previous
kubectl logs <pod> -n <ns> -c <container> --previous

# Step 2: Check exit code
kubectl describe pod <pod> -n <ns> | grep -A5 "Last State:"
# Exit code 0   → container completed (use Job, not Deployment)
# Exit code 1   → application error (check logs)
# Exit code 137 → OOMKilled (SIGKILL, memory limit exceeded)
# Exit code 139 → Segfault (SIGSEGV)
# Exit code 143 → Graceful termination (SIGTERM) timeout

# Step 3: Common fixes per exit code
```

**Exit code 1 — app crash:**
```bash
# Get more logs
kubectl logs <pod> --previous --tail=200

# Check environment variables
kubectl exec <pod> -- env | grep -i password
kubectl describe pod <pod> | grep -A20 "Environment:"

# Check mounted secrets/configmaps
kubectl exec <pod> -- cat /path/to/mounted/config

# Common: wrong DB connection string / missing env var
kubectl describe secret <secret-name> -n <ns>
```

**Exit code 137 — OOMKilled:**
```bash
# Confirm OOMKill
kubectl describe pod <pod> | grep -A5 "OOMKilled\|Exit Code"

# Check memory usage before death
kubectl top pod <pod> -n <ns> --containers

# Fix: increase memory limit
kubectl patch deployment <dep> -n <ns> \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"<c>","resources":{"limits":{"memory":"2Gi"}}}]}}}}'

# Or use VPA
kubectl get vpa <name> -n <ns> -o jsonpath='{.status.recommendation.containerRecommendations}'
```

**Liveness probe failing:**
```bash
kubectl describe pod <pod> | grep -A10 "Liveness\|Readiness\|Startup"
# "Liveness probe failed: Get http://...: connection refused"

# Debug: manually check health endpoint
kubectl exec <pod> -- wget -qO- http://localhost:8080/health

# Fix: increase initialDelaySeconds or failureThreshold
# Or add startupProbe to give slow apps time to start
```

---

### 3C. Pod — `ImagePullBackOff` / `ErrImagePull`

```bash
# Check the exact error
kubectl describe pod <pod> -n <ns>
# "Failed to pull image... 401 Unauthorized"
# "Failed to pull image... not found"
# "Failed to pull image... x509: certificate signed by unknown authority"
```

| Error | Cause | Fix |
|-------|-------|-----|
| `401 Unauthorized` | Missing/wrong pull secret | Create `imagePullSecret` |
| `not found` | Wrong image name/tag | Verify image exists in registry |
| `x509` error | Self-signed registry TLS | Add registry as insecure or add CA cert |
| `NameResolutionFailure` | Registry DNS unreachable | Check DNS / network policies |

```bash
# Fix 1: Create pull secret (generic registry)
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=me@example.com \
  -n <namespace>

# Add to pod/serviceaccount
kubectl patch serviceaccount default -n <ns> \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'

# AKS: attach ACR
az aks update -g myRG -n myCluster --attach-acr myACR
# Grants AcrPull role to kubelet MSI automatically

# EKS: ECR authentication (token expires every 12h)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com
# Or use managed add-on: attach ACR role to node group IAM role
# Verify: AmazonEC2ContainerRegistryReadOnly policy on NodeGroupRole

# GKE: Artifact Registry (auto-authenticated in same project)
# Cross-project: grant roles/artifactregistry.reader to node SA
gcloud projects add-iam-policy-binding SOURCE_PROJECT \
  --member "serviceAccount:NODE_SA@TARGET_PROJECT.iam.gserviceaccount.com" \
  --role roles/artifactregistry.reader

# Debug: test pull manually on node
kubectl debug node/<node> -it --image=busybox -- sh
# crictl pull myregistry.io/myimage:tag
```

---

### 3D. Pod — `OOMKilled`

```bash
# 1. Confirm OOM
kubectl describe pod <pod> -n <ns> | grep -B2 -A5 "OOMKilled"
# Container: app, Exit Code: 137, Reason: OOMKilled

# 2. Check what the pod was using at peak
kubectl top pod <pod> -n <ns> --containers

# 3. Look at VPA recommendation (if VPA installed)
kubectl get vpa <name> -n <ns> \
  -o jsonpath='{.status.recommendation.containerRecommendations[*]}'

# 4. Query Cloud Monitoring/CloudWatch/Azure Monitor for memory graph
# AKS:
az monitor metrics list \
  --resource /subscriptions/.../managedClusters/myCluster \
  --metric "node_memory_working_set_bytes"

# 5. Fix: increase limit
kubectl edit deployment <name> -n <ns>
# resources.limits.memory: "2Gi"  → "4Gi"

# 6. Prevention: always set requests AND limits
# requests < limits → guaranteed burst
# requests == limits → Guaranteed QoS class (never OOMKilled by kubelet)
```

---

### 3E. Pod — `CreateContainerConfigError`

```bash
# Symptom: pod stuck in Init phase or CreateContainerConfigError
kubectl describe pod <pod> -n <ns>
# "Error: secret "db-password" not found"
# "Error: configmap "app-config" not found"

# Fix: verify secret/configmap exists
kubectl get secret db-password -n <ns>
kubectl get configmap app-config -n <ns>

# Create missing secret
kubectl create secret generic db-password \
  --from-literal=password=mysecret \
  -n <ns>

# Check env var reference is correct
kubectl get pod <pod> -o jsonpath='{.spec.containers[*].env}'

# Common mistake: wrong namespace
kubectl get secret db-password -n production  # must be same NS as pod
```

---

### 3F. Pod — `Terminating` (stuck)

```bash
# Pod stuck in Terminating state — usually finalizer issue
kubectl describe pod <pod> -n <ns>
# Check Finalizers field

# Remove stuck finalizer
kubectl patch pod <pod> -n <ns> \
  -p '{"metadata":{"finalizers":null}}' \
  --type=merge

# Force delete (last resort — may cause data inconsistency)
kubectl delete pod <pod> -n <ns> --force --grace-period=0

# Find all pods with finalizers
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.metadata.finalizers != null) | 
  "\(.metadata.namespace)/\(.metadata.name): \(.metadata.finalizers)"'
```

---

## 4. Service Troubleshooting

### 4.1 Service — Not Reachable

```bash
# ── Check layer by layer ──────────────────────────────────────

# Layer 1: Does the Service exist and has correct selector?
kubectl get svc <svc> -n <ns>
kubectl describe svc <svc> -n <ns>
# Check: Port, TargetPort, Selector

# Layer 2: Does the Service have Endpoints?
kubectl get endpoints <svc> -n <ns>
# If ENDPOINTS = <none> → selector doesn't match any pods

# Debug selector mismatch
kubectl get pods -n <ns> -l <key>=<value>   # use same labels as service selector
kubectl get pods -n <ns> --show-labels

# Layer 3: Is the pod's container port correct?
kubectl describe pod <pod> -n <ns> | grep "Port:"
# Must match service.spec.ports.targetPort

# Layer 4: Test from inside cluster
kubectl run curl-test --image=curlimages/curl --rm -it -- \
  curl http://<svc>.<ns>.svc.cluster.local:<port>/health

# Layer 5: Check kube-proxy (iptables rules)
# SSH to node
iptables -t nat -L KUBE-SERVICES | grep <service-cluster-ip>
```

**Common Fixes:**
```bash
# Fix 1: Selector mismatch — align pod labels with service selector
kubectl label pod <pod> -n <ns> app=myapp  # add missing label

# Fix 2: Wrong targetPort
kubectl patch svc <svc> -n <ns> \
  -p '{"spec":{"ports":[{"port":80,"targetPort":8080}]}}'

# Fix 3: Service type LoadBalancer pending (no external IP)
kubectl get svc <svc> -n <ns>
# EXTERNAL-IP = <pending>

# AKS fix: check if Azure LB was created
az network lb list -g MC_myRG_myCluster_eastus

# EKS fix: check AWS LB Controller
kubectl get ingress,svc -A | grep LoadBalancer
kubectl logs -n kube-system deployment/aws-load-balancer-controller | tail -50

# GKE fix: check GCP LB
gcloud compute forwarding-rules list
gcloud compute backend-services get-health <backend-service> --global
```

---

### 4.2 Service — `ExternalTrafficPolicy: Local` Issues

```bash
# When externalTrafficPolicy=Local, traffic only routes to nodes with healthy pods
# Symptom: intermittent 504/connection refused

# Check which nodes have pod endpoints
kubectl get endpoints <svc> -n <ns> -o yaml
# addresses[].nodeName → only these nodes receive traffic

# Fix option 1: switch to Cluster policy
kubectl patch svc <svc> -n <ns> \
  -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'

# Fix option 2: ensure pods are spread across all nodes with topology spread
```

---

## 5. Ingress Troubleshooting

### 5.1 Ingress — 404 / 502 / 503

```bash
# ── Structured diagnosis ──────────────────────────────────────

# Step 1: Check Ingress resource
kubectl describe ingress <ingress> -n <ns>
# Look for: Address (LB IP), Rules, Backend services
# Events section: "Error getting service..."

# Step 2: Check Ingress Controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100
# Look for: "upstream not found", "no endpoints", "connection refused"

# Step 3: Verify backend service + endpoints
kubectl get endpoints <backend-svc> -n <ns>
# Must not be empty

# Step 4: Test routing from inside cluster
kubectl run test --image=curlimages/curl --rm -it -- \
  curl -H "Host: myapp.example.com" http://<ingress-controller-svc-ip>/

# Step 5: Check TLS certificate (if HTTPS)
echo | openssl s_client -connect myapp.example.com:443 2>/dev/null \
  | openssl x509 -noout -dates -subject
```

**NGINX Ingress specific:**
```bash
# Check nginx.conf was updated
kubectl exec -n ingress-nginx deployment/ingress-nginx-controller \
  -- nginx -T | grep myapp

# Reload nginx manually (if stuck)
kubectl exec -n ingress-nginx deployment/ingress-nginx-controller \
  -- nginx -s reload

# 502 Bad Gateway → backend pods not responding on targetPort
# 503 Service Unavailable → no healthy backends (endpoints empty)
# 404 Not Found → path/host rule doesn't match

# Check annotation issues
kubectl get ingress <ingress> -n <ns> -o yaml | grep annotations -A20
# Wrong ingressClass? Missing rewrite-target?
```

**AKS AGIC specific:**
```bash
# Check AGIC pod
kubectl get pods -n kube-system | grep ingress-appgw
kubectl logs -n kube-system -l app=ingress-appgw | tail -100

# Check Application Gateway health
az network application-gateway show-backend-health \
  --resource-group myRG \
  --name myAppGW

# Check backend pool
az network application-gateway address-pool list \
  --resource-group myRG \
  --gateway-name myAppGW
```

**EKS ALB Ingress specific:**
```bash
# Check AWS LB Controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller | grep -i error

# Check ALB target group health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...

# 403: check security group allows ALB → pods on port
aws ec2 describe-security-group-rules \
  --filters Name=group-id,Values=<sg-id>
```

**GKE GCLB Ingress specific:**
```bash
# Check backend service health
gcloud compute backend-services list
gcloud compute backend-services get-health <BACKEND_SERVICE> --global

# Check NEG status
gcloud compute network-endpoint-groups list

# 502: firewall blocking health check probes from GCP health check IPs
# GCP health check source: 35.191.0.0/16 and 130.211.0.0/22
gcloud compute firewall-rules create allow-gclb-health \
  --network my-vpc \
  --allow tcp \
  --source-ranges 35.191.0.0/16,130.211.0.0/22 \
  --target-tags gke-node
```

---

### 5.2 Ingress TLS Certificate Issues

```bash
# cert-manager: check Certificate status
kubectl get certificate -n <ns>
kubectl describe certificate <cert> -n <ns>
# Look for: "Not Ready", "Failed to obtain certificate"

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager | tail -50

# ClusterIssuer status
kubectl describe clusterissuer letsencrypt-prod

# Common issue 1: ACME HTTP-01 challenge failing (Ingress class mismatch)
kubectl get challenge -n <ns>
kubectl describe challenge <challenge> -n <ns>
# Fix: ensure Ingress controller serves /.well-known/acme-challenge/

# Common issue 2: cert already exists but expired
kubectl delete secret <tls-secret> -n <ns>
kubectl delete certificate <cert> -n <ns>
# cert-manager will re-issue

# Manual check
kubectl get secret <tls-secret> -n <ns> -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -dates
```

---

## 6. DNS / CoreDNS Troubleshooting

### 6.1 DNS Resolution Failing

```bash
# Test DNS from a pod
kubectl run dns-test --image=busybox --rm -it -- \
  nslookup kubernetes.default.svc.cluster.local
# If this fails → CoreDNS is the problem

kubectl run dns-test --image=busybox --rm -it -- \
  nslookup myapp.production.svc.cluster.local
# If this fails but above works → service doesn't exist in that namespace

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check CoreDNS configmap
kubectl get configmap coredns -n kube-system -o yaml

# Common errors in CoreDNS logs:
# "NXDOMAIN" → name doesn't exist
# "SERVFAIL" → upstream DNS issue
# "i/o timeout" → network policy blocking DNS port 53
```

**AKS CoreDNS custom config:**
```bash
# Check custom CoreDNS config
kubectl get configmap coredns-custom -n kube-system -o yaml

# Test forwarding to custom DNS
kubectl run dns-debug --image=busybox --rm -it -- \
  nslookup corp.internal.example.com

# Restart CoreDNS to pick up config changes
kubectl rollout restart deployment/coredns -n kube-system
```

**EKS CoreDNS:**
```bash
# EKS CoreDNS is a managed add-on
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Common: CoreDNS ConfigMap overwritten by update
# Always use the add-on advanced config to preserve custom settings

# Fix: scale CoreDNS if under heavy load
kubectl scale deployment coredns --replicas=4 -n kube-system

# Enable CoreDNS autoscaling
kubectl apply -f https://github.com/kubernetes-sigs/cluster-proportional-autoscaler/...
```

**GKE CoreDNS:**
```bash
# GKE uses kube-dns (not CoreDNS) on older clusters
# GKE 1.25+ uses CoreDNS by default
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check Cloud DNS stub zones
kubectl get configmap coredns -n kube-system -o yaml
```

---

## 7. NetworkPolicy Troubleshooting

### 7.1 Traffic Blocked by NetworkPolicy

```bash
# Check applied NetworkPolicies
kubectl get networkpolicy -n <ns>
kubectl describe networkpolicy <policy> -n <ns>

# Test connectivity
kubectl run nettest --image=nicolaka/netshoot --rm -it -- bash
# ping <pod-ip>
# curl http://<pod-ip>:<port>/health

# Check if CNI enforces NetworkPolicy
# NetworkPolicy requires a CNI that supports it:
# ✅ Calico, Cilium, Weave, Canal
# ❌ Flannel (default), basic kubenet

# AKS: verify network policy provider
kubectl describe daemonset -n kube-system | grep -i calico
# Or Cilium:
kubectl get pods -n kube-system -l k8s-app=cilium

# EKS: verify VPC CNI network policy is enabled
kubectl get pods -n kube-system | grep network-policy-agent
kubectl set env daemonset aws-node ENABLE_NETWORK_POLICY=true -n kube-system

# GKE: Dataplane V2 (Cilium) with Hubble observability
kubectl exec -n kube-system daemonset/cilium -c cilium-agent -- \
  hubble observe --namespace production --type drop --last 20

# Debug: temporarily allow all traffic to isolate issue
kubectl delete networkpolicy --all -n <ns>
# If traffic works after deletion → policy is the problem
# Restore policy after testing
```

---

## 8. Storage / PVC Troubleshooting

### 8.1 PVC — Stuck in `Pending`

```bash
# Check PVC status
kubectl describe pvc <pvc-name> -n <ns>
# Events: "waiting for first consumer" → normal with WaitForFirstConsumer
# Events: "no persistent volumes available" → no matching PV

# Check StorageClass
kubectl get storageclass
kubectl describe storageclass <sc-name>
# Provisioner must be installed and working

# Check CSI driver pod health
# AKS:
kubectl get pods -n kube-system | grep "csi"
kubectl logs -n kube-system <csi-driver-pod>

# EKS:
kubectl get pods -n kube-system | grep ebs-csi
kubectl logs -n kube-system -l app=ebs-csi-controller -c ebs-plugin | tail -50

# GKE:
kubectl get pods -n kube-system | grep "pd-csi"
kubectl logs -n kube-system -l app=gce-pd-csi-driver
```

| PVC Event | Cause | Fix |
|-----------|-------|-----|
| `no persistent volumes available` | No PV matches | Check StorageClass / AccessMode |
| `waiting for first consumer` | `WaitForFirstConsumer` binding mode | Normal — create pod to trigger |
| `failed to provision volume` | CSI driver error | Check CSI driver logs |
| `failed to get AWS metadata` | Node IAM role missing | Add `AmazonEBSCSIDriverPolicy` |
| `Insufficient regional quota` | GCP quota hit | Request quota increase |
| `StorageAccountProvisioning` timeout | AKS storage account throttled | Use Premium tier or reduce requests |

```bash
# AKS: check Azure Disk CSI errors
kubectl logs -n kube-system -l app=csi-azuredisk-controller -c azuredisk | tail -50

# EKS: check EBS CSI errors
kubectl logs -n kube-system deployment/ebs-csi-controller -c ebs-plugin | grep -i error

# GKE: check PD CSI
kubectl logs -n kube-system -l app=gce-pd-csi-driver | grep -i error

# Manual PV creation (static provisioning as fallback)
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv
spec:
  capacity:
    storage: 10Gi
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /data/manual-pv     # for testing only — use cloud disk in prod
EOF
```

---

### 8.2 Volume Mount — `CrashLoopBackOff` or Permission Denied

```bash
# Check pod events for mount errors
kubectl describe pod <pod> -n <ns>
# "MountVolume.SetUp failed"
# "Unable to attach or mount volumes"

# Common: wrong fsGroup / file permission
kubectl describe pod <pod> | grep fsGroup
# Fix: add securityContext.fsGroup = GID that owns the files

# Fix in pod spec
spec:
  securityContext:
    fsGroup: 1000              # files in mounted volume owned by GID 1000
    runAsUser: 1000
    runAsGroup: 1000

# Check actual volume mount
kubectl exec <pod> -n <ns> -- ls -la /mnt/data

# AKS: Azure Files mount failing (SMB auth issue)
kubectl describe pod <pod> | grep -A5 "Warning MountVolume"
# "mount error(115): Operation now in progress"
# Fix: ensure port 445 (SMB) is open in NSG

# EKS: EBS volume stuck attaching
aws ec2 describe-volumes --volume-ids vol-0123456789abcdef0 \
  --query "Volumes[0].Attachments"
# If still attached to old instance after pod migration:
aws ec2 detach-volume --volume-id vol-0123456789abcdef0 --force

# GKE: PD stuck detaching
gcloud compute disks describe <disk-name> --zone us-central1-a \
  --format="value(users)"
# Force detach if stuck
gcloud compute instances detach-disk <instance> \
  --disk <disk-name> --zone us-central1-a
```

---

### 8.3 PVC — Disk Full / Volume Expansion

```bash
# Check usage inside pod
kubectl exec <pod> -n <ns> -- df -h /mnt/data

# Expand PVC (StorageClass must have allowVolumeExpansion: true)
kubectl patch pvc <pvc-name> -n <ns> \
  -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# Check expansion status
kubectl describe pvc <pvc-name> -n <ns>
# "Waiting for user to (re-)start a pod to finish file system resize"
# → Delete + recreate pod to trigger fs resize

# AKS: verify StorageClass allows expansion
kubectl get storageclass managed-csi -o yaml | grep allowVolumeExpansion

# EKS: gp3 expansion is online (no pod restart needed for block)
kubectl get pvc <pvc> -n <ns> -w  # watch expansion progress

# GKE: pd-ssd expansion
kubectl describe pvc <pvc> | grep "Resizing the volume"
```

---

## 9. Autoscaling Troubleshooting

### 9.1 HPA — Not Scaling

```bash
# Check HPA status
kubectl describe hpa <hpa-name> -n <ns>
# TARGETS column: <current>/<target>
# "AbleToScale: False" → blocked
# "ScalingActive: False" → metrics not available

# Common error messages:
# "unable to get metrics for resource cpu" → metrics-server not running
# "the HPA was unable to compute the replica count" → pod selector mismatch
# "DesiredReplicas: X, CurrentReplicas: X (scale blocked by PDB)" → PDB blocking

# Fix: check metrics-server
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl top pods -n <ns>   # if this fails, metrics-server is down

# Fix: install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# AKS: metrics-server is a managed add-on (auto-installed)
# EKS: install as managed add-on
aws eks create-addon --cluster-name myCluster --addon-name metrics-server
# GKE: metrics-server is included by default

# Fix: pod doesn't have resource requests set (required for CPU-based HPA)
kubectl get pods -n <ns> -o json | jq -r \
  '.items[] | .spec.containers[] | select(.resources.requests == null) | .name'

# Check HPA events
kubectl get events -n <ns> | grep HorizontalPodAutoscaler
```

---

### 9.2 Cluster Autoscaler — Not Scaling Up/Down

```bash
# Check CA logs
kubectl logs -n kube-system deployment/cluster-autoscaler | grep -E "ERROR|scale|unschedulable" | tail -50

# Check CA status configmap
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml
# Look for: "lastScaleUpTime", "lastScaleDownTime", errors

# Common reasons CA doesn't scale up:
# 1. Pod has local storage (emptyDir) → CA won't evict
# 2. PodDisruptionBudget blocks draining
# 3. Pod has node affinity that doesn't match any unscheduled node type
# 4. Max node count already reached

# Common reasons CA doesn't scale down:
# 1. Pod with annotation cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
# 2. System pod on node (kube-system pods block scale-down)
# 3. Node utilization > scale-down threshold (default 50%)

# Check which pods block scale-down
kubectl get pods -A -o json | jq -r \
  '.items[] | select(.metadata.annotations."cluster-autoscaler.kubernetes.io/safe-to-evict" == "false") | 
  "\(.metadata.namespace)/\(.metadata.name)"'

# AKS: check CA profile
az aks show -g myRG -n myCluster --query "autoScalerProfile"

# EKS: check ASG min/max
aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(Tags[?Key=='eks:cluster-name'].Value,'myCluster')]"

# GKE: check CA config
gcloud container clusters describe myCluster --region us-central1 \
  --format="value(autoscaling)"
```

---

### 9.3 KEDA — ScaledObject Not Scaling

```bash
# Check ScaledObject status
kubectl describe scaledobject <name> -n <ns>
# Look for: "ScalerError", "ReplicaCount"

# Check KEDA operator logs
kubectl logs -n keda deployment/keda-operator | tail -50
kubectl logs -n keda deployment/keda-metrics-apiserver | tail -50

# Common errors:
# "error connecting to scaler" → wrong connection string / auth
# "KEDA is disabled" → operator not running
# "TriggerAuthentication not found" → missing auth resource

# Test trigger authentication manually
kubectl describe triggerauthentication <name> -n <ns>

# AKS: KEDA managed add-on
az aks show -g myRG -n myCluster --query "workloadAutoScalerProfile"
# Check KEDA system pods
kubectl get pods -n kube-system | grep keda

# EKS: KEDA managed add-on
aws eks describe-addon --cluster-name myCluster --addon-name keda
```

---

## 10. RBAC / Authorization Troubleshooting

### 10.1 `Forbidden` — 403 Errors

```bash
# Exact error from kubectl
kubectl auth can-i create pods -n production --as system:serviceaccount:default:myapp-sa
# Output: no

# Check what a user/SA can do
kubectl auth can-i --list --as system:serviceaccount:default:myapp-sa -n production

# Find all RoleBindings for a ServiceAccount
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq -r '.items[] | select(
    .subjects[]? | 
    (.kind == "ServiceAccount" and .name == "myapp-sa")
  ) | "\(.kind)/\(.metadata.name) → \(.roleRef.name)"'

# Check ClusterRole permissions
kubectl get clusterrole <role-name> -o yaml | grep -A5 rules

# Common fixes:
```

```yaml
# Fix: create Role + RoleBinding for a service account
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "pods/exec"]
  verbs: ["get", "list", "watch", "create"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-pod-reader
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# AKS: Azure RBAC + AAD group mapping
# Check if user is in correct AAD group
az ad group member check --group <group-id> --member-id <user-id>

# EKS: aws-auth ConfigMap issue
kubectl describe configmap aws-auth -n kube-system
# Ensure IAM role ARN is correct (no trailing whitespace, lowercase exact)

# EKS: modern — check Access Entry
aws eks list-access-entries --cluster-name myCluster
aws eks describe-access-entry --cluster-name myCluster --principal-arn arn:aws:iam::xxx:role/DevRole

# GKE: Workload Identity issue
kubectl describe serviceaccount myapp-ksa -n production
# Check annotation: iam.gke.io/gcp-service-account = GSA@PROJECT.iam.gserviceaccount.com

# Test GCP permissions
gcloud projects get-iam-policy myproject \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:myapp-gsa@myproject.iam.gserviceaccount.com"
```

---

## 11. StatefulSet Troubleshooting

### 11.1 StatefulSet — Pods Not Starting in Order

```bash
# StatefulSets start pods sequentially: pod-0, pod-1, pod-2
# If pod-0 is not Ready, pod-1 won't start

kubectl get pods -l app=mystatefulset -n <ns> -w
kubectl describe pod mystatefulset-0 -n <ns>
# Fix pod-0 first before looking at pod-1

# Check PVC per pod (each pod has its own PVC)
kubectl get pvc -l app=mystatefulset -n <ns>

# PVC naming: <template-name>-<pod-name>
# e.g., data-mystatefulset-0, data-mystatefulset-1

# Fix stuck PVC
kubectl describe pvc data-mystatefulset-0 -n <ns>
```

---

### 11.2 StatefulSet — Pod Rescheduled to Wrong Zone (PVC binding)

```bash
# Problem: StatefulSet pod rescheduled to different AZ than its PVC
# Symptom: Pod Pending with "node(s) had volume node affinity conflict"

kubectl describe pod <pod> -n <ns>
# "1 node(s) had volume node affinity conflict"

# Check PV zone
kubectl get pv <pv-name> -o yaml | grep topology

# Fix: use WaitForFirstConsumer binding mode
# StorageClass: volumeBindingMode: WaitForFirstConsumer

# Fix existing stuck pod (delete and let StatefulSet recreate)
kubectl delete pod mystatefulset-0 -n <ns>
# StatefulSet controller will recreate it; with WaitForFirstConsumer,
# PVC will bind to the zone where the pod is scheduled

# Prevention: use topology-aware StorageClass
# AKS: managed-csi has WaitForFirstConsumer by default
# EKS: WaitForFirstConsumer on ebs-gp3
# GKE: WaitForFirstConsumer on premium-rwo
```

---

### 11.3 StatefulSet — Rolling Update Stuck

```bash
# Check rollout status
kubectl rollout status statefulset/<name> -n <ns>

# Find which pod is blocking
kubectl get pods -l app=<name> -n <ns>
# pod-1 is still running old version while pod-2 is stuck

# StatefulSet update strategy: RollingUpdate (default)
# updateStrategy.rollingUpdate.maxUnavailable = 1 (K8s 1.24+)

# Pause/resume rollout
kubectl rollout pause statefulset/<name> -n <ns>
kubectl rollout resume statefulset/<name> -n <ns>

# Rollback
kubectl rollout undo statefulset/<name> -n <ns>

# Check partition-based update (OnDelete strategy)
kubectl get statefulset <name> -n <ns> -o yaml | grep -A5 updateStrategy
```

---

## 12. DaemonSet Troubleshooting

### 12.1 DaemonSet Pod Not Scheduled on All Nodes

```bash
# Check DaemonSet status
kubectl get daemonset <name> -n <ns>
# DESIRED vs READY vs AVAILABLE

kubectl describe daemonset <name> -n <ns>
# Events: "Failed to create pod: ..."

# Common reason: node selector / affinity doesn't match node
kubectl get daemonset <name> -n <ns> -o yaml | grep -A10 nodeSelector
kubectl get nodes --show-labels | grep <key>

# Common reason: node taint not tolerated
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
kubectl get daemonset <name> -n <ns> -o yaml | grep -A10 tolerations

# Fix: add toleration for control-plane taint
kubectl patch daemonset <name> -n <ns> \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/tolerations/-",
       "value":{"key":"node-role.kubernetes.io/control-plane","operator":"Exists","effect":"NoSchedule"}}]'
```

---

## 13. Jobs & CronJobs Troubleshooting

### 13.1 Job — Failing / Not Completing

```bash
# Check job status
kubectl describe job <job> -n <ns>
kubectl get pods -l job-name=<job> -n <ns>

# Job pod in Error state
kubectl logs -l job-name=<job> -n <ns> --previous

# Job spec: backoffLimit controls retries
# If all retries exhausted: job status = Failed

# Fix: increase backoffLimit or fix application error
kubectl patch job <job> -n <ns> \
  -p '{"spec":{"backoffLimit":6}}'

# Check job completion conditions
kubectl describe job <job> | grep -A10 "Status:"
```

---

### 13.2 CronJob — Not Triggering

```bash
# Check CronJob
kubectl describe cronjob <name> -n <ns>
# LastScheduleTime, Active, Suspended

# Check if CronJob is suspended
kubectl get cronjob <name> -n <ns> -o jsonpath='{.spec.suspend}'
# If "true", unsuspend:
kubectl patch cronjob <name> -n <ns> -p '{"spec":{"suspend":false}}'

# Timezone issues (K8s 1.25+: timezone field supported)
kubectl get cronjob <name> -n <ns> -o yaml | grep -E "schedule|timezone"

# startingDeadlineSeconds too short
# If the CronJob controller misses a run (controller was down), it won't catch up
# if startingDeadlineSeconds is exceeded

# Manually trigger a CronJob
kubectl create job --from=cronjob/<name> manual-run-$(date +%s) -n <ns>

# Check failed job history (failedJobsHistoryLimit)
kubectl get jobs -n <ns> -l app=<name>
```

---

## 14. Helm Troubleshooting

### 14.1 Helm Release — Failed / Stuck

```bash
# Check release status
helm list -n <ns> -a
helm status <release> -n <ns>

# Get full history
helm history <release> -n <ns>

# Release stuck in "pending-install" or "pending-upgrade"
# Secret still exists but release is broken
helm ls --pending -n <ns>

# Force delete stuck Helm release secret
kubectl delete secret -n <ns> sh.helm.release.v1.<release>.v<N>

# Or rollback
helm rollback <release> <revision> -n <ns>

# Upgrade with force
helm upgrade <release> ./chart -n <ns> --force

# Debug dry-run
helm upgrade <release> ./chart -n <ns> --dry-run --debug 2>&1 | head -100

# Check template rendering
helm template <release> ./chart -n <ns> | kubectl apply --dry-run=client -f -

# Common: "cannot patch ... field is immutable"
# Cause: trying to change immutable field (e.g., selector on Deployment)
# Fix: delete deployment and re-apply (or use --force flag)
helm upgrade <release> ./chart -n <ns> --force

# AKS: AKS-managed Helm releases (aks-managed-* prefix)
kubectl get helmreleases -A  # if using Flux
```

---

## 15. Admission Webhook Troubleshooting

### 15.1 Webhook — Blocking Pod Creation

```bash
# Symptom: pod creation fails with
# "Error from server: ... admission webhook ... denied the request"
# or
# "Error from server: ... connection refused" (webhook server unreachable)

# List all webhooks
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Describe a webhook
kubectl describe mutatingwebhookconfiguration <name>
# Check: namespaceSelector, rules, failurePolicy

# Check webhook server pod
kubectl get pods -n <namespace> -l <webhook-app-label>
kubectl logs -n <namespace> <webhook-pod>

# Test webhook endpoint
kubectl exec -n <ns> <pod> -- curl -k \
  https://<webhook-service>.<webhook-ns>.svc.cluster.local:<port>/validate

# Bypass webhook for emergency (if failurePolicy: Fail)
# Option 1: Label namespace to skip webhook
kubectl label namespace <ns> admission.webhook.io/skip=true

# Option 2: Temporarily delete webhook (only in emergency)
kubectl delete mutatingwebhookconfiguration <name>
# Restore after fixing webhook server

# Fix: webhook TLS cert expired
kubectl get secret <webhook-cert-secret> -n <ns> -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -dates
# Renew cert via cert-manager or manually regenerate

# AKS: Azure Policy add-on uses Gatekeeper
kubectl get pods -n gatekeeper-system
kubectl logs -n gatekeeper-system deployment/gatekeeper-controller-manager | tail -50

# EKS/GKE: Kyverno webhook issues
kubectl get pods -n kyverno
kubectl logs -n kyverno deployment/kyverno | grep -i error
```

---

## 16. Cloud-Specific Cluster Troubleshooting

### 16.1 AKS — Cluster-Level Issues

```bash
# ── AKS Diagnostic Commands ──────────────────────────────────

# Check cluster health
az aks show -g myRG -n myCluster --query "provisioningState"
# Succeeded = healthy | Failed/Canceled = broken

# Check node pool health
az aks nodepool list -g myRG --cluster-name myCluster -o table

# Run AKS diagnostics (detects common issues automatically)
az aks diagnose-solve -g myRG -n myCluster --enable-interactive

# Check AKS operation logs
az aks get-credentials -g myRG -n myCluster --admin
kubectl get events -A --sort-by='.lastTimestamp' | tail -50

# Periscope: collect cluster diagnostics
kubectl apply -f https://raw.githubusercontent.com/Azure/aks-periscope/master/deployment/aks-periscope.yaml
az storage blob list --account-name <diag-storage> -c <container> | grep log

# AKS-specific troubleshooting scenarios:

# 1. Nodes stuck in "Not Ready" after upgrade
az aks nodepool upgrade -g myRG --cluster-name myCluster -n nodepool1 \
  --kubernetes-version 1.30.5 --node-image-only

# 2. PVC Pending after Azure Disk quota exhaustion
az vm list-usage --location eastus --query "[?name.value=='Premium Managed Disks'].{Used:currentValue,Limit:limit}"

# 3. Pod identity / Workload Identity issues
kubectl logs -n kube-system -l app=azure-wi-webhook-controller-manager | tail -30

# 4. AKS private cluster — kubectl can't connect
az aks command invoke -g myRG -n myCluster --command "kubectl get nodes"

# 5. API server authorized IP blocking your IP
az aks update -g myRG -n myCluster \
  --api-server-authorized-ip-ranges "$(curl -s ifconfig.me)/32,10.0.0.0/8"

# Useful AKS log queries (Log Analytics):
# Pod OOMKill
# ContainerLog | where LogEntry contains "OOMKilled" | top 50 by TimeGenerated
# Node disk pressure
# KubeNodeInventory | where Status contains "DiskPressure"
# Failed scheduling
# KubeEvents | where Reason == "FailedScheduling" | summarize count() by bin(TimeGenerated, 1h)
```

---

### 16.2 EKS — Cluster-Level Issues

```bash
# ── EKS Diagnostic Commands ──────────────────────────────────

# Check cluster health
aws eks describe-cluster --name myCluster --query "cluster.{Status:status,Health:health}"

# Check cluster issues (EKS health alerts)
aws eks describe-cluster --name myCluster \
  --query "cluster.health.issues"

# Check control plane logs
aws logs tail /aws/eks/myCluster/cluster --follow --filter-pattern "ERROR"

# Enable control plane logging if not already
aws eks update-cluster-config --name myCluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# EKS-specific troubleshooting scenarios:

# 1. Nodes not joining cluster (bootstrap failure)
aws ec2 get-console-output --instance-id i-xxx --output text | grep -i error
# Check: node IAM role has AmazonEKSWorkerNodePolicy, AmazonEKS_CNI_Policy, AmazonEC2ContainerRegistryReadOnly

# 2. VPC CNI pod IP exhaustion
kubectl describe node <node> | grep "vpc.amazonaws.com/pod-eni"
kubectl get pods -n kube-system -l k8s-app=aws-node -o wide
kubectl logs -n kube-system -l k8s-app=aws-node | grep -i error

# Fix: enable prefix delegation
kubectl set env daemonset aws-node ENABLE_PREFIX_DELEGATION=true -n kube-system

# 3. Fargate pod can't pull image (ECR)
# Check Fargate pod execution role has AmazonEC2ContainerRegistryReadOnly
aws iam list-attached-role-policies --role-name MyFargatePodExecutionRole

# 4. Karpenter not provisioning nodes
kubectl logs -n karpenter deployment/karpenter --tail=100
kubectl describe nodeclaim <name>          # check why claim is unsatisfied
kubectl get nodepool -o yaml              # check requirements + limits

# 5. EKS add-on update conflict
aws eks describe-addon --cluster-name myCluster --addon-name vpc-cni \
  --query "addon.{Status:status,Issues:health.issues}"
# Fix conflicts
aws eks update-addon --cluster-name myCluster \
  --addon-name vpc-cni --resolve-conflicts OVERWRITE

# Useful CloudWatch queries:
# API server errors
# fields @timestamp, @message
# | filter @logStream like "kube-apiserver-audit"
# | filter ispresent(responseStatus.code) and responseStatus.code >= 400
# | stats count(*) as errorCount by responseStatus.code
# | sort errorCount desc

# Unauthorized access
# fields @timestamp, @message
# | filter @logStream like "authenticator"
# | filter @message like "Unauthorized"
# | limit 50
```

---

### 16.3 GKE — Cluster-Level Issues

```bash
# ── GKE Diagnostic Commands ──────────────────────────────────

# Check cluster status
gcloud container clusters describe myCluster --region us-central1 \
  --format="value(status,conditions)"

# Check node pool status
gcloud container node-pools list \
  --cluster myCluster --region us-central1 \
  --format="table(name,status,version)"

# View cluster operations (upgrades, repairs)
gcloud container operations list \
  --filter="zone:us-central1" \
  --format="table(name,operationType,status,targetLink)"

# GKE-specific troubleshooting scenarios:

# 1. Nodes stuck in "Repair" state
gcloud container operations describe <operation-id>
# Wait or trigger a new rollout:
gcloud container clusters upgrade myCluster --region us-central1 \
  --node-pool primary-pool --cluster-version $(gcloud container clusters describe myCluster --region us-central1 --format="value(currentMasterVersion)")

# 2. Autopilot pod rejected (compute class constraints)
kubectl get events -n <ns> | grep FailedScheduling
# Check annotation: cloud.google.com/compute-class is valid

# 3. Workload Identity not working
kubectl describe serviceaccount <ksa> -n <ns>
# Must have annotation: iam.gke.io/gcp-service-account
# Test:
kubectl run wi-test \
  --image=google/cloud-sdk:slim \
  --serviceaccount=<ksa> \
  --rm -it -- gcloud auth print-access-token

# 4. Cloud Load Balancer backend unhealthy
gcloud compute backend-services get-health <backend-service-name> --global
# Check: firewall allows 130.211.0.0/22 and 35.191.0.0/16 (GCP health check IPs)

# 5. GKE Dataplane V2 (Cilium) dropping traffic
kubectl exec -n kube-system daemonset/cilium -c cilium-agent -- \
  cilium monitor --type drop 2>&1 | head -50

# 6. Node auto-provisioning (NAP) capacity issues
gcloud container clusters describe myCluster --region us-central1 \
  --format="value(autoscaling.autoprovisioningNodePoolDefaults)"

# Useful Cloud Logging queries:
# OOM events
gcloud logging read \
  'resource.type="k8s_node" jsonPayload.reason="OOMKilling"' \
  --limit 20

# Pod evictions
gcloud logging read \
  'resource.type="k8s_pod" protoPayload.methodName="io.k8s.core.v1.pods.eviction.create"' \
  --limit 20 --format=json

# RBAC forbidden
gcloud logging read \
  'resource.type="k8s_cluster" httpRequest.status=403' \
  --limit 20
```

---

## 17. Performance Troubleshooting

### 17.1 High Latency / Slow Pods

```bash
# Profile pod CPU usage
kubectl top pod <pod> -n <ns> --containers

# Check throttling (CPU throttled ≠ OOMKilled)
kubectl exec <pod> -n <ns> -- cat /sys/fs/cgroup/cpu/cpu.stat | grep throttled
# throttled_time > 0 → CPU limit too low

# Network latency test between pods
kubectl run netshoot-a --image=nicolaka/netshoot -it --rm -- bash
# Inside pod:
ping <other-pod-ip>
iperf3 -c <other-pod-ip> -p 5201      # bandwidth test

# Disk I/O (slow storage)
kubectl exec <pod> -n <ns> -- dd if=/dev/zero of=/tmp/testfile bs=1M count=100
# Compare with expected storage performance

# Node-level: check system load
kubectl debug node/<node> -it --image=nicolaka/netshoot -- bash
# uptime, iostat, free -m, ss -s

# AKS: check Azure Monitor metrics
az monitor metrics list \
  --resource /subscriptions/.../managedClusters/myCluster \
  --metric "node_cpu_usage_percentage" \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)

# EKS: check CloudWatch Container Insights
aws cloudwatch get-metric-statistics \
  --namespace ContainerInsights \
  --metric-name pod_cpu_utilization \
  --dimensions Name=ClusterName,Value=myCluster \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Average

# GKE: check Cloud Monitoring
gcloud monitoring metrics list | grep kubernetes
```

---

## 18. Master Troubleshooting Cheatsheet

### Quick Diagnosis Commands
```bash
# ── 60-second cluster health check ─────────────────────────────
kubectl get nodes                                                # node status
kubectl get pods -A --field-selector=status.phase!=Running      # all non-running pods
kubectl get events -A --sort-by='.lastTimestamp' | tail -30     # recent events
kubectl top nodes                                               # resource usage
kubectl get pvc -A | grep -v Bound                              # unbound PVCs
kubectl get hpa -A                                              # autoscaling status

# ── Per-object quick debug ──────────────────────────────────────
# Pod
kubectl describe pod <pod> -n <ns>         # events + state
kubectl logs <pod> -n <ns> --previous      # last crash logs
kubectl exec -it <pod> -n <ns> -- sh       # shell

# Node
kubectl describe node <node>               # conditions, capacity, events
kubectl top node <node>                    # CPU/memory

# Service
kubectl get endpoints <svc> -n <ns>        # check endpoints exist
kubectl describe svc <svc> -n <ns>         # selector, ports

# Ingress
kubectl describe ingress <ing> -n <ns>     # rules, backend, TLS
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller   # NGINX logs

# Storage
kubectl describe pvc <pvc> -n <ns>         # binding status
kubectl describe pv <pv>                   # reclaim policy, capacity

# RBAC
kubectl auth can-i <verb> <resource> -n <ns> --as system:serviceaccount:<ns>:<sa>
kubectl get rolebindings,clusterrolebindings -A | grep <sa-name>
```

### Error → Fix Quick Reference

| Symptom | First Command | Common Fix |
|---------|--------------|------------|
| Pod Pending | `kubectl describe pod` | Add nodes / fix resources / fix taint |
| CrashLoopBackOff | `kubectl logs --previous` | Fix app error / increase memory |
| ImagePullBackOff | `kubectl describe pod` | Fix pull secret / image name |
| OOMKilled | `kubectl describe pod` | Increase memory limit |
| PVC Pending | `kubectl describe pvc` | Fix StorageClass / CSI driver |
| Service no endpoints | `kubectl get endpoints` | Fix label selector |
| Ingress 502 | Ingress controller logs | Fix backend service/pod |
| DNS failing | `nslookup kubernetes.default` | Restart CoreDNS |
| RBAC Forbidden | `kubectl auth can-i` | Add Role + RoleBinding |
| Node NotReady | `kubectl describe node` | Restart kubelet / fix disk/memory |
| HPA not scaling | `kubectl describe hpa` | Install metrics-server / add requests |
| Webhook blocking | `kubectl describe pod` | Fix webhook server / check TLS |

### AKS-Specific Commands
```bash
az aks diagnose-solve -g myRG -n myCluster --enable-interactive
az aks command invoke -g myRG -n myCluster --command "kubectl get nodes"
az aks show -g myRG -n myCluster --query "provisioningState"
az aks nodepool list -g myRG --cluster-name myCluster -o table
az aks get-credentials -g myRG -n myCluster --admin --overwrite-existing
kubectl get events -n kube-system | grep -E "cluster-autoscaler|azure"
```

### EKS-Specific Commands
```bash
aws eks describe-cluster --name myCluster --query "cluster.health"
aws eks update-kubeconfig --name myCluster --region us-east-1
aws logs tail /aws/eks/myCluster/cluster --follow
kubectl logs -n kube-system -l k8s-app=aws-node | grep -i error
kubectl get nodeclaims                               # Karpenter
aws elbv2 describe-target-health --target-group-arn <arn>
```

### GKE-Specific Commands
```bash
gcloud container clusters describe myCluster --region us-central1 --format="value(status)"
gcloud container clusters get-credentials myCluster --region us-central1
gcloud container operations list --filter="status:RUNNING"
kubectl exec -n kube-system daemonset/cilium -- hubble observe --type drop
gcloud compute backend-services get-health <service> --global
gcloud logging read 'resource.type="k8s_node"' --limit 20
```

### Useful Tools
```bash
# k9s - interactive terminal UI
brew install k9s && k9s

# stern - multi-pod log tailing
stern <pod-prefix> -n <ns>

# kubens / kubectx - fast context/namespace switching
kubectx myCluster
kubens production

# netshoot - network debugging Swiss army knife
kubectl run netshoot --image=nicolaka/netshoot --rm -it -- bash

# kubectl-neat - clean up k8s YAML output
kubectl get pod <pod> -o yaml | kubectl neat

# Popeye - cluster sanitizer / linter
popeye --context myCluster

# Trivy - vulnerability scanner
trivy image myapp:latest

# Pluto - detect deprecated K8s API versions in Helm charts
pluto detect-helm --target-versions k8s=v1.30.0
```

---

*Generated June 2026 | Covers Kubernetes 1.28–1.36, AKS, EKS (including Auto Mode), GKE (Autopilot + Standard)*  
*References: kubernetes.io/docs/tasks/debug | learn.microsoft.com/azure/aks | docs.aws.amazon.com/eks | cloud.google.com/kubernetes-engine*

---

# PART 2 — Gap Fill: Additional Troubleshooting Sections

---

## 19. Deployment Troubleshooting

### 19.1 Deployment — Rollout Stuck / Progressing

```bash
# Check rollout status
kubectl rollout status deployment/<name> -n <ns>
# "Waiting for deployment rollout to finish: 1 out of 3 new replicas updated"

# Check rollout history
kubectl rollout history deployment/<name> -n <ns>
# REVISION  CHANGE-CAUSE
# 1         initial deploy
# 2         image update to v1.2

# Inspect a specific revision
kubectl rollout history deployment/<name> -n <ns> --revision=2

# Why is it stuck? Check new pods
kubectl get pods -n <ns> -l app=<name>
kubectl describe pod <new-pod> -n <ns>
# Events will show the cause (ImagePullBackOff, CrashLoop, Pending, etc.)

# Pause a rollout to stop it mid-progress
kubectl rollout pause deployment/<name> -n <ns>

# Resume after fix
kubectl rollout resume deployment/<name> -n <ns>

# Rollback to previous revision
kubectl rollout undo deployment/<name> -n <ns>

# Rollback to specific revision
kubectl rollout undo deployment/<name> -n <ns> --to-revision=1

# Verify rollback is complete
kubectl rollout status deployment/<name> -n <ns>

# Force restart all pods (new rollout with same image — picks up configmap changes)
kubectl rollout restart deployment/<name> -n <ns>
```

---

### 19.2 Deployment — Zero-Downtime Rollout Problems

```bash
# Symptom: brief 503s during deployment
# Root cause: new pods not ready before old pods terminate

# Check rollout strategy
kubectl get deployment <name> -n <ns> -o yaml | grep -A8 strategy

# Fix: ensure readinessProbe is set so pods only receive traffic when ready
# Fix: set maxUnavailable=0 for zero-downtime
kubectl patch deployment <name> -n <ns> \
  -p '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":0,"maxSurge":1}}}}'

# Fix: add minReadySeconds to stabilize before marking as available
kubectl patch deployment <name> -n <ns> \
  -p '{"spec":{"minReadySeconds":10}}'

# Fix: set terminationGracePeriodSeconds > load balancer drain timeout
kubectl patch deployment <name> -n <ns> \
  -p '{"spec":{"template":{"spec":{"terminationGracePeriodSeconds":60}}}}'

# Check if PDB is blocking rollout
kubectl get pdb -n <ns>
# minAvailable must be < replicas, otherwise rollout can't drain old pods
```

---

### 19.3 Deployment — ReplicaSet Not Scaling

```bash
# Check ReplicaSets owned by deployment
kubectl get replicaset -n <ns> -l app=<name>
# DESIRED  CURRENT  READY
# 3        3        0        ← pods not becoming ready

# Find which RS is active
kubectl get deployment <name> -n <ns> \
  -o jsonpath='{.spec.selector.matchLabels}'
kubectl get rs -n <ns> --selector=app=<name>

# RS owns pods — check pod events
kubectl describe rs <rs-name> -n <ns>

# Common: deployment quota hit (ResourceQuota)
kubectl describe resourcequota -n <ns>
# "pods: 10/10" → at quota limit

# Orphaned ReplicaSets (old ones) not cleaned up
kubectl get rs -n <ns> | grep " 0 " | awk '{print $1}' | \
  xargs kubectl delete rs -n <ns>
```

---

## 20. Init Container Troubleshooting

### 20.1 Pod Stuck in `Init:0/1` or `Init:Error`

```bash
# Init containers run before app containers — all must succeed

# Check init container status
kubectl get pod <pod> -n <ns>
# STATUS: Init:0/2 → 0 of 2 init containers completed

kubectl describe pod <pod> -n <ns>
# "Init Containers" section shows each init container state
# Look for: State=Waiting/Running/Terminated, Exit Code, Reason

# Get init container logs (specify container name)
kubectl logs <pod> -n <ns> -c <init-container-name>
kubectl logs <pod> -n <ns> -c <init-container-name> --previous

# Example: init container waiting for database
# logs: "waiting for postgres:5432 to be ready..."
# Fix: ensure DB service is up and DNS resolves
kubectl run db-check --rm -it --image=busybox -- \
  wget -qO- http://postgres.production.svc.cluster.local:5432

# Example: init container running migrations fails
# logs: "ERROR: relation does not exist"
# Fix: apply DB schema first, or fix migration script

# Check init container exit code
kubectl get pod <pod> -n <ns> -o jsonpath=\
  '{.status.initContainerStatuses[*].state.terminated.exitCode}'

# Init:Error — exit code 1 → check logs
# Init:OOMKilled — exit code 137 → increase memory limit for init container

# Fix init container resource limits
kubectl patch pod <pod> -n <ns> --type='json' -p='[{
  "op": "replace",
  "path": "/spec/initContainers/0/resources/limits/memory",
  "value": "512Mi"
}]'
# Note: must update via Deployment/StatefulSet spec, not patch pod directly
```

---

## 21. Multi-Container Pod Troubleshooting

### 21.1 Sidecar Container Issues

```bash
# List all containers in a pod
kubectl get pod <pod> -n <ns> \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'

# Get logs from specific container
kubectl logs <pod> -n <ns> -c <container-name>
kubectl logs <pod> -n <ns> -c <container-name> --previous

# Exec into a specific container
kubectl exec -it <pod> -n <ns> -c <container-name> -- /bin/sh

# Check each container's status
kubectl get pod <pod> -n <ns> -o json | \
  jq '.status.containerStatuses[] | {name: .name, ready: .ready, state: .state}'

# Common sidecar issue: Istio proxy not ready before app starts
kubectl describe pod <pod> -n <ns> | grep -A5 "istio-proxy"
# Fix: holdApplicationUntilProxyStarts in IstioOperator
kubectl get configmap istio -n istio-system -o yaml | grep holdApplication

# Common: Fluent Bit / log forwarder sidecar crashing
kubectl logs <pod> -n <ns> -c fluent-bit --previous
# Fix: check Fluent Bit config for invalid parsers or unreachable backends

# Check if app container depends on sidecar readiness
# postStart hook can block main container
kubectl get pod <pod> -n <ns> -o yaml | grep -A10 lifecycle
```

---

## 22. ConfigMap & Secret Troubleshooting

### 22.1 ConfigMap/Secret Not Mounted or Updated

```bash
# Verify ConfigMap exists in correct namespace
kubectl get configmap <name> -n <ns>
kubectl describe configmap <name> -n <ns>

# Verify Secret exists in correct namespace
kubectl get secret <name> -n <ns>
# Type: Opaque / kubernetes.io/tls / kubernetes.io/dockerconfigjson

# Decode a secret value
kubectl get secret <name> -n <ns> \
  -o jsonpath='{.data.password}' | base64 --decode

# Check all secrets in namespace
kubectl get secrets -n <ns> -o custom-columns=\
  NAME:.metadata.name,TYPE:.type,CREATED:.metadata.creationTimestamp

# Check if pod has the correct volume mount
kubectl describe pod <pod> -n <ns> | grep -A20 "Volumes:"
kubectl describe pod <pod> -n <ns> | grep -A20 "Mounts:"

# Verify file is actually mounted in container
kubectl exec <pod> -n <ns> -- ls -la /etc/config/
kubectl exec <pod> -n <ns> -- cat /etc/config/app.properties

# ConfigMap update NOT reflected in running pod
# Volumes are automatically updated ~60s after CM update (no restart needed)
# BUT env vars from CM require pod restart to pick up changes

# Force pod restart to pick up CM/Secret changes
kubectl rollout restart deployment/<name> -n <ns>

# Check if CM update propagated to volume
kubectl exec <pod> -n <ns> -- cat /etc/config/version.txt
# Should show new value within 60–120 seconds

# Immutable ConfigMap/Secret (K8s 1.21+) — cannot be updated
kubectl get configmap <name> -n <ns> -o yaml | grep immutable
# If immutable: true → must delete + recreate with new name, update pod ref

# Common: wrong key name in secretKeyRef / configMapKeyRef
kubectl get pod <pod> -n <ns> -o yaml | grep -A5 valueFrom
# Key must exactly match the CM/Secret key
kubectl get configmap <name> -n <ns> -o yaml | grep -A5 data
```

---

## 23. EndpointSlices Troubleshooting

### 23.1 Service EndpointSlices Not Populating

```bash
# EndpointSlices replaced Endpoints in K8s 1.21+ (more scalable)

# Check EndpointSlices for a service
kubectl get endpointslices -n <ns> -l kubernetes.io/service-name=<svc-name>
kubectl describe endpointslice <slice-name> -n <ns>

# Compare with old Endpoints object
kubectl get endpoints <svc-name> -n <ns>

# Check endpoint conditions
kubectl get endpointslice -n <ns> -o json | \
  jq '.items[] | .endpoints[] | {address: .addresses, ready: .conditions.ready, serving: .conditions.serving}'

# Common: pod is running but endpoint marked as notReady
# Cause: readinessProbe failing
kubectl describe pod <pod> -n <ns> | grep -A10 "Readiness:"
# Fix: check why readiness probe is failing
kubectl exec <pod> -n <ns> -- wget -qO- http://localhost:8080/health

# Common: EndpointSlice controller backoff (large clusters)
kubectl get events -n <ns> | grep EndpointSlice

# Service selecting wrong pods (selector mismatch)
kubectl get svc <svc-name> -n <ns> -o jsonpath='{.spec.selector}'
kubectl get pods -n <ns> --selector=<key>=<value> --show-labels
# If empty result → selector doesn't match any pod

# kube-proxy not syncing EndpointSlices to iptables
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy | grep -i error

# On a node: verify iptables rules for service
# SSH to node
sudo iptables -t nat -L KUBE-SERVICES -n | grep <cluster-ip>
sudo iptables -t nat -L KUBE-SVC-XXXXXXXX -n
```

---

## 24. kube-proxy & iptables Troubleshooting

### 24.1 Service Traffic Not Routing Correctly

```bash
# Check kube-proxy mode (iptables / ipvs / nftables)
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy | tail -50

# Check iptables rules for a ClusterIP service
SVC_IP=$(kubectl get svc <svc> -n <ns> -o jsonpath='{.spec.clusterIP}')
sudo iptables -t nat -S | grep $SVC_IP

# IPVS mode: use ipvsadm
sudo ipvsadm -Ln | grep $SVC_IP

# Verify conntrack table
sudo conntrack -L | grep $SVC_IP

# Test ClusterIP from inside cluster
kubectl run test --image=busybox --rm -it -- \
  wget -O- http://$SVC_IP:<port>/health

# kube-proxy health endpoint
curl http://127.0.0.1:10256/healthz    # on node

# Common: NodePort not accessible externally
# Check if firewall/security-group allows NodePort range (30000-32767)
# AKS: NSG rule for NodePort range
# EKS: Security group rule for 30000-32767
# GKE: Firewall rule for tcp:30000-32767

# kube-proxy not running on a node
kubectl get pods -n kube-system -o wide | grep kube-proxy
# Find which node is missing kube-proxy

# Restart kube-proxy daemonset
kubectl rollout restart daemonset/kube-proxy -n kube-system
```

---

## 25. CNI Plugin Troubleshooting

### 25.1 CNI Plugin Errors — Pod Network Setup Failing

```bash
# Symptom: pod stuck in ContainerCreating with network errors
kubectl describe pod <pod> -n <ns>
# "NetworkPlugin cni failed to set up pod..."
# "failed to find plugin 'cilium' in path..."

# Check CNI plugin pods
kubectl get pods -n kube-system | grep -E "calico|cilium|flannel|weave|aws-node"

# Check CNI binary on node
kubectl debug node/<node> -it --image=busybox -- sh
# ls /opt/cni/bin/              # CNI binaries
# ls /etc/cni/net.d/            # CNI config files
# cat /etc/cni/net.d/10-*.conf  # active CNI config

# Calico troubleshooting
kubectl logs -n kube-system -l k8s-app=calico-node | grep -i error
# Check BGP peers
kubectl exec -n kube-system <calico-pod> -- calicoctl node status
# Check IP pool
kubectl exec -n kube-system <calico-pod> -- calicoctl get ippool -o wide

# Cilium troubleshooting
kubectl get pods -n kube-system -l k8s-app=cilium
kubectl exec -n kube-system <cilium-pod> -- cilium status
kubectl exec -n kube-system <cilium-pod> -- cilium endpoint list
# Check for dropped packets
kubectl exec -n kube-system <cilium-pod> -- cilium monitor --type drop

# Flannel troubleshooting
kubectl logs -n kube-system -l app=flannel | grep -i error
# Check flannel.1 interface exists on node
ip link show flannel.1
# Check flannel subnet file
cat /run/flannel/subnet.env

# AWS VPC CNI (EKS)
kubectl logs -n kube-system -l k8s-app=aws-node | grep -i "error\|warn"
# Check IP allocation
kubectl get node <node> -o json | \
  jq '.metadata.annotations."vpc.amazonaws.com/node-capacity-pod-ip"'

# Weave Net
kubectl exec -n kube-system <weave-pod> -c weave -- /home/weave/weave --local status
```

---

## 26. Service Mesh (Istio) Troubleshooting

### 26.1 Istio Sidecar Injection Failing

```bash
# Check sidecar injection is enabled on namespace
kubectl get namespace <ns> --show-labels | grep istio-injection

# Enable injection
kubectl label namespace <ns> istio-injection=enabled

# Check if pod has sidecar (2 containers in pod)
kubectl get pod <pod> -n <ns> \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
# Should show: app-container + istio-proxy

# Webhook not injecting? Check istiod
kubectl get pods -n istio-system
kubectl logs -n istio-system deployment/istiod | grep -i error

# Check mutating webhook
kubectl get mutatingwebhookconfiguration istio-sidecar-injector -o yaml \
  | grep -A5 namespaceSelector

# Test injection manually
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: injection-test
  namespace: <ns>
spec:
  containers:
  - name: test
    image: busybox
    command: [sleep, "3600"]
EOF
kubectl get pod injection-test -n <ns> -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
# Should show test + istio-proxy
kubectl delete pod injection-test -n <ns>
```

---

### 26.2 Istio — 503 / mTLS / Traffic Management Issues

```bash
# istioctl analysis (best first step)
istioctl analyze -n <ns>
# Reports: conflicting VirtualServices, missing DestinationRules, port name issues

# Check proxy status
istioctl proxy-status
# Shows sync state of each proxy with istiod

# Check proxy config for a pod
istioctl proxy-config clusters <pod> -n <ns>
istioctl proxy-config listeners <pod> -n <ns>
istioctl proxy-config routes <pod> -n <ns>
istioctl proxy-config endpoints <pod> -n <ns>

# Check Envoy access logs
kubectl logs <pod> -n <ns> -c istio-proxy | tail -50
# Look for: upstream_reset_before_response_started, 503, ECONNREFUSED

# mTLS issues
istioctl x check-inject -n <ns>
kubectl get peerauthentication -n <ns>
kubectl get peerauthentication -n istio-system  # global policy

# Force mTLS to PERMISSIVE for debugging
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: <ns>
spec:
  mtls:
    mode: PERMISSIVE
EOF

# Debug traffic with Envoy admin API
kubectl exec <pod> -n <ns> -c istio-proxy -- \
  curl -s http://localhost:15000/stats | grep "upstream_rq_5xx"

# Check Envoy clusters for an upstream service
kubectl exec <pod> -n <ns> -c istio-proxy -- \
  curl -s http://localhost:15000/clusters | grep <service-name>

# Istio ingress gateway 503
kubectl logs -n istio-system deployment/istio-ingressgateway | tail -30
# Check: VirtualService host matches Gateway listener
# Check: DestinationRule subset labels match pod labels

# tcpdump on istio-proxy loopback (capture plaintext between app and proxy)
kubectl exec <pod> -n <ns> -c istio-proxy -- \
  tcpdump -i lo -A port 8080 -w /tmp/capture.pcap
kubectl cp <pod>:/tmp/capture.pcap ./capture.pcap -c istio-proxy -n <ns>
```

---

### 26.3 Istio — Ambient Mesh Troubleshooting (K8s 1.28+)

```bash
# Ambient mode: ztunnel replaces per-pod sidecars
kubectl get pods -n istio-system | grep ztunnel
kubectl logs -n istio-system daemonset/ztunnel | grep -i error

# Check if namespace is in ambient mesh
kubectl get namespace <ns> -o yaml | grep istio.io/dataplane-mode

# Add namespace to ambient mesh
kubectl label namespace <ns> istio.io/dataplane-mode=ambient

# Check waypoint proxy (L7 policy enforcement)
kubectl get gateway -n <ns>
istioctl x waypoint status -n <ns>

# ztunnel connectivity test
kubectl exec -n istio-system <ztunnel-pod> -- \
  curl -v http://<pod-ip>:15008   # HBONE tunnel port

# NetworkPolicy blocking port 15008 (ztunnel tunnel port)
# Must allow TCP 15008 between pods when using ambient mode
```

---

## 27. Namespace Troubleshooting

### 27.1 Namespace Stuck in `Terminating`

```bash
# Namespace stuck terminating → finalizers not cleared

# Check what's blocking deletion
kubectl describe namespace <ns>
# Look for: "Finalizers" field, conditions

# Find all remaining resources in the namespace
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I {} kubectl get {} -n <ns> --ignore-not-found 2>/dev/null | \
  grep -v "^NAME" | grep -v "^$"

# Common culprits:
kubectl get pods,pvc,svc,ingress,configmap,secret,serviceaccount \
  -n <ns> --ignore-not-found

# Force-remove finalizer from namespace
kubectl get namespace <ns> -o json > /tmp/ns.json
# Edit: remove "kubernetes" from spec.finalizers array
kubectl replace --raw "/api/v1/namespaces/<ns>/finalize" -f /tmp/ns.json

# Or use patch
kubectl patch namespace <ns> \
  -p '{"spec":{"finalizers":[]}}' \
  --type=merge

# Clean up stuck CRD resources first (often the real blocker)
kubectl get <crd-resource> -n <ns>
kubectl patch <crd-resource> <name> -n <ns> \
  -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl delete <crd-resource> <name> -n <ns> --force --grace-period=0
```

---

## 28. ResourceQuota & LimitRange Troubleshooting

### 28.1 Quota Exceeded / Pods Rejected

```bash
# Check quota usage
kubectl describe resourcequota -n <ns>
# requests.cpu: 18/20 → 90% used
# pods: 48/50       → near limit

# Find which pods are consuming the most
kubectl top pods -n <ns> --sort-by=cpu
kubectl top pods -n <ns> --sort-by=memory

# Find pods WITHOUT resource requests (counting against quota unfairly)
kubectl get pods -n <ns> -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.requests == null) | 
  .metadata.name'

# LimitRange defaults being applied
kubectl describe limitrange -n <ns>
# If container has no limits, LimitRange injects defaults

# Increase quota (requires cluster-admin)
kubectl patch resourcequota <quota-name> -n <ns> \
  -p '{"spec":{"hard":{"pods":"100","requests.cpu":"40"}}}'

# Check if ResourceQuota is blocking a deployment
kubectl get events -n <ns> | grep -i quota

# Quota for specific resource types
kubectl describe resourcequota -n <ns> | grep -E "services|secrets|configmaps|pvc"
```

---

## 29. Ephemeral Containers & Debug Profiles

### 29.1 Advanced Debugging with Ephemeral Containers

```bash
# Ephemeral containers: add debug tools to running pod without restart
# Useful for distroless/scratch images with no shell

# Basic ephemeral container
kubectl debug -it <pod> -n <ns> \
  --image=nicolaka/netshoot \
  --target=<app-container-name>
# --target shares process namespace with app container

# Debug profiles (K8s 1.30+)
# general: basic debug tools, no special privileges
kubectl debug -it <pod> -n <ns> --image=busybox \
  --profile=general

# restricted: respects pod security policies
kubectl debug -it <pod> -n <ns> --image=busybox \
  --profile=restricted

# sysadmin: full node access (privileged)
kubectl debug node/<node> -it --image=ubuntu \
  --profile=sysadmin
# Inside: chroot /host → access node filesystem

# netadmin: network debugging (NET_ADMIN capability)
kubectl debug -it <pod> -n <ns> --image=nicolaka/netshoot \
  --profile=netadmin

# Copy pod with modified entrypoint (for crash-at-startup debugging)
kubectl debug <pod> -n <ns> \
  --copy-to=debug-pod \
  --image=myapp:latest \
  --container=app -- /bin/sh   # override entrypoint

# Debug from node level (access node filesystem)
kubectl debug node/<node> -it --image=ubuntu -- chroot /host bash
# Can then run: journalctl -u kubelet, systemctl status containerd, etc.

# Port-forward for local access
kubectl port-forward pod/<pod> 8080:8080 -n <ns>
kubectl port-forward svc/<svc> 9090:80 -n <ns>
# Then: curl http://localhost:8080/health
```

---

## 30. Packet Capture & Network Deep Debugging

### 30.1 tcpdump / Packet Capture in Pods

```bash
# Install tcpdump in ephemeral container
kubectl debug -it <pod> -n <ns> \
  --image=nicolaka/netshoot \
  --target=<container> -- tcpdump -i eth0 -w /tmp/cap.pcap

# Copy pcap to local for Wireshark analysis
kubectl cp <ns>/<pod>:/tmp/cap.pcap ./debug.pcap -c debugger

# Capture DNS queries (diagnose DNS issues)
kubectl debug -it <pod> -n <ns> \
  --image=nicolaka/netshoot \
  --target=<container> -- \
  tcpdump -i eth0 -n port 53

# Capture HTTP traffic
kubectl debug -it <pod> -n <ns> \
  --image=nicolaka/netshoot \
  --target=<container> -- \
  tcpdump -i eth0 -A 'tcp port 8080'

# For Istio: capture on loopback (plaintext between app and proxy)
kubectl exec <pod> -n <ns> -c istio-proxy -- \
  tcpdump -i lo -A 'tcp port 8080' 2>&1 | head -50

# On node: capture by pod interface
# Step 1: find pod's network interface on node
POD_IP=$(kubectl get pod <pod> -n <ns> -o jsonpath='{.status.podIP}')
ssh <node>
# Find veth interface matching pod IP
ip route get $POD_IP
# Step 2: capture on that interface
tcpdump -i veth<xxxxx> -w /tmp/node-cap.pcap

# NSenter into pod's network namespace from node
PID=$(docker inspect --format '{{.State.Pid}}' <container-id>)
# Or for containerd:
PID=$(crictl inspect <container-id> | jq '.info.pid')
nsenter -n -t $PID -- ss -tuln   # see open ports from pod's perspective
nsenter -n -t $PID -- ip route   # routing table from pod's perspective
```

---

## 31. Resource Leak & Cleanup Troubleshooting

### 31.1 Orphaned Resources & Finalizer Leaks

```bash
# Find all pods with non-empty finalizers
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.metadata.finalizers != null and (.metadata.finalizers | length > 0)) | 
  "\(.metadata.namespace)/\(.metadata.name): \(.metadata.finalizers)"'

# Find all PVCs with Terminating state
kubectl get pvc -A | grep Terminating

# Find all namespaces stuck Terminating
kubectl get ns | grep Terminating

# Force-remove finalizer from PVC
kubectl patch pvc <pvc-name> -n <ns> \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

# Find leaked/orphaned ConfigMaps and Secrets (not referenced by any pod)
kubectl get configmap -n <ns> -o name | while read cm; do
  name=$(echo $cm | cut -d/ -f2)
  refs=$(kubectl get pods -n <ns> -o json | \
    jq -r --arg cm "$name" \
    '[.items[].spec | .. | objects | select(.configMap.name == $cm or .configMapKeyRef.name == $cm)] | length')
  [ "$refs" -eq 0 ] && echo "Unused: $cm"
done

# Clean up completed/failed pods
kubectl delete pods -n <ns> \
  --field-selector=status.phase==Succeeded
kubectl delete pods -n <ns> \
  --field-selector=status.phase==Failed

# Clean up evicted pods cluster-wide
kubectl get pods -A | grep Evicted | \
  awk '{print "kubectl delete pod "$2" -n "$1}' | sh

# Check for leaked LB resources (EKS/AKS/GKE)
kubectl get svc -A | grep LoadBalancer
# Any abandoned LB services? Delete if not needed
```

---

## 32. Upgrade & Version Skew Troubleshooting

### 32.1 API Deprecation / Version Compatibility Issues

```bash
# Check deprecated API usage in cluster
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

# Use pluto to scan Helm releases for deprecated APIs
helm list -A -o json | jq -r '.[] | .name + " " + .namespace' | \
  while read name ns; do
    helm get manifest $name -n $ns | pluto detect - --target-versions k8s=v1.30
  done

# Check all resources using deprecated API versions
kubectl api-resources --verbs=list -o name | \
  xargs -I {} kubectl get {} -A --ignore-not-found 2>/dev/null | head -5

# kubent - Kubernetes Node Triage (find deprecated APIs)
kubent

# Check version skew between components
kubectl version --short
# Must be: apiserver ≥ kubelet version (max 2 minor versions skew)

# Check component versions
kubectl get nodes -o custom-columns=\
  NAME:.metadata.name,\
  KUBELET:.status.nodeInfo.kubeletVersion,\
  CONTAINER-RUNTIME:.status.nodeInfo.containerRuntimeVersion

# After upgrade: verify all add-ons are compatible
kubectl get pods -n kube-system -o wide

# AKS: check for breaking changes before upgrade
az aks get-upgrades -g myRG -n myCluster -o table

# EKS: check release notes before upgrading
aws eks describe-addon-versions --kubernetes-version 1.31 \
  --query "addons[].{Name:addonName,Version:addonVersions[0].addonVersion}"

# GKE: simulate upgrade (check-only)
gcloud container clusters upgrade myCluster --region us-central1 \
  --master --no-async --cluster-version 1.31 --dry-run
```

---

## 33. Security / Audit Troubleshooting

### 33.1 Audit Log Analysis

```bash
# Enable audit logging (kubeadm / self-managed)
# /etc/kubernetes/manifests/kube-apiserver.yaml:
# --audit-policy-file=/etc/kubernetes/audit-policy.yaml
# --audit-log-path=/var/log/kubernetes/audit.log
# --audit-log-maxage=30

# Parse audit logs
cat /var/log/kubernetes/audit.log | \
  jq 'select(.verb == "delete") | {user: .user.username, resource: .objectRef.resource, name: .objectRef.name}'

# Find unauthorized access attempts
cat /var/log/kubernetes/audit.log | \
  jq 'select(.responseStatus.code == 403) | {user: .user.username, resource: .objectRef.resource}'

# AKS audit logs (Log Analytics)
# AzureDiagnostics
# | where Category == "kube-audit"
# | extend log = parse_json(log_s)
# | where log.responseStatus.code >= 400
# | project TimeGenerated, user=log.user.username, verb=log.verb, 
#           resource=log.objectRef.resource, code=log.responseStatus.code
# | order by TimeGenerated desc

# EKS audit logs (CloudWatch Logs Insights)
# fields @timestamp, @message
# | filter @logStream like "kube-apiserver-audit"
# | filter ispresent(responseStatus.code) and responseStatus.code == 403
# | parse @message '"username":*,"' as username
# | stats count(*) by username
# | sort count desc

# GKE audit logs (Cloud Logging)
# resource.type="k8s_cluster"
# protoPayload.authorizationInfo.granted=false
# | summarize count by protoPayload.authenticationInfo.principalEmail

# Find who deleted a resource
kubectl get events -A | grep Deleted

# Runtime security alert (Falco)
kubectl logs -n falco daemonset/falco | grep -i "warning\|error\|critical" | tail -30
# Common alerts:
# "Terminal shell in container" → exec into pod
# "Write below etc" → file write to /etc
# "Contact K8s API server from container" → SA token used
```

---

## 34. Complete kubectl Command Reference

### All Objects — Diagnostic Commands

```bash
# ╔══════════════════════════════════════════════════════════════╗
# ║              CLUSTER HEALTH                                  ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl cluster-info
kubectl cluster-info dump > /tmp/cluster-dump.txt           # full dump
kubectl get componentstatuses                               # legacy
kubectl get --raw /healthz
kubectl get --raw /readyz
kubectl get --raw /livez
kubectl api-versions                                        # all API groups
kubectl api-resources                                       # all resource types

# ╔══════════════════════════════════════════════════════════════╗
# ║              NODES                                           ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl describe node <node>
kubectl top node
kubectl top node --sort-by=cpu
kubectl top node --sort-by=memory
kubectl cordon <node>                                       # mark unschedulable
kubectl uncordon <node>                                     # mark schedulable
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl taint node <node> key=value:NoSchedule              # add taint
kubectl taint node <node> key=value:NoSchedule-             # remove taint

# ╔══════════════════════════════════════════════════════════════╗
# ║              PODS                                            ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get pods -A -o wide                                 # all pods, all namespaces
kubectl get pods -n <ns> --sort-by=.metadata.creationTimestamp
kubectl get pods -n <ns> --field-selector=status.phase=Running
kubectl get pods -n <ns> -o custom-columns=\
  NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName,IP:.status.podIP
kubectl get pod <pod> -n <ns> -o yaml                       # full spec
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous                       # previous container
kubectl logs <pod> -n <ns> -c <container>                   # multi-container
kubectl logs <pod> -n <ns> --tail=100 -f                    # follow
kubectl logs -l app=myapp -n <ns> --all-containers=true     # by label
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl exec <pod> -n <ns> -- env                           # check env vars
kubectl exec <pod> -n <ns> -- cat /etc/resolv.conf          # DNS config
kubectl exec <pod> -n <ns> -- nslookup kubernetes.default   # DNS test
kubectl exec <pod> -n <ns> -- wget -qO- http://svc:port/   # HTTP test
kubectl top pod <pod> -n <ns> --containers
kubectl delete pod <pod> -n <ns> --force --grace-period=0   # force delete
kubectl debug -it <pod> -n <ns> --image=nicolaka/netshoot   # ephemeral debug

# ╔══════════════════════════════════════════════════════════════╗
# ║              DEPLOYMENTS                                     ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get deploy -n <ns> -o wide
kubectl describe deploy <name> -n <ns>
kubectl rollout status deploy/<name> -n <ns>
kubectl rollout history deploy/<name> -n <ns>
kubectl rollout history deploy/<name> -n <ns> --revision=2
kubectl rollout undo deploy/<name> -n <ns>
kubectl rollout undo deploy/<name> -n <ns> --to-revision=1
kubectl rollout pause deploy/<name> -n <ns>
kubectl rollout resume deploy/<name> -n <ns>
kubectl rollout restart deploy/<name> -n <ns>
kubectl scale deploy/<name> -n <ns> --replicas=5
kubectl set image deploy/<name> <container>=<image>:<tag> -n <ns>
kubectl get rs -n <ns>                                       # check ReplicaSets
kubectl get rs -n <ns> --show-labels

# ╔══════════════════════════════════════════════════════════════╗
# ║              SERVICES & ENDPOINTS                            ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get svc -n <ns> -o wide
kubectl describe svc <name> -n <ns>
kubectl get endpoints <name> -n <ns>
kubectl get endpointslices -n <ns> -l kubernetes.io/service-name=<svc>
kubectl describe endpointslice <slice> -n <ns>
kubectl port-forward svc/<name> -n <ns> 8080:80

# ╔══════════════════════════════════════════════════════════════╗
# ║              INGRESS                                         ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get ingress -n <ns> -o wide
kubectl describe ingress <name> -n <ns>
kubectl get ingressclass
kubectl get gateway -n <ns>                                 # Gateway API
kubectl get httproute -n <ns>                               # Gateway API

# ╔══════════════════════════════════════════════════════════════╗
# ║              CONFIGMAPS & SECRETS                            ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get configmap -n <ns>
kubectl describe configmap <name> -n <ns>
kubectl get secret -n <ns>
kubectl describe secret <name> -n <ns>                      # (values base64)
kubectl get secret <name> -n <ns> \
  -o jsonpath='{.data.password}' | base64 --decode

# ╔══════════════════════════════════════════════════════════════╗
# ║              STORAGE                                         ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get pvc -A
kubectl describe pvc <name> -n <ns>
kubectl get pv
kubectl describe pv <name>
kubectl get storageclass
kubectl describe storageclass <name>
kubectl get volumesnapshot -n <ns>                          # if CSI snapshots

# ╔══════════════════════════════════════════════════════════════╗
# ║              RBAC                                            ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get serviceaccount -n <ns>
kubectl get role -n <ns>
kubectl get rolebinding -n <ns>
kubectl get clusterrole
kubectl get clusterrolebinding
kubectl describe rolebinding <name> -n <ns>
kubectl auth can-i <verb> <resource> -n <ns>
kubectl auth can-i --list -n <ns>
kubectl auth can-i create pods --as system:serviceaccount:<ns>:<sa> -n <ns>
kubectl auth whoami                                         # current identity

# ╔══════════════════════════════════════════════════════════════╗
# ║              EVENTS                                          ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get events -n <ns>
kubectl get events -A --sort-by='.lastTimestamp'
kubectl get events -n <ns> --field-selector involvedObject.name=<pod>
kubectl get events -n <ns> --field-selector reason=OOMKilling
kubectl get events -n <ns> --field-selector reason=FailedScheduling
kubectl get events -n <ns> --field-selector type=Warning

# ╔══════════════════════════════════════════════════════════════╗
# ║              STATEFULSETS                                    ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get statefulset -n <ns>
kubectl describe statefulset <name> -n <ns>
kubectl rollout status statefulset/<name> -n <ns>
kubectl rollout history statefulset/<name> -n <ns>
kubectl rollout undo statefulset/<name> -n <ns>
kubectl rollout restart statefulset/<name> -n <ns>
kubectl get pvc -l app=<name> -n <ns>                       # per-pod PVCs

# ╔══════════════════════════════════════════════════════════════╗
# ║              DAEMONSETS                                      ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get daemonset -n <ns>
kubectl describe daemonset <name> -n <ns>
kubectl rollout status daemonset/<name> -n <ns>
kubectl rollout restart daemonset/<name> -n <ns>

# ╔══════════════════════════════════════════════════════════════╗
# ║              JOBS & CRONJOBS                                 ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get job -n <ns>
kubectl describe job <name> -n <ns>
kubectl get cronjob -n <ns>
kubectl describe cronjob <name> -n <ns>
kubectl create job --from=cronjob/<name> manual-$(date +%s) -n <ns>
kubectl patch cronjob <name> -n <ns> -p '{"spec":{"suspend":false}}'

# ╔══════════════════════════════════════════════════════════════╗
# ║              HPA / VPA / KEDA                                ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get hpa -n <ns>
kubectl describe hpa <name> -n <ns>
kubectl get vpa -n <ns>
kubectl describe vpa <name> -n <ns>
kubectl get scaledobject -n <ns>                            # KEDA
kubectl describe scaledobject <name> -n <ns>
kubectl get nodepools                                       # Karpenter/NAP
kubectl get nodeclaims                                      # Karpenter

# ╔══════════════════════════════════════════════════════════════╗
# ║              NETWORKPOLICY                                   ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get networkpolicy -n <ns>
kubectl describe networkpolicy <name> -n <ns>

# ╔══════════════════════════════════════════════════════════════╗
# ║              NAMESPACES                                      ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get namespace
kubectl describe namespace <name>
kubectl get resourcequota -n <ns>
kubectl describe resourcequota -n <ns>
kubectl get limitrange -n <ns>
kubectl describe limitrange -n <ns>

# ╔══════════════════════════════════════════════════════════════╗
# ║              WEBHOOK / ADMISSION                             ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl get mutatingwebhookconfiguration
kubectl describe mutatingwebhookconfiguration <name>
kubectl get validatingwebhookconfiguration
kubectl describe validatingwebhookconfiguration <name>

# ╔══════════════════════════════════════════════════════════════╗
# ║              HELM                                            ║
# ╚══════════════════════════════════════════════════════════════╝
helm list -A -a                                             # all releases incl failed
helm status <release> -n <ns>
helm history <release> -n <ns>
helm rollback <release> <rev> -n <ns>
helm get values <release> -n <ns>
helm get manifest <release> -n <ns>
helm template <release> ./chart --debug
helm upgrade <release> ./chart -n <ns> --dry-run

# ╔══════════════════════════════════════════════════════════════╗
# ║              CONTEXT & CLUSTER ACCESS                        ║
# ╚══════════════════════════════════════════════════════════════╝
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context>
kubectl config set-context --current --namespace=<ns>       # set default ns
kubectx <context>                                          # kubectx tool
kubens <namespace>                                         # kubens tool
```

---

## Updated Master Gap Summary Table

The following topics were **missing** from Part 1 and have been **added** in Part 2:

| Section | Topic Added |
|---------|------------|
| §19 | Deployment rollout stuck / rollback / zero-downtime / ReplicaSet |
| §20 | Init container debugging (Init:0/1, Init:Error, exit codes) |
| §21 | Multi-container pods (sidecar, Istio proxy, log forwarder) |
| §22 | ConfigMap/Secret debugging (missing, stale, immutable) |
| §23 | EndpointSlices (vs Endpoints, conditions, kube-proxy sync) |
| §24 | kube-proxy & iptables (IPVS, conntrack, NodePort, modes) |
| §25 | CNI plugin errors (Calico/Cilium/Flannel/Weave/VPC CNI) |
| §26 | Istio (sidecar injection, mTLS, 503s, Envoy config, ambient) |
| §27 | Namespace stuck Terminating + force-remove finalizers |
| §28 | ResourceQuota & LimitRange exhaustion debugging |
| §29 | Ephemeral containers & debug profiles (general/restricted/sysadmin/netadmin) |
| §30 | Packet capture (tcpdump, Wireshark, netshoot, nsenter) |
| §31 | Resource leaks, orphaned finalizers, evicted pod cleanup |
| §32 | Upgrade/version skew, deprecated API detection (pluto, kubent) |
| §33 | Security audit logs (AKS/EKS/GKE), Falco runtime alerts |
| §34 | Complete kubectl command reference for ALL K8s objects |

---
*Part 2 added June 2026 — gap-filled from kubernetes.io/docs/tasks/debug, official CNI docs, Istio troubleshooting guide*
