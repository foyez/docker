# 9.2 Kubernetes Architecture

Understanding Kubernetes architecture, core concepts, and essential components for container orchestration.

---

## What is Kubernetes?

**Definition:**
```
Kubernetes (K8s) = Container orchestration platform
- Automates deployment
- Manages scaling
- Handles operations
- Maintains desired state
```

**Origin:**
```
Created: Google (2014)
Based on: Borg and Omega systems
Open Source: 2014
Donated to: CNCF (Cloud Native Computing Foundation)
Current: Industry standard for container orchestration
```

---

## Core Architecture

### Cluster Components

```
Kubernetes Cluster:
├── Control Plane (Master)
│   ├── API Server
│   ├── Scheduler
│   ├── Controller Manager
│   └── etcd
│
└── Worker Nodes
    ├── kubelet
    ├── kube-proxy
    └── Container Runtime
```

---

### Control Plane Components

**1. API Server (kube-apiserver):**
```
Purpose: Frontend for Kubernetes control plane
- Exposes Kubernetes API
- Processes REST operations
- Validates and configures data
- Gateway for all cluster operations

Communication:
All components → API Server
Users/kubectl → API Server
```

**2. Scheduler (kube-scheduler):**
```
Purpose: Assigns Pods to Nodes
- Watches for new Pods
- Selects best node
- Considers resource requirements
- Respects constraints and affinity

Process:
New Pod → Scheduler → Evaluates nodes → Assigns to best node
```

**3. Controller Manager (kube-controller-manager):**
```
Purpose: Runs controller processes
- Node Controller: Monitors node health
- Replication Controller: Maintains Pod count
- Endpoints Controller: Populates endpoints
- Service Account Controller: Creates default accounts

Reconciliation loop:
Desired state → Controller → Current state → Actions
```

**4. etcd:**
```
Purpose: Distributed key-value store
- Stores cluster state
- Configuration data
- Metadata
- Consistent and highly-available

Critical:
etcd down = cluster down
Regular backups essential
```

---

### Worker Node Components

**1. kubelet:**
```
Purpose: Node agent
- Runs on each node
- Manages Pods and containers
- Reports node status
- Executes container actions

Process:
API Server → kubelet → Container Runtime → Containers
```

**2. kube-proxy:**
```
Purpose: Network proxy
- Maintains network rules
- Enables service abstraction
- Load balances traffic
- Implements Services

Modes:
- iptables (default)
- IPVS (high performance)
- userspace (legacy)
```

**3. Container Runtime:**
```
Purpose: Runs containers
Options:
- containerd (most common)
- CRI-O
- Docker (deprecated)

Interface: CRI (Container Runtime Interface)
```

---

## Core Concepts

### Pods

**Definition:**
```
Pod = Smallest deployable unit
- One or more containers
- Shared network namespace
- Shared storage volumes
- Single IP address
```

**Simple Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

**Multi-Container Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
  
  - name: sidecar
    image: logging-agent
    volumeMounts:
    - name: logs
      mountPath: /var/log
  
  volumes:
  - name: logs
    emptyDir: {}
```

**Pod Lifecycle:**
```
Pending → Running → Succeeded/Failed

Pending: Waiting for scheduling
Running: Bound to node, containers running
Succeeded: All containers terminated successfully
Failed: At least one container failed
Unknown: Node communication lost
```

---

### Deployments

**Definition:**
```
Deployment = Manages Pod replicas
- Declarative updates
- Rolling updates
- Rollback capability
- Scaling
```

**Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

**How it works:**
```
Deployment
    ↓
ReplicaSet (manages replicas)
    ↓
Pods (3 replicas)

Update Deployment → New ReplicaSet → New Pods
Old ReplicaSet scales down
Rollback → Old ReplicaSet scales up
```

---

### Services

**Definition:**
```
Service = Stable endpoint for Pods
- Abstract access to Pods
- Load balancing
- Service discovery
- Stable IP and DNS
```

**Service Types:**

**1. ClusterIP (Default):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - port: 3000
    targetPort: 8080

# Internal only: api-service.default.svc.cluster.local
```

**2. NodePort:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30000

# Accessible: <NodeIP>:30000
```

**3. LoadBalancer:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080

# Creates cloud load balancer
# Accessible: <LoadBalancerIP>:80
```

**4. ExternalName:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com

