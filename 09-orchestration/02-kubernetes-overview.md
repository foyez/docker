# 9.2 Kubernetes Overview

Understanding what Kubernetes provides that Docker Compose cannot.

---

## What is Kubernetes?

**Kubernetes (K8s)** is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications **across multiple servers**.

### Core Concept

> **Docker** = Run containers  
> **Docker Compose** = Run multi-container apps on ONE server  
> **Kubernetes** = Run multi-container apps across MANY servers

---

## What Docker Compose Cannot Do

### Problem 1: Single Host Limitation

**Docker Compose:**
```
┌─────────────────────────────────┐
│      Single Server              │
│                                 │
│  ┌──────┐ ┌──────┐ ┌──────┐     │
│  │ Web  │ │ API  │ │  DB  │     │
│  └──────┘ └──────┘ └──────┘     │
│                                 │
│  If this server fails =         │
│  ENTIRE APPLICATION FAILS       │
└─────────────────────────────────┘
```

**What happens:**
- Server crashes → All services down
- No redundancy
- No failover
- Cannot distribute load across servers

**Kubernetes:**
```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Server 1    │  │  Server 2    │  │  Server 3    │
│  ┌──────┐    │  │  ┌──────┐    │  │  ┌──────┐    │
│  │ Web  │    │  │  │ Web  │    │  │  │ API  │    │
│  └──────┘    │  │  └──────┘    │  │  └──────┘    │
│  ┌──────┐    │  │  ┌──────┐    │  │  ┌──────┐    │
│  │ API  │    │  │  │ DB   │    │  │  │ DB   │    │
│  └──────┘    │  │  └──────┘    │  │  └──────┘    │
└──────────────┘  └──────────────┘  └──────────────┘

If Server 1 fails → Traffic redirected to Server 2 & 3
Application stays online ✓
```

### Problem 2: No Auto-Scaling

**Docker Compose:**

```yaml
services:
  web:
    image: nginx
    deploy:
      replicas: 3  # Fixed number
```

```
Traffic pattern:
├── Normal: 100 req/s → 3 containers sufficient
├── Black Friday: 10,000 req/s → 3 containers overloaded
└── 3 AM: 10 req/s → 3 containers wasteful

You must manually scale:
docker-compose up --scale web=50    # Black Friday
docker-compose up --scale web=3     # Back to normal
```

**Kubernetes:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

```
Automatic behavior:
├── Normal load (70% CPU) → 3 pods
├── High load (90% CPU) → Scales to 20 pods automatically
├── Black Friday (95% CPU) → Scales to 100 pods
└── Low load (20% CPU) → Scales down to 3 pods

No manual intervention needed!
```

### Problem 3: No Self-Healing

**Docker Compose:**

```
Container crashes:
┌─────────────────────────────┐
│  docker-compose.yml         │
│                             │
│  services:                  │
│    api:                     │
│      image: myapi           │
│      restart: unless-stopped│
└─────────────────────────────┘

Behavior:
- Container crashes on Server A
- Restarts on Server A (same failed server)
- If Server A is down → Service stays down
- No health checks
- No replacement on healthy server
```

**Kubernetes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: api
        image: myapi
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
```

```
Automatic healing:
├── Pod crashes → New pod created automatically
├── Health check fails → Pod replaced
├── Node fails → Pods rescheduled to healthy nodes
├── Not enough CPU → Pod not scheduled until resources available
└── Maintains desired state (3 replicas) always
```

### Problem 4: No Load Balancing

**Docker Compose:**

```
External Load Balancer (nginx/HAProxy) required:

┌──────────────┐
│  nginx       │  You must configure manually
│  (external)  │
└──────┬───────┘
       │
   ┌───┴────────────────┐
   │                    │
┌──▼──┐              ┌──▼──┐
│ Web1│              │ Web2│
└─────┘              └─────┘

Problem: Manual configuration, single point of failure
```

**Kubernetes:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer  # Automatic!
```

