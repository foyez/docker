# 8.1 Docker Compose Fundamentals

Docker Compose is a tool for defining and running multi-container Docker applications.

---

## What is Docker Compose?

**Docker Compose** is a tool that allows you to define and manage multi-container applications using a single YAML file.

Instead of running multiple `docker run` commands, you describe your entire application stack in `docker-compose.yml` and manage it with simple commands.

### Core Concept

> **Docker** = Run one container  
> **Docker Compose** = Run multiple containers as one application

---

## Why Docker Compose?

### Problem: Managing Multiple Containers

Imagine building an e-commerce application:

```
Required Services:
├── Web Application (Node.js)
├── API Server (Python)
├── Database (PostgreSQL)
├── Cache (Redis)
├── Message Queue (RabbitMQ)
└── Worker Service (Go)
```

**Without Docker Compose:**

```bash
# Create network
docker network create ecommerce-network

# Run database
docker run -d \
  --name postgres-db \
  --network ecommerce-network \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=ecommerce \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15

# Run Redis
docker run -d \
  --name redis-cache \
  --network ecommerce-network \
  redis:7

# Run RabbitMQ
docker run -d \
  --name rabbitmq \
  --network ecommerce-network \
  -p 5672:5672 \
  rabbitmq:3

# Run API
docker run -d \
  --name api-server \
  --network ecommerce-network \
  -e DATABASE_URL=postgres://postgres:secret@postgres-db:5432/ecommerce \
  -e REDIS_URL=redis://redis-cache:6379 \
  -p 8000:8000 \
  my-api:latest

# Run Web App
docker run -d \
  --name web-app \
  --network ecommerce-network \
  -e API_URL=http://api-server:8000 \
  -p 3000:3000 \
  my-web:latest

# Run Worker
docker run -d \
  --name worker \
  --network ecommerce-network \
  -e RABBITMQ_URL=amqp://rabbitmq:5672 \
  my-worker:latest
```

**Problems:**
- ❌ 6 separate commands to memorize
- ❌ Easy to make mistakes
- ❌ Hard to share with team
- ❌ Difficult to start/stop everything
- ❌ No dependency management
- ❌ Manual cleanup required

**With Docker Compose:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: ecommerce
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7

  rabbitmq:
    image: rabbitmq:3
    ports:
      - "5672:5672"

  api:
    image: my-api:latest
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/ecommerce
      REDIS_URL: redis://redis:6379
    ports:
      - "8000:8000"
    depends_on:
      - db
      - redis

  web:
    image: my-web:latest
    environment:
      API_URL: http://api:8000
    ports:
      - "3000:3000"
    depends_on:
      - api

  worker:
    image: my-worker:latest
    environment:
      RABBITMQ_URL: amqp://rabbitmq:5672
    depends_on:
      - rabbitmq

volumes:
  postgres-data:
```

```bash
# Start everything
docker-compose up -d

# Stop everything
docker-compose down

# View logs
docker-compose logs -f
```

**Benefits:**
- ✅ One file describes entire stack
- ✅ One command to start/stop
- ✅ Easy to share and version control
- ✅ Automatic network creation
- ✅ Dependency management
- ✅ Easy cleanup

---

## Docker Compose vs Docker CLI

### Real-World Example: WordPress with MySQL

#### Using Docker CLI

```bash
# Step 1: Create network
docker network create wordpress-net

# Step 2: Run MySQL
docker run -d \
  --name mysql-db \
  --network wordpress-net \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wpuser \
  -e MYSQL_PASSWORD=wppassword \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0

# Step 3: Wait for MySQL to be ready (manual)
docker logs mysql-db

# Step 4: Run WordPress
docker run -d \
  --name wordpress-app \
  --network wordpress-net \
  -e WORDPRESS_DB_HOST=mysql-db:3306 \
  -e WORDPRESS_DB_USER=wpuser \
  -e WORDPRESS_DB_PASSWORD=wppassword \
  -e WORDPRESS_DB_NAME=wordpress \
  -p 8080:80 \
  wordpress:latest

# To stop:
docker stop wordpress-app mysql-db