# DNS alias to external service
```

---

### ConfigMaps

**Definition:**
```
ConfigMap = Configuration data
- Key-value pairs
- Configuration files
- Environment variables
- Not for secrets
```

**Create from literals:**
```bash
kubectl create configmap app-config \
  --from-literal=app.env=production \
  --from-literal=app.port=3000
```

**Create from file:**
```bash
kubectl create configmap nginx-config \
  --from-file=nginx.conf
```

**YAML:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.env: production
  app.port: "3000"
  database.url: postgresql://db:5432/myapp
```

**Use in Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: app.env
    - name: APP_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: app.port
```

---

### Secrets

**Definition:**
```
Secret = Sensitive data
- Passwords
- Tokens
- SSH keys
- Base64 encoded (not encrypted by default)
```

**Create Secret:**
```bash
# From literal
kubectl create secret generic db-password \
  --from-literal=password=mysecret

# From file
kubectl create secret generic ssh-key \
  --from-file=ssh-privatekey=~/.ssh/id_rsa
```

**YAML:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: admin
  password: mysecret
```

**Use in Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: db-credentials
```

---

### Namespaces

**Definition:**
```
Namespace = Virtual cluster
- Organize resources
- Resource isolation
- Access control
- Resource quotas
```

**Default Namespaces:**
```
default: Default namespace
kube-system: System components
kube-public: Publicly accessible
kube-node-lease: Node heartbeats
```

**Create Namespace:**
```bash
kubectl create namespace development
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

**Use Namespace:**
```bash
# Deploy to namespace
kubectl apply -f deployment.yaml -n production

# Set default namespace
kubectl config set-context --current --namespace=production

# List all namespaces
kubectl get namespaces
```

**Resource Quota:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    persistentvolumeclaims: "10"
```

---

### Volumes

**Volume Types:**

**1. emptyDir:**
```yaml
# Temporary storage, deleted when Pod deleted
volumes:
- name: cache
  emptyDir: {}
```

**2. hostPath:**
```yaml
# Node's filesystem
volumes:
- name: data
  hostPath:
    path: /mnt/data
    type: Directory
```

**3. persistentVolumeClaim:**
```yaml
# Persistent storage
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

**PersistentVolume and PersistentVolumeClaim:**

```yaml
# PersistentVolume (admin creates)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast
  hostPath:
    path: /mnt/data

---
# PersistentVolumeClaim (user requests)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast

---
# Use in Pod
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-data
```

---

## Basic kubectl Commands

### Cluster Information

```bash
# Cluster info
kubectl cluster-info

# Node status
kubectl get nodes

# Component status
kubectl get componentstatuses

# API resources
kubectl api-resources
```

---

### Working with Pods

```bash
# List pods
kubectl get pods

# All namespaces
kubectl get pods -A

# Detailed info
kubectl get pods -o wide

# Describe pod
kubectl describe pod nginx-pod

# Pod logs
kubectl logs nginx-pod

# Follow logs
kubectl logs -f nginx-pod

# Previous container logs
kubectl logs nginx-pod --previous

# Execute command
kubectl exec nginx-pod -- ls /

# Interactive shell
kubectl exec -it nginx-pod -- /bin/bash

# Port forward
kubectl port-forward nginx-pod 8080:80
```

---

### Creating Resources

```bash
# From file
kubectl apply -f deployment.yaml

# From directory
kubectl apply -f ./configs/

# From URL
kubectl apply -f https://example.com/deployment.yaml

# Create from command
kubectl create deployment nginx --image=nginx

# Dry run
kubectl apply -f deployment.yaml --dry-run=client
```

---

### Updating Resources

```bash
# Update from file
kubectl apply -f deployment.yaml

# Edit resource
kubectl edit deployment nginx

# Set image
kubectl set image deployment/nginx nginx=nginx:1.25

# Scale
kubectl scale deployment nginx --replicas=5

# Autoscale
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80
```

---

### Deleting Resources

```bash
# Delete pod
kubectl delete pod nginx-pod

# Delete from file
kubectl delete -f deployment.yaml

# Delete by label
kubectl delete pods -l app=nginx

# Delete all pods
kubectl delete pods --all

# Delete namespace (and all resources)
kubectl delete namespace development
```

---

### Debugging

```bash
# Describe resource
kubectl describe pod nginx-pod

# Get events
kubectl get events

# Get logs
kubectl logs nginx-pod

# Multi-container pod logs
kubectl logs nginx-pod -c container-name

# Previous logs
kubectl logs nginx-pod --previous

# Top (resource usage)
kubectl top nodes
kubectl top pods
```

