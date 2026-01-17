# 10.1 Dockerfile Best Practices

Production-ready Dockerfile practices for security, performance, and maintainability.

---

## Use Official Base Images

**Bad:**
```dockerfile
FROM ubuntu
RUN apt-get update && apt-get install -y python3
```

**Good:**
```dockerfile
FROM python:3.11-slim
```

**Why:**
```
✓ Pre-configured and optimized
✓ Security patches
✓ Well-maintained
✓ Smaller size
✓ Faster builds
```

---

## Minimize Image Layers

**Bad:**
```dockerfile
FROM node:18
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean
```

**Good:**
```dockerfile
FROM node:18
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**Why:**
```
✓ Fewer layers = smaller image
✓ Better caching
✓ Faster builds
✓ Less complexity
```

---

## Order Instructions by Change Frequency

**Bad:**
```dockerfile
FROM node:18
COPY . /app
RUN npm install
```

**Good:**
```dockerfile
FROM node:18
WORKDIR /app

# Dependencies change less frequently
COPY package*.json ./
RUN npm ci --production

# Code changes frequently
COPY . .
```

**Why:**
```
First build: All layers rebuilt
Code change: Only COPY . . layer rebuilt
Dependencies cached ✓

Time saved: Minutes per build
```

---

## Use Multi-Stage Builds

**Bad (Single Stage):**
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
CMD ["node", "dist/server.js"]

# Final image: ~1GB
# Contains: npm, build tools, source code, node_modules
```

**Good (Multi-Stage):**
```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:18-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY --from=builder /app/dist ./dist
USER node
CMD ["node", "dist/server.js"]

# Final image: ~200MB
# Contains: Only runtime dependencies and built code
```

**Comparison:**
```
Single-stage: 1GB
Multi-stage:  200MB
Reduction:    80%
```

---

## Real-World Multi-Stage Examples

### Go Application

```dockerfile
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

# Production stage
FROM alpine:3.19
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /app/server .
USER nobody
EXPOSE 8080
CMD ["./server"]

# Size: ~15MB (vs ~1GB with golang base)
```

---

### Python Application

```dockerfile
# Build stage
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Production stage
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
ENV PATH=/root/.local/bin:$PATH
USER nobody
CMD ["python", "app.py"]

# Size: ~150MB (vs ~900MB with full python)
```

---

### React Application

```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

# Size: ~25MB (vs ~1GB with node)
```

---

## Use .dockerignore

**.dockerignore:**
```
# Version control
.git
.gitignore

# Dependencies
node_modules
__pycache__
*.pyc

# IDE
.vscode
.idea
*.swp

# OS
.DS_Store
Thumbs.db

# Build artifacts
dist
build
*.log

# Tests
tests
*.test.js
coverage

# Documentation
README.md
docs

# CI/CD
.github
.gitlab-ci.yml
Jenkinsfile
```

**Impact:**
```
Without .dockerignore:
COPY . .  → 500MB (includes node_modules, .git, etc.)

With .dockerignore:
COPY . .  → 5MB (only source code)

Build time: 2 min → 10 sec
```

---

## Don't Run as Root

**Bad:**
```dockerfile
FROM node:18
WORKDIR /app
COPY . .
CMD ["node", "server.js"]

# Runs as root (UID 0)
```

**Good:**
```dockerfile
FROM node:18
WORKDIR /app
COPY --chown=node:node . .
USER node
CMD ["node", "server.js"]

# Runs as node user (UID 1000)
```

**Custom User:**
```dockerfile
FROM alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["./app"]
```

**Why:**
```
Security: Container escape = limited privileges
Best practice: Principle of least privilege
```

---

## Use Specific Tags

**Bad:**
```dockerfile
FROM node
FROM node:latest
FROM python
```

**Good:**
```dockerfile
FROM node:18.19.0-alpine3.19
FROM python:3.11.7-slim
FROM nginx:1.25.3-alpine
```

**Why:**
```
node:latest → Changes unexpectedly
node:18 → Gets updates (breaking changes possible)
node:18.19.0-alpine3.19 → Predictable, reproducible

Production: Use exact versions
Development: Can use major versions
```

---

## Minimize Installed Packages

**Bad:**
```dockerfile
FROM ubuntu
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    vim \
    nano \
    git \
    build-essential
```

