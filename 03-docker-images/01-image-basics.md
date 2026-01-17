# 3.1 Working with Images

Understanding Docker images: what they are, how to build, pull, push, and manage them.

---

## What is a Docker Image?

A **Docker image** is a read-only template containing everything needed to run an application:

```
Docker Image Contains:
├── Base operating system (minimal)
├── Application code
├── Runtime (Node.js, Python, Java, etc.)
├── Libraries and dependencies
├── Environment variables
├── Configuration files
└── Default commands to run
```

### Image Characteristics

```
Read-only: Cannot be changed once built
Layered: Built from multiple layers stacked
Portable: Runs anywhere Docker runs
Versioned: Tagged for different versions
Shareable: Stored in registries
```

---

## Image Layers

Docker images are built from **layers** stacked on top of each other.

### Layer Concept

```
Image: myapp:1.0

┌───────────────────────────┐
│ Layer 4: COPY app.py      │  2 MB
├───────────────────────────┤
│ Layer 3: RUN pip install  │  50 MB
├───────────────────────────┤
│ Layer 2: COPY requirements│ 1 KB
├───────────────────────────┤
│ Layer 1: FROM python:3.9  │  150 MB
└───────────────────────────┘

Each Dockerfile instruction = One layer
Layers are cached and shared
Total size: ~203 MB
```

### Why Layers Matter

**1. Storage Efficiency:**
```
Image A: nginx:latest
├── Layer 1: Ubuntu base (50 MB)
├── Layer 2: nginx binary (30 MB)
└── Layer 3: config (1 MB)

Image B: nginx:alpine
├── Layer 1: Alpine base (5 MB)
├── Layer 2: nginx binary (30 MB)  ← Same layer!
└── Layer 3: config (1 MB)

Storage used:
Without sharing: 50+30+1 + 5+30+1 = 117 MB
With sharing: 50+30+1+5 = 86 MB (30 MB shared)
```

**2. Build Speed:**
```
Building myapp:1.0:
Step 1: FROM python:3.9 → Cached ✓
Step 2: COPY requirements.txt → Cached ✓
Step 3: RUN pip install → Cached ✓
Step 4: COPY app.py → New (code changed)

Only Step 4 executes
Build time: 5 seconds instead of 5 minutes
```

**3. Layer Reuse:**
```
Multiple developers:
Developer A builds: myapp:dev
Developer B builds: myapp:test
Both use: FROM python:3.9

Python base layer downloaded once
Saved: 150 MB × number of developers
```

### Viewing Layers

```bash
# View image layers
docker history nginx:latest

IMAGE          CREATED BY                                     SIZE
a1b2c3d4e5f6   CMD ["nginx" "-g" "daemon off;"]               0B
b2c3d4e5f6a7   EXPOSE 80                                      0B
c3d4e5f6a7b8   COPY file:abc123 in /etc/nginx/                1.2kB
d4e5f6a7b8c9   RUN apt-get update && apt-get install          52MB
e5f6a7b8c9d0   FROM ubuntu:20.04                              72MB

# Detailed layer info
docker inspect nginx:latest
```

---

## Building Images

### From Dockerfile

```bash
# Build image from current directory
docker build -t myapp:1.0 .

# Build with specific Dockerfile
docker build -f Dockerfile.prod -t myapp:prod .

# Build without cache
docker build --no-cache -t myapp:1.0 .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t myapp:1.0 .
```

### Build Context

```
Build context = Files sent to Docker daemon

Current directory structure:
.
├── Dockerfile
├── app/
│   ├── main.py
│   └── utils.py
├── requirements.txt
├── tests/         ← Not needed in image
└── .git/          ← Not needed in image

# Bad: Sends everything (slow, large)
docker build -t myapp .

# Good: Use .dockerignore
cat .dockerignore
tests/
.git/
*.pyc
__pycache__/

Result: Only sends necessary files
```

### Multi-stage Builds

**Problem:**
```dockerfile
# Single stage (produces large image)
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install  # Includes dev dependencies
COPY . .
RUN npm run build
CMD ["npm", "start"]

Result: 1.2 GB image (includes build tools, dev deps)
```

**Solution:**
```dockerfile
# Multi-stage build (smaller final image)

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
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]

Result: 200 MB image (only runtime needed)
Savings: 1 GB!
```

