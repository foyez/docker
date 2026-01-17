# 9.1 From Compose to Orchestration

Understanding the transition from Docker Compose to production orchestration platforms.

---

## Why Move Beyond Compose?

### Docker Compose Limitations

**Single Host:**
```
Docker Compose:
- Runs on single Docker host
- No distribution across servers
- Limited by single machine resources
- No automatic failover
```

**No Self-Healing:**
```
If container crashes:
- Manual restart required (unless restart policy)
- No automatic rescheduling
- No health-based replacement
```

**Limited Scaling:**
```
Scaling:
- Manual scaling only
- No auto-scaling based on load
- No resource-based scaling
```

**No Service Discovery:**
```
Networking:
- Basic DNS resolution
- No advanced load balancing
- No service mesh
```

---

## When to Use Orchestration

**Use Docker Compose for:**
```
✓ Development environments
✓ Small applications (single server)
✓ Testing and CI/CD
✓ Simple deployments
✓ Learning Docker
```

**Use Orchestration for:**
```
✓ Production workloads
✓ Multi-server deployments
✓ High availability requirements
✓ Auto-scaling needed
✓ Complex microservices
✓ Zero-downtime deployments
✓ Enterprise applications
```

---

## Orchestration Platforms

### Kubernetes (K8s)

**Overview:**
```
Most popular orchestration platform
Originally by Google, now CNCF
Industry standard for production
Steep learning curve
Powerful and feature-rich
```

**Key Features:**
```
✓ Multi-host deployment
✓ Automatic scaling
✓ Self-healing
✓ Rolling updates
✓ Service discovery
✓ Load balancing
✓ Secret management
✓ Storage orchestration
```

**When to Choose:**
```
✓ Large-scale applications
✓ Multiple teams
✓ Cloud-native architecture
✓ Need for ecosystem (Helm, Istio)
✓ Industry standard compliance
```

---

### Docker Swarm

**Overview:**
```
Native Docker orchestration
Built into Docker Engine
Easier learning curve
Less complex than Kubernetes
Smaller ecosystem
```

**Key Features:**
```
✓ Native Docker integration
✓ Simple setup
✓ Declarative service model
✓ Scaling and load balancing
✓ Rolling updates
✓ Multi-host networking
✓ Built-in security
```

**When to Choose:**
```
✓ Smaller deployments
✓ Team knows Docker well
✓ Simpler requirements
✓ Quick setup needed
✓ Less operational overhead
```

---

### Other Options

**Amazon ECS (Elastic Container Service):**
```
AWS-native orchestration
Deep AWS integration
No cluster management needed
Pay for resources only
```

**Amazon EKS (Elastic Kubernetes Service):**
```
Managed Kubernetes on AWS
Standard Kubernetes API
AWS integration
Automated upgrades
```

**Google GKE (Google Kubernetes Engine):**
```
Managed Kubernetes on GCP
Auto-scaling
Auto-upgrades
Integrated monitoring
```

**Azure AKS (Azure Kubernetes Service):**
```
Managed Kubernetes on Azure
Azure integration
Serverless Kubernetes
Integrated DevOps
```

---

## Comparison Table

| Feature | Docker Compose | Docker Swarm | Kubernetes |
|---------|---------------|--------------|------------|
| **Complexity** | Low | Medium | High |
| **Learning Curve** | Easy | Moderate | Steep |
| **Setup Time** | Minutes | Minutes | Hours/Days |
| **Multi-Host** | ❌ No | ✅ Yes | ✅ Yes |
| **Auto-Scaling** | ❌ No | ✅ Limited | ✅ Advanced |
| **Self-Healing** | ⚠️ Limited | ✅ Yes | ✅ Yes |
| **Load Balancing** | ⚠️ Basic | ✅ Built-in | ✅ Advanced |
| **Rolling Updates** | ⚠️ Manual | ✅ Yes | ✅ Yes |
| **Ecosystem** | Small | Small | Large |
| **Production Ready** | ⚠️ Small scale | ✅ Yes | ✅ Yes |
| **Community** | Large | Medium | Very Large |

---

## Migration Path

### Stage 1: Development (Compose)

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: ./web
    ports:
      - "80:80"
    depends_on:
      - api

  api:
    build: ./api
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
    depends_on:
      - db

  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

