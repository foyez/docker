# 3.2 Dockerfile

Understanding Dockerfile: the blueprint for building Docker images.

---

## What is a Dockerfile?

A **Dockerfile** is a text file containing instructions to build a Docker image.

```
Dockerfile = Recipe for creating an image

Instructions:
├── What base to start from (FROM)
├── What to copy (COPY, ADD)
├── What to run (RUN)
├── What to expose (EXPOSE)
├── What environment to set (ENV)
└── What to execute (CMD, ENTRYPOINT)
```

### Basic Structure

```dockerfile
# Comment
INSTRUCTION arguments

# Example
FROM ubuntu:20.04
RUN apt-get update
COPY app.py /app/
CMD ["python", "/app/app.py"]
```

---

## Core Instructions

### FROM - Base Image

**Purpose:** Specifies the base image to build from.

**Syntax:**
```dockerfile
FROM image:tag
FROM image:tag AS stage-name  # Multi-stage builds
```

**Examples:**
```dockerfile
# Official base images
FROM ubuntu:20.04
FROM python:3.9
FROM node:18-alpine
FROM nginx:latest

# Minimal base
FROM alpine:3.19
FROM scratch  # Empty base (for static binaries)

# Multi-stage builds
FROM golang:1.21 AS builder
FROM alpine:latest AS runtime
```

**Real-World Example:**
```dockerfile
# Web application
FROM node:18-alpine

# Why alpine?
# - Smaller size: 40 MB vs 300 MB
# - Faster downloads
# - Smaller attack surface
# - Production-ready
```

---

### RUN - Execute Commands

**Purpose:** Execute commands during image build.

**Syntax:**
```dockerfile
# Shell form (runs in shell: /bin/sh -c)
RUN command

# Exec form (runs directly, no shell)
RUN ["executable", "param1", "param2"]
```

**Examples:**

**Installing Packages:**
```dockerfile
# Ubuntu/Debian
RUN apt-get update && \
    apt-get install -y \
        python3 \
        python3-pip \
        curl \
    && rm -rf /var/lib/apt/lists/*

# Alpine
RUN apk add --no-cache \
    python3 \
    py3-pip \
    curl
```

**Building Application:**
```dockerfile
# Compile Go application
RUN go build -o server .

# Build Node.js application
RUN npm ci --only=production

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt
```

**Best Practices:**
```dockerfile
# Bad: Multiple layers, cache not cleaned
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y curl
# Leaves package cache

# Good: One layer, cache cleaned
RUN apt-get update && \
    apt-get install -y \
        python3 \
        curl \
    && rm -rf /var/lib/apt/lists/*
```

---

### COPY - Copy Files

**Purpose:** Copy files from build context to image.

**Syntax:**
```dockerfile
COPY <src> <dest>
COPY ["<src>", "<dest>"]  # For paths with spaces
```

**Examples:**
```dockerfile
# Copy single file
COPY app.py /app/

# Copy directory
COPY src/ /app/src/

# Copy multiple files
COPY package.json package-lock.json /app/

# Copy with wildcards
COPY *.py /app/

# Copy and rename
COPY config.prod.json /app/config.json

# Copy from specific stage (multi-stage)
COPY --from=builder /app/dist /app/dist
```

**Ownership:**
```dockerfile
# Copy with specific user/group
COPY --chown=node:node package*.json /app/

# Copy as specific UID:GID
COPY --chown=1000:1000 app.py /app/
```

---

### ADD - Advanced Copy

**Purpose:** Like COPY, but with extra features (URL download, tar extraction).

**Syntax:**
```dockerfile
ADD <src> <dest>
```

**Examples:**
```dockerfile
# Download from URL
ADD https://example.com/file.tar.gz /tmp/

# Auto-extract tar files
ADD archive.tar.gz /app/
# Automatically extracts to /app/

# Regular file copy (same as COPY)
ADD app.py /app/
```

**⚠️ Best Practice: Prefer COPY over ADD**
```dockerfile
# Use COPY for simple file copying
COPY app.py /app/  ✓

# Only use ADD for:
# - Auto-extracting tar files
# - Downloading from URLs (rare, prefer RUN curl)
```

---

### WORKDIR - Set Working Directory

**Purpose:** Set the working directory for subsequent instructions.