**Real-World Example:**

```dockerfile
# Go application multi-stage build

# Stage 1: Build
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server

# Stage 2: Minimal runtime
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]

Benefits:
- Build stage: 800 MB (Go compiler, tools)
- Final image: 15 MB (just binary + alpine)
- 98% size reduction
```

---

## Pulling Images

### From Docker Hub

```bash
# Pull latest version
docker pull nginx

# Pull specific version
docker pull nginx:1.25

# Pull specific digest (exact image)
docker pull nginx@sha256:abc123...

# Pull from different registry
docker pull ghcr.io/company/app:latest
```

### Image Naming Convention

```
Format: [registry/][namespace/]repository:tag

Examples:

nginx
├── Registry: docker.io (default)
├── Namespace: library (official)
├── Repository: nginx
└── Tag: latest (default)

mycompany/webapp:1.0
├── Registry: docker.io
├── Namespace: mycompany
├── Repository: webapp
└── Tag: 1.0

gcr.io/myproject/api:v2.3
├── Registry: gcr.io
├── Namespace: myproject
├── Repository: api
└── Tag: v2.3
```

### Verifying Image Integrity

```bash
# Pull with digest for reproducibility
docker pull nginx@sha256:abc123def456...

# Image with this digest is ALWAYS the same
# Even if nginx:latest changes, this digest won't

# View image digest
docker images --digests

REPOSITORY   TAG      DIGEST                    IMAGE ID
nginx        latest   sha256:abc123def456...    a1b2c3d4e5f6
```

---

## Pushing Images

### To Docker Hub

```bash
# Login to Docker Hub
docker login

# Tag image with username
docker tag myapp:1.0 username/myapp:1.0

# Push to Docker Hub
docker push username/myapp:1.0

# Push all tags
docker push username/myapp --all-tags
```

### To Private Registry

```bash
# Tag for private registry
docker tag myapp:1.0 registry.company.com/myapp:1.0

# Login to private registry
docker login registry.company.com

# Push to private registry
docker push registry.company.com/myapp:1.0
```

### Automated Build Pipeline

```bash
# CI/CD example (GitHub Actions)

# 1. Build
docker build -t myapp:${GIT_SHA} .

# 2. Tag with multiple tags
docker tag myapp:${GIT_SHA} company/myapp:latest
docker tag myapp:${GIT_SHA} company/myapp:${VERSION}
docker tag myapp:${GIT_SHA} company/myapp:${GIT_SHA}

# 3. Push all tags
docker push company/myapp:latest
docker push company/myapp:${VERSION}
docker push company/myapp:${GIT_SHA}

Result:
- latest: Points to newest version
- v1.2.3: Specific version
- abc123: Specific commit (reproducible)
```

---

## Managing Images

### Listing Images

```bash
# List all images
docker images
docker image ls

# List with specific format
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Filter images
docker images --filter "dangling=true"  # Untagged images
docker images --filter "reference=nginx*"  # Name pattern
docker images --filter "before=nginx:latest"  # Older than

# Show image sizes
docker images --format "{{.Repository}}:{{.Tag}}: {{.Size}}"
```

### Removing Images

```bash
# Remove by name
docker rmi nginx:latest
docker image rm nginx:latest

# Remove by ID
docker rmi a1b2c3d4e5f6

# Remove multiple
docker rmi nginx:1.25 nginx:1.24 redis:7

# Force remove (even if containers using it)
docker rmi -f nginx:latest

# Remove dangling images (untagged)
docker image prune

# Remove all unused images
docker image prune -a

# Remove with filter
docker image prune -a --filter "until=24h"
```

### Tagging Images

```bash
# Create new tag
docker tag myapp:1.0 myapp:latest

# Tag for different registry
docker tag myapp:1.0 gcr.io/project/myapp:1.0

# Multiple tags for same image
docker tag myapp:1.0 myapp:stable
docker tag myapp:1.0 myapp:production
docker tag myapp:1.0 myapp:2024-01-17

# All point to same image (same IMAGE ID)
```

---

## Inspecting Images

### Basic Information

```bash
# Detailed image information
docker inspect nginx:latest

# Get specific field
docker inspect --format='{{.Architecture}}' nginx
docker inspect --format='{{.Size}}' nginx

# View environment variables
docker inspect --format='{{.Config.Env}}' nginx

# View exposed ports
docker inspect --format='{{.Config.ExposedPorts}}' nginx
```