**Good:**
```dockerfile
FROM alpine:3.19
RUN apk add --no-cache curl

# Or for specific tools
FROM node:18-alpine
RUN apk add --no-cache git
```

**Only Install What's Needed:**
```dockerfile
FROM python:3.11-slim

# Install only runtime dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    libpq5 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

---

## Use Alpine Images

**Comparison:**

```dockerfile
# Ubuntu-based
FROM ubuntu:22.04
# Size: ~77MB base

FROM python:3.11
# Size: ~1GB

# Alpine-based
FROM alpine:3.19
# Size: ~7MB base

FROM python:3.11-alpine
# Size: ~50MB
```

**Example:**
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

USER node
CMD ["node", "server.js"]

# Size: ~180MB (vs ~1GB with node:18)
```

**Caveat:**
```
Alpine uses musl libc (not glibc)
Some packages may have compatibility issues
Test thoroughly
```

---

## Leverage Build Cache

**Cache Strategy:**

```dockerfile
FROM node:18-alpine

WORKDIR /app

# 1. Copy only dependency files (changes rarely)
COPY package.json package-lock.json ./

# 2. Install dependencies (cached unless package.json changes)
RUN npm ci --production

# 3. Copy source code (changes frequently)
COPY . .

# 4. Build (only runs if source changes)
RUN npm run build

CMD ["node", "dist/server.js"]
```

**Build Flow:**
```
First build:
Step 1: package.json → Cache miss
Step 2: npm ci → Cache miss (5 min)
Step 3: COPY . → Cache miss
Step 4: npm run build → Cache miss (2 min)
Total: 7 min

Code change:
Step 1: package.json → Cache hit ✓
Step 2: npm ci → Cache hit ✓
Step 3: COPY . → Cache miss
Step 4: npm run build → Cache miss (2 min)
Total: 2 min

Dependency change:
Step 1: package.json → Cache miss
Step 2: npm ci → Cache miss (5 min)
Step 3: COPY . → Cache miss
Step 4: npm run build → Cache miss (2 min)
Total: 7 min
```

---

## Use COPY Instead of ADD

**Bad:**
```dockerfile
ADD . /app
ADD https://example.com/file.tar.gz /tmp/
```

**Good:**
```dockerfile
COPY . /app

# For URLs, use curl/wget
RUN curl -o /tmp/file.tar.gz https://example.com/file.tar.gz
```

**Why:**
```
ADD has implicit behavior:
- Auto-extracts tar files
- Can fetch URLs
- Less predictable

COPY is explicit:
- Only copies files
- Clear intent
- Preferred unless you need ADD features
```

---

## Set Proper Metadata

```dockerfile
FROM node:18-alpine

LABEL maintainer="dev@example.com" \
      version="1.0" \
      description="Production API server" \
      org.opencontainers.image.source="https://github.com/myorg/myapp" \
      org.opencontainers.image.licenses="MIT"

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js || exit 1

USER node
CMD ["node", "server.js"]
```

---

## Use HEALTHCHECK

```dockerfile
FROM nginx:alpine

COPY nginx.conf /etc/nginx/nginx.conf
COPY html /usr/share/nginx/html

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost/health || exit 1

EXPOSE 80
```

**Custom Health Check:**
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

HEALTHCHECK --interval=30s --timeout=3s \
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1); });"

CMD ["node", "server.js"]
```

---

## Combine RUN Commands

**Bad:**
```dockerfile
FROM ubuntu
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN curl -o file.tar.gz https://example.com/file.tar.gz
RUN tar -xzf file.tar.gz
RUN rm file.tar.gz

# Creates 6 layers
```

**Good:**
```dockerfile
FROM ubuntu
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git && \
    curl -o file.tar.gz https://example.com/file.tar.gz && \
    tar -xzf file.tar.gz && \
    rm file.tar.gz && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Creates 1 layer
```

---

## Clean Up in Same Layer

**Bad:**
```dockerfile
FROM node:18
WORKDIR /app
RUN npm install -g typescript
COPY . .
RUN npm install
RUN npm run build
RUN npm cache clean --force

# npm cache still in earlier layer
```

**Good:**
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci && npm cache clean --force
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production && npm cache clean --force
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/server.js"]

# Cache cleaned in same layer
```

---

## Use ARG for Build-Time Variables

