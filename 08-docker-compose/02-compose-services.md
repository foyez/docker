# 8.2 Compose Services

Advanced service configuration in Docker Compose: scaling, resource management, health checks, and production patterns.

---

## Service Configuration

### Image vs Build

**Using Pre-built Image:**
```yaml
services:
  web:
    image: nginx:1.25
    ports:
      - "80:80"
```

**Building from Source:**
```yaml
services:
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
```

**Both (Build + Tag):**
```yaml
services:
  app:
    build: ./app
    image: myapp:latest
    ports:
      - "3000:3000"

# Builds and tags as myapp:latest
# Can push to registry later
```

---

## Container Naming

**Default Naming:**
```yaml
services:
  web:
    image: nginx

# Container name: projectname-web-1
# Project name from directory or -p flag
```

**Custom Name:**
```yaml
services:
  web:
    image: nginx
    container_name: my-web-server

# Exact name: my-web-server
# Warning: Cannot scale when using container_name
```

---

## Commands and Entrypoints

**Override Command:**
```yaml
services:
  app:
    image: node:18
    command: npm start

  # Multiple arguments
  worker:
    image: node:18
    command: ["npm", "run", "worker"]

  # Shell form
  script:
    image: alpine
    command: sh -c "echo hello && sleep 30"
```

**Override Entrypoint:**
```yaml
services:
  app:
    image: myapp
    entrypoint: /app/custom-entrypoint.sh
    command: ["--config", "/etc/app.conf"]

  # Disable entrypoint
  debug:
    image: myapp
    entrypoint: ""
    command: /bin/sh
```

---

## Working Directory

```yaml
services:
  app:
    image: node:18
    working_dir: /app
    volumes:
      - ./code:/app
    command: npm start

# Commands run from /app directory
```

---

## User and Permissions

**Run as Specific User:**
```yaml
services:
  app:
    image: node:18
    user: "1000:1000"  # UID:GID

  # By username
  postgres:
    image: postgres
    user: postgres

  # Root
  admin:
    image: alpine
    user: root
```

---

## Environment Variables

### Inline Variables

```yaml
services:
  app:
    image: myapp
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgresql://db:5432/myapp
```

### From File

```yaml
services:
  app:
    image: myapp
    env_file:
      - common.env
      - production.env

  # Multiple files (later override earlier)
```

### Variable Substitution

```yaml
services:
  app:
    image: myapp:${VERSION:-latest}
    environment:
      PORT: ${APP_PORT:-3000}
```

---

## Resource Limits

### Memory Limits

```yaml
services:
  app:
    image: myapp
    mem_limit: 512m
    mem_reservation: 256m

  # Alternative syntax
  database:
    image: postgres
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M
```

### CPU Limits

```yaml
services:
  app:
    image: myapp
    cpus: 0.5  # 50% of one CPU

  # Alternative
  worker:
    image: myworker
    deploy:
      resources:
        limits:
          cpus: '2.0'
        reservations:
          cpus: '0.5'
```

### Combined Example

```yaml
services:
  api:
    image: myapi
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
```

---

## Health Checks

**Basic Health Check:**
```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

**Different Test Types:**
```yaml
services:
  # HTTP endpoint
  api:
    image: myapi
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s

  # Database
  postgres:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5

  # Redis
  redis:
    image: redis
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

  # Custom script
  app:
    image: myapp
    healthcheck:
      test: ["CMD", "/app/healthcheck.sh"]
      interval: 30s
```

**Disable Health Check:**
```yaml
services:
  app:
    image: myapp
    healthcheck:
      disable: true
```

---

## Restart Policies

```yaml
services:
  # Never restart
  temporary:
    image: alpine
    restart: "no"

  # Restart on failure
  worker:
    image: myworker
    restart: on-failure

  # Always restart
  critical:
    image: mycritical
    restart: always

  # Restart unless stopped manually
  webapp:
    image: myapp
    restart: unless-stopped
```

**With Max Retries:**
```yaml
services:
  app:
    image: myapp
    restart: on-failure:5  # Max 5 retries