```
Built-in load balancing:
┌──────────────────────────┐
│  Kubernetes Service      │
│  (automatic load balance)│
└────────┬─────────────────┘
         │
   ┌─────┴──────┬───────────┬──────────┐
   │            │           │          │
┌──▼──┐      ┌──▼──┐    ┌──▼──┐    ┌──▼──┐
│ Web1│      │ Web2│    │ Web3│    │ Web4│
└─────┘      └─────┘    └─────┘    └─────┘

Automatic:
- Health-based routing
- Session affinity if needed
- Distributes load evenly
- No manual configuration
```

### Problem 5: No Rolling Updates

**Docker Compose:**

```bash
# Current version running
docker-compose up -d

# Update to new version
# ALL containers stop at once!
docker-compose pull
docker-compose up -d

Downtime window:
├── t=0s: Stop all old containers
├── t=5s: Download new images
├── t=30s: Start new containers
└── t=35s: Service back online

⚠️ 35 seconds of downtime
```

**Kubernetes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
```

```
Zero-downtime rolling update:
t=0s:  10 old pods running
t=10s: Create 2 new pods (12 total)
t=20s: New pods healthy, kill 1 old pod
t=30s: Create 1 new pod
t=40s: Kill 1 old pod
...continues...
t=120s: All 10 pods on new version

✓ Zero downtime
✓ Gradual rollout
✓ Automatic rollback if new version fails
```

### Problem 6: No Secrets Management

**Docker Compose:**

```yaml
services:
  db:
    environment:
      POSTGRES_PASSWORD: "SuperSecret123"  # 😱 In plain text!
```

```
Problems:
- Secrets visible in docker-compose.yml
- Committed to Git (major security issue)
- No encryption
- No access control
- No audit trail
```

**Kubernetes:**

```bash
# Create secret (encrypted)
kubectl create secret generic db-password \
  --from-literal=password=SuperSecret123

# Secret stored encrypted in etcd
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db
spec:
  containers:
  - name: postgres
    env:
    - name: POSTGRES_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-password
          key: password
```

```
Benefits:
- Encrypted at rest
- Encrypted in transit
- RBAC (role-based access control)
- Audit logging
- Can integrate with external secret managers (Vault)
- Never in Git
```

---

## Feature Comparison

| Feature | Docker Compose | Kubernetes |
|---------|----------------|-----------|
| **Multi-host** | ❌ No | ✅ Yes |
| **Auto-scaling** | ❌ Manual only | ✅ Automatic (HPA, VPA) |
| **Self-healing** | ⚠️ Restart only | ✅ Replace, reschedule |
| **Load balancing** | ❌ External required | ✅ Built-in |
| **Rolling updates** | ❌ Downtime | ✅ Zero-downtime |
| **Rollback** | ❌ Manual | ✅ Automatic |
| **Health checks** | ⚠️ Basic | ✅ Advanced (liveness, readiness, startup) |
| **Secrets management** | ❌ Plain text | ✅ Encrypted |
| **Resource limits** | ⚠️ Per service | ✅ Per pod + namespace quotas |
| **Service discovery** | ⚠️ DNS only | ✅ DNS + env vars + API |
| **Storage** | ⚠️ Local volumes | ✅ Persistent volumes (cloud, NFS, etc.) |
| **Networking** | ⚠️ Simple | ✅ Advanced (Network policies) |
| **Monitoring** | ❌ External tools | ✅ Built-in metrics |
| **Logging** | ⚠️ Per container | ✅ Centralized |
| **RBAC** | ❌ No | ✅ Yes |

---

## Real-World Scaling Example

### Scenario: E-commerce During Black Friday

**With Docker Compose:**

```
Friday 11:59 PM - Normal traffic (1,000 users):
Server: 4 web containers, 2 API containers

Black Friday 12:00 AM - Traffic spike (50,000 users):
Problems:
1. Server CPU at 100%
2. Containers crashing
3. Manual intervention needed
4. By the time you scale, customers left
5. Revenue lost

