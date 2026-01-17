# 6.1 Network Fundamentals

Understanding Docker networking: how containers communicate with each other and the outside world.

---

## Docker Network Overview

Docker provides networking capabilities that allow containers to communicate with each other and with external networks.

```
Docker Networking Components:

┌─────────────────────────────────────────┐
│           Docker Host                   │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  Container 1 (172.17.0.2)       │    │
│  └──────────────┬──────────────────┘    │
│                 │                       │
│  ┌─────────────▼──────────────────┐     │
│  │    Docker Network Bridge       │     │
│  │    (docker0: 172.17.0.1)       │     │
│  └──────────────┬─────────────────┘     │
│                 │                       │
│  ┌─────────────▼──────────────────┐     │
│  │  Container 2 (172.17.0.3)      │     │
│  └────────────────────────────────┘     │
│                 │                       │
└─────────────────┼───────────────────────┘
                  │
                  ▼
            Host Network (eth0)
                  │
                  ▼
              Internet
```

---

## Network Drivers

Docker supports different network drivers for different use cases.

### 1. Bridge (Default)

**What it is:**
```
Private internal network on host
Containers get IP addresses from bridge subnet
Default for containers that don't specify network
```

**How it works:**
```
Docker creates:
├── Virtual bridge (docker0)
├── IP range (172.17.0.0/16 by default)
└── Network namespace per container

Container connections:
Container → Bridge → Host → Internet
```

**Example:**
```bash
# Default bridge network
docker run -d --name web nginx

# Check container IP
docker inspect web | grep IPAddress
# "IPAddress": "172.17.0.2"

# Containers on default bridge
docker run -d --name api nginx
# "IPAddress": "172.17.0.3"

# Can communicate via IP
docker exec web ping 172.17.0.3
# PING 172.17.0.3: 64 bytes...
```

**Custom Bridge:**
```bash
# Create custom bridge network
docker network create my-network

# Run containers on custom network
docker run -d --name web --network my-network nginx
docker run -d --name api --network my-network nginx

# Can communicate via container name (DNS)
docker exec web ping api
# PING api (172.18.0.3): 64 bytes...
```

**Default vs Custom Bridge:**
```
Default Bridge:
✗ No automatic DNS resolution
✗ Must use --link (deprecated)
✓ Simple for single-host

Custom Bridge:
✓ Automatic DNS resolution
✓ Better isolation
✓ Configurable
✓ Recommended for production
```

---

### 2. Host

**What it is:**
```
Container shares host's network namespace
No network isolation
Container uses host's IP directly
```

**How it works:**
```
Container network = Host network
No separate IP address
Ports bind directly to host
```

**Example:**
```bash
# Run with host network
docker run -d --name web --network host nginx

# Container uses host IP
# nginx binds to host's port 80 directly
# No -p needed

# Check
curl http://localhost:80
# Works - nginx on host network
```

**Use Cases:**
```
✓ Maximum network performance
✓ Lots of port mappings needed
✓ Network monitoring tools
✗ Not isolated
✗ Port conflicts possible
✗ Not portable
```

---

### 3. None

**What it is:**
```
No networking
Complete network isolation
Loopback interface only
```

**Example:**
```bash
# Container with no network
docker run -d --name isolated --network none alpine sleep infinity

# Only loopback
docker exec isolated ip addr
# 1: lo: <LOOPBACK,UP>
#     inet 127.0.0.1/8

# No external connectivity
docker exec isolated ping google.com
# Error: network is unreachable
```

**Use Cases:**
```
✓ Maximum security/isolation
✓ Batch processing (no network needed)
✓ Testing
```

---

### 4. Overlay

**What it is:**
```
Multi-host networking
Containers across different Docker hosts
Used in Docker Swarm
```

**How it works:**
```
VXLAN tunnel between hosts

Host 1:
Container A (10.0.0.2)
       │
       ▼
   Overlay Network (10.0.0.0/24)
       │
       ▼
Host 2:
Container B (10.0.0.3)

Containers communicate as if on same network
```