```

---

## Networking

### Multiple Networks

```yaml
services:
  web:
    image: nginx
    networks:
      - frontend
      - backend

  api:
    image: myapi
    networks:
      - backend

  db:
    image: postgres
    networks:
      - backend

networks:
  frontend:
  backend:
```

### Network Aliases

```yaml
services:
  api:
    image: myapi
    networks:
      backend:
        aliases:
          - api-server
          - backend-api

# Other containers can reach at:
# - api (service name)
# - api-server (alias)
# - backend-api (alias)
```

### Static IP

```yaml
services:
  web:
    image: nginx
    networks:
      frontend:
        ipv4_address: 172.20.0.10

networks:
  frontend:
    ipam:
      config:
        - subnet: 172.20.0.0/24
```

---

## Volumes

### Named Volumes

```yaml
services:
  db:
    image: postgres
    volumes:
      - db-data:/var/lib/postgresql/data
      - db-logs:/var/log/postgresql

volumes:
  db-data:
  db-logs:
```

### Volume Options

```yaml
services:
  app:
    image: myapp
    volumes:
      # Read-only
      - config:/etc/app:ro
      
      # Read-write (default)
      - data:/app/data:rw
      
      # Cached (macOS performance)
      - ./code:/app:cached
      
      # Delegated (macOS performance)
      - ./build:/app/build:delegated

volumes:
  config:
  data:
```

### Volume Driver

```yaml
volumes:
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.1,rw
      device: ":/path/to/share"
```

---

## Ports

**Short Syntax:**
```yaml
services:
  web:
    image: nginx
    ports:
      - "80:80"           # HOST:CONTAINER
      - "443:443"
      - "8080:80"         # Different host port
      - "127.0.0.1:8000:8000"  # Specific interface
```

**Long Syntax:**
```yaml
services:
  web:
    image: nginx
    ports:
      - target: 80
        published: 8080
        protocol: tcp
        mode: host

      - target: 443
        published: 8443
```

**Port Ranges:**
```yaml
services:
  app:
    image: myapp
    ports:
      - "3000-3005:3000-3005"
```

**Expose (Internal Only):**
```yaml
services:
  api:
    image: myapi
    expose:
      - "8000"  # Available to linked services only
```

---

## Dependencies

**Basic Dependency:**
```yaml
services:
  web:
    image: nginx
    depends_on:
      - api

  api:
    image: myapi
    depends_on:
      - db

  db:
    image: postgres
```

**With Conditions:**
```yaml
services:
  web:
    depends_on:
      api:
        condition: service_healthy
      db:
        condition: service_started

  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s

  db:
    image: postgres
```

---

## Scaling Services

**Command Line:**
```bash
# Scale specific service
docker compose up -d --scale worker=3

# Multiple services
docker compose up -d --scale worker=3 --scale api=2
```

**In Compose File:**
```yaml
services:
  worker:
    image: myworker
    deploy:
      replicas: 3

# Creates 3 worker containers
```

**Load Balancing:**
```yaml
services:
  web:
    image: nginx
    ports:
      - "80:80"

  api:
    image: myapi
    deploy:
      replicas: 3
    # No port mapping! Only internal

# Nginx load balances to 3 api replicas
# via service name 'api'
```

---

## Labels

**Container Labels:**
```yaml
services:
  web:
    image: nginx
    labels:
      com.example.description: "Web server"
      com.example.version: "1.0"
      com.example.team: "frontend"
```

**Image Labels:**
```yaml
services:
  app:
    build: .
    labels:
      com.example.build-date: "2024-01-17"
      com.example.vcs-ref: "abc123"
```

---

## Logging

**Driver and Options:**
```yaml
services:
  web:
    image: nginx
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  app:
    image: myapp
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://192.168.1.1:514"
```

---

## Complete Production Example

```yaml
version: '3.8'