---

## Complete Application Example

### Three-Tier Application

**1. Database (StatefulSet):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
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
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

**2. Backend API (Deployment):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myapi:1.0
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: env
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
  - port: 3000
    targetPort: 3000
```

**3. Frontend (Deployment):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: myfrontend:1.0
        ports:
        - containerPort: 80
        env:
        - name: API_URL
          value: http://api:3000
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 256Mi

---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

**4. ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  env: production
  log.level: info
```

**5. Secrets:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  password: mysecretpassword
  url: postgresql://postgres:5432/myapp
```

**Deploy:**
```bash
# Create namespace
kubectl create namespace myapp

# Deploy
kubectl apply -f secrets.yaml -n myapp
kubectl apply -f configmap.yaml -n myapp
kubectl apply -f database.yaml -n myapp
kubectl apply -f api.yaml -n myapp
kubectl apply -f frontend.yaml -n myapp

# Verify
kubectl get all -n myapp

# Check logs
kubectl logs -l app=api -n myapp

# Get service URL
kubectl get svc frontend -n myapp
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The __________ is the frontend for the Kubernetes control plane.
2. A __________ is the smallest deployable unit in Kubernetes.
3. __________ stores the cluster's state and configuration data.
4. A __________ provides a stable endpoint for accessing Pods.
5. __________ manage Pod replicas and provide declarative updates.
6. The __________ component runs on each node and manages containers.

### True/False

1. ⬜ Pods can contain multiple containers
2. ⬜ Services provide load balancing across Pods
3. ⬜ ConfigMaps should be used for sensitive data
4. ⬜ Each Pod gets its own IP address
5. ⬜ Deployments directly create Pods
6. ⬜ Namespaces provide complete network isolation by default
7. ⬜ etcd is optional in Kubernetes

### Multiple Choice

1. Which component schedules Pods to Nodes?
   - A) API Server
   - B) Scheduler
   - C) kubelet
   - D) Controller Manager

2. What is the default Service type?
   - A) NodePort
   - B) LoadBalancer
   - C) ClusterIP
   - D) ExternalName

3. Which creates a ReplicaSet?
   - A) Pod
   - B) Service
   - C) Deployment
   - D) StatefulSet

4. Where should passwords be stored?
   - A) ConfigMap
   - B) Secret
   - C) Pod spec
   - D) Environment variables

5. What does kubectl apply do?
   - A) Creates resources only
   - B) Updates resources only
   - C) Creates or updates resources
   - D) Deletes resources

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. API Server (kube-apiserver)
2. Pod
3. etcd
4. Service
5. Deployments
6. kubelet

**True/False:**
1. ✅ True - Pods can have multiple containers
2. ✅ True - Services load balance to backend Pods
3. ❌ False - Use Secrets for sensitive data, not ConfigMaps
4. ✅ True - Each Pod has unique IP
5. ❌ False - Deployments create ReplicaSets, which create Pods
6. ❌ False - Need NetworkPolicies for network isolation
7. ❌ False - etcd is required, stores all cluster state

**Multiple Choice:**
1. **B** - Scheduler
2. **C** - ClusterIP
3. **C** - Deployment
4. **B** - Secret
5. **C** - Creates or updates resources

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the Kubernetes control plane and how it manages the cluster

<details>
<summary><strong>View Answer</strong></summary>

**Control Plane Architecture:**

---

### Components Overview

```
Control Plane (Master Node):
├── API Server (kube-apiserver)
├── Scheduler (kube-scheduler)
├── Controller Manager (kube-controller-manager)
└── etcd

Purpose: Manages cluster state and makes global decisions
```

---

### 1. API Server (kube-apiserver)

**Role:**
```
Central hub for all cluster operations
RESTful API gateway
Authentication and authorization
Admission control
```

**How it works:**
```
kubectl apply -f deployment.yaml
    ↓
1. Request sent to API Server
2. Authentication: Verify user identity
3. Authorization: Check user permissions
4. Admission Control: Validate/mutate request
5. Write to etcd
6. Response to kubectl
```

**Communication:**
```
All components talk to API Server:
- kubectl → API Server
- Scheduler → API Server
- Controller Manager → API Server
- kubelet → API Server
- kube-proxy → API Server

API Server is the ONLY component that talks to etcd
```