```dockerfile
FROM node:18-alpine

ARG NODE_ENV=production
ARG APP_VERSION=1.0.0

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

ENV NODE_ENV=${NODE_ENV} \
    APP_VERSION=${APP_VERSION}

CMD ["node", "server.js"]
```

**Build:**
```bash
docker build \
  --build-arg NODE_ENV=production \
  --build-arg APP_VERSION=2.0.0 \
  -t myapp:2.0.0 .
```

---

## Use ENV for Runtime Variables

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Build-time
ARG INSTALL_DEV=false

# Runtime
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PORT=8000

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE ${PORT}
CMD ["python", "app.py"]
```

---

## Complete Production Example

```dockerfile
# syntax=docker/dockerfile:1

# Build stage
FROM node:18-alpine AS builder

# Build arguments
ARG NODE_ENV=production
ARG APP_VERSION=1.0.0

# Install build dependencies
RUN apk add --no-cache python3 make g++

WORKDIR /app

# Copy dependency files
COPY package.json package-lock.json ./

# Install all dependencies (including dev)
RUN npm ci && npm cache clean --force

# Copy source code
COPY . .

# Build application
RUN npm run build && \
    npm prune --production

# Production stage
FROM node:18-alpine

# Metadata
LABEL maintainer="dev@example.com" \
      version="${APP_VERSION}" \
      description="Production API server"

# Runtime environment variables
ENV NODE_ENV=production \
    PORT=3000 \
    LOG_LEVEL=info

# Install runtime dependencies only
RUN apk add --no-cache \
    tini \
    dumb-init

WORKDIR /app

# Create non-root user
RUN addgroup -g 1000 appgroup && \
    adduser -D -u 1000 -G appgroup appuser && \
    chown -R appuser:appgroup /app

# Copy from builder
COPY --from=builder --chown=appuser:appgroup /app/package*.json ./
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD node healthcheck.js || exit 1

# Expose port
EXPOSE ${PORT}

# Use tini for proper signal handling
ENTRYPOINT ["/sbin/tini", "--"]

# Start application
CMD ["node", "dist/server.js"]
```

**Build:**
```bash
docker build \
  --build-arg APP_VERSION=2.1.0 \
  -t myapp:2.1.0 \
  -t myapp:latest \
  .
```

**Size Comparison:**
```
Without optimization: ~1.2GB
With all optimizations: ~180MB
Reduction: 85%
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Use __________ instead of ADD for copying files.
2. Multi-stage builds use __________ to copy files between stages.
3. The __________ file excludes files from the build context.
4. Always run containers as __________ user, not root.
5. __________ images are smaller but use musl libc.
6. Use __________ for build-time variables and ENV for runtime variables.

### True/False

1. ⬜ Latest tags provide reproducible builds
2. ⬜ Multi-stage builds reduce final image size
3. ⬜ Each RUN command creates a new layer
4. ⬜ COPY and ADD are identical
5. ⬜ Alpine images are always the best choice
6. ⬜ Cleaning up in the same RUN command reduces layer size
7. ⬜ Root user is required for most applications

### Multiple Choice

1. Which is better for base images?
   - A) FROM ubuntu
   - B) FROM node:latest
   - C) FROM python
   - D) FROM node:18-alpine

2. What does COPY --from=builder do?
   - A) Copies from build context
   - B) Copies from another stage
   - C) Copies from host
   - D) Invalid syntax

3. Best practice for package installation?
   - A) Multiple RUN commands
   - B) Combined RUN command with cleanup
   - C) Install everything
   - D) Use ADD

4. Why use .dockerignore?
   - A) Faster builds
   - B) Smaller context
   - C) Both A and B
   - D) Neither

5. HEALTHCHECK purpose?
   - A) Build validation
   - B) Container health monitoring
   - C) Security scanning
   - D) Performance testing

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. COPY
2. COPY --from
3. .dockerignore
4. non-root
5. Alpine
6. ARG

**True/False:**
1. ❌ False - Latest tags change, not reproducible
2. ✅ True - Multi-stage removes build dependencies
3. ✅ True - Each RUN creates a layer
4. ❌ False - ADD has auto-extract and URL features
5. ❌ False - Not always compatible (musl vs glibc)
6. ✅ True - Cleanup in same layer reduces size
7. ❌ False - Most apps don't need root

**Multiple Choice:**
1. **D** - FROM node:18-alpine (specific + small)
2. **B** - Copies from another stage
3. **B** - Combined RUN with cleanup
4. **C** - Both faster builds and smaller context
5. **B** - Container health monitoring

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: How do multi-stage builds work and why use them?