---

### Stage 2: Production (Docker Swarm)

**Initialize Swarm:**
```bash
# On manager node
docker swarm init

# On worker nodes
docker swarm join --token <token> <manager-ip>:2377
```

**Deploy Stack:**
```yaml
# docker-stack.yml
version: '3.8'

services:
  web:
    image: myregistry.com/web:latest
    ports:
      - "80:80"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    depends_on:
      - api

  api:
    image: myregistry.com/api:latest
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    secrets:
      - db_password

  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data
    deploy:
      placement:
        constraints:
          - node.role == manager
    secrets:
      - db_password

volumes:
  db-data:

secrets:
  db_password:
    external: true
```

**Deploy:**
```bash
docker stack deploy -c docker-stack.yml myapp
```

---

### Stage 3: Production (Kubernetes)

**Deployment:**
```yaml
# web-deployment.yaml
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
      - name: web
        image: myregistry.com/web:latest
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
```

**Service:**
```yaml
# web-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

**Deploy:**
```bash
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
```

---

## Docker Compose to Swarm

### Conversion

**Compose File:**
```yaml
version: '3.8'

services:
  api:
    image: myapi
    ports:
      - "3000:3000"
```

**Swarm Stack File:**
```yaml
version: '3.8'

services:
  api:
    image: myapi
    ports:
      - "3000:3000"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

**Key Differences:**
```
1. Add deploy section
2. Define replicas
3. Set update_config
4. Add resource limits
5. Configure restart policy
6. Remove build (use images)
```

---

### Kompose (Compose to Kubernetes)

**Install Kompose:**
```bash
# macOS
brew install kompose

# Linux
curl -L https://github.com/kubernetes/kompose/releases/download/v1.31.2/kompose-linux-amd64 -o kompose
chmod +x kompose
sudo mv kompose /usr/local/bin/
```

**Convert:**
```bash
# Convert Compose to Kubernetes manifests
kompose convert -f docker-compose.yml

# Output:
# web-deployment.yaml
# web-service.yaml
# api-deployment.yaml
# api-service.yaml
# db-deployment.yaml
# db-persistentvolumeclaim.yaml
```

**Generated Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - image: myapi
        name: api
        ports:
        - containerPort: 3000
```

---

## Orchestration Concepts

### Pods (Kubernetes)

**Definition:**
```
Pod = Smallest deployable unit
Contains one or more containers
Shared network namespace
Shared storage volumes
```

**Example:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp
  - name: sidecar
    image: logging-agent
```

---

### Services (Swarm/Kubernetes)

**Swarm Service:**
```bash
# Create service
docker service create \
    --name web \
    --replicas 3 \
    --publish 80:80 \
    nginx

# Scale
docker service scale web=5

# Update
docker service update --image nginx:1.25 web
```

**Kubernetes Service:**
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
    targetPort: 80
  type: LoadBalancer
```

---

### Replicas and Scaling

**Manual Scaling:**
```bash
# Swarm
docker service scale web=5

# Kubernetes
kubectl scale deployment web --replicas=5
```

**Auto-Scaling (Kubernetes):**
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
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

### Health Checks

**Swarm:**
```yaml
version: '3.8'

services:
  api:
    image: myapi
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

**Kubernetes:**
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: api
    image: myapi
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        Port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
```

---

### Rolling Updates

**Swarm:**
```bash
# Update service
docker service update \
    --image myapi:v2 \
    --update-parallelism 2 \
    --update-delay 10s \
    api
```

**Kubernetes:**
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

```bash
# Update image
kubectl set image deployment/api api=myapi:v2

# Rollback
kubectl rollout undo deployment/api
```

---

### Secrets Management

**Swarm:**
```bash
# Create secret
echo "mysecret" | docker secret create db_password -

# Use in service
docker service create \
    --name api \
    --secret db_password \
    myapi
```

**Kubernetes:**
```bash
# Create secret
kubectl create secret generic db-password \
    --from-literal=password=mysecret

# Use in deployment
```

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: api
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-password
          key: password
```

---

### Storage

**Swarm Volumes:**
```yaml
version: '3.8'

services:
  db:
    image: postgres
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
    driver: local
```