### Image Contents

```bash
# View filesystem changes in layers
docker history nginx:latest

# Export image to tar
docker save nginx:latest -o nginx.tar

# Extract and explore
tar -xf nginx.tar
ls -la

# View image configuration
cat manifest.json | jq

# Load image from tar
docker load -i nginx.tar
```

---

## Image Storage

### Local Storage Location

```bash
# Default storage location
/var/lib/docker/

Structure:
/var/lib/docker/
├── image/
│   └── overlay2/
│       ├── imagedb/     # Image metadata
│       ├── layerdb/     # Layer metadata
│       └── repositories.json
│
└── overlay2/            # Actual layer data
    ├── abc123/          # Layer 1
    ├── def456/          # Layer 2
    └── ghi789/          # Layer 3
```

### Storage Drivers

```bash
# Check storage driver
docker info | grep "Storage Driver"

Common drivers:
- overlay2 (recommended, default on modern systems)
- aufs (older systems)
- devicemapper (RHEL 7)
- btrfs (if using btrfs filesystem)

# Configure storage driver
# /etc/docker/daemon.json
{
  "storage-driver": "overlay2"
}
```

---

## Real-World Examples

### Example 1: Microservice Deployment

```bash
# Development workflow

# 1. Developer builds locally
cd user-service
docker build -t user-service:dev .
docker run -d -p 3001:3000 user-service:dev

# 2. Test locally
curl http://localhost:3001/health

# 3. Tag for registry
docker tag user-service:dev registry.company.com/user-service:v1.2.3

# 4. Push to registry
docker push registry.company.com/user-service:v1.2.3

# 5. Deploy to staging
ssh staging-server
docker pull registry.company.com/user-service:v1.2.3
docker run -d -p 3001:3000 registry.company.com/user-service:v1.2.3

# 6. If successful, tag as stable
docker tag registry.company.com/user-service:v1.2.3 \
           registry.company.com/user-service:stable
docker push registry.company.com/user-service:stable

# 7. Production deployment
ssh prod-server
docker pull registry.company.com/user-service:stable
docker run -d -p 3001:3000 registry.company.com/user-service:stable
```

### Example 2: Multi-Architecture Build

```bash
# Build for multiple platforms

# Setup buildx
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# Build for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t username/myapp:latest \
  --push \
  .

Result:
- One tag (latest)
- Three architecture variants
- Docker automatically pulls correct architecture
```

### Example 3: Image Optimization

```bash
# Before optimization
FROM ubuntu:20.04
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y python3-pip
COPY requirements.txt .
RUN pip3 install -r requirements.txt
COPY . .
CMD ["python3", "app.py"]

Size: 850 MB
Layers: 7
Build time: 3 minutes

# After optimization
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "app.py"]

Size: 180 MB
Layers: 4
Build time: 1 minute

Improvements:
- Smaller base image (slim vs ubuntu)
- Combined RUN commands
- No package manager cache
- 80% size reduction
```

---

## Best Practices

### 1. Use Specific Tags

```bash
# Bad: Tag changes over time
FROM node:latest

# Good: Reproducible builds
FROM node:18.19.0-alpine3.19
```

### 2. Minimize Layers

```bash
# Bad: Many layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get install -y vim

# Good: One layer
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    vim \
 && rm -rf /var/lib/apt/lists/*
```

### 3. Use .dockerignore

```bash
# .dockerignore
.git
node_modules
npm-debug.log
Dockerfile
.dockerignore
.env
*.md
tests/

Benefits:
- Faster builds
- Smaller context
- Better security (no secrets copied)
```

### 4. Order Layers by Change Frequency

```dockerfile
# Bad: Frequently changing layers first
FROM node:18
COPY . .                    # Changes often
COPY package*.json ./       # Changes rarely
RUN npm install

# Good: Rarely changing layers first
FROM node:18
COPY package*.json ./       # Changes rarely
RUN npm install             # Cached if package.json unchanged
COPY . .                    # Changes often
```

### 5. Multi-stage for Production