**Syntax:**
```dockerfile
WORKDIR /path/to/directory
```

**Examples:**
```dockerfile
# Set working directory
WORKDIR /app

# All subsequent commands run in /app
COPY . .           # Copies to /app
RUN npm install    # Runs in /app

# Can be nested
WORKDIR /app
WORKDIR src        # Now in /app/src
WORKDIR utils      # Now in /app/src/utils

# Create directory if it doesn't exist
WORKDIR /app/data  # Creates /app and /app/data
```

**Best Practice:**
```dockerfile
# Bad: Using cd
RUN cd /app && npm install

# Good: Using WORKDIR
WORKDIR /app
RUN npm install
```

---

### ENV - Environment Variables

**Purpose:** Set environment variables.

**Syntax:**
```dockerfile
ENV <key>=<value>
ENV <key1>=<value1> <key2>=<value2>
```

**Examples:**
```dockerfile
# Single variable
ENV NODE_ENV=production

# Multiple variables
ENV APP_HOME=/app \
    APP_USER=appuser \
    APP_VERSION=1.0.0

# Using in subsequent instructions
ENV DATA_DIR=/data
WORKDIR ${DATA_DIR}  # Uses /data
```

**Real-World Example:**
```dockerfile
# Python application
FROM python:3.9-slim

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

CMD ["python", "app.py"]
```

---

### EXPOSE - Document Ports

**Purpose:** Document which ports the container listens on.

**Syntax:**
```dockerfile
EXPOSE <port>
EXPOSE <port>/<protocol>
```

**Examples:**
```dockerfile
# HTTP
EXPOSE 80

# HTTPS
EXPOSE 443

# Multiple ports
EXPOSE 80 443

# Specific protocol
EXPOSE 53/udp
EXPOSE 80/tcp

# Application ports
EXPOSE 3000  # Node.js
EXPOSE 8080  # Java Spring
EXPOSE 5000  # Flask
```

**⚠️ Important: EXPOSE is documentation only!**
```dockerfile
# EXPOSE doesn't actually publish ports
EXPOSE 80

# Must use -p when running
docker run -p 8080:80 myapp  # Maps host:container
```

---

### VOLUME - Mount Points

**Purpose:** Create mount points for persistent data.

**Syntax:**
```dockerfile
VOLUME /path/to/volume
VOLUME ["/path1", "/path2"]
```

**Examples:**
```dockerfile
# Database data directory
VOLUME /var/lib/mysql

# Multiple volumes
VOLUME ["/var/log", "/var/data"]

# Application data
VOLUME /app/uploads
```

**How It Works:**
```dockerfile
FROM nginx
VOLUME /usr/share/nginx/html

# When container runs:
# Docker creates anonymous volume
# Data persists even if container removed
```

---

### USER - Set User Context

**Purpose:** Set user/group for subsequent instructions and container runtime.

**Syntax:**
```dockerfile
USER <user>
USER <user>:<group>
USER <UID>:<GID>
```

**Examples:**
```dockerfile
# Switch to non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

# With group
USER appuser:appgroup

# Using UID/GID
USER 1000:1000

# Root for installation, non-root for runtime
RUN apt-get update && apt-get install -y python3
USER nobody
CMD ["python3", "app.py"]
```

**Security Best Practice:**
```dockerfile
FROM python:3.9-slim

# Install as root
RUN pip install flask

# Create non-root user
RUN adduser --disabled-password --gecos '' appuser

# Switch to non-root
USER appuser

WORKDIR /app
COPY --chown=appuser:appuser . .

# Runs as non-root
CMD ["python", "app.py"]
```

---

### ARG - Build Arguments

**Purpose:** Define build-time variables.

**Syntax:**
```dockerfile
ARG <name>
ARG <name>=<default value>
```

**Examples:**
```dockerfile
# With default value
ARG PYTHON_VERSION=3.9
FROM python:${PYTHON_VERSION}

ARG APP_VERSION=1.0.0
LABEL version=${APP_VERSION}

# Build with custom value
# docker build --build-arg PYTHON_VERSION=3.10 .
```

**ARG vs ENV:**
```dockerfile
# ARG: Build-time only
ARG BUILD_DATE
RUN echo "Built on ${BUILD_DATE}"  # Works
# Not available at runtime

# ENV: Build-time and runtime
ENV APP_ENV=production
RUN echo "${APP_ENV}"  # Works
# Also available in running container
```