**Kubernetes Persistent Volumes:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: db
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: db-data
```

---

## Migration Checklist

### Pre-Migration

```
☐ Identify orchestration platform
☐ Understand resource requirements
☐ Plan network architecture
☐ Design storage strategy
☐ Document current setup
☐ Test migration in staging
☐ Prepare rollback plan
```

### During Migration

```
☐ Convert Compose files
☐ Create container registry
☐ Build and push images
☐ Configure secrets
☐ Set up persistent storage
☐ Deploy to cluster
☐ Configure ingress/load balancer
☐ Test application
☐ Monitor metrics
```

### Post-Migration

```
☐ Verify all services running
☐ Test failover
☐ Validate backups
☐ Configure monitoring
☐ Set up alerts
☐ Document new architecture
☐ Train team
☐ Decommission old infrastructure
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Docker Compose is limited to __________ host deployments.
2. __________ is the most popular container orchestration platform.
3. __________ is Docker's native orchestration solution.
4. A __________ is the smallest deployable unit in Kubernetes.
5. __________ converts Compose files to Kubernetes manifests.
6. Container orchestration provides __________ capabilities for automatic recovery.

### True/False

1. ⬜ Docker Compose can distribute containers across multiple hosts
2. ⬜ Kubernetes has a steeper learning curve than Docker Swarm
3. ⬜ Docker Swarm requires separate installation from Docker
4. ⬜ Orchestration platforms provide automatic scaling
5. ⬜ Compose files can be used directly with Swarm
6. ⬜ Kubernetes pods can contain multiple containers
7. ⬜ Docker Compose is suitable for large production deployments

### Multiple Choice

1. Which is NOT an orchestration platform?
   - A) Kubernetes
   - B) Docker Swarm
   - C) Docker Compose
   - D) Amazon ECS

2. What does Kompose do?
   - A) Manages Compose files
   - B) Converts Compose to Kubernetes
   - C) Orchestrates containers
   - D) Builds images

3. What's the smallest unit in Kubernetes?
   - A) Container
   - B) Service
   - C) Pod
   - D) Node

4. Which has the largest ecosystem?
   - A) Docker Compose
   - B) Docker Swarm
   - C) Kubernetes
   - D) All equal

5. Docker Swarm is built into?
   - A) Kubernetes
   - B) Docker Engine
   - C) Docker Compose
   - D) Separate tool

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. single
2. Kubernetes
3. Docker Swarm
4. Pod
5. Kompose
6. self-healing

**True/False:**
1. ❌ False - Compose is single-host only
2. ✅ True - Kubernetes is more complex
3. ❌ False - Swarm is built into Docker Engine
4. ✅ True - Orchestration enables auto-scaling
5. ✅ True - Swarm uses Compose file format
6. ✅ True - Pods can have multiple containers
7. ❌ False - Compose is for development/small deployments

**Multiple Choice:**
1. **C** - Docker Compose (not orchestration)
2. **B** - Converts Compose to Kubernetes
3. **C** - Pod
4. **C** - Kubernetes
5. **B** - Docker Engine

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: When would you choose Docker Swarm over Kubernetes?

<details>
<summary><strong>View Answer</strong></summary>

**Decision Criteria:**

---

### Choose Docker Swarm When:

**1. Simpler Requirements**
```
Application Needs:
✓ Basic orchestration (scaling, load balancing)
✓ No complex service mesh needed
✓ Standard rolling updates sufficient
✓ Simple networking requirements
```

**2. Team Familiarity**
```
Team Skills:
✓ Team knows Docker well
✓ Limited Kubernetes experience
✓ Small operations team
✓ Quick learning curve needed
```

**3. Faster Setup**
```
Time Constraints:
✓ Need production fast
✓ Limited setup time
✓ Minimal infrastructure
✓ Quick deployment cycles
```

**4. Smaller Scale**
```
Application Scale:
✓ < 100 nodes
✓ Fewer services
✓ Lower traffic
✓ Simpler architecture
```

**5. Docker-Native Integration**
```
Existing Setup:
✓ Already using Docker
✓ Docker Compose in development
✓ Docker registry in place
✓ Docker tooling familiar
```

---

### Choose Kubernetes When:

**1. Complex Requirements**
```
Application Needs:
✓ Advanced scheduling
✓ Complex networking (service mesh)
✓ Sophisticated auto-scaling
✓ Custom resource definitions
✓ Advanced deployment strategies
```

**2. Large Scale**
```
Scale Requirements:
✓ > 100 nodes
✓ Many microservices
✓ High traffic
✓ Multi-region deployment
✓ Complex service dependencies
```

**3. Ecosystem Needs**
```
Tooling:
✓ Helm for package management
✓ Istio for service mesh
✓ Prometheus for monitoring
✓ GitOps (ArgoCD, Flux)
✓ Rich third-party integrations
```

**4. Cloud-Native Architecture**
```
Architecture:
✓ Cloud-native best practices
✓ Microservices architecture
✓ Multi-cloud strategy
✓ Hybrid cloud deployment
```

**5. Industry Standard**
```
Business Needs:
✓ Vendor neutrality
✓ Portability across clouds
✓ Large community support
✓ Enterprise compliance
✓ Long-term investment
```

---

### Detailed Comparison

**Setup Complexity:**

```bash
# Docker Swarm (Minutes)
docker swarm init
docker stack deploy -c stack.yml myapp

# Kubernetes (Hours/Days)
# - Choose distribution (EKS, GKE, AKS, self-managed)
# - Setup cluster
# - Configure kubectl
# - Setup ingress controller
# - Setup storage classes
# - Configure RBAC
# - Deploy application
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

**Learning Curve:**
```
Docker Swarm:
- Familiar Docker concepts
- Same Compose file format
- Simple commands
- Less new terminology

Kubernetes:
- New concepts (Pods, ReplicaSets, etc.)
- Different YAML format
- Complex architecture
- Steep learning curve
- Extensive documentation needed
```

**Operational Overhead:**
```
Docker Swarm:
✓ Minimal maintenance
✓ Built into Docker
✓ Simple upgrades
✓ Less monitoring needed

Kubernetes:
✗ Regular cluster upgrades
✗ Component management
✗ Complex troubleshooting
✗ Extensive monitoring
```

---

### Real-World Scenarios

**Scenario 1: Small SaaS Startup**
```
Context:
- Team of 5 developers
- Monolithic application
- 3-5 services total
- Single region deployment
- Budget constraints

Recommendation: Docker Swarm

Reasons:
✓ Fast time to market
✓ Simple to learn
✓ Low operational overhead
✓ Sufficient for current scale
✓ Can migrate to K8s later if needed
```

**Scenario 2: Enterprise Microservices**
```
Context:
- 50+ microservices
- Multiple teams
- Multi-region deployment
- Complex service mesh
- High availability requirements

Recommendation: Kubernetes

Reasons:
✓ Handles complexity
✓ Rich ecosystem
✓ Advanced features needed
✓ Industry standard
✓ Multi-cloud portability
```

**Scenario 3: Mid-Size E-Commerce**
```
Context:
- 10-15 services
- Growing team
- Seasonal traffic spikes
- Need auto-scaling
- Single cloud provider

Recommendation: Managed Kubernetes (EKS/GKE/AKS)

Reasons:
✓ Auto-scaling capabilities
✓ Managed service reduces overhead
✓ Room to grow
✓ Cloud provider integration
✓ Standard tooling
```

---

### Migration Path

**Start with Swarm, Move to K8s:**
```
Phase 1: Development (Compose)
- Use Docker Compose for development
- Local testing

Phase 2: Initial Production (Swarm)
- Deploy to Swarm for quick launch
- Learn orchestration basics
- Validate production setup

Phase 3: Scale Up (Kubernetes)
- As complexity grows, migrate to K8s
- Leverage Swarm experience
- Gradual migration
```

---

### Cost Considerations

**Docker Swarm:**
```
✓ No licensing costs
✓ Lower infrastructure (simpler)
✓ Less operational staff needed
✓ Faster development

Total Cost: Lower
```

**Kubernetes:**
```
✓ No licensing costs (open source)
✗ Managed K8s costs (EKS/GKE/AKS)
✗ More infrastructure complexity
✗ Specialized staff needed
✗ Training costs