Manual response:
1. SSH to server (1 min)
2. docker-compose up --scale web=20 (2 min)
3. Server out of memory (3 min)
4. Provision new server (30 min)
5. Manually configure new server (15 min)
Total downtime: ~50 minutes
Lost revenue: $500,000
```

**With Kubernetes:**

```
Friday 11:59 PM - Normal traffic (1,000 users):
Cluster: 10 pods distributed across 3 nodes

Black Friday 12:00 AM - Traffic spike (50,000 users):
Automatic response:
t=0s:   CPU spike detected (80% → 95%)
t=30s:  HPA triggers scale up
t=60s:  50 new pods created
t=90s:  New pods healthy and receiving traffic
t=120s: Traffic distributed, CPU normal

If 3 nodes not enough:
t=150s: Cluster Autoscaler provisions new nodes (AWS/GCP)
t=300s: New nodes ready
t=330s: Pods scheduled to new nodes

Total: 2-5 minutes fully automatic
Zero downtime
Zero revenue lost
```

---

## When to Use Each

### Use Docker Compose When:

✅ **Local Development**
```
Every developer needs full stack locally:
- Quick start/stop
- Easy debugging
- Fast iteration
```

✅ **Small Production (Single Server)**
```
Applications that fit on one server:
- Personal blog
- Internal tools
- Proof of concept
- < 1000 users
```

✅ **Simple Deployments**
```
No complex requirements:
- No auto-scaling needed
- Downtime acceptable
- Single region
- Small team
```

### Use Kubernetes When:

✅ **Production at Scale**
```
Requirements:
- Multiple servers needed
- High availability critical
- Auto-scaling required
- > 10,000 users
```

✅ **Microservices Architecture**
```
Complex applications:
- 20+ services
- Independent scaling per service
- Service mesh (Istio, Linkerd)
- Advanced traffic management
```

✅ **Global Distribution**
```
Multi-region deployment:
- Users worldwide
- Data residency requirements
- Disaster recovery
- Multi-cloud strategy
```

✅ **Enterprise Requirements**
```
Compliance and governance:
- RBAC and security policies
- Audit trails
- Secrets management
- Resource quotas
```

---

## Migration Path

### Typical Company Journey

```
Stage 1: Docker Compose (Month 0-6)
├── 1-5 developers
├── MVP development
├── Single server deployment
└── Docker Compose perfect fit

Stage 2: Docker Compose (Month 6-12)
├── 5-15 developers
├── Growing user base
├── 2-3 servers (manual)
├── Starting to feel pain
└── Still manageable with Compose

Stage 3: Kubernetes (Month 12+)
├── 15+ developers
├── Significant traffic
├── Need auto-scaling
├── Multiple environments
├── Cannot afford downtime
└── Migrate to Kubernetes