services:
  # Nginx Load Balancer
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    networks:
      - frontend
    depends_on:
      - api
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  # API Servers (3 replicas)
  api:
    build:
      context: ./api
      dockerfile: Dockerfile.prod
    image: myapi:latest
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://db:5432/myapp
      REDIS_URL: redis://redis:6379
    env_file:
      - ./config/api.env
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
      migrations:
        condition: service_completed_successfully
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "node", "/app/healthcheck.js"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

  # Database Migrations
  migrations:
    image: myapi:latest
    command: npm run migrate
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
    env_file:
      - ./config/api.env
    networks:
      - backend
    depends_on:
      db:
        condition: service_healthy
    restart: on-failure

  # PostgreSQL Database
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
    env_file:
      - ./config/db.env
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  # Redis Cache
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: "5m"
        max-file: "3"

  # Background Workers
  worker:
    image: myapi:latest
    command: npm run worker
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    environment:
      NODE_ENV: production
      REDIS_URL: redis://redis:6379
    env_file:
      - ./config/api.env
    networks:
      - backend
    depends_on:
      redis:
        condition: service_healthy
      db:
        condition: service_healthy
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true

volumes:
  db-data:
  redis-data:
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The __________ field specifies resource limits for a service.
2. Use __________ to define a container health check in Compose.
3. The __________ restart policy restarts containers unless manually stopped.
4. __________ creates multiple instances of a service.
5. Service __________ provide alternative DNS names on networks.
6. The __________ field sets the working directory in the container.

### True/False

1. ⬜ You can scale services that have container_name defined
2. ⬜ Health checks require curl to be installed in the image
3. ⬜ depends_on with service_healthy waits for health check to pass
4. ⬜ Multiple services can use the same container_name
5. ⬜ Resource limits are enforced by Docker
6. ⬜ Services can be on multiple networks simultaneously
7. ⬜ The restart policy applies to docker compose down

### Multiple Choice

1. Which restart policy is best for production?
   - A) no
   - B) on-failure
   - C) always
   - D) unless-stopped

2. What does mem_limit: 512m do?
   - A) Sets minimum memory
   - B) Sets maximum memory
   - C) Reserves memory
   - D) Nothing

3. How to run 3 worker instances?
   - A) docker compose up worker worker worker
   - B) docker compose up --scale worker=3
   - C) docker compose scale worker=3
   - D) docker compose replicas worker=3

4. Health check interval default?
   - A) 10s
   - B) 30s
   - C) 60s
   - D) No default

5. What does expose do?
   - A) Publishes port to host
   - B) Makes port available to linked services
   - C) Nothing
   - D) Same as ports

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. deploy (or resources)
2. healthcheck
3. unless-stopped
4. Scaling (or replicas, deploy)
5. aliases
6. working_dir

**True/False:**
1. ❌ False - Cannot scale with container_name (names must be unique)
2. ❌ False - Can use any command, not just curl
3. ✅ True - Waits for health check to pass
4. ❌ False - Container names must be unique
5. ✅ True - Docker enforces resource limits
6. ✅ True - Services can join multiple networks
7. ❌ False - down stops and removes, restart policy doesn't apply

**Multiple Choice:**
1. **D** - unless-stopped (respects manual stops)
2. **B** - Sets maximum memory
3. **B** - docker compose up --scale worker=3
4. **B** - 30s (default interval)
5. **B** - Makes port available to linked services only

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: How do you optimize Compose for production deployment?

<details>
<summary><strong>View Answer</strong></summary>

**Production Optimization Strategies:**

---

### 1. Resource Management

**Set Resource Limits:**
```yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M

# Prevents resource exhaustion
# Ensures fair resource allocation
```

**Why:**
```
✓ Prevents single service consuming all resources
✓ Predictable performance
✓ Better stability
✓ Protects against memory leaks
```

---

### 2. Health Checks

**Implement Comprehensive Health Checks:**
```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "node", "/app/healthcheck.js"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5
```

**Custom Health Check Script:**
```javascript
// healthcheck.js
const http = require('http');

const options = {
  host: 'localhost',
  port: 3000,
  path: '/health',
  timeout: 2000
};

const req = http.request(options, (res) => {
  process.exit(res.statusCode === 200 ? 0 : 1);
});

req.on('error', () => process.exit(1));
req.on('timeout', () => process.exit(1));
req.end();
```

---

### 3. Restart Policies