**Example Flow:**
```bash
# User creates deployment
kubectl apply -f deployment.yaml

# API Server:
1. Authenticates user (certificate/token)
2. Authorizes (RBAC check)
3. Validates YAML syntax
4. Runs admission controllers
5. Stores in etcd: /registry/deployments/default/nginx
6. Returns success to user
```

---

### 2. Scheduler (kube-scheduler)

**Role:**
```
Assigns Pods to Nodes
Watches for unscheduled Pods
Selects best node
```

**Scheduling Process:**
```
1. Watch API Server for new Pods (no node assigned)
2. Filter nodes (predicates):
   - Sufficient resources?
   - Node selector match?
   - Taints tolerated?
   - Affinity rules satisfied?
3. Score remaining nodes (priorities):
   - Resource utilization
   - Spreading across zones
   - Custom priorities
4. Select highest-scoring node
5. Bind Pod to node (via API Server)
```

**Example:**
```yaml
# New Pod created
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: 500m
        memory: 512Mi

# Scheduler process:
1. Detects Pod with no nodeName
2. Filters nodes:
   - Node1: 2 CPU, 4GB RAM ✓
   - Node2: 1 CPU, 2GB RAM ✗ (insufficient)
   - Node3: 4 CPU, 8GB RAM ✓
3. Scores:
   - Node1: 70 (moderate resources)
   - Node3: 85 (more available resources)
4. Assigns to Node3
5. Updates Pod: nodeName: node3
```

---

### 3. Controller Manager (kube-controller-manager)

**Role:**
```
Runs controller processes
Maintains desired state
Watches for changes
Reconciles actual with desired state
```

**Built-in Controllers:**

**Node Controller:**
```
Purpose: Monitor node health
- Heartbeat checks (every 5s)
- Mark nodes NotReady if unresponsive (40s)
- Evict Pods from failed nodes (5m)

Process:
1. kubelet sends heartbeat
2. Node Controller receives
3. Updates node status
4. If no heartbeat → NotReady
5. After grace period → Evict Pods
```

**Replication Controller:**
```
Purpose: Maintain desired replica count

Process:
1. Watch Deployments/ReplicaSets
2. Count current Pods
3. Compare to desired replicas
4. Create or delete Pods to match

Example:
Desired: 3 replicas
Actual: 2 replicas
Action: Create 1 Pod
```

**Endpoints Controller:**
```
Purpose: Populate Service endpoints

Process:
1. Watch Services
2. Find matching Pods
3. Create Endpoints object
4. Update when Pods change

Example:
Service selector: app=nginx
Finds: 3 Pods with app=nginx
Creates Endpoints: [PodIP1, PodIP2, PodIP3]
```

**Service Account Controller:**
```
Purpose: Create default ServiceAccounts

Process:
1. Watch for new namespaces
2. Create "default" ServiceAccount
3. Generate tokens
```

---

### 4. etcd

**Role:**
```
Distributed key-value store
Source of truth for cluster state
Stores all cluster data
```

**Data Stored:**
```
/registry/
├── pods/
│   ├── default/
│   │   ├── nginx-1
│   │   └── nginx-2
├── deployments/
├── services/
├── configmaps/
├── secrets/
└── nodes/
```

**Consistency:**
```
Uses Raft consensus algorithm
Maintains consistency across replicas
Requires quorum for writes
Typically 3 or 5 nodes (odd number)
```

**Example Write:**
```bash
kubectl create deployment nginx --image=nginx

1. API Server receives request
2. Validates and authenticates
3. Writes to etcd:
   PUT /registry/deployments/default/nginx
   {
     "apiVersion": "apps/v1",
     "kind": "Deployment",
     "metadata": {...},
     "spec": {...}
   }
4. etcd replicates to all nodes
5. Returns success
```

---

### Complete Workflow Example

**Creating a Deployment:**

```bash
kubectl apply -f deployment.yaml
```

**Step-by-step process:**

```
1. API Server:
   - Receives request
   - Authenticates user
   - Authorizes action
   - Validates YAML
   - Writes Deployment to etcd
   - Returns success

2. Controller Manager (Deployment Controller):
   - Watches etcd for Deployments
   - Detects new Deployment
   - Creates ReplicaSet
   - Writes ReplicaSet to etcd (via API Server)

3. Controller Manager (ReplicaSet Controller):
   - Watches etcd for ReplicaSets
   - Detects new ReplicaSet
   - Creates Pod specifications
   - Writes Pods to etcd (via API Server)

4. Scheduler:
   - Watches etcd for unscheduled Pods
   - Detects new Pods (no nodeName)
   - Evaluates nodes
   - Selects best nodes
   - Updates Pods with nodeName (via API Server)

5. kubelet (on selected node):
   - Watches API Server for Pods assigned to it
   - Detects new Pod assignment
   - Pulls container image
   - Starts container
   - Reports status to API Server

6. API Server:
   - Receives status update from kubelet
   - Updates Pod status in etcd
   - Pod status: Running

7. Service (if exists):
   - Endpoints Controller watches for Pods
   - Updates Service endpoints
   - kube-proxy updates network rules
   - Traffic can reach Pods
```

