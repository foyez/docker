# 8.3 Compose Operations

Advanced Docker Compose operations: multi-file configs, overrides, profiles, and production workflows.

---

## Configuration Files

### Multiple Compose Files

**Merging Configuration:**
```bash
# Base configuration
docker-compose.yml

# Override files
docker-compose.override.yml    # Automatically loaded
docker-compose.prod.yml
docker-compose.dev.yml
```

**Usage:**
```bash
# Automatic merge (base + override)
docker compose up

# Specific files
docker compose -f docker-compose.yml -f docker-compose.prod.yml up

# Multiple overrides
docker compose \
    -f docker-compose.yml \
    -f docker-compose.prod.yml \
    -f docker-compose.monitoring.yml \
    up -d
```

---

### Base Configuration

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  web:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html

  api:
    build: ./api
    environment:
      NODE_ENV: development
```

---

### Override File

**docker-compose.override.yml (automatically loaded):**
```yaml
version: '3.8'

services:
  api:
    command: npm run dev
    volumes:
      - ./api:/app
    environment:
      DEBUG: "*"
```

**Result:**
```yaml
# Merged configuration
services:
  web:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html

  api:
    build: ./api
    command: npm run dev  # From override
    volumes:
      - ./api:/app        # From override
    environment:
      NODE_ENV: development
      DEBUG: "*"          # From override
```

---

### Environment-Specific Files

**docker-compose.dev.yml:**
```yaml
version: '3.8'

services:
  api:
    build:
      context: ./api
      target: development
    volumes:
      - ./api:/app
      - /app/node_modules
    command: npm run dev
    environment:
      NODE_ENV: development
      DEBUG: "*"
    ports:
      - "9229:9229"  # Debug port

  db:
    ports:
      - "5432:5432"  # Expose for development
```

**docker-compose.prod.yml:**
```yaml
version: '3.8'

services:
  api:
    build:
      context: ./api
      target: production
    command: node server.js
    environment:
      NODE_ENV: production
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

  db:
    # No ports exposed in production
    deploy:
      resources:
        limits:
          memory: 2G
```

**Usage:**
```bash
# Development
docker compose -f docker-compose.yml -f docker-compose.dev.yml up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Profiles

**Feature:** Run different sets of services.

### Basic Profiles

```yaml
version: '3.8'

services:
  # Always runs
  web:
    image: nginx

  # Only with 'debug' profile
  debug:
    image: nicolaka/netshoot
    profiles:
      - debug
    command: sleep infinity

  # Only with 'tools' profile
  adminer:
    image: adminer
    profiles:
      - tools
    ports:
      - "8080:8080"
```

**Usage:**
```bash
# Default (only web)
docker compose up

# With debug profile
docker compose --profile debug up

# With tools profile
docker compose --profile tools up

# Multiple profiles
docker compose --profile debug --profile tools up
```

---

### Real-World Example

```yaml
version: '3.8'

services:
  # Core services (always run)
  api:
    image: myapi
    ports:
      - "3000:3000"

  db:
    image: postgres

  # Development tools
  pgadmin:
    image: dpage/pgadmin4
    profiles:
      - dev
    ports:
      - "5050:80"

  # Monitoring
  prometheus:
    image: prom/prometheus
    profiles:
      - monitoring
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    profiles:
      - monitoring
    ports:
      - "3001:3000"

  # Testing
  test:
    build: ./tests
    profiles:
      - test
    command: npm test
```

**Usage:**
```bash
# Production (core only)
docker compose up -d

# Development (core + pgadmin)
docker compose --profile dev up -d

# Full monitoring
docker compose --profile monitoring up -d

# Run tests
docker compose --profile test up --abort-on-container-exit
```

---

## Environment Variables

### .env File

**.env:**
```env
# Application
APP_VERSION=1.0.0
NODE_ENV=production

# Database
POSTGRES_VERSION=15
POSTGRES_DB=myapp
POSTGRES_USER=admin

# Ports
WEB_PORT=80
API_PORT=3000
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  web:
    image: nginx:${NGINX_VERSION:-alpine}
    ports:
      - "${WEB_PORT}:80"

  api:
    image: myapi:${APP_VERSION}
    ports:
      - "${API_PORT}:3000"
    environment:
      NODE_ENV: ${NODE_ENV}

  db:
    image: postgres:${POSTGRES_VERSION}
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
```

---

### Multiple .env Files

```bash
# Different environments
.env                 # Default
.env.local          # Local overrides (gitignored)
.env.development
.env.production
```

**Load specific file:**
```bash
docker compose --env-file .env.production up
```

---

### Variable Precedence

**Order (highest to lowest):**
```
1. Shell environment variables
2. Environment variables in docker-compose.yml
3. .env file
4. Dockerfile ENV
```