Total Cost: Higher
```

---

### Decision Matrix

| Factor | Swarm | Kubernetes |
|--------|-------|------------|
| **Setup Time** | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Learning Curve** | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Features** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Ecosystem** | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Scalability** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Community** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Ops Overhead** | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Multi-Cloud** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

---

### Final Recommendation

```
Use Docker Swarm if:
- Small/medium applications
- Quick deployment needed
- Team prefers simplicity
- Budget constraints
- Docker-focused workflow

Use Kubernetes if:
- Large-scale applications
- Complex requirements
- Growing ecosystem needs
- Multi-cloud strategy
- Long-term enterprise investment
```

</details>

### Question 2: How do you migrate from Compose to Kubernetes?

<details>
<summary><strong>View Answer</strong></summary>

**Complete Migration Guide:**

---

### Step 1: Assessment

**Inventory Services:**
```yaml
# docker-compose.yml
services:
  frontend:
    build: ./frontend
    ports:
      - "80:80"
  
  api:
    build: ./api
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
  
  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data
```

**Document:**
```
Services: 3 (frontend, api, db)
Dependencies: frontend → api → db
Volumes: 1 (db-data)
Networks: 1 (default)
Secrets: DATABASE_URL
Build steps: frontend, api
```

---

### Step 2: Build and Push Images

**Create Dockerfile for each service:**
```dockerfile
# api/Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
CMD ["node", "server.js"]
```

**Build and push:**
```bash
# Set registry
REGISTRY="myregistry.com"
VERSION="1.0.0"

# Build
docker build -t $REGISTRY/frontend:$VERSION ./frontend
docker build -t $REGISTRY/api:$VERSION ./api

# Push
docker push $REGISTRY/frontend:$VERSION
docker push $REGISTRY/api:$VERSION
```

---

### Step 3: Convert with Kompose

**Install Kompose:**
```bash
brew install kompose  # macOS
# or
curl -L https://github.com/kubernetes/kompose/releases/download/v1.31.2/kompose-linux-amd64 -o kompose
chmod +x kompose
sudo mv kompose /usr/local/bin/
```

**Convert:**
```bash
kompose convert -f docker-compose.yml

# Generated files:
# frontend-deployment.yaml
# frontend-service.yaml
# api-deployment.yaml
# api-service.yaml
# db-deployment.yaml
# db-persistentvolumeclaim.yaml
```

---

### Step 4: Refine Kubernetes Manifests

**Original Kompose Output:**
```yaml
# api-deployment.yaml (kompose generated)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - image: myapi
        name: api
        env:
        - name: DATABASE_URL
          value: postgresql://db:5432/myapp
```

**Refined for Production:**
```yaml
# api-deployment.yaml (production-ready)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: api
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api
        version: v1
    spec:
      containers:
      - name: api
        image: myregistry.com/api:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
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
```

---

### Step 5: Create Secrets

**From Compose:**
```yaml
environment:
  DATABASE_URL: postgresql://db:5432/myapp
```

**To Kubernetes Secret:**
```bash
# Create secret
kubectl create secret generic db-credentials \
  --from-literal=url='postgresql://db:5432/myapp' \
  --from-literal=password='mysecret'

# Or from file
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  url: postgresql://db:5432/myapp
  password: mysecret
EOF
```

---

### Step 6: Setup Persistent Storage

**From Compose Volume:**
```yaml
volumes:
  db-data:
```

**To Kubernetes PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd
```

**Use in Deployment:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db
  replicas: 1
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
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        ports:
        - containerPort: 5432
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

---

### Step 7: Configure Services

**Internal Service (ClusterIP):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - port: 3000
    targetPort: 3000
    name: http
```

**External Service (LoadBalancer):**
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
    targetPort: 80
    name: http
```

---

### Step 8: Setup Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 3000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

---

### Step 9: Deploy to Kubernetes

**Deploy in order:**
```bash
# 1. Secrets
kubectl apply -f secrets/

# 2. PersistentVolumeClaims
kubectl apply -f storage/

# 3. Database (StatefulSet)
kubectl apply -f db-statefulset.yaml
kubectl apply -f db-service.yaml

# Wait for db to be ready
kubectl wait --for=condition=ready pod -l app=db --timeout=300s

# 4. API
kubectl apply -f api-deployment.yaml
kubectl apply -f api-service.yaml

# Wait for API
kubectl wait --for=condition=ready pod -l app=api --timeout=300s

# 5. Frontend
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml

# 6. Ingress
kubectl apply -f ingress.yaml
```

