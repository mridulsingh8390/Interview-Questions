# Kubernetes Certification Complete Study Guide
## CKA · CKAD · CKS — Detailed Q&A + Master Cheatsheets

> **Based on official 2025 CNCF curriculum** · CKA v1.34+ · CKAD v1.35 · CKS v1.31+
> Levels: 🟢 Basic · 🟡 Intermediate · 🔴 Advanced

---

## Table of Contents

### CKA — Certified Kubernetes Administrator
- [Domain 1: Cluster Architecture, Installation & Configuration (25%)](#cka-domain-1)
- [Domain 2: Workloads & Scheduling (15%)](#cka-domain-2)
- [Domain 3: Services & Networking (20%)](#cka-domain-3)
- [Domain 4: Storage (10%)](#cka-domain-4)
- [Domain 5: Troubleshooting (30%)](#cka-domain-5)

### CKAD — Certified Kubernetes Application Developer
- [Domain 1: Application Design & Build (20%)](#ckad-domain-1)
- [Domain 2: Application Deployment (20%)](#ckad-domain-2)
- [Domain 3: Application Observability & Maintenance (15%)](#ckad-domain-3)
- [Domain 4: Application Environment, Configuration & Security (25%)](#ckad-domain-4)
- [Domain 5: Services & Networking (20%)](#ckad-domain-5)

### CKS — Certified Kubernetes Security Specialist
- [Domain 1: Cluster Setup (15%)](#cks-domain-1)
- [Domain 2: Cluster Hardening (15%)](#cks-domain-2)
- [Domain 3: System Hardening (10%)](#cks-domain-3)
- [Domain 4: Minimize Microservice Vulnerabilities (20%)](#cks-domain-4)
- [Domain 5: Supply Chain Security (20%)](#cks-domain-5)
- [Domain 6: Monitoring, Logging & Runtime Security (20%)](#cks-domain-6)

### Master Cheatsheets
- [CKA Master Cheatsheet](#cka-master-cheatsheet)
- [CKAD Master Cheatsheet](#ckad-master-cheatsheet)
- [CKS Master Cheatsheet](#cks-master-cheatsheet)

---

# CKA — Certified Kubernetes Administrator

| Domain | Weight |
|---|---|
| Cluster Architecture, Installation & Configuration | 25% |
| Workloads & Scheduling | 15% |
| Services & Networking | 20% |
| Storage | 10% |
| Troubleshooting | **30%** ← highest weight |

---

## CKA Domain 1: Cluster Architecture, Installation & Configuration (25%) {#cka-domain-1}

---

### 🟢 Q1. What is a Kubernetes cluster and what are its core components?

**Explanation:**
A Kubernetes cluster is a group of machines (nodes) that collectively run containerised workloads managed by the Kubernetes control plane. The cluster has two types of machines: **control plane nodes** that make decisions about the cluster (scheduling, maintaining desired state, responding to events), and **worker nodes** that actually run the application containers.

The key insight is the separation of concerns: the control plane handles *what should run and where*, while worker nodes handle *actually running it*. This separation allows Kubernetes to recover from node failures, scale workloads, and maintain the desired state without manual intervention.

**Control Plane Components:**

| Component | Role | Where it runs |
|---|---|---|
| `kube-apiserver` | Single entry point for all API calls; validates and processes REST requests; stores state in etcd | Control plane |
| `etcd` | Distributed, consistent key-value store; only the API server reads/writes to etcd | Control plane |
| `kube-scheduler` | Watches for unscheduled pods; selects the best node based on resource requirements, affinity, taints, etc. | Control plane |
| `kube-controller-manager` | Runs controller loops: node controller, replication controller, endpoints controller, etc. | Control plane |
| `cloud-controller-manager` | Interacts with cloud provider APIs (load balancers, nodes, routes) | Control plane |

**Worker Node Components:**

| Component | Role |
|---|---|
| `kubelet` | Primary node agent; ensures containers described in PodSpecs are running and healthy |
| `kube-proxy` | Maintains network rules (iptables/IPVS) to implement Services |
| `Container Runtime` | Software that runs containers: containerd, CRI-O |

```bash
# Inspect control plane pods
kubectl get pods -n kube-system

# Check component status
kubectl get componentstatuses

# View API server manifest (kubeadm clusters)
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Check kubelet on a node
systemctl status kubelet
journalctl -u kubelet -n 50
```

---

### 🟢 Q2. How do you bootstrap a Kubernetes cluster with kubeadm?

**Explanation:**
`kubeadm` is the official tool for bootstrapping production-grade Kubernetes clusters. It handles the complex task of generating TLS certificates, configuring the API server, setting up etcd, creating the kubeconfig files, and generating the join tokens for worker nodes.

The process follows a clear sequence:
1. Install container runtime and kubeadm/kubelet/kubectl on all nodes
2. Run `kubeadm init` on the control plane — this generates all certificates, starts static pods, and prints the join command
3. Configure kubectl to talk to the new cluster
4. Install a CNI plugin so pods can communicate
5. Join worker nodes using the token from step 2

**Important:** Without a CNI plugin, pods will remain in `Pending` state because there is no network for them.

```bash
# Step 1: Pre-flight — disable swap (required by kubelet)
swapoff -a
sed -i '/swap/d' /etc/fstab

# Step 2: Load required kernel modules
modprobe overlay
modprobe br_netfilter
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Step 3: Set kernel parameters
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# Step 4: Install containerd + kubeadm/kubelet/kubectl
apt-get install -y containerd kubeadm kubelet kubectl
apt-mark hold kubelet kubeadm kubectl

# Step 5: Bootstrap control plane
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER_IP> \
  --kubernetes-version=v1.29.0

# Step 6: Configure kubectl for current user
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Step 7: Install CNI (Calico example)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Step 8: Join worker nodes (use command printed by kubeadm init)
kubeadm join <MASTER_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# Generate a new join token if old one expired
kubeadm token create --print-join-command

# Verify cluster is healthy
kubectl get nodes
kubectl get pods -n kube-system
```

---

### 🟢 Q3. How does RBAC work in Kubernetes?

**Explanation:**
RBAC (Role-Based Access Control) is the primary authorisation mechanism in Kubernetes. It answers the question: **"Is this user/service allowed to perform this action on this resource?"**

The model has four key objects:
- **Role** — defines a set of permissions (rules) *within a namespace*
- **ClusterRole** — same as Role but *cluster-wide* (also used for non-namespaced resources like nodes)
- **RoleBinding** — attaches a Role to subjects (users, groups, service accounts) *in a namespace*
- **ClusterRoleBinding** — attaches a ClusterRole to subjects *cluster-wide*

A key subtlety: a **ClusterRole can be bound with a RoleBinding** — this gives the subject the ClusterRole's permissions but only within the namespace of the RoleBinding. This is useful for reusing common permission sets across namespaces.

RBAC is **additive only** — you cannot explicitly deny actions. If a subject has no matching rule, the request is denied by default.

```yaml
# Role — pod reader in 'dev' namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]

---
# ClusterRole — read nodes (non-namespaced)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding — bind pod-reader to 'jane' in 'dev'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-pod-reader
  namespace: dev
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: my-app-sa
  namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# ClusterRoleBinding — bind cluster-admin to 'ops-team' group
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-team-admin
subjects:
- kind: Group
  name: ops-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Imperative RBAC creation
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  --namespace=dev

kubectl create rolebinding jane-pod-reader \
  --role=pod-reader \
  --user=jane \
  --namespace=dev

kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

kubectl create clusterrolebinding ops-admin \
  --clusterrole=cluster-admin \
  --group=ops-team

# Test permissions
kubectl auth can-i get pods --namespace=dev --as=jane
kubectl auth can-i delete nodes --as=jane          # should be no
kubectl auth can-i '*' '*' --as=system:admin       # should be yes

# List roles and bindings
kubectl get roles,rolebindings -n dev
kubectl get clusterroles,clusterrolebindings
kubectl describe rolebinding jane-pod-reader -n dev
```

---

### 🟡 Q4. How do you perform a cluster upgrade using kubeadm?

**Explanation:**
Upgrading a Kubernetes cluster is a critical operational task. Kubernetes supports upgrading **one minor version at a time** (e.g. 1.27 → 1.28, not 1.27 → 1.29). The process must be done node by node to maintain availability.

The correct order is:
1. **Upgrade control plane first** — kubeadm, then apply the upgrade, then kubelet
2. **Upgrade worker nodes one at a time** — drain (evict pods), upgrade, uncordon

**Why drain before upgrading a node?** Draining gracefully moves all workloads off the node so there's no disruption during the upgrade. `--ignore-daemonsets` is needed because DaemonSet pods can't be moved — they'll be recreated on the node after the upgrade.

```bash
# ── CONTROL PLANE ──────────────────────────────────────────────

# Step 1: Check available versions
apt-cache madison kubeadm | head -5

# Step 2: Unhold and upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# Step 3: Verify kubeadm upgrade plan
kubeadm upgrade plan
# Shows: current version, latest version, component changes

# Step 4: Apply the upgrade (control plane components)
kubeadm upgrade apply v1.29.0
# This upgrades: kube-apiserver, kube-scheduler, kube-controller-manager,
# etcd, CoreDNS, kube-proxy

# Step 5: Upgrade kubelet and kubectl on control plane
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet

# Step 6: Verify control plane is upgraded
kubectl get nodes   # control plane should show v1.29.0

# ── WORKER NODES (repeat for each worker) ──────────────────────

# Step 7: Drain the worker node (from control plane)
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

# Step 8: On the worker node — upgrade kubeadm
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# Step 9: Upgrade node configuration
kubeadm upgrade node

# Step 10: Upgrade kubelet on worker
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet

# Step 11: Uncordon from control plane
kubectl uncordon worker-1

# Verify all nodes are Ready with new version
kubectl get nodes
```

---

### 🟡 Q5. How do you back up and restore etcd?

**Explanation:**
etcd is the most critical component of a Kubernetes cluster — it stores the entire cluster state. Without it, the cluster cannot function. Regular etcd backups are essential for disaster recovery.

The backup creates a **snapshot** of the entire etcd data. A restore creates a new data directory from the snapshot. After restore, you must update the etcd configuration to point to the new data directory.

**Important:** Always use `ETCDCTL_API=3`. Always provide the TLS certificates — etcd requires mutual TLS authentication.

```bash
# ── BACKUP ──────────────────────────────────────────────────────

# Find etcd TLS cert paths (from the static pod manifest)
cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'cert|key|trusted-ca'

# Take snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the backup is valid
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-*.db \
  --write-out=table
# Shows: hash, revision, total keys, total size

# ── RESTORE ─────────────────────────────────────────────────────

# Step 1: Restore snapshot to a new data directory
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-backup.db \
  --data-dir=/var/lib/etcd-new \
  --name=master \
  --initial-cluster=master=https://127.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# Step 2: Update etcd static pod manifest to use new data dir
# Edit /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd-new
# AND update the hostPath volume:
#   path: /var/lib/etcd  →  path: /var/lib/etcd-new

# Step 3: Wait for etcd pod to restart
watch kubectl get pods -n kube-system | grep etcd

# Step 4: Verify cluster is healthy
kubectl get nodes
kubectl get pods -A
```

---

### 🟡 Q6. How do you configure kubeconfig for multiple clusters?

**Explanation:**
`kubeconfig` is the configuration file that `kubectl` uses to determine which cluster to talk to and how to authenticate. It can contain multiple clusters, users, and contexts, making it easy to switch between environments.

A **context** binds together: a cluster (API server URL + CA), a user (credentials), and optionally a default namespace. `kubectl config use-context` switches the active context.

```yaml
# ~/.kube/config — example multi-cluster config
apiVersion: v1
kind: Config
current-context: dev

clusters:
- name: prod-cluster
  cluster:
    server: https://prod-api.example.com:6443
    certificate-authority-data: <base64-ca-cert>
- name: dev-cluster
  cluster:
    server: https://dev-api.example.com:6443
    certificate-authority-data: <base64-ca-cert>
- name: staging-cluster
  cluster:
    server: https://staging-api:6443
    insecure-skip-tls-verify: true   # dev only — not for prod!

users:
- name: prod-admin
  user:
    client-certificate-data: <base64-cert>
    client-key-data: <base64-key>
- name: dev-user
  user:
    token: <bearer-token>
- name: staging-user
  user:
    username: admin
    password: secret

contexts:
- name: prod
  context:
    cluster: prod-cluster
    user: prod-admin
    namespace: production
- name: dev
  context:
    cluster: dev-cluster
    user: dev-user
    namespace: development
```

```bash
# Switch context
kubectl config use-context prod
kubectl config use-context dev

# View all contexts
kubectl config get-contexts
# CURRENT   NAME   CLUSTER         AUTHINFO      NAMESPACE
# *         prod   prod-cluster    prod-admin    production

# Get current context
kubectl config current-context

# View full config
kubectl config view
kubectl config view --minify   # only current context

# Set default namespace for current context
kubectl config set-context --current --namespace=my-namespace

# Merge multiple kubeconfig files
KUBECONFIG=~/.kube/config:~/.kube/prod-config:~/.kube/dev-config \
  kubectl config view --flatten > ~/.kube/merged-config

# Add a new cluster to existing config
kubectl config set-cluster new-cluster \
  --server=https://new-api:6443 \
  --certificate-authority=/path/to/ca.crt

# Add credentials
kubectl config set-credentials new-user \
  --client-certificate=/path/to/client.crt \
  --client-key=/path/to/client.key

# Create context
kubectl config set-context new-context \
  --cluster=new-cluster \
  --user=new-user \
  --namespace=default
```

---

### 🟡 Q7. What is Helm and how do you use it to manage Kubernetes applications?

**Explanation:**
Helm is the Kubernetes package manager. It solves a fundamental problem: Kubernetes applications often consist of many inter-related resources (Deployments, Services, ConfigMaps, RBAC, etc.) that need to be deployed, upgraded, and removed together.

A **Helm chart** packages all these resources together with templating (Go templates) and a `values.yaml` file that allows customisation without modifying the templates directly.

Key concepts:
- **Chart** — the package (like a .deb or .rpm)
- **Release** — an installed instance of a chart in the cluster
- **Repository** — a collection of charts (like apt/yum repos)
- **Values** — configuration that overrides chart defaults
- **Revision** — each upgrade creates a new revision, enabling rollbacks

```bash
# ── REPOSITORY MANAGEMENT ────────────────────────────────────────
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update               # refresh repo index
helm repo list                 # list configured repos
helm repo remove stable        # remove a repo

# ── SEARCHING ────────────────────────────────────────────────────
helm search repo nginx         # search in added repos
helm search hub wordpress      # search Artifact Hub
helm show values bitnami/nginx # show all configurable values

# ── INSTALLING ───────────────────────────────────────────────────
# Basic install
helm install my-nginx bitnami/nginx

# Install with custom values
helm install my-nginx bitnami/nginx \
  --namespace ingress \
  --create-namespace \
  --values custom-values.yaml \
  --set replicaCount=3 \
  --set service.type=LoadBalancer

# Dry run (shows rendered manifests without applying)
helm install my-nginx bitnami/nginx --dry-run --debug

# Install from local chart directory
helm install my-app ./my-chart/

# Install a specific version
helm install my-nginx bitnami/nginx --version 13.2.0

# Wait for resources to be ready
helm install my-nginx bitnami/nginx --wait --timeout 5m

# ── INSPECTING ───────────────────────────────────────────────────
helm list                     # list releases in current namespace
helm list -A                  # all namespaces
helm status my-nginx          # detailed release status
helm get values my-nginx      # values used for this release
helm get manifest my-nginx    # rendered Kubernetes manifests
helm history my-nginx         # revision history

# ── UPGRADING ────────────────────────────────────────────────────
helm upgrade my-nginx bitnami/nginx \
  --set replicaCount=5 \
  --reuse-values               # keep previous values, only override specified

helm upgrade --install my-nginx bitnami/nginx  # install if not exists

# ── ROLLBACK ─────────────────────────────────────────────────────
helm rollback my-nginx 1       # roll back to revision 1
helm rollback my-nginx         # roll back to previous revision

# ── UNINSTALLING ─────────────────────────────────────────────────
helm uninstall my-nginx
helm uninstall my-nginx --keep-history  # keep history for rollback capability

# ── TEMPLATING ───────────────────────────────────────────────────
helm template my-nginx bitnami/nginx --values custom-values.yaml
helm lint ./my-chart/          # validate chart structure
```

**Chart directory structure:**
```
my-chart/
  Chart.yaml          # Metadata: name, version, appVersion, dependencies
  values.yaml         # Default values
  charts/             # Dependencies (sub-charts)
  templates/
    _helpers.tpl      # Template helpers (define blocks)
    deployment.yaml
    service.yaml
    ingress.yaml
    NOTES.txt         # Printed after installation
  .helmignore         # Files to ignore during packaging
```

---

### 🟡 Q8. What is Kustomize and how does it differ from Helm?

**Explanation:**
Kustomize takes a different approach to Kubernetes configuration management compared to Helm. Instead of templates with placeholder variables, Kustomize uses **patches and overlays** applied on top of plain Kubernetes YAML files.

The philosophy: your base manifests are valid Kubernetes YAML that can be applied directly. Kustomize only adds transformations on top — changing image tags, adding labels, patching replica counts, etc. This makes the base manifests always readable and valid, unlike Helm templates which contain Go template syntax.

**Kustomize is built into kubectl** (`kubectl apply -k`), so no extra tool installation is needed.

| Aspect | Helm | Kustomize |
|---|---|---|
| Approach | Go templates with values files | Patch overlays on plain YAML |
| Learning curve | Steeper (Go template syntax) | Gentler (just YAML) |
| Package sharing | Charts in Helm repos | Git repos / OCI registries |
| Built into kubectl | No | Yes (`kubectl apply -k`) |
| Secret handling | Needs plugins | Has secretGenerator |
| Rollback support | Yes (Helm history) | No (use GitOps) |
| Best for | Packaging third-party apps | Environment-specific patches |

```
project/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── replica-patch.yaml
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
commonLabels:
  app: my-app

# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: production
resources:
- ../../base

# Override image tag
images:
- name: my-app
  newTag: v2.1.0

# Add environment-specific labels
commonLabels:
  env: production

# JSON patch — change replicas to 5
patches:
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
  target:
    kind: Deployment
    name: my-app

# Strategic merge patch — add resource limits
- path: resource-patch.yaml
  target:
    kind: Deployment
    name: my-app

# Add ConfigMap from literals
configMapGenerator:
- name: app-config
  literals:
  - ENV=production
  - LOG_LEVEL=warn

# Add Secret from file
secretGenerator:
- name: db-secret
  files:
  - password.txt
  type: Opaque
```

```bash
# Preview output
kubectl kustomize overlays/prod/

# Apply
kubectl apply -k overlays/prod/

# Diff (what would change)
kubectl diff -k overlays/prod/

# Delete
kubectl delete -k overlays/prod/
```

---

### 🟡 Q9. What are CRDs and Operators?

**Explanation:**
**Custom Resource Definitions (CRDs)** extend the Kubernetes API with your own resource types. Once you define a CRD, Kubernetes treats it like any first-class resource — you can `kubectl get`, `kubectl describe`, and apply YAML for it. etcd stores CRD instances just like built-in objects.

CRDs alone are just data storage. The real power comes from **Operators** — software that watches for CRD instances and takes action to manage complex applications. An Operator encodes operational knowledge: how to install, upgrade, back up, and recover an application.

The pattern is: **Operator = CRD (desired state) + Controller (reconciliation loop)**

Popular operators:
- **cert-manager** — automatically provisions TLS certificates
- **prometheus-operator** — manages Prometheus and Alertmanager
- **strimzi** — manages Apache Kafka
- **postgres-operator** — manages PostgreSQL clusters

```yaml
# Step 1: Define the CRD
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.io
spec:
  group: mycompany.io
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: ["engine", "storage"]
            properties:
              engine:
                type: string
                enum: ["postgres", "mysql", "mongodb"]
              version:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 5
              storage:
                type: string
                pattern: '^[0-9]+(Gi|Ti)$'
          status:
            type: object
            properties:
              phase:
                type: string
              readyReplicas:
                type: integer
    additionalPrinterColumns:
    - name: Engine
      type: string
      jsonPath: .spec.engine
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Status
      type: string
      jsonPath: .status.phase
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames: [db]
```

```yaml
# Step 2: Create a custom resource instance
apiVersion: mycompany.io/v1
kind: Database
metadata:
  name: production-db
  namespace: data
spec:
  engine: postgres
  version: "15.2"
  replicas: 3
  storage: 100Gi
```

```bash
# Work with custom resources
kubectl get crds
kubectl get databases -A
kubectl get db production-db -n data    # shortName
kubectl describe db production-db -n data
kubectl explain database.spec

# Install an operator via Helm (cert-manager example)
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Check CRDs registered by the operator
kubectl get crds | grep cert-manager.io
```

---

### 🟡 Q10. What are CNI, CSI, and CRI interfaces?

**Explanation:**
These three interfaces define the plugin architecture that makes Kubernetes vendor-neutral and extensible:

**CRI (Container Runtime Interface):** Kubernetes used to have code directly supporting Docker. This created a problem: every new runtime required changing the Kubernetes source code. CRI solves this by defining a standard gRPC API. Any runtime implementing this API works with Kubernetes. `kubelet` talks to the CRI runtime via a Unix socket.

**CNI (Container Network Interface):** Kubernetes itself doesn't implement pod networking — it only specifies the requirements (every pod gets a unique IP, pods can communicate without NAT). CNI plugins implement these requirements. When kubelet creates a pod, it calls the configured CNI plugin to set up the network namespace.

**CSI (Container Storage Interface):** Similar to CNI but for storage. Previously adding a new storage backend required changes to Kubernetes core. CSI allows storage vendors to write out-of-tree drivers. `kubectl get csidriver` shows installed CSI drivers.

| Interface | What it abstracts | Examples |
|---|---|---|
| CRI | Container runtime (pulling images, running containers) | containerd, CRI-O |
| CNI | Pod networking (IP assignment, routing) | Calico, Cilium, Flannel |
| CSI | Storage backends (provisioning, mounting volumes) | AWS EBS, GCP PD, Ceph |

```bash
# ── CRI ────────────────────────────────────────────────────────
# Check runtime being used
kubectl get nodes -o wide      # CONTAINER-RUNTIME column
crictl info                    # detailed runtime info
crictl ps                      # list running containers
crictl images                  # list cached images
crictl logs <container-id>     # container logs

# CRI socket locations
# containerd: /run/containerd/containerd.sock
# CRI-O:      /var/run/crio/crio.sock

# ── CNI ────────────────────────────────────────────────────────
# Check CNI plugins installed
ls /opt/cni/bin/
ls /etc/cni/net.d/         # active CNI configs

# Check CNI pod health (Calico example)
kubectl get pods -n kube-system | grep calico

# Verify pod networking works
kubectl run test-net --image=busybox --rm -it \
  -- wget -qO- https://kubernetes.io

# ── CSI ────────────────────────────────────────────────────────
# List CSI drivers in cluster
kubectl get csidrivers
kubectl get csinodes

# Check storage capacity tracking
kubectl get csistoragecapacities -A

# Verify dynamic provisioning
kubectl get storageclass
kubectl describe storageclass standard
```

---

### 🔴 Q11. How do you set up a highly available control plane?

**Explanation:**
A single control plane node is a single point of failure. HA requires multiple control plane nodes, each running all control plane components, with a load balancer distributing traffic to the API servers.

etcd consensus requires a quorum: **more than half the nodes must be healthy**. This means you need an odd number of etcd nodes (3 or 5) — with 3 nodes you can tolerate 1 failure, with 5 nodes you can tolerate 2.

Two HA topologies:
- **Stacked etcd** — etcd runs on the same nodes as control plane components (simpler, less infrastructure)
- **External etcd** — etcd runs on dedicated nodes (more resilient, more infrastructure)

The key to HA is that scheduler and controller-manager use **leader election** — only one instance is active at a time, but others are ready to take over immediately on failure.

```bash
# ── PREREQUISITES ─────────────────────────────────────────────
# Set up load balancer (HAProxy/NLB) pointing to all control plane nodes
# LB_IP:6443 → [cp-node-1:6443, cp-node-2:6443, cp-node-3:6443]

# ── FIRST CONTROL PLANE NODE ──────────────────────────────────
kubeadm init \
  --control-plane-endpoint "LB_IP:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# Save output — you'll get TWO join commands:
# 1. For additional control plane nodes (includes --control-plane --certificate-key)
# 2. For worker nodes (no --control-plane)

# Set up kubectl
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config

# Install CNI
kubectl apply -f calico.yaml

# ── ADDITIONAL CONTROL PLANE NODES ────────────────────────────
kubeadm join LB_IP:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key>    # from --upload-certs

# ── WORKER NODES ───────────────────────────────────────────────
kubeadm join LB_IP:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# ── VERIFY ─────────────────────────────────────────────────────
kubectl get nodes
# NAME         STATUS   ROLES           AGE
# cp-node-1    Ready    control-plane   10m
# cp-node-2    Ready    control-plane   8m
# cp-node-3    Ready    control-plane   6m
# worker-1     Ready    <none>          3m

# Check leader election
kubectl get endpoints kube-scheduler -n kube-system -o yaml
kubectl get endpoints kube-controller-manager -n kube-system -o yaml

# Check etcd cluster health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://cp-1:2379,https://cp-2:2379,https://cp-3:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## CKA Domain 2: Workloads & Scheduling (15%) {#cka-domain-2}

---

### 🟢 Q12. What are taints and tolerations?

**Explanation:**
Taints and tolerations are a mechanism to **repel pods from nodes** unless the pod explicitly tolerates the taint. They work like a lock (taint on the node) and key (toleration in the pod spec).

Use cases:
- **Dedicated nodes** — taint GPU nodes so only GPU workloads run on them
- **Node with special hardware** — taint nodes with SSDs or large memory
- **Node maintenance** — mark a node as not schedulable while keeping existing pods
- **Control plane isolation** — control plane nodes are tainted `node-role.kubernetes.io/control-plane:NoSchedule`

The three taint effects have different behaviours:
- `NoSchedule` — new pods without toleration won't be scheduled. Existing pods are NOT evicted.
- `PreferNoSchedule` — the scheduler tries to avoid placing pods without toleration, but will if necessary
- `NoExecute` — both prevents scheduling AND evicts existing pods that don't tolerate it (optionally after a `tolerationSeconds` grace period)

```bash
# Add taints
kubectl taint nodes node-1 env=prod:NoSchedule
kubectl taint nodes node-1 gpu=true:NoSchedule
kubectl taint nodes node-1 maintenance=true:NoExecute

# Remove taint (note the minus sign at the end)
kubectl taint nodes node-1 env=prod:NoSchedule-
kubectl taint nodes node-1 maintenance=true:NoExecute-

# View taints on a node
kubectl describe node node-1 | grep Taint
```

```yaml
spec:
  tolerations:
  # Exact match
  - key: "env"
    operator: "Equal"
    value: "prod"
    effect: "NoSchedule"
  # Tolerate any value for the key
  - key: "gpu"
    operator: "Exists"
    effect: "NoSchedule"
  # Tolerate NoExecute for up to 3600 seconds (then evicted)
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 3600
  # Tolerate ALL taints on the node
  - operator: "Exists"
```

---

### 🟢 Q13. What are resource requests and limits?

**Explanation:**
Resource requests and limits allow Kubernetes to make intelligent scheduling decisions and prevent noisy neighbours.

**Requests** are what the container is **guaranteed** to get. The scheduler uses requests to determine which nodes have enough capacity. A container will always have at least its requested amount available.

**Limits** are the **maximum** a container can use. If a container tries to use more CPU than its limit, it is throttled. If it uses more memory than its limit, the container is OOMKilled (killed with Out Of Memory signal).

The **QoS class** affects eviction priority during node pressure:
- `Guaranteed` (requests = limits): evicted last
- `Burstable` (requests < limits): evicted after BestEffort
- `BestEffort` (no requests/limits): evicted first

CPU is measured in millicores (`m`): `1000m = 1 CPU core`. Memory uses binary suffixes: `Mi` (mebibytes), `Gi` (gibibytes).

```yaml
spec:
  containers:
  - name: app
    image: my-app:v1
    resources:
      requests:
        memory: "128Mi"    # Guaranteed this much
        cpu: "250m"        # 0.25 of a CPU core
      limits:
        memory: "256Mi"    # OOMKilled if exceeded
        cpu: "500m"        # Throttled if exceeded
```

```bash
# View resource usage
kubectl top nodes
kubectl top pods
kubectl top pods --containers   # per-container breakdown
kubectl top pods -A --sort-by=memory

# Describe pod to see QoS class
kubectl describe pod my-pod | grep QoS

# Check nodes have enough capacity
kubectl describe nodes | grep -A 5 "Allocated resources"
```

---

### 🟡 Q14. What is node affinity?

**Explanation:**
Node affinity is a more expressive replacement for `nodeSelector`. While `nodeSelector` only supports exact equality checks, node affinity supports operators like `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`.

There are two types:
- **requiredDuringSchedulingIgnoredDuringExecution** — "hard" requirement; pod will not be scheduled if no matching node exists. The `IgnoredDuringExecution` part means if a node's labels change after scheduling, the pod is NOT evicted.
- **preferredDuringSchedulingIgnoredDuringExecution** — "soft" preference with a weight (1-100); scheduler tries to honour it but will place the pod elsewhere if needed.

**Pod affinity/anti-affinity** is the same concept but between pods — "schedule this pod on the same node as pods with label X" (affinity) or "never schedule this pod on a node that already has pods with label Y" (anti-affinity).

```yaml
affinity:
  nodeAffinity:
    # REQUIRED: must have disktype=ssd or disktype=nvme
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: [ssd, nvme]
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
    # PREFERRED: prefer nodes in us-east-1a (weight 80)
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values: [us-east-1a]
    - weight: 20
      preference:
        matchExpressions:
        - key: node-type
          operator: In
          values: [high-memory]

  # Pod anti-affinity — don't schedule on node that already has app=my-app
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: [my-app]
      topologyKey: kubernetes.io/hostname   # unique per node
```

---

### 🟡 Q15. How does HPA work?

**Explanation:**
The Horizontal Pod Autoscaler (HPA) automatically adjusts the number of pod replicas based on observed metrics. It queries `metrics-server` (or a custom metrics adapter) every 15 seconds and calculates the desired replicas using:

`desiredReplicas = ceil(currentReplicas × (currentMetric / desiredMetric))`

For example: 3 pods at 80% CPU with a target of 50% → `ceil(3 × 80/50)` = `ceil(4.8)` = 5 replicas.

**Requirements:**
- `metrics-server` must be installed for CPU/memory metrics
- Containers must have resource requests set (HPA uses requests as the 100% baseline)
- For custom metrics, a custom metrics adapter (like Prometheus Adapter) is needed

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # CPU utilization (relative to requests)
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # scale when avg CPU > 70% of request
  # Memory utilization
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0     # scale up immediately
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60              # add max 4 pods/min
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60              # remove max 10%/min
```

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Create HPA imperatively
kubectl autoscale deployment my-app \
  --cpu-percent=70 \
  --min=2 \
  --max=10

# Monitor HPA status
kubectl get hpa
kubectl describe hpa my-app-hpa
# Shows: Targets, MinPods, MaxPods, Replicas, Conditions

# Generate load for testing
kubectl run load-gen --image=busybox --rm -it \
  -- sh -c "while true; do wget -qO- http://my-app; done"
```

---

### 🟡 Q16. What is VPA and how does it differ from HPA?

**Explanation:**
The Vertical Pod Autoscaler (VPA) automatically adjusts **resource requests and limits** for containers based on actual historical usage. While HPA scales *out* (more pods), VPA scales *up* (bigger pods).

VPA is particularly useful when:
- You don't know the right resource requests for a new application
- Your workload's resource needs change over time
- You want to avoid over-provisioning or under-provisioning

VPA has four update modes:
- **Off** — only shows recommendations, doesn't apply them
- **Initial** — only sets resources when pod is first created
- **Recreate** — evicts pods and recreates them with new resources when needed
- **Auto** — same as Recreate currently, will use in-place updates in future

**Why not use HPA and VPA together on CPU/memory?** They can interfere — HPA scales based on current CPU %, but VPA is changing the CPU request baseline. Use HPA for custom metrics and VPA for CPU/memory, or use HPA only.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"    # Off | Initial | Recreate | Auto
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 50m
        memory: 64Mi
      maxAllowed:
        cpu: "4"
        memory: 4Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
```

```bash
# Install VPA (requires metrics-server)
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-install.sh

# Check VPA recommendations
kubectl describe vpa my-app-vpa
# Shows:
#   Lower Bound: cpu 25m, memory 32Mi
#   Target:      cpu 100m, memory 128Mi
#   Upper Bound: cpu 500m, memory 512Mi
#   Uncapped Target: cpu 89m, memory 115Mi

kubectl get vpa
```

---

### 🔴 Q17. What is a PriorityClass and how does preemption work?

**Explanation:**
PriorityClass allows you to express the relative importance of pods. When the cluster is full and a high-priority pod cannot be scheduled, Kubernetes can **preempt** (evict) lower-priority pods to make room.

**Preemption process:**
1. High-priority pod P cannot be scheduled (no node has enough resources)
2. Scheduler scans all nodes looking for candidates where evicting lower-priority pods would free enough resources
3. If found, lower-priority pods on that node are gracefully terminated
4. Pod P is placed on that node

**System priority classes** (don't modify these):
- `system-cluster-critical`: 2,000,000,000 — for cluster-critical components (CoreDNS, CNI)
- `system-node-critical`: 2,000,001,000 — for node-critical components (kubelet, container runtime)

You can set `preemptionPolicy: Never` to prevent a priority class from preempting others (pod stays pending instead of evicting).

```yaml
# Create priority classes
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-workload
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority   # default
description: "Critical production workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low
value: 100
globalDefault: false
preemptionPolicy: Never                  # won't preempt others
description: "Low-priority batch jobs"
---
# Default for all pods without explicit priorityClassName
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default
value: 500
globalDefault: true
```

```yaml
# Use in pod spec
spec:
  priorityClassName: critical-workload
  containers:
  - name: app
    image: my-app:v1
```

```bash
kubectl get priorityclasses
kubectl describe priorityclass critical-workload
```

---

## CKA Domain 3: Services & Networking (20%) {#cka-domain-3}

---

### 🟢 Q18. What are the different Service types and when do you use each?

**Explanation:**
Services provide a stable network identity for a dynamic set of pods (selected by labels). Since pods are ephemeral and their IPs change, Services give you a stable IP and DNS name that load-balances across matching pods.

| Type | Exposes | Use Case |
|---|---|---|
| `ClusterIP` | Internal cluster IP only | Microservice-to-microservice communication |
| `NodePort` | ClusterIP + port on every node | Simple external access in dev/test |
| `LoadBalancer` | NodePort + cloud load balancer | Production external access (cloud) |
| `ExternalName` | DNS CNAME alias | Connect pods to external services by name |
| `Headless` | No ClusterIP, DNS only | StatefulSets, direct pod-to-pod discovery |

```yaml
# ClusterIP (default)
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80          # service port
    targetPort: 8080  # container port
    protocol: TCP

---
# NodePort — accessible at <NodeIP>:30080
apiVersion: v1
kind: Service
metadata:
  name: frontend-np
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080   # optional: specify port (30000-32767)

---
# LoadBalancer — cloud provisions external LB
apiVersion: v1
kind: Service
metadata:
  name: api-lb
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - port: 443
    targetPort: 8443

---
# ExternalName — maps to external DNS
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: prod-db.example.com

---
# Headless — no ClusterIP, returns pod IPs directly
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None    # headless
  selector:
    app: mysql
  ports:
  - port: 3306
```

```bash
# Inspect services
kubectl get svc
kubectl describe svc backend
kubectl get endpoints backend   # shows pod IPs backing the service

# Test service from inside cluster
kubectl run test --image=busybox --rm -it \
  -- wget -qO- http://backend.default.svc.cluster.local

# Test NodePort
curl http://<node-ip>:30080
```

---

### 🟢 Q19. What is a NetworkPolicy and how do you write one?

**Explanation:**
By default, **all pods can communicate with all other pods** in a Kubernetes cluster. NetworkPolicies allow you to restrict this. They work at Layer 3/4 (IP/port level) using label selectors.

**Key concepts:**
- NetworkPolicies are **namespace-scoped** and select pods within that namespace using `podSelector`
- If NO NetworkPolicy selects a pod, the pod has open ingress and egress
- Once ANY NetworkPolicy selects a pod for a direction (ingress/egress), only explicitly allowed traffic flows
- NetworkPolicies are **additive** — you can have multiple policies and their rules combine with OR logic
- Requires a CNI plugin that supports NetworkPolicy: Calico, Cilium, Weave Net (NOT Flannel alone)

```yaml
# Allow ingress to backend only from frontend pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    # OR logic between list items; AND logic within an item
    - podSelector:
        matchLabels:
          app: frontend
      namespaceSelector:         # AND: must be from production namespace
        matchLabels:
          name: production
    - namespaceSelector:         # OR: allow from monitoring namespace
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - ports:                       # Allow DNS
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

```bash
# Test NetworkPolicy is working
kubectl exec -n production frontend-pod -- \
  wget -qO- http://backend:8080         # should work

kubectl exec -n production other-pod -- \
  wget -qO- http://backend:8080         # should fail

# Debug NetworkPolicy
kubectl get networkpolicies -A
kubectl describe networkpolicy backend-allow-frontend -n production
```

---

### 🟡 Q20. How does Kubernetes DNS work?

**Explanation:**
CoreDNS is the default DNS server in Kubernetes clusters. It runs as a Deployment in `kube-system` and all pods are configured (via `/etc/resolv.conf`) to use it for name resolution.

DNS records are automatically created for Services and pods. The format follows a predictable pattern, allowing pods to discover services by name without hardcoding IP addresses.

**Service DNS records:**
- `<service>.<namespace>.svc.cluster.local` → ClusterIP of the service
- For headless services: returns all pod IPs (A records)
- For ExternalName services: returns a CNAME

**Pod DNS records:**
- `<pod-ip-with-dashes>.<namespace>.pod.cluster.local`

**Search domains** in `/etc/resolv.conf` allow short names:
- `my-service` → resolves because `default.svc.cluster.local` is in search path
- `my-service.other-ns` → resolves cross-namespace

```bash
# Inspect CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get configmap coredns -n kube-system -o yaml
kubectl logs -n kube-system -l k8s-app=kube-dns

# Debug DNS from inside a pod
kubectl run dns-test --image=busybox --rm -it -- sh

# Inside the pod:
cat /etc/resolv.conf
# search default.svc.cluster.local svc.cluster.local cluster.local
# nameserver 10.96.0.10   (CoreDNS ClusterIP)

nslookup kubernetes.default           # short form
nslookup kubernetes.default.svc.cluster.local  # FQDN
nslookup my-service.other-namespace   # cross-namespace

# Check CoreDNS service
kubectl get svc kube-dns -n kube-system

# Edit CoreDNS configmap to customise
kubectl edit configmap coredns -n kube-system
```

---

### 🟡 Q21. What is the Gateway API and how does it improve on Ingress?

**Explanation:**
The Ingress API has significant limitations: it's too simple for many use cases, relies heavily on annotations (which are not standardised across controllers), and conflates infrastructure management (provisioning a load balancer) with application routing concerns.

The **Gateway API** redesigns Kubernetes network routing with role-oriented separation:

- **Infrastructure Admin** creates `GatewayClass` — defines the type of load balancer
- **Cluster Operator** creates `Gateway` — instantiates a load balancer with specific listeners and TLS
- **Developer** creates `HTTPRoute` (or `GRPCRoute`, `TCPRoute`) — defines routing rules

This separation is critical in large organisations: platform teams manage the infrastructure, while application teams independently manage their routing rules without needing cluster-admin access.

```yaml
# Step 1: GatewayClass (infrastructure admin)
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-gateway
spec:
  controllerName: k8s-gateway.nginx.org/nginx-gateway-controller

---
# Step 2: Gateway (cluster operator — controls infra)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: infra
spec:
  gatewayClassName: nginx-gateway
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All   # allow routes from any namespace
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: wildcard-tls-secret
        namespace: infra
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: "true"

---
# Step 3: HTTPRoute (developer — no infra access needed)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-routes
  namespace: my-team
spec:
  parentRefs:
  - name: main-gateway
    namespace: infra
    sectionName: https
  hostnames: ["myapp.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
      headers:
      - name: X-Version
        value: v2
    backendRefs:
    - name: api-v2-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-v1-service
      port: 80
  # Traffic splitting (canary)
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-stable
      port: 80
      weight: 90
    - name: frontend-canary
      port: 80
      weight: 10
```

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml

# Check gateway status
kubectl get gateways -A
kubectl describe gateway main-gateway -n infra

# Check routes
kubectl get httproutes -A
```

---

### 🟡 Q22. How do you set up an Ingress controller?

**Explanation:**
An Ingress controller is a pod running in the cluster that watches `Ingress` resources and configures a reverse proxy (NGINX, Traefik, HAProxy) accordingly. The `Ingress` resource itself is just configuration — you MUST have a controller installed for Ingress to work.

Multiple controllers can coexist in the same cluster using `ingressClassName` to target the right controller.

```bash
# ── NGINX INGRESS CONTROLLER (most common) ──────────────────────

# Install via Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx   # check EXTERNAL-IP

# ── INGRESS RESOURCE ────────────────────────────────────────────
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
  - hosts: [myapp.example.com, api.example.com]
    secretName: my-tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080
```

```bash
# Check ingress
kubectl get ingress -A
kubectl describe ingress my-app-ingress -n production

# View Ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Test
curl -H "Host: myapp.example.com" http://<EXTERNAL-IP>
```

---

### 🔴 Q23. How does kube-proxy work in iptables vs IPVS mode?

**Explanation:**
`kube-proxy` is responsible for implementing the `Service` abstraction. When you create a Service, kube-proxy programs network rules on every node so that traffic to the Service IP is redirected to one of the backing pods.

**iptables mode** (default): For each Service, kube-proxy creates a chain of iptables rules. When a packet arrives for a Service IP, iptables randomly selects one of the pod IPs (probabilistic load balancing). The problem: with thousands of services, iptables rule sets become very large, and each packet must traverse O(n) rules.

**IPVS mode**: Uses Linux kernel's IPVS (IP Virtual Server) which was designed for load balancing. IPVS maintains a hash table of service-to-pod mappings, giving O(1) lookup time regardless of the number of services. It also supports more load balancing algorithms: rr (round-robin), lc (least connection), dh (destination hash), etc.

```bash
# Check current mode
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# Switch to IPVS
kubectl edit configmap kube-proxy -n kube-system
# Change: mode: "ipvs"
# Also set: ipvs.scheduler: "lc" (optional - least connection)

# Restart kube-proxy pods
kubectl rollout restart daemonset kube-proxy -n kube-system

# Verify IPVS is active on a node
ipvsadm -Ln
ipvsadm -Ln --stats   # with statistics

# View iptables rules (iptables mode)
iptables -t nat -L KUBE-SERVICES | head -20
iptables -t nat -L | grep <service-cluster-ip>
```

---

## CKA Domain 4: Storage (10%) {#cka-domain-4}

---

### 🟢 Q24. What is the difference between a PV and a PVC?

**Explanation:**
The PV/PVC model separates the provisioning of storage (admin concern) from consuming storage (developer concern).

A **PersistentVolume (PV)** represents a piece of storage in the cluster — it could be an NFS share, a cloud disk, or a local path. It's a cluster-level resource, not namespaced.

A **PersistentVolumeClaim (PVC)** is a request for storage by a user. It specifies the required size, access mode, and optionally a StorageClass. Kubernetes matches PVCs to PVs through the **binding** process.

**Binding rules:**
- PVC capacity request must be ≤ PV capacity
- Access modes must match
- StorageClass must match (or both empty for static provisioning)
- VolumeMode must match
- Labels/selectors if specified

Once bound, the PV-PVC relationship is **exclusive** — that PV cannot be bound to another PVC.

```yaml
# Static PV (admin creates)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    type: nfs
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""    # empty = static, no StorageClass
  nfs:
    server: 192.168.1.100
    path: /exports/data

---
# PVC (developer creates)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: production
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
  storageClassName: ""    # match static PV
  selector:
    matchLabels:
      type: nfs   # optional: target specific PVs

---
# Use PVC in Pod
spec:
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /app/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data-pvc
```

```bash
# Check PV and PVC status
kubectl get pv
kubectl get pvc -A
kubectl describe pvc app-data-pvc -n production

# PV states: Available → Bound → Released → Failed
# PVC states: Pending → Bound

# Check why PVC is Pending
kubectl describe pvc app-data-pvc
# Look for "no persistent volumes available for this claim"
```

---

### 🟡 Q25. What are StorageClasses and how does dynamic provisioning work?

**Explanation:**
Dynamic provisioning eliminates the need for admins to manually pre-create PVs. When a PVC is created with a `storageClassName`, the StorageClass's **provisioner** automatically creates a PV in the backing storage system.

The `provisioner` field identifies the CSI driver or built-in provisioner that handles the creation. Cloud providers have their own provisioners (EBS, GCE PD, Azure Disk), and third-party CSI drivers provide theirs.

The `reclaimPolicy` on a StorageClass applies to dynamically provisioned PVs:
- `Delete` — when PVC is deleted, the PV and underlying storage are deleted automatically
- `Retain` — PV remains after PVC deletion (data preserved, must clean up manually)

`allowVolumeExpansion: true` allows PVCs to be resized after creation (by editing the PVC's `spec.resources.requests.storage`).

```yaml
# StorageClass for AWS EBS gp3
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # make default
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123:key/abc
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer  # only provision when pod is scheduled
```

```yaml
# PVC using StorageClass
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-dynamic-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 20Gi
```

```bash
kubectl get storageclass
kubectl describe storageclass fast-ssd

# Expand PVC (must have allowVolumeExpansion: true)
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

# Check expansion status
kubectl describe pvc my-pvc | grep -A 5 Conditions
```

---

### 🟡 Q26. What are PV access modes and volume modes?

**Explanation:**
**Access modes** define how many nodes can mount the volume simultaneously and with what permissions. This is a property of the storage backend — for example, a cloud block device (AWS EBS, GCP PD) can only be attached to one node at a time (RWO), while NFS can be mounted from many nodes simultaneously (RWX).

The `ReadWriteOncePod` (RWOP) mode introduced in Kubernetes 1.22 is even stricter than RWO — it ensures only ONE pod (not just one node) can write. This prevents race conditions in scenarios where multiple pods on the same node might otherwise share a volume.

**Volume modes** determine how storage is presented to the container:
- `Filesystem` (default) — volume is formatted and mounted as a directory
- `Block` — volume is presented as a raw block device (like `/dev/sdb`). The application manages the filesystem directly. Used by databases like Ceph, Cassandra, or applications needing raw I/O performance.

```yaml
# Block mode PV and PVC
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 100Gi
  volumeMode: Block        # raw block device
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /dev/sdb
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values: [worker-1]
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  volumeMode: Block        # must match PV
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi
---
# Pod using block volume
spec:
  containers:
  - name: db
    image: ceph-db:v1
    volumeDevices:          # NOT volumeMounts for block devices
    - name: data
      devicePath: /dev/xvda
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: block-pvc
```

---

## CKA Domain 5: Troubleshooting (30%) {#cka-domain-5}

---

### 🟢 Q27. How do you troubleshoot a pod stuck in `Pending`?

**Explanation:**
A pod in `Pending` state means the scheduler has not yet assigned it to a node. This always has a reason, and `kubectl describe` is your first tool — always check the **Events** section at the bottom.

The most common causes and their diagnostic signatures:

| Cause | Event message pattern |
|---|---|
| Insufficient CPU | `0/3 nodes available: 3 Insufficient cpu` |
| Insufficient memory | `0/3 nodes available: 3 Insufficient memory` |
| No nodes match node selector | `0/3 nodes available: 3 node(s) didn't match node selector` |
| Taint not tolerated | `0/3 nodes available: 3 node(s) had taint ... that the pod didn't tolerate` |
| PVC not bound | `pod has unbound immediate PersistentVolumeClaims` |
| Image pull issue | Pod moves to ContainerCreating, not Pending |

```bash
# Step 1: Always start here
kubectl describe pod <pod-name> -n <namespace>
# Read the Events section carefully

# Step 2: Resource issues
kubectl top nodes                          # check current usage
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl get nodes -o custom-columns=\
  NAME:.metadata.name,\
  CPU:.status.capacity.cpu,\
  MEM:.status.capacity.memory

# Step 3: Scheduling constraints
kubectl get pods <pod> -o yaml | grep -A 10 affinity
kubectl get pods <pod> -o yaml | grep -A 5 nodeSelector
kubectl get nodes --show-labels            # check node labels

# Step 4: Taints check
kubectl describe nodes | grep -A 3 Taints

# Step 5: PVC status
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>

# Step 6: Quota exceeded
kubectl describe resourcequota -n <namespace>
kubectl describe limitrange -n <namespace>
```

---

### 🟢 Q28. How do you diagnose and fix `CrashLoopBackOff`?

**Explanation:**
`CrashLoopBackOff` means the container is starting, crashing, and Kubernetes is restarting it repeatedly with exponential backoff (10s, 20s, 40s, up to 5 minutes). The `BackOff` part refers to this wait time between restart attempts.

The key is finding WHY the container crashes. Exit codes give important hints:

| Exit Code | Meaning |
|---|---|
| 0 | Clean exit (misconfigured command?) |
| 1 | General application error |
| 2 | Misuse of shell built-in |
| 126 | Command cannot execute (permission) |
| 127 | Command not found |
| 137 | Killed by SIGKILL (OOMKilled, or `kill -9`) |
| 139 | Segmentation fault |
| 143 | Killed by SIGTERM (graceful kill) |

```bash
# Step 1: Get current and previous logs
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous     # last crashed container
kubectl logs <pod> -n <namespace> -c <container> # specific container

# Step 2: Check exit code and reason
kubectl describe pod <pod>
# Look for: Last State, Exit Code, Reason

# Step 3: Check resource limits (OOMKilled)
kubectl describe pod <pod> | grep -A 5 "Last State"
# Reason: OOMKilled → increase memory limit

# Step 4: Debug with sleep override (keep container running)
kubectl run debug-pod --image=my-app:v1 \
  --command -- sleep 3600
kubectl exec -it debug-pod -- /bin/sh
# Manually run the app command to see error

# Step 5: Use ephemeral debug container
kubectl debug -it <pod> --image=busybox --target=<container>

# Step 6: Check config and secrets
kubectl exec <pod> -- env | grep -i config
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

---

### 🟡 Q29. How do you troubleshoot node `NotReady` status?

**Explanation:**
A node in `NotReady` state means the control plane has lost communication with the node's kubelet, or the kubelet itself is reporting problems. The kubelet sends a heartbeat every few seconds; if it misses enough, the node manager marks it `NotReady`.

The node conditions tell you the specific problem:

| Condition | Meaning |
|---|---|
| `Ready=False` | kubelet is not healthy |
| `MemoryPressure=True` | Node is running out of memory |
| `DiskPressure=True` | Node is running out of disk space |
| `PIDPressure=True` | Too many processes on the node |
| `NetworkUnavailable=True` | CNI plugin not configured properly |

```bash
# From control plane
kubectl get nodes
kubectl describe node <node-name>
# Check: Conditions, Events, Allocated resources

# SSH to the failing node
ssh <node-ip>

# 1. Is kubelet running?
systemctl status kubelet
# If stopped: systemctl start kubelet

# 2. kubelet logs — the most detailed source
journalctl -u kubelet -n 200 --no-pager
journalctl -u kubelet --since "30 minutes ago" | grep -i error

# 3. Container runtime
systemctl status containerd
crictl ps
crictl info

# 4. Disk pressure
df -h
du -sh /var/lib/kubelet/*   # largest directories
du -sh /var/lib/containerd  # container layers

# 5. Memory pressure
free -m
vmstat 1 5

# 6. Network connectivity to API server
curl -k https://<API_SERVER_IP>:6443/healthz
ping <API_SERVER_IP>

# 7. Certificate issues
openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -noout -dates
kubeadm certs check-expiration

# Recovery: if kubelet config is corrupt
kubeadm upgrade node       # re-applies node config
```

---

### 🟡 Q30. How do you troubleshoot control plane component failures?

**Explanation:**
In kubeadm clusters, control plane components run as **static pods** — their manifests live in `/etc/kubernetes/manifests/`. The kubelet on the control plane node watches this directory and starts/restarts these pods automatically. If a manifest is missing or has syntax errors, the pod won't start.

If the API server is down, `kubectl` won't work. Use `crictl` (CRI-level tool) to inspect containers directly.

```bash
# Check static pod manifests
ls -la /etc/kubernetes/manifests/
# Should have: etcd.yaml, kube-apiserver.yaml,
#              kube-controller-manager.yaml, kube-scheduler.yaml

# Check if static pods are running via CRI (when kubectl is unavailable)
crictl ps | grep -E 'etcd|apiserver|controller|scheduler'
crictl logs <container-id>

# When kubectl works, check logs
kubectl logs kube-apiserver-<node-name> -n kube-system
kubectl logs kube-controller-manager-<node-name> -n kube-system
kubectl logs kube-scheduler-<node-name> -n kube-system
kubectl logs etcd-<node-name> -n kube-system

# Check etcd health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Certificate expiry — common cause of control plane failures
kubeadm certs check-expiration

# Renew all certificates (when < 1 year left)
kubeadm certs renew all

# After certificate renewal, restart static pods
mv /etc/kubernetes/manifests/*.yaml /tmp/
sleep 10
mv /tmp/*.yaml /etc/kubernetes/manifests/

# Common issues and fixes
# "dial tcp: connection refused" → API server down, check apiserver manifest
# "x509: certificate has expired" → renew certs
# "context deadline exceeded" → network issue or overloaded node
```

---

### 🔴 Q31. How do you use journalctl for Kubernetes node diagnosis?

**Explanation:**
`journalctl` is the systemd journal viewer. On Kubernetes nodes, it's the primary tool for reading kubelet and container runtime logs when you need more history than `kubectl logs` provides, or when `kubectl` itself isn't working.

Key journalctl flags for Kubernetes troubleshooting:
- `-u <service>` — filter by service unit
- `-f` — follow (like `tail -f`)
- `--since` / `--until` — time range
- `-n <lines>` — last N lines
- `-p err` — only errors and above
- `--no-pager` — output to stdout (good for grep)
- `-k` — kernel messages only

```bash
# ── KUBELET LOGS ──────────────────────────────────────────────
journalctl -u kubelet -f                         # live follow
journalctl -u kubelet -n 100 --no-pager          # last 100 lines
journalctl -u kubelet --since "1 hour ago"        # last hour
journalctl -u kubelet --since "2024-01-15 10:00" --until "2024-01-15 11:00"
journalctl -u kubelet -p err -n 50               # errors only
journalctl -u kubelet -n 200 | grep -i "fail\|error\|warn"

# ── CONTAINER RUNTIME ─────────────────────────────────────────
journalctl -u containerd -n 100
journalctl -u containerd --since "30 min ago" | grep -i error

# ── KERNEL / SYSTEM ───────────────────────────────────────────
journalctl -k --since "1 hour ago"               # kernel messages
dmesg | grep -i "oom\|killed\|error\|warn" | tail -50
journalctl -p 0..3 -n 50                         # emerg,alert,crit,err

# ── USEFUL PATTERNS TO SEARCH FOR ────────────────────────────
journalctl -u kubelet | grep "Out of memory"
journalctl -u kubelet | grep "certificate"
journalctl -u kubelet | grep "failed to"
journalctl -u kubelet | grep "panic:"
journalctl -u kubelet | grep "node condition"

# ── SPECIFIC TROUBLESHOOTING SCENARIOS ───────────────────────

# Scenario: Node was just rebooted, why did pods not restart?
journalctl -u kubelet --since "$(last reboot | head -1 | awk '{print $5,$6,$7}')"

# Scenario: OOMKill investigation
journalctl -k | grep -i "oom\|killed process"
# Output: kernel: Out of memory: Kill process 12345 (my-app) ...

# Scenario: Why did kubelet restart?
journalctl _SYSTEMD_INVOCATION_ID=$(systemctl show kubelet \
  -p InvocationID --value) -u kubelet

# Export logs to file for analysis
journalctl -u kubelet --since "3 hours ago" \
  --no-pager > /tmp/kubelet-logs-$(date +%Y%m%d).txt
```

---

# CKAD — Certified Kubernetes Application Developer

| Domain | Weight |
|---|---|
| Application Design & Build | 20% |
| Application Deployment | 20% |
| Application Observability & Maintenance | 15% |
| Application Environment, Configuration & Security | **25%** ← highest |
| Services & Networking | 20% |

---

## CKAD Domain 1: Application Design & Build (20%) {#ckad-domain-1}

---

### 🟢 Q32. What multi-container pod patterns exist and when do you use them?

**Explanation:**
Kubernetes pods allow running multiple containers that share the same network namespace and can share volumes. This enables powerful patterns where containers collaborate closely. The key principle: containers in a pod always co-schedule and co-terminate together.

**Sidecar** — augments the main container without modifying its code. The main container does its job; the sidecar enhances or extends it. Examples: logging agents (Fluentd, Filebeat), service mesh proxies (Envoy/Istio), secret rotators.

**Ambassador** — acts as a proxy for outgoing traffic from the main container. The main container always connects to `localhost`, and the ambassador handles the complexity of service discovery, retry logic, or connection pooling. Example: a main app always connects to `localhost:5432`, and the ambassador decides which database replica to forward to.

**Adapter** — transforms output from the main container into a format expected by an external system. Example: main container writes metrics in a proprietary format, adapter converts to Prometheus format.

**Init container** — runs to completion before any app containers start. Used for setup tasks: waiting for dependencies, running database migrations, creating config files, downloading data. Critical property: if an init container fails, Kubernetes restarts it (not the whole pod) until it succeeds.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-demo
spec:
  # Init containers run first, in order
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres-service 5432; do echo waiting; sleep 2; done']
  - name: run-migrations
    image: my-app:v1
    command: ['python', 'manage.py', 'migrate']
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: url

  containers:
  # Main application container
  - name: app
    image: my-app:v1
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
    - name: shared-config
      mountPath: /etc/config

  # Sidecar: ships logs to centralised logging
  - name: log-shipper
    image: fluentd:v1.16
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
      readOnly: true
    - name: fluentd-config
      mountPath: /fluentd/etc

  # Adapter: converts app metrics to Prometheus format
  - name: metrics-adapter
    image: prom-exporter:v1
    ports:
    - containerPort: 9090
    env:
    - name: APP_METRICS_URL
      value: "http://localhost:8080/metrics/internal"

  volumes:
  - name: shared-logs
    emptyDir: {}
  - name: shared-config
    configMap:
      name: app-config
  - name: fluentd-config
    configMap:
      name: fluentd-config
```

---

### 🟢 Q33. How do you write a Dockerfile and build a container image?

**Explanation:**
A Dockerfile is a text file with instructions that define how to build a container image. Each instruction creates a new layer in the image. Understanding layers is important: layers are cached, so putting frequently-changing instructions (like `COPY . .`) at the end allows earlier layers to use cache, speeding up builds.

**Multi-stage builds** are the most important Dockerfile technique for production images. The idea: use a large image with all build tools to compile your application, then copy only the compiled binary into a minimal final image. This dramatically reduces image size and attack surface (no compiler, no debug tools, no package manager in production).

**Security best practices:**
- Never run as root — use `USER` instruction
- Use minimal base images (distroless, alpine, scratch)
- Pin specific image tags/digests, never `latest` in production
- Don't store secrets in Dockerfiles (they stay in layer history)
- Use `.dockerignore` to prevent accidentally copying credentials

```dockerfile
# ── MULTI-STAGE BUILD (Go example) ──────────────────────────────
# Stage 1: Build
FROM golang:1.21-alpine AS builder
WORKDIR /app

# Copy dependency files first (cached if unchanged)
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build the binary
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-w -s" \    # strip debug info — smaller binary
    -o /server ./cmd/server

# Stage 2: Final image (distroless — no shell, no package manager)
FROM gcr.io/distroless/static:nonroot
WORKDIR /app
COPY --from=builder /server /server

# Run as non-root (distroless nonroot = UID 65532)
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

```dockerfile
# ── PYTHON EXAMPLE ────────────────────────────────────────────────
FROM python:3.11-slim AS base
WORKDIR /app

# Install dependencies separately for caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Remove SUID/SGID bits from binaries
RUN find / -perm /6000 -type f -exec chmod a-s {} \; 2>/dev/null || true

RUN addgroup --system app && adduser --system --group app
USER app

EXPOSE 8000
CMD ["gunicorn", "wsgi:application", "--bind", "0.0.0.0:8000"]
```

```bash
# Build with tag
docker build -t my-registry.io/my-app:v1.2.3 .

# Build with specific Dockerfile
docker build -f Dockerfile.prod -t my-app:prod .

# Build with build args
docker build \
  --build-arg VERSION=1.2.3 \
  --build-arg ENV=production \
  -t my-app:1.2.3 .

# Multi-platform build
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t my-app:v1 --push .

# Push to registry
docker push my-registry.io/my-app:v1.2.3

# Inspect image layers
docker history my-app:v1
docker inspect my-app:v1 | jq '.[0].Config'

# Scan before pushing
trivy image my-app:v1.2.3
```

```
# .dockerignore — exclude from build context
.git
.gitignore
README.md
Dockerfile*
.dockerignore
node_modules
__pycache__
*.pyc
.env
secrets/
```

---

### 🟢 Q34. What are Jobs and CronJobs and when do you use them?

**Explanation:**
Deployments manage long-running services that should always be running. **Jobs** manage workloads that run to completion — a finite task like processing a batch of data, sending emails, or running a database migration.

Key Job concepts:
- `completions` — total number of successful pod completions required
- `parallelism` — how many pods to run simultaneously
- `backoffLimit` — how many times to retry before marking the Job as failed
- `activeDeadlineSeconds` — hard timeout for the entire job
- `ttlSecondsAfterFinished` — auto-clean up Job after N seconds

**CronJob** creates Jobs on a schedule using cron syntax.

`concurrencyPolicy`:
- `Allow` (default) — concurrent Jobs allowed
- `Forbid` — skip new Job if previous is still running
- `Replace` — cancel running Job and start new one

```yaml
# Job — process a batch of items
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processor
spec:
  completions: 10          # need 10 successful completions
  parallelism: 3           # run 3 pods at a time
  backoffLimit: 5          # retry failed pods up to 5 times
  activeDeadlineSeconds: 600  # kill after 10 minutes
  ttlSecondsAfterFinished: 3600  # clean up after 1 hour
  template:
    spec:
      restartPolicy: Never   # Never or OnFailure (not Always)
      containers:
      - name: processor
        image: data-processor:v1
        env:
        - name: BATCH_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
---
# Indexed Job — each pod gets a unique index (JOB_COMPLETION_INDEX env var)
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-batch
spec:
  completions: 5
  parallelism: 5
  completionMode: Indexed   # pods get JOB_COMPLETION_INDEX=0,1,2,3,4
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: worker:v1
---
# CronJob — run at 2:30 AM every day
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "30 2 * * *"      # min hour day month weekday
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300  # if missed, must start within 5 min
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: report-generator
            image: report-tool:v1
```

```bash
# Monitor Job progress
kubectl get jobs
kubectl describe job batch-processor
kubectl get pods -l job-name=batch-processor

# View Job logs
kubectl logs -l job-name=batch-processor

# Manually trigger a CronJob
kubectl create job --from=cronjob/nightly-report manual-run-$(date +%s)

# Clean up completed Jobs
kubectl delete jobs --field-selector status.successful=1
```

---

### 🟡 Q35. When do you use StatefulSet vs Deployment?

**Explanation:**
**Deployments** treat all pods as interchangeable — they're identical, disposable, and can be replaced in any order. This is perfect for stateless applications like web servers or REST APIs.

**StatefulSets** are for applications that need identity and/or stable persistent storage:
- **Stable, unique pod names** — `pod-0`, `pod-1`, `pod-2` (not random hash suffixes)
- **Stable DNS hostnames** — `pod-0.headless-service.namespace.svc.cluster.local`
- **Ordered deployment** — pods start in order (0, 1, 2) and scale down in reverse (2, 1, 0)
- **Per-pod PVCs** — each pod gets its own PVC from `volumeClaimTemplates`, and these PVCs are NOT deleted when the pod is deleted

The ordered startup matters for databases: the primary (pod-0) starts first and establishes itself; replicas (pod-1, pod-2) start afterward and can connect to the primary for replication.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless    # must create this headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  # Each pod gets its own PVC
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi
---
# Headless service for stable DNS
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None    # headless — no virtual IP
  selector:
    app: postgres
  ports:
  - port: 5432
```

```bash
# Access pods by stable hostname
# postgres-0.postgres-headless.default.svc.cluster.local
# postgres-1.postgres-headless.default.svc.cluster.local

# Scale up (pods added in order)
kubectl scale statefulset postgres --replicas=5

# Scale down (pods removed in reverse order)
kubectl scale statefulset postgres --replicas=2

# Delete StatefulSet but keep PVCs
kubectl delete statefulset postgres --cascade=orphan

# List PVCs created by StatefulSet
kubectl get pvc -l app=postgres
```

---

## CKAD Domain 2: Application Deployment (20%) {#ckad-domain-2}

---

### 🟡 Q36. How do you perform a rolling update and rollback?

**Explanation:**
Kubernetes Deployments support rolling updates out of the box — they gradually replace old pods with new ones, ensuring the application stays available throughout.

`RollingUpdate` strategy parameters:
- `maxUnavailable` — max number of pods that can be unavailable during update (absolute number or percentage)
- `maxSurge` — max number of extra pods that can be created above desired count

With `maxUnavailable: 0` and `maxSurge: 1`, updates are zero-downtime: a new pod is created first, and only when ready is an old pod removed.

Kubernetes maintains a **revision history** (controlled by `revisionHistoryLimit`). Each upgrade creates a new revision. `kubectl rollout undo` rolls back to the previous revision.

```yaml
spec:
  replicas: 3
  revisionHistoryLimit: 10   # keep last 10 revisions
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0      # zero-downtime: never have fewer than desired
      maxSurge: 1            # can have 1 extra pod during update
```

```bash
# Update image
kubectl set image deployment/my-app app=my-app:v2
kubectl set image deployment/my-app app=my-app:v2 --record  # deprecated but still seen

# Edit deployment directly
kubectl edit deployment my-app

# Apply from file
kubectl apply -f deployment-v2.yaml

# Monitor rollout progress
kubectl rollout status deployment/my-app
kubectl get pods -w -l app=my-app

# View rollout history
kubectl rollout history deployment/my-app
kubectl rollout history deployment/my-app --revision=3  # details of revision 3

# Pause a rollout (while debugging)
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app

# Rollback to previous revision
kubectl rollout undo deployment/my-app

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=2

# Restart all pods (force image pull, config refresh)
kubectl rollout restart deployment/my-app
```

---

### 🟡 Q37. How do you implement Canary deployments?

**Explanation:**
Canary deployment sends a small percentage of traffic to the new version while most traffic goes to the stable version. It allows testing in production with real users before committing to a full rollout.

The traffic split approach in Kubernetes (without a service mesh) uses **replica count ratio**. If you have 9 stable pods and 1 canary pod, approximately 10% of requests hit the canary. Both Deployments share a common label that the Service uses to select pods.

The labels design is crucial:
- Shared label (e.g. `app: my-app`) → Service uses this to route to BOTH
- Unique label (e.g. `track: stable/canary`) → used for monitoring and identifying each version

For more precise traffic splitting (e.g. exactly 5%, header-based routing), you need a service mesh (Istio, Linkerd) or the Gateway API.

```yaml
# Stable version — 9 replicas
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: my-app
      track: stable
  template:
    metadata:
      labels:
        app: my-app
        track: stable
        version: "1.0"
    spec:
      containers:
      - name: app
        image: my-app:v1.0
---
# Canary version — 1 replica (~10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
        version: "2.0"
    spec:
      containers:
      - name: app
        image: my-app:v2.0
---
# Service selects ALL pods with app=my-app
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app    # intentionally no 'track' label — selects both
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Monitor canary health
kubectl logs -l track=canary -f
kubectl top pods -l track=canary

# If canary is healthy — promote: increase canary replicas, reduce stable
kubectl scale deployment app-canary --replicas=5
kubectl scale deployment app-stable --replicas=5
# Eventually:
kubectl scale deployment app-canary --replicas=10
kubectl scale deployment app-stable --replicas=0

# If canary is bad — rollback: scale canary to 0
kubectl scale deployment app-canary --replicas=0
```

---

### 🟡 Q38. How do you implement Blue/Green deployment?

**Explanation:**
Blue/Green deployment maintains two complete, identical environments (blue = current, green = new). Traffic switch is **instantaneous** — you update the Service selector to point to the new version. Rollback is equally instant.

Key advantage over canary: zero transition period — either ALL traffic goes to v1 or ALL traffic goes to v2. This is important for applications where serving two different versions simultaneously causes issues (database schema changes, API breaking changes).

The cost: you need double the resources during the deployment window.

```yaml
# Blue (current) deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      color: blue
  template:
    metadata:
      labels:
        app: my-app
        color: blue
    spec:
      containers:
      - name: app
        image: my-app:v1
---
# Green (new) deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      color: green
  template:
    metadata:
      labels:
        app: my-app
        color: green
    spec:
      containers:
      - name: app
        image: my-app:v2
---
# Service — starts pointing at blue
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    color: blue    # ← change this to switch traffic
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Verify green is healthy before switching
kubectl get pods -l color=green
kubectl exec -it <green-pod> -- curl localhost:8080/healthz

# Instant switch to green
kubectl patch service my-app \
  -p '{"spec":{"selector":{"color":"green"}}}'

# Verify traffic is flowing to green
kubectl get endpoints my-app

# Instant rollback
kubectl patch service my-app \
  -p '{"spec":{"selector":{"color":"blue"}}}'

# Once green is confirmed stable — delete blue
kubectl delete deployment app-blue
```

---

### 🟡 Q39. How do you use Helm in the CKAD context?

**Explanation:**
In the CKAD exam, Helm questions focus on managing releases and values, not writing charts from scratch. Key operations: installing with custom values, upgrading, viewing rendered templates, and rollback.

The `helm template` command is particularly useful — it renders the final Kubernetes manifests without deploying. Use it to verify what will be created before applying.

```bash
# Install a chart
helm install my-release bitnami/nginx \
  --namespace my-namespace \
  --create-namespace \
  --values custom-values.yaml \
  --set image.tag=1.25 \
  --set replicaCount=3

# Install from local chart
helm install my-app ./my-chart --values=values-prod.yaml

# Upgrade and reset all values (fresh start with new values)
helm upgrade my-release bitnami/nginx \
  --reset-values \
  --values new-values.yaml

# Upgrade keeping existing values (only override specified)
helm upgrade my-release bitnami/nginx \
  --reuse-values \
  --set image.tag=1.26

# Render templates locally (no cluster needed)
helm template my-release bitnami/nginx \
  --values custom-values.yaml \
  --output-dir ./rendered-manifests/

# Check what changed between versions
helm diff upgrade my-release bitnami/nginx --version 14.0.0

# Show current values
helm get values my-release
helm get values my-release --all   # includes chart defaults

# Rollback
helm rollback my-release 1        # to revision 1
helm rollback my-release           # to previous revision

# Verify release
helm status my-release
helm test my-release               # run chart's test pods
```

---

### 🟡 Q40. How do you use Kustomize in the CKAD context?

**Explanation:**
In CKAD, Kustomize questions typically involve: understanding the overlay structure, applying overlays, and using common transformations. The built-in `kubectl apply -k` is all you need — no extra tools.

Key Kustomize features for CKAD:
- `images` — change image tags without editing deployment files
- `namePrefix/nameSuffix` — add environment prefix to all resource names
- `namespace` — set/override namespace for all resources
- `patches` — JSON or strategic merge patches to modify any field
- `configMapGenerator` / `secretGenerator` — create CM/Secrets from files or literals
- `commonLabels` — add labels to all resources
- `replicas` — change replica count

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
- configmap.yaml

# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: api
        image: my-org/api:latest
        env:
        - name: LOG_LEVEL
          value: debug
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Resources to overlay
resources:
- ../../base

# Namespace for all resources
namespace: production

# Add prefix to all resource names
namePrefix: prod-

# Add labels to all resources
commonLabels:
  env: production
  team: backend

# Update image tag
images:
- name: my-org/api
  newName: my-registry.io/api    # can also change registry
  newTag: v2.1.0

# Change replicas
replicas:
- name: api
  count: 5

# Patch: change environment variable
patches:
- patch: |-
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: api
    spec:
      template:
        spec:
          containers:
          - name: api
            env:
            - name: LOG_LEVEL
              value: warn         # override base value
  target:
    kind: Deployment
    name: api

# Add ConfigMap from literals
configMapGenerator:
- name: api-config
  literals:
  - DATABASE_URL=postgres://prod-db:5432/mydb
  - REDIS_URL=redis://prod-redis:6379

# Generate secret from files
secretGenerator:
- name: api-secrets
  files:
  - secret.properties
  type: Opaque
```

```bash
# Preview all generated manifests
kubectl kustomize overlays/production/

# Apply
kubectl apply -k overlays/production/

# Diff (what will change)
kubectl diff -k overlays/production/

# Delete all resources defined by overlay
kubectl delete -k overlays/production/
```

---

## CKAD Domain 3: Application Observability & Maintenance (15%) {#ckad-domain-3}

---

### 🟢 Q41. How do you use kubectl to observe application behaviour?

**Explanation:**
Observing application behaviour in Kubernetes requires combining multiple kubectl commands. A systematic approach: first check pod status, then logs, then events, then resource usage.

```bash
# ── POD STATUS ──────────────────────────────────────────────────
kubectl get pods                             # overview
kubectl get pods -o wide                     # with node, IP
kubectl get pods -w                          # watch for changes
kubectl get pods --field-selector status.phase=Running
kubectl get pods -l app=my-app --show-labels

# ── LOGS ────────────────────────────────────────────────────────
kubectl logs my-pod                          # current logs
kubectl logs my-pod --previous               # previous container (after crash)
kubectl logs my-pod -c my-container          # specific container
kubectl logs my-pod --all-containers=true    # all containers
kubectl logs my-pod -f                       # follow live
kubectl logs my-pod --tail=100               # last 100 lines
kubectl logs my-pod --since=1h               # last 1 hour
kubectl logs my-pod --since-time="2024-01-15T10:00:00Z"
kubectl logs -l app=my-app --prefix          # logs from all matching pods

# ── EVENTS ──────────────────────────────────────────────────────
kubectl get events -n my-namespace
kubectl get events --sort-by=.lastTimestamp
kubectl get events --field-selector reason=BackOff

# ── RESOURCE METRICS ────────────────────────────────────────────
kubectl top pods
kubectl top pods -l app=my-app
kubectl top pods --containers
kubectl top nodes

# ── DESCRIBE (comprehensive info + events) ──────────────────────
kubectl describe pod my-pod
kubectl describe deployment my-app
kubectl describe node worker-1

# ── PORT FORWARDING (access without exposing) ───────────────────
kubectl port-forward pod/my-pod 8080:80
kubectl port-forward service/my-service 9090:80
kubectl port-forward deployment/my-app 8080:8080

# ── EXEC INTO RUNNING CONTAINER ─────────────────────────────────
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -c sidecar -- /bin/sh
kubectl exec my-pod -- env                   # without interactive shell
kubectl exec my-pod -- cat /etc/config/app.properties
```

---

### 🟢 Q42. What are liveness, readiness, and startup probes?

**Explanation:**
Probes are how Kubernetes determines the health of your application. Without probes, Kubernetes only knows if the container process is running — it doesn't know if the app inside is actually functioning correctly.

**Startup probe** — introduced for applications that have a slow startup (like Java apps with JVM warmup, apps that need to load large datasets). Without it, liveness probes would kill the pod before it finishes starting. The startup probe deactivates liveness and readiness probes until it succeeds.

**Liveness probe** — answers "is this container still alive?". If it fails, the container is restarted. Use for detecting deadlocks, infinite loops, or any state where the app is running but non-functional.

**Readiness probe** — answers "is this container ready to accept traffic?". If it fails, the pod is removed from Service endpoints (no traffic sent to it) but NOT restarted. Use during startup (before the app is ready) and for temporary overload conditions.

Probe types: `httpGet` (HTTP GET request), `tcpSocket` (TCP connection), `exec` (run command), `grpc` (gRPC health check).

```yaml
spec:
  containers:
  - name: app
    image: my-app:v1
    ports:
    - containerPort: 8080

    # Startup probe — gives slow apps time to initialise
    # Allows up to 5 minutes: 30 * 10 seconds
    startupProbe:
      httpGet:
        path: /actuator/health
        port: 8080
      failureThreshold: 30       # can fail 30 times
      periodSeconds: 10           # check every 10 seconds
      # After 30 × 10s = 300s, if still failing → container killed

    # Liveness probe — restart if app is stuck
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 10    # wait 10s before first check
      periodSeconds: 10
      timeoutSeconds: 5          # probe times out after 5s
      failureThreshold: 3        # fail 3 times → restart

    # Readiness probe — remove from load balancer if not ready
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      successThreshold: 1        # 1 success → mark ready
      failureThreshold: 3        # 3 failures → mark not ready

    # TCP socket probe (for apps without HTTP)
    # livenessProbe:
    #   tcpSocket:
    #     port: 3306
    #   periodSeconds: 10

    # Exec probe (run command inside container)
    # readinessProbe:
    #   exec:
    #     command: ["/bin/sh", "-c", "redis-cli ping | grep PONG"]
    #   periodSeconds: 5
```

---

### 🟡 Q43. How do you handle Kubernetes API deprecations?

**Explanation:**
Kubernetes follows a deprecation policy: APIs are deprecated (with warnings) for at least 2 minor versions before removal. When upgrading clusters, you must update deprecated API versions in your manifests before they're removed.

Common deprecations:
- `extensions/v1beta1` → `apps/v1` for Deployments, DaemonSets, ReplicaSets
- `networking.k8s.io/v1beta1` → `networking.k8s.io/v1` for Ingress
- `rbac.authorization.k8s.io/v1beta1` → `rbac.authorization.k8s.io/v1`
- `batch/v1beta1` → `batch/v1` for CronJobs

```bash
# Check which API versions are available
kubectl api-versions
kubectl api-resources

# Explain a resource with a specific API version
kubectl explain deployment --api-version=apps/v1
kubectl explain ingress --api-version=networking.k8s.io/v1

# Find deprecated API usage — check metrics
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

# Use pluto (third-party) to scan manifests
pluto detect-files -d ./manifests/
pluto detect-helm

# Convert manifest to newer version (output only)
kubectl convert -f old-deployment.yaml --output-version apps/v1
# kubectl convert requires the convert plugin:
kubectl krew install convert
```

```yaml
# ── DEPRECATED (will fail on newer clusters) ──────────────────
apiVersion: extensions/v1beta1    # ← deprecated and removed
kind: Deployment

# ── CORRECT ───────────────────────────────────────────────────
apiVersion: apps/v1               # ← use this
kind: Deployment
spec:
  selector:                       # selector is required in apps/v1
    matchLabels:
      app: my-app

# ── INGRESS DEPRECATED ────────────────────────────────────────
apiVersion: extensions/v1beta1    # ← deprecated
kind: Ingress
spec:
  backend:
    serviceName: my-service       # ← old format
    servicePort: 80

# ── INGRESS CORRECT ───────────────────────────────────────────
apiVersion: networking.k8s.io/v1  # ← correct
kind: Ingress
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix           # ← required in v1
        backend:
          service:                 # ← new nested format
            name: my-service
            port:
              number: 80
```

---

## CKAD Domain 4: Application Environment, Configuration & Security (25%) {#ckad-domain-4}

---

### 🟢 Q44. How do you use ConfigMaps and Secrets?

**Explanation:**
ConfigMaps and Secrets allow you to decouple configuration from application code. The same container image can run in dev, staging, and prod with different configuration injected at runtime.

**ConfigMap vs Secret:**
- ConfigMap — for non-sensitive data (URLs, feature flags, log levels, config files)
- Secret — for sensitive data (passwords, API keys, TLS certs). Data is base64-encoded in etcd (NOT encrypted by default — configure EncryptionConfiguration for real encryption)

**Injection methods:**
1. **Environment variables** — simple but limited; changes require pod restart
2. **Volume mounts** — mounts as files; supports **dynamic updates** (no pod restart needed for most changes)
3. `envFrom` — inject all keys as env vars at once

```bash
# Create ConfigMap
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=DB_HOST=postgres:5432 \
  --from-file=config.properties

# Create Secret
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD='my$ecretPass' \
  --from-literal=DB_USER=admin

# TLS secret
kubectl create secret tls my-tls \
  --cert=server.crt \
  --key=server.key

# Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=my-registry.io \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=user@example.com

# View secret value
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

```yaml
spec:
  containers:
  - name: app
    image: my-app:v1

    # Inject specific keys as env vars
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASSWORD

    # Inject ALL keys from ConfigMap/Secret as env vars
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: db-secret

    # Mount as files
    volumeMounts:
    - name: config-files
      mountPath: /etc/app-config
      readOnly: true
    - name: secret-files
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: config-files
    configMap:
      name: app-config
      items:               # optional: only mount specific keys
      - key: config.properties
        path: config.properties
  - name: secret-files
    secret:
      secretName: db-secret
      defaultMode: 0400    # read-only for owner only
```

---

### 🟢 Q45. What is a ServiceAccount and how does it affect pod permissions?

**Explanation:**
Every pod runs with a ServiceAccount (SA). The SA determines what Kubernetes API permissions the pod has. By default, every namespace has a `default` ServiceAccount, and all pods in that namespace use it unless specified otherwise.

The SA token is automatically mounted into every pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. This token can be used to call the Kubernetes API. If a pod doesn't need cluster API access, you should disable this automatic mounting to reduce attack surface.

In modern Kubernetes (1.22+), service account tokens are **projected tokens** — time-limited and audience-bound, much more secure than the old long-lived tokens.

```yaml
# Create SA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
automountServiceAccountToken: false   # disable auto-mounting for all pods using this SA
---
# RBAC for the SA
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-configmap-reader
  namespace: production
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
---
# Pod using the SA
spec:
  serviceAccountName: my-app-sa
  automountServiceAccountToken: false   # can override SA setting per-pod
```

```bash
# Check SA permissions
kubectl auth can-i get configmaps \
  --as=system:serviceaccount:production:my-app-sa \
  -n production

# Create projected SA token with custom expiry
kubectl create token my-app-sa \
  --duration=1h \
  --namespace=production
```

---

### 🟡 Q46. What is a SecurityContext and what options does it provide?

**Explanation:**
SecurityContext defines privilege and access control settings for pods and containers. Settings at the pod level apply to all containers; container-level settings override pod-level settings.

The most important settings for the CKS exam:
- `runAsNonRoot: true` — Kubernetes will refuse to start a container as root (even if the image's Dockerfile USER is root)
- `readOnlyRootFilesystem: true` — container cannot write to its own filesystem (immutable)
- `allowPrivilegeEscalation: false` — child processes cannot gain more privileges than the parent
- `capabilities: drop: ["ALL"]` — drop all Linux capabilities (least privilege)
- `seccompProfile: RuntimeDefault` — restrict syscalls to a safe subset

```yaml
apiVersion: v1
kind: Pod
spec:
  # Pod-level security context (applies to all containers)
  securityContext:
    runAsUser: 1000          # UID to run containers as
    runAsGroup: 3000         # GID
    fsGroup: 2000            # GID for mounted volumes
    runAsNonRoot: true       # refuse to run as root
    sysctls:
    - name: net.core.somaxconn
      value: "1024"
    seccompProfile:
      type: RuntimeDefault   # restrict syscalls

  containers:
  - name: app
    image: my-app:v1
    # Container-level security context (overrides pod-level)
    securityContext:
      allowPrivilegeEscalation: false   # cannot gain more privileges
      readOnlyRootFilesystem: true      # immutable filesystem
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL                           # drop all capabilities
        add:
        - NET_BIND_SERVICE              # only re-add what's needed
                                        # (bind ports < 1024)
    volumeMounts:
    # Since rootFS is read-only, mount writable dirs explicitly
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache

  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

```bash
# Verify security context is applied
kubectl exec my-pod -- id
kubectl exec my-pod -- cat /proc/1/status | grep Cap
kubectl exec my-pod -- touch /test-write    # should fail with readOnly
```

---

### 🔴 Q47. What are LimitRange and ResourceQuota?

**Explanation:**
**LimitRange** operates at the individual container/pod level. It sets defaults (what happens if a developer doesn't set requests/limits) and maximum allowed values. Without LimitRange, a developer could accidentally create a pod with no resource limits that consumes all node resources.

**ResourceQuota** operates at the namespace level. It caps total resource consumption across ALL pods in a namespace. Critical in multi-tenant clusters to prevent one team's namespace from consuming all cluster resources.

They work together: LimitRange ensures every container has requests/limits set (otherwise ResourceQuota can't track it properly), and ResourceQuota ensures the total namespace usage stays within bounds.

```yaml
# LimitRange — defaults and maxima per container
apiVersion: v1
kind: LimitRange
metadata:
  name: container-defaults
  namespace: team-a
spec:
  limits:
  - type: Container
    default:               # applied if limit is not specified
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:        # applied if request is not specified
      cpu: "100m"
      memory: "128Mi"
    max:                   # hard ceiling — pods exceeding this are rejected
      cpu: "2"
      memory: "2Gi"
    min:                   # floor — pods below this are rejected
      cpu: "50m"
      memory: "64Mi"
  - type: Pod
    max:                   # max across all containers in pod
      cpu: "4"
      memory: "4Gi"
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"
---
# ResourceQuota — total namespace limits
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    # Compute
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    # Objects
    pods: "50"
    services: "20"
    services.loadbalancers: "2"
    services.nodeports: "0"     # no NodePorts allowed
    persistentvolumeclaims: "10"
    configmaps: "50"
    secrets: "20"
    # Storage
    requests.storage: "200Gi"
    fast-ssd.storageclass.storage.k8s.io/requests.storage: "100Gi"
```

```bash
# View quota status
kubectl describe resourcequota team-a-quota -n team-a
# Shows: hard limits vs current used values

# View limitrange
kubectl describe limitrange container-defaults -n team-a
```

---

## CKAD Domain 5: Services & Networking (20%) {#ckad-domain-5}

---

### 🟢 Q48. What is an Ingress and how do you configure it?

**Explanation:**
Ingress is a Kubernetes API object that manages external HTTP/HTTPS access to services within a cluster. It provides:
- **Host-based routing** — route `api.example.com` to one service, `app.example.com` to another
- **Path-based routing** — route `/api/*` to one service, `/static/*` to another
- **TLS termination** — decrypt HTTPS at the ingress and forward plain HTTP to backends

The `ingressClassName` field (preferred over annotations in newer versions) selects which Ingress controller handles this Ingress resource. Multiple controllers can coexist (nginx, traefik, haproxy).

`pathType` options:
- `Exact` — matches exact path only
- `Prefix` — matches path and all subpaths (`/api` matches `/api/v1/users`)
- `ImplementationSpecific` — depends on the controller

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: production
  annotations:
    # NGINX-specific annotations
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    - api.example.com
    secretName: example-tls   # kubectl create secret tls example-tls ...
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /v1(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-v1
            port:
              number: 8080
      - path: /v2(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-v2
            port:
              number: 8080
```

---

### 🟡 Q49. How do NetworkPolicies work in the CKAD context?

**Explanation:**
In CKAD, NetworkPolicy questions focus on correctly writing label-based rules. The most common scenario: "isolate namespace X so only pods with label Y can communicate with pods with label Z".

Understanding the `from`/`to` list structure is critical:
- **Items in the same list entry** are AND conditions (podSelector AND namespaceSelector)
- **Separate list entries** are OR conditions (this selector OR that selector)

```yaml
# ── SCENARIO: Complete network isolation for a namespace ──────

# Step 1: Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: secure-ns
spec:
  podSelector: {}       # {} selects ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress
---
# Step 2: Allow specific ingress from specific pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-ingress
  namespace: secure-ns
spec:
  podSelector:
    matchLabels:
      role: api            # applies to pods with role=api
  policyTypes:
  - Ingress
  ingress:
  # Entry 1: allows traffic from frontend pods in the SAME namespace
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  # Entry 2 (OR): allows traffic from ANY pod in monitoring namespace
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: monitoring
    ports:
    - protocol: TCP
      port: 9090
---
# Step 3: Allow egress to database AND DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-egress
  namespace: secure-ns
spec:
  podSelector:
    matchLabels:
      role: api
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
  # Always allow DNS (without this, name resolution fails!)
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

```bash
# Verify policy works
kubectl exec -n secure-ns frontend-pod -- nc -zv api-pod 8080   # should work
kubectl exec -n secure-ns rogue-pod -- nc -zv api-pod 8080       # should fail

# Debug NetworkPolicy
kubectl get networkpolicies -n secure-ns
kubectl describe networkpolicy allow-api-ingress -n secure-ns
```

# CKS — Certified Kubernetes Security Specialist

> **Prerequisite:** Valid CKA certification required.
> **Exam:** 2 hours · 15–20 tasks · Passing score: 67%

| Domain | Weight |
|---|---|
| Cluster Setup | 15% |
| Cluster Hardening | 15% |
| System Hardening | 10% |
| Minimize Microservice Vulnerabilities | 20% |
| Supply Chain Security | 20% |
| Monitoring, Logging & Runtime Security | 20% |

---

## CKS Domain 1: Cluster Setup (15%) {#cks-domain-1}

---

### 🟢 Q50. What is the CIS Benchmark and how do you use kube-bench?

**Explanation:**
The **CIS (Center for Internet Security) Kubernetes Benchmark** is a set of security configuration guidelines developed by security experts. It covers the API server, etcd, kubelet, scheduler, controller manager, and networking. Each check is categorised as Level 1 (basic security) or Level 2 (defense-in-depth, may impact functionality).

`kube-bench` is an open-source tool that automates CIS Benchmark checks. It reads the actual configuration of your cluster components and reports PASS, FAIL, or WARN for each check, along with remediation steps.

**Common critical failures:**
- Anonymous authentication enabled on API server or kubelet
- Read-only API port enabled on kubelet (10255)
- `AlwaysAllow` authorization mode
- `--profiling` enabled
- No audit log configured
- etcd not using TLS for peer communication

```bash
# ── INSTALL AND RUN kube-bench ──────────────────────────────────

# As a Kubernetes Job (recommended — runs inside cluster)
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl wait --for=condition=complete job/kube-bench --timeout=120s
kubectl logs job/kube-bench

# Targeted jobs (control plane vs node)
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-master.yaml
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-node.yaml

# Locally on control plane
kube-bench run --targets master
kube-bench run --targets etcd
kube-bench run --targets controlplane

# Locally on worker node
kube-bench run --targets node
kube-bench run --targets policies

# JSON output for automation
kube-bench run --json --targets master > kube-bench-results.json

# Only show failures
kube-bench run --targets master 2>/dev/null | grep -A 5 '\[FAIL\]'

# ── INTERPRETING OUTPUT ─────────────────────────────────────────
# [PASS] 1.2.5  Ensure API server --anonymous-auth is set to false
# [FAIL] 1.2.6  Ensure --kubelet-certificate-authority is set
#   Remediation: Edit /etc/kubernetes/manifests/kube-apiserver.yaml
#   Add: --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt

# ── COMMON FIXES ───────────────────────────────────────────────
# Fix: disable anonymous auth
# Edit /etc/kubernetes/manifests/kube-apiserver.yaml
# Add: --anonymous-auth=false

# Fix: disable profiling
# Add: --profiling=false

# Fix: enable audit logging
# Add: --audit-log-path=/var/log/kubernetes/audit.log
#      --audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

---

### 🟢 Q51. How do you implement default-deny NetworkPolicy?

**Explanation:**
By default in Kubernetes, every pod can communicate with every other pod. This violates the principle of least privilege. The correct approach is to start with default-deny policies and then explicitly allow required communication.

This is especially important in multi-tenant clusters where different teams' workloads run in different namespaces. Without default-deny, a compromised pod in namespace A could easily reach databases in namespace B.

```yaml
# ── DENY ALL INGRESS in a namespace ──────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  # No ingress rules = deny all ingress

---
# ── DENY ALL EGRESS in a namespace ───────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  # No egress rules = deny all egress

---
# ── DENY ALL (ingress + egress) — most secure ─────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# ── ALWAYS ALLOW DNS AFTER DEFAULT-DENY ───────────────────────
# Without this, pods cannot resolve service names
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

---

### 🟡 Q52. How do you configure TLS for Kubernetes services?

**Explanation:**
TLS protects data in transit. For Kubernetes services exposed via Ingress, TLS is terminated at the Ingress controller. For internal service-to-service communication, a service mesh (Istio/Linkerd) provides mTLS.

For certificate management, `cert-manager` is the de-facto standard. It automates certificate provisioning from Let's Encrypt, Vault, or self-signed CAs, and automatically renews certificates before expiry.

```bash
# ── MANUAL TLS SECRET ─────────────────────────────────────────
# Generate self-signed cert (dev/testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=myapp.example.com/O=myapp"

kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n production

# ── CERT-MANAGER (production) ──────────────────────────────────
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.0 \
  --set installCRDs=true

# Verify cert-manager is running
kubectl get pods -n cert-manager
```

```yaml
# ClusterIssuer — Let's Encrypt production
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx

---
# ClusterIssuer — Self-signed (for internal use)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}

---
# Request a specific certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: production
spec:
  secretName: myapp-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - myapp.example.com
  - api.example.com
  duration: 2160h     # 90 days
  renewBefore: 360h   # renew 15 days before expiry
```

```bash
# Monitor certificate status
kubectl get certificates -n production
kubectl describe certificate myapp-cert -n production

# Check expiry of existing certs
kubectl get secret myapp-tls -n production \
  -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | \
  openssl x509 -noout -dates

# Cluster certificate expiry
kubeadm certs check-expiration
```

---

### 🔴 Q53. How do you configure Kubernetes audit logging?

**Explanation:**
Kubernetes audit logging records every request to the API server: who made the request, what they did, on which resource, and what the response was. This is essential for security investigations, compliance (SOC2, PCI-DSS), and detecting unauthorized access.

The audit policy controls what gets logged and at what detail level:
- `None` — don't log
- `Metadata` — log request metadata (user, time, resource) but NOT the request/response body
- `Request` — log metadata + request body but NOT response body
- `RequestResponse` — log everything including response body (very verbose)

**Important:** Log sensitive resources (secrets) at `Metadata` level only — never at `RequestResponse` (that would log secret values in the audit log).

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
# Don't log read-only requests to these noisy resources
omitStages:
- RequestReceived     # don't log initial receive stage
rules:

# Log secret access at Metadata level only (don't expose values)
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["get", "create", "update", "delete", "patch"]

# Log pod exec/attach/portforward at Request level
- level: Request
  resources:
  - group: ""
    resources: ["pods/exec", "pods/attach", "pods/portforward"]

# Log modifications to RBAC at Request level
- level: Request
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  verbs: ["create", "update", "patch", "delete"]

# Skip health checks from system components
- level: None
  users: ["system:kube-controller-manager", "system:kube-scheduler"]
  verbs: ["get"]
  resources:
  - group: ""
    resources: ["endpoints"]

# Skip read-only API calls (too noisy)
- level: None
  verbs: ["get", "list", "watch"]
  resources:
  - group: ""
    resources: ["configmaps", "pods", "services", "endpoints"]

# Default: log everything else at Metadata level
- level: Metadata
```

```bash
# Add to /etc/kubernetes/manifests/kube-apiserver.yaml
# Under spec.containers[0].command:
- --audit-log-path=/var/log/kubernetes/audit.log
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-maxage=30      # days to retain
- --audit-log-maxbackup=10   # number of backup files
- --audit-log-maxsize=100    # megabytes before rotation

# Also mount the policy file and log directory as volumes in the static pod

# Query audit logs
tail -f /var/log/kubernetes/audit.log | jq .
cat /var/log/kubernetes/audit.log | \
  jq 'select(.user.username != "system:serviceaccount:kube-system:kube-proxy")'

# Find all secret reads
cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource == "secrets") | 
      {time: .requestReceivedTimestamp, user: .user.username, name: .objectRef.name}'
```

---

## CKS Domain 2: Cluster Hardening (15%) {#cks-domain-2}

---

### 🟢 Q54. How do you apply least-privilege RBAC?

**Explanation:**
The principle of least privilege means granting only the minimum permissions required to perform a task. In Kubernetes RBAC, this means:

1. **No wildcards** — never use `resources: ["*"]` or `verbs: ["*"]` in production
2. **Use Roles, not ClusterRoles** — if a workload only needs namespace access, use Role/RoleBinding
3. **Narrow resource scope** — use `resourceNames` to limit access to specific resources
4. **Regular audits** — periodically review what permissions are being granted and used

The built-in `ClusterRoles` like `view`, `edit`, and `admin` are useful starting points but often grant more than needed. Custom roles are usually better.

```yaml
# Bad — too permissive
kind: ClusterRole
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# Bad — namespace-scoped but still too permissive
kind: Role
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["*"]

# Good — minimal permissions, namespace-scoped
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-deployer
  namespace: production
rules:
# Can manage deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
# Can read configmaps and secrets
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["app-secret", "db-secret"]   # specific secrets only

# Good — read-only with resource name restriction
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
  namespace: staging
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
  resourceNames: ["app-config"]   # only this specific configmap
```

```bash
# Audit who has access to what
kubectl get rolebindings,clusterrolebindings -A \
  -o custom-columns='KIND:.kind,NAMESPACE:.metadata.namespace,NAME:.metadata.name,SUBJECTS:.subjects'

# Check if SA has excessive permissions
kubectl auth can-i --list \
  --as=system:serviceaccount:production:my-app-sa \
  -n production

# Find cluster-admin bindings (should be very few)
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name == "cluster-admin") | 
      {name: .metadata.name, subjects: .subjects}'
```

---

### 🟢 Q55. How do you restrict ServiceAccount token auto-mounting?

**Explanation:**
By default, Kubernetes automatically mounts a service account token into every pod. This is a security risk: any code running in the pod can use this token to call the Kubernetes API. A compromised or malicious container could use this to escalate privileges or exfiltrate cluster information.

**Best practice:** Disable auto-mounting globally and only enable it for pods that specifically need API access.

```yaml
# Disable at ServiceAccount level (all pods using this SA)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-access-sa
  namespace: production
automountServiceAccountToken: false

---
# Disable per-pod (overrides SA setting if SA allows)
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: no-api-access-sa
  automountServiceAccountToken: false  # explicit per-pod override
  containers:
  - name: app
    image: my-app:v1
```

```bash
# Find all pods with auto-mounted tokens
kubectl get pods -A -o json | \
  jq '.items[] | select(
    .spec.automountServiceAccountToken != false and
    .spec.serviceAccountName != "no-api-access-sa"
  ) | "\(.metadata.namespace)/\(.metadata.name)"'

# Check if token is actually mounted
kubectl exec my-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# Should be empty or not exist

# Find default SA in all namespaces with automounting enabled
kubectl get serviceaccounts -A -o json | \
  jq '.items[] | select(
    .metadata.name == "default" and 
    .automountServiceAccountToken != false
  ) | .metadata.namespace'
```

---

### 🟡 Q56. What admission controllers are important for security?

**Explanation:**
Admission controllers are plugins that intercept API requests after authentication and authorization but before persistence. They can validate or mutate objects. They're the last line of defense before an object is stored.

**Mutating admission webhooks** modify objects (add defaults, inject sidecars like Istio envoy).
**Validating admission webhooks** reject objects that don't meet policy (OPA/Gatekeeper, Kyverno).

The `PodSecurity` admission controller (replacing the deprecated PodSecurityPolicy) enforces **Pod Security Standards** at the namespace level using labels.

```bash
# Check enabled admission controllers
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission

# Recommended set for security
--enable-admission-plugins=\
  NodeRestriction,\
  PodSecurity,\
  ResourceQuota,\
  LimitRanger,\
  ServiceAccount,\
  DefaultStorageClass,\
  DefaultTolerationSeconds,\
  MutatingAdmissionWebhook,\
  ValidatingAdmissionWebhook

# Dangerous (should be disabled)
# --enable-admission-plugins=AlwaysAdmit  (bypasses all checks)
```

```yaml
# Pod Security Standards via namespace labels
# Standards: privileged, baseline, restricted
# Modes: enforce (block), audit (log), warn (warn)

# Restricted namespace — highest security
apiVersion: v1
kind: Namespace
metadata:
  name: secure-apps
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

```bash
# Set PSA mode on existing namespace
kubectl label namespace secure-apps \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# Test if a pod would pass
kubectl apply --dry-run=server -n secure-apps -f pod.yaml
# Will show violations if any
```

---

### 🔴 Q57. How do you harden the Kubernetes API server?

**Explanation:**
The API server is the central nervous system of Kubernetes — every action goes through it. Hardening it means restricting what can authenticate, what they can do, and how they connect.

Key hardening areas:
- **Anonymous auth** — should be disabled; only authenticated requests should reach the API
- **Authorization modes** — must include `RBAC` and `Node`; never use `AlwaysAllow`
- **TLS** — minimum TLS 1.2, strong cipher suites only
- **Profiling** — disable in production (can expose performance data to attackers)
- **Audit logging** — must be enabled for security compliance

```bash
# Edit /etc/kubernetes/manifests/kube-apiserver.yaml
# Under spec.containers[0].command, add/verify:
```

```yaml
# kube-apiserver hardening flags
- --anonymous-auth=false
- --authorization-mode=Node,RBAC          # never AlwaysAllow
- --enable-admission-plugins=NodeRestriction,PodSecurity,ResourceQuota,LimitRanger
- --disable-admission-plugins=AlwaysAdmit
- --tls-min-version=VersionTLS12
- --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
- --service-account-lookup=true           # verify SA token exists in etcd
- --service-account-key-file=/etc/kubernetes/pki/sa.pub
- --profiling=false
- --audit-log-path=/var/log/kubernetes/audit.log
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --request-timeout=300s
- --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
- --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
- --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
- --kubelet-certificate-authority=/etc/kubernetes/pki/ca.crt
- --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
- --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
```

```bash
# Verify anonymous auth is disabled
curl -k https://localhost:6443/api
# Should return 401 Unauthorized

# Verify TLS version
openssl s_client -connect localhost:6443 2>&1 | grep Protocol

# Verify RBAC is being used
kubectl get clusterrolebindings system:anonymous 2>/dev/null \
  || echo "No anonymous bindings - good"
```

---

## CKS Domain 3: System Hardening (10%) {#cks-domain-3}

---

### 🟢 Q58. How do you reduce the OS attack surface on Kubernetes nodes?

**Explanation:**
The attack surface of a Kubernetes node is the sum of all possible ways an attacker could interact with it. Reducing attack surface means: removing unused software, disabling unused services, closing unnecessary ports, and restricting filesystem permissions.

This is important because if an attacker escapes a container (container breakout), they land on the node OS. A hardened OS makes it harder for them to establish persistence or move laterally.

```bash
# ── REMOVE UNNECESSARY PACKAGES ─────────────────────────────────
apt-get remove --purge -y \
  telnet ftp vsftpd rsh-client \
  samba ldap-utils nis talk \
  xserver-xorg-core                # GUI packages on servers

# Check what's installed
dpkg -l | grep -E 'telnet|ftp|rsh|nis|talk|samba'

# ── DISABLE UNNECESSARY SERVICES ────────────────────────────────
systemctl list-units --type=service --state=active
systemctl disable --now snapd bluetooth avahi-daemon cups \
  isc-dhcp-server postfix

# ── FIREWALL: only allow necessary ports ─────────────────────────
ufw reset
ufw default deny incoming
ufw default deny outgoing

# Control plane node
ufw allow 22/tcp           # SSH
ufw allow 6443/tcp         # Kubernetes API
ufw allow 2379:2380/tcp    # etcd
ufw allow 10250/tcp        # kubelet
ufw allow 10257/tcp        # kube-controller-manager
ufw allow 10259/tcp        # kube-scheduler

# Worker node
ufw allow 22/tcp
ufw allow 10250/tcp        # kubelet
ufw allow 30000:32767/tcp  # NodePort services
ufw allow out 443/tcp      # HTTPS outbound
ufw allow out 53/udp       # DNS
ufw allow out 53/tcp       # DNS

ufw enable
ufw status verbose

# ── CHECK OPEN PORTS ─────────────────────────────────────────────
ss -tlnp
netstat -tlnp | grep LISTEN

# ── FILE PERMISSIONS ─────────────────────────────────────────────
# Critical Kubernetes files should be owned by root
stat /etc/kubernetes/manifests/kube-apiserver.yaml
chmod 600 /etc/kubernetes/pki/*.key
chmod 644 /etc/kubernetes/pki/*.crt
chmod 600 /etc/kubernetes/admin.conf

# Find world-writable files
find / -xdev -type f -perm -002 2>/dev/null

# Find SUID/SGID binaries
find / -xdev -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null
```

---

### 🟢 Q59. How do you manage and restrict kernel modules?

**Explanation:**
Kernel modules extend the Linux kernel's functionality. Some modules introduce security vulnerabilities or enable attack vectors that are not needed in a Kubernetes environment. Blacklisting modules prevents them from being loaded, even by privileged containers.

`dccp`, `sctp`, `rds`, and `tipc` are network protocol modules that are rarely used but have had security vulnerabilities. They should be blacklisted on Kubernetes nodes.

```bash
# List currently loaded modules
lsmod
lsmod | grep -E 'dccp|sctp|rds|tipc|cramfs|freevxfs|jffs2|hfs|hfsplus|squashfs|udf'

# Check if a specific module is loaded
modinfo dccp 2>/dev/null && echo "LOADED" || echo "NOT LOADED"

# ── BLACKLIST MODULES ─────────────────────────────────────────────
cat >> /etc/modprobe.d/kubernetes-blacklist.conf << 'EOF'
# Network protocols not needed in Kubernetes
install dccp /bin/false
install sctp /bin/false
install rds /bin/false
install tipc /bin/false
install n-hdlc /bin/false

# Filesystem modules not needed on servers
install cramfs /bin/false
install freevxfs /bin/false
install jffs2 /bin/false
install hfs /bin/false
install hfsplus /bin/false
install squashfs /bin/false
install udf /bin/false
EOF

# Apply blacklist (requires re-generating initramfs)
update-initramfs -u
# or
dracut -f   # on RHEL/CentOS

# Unload currently loaded modules
modprobe -r dccp 2>/dev/null || true
modprobe -r sctp 2>/dev/null || true

# Verify module is blocked
modprobe dccp
# ERROR: could not insert 'dccp': Operation not permitted
```

---

### 🟡 Q60. What is AppArmor and how do you apply it to Kubernetes pods?

**Explanation:**
AppArmor is a Linux Security Module (LSM) that enforces **Mandatory Access Control (MAC)** at the process level. Unlike traditional Unix permissions (DAC - Discretionary Access Control), AppArmor policies can restrict what files a process can access, what system calls it can make, and what network operations it can perform — regardless of what the process owner's permissions allow.

In Kubernetes, AppArmor profiles can be applied to containers to restrict what they can do even if the container process runs as root. This provides defence-in-depth: even if a container is compromised, the AppArmor profile limits blast radius.

AppArmor modes:
- `enforce` — policy is enforced; violations are blocked and logged
- `complain` — policy violations are only logged, not blocked (useful for developing profiles)

```bash
# ── NODE SETUP ───────────────────────────────────────────────────
# Check AppArmor is enabled
cat /sys/module/apparmor/parameters/enabled   # should be Y
aa-status

# Sample AppArmor profile for a web container
cat > /etc/apparmor.d/k8s-nginx-profile << 'EOF'
#include <tunables/global>

profile k8s-nginx-profile flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  network tcp,
  network udp,

  # Allow nginx to read its config
  /etc/nginx/** r,
  /var/log/nginx/** rw,
  /var/run/nginx.pid rw,

  # Allow access to web content
  /usr/share/nginx/** r,
  /var/www/** r,

  # Deny everything else
  deny /proc/** w,
  deny /sys/** w,
  deny /etc/passwd r,
  deny /etc/shadow r,
}
EOF

# Load the profile
apparmor_parser -q /etc/apparmor.d/k8s-nginx-profile

# Verify it's loaded
aa-status | grep k8s-nginx-profile
```

```yaml
# Apply AppArmor profile in Kubernetes (v1.30+)
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    securityContext:
      appArmorProfile:
        type: Localhost
        localhostProfile: k8s-nginx-profile    # profile must be loaded on node

---
# For Kubernetes < 1.30 (annotation-based)
apiVersion: v1
kind: Pod
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/nginx: localhost/k8s-nginx-profile
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

```bash
# Verify AppArmor is enforced for the container
kubectl exec my-pod -- cat /proc/1/attr/current
# Should show the profile name and mode: enforce
```

---

### 🟡 Q61. What is Seccomp and how do you configure it in Kubernetes?

**Explanation:**
Seccomp (Secure Computing mode) is a Linux kernel feature that restricts the system calls a process can make. Even a root process in a container cannot use syscalls that are blocked by seccomp. This provides a powerful layer of protection against privilege escalation exploits that rely on specific syscalls.

Kubernetes has built-in seccomp profile types:
- `Unconfined` — no seccomp restrictions (the old default)
- `RuntimeDefault` — the container runtime's default profile (blocks ~40+ dangerous syscalls)
- `Localhost` — a custom profile stored on each node

For most workloads, `RuntimeDefault` is sufficient and safe. Custom profiles give more control but require detailed knowledge of what syscalls your application uses (use `strace` to find out).

```yaml
# RuntimeDefault — recommended for all containers
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault

---
# Custom profile (more restrictive)
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/my-app.json   # relative to /var/lib/kubelet/seccomp/
```

```json
// /var/lib/kubelet/seccomp/profiles/my-app.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64", "SCMP_ARCH_X86"],
  "syscalls": [
    {
      "names": [
        "accept", "accept4", "access", "brk", "close", "connect",
        "epoll_create", "epoll_create1", "epoll_ctl", "epoll_wait",
        "exit", "exit_group", "fstat", "futex", "getcwd", "getdents64",
        "getpid", "getrandom", "getuid", "getsockname", "getsockopt",
        "listen", "lstat", "mmap", "mprotect", "nanosleep",
        "newfstatat", "openat", "pipe2", "poll", "read", "recvfrom",
        "recvmsg", "rt_sigaction", "rt_sigprocmask", "sendmsg",
        "sendto", "setsockopt", "sigaltstack", "socket", "stat",
        "uname", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

```bash
# Place profile on all nodes (can use DaemonSet for this)
mkdir -p /var/lib/kubelet/seccomp/profiles
cp my-app.json /var/lib/kubelet/seccomp/profiles/

# Verify seccomp is applied
kubectl exec my-pod -- \
  grep Seccomp /proc/1/status
# Seccomp: 2  (2 = filter mode = seccomp is active)
```

---

### 🔴 Q62. How do you harden the kubelet?

**Explanation:**
The kubelet is the most critical component on each node — it runs all pods and has deep OS access. A vulnerable or misconfigured kubelet can allow attackers to run arbitrary containers with any privileges, access all pod secrets, or gain full node access.

Key kubelet hardening:
- **Disable anonymous authentication** — only authenticated requests
- **Enable webhook authorization** — authorize via the API server, not `AlwaysAllow`
- **Disable read-only port (10255)** — this port allows unauthenticated reads of pod info
- **Enable certificate rotation** — automatically rotate kubelet's client certificate
- **Protect kernel defaults** — fail if kernel parameters are not as expected

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# Disable anonymous access
authentication:
  anonymous:
    enabled: false      # ← key change
  webhook:
    enabled: true       # use API server for auth
    cacheTTL: 2m
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt

# Require authorization (not AlwaysAllow)
authorization:
  mode: Webhook         # ← must be Webhook, not AlwaysAllow
  webhook:
    cacheAuthorizedTTL: 5m
    cacheUnauthorizedTTL: 30s

# Disable read-only port
readOnlyPort: 0         # ← 0 = disabled

# Protect kernel defaults (kubelet will fail to start if kernel params differ)
protectKernelDefaults: true

# Certificate rotation
rotateCertificates: true
serverTLSBootstrap: true

# TLS hardening
tlsMinVersion: VersionTLS12
tlsCipherSuites:
- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
- TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384

# Limits
eventRecordQPS: 5       # limit event flood
streamingConnectionIdleTimeout: 4h
makeIPTablesUtilChains: true
```

```bash
# Apply changes
systemctl daemon-reload && systemctl restart kubelet

# Verify anonymous auth is disabled
curl -sk https://localhost:10250/pods
# Should return 401 Unauthorized

# Verify read-only port is closed
curl http://localhost:10255/pods
# Connection refused
```

---

## CKS Domain 4: Minimize Microservice Vulnerabilities (20%) {#cks-domain-4}

---

### 🟢 Q63. How do you manage and encrypt Kubernetes secrets?

**Explanation:**
Kubernetes Secrets are base64-encoded by default, NOT encrypted. Anyone with read access to etcd gets all secrets in plaintext. Encryption at rest solves this by encrypting secrets before storing them in etcd.

For production security, use a dedicated secrets management solution:
- **HashiCorp Vault** — most feature-rich, supports dynamic secrets, leasing
- **AWS Secrets Manager / Parameter Store** — managed, integrates with EKS via IRSA
- **External Secrets Operator** — syncs secrets from external sources into Kubernetes Secrets

```bash
# Generate a 32-byte key for AES-CBC encryption
head -c 32 /dev/urandom | base64
```

```yaml
# /etc/kubernetes/enc/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps   # optionally encrypt configmaps too
  providers:
  # First provider is used for writing
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  # identity provider must be last — reads unencrypted existing secrets
  - identity: {}
```

```bash
# Add to kube-apiserver:
# --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
# Mount the file in the static pod:
# volumeMounts:
# - name: enc-config
#   mountPath: /etc/kubernetes/enc
#   readOnly: true
# volumes:
# - name: enc-config
#   hostPath:
#     path: /etc/kubernetes/enc

# Restart API server
# (it will restart automatically when manifest changes)

# Verify encryption is working
# Create a test secret
kubectl create secret generic test-encryption \
  --from-literal=test=supersecret

# Check raw etcd value — should be encrypted
ETCDCTL_API=3 etcdctl get /registry/secrets/default/test-encryption \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | \
  strings
# Should see "k8s:enc:aescbc" prefix — not plaintext

# Re-encrypt ALL existing secrets with new key
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

---

### 🟡 Q64. What is container sandboxing and how do you use RuntimeClass?

**Explanation:**
Standard containers share the host kernel. If a container exploits a kernel vulnerability (e.g. a CVE in a syscall), it can gain host access. Container sandboxing adds an additional isolation layer:

**gVisor (runsc)** — implements most Linux syscalls in a user-space Go application. The container talks to gVisor's Sentry instead of the real kernel. gVisor intercepts and handles syscalls, providing isolation even if the host kernel has vulnerabilities.

**Kata Containers** — runs each container (or pod) inside a lightweight virtual machine. Provides stronger isolation (hardware VM boundary) at the cost of slightly more overhead (VM startup time, memory).

RuntimeClass lets you select which runtime a pod uses. The platform team configures the RuntimeClass; developers just reference it by name.

```bash
# Step 1: Install gVisor on the node
curl -fsSL https://gvisor.dev/archive.key | gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main" | tee /etc/apt/sources.list.d/gvisor.list
apt-get update && apt-get install -y runsc

# Step 2: Configure containerd to use gVisor
cat >> /etc/containerd/config.toml << 'EOF'
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
EOF
systemctl restart containerd
```

```yaml
# Step 3: Create RuntimeClass
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc           # matches the containerd runtime name
scheduling:
  nodeSelector:
    runtime.sandbox: gvisor   # only schedule on nodes with gVisor
  tolerations:
  - key: runtime.sandbox
    value: gvisor
    effect: NoSchedule

---
# RuntimeClass for Kata Containers
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-containers
handler: kata-qemu
overhead:
  podFixed:
    memory: "160Mi"     # overhead of the VM
    cpu: "250m"
```

```yaml
# Step 4: Use in pod spec
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-app
spec:
  runtimeClassName: gvisor   # ← use gVisor
  containers:
  - name: app
    image: my-untrusted-app:v1
```

```bash
# Verify gVisor is being used
kubectl exec sandboxed-app -- uname -r
# Shows gVisor kernel, not host kernel
# e.g.: 4.4.0 #1 SMP gVisor container kernel

kubectl exec sandboxed-app -- dmesg | head -3
```

---

### 🟡 Q65. What are Pod Security Standards (PSS) and how do you enforce them?

**Explanation:**
Pod Security Standards are built-in Kubernetes security policies that define three profiles:

- **Privileged** — no restrictions; identical to running without policies
- **Baseline** — minimal restrictions; prevents known privilege escalations; allows most containerised workloads
- **Restricted** — heavily restricted; requires `runAsNonRoot`, `seccompProfile`, dropping all capabilities; best practices enforced

PSS replaced the deprecated PodSecurityPolicy (PSP). They're enforced via the `PodSecurity` admission controller using namespace labels.

```bash
# Label namespace with PSS mode
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# Audit (log but don't block violations)
kubectl label namespace staging \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=restricted

# Test a pod against the policy
kubectl apply --dry-run=server -n production -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - name: test
    image: nginx
EOF
# Will show: Error: pod violates PodSecurity "restricted:latest"
# Details: allowPrivilegeEscalation != false, seccompProfile, etc.
```

```yaml
# Pod that passes RESTRICTED policy
apiVersion: v1
kind: Pod
metadata:
  name: restricted-compliant
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:v1
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop:
        - ALL
    # Must provide writable dirs if rootFS is read-only
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

---

### 🔴 Q66. How does Cilium provide network encryption?

**Explanation:**
Traditional Kubernetes NetworkPolicies operate at Layer 3/4 (IP/port) and don't encrypt traffic. Cilium adds transparent encryption at the network layer using either **WireGuard** (simpler, better performance) or **IPsec** (FIPS-compliant, more complex).

This is a **new CKS exam topic in 2025**. The encryption is transparent — existing pods don't need any changes. Cilium handles key exchange, encryption, and decryption automatically.

WireGuard vs IPsec:
- **WireGuard** — simpler key management, built into Linux kernel 5.6+, faster
- **IPsec** — more mature, FIPS-compliant options, more complex

```bash
# ── INSTALL CILIUM WITH WIREGUARD ENCRYPTION ─────────────────────
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set encryption.enabled=true \
  --set encryption.type=wireguard \
  --set encryption.wireguard.userspaceFallback=false  # use kernel WireGuard

# ── INSTALL WITH IPSEC ────────────────────────────────────────────
# Generate IPsec keys
kubectl create secret generic cilium-ipsec-keys \
  --from-literal=keys="3 rfc4106(gcm(aes)) $(echo $(dd if=/dev/urandom count=20 bs=1 2> /dev/null | xxd -p -c 64)) 128"

helm install cilium cilium/cilium \
  --namespace kube-system \
  --set encryption.enabled=true \
  --set encryption.type=ipsec \
  --set encryption.keyFile=keys

# ── VERIFY ENCRYPTION STATUS ──────────────────────────────────────
kubectl exec -n kube-system ds/cilium -- cilium encrypt status
# WireGuard    [cilium-encrypt] 2/2 peers, 12 keys, 1024 bytes/s

kubectl exec -n kube-system ds/cilium -- \
  wg show   # show WireGuard configuration and peers

# Verify traffic is encrypted (packet capture should show WireGuard packets)
tcpdump -i eth0 'udp port 51871'  # WireGuard port
```

---

## CKS Domain 5: Supply Chain Security (20%) {#cks-domain-5}

---

### 🟢 Q67. How do you scan container images for vulnerabilities?

**Explanation:**
Image scanning analyses container image layers for known CVEs (Common Vulnerabilities and Exposures) in OS packages, language runtimes, and application libraries. It's a critical gate in the CI/CD pipeline — catch vulnerabilities before they reach production.

**Trivy** (by Aqua Security) is the most popular open-source scanner. It scans:
- OS packages (Ubuntu, Alpine, RHEL)
- Language packages (pip, npm, gem, go.sum, cargo)
- Kubernetes configs
- IaC files (Terraform, Dockerfile)
- SBOMs

Integrate scanning into your pipeline: scan the image after build, fail the pipeline on CRITICAL vulnerabilities, and only push clean images to the registry.

```bash
# ── BASIC SCANNING ─────────────────────────────────────────────
trivy image nginx:latest
trivy image alpine:3.19
trivy image my-registry.io/my-app:v1

# Only show HIGH and CRITICAL
trivy image --severity HIGH,CRITICAL nginx:latest

# Fail pipeline on CRITICAL (exit code 1)
trivy image --exit-code 1 --severity CRITICAL nginx:latest
echo $?  # 1 if vulnerabilities found

# ── OUTPUT FORMATS ─────────────────────────────────────────────
trivy image --format json nginx:latest > scan-results.json
trivy image --format sarif nginx:latest > scan-results.sarif  # GitHub/GitLab
trivy image --format table nginx:latest   # default human-readable

# ── SCANNING WITHOUT PULLING (from local archive) ─────────────
docker save nginx:latest > nginx.tar
trivy image --input nginx.tar

# ── SCANNING IN CI/CD ─────────────────────────────────────────
# GitLab CI example:
# trivy-scan:
#   image: aquasec/trivy:latest
#   script:
#     - trivy image --exit-code 1 --severity CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# ── CLUSTER SCANNING ──────────────────────────────────────────
trivy k8s --report summary cluster
trivy k8s --report all --namespace production cluster

# ── FILESYSTEM SCANNING ───────────────────────────────────────
trivy fs --severity HIGH,CRITICAL .
trivy fs --scanners vuln,secret,config .   # also check secrets and misconfigs

# ── KUBERNETES MANIFEST SCANNING ─────────────────────────────
trivy config k8s-manifests/
trivy config deployment.yaml

# ── ALTERNATIVE: Grype ────────────────────────────────────────
grype my-app:v1
grype --fail-on critical my-app:v1
```

---

### 🟢 Q68. What are Dockerfile security best practices?

**Explanation:**
A secure Dockerfile reduces attack surface in three ways: smaller images have fewer packages and therefore fewer vulnerabilities; non-root execution limits impact of exploits; minimal base images contain only what's needed.

The most impactful practices:
1. **Multi-stage builds** — compile code in a fat image, copy only the binary to a minimal final image
2. **Distroless images** — no shell, no package manager, no debugging tools; the smallest possible attack surface
3. **Non-root user** — even if a container is compromised, attacker doesn't have root
4. **Pinned versions** — prevent unexpected changes from upstream updates

```dockerfile
# ── PRODUCTION-READY DOCKERFILE ──────────────────────────────────

# Stage 1: Build (fat image with all tools)
FROM golang:1.21-alpine AS builder

# Create non-root user for build
RUN addgroup -S build && adduser -S build -G build

WORKDIR /app

# Copy dependency files first (better caching)
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Copy source
COPY --chown=build:build . .

# Build as non-root
USER build
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-w -s -extldflags '-static'" \
    -trimpath \
    -o /app/server ./cmd/server

# Stage 2: Final image (distroless — no shell at all)
FROM gcr.io/distroless/static:nonroot

# Copy binary from builder
COPY --from=builder /app/server /server

# Expose only the application port
EXPOSE 8080

# Run as non-root (distroless nonroot user = UID 65532)
USER nonroot:nonroot

# Use exec form (not shell form) for proper signal handling
ENTRYPOINT ["/server"]
```

```dockerfile
# ── PYTHON PRODUCTION DOCKERFILE ──────────────────────────────────
FROM python:3.11-slim AS base

# Security: remove SUID/SGID binaries
RUN find / -perm /6000 -type f -exec chmod a-s {} \; 2>/dev/null || true

# Create app user
RUN groupadd --gid 1000 app && \
    useradd --uid 1000 --gid 1000 --no-create-home app

WORKDIR /app

# Install dependencies as root, then drop to app user
COPY requirements.txt .
RUN pip install --no-cache-dir --require-hashes -r requirements.txt

COPY --chown=app:app . .

# Use non-root user
USER app

EXPOSE 8000

# Exec form for proper SIGTERM handling
CMD ["gunicorn", "wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

---

### 🟡 Q69. How do you enforce image registry allowlists?

**Explanation:**
Without controls, developers can deploy containers from any public registry — including malicious or vulnerable images. Registry allowlisting ensures only images from trusted registries can run in the cluster.

**ImagePolicyWebhook** — built-in Kubernetes admission controller that calls an external webhook to approve/deny images. Requires building and running a webhook service.

**Kyverno or OPA/Gatekeeper** — easier approach; write a policy that validates image references against an allowed list. No external service needed.

```yaml
# ── KYVERNO APPROACH (simpler) ────────────────────────────────────
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce    # Enforce or Audit
  background: true
  rules:
  - name: validate-registries
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "Images must be from approved registries: my-registry.io or gcr.io/distroless"
      pattern:
        spec:
          initContainers:
          - image: "my-registry.io/* | gcr.io/distroless/* | ?*"
          containers:
          - image: "my-registry.io/* | gcr.io/distroless/*"

---
# Also block :latest tag
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-image-tag
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "Image tag ':latest' is not allowed. Use a specific version tag."
      pattern:
        spec:
          containers:
          - image: "!*:latest"
```

```bash
# ImagePolicyWebhook configuration (more complex)
# /etc/kubernetes/manifests/kube-apiserver.yaml
# Add: --enable-admission-plugins=...,ImagePolicyWebhook
# Add: --admission-control-config-file=/etc/kubernetes/admission-config.yaml

# /etc/kubernetes/admission-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/image-webhook.kubeconfig
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: false   # deny if webhook is unreachable
```

---

### 🟡 Q70. How do you use Kubesec and KubeLinter for static analysis?

**Explanation:**
Before deploying manifests to a cluster, static analysis tools check for security misconfigurations. This shift-left security approach catches issues early in the development pipeline, before they reach production.

**Kubesec** scores Kubernetes resources against a security model. Higher score = more secure configuration. It gives actionable advice on what to add (run as non-root, add seccompProfile) and what to remove (privileged mode, host networking).

**KubeLinter** is a more comprehensive linter that checks for both security issues and operational best practices (missing readiness probes, no resource limits, etc.).

```bash
# ── KUBESEC ────────────────────────────────────────────────────
# Scan a manifest file
kubesec scan deployment.yaml

# Scan from stdin
cat deployment.yaml | kubesec scan -

# Docker (if not installed locally)
docker run -i kubesec/kubesec:latest scan - < deployment.yaml

# Sample output:
# {
#   "score": 3,
#   "advise": [
#     { "selector": ".spec.securityContext .runAsNonRoot == true", "reason": "..." }
#   ],
#   "scoring": {
#     "critical": [
#       { "selector": ".spec.containers[] .securityContext .privileged == true",
#         "reason": "Privileged containers can allow almost completely unrestricted host access" }
#     ]
#   }
# }

# ── KUBE-LINTER ──────────────────────────────────────────────────
# Scan a file
kube-linter lint deployment.yaml

# Scan a directory
kube-linter lint ./manifests/

# Scan a Helm chart
kube-linter lint --charts ./my-chart/

# Custom configuration
cat > .kube-linter.yaml << 'EOF'
checks:
  addAllBuiltIn: true
  exclude:
  - "no-extensions-v1beta"   # allow older API versions if needed
customChecks:
- name: require-label-team
  template: required-label
  params:
    key: team
EOF
kube-linter lint --config .kube-linter.yaml ./manifests/

# Integrate in CI
kube-linter lint deployment.yaml 2>&1
echo "Exit code: $?"  # 0 = pass, non-zero = issues found
```

---

### 🔴 Q71. How do you sign and verify container images with Cosign?

**Explanation:**
Image signing creates a cryptographic proof that a specific image was built by a specific entity and has not been tampered with. When combined with a policy engine (Kyverno, OPA), you can enforce that only signed images run in your cluster.

**Cosign** (by Sigstore) is the standard tool for signing container images. It supports:
- **Key-based signing** — traditional asymmetric key pair
- **Keyless signing** — uses OIDC identity (GitHub Actions, GCP, AWS) with Fulcio CA and Rekor transparency log
- **Attestations** — signed metadata attached to an image (SBOM, vulnerability scan results, provenance)

```bash
# ── KEY-BASED SIGNING ─────────────────────────────────────────────
# Generate key pair
cosign generate-key-pair
# Creates: cosign.key (private), cosign.pub (public)

# Store private key in Kubernetes Secret
kubectl create secret generic cosign-key \
  --from-file=cosign.key=./cosign.key

# Sign after building and pushing
docker build -t my-registry.io/my-app:v1.0.0 .
docker push my-registry.io/my-app:v1.0.0
cosign sign --key cosign.key my-registry.io/my-app:v1.0.0

# Verify signature
cosign verify \
  --key cosign.pub \
  my-registry.io/my-app:v1.0.0 | jq .

# ── KEYLESS SIGNING (GitHub Actions) ──────────────────────────────
# In GitHub Actions workflow:
# - uses: sigstore/cosign-installer@v3
# - run: cosign sign --yes ${{ env.REGISTRY }}/${{ env.IMAGE }}:${{ github.sha }}
# No keys needed — uses GitHub OIDC token

# ── ATTACH SBOM AS ATTESTATION ────────────────────────────────────
syft my-registry.io/my-app:v1.0.0 -o spdx-json > sbom.spdx.json
cosign attest --predicate sbom.spdx.json --type spdxjson \
  --key cosign.key \
  my-registry.io/my-app:v1.0.0

# Verify attestation
cosign verify-attestation \
  --key cosign.pub \
  --type spdxjson \
  my-registry.io/my-app:v1.0.0 | jq '.payload | @base64d | fromjson'

# ── ENFORCE IN CLUSTER WITH KYVERNO ───────────────────────────────
```

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-image-signature
    match:
      any:
      - resources:
          kinds: [Pod]
    verifyImages:
    - imageReferences:
      - "my-registry.io/*"
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              <content of cosign.pub>
              -----END PUBLIC KEY-----
```

---

### 🔴 Q72. What is an SBOM and how do you generate one?

**Explanation:**
An **SBOM (Software Bill of Materials)** is a machine-readable inventory of all components in a software artifact — OS packages, libraries, transitive dependencies, and their versions and licenses.

SBOMs enable:
- **Vulnerability management** — when a new CVE is published, you can immediately check which images/services are affected
- **License compliance** — ensure no GPL-licensed code is in proprietary products
- **Supply chain security** — know exactly what's in your software
- **Incident response** — quickly identify blast radius of a compromised library

Standard formats: **SPDX** (Linux Foundation), **CycloneDX** (OWASP).

```bash
# ── GENERATE SBOM WITH SYFT ──────────────────────────────────────
# Install syft
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh

# Generate SBOM from container image
syft my-registry.io/my-app:v1.0.0 -o spdx-json > sbom.spdx.json
syft my-registry.io/my-app:v1.0.0 -o cyclonedx-json > sbom.cdx.json
syft my-registry.io/my-app:v1.0.0 -o table          # human-readable

# Generate from filesystem (in CI)
syft packages dir:. -o spdx-json > sbom.spdx.json

# ── SCAN SBOM FOR VULNERABILITIES ─────────────────────────────────
grype sbom:sbom.spdx.json
grype sbom:sbom.spdx.json --fail-on critical

# ── ATTACH SBOM TO IMAGE ──────────────────────────────────────────
cosign attest \
  --predicate sbom.spdx.json \
  --type spdxjson \
  --key cosign.key \
  my-registry.io/my-app:v1.0.0

# ── TRIVY SBOM ────────────────────────────────────────────────────
trivy image --format spdx-json my-registry.io/my-app:v1.0.0 > sbom.trivy.spdx.json
```

---

## CKS Domain 6: Monitoring, Logging & Runtime Security (20%) {#cks-domain-6}

---

### 🟢 Q73. What is Falco and how do you use it?

**Explanation:**
Falco is a **Cloud Native Runtime Security** tool that monitors system calls at the kernel level and generates alerts when suspicious behaviour is detected. Unlike image scanning (which is static/pre-deployment), Falco monitors what's actually happening at runtime.

Falco uses `eBPF` or a kernel module to intercept all system calls made by all processes on the node. It evaluates each syscall against a set of rules using the Falco expression language. When a rule matches, Falco generates an alert with configurable priority (EMERGENCY, ALERT, CRITICAL, ERROR, WARNING, NOTICE, INFO, DEBUG).

**What Falco can detect:**
- Shell spawned inside a container (potential attack/debugging)
- Sensitive file reads (`/etc/shadow`, `/root/.ssh`)
- Container escape attempts
- Unexpected network connections
- Privilege escalation attempts
- New file writes to non-standard paths

```bash
# ── INSTALL FALCO ──────────────────────────────────────────────
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \          # use eBPF driver (preferred)
  --set tty=true

# Verify Falco is running
kubectl get pods -n falco
kubectl logs -n falco daemonset/falco | head -30

# ── FALCO COMMANDS ────────────────────────────────────────────
# List all available fields
falco --list

# List all built-in rules
falco --list-rules

# Test Falco is working (triggers a "shell in container" alert)
kubectl exec -it some-pod -- /bin/sh
# Check Falco logs:
kubectl logs -n falco daemonset/falco | grep "A shell was spawned"

# ── ALERT EXAMPLE ──────────────────────────────────────────────
# 10:15:35.123456789: Warning A shell was spawned in a container
# (user=root user_loginuid=-1
#  k8s.ns=production k8s.pod=my-app-xxx
#  container=my-app
#  shell=bash parent=runc cmdline=bash
#  terminal=34816 container_id=abc123
#  image=my-app:v1)

# ── FALCO WITH FALCOSIDEKICK ───────────────────────────────────
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/..." \
  --set falcosidekick.config.pagerduty.routingKey="abc123" \
  --set falcosidekick.webui.enabled=true
```

---

### 🟡 Q74. How do you write custom Falco rules?

**Explanation:**
The default Falco rules cover common attack patterns, but you'll need custom rules for application-specific security requirements. The Falco rule language uses conditions built from fields (like `proc.name`, `fd.name`, `container.name`) with standard comparison operators.

Key concepts:
- **Macros** — reusable condition fragments (like functions)
- **Lists** — named lists of values to check against
- **Rules** — the actual detection logic with output and priority

Writing good rules requires knowing the available Falco fields (`falco --list`) and testing them without `container` scope first.

```yaml
# /etc/falco/custom-rules.yaml

# ── LISTS (reusable value sets) ────────────────────────────────
- list: approved_tools
  items: [kubectl, helm, terraform]

- list: sensitive_files
  items: [/etc/shadow, /etc/gshadow, /etc/passwd, /etc/sudoers,
          /root/.ssh/authorized_keys, /root/.aws/credentials]

# ── MACROS (reusable conditions) ───────────────────────────────
- macro: is_container
  condition: container.id != host

- macro: is_production
  condition: k8s.ns.name = "production"

# ── RULES ──────────────────────────────────────────────────────

# Detect network tools in containers
- rule: Network Tool Executed in Container
  desc: >
    Detect execution of common network tools (curl, wget, nc)
    inside a container — potential data exfiltration or C2 communication
  condition: >
    spawned_process
    and is_container
    and proc.name in (curl, wget, nc, ncat, netcat, nmap, masscan)
    and not proc.pname in (apt-get, apk)  # exclude package managers
  output: >
    Network tool executed (user=%user.name command=%proc.cmdline
    container=%container.name image=%container.image.repository
    namespace=%k8s.ns.name pod=%k8s.pod.name)
  priority: WARNING
  tags: [network, container, T1048]   # MITRE ATT&CK tag

# Detect writes to sensitive files
- rule: Write to Sensitive File
  desc: Detect attempts to write to system-sensitive files
  condition: >
    open_write
    and is_container
    and fd.name in (sensitive_files)
  output: >
    Write to sensitive file
    (user=%user.name file=%fd.name container=%container.name
     image=%container.image.repository pid=%proc.pid)
  priority: ERROR

# Detect privileged container launch
- rule: Privileged Container Started
  desc: A privileged container was started
  condition: >
    container_started
    and container.privileged = true
    and not k8s.ns.name in (kube-system, monitoring)  # allow in infra namespaces
  output: >
    Privileged container started
    (container=%container.name image=%container.image.repository
     namespace=%k8s.ns.name pod=%k8s.pod.name)
  priority: CRITICAL

# Detect unexpected outbound connection from production pods
- rule: Unexpected Outbound Connection
  desc: Container making outbound connection to unexpected destination
  condition: >
    outbound
    and is_container
    and is_production
    and not (fd.sport in (80, 443, 5432, 6379, 27017))  # approved ports
    and not fd.sip.name = "*.internal.example.com"
  output: >
    Unexpected outbound connection
    (user=%user.name command=%proc.cmdline dest=%fd.rip
     port=%fd.rport container=%container.name pod=%k8s.pod.name)
  priority: WARNING
```

```bash
# Load custom rules
# Edit /etc/falco/falco.yaml:
# rules_file:
#   - /etc/falco/falco_rules.yaml
#   - /etc/falco/custom-rules.yaml

# In Kubernetes, use ConfigMap
kubectl create configmap falco-custom-rules \
  --from-file=custom-rules.yaml=./custom-rules.yaml \
  -n falco

# Hot-reload Falco rules (without restart)
kill -1 $(cat /var/run/falco.pid)
# or
kubectl exec -n falco daemonset/falco -- kill -1 1
```

---

### 🟡 Q75. How do you enforce immutable container filesystems?

**Explanation:**
An immutable root filesystem prevents attackers from modifying container binaries, installing tools, or leaving backdoors. Combined with seccomp and AppArmor, it significantly reduces what an attacker can do inside a compromised container.

The practical challenge: many applications need to write to disk for temp files, logs, caches, etc. The solution is to use `readOnlyRootFilesystem: true` and then mount specific writable paths as `emptyDir` volumes.

```yaml
# Pod with immutable filesystem + writable mounts
apiVersion: v1
kind: Pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: my-app:v1
    securityContext:
      readOnlyRootFilesystem: true          # immutable root
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
    volumeMounts:
    - name: tmp             # app needs temp space
      mountPath: /tmp
    - name: var-run         # for pid files, sockets
      mountPath: /var/run
    - name: app-logs        # application log directory
      mountPath: /app/logs
    - name: app-cache       # application cache directory
      mountPath: /app/cache
  volumes:
  - name: tmp
    emptyDir: {}
  - name: var-run
    emptyDir: {}
  - name: app-logs
    emptyDir: {}           # or a PVC for persistent logs
  - name: app-cache
    emptyDir:
      sizeLimit: 500Mi     # limit cache size
```

```bash
# Verify read-only filesystem
kubectl exec my-pod -- touch /test-write
# touch: /test-write: Read-only file system ← expected

# Writable directories work
kubectl exec my-pod -- touch /tmp/test
# (no error) ← expected

# Enforce with OPA Gatekeeper
kubectl apply -f - <<'EOF'
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sreadonlyrootfilesystem
spec:
  crd:
    spec:
      names:
        kind: K8sReadOnlyRootFilesystem
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sreadonlyrootfilesystem
      violation[{"msg": msg}] {
        c := input.review.object.spec.containers[_]
        not c.securityContext.readOnlyRootFilesystem
        msg := sprintf("Container %v must set readOnlyRootFilesystem: true", [c.name])
      }
EOF

kubectl apply -f - <<'EOF'
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sReadOnlyRootFilesystem
metadata:
  name: require-read-only-root-filesystem
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: ["production", "staging"]
EOF
```

---

### 🔴 Q76. How do you investigate suspicious activity using audit logs?

**Explanation:**
Audit logs are the forensic trail of everything that happened in the cluster. During a security incident, audit logs help answer:
- What did the attacker do after gaining access?
- Which resources were accessed or modified?
- Which service accounts or users were abused?
- When did the breach start?

Effective audit log analysis requires filtering by the right fields and correlating events across time. The audit log format is JSON — use `jq` for parsing.

```bash
# ── SETUP: assumes audit logging is configured ──────────────────
# Audit log location: /var/log/kubernetes/audit.log

# ── GENERAL QUERIES ────────────────────────────────────────────

# Pretty-print the last 10 audit events
tail -10 /var/log/kubernetes/audit.log | jq '.'

# Filter by specific user (potential compromised account)
cat /var/log/kubernetes/audit.log | \
  jq 'select(.user.username == "suspicious-user") | 
      {time: .requestReceivedTimestamp, verb: .verb,
       resource: .objectRef.resource, name: .objectRef.name}'

# Filter by service account
cat /var/log/kubernetes/audit.log | \
  jq 'select(.user.username | startswith("system:serviceaccount:production"))'

# ── SECRET ACCESS INVESTIGATION ────────────────────────────────

# Find all secret reads (who is reading secrets and when)
cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource == "secrets" and .verb == "get") | 
      {time: .requestReceivedTimestamp, user: .user.username,
       ns: .objectRef.namespace, secret: .objectRef.name}'

# Find secret creation (potential data exfiltration staging)
cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource == "secrets" and .verb == "create")'

# ── EXEC/ATTACH INVESTIGATION ──────────────────────────────────

# Find all exec-into-container events
cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.subresource == "exec") | 
      {time: .requestReceivedTimestamp, user: .user.username,
       pod: .objectRef.name, ns: .objectRef.namespace}'

# ── AUTH FAILURE INVESTIGATION ─────────────────────────────────

# Find 403 forbidden responses (auth failures)
cat /var/log/kubernetes/audit.log | \
  jq 'select(.responseStatus.code == 403) | 
      {time: .requestReceivedTimestamp, user: .user.username,
       action: .verb, resource: .objectRef.resource}'

# Find 401 unauthorised (unauthenticated access attempts)
cat /var/log/kubernetes/audit.log | \
  jq 'select(.responseStatus.code == 401)'

# ── RBAC CHANGES ───────────────────────────────────────────────

# Find RBAC modifications (privilege escalation attempts)
cat /var/log/kubernetes/audit.log | \
  jq 'select(
    .objectRef.apiGroup == "rbac.authorization.k8s.io" and
    .verb in ["create", "update", "patch", "delete"]
  ) | {time: .requestReceivedTimestamp, user: .user.username,
       action: .verb, resource: .objectRef.resource}'

# ── DEPLOYMENT CHANGES ─────────────────────────────────────────

# Track all deployment modifications
cat /var/log/kubernetes/audit.log | \
  jq 'select(
    .objectRef.resource == "deployments" and
    .verb in ["create", "update", "patch", "delete"]
  ) | {time: .requestReceivedTimestamp, user: .user.username,
       deployment: .objectRef.name, ns: .objectRef.namespace}'
```

---

# Master Cheatsheets

---

## CKA Master Cheatsheet {#cka-master-cheatsheet}

### Cluster Bootstrap & Management
```bash
# Bootstrap
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<IP>
kubeadm init --control-plane-endpoint "LB_IP:6443" --upload-certs  # HA
kubeadm join <IP>:6443 --token <t> --discovery-token-ca-cert-hash sha256:<h>
kubeadm join <IP>:6443 --token <t> --discovery-token-ca-cert-hash sha256:<h> --control-plane --certificate-key <k>
kubeadm token create --print-join-command
kubeadm token list
kubeadm certs check-expiration
kubeadm certs renew all
kubeadm upgrade plan
kubeadm upgrade apply v1.29.0
kubeadm upgrade node                         # on worker nodes
kubeadm reset                                # wipe a node

# kubectl configure
mkdir -p $HOME/.kube && cp /etc/kubernetes/admin.conf $HOME/.kube/config

# Cluster health
kubectl cluster-info
kubectl get componentstatuses
kubectl get nodes -o wide
kubectl get pods -n kube-system

# Drain / Cordon / Uncordon
kubectl cordon <node>                        # mark unschedulable
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --force
kubectl uncordon <node>
kubectl taint nodes <node> key=val:NoSchedule
kubectl taint nodes <node> key=val:NoSchedule-   # remove taint
```

### etcd Backup & Restore
```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Status check
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd.db --write-out=table

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd.db \
  --data-dir=/var/lib/etcd-new

# Health check
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Member list
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### RBAC
```bash
# Roles
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create clusterrole node-viewer --verb=get,list,watch --resource=nodes
kubectl create clusterrole deploy-manager --verb=get,list,create,update,delete --resource=deployments

# Bindings
kubectl create rolebinding rb1 --role=pod-reader --user=jane -n dev
kubectl create rolebinding rb2 --role=pod-reader --serviceaccount=dev:my-sa -n dev
kubectl create clusterrolebinding crb1 --clusterrole=cluster-admin --user=admin

# Test
kubectl auth can-i get pods -n dev --as=jane
kubectl auth can-i '*' '*' --as=system:admin
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa

# Inspect
kubectl get roles,rolebindings -A
kubectl get clusterroles,clusterrolebindings
kubectl describe rolebinding <name> -n <ns>
```

### kubeconfig & Contexts
```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <name>
kubectl config set-context --current --namespace=<ns>
kubectl config view --minify
kubectl config set-cluster <name> --server=<url> --certificate-authority=<ca>
kubectl config set-credentials <name> --client-certificate=<cert> --client-key=<key>
kubectl config set-context <name> --cluster=<c> --user=<u> --namespace=<ns>
# Merge configs
KUBECONFIG=~/.kube/config:~/.kube/other-config kubectl config view --flatten > merged
```

### Helm
```bash
# Repos
helm repo add <name> <url>
helm repo update
helm repo list
helm repo remove <name>
helm search repo <keyword>
helm search hub <keyword>

# Install
helm install <release> <chart> -n <ns> --create-namespace
helm install <release> <chart> -f values.yaml
helm install <release> <chart> --set key=val --set key2=val2
helm install <release> <chart> --dry-run --debug
helm install <release> <chart> --wait --timeout 5m

# Inspect
helm list -A
helm status <release>
helm get values <release>
helm get values <release> --all
helm get manifest <release>
helm history <release>
helm show values <chart>

# Upgrade / Rollback
helm upgrade <release> <chart> --set key=val
helm upgrade <release> <chart> --reuse-values
helm upgrade --install <release> <chart>
helm rollback <release> <revision>
helm rollback <release>

# Cleanup
helm uninstall <release> -n <ns>
helm uninstall <release> --keep-history

# Template / Package
helm template <release> <chart> -f values.yaml
helm lint ./my-chart
helm package ./my-chart
helm dependency update ./my-chart
```

### Kustomize
```bash
kubectl apply -k <overlay-dir>
kubectl delete -k <overlay-dir>
kubectl diff -k <overlay-dir>
kubectl kustomize <overlay-dir>         # print rendered YAML without applying
kubectl kustomize <overlay-dir> > rendered.yaml
```

### CRDs & Operators
```bash
kubectl get crds
kubectl get crd <name> -o yaml
kubectl describe crd <name>
kubectl explain <crd-kind>.<group>
kubectl explain <crd-kind>.<group>.spec
kubectl get <crd-plural>
kubectl get <crd-plural> -A
```

### Nodes & Scheduling
```bash
# Labels and taints
kubectl label node <node> disktype=ssd
kubectl label node <node> disktype-              # remove label
kubectl taint nodes <node> key=val:NoSchedule
kubectl taint nodes <node> key=val:NoSchedule-

# Resources and metrics
kubectl top nodes
kubectl top pods
kubectl top pods --containers
kubectl top pods -A --sort-by=memory
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl describe nodes | grep Taint

# HPA
kubectl autoscale deployment <name> --cpu-percent=70 --min=2 --max=10
kubectl get hpa
kubectl describe hpa <name>

# VPA
kubectl describe vpa <name>
kubectl get vpa

# PriorityClass
kubectl get priorityclasses
```

### Services & Networking
```bash
# Services
kubectl expose deployment <name> --type=ClusterIP --port=80 --target-port=8080
kubectl expose deployment <name> --type=NodePort --port=80 --node-port=30080
kubectl get svc
kubectl get endpoints <svc>
kubectl describe svc <name>

# NetworkPolicy
kubectl get networkpolicies -A
kubectl describe networkpolicy <name> -n <ns>

# DNS debug
kubectl run dns-test --image=busybox --rm -it -- nslookup <service>
kubectl run dns-test --image=busybox --rm -it -- nslookup kubernetes.default

# kube-proxy
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
kubectl rollout restart daemonset kube-proxy -n kube-system
iptables -t nat -L KUBE-SERVICES | head -30
ipvsadm -Ln

# Gateway API
kubectl get gateways -A
kubectl get httproutes -A
kubectl describe gateway <name> -n <ns>
```

### Storage
```bash
kubectl get pv
kubectl get pvc -A
kubectl describe pv <name>
kubectl describe pvc <name> -n <ns>
kubectl get storageclass
kubectl describe storageclass <name>
kubectl get csidrivers
kubectl get csinodes

# Expand PVC
kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

### Troubleshooting
```bash
# Pod issues
kubectl describe pod <pod>
kubectl logs <pod>
kubectl logs <pod> --previous
kubectl logs <pod> -c <container>
kubectl logs <pod> --tail=100 --since=1h
kubectl logs -l app=my-app --prefix
kubectl get events --sort-by=.lastTimestamp -n <ns>
kubectl get events --field-selector reason=BackOff

# Exec / Debug
kubectl exec -it <pod> -- /bin/bash
kubectl exec <pod> -- env
kubectl debug <pod> -it --image=busybox
kubectl debug node/<node> -it --image=ubuntu

# Node troubleshooting (on the node)
systemctl status kubelet
systemctl restart kubelet
journalctl -u kubelet -n 200 --no-pager
journalctl -u kubelet --since "1 hour ago" | grep -i error
journalctl -u containerd -n 50
journalctl -k | grep -i oom
crictl ps
crictl logs <container-id>
crictl info
df -h
free -m
ss -tlnp
```

---

## CKAD Master Cheatsheet {#ckad-master-cheatsheet}

### Imperative Pod & Deployment Creation
```bash
# Pods
kubectl run nginx --image=nginx
kubectl run nginx --image=nginx --port=80
kubectl run nginx --image=nginx --env="ENV=prod" --labels="app=nginx,tier=frontend"
kubectl run nginx --image=nginx --requests="cpu=100m,memory=128Mi" --limits="cpu=500m,memory=256Mi"
kubectl run nginx --image=nginx --serviceaccount=my-sa
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl run nginx --image=nginx --command -- sleep 3600
kubectl run nginx --image=busybox --rm -it -- /bin/sh   # ephemeral debug pod

# Deployments
kubectl create deployment my-app --image=my-app:v1 --replicas=3
kubectl create deployment my-app --image=my-app:v1 --dry-run=client -o yaml > deploy.yaml

# Services
kubectl create service clusterip my-svc --tcp=80:8080
kubectl create service nodeport my-svc --tcp=80:8080 --node-port=30080
kubectl expose pod my-pod --name=my-svc --port=80 --target-port=8080
kubectl expose deployment my-app --name=my-app-svc --port=80 --type=ClusterIP

# ConfigMaps
kubectl create configmap my-cm --from-literal=KEY=VALUE
kubectl create configmap my-cm --from-literal=K1=V1 --from-literal=K2=V2
kubectl create configmap my-cm --from-file=config.properties
kubectl create configmap my-cm --from-file=key=./file.txt
kubectl create configmap my-cm --from-env-file=.env

# Secrets
kubectl create secret generic my-secret --from-literal=password=s3cr3t
kubectl create secret generic my-secret --from-file=password.txt
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key
kubectl create secret docker-registry regcred --docker-server=my-reg.io \
  --docker-username=user --docker-password=pass

# Jobs & CronJobs
kubectl create job my-job --image=busybox -- echo "done"
kubectl create job my-job --image=busybox --dry-run=client -o yaml -- echo done > job.yaml
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" -- echo tick
kubectl create job manual-run --from=cronjob/my-cron    # manually trigger CronJob
```

### Deployments: Updates & Rollbacks
```bash
# Update image
kubectl set image deployment/my-app app=my-app:v2
kubectl set image deployment/my-app app=my-app:v2 container2=other:v3

# Apply updated file
kubectl apply -f deployment.yaml

# Monitor rollout
kubectl rollout status deployment/my-app
kubectl rollout history deployment/my-app
kubectl rollout history deployment/my-app --revision=3

# Pause / Resume rollout
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app

# Rollback
kubectl rollout undo deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=2

# Restart (force pod restart with same config)
kubectl rollout restart deployment/my-app
kubectl rollout restart daemonset/my-ds

# Scale
kubectl scale deployment my-app --replicas=5
kubectl autoscale deployment my-app --cpu-percent=70 --min=2 --max=10
```

### Labels, Selectors & Annotations
```bash
# Labels
kubectl get pods --show-labels
kubectl get pods -l app=my-app
kubectl get pods -l app=my-app,env=prod
kubectl get pods -l 'app in (my-app, other-app)'
kubectl get pods -l 'app notin (old-app)'
kubectl get pods -l '!deprecated'
kubectl label pod my-pod env=prod
kubectl label pod my-pod env=staging --overwrite
kubectl label pod my-pod env-              # remove label

# Annotations
kubectl annotate pod my-pod description="production pod"
kubectl annotate pod my-pod description-   # remove annotation

# Field selectors
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector spec.nodeName=worker-1
```

### Debugging & Observability
```bash
# Logs
kubectl logs my-pod
kubectl logs my-pod --previous
kubectl logs my-pod -c container-name
kubectl logs my-pod --all-containers=true
kubectl logs -f my-pod                      # follow
kubectl logs my-pod --tail=50 --since=30m
kubectl logs -l app=my-app --prefix=true    # multi-pod logs

# Events
kubectl get events -n default
kubectl get events --sort-by=.lastTimestamp
kubectl get events --field-selector type=Warning
kubectl get events --field-selector reason=BackOff,type=Warning

# Describe
kubectl describe pod my-pod
kubectl describe deployment my-app
kubectl describe service my-svc
kubectl describe node worker-1

# Execute
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -c sidecar -- /bin/sh
kubectl exec my-pod -- env | grep DB
kubectl exec my-pod -- cat /etc/config/app.properties

# Debug containers
kubectl debug my-pod -it --image=busybox    # add ephemeral container
kubectl debug my-pod --copy-to=debug-pod -it --image=ubuntu  # copy pod to debug

# Port forward
kubectl port-forward pod/my-pod 8080:80
kubectl port-forward svc/my-svc 8080:80
kubectl port-forward deployment/my-app 8080:8080

# Resource usage
kubectl top pods
kubectl top pods --containers
kubectl top pods -l app=my-app
kubectl top nodes
```

### Configuration
```bash
# ConfigMap operations
kubectl get configmap my-cm -o yaml
kubectl get configmap my-cm -o jsonpath='{.data}'
kubectl edit configmap my-cm
kubectl describe configmap my-cm

# Secret operations
kubectl get secret my-secret -o yaml
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
kubectl edit secret my-secret

# Service accounts
kubectl create serviceaccount my-sa
kubectl get serviceaccounts
kubectl describe serviceaccount my-sa
kubectl create token my-sa                  # create a short-lived token
kubectl create token my-sa --duration=1h

# Resource quotas and limits
kubectl get resourcequota -n my-ns
kubectl describe resourcequota my-quota -n my-ns
kubectl get limitrange -n my-ns
kubectl describe limitrange my-lr -n my-ns
```

### Output Formatting
```bash
# Output formats
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o wide
kubectl get pods -o name
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# JSONPath
kubectl get pod my-pod -o jsonpath='{.spec.containers[0].image}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'

# go-template
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}} {{.status.phase}}{{"\n"}}{{end}}'

# Sort
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.metadata.name

# Explain
kubectl explain pod.spec
kubectl explain pod.spec.containers.securityContext
kubectl explain deployment.spec.strategy
```

---

## CKS Master Cheatsheet {#cks-master-cheatsheet}

### Scanning & Benchmarking
```bash
# kube-bench
kube-bench run --targets master
kube-bench run --targets node
kube-bench run --targets etcd
kube-bench run --json > results.json
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench

# Trivy — image scanning
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL nginx:latest
trivy image --exit-code 1 --severity CRITICAL my-app:v1
trivy image --format json my-app:v1 > scan.json
trivy image --format sarif my-app:v1 > scan.sarif
trivy image --input nginx.tar            # scan from tar archive
trivy fs --severity HIGH,CRITICAL .      # scan filesystem
trivy k8s --report summary cluster       # scan whole cluster
trivy config k8s-manifests/              # scan manifests
trivy k8s --namespace production cluster # specific namespace

# Kubesec
kubesec scan pod.yaml
cat pod.yaml | kubesec scan -
docker run -i kubesec/kubesec:latest scan - < pod.yaml

# KubeLinter
kube-linter lint deployment.yaml
kube-linter lint ./manifests/
kube-linter lint --charts ./my-chart/
kube-linter checks list
```

### RBAC & Permissions
```bash
# Check permissions
kubectl auth can-i get pods --as=jane -n dev
kubectl auth can-i --list --as=system:serviceaccount:prod:my-sa -n prod
kubectl auth can-i '*' '*'                           # am I cluster-admin?

# Find over-privileged bindings
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name == "cluster-admin") | 
      {name: .metadata.name, subjects: .subjects}'

# Find all SA permissions in a namespace
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq '.items[] | select(.subjects[]? | .kind == "ServiceAccount")'
```

### Pod Security Admission (PSA)
```bash
# Label namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

kubectl label namespace dev \
  pod-security.kubernetes.io/enforce=baseline

# Check namespace labels
kubectl get namespace production --show-labels

# Dry-run to test compliance
kubectl apply --dry-run=server -n production -f pod.yaml
```

### Secrets & Encryption
```bash
# Create secrets
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password='s3cr3t!'
kubectl create secret tls my-tls --cert=cert.pem --key=key.pem

# Read secret value
kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 -d

# Check encryption (etcdctl)
ETCDCTL_API=3 etcdctl get /registry/secrets/default/db-creds \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | strings
# Encrypted: shows "k8s:enc:aescbc" prefix

# Re-encrypt all secrets after enabling encryption
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

### AppArmor
```bash
# Check if AppArmor is enabled
cat /sys/module/apparmor/parameters/enabled
aa-status
aa-status | grep -i enforce

# Load a profile
apparmor_parser -q /etc/apparmor.d/my-profile
apparmor_parser -r /etc/apparmor.d/my-profile   # reload

# List loaded profiles
aa-status --json | jq '.profiles | keys'

# Verify in container
kubectl exec my-pod -- cat /proc/1/attr/current

# Apply profile (K8s 1.30+)
# spec.containers[].securityContext.appArmorProfile.type=Localhost
# spec.containers[].securityContext.appArmorProfile.localhostProfile=profile-name

# Apply profile (K8s < 1.30)
# annotation: container.apparmor.security.beta.kubernetes.io/<container>: localhost/<profile>
```

### Seccomp
```bash
# Profile location on nodes
ls /var/lib/kubelet/seccomp/profiles/

# Copy profile to all nodes (via DaemonSet or manually)
scp my-app.json worker-1:/var/lib/kubelet/seccomp/profiles/
scp my-app.json worker-2:/var/lib/kubelet/seccomp/profiles/

# Verify seccomp is active in container
kubectl exec my-pod -- grep Seccomp /proc/1/status
# Seccomp: 2   (2 = SECCOMP_MODE_FILTER = active)

# RuntimeDefault profile in pod spec
# spec.securityContext.seccompProfile.type=RuntimeDefault
```

### Image Signing (Cosign)
```bash
# Key management
cosign generate-key-pair
cosign generate-key-pair --kms gcpkms://...   # hardware-backed key

# Sign
cosign sign --key cosign.key my-registry.io/my-app:v1
cosign sign --key cosign.key --annotations env=production my-registry.io/my-app:v1

# Verify
cosign verify --key cosign.pub my-registry.io/my-app:v1
cosign verify --key cosign.pub my-registry.io/my-app:v1 | jq '.[0]'

# Attach SBOM
cosign attest --predicate sbom.json --type spdxjson --key cosign.key my-app:v1
cosign verify-attestation --key cosign.pub --type spdxjson my-app:v1

# Keyless (OIDC)
COSIGN_EXPERIMENTAL=1 cosign sign my-registry.io/my-app:v1
COSIGN_EXPERIMENTAL=1 cosign verify \
  --certificate-identity-regexp=".*@mycompany.com" \
  --certificate-oidc-issuer="https://accounts.google.com" \
  my-registry.io/my-app:v1
```

### SBOM Generation
```bash
# Syft
syft my-app:v1 -o spdx-json > sbom.spdx.json
syft my-app:v1 -o cyclonedx-json > sbom.cdx.json
syft my-app:v1 -o table                           # human readable
syft packages dir:. -o spdx-json > sbom.spdx.json # from filesystem

# Scan SBOM
grype sbom:sbom.spdx.json
grype sbom:sbom.spdx.json --fail-on critical

# Trivy SBOM
trivy image --format spdx-json my-app:v1 > sbom.trivy.spdx.json
```

### Audit Logging
```bash
# kube-apiserver flags
# --audit-log-path=/var/log/kubernetes/audit.log
# --audit-policy-file=/etc/kubernetes/audit-policy.yaml
# --audit-log-maxage=30
# --audit-log-maxbackup=10
# --audit-log-maxsize=100

# View and query
tail -f /var/log/kubernetes/audit.log | jq .
cat /var/log/kubernetes/audit.log | jq 'select(.verb == "delete")'
cat /var/log/kubernetes/audit.log | \
  jq 'select(.user.username == "bad-actor") | 
      {time: .requestReceivedTimestamp, action: .verb, resource: .objectRef.resource}'
cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource == "secrets" and .verb == "get")'
cat /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.subresource == "exec")'
cat /var/log/kubernetes/audit.log | \
  jq 'select(.responseStatus.code == 403)'
```

### Falco
```bash
# Install
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco -n falco --create-namespace \
  --set driver.kind=ebpf \
  --set falcosidekick.enabled=true

# Manage
kubectl get pods -n falco
kubectl logs -n falco daemonset/falco
kubectl logs -n falco daemonset/falco | grep -i "warning\|critical\|error"

# List rules and fields
falco --list                         # all available fields
falco --list-rules                   # all loaded rules
falco --validate /etc/falco/custom-rules.yaml  # validate rule file

# Reload rules
kill -1 $(pgrep falco)
kubectl exec -n falco daemonset/falco -- kill -1 1

# Rule files
ls /etc/falco/
# falco_rules.yaml         built-in rules
# falco_rules.local.yaml   override/extend built-in rules
# custom-rules.yaml        your custom rules

# Falco config
cat /etc/falco/falco.yaml | grep rules_file
```

### Network Security
```bash
# NetworkPolicy debug
kubectl get networkpolicies -A
kubectl describe networkpolicy <name> -n <ns>

# Test connectivity (from pod)
kubectl exec my-pod -- nc -zv target-pod-ip 8080    # TCP test
kubectl exec my-pod -- curl -sf http://service:80   # HTTP test
kubectl exec my-pod -- nslookup kubernetes.default   # DNS test

# Cilium encryption
kubectl exec -n kube-system ds/cilium -- cilium encrypt status
kubectl exec -n kube-system ds/cilium -- wg show
kubectl exec -n kube-system ds/cilium -- cilium status | grep Encryption
```

### Node / Kubelet Hardening
```bash
# kubelet config
cat /var/lib/kubelet/config.yaml
# Check: anonymous.enabled=false, authorization.mode=Webhook, readOnlyPort=0

# Verify anonymous auth disabled
curl -sk https://localhost:10250/pods          # should return 401
curl -sk http://localhost:10255/pods 2>&1      # should be refused (port closed)

# Verify read-only port disabled
ss -tlnp | grep 10255                         # should show nothing

# kube-apiserver hardening checks
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep anonymous
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep profiling
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep audit

# TLS check
openssl s_client -connect localhost:6443 2>&1 | grep -E 'Protocol|Cipher'
```

### Container Runtime
```bash
# RuntimeClass
kubectl get runtimeclass
kubectl describe runtimeclass gvisor

# gVisor verification
kubectl exec my-sandboxed-pod -- uname -r    # shows gVisor kernel
kubectl exec my-sandboxed-pod -- dmesg | head -3

# containerd
crictl ps
crictl images
crictl inspect <container-id>
crictl exec -it <container-id> /bin/sh
crictl logs <container-id>
crictl info | jq .config.containerd.runtimes
```

### OPA / Gatekeeper
```bash
kubectl get constrainttemplates
kubectl get constraints -A
kubectl describe constrainttemplate <name>
kubectl describe <constraint-kind> <name>

# Check violations
kubectl get <constraint-kind> <name> -o jsonpath='{.status.violations}'

# Dry-run test
kubectl apply --dry-run=server -f pod.yaml
```

---

## Exam Quick Reference

### Time-Saving Tips
```bash
# Alias setup (do this at exam start)
alias k=kubectl
alias kn='kubectl -n'
alias kaf='kubectl apply -f'
export do='--dry-run=client -o yaml'
export now='--force --grace-period 0'
source <(kubectl completion bash)
complete -o default -F __start_kubectl k

# Generate YAML without applying
kubectl run nginx --image=nginx $do > pod.yaml
kubectl create deployment my-app --image=nginx --replicas=3 $do > deploy.yaml
kubectl create service clusterip my-svc --tcp=80:80 $do > svc.yaml
kubectl create configmap my-cm --from-literal=k=v $do > cm.yaml
kubectl create secret generic my-sec --from-literal=pass=abc $do > sec.yaml

# Force delete
kubectl delete pod stuck-pod $now

# Quick namespace
kubectl get pods -n kube-system
```

### Static Pod Locations
```bash
/etc/kubernetes/manifests/             # kubeadm default
/var/lib/kubelet/config.yaml           # kubelet config
/var/lib/kubelet/kubeconfig            # kubelet kubeconfig
/etc/kubernetes/pki/                   # certificates
/etc/cni/net.d/                        # CNI config
/var/lib/kubelet/seccomp/profiles/     # seccomp profiles
/etc/apparmor.d/                       # AppArmor profiles
/var/log/kubernetes/audit.log          # audit log
/etc/falco/                            # Falco config and rules
```

### Critical YAML Snippets
```yaml
# SecurityContext — restricted
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop: [ALL]

# Resource requests/limits
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

# Node affinity — required
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: [ssd]

# Toleration
tolerations:
- key: "env"
  operator: "Equal"
  value: "prod"
  effect: "NoSchedule"

# Rolling update strategy
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1

# Liveness probe
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3

# Init container
initContainers:
- name: init
  image: busybox
  command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']

# Volume + mount
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
containers:
- name: app
  volumeMounts:
  - name: data
    mountPath: /data
```

---

*This guide covers the complete official 2025 CNCF curriculum for CKA v1.34+, CKAD v1.35, and CKS v1.31+.*
*Last updated: June 2026*