**Example:**
```bash
# Initialize swarm
docker swarm init

# Create overlay network
docker network create \
    --driver overlay \
    my-overlay

# Deploy services
docker service create \
    --name web \
    --network my-overlay \
    nginx

docker service create \
    --name api \
    --network my-overlay \
    myapi

# Services can communicate across hosts
```

---

### 5. Macvlan

**What it is:**
```
Assign MAC address to container
Container appears as physical device on network
Direct connection to physical network
```

**Example:**
```bash
# Create macvlan network
docker network create -d macvlan \
    --subnet=192.168.1.0/24 \
    --gateway=192.168.1.1 \
    -o parent=eth0 \
    macvlan-net

# Container gets IP on physical network
docker run -d \
    --name web \
    --network macvlan-net \
    --ip 192.168.1.100 \
    nginx

# Container accessible at 192.168.1.100 on LAN
```

**Use Cases:**
```
✓ Legacy applications expecting physical network
✓ Network monitoring
✓ Direct LAN access needed
```

---

## Container Communication

### Same Network

**Custom Bridge Network:**
```bash
# Create network
docker network create app-network

# Run containers
docker run -d --name db --network app-network postgres
docker run -d --name api --network app-network myapi
docker run -d --name web --network app-network nginx

# Communicate via container names
docker exec api ping db
docker exec web curl http://api:8000/health

# Automatic DNS resolution
# db → IP of db container
# api → IP of api container
# web → IP of web container
```

**docker-compose (Automatic):**
```yaml
version: '3.8'

services:
  db:
    image: postgres
    # Accessible as 'db'

  api:
    image: myapi
    # Accessible as 'api'
    environment:
      - DATABASE_URL=postgresql://db:5432/mydb

  web:
    image: nginx
    # Can reach api at http://api:8000
```

---

### Different Networks

**Isolated by Default:**
```bash
# Network 1
docker network create network1
docker run -d --name app1 --network network1 alpine

# Network 2
docker network create network2
docker run -d --name app2 --network network2 alpine

# Cannot communicate
docker exec app1 ping app2
# Error: app2 not found
```

**Connect Container to Multiple Networks:**
```bash
# Connect app1 to network2
docker network connect network2 app1

# Now app1 is on both networks
docker exec app1 ping app2
# PING app2: 64 bytes...
```

---

## Port Mapping

### Publishing Ports

**Syntax:**
```bash
-p HOST_PORT:CONTAINER_PORT
-p HOST_IP:HOST_PORT:CONTAINER_PORT
```

**Examples:**
```bash
# Map port 8080 to 80
docker run -d -p 8080:80 nginx
# Access at http://localhost:8080

# Map to all interfaces
docker run -d -p 80:80 nginx

# Bind to specific IP
docker run -d -p 127.0.0.1:8080:80 nginx
# Only accessible from localhost

# Random host port
docker run -d -p 80 nginx
# Docker assigns random port (e.g., 32768)

# Multiple ports
docker run -d \
    -p 8080:80 \
    -p 8443:443 \
    nginx

# UDP port
docker run -d -p 53:53/udp dns-server
```

**Port Mapping in docker-compose:**
```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"           # HOST:CONTAINER
      - "127.0.0.1:8443:443" # IP:HOST:CONTAINER
      - "3000-3005:3000-3005" # Range
```

---

## DNS Resolution

### Automatic DNS

**Custom Bridge Networks:**
```bash
# Containers on same custom network
docker network create mynet
docker run -d --name db --network mynet postgres
docker run -d --name app --network mynet myapp

# Automatic DNS
docker exec app ping db
# PING db (172.18.0.2)

# Name resolution works
docker exec app nslookup db
# Server: 127.0.0.11
# Name: db
# Address: 172.18.0.2
```

**Embedded DNS Server:**
```
Docker runs DNS server at 127.0.0.11
Resolves container names to IPs
Available in custom networks only
```

---

### DNS Configuration