**Real-World Example:**
```dockerfile
# Flexible base image
ARG NODE_VERSION=18
ARG ALPINE_VERSION=3.19

FROM node:${NODE_VERSION}-alpine${ALPINE_VERSION}

ARG BUILD_DATE
ARG GIT_COMMIT

LABEL build-date=${BUILD_DATE} \
      git-commit=${GIT_COMMIT}

# Build
# docker build \
#   --build-arg NODE_VERSION=20 \
#   --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
#   --build-arg GIT_COMMIT=$(git rev-parse HEAD) \
#   -t myapp:latest .
```

---

### LABEL - Metadata

**Purpose:** Add metadata to image.

**Syntax:**
```dockerfile
LABEL <key>=<value>
LABEL <key1>=<value1> <key2>=<value2>
```

**Examples:**
```dockerfile
# Single label
LABEL version="1.0"

# Multiple labels
LABEL maintainer="dev@company.com" \
      version="1.0.0" \
      description="My application"

# Standard labels
LABEL org.opencontainers.image.created="2024-01-17" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.source="https://github.com/company/app"
```

**View Labels:**
```bash
docker inspect myapp:1.0 | grep Labels -A 10
```

---

### HEALTHCHECK - Container Health

**Purpose:** Define how to check if container is healthy.

**Syntax:**
```dockerfile
HEALTHCHECK [OPTIONS] CMD command
HEALTHCHECK NONE  # Disable healthcheck
```

**Options:**
```dockerfile
--interval=DURATION    # Check frequency (default: 30s)
--timeout=DURATION     # Timeout per check (default: 30s)
--start-period=DURATION # Grace period (default: 0s)
--retries=N           # Failures needed to be unhealthy (default: 3)
```

**Examples:**
```dockerfile
# Web server health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

# Database health check
HEALTHCHECK --interval=10s --timeout=5s \
  CMD pg_isready -U postgres || exit 1

# Custom health endpoint
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s \
  CMD curl -f http://localhost:8080/health || exit 1

# Disable inherited health check
HEALTHCHECK NONE
```

**Real-World Example:**
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

EXPOSE 3000

# Health check for Express app
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD node healthcheck.js || exit 1

CMD ["node", "server.js"]
```

---

## Complete Real-World Examples

### Example 1: Python Flask API

```dockerfile
# Multi-stage build for Python Flask API

# Stage 1: Build
FROM python:3.9-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Runtime
FROM python:3.9-slim

# Create non-root user
RUN useradd -m -u 1000 appuser

WORKDIR /app

# Copy dependencies from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application
COPY --chown=appuser:appuser . .

# Set environment
ENV PATH=/home/appuser/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    FLASK_APP=app.py

# Switch to non-root user
USER appuser

EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:5000/health')" || exit 1

CMD ["flask", "run", "--host=0.0.0.0"]
```

### Example 2: Node.js Application

```dockerfile
# Node.js production-ready Dockerfile

FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source
COPY . .

# Build application
RUN npm run build

# Production image
FROM node:18-alpine

# Install dumb-init (proper signal handling)
RUN apk add --no-cache dumb-init

# Create app user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy built application
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./

# Set environment
ENV NODE_ENV=production

USER nodejs

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s \
  CMD node healthcheck.js || exit 1

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

### Example 3: Go Application

```dockerfile
# Go multi-stage build

# Builder stage
FROM golang:1.21-alpine AS builder

RUN apk add --no-cache git ca-certificates

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o server .

# Final stage - minimal image
FROM alpine:latest

RUN apk --no-cache add ca-certificates

# Create non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -D -u 1001 -G appgroup appuser

WORKDIR /app

# Copy binary from builder
COPY --from=builder /app/server .
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

USER appuser

EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

CMD ["./server"]
```

### Example 4: Database (PostgreSQL)