<details>
<summary><strong>View Answer</strong></summary>

**Multi-Stage Build Deep Dive:**

---

### How They Work

**Concept:**
```
Multiple FROM statements in one Dockerfile
Each FROM starts a new stage
Final stage becomes the image
Earlier stages discarded
```

**Basic Example:**
```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/server.js"]
```

**Process:**
```
Stage 1 (builder):
1. Use node:18 (full image)
2. Install ALL dependencies
3. Build application
4. Stage size: ~1GB

Stage 2 (final):
1. Use node:18-alpine (minimal)
2. Copy ONLY built artifacts
3. No build tools, no source
4. Final size: ~180MB

Discarded:
- node:18 base image
- Build tools
- Source code
- Dev dependencies
```

---

### Why Use Multi-Stage Builds

**1. Smaller Image Size**

```dockerfile
# Without multi-stage
FROM golang:1.21
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["./server"]

# Image size: ~1GB
# Contains: Go toolchain, build cache, source code
```

```dockerfile
# With multi-stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM alpine:3.19
COPY --from=builder /app/server .
CMD ["./server"]

# Image size: ~15MB
# Contains: Only compiled binary
```

**Reduction: 98.5%**

---

**2. Security**

```
Without multi-stage:
- Build tools present
- Source code included
- Dev dependencies
- Larger attack surface

With multi-stage:
- Only runtime binaries
- No build tools
- No source code
- Minimal attack surface
```

**Example:**
```dockerfile
# Bad: Compiler in production
FROM gcc:11
COPY server.c .
RUN gcc server.c -o server
CMD ["./server"]

# Attacker can:
# - Compile malicious code
# - Read source code
# - Exploit build tools


# Good: No compiler
FROM gcc:11 AS builder
COPY server.c .
RUN gcc server.c -o server

FROM alpine
COPY --from=builder /server .
CMD ["./server"]

# Attacker cannot:
# - Compile code (no gcc)
# - Read source (not included)
```

---

**3. Separation of Concerns**

```dockerfile
# Build environment vs Runtime environment

# Build stage: Tools needed
FROM node:18 AS builder
RUN apt-get update && apt-get install -y \
    python3 \
    make \
    g++
COPY . .
RUN npm install
RUN npm run build

# Runtime stage: Clean environment
FROM node:18-slim
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --production
CMD ["node", "dist/server.js"]
```

---

### Advanced Patterns

**Multiple Build Stages:**

```dockerfile
# Stage 1: Base dependencies
FROM node:18 AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Stage 2: Build
FROM base AS builder
COPY . .
RUN npm run build

# Stage 3: Test
FROM base AS tester
COPY . .
RUN npm run test

# Stage 4: Production
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY --from=builder /app/dist ./dist
USER node
CMD ["node", "dist/server.js"]
```

**Build specific stage:**
```bash
# Build for testing
docker build --target tester -t myapp:test .

# Build for production
docker build --target production -t myapp:prod .
```

---

**Copy from External Images:**

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

# Copy from official nginx
FROM nginx:alpine
COPY --from=builder /app/server /usr/local/bin/
COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=nginx:alpine /usr/share/nginx/html /usr/share/nginx/html
```

---

**Named Stages for Clarity:**

```dockerfile
# Clear stage names
FROM node:18 AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:18 AS builder
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:18 AS test-runner
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
RUN npm test

FROM node:18-alpine AS production-runtime
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY --from=builder /app/dist ./dist
USER node
CMD ["node", "dist/server.js"]
```

---

### Real-World Example: React + Node.js

```dockerfile
# Build frontend
FROM node:18 AS frontend-builder
WORKDIR /frontend
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

# Build backend
FROM node:18 AS backend-builder
WORKDIR /backend
COPY backend/package*.json ./
RUN npm ci
COPY backend/ .
RUN npm run build

# Production
FROM node:18-alpine
WORKDIR /app

# Copy backend
COPY --from=backend-builder /backend/dist ./dist
COPY --from=backend-builder /backend/package*.json ./
RUN npm ci --production

# Copy frontend build
COPY --from=frontend-builder /frontend/build ./public

USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

**Sizes:**
```
frontend-builder: 1.2GB
backend-builder:  1.1GB
Final image:      220MB
```