# To remove:
docker rm wordpress-app mysql-db
docker network rm wordpress-net
docker volume rm mysql-data
```

#### Using Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppassword
    volumes:
      - mysql-data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppassword
      WORDPRESS_DB_NAME: wordpress
    depends_on:
      - db

volumes:
  mysql-data:
```

```bash
# Start everything
docker-compose up -d

# Stop everything
docker-compose down

# Stop and remove volumes
docker-compose down -v
```

---

## docker-compose.yml Structure

```yaml
version: '3.8'  # Compose file version

services:       # Define containers
  service1:
    # Service configuration
  service2:
    # Service configuration

networks:       # Define networks (optional)
  network1:
    # Network configuration

volumes:        # Define volumes (optional)
  volume1:
    # Volume configuration
```

### Service Configuration Options

```yaml
services:
  app:
    # Image to use
    image: nginx:latest
    
    # OR build from Dockerfile
    build:
      context: .
      dockerfile: Dockerfile
    
    # Container name
    container_name: my-app
    
    # Ports mapping
    ports:
      - "8080:80"          # host:container
      - "443:443"
    
    # Environment variables
    environment:
      NODE_ENV: production
      API_KEY: secret123
    
    # OR use env file
    env_file:
      - .env
    
    # Volumes
    volumes:
      - ./app:/usr/share/nginx/html
      - nginx-config:/etc/nginx
    
    # Networks
    networks:
      - frontend
      - backend
    
    # Dependencies
    depends_on:
      - db
      - redis
    
    # Restart policy
    restart: unless-stopped
    
    # Command override
    command: npm start
    
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          memory: 256M
```

---

## Key Concepts

### 1. Services

A **service** is a container definition in your application.

```yaml
services:
  web:        # Service name (becomes hostname)
    image: nginx:latest
  
  api:
    image: node:18
  
  db:
    image: postgres:15
```

Each service:
- Gets its own container
- Can communicate using service names
- Can be scaled independently

### 2. Networks

Docker Compose **automatically creates a network** for your application.

```yaml
# Default - automatic network
services:
  web:
    image: nginx
  api:
    image: node

# Result:
# - Network created: projectname_default
# - web can reach api at hostname: api
# - api can reach web at hostname: web
```

**Custom networks:**

```yaml
services:
  frontend:
    image: nginx
    networks:
      - frontend-net
  
  backend:
    image: node
    networks:
      - frontend-net
      - backend-net
  
  db:
    image: postgres
    networks:
      - backend-net

networks:
  frontend-net:
  backend-net:

# Result:
# frontend ←→ backend (via frontend-net)
# backend ←→ db (via backend-net)
# frontend ✗ db (no shared network)
```

### 3. Volumes

Persist data across container restarts.

```yaml
services:
  db:
    image: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data  # Named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Bind mount

volumes:
  postgres-data:  # Declare named volume
```

### 4. depends_on

Control startup order.

```yaml
services:
  web:
    image: nginx
    depends_on:
      - api
  
  api:
    image: node
    depends_on:
      - db
  
  db:
    image: postgres

# Startup order: db → api → web
```

⚠️ **Important:** `depends_on` only waits for the container to start, **not for the service to be ready**.

**Better approach with health checks:**

```yaml
services:
  db:
    image: postgres
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
  
  api:
    image: node
    depends_on:
      db:
        condition: service_healthy
```

---

## Real-World Example: Microservices E-commerce

```yaml
version: '3.8'

services:
  # Frontend
  web:
    build: ./web
    ports:
      - "3000:3000"
    environment:
      API_URL: http://api:8000
    depends_on:
      - api

  # API Gateway
  api:
    build: ./api
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/ecommerce
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./api/logs:/app/logs

  # Product Service
  product-service:
    build: ./services/product
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/ecommerce
    depends_on:
      - db

  # Order Service
  order-service:
    build: ./services/order
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/ecommerce
      RABBITMQ_URL: amqp://rabbitmq:5672
    depends_on:
      - db
      - rabbitmq

  # Payment Worker
  payment-worker:
    build: ./workers/payment
    environment:
      RABBITMQ_URL: amqp://rabbitmq:5672
      STRIPE_API_KEY: ${STRIPE_API_KEY}
    depends_on:
      - rabbitmq
    deploy:
      replicas: 3  # Run 3 instances

  # Database
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: ecommerce
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

  # Message Queue
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"  # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq

  # Monitoring
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

volumes:
  postgres-data:
  redis-data:
  rabbitmq-data:
  prometheus-data:
```