```dockerfile
FROM postgres:15-alpine

# Set environment variables
ENV POSTGRES_DB=myapp \
    POSTGRES_USER=myapp \
    POSTGRES_PASSWORD_FILE=/run/secrets/db_password

# Copy initialization scripts
COPY ./init-scripts/ /docker-entrypoint-initdb.d/

# Copy custom configuration
COPY ./postgresql.conf /etc/postgresql/postgresql.conf

# Create data directory with correct permissions
RUN mkdir -p /var/lib/postgresql/data && \
    chown -R postgres:postgres /var/lib/postgresql/data

VOLUME /var/lib/postgresql/data

EXPOSE 5432

# Health check
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD pg_isready -U myapp || exit 1

# Use custom config
CMD ["postgres", "-c", "config_file=/etc/postgresql/postgresql.conf"]
```

---

## Dockerfile Best Practices

### 1. Use Specific Tags

```dockerfile
# Bad: Tag changes over time
FROM node:latest

# Good: Reproducible
FROM node:18.19.0-alpine3.19
```

### 2. Order Instructions by Change Frequency

```dockerfile
# Bad: Frequently changing first
FROM node:18
COPY . .              # Changes often
COPY package*.json ./ # Changes rarely
RUN npm install

# Good: Rarely changing first
FROM node:18
COPY package*.json ./ # Changes rarely
RUN npm install       # Cached if package.json unchanged
COPY . .              # Changes often
```

### 3. Combine RUN Commands

```dockerfile
# Bad: Multiple layers
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Good: One layer
RUN apt-get update && \
    apt-get install -y python3 curl && \
    rm -rf /var/lib/apt/lists/*
```

### 4. Use Multi-stage Builds

```dockerfile
# Separate build and runtime stages
# Smaller final images
# More secure (no build tools in production)
```

### 5. Don't Run as Root

```dockerfile
# Create and use non-root user
RUN useradd -m appuser
USER appuser
```

### 6. Use .dockerignore

```
.git
.github
node_modules
npm-debug.log
Dockerfile
docker-compose.yml
README.md
.env
*.md
```

### 7. Minimize Layers

```dockerfile
# Each RUN, COPY, ADD creates a layer
# Combine when possible
# But balance with caching
```

### 8. Use HEALTHCHECK

```dockerfile
# Always include health checks
# Enables automatic recovery
# Better monitoring
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The __________ instruction specifies the base image for the build.
2. __________ is preferred over ADD for simple file copying.
3. The __________ instruction sets the working directory for subsequent commands.
4. Environment variables are set using the __________ instruction.
5. The __________ instruction is documentation-only and doesn't actually publish ports.
6. __________ are available only during build time, while __________ are available at runtime too.

### True/False

1. ⬜ Each RUN instruction creates a new layer in the image
2. ⬜ EXPOSE actually publishes the port to the host
3. ⬜ WORKDIR creates the directory if it doesn't exist
4. ⬜ ADD and COPY are completely interchangeable
5. ⬜ ARG variables are available in the running container
6. ⬜ Multi-stage builds help reduce final image size
7. ⬜ HEALTHCHECK is required in every Dockerfile

### Multiple Choice

1. Which instruction should come first in a Dockerfile?
   - A) RUN
   - B) FROM
   - C) COPY
   - D) ENV

2. What's the difference between COPY and ADD?
   - A) They're identical
   - B) ADD can extract tar files and download URLs
   - C) COPY is faster
   - D) ADD is deprecated

3. Why combine RUN commands?
   - A) Easier to read
   - B) Reduces number of layers
   - C) Faster execution
   - D) Required by Docker

4. What does `USER appuser` do?
   - A) Creates a new user
   - B) Switches to existing user for subsequent commands
   - C) Deletes a user
   - D) Lists users

5. When are ARG values available?
   - A) Build time only
   - B) Runtime only
   - C) Both build and runtime
   - D) Never

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. FROM
2. COPY
3. WORKDIR
4. ENV
5. EXPOSE
6. ARG, ENV (or ARG values, ENV variables)

**True/False:**
1. ✅ True - Each RUN creates a layer (also COPY, ADD)
2. ❌ False - EXPOSE is documentation, use -p flag to publish
3. ✅ True - WORKDIR creates directory if missing
4. ❌ False - ADD has extra features (tar extraction, URL download)
5. ❌ False - ARG only available at build time
6. ✅ True - Multi-stage excludes build tools from final image
7. ❌ False - HEALTHCHECK is recommended but not required

**Multiple Choice:**
1. **B** - FROM (must be first, except ARG before FROM)
2. **B** - ADD can extract tar files and download URLs
3. **B** - Reduces number of layers
4. **B** - Switches to existing user for subsequent commands
5. **A** - Build time only

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the difference between RUN, CMD, and ENTRYPOINT

<details>
<summary><strong>View Answer</strong></summary>

**Quick Summary:**

- **RUN**: Executes during **build**, creates layer
- **CMD**: Provides **default** command at **runtime**, can be overridden
- **ENTRYPOINT**: Defines **main** executable at **runtime**, harder to override

**Detailed Explanation:**

**RUN - Build Time Execution**
```dockerfile
FROM ubuntu:20.04