---

### Build Cache Optimization

```dockerfile
# Optimize for cache hits
FROM node:18 AS builder

# Layer 1: Dependencies (changes rarely)
COPY package*.json ./
RUN npm ci

# Layer 2: Source code (changes often)
COPY . .
RUN npm run build

FROM node:18-alpine
COPY --from=builder /app/dist ./dist
```

**Build times:**
```
First build:
- Layer 1: 5 min (npm ci)
- Layer 2: 2 min (build)
Total: 7 min

Code change:
- Layer 1: cached ✓
- Layer 2: 2 min
Total: 2 min

Dependency change:
- Layer 1: 5 min
- Layer 2: 2 min
Total: 7 min
```

---

### Common Use Cases

**1. Compiled Languages:**
```dockerfile
# Go
FROM golang:1.21 AS builder
RUN go build -o app .

FROM scratch
COPY --from=builder /app/app .
CMD ["./app"]

# Size: 5-10MB
```

**2. Frontend Frameworks:**
```dockerfile
# React, Vue, Angular
FROM node:18 AS builder
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html

# Size: 25-30MB
```

**3. Python Applications:**
```dockerfile
# Build wheels
FROM python:3.11 AS builder
RUN pip wheel -r requirements.txt

# Runtime
FROM python:3.11-slim
COPY --from=builder /wheels /wheels
RUN pip install /wheels/*

# Size: 150-200MB (vs 900MB)
```

---

### BuildKit Features

```dockerfile
# syntax=docker/dockerfile:1

# BuildKit-specific features
FROM node:18 AS builder
RUN --mount=type=cache,target=/root/.npm \
    npm install

# Persistent cache across builds
```

**Build with BuildKit:**
```bash
DOCKER_BUILDKIT=1 docker build -t myapp .
```

</details>

### Question 2: How do you optimize Docker images for production?

<details>
<summary><strong>View Answer</strong></summary>

**Complete Optimization Strategy:**

---

### 1. Choose Minimal Base Images

**Size Comparison:**
```
ubuntu:22.04       → 77MB
debian:12-slim     → 74MB
alpine:3.19        → 7MB
scratch            → 0MB (empty)
```

**Decision Matrix:**
```
Use scratch:
✓ Go applications (static binaries)
✓ Rust applications
✓ Any statically linked binary

Use alpine:
✓ Node.js applications
✓ Python applications
✓ Need shell/basic utilities
✓ Size is critical

Use slim variants:
✓ Need glibc compatibility
✓ Complex dependencies
✓ Some C extensions (Python)

Use full images:
✗ Never in production
```

**Example:**
```dockerfile
# Go with scratch
FROM golang:1.21 AS builder
RUN CGO_ENABLED=0 go build -o app .

FROM scratch
COPY --from=builder /app/app .
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
CMD ["./app"]

# Size: 8MB
```

---

### 2. Multi-Stage Builds

```dockerfile
# Before: 1.2GB
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/server.js"]

# After: 180MB
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --production
USER node
CMD ["node", "dist/server.js"]

# Reduction: 85%
```

---

### 3. Layer Optimization

**Bad (Many Layers):**
```dockerfile
FROM alpine
RUN apk add curl
RUN apk add git
RUN apk add vim
RUN curl -o file.tar.gz https://example.com/file.tar.gz
RUN tar -xzf file.tar.gz
RUN rm file.tar.gz

# 6 layers
```

**Good (Minimal Layers):**
```dockerfile
FROM alpine
RUN apk add --no-cache curl git && \
    curl -o file.tar.gz https://example.com/file.tar.gz && \
    tar -xzf file.tar.gz && \
    rm file.tar.gz

# 1 layer
```

---

### 4. Dependency Management

**Node.js:**
```dockerfile
# Production dependencies only
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production --ignore-scripts && \
    npm cache clean --force
COPY . .
CMD ["node", "server.js"]
```

**Python:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**Go:**
```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o app .

FROM scratch
COPY --from=builder /app/app .
CMD ["./app"]

# -ldflags="-s -w" removes debug info
```

---

### 5. Cache Optimization