**Usage:**

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f api

# Scale payment workers
docker-compose up -d --scale payment-worker=5

# Stop everything
docker-compose down

# Stop and remove volumes
docker-compose down -v
```

---

## Docker Commands → Docker Compose

### Starting Containers

**Docker CLI:**
```bash
docker run -d --name web -p 8080:80 nginx
docker run -d --name db -e POSTGRES_PASSWORD=secret postgres
```

**Docker Compose:**
```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
  
  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD: secret
```

```bash
docker-compose up -d
```

### Viewing Logs

**Docker CLI:**
```bash
docker logs -f web
docker logs -f db
```

**Docker Compose:**
```bash
docker-compose logs -f        # All services
docker-compose logs -f web    # Specific service
```

### Stopping Containers

**Docker CLI:**
```bash
docker stop web db
docker rm web db
```

**Docker Compose:**
```bash
docker-compose down
```

### Executing Commands

**Docker CLI:**
```bash
docker exec -it web bash
docker exec db psql -U postgres
```

**Docker Compose:**
```bash
docker-compose exec web bash
docker-compose exec db psql -U postgres
```

### Viewing Running Containers

**Docker CLI:**
```bash
docker ps
```

**Docker Compose:**
```bash
docker-compose ps
```

### Building Images

**Docker CLI:**
```bash
docker build -t myapp .
```

**Docker Compose:**
```yaml
services:
  app:
    build: .
```

```bash
docker-compose build
```

### Port Mapping

**Docker CLI:**
```bash
docker run -p 8080:80 -p 443:443 nginx
```

**Docker Compose:**
```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
      - "443:443"
```

### Environment Variables

**Docker CLI:**
```bash
docker run -e NODE_ENV=production -e API_KEY=secret myapp
```

**Docker Compose:**
```yaml
services:
  app:
    image: myapp
    environment:
      NODE_ENV: production
      API_KEY: secret
```

### Volumes

**Docker CLI:**
```bash
docker run -v /host/path:/container/path nginx
docker run -v named-volume:/data postgres
```

**Docker Compose:**
```yaml
services:
  web:
    image: nginx
    volumes:
      - /host/path:/container/path
  
  db:
    image: postgres
    volumes:
      - named-volume:/data

volumes:
  named-volume:
```

### Networks

**Docker CLI:**
```bash
docker network create mynetwork
docker run --network mynetwork nginx
```

**Docker Compose:**
```yaml
services:
  web:
    image: nginx
    networks:
      - mynetwork

networks:
  mynetwork:
```

---

## Complete Comparison Table

| Task | Docker CLI | Docker Compose |
|------|------------|----------------|
| Start containers | `docker run -d nginx` | `docker-compose up -d` |
| Stop containers | `docker stop container_name` | `docker-compose down` |
| View logs | `docker logs -f container_name` | `docker-compose logs -f service_name` |
| Execute command | `docker exec -it container_name bash` | `docker-compose exec service_name bash` |
| List containers | `docker ps` | `docker-compose ps` |
| Build image | `docker build -t name .` | `docker-compose build` |
| Pull images | `docker pull image:tag` | `docker-compose pull` |
| Remove containers | `docker rm container_name` | `docker-compose down` |
| Scale services | N/A (run multiple times) | `docker-compose up -d --scale service=3` |
| Update containers | `docker stop/rm/run` | `docker-compose up -d` |

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Docker Compose allows you to define multi-container applications using a __________ file.
2. In docker-compose.yml, each container definition is called a __________.
3. The __________ field controls the startup order of services.
4. Docker Compose automatically creates a __________ for all services.
5. To start all services in detached mode, use the command __________.

### True/False

1. ⬜ Docker Compose can only run containers from Docker Hub
2. ⬜ Services in docker-compose.yml can communicate using service names as hostnames
3. ⬜ depends_on waits for a service to be fully ready before starting the next service
4. ⬜ Docker Compose automatically creates a network for your application
5. ⬜ You need to manually create volumes before using them in docker-compose.yml
6. ⬜ docker-compose down removes volumes by default
7. ⬜ Each service in docker-compose.yml runs in its own container

### Multiple Choice

1. What is the primary benefit of Docker Compose over Docker CLI?
   - A) Faster container startup
   - B) Better security
   - C) Managing multi-container applications with one file
   - D) Automatic scaling

2. Which command starts all services defined in docker-compose.yml?
   - A) docker-compose start
   - B) docker-compose run
   - C) docker-compose up
   - D) docker-compose create

3. How do services communicate in Docker Compose?
   - A) Using IP addresses
   - B) Using container IDs
   - C) Using service names as hostnames
   - D) Using localhost

4. What does docker-compose down do?
   - A) Stops containers only
   - B) Stops and removes containers and networks
   - C) Stops and removes containers, networks, and volumes
   - D) Pauses all containers

5. Where should you store sensitive data like passwords in Docker Compose?
   - A) Directly in docker-compose.yml
   - B) In environment files (.env)
   - C) In service names
   - D) In volume names

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. YAML (or docker-compose.yml)
2. service
3. depends_on
4. network (or default network)
5. `docker-compose up -d`

**True/False:**
1. ❌ False - Can build custom images using the build directive
2. ✅ True - Service names become DNS hostnames automatically
3. ❌ False - It only waits for container start, not for service readiness
4. ✅ True - Creates a default network automatically
5. ❌ False - Compose creates volumes automatically when referenced
6. ❌ False - Use `docker-compose down -v` to remove volumes
7. ✅ True - Each service definition creates one container (unless scaled)

**Multiple Choice:**
1. **C** - Managing multi-container applications with one file
2. **C** - docker-compose up (or docker-compose up -d for detached)
3. **C** - Using service names as hostnames
4. **B** - Stops and removes containers and networks (not volumes by default)
5. **B** - In environment files (.env) or use secrets management

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the difference between `docker run` and `docker-compose up`

<details>
<summary><strong>View Answer</strong></summary>

**docker run** - Runs a **single container** with specified parameters

**docker-compose up** - Starts **multiple containers** defined in docker-compose.yml

**Key Differences:**

```
docker run:
- One container at a time
- Parameters passed via CLI flags
- Manual network/volume creation
- No dependency management
- Hard to reproduce setup

docker-compose up:
- Multiple containers together
- Configuration in YAML file
- Automatic network/volume creation
- Handles dependencies
- Easy to version control and share
```

**Example:**

```bash
# docker run approach (manual, error-prone)
docker network create app-net
docker run -d --name db --network app-net postgres
docker run -d --name api --network app-net -e DB_HOST=db myapi
docker run -d --name web --network app-net -p 80:80 myweb

# docker-compose approach (declarative, reproducible)
docker-compose up -d  # Reads docker-compose.yml, starts everything
```

**When to use each:**
- Use `docker run` for: Quick tests, one-off containers, simple setups
- Use `docker-compose` for: Multi-container apps, development, consistent environments

</details>

### Question 2: Why doesn't `depends_on` guarantee service readiness?

<details>
<summary><strong>View Answer</strong></summary>

**Problem:** `depends_on` only ensures container **start order**, not **readiness**.

```yaml
services:
  api:
    image: myapi
    depends_on:
      - db  # Waits for db container to START
  
  db:
    image: postgres
```

**What happens:**

```
1. Docker starts db container
2. Container state = "running"
3. Docker immediately starts api container
4. PostgreSQL is still initializing (not ready)
5. API tries to connect → connection refused
6. API crashes or retries in loop
```

**PostgreSQL startup timeline:**
```
0s:  Container starts → depends_on satisfied
2s:  PostgreSQL initializing database
5s:  Loading configuration
8s:  Creating default database
10s: Ready to accept connections ← API should connect now
```

**Solution: Use health checks**

```yaml
services:
  db:
    image: postgres
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
  
  api:
    image: myapi
    depends_on:
      db:
        condition: service_healthy  # Waits for healthy state