# RUN executes DURING BUILD
RUN apt-get update
RUN apt-get install -y python3
RUN pip3 install flask

# Creates layers in the image
# Executed once when building
# Not executed when container runs
```

**Timing:**
```bash
docker build -t myapp .
  Step 1: FROM ubuntu:20.04
  Step 2: RUN apt-get update     ← Executes now
  Step 3: RUN apt-get install    ← Executes now
  Image built ✓

docker run myapp
  RUN commands don't execute again
```

---

**CMD - Default Runtime Command**
```dockerfile
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y python3

# CMD provides default command when container starts
CMD ["python3", "-m", "http.server", "8000"]
```

**Behavior:**
```bash
# Use default CMD
docker run myapp
# Runs: python3 -m http.server 8000

# Override CMD
docker run myapp echo "Hello"
# Runs: echo "Hello"
# CMD completely replaced
```

---

**ENTRYPOINT - Main Executable**
```dockerfile
FROM ubuntu:20.04
RUN apt-get update && apt-get install -y curl

# ENTRYPOINT defines the main executable
ENTRYPOINT ["curl"]

# CMD provides default arguments
CMD ["--help"]
```

**Behavior:**
```bash
# Use ENTRYPOINT + default CMD
docker run myapp
# Runs: curl --help

# Override CMD (provide different arguments)
docker run myapp http://google.com
# Runs: curl http://google.com

# To override ENTRYPOINT (harder, need flag)
docker run --entrypoint bash myapp
# Runs: bash
```

---

**Comparison Table:**

| Feature | RUN | CMD | ENTRYPOINT |
|---------|-----|-----|------------|
| **When** | Build time | Runtime | Runtime |
| **Purpose** | Install/setup | Default command | Main executable |
| **Override** | Can't (rebuild needed) | Easy (just add command) | Harder (need --entrypoint) |
| **Multiple** | Yes (each creates layer) | Only last one used | Only last one used |
| **Required** | No | No | No |

---

**Real-World Examples:**

**Example 1: Web Server**
```dockerfile
FROM node:18

WORKDIR /app

# RUN: Setup during build
RUN npm install express

# COPY: Add code
COPY server.js .

# CMD: Default runtime command
CMD ["node", "server.js"]

# Build
docker build -t webapp .
  RUN npm install executes ← Build time

# Run with default
docker run webapp
  Executes: node server.js ← Runtime

# Run with override
docker run webapp node debug.js
  Executes: node debug.js ← CMD overridden
```

**Example 2: CLI Tool**
```dockerfile
FROM alpine:latest

RUN apk add --no-cache curl

# ENTRYPOINT: The tool itself
ENTRYPOINT ["curl"]

# CMD: Default flags
CMD ["--help"]

# Usage:
docker run curltool
  → curl --help

docker run curltool -I http://google.com
  → curl -I http://google.com

docker run curltool -v http://api.example.com
  → curl -v http://api.example.com
```

**Example 3: Database**
```dockerfile
FROM postgres:15

# RUN: Setup during build
RUN apt-get update && apt-get install -y postgresql-contrib

# ENTRYPOINT: Database executable
ENTRYPOINT ["docker-entrypoint.sh"]

# CMD: Default command
CMD ["postgres"]

# Run:
docker run postgres-image
  → docker-entrypoint.sh postgres

# With custom args:
docker run postgres-image postgres --config-file=/custom.conf
  → docker-entrypoint.sh postgres --config-file=/custom.conf
```

---

**Common Patterns:**

**Pattern 1: ENTRYPOINT + CMD for Flexible Tool**
```dockerfile
# Python script runner
FROM python:3.9