**Custom DNS:**
```bash
# Set DNS servers
docker run -d \
    --dns 8.8.8.8 \
    --dns 8.8.4.4 \
    nginx

# Set search domains
docker run -d \
    --dns-search example.com \
    nginx

# Set hostname
docker run -d \
    --hostname web-server \
    nginx
```

**In docker-compose:**
```yaml
services:
  web:
    image: nginx
    dns:
      - 8.8.8.8
      - 8.8.4.4
    dns_search:
      - example.com
    hostname: web-server
```

---

## Network Isolation

### Security Groups

**Separate Networks:**
```bash
# Frontend network
docker network create frontend

# Backend network
docker network create backend

# Web server (public facing)
docker run -d --name web --network frontend nginx

# API (both networks)
docker run -d --name api myapi
docker network connect frontend api
docker network connect backend api

# Database (internal only)
docker run -d --name db --network backend postgres

# Result:
# web can reach api (frontend network)
# api can reach db (backend network)
# web CANNOT reach db (isolated)
```

**docker-compose Example:**
```yaml
services:
  web:
    image: nginx
    networks:
      - frontend
    ports:
      - "80:80"

  api:
    image: myapi
    networks:
      - frontend
      - backend

  db:
    image: postgres
    networks:
      - backend
    # Not exposed to frontend

networks:
  frontend:
  backend:
```

---

## Real-World Examples

### Example 1: Three-Tier Application

```yaml
version: '3.8'

services:
  # Load Balancer (public)
  nginx:
    image: nginx
    ports:
      - "80:80"
      - "443:443"
    networks:
      - frontend

  # Application Servers (private)
  app:
    image: myapp
    deploy:
      replicas: 3
    networks:
      - frontend
      - backend
    environment:
      - DATABASE_URL=postgresql://db:5432/mydb

  # Database (private)
  db:
    image: postgres
    networks:
      - backend
    volumes:
      - db-data:/var/lib/postgresql/data

  # Cache (private)
  redis:
    image: redis
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # No external access

volumes:
  db-data:
```

**Network Flow:**
```
Internet
   │
   ▼
nginx (port 80/443)
   │
   ▼ (frontend network)
app (3 replicas)
   │
   ▼ (backend network)
db, redis

Backend network: internal: true
→ db and redis cannot access internet
→ Extra security layer
```

---

### Example 2: Microservices

```yaml
version: '3.8'

services:
  # API Gateway
  gateway:
    image: kong
    ports:
      - "8000:8000"
    networks:
      - public
      - services

  # User Service
  user-service:
    image: user-service
    networks:
      - services
      - data
    environment:
      - DB_HOST=user-db

  # Order Service
  order-service:
    image: order-service
    networks:
      - services
      - data
    environment:
      - DB_HOST=order-db

  # Databases
  user-db:
    image: postgres
    networks:
      - data

  order-db:
    image: postgres
    networks:
      - data

  # Message Queue
  rabbitmq:
    image: rabbitmq
    networks:
      - services

networks:
  public:      # Internet facing
  services:    # Microservices mesh
  data:        # Database tier
    internal: true
```

---

### Example 3: Development Environment