```

**Alternative: Application-level retry logic**

```javascript
// In API code
const connectWithRetry = async () => {
  for (let i = 0; i < 10; i++) {
    try {
      await db.connect();
      console.log('Connected to database');
      return;
    } catch (err) {
      console.log(`Retry ${i + 1}/10...`);
      await sleep(2000);
    }
  }
  throw new Error('Could not connect to database');
};
```

**Real-world impact:**
Without health checks, your application might fail during startup in production, especially when:
- Database takes time to initialize
- Service has slow startup
- Running on resource-constrained systems

</details>

### Question 3: How would you convert this Docker CLI setup to Docker Compose?

```bash
docker network create blog-network

docker run -d \
  --name mysql-db \
  --network blog-network \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wpuser \
  -e MYSQL_PASSWORD=wppass \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0

docker run -d \
  --name wordpress-site \
  --network blog-network \
  -e WORDPRESS_DB_HOST=mysql-db:3306 \
  -e WORDPRESS_DB_USER=wpuser \
  -e WORDPRESS_DB_PASSWORD=wppass \
  -e WORDPRESS_DB_NAME=wordpress \
  -p 8080:80 \
  -v wordpress-data:/var/www/html \
  wordpress:latest
```

<details>
<summary><strong>View Answer</strong></summary>

**Docker Compose version:**

```yaml
version: '3.8'

services:
  db:
    image: mysql:8.0
    container_name: mysql-db
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - blog-network

  wordpress:
    image: wordpress:latest
    container_name: wordpress-site
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress-data:/var/www/html
    depends_on:
      - db
    networks:
      - blog-network

volumes:
  mysql-data:
  wordpress-data:

networks:
  blog-network:
```

**Key mappings:**

| Docker CLI | Docker Compose |
|------------|----------------|
| `--name mysql-db` | `container_name: mysql-db` |
| `--network blog-network` | `networks: - blog-network` |
| `-e VAR=value` | `environment: VAR: value` |
| `-v mysql-data:/path` | `volumes: - mysql-data:/path` |
| `-p 8080:80` | `ports: - "8080:80"` |
| `mysql-db:3306` (hostname) | `db:3306` (service name) |

**Usage:**

```bash
# Start
docker-compose up -d

# Stop
docker-compose down

# Stop and remove volumes
docker-compose down -v
```

**Benefits of Compose version:**
- ✅ Single file to manage
- ✅ Version controlled
- ✅ One command to start/stop
- ✅ Easy to share with team
- ✅ Automatic network creation
- ✅ Dependency management with depends_on

</details>

### Question 4: When should you use Docker Compose vs Kubernetes?

<details>
<summary><strong>View Answer</strong></summary>

**Use Docker Compose for:**

**1. Local Development**
```
Developer machine:
- Multiple services needed (DB, cache, API)
- Quick start/stop
- Easy debugging
- Configuration changes frequent

Example: Developer running entire e-commerce stack locally
```

**2. Single Host Deployments**
```
Small application on one server:
- Blog with WordPress + MySQL
- Internal tool with simple stack
- Proof of concept
- Testing environment

Example: Company internal documentation site
```

**3. Simple Applications**
```
Characteristics:
- Few services (< 10)
- No complex scaling needs
- Single server sufficient
- Team size < 10 developers

Example: Startup MVP with web + API + database
```

**Use Kubernetes for:**

**1. Production at Scale**
```
Characteristics:
- Multiple servers needed
- High availability required
- Thousands of requests/second
- Global distribution

Example: Netflix streaming platform
```

**2. Auto-Scaling Required**
```
Traffic patterns:
- Black Friday (10x traffic)
- Viral events
- Time-zone based peaks
- Need automatic scale up/down

Example: E-commerce during sales events
```

**3. Complex Deployments**
```
Requirements:
- Blue-green deployments
- Canary releases
- Rolling updates
- Zero-downtime deployments