COPY script.py /
ENTRYPOINT ["python", "/script.py"]
CMD ["--help"]

# Usage:
docker run pyscript                    → python /script.py --help
docker run pyscript --input data.csv   → python /script.py --input data.csv
```

**Pattern 2: CMD Only for Simple Apps**
```dockerfile
# Simple web server
FROM nginx:alpine

COPY html/ /usr/share/nginx/html/
CMD ["nginx", "-g", "daemon off;"]

# Usage:
docker run webapp  → nginx -g daemon off;
```

**Pattern 3: ENTRYPOINT for Wrapper Script**
```dockerfile
# Entrypoint script handles initialization
FROM postgres:15

COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["postgres"]

# entrypoint.sh can:
# - Set permissions
# - Initialize database
# - Run migrations
# - Then exec "$@" (runs CMD)
```

</details>

### Question 2: How would you optimize this Dockerfile?

```dockerfile
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y python3-pip
RUN apt-get install -y curl
RUN apt-get install -y git
COPY . /app
WORKDIR /app
RUN pip3 install -r requirements.txt
EXPOSE 8000
CMD python3 app.py
```

<details>
<summary><strong>View Answer</strong></summary>

**Problems with Original:**

1. ❌ Uses `latest` tag (not reproducible)
2. ❌ Heavy base image (ubuntu)
3. ❌ Multiple RUN commands (many layers)
4. ❌ Doesn't clean package cache
5. ❌ Installs unnecessary tools (git, curl)
6. ❌ Copies everything first (poor caching)
7. ❌ No multi-stage build
8. ❌ Runs as root
9. ❌ No .dockerignore
10. ❌ Shell form CMD (no proper signal handling)

---

**Optimized Version:**

```dockerfile
# Use specific Python base (smaller, includes Python)
FROM python:3.9-slim AS builder

# Install only necessary build dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy only requirements first (better caching)
COPY requirements.txt .

# Install dependencies in user location
RUN pip install --user --no-cache-dir -r requirements.txt

# Production stage
FROM python:3.9-slim

# Create non-root user
RUN useradd -m -u 1000 appuser

WORKDIR /app

# Copy installed dependencies from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Set PATH for user-installed packages
ENV PATH=/home/appuser/.local/bin:$PATH

# Switch to non-root user
USER appuser

EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

# Use exec form
CMD ["python", "app.py"]
```

**With .dockerignore:**
```
.git
.gitignore
.github
__pycache__
*.pyc
*.pyo
*.pyd
.pytest_cache
.coverage
htmlcov
dist
build
*.egg-info
.env
.venv
venv/
.DS_Store
Dockerfile
docker-compose.yml
README.md
*.md
tests/
```

---

**Improvements Explained:**

**1. Specific Base Image**
```dockerfile
# Before
FROM ubuntu:latest  # 72 MB, changes over time

# After
FROM python:3.9-slim  # 122 MB, but includes Python
# Specific version, reproducible
```

**2. Multi-stage Build**
```dockerfile
# Stage 1: Build dependencies
FROM python:3.9-slim AS builder
# Install gcc for compiling Python packages
# Install dependencies

# Stage 2: Runtime
FROM python:3.9-slim
# Copy only installed packages
# No build tools in final image

Result:
- Build stage: 200 MB
- Final image: 150 MB
- Saved: 50 MB + build tools
```

**3. Combined RUN Commands**
```dockerfile
# Before: 5 layers, cache not cleaned
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y python3-pip
RUN apt-get install -y curl
RUN apt-get install -y git

# After: 1 layer, cache cleaned
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

Saved: 4 layers, ~50 MB (package cache)
```

**4. Better Layer Ordering**
```dockerfile
# Before: Poor caching
COPY . /app              # Changes often
RUN pip3 install -r requirements.txt  # Runs every time

# After: Good caching
COPY requirements.txt .  # Changes rarely
RUN pip install -r requirements.txt  # Cached
COPY . .                 # Changes often

Result: Pip install cached when only code changes
```

**5. Security - Non-root User**
```dockerfile
# Before: Runs as root
# Vulnerable if container compromised

# After: Runs as appuser
RUN useradd -m -u 1000 appuser
USER appuser

Result: Limited permissions, better security
```

**6. Removed Unnecessary Tools**
```dockerfile
# Before:
# git, curl installed but never used