Learning curve trade-off:
- Compose: Learn in 1 day, use immediately
- Kubernetes: Learn in weeks, massive benefits long-term
```

---

## Key Kubernetes Concepts (Brief)

### Pods
- Smallest deployable unit
- One or more containers
- Shared network and storage

### Deployments
- Manages desired state
- Handles rolling updates
- Self-healing

### Services
- Stable networking
- Load balancing
- Service discovery

### ConfigMaps & Secrets
- Configuration management
- Secrets encryption

### Ingress
- HTTP/HTTPS routing
- SSL termination
- Path-based routing

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Docker Compose runs containers on __________ server, while Kubernetes runs across __________ servers.
2. Kubernetes provides __________ scaling based on CPU/memory metrics.
3. When a container fails in Kubernetes, the __________ automatically creates a replacement.
4. Docker Compose requires an __________ load balancer, while Kubernetes has __________ load balancing.
5. Kubernetes stores secrets in __________ form, while Docker Compose uses __________ text.

### True/False

1. ⬜ Docker Compose can automatically distribute containers across multiple servers
2. ⬜ Kubernetes can perform zero-downtime rolling updates
3. ⬜ Docker Compose has built-in auto-scaling capabilities
4. ⬜ If a server fails in Kubernetes, pods are automatically rescheduled to healthy servers
5. ⬜ Kubernetes is simpler to learn and use than Docker Compose
6. ⬜ Docker Compose is suitable for production deployments at scale
7. ⬜ Kubernetes can automatically scale clusters by adding new nodes

### Multiple Choice

1. What is the main limitation of Docker Compose?
   - A) Cannot run multiple containers
   - B) Limited to single host
   - C) No networking support
   - D) Cannot use volumes

2. Which Kubernetes component provides zero-downtime deployments?
   - A) Pod
   - B) Service
   - C) Deployment with RollingUpdate
   - D) ConfigMap

3. What happens when a server fails in a Docker Compose setup?
   - A) Services automatically migrate to another server
   - B) Services restart on the same server
   - C) All services fail
   - D) Load balancer redirects traffic

4. How does Kubernetes handle auto-scaling?
   - A) Manual scaling only
   - B) Horizontal Pod Autoscaler (HPA)
   - C) Requires external tools
   - D) Not supported

5. When should you choose Docker Compose over Kubernetes?
   - A) Production with 1 million users
   - B) Multi-region deployment
   - C) Local development environment
   - D) Need for auto-scaling

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. one (or single), multiple (or many)
2. automatic (or horizontal)
3. scheduler (or control plane, or deployment controller)
4. external, built-in (or native, or integrated)
5. encrypted, plain

**True/False:**
1. ❌ False - Docker Compose is single-host only
2. ✅ True - RollingUpdate strategy enables zero-downtime
3. ❌ False - Docker Compose requires manual scaling
4. ✅ True - Self-healing and automatic rescheduling
5. ❌ False - Kubernetes has a steeper learning curve
6. ❌ False - Limited to small-scale, single-server deployments
7. ✅ True - Cluster Autoscaler can add nodes automatically

**Multiple Choice:**
1. **B** - Limited to single host
2. **C** - Deployment with RollingUpdate strategy
3. **C** - All services fail (single point of failure)
4. **B** - Horizontal Pod Autoscaler (HPA)
5. **C** - Local development environment

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Why can't Docker Compose handle a multi-server deployment?

<details>
<summary><strong>View Answer</strong></summary>

**Fundamental Design Limitation:**

Docker Compose is designed to run on a **single Docker host** only. It lacks the distributed system capabilities needed for multi-server orchestration.

**Technical Reasons:**

**1. No Cluster Awareness**
```
Docker Compose view:
- Knows about: localhost Docker daemon
- Cannot see: Other servers
- Cannot coordinate: Across machines

docker-compose.yml:
services:
  web:
    replicas: 10

Result: All 10 containers on ONE server
Cannot distribute across servers
```

**2. No Scheduler**
```
Kubernetes has scheduler:
- Analyzes resource availability across cluster
- Decides optimal node for each pod
- Considers CPU, memory, affinity rules

Docker Compose:
- No scheduler component
- Always runs on local Docker daemon
- Cannot make placement decisions
```

**3. No Service Discovery Across Hosts**
```
Single host (Compose):
web → api (via hostname: api) ✓

Multi-host scenario:
Server 1: web
Server 2: api
web → api ❌ (cannot resolve hostname across servers)

Kubernetes:
- Cluster-wide DNS
- Service discovery across all nodes
- Automatic endpoint updates
```

**4. No State Management**
```
Kubernetes Control Plane:
- etcd: Distributed key-value store
- Stores cluster state
- Coordinates across nodes

Docker Compose:
- State stored locally only
- No distributed consensus
- Cannot coordinate multiple servers
```

**Example - What Happens If You Try:**

```bash
# Server 1
docker-compose up -d

# Server 2
docker-compose up -d