```yaml
services:
  # Critical services
  api:
    restart: unless-stopped
  
  # Background jobs
  worker:
    restart: on-failure:5
  
  # One-time tasks
  migrations:
    restart: "no"
```

---

### 4. Logging Configuration

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
        labels: "service,environment"
        env: "NODE_ENV"
```

**Or use centralized logging:**
```yaml
services:
  api:
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://logs.example.com:514"
        tag: "api"
```

---

### 5. Security Hardening

**Use Non-Root User:**
```yaml
services:
  api:
    image: myapi
    user: "1000:1000"
```

**Read-Only Root Filesystem:**
```yaml
services:
  api:
    image: myapi
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

**Limit Capabilities:**
```yaml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

---

### 6. Environment Separation

**Base Configuration:**
```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    image: myapi:${VERSION}
    environment:
      NODE_ENV: ${NODE_ENV}
```

**Production Override:**
```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  api:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
    restart: unless-stopped
```

**Usage:**
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

### 7. Image Optimization

**Use Specific Tags:**
```yaml
services:
  api:
    image: node:18.19.0-alpine3.19  # Not 'latest'
  
  db:
    image: postgres:15.5-alpine  # Specific version
```

**Multi-Stage Builds:**
```dockerfile
# Dockerfile
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
USER node
CMD ["node", "server.js"]
```

```yaml
services:
  api:
    build:
      context: .
      target: production
    image: myapi:${VERSION}
```

---

### 8. Network Segmentation

```yaml
services:
  nginx:
    networks:
      - frontend

  api:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend

networks:
  frontend:
  backend:
    internal: true  # No internet access
```

---

### 9. Secrets Management

```yaml
services:
  api:
    env_file:
      - ./config/common.env
      - ./config/production.env
    environment:
      DATABASE_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

### 10. Monitoring and Observability

```yaml
services:
  api:
    labels:
      prometheus.scrape: "true"
      prometheus.port: "9090"
      prometheus.path: "/metrics"

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

---

### Complete Production Example

```yaml
version: '3.8'

services:
  api:
    build:
      context: ./api
      dockerfile: Dockerfile.prod
    image: myapi:${VERSION:-latest}
    user: "1000:1000"
    read_only: true
    tmpfs:
      - /tmp
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.25'
          memory: 256M
      restart_policy:
        condition: unless-stopped
        delay: 5s
        max_attempts: 3
        window: 120s
    environment:
      NODE_ENV: production
    env_file:
      - ./config/production.env
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "node", "/app/healthcheck.js"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
    labels:
      com.example.service: "api"
      prometheus.scrape: "true"

  db:
    image: postgres:15.5-alpine
    user: postgres
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
    environment:
      POSTGRES_DB: myapp
    env_file:
      - ./config/db.env
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

networks:
  frontend:
  backend:
    internal: true

volumes:
  db-data:
    driver: local
```

---

### Deployment Checklist

```
Before Production:
☐ Set specific image versions (no :latest)
☐ Configure resource limits
☐ Implement health checks
☐ Set restart policies
☐ Configure logging (rotation)
☐ Secure secrets (no plaintext)
☐ Network segmentation
☐ Use non-root users
☐ Enable monitoring
☐ Test disaster recovery
☐ Document configuration
☐ Set up backups
☐ Configure alerts
```

</details>

### Question 2: How do you handle database migrations in Docker Compose?

<details>
<summary><strong>View Answer</strong></summary>

**Database Migration Strategies:**

---

### Method 1: Separate Migration Service

```yaml
version: '3.8'

services:
  # Database
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5

  # Migrations (runs once)
  migrations:
    image: myapp:latest
    command: npm run migrate
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
    depends_on:
      db:
        condition: service_healthy
    restart: on-failure

  # Application
  api:
    image: myapp:latest
    command: npm start
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
    depends_on:
      migrations:
        condition: service_completed_successfully
    ports:
      - "3000:3000"

volumes:
  db-data:
```

**How it works:**
```
1. db starts
2. Health check passes
3. migrations runs
4. migrations completes (exit 0)
5. api starts
```

---

### Method 2: Init Container Pattern

```yaml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s

  # Init container
  db-init:
    image: myapp:latest
    command: |
      sh -c '
        echo "Waiting for database..."
        npm run wait-for-db
        echo "Running migrations..."
        npm run migrate
        echo "Seeding database..."
        npm run seed
        echo "Database ready!"
      '
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  api:
    image: myapp:latest
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
    depends_on:
      db-init:
        condition: service_completed_successfully
    ports:
      - "3000:3000"
```

---

### Method 3: Entrypoint Script

**entrypoint.sh:**
```bash
#!/bin/bash
set -e

# Wait for database
echo "Waiting for database..."
while ! nc -z db 5432; do
  sleep 1
done

echo "Database is ready!"

# Run migrations
if [ "$RUN_MIGRATIONS" = "true" ]; then
  echo "Running migrations..."
  npm run migrate
  
  if [ $? -eq 0 ]; then
    echo "Migrations successful"
  else
    echo "Migrations failed"
    exit 1
  fi
fi

# Start application
exec "$@"
```

**Dockerfile:**
```dockerfile
FROM node:18

WORKDIR /app
COPY package*.json ./
RUN npm ci --production

COPY . .
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "server.js"]
```

**docker-compose.yml:**
```yaml
services:
  api:
    build: .
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
      RUN_MIGRATIONS: "true"
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
```

---

### Method 4: Application-Level Migration

**Auto-migrate on startup:**

```javascript
// server.js
const { migrate } = require('./migrations');

async function start() {
  try {
    // Run migrations
    console.log('Running migrations...');
    await migrate();
    console.log('Migrations complete');
    
    // Start server
    const app = require('./app');
    app.listen(3000, () => {
      console.log('Server running on port 3000');
    });
    
  } catch (error) {
    console.error('Startup failed:', error);
    process.exit(1);
  }
}

start();
```

```yaml
services:
  api:
    image: myapp
    command: node server.js
    depends_on:
      db:
        condition: service_healthy
```

---

### Method 5: Manual Migration

**For production safety:**

```yaml
services:
  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data

  # Application (no auto-migrate)
  api:
    image: myapp:latest
    command: node server.js
    environment:
      DATABASE_URL: postgresql://db:5432/myapp

volumes:
  db-data:
```

**Manual migration:**
```bash
# Run migration manually
docker compose run --rm api npm run migrate

# Then start services
docker compose up -d
```

---

### Method 6: CI/CD Pipeline Migration

**GitLab CI:**
```yaml
# .gitlab-ci.yml
stages:
  - build
  - migrate
  - deploy

build:
  stage: build
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myapp:$CI_COMMIT_SHA

migrate:
  stage: migrate
  script:
    - docker compose run --rm migrations npm run migrate
  only:
    - main

deploy:
  stage: deploy
  script:
    - docker compose up -d api
  only:
    - main
```

---

### Method 7: Rolling Migrations

**Backward compatible migrations:**

```yaml
services:
  # Old version (v1)
  api-v1:
    image: myapp:v1
    deploy:
      replicas: 2

  # New version (v2)
  api-v2:
    image: myapp:v2
    deploy:
      replicas: 1

  # Migration service
  migrations:
    image: myapp:v2
    command: npm run migrate
    depends_on:
      db:
        condition: service_healthy
    restart: on-failure
```

**Process:**
```
1. Deploy v2 (1 replica) with migrations
2. Migrations run (backward compatible)
3. Test v2
4. Scale up v2
5. Scale down v1
6. Remove v1
```

---

### Best Practices

**1. Idempotent Migrations:**
```sql
-- Good: Can run multiple times
CREATE TABLE IF NOT EXISTS users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL
);

-- Bad: Fails on second run
CREATE TABLE users (
  id SERIAL PRIMARY KEY
);
```

**2. Separate Up/Down:**
```javascript
// migrations/001_create_users.js
exports.up = async (db) => {
  await db.query(`
    CREATE TABLE users (
      id SERIAL PRIMARY KEY,
      email VARCHAR(255) UNIQUE NOT NULL
    )
  `);
};