```dockerfile
# Always use multi-stage for compiled languages
# Keeps build tools out of final image
# Results in smaller, more secure images
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Docker images are built from multiple __________ stacked on top of each other.
2. Each instruction in a Dockerfile creates a new __________.
3. The default registry for Docker images is __________.
4. An image tagged with __________ will be pulled if no tag is specified.
5. The __________ file tells Docker which files to exclude from the build context.
6. Multi-stage builds help reduce final image __________ by excluding build tools.

### True/False

1. ⬜ Docker images are mutable and can be changed after creation
2. ⬜ Layers are shared between images that have common base layers
3. ⬜ The FROM instruction must be the first instruction in a Dockerfile
4. ⬜ Pulling nginx:latest today will always give you the same image forever
5. ⬜ You can have multiple tags pointing to the same image
6. ⬜ Dangling images are images without a tag
7. ⬜ Multi-stage builds require multiple Dockerfiles

### Multiple Choice

1. What is the benefit of Docker image layers?
   - A) Makes images larger
   - B) Enables caching and sharing
   - C) Slows down builds
   - D) Increases security only

2. Which command removes all unused images?
   - A) docker rmi -a
   - B) docker image prune
   - C) docker image prune -a
   - D) docker clean

3. What does the digest in an image represent?
   - A) Image size
   - B) Creation date
   - C) Cryptographic hash (exact image content)
   - D) Number of layers

4. Where are Docker images stored locally by default?
   - A) /home/docker/images
   - B) /var/lib/docker
   - C) /etc/docker/images
   - D) /opt/docker

5. What is the purpose of multi-stage builds?
   - A) Build multiple images at once
   - B) Reduce final image size
   - C) Speed up builds
   - D) Enable parallel builds

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. layers
2. layer
3. Docker Hub (or docker.io, registry-1.docker.io)
4. latest
5. .dockerignore
6. size (or footprint)

**True/False:**
1. ❌ False - Images are immutable (read-only)
2. ✅ True - Layer sharing saves storage and bandwidth
3. ✅ True - FROM must be first (except ARG)
4. ❌ False - latest tag can point to different images over time
5. ✅ True - Multiple tags can reference same IMAGE ID
6. ✅ True - Dangling images are untagged intermediate images
7. ❌ False - Multi-stage uses one Dockerfile with multiple FROM statements

**Multiple Choice:**
1. **B** - Enables caching and sharing
2. **C** - docker image prune -a (removes all unused, not just dangling)
3. **C** - Cryptographic hash (exact image content)
4. **B** - /var/lib/docker
5. **B** - Reduce final image size (by excluding build tools)

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain how Docker image layers work and why they're beneficial

<details>
<summary><strong>View Answer</strong></summary>

**How Layers Work:**

Each instruction in a Dockerfile creates a new read-only layer:

```dockerfile
FROM ubuntu:20.04        # Layer 1: 72 MB
RUN apt-get update       # Layer 2: 50 MB
COPY app.py /app/        # Layer 3: 1 MB
CMD ["python", "app.py"] # Layer 4: 0 MB (metadata only)
```

```
Resulting Image:
┌─────────────────────────┐
│ Layer 4: CMD           │  0 B
├─────────────────────────┤
│ Layer 3: COPY app.py   │  1 MB
├─────────────────────────┤
│ Layer 2: RUN apt-get   │  50 MB
├─────────────────────────┤
│ Layer 1: FROM ubuntu   │  72 MB
└─────────────────────────┘
Total: 123 MB
```

**Key Benefits:**

**1. Storage Efficiency**
```
Image A (nginx:latest):
Layer 1: Ubuntu base (50 MB)
Layer 2: nginx (30 MB)

Image B (nginx:custom):
Layer 1: Ubuntu base (50 MB) ← Same layer, stored once!
Layer 2: nginx (30 MB)       ← Same layer, stored once!
Layer 3: Custom config (1 MB)

Storage: 50 + 30 + 1 = 81 MB
Not: 50 + 30 + 50 + 30 + 1 = 161 MB
Saved: 80 MB
```

**2. Build Performance**
```
Rebuilding after code change:

Dockerfile:
FROM python:3.9          # Layer 1: Cached ✓
COPY requirements.txt .  # Layer 2: Cached ✓
RUN pip install          # Layer 3: Cached ✓
COPY app.py .            # Layer 4: NEW (app changed)