# After:
# Only gcc for building
# Removed in final stage

Result: Smaller image, smaller attack surface
```

---

**Size Comparison:**

```
Original:
├── ubuntu:latest base: 72 MB
├── python3 + pip: 100 MB
├── curl + git: 50 MB
├── dependencies: 80 MB
├── app code: 10 MB
├── package cache: 50 MB
└── Total: 362 MB

Optimized:
├── python:3.9-slim: 122 MB
├── dependencies: 40 MB
├── app code: 10 MB
└── Total: 172 MB

Reduction: 52% (362 MB → 172 MB)
```

---

**Build Time Comparison:**

```
Original (no cache):
Step 1: FROM ubuntu           (30s)
Step 2: RUN apt-get update    (20s)
Step 3-6: RUN apt-get install (40s)
Step 7: COPY .                (5s)
Step 8: RUN pip install       (60s)
Total: 155s

Optimized (no cache):
Step 1: FROM python:3.9-slim  (20s)
Step 2: RUN apt-get install   (15s)
Step 3: COPY requirements     (1s)
Step 4: RUN pip install       (60s)
Step 5: FROM python:3.9-slim  (cached)
Step 6: COPY from builder     (5s)
Step 7: COPY .                (3s)
Total: 104s

Optimized (code change, cache hit):
Steps 1-4: Cached
Step 5-7: Execute
Total: 8s

Improvement: 95% faster on rebuilds!
```

---

**Additional Optimizations for Production:**

```dockerfile
# Even more optimized for Go/Rust (static binaries)
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server

# Minimal runtime
FROM scratch
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8000
CMD ["/server"]

Final size: 15 MB (just the binary!)
```

</details>

### Question 3: When would you use ARG vs ENV?

<details>
<summary><strong>View Answer</strong></summary>

**Key Difference:**

- **ARG**: Build-time only, not available in running container
- **ENV**: Build-time AND runtime, available in running container

---

**ARG - Build Arguments**

**When to Use:**
```
✓ Parameterize base image version
✓ Build-time configuration
✓ CI/CD variables (git commit, build date)
✓ Values that shouldn't be in final image
✓ Toggle features during build
```

**Examples:**

**1. Flexible Base Image**
```dockerfile
ARG PYTHON_VERSION=3.9
FROM python:${PYTHON_VERSION}-slim

ARG NODE_VERSION=18
FROM node:${NODE_VERSION}-alpine

# Build with different versions:
docker build --build-arg PYTHON_VERSION=3.11 -t app:py311 .
docker build --build-arg PYTHON_VERSION=3.10 -t app:py310 .
```

**2. Build Metadata**
```dockerfile
ARG BUILD_DATE
ARG GIT_COMMIT
ARG VERSION

FROM node:18-alpine

LABEL build-date=${BUILD_DATE} \
      git-commit=${GIT_COMMIT} \
      version=${VERSION}

# Build:
docker build \
  --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --build-arg GIT_COMMIT=$(git rev-parse HEAD) \
  --build-arg VERSION=1.2.3 \
  -t app:1.2.3 .

# These values NOT available at runtime
# Only in image metadata
```

**3. Conditional Build Steps**
```dockerfile
ARG INCLUDE_DEV_TOOLS=false

FROM node:18-alpine

RUN if [ "$INCLUDE_DEV_TOOLS" = "true" ]; then \
        apk add --no-cache vim curl; \
    fi

# Development build:
docker build --build-arg INCLUDE_DEV_TOOLS=true -t app:dev .

# Production build:
docker build --build-arg INCLUDE_DEV_TOOLS=false -t app:prod .
```

**4. Private Registry Authentication**
```dockerfile
ARG NPM_TOKEN

FROM node:18

# Use token during build, don't persist it
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc && \
    npm install && \
    rm .npmrc

# Token not in final image
```

---

**ENV - Environment Variables**

**When to Use:**
```
✓ Application configuration
✓ Runtime behavior
✓ Paths and locations
✓ Service URLs
✓ Feature flags
✓ Any value needed by running application
```

**Examples:**

**1. Application Configuration**
```dockerfile
FROM python:3.9-slim

ENV FLASK_APP=app.py \
    FLASK_ENV=production \
    PYTHONUNBUFFERED=1