exports.down = async (db) => {
  await db.query(`DROP TABLE users`);
};
```

**3. Transaction Support:**
```javascript
async function migrate() {
  const client = await db.connect();
  
  try {
    await client.query('BEGIN');
    
    // Run migrations
    await runMigrations(client);
    
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

**4. Version Tracking:**
```sql
CREATE TABLE schema_migrations (
  version VARCHAR(255) PRIMARY KEY,
  applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**5. Health Checks After Migration:**
```yaml
services:
  migrations:
    image: myapp
    command: |
      sh -c '
        npm run migrate && 
        npm run verify-schema
      '
    healthcheck:
      test: ["CMD", "npm", "run", "verify-schema"]
```

---

### Error Handling

```yaml
services:
  migrations:
    image: myapp
    command: npm run migrate
    restart: on-failure:3  # Retry up to 3 times
    depends_on:
      db:
        condition: service_healthy

  api:
    depends_on:
      migrations:
        condition: service_completed_successfully
```

**Migration script with retry:**
```javascript
async function migrateWithRetry(maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      await migrate();
      console.log('Migrations successful');
      return;
    } catch (error) {
      console.error(`Migration attempt ${i + 1} failed:`, error);
      
      if (i === maxRetries - 1) {
        throw error;
      }
      
      await new Promise(resolve => setTimeout(resolve, 5000));
    }
  }
}
```

---

### Rollback Strategy

```bash
#!/bin/bash
# rollback.sh

# Get current version
CURRENT=$(docker compose exec db psql -U postgres -d myapp -t -c \
  "SELECT version FROM schema_migrations ORDER BY applied_at DESC LIMIT 1")

echo "Current version: $CURRENT"
echo "Rolling back..."

# Run rollback
docker compose run --rm api npm run migrate:rollback

# Verify
NEW=$(docker compose exec db psql -U postgres -d myapp -t -c \
  "SELECT version FROM schema_migrations ORDER BY applied_at DESC LIMIT 1")

echo "Rolled back to: $NEW"
```

</details>

### Question 3: How do you debug issues in Docker Compose applications?

<details>
<summary><strong>View Answer</strong></summary>

**Debugging Strategies:**

---

### 1. Check Service Status

```bash
# View all services
docker compose ps

# Output shows:
# NAME      STATUS       PORTS
# web-1     Up 5 min     0.0.0.0:80->80/tcp
# api-1     Restarting   
# db-1      Up (healthy) 5432/tcp

# Identify problematic services
# api-1 is restarting → investigate
```

---

### 2. View Logs

**All services:**
```bash
docker compose logs
```

**Specific service:**
```bash
docker compose logs api
```

**Follow logs:**
```bash
docker compose logs -f api

# Tail last 100 lines
docker compose logs --tail=100 -f api

# With timestamps
docker compose logs -t api

# Multiple services
docker compose logs api db
```

---

### 3. Inspect Container

```bash
# Get container ID
docker compose ps -q api

# Inspect
docker inspect api-1

# Specific info
docker inspect api-1 --format='{{.State.Status}}'
docker inspect api-1 --format='{{.State.ExitCode}}'
docker inspect api-1 --format='{{json .State}}' | jq
```

---

### 4. Execute Commands

**Interactive shell:**
```bash
docker compose exec api sh
docker compose exec api bash

# As root
docker compose exec -u root api bash
```

**One-off commands:**
```bash
# Check environment
docker compose exec api env

# Check network
docker compose exec api ping db

# Check files
docker compose exec api ls -la /app

# Check process
docker compose exec api ps aux

# Check ports
docker compose exec api netstat -tlnp
```

---

### 5. Check Network Connectivity

```bash
# Ping another service
docker compose exec api ping db

# DNS resolution
docker compose exec api nslookup db

# Port connectivity
docker compose exec api nc -zv db 5432

# HTTP request
docker compose exec api curl http://web:80
```

---

### 6. Validate Configuration

```bash
# Validate docker-compose.yml
docker compose config

# View merged configuration
docker compose -f docker-compose.yml -f docker-compose.prod.yml config

# Check for errors
docker compose config --quiet
```

---

### 7. Common Issues

**Issue 1: Service Won't Start**

```bash
# Check logs
docker compose logs api