---

### Step 10: Verify Deployment

```bash
# Check pods
kubectl get pods

# Check services
kubectl get svc

# Check ingress
kubectl get ingress

# Check logs
kubectl logs -l app=api

# Test endpoint
curl http://myapp.example.com/api/health
```

---

### Step 11: Configure Auto-Scaling

```yaml
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
  maxReplicas: 10
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
```

---

### Step 12: Setup Monitoring

```yaml
# prometheus-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api
spec:
  selector:
    matchLabels:
      app: api
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

---

### Migration Checklist

```
Pre-Migration:
☐ Document Compose setup
☐ Build production Dockerfiles
☐ Setup container registry
☐ Choose Kubernetes distribution
☐ Setup K8s cluster
☐ Install kubectl

Conversion:
☐ Use Kompose for initial conversion
☐ Refine generated manifests
☐ Add health checks
☐ Configure resource limits
☐ Setup secrets
☐ Configure persistent storage

Deployment:
☐ Create namespace
☐ Deploy secrets
☐ Deploy storage
☐ Deploy database
☐ Deploy backend services
☐ Deploy frontend
☐ Configure ingress

Post-Deployment:
☐ Verify all pods running
☐ Test application
☐ Setup monitoring
☐ Configure auto-scaling
☐ Setup backups
☐ Document new architecture
☐ Train team
```

---

### Common Pitfalls

```
1. Stateful Services
   Problem: Database data loss
   Solution: Use StatefulSets, not Deployments

2. Secrets in Code
   Problem: Hardcoded secrets
   Solution: Use Kubernetes Secrets

3. No Resource Limits
   Problem: Pod evictions
   Solution: Set requests and limits

4. Missing Health Checks
   Problem: Traffic to unhealthy pods
   Solution: Add liveness/readiness probes

5. Wrong Service Type
   Problem: Services not accessible
   Solution: ClusterIP (internal), LoadBalancer (external)
```

</details>

### Question 3: What are the key differences in networking between Compose and orchestration?

<details>
<summary><strong>View Answer</strong></summary>

**Networking Comparison:**

---

### Docker Compose Networking

**Single Host Network:**
```yaml
version: '3.8'

services:
  web:
    image: nginx
    networks:
      - frontend

  api:
    image: myapi
    networks:
      - frontend
      - backend

  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
  backend:
```

**Characteristics:**
```
✓ Simple DNS resolution
✓ Service name = hostname
✓ Single host only
✓ Bridge network by default
✗ No multi-host networking
✗ No advanced load balancing
✗ No service mesh
```

**Communication:**
```bash
# From web container
curl http://api:3000  # Works via DNS

# From api container
curl http://db:5432   # Works via DNS
```

---

### Docker Swarm Networking

**Overlay Network:**
```yaml
version: '3.8'

services:
  web:
    image: nginx
    networks:
      - frontend
    deploy:
      replicas: 3

  api:
    image: myapi
    networks:
      - frontend
      - backend
    deploy:
      replicas: 5

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    attachable: true
```

**Characteristics:**
```
✓ Multi-host networking (overlay)
✓ Built-in load balancing
✓ Service discovery across hosts
✓ Encrypted overlay networks
✓ VIP (Virtual IP) per service
✗ Limited compared to K8s
```

**How It Works:**
```
Service: api (5 replicas across 3 nodes)

Node 1: api-1, api-2
Node 2: api-3, api-4
Node 3: api-5

Request flow:
web → api (service VIP)
    → Swarm routing mesh
    → Any api replica (load balanced)
```

**Ingress Routing:**
```
External request: 80
    ↓
Any Swarm node
    ↓
Routing mesh
    ↓
Correct service container

All nodes listen on published ports
Traffic routed to correct container
```

---

### Kubernetes Networking

**Flat Network Model:**
```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 5
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
        image: myapi
        ports:
        - containerPort: 3000

---
# api-service.yaml
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
  type: ClusterIP