Build time:
- Without cache: 5 minutes (download Python, install packages)
- With cache: 5 seconds (only copy new app.py)
```

**3. Distribution Efficiency**
```
Developer A already has: python:3.9 base

Developer B pushes: myapp:1.0
- Layer 1: python:3.9 (already on A's machine)
- Layer 2: pip packages
- Layer 3: app code

Download for A:
- Skips Layer 1 (already has it)
- Downloads only Layers 2 & 3
- Saves: 150 MB, 2 minutes
```

**4. Atomic Updates**
```
Layers are immutable:
- Once created, never changes
- Identified by hash
- Pull/push only new layers

Update workflow:
Image v1.0: Layers [A, B, C]
Image v1.1: Layers [A, B, D]

Update:
- Layers A, B: Already have
- Layer D: Download only this
- Fast, efficient update
```

**Real-World Example:**

```
Company with 50 developers:

Base image: company-base:latest (500 MB)
- Python 3.9
- Common libraries
- Company tools

Each microservice:
- Starts FROM company-base:latest
- Adds service code (10-50 MB)

Without layers:
50 services × 550 MB = 27,500 MB per developer
Total: 1,375 GB for 50 developers

With layers:
Base layer: 500 MB (shared)
Service layers: 50 × 30 MB = 1,500 MB
Total: 2 GB per developer = 100 GB for 50 developers

Savings: 92% storage reduction
```

**Layer Caching Strategy:**

```dockerfile
# Optimal layer order:

# Rarely changes → Early layers
FROM python:3.9
COPY requirements.txt .
RUN pip install -r requirements.txt

# Frequently changes → Late layers  
COPY . .

Benefits:
- requirements.txt rarely changes
- pip install layer cached
- Only COPY . . invalidates cache
- Fast rebuilds during development
```

</details>

### Question 2: How would you optimize a Docker image that's currently 2GB down to under 200MB?

<details>
<summary><strong>View Answer</strong></summary>

**Step-by-Step Optimization:**

**Current Image (2GB):**
```dockerfile
FROM ubuntu:20.04
RUN apt-get update
RUN apt-get install -y python3 python3-pip curl wget vim nano
COPY requirements.txt .
RUN pip3 install -r requirements.txt
COPY . .
RUN python3 -m pytest  # Running tests in image
CMD ["python3", "app.py"]
```

**Problems:**
- Heavy base image (ubuntu)
- Installing unnecessary tools (vim, nano, wget)
- Not combining RUN commands
- Including test files and test runs
- Not cleaning package manager cache
- Single-stage build

**Optimization Steps:**

**Step 1: Use Minimal Base Image**
```dockerfile
# Before: ubuntu:20.04 (72 MB)
FROM ubuntu:20.04

# After: python:3.9-slim (122 MB) or python:3.9-alpine (45 MB)
FROM python:3.9-alpine

Savings: 27-72 MB
```

**Step 2: Multi-stage Build**
```dockerfile
# Stage 1: Build and test
FROM python:3.9 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt
COPY . .
RUN python -m pytest  # Test in builder, not in final image

# Stage 2: Production
FROM python:3.9-alpine
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY --from=builder /app/app.py .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]

Savings: Don't include test files, test dependencies, build tools
```

**Step 3: Optimize Dependencies**
```dockerfile
# Before: Install everything
pip install -r requirements.txt

# After: No cache, minimal footprint
RUN pip install --no-cache-dir -r requirements.txt

Savings: pip cache can be 100+ MB
```

**Step 4: Combine RUN Commands**
```dockerfile
# Before: Multiple layers
RUN apk add --no-cache gcc
RUN apk add --no-cache musl-dev
RUN apk add --no-cache libffi-dev

# After: One layer
RUN apk add --no-cache \
    gcc \
    musl-dev \
    libffi-dev

Savings: Reduced layer overhead
```

**Step 5: Clean Up in Same Layer**
```dockerfile
# Before: Leaves temp files
RUN apt-get update && apt-get install -y build-essential