---

### High Availability

**Multi-Master Setup:**
```
Load Balancer
    ↓
├── Master 1 (API Server, Scheduler, Controllers)
├── Master 2 (API Server, Scheduler, Controllers)
└── Master 3 (API Server, Scheduler, Controllers)
    ↓
etcd Cluster (3 or 5 nodes)
```

**Leader Election:**
```
Controller Manager:
- Only one active (leader)
- Others standby
- Leader lease in etcd
- If leader fails, election starts

Scheduler:
- Only one active (leader)
- Same election mechanism

API Server:
- All active
- Load balanced
- Stateless
```

---

### Failure Scenarios

**API Server Fails:**
```
Impact:
- No new changes possible
- Existing Pods continue running
- kubelet caches Pod specs
- Recovery: Restart API Server

Critical: Need HA setup
```

**Scheduler Fails:**
```
Impact:
- No new Pods scheduled
- Existing Pods unaffected
- Recovery: Restart Scheduler

Mitigation: Multiple schedulers (leader election)
```

**Controller Manager Fails:**
```
Impact:
- No reconciliation
- Deployments not managed
- Dead Pods not replaced
- Recovery: Restart Controller Manager

Mitigation: Multiple managers (leader election)
```

**etcd Fails:**
```
Impact:
- Cluster state unavailable
- API Server cannot read/write
- Cluster effectively down
- Recovery: Restore from backup

Critical: ALWAYS backup etcd!
```

---

### Monitoring Control Plane

```bash
# Component status
kubectl get componentstatuses

# API Server health
curl -k https://localhost:6443/healthz

# etcd health
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Logs
kubectl logs -n kube-system kube-apiserver-master
kubectl logs -n kube-system kube-scheduler-master
kubectl logs -n kube-system kube-controller-manager-master
```

</details>

### Question 2: How do Pod networking and Service discovery work in Kubernetes?

<details>
<summary><strong>View Answer</strong></summary>

**Kubernetes Networking Model:**

---

### Core Principles

```
1. All Pods can communicate with all other Pods
2. Nodes can communicate with all Pods (and vice versa)
3. No NAT: Pod sees its own IP, others see same IP
4. Each Pod gets unique IP address
5. Containers in Pod share network namespace
```

---

### Pod Networking

**Pod IP Assignment:**
```
Cluster: 10.244.0.0/16

Node1: 10.244.1.0/24
├── Pod1: 10.244.1.5
├── Pod2: 10.244.1.6
└── Pod3: 10.244.1.7

Node2: 10.244.2.0/24
├── Pod4: 10.244.2.5
├── Pod5: 10.244.2.6
└── Pod6: 10.244.2.7

Node3: 10.244.3.0/24
├── Pod7: 10.244.3.5
├── Pod8: 10.244.3.6
└── Pod9: 10.244.3.7
```

**Container Network Interface (CNI):**
```
CNI Plugin responsibilities:
1. Allocate IP to Pod
2. Setup network namespace
3. Configure routes
4. Enable Pod-to-Pod communication

Popular CNI plugins:
- Calico (most common)
- Flannel (simple)
- Weave Net
- Cilium (eBPF-based)
```

---

### Same-Node Communication

```
Pod1 (10.244.1.5) → Pod2 (10.244.1.6)

1. Pod1 sends packet to 10.244.1.6
2. Packet goes to Node's bridge (cni0)
3. Bridge forwards to Pod2's veth interface
4. Pod2 receives packet

┌─────────────────────────────────┐
│         Node1                   │
│                                 │
│  ┌─────┐         ┌─────┐       │
│  │Pod1 │         │Pod2 │       │
│  │10.  │         │10.  │       │
│  │244. │         │244. │       │
│  │1.5  │         │1.6  │       │
│  └──┬──┘         └──┬──┘       │
│     │               │           │
│     └───────┬───────┘           │
│             │                   │
│         ┌───┴───┐               │
│         │ cni0  │               │
│         │bridge │               │
│         └───────┘               │
└─────────────────────────────────┘
```