**Proper Layering:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Layer 1: System dependencies (rarely change)
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq5 && \
    rm -rf /var/lib/apt/lists/*

# Layer 2: Python dependencies (change occasionally)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Layer 3: Application code (changes frequently)
COPY . .

CMD ["python", "app.py"]
```

**Build Cache Usage:**
```
First build:  All layers built (5 min)
Code change:  Only Layer 3 rebuilt (10 sec)
Deps change:  Layer 2-3 rebuilt (2 min)
```

---

### 6. Remove Unnecessary Files

**.dockerignore:**
```
# Development
.git
.gitignore
.env
.env.*

# Dependencies
node_modules
venv
__pycache__
*.pyc

# IDE
.vscode
.idea
*.swp

# Tests
tests
test
*.test.js
*.spec.js
__tests__
coverage

# Documentation
README.md
docs
*.md
LICENSE

# Build artifacts
dist
build
*.log
tmp
temp

# OS
.DS_Store
Thumbs.db
```

**Cleanup in Dockerfile:**
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && \
    rm -rf node_modules src tests && \
    npm ci --production && \
    npm cache clean --force

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json .
USER node
CMD ["node", "dist/server.js"]
```

---

### 7. Security Hardening

**Non-Root User:**
```dockerfile
FROM node:18-alpine

# Create user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Install dependencies as root
COPY package*.json ./
RUN npm ci --production

# Copy app files with correct ownership
COPY --chown=nodejs:nodejs . .

# Switch to non-root
USER nodejs

CMD ["node", "server.js"]
```

**Read-Only Filesystem:**
```dockerfile
FROM python:3.11-slim

RUN useradd -m -u 1000 appuser

WORKDIR /app
COPY --chown=appuser:appuser . .
RUN pip install --no-cache-dir -r requirements.txt

USER appuser

# Run with read-only root filesystem
# Only /tmp writable
```

**Run Container:**
```bash
docker run \
  --read-only \
  --tmpfs /tmp \
  --security-opt=no-new-privileges \
  myapp
```

---

### 8. Compression Techniques

**Binary Stripping:**
```dockerfile
# Go
RUN CGO_ENABLED=0 go build \
    -ldflags="-s -w" \
    -o app .

# -s: Omit symbol table
# -w: Omit DWARF debug info
# Reduction: 30-40%
```

**UPX Compression:**
```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o app .

FROM alpine:3.19 AS compressor
RUN apk add --no-cache upx
COPY --from=builder /app/app .
RUN upx --best --lzma app

FROM scratch
COPY --from=compressor /app .
CMD ["./app"]

# Additional 50-70% reduction
```

---

### 9. Production-Ready Example

```dockerfile
# syntax=docker/dockerfile:1

# Build stage
FROM node:18-alpine AS builder

# Build args
ARG NODE_ENV=production

# Install build tools
RUN apk add --no-cache python3 make g++

WORKDIR /app

# Dependencies
COPY package.json package-lock.json ./
RUN npm ci --include=dev

# Build
COPY . .
RUN npm run build && \
    npm prune --production

# Production stage
FROM node:18-alpine

# Metadata
LABEL maintainer="dev@example.com"

# Install runtime dependencies
RUN apk add --no-cache \
    tini \
    dumb-init

# Create user
RUN addgroup -g 1001 nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

WORKDIR /app

# Copy from builder
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs package.json ./

# Environment
ENV NODE_ENV=production \
    PORT=3000

# Switch to non-root
USER nodejs

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js || exit 1

EXPOSE 3000

# Use tini for signal handling
ENTRYPOINT ["/sbin/tini", "--"]

CMD ["node", "dist/server.js"]
```

**Optimizations Applied:**
```
✓ Multi-stage build
✓ Alpine base
✓ Layer optimization
✓ Production dependencies only
✓ Non-root user
✓ Health check
✓ Proper signal handling
✓ Metadata labels

Final size: 180MB (vs 1.2GB original)
```

---

### 10. Measurement and Monitoring

**Analyze Image:**
```bash
# Image size
docker images myapp

# Layer sizes
docker history myapp

# Detailed analysis
docker image inspect myapp

# Dive tool (layer-by-layer analysis)
dive myapp
```

**Optimization Checklist:**
```
☐ Use minimal base image
☐ Multi-stage build implemented
☐ Layers optimized and combined
☐ Production dependencies only
☐ .dockerignore configured
☐ Non-root user
☐ Health check added
☐ Cleanup in same layer
☐ No unnecessary packages
☐ Specific image tags
☐ Security scanning passed
☐ Image size < 500MB (target)
```

</details>

### Question 3: What are the security best practices for Dockerfiles?

<details>
<summary><strong>View Answer</strong></summary>

**Security-Hardened Dockerfile Guide:**

---

### 1. Never Run as Root

**Problem:**
```dockerfile
FROM node:18
WORKDIR /app
COPY . .
CMD ["node", "server.js"]

# Runs as root (UID 0)
# Container escape = root on host
```

**Solution:**
```dockerfile
FROM node:18-alpine

# Create dedicated user
RUN addgroup -g 1001 appgroup && \
    adduser -D -u 1001 -G appgroup appuser

WORKDIR /app

# Install dependencies as root (if needed)
COPY package*.json ./
RUN npm ci --production

# Copy with correct ownership
COPY --chown=appuser:appgroup . .

# Switch to non-root user
USER appuser

CMD ["node", "server.js"]
```

**Verify:**
```bash
docker run myapp whoami
# Output: appuser (not root)
```

---

### 2. Use Specific Image Tags

**Bad:**
```dockerfile
FROM node
FROM node:latest
FROM python
```

**Good:**
```dockerfile
FROM node:18.19.0-alpine3.19
FROM python:3.11.7-slim-bookworm
```

**Why:**
```
latest/node:
- Content changes
- Security updates break apps
- Not reproducible
- Supply chain risk

node:18.19.0-alpine3.19:
- Exact version
- Reproducible
- Controlled updates
- Audit trail
```

---

### 3. Scan Images for Vulnerabilities

**Tools:**

**Docker Scout:**
```bash
# Enable Scout
docker scout quickview myapp:latest

# Detailed analysis
docker scout cves myapp:latest

# Policy violations
docker scout policy myapp:latest
```

**Trivy:**
```bash
# Install
brew install aquasecurity/trivy/trivy

# Scan image
trivy image myapp:latest

# Only high/critical
trivy image --severity HIGH,CRITICAL myapp:latest

# Scan Dockerfile
trivy config Dockerfile
```

**Snyk:**
```bash
# Install
npm install -g snyk

# Scan
snyk container test myapp:latest
```

**Example Output:**
```
HIGH: CVE-2023-12345
Package: openssl@1.1.1
Severity: HIGH
Fixed in: 1.1.1w

Action: Update base image
```

---

### 4. Minimize Attack Surface

**Remove Unnecessary Tools:**
```dockerfile
# Bad
FROM ubuntu
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    vim \
    git \
    gcc \
    make

# Attacker has many tools


# Good
FROM alpine:3.19
RUN apk add --no-cache curl

# Minimal tools available
```

**Use Distroless:**
```dockerfile
# Google Distroless (no shell, no package manager)
FROM gcr.io/distroless/nodejs18-debian11

COPY --chown=nonroot:nonroot package*.json ./
COPY --chown=nonroot:nonroot node_modules ./node_modules
COPY --chown=nonroot:nonroot dist ./dist

USER nonroot

CMD ["dist/server.js"]

# No shell: Cannot execute arbitrary commands
```

---

### 5. Handle Secrets Securely

**Never Hardcode Secrets:**
```dockerfile
# BAD - Secrets in image
ENV DB_PASSWORD=supersecret
ENV API_KEY=abc123

# Visible in:
# - docker history
# - docker inspect
# - Image layers
```

**Use Build Secrets:**
```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine

WORKDIR /app

# Secret only during build
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install
```

**Build:**
```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp .
```

**Runtime Secrets:**
```bash
# Pass at runtime
docker run -e DB_PASSWORD=$DB_PASSWORD myapp

# Or use secrets management
docker run --env-file secrets.env myapp

# Kubernetes
# Use Secrets resource
```

---

### 6. Use Minimal Privileges

**Capabilities:**
```bash
# Drop all capabilities
docker run --cap-drop=ALL myapp

# Add only needed
docker run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  myapp
```

**Read-Only Filesystem:**
```dockerfile
FROM python:3.11-slim

RUN useradd -m appuser

WORKDIR /app
COPY --chown=appuser:appuser . .
RUN pip install -r requirements.txt

USER appuser

# Application must handle read-only /
```

**Run:**
```bash
docker run \
  --read-only \
  --tmpfs /tmp \
  myapp
```

**Security Options:**
```bash
docker run \
  --security-opt=no-new-privileges \
  --security-opt=seccomp=seccomp-profile.json \
  myapp
```

---

### 7. Verify Image Integrity

**Content Trust:**
```bash
# Enable
export DOCKER_CONTENT_TRUST=1

# Pull only signed images
docker pull myapp:latest

# Sign and push
docker trust sign myapp:1.0.0
docker push myapp:1.0.0
```

**Image Signing (Cosign):**
```bash
# Install cosign
brew install cosign

# Generate keys
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key myapp:1.0.0

# Verify
cosign verify --key cosign.pub myapp:1.0.0
```

---

### 8. Network Security

**Disable Network (if possible):**
```bash
docker run --network=none myapp
```

**Limit Network Access:**
```dockerfile
# Only allow specific connections
FROM node:18-alpine

# Install and configure firewall in container
RUN apk add --no-cache iptables
```

---

### 9. Resource Limits

```dockerfile
FROM node:18-alpine

# Set in runtime, but document expected limits
LABEL memory.limit="512m" \
      cpu.limit="0.5"

WORKDIR /app
COPY . .

USER node
CMD ["node", "server.js"]
```

**Run with limits:**
```bash
docker run \
  --memory=512m \
  --cpus=0.5 \
  --pids-limit=100 \
  myapp
```

---

### 10. Audit and Logging

```dockerfile
FROM node:18-alpine

# Install audit tools
RUN apk add --no-cache \
    auditd \
    aide

WORKDIR /app
COPY . .

USER node

# Log to stdout/stderr
CMD ["node", "server.js"]
```

**Run with logging:**
```bash
docker run \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp
```

---

### 11. Complete Security-Hardened Example

```dockerfile
# syntax=docker/dockerfile:1

# Build stage
FROM node:18-alpine AS builder

# Install only build dependencies
RUN apk add --no-cache python3 make g++

WORKDIR /app

# Copy dependency files
COPY package.json package-lock.json ./

# Install dependencies (use npm ci for reproducible builds)
RUN npm ci --production && \
    npm cache clean --force

# Copy source
COPY . .

# Build
RUN npm run build

# Production stage - Use distroless for minimal attack surface
FROM gcr.io/distroless/nodejs18-debian11

# Metadata
LABEL maintainer="security@example.com" \
      version="1.0" \
      security.scan="passed" \
      security.scan-date="2024-01-17"

WORKDIR /app

# Copy only production files
COPY --from=builder --chown=nonroot:nonroot /app/dist ./dist
COPY --from=builder --chown=nonroot:nonroot /app/node_modules ./node_modules
COPY --chown=nonroot:nonroot package.json ./

# Use non-root user (distroless provides nonroot)
USER nonroot

# Expose port
EXPOSE 3000

# No health check command available (no shell)
# Must configure externally

# Start application
CMD ["dist/server.js"]
```

**Build:**
```bash
# Scan before building
trivy config Dockerfile

# Build
docker build -t myapp:1.0.0 .

# Scan image
trivy image --severity HIGH,CRITICAL myapp:1.0.0

# Sign
cosign sign --key cosign.key myapp:1.0.0
```

**Run:**
```bash
docker run \
  --name myapp \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --memory=512m \
  --cpus=0.5 \
  --pids-limit=100 \
  --network=myapp-network \
  --health-cmd='wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1' \
  --health-interval=30s \
  -e NODE_ENV=production \
  myapp:1.0.0
```

---

### Security Checklist

```
Image:
☐ Specific version tags
☐ Official or verified base images
☐ Minimal base (alpine/distroless)
☐ Multi-stage builds
☐ No unnecessary tools

User:
☐ Non-root user
☐ Correct file ownership
☐ Minimal permissions

Secrets:
☐ No hardcoded secrets
☐ Build secrets used properly
☐ Runtime secrets via env/files
☐ No secrets in layers

Scanning:
☐ Vulnerability scan passed
☐ No HIGH/CRITICAL CVEs
☐ SBOM generated
☐ Image signed

Runtime:
☐ Read-only filesystem
☐ Dropped capabilities
☐ Resource limits
☐ No new privileges
☐ Network restrictions

Monitoring:
☐ Health checks configured
☐ Logging enabled
☐ Audit trail
☐ Security updates planned
```

</details>

</details>

---

[← Previous: 9.2 Kubernetes Architecture](../09-orchestration/03-kubernetes-architecture.md) | [Next: 10.1 Best Practices →](../10-best-practices/02-production.md)