# Common causes:
# - Port already in use
# - Missing environment variables
# - Dependency not ready
# - Image pull failure

# Check exit code
docker inspect api-1 --format='{{.State.ExitCode}}'

# 0   = Success
# 1   = Application error
# 127 = Command not found
# 137 = Out of memory (SIGKILL)
# 143 = Terminated (SIGTERM)
```

**Issue 2: Cannot Connect to Database**

```bash
# Check database running
docker compose ps db

# Check database health
docker compose exec db pg_isready -U postgres

# Check connection from API
docker compose exec api ping db

# Check environment variables
docker compose exec api env | grep DATABASE

# Try direct connection
docker compose exec api psql postgresql://db:5432/myapp
```

**Issue 3: Volume Permission Issues**

```bash
# Check volume permissions
docker compose exec api ls -la /app

# Check user
docker compose exec api whoami
docker compose exec api id

# Fix permissions (temporary)
docker compose exec -u root api chown -R appuser:appuser /app

# Permanent fix in Dockerfile
USER appuser
```

---

### 8. Debug Mode

**Enable debug output:**
```bash
# Compose debug
docker compose --verbose up

# Docker debug
export DOCKER_BUILDKIT=0
export COMPOSE_DOCKER_CLI_BUILD=0
docker compose up
```

---

### 9. Rebuild and Recreate

```bash
# Rebuild images
docker compose build --no-cache

# Recreate containers
docker compose up -d --force-recreate

# Remove and recreate
docker compose down
docker compose up -d
```

---

### 10. Check Resource Usage

```bash
# Resource stats
docker compose ps -q | xargs docker stats

# Specific service
docker stats api-1

# Check disk usage
docker system df
```

---

### Complete Debugging Example

**Scenario: API service keeps restarting**

```bash
# Step 1: Check status
docker compose ps
# api-1  Restarting

# Step 2: View logs
docker compose logs api
# Error: Cannot connect to database

# Step 3: Check database
docker compose ps db
# db-1  Up (healthy)

# Step 4: Check network
docker compose exec api ping db
# PING db (172.18.0.2): 64 bytes

# Step 5: Check environment
docker compose exec api env | grep DATABASE
# DATABASE_URL=postgresql://wrong-host:5432/myapp

# Found problem: Wrong hostname in DATABASE_URL

# Step 6: Fix docker-compose.yml
# environment:
#   DATABASE_URL: postgresql://db:5432/myapp

# Step 7: Restart
docker compose up -d api

# Step 8: Verify
docker compose logs -f api
# Server running on port 3000
```

---

### Debugging Checklist

```
Service won't start:
☐ Check logs (docker compose logs service)
☐ Check exit code (docker inspect)
☐ Verify image exists (docker images)
☐ Check port conflicts (netstat -tlnp)
☐ Verify environment variables
☐ Check dependencies (depends_on)

Network issues:
☐ Ping other services
☐ Check DNS resolution (nslookup)
☐ Test port connectivity (nc -zv)
☐ Verify network configuration
☐ Check firewall rules

Performance issues:
☐ Check resource usage (docker stats)
☐ Review logs for errors
☐ Check health check settings
☐ Verify resource limits
☐ Monitor disk I/O

Data issues:
☐ Check volume mounts
☐ Verify permissions
☐ Inspect volume contents
☐ Check database logs
☐ Verify migrations ran
```

---

### Advanced Debugging Tools

**1. Docker Compose Logs with jq:**
```bash
docker compose logs --json api | jq -r '.log'
```

**2. Network Inspection:**
```bash
# Get network details
docker network inspect project_default

# Check connected containers
docker network inspect project_default | \
  jq '.[0].Containers'
```

**3. Volume Inspection:**
```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect project_db-data

# Check volume contents
docker run --rm -v project_db-data:/data alpine ls -la /data
```

**4. Event Stream:**
```bash
# Monitor events
docker compose events

# Filter by service
docker compose events api
```

</details>

</details>

---

[← Previous: 8.1 Compose Fundamentals](01-compose-fundamentals.md)