**Example:**
```bash
# Shell
export API_PORT=4000

# .env
API_PORT=3000

# docker-compose.yml uses ${API_PORT}
# Result: 4000 (shell takes precedence)
```

---

## Extends (Deprecated but still used)

**Base service:**
```yaml
# common-services.yml
version: '3.8'

services:
  base-api:
    image: node:18
    working_dir: /app
    environment:
      NODE_ENV: production
```

**Extending:**
```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    extends:
      file: common-services.yml
      service: base-api
    ports:
      - "3000:3000"
    volumes:
      - ./api:/app
```

**Modern Alternative (Anchors & Aliases):**
```yaml
version: '3.8'

x-common-api: &common-api
  image: node:18
  working_dir: /app
  environment:
    NODE_ENV: production

services:
  api:
    <<: *common-api
    ports:
      - "3000:3000"

  worker:
    <<: *common-api
    command: npm run worker
```

---

## YAML Anchors and Aliases

### Basic Anchors

```yaml
version: '3.8'

x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  api:
    image: myapi
    logging: *default-logging

  worker:
    image: myworker
    logging: *default-logging

  db:
    image: postgres
    logging: *default-logging
```

---

### Complex Anchors

```yaml
version: '3.8'

# Define templates
x-api-template: &api-template
  build:
    context: ./api
  environment: &api-env
    NODE_ENV: production
    LOG_LEVEL: info
  deploy: &api-deploy
    resources:
      limits:
        cpus: '1.0'
        memory: 1G
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"

services:
  api-1:
    <<: *api-template
    ports:
      - "3000:3000"

  api-2:
    <<: *api-template
    ports:
      - "3001:3000"

  worker:
    <<: *api-template
    command: npm run worker
    # No ports for worker
```

---

### Merging Anchors

```yaml
version: '3.8'

x-base: &base
  restart: unless-stopped
  logging:
    driver: json-file

x-api: &api
  <<: *base
  image: myapi
  environment:
    NODE_ENV: production

services:
  api:
    <<: *api
    ports:
      - "3000:3000"
```

---

## Project Name

**Default:** Directory name

**Set project name:**
```bash
# Via flag
docker compose -p myproject up

# Via environment variable
export COMPOSE_PROJECT_NAME=myproject
docker compose up

# In .env file
COMPOSE_PROJECT_NAME=myproject
```

**Impact:**
```
Default:
- Project: mydirectory
- Containers: mydirectory-web-1, mydirectory-db-1
- Network: mydirectory_default

Custom (myproject):
- Containers: myproject-web-1, myproject-db-1
- Network: myproject_default
```

---

## Working Directory

**Change directory:**
```bash
# Run from different directory
docker compose --project-directory /path/to/project up

# Or use -f with absolute paths
docker compose -f /path/to/docker-compose.yml up
```

---

## Validation and Configuration

### Validate Config

```bash
# Validate syntax
docker compose config

# Quiet validation
docker compose config --quiet

# Exit code 0 = valid, 1 = invalid
```

---

### View Merged Config

```bash
# View final configuration
docker compose config

# With overrides
docker compose -f docker-compose.yml -f docker-compose.prod.yml config

# Output to file
docker compose config > final-config.yml
```

---

### Resolve Variables

```bash
# Show with resolved variables
docker compose config

# Resolve without interpolation
docker compose config --no-interpolate
```

---

## Service Selection

**Run specific services:**
```bash
# Start only web
docker compose up web

# Start web and dependencies
docker compose up web

# Start multiple
docker compose up web api

# Exclude services
docker compose up --scale worker=0
```

---

## Lifecycle Operations

### Up Variations

```bash
# Standard
docker compose up -d

# Build before starting
docker compose up -d --build

# Force recreate
docker compose up -d --force-recreate

# No recreate
docker compose up -d --no-recreate

# Remove orphans
docker compose up -d --remove-orphans

# Abort on any container exit
docker compose up --abort-on-container-exit

# Exit after all stop
docker compose up --exit-code-from web
```

---

### Down Variations

```bash
# Standard
docker compose down

# Remove volumes
docker compose down -v

# Remove images
docker compose down --rmi all     # All images
docker compose down --rmi local   # Only built images

# Timeout
docker compose down -t 30         # 30 seconds
```

---

## Scaling

```bash
# Scale service
docker compose up -d --scale worker=5

# Multiple services
docker compose up -d --scale api=3 --scale worker=5

# Scale down
docker compose up -d --scale worker=2
```

**Constraints:**
```yaml
# Cannot scale with container_name
services:
  web:
    image: nginx
    container_name: my-web  # ✗ Cannot scale

# Can scale without container_name
services:
  worker:
    image: myworker  # ✓ Can scale
```

---

## Resource Management

### Pull Images

```bash
# Pull all images
docker compose pull

# Pull specific service
docker compose pull api

# Ignore pull failures
docker compose pull --ignore-pull-failures

# Quiet mode
docker compose pull -q
```

---

### Build Images

```bash
# Build all
docker compose build

# Build specific service
docker compose build api

# No cache
docker compose build --no-cache

# Parallel build
docker compose build --parallel

# Pull base images
docker compose build --pull
```

---

### Push Images

```bash
# Push all images
docker compose push

# Push specific service
docker compose push api

# Ignore push failures
docker compose push --ignore-push-failures
```

---

## Execution

### Run One-off Commands

```bash
# Run command (creates new container)
docker compose run api npm test

# Without dependencies
docker compose run --no-deps api npm test

# Remove after run
docker compose run --rm api npm test

# Custom entrypoint
docker compose run --entrypoint /bin/sh api

# Publish ports
docker compose run -p 8080:3000 api

# As different user
docker compose run -u root api bash
```

---

### Execute in Running Container

```bash
# Execute command
docker compose exec api npm run migrate

# Interactive shell
docker compose exec api bash

# As root
docker compose exec -u root api bash

# Without TTY
docker compose exec -T api cat /etc/hosts

# Custom working directory
docker compose exec -w /tmp api ls
```

---

## Monitoring

### View Logs

```bash
# All services
docker compose logs

# Specific service
docker compose logs api

# Follow
docker compose logs -f

# Tail
docker compose logs --tail=100

# Timestamps
docker compose logs -t

# Since
docker compose logs --since 2024-01-17T10:00:00
```

---

### Process Status

```bash
# List containers
docker compose ps

# All (including stopped)
docker compose ps -a

# Quiet (IDs only)
docker compose ps -q

# Services only
docker compose ps --services

# Filter
docker compose ps --filter status=running
```

---

### Top (Processes)

```bash
# View processes
docker compose top

# Specific service
docker compose top api
```

---

### Events

```bash
# Stream events
docker compose events

# JSON format
docker compose events --json

# Specific service
docker compose events api
```

---

## Pause and Unpause

```bash
# Pause all services
docker compose pause

# Pause specific service
docker compose pause api

# Unpause
docker compose unpause
docker compose unpause api
```

---

## Complete Workflow Example

### Development Workflow

**Project structure:**
```
myapp/
├── docker-compose.yml
├── docker-compose.override.yml
├── docker-compose.prod.yml
├── .env
├── .env.production
└── api/
    └── Dockerfile
```

**docker-compose.yml (base):**
```yaml
version: '3.8'

services:
  api:
    build: ./api
    image: myapi:${VERSION:-latest}
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s

volumes:
  db-data:
```

**docker-compose.override.yml (auto-loaded for dev):**
```yaml
version: '3.8'

services:
  api:
    command: npm run dev
    volumes:
      - ./api:/app
      - /app/node_modules
    ports:
      - "3000:3000"
      - "9229:9229"  # Debug
    environment:
      DEBUG: "*"

  db:
    ports:
      - "5432:5432"
```

**docker-compose.prod.yml:**
```yaml
version: '3.8'

services:
  api:
    command: node server.js
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

  db:
    deploy:
      resources:
        limits:
          memory: 2G
```

**Usage:**
```bash
# Development (uses override automatically)
docker compose up -d
docker compose logs -f api

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# With environment file
docker compose --env-file .env.production \
    -f docker-compose.yml \
    -f docker-compose.prod.yml \
    up -d
```

---

## Best Practices

### 1. File Organization

```
✓ Base config in docker-compose.yml
✓ Dev overrides in docker-compose.override.yml
✓ Env-specific in docker-compose.{env}.yml
✓ Secrets in .env (gitignored)
```

### 2. Environment Variables

```
✓ Use .env for defaults
✓ Override with environment-specific files
✓ Never commit secrets
✓ Document required variables
```

### 3. Service Dependencies

```
✓ Use depends_on with conditions
✓ Implement health checks
✓ Handle startup failures gracefully
```

### 4. Resource Limits

```
✓ Set memory limits in production
✓ Set CPU limits for critical services
✓ Monitor resource usage
```

### 5. Logging

```
✓ Configure log rotation
✓ Use structured logging
✓ Centralize logs in production
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The file __________ is automatically merged with docker-compose.yml.
2. Use __________ to run only specific service groups.
3. YAML __________ allow reusing configuration blocks.
4. The __________ command shows the merged configuration.
5. Use __________ to run a one-off command in a new container.
6. The __________ sets which services start together.

### True/False

1. ⬜ docker-compose.override.yml is automatically loaded
2. ⬜ Multiple -f flags merge configurations left to right
3. ⬜ Profiles allow running different service sets
4. ⬜ Shell environment variables override .env file
5. ⬜ YAML anchors are a Docker feature
6. ⬜ docker compose run creates a new container
7. ⬜ You can scale services with container_name set

### Multiple Choice

1. Which file is auto-loaded?
   - A) docker-compose.prod.yml
   - B) docker-compose.override.yml
   - C) docker-compose.dev.yml
   - D) None

2. How to view merged config?
   - A) docker compose show
   - B) docker compose config
   - C) docker compose view
   - D) docker compose inspect

3. What are YAML anchors marked with?
   - A) *
   - B) &
   - C) #
   - D) @

4. Which starts services with profile "debug"?
   - A) docker compose up debug
   - B) docker compose --profile debug up
   - C) docker compose up --profile debug
   - D) Both B and C

5. docker compose run vs exec?
   - A) Same thing
   - B) run creates new, exec uses existing
   - C) run is faster
   - D) exec is deprecated

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. docker-compose.override.yml
2. profiles
3. anchors (and aliases)
4. docker compose config
5. docker compose run
6. profile (or depends_on)

**True/False:**
1. ✅ True - override.yml is automatically merged
2. ✅ True - Files merge in order specified
3. ✅ True - Profiles enable service grouping
4. ✅ True - Shell variables have highest precedence
5. ❌ False - YAML anchors are a YAML feature
6. ✅ True - run creates new container
7. ❌ False - Cannot scale with container_name

**Multiple Choice:**
1. **B** - docker-compose.override.yml
2. **B** - docker compose config
3. **B** - & (anchors defined with &, referenced with *)
4. **D** - Both B and C work
5. **B** - run creates new, exec uses existing

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: How do you manage different environments (dev, staging, prod) with Compose?

<details>
<summary><strong>View Answer</strong></summary>

**Multi-Environment Strategy:**

---

### Approach 1: Multiple Compose Files

**File Structure:**
```
project/
├── docker-compose.yml           # Base config
├── docker-compose.dev.yml       # Development
├── docker-compose.staging.yml   # Staging
├── docker-compose.prod.yml      # Production
├── .env                         # Default variables
├── .env.dev
├── .env.staging
└── .env.prod
```

**docker-compose.yml (base):**
```yaml
version: '3.8'

services:
  api:
    build:
      context: ./api
      args:
        NODE_VERSION: ${NODE_VERSION:-18}
    image: myapi:${VERSION:-latest}
    environment:
      DATABASE_URL: postgresql://db:5432/${POSTGRES_DB}
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:${POSTGRES_VERSION:-15}
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-myapp}
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s

volumes:
  db-data:
```

**docker-compose.dev.yml:**
```yaml
version: '3.8'

services:
  api:
    build:
      target: development
    command: npm run dev
    volumes:
      - ./api:/app
      - /app/node_modules
    ports:
      - "3000:3000"
      - "9229:9229"  # Debugger
    environment:
      NODE_ENV: development
      DEBUG: "*"
      LOG_LEVEL: debug

  db:
    ports:
      - "5432:5432"  # Expose for development tools

  # Development tools
  adminer:
    image: adminer
    ports:
      - "8080:8080"
```

**docker-compose.staging.yml:**
```yaml
version: '3.8'

services:
  api:
    build:
      target: production
    command: node server.js
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    environment:
      NODE_ENV: staging
      LOG_LEVEL: info
    restart: unless-stopped

  db:
    deploy:
      resources:
        limits:
          memory: 1G
```

**docker-compose.prod.yml:**
```yaml
version: '3.8'

services:
  api:
    build:
      target: production
    command: node server.js
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
      LOG_LEVEL: warn
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

  db:
    deploy:
      resources:
        limits:
          memory: 2G
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  # Monitoring (production only)
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
```

**Environment Files:**

**.env.dev:**
```env
NODE_VERSION=18
POSTGRES_VERSION=15
POSTGRES_DB=myapp_dev
VERSION=dev
```

**.env.staging:**
```env
NODE_VERSION=18
POSTGRES_VERSION=15
POSTGRES_DB=myapp_staging
VERSION=staging-${CI_COMMIT_SHA}
```

**.env.prod:**
```env
NODE_VERSION=18
POSTGRES_VERSION=15
POSTGRES_DB=myapp_production
VERSION=v${RELEASE_VERSION}
```

**Usage:**
```bash
# Development
docker compose \
    --env-file .env.dev \
    -f docker-compose.yml \
    -f docker-compose.dev.yml \
    up -d

# Staging
docker compose \
    --env-file .env.staging \
    -f docker-compose.yml \
    -f docker-compose.staging.yml \
    up -d

# Production
docker compose \
    --env-file .env.prod \
    -f docker-compose.yml \
    -f docker-compose.prod.yml \
    up -d
```

---

### Approach 2: Using Profiles

```yaml
version: '3.8'

services:
  api:
    build: ./api
    environment:
      NODE_ENV: ${NODE_ENV:-development}

  db:
    image: postgres

  # Development tools
  adminer:
    image: adminer
    profiles:
      - dev
    ports:
      - "8080:8080"

  # Staging monitoring
  monitoring:
    image: prom/prometheus
    profiles:
      - staging
      - prod

  # Production scaling
  api-worker:
    build: ./api
    command: npm run worker
    profiles:
      - prod
    deploy:
      replicas: 3
```

**Usage:**
```bash
# Development
NODE_ENV=development docker compose --profile dev up -d

# Staging
NODE_ENV=staging docker compose --profile staging up -d

# Production
NODE_ENV=production docker compose --profile prod up -d
```

---

### Approach 3: Wrapper Scripts

**deploy.sh:**
```bash
#!/bin/bash

ENV=$1

if [ -z "$ENV" ]; then
    echo "Usage: $0 {dev|staging|prod}"
    exit 1
fi

case $ENV in
    dev)
        export COMPOSE_FILE="docker-compose.yml:docker-compose.dev.yml"
        export COMPOSE_ENV_FILE=".env.dev"
        export COMPOSE_PROJECT_NAME="myapp-dev"
        ;;
    staging)
        export COMPOSE_FILE="docker-compose.yml:docker-compose.staging.yml"
        export COMPOSE_ENV_FILE=".env.staging"
        export COMPOSE_PROJECT_NAME="myapp-staging"
        ;;
    prod)
        export COMPOSE_FILE="docker-compose.yml:docker-compose.prod.yml"
        export COMPOSE_ENV_FILE=".env.prod"
        export COMPOSE_PROJECT_NAME="myapp-prod"
        ;;
    *)
        echo "Invalid environment: $ENV"
        exit 1
        ;;
esac

echo "Deploying to $ENV environment..."
echo "Files: $COMPOSE_FILE"
echo "Env file: $COMPOSE_ENV_FILE"

docker compose --env-file $COMPOSE_ENV_FILE up -d

echo "Deployment complete!"
```

**Usage:**
```bash
./deploy.sh dev
./deploy.sh staging
./deploy.sh prod
```

---

### Approach 4: Makefile

**Makefile:**
```makefile
.PHONY: dev staging prod clean

# Variables
COMPOSE_FILES_DEV := -f docker-compose.yml -f docker-compose.dev.yml
COMPOSE_FILES_STAGING := -f docker-compose.yml -f docker-compose.staging.yml
COMPOSE_FILES_PROD := -f docker-compose.yml -f docker-compose.prod.yml

dev:
	docker compose --env-file .env.dev $(COMPOSE_FILES_DEV) up -d
	@echo "Development environment started"

staging:
	docker compose --env-file .env.staging $(COMPOSE_FILES_STAGING) up -d
	@echo "Staging environment started"

prod:
	docker compose --env-file .env.prod $(COMPOSE_FILES_PROD) up -d
	@echo "Production environment started"

dev-logs:
	docker compose --env-file .env.dev $(COMPOSE_FILES_DEV) logs -f

staging-logs:
	docker compose --env-file .env.staging $(COMPOSE_FILES_STAGING) logs -f

prod-logs:
	docker compose --env-file .env.prod $(COMPOSE_FILES_PROD) logs -f

clean:
	docker compose down -v
	@echo "Environment cleaned"
```

**Usage:**
```bash
make dev
make staging
make prod

make dev-logs
make staging-logs
```

---

### Best Practices

```
1. Separation of Concerns
   ✓ Base config for all environments
   ✓ Environment-specific overrides
   ✓ Never duplicate configuration

2. Environment Variables
   ✓ Different .env files per environment
   ✓ Never commit secrets
   ✓ Use CI/CD secrets for sensitive data

3. Resource Limits
   ✓ Lower limits in dev/staging
   ✓ Production limits match infrastructure
   ✓ Test with production-like resources

4. Service Scaling
   ✓ Single instance in dev
   ✓ Multiple instances in staging/prod
   ✓ Match production topology in staging

5. Monitoring
   ✓ Optional in development
   ✓ Required in staging/production
   ✓ Same tools across environments

6. Deployment
   ✓ Automate with scripts or Makefile
   ✓ Validate config before deploy
   ✓ Use version tags, not :latest
```

</details>

### Question 2: How do you perform zero-downtime deployments with Compose?

<details>
<summary><strong>View Answer</strong></summary>

**Zero-Downtime Deployment Strategies:**

---

### Strategy 1: Rolling Update with Health Checks

```yaml
version: '3.8'

services:
  web:
    image: nginx
    ports:
      - "80:80"
    depends_on:
      - api

  api:
    image: myapi:${VERSION}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 5s
      timeout: 3s
      retries: 3
      start_period: 40s
    environment:
      NODE_ENV: production
```

**Deployment:**
```bash
# Update image version
export VERSION=v2.0.0

# Rolling update
docker compose up -d --no-deps --scale api=6 api

# Wait for new instances to be healthy
sleep 30

# Scale down old instances
docker compose up -d --no-deps --scale api=3 api
```

---

### Strategy 2: Blue-Green Deployment

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  nginx:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - api-blue

  # Blue (current production)
  api-blue:
    image: myapi:v1.0.0
    deploy:
      replicas: 3
    environment:
      NODE_ENV: production
      VERSION: blue
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s

  # Green (new version)
  api-green:
    image: myapi:v2.0.0
    deploy:
      replicas: 3
    environment:
      NODE_ENV: production
      VERSION: green
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
    profiles:
      - green
```

**nginx.conf (initial):**
```nginx
upstream backend {
    server api-blue:3000;
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
# 1. Deploy green version
docker compose --profile green up -d api-green

# 2. Test green version
curl http://api-green:3000/health

# 3. Update nginx to point to green
# nginx.conf:
#   upstream backend {
#       server api-green:3000;
#   }

# 4. Reload nginx
docker compose exec nginx nginx -s reload

# 5. Monitor for issues
docker compose logs -f api-green

# 6. If all good, remove blue
docker compose stop api-blue
docker compose rm -f api-blue

# 7. If issues, rollback
# Revert nginx.conf to api-blue
docker compose exec nginx nginx -s reload
```

---

### Strategy 3: Canary Deployment

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  nginx:
    image: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  # Stable version (90%)
  api-stable:
    image: myapi:v1.0.0
    deploy:
      replicas: 9
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]

  # Canary version (10%)
  api-canary:
    image: myapi:v2.0.0
    deploy:
      replicas: 1
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
```

**nginx.conf (weighted load balancing):**
```nginx
upstream backend {
    server api-stable:3000 weight=9;
    server api-canary:3000 weight=1;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
    }
}
```

**Gradual Rollout:**
```bash
# Phase 1: 10% canary
docker compose up -d --scale api-stable=9 --scale api-canary=1

# Monitor metrics
watch -n 5 'docker compose logs --tail=50 api-canary'

# Phase 2: 50% canary (if all good)
docker compose up -d --scale api-stable=5 --scale api-canary=5

# Phase 3: 100% canary
docker compose up -d --scale api-stable=0 --scale api-canary=10

# Phase 4: Promote canary to stable
docker compose down api-stable
```

---

### Strategy 4: Load Balancer with Health Checks

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  api:
    image: myapi:${VERSION}
    deploy:
      replicas: 3
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=PathPrefix(`/api`)"
      - "traefik.http.services.api.loadbalancer.healthcheck.path=/health"
      - "traefik.http.services.api.loadbalancer.healthcheck.interval=10s"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
```

**Deployment:**
```bash
# Update version
export VERSION=v2.0.0

# Deploy new version
# Traefik automatically:
# 1. Starts routing to new containers as they become healthy
# 2. Stops routing to old containers
# 3. Removes old containers
docker compose up -d
```

---

### Strategy 5: Database Migrations with Zero Downtime

**Backward Compatible Migrations:**

```yaml
version: '3.8'

services:
  # Run migrations before deployment
  migrations:
    image: myapi:${VERSION}
    command: npm run migrate
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  # Old version (supports both schemas)
  api-old:
    image: myapi:v1.0.0
    deploy:
      replicas: 3
    depends_on:
      migrations:
        condition: service_completed_successfully

  # New version
  api-new:
    image: myapi:v2.0.0
    deploy:
      replicas: 0  # Start at 0
    depends_on:
      migrations:
        condition: service_completed_successfully

  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
```

**Migration Process:**
```bash
# Step 1: Deploy backward-compatible migration
export VERSION=v2.0.0
docker compose up -d migrations

# Step 2: Verify migration success
docker compose logs migrations

# Step 3: Gradually shift traffic to new version
docker compose up -d --scale api-new=1  # 1 new, 3 old
sleep 30
docker compose up -d --scale api-new=2 --scale api-old=2  # 2 new, 2 old
sleep 30
docker compose up -d --scale api-new=3 --scale api-old=1  # 3 new, 1 old
sleep 30
docker compose up -d --scale api-new=3 --scale api-old=0  # 3 new, 0 old

# Step 4: Remove old version
docker compose stop api-old
docker compose rm -f api-old
```

---

### Complete Example with Automation

**deploy.sh:**
```bash
#!/bin/bash
set -e

NEW_VERSION=$1
OLD_VERSION=$(docker compose ps -q api | head -1 | xargs docker inspect --format='{{.Config.Image}}')

echo "Current version: $OLD_VERSION"
echo "New version: $NEW_VERSION"

# Health check function
health_check() {
    local service=$1
    local max_attempts=30
    local attempt=0
    
    while [ $attempt -lt $max_attempts ]; do
        if docker compose exec -T $service curl -f http://localhost:3000/health; then
            echo "✓ $service is healthy"
            return 0
        fi
        echo "Waiting for $service to be healthy... ($attempt/$max_attempts)"
        sleep 10
        ((attempt++))
    done
    
    echo "✗ $service failed health check"
    return 1
}

# Deploy new version alongside old
echo "Deploying new version..."
export VERSION=$NEW_VERSION
docker compose up -d --scale api=6 --no-deps api

# Wait for new instances
sleep 30

# Verify health
if ! health_check api; then
    echo "Deployment failed! Rolling back..."
    docker compose up -d --scale api=3 --no-deps api
    exit 1
fi

# Scale down old instances
echo "Scaling down old instances..."
docker compose up -d --scale api=3 --no-deps api

echo "Deployment successful!"
```

**Usage:**
```bash
./deploy.sh myapi:v2.0.0
```

---

### Best Practices

```
1. Health Checks
   ✓ Implement comprehensive health endpoints
   ✓ Check dependencies (database, cache)
   ✓ Monitor during deployment

2. Gradual Rollout
   ✓ Start with small percentage
   ✓ Monitor metrics closely
   ✓ Increase gradually

3. Rollback Plan
   ✓ Keep old version running
   ✓ Automate rollback
   ✓ Test rollback procedure

4. Database Migrations
   ✓ Backward compatible changes
   ✓ Deploy schema before code
   ✓ Multiple deployment phases

5. Monitoring
   ✓ Error rate metrics
   ✓ Response time tracking
   ✓ Automated alerts

6. Testing
   ✓ Smoke tests after deployment
   ✓ Canary analysis
   ✓ Synthetic transactions
```

</details>

### Question 3: How do you troubleshoot performance issues in Compose applications?

<details>
<summary><strong>View Answer</strong></summary>

**Performance Troubleshooting Guide:**

---

### Step 1: Identify the Problem

**Check Resource Usage:**
```bash
# Overall stats
docker compose ps -q | xargs docker stats

# Specific service
docker stats api-1

# Sort by memory
docker stats --no-stream --format "table {{.Container}}\t{{.MemUsage}}" | sort -k 2 -h

# Sort by CPU
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}" | sort -k 2 -h
```

**Output:**
```
CONTAINER   CPU %   MEM USAGE / LIMIT     NET I/O
api-1       45.2%   512MB / 1GB          1.2MB / 800KB
api-2       42.1%   498MB / 1GB          1.1MB / 750KB
db-1        15.3%   1.5GB / 2GB          500KB / 300KB
```

---

### Step 2: Analyze Logs

**Check for Errors:**
```bash
# Recent errors
docker compose logs --tail=1000 api | grep -i error

# Slow queries
docker compose logs db | grep "duration:"

# Memory issues
docker compose logs api | grep -i "out of memory\|heap"

# Connection issues
docker compose logs api | grep -i "connection\|timeout"
```

---

### Step 3: Database Performance

**Check Queries:**
```bash
# Enable query logging
docker compose exec db psql -U postgres -c \
    "ALTER SYSTEM SET log_min_duration_statement = 1000;"

docker compose restart db

# View slow queries
docker compose logs db | grep "duration:"
```

**Check Connections:**
```bash
# Active connections
docker compose exec db psql -U postgres -c \
    "SELECT count(*) FROM pg_stat_activity;"

# Connection details
docker compose exec db psql -U postgres -c \
    "SELECT pid, usename, application_name, client_addr, state, query 
     FROM pg_stat_activity 
     WHERE state != 'idle';"
```

**Index Usage:**
```bash
# Missing indexes
docker compose exec db psql -U postgres -d myapp -c \
    "SELECT schemaname, tablename, indexname 
     FROM pg_indexes 
     WHERE indexname NOT LIKE '%_pkey';"

# Unused indexes
docker compose exec db psql -U postgres -d myapp -c \
    "SELECT schemaname, tablename, indexname, idx_scan 
     FROM pg_stat_user_indexes 
     WHERE idx_scan = 0;"
```

---

### Step 4: Network Performance

**Test Network Speed:**
```bash
# Install iperf
docker compose exec api apk add iperf3

# Server side
docker compose exec db iperf3 -s

# Client side (from another terminal)
docker compose exec api iperf3 -c db

# DNS resolution time
docker compose exec api time nslookup db
```

**Check Network Issues:**
```bash
# Packet loss
docker compose exec api ping -c 100 db | grep loss

# Latency
docker compose exec api ping -c 10 db | tail -1
```

---

### Step 5: Resource Limits

**Check Current Limits:**
```bash
# Memory limit
docker inspect api-1 --format='{{.HostConfig.Memory}}'

# CPU limit
docker inspect api-1 --format='{{.HostConfig.CpuQuota}}'
```

**Adjust Limits:**
```yaml
version: '3.8'

services:
  api:
    image: myapi
    deploy:
      resources:
        limits:
          cpus: '2.0'      # Increase from 1.0
          memory: 2G       # Increase from 1G
        reservations:
          cpus: '1.0'
          memory: 1G
```

---

### Step 6: Volume Performance

**Check I/O:**
```bash
# Install iostat
docker compose exec api apk add sysstat

# Monitor I/O
docker compose exec api iostat -x 1
```

**Volume Driver:**
```yaml
volumes:
  db-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/fast-ssd/db-data  # Use faster storage
```

---

### Step 7: Application Profiling

**Node.js Profiling:**
```yaml
services:
  api:
    image: myapi
    command: node --inspect=0.0.0.0:9229 server.js
    ports:
      - "9229:9229"
```

**Connect Chrome DevTools:**
```
chrome://inspect
```

**Memory Heap Snapshot:**
```bash
# Take heap snapshot
docker compose exec api kill -USR2 $(docker compose exec api pidof node)

# Copy snapshot
docker compose cp api:/app/heapdump.heapsnapshot ./
```

---

### Step 8: Database Connection Pooling

**Before (No Pooling):**
```javascript
// Each request creates new connection
app.get('/users', async (req, res) => {
  const client = new Client({
    host: 'db',
    database: 'myapp'
  });
  await client.connect();
  const result = await client.query('SELECT * FROM users');
  await client.end();
  res.json(result.rows);
});
```

**After (With Pooling):**
```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: 'db',
  database: 'myapp',
  max: 20,  // Maximum connections
  idleTimeoutMillis: 30000
});

app.get('/users', async (req, res) => {
  const result = await pool.query('SELECT * FROM users');
  res.json(result.rows);
});
```

---

### Step 9: Caching

**Add Redis Cache:**
```yaml
services:
  api:
    environment:
      REDIS_URL: redis://redis:6379

  redis:
    image: redis:7-alpine
    deploy:
      resources:
        limits:
          memory: 256M
```

**Implement Caching:**
```javascript
const redis = require('redis');
const client = redis.createClient({ url: process.env.REDIS_URL });

app.get('/users', async (req, res) => {
  // Check cache
  const cached = await client.get('users');
  if (cached) {
    return res.json(JSON.parse(cached));
  }
  
  // Query database
  const result = await pool.query('SELECT * FROM users');
  
  // Cache result
  await client.setEx('users', 3600, JSON.stringify(result.rows));
  
  res.json(result.rows);
});
```

---

### Step 10: Load Balancing

**Add More Replicas:**
```yaml
services:
  api:
    image: myapi
    deploy:
      replicas: 5  # Increase from 3
```

**Monitor Distribution:**
```bash
# Request distribution
for i in {1..100}; do
    curl -s http://localhost/api/health | grep hostname
done | sort | uniq -c
```

---

### Complete Performance Optimization

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
    deploy:
      resources:
        limits:
          memory: 128M

  api:
    image: myapi
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
    environment:
      DATABASE_URL: postgresql://db:5432/myapp
      REDIS_URL: redis://redis:6379
      NODE_ENV: production
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s

  db:
    image: postgres:15
    command:
      - postgres
      - -c
      - max_connections=200
      - -c
      - shared_buffers=256MB
      - -c
      - effective_cache_size=1GB
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      POSTGRES_SHARED_PRELOAD_LIBRARIES: pg_stat_statements

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    deploy:
      resources:
        limits:
          memory: 256M

volumes:
  db-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/fast-ssd/db-data
```

---

### Monitoring Setup

```yaml
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
```

**prometheus.yml:**
```yaml
scrape_configs:
  - job_name: 'docker'
    static_configs:
      - targets: ['host.docker.internal:9323']

  - job_name: 'api'
    static_configs:
      - targets: ['api:3000']
```

---

### Performance Checklist

```
Application:
☐ Connection pooling enabled
☐ Caching implemented
☐ Queries optimized
☐ Indexes created
☐ N+1 queries eliminated

Infrastructure:
☐ Resource limits set appropriately
☐ Fast storage for databases
☐ Sufficient replicas
☐ Load balancing configured
☐ Network optimized

Database:
☐ Proper indexes
☐ Query performance analyzed
☐ Connection pool sized
☐ Statistics up to date
☐ Vacuuming configured

Monitoring:
☐ Metrics collected
☐ Dashboards created
☐ Alerts configured
☐ Logs centralized
☐ Profiling available
```

</details>

</details>

---

[← Previous: 8.2 Compose Services](02-compose-services.md)