# Available at runtime:
docker run myapp
# Flask uses FLASK_APP and FLASK_ENV automatically
```

**2. Service URLs**
```dockerfile
FROM node:18-alpine

ENV DATABASE_URL=postgres://localhost:5432/myapp \
    REDIS_URL=redis://localhost:6379 \
    API_URL=http://api.example.com

# Can override at runtime:
docker run -e DATABASE_URL=postgres://prod-db:5432/myapp myapp
```

**3. Path Configuration**
```dockerfile
FROM openjdk:11

ENV JAVA_HOME=/usr/lib/jvm/java-11-openjdk \
    APP_HOME=/app \
    LOG_DIR=/var/log/myapp

WORKDIR ${APP_HOME}
```

**4. Feature Toggles**
```dockerfile
FROM node:18-alpine

ENV ENABLE_METRICS=true \
    ENABLE_DEBUG_LOGS=false \
    MAX_CONNECTIONS=100

# Runtime override:
docker run -e ENABLE_DEBUG_LOGS=true myapp
```

---

**ARG + ENV Pattern**

**Use Case: Build-time value that's also needed at runtime**

```dockerfile
# Pass ARG that becomes ENV

ARG APP_VERSION=1.0.0
ENV APP_VERSION=${APP_VERSION}

FROM node:18-alpine

# Now available at both build and runtime
LABEL version=${APP_VERSION}

# Build:
docker build --build-arg APP_VERSION=2.0.0 -t app:2.0.0 .

# Runtime:
docker run app:2.0.0 env | grep APP_VERSION
# APP_VERSION=2.0.0
```

**More Complex Example:**
```dockerfile
# Defaults for different environments

ARG ENVIRONMENT=production
ARG APP_VERSION=1.0.0

FROM node:18-alpine

# Convert ARG to ENV
ENV NODE_ENV=${ENVIRONMENT} \
    APP_VERSION=${APP_VERSION}

# Use at build time
RUN if [ "$NODE_ENV" = "development" ]; then \
        npm install; \
    else \
        npm ci --only=production; \
    fi

# Also available at runtime
CMD ["node", "server.js"]

# Build for different environments:
docker build --build-arg ENVIRONMENT=development -t app:dev .
docker build --build-arg ENVIRONMENT=production -t app:prod .
```

---

**Comparison Table:**

| Feature | ARG | ENV |
|---------|-----|-----|
| **Available when** | Build time | Build time + Runtime |
| **Set with** | --build-arg | --env / -e |
| **In Dockerfile** | ARG NAME=value | ENV NAME=value |
| **Visibility** | Build only | Image + Container |
| **Override** | During build | During run |
| **Use for** | Build config | App config |
| **Security** | Safer (not persisted) | Careful (persists) |

---

**Security Considerations:**

**DON'T Store Secrets in ENV:**
```dockerfile
# BAD: Secret persists in image
ENV DATABASE_PASSWORD=secret123

# Anyone can see it:
docker inspect myapp | grep DATABASE_PASSWORD

# GOOD: Use secrets management
docker run --env-file secrets.env myapp
# or
docker run -e DATABASE_PASSWORD=$(cat secret.txt) myapp
# or use Docker secrets / Kubernetes secrets
```

**ARG for Secrets During Build:**
```dockerfile
# Careful: ARG still appears in image history
ARG NPM_TOKEN

# Better: Use buildkit secrets
# docker build --secret id=npm_token,src=token.txt .

RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) \
    npm install && \
    rm .npmrc
```

---

**Real-World Decision Tree:**

```
Need value during build?
├─ Yes
│  ├─ Also need at runtime?
│  │  ├─ Yes → ARG + ENV pattern
│  │  └─ No → ARG only
│  └─ Different per environment?
│     └─ Yes → ARG
│
└─ No (runtime only)
   └─ ENV

Examples:
- Base image version → ARG
- Build date → ARG
- Git commit → ARG
- Database URL → ENV
- API key → Runtime flag (not ENV!)
- Feature flag → ENV
- Build environment → ARG + ENV
```

</details>

</details>

---

[← Previous: 3.1 Working with Images](01-image-basics.md) | [Next: 3.3 CMD vs ENTRYPOINT →](03-cmd-vs-entrypoint.md)