Result:
- Two SEPARATE deployments
- No communication between them
- No load balancing
- No failover
- Just two independent stacks
```

**What You'd Need:**

```
Manual multi-server setup:
1. External load balancer (nginx, HAProxy)
2. Shared storage (NFS, GlusterFS)
3. Service discovery (Consul, etcd)
4. Health monitoring (external)
5. Manual failover procedures

This is exactly what Kubernetes provides automatically!
```

</details>

### Question 2: Explain how Kubernetes achieves zero-downtime deployments

<details>
<summary><strong>View Answer</strong></summary>

**Rolling Update Strategy:**

Kubernetes gradually replaces old pods with new ones while maintaining service availability.

**Step-by-Step Process:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Max 2 extra pods during update
      maxUnavailable: 1  # Max 1 pod can be unavailable
```

**Timeline:**

```
Initial state: 6 pods on v1.0
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
│ v1 │ │ v1 │ │ v1 │ │ v1 │ │ v1 │ │ v1 │
└────┘ └────┘ └────┘ └────┘ └────┘ └────┘
Total capacity: 6 pods

t=10s: Create 2 new pods (maxSurge: 2)
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
│ v1 │ │ v1 │ │ v1 │ │ v1 │ │ v1 │ │ v1 │ │ v2*│ │ v2*│
└────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘
*starting up, not ready yet

t=30s: New pods ready, terminate 1 old pod
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
│ v1 │ │ v1 │ │ v1 │ │ v1 │ │ v1 │ │ v2 │ │ v2 │
└────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘

t=40s: Create 1 new pod
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
│ v1 │ │ v1 │ │ v1 │ │ v1 │ │ v1 │ │ v2 │ │ v2 │ │ v2*│
└────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘

...continues until...

t=180s: All pods on v2.0
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
│ v2 │ │ v2 │ │ v2 │ │ v2 │ │ v2 │ │ v2 │
└────┘ └────┘ └────┘ └────┘ └────┘ └────┘

Throughout the entire process:
- Minimum 5 pods available (6 - maxUnavailable: 1)
- Maximum 8 pods running (6 + maxSurge: 2)
- Service always available
```

**Key Mechanisms:**

**1. Readiness Probes**
```yaml
spec:
  containers:
  - name: web
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 3

Behavior:
- New pod starts
- Not added to Service endpoints until ready
- Readiness probe succeeds
- Only then receives traffic
- Old pod terminated only after new pod ready
```

**2. Service Selector**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web  # Matches BOTH v1 and v2 pods
```

```
During rollout, Service routes to:
├── v1 pods (still running)
└── v2 pods (newly ready)

Traffic gradually shifts as old pods terminate
```

**3. Pod Disruption Budget**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 5  # At least 5 pods must always be available
  selector:
    matchLabels:
      app: web

Prevents:
- Too many pods terminating at once
- Service degradation
```

**Automatic Rollback:**

```
If new version fails health checks:
t=30s: v2 pod created
t=60s: Readiness probe failing
t=90s: Kubernetes detects failure
t=100s: Stops rollout automatically
t=110s: Keeps v1 pods running

Result: Failed update contained, service stays on v1
```

**Comparison:**

```
Docker Compose:
docker-compose down  # Stop all
docker-compose up -d # Start all new
└── 30-60s downtime window

Kubernetes:
kubectl set image deployment/web web=v2
└── Zero downtime, gradual rollout, automatic rollback
```

</details>

### Question 3: A company has 20 microservices running on Docker Compose. They're experiencing frequent downtime and manual scaling issues. How would you propose migration to Kubernetes?

<details>
<summary><strong>View Answer</strong></summary>

**Migration Strategy:**

**Phase 1: Assessment (Week 1-2)**

Analyze current setup:
```
Current state with Docker Compose:
- 20 services on 3 servers
- Manual scaling every morning/evening
- 2-3 outages per week
- Deployment takes 30 minutes
- Downtime during updates

Kubernetes benefits:
- Auto-scaling
- Self-healing
- Zero-downtime deployments
- Better resource utilization
```