```yaml
version: '3.8'

services:
  web:
    build: ./web
    ports:
      - "3000:3000"
    networks:
      - app-network
    volumes:
      - ./web:/app

  api:
    build: ./api
    ports:
      - "8000:8000"
    networks:
      - app-network
    depends_on:
      - db
      - redis

  db:
    image: postgres
    networks:
      - app-network
    environment:
      - POSTGRES_DB=myapp
    ports:
      - "5432:5432"  # Exposed for development

  redis:
    image: redis
    networks:
      - app-network
    ports:
      - "6379:6379"  # Exposed for development

networks:
  app-network:
    driver: bridge
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The __________ network driver is the default for Docker containers.
2. Containers on the same __________ bridge network can communicate using container names.
3. The __________ network driver removes all network isolation and uses the host's network.
4. Port mapping is specified with the __________ flag.
5. Docker's embedded DNS server runs at __________.
6. The __________ network driver enables multi-host networking.

### True/False

1. ⬜ Containers on the default bridge network can use DNS to resolve container names
2. ⬜ Custom bridge networks provide automatic DNS resolution
3. ⬜ The host network driver provides network isolation
4. ⬜ Containers on different networks cannot communicate by default
5. ⬜ Port mapping is required for containers to communicate with each other
6. ⬜ The none network driver gives containers no networking capabilities
7. ⬜ A container can be connected to multiple networks

### Multiple Choice

1. Which network driver is default?
   - A) host
   - B) bridge
   - C) none
   - D) overlay

2. How do containers on a custom bridge network communicate?
   - A) IP addresses only
   - B) Container names (DNS)
   - C) Host ports
   - D) Cannot communicate

3. What does -p 8080:80 mean?
   - A) Container port 8080 maps to host port 80
   - B) Host port 8080 maps to container port 80
   - C) Both ports are 8080
   - D) Invalid syntax

4. Which network driver is for multi-host setups?
   - A) bridge
   - B) host
   - C) overlay
   - D) macvlan

5. What's the embedded DNS server address?
   - A) 127.0.0.1
   - B) 127.0.0.11
   - C) 8.8.8.8
   - D) 172.17.0.1

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. bridge
2. custom
3. host
4. -p (or --publish)
5. 127.0.0.11
6. overlay

**True/False:**
1. ❌ False - Default bridge doesn't support DNS resolution
2. ✅ True - Custom bridge networks have automatic DNS
3. ❌ False - Host network has no isolation
4. ✅ True - Networks are isolated by default
5. ❌ False - Port mapping is for external access, not inter-container
6. ✅ True - None network provides no networking
7. ✅ True - Containers can join multiple networks

**Multiple Choice:**
1. **B** - bridge
2. **B** - Container names (DNS)
3. **B** - Host port 8080 maps to container port 80
4. **C** - overlay
5. **B** - 127.0.0.11

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the difference between default bridge and custom bridge networks

<details>
<summary><strong>View Answer</strong></summary>

**Key Differences:**

---

**Default Bridge Network**

**Characteristics:**
```
Name: bridge (docker0)
Created by: Docker automatically
IP Range: 172.17.0.0/16 (default)
DNS: Not supported
```

**Limitations:**
```bash
# Containers on default bridge
docker run -d --name web nginx
docker run -d --name api nodejs

# Cannot use container names
docker exec web ping api
# ping: api: Name or service not known

# Must use IP addresses
docker exec web ping 172.17.0.3
# PING 172.17.0.3: 64 bytes

# Or deprecated --link
docker run -d --name api --link web nodejs
```

**Problems:**
```
✗ No automatic DNS resolution
✗ Must use IPs (fragile)
✗ --link is deprecated
✗ Less secure
✗ Harder to manage
```

---

**Custom Bridge Network**

**Characteristics:**
```
Name: User-defined
Created by: docker network create
IP Range: Configurable
DNS: Automatic resolution
```

**Benefits:**
```bash
# Create custom network
docker network create myapp

# Containers on custom network
docker run -d --name web --network myapp nginx
docker run -d --name api --network myapp nodejs

# Automatic DNS resolution
docker exec web ping api
# PING api (172.18.0.3): 64 bytes

# Works both ways
docker exec api ping web
# PING web (172.18.0.2): 64 bytes
```

**Advantages:**
```
✓ Automatic DNS resolution
✓ Better isolation
✓ Configurable subnet
✓ Can attach/detach while running
✓ Better security
✓ Easier management
```

---

**Comparison Table:**

| Feature | Default Bridge | Custom Bridge |
|---------|---------------|---------------|
| **DNS resolution** | ✗ No | ✓ Yes |
| **Isolation** | All containers share | Separate per network |
| **Configuration** | Fixed | Customizable |
| **IP range** | 172.17.0.0/16 | User-defined |
| **Service discovery** | --link only | Automatic |
| **Production use** | ✗ Not recommended | ✓ Recommended |

---

**Real-World Example:**

**Default Bridge (Bad):**
```bash
# E-commerce application
docker run -d --name db postgres
docker run -d --name api myapi
docker run -d --name web nginx

