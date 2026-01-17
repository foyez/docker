# 10.2 Production Considerations

Critical considerations for running Docker containers in production environments.

---

## Health Checks

### Container Health Checks

**Definition:**
```
Health check = Test if container is working
- Runs periodically
- Reports health status
- Triggers restart if unhealthy
- Used by orchestrators
```

**Dockerfile:**
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY . .

HEALTHCHECK --interval=30s \
            --timeout=3s \
            --start-period=40s \
            --retries=3 \
  CMD node healthcheck.js || exit 1

CMD ["node", "server.js"]
```

**healthcheck.js:**
```javascript
const http = require('http');

const options = {
  host: 'localhost',
  port: 3000,
  path: '/health',
  timeout: 2000
};

const req = http.request(options, (res) => {
  if (res.statusCode === 200) {
    process.exit(0);
  } else {
    process.exit(1);
  }
});

req.on('error', () => process.exit(1));
req.on('timeout', () => process.exit(1));
req.end();
```

---

### Health Check Endpoints

**Basic Health:**
```javascript
// app.js
const express = require('express');
const app = express();

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(3000);
```

**Deep Health Check:**
```javascript
app.get('/health', async (req, res) => {
  const checks = {
    uptime: process.uptime(),
    timestamp: Date.now(),
    status: 'healthy'
  };

  try {
    // Check database
    await db.query('SELECT 1');
    checks.database = 'healthy';

    // Check Redis
    await redis.ping();
    checks.redis = 'healthy';

    // Check disk space
    const diskUsage = await checkDiskSpace('/');
    if (diskUsage.free < 1000000000) { // Less than 1GB
      throw new Error('Low disk space');
    }
    checks.disk = 'healthy';

    res.status(200).json(checks);
  } catch (error) {
    checks.status = 'unhealthy';
    checks.error = error.message;
    res.status(503).json(checks);
  }
});
```

**Readiness vs Liveness:**
```javascript
// Liveness: Is container alive?
app.get('/healthz', (req, res) => {
  res.status(200).send('OK');
});

// Readiness: Can container handle traffic?
app.get('/ready', async (req, res) => {
  try {
    await db.ping();
    await cache.ping();
    res.status(200).send('Ready');
  } catch (error) {
    res.status(503).send('Not Ready');
  }
});
```

---

## Logging

### Logging Best Practices

**Log to stdout/stderr:**
```javascript
// Good
console.log('Info message');
console.error('Error message');

// Bad - Don't write to files
const fs = require('fs');
fs.appendFileSync('/var/log/app.log', 'message');
```

**Why stdout/stderr:**
```
✓ Docker captures automatically
✓ Centralized logging easy
✓ No disk space issues
✓ Log rotation handled externally
✓ Works with orchestrators
```

---

### Structured Logging

**JSON Logs:**
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console()
  ]
});

logger.info('User logged in', {
  userId: '123',
  ip: '1.2.3.4',
  timestamp: new Date().toISOString()
});

// Output:
// {"level":"info","message":"User logged in","userId":"123","ip":"1.2.3.4","timestamp":"2024-01-17T10:00:00.000Z"}
```

**Python:**
```python
import logging
import json
from datetime import datetime

class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName
        }
        return json.dumps(log_data)

handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())

logger = logging.getLogger()
logger.addHandler(handler)
logger.setLevel(logging.INFO)

logger.info('Application started', extra={'version': '1.0'})
```

---

### Log Aggregation

**Docker Logging Drivers:**

```bash
# JSON file (default)
docker run --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp

# Syslog
docker run --log-driver=syslog \
  --log-opt syslog-address=tcp://192.168.1.1:514 \
  myapp

# Fluentd
docker run --log-driver=fluentd \
  --log-opt fluentd-address=localhost:24224 \
  myapp

# AWS CloudWatch
docker run --log-driver=awslogs \
  --log-opt awslogs-region=us-east-1 \
  --log-opt awslogs-group=myapp \
  myapp
```

---

### Centralized Logging Stack

**ELK Stack (docker-compose.yml):**
```yaml
version: '3.8'

services:
  # Application
  app:
    image: myapp
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: myapp

  # Fluentd
  fluentd:
    image: fluent/fluentd:v1.16
    ports:
      - "24224:24224"
    volumes:
      - ./fluentd.conf:/fluentd/etc/fluent.conf
    depends_on:
      - elasticsearch

  # Elasticsearch
  elasticsearch:
    image: elasticsearch:8.11.3
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - es-data:/usr/share/elasticsearch/data

  # Kibana
  kibana:
    image: kibana:8.11.3
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

**fluentd.conf:**
```
<source>
  @type forward
  port 24224
</source>

<match myapp>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix myapp
  include_tag_key true
  tag_key @log_name
</match>
```

---

## Resource Management

### CPU Limits

```bash
# Limit to 50% of one CPU
docker run --cpus=0.5 myapp

# Limit to 2 CPUs
docker run --cpus=2 myapp

# CPU shares (relative weight)
docker run --cpu-shares=512 myapp
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  api:
    image: myapi
    deploy:
      resources:
        limits:
          cpus: '1.0'
        reservations:
          cpus: '0.5'
```

---

### Memory Limits

```bash
# Hard limit
docker run --memory=512m myapp

# Soft limit (reservation)
docker run --memory-reservation=256m myapp

# Combined
docker run \
  --memory=512m \
  --memory-reservation=256m \
  myapp

# OOM kill disable (dangerous)
docker run --oom-kill-disable myapp
```

**docker-compose.yml:**
```yaml
services:
  db:
    image: postgres
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
```

---

### Monitoring Resource Usage

```bash
# Real-time stats
docker stats

# Specific container
docker stats myapp

# Format output
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

---

## Environment Variables

### Managing Secrets

**Bad:**
```dockerfile
ENV DB_PASSWORD=supersecret
ENV API_KEY=abc123
```

**Good:**

**1. Environment Files:**
```bash
# secrets.env (gitignored)
DB_PASSWORD=supersecret
API_KEY=abc123

# Run
docker run --env-file secrets.env myapp
```

**2. Docker Secrets (Swarm):**
```bash
# Create secret
echo "mysecret" | docker secret create db_password -

# Use in service
docker service create \
  --name myapp \
  --secret db_password \
  myapp
```

**3. Kubernetes Secrets:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  db-password: supersecret
  api-key: abc123
---
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db-password
```

**4. External Secret Managers:**
```javascript
// AWS Secrets Manager
const AWS = require('aws-sdk');
const secretsManager = new AWS.SecretsManager();

async function getSecret(secretName) {
  const data = await secretsManager.getSecretValue({
    SecretId: secretName
  }).promise();
  
  return JSON.parse(data.SecretString);
}

// At startup
const secrets = await getSecret('myapp/production');
const db = connect(secrets.DB_PASSWORD);
```

---

### Configuration Management

**Environment-based Config:**
```javascript
const config = {
  development: {
    db: {
      host: 'localhost',
      port: 5432
    },
    logging: 'debug'
  },
  production: {
    db: {
      host: process.env.DB_HOST,
      port: process.env.DB_PORT
    },
    logging: 'warn'
  }
};

const env = process.env.NODE_ENV || 'development';
module.exports = config[env];
```

**12-Factor App Config:**
```javascript
// config.js
module.exports = {
  port: process.env.PORT || 3000,
  database: {
    host: process.env.DB_HOST || 'localhost',
    port: process.env.DB_PORT || 5432,
    name: process.env.DB_NAME || 'myapp',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD // Required
  },
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379'
  },
  logLevel: process.env.LOG_LEVEL || 'info'
};
```

---

## Monitoring

### Prometheus Metrics

**Node.js:**
```javascript
const express = require('express');
const promClient = require('prom-client');

const app = express();

// Create registry
const register = new promClient.Registry();

// Default metrics
promClient.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  registers: [register]
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status'],
  registers: [register]
});

// Middleware
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    
    httpRequestDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .observe(duration);
    
    httpRequestTotal
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .inc();
  });
  
  next();
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(3000);
```

---

### Monitoring Stack

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  # Application
  app:
    image: myapp
    ports:
      - "3000:3000"
    labels:
      - "prometheus.scrape=true"
      - "prometheus.port=3000"
      - "prometheus.path=/metrics"

  # Prometheus
  prometheus:
    image: prom/prometheus:v2.48.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

  # Grafana
  grafana:
    image: grafana/grafana:10.2.2
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources

  # Node Exporter (host metrics)
  node-exporter:
    image: prom/node-exporter:v1.7.0
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

  # cAdvisor (container metrics)
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
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
```

**prometheus.yml:**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Application
  - job_name: 'app'
    static_configs:
      - targets: ['app:3000']

  # Node Exporter
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # cAdvisor
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

---

## High Availability

### Replicas

**Docker Swarm:**
```bash
docker service create \
  --name web \
  --replicas 3 \
  --update-parallelism 1 \
  --update-delay 10s \
  nginx
```

**docker-compose (Swarm mode):**
```yaml
version: '3.8'

services:
  web:
    image: nginx
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
```

**Kubernetes:**
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
        image: nginx:1.25
```

---

### Load Balancing

**Nginx:**
```nginx
upstream backend {
    least_conn;
    server app1:3000 max_fails=3 fail_timeout=30s;
    server app2:3000 max_fails=3 fail_timeout=30s;
    server app3:3000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Health check
        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
    }

    location /health {
        access_log off;
        return 200 "healthy\n";
    }
}
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app

  app:
    image: myapp
    deploy:
      replicas: 3
```

---

### Graceful Shutdown

**Node.js:**
```javascript
const express = require('express');
const app = express();

// Track active connections
let connections = [];
let isShuttingDown = false;

const server = app.listen(3000, () => {
  console.log('Server started');
});

// Track connections
server.on('connection', (conn) => {
  connections.push(conn);
  conn.on('close', () => {
    connections = connections.filter(c => c !== conn);
  });
});

// Health endpoint
app.get('/health', (req, res) => {
  if (isShuttingDown) {
    res.status(503).send('Shutting down');
  } else {
    res.status(200).send('OK');
  }
});

// Graceful shutdown
function gracefulShutdown(signal) {
  console.log(`Received ${signal}, starting graceful shutdown`);
  isShuttingDown = true;

  // Stop accepting new connections
  server.close(() => {
    console.log('Server closed');

    // Close database connections
    db.close(() => {
      console.log('Database closed');
      process.exit(0);
    });
  });

  // Force shutdown after timeout
  setTimeout(() => {
    console.error('Forced shutdown');
    process.exit(1);
  }, 30000);

  // Close existing connections
  connections.forEach(conn => conn.end());
  setTimeout(() => {
    connections.forEach(conn => conn.destroy());
  }, 10000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
```

**Python (Flask):**
```python
from flask import Flask
import signal
import sys
import time

app = Flask(__name__)
is_shutting_down = False

@app.route('/health')
def health():
    if is_shutting_down:
        return 'Shutting down', 503
    return 'OK', 200

def graceful_shutdown(signum, frame):
    global is_shutting_down
    print(f'Received signal {signum}, starting graceful shutdown')
    is_shutting_down = True
    
    # Finish pending requests (give 30 seconds)
    time.sleep(30)
    
    # Close database
    db.close()
    
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
signal.signal(signal.SIGINT, graceful_shutdown)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## Data Persistence

### Volume Best Practices

**Named Volumes:**
```bash
# Create volume
docker volume create app-data

# Use volume
docker run -v app-data:/data myapp

# Inspect
docker volume inspect app-data

# Backup
docker run --rm \
  -v app-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/data.tar.gz /data

# Restore
docker run --rm \
  -v app-data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/data.tar.gz -C /
```

---

### Database Persistence

**PostgreSQL:**
```yaml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped

volumes:
  db-data:
    driver: local
```

**Backup Script:**
```bash
#!/bin/bash

# backup.sh
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups"

# Create backup
docker exec postgres pg_dump -U postgres myapp > "$BACKUP_DIR/myapp_$TIMESTAMP.sql"

# Compress
gzip "$BACKUP_DIR/myapp_$TIMESTAMP.sql"

# Delete old backups (keep last 7 days)
find "$BACKUP_DIR" -name "myapp_*.sql.gz" -mtime +7 -delete

echo "Backup completed: myapp_$TIMESTAMP.sql.gz"
```

**Cron:**
```bash
# Backup daily at 2 AM
0 2 * * * /path/to/backup.sh
```

---

## Security

### Security Scanning

**Trivy:**
```bash
# Scan image
trivy image myapp:latest

# Only high/critical
trivy image --severity HIGH,CRITICAL myapp:latest

# Output to file
trivy image -f json -o results.json myapp:latest

# Fail on vulnerabilities
trivy image --exit-code 1 --severity CRITICAL myapp:latest
```

**Docker Scout:**
```bash
# Quick view
docker scout quickview myapp:latest

# CVEs
docker scout cves myapp:latest

# Recommendations
docker scout recommendations myapp:latest
```

---

### Runtime Security

**AppArmor:**
```bash
# Create profile
cat > /etc/apparmor.d/docker-myapp << EOF
#include <tunables/global>

profile docker-myapp flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  
  # Deny network access
  deny network,
  
  # Allow specific file access
  /app/** r,
  /tmp/** rw,
}
EOF

# Load profile
apparmor_parser -r /etc/apparmor.d/docker-myapp

# Run container
docker run --security-opt apparmor=docker-myapp myapp
```

**Seccomp:**
```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64"
  ],
  "syscalls": [
    {
      "names": [
        "read",
        "write",
        "open",
        "close",
        "stat",
        "fstat",
        "mmap",
        "brk",
        "exit_group"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

```bash
docker run --security-opt seccomp=seccomp-profile.json myapp
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Health checks should return exit code __________ for healthy and __________ for unhealthy.
2. Logs should be written to __________ and __________, not files.
3. Use __________ to limit container CPU usage.
4. __________ provide a way to pass sensitive data to containers securely.
5. Prometheus scrapes metrics from the __________ endpoint.
6. Graceful shutdown handles __________ and __________ signals.

### True/False

1. ⬜ Health checks are optional in production
2. ⬜ Writing logs to files inside containers is recommended
3. ⬜ Memory limits prevent containers from consuming all host RAM
4. ⬜ Environment variables are secure for storing passwords
5. ⬜ Prometheus requires application code changes
6. ⬜ Containers should handle SIGTERM for graceful shutdown
7. ⬜ Named volumes persist data across container restarts

### Multiple Choice

1. Best practice for logging?
   - A) Log to files
   - B) Log to stdout/stderr
   - C) Log to syslog only
   - D) Disable logging

2. What does --memory=512m do?
   - A) Sets minimum memory
   - B) Sets maximum memory
   - C) Reserves memory
   - D) Nothing

3. Health check interval default?
   - A) 10s
   - B) 30s
   - C) 60s
   - D) No default

4. Graceful shutdown timeout?
   - A) 5 seconds
   - B) 10 seconds
   - C) 30 seconds
   - D) Depends on application

5. Best way to store secrets?
   - A) Environment variables
   - B) Dockerfile ENV
   - C) Docker secrets
   - D) Hardcoded

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. 0 (zero), 1 (non-zero)
2. stdout, stderr
3. --cpus (or --cpu-quota)
4. Secrets (or Docker secrets, Kubernetes secrets)
5. /metrics
6. SIGTERM, SIGINT

**True/False:**
1. ❌ False - Health checks critical for production
2. ❌ False - Use stdout/stderr
3. ✅ True - Prevents OOM on host
4. ❌ False - Use secrets management
5. ✅ True - Need to expose metrics endpoint
6. ✅ True - Required for graceful shutdown
7. ✅ True - Volumes persist independently

**Multiple Choice:**
1. **B** - Log to stdout/stderr
2. **B** - Sets maximum memory
3. **B** - 30s
4. **D** - Depends on application needs
5. **C** - Docker secrets (or external secret managers)

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: How do you implement zero-downtime deployments with containers?

<details>
<summary><strong>View Answer</strong></summary>

**Zero-Downtime Deployment Strategy:**

---

### Prerequisites

**1. Health Checks:**
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY . .

HEALTHCHECK --interval=10s --timeout=3s --start-period=30s \
  CMD node healthcheck.js || exit 1

CMD ["node", "server.js"]
```

**2. Graceful Shutdown:**
```javascript
let isShuttingDown = false;

app.get('/health', (req, res) => {
  if (isShuttingDown) {
    return res.status(503).send('Shutting down');
  }
  res.send('OK');
});

process.on('SIGTERM', async () => {
  console.log('SIGTERM received');
  isShuttingDown = true;
  
  // Stop accepting new requests
  server.close(async () => {
    // Close connections
    await db.close();
    await redis.quit();
    process.exit(0);
  });
  
  // Force exit after 30s
  setTimeout(() => process.exit(1), 30000);
});
```

**3. Load Balancer Health Checks:**
```nginx
upstream backend {
    server app1:3000 max_fails=3 fail_timeout=30s;
    server app2:3000 max_fails=3 fail_timeout=30s;
    server app3:3000 max_fails=3 fail_timeout=30s;
    
    # Health check
    check interval=5000 rise=2 fall=3 timeout=3000;
}
```

---

### Strategy 1: Rolling Updates

**Docker Swarm:**
```yaml
version: '3.8'

services:
  app:
    image: myapp:${VERSION}
    deploy:
      replicas: 6
      update_config:
        parallelism: 2        # Update 2 at a time
        delay: 10s            # Wait 10s between batches
        failure_action: rollback
        monitor: 30s          # Monitor for 30s
        order: start-first    # Start new before stopping old
      rollback_config:
        parallelism: 2
        delay: 5s
      restart_policy:
        condition: on-failure
```

**Process:**
```
Initial: 6 containers running v1.0

Step 1: Start 2 new (v2.0)
- 6 old + 2 new = 8 total
- Wait for health checks
- Monitor for 30s

Step 2: Stop 2 old (v1.0)
- 4 old + 2 new = 6 total
- Wait 10s

Step 3: Start 2 new (v2.0)
- 4 old + 4 new = 8 total
- Wait for health checks
- Monitor for 30s

Step 4: Stop 2 old (v1.0)
- 2 old + 4 new = 6 total
- Wait 10s

Step 5: Start 2 new (v2.0)
- 2 old + 6 new = 8 total
- Wait for health checks
- Monitor for 30s

Step 6: Stop 2 old (v1.0)
- 0 old + 6 new = 6 total

Complete: All v2.0
Zero downtime: Always 6+ healthy containers
```

**Deploy:**
```bash
export VERSION=2.0.0
docker stack deploy -c docker-stack.yml myapp

# Monitor
watch docker service ps myapp
```

---

### Strategy 2: Blue-Green Deployment

**Setup:**
```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  # Blue environment (current)
  app-blue:
    image: myapp:1.0.0
    deploy:
      replicas: 3
    environment:
      COLOR: blue

  # Green environment (new)
  app-green:
    image: myapp:2.0.0
    deploy:
      replicas: 3
    environment:
      COLOR: green
```

**nginx.conf (pointing to blue):**
```nginx
upstream backend {
    server app-blue:3000;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
    }
}
```

**Deployment Process:**
```bash
# 1. Deploy green environment
docker stack deploy -c docker-stack.yml myapp

# 2. Verify green is healthy
curl http://app-green:3000/health

# 3. Run smoke tests
./smoke-tests.sh http://app-green:3000

# 4. Switch nginx to green
cat > nginx.conf << EOF
upstream backend {
    server app-green:3000;
}
EOF

# 5. Reload nginx
docker exec nginx nginx -s reload

# 6. Monitor for issues
./monitor.sh

# 7. If all good, remove blue
# If issues, switch back to blue
```

**Automated Switch:**
```bash
#!/bin/bash

GREEN_HEALTH=$(curl -s http://app-green:3000/health)

if [ "$GREEN_HEALTH" == "OK" ]; then
  # Switch to green
  sed -i 's/app-blue/app-green/' nginx.conf
  docker exec nginx nginx -s reload
  echo "Switched to green"
else
  echo "Green unhealthy, staying on blue"
  exit 1
fi
```

---

### Strategy 3: Canary Deployment

**Gradual Traffic Shift:**
```nginx
# Phase 1: 10% to canary
upstream backend {
    server app-stable:3000 weight=9;
    server app-canary:3000 weight=1;
}

# Phase 2: 50% to canary
upstream backend {
    server app-stable:3000 weight=5;
    server app-canary:3000 weight=5;
}

# Phase 3: 100% to canary
upstream backend {
    server app-canary:3000;
}
```

**Monitoring During Canary:**
```bash
#!/bin/bash

# Deploy canary
docker service scale app-canary=1

# Monitor error rate
for i in {1..60}; do
  ERROR_RATE=$(curl -s http://prometheus:9090/api/v1/query \
    --data-urlencode 'query=rate(http_requests_total{status=~"5.."}[1m])' | \
    jq -r '.data.result[0].value[1]')
  
  if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
    echo "Error rate too high: $ERROR_RATE"
    docker service scale app-canary=0
    exit 1
  fi
  
  sleep 10
done

# If successful, continue rollout
docker service scale app-canary=3
```

---

### Strategy 4: Kubernetes Rolling Update

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2         # 2 extra pods during update
      maxUnavailable: 0   # No downtime
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:2.0.0
        ports:
        - containerPort: 3000
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
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

**Process:**
```
Initial: 6 pods running v1.0

Step 1: Create 2 new pods (v2.0)
- 6 old + 2 new = 8 total
- Wait for readiness probe

Step 2: Delete 2 old pods (v1.0)
- 4 old + 2 new = 6 total

Step 3: Create 2 new pods (v2.0)
- 4 old + 4 new = 8 total
- Wait for readiness probe

Step 4: Delete 2 old pods (v1.0)
- 2 old + 4 new = 6 total

Step 5: Create 2 new pods (v2.0)
- 2 old + 6 new = 8 total
- Wait for readiness probe

Step 6: Delete 2 old pods (v1.0)
- 0 old + 6 new = 6 total

Always >= 6 pods available
Zero downtime
```

**Deploy:**
```bash
# Update image
kubectl set image deployment/app app=myapp:2.0.0

# Monitor
kubectl rollout status deployment/app

# Pause if issues
kubectl rollout pause deployment/app

# Rollback if needed
kubectl rollout undo deployment/app
```

---

### Database Migrations

**Challenge:**
```
Problem: Database schema changes
- Old code expects old schema
- New code expects new schema
- Need both during deployment
```

**Solution: Backward Compatible Migrations**

```sql
-- Phase 1: Add new column (nullable)
ALTER TABLE users ADD COLUMN email_new VARCHAR(255);

-- Deploy v2.0 (uses email_new)
-- Keep v1.0 running (uses email)

-- Phase 2: Backfill data
UPDATE users SET email_new = email WHERE email_new IS NULL;

-- Phase 3: Make not null
ALTER TABLE users ALTER COLUMN email_new SET NOT NULL;

-- Phase 4: Drop old column
ALTER TABLE users DROP COLUMN email;
```

**Three-Phase Deployment:**
```
Phase 1:
- Deploy migration (add email_new)
- Deploy v2.0 (writes both email and email_new)
- v1.0 still running (uses email)

Phase 2:
- Backfill email_new
- All v1.0 pods replaced by v2.0

Phase 3:
- Remove old column
- Deploy v3.0 (uses only email_new)
```

---

### Best Practices

```
Health Checks:
✓ Implement liveness and readiness
✓ Different endpoints for each
✓ Fast health checks (<3s)
✓ Include dependency checks

Graceful Shutdown:
✓ Handle SIGTERM
✓ Stop accepting new requests
✓ Finish pending requests
✓ Close connections
✓ Timeout after 30s

Load Balancer:
✓ Remove unhealthy backends
✓ Health check every 5-10s
✓ Fail after 3 consecutive failures

Monitoring:
✓ Error rate
✓ Response time
✓ Request count
✓ Active connections

Rollback Plan:
✓ Automated rollback on failure
✓ Keep old version accessible
✓ Test rollback procedure
```

</details>

### Question 2: How do you handle secrets in production Docker deployments?

<details>
<summary><strong>View Answer</strong></summary>

**Production Secrets Management:**

---

### Never Do This

**❌ Hardcoded in Dockerfile:**
```dockerfile
ENV DB_PASSWORD=supersecret
ENV API_KEY=abc123xyz
```

**Why it's bad:**
```
- Visible in docker inspect
- Visible in docker history
- Stored in image layers
- Pushed to registry
- Anyone with image has secrets
```

---

### Solution 1: Environment Variables (Basic)

**Runtime Injection:**
```bash
# Pass at runtime
docker run -e DB_PASSWORD=secret myapp

# From file
docker run --env-file secrets.env myapp
```

**secrets.env:**
```env
DB_PASSWORD=supersecret
API_KEY=abc123xyz
JWT_SECRET=random-secret-key
```

**.gitignore:**
```
secrets.env
.env.production
*.secret
```

**Pros:**
```
✓ Simple
✓ Not in image
✓ Easy to change
```

**Cons:**
```
✗ Visible in 'docker inspect'
✗ Visible in process list
✗ No encryption at rest
✗ No access control
```

---

### Solution 2: Docker Secrets (Swarm)

**Create Secret:**
```bash
# From stdin
echo "mysecretpassword" | docker secret create db_password -

# From file
docker secret create db_password ./password.txt

# List secrets
docker secret ls

# Inspect (no value shown)
docker secret inspect db_password
```

**Use in Service:**
```yaml
version: '3.8'

services:
  app:
    image: myapp
    secrets:
      - db_password
      - api_key
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
      API_KEY_FILE: /run/secrets/api_key

secrets:
  db_password:
    external: true
  api_key:
    external: true
```

**Access in Application:**
```javascript
const fs = require('fs');

function readSecret(secretName) {
  try {
    return fs.readFileSync(`/run/secrets/${secretName}`, 'utf8').trim();
  } catch (err) {
    // Fallback to environment variable
    return process.env[secretName];
  }
}

const dbPassword = readSecret('db_password');
const apiKey = readSecret('api_key');
```

**Pros:**
```
✓ Encrypted at rest
✓ Encrypted in transit
✓ Not in image
✓ Access control
✓ Audit trail
```

**Cons:**
```
✗ Requires Docker Swarm
✗ Swarm-specific
```

---

### Solution 3: Kubernetes Secrets

**Create Secret:**
```bash
# From literals
kubectl create secret generic app-secrets \
  --from-literal=db-password=supersecret \
  --from-literal=api-key=abc123

# From files
kubectl create secret generic app-secrets \
  --from-file=db-password=./password.txt \
  --from-file=api-key=./api-key.txt
```

**YAML:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  db-password: supersecret
  api-key: abc123xyz
```

**Use as Environment Variables:**
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db-password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api-key
```

**Use as Files:**
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secrets
    secret:
      secretName: app-secrets
```

**Access:**
```javascript
// From environment
const dbPassword = process.env.DB_PASSWORD;

// From file
const fs = require('fs');
const dbPassword = fs.readFileSync('/etc/secrets/db-password', 'utf8');
```

**Encryption at Rest:**
```yaml
# Enable encryption
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64-encoded-secret>
    - identity: {}
```

---

### Solution 4: HashiCorp Vault

**Setup Vault:**
```bash
# Start Vault
docker run -d --name=vault \
  --cap-add=IPC_LOCK \
  -e 'VAULT_DEV_ROOT_TOKEN_ID=myroot' \
  vault

# Store secret
docker exec vault vault kv put secret/myapp \
  db_password=supersecret \
  api_key=abc123
```

**Application Integration:**
```javascript
const vault = require('node-vault')({
  endpoint: 'http://vault:8200',
  token: process.env.VAULT_TOKEN
});

async function getSecrets() {
  const response = await vault.read('secret/data/myapp');
  return response.data.data;
}

// At startup
const secrets = await getSecrets();
const db = connect(secrets.db_password);
```

**Vault Agent (Sidecar):**
```yaml
version: '3.8'

services:
  vault-agent:
    image: vault:latest
    volumes:
      - ./vault-agent.hcl:/vault/config/vault-agent.hcl
      - secrets:/vault/secrets
    command: vault agent -config=/vault/config/vault-agent.hcl

  app:
    image: myapp
    volumes:
      - secrets:/secrets:ro
    depends_on:
      - vault-agent

volumes:
  secrets:
```

**vault-agent.hcl:**
```hcl
vault {
  address = "http://vault:8200"
}

auto_auth {
  method {
    type = "token_file"
    config = {
      token_file_path = "/vault-token"
    }
  }
}

template {
  source      = "/vault/templates/secrets.ctmpl"
  destination = "/vault/secrets/secrets.env"
}
```

---

### Solution 5: Cloud Provider Secrets

**AWS Secrets Manager:**
```javascript
const AWS = require('aws-sdk');
const secretsManager = new AWS.SecretsManager({
  region: 'us-east-1'
});

async function getSecret(secretName) {
  const data = await secretsManager.getSecretValue({
    SecretId: secretName
  }).promise();
  
  return JSON.parse(data.SecretString);
}

// At startup
const secrets = await getSecret('myapp/production');
```

**Dockerfile:**
```dockerfile
FROM node:18-alpine

# No secrets in image
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

# Secrets loaded from AWS at runtime
CMD ["node", "server.js"]
```

**ECS Task Definition:**
```json
{
  "family": "myapp",
  "taskRoleArn": "arn:aws:iam::123456789:role/myapp-task-role",
  "containerDefinitions": [{
    "name": "app",
    "image": "myapp:latest",
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/db-password"
      }
    ]
  }]
}
```

**GCP Secret Manager:**
```javascript
const {SecretManagerServiceClient} = require('@google-cloud/secret-manager');
const client = new SecretManagerServiceClient();

async function getSecret(name) {
  const [version] = await client.accessSecretVersion({
    name: `projects/my-project/secrets/${name}/versions/latest`
  });
  
  return version.payload.data.toString();
}
```

---

### Solution 6: Sealed Secrets (Kubernetes)

**Install Sealed Secrets:**
```bash
# Install controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Install kubeseal CLI
brew install kubeseal
```

**Create Sealed Secret:**
```bash
# Create secret (don't apply)
kubectl create secret generic app-secrets \
  --from-literal=db-password=supersecret \
  --dry-run=client -o yaml > secret.yaml

# Seal it
kubeseal < secret.yaml > sealed-secret.yaml

# Safe to commit sealed-secret.yaml
git add sealed-secret.yaml
git commit -m "Add sealed secrets"
```

**Apply:**
```bash
kubectl apply -f sealed-secret.yaml

# Controller decrypts and creates Secret
kubectl get secret app-secrets
```

---

### Best Practices

**1. Separation of Concerns:**
```
Separate secrets per environment:
- secrets/dev
- secrets/staging
- secrets/production

Separate secrets per service:
- myapp/db
- myapp/redis
- myapp/api-keys
```

**2. Rotation:**
```bash
# Rotate regularly
aws secretsmanager rotate-secret \
  --secret-id myapp/db-password \
  --rotation-lambda-arn arn:aws:lambda:...

# Application should support rotation
# - Reload on SIGHUP
# - Poll for changes
# - Vault agent auto-renew
```

**3. Least Privilege:**
```
Grant minimum access:
- Read-only access
- Specific secrets only
- Time-limited tokens
- Audit access
```

**4. Never Log Secrets:**
```javascript
// Bad
console.log('DB Password:', dbPassword);

// Good
console.log('DB Password: [REDACTED]');

// Sanitize logs
function sanitize(message) {
  return message.replace(/password=\S+/gi, 'password=[REDACTED]');
}
```

**5. Encryption in Transit:**
```
✓ TLS for all communications
✓ Encrypted connections to secret stores
✓ mTLS between services
```

---

### Comparison

| Method | Security | Complexity | Cost | Use Case |
|--------|----------|------------|------|----------|
| **Env Vars** | Low | Low | Free | Development |
| **Docker Secrets** | Medium | Low | Free | Swarm |
| **K8s Secrets** | Medium | Medium | Free | Kubernetes |
| **Vault** | High | High | Free/Paid | Enterprise |
| **AWS Secrets** | High | Medium | Paid | AWS |
| **Sealed Secrets** | High | Medium | Free | K8s GitOps |

</details>

### Question 3: How do you monitor and troubleshoot containers in production?

<details>
<summary><strong>View Answer</strong></summary>

**Production Monitoring and Troubleshooting:**

---

### Monitoring Stack

**Components:**
```
1. Metrics Collection (Prometheus)
2. Log Aggregation (ELK/Loki)
3. Tracing (Jaeger/Zipkin)
4. Visualization (Grafana)
5. Alerting (Alertmanager)
```

---

### 1. Metrics Collection

**Application Metrics (Prometheus):**

```javascript
const express = require('express');
const promClient = require('prom-client');

const app = express();
const register = new promClient.Registry();

// Default metrics (CPU, memory, etc.)
promClient.collectDefaultMetrics({ register });

// Custom metrics
const httpDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.5, 1, 2, 5],
  registers: [register]
});

const httpTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register]
});

const activeConnections = new promClient.Gauge({
  name: 'active_connections',
  help: 'Active connections',
  registers: [register]
});

// Middleware
app.use((req, res, next) => {
  const start = Date.now();
  activeConnections.inc();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route?.path || req.path;
    
    httpDuration.labels(req.method, route, res.statusCode).observe(duration);
    httpTotal.labels(req.method, route, res.statusCode).inc();
    activeConnections.dec();
  });
  
  next();
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(3000);
```

**Prometheus Config:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'app'
    static_configs:
      - targets: ['app:3000']
    
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
    
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

---

### 2. Log Aggregation

**Structured Logging:**
```javascript
const winston = require('winston');

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'myapp',
    version: process.env.VERSION
  },
  transports: [
    new winston.transports.Console()
  ]
});

// Usage
logger.info('User logged in', {
  userId: '123',
  ip: req.ip,
  userAgent: req.get('user-agent')
});

logger.error('Database error', {
  error: err.message,
  stack: err.stack,
  query: query
});
```

**ELK Stack:**
```yaml
version: '3.8'

services:
  app:
    image: myapp
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: myapp

  fluentd:
    image: fluent/fluentd:v1.16
    ports:
      - "24224:24224"
    volumes:
      - ./fluentd.conf:/fluentd/etc/fluent.conf
    depends_on:
      - elasticsearch

  elasticsearch:
    image: elasticsearch:8.11.3
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es-data:/usr/share/elasticsearch/data

  kibana:
    image: kibana:8.11.3
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

---

### 3. Distributed Tracing

**OpenTelemetry:**
```javascript
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

// Setup tracing
const provider = new NodeTracerProvider();

provider.addSpanProcessor(
  new BatchSpanProcessor(
    new JaegerExporter({
      endpoint: 'http://jaeger:14268/api/traces'
    })
  )
);

provider.register();

// Auto-instrument
registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation()
  ]
});

// Manual spans
const tracer = provider.getTracer('myapp');

app.get('/api/users', async (req, res) => {
  const span = tracer.startSpan('get-users');
  
  try {
    const users = await db.query('SELECT * FROM users');
    span.setAttributes({
      'db.query': 'SELECT * FROM users',
      'db.count': users.length
    });
    res.json(users);
  } catch (error) {
    span.recordException(error);
    span.setStatus({ code: SpanStatusCode.ERROR });
    throw error;
  } finally {
    span.end();
  }
});
```

---

### 4. Dashboards (Grafana)

**Grafana Dashboard:**
```json
{
  "dashboard": {
    "title": "Application Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [{
          "expr": "rate(http_requests_total[5m])"
        }]
      },
      {
        "title": "Response Time (p95)",
        "targets": [{
          "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"
        }]
      },
      {
        "title": "Error Rate",
        "targets": [{
          "expr": "rate(http_requests_total{status_code=~\"5..\"}[5m])"
        }]
      },
      {
        "title": "CPU Usage",
        "targets": [{
          "expr": "rate(process_cpu_seconds_total[5m]) * 100"
        }]
      },
      {
        "title": "Memory Usage",
        "targets": [{
          "expr": "process_resident_memory_bytes / 1024 / 1024"
        }]
      }
    ]
  }
}
```

---

### 5. Alerting

**Alertmanager Config:**
```yaml
route:
  receiver: 'team-emails'
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  routes:
  - match:
      severity: critical
    receiver: 'pagerduty'

receivers:
- name: 'team-emails'
  email_configs:
  - to: 'team@example.com'

- name: 'pagerduty'
  pagerduty_configs:
  - service_key: 'your-key'
```

**Alert Rules:**
```yaml
groups:
- name: application
  rules:
  # High error rate
  - alert: HighErrorRate
    expr: |
      rate(http_requests_total{status_code=~"5.."}[5m]) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value }} (threshold: 0.05)"

  # High response time
  - alert: HighResponseTime
    expr: |
      histogram_quantile(0.95, 
        rate(http_request_duration_seconds_bucket[5m])
      ) > 2
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High response time"
      description: "P95 latency is {{ $value }}s"

  # Container down
  - alert: ContainerDown
    expr: up{job="app"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Container is down"

  # High memory usage
  - alert: HighMemoryUsage
    expr: |
      (process_resident_memory_bytes / 1024 / 1024 / 1024) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High memory usage"
      description: "Memory usage is {{ $value }}GB"
```

---

### Troubleshooting

**1. Check Container Status:**
```bash
# List containers
docker ps -a

# Specific container
docker ps -f name=myapp

# Container logs
docker logs myapp
docker logs -f --tail=100 myapp

# Container stats
docker stats myapp

# Container processes
docker top myapp
```

---

**2. Inspect Container:**
```bash
# Full inspection
docker inspect myapp

# Specific info
docker inspect myapp --format='{{.State.Status}}'
docker inspect myapp --format='{{.RestartCount}}'
docker inspect myapp --format='{{.NetworkSettings.IPAddress}}'

# JSON output
docker inspect myapp | jq '.[0].State'
```

---

**3. Execute Commands:**
```bash
# Interactive shell
docker exec -it myapp /bin/sh

# Run command
docker exec myapp ps aux
docker exec myapp netstat -tlnp
docker exec myapp df -h

# Environment variables
docker exec myapp env
```

---

**4. Network Debugging:**
```bash
# Inside container
docker exec myapp ping api
docker exec myapp nslookup api
docker exec myapp curl http://api:3000/health
docker exec myapp nc -zv api 3000

# Network inspection
docker network inspect myapp_default

# Container IP
docker inspect myapp --format='{{.NetworkSettings.IPAddress}}'
```

---

**5. Resource Usage:**
```bash
# Real-time stats
docker stats

# Historical data (Prometheus)
curl 'http://prometheus:9090/api/v1/query?query=container_memory_usage_bytes{name="myapp"}'

# cAdvisor UI
http://cadvisor:8080/containers/
```

---

**6. Logs Analysis:**
```bash
# Search logs
docker logs myapp 2>&1 | grep -i error

# Count errors
docker logs myapp 2>&1 | grep -c ERROR

# Recent errors
docker logs --since 1h myapp 2>&1 | grep ERROR

# Kibana queries
# Go to http://kibana:5601
# Query: level:"error" AND service:"myapp"
```

---

**7. Health Checks:**
```bash
# Manual health check
curl http://localhost:3000/health

# Container health status
docker inspect myapp --format='{{.State.Health.Status}}'

# Health check logs
docker inspect myapp --format='{{json .State.Health}}' | jq
```

---

**8. Performance Profiling:**
```bash
# Node.js heap snapshot
docker exec myapp kill -USR2 $(docker exec myapp pidof node)
docker cp myapp:/tmp/heapsnapshot-*.heapsnapshot ./

# CPU profiling
docker exec myapp kill -SIGUSR2 $(docker exec myapp pidof node)

# strace
docker exec myapp apk add strace
docker exec myapp strace -p $(docker exec myapp pidof node)
```

---

### Troubleshooting Checklist

```
Container Issues:
☐ Check container status (running/stopped)
☐ Review container logs
☐ Check restart count
☐ Verify health check status
☐ Check resource limits
☐ Review exit code

Network Issues:
☐ Ping other containers
☐ Check DNS resolution
☐ Verify port bindings
☐ Test connectivity
☐ Inspect network config
☐ Check firewall rules

Performance Issues:
☐ Check CPU usage
☐ Check memory usage
☐ Review response times
☐ Check error rates
☐ Analyze slow queries
☐ Review metrics

Application Issues:
☐ Check application logs
☐ Review error messages
☐ Check configuration
☐ Verify environment variables
☐ Test dependencies
☐ Check disk space
```

</details>

</details>

---

[← Previous: 10.1 Dockerfile Best Practices](01-dockerfile-best-practices.md)