```

**Characteristics:**
```
✓ Flat network (all pods can reach each other)
✓ Each pod gets unique IP
✓ Services provide stable endpoint
✓ Multiple service types
✓ Advanced networking (CNI plugins)
✓ Network policies for security
✓ Service mesh integration
```

**Pod-to-Pod Communication:**
```
Pod 1 (10.244.1.5) → Pod 2 (10.244.2.8)
Direct communication
No NAT
Any pod can reach any pod
```

**Service Communication:**
```
Request to api:3000
    ↓
Service (ClusterIP: 10.96.0.10)
    ↓
iptables/IPVS load balancing
    ↓
One of 5 api pods
```

---

### Service Types Comparison

**Docker Compose:**
```yaml
# Only internal networking
services:
  api:
    ports:
      - "3000:3000"  # Host port mapping
```

**Docker Swarm:**
```yaml
# Published ports + routing mesh
services:
  api:
    ports:
      - "3000:3000"
    deploy:
      mode: replicated
      # All nodes accept traffic on 3000
      # Routing mesh routes to correct container
```

**Kubernetes:**
```yaml
# Multiple service types

# ClusterIP (internal only)
apiVersion: v1
kind: Service
spec:
  type: ClusterIP
  ports:
  - port: 3000

# NodePort (expose on all nodes)
apiVersion: v1
kind: Service
spec:
  type: NodePort
  ports:
  - port: 3000
    nodePort: 30000

# LoadBalancer (cloud load balancer)
apiVersion: v1
kind: Service
spec:
  type: LoadBalancer
  ports:
  - port: 3000
```

---

### Load Balancing

**Compose:**
```
No built-in load balancing
Manual setup needed (nginx)
```

**Swarm:**
```
Built-in routing mesh
VIP per service
Round-robin by default
```

**Kubernetes:**
```
Service abstraction
kube-proxy handles load balancing
Multiple algorithms:
- iptables (default)
- IPVS (advanced)
- eBPF (modern)
```

---

### Network Policies

**Compose:**
```yaml
# No network policies
# Isolation via separate networks only

networks:
  frontend:
  backend:
```

**Swarm:**
```yaml
# No native network policies
# Isolation via networks
# Can use third-party solutions
```

**Kubernetes:**
```yaml
# Native network policies
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
      port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432
```

---

### Service Discovery

**Compose:**
```
DNS: service-name
Example: http://api
```

**Swarm:**
```
DNS: service-name
VIP load balancing
Example: http://api (→ any replica)
```

**Kubernetes:**
```
DNS: service-name.namespace.svc.cluster.local
Example: http://api.default.svc.cluster.local
Short form: http://api (same namespace)
```

---

### Ingress/External Access

**Compose:**
```yaml
# Port mapping only
ports:
  - "80:80"
```

**Swarm:**
```yaml
# Routing mesh
ports:
  - "80:80"
# Any node accepts traffic
```

**Kubernetes:**
```yaml
# Ingress controller
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        backend:
          service:
            name: api
            port:
              number: 3000
```

---

### Service Mesh

**Compose:**
```
Not applicable
```

**Swarm:**
```
No native service mesh
Can integrate third-party
```

**Kubernetes:**
```
Native service mesh support:
- Istio
- Linkerd
- Consul Connect

Features:
- Mutual TLS
- Traffic management
- Observability
- Advanced routing
```

---

### Comparison Table

| Feature | Compose | Swarm | Kubernetes |
|---------|---------|-------|------------|
| **Multi-host** | ❌ | ✅ | ✅ |
| **Load Balancing** | ❌ | ✅ Basic | ✅ Advanced |
| **Service Discovery** | ✅ DNS | ✅ DNS + VIP | ✅ DNS + Service |
| **Network Policies** | ❌ | ❌ | ✅ |
| **Service Mesh** | ❌ | ⚠️ Limited | ✅ |
| **Ingress** | ❌ | ⚠️ Routing Mesh | ✅ Controllers |
| **Encryption** | ❌ | ✅ Optional | ✅ Optional |
| **CNI Plugins** | ❌ | ❌ | ✅ |

</details>

</details>

---

[← Previous: 8.3 Compose Operations](../08-docker-compose/03-compose-operations.md)