**Phase 2: Preparation (Week 3-4)**

```
1. Dockerize remaining services
   - Ensure all services have Dockerfiles
   - Follow best practices (multi-stage builds)

2. Set up Kubernetes cluster
   - Start with managed service (EKS, GKE, AKS)
   - 3-node cluster initially
   - Configure networking

3. Team training
   - Kubernetes fundamentals
   - kubectl commands
   - Troubleshooting basics
```

**Phase 3: Pilot Migration (Week 5-6)**

```
Choose 2-3 low-risk services:
- Minimal dependencies
- Low traffic
- Easy rollback

Example: Logging service, Internal dashboard

docker-compose.yml:
services:
  logging:
    image: logging-service:v1
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db

Convert to Kubernetes:
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: logging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: logging
  template:
    metadata:
      labels:
        app: logging
    spec:
      containers:
      - name: logging
        image: logging-service:v1
        ports:
        - containerPort: 5000
        env:
        - name: DB_HOST
          value: db
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: logging
spec:
  selector:
    app: logging
  ports:
  - port: 5000
    targetPort: 5000
```

**Phase 4: Incremental Migration (Week 7-12)**

```
Migrate in waves:
Week 7-8: Stateless services (API gateways, web servers)
Week 9-10: Business logic services
Week 11-12: Databases and stateful services

Run hybrid setup:
┌─────────────────────────────┐
│  Kubernetes Cluster          │
│  ┌────┐ ┌────┐ ┌────┐       │
│  │API │ │Web │ │Auth│       │
│  └────┘ └────┘ └────┘       │
└─────────────┬───────────────┘
              │
              │ Can communicate with
              │
┌─────────────▼───────────────┐
│  Docker Compose (temporary)  │
│  ┌────┐ ┌────┐              │
│  │DB  │ │Cache│             │
│  └────┘ └────┘              │
└─────────────────────────────┘
```

**Phase 5: Add Kubernetes Features (Week 13-16)**

```yaml
# Auto-scaling
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
---
# Health checks
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
---
# Secrets management
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: c3VwZXJzZWNyZXQ=  # base64 encoded
```

**Phase 6: Monitoring & Optimization (Week 17-20)**

```
Set up monitoring:
- Prometheus for metrics
- Grafana for dashboards
- ELK stack for logging

Configure alerts:
- High CPU usage
- Pod restarts
- Failed deployments
```

**Migration Checklist:**

```
✓ Pre-migration:
  ✓ Document current architecture
  ✓ Identify dependencies
  ✓ Set up Kubernetes cluster
  ✓ Team training completed

✓ During migration:
  ✓ Convert docker-compose to K8s manifests
  ✓ Test in staging
  ✓ Migrate incrementally
  ✓ Monitor performance

✓ Post-migration:
  ✓ Implement auto-scaling
  ✓ Set up monitoring
  ✓ Configure CI/CD
  ✓ Document new processes
  ✓ Decommission old infrastructure
```

**Results After Migration:**

```
Before (Docker Compose):
- Downtime: 2-3 outages/week
- Deployment: 30 minutes with downtime
- Scaling: Manual, takes 15 minutes
- Recovery: Manual, 20-30 minutes

After (Kubernetes):
- Downtime: 0 outages (self-healing)
- Deployment: 5 minutes, zero downtime
- Scaling: Automatic, seconds
- Recovery: Automatic, 30 seconds
```

</details>

</details>

---

**Summary:** Kubernetes solves critical production problems that Docker Compose cannot: multi-server orchestration, automatic scaling, self-healing, zero-downtime deployments, and advanced networking. Use Compose for development and simple deployments; use Kubernetes for production at scale.

---

[← Previous: 9.1 Orchestration Introduction](01-orchestration-intro.md) | [Next: 9.3 Kubernetes Architecture →](../09-orchestration/03-kubernetes-architecture.md)