# After: Clean up in same command
RUN apt-get update && \
    apt-get install -y --no-install-recommends build-essential && \
    rm -rf /var/lib/apt/lists/*

Savings: Package manager cache (50-100 MB)
```

**Step 6: Use .dockerignore**
```bash
# .dockerignore
.git
.github
tests/
*.pyc
__pycache__/
*.md
.env.example
Dockerfile
docker-compose.yml

Savings: Faster builds, smaller context
```

**Final Optimized Dockerfile:**
```dockerfile
# Multi-stage build
FROM python:3.9-slim AS builder

WORKDIR /app

# Install build dependencies in one layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

# Copy only requirements first (caching)
COPY requirements.txt .

# Install Python packages
RUN pip install --user --no-cache-dir -r requirements.txt

# Copy application code
COPY app/ ./app/

# Run tests (fail build if tests fail)
RUN python -m pytest app/tests/

# Production image
FROM python:3.9-alpine

WORKDIR /app

# Copy only necessary files from builder
COPY --from=builder /root/.local /root/.local
COPY --from=builder /app/app/*.py ./

# Make sure scripts are executable
ENV PATH=/root/.local/bin:$PATH

# Non-root user for security
RUN adduser -D appuser
USER appuser

CMD ["python", "main.py"]
```

**Results:**

```
Before Optimization:
├── Base: ubuntu:20.04 (72 MB)
├── Tools: vim, nano, wget, curl (100 MB)
├── Python packages: (500 MB)
├── Test files: (50 MB)
├── Build cache: (200 MB)
├── Application: (10 MB)
└── Total: 2.0 GB

After Optimization:
├── Base: python:3.9-alpine (45 MB)
├── Python packages: (80 MB - production only)
├── Application: (10 MB)
└── Total: 135 MB

Size reduction: 93% (2GB → 135MB)
```

**Additional Techniques:**

**For Static Binaries (Go, Rust):**
```dockerfile
# Go multi-stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server

# Minimal runtime
FROM scratch  # Empty image!
COPY --from=builder /app/server /
EXPOSE 8080
CMD ["/server"]

Final image: 10-20 MB (just the binary)
```

**For Node.js:**
```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Production
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY src/ ./src/
USER node
CMD ["node", "src/server.js"]

Savings: No devDependencies, no build tools
```

**Verification:**
```bash
# Check image size
docker images myapp

# Check layer sizes
docker history myapp:optimized

# Compare
echo "Before: 2GB"
echo "After: 135MB"
echo "Reduction: 93%"
```

</details>

### Question 3: Explain the difference between docker save/load and docker export/import

<details>
<summary><strong>View Answer</strong></summary>

**docker save/load - For Images**

**Purpose:** Save and load complete Docker **images** with all layers and metadata.

```bash
# Save image to tar file
docker save nginx:latest -o nginx.tar

# Load image from tar file
docker load -i nginx.tar
```

**What's Saved:**
```
nginx.tar contains:
├── manifest.json       # Image metadata
├── repositories        # Tags
├── layer1/            # Layer 1 tar
│   ├── layer.tar
│   └── json
├── layer2/            # Layer 2 tar
│   ├── layer.tar
│   └── json
└── layer3/            # Layer 3 tar
    ├── layer.tar
    └── json

Preserves:
✓ All layers
✓ Image history
✓ Tags
✓ Metadata
✓ Build configuration
```

**Use Cases:**
```
1. Transfer images between Docker hosts without registry
2. Backup images
3. Airgapped/offline environments
4. Archive specific image versions

Example:
# Development machine
docker save myapp:1.0 -o myapp.tar
scp myapp.tar production-server:/tmp/

# Production server (no internet)
docker load -i /tmp/myapp.tar
docker run myapp:1.0
```

---

**docker export/import - For Containers**

**Purpose:** Export and import **container filesystems** as flat tar archives.

```bash
# Export container filesystem
docker export container_name -o container.tar

# Import as new image
docker import container.tar new-image:tag
```

**What's Exported:**
```
container.tar contains:
└── Flat filesystem (no layers)
    ├── bin/
    ├── etc/
    ├── usr/
    ├── var/
    └── app/

Does NOT preserve:
✗ Layer history
✗ Build metadata
✗ Image tags
✗ Volume data
✗ Configuration (CMD, ENV, etc.)
```

**Use Cases:**
```
1. Create image from modified container
2. Flatten image (merge all layers)
3. Backup container state
4. Create minimal custom base images

Example:
# Modify running container
docker run -it ubuntu bash
# (install packages, configure)
# Exit container

# Export modified filesystem
docker export container_id -o custom-ubuntu.tar

# Import as new image
docker import custom-ubuntu.tar mycompany/custom-ubuntu:1.0

# Must specify CMD (not preserved)
docker import custom-ubuntu.tar mycompany/custom-ubuntu:1.0 \
  --change='CMD ["/bin/bash"]'
```

---

**Key Differences:**

| Feature | save/load | export/import |
|---------|-----------|---------------|
| **Operates on** | Images | Containers |
| **Preserves layers** | ✅ Yes | ❌ No (flattened) |
| **Preserves history** | ✅ Yes | ❌ No |
| **Preserves tags** | ✅ Yes | ❌ No |
| **Preserves metadata** | ✅ Yes | ❌ No |
| **Size** | Larger (all layers) | Smaller (single tar) |
| **Use case** | Backup images | Flatten containers |

---

**Practical Examples:**

**Example 1: Image Transfer (save/load)**
```bash
# Scenario: Transfer image to airgapped server

# Source machine
docker pull nginx:latest
docker save nginx:latest -o nginx.tar
ls -lh nginx.tar
# 142M

# Transfer file
scp nginx.tar airgapped-server:/tmp/

# Airgapped server
docker load -i /tmp/nginx.tar
docker images
# REPOSITORY   TAG      IMAGE ID
# nginx        latest   a1b2c3d4e5f6

docker run -d nginx:latest
# Works perfectly with all metadata preserved
```

**Example 2: Container Flattening (export/import)**
```bash
# Scenario: Flatten layers to reduce size

# Original image: 10 layers, 500 MB
docker history myapp:1.0
# 10 layers shown

# Run container
docker run -d --name myapp-running myapp:1.0

# Export (flattens all layers)
docker export myapp-running -o myapp-flat.tar

# Import as new image
docker import myapp-flat.tar myapp:flat \
  --change='CMD ["python", "app.py"]' \
  --change='WORKDIR /app'

# Result: 1 layer, 450 MB
docker history myapp:flat
# 1 layer shown

Savings: 50 MB + better storage efficiency
```

**Example 3: Creating Custom Base Image**
```bash
# Start with minimal base
docker run -it --name custom-base alpine sh

# Inside container - customize
apk add python3 python3-pip curl
pip3 install flask requests
# Exit

# Export
docker export custom-base -o custom-base.tar

# Import as base image
docker import custom-base.tar mycompany/python-base:1.0 \
  --change='CMD ["/bin/sh"]'

# Use in Dockerfile
FROM mycompany/python-base:1.0
COPY app.py .
CMD ["python3", "app.py"]
```

**Example 4: When save/load is Better**
```bash
# Scenario: Preserve multi-stage build

# Multi-stage image
docker build -t complex-app:1.0 .

# Save preserves all metadata
docker save complex-app:1.0 -o complex-app.tar

# Load on another machine
docker load -i complex-app.tar

# Check preserved data
docker inspect complex-app:1.0
# Shows:
# - All ENV variables
# - EXPOSE ports
# - WORKDIR
# - CMD/ENTRYPOINT
# - Labels
# - Build arguments

All preserved perfectly!
```

**Example 5: When export/import is Better**
```bash
# Scenario: Remove intermediate layers

# Image built with many layers
docker history bloated-app:1.0
# 50 layers, 1.2 GB

# Run and export
docker run -d --name bloat bloated-app:1.0
docker export bloat -o flat.tar

# Import
docker import flat.tar bloated-app:flat \
  --change='CMD ["npm", "start"]' \
  --change='EXPOSE 3000'

# Result
docker history bloated-app:flat
# 1 layer, 900 MB

Benefits:
- Fewer layers (better performance)
- Smaller size
- Simpler structure
```

**Summary Table:**

```
Choose save/load when:
✓ Transferring images to other hosts
✓ Backing up images with all metadata
✓ Need to preserve build history
✓ Working with multi-stage builds
✓ Preserving tags important

Choose export/import when:
✓ Flattening layers to reduce size
✓ Creating custom base images
✓ Backing up container state
✓ Don't need layer history
✓ Want single-layer image
```

</details>

</details>

---

[← Previous: 2.2 Linux Kernel Features](../02-docker-engine/02-kernel-features.md) | [Next: 3.2 Dockerfile →](02-dockerfile.md)