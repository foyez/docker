# 11.3 Common Patterns

Production-proven Docker patterns and architectures for real-world applications.

---

## Microservices Architecture

### Three-Tier Application

**Architecture:**
```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
┌──────▼──────────┐
│  Load Balancer  │  (Nginx)
└──────┬──────────┘
       │
┌──────▼──────────┐
│   Frontend      │  (React, Next.js)
└──────┬──────────┘
       │
┌──────▼──────────┐
│   API Gateway   │  (Express)
└──────┬──────────┘
       │
    ┌──┴──┐
┌───▼──┐ ┌▼───────┐
│ API  │ │Workers │  (Node.js, Python)
└───┬──┘ └────────┘
    │
┌───▼──────┐
│ Database │  (PostgreSQL)
└──────────┘
```

**docker-compose.yml:**
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
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - frontend
    networks:
      - frontend-net
    restart: unless-stopped

  # Frontend (React/Next.js)
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.prod
    environment:
      - API_URL=http://api:3000
      - NODE_ENV=production
    networks:
      - frontend-net
    restart: unless-stopped

  # API Gateway
  api:
    build: ./api
    environment:
      - DATABASE_URL=postgresql://db:5432/myapp
      - REDIS_URL=redis://redis:6379
      - NODE_ENV=production
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - frontend-net
      - backend-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s

  # Background Workers
  worker:
    build: ./worker
    environment:
      - DATABASE_URL=postgresql://db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    networks:
      - backend-net
    restart: unless-stopped
    deploy:
      replicas: 3

  # PostgreSQL Database
  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s

  # Redis Cache
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - backend-net
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

networks:
  frontend-net:
  backend-net:
    internal: true

volumes:
  db-data:
  redis-data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

---

## Database Patterns

### Database with Initialization

**docker-compose.yml:**
```yaml
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 5s

volumes:
  db-data:
```

**init-scripts/01-schema.sql:**
```sql
-- Create tables
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

**init-scripts/02-seed.sql:**
```sql
-- Seed data
INSERT INTO users (email) VALUES
    ('admin@example.com'),
    ('user@example.com')
ON CONFLICT (email) DO NOTHING;
```

---

### Database Backup Pattern

**backup-service:**
```yaml
services:
  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data

  backup:
    image: postgres:15
    environment:
      - BACKUP_DIR=/backups
      - PGHOST=db
      - PGUSER=admin
      - PGPASSWORD=secret
    volumes:
      - ./backups:/backups
      - ./backup.sh:/backup.sh
    entrypoint: /backup.sh
    depends_on:
      - db

volumes:
  db-data:
```

**backup.sh:**
```bash
#!/bin/bash
set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$TIMESTAMP.sql"

# Create backup
pg_dump myapp > "$BACKUP_FILE"

# Compress
gzip "$BACKUP_FILE"

# Delete old backups (keep last 7 days)
find "$BACKUP_DIR" -name "backup_*.sql.gz" -mtime +7 -delete

echo "Backup completed: backup_$TIMESTAMP.sql.gz"
```

**Cron backup:**
```yaml
services:
  backup-cron:
    image: postgres:15
    environment:
      - PGHOST=db
      - PGUSER=admin
      - PGPASSWORD=secret
    volumes:
      - ./backups:/backups
    command: >
      sh -c "
        echo '0 2 * * * pg_dump myapp | gzip > /backups/backup_$$(date +\%Y\%m\%d_\%H\%M\%S).sql.gz' | crontab -
        && crond -f
      "
```

---

## Caching Patterns

### Redis Cache Layer

**docker-compose.yml:**
```yaml
services:
  api:
    build: ./api
    environment:
      - REDIS_URL=redis://redis:6379
      - DATABASE_URL=postgresql://db:5432/myapp
    depends_on:
      - redis
      - db

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

volumes:
  redis-data:
```

**Application (Node.js):**
```javascript
const redis = require('redis');
const { Pool } = require('pg');

const redisClient = redis.createClient({
  url: process.env.REDIS_URL
});

const pool = new Pool({
  connectionString: process.env.DATABASE_URL
});

// Cache-aside pattern
async function getUser(userId) {
  const cacheKey = `user:${userId}`;
  
  // Try cache first
  const cached = await redisClient.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Cache miss - fetch from database
  const result = await pool.query(
    'SELECT * FROM users WHERE id = $1',
    [userId]
  );
  
  if (result.rows.length > 0) {
    const user = result.rows[0];
    
    // Cache for 1 hour
    await redisClient.setEx(cacheKey, 3600, JSON.stringify(user));
    
    return user;
  }
  
  return null;
}

// Cache invalidation
async function updateUser(userId, data) {
  await pool.query(
    'UPDATE users SET email = $1 WHERE id = $2',
    [data.email, userId]
  );
  
  // Invalidate cache
  await redisClient.del(`user:${userId}`);
}
```

---

## Message Queue Patterns

### Background Job Processing

**docker-compose.yml:**
```yaml
services:
  # API Server
  api:
    build: ./api
    environment:
      - RABBITMQ_URL=amqp://rabbitmq:5672
    depends_on:
      - rabbitmq
    ports:
      - "3000:3000"

  # Worker (processes jobs)
  worker:
    build: ./worker
    environment:
      - RABBITMQ_URL=amqp://rabbitmq:5672
      - DATABASE_URL=postgresql://db:5432/myapp
    depends_on:
      - rabbitmq
      - db
    deploy:
      replicas: 3

  # RabbitMQ
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"  # Management UI
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]

  db:
    image: postgres:15

volumes:
  rabbitmq-data:
```

**Producer (API):**
```javascript
const amqp = require('amqplib');

let channel;

async function connectQueue() {
  const connection = await amqp.connect(process.env.RABBITMQ_URL);
  channel = await connection.createChannel();
  await channel.assertQueue('jobs', { durable: true });
}

// Enqueue job
async function sendEmail(to, subject, body) {
  const job = {
    type: 'email',
    data: { to, subject, body }
  };
  
  channel.sendToQueue(
    'jobs',
    Buffer.from(JSON.stringify(job)),
    { persistent: true }
  );
}

app.post('/send-email', async (req, res) => {
  await sendEmail(req.body.to, req.body.subject, req.body.body);
  res.json({ status: 'queued' });
});
```

**Consumer (Worker):**
```javascript
const amqp = require('amqplib');

async function startWorker() {
  const connection = await amqp.connect(process.env.RABBITMQ_URL);
  const channel = await connection.createChannel();
  
  await channel.assertQueue('jobs', { durable: true });
  channel.prefetch(1);
  
  channel.consume('jobs', async (msg) => {
    const job = JSON.parse(msg.content.toString());
    
    try {
      // Process job
      if (job.type === 'email') {
        await sendEmail(job.data);
      }
      
      // Acknowledge
      channel.ack(msg);
    } catch (error) {
      console.error('Job failed:', error);
      // Reject and requeue
      channel.nack(msg, false, true);
    }
  });
}

startWorker();
```

---

## Monitoring Patterns

### Complete Monitoring Stack

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  # Application
  app:
    build: ./app
    ports:
      - "3000:3000"
    labels:
      - "prometheus.scrape=true"
      - "prometheus.port=3000"

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'

  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources

  # Loki (Logs)
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki

  # Promtail (Log collector)
  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

  # Node Exporter (Host metrics)
  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'

  # cAdvisor (Container metrics)
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro

volumes:
  prometheus-data:
  grafana-data:
  loki-data:
```

**prometheus/prometheus.yml:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'app'
    static_configs:
      - targets: ['app:3000']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

---

## Development Patterns

### Hot Reload Development

**docker-compose.dev.yml:**
```yaml
version: '3.8'

services:
  # Frontend with hot reload
  frontend:
    build:
      context: ./frontend
      target: development
    volumes:
      - ./frontend/src:/app/src
      - ./frontend/public:/app/public
      - /app/node_modules  # Anonymous volume for node_modules
    environment:
      - CHOKIDAR_USEPOLLING=true  # For Docker on Windows/Mac
    ports:
      - "3000:3000"
    command: npm start

  # Backend with nodemon
  api:
    build:
      context: ./api
      target: development
    volumes:
      - ./api/src:/app/src
      - /app/node_modules
    environment:
      - NODE_ENV=development
    ports:
      - "3001:3000"
      - "9229:9229"  # Debug port
    command: npm run dev
```

**frontend/Dockerfile:**
```dockerfile
# Development stage
FROM node:18 AS development
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]

# Production stage
FROM node:18 AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
RUN npm run build
CMD ["npm", "run", "serve"]
```

---

## Testing Patterns

### Integration Testing

**docker-compose.test.yml:**
```yaml
version: '3.8'

services:
  # Application under test
  api:
    build: ./api
    environment:
      - NODE_ENV=test
      - DATABASE_URL=postgresql://db:5432/test
    depends_on:
      db:
        condition: service_healthy

  # Test database
  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=test
      - POSTGRES_USER=test
      - POSTGRES_PASSWORD=test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 2s

  # Test runner
  test:
    build:
      context: ./api
      target: test
    environment:
      - API_URL=http://api:3000
      - DATABASE_URL=postgresql://db:5432/test
    depends_on:
      - api
      - db
    command: npm test
```

**Run tests:**
```bash
docker compose -f docker-compose.test.yml up --abort-on-container-exit
docker compose -f docker-compose.test.yml down -v
```

---

## CI/CD Patterns

### Multi-Stage Pipeline

**.gitlab-ci.yml:**
```yaml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# Build stage
build:
  stage: build
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

# Test stage
test:
  stage: test
  script:
    - docker compose -f docker-compose.test.yml up --abort-on-container-exit
    - docker compose -f docker-compose.test.yml down -v

# Deploy to staging
deploy-staging:
  stage: deploy
  script:
    - docker pull $IMAGE_TAG
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:staging
    - docker push $CI_REGISTRY_IMAGE:staging
    - ssh staging "docker pull $CI_REGISTRY_IMAGE:staging"
    - ssh staging "docker compose up -d"
  only:
    - develop

# Deploy to production
deploy-production:
  stage: deploy
  script:
    - docker pull $IMAGE_TAG
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
    - ssh production "docker pull $CI_REGISTRY_IMAGE:latest"
    - ssh production "docker compose up -d"
  only:
    - main
  when: manual
```

---

## Security Patterns

### Secrets Management

**Using Docker Secrets (Swarm):**
```yaml
version: '3.8'

services:
  app:
    image: myapp
    secrets:
      - db_password
      - api_key
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - API_KEY_FILE=/run/secrets/api_key

secrets:
  db_password:
    external: true
  api_key:
    external: true
```

**Application code:**
```javascript
const fs = require('fs');

function readSecret(name) {
  const secretPath = process.env[`${name.toUpperCase()}_FILE`];
  if (secretPath && fs.existsSync(secretPath)) {
    return fs.readFileSync(secretPath, 'utf8').trim();
  }
  return process.env[name.toUpperCase()];
}

const dbPassword = readSecret('db_password');
const apiKey = readSecret('api_key');
```

---

## Reverse Proxy Patterns

### Nginx as Reverse Proxy

**docker-compose.yml:**
```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/ssl:ro
    depends_on:
      - frontend
      - api

  frontend:
    build: ./frontend

  api:
    build: ./api
```

**nginx/nginx.conf:**
```nginx
upstream frontend {
    server frontend:3000;
}

upstream api {
    server api:3001;
}

server {
    listen 80;
    server_name example.com;

    # Frontend
    location / {
        proxy_pass http://frontend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # API
    location /api {
        proxy_pass http://api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # CORS
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
        add_header Access-Control-Allow-Headers "Authorization, Content-Type";
    }

    # WebSocket support
    location /ws {
        proxy_pass http://api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Health check
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}

# HTTPS redirect
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;

    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Same locations as above
    location / {
        proxy_pass http://frontend;
    }

    location /api {
        proxy_pass http://api;
    }
}
```

---

## Multi-Environment Pattern

**Base configuration:**
```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    environment:
      - NODE_ENV=${NODE_ENV}
  
  db:
    image: postgres:15
```

**Development override:**
```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    volumes:
      - ./src:/app/src
    ports:
      - "3000:3000"
      - "9229:9229"
    command: npm run dev

  db:
    ports:
      - "5432:5432"
```

**Production override:**
```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
    restart: unless-stopped

  db:
    deploy:
      resources:
        limits:
          memory: 2G
    restart: unless-stopped
```

**Usage:**
```bash
# Development
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. A __________ network prevents services from accessing the internet.
2. Use __________ pattern to check cache before querying database.
3. __________ processes background jobs from a message queue.
4. Multiple Compose files are merged __________ to right.
5. Init scripts run when database is __________ created.
6. Nginx acts as a __________ proxy to route requests.

### True/False

1. ⬜ Internal networks can access external services
2. ⬜ Worker pattern requires message queue
3. ⬜ Health checks are optional for depends_on conditions
4. ⬜ Multiple Compose files can be combined
5. ⬜ Init scripts run every time database starts
6. ⬜ Secrets should be stored in Dockerfile
7. ⬜ Hot reload requires mounting source code

### Multiple Choice

1. Best pattern for background jobs?
   - A) Cron in container
   - B) Message queue + workers
   - C) Database polling
   - D) REST API calls

2. Cache-aside pattern checks?
   - A) Database first
   - B) Cache first
   - C) Both simultaneously
   - D) Random

3. How to isolate backend services?
   - A) Different ports
   - B) Internal network
   - C) Separate hosts
   - D) Firewall rules

4. Multi-environment approach?
   - A) Multiple Dockerfiles
   - B) Multiple Compose files
   - C) Multiple images
   - D) Multiple registries

5. Reverse proxy benefits?
   - A) Load balancing
   - B) SSL termination
   - C) Request routing
   - D) All of the above

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. internal
2. cache-aside
3. Worker
4. left
5. first (or initially)
6. reverse

**True/False:**
1. ❌ False - Internal networks have no internet access
2. ✅ True - Worker pattern uses message queue
3. ❌ False - Conditions like service_healthy require health checks
4. ✅ True - Use multiple -f flags
5. ❌ False - Only on first creation
6. ❌ False - Never store secrets in Dockerfile
7. ✅ True - Mount source for hot reload

**Multiple Choice:**
1. **B** - Message queue + workers
2. **B** - Cache first
3. **B** - Internal network
4. **B** - Multiple Compose files
5. **D** - All of the above

</details>

</details>

---

[← Previous: 11.2 Troubleshooting Guide](02-troubleshooting-guide.md)