Example: Banking application (can't have downtime)
```

**4. Multi-Cloud/Hybrid**
```
Infrastructure:
- AWS + GCP + On-premise
- Disaster recovery across regions
- Vendor lock-in prevention

Example: Enterprise with regulatory requirements
```

**Comparison Table:**

| Feature | Docker Compose | Kubernetes |
|---------|---------------|-----------|
| **Complexity** | Simple | Complex |
| **Learning curve** | Hours | Weeks/Months |
| **Setup time** | Minutes | Days |
| **Scaling** | Manual | Automatic |
| **High availability** | No | Yes |
| **Multi-host** | No | Yes |
| **Self-healing** | Limited | Advanced |
| **Load balancing** | Basic | Advanced |
| **Rolling updates** | Manual | Built-in |
| **Resource limits** | Per service | Per pod + namespace |
| **Best for** | Dev/Small apps | Production/Large scale |

**Real-World Example:**

```
Startup Journey:

Phase 1 (Months 0-6): Docker Compose
- 3 developers
- 1 server
- Simple web app
→ Docker Compose perfect

Phase 2 (Months 6-12): Still Docker Compose
- 10 developers
- 2 servers (manual scaling)
- Growing user base
→ Docker Compose still works

Phase 3 (Months 12-24): Migrate to Kubernetes
- 50 developers
- Need auto-scaling
- Multi-region deployment
- 1 million users
→ Kubernetes becomes necessary
```

**Decision Framework:**

Ask yourself:
1. Running on single server? → Compose
2. Need auto-scaling? → Kubernetes
3. High availability required? → Kubernetes
4. Team familiar with Compose? → Start with Compose
5. Managing 100+ containers? → Kubernetes
6. Local development? → Compose

**Summary:**
- Docker Compose: Development and small production deployments
- Kubernetes: Large-scale production with complex requirements
- Many companies use BOTH (Compose for dev, K8s for production)

</details>

### Question 5: Your docker-compose.yml has 10 services, but only 3 need to be accessible from the host. How would you configure this?

<details>
<summary><strong>View Answer</strong></summary>

**Scenario:**
```
E-commerce Application:
Public (need host access):
- Web frontend (port 80)
- API gateway (port 8000)
- Admin panel (port 3000)

Internal (no host access needed):
- User service
- Product service
- Order service
- Payment service
- Database
- Redis
- RabbitMQ
```

**Solution:**

```yaml
version: '3.8'

services:
  # Public services - expose ports
  web:
    image: myapp/frontend
    ports:
      - "80:80"      # Accessible from host
    networks:
      - frontend

  api-gateway:
    image: myapp/api-gateway
    ports:
      - "8000:8000"  # Accessible from host
    networks:
      - frontend
      - backend

  admin:
    image: myapp/admin
    ports:
      - "3000:3000"  # Accessible from host
    networks:
      - backend

  # Internal services - NO ports exposed
  user-service:
    image: myapp/user-service
    # No ports! Only accessible via Docker network
    networks:
      - backend

  product-service:
    image: myapp/product-service
    networks:
      - backend

  order-service:
    image: myapp/order-service
    networks:
      - backend

  payment-service:
    image: myapp/payment-service
    networks:
      - backend

  db:
    image: postgres:15
    # No ports! Only accessible from containers
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend

  redis:
    image: redis:7
    networks:
      - backend

  rabbitmq:
    image: rabbitmq:3
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  db-data:
```

**Why this works:**

```
Host Machine:
├── Can access: web:80, api:8000, admin:3000
└── Cannot access: database, redis, internal services

Docker Network (internal):
├── All services can communicate
├── api-gateway → user-service ✓
├── order-service → database ✓
└── payment-service → rabbitmq ✓

Security benefits:
- Database not exposed to internet
- Internal services protected
- Reduced attack surface
```

**Access patterns:**

```
External users:
http://host:80 → web (frontend)
http://host:8000 → api-gateway

web → api-gateway (via Docker network)
api-gateway → user-service (via Docker network)
user-service → db (via Docker network)

External ✗ db (no port exposed)
External ✗ redis (no port exposed)
```

**For development debugging:**

If you need temporary access to database:

```yaml
  db:
    image: postgres:15
    ports:
      - "5432:5432"  # Add temporarily for debugging
    # Remove before deploying to production!
```

**Best practice:**
- Only expose ports that must be publicly accessible
- Use Docker networks for inter-service communication
- Never expose databases or internal services in production
- Use separate networks for frontend and backend isolation

**Production tip:**

```yaml
services:
  db:
    ports:
      - "127.0.0.1:5432:5432"  # Only localhost, not 0.0.0.0
    # Accessible only from host machine, not from network
```

</details>

</details>

---

[Next: 8.2 Compose Services →](02-compose-services.md)