---

### Cross-Node Communication

```
Pod1 (Node1: 10.244.1.5) → Pod7 (Node3: 10.244.3.5)

1. Pod1 sends to 10.244.3.5
2. Routes through Node1 network
3. Node1 forwards to Node3
4. Node3 routes to Pod7

With Calico (IP-in-IP or BGP):
┌─────────────┐                  ┌─────────────┐
│   Node1     │                  │   Node3     │
│             │                  │             │
│  ┌─────┐    │    Internet     │    ┌─────┐  │
│  │Pod1 │────┼─────────────────┼────│Pod7 │  │
│  └─────┘    │                  │    └─────┘  │
│ 10.244.1.5  │                  │ 10.244.3.5  │
└─────────────┘                  └─────────────┘

Packet:
Src: 10.244.1.5
Dst: 10.244.3.5
```

---

### Service Discovery

**Problem:**
```
Pods are ephemeral:
- Created and destroyed
- IPs change
- Cannot hardcode Pod IPs

Solution: Services
```

---

### How Services Work

**Service Creation:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
```

**What happens:**
```
1. Service created with selector: app=api
2. Endpoints Controller watches Service
3. Finds Pods matching selector
4. Creates Endpoints object:
   Endpoints: [10.244.1.5:8080, 10.244.2.6:8080, 10.244.3.7:8080]
5. Service gets ClusterIP: 10.96.0.10
6. kube-proxy configures iptables/IPVS rules
```

---

### ClusterIP Service

**Internal Load Balancing:**
```
Service: api
ClusterIP: 10.96.0.10:80

Backend Pods:
- 10.244.1.5:8080
- 10.244.2.6:8080
- 10.244.3.7:8080

Client request:
curl http://10.96.0.10:80

kube-proxy (iptables mode):
1. Intercepts packet to 10.96.0.10:80
2. Randomly selects backend Pod
3. DNATs to selected Pod IP
4. Packet forwarded to Pod
```

**iptables Rules:**
```bash
# Service chain
-A KUBE-SERVICES -d 10.96.0.10/32 -p tcp -m tcp --dport 80 \
   -j KUBE-SVC-API

# Load balancing (1/3 probability each)
-A KUBE-SVC-API -m statistic --mode random --probability 0.33 \
   -j KUBE-SEP-POD1
-A KUBE-SVC-API -m statistic --mode random --probability 0.50 \
   -j KUBE-SEP-POD2
-A KUBE-SVC-API -j KUBE-SEP-POD3

# DNAT to Pods
-A KUBE-SEP-POD1 -p tcp -m tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-POD2 -p tcp -m tcp -j DNAT --to-destination 10.244.2.6:8080
-A KUBE-SEP-POD3 -p tcp -m tcp -j DNAT --to-destination 10.244.3.7:8080
```

---

### DNS Service Discovery

**CoreDNS:**
```
Kubernetes DNS:
- CoreDNS runs as Deployment
- Provides DNS for Services and Pods
- Every Pod configured to use CoreDNS
```

**DNS Names:**
```
Service: api
Namespace: default

Full DNS name:
api.default.svc.cluster.local

Short forms (same namespace):
api
api.default
api.default.svc

DNS Resolution:
┌────────┐
│  Pod   │
│        │
│ Query: │
│  api   │────┐
└────────┘    │
              │
              ↓
        ┌──────────┐
        │ CoreDNS  │
        │          │
        │ Returns: │
        │10.96.0.10│
        └──────────┘
```

**Pod DNS:**
```bash
# Pod: 10.244.1.5 in default namespace
# DNS name: 10-244-1-5.default.pod.cluster.local

# Lookup
nslookup 10-244-1-5.default.pod.cluster.local
# Returns: 10.244.1.5
```

---

### NodePort Service

**External Access:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30000
```

**How it works:**
```
Every Node listens on port 30000

Client → NodeIP:30000 → kube-proxy → Pod

┌─────────┐
│ Client  │
│         │
└────┬────┘
     │
     ↓ :30000
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Node1     │     │   Node2     │     │   Node3     │
│             │     │             │     │             │
│ kube-proxy  │     │ kube-proxy  │     │ kube-proxy  │
│     │       │     │     │       │     │     │       │
└─────┼───────┘     └─────┼───────┘     └─────┼───────┘
      │                   │                   │
      └───────────────────┼───────────────────┘
                          │
                          ↓
                    ┌──────────┐
                    │   Pods   │
                    │          │
                    │ web-1    │
                    │ web-2    │
                    │ web-3    │
                    └──────────┘

Can access via any Node IP:
- http://node1:30000
- http://node2:30000
- http://node3:30000
```