# API needs to connect to database
docker inspect db | grep IPAddress
# "IPAddress": "172.17.0.2"

# Hardcode IP in API
DATABASE_URL=postgres://172.17.0.2:5432

# Problems:
# - IP changes if container recreated
# - Brittle configuration
# - Hard to maintain
```

**Custom Bridge (Good):**
```yaml
version: '3.8'

services:
  db:
    image: postgres
    networks:
      - backend

  api:
    image: myapi
    networks:
      - backend
    environment:
      # Use service name
      - DATABASE_URL=postgresql://db:5432/mydb

  web:
    image: nginx
    networks:
      - frontend
      - backend
    ports:
      - "80:80"

networks:
  frontend:
  backend:
```

**Benefits:**
```
✓ DNS: "db" always resolves correctly
✓ Network isolation (frontend/backend)
✓ No IP hardcoding
✓ Easy to maintain
✓ Portable configuration
```

---

**Migration Guide:**

```bash
# Old way (default bridge)
docker run -d --name db postgres
docker run -d --name api --link db myapi

# New way (custom bridge)
docker network create myapp
docker run -d --name db --network myapp postgres
docker run -d --name api --network myapp myapi
# Remove --link, use DNS
```

**Recommendation:**
```
Always use custom bridge networks in production
Default bridge is for quick testing only
```

</details>

### Question 2: How would you design network architecture for a microservices application?

<details>
<summary><strong>View Answer</strong></summary>

**Multi-Tier Network Architecture:**

---

### Design Principles

```
1. Network Segmentation
   - Separate public and private services
   - Isolate data tier
   - Control traffic flow

2. Least Privilege
   - Only expose what's necessary
   - Internal services stay internal
   - No direct database access from internet

3. Service Discovery
   - Use DNS for service communication
   - Avoid IP hardcoding
   - Easy scaling
```

---

### Architecture Design

**Three-Network Approach:**

```yaml
version: '3.8'

networks:
  # Public-facing network
  public:
    driver: bridge
    
  # Service mesh (microservices)
  services:
    driver: bridge
    
  # Data tier (databases)
  data:
    driver: bridge
    internal: true  # No internet access
```

**Complete Implementation:**

```yaml
version: '3.8'

services:
  # API Gateway (public entry point)
  gateway:
    image: kong:latest
    ports:
      - "80:8000"
      - "443:8443"
    networks:
      - public
      - services
    environment:
      - KONG_DATABASE=off

  # User Service
  user-service:
    image: user-service:latest
    deploy:
      replicas: 3
    networks:
      - services
      - data
    environment:
      - DATABASE_URL=postgresql://user-db:5432/users
      - REDIS_URL=redis://cache:6379
    depends_on:
      - user-db
      - cache

  # Order Service
  order-service:
    image: order-service:latest
    deploy:
      replicas: 3
    networks:
      - services
      - data
    environment:
      - DATABASE_URL=postgresql://order-db:5432/orders
      - REDIS_URL=redis://cache:6379
      - RABBITMQ_URL=amqp://queue:5672
    depends_on:
      - order-db
      - cache
      - queue

  # Product Service
  product-service:
    image: product-service:latest
    deploy:
      replicas: 2
    networks:
      - services
      - data
    environment:
      - DATABASE_URL=postgresql://product-db:5432/products
      - ELASTICSEARCH_URL=http://search:9200
    depends_on:
      - product-db
      - search

  # Databases (isolated)
  user-db:
    image: postgres:15
    networks:
      - data
    volumes:
      - user-db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=users

  order-db:
    image: postgres:15
    networks:
      - data
    volumes:
      - order-db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=orders

  product-db:
    image: postgres:15
    networks:
      - data
    volumes:
      - product-db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=products

  # Shared Services
  cache:
    image: redis:7-alpine
    networks:
      - data

  queue:
    image: rabbitmq:3-management
    networks:
      - services
      - data

  search:
    image: elasticsearch:8.11.0
    networks:
      - data
    environment:
      - discovery.type=single-node