---

### LoadBalancer Service

**Cloud Load Balancer:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
```

**How it works:**
```
1. Service created
2. Cloud controller creates cloud LB
3. LB configured to forward to NodePorts
4. External IP assigned

┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ↓
┌─────────────────┐
│  Cloud LB       │
│  IP: 1.2.3.4    │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ↓         ↓
┌────────┐ ┌────────┐
│ Node1  │ │ Node2  │
│ :30000 │ │ :30000 │
└────────┘ └────────┘
    │         │
    └────┬────┘
         │
         ↓
    ┌─────────┐
    │  Pods   │
    └─────────┘
```

---

### Headless Service

**No ClusterIP:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None
  selector:
    app: db
  ports:
  - port: 5432
```

**DNS Returns Pod IPs:**
```bash
nslookup db.default.svc.cluster.local

# Returns all Pod IPs:
10.244.1.5
10.244.2.6
10.244.3.7

# Use for StatefulSets:
db-0.db.default.svc.cluster.local → 10.244.1.5
db-1.db.default.svc.cluster.local → 10.244.2.6
db-2.db.default.svc.cluster.local → 10.244.3.7
```

---

### Network Policies

**Restrict Traffic:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432
```

**Effect:**
```
API Pods can:
- Receive traffic from frontend Pods (port 8080)
- Send traffic to db Pods (port 5432)
- Nothing else allowed
```

</details>

### Question 3: What's the difference between Deployments and StatefulSets?

<details>
<summary><strong>View Answer</strong></summary>

**Deployments vs StatefulSets:**

---

### Deployments

**Purpose:**
```
For stateless applications
Interchangeable Pods
No persistent identity
No ordering guarantees
```

**Characteristics:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
```

**Pod Names:**
```
web-7d8c9f6b5c-abcd1  (random suffix)
web-7d8c9f6b5c-efgh2
web-7d8c9f6b5c-ijkl3

Names change on recreation
No stable identity
```

**Scaling:**
```bash
kubectl scale deployment web --replicas=5

Creates:
web-7d8c9f6b5c-mnop4
web-7d8c9f6b5c-qrst5

Any Pod can be deleted
No specific order
```

**Updates:**
```bash
kubectl set image deployment/web nginx=nginx:1.25

Rolling update:
1. Create new Pod (nginx:1.25)
2. Wait for ready
3. Delete old Pod (nginx:1.24)
4. Repeat

Order doesn't matter
```

**Storage:**
```yaml
volumes:
- name: cache
  emptyDir: {}
# Ephemeral storage
# Lost when Pod deleted
```

**Use Cases:**
```
✓ Web servers
✓ API servers
✓ Microservices
✓ Workers
✓ Any stateless app
```

---

### StatefulSets

**Purpose:**
```
For stateful applications
Unique Pod identity
Persistent storage
Ordered operations
Stable network identity
```

**Characteristics:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: postgres
        image: postgres
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

**Pod Names:**
```
db-0  (stable, predictable)
db-1
db-2

Names never change
Always same identity
```

**Scaling:**
```bash
kubectl scale statefulset db --replicas=5

Sequential creation:
1. Create db-3
2. Wait for db-3 ready
3. Create db-4
4. Wait for db-4 ready

Scale down:
1. Delete db-4
2. Wait for termination
3. Delete db-3
4. Wait for termination

LIFO: Last In, First Out
```

**Updates:**
```bash
kubectl set image statefulset/db postgres=postgres:15

Rolling update (reverse order):
1. Update db-2 (last pod)
2. Wait for ready
3. Update db-1
4. Wait for ready
5. Update db-0
6. Wait for ready

Can use OnDelete strategy:
- Manual control over updates
```

**Storage:**
```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    accessModes: [ "ReadWriteOnce" ]
    resources:
      requests:
        storage: 10Gi

Creates:
data-db-0  (PVC for db-0)
data-db-1  (PVC for db-1)
data-db-2  (PVC for db-2)

Persistent across Pod recreation
PVC not deleted with Pod
```

**Network Identity:**
```yaml
# Headless Service required
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None
  selector:
    app: db

# Each Pod gets stable DNS:
db-0.db.default.svc.cluster.local
db-1.db.default.svc.cluster.local
db-2.db.default.svc.cluster.local
```

**Use Cases:**
```
✓ Databases (PostgreSQL, MySQL, MongoDB)
✓ Message queues (Kafka, RabbitMQ)
✓ Distributed systems (Elasticsearch, Cassandra)
✓ Stateful workloads
```

---

### Key Differences

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| **Pod Identity** | Random | Stable (db-0, db-1) |
| **Pod Names** | Hash-based | Ordinal-based |
| **Creation Order** | Parallel | Sequential |
| **Deletion Order** | Random | Reverse sequential |
| **Storage** | Shared/Ephemeral | Persistent per Pod |
| **Network Identity** | No guarantee | Stable DNS |
| **Use Case** | Stateless | Stateful |
| **Scaling** | Fast | Slow (sequential) |
| **Updates** | Parallel | Sequential |

---

### Detailed Comparison

**Creation:**

```bash
# Deployment (parallel)
kubectl apply -f deployment.yaml

Pod creation:
web-1 ──┐
web-2 ──┼── All at once
web-3 ──┘

Time: Fast


# StatefulSet (sequential)
kubectl apply -f statefulset.yaml

Pod creation:
db-0 (wait for ready)
  ↓
db-1 (wait for ready)
  ↓
db-2 (wait for ready)

Time: Slower
```

**Deletion:**

```bash
# Deployment (parallel)
kubectl delete deployment web

Pod deletion:
web-1 ──┐
web-2 ──┼── All at once
web-3 ──┘


# StatefulSet (reverse sequential)
kubectl delete statefulset db

Pod deletion:
db-2 (wait for termination)
  ↓
db-1 (wait for termination)
  ↓
db-0 (wait for termination)
```

**Storage Behavior:**

```yaml
# Deployment
spec:
  template:
    spec:
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: shared-data

All Pods share same PVC
OR
All Pods use emptyDir (ephemeral)


# StatefulSet
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    resources:
      requests:
        storage: 10Gi

Each Pod gets own PVC:
data-db-0 → db-0
data-db-1 → db-1
data-db-2 → db-2

PVCs persist even if Pod deleted
```

**Pod Replacement:**

```bash
# Deployment
kubectl delete pod web-7d8c9f6b5c-abcd1

New Pod:
- Different name: web-7d8c9f6b5c-xyz99
- Different IP
- No persistent data
- Can be on any node


# StatefulSet
kubectl delete pod db-1

New Pod:
- Same name: db-1
- Same PVC: data-db-1
- Same DNS: db-1.db...
- Preserves identity
```

---

### Real-World Examples

**Web Application (Deployment):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 10
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: app
        image: mywebapp:1.0
        ports:
        - containerPort: 8080

# Characteristics:
- 10 identical Pods
- Can scale to 100 instantly
- Any Pod can serve any request
- No data persistence needed
- Fast updates
```

**PostgreSQL Cluster (StatefulSet):**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
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
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Gi

# Characteristics:
- postgres-0 (primary)
- postgres-1 (replica)
- postgres-2 (replica)
- Each has own storage
- Ordered initialization
- Stable network identity
```

---

### When to Use Which

**Use Deployment when:**
```
✓ Application is stateless
✓ Pods are interchangeable
✓ Fast scaling needed
✓ Parallel updates acceptable
✓ No persistent identity needed
✓ Shared storage or no storage

Examples:
- REST APIs
- Web servers
- Microservices
- Background workers
```

**Use StatefulSet when:**
```
✓ Application is stateful
✓ Need stable identity
✓ Ordered operations required
✓ Persistent storage per Pod
✓ Network identity matters

Examples:
- Databases
- Message brokers
- Distributed systems
- Clustered applications
```

---

### Migration Pattern

**From Deployment to StatefulSet:**

```
Scenario: Need to add persistence

1. Create StatefulSet with PVCs
2. Migrate data
3. Switch traffic
4. Delete Deployment

# Before (Deployment)
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  template:
    spec:
      volumes:
      - name: data
        emptyDir: {}

# After (StatefulSet)
apiVersion: apps/v1
kind: StatefulSet
spec:
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 10Gi
```

</details>

</details>

---

[← Previous: 9.2 Kubernetes Overview](02-kubernetes-overview.md) | [Next: 10.1 Best Practices →](../10-best-practices/01-dockerfile-practices.md)