networks:
  public:
    driver: bridge
    
  services:
    driver: bridge
    
  data:
    driver: bridge
    internal: true

volumes:
  user-db-data:
  order-db-data:
  product-db-data:
```

---

### Network Flow

```
Internet
   │
   ▼
API Gateway (public + services networks)
   │
   ▼ (services network)
   ├── User Service
   ├── Order Service
   └── Product Service
       │
       ▼ (data network)
       ├── Databases (isolated)
       ├── Cache (isolated)
       ├── Queue
       └── Search (isolated)

Security:
- Public: Internet-accessible
- Services: Microservices mesh
- Data: Internal only (no internet)
```

---

### Service Communication

**Within Services Network:**
```yaml
# Order service calls user service
order-service:
  environment:
    - USER_SERVICE_URL=http://user-service:8000
    
# Automatic DNS resolution
# "user-service" → IP of user-service
```

**Cross-Network Access:**
```yaml
# Gateway routes to services
gateway:
  networks:
    - public   # Receives internet traffic
    - services # Forwards to microservices

# Services access data tier
user-service:
  networks:
    - services # Communicates with other services
    - data     # Accesses user-db
```

---

### Security Layers

**1. Network Isolation:**
```yaml
networks:
  data:
    internal: true  # No external access
    
# Result:
# - Databases cannot access internet
# - No inbound connections from internet
# - Only connected containers can access
```

**2. Firewall Rules:**
```yaml
services:
  gateway:
    # Only expose necessary ports
    ports:
      - "80:8000"
      - "443:8443"
    
  # Services: No exposed ports
  # Only accessible via gateway
```

**3. Access Control:**
```yaml
services:
  admin-db:
    image: postgres
    networks:
      - admin-only  # Separate network
    
networks:
  admin-only:
    driver: bridge
    # Only admin tools connected
```

---

### Scalability Considerations

**Load Balancing:**
```yaml
services:
  user-service:
    deploy:
      replicas: 3
      
# DNS round-robin between replicas
# gateway → user-service (any of 3 replicas)
```

**Service Discovery:**
```yaml
# Services find each other automatically
order-service:
  environment:
    # DNS-based discovery
    - USER_SERVICE=http://user-service:8000
    - PRODUCT_SERVICE=http://product-service:8000
    
# No hardcoded IPs
# Scales automatically
```

---

### Monitoring Network

```yaml
services:
  # Monitoring tools
  prometheus:
    image: prom/prometheus
    networks:
      - services    # Scrape service metrics
      - monitoring
    
  grafana:
    image: grafana/grafana
    networks:
      - monitoring
    ports:
      - "3000:3000"
    
networks:
  monitoring:
    driver: bridge
```

---

### Production Enhancements

**1. Service Mesh (Advanced):**
```yaml
# Use Istio or Linkerd for:
# - Mutual TLS between services
# - Advanced traffic routing
# - Circuit breaking
# - Observability
```

**2. External Load Balancer:**
```
Internet
   │
   ▼
Cloud Load Balancer (AWS ALB, GCP LB)
   │
   ├─ Gateway Instance 1
   ├─ Gateway Instance 2
   └─ Gateway Instance 3
```

**3. Multi-Host Overlay:**
```yaml
# For Docker Swarm or Kubernetes
networks:
  services:
    driver: overlay
    # Spans multiple hosts
```

---

### Best Practices

```
1. Network Segmentation
   ✓ Public, services, data tiers
   ✓ Internal networks for sensitive data
   ✓ Least privilege access

2. DNS-Based Discovery
   ✓ Use container names
   ✓ Avoid IP hardcoding
   ✓ Enable easy scaling

3. Security
   ✓ internal: true for data networks
   ✓ Expose only necessary ports
   ✓ Use encryption (TLS)

4. Monitoring
   ✓ Dedicated monitoring network
   ✓ Service mesh for observability
   ✓ Network performance metrics

5. Documentation
   ✓ Document network architecture
   ✓ Service dependencies
   ✓ Port assignments
```

</details>

### Question 3: Troubleshoot a scenario where containers cannot communicate

<details>
<summary><strong>View Answer</strong></summary>

**Systematic Troubleshooting:**

---

### Scenario

```
Problem: Container A cannot reach Container B
Error: "Connection refused" or "Host not found"
```

---

### Step 1: Verify Both Containers Running

```bash
# Check container status
docker ps

# Expected: Both containers in "Up" status
# If not running:
docker start container-a
docker start container-b

# Check for restart loops
docker ps -a
# STATUS: Restarting (1) 5 seconds ago

# View logs
docker logs container-b
```

---

### Step 2: Check Network Configuration

```bash
# List networks
docker network ls

# Inspect containers' networks
docker inspect container-a --format='{{json .NetworkSettings.Networks}}' | jq
docker inspect container-b --format='{{json .NetworkSettings.Networks}}' | jq

# Common issue: Different networks
Container A: network1
Container B: network2
# Cannot communicate!
```

**Fix: Same Network**
```bash
# Connect container-b to container-a's network
docker network connect network1 container-b

# Verify
docker network inspect network1
```

---

### Step 3: DNS Resolution

```bash
# Test DNS from container-a
docker exec container-a nslookup container-b

# If fails: "server can't find container-b"

# Check network type
docker network inspect network1 --format='{{.Driver}}'

# If "bridge" (default bridge):
# - No DNS support
# - Must use IP or custom bridge

# Solution: Use custom bridge
docker network create mynetwork
docker network connect mynetwork container-a
docker network connect mynetwork container-b

# Test again
docker exec container-a ping container-b
```

---

### Step 4: IP Connectivity

```bash
# Get container-b IP
IP=$(docker inspect container-b --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}')
echo $IP  # e.g., 172.18.0.3

# Test from container-a
docker exec container-a ping $IP

# If ping works but DNS doesn't:
# - Network connectivity OK
# - DNS issue only
# - Use custom bridge network

# If ping fails:
# - Network isolation
# - Firewall rules
# - Different networks
```

---

### Step 5: Port Availability

```bash
# Check if service listening
docker exec container-b netstat -tlnp

# Expected output:
# tcp  0  0.0.0.0:8000  0.0.0.0:*  LISTEN  1/python

# If not listening:
# - Application not started
# - Bound to wrong interface
# - Wrong port

# Test from inside container-b
docker exec container-b curl http://localhost:8000

# If works locally but not from container-a:
# - Might be bound to 127.0.0.1 only
# - Should bind to 0.0.0.0
```

---

### Step 6: Firewall Rules

```bash
# Check iptables (on host)
sudo iptables -L DOCKER -n

# Docker creates rules automatically
# If custom firewall:
# - Might block Docker traffic
# - Check rules carefully

# Test with firewall disabled (temporarily)
sudo systemctl stop firewalld  # CentOS/RHEL
sudo ufw disable  # Ubuntu

# If works now:
# - Firewall blocking
# - Add Docker rules
```

---

### Step 7: Application Configuration

```bash
# Check application logs
docker logs container-b

# Common issues:
# - Binding to 127.0.0.1 instead of 0.0.0.0
# - Wrong port configuration
# - Authentication required
# - TLS certificate errors

# Example: Python Flask
# Bad:  app.run(host='127.0.0.1')
# Good: app.run(host='0.0.0.0')
```

---

### Complete Diagnostic Script

```bash
#!/bin/bash

CONTAINER_A=$1
CONTAINER_B=$2

echo "=== Docker Container Communication Diagnostics ==="
echo ""

# Step 1: Container Status
echo "1. Checking container status..."
A_STATUS=$(docker inspect -f '{{.State.Status}}' $CONTAINER_A 2>/dev/null)
B_STATUS=$(docker inspect -f '{{.State.Status}}' $CONTAINER_B 2>/dev/null)

echo "  $CONTAINER_A: $A_STATUS"
echo "  $CONTAINER_B: $B_STATUS"

if [ "$A_STATUS" != "running" ] || [ "$B_STATUS" != "running" ]; then
    echo "  ✗ One or both containers not running"
    exit 1
fi

# Step 2: Network Check
echo ""
echo "2. Checking networks..."
A_NETWORKS=$(docker inspect -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}' $CONTAINER_A)
B_NETWORKS=$(docker inspect -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}' $CONTAINER_B)

echo "  $CONTAINER_A networks: $A_NETWORKS"
echo "  $CONTAINER_B networks: $B_NETWORKS"

# Find common network
COMMON=""
for net_a in $A_NETWORKS; do
    for net_b in $B_NETWORKS; do
        if [ "$net_a" = "$net_b" ]; then
            COMMON=$net_a
        fi
    done
done

if [ -z "$COMMON" ]; then
    echo "  ✗ No common network found"
    echo "  → Solution: docker network connect <network> $CONTAINER_B"
    exit 1
else
    echo "  ✓ Common network: $COMMON"
fi

# Step 3: DNS Test
echo ""
echo "3. Testing DNS resolution..."
if docker exec $CONTAINER_A nslookup $CONTAINER_B > /dev/null 2>&1; then
    echo "  ✓ DNS resolution works"
else
    echo "  ✗ DNS resolution failed"
    
    # Check if default bridge
    DRIVER=$(docker network inspect $COMMON -f '{{.Driver}}')
    NAME=$(docker network inspect $COMMON -f '{{.Name}}')
    
    if [ "$NAME" = "bridge" ]; then
        echo "  → Using default bridge (no DNS support)"
        echo "  → Solution: Use custom bridge network"
    fi
fi

# Step 4: IP Connectivity
echo ""
echo "4. Testing IP connectivity..."
B_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $CONTAINER_B | head -1)
echo "  $CONTAINER_B IP: $B_IP"

if docker exec $CONTAINER_A ping -c 1 $B_IP > /dev/null 2>&1; then
    echo "  ✓ IP connectivity works"
else
    echo "  ✗ Cannot ping $B_IP"
fi

# Step 5: Port Check
echo ""
echo "5. Checking listening ports on $CONTAINER_B..."
docker exec $CONTAINER_B netstat -tlnp 2>/dev/null || \
docker exec $CONTAINER_B ss -tlnp 2>/dev/null || \
echo "  (netstat/ss not available in container)"

echo ""
echo "=== Diagnostics Complete ==="
```

**Usage:**
```bash
chmod +x diagnose.sh
./diagnose.sh container-a container-b
```

---

### Common Issues & Solutions

**Issue 1: Default Bridge Network**
```bash
# Problem
docker run -d --name app1 nginx
docker run -d --name app2 nginx
docker exec app1 ping app2
# Error: ping: app2: Name or service not known

# Solution
docker network create mynet
docker network connect mynet app1
docker network connect mynet app2
docker exec app1 ping app2  # Works!
```

**Issue 2: Different Networks**
```bash
# Problem
docker network create net1
docker network create net2
docker run -d --name app1 --network net1 nginx
docker run -d --name app2 --network net2 nginx
docker exec app1 ping app2  # Fails

# Solution
docker network connect net1 app2
# Now app2 is on both networks
```

**Issue 3: Binding to 127.0.0.1**
```python
# Problem (Python app)
app.run(host='127.0.0.1', port=8000)
# Only accessible from inside container

# Solution
app.run(host='0.0.0.0', port=8000)
# Accessible from other containers
```

**Issue 4: Wrong Port**
```bash
# Problem
# Container B listening on port 8000
# Container A trying to connect to port 3000

# Check what's actually listening
docker exec container-b netstat -tlnp

# Fix connection URL in container-a
```

</details>

</details>

---

[← Previous: 5.2 Volume Operations](../05-docker-volumes/02-volume-operations.md) | [Next: 6.2 Network Operations →](02-network-operations.md)