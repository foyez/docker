# 1.3 Docker Basics

Understanding core Docker concepts: what Docker is, images vs containers, and Docker architecture.

---

## What Is Docker?

**Docker** is a platform that enables developers to **build, ship, and run applications in containers**.

### Core Purpose

> Docker makes it easy to create, deploy, and run applications by using containers.

Docker provides:
1. **Tools** to build container images
2. **Runtime** to run containers
3. **Registry** to share images

### Docker vs Traditional Deployment

**Traditional Deployment:**
```
Developer:
├── Write code
├── Write deployment guide
└── "Good luck!"

Operations:
├── Read 50-page guide
├── Install dependencies
├── Configure servers
└── Hope it works
```

**Docker Deployment:**
```
Developer:
├── Write code
├── Write Dockerfile
└── docker build → Create image

Operations:
├── docker pull → Get image
└── docker run → Application running
```

### Real-World Analogy

**Traditional Approach = Recipe**
```
Recipe: "Chocolate Cake"
1. Preheat oven to 350°F (but what if my oven is different?)
2. Mix flour, sugar, eggs (what brands? what ratios exactly?)
3. Bake for 30 minutes (but my oven might need 35 minutes)

Result: Everyone gets different cakes
```

**Docker Approach = Frozen Cake**
```
Frozen Cake:
1. Remove from box
2. Thaw
3. Serve

Result: Everyone gets the exact same cake
```

Docker doesn't ship **instructions** (recipe), it ships the **final product** (frozen cake).

---

## Docker Image vs Docker Container

### Definitions

**Docker Image:**
- A **blueprint** or **template**
- **Read-only** file containing everything needed to run an application
- Stored in a registry (DockerHub, private registry)
- Like a **class** in programming

**Docker Container:**
- A **running instance** of an image
- **Read-write** layer on top of the image
- Isolated process running on the host
- Like an **object** in programming

### Analogy 1: Class vs Object

```
Programming Analogy:

class Dog {
    bark() { }
    eat() { }
}

Image = Class definition (blueprint)

dog1 = new Dog();
dog2 = new Dog();

Container = Objects (running instances)
```

### Analogy 2: Program vs Process

```
Operating System Analogy:

nginx.exe (on disk)
└── Image (executable file)

Running nginx processes:
├── nginx process 1 (PID 1234)
├── nginx process 2 (PID 5678)
└── nginx process 3 (PID 9012)
└── Containers (running processes)
```

### Analogy 3: Recipe vs Meal

```
Cooking Analogy:

Recipe in cookbook
└── Image (instructions)

Actual prepared meals:
├── Meal 1 (served to table 1)
├── Meal 2 (served to table 2)
└── Meal 3 (served to table 3)
└── Containers (actual meals)
```

### Visual Representation

```
┌─────────────────────────────┐
│     Docker Image            │
│     (nginx:latest)          │
│                             │
│  ┌───────────────────────┐  │
│  │  nginx binary         │  │
│  │  Config files         │  │
│  │  HTML files           │  │
│  │  Dependencies         │  │
│  └───────────────────────┘  │
│                             │
│  Read-Only                  │
│  Immutable                  │
└─────────────────────────────┘
              │
              │ docker run (creates)
              │
              ▼
┌─────────────┬─────────────┬─────────────┐
│ Container 1 │ Container 2 │ Container 3 │
│             │             │             │
│ ┌─────────┐ │ ┌─────────┐ │ ┌─────────┐ │
│ │ Writable│ │ │ Writable│ │ │ Writable│ │
│ │  Layer  │ │ │  Layer  │ │ │  Layer  │ │
│ └─────────┘ │ └─────────┘ │ └─────────┘ │
│ ┌─────────┐ │ ┌─────────┐ │ ┌─────────┐ │
│ │  Image  │ │ │  Image  │ │ │  Image  │ │
│ │  Layer  │ │ │  Layer  │ │ │  Layer  │ │
│ └─────────┘ │ └─────────┘ │ └─────────┘ │
│             │             │             │
│ Running     │ Running     │ Running     │
└─────────────┴─────────────┴─────────────┘
```

### Key Differences

| Aspect | Docker Image | Docker Container |
|--------|--------------|------------------|
| **Nature** | Template/Blueprint | Running instance |
| **State** | Static | Dynamic |
| **Mutability** | Immutable | Mutable (while running) |
| **Storage** | Read-only layers | Read-write layer + image layers |
| **Lifecycle** | Permanent (until deleted) | Temporary (until stopped/removed) |
| **Location** | Registry/Local storage | Running on host |
| **Analogy** | .exe file | Running process |
| **Count** | One image | Many containers from one image |

### Practical Example

**Image: nginx:latest**
```bash
# Pull image once
docker pull nginx:latest

# Image stored locally
# Size: 142 MB
# Read-only
```

**Containers from that image:**
```bash
# Create container 1 - Website A
docker run -d --name website-a -p 8080:80 nginx:latest

# Create container 2 - Website B
docker run -d --name website-b -p 8081:80 nginx:latest

# Create container 3 - Website C
docker run -d --name website-c -p 8082:80 nginx:latest

# Three different websites
# All running from same image
# Each with its own writable layer
# Each isolated from the others
```

### Lifecycle Relationship

```
Image Lifecycle:
Create → Store → Distribute → Run

┌──────────┐     ┌──────────┐     ┌──────────┐
│  Build   │ →   │   Push   │ →   │   Pull   │
│  Image   │     │    to    │     │   from   │
│          │     │ Registry │     │ Registry │
└──────────┘     └──────────┘     └──────────┘
                                        │
                                        ▼
Container Lifecycle:                 ┌──────────┐
Create → Start → Stop → Remove       │   Run    │
                                     │ (Create  │
┌──────────┐     ┌──────────┐       │Container)│
│  Start   │ ←   │  Create  │  ←    └──────────┘
└──────────┘     └──────────┘
     │                                    
     ▼                                    
┌──────────┐     ┌──────────┐       
│   Stop   │ →   │  Remove  │       
└──────────┘     └──────────┘       
     │
     │ (can restart)
     └────────────────┐
                      ▼
                 ┌──────────┐
                 │  Start   │
                 └──────────┘
```

### Real-World Example: Web Hosting Company

**Scenario:**
```
Hosting company manages 1000 websites
All websites use nginx web server

Traditional approach:
- Install nginx 1000 times
- Configure 1000 times
- Update 1000 times
- Disk usage: 142 GB (142 MB × 1000)

Docker approach:
- One nginx image: 142 MB
- 1000 containers: ~1 GB additional (writable layers)
- Total disk: ~1.2 GB
- Updates: Update image once, recreate containers
```

**How it works:**
```
1 nginx Image (142 MB)
     │
     ├──> Container 1 (website1.com)
     ├──> Container 2 (website2.com)
     ├──> Container 3 (website3.com)
     ├──> ...
     └──> Container 1000 (website1000.com)

Each container:
- Uses same image (shared)
- Has own writable layer (unique content)
- Own network (unique port/IP)
- Isolated processes
```

---

## Docker Architecture

Docker uses a **client-server architecture**.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Host                          │
│                                                         │
│  ┌──────────────┐                                       │
│  │  Docker CLI  │                                       │
│  │   (client)   │                                       │
│  └──────┬───────┘                                       │
│         │                                               │
│         │ REST API (HTTP/Unix Socket)                   │
│         │                                               │
│         ▼                                               │
│  ┌─────────────────────────────────────┐               │
│  │       Docker Daemon (dockerd)       │               │
│  │                                     │               │
│  │  ┌──────────────────────────────┐  │               │
│  │  │   Container Management       │  │               │
│  │  └──────────────────────────────┘  │               │
│  │  ┌──────────────────────────────┐  │               │
│  │  │   Image Management           │  │               │
│  │  └──────────────────────────────┘  │               │
│  │  ┌──────────────────────────────┐  │               │
│  │  │   Volume Management          │  │               │
│  │  └──────────────────────────────┘  │               │
│  │  ┌──────────────────────────────┐  │               │
│  │  │   Network Management         │  │               │
│  │  └──────────────────────────────┘  │               │
│  └─────────────────────────────────────┘               │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Containers                            │   │
│  │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐           │   │
│  │  │  C1 │  │  C2 │  │  C3 │  │  C4 │           │   │
│  │  └─────┘  └─────┘  └─────┘  └─────┘           │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │           Local Images                          │   │
│  │  ┌─────┐  ┌─────┐  ┌─────┐                     │   │
│  │  │ I1  │  │ I2  │  │ I3  │                     │   │
│  │  └─────┘  └─────┘  └─────┘                     │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
                         │
                         │ Network (HTTP/HTTPS)
                         ▼
              ┌─────────────────────┐
              │   Docker Registry   │
              │   (Docker Hub, ECR) │
              │                     │
              │  ┌───────────────┐  │
              │  │  nginx:latest │  │
              │  │  postgres:13  │  │
              │  │  redis:6      │  │
              │  └───────────────┘  │
              └─────────────────────┘
```

### Components Explained

#### 1. Docker CLI (Client)

The **command-line interface** you interact with.

```bash
# User types commands
docker run nginx
docker ps
docker build -t myapp .
docker pull postgres
```

**What it does:**
- Accepts user commands
- Translates to REST API calls
- Sends to Docker daemon
- Displays results to user

**Example flow:**
```
User types: docker run nginx

CLI converts to API call:
POST /containers/create
{
  "Image": "nginx"
}

Sends to: Docker Daemon
```

#### 2. Docker Daemon (dockerd)

The **background service** that does the actual work.

**Responsibilities:**
- Listen for API requests
- Manage containers (create, start, stop, remove)
- Manage images (build, pull, push, remove)
- Manage networks
- Manage volumes
- Talk to container runtime (containerd)

**Location:**
- Linux: Runs as system service
- macOS/Windows: Runs in VM (Docker Desktop)

**Communication:**
```
Default: Unix socket
/var/run/docker.sock

Can also use TCP:
tcp://localhost:2375 (insecure)
tcp://localhost:2376 (TLS)
```

#### 3. Docker Registry

**Storage and distribution** system for images.

**Types:**

**Public Registry:**
```
Docker Hub (hub.docker.com)
- Public images (nginx, postgres, redis)
- Private images (with paid account)
- Official images
- Community images
```

**Private Registry:**
```
Company registry (registry.company.com)
- Internal images
- Proprietary applications
- Better security
- Faster (local network)

Cloud registries:
- Amazon ECR
- Google GCR
- Azure ACR
```

### How Components Work Together

**Example: Running nginx**

```
Step-by-step flow:

1. User command:
   $ docker run nginx

2. Docker CLI:
   - Parses command
   - Creates REST API request
   - Sends to daemon

3. Docker Daemon:
   - Receives request
   - Checks if nginx image exists locally
   - Image not found locally

4. Docker Daemon → Registry:
   - Connects to Docker Hub
   - Pulls nginx:latest image
   - Downloads layers
   - Stores in local image cache

5. Docker Daemon:
   - Creates container from image
   - Allocates resources
   - Sets up networking
   - Starts container process

6. Container Running:
   - nginx process starts
   - Listens on port 80
   - Isolated environment

7. Docker CLI:
   - Receives response
   - Shows container ID to user
```

**Visual flow:**
```
┌─────────┐
│  User   │
└────┬────┘
     │ docker run nginx
     ▼
┌─────────────┐
│ Docker CLI  │
└─────┬───────┘
      │ POST /containers/create
      ▼
┌──────────────────┐
│  Docker Daemon   │
└─────┬────────────┘
      │ Image exists?
      ├─ No ─→ Pull from registry
      │         ↓
      │    ┌─────────────┐
      │    │ Docker Hub  │
      │    └─────────────┘
      │         ↓
      │    Download image
      ▼
┌──────────────────┐
│ Create Container │
│ Start Process    │
└──────────────────┘
      │
      ▼
┌──────────────────┐
│ nginx running    │
└──────────────────┘
```

### Docker on Different Operating Systems

#### Linux

```
┌────────────────────────┐
│   Linux Host OS        │
│                        │
│  ┌──────────────────┐  │
│  │  Docker Daemon   │  │
│  └──────────────────┘  │
│  ┌──────────────────┐  │
│  │  Containers      │  │
│  └──────────────────┘  │
│                        │
│  Linux Kernel          │
│  (Native)              │
└────────────────────────┘

Direct kernel access
Best performance
Native containers
```

#### macOS / Windows

```
┌─────────────────────────────┐
│   macOS / Windows           │
│                             │
│  ┌───────────────────────┐  │
│  │  Docker Desktop       │  │
│  │                       │  │
│  │  ┌─────────────────┐  │  │
│  │  │ Linux VM        │  │  │
│  │  │ (lightweight)   │  │  │
│  │  │                 │  │  │
│  │  │ Docker Daemon   │  │  │
│  │  │ Containers      │  │  │
│  │  └─────────────────┘  │  │
│  └───────────────────────┘  │
│                             │
│  macOS/Windows Kernel       │
└─────────────────────────────┘

Runs Linux VM
Slight overhead
Seamless to user
```

**Why VM needed:**
- Docker containers need Linux kernel
- macOS/Windows have different kernels
- Docker Desktop provides lightweight Linux VM
- Transparent to user (you just use docker commands)

### Real-World Architecture Example

**Netflix Streaming Service (Simplified):**

```
┌──────────────────────────────────────────────┐
│           Developer Laptop                   │
│                                              │
│  docker build -t api:v1 .                    │
│  docker push netflix-registry/api:v1        │
└──────────────────┬───────────────────────────┘
                   │
                   │ Push image
                   ▼
        ┌─────────────────────┐
        │  Private Registry   │
        │  (AWS ECR)          │
        └──────────┬──────────┘
                   │
                   │ Pull image
                   ▼
┌──────────────────────────────────────────────┐
│        Production Server 1                   │
│                                              │
│  ┌────────────────────────────────────┐     │
│  │  Docker Daemon                     │     │
│  └────────────────────────────────────┘     │
│                                              │
│  Containers:                                 │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐               │
│  │API │ │API │ │API │ │API │               │
│  │ v1 │ │ v1 │ │ v1 │ │ v1 │               │
│  └────┘ └────┘ └────┘ └────┘               │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│        Production Server 2                   │
│                                              │
│  Containers:                                 │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐               │
│  │API │ │API │ │API │ │API │               │
│  │ v1 │ │ v1 │ │ v1 │ │ v1 │               │
│  └────┘ └────┘ └────┘ └────┘               │
└──────────────────────────────────────────────┘

Benefits:
- Same image everywhere (consistency)
- Easy scaling (run more containers)
- Fast deployment (pull + run)
- Easy rollback (switch image versions)
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Docker is a platform that enables developers to __________, __________, and __________ applications in containers.
2. A Docker image is a __________ template, while a container is a __________ instance of an image.
3. The Docker __________ is the client that sends commands to the Docker __________.
4. Docker images are stored in a __________ such as Docker Hub.
5. The Docker daemon communicates with the CLI through a __________ API.
6. An image is __________, while a container has a __________ layer on top.

### True/False

1. ⬜ A Docker image is a running instance of a container
2. ⬜ You can create multiple containers from a single image
3. ⬜ The Docker CLI directly manages containers without the daemon
4. ⬜ Docker images are mutable and can be changed after creation
5. ⬜ On macOS, Docker runs containers directly on the host kernel
6. ⬜ Docker Hub is the only registry where images can be stored
7. ⬜ The Docker daemon is responsible for actually running containers

### Multiple Choice

1. What is the relationship between an image and a container?
   - A) Image is an instance of a container
   - B) Container is an instance of an image
   - C) They are the same thing
   - D) Containers create images

2. Which component actually runs containers?
   - A) Docker CLI
   - B) Docker Registry
   - C) Docker Daemon
   - D) Docker Hub

3. Where are Docker images stored locally?
   - A) In containers
   - B) In the Docker daemon's image cache
   - C) In Docker Hub only
   - D) In /var/log/docker

4. What analogy best describes Docker images and containers?
   - A) File and folder
   - B) Class and object
   - C) Server and client
   - D) Input and output

5. How does Docker CLI communicate with Docker daemon?
   - A) SSH
   - B) FTP
   - C) REST API over Unix socket or TCP
   - D) Direct function calls

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. build, ship, run (order may vary)
2. read-only (or immutable, static), running (or active)
3. CLI (or client), daemon (or server, dockerd)
4. registry (or repository)
5. REST (or HTTP)
6. read-only (or immutable), read-write (or writable)

**True/False:**
1. ❌ False - Container is a running instance of an image, not the other way around
2. ✅ True - One image can spawn many containers
3. ❌ False - CLI sends commands to daemon, daemon manages containers
4. ❌ False - Images are immutable (read-only)
5. ❌ False - macOS uses a Linux VM since containers need Linux kernel
6. ❌ False - Can use private registries (ECR, GCR, ACR, self-hosted)
7. ✅ True - Daemon does the actual work of managing containers

**Multiple Choice:**
1. **B** - Container is an instance of an image (like object from class)
2. **C** - Docker Daemon (dockerd) runs containers
3. **B** - In the Docker daemon's image cache (local storage)
4. **B** - Class and object (image = class, container = object)
5. **C** - REST API over Unix socket or TCP

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the difference between a Docker image and a Docker container with a real-world example

<details>
<summary><strong>View Answer</strong></summary>

**Core Concept:**

**Image** = Blueprint (template, recipe, class)  
**Container** = Instance (actual running thing, object)

**Technical Explanation:**

**Docker Image:**
- Read-only template
- Contains application code, libraries, dependencies
- Stored as layers
- Immutable (doesn't change)
- Can be shared and reused

**Docker Container:**
- Running instance of an image
- Has read-write layer on top of image
- Isolated process
- Temporary (can be stopped/removed)
- Each container is independent

**Real-World Example: Netflix Streaming API**

```
Image: netflix-api:v1.5.2
├── Base OS (Ubuntu)
├── Java Runtime
├── Application JAR
├── Configuration files
└── Dependencies

Size: 500 MB
State: Immutable
Location: AWS ECR Registry
```

**Creating Containers from this Image:**

```
Production Server 1:
├── Container: netflix-api-1 (from netflix-api:v1.5.2)
│   ├── Serving users in US East
│   ├── Handling 10,000 requests/sec
│   └── Writable layer: 50 MB (logs, temp files)
│
├── Container: netflix-api-2 (from netflix-api:v1.5.2)
│   ├── Serving users in US East
│   ├── Handling 10,000 requests/sec
│   └── Writable layer: 45 MB
│
└── Container: netflix-api-3 (from netflix-api:v1.5.2)
    ├── Serving users in US East
    ├── Handling 10,000 requests/sec
    └── Writable layer: 52 MB

Production Server 2:
├── Container: netflix-api-4 (from netflix-api:v1.5.2)
├── Container: netflix-api-5 (from netflix-api:v1.5.2)
└── Container: netflix-api-6 (from netflix-api:v1.5.2)
```

**Key Points:**

1. **One Image, Many Containers**
```
1 image (netflix-api:v1.5.2)
↓
6 running containers
All identical application
Different runtime state (different users, requests, logs)
```

2. **Immutability vs Mutability**
```
Image:
- Cannot modify
- Want to change? Build new image
- netflix-api:v1.5.2 is ALWAYS the same

Container:
- Can write files (logs, temp data)
- Can modify in memory
- Each container's changes are isolated
- Changes lost when container removed
```

3. **Storage Efficiency**
```
Without sharing (traditional):
6 servers × 500 MB = 3000 MB

With Docker:
1 image: 500 MB
6 containers' writable layers: ~300 MB
Total: 800 MB
```

**Analogy Summary:**

```
Recipe (Image):
- Written once
- Stored in cookbook
- Doesn't change
- Can be photocopied and shared

Prepared Meals (Containers):
- Made from recipe
- Each meal is separate
- Can modify individual meals (add salt to one)
- Meals are temporary (eaten)
- Make new meals from same recipe anytime
```

**Interview Response Pattern:**
1. State the key difference (template vs instance)
2. Give technical characteristics
3. Provide real-world example
4. Explain the benefits (storage efficiency, consistency)

</details>

### Question 2: Walk me through what happens when you run `docker run nginx`

<details>
<summary><strong>View Answer</strong></summary>

**Step-by-Step Technical Flow:**

**Command:**
```bash
docker run nginx
```

**Step 1: CLI Parses Command**
```
Docker CLI receives: docker run nginx

Parses into:
- Command: run
- Image: nginx (implicitly nginx:latest)
- Options: none (defaults)
- Arguments: none
```

**Step 2: CLI → Daemon Communication**
```
Docker CLI creates REST API request:

POST /v1.41/containers/create
{
  "Image": "nginx:latest",
  "AttachStdout": true,
  "AttachStderr": true
}

Sends to: Docker Daemon
Via: /var/run/docker.sock (Unix socket)
```

**Step 3: Daemon Checks Local Images**
```
Docker Daemon:
1. Receives API request
2. Checks local image cache
3. Looks for: nginx:latest

Local image list:
├── postgres:13 (exists)
├── redis:6 (exists)
└── nginx:latest (NOT FOUND)

Decision: Need to pull from registry
```

**Step 4: Pull Image from Registry**
```
Docker Daemon → Docker Hub:

1. Resolve registry: registry-1.docker.io
2. Find nginx repository
3. Get latest tag manifest
4. Download layers

nginx:latest consists of layers:
Layer 1: 7c3b88808835 (Base OS - 7 MB)
Layer 2: 92f2f8e8afd9 (nginx binary - 50 MB)
Layer 3: 8a2c7e4daeff (config files - 1 MB)
Layer 4: c8b9881f2c6a (content - 2 MB)
...

Download progress:
7c3b88808835: Pull complete
92f2f8e8afd9: Pull complete
8a2c7e4daeff: Pull complete
c8b9881f2c6a: Pull complete

Total download: ~142 MB
```

**Step 5: Store Image Locally**
```
Docker Daemon stores in:
/var/lib/docker/overlay2/

Image layers stored as:
├── /var/lib/docker/overlay2/layer1/
├── /var/lib/docker/overlay2/layer2/
└── /var/lib/docker/overlay2/layer3/

Image metadata stored in:
/var/lib/docker/image/overlay2/imagedb/
```

**Step 6: Create Container**
```
Docker Daemon creates container:

1. Generate container ID:
   a1b2c3d4e5f6...

2. Create writable layer:
   /var/lib/docker/overlay2/container-a1b2c3d4/

3. Set up networking:
   - Create network namespace
   - Assign IP: 172.17.0.2
   - Create virtual ethernet pair

4. Set up filesystem:
   - Mount image layers (read-only)
   - Mount writable layer (read-write)
   - Create /etc/hosts, /etc/resolv.conf

5. Set up isolation:
   - PID namespace (process isolation)
   - NET namespace (network isolation)
   - MNT namespace (filesystem isolation)
   - UTS namespace (hostname isolation)

6. Set resource limits (cgroups):
   - CPU: unlimited (default)
   - Memory: unlimited (default)
   - Disk I/O: unlimited (default)
```

**Step 7: Start Container**
```
Docker Daemon starts container:

1. Execute entry point:
   CMD: nginx -g 'daemon off;'

2. Process starts:
   PID 1 (inside container): nginx
   PID (on host): 12345

3. nginx process:
   - Reads configuration: /etc/nginx/nginx.conf
   - Binds to port: 80 (inside container)
   - Starts worker processes
   - Begins listening for connections

4. Container state: RUNNING
```

**Step 8: Output to User**
```
Docker CLI receives response:
{
  "Id": "a1b2c3d4e5f6...",
  "Status": "running"
}

Displays to terminal:
a1b2c3d4e5f6 (container ID)

If -d flag was used: returns to prompt
If no -d flag: attached to container output (shows nginx logs)
```

**Complete Timeline:**

```
t=0ms:    User types: docker run nginx
t=5ms:    CLI sends API request
t=10ms:   Daemon checks local images (not found)
t=15ms:   Daemon connects to Docker Hub
t=20ms:   Begin downloading image layers
t=5000ms: Download complete (142 MB)
t=5100ms: Image stored locally
t=5200ms: Container created
t=5300ms: Namespaces and cgroups configured
t=5400ms: nginx process starts
t=5500ms: Container running, nginx listening on port 80

Total time: ~5.5 seconds
```

**Subsequent Runs (Image Cached):**

```
docker run nginx (second time)

t=0ms:    User types: docker run nginx
t=5ms:    CLI sends API request
t=10ms:   Daemon checks local images (FOUND!)
t=50ms:   Container created
t=100ms:  nginx process starts
t=150ms:  Container running

Total time: ~150ms (much faster!)
```

**Visual Summary:**

```
User
  │
  │ docker run nginx
  ▼
┌─────────┐
│ CLI     │
└────┬────┘
     │ POST /containers/create
     ▼
┌──────────────┐
│ Daemon       │
└─────┬────────┘
      │ Image exists?
      ├─ No
      │
      ▼
┌──────────────┐
│ Docker Hub   │──→ Download layers
└──────────────┘
      │
      ▼
┌──────────────┐
│ Store Image  │
└─────┬────────┘
      │
      ▼
┌──────────────┐
│ Create       │──→ Setup namespaces
│ Container    │──→ Setup cgroups
└─────┬────────┘──→ Mount filesystem
      │
      ▼
┌──────────────┐
│ Start nginx  │──→ PID 1: nginx
└─────┬────────┘
      │
      ▼
┌──────────────┐
│ Running      │
└──────────────┘
```

**Interview Tips:**
- Start with high-level flow
- Mention key components (CLI, daemon, registry)
- Explain image pull only happens once
- Discuss namespaces and cgroups briefly
- End with container running state

</details>

### Question 3: How does Docker work differently on Linux vs macOS/Windows?

<details>
<summary><strong>View Answer</strong></summary>

**Core Difference:**

**Linux:** Containers run **natively** on Linux kernel  
**macOS/Windows:** Containers run in a **Linux VM**

**Technical Explanation:**

**Why This Difference Exists:**

```
Containers require Linux kernel features:
├── Namespaces (PID, NET, MNT, UTS, IPC, USER)
├── cgroups (resource limiting)
├── UnionFS (layered filesystem)
└── seccomp, AppArmor (security)

Linux: ✓ Has all these features
macOS: ✗ Different kernel (XNU/Darwin)
Windows: ✗ Different kernel (NT kernel)
```

**Docker on Linux:**

```
┌──────────────────────────────────┐
│    Linux Host (Ubuntu/CentOS)    │
│                                  │
│  ┌────────────────────────────┐  │
│  │  Docker Daemon             │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌────────────────────────────┐  │
│  │  Containers                │  │
│  │  ┌─────┐  ┌─────┐ ┌─────┐ │  │
│  │  │ C1  │  │ C2  │ │ C3  │ │  │
│  │  └─────┘  └─────┘ └─────┘ │  │
│  └────────────────────────────┘  │
│                                  │
│  ────────────────────────────────│
│      Linux Kernel                │
│  ────────────────────────────────│
│                                  │
│      Physical Hardware           │
└──────────────────────────────────┘

Characteristics:
✓ Direct kernel access
✓ Native performance
✓ No virtualization overhead
✓ Containers share host kernel
✓ Minimal resource usage
```

**Docker on macOS:**

```
┌────────────────────────────────────┐
│        macOS Host                  │
│                                    │
│  ┌──────────────────────────────┐  │
│  │   Docker Desktop for Mac     │  │
│  │                              │  │
│  │  ┌────────────────────────┐  │  │
│  │  │  Linux VM              │  │  │
│  │  │  (HyperKit)            │  │  │
│  │  │                        │  │  │
│  │  │  ┌──────────────────┐  │  │  │
│  │  │  │ Docker Daemon    │  │  │  │
│  │  │  └──────────────────┘  │  │  │
│  │  │                        │  │  │
│  │  │  ┌──────────────────┐  │  │  │
│  │  │  │  Containers      │  │  │  │
│  │  │  │  ┌───┐ ┌───┐    │  │  │  │
│  │  │  │  │C1 │ │C2 │    │  │  │  │
│  │  │  │  └───┘ └───┘    │  │  │  │
│  │  │  └──────────────────┘  │  │  │
│  │  │                        │  │  │
│  │  │  Linux Kernel          │  │  │
│  │  └────────────────────────┘  │  │
│  └──────────────────────────────┘  │
│                                    │
│  ─────────────────────────────────│
│        macOS Kernel (XNU)         │
│  ─────────────────────────────────│
│                                    │
│        Physical Hardware           │
└────────────────────────────────────┘

Characteristics:
⚠ Runs in lightweight Linux VM
⚠ Slight performance overhead
⚠ Transparent to user
✓ Still uses docker commands
✓ File sharing between macOS and VM
```

**Docker on Windows:**

```
┌────────────────────────────────────┐
│      Windows Host                  │
│                                    │
│  ┌──────────────────────────────┐  │
│  │  Docker Desktop for Windows  │  │
│  │                              │  │
│  │  ┌────────────────────────┐  │  │
│  │  │  Linux VM              │  │  │
│  │  │  (WSL 2)               │  │  │
│  │  │                        │  │  │
│  │  │  Docker Daemon         │  │  │
│  │  │  Containers            │  │  │
│  │  │  ┌───┐ ┌───┐          │  │  │
│  │  │  │C1 │ │C2 │          │  │  │
│  │  │  └───┘ └───┘          │  │  │
│  │  │                        │  │  │
│  │  │  Linux Kernel          │  │  │
│  │  └────────────────────────┘  │  │
│  └──────────────────────────────┘  │
│                                    │
│  ─────────────────────────────────│
│      Windows NT Kernel            │
│  ─────────────────────────────────│
│                                    │
│        Physical Hardware           │
└────────────────────────────────────┘

Note: Can also run Windows containers
(requires Windows Server base image)
```

**Performance Comparison:**

```
Benchmark: Running 100 containers

Linux Native:
├── Container startup: 100ms average
├── CPU overhead: <1%
├── Memory overhead: ~100MB (Docker daemon)
└── Disk I/O: Native speed

macOS (HyperKit VM):
├── Container startup: 150ms average
├── CPU overhead: 5-10% (VM)
├── Memory overhead: ~2GB (VM + Docker)
└── Disk I/O: Slower (VM filesystem layer)

Windows (WSL 2):
├── Container startup: 120ms average
├── CPU overhead: 2-5% (WSL 2 optimized)
├── Memory overhead: ~1.5GB (WSL 2 + Docker)
└── Disk I/O: Good (WSL 2 optimizations)
```

**File Sharing Differences:**

**Linux:**
```bash
# Direct mount (native)
docker run -v /home/user/data:/data nginx

Host: /home/user/data
Container: /data
Performance: Native (same filesystem)
```

**macOS:**
```bash
# Requires VM file sharing
docker run -v /Users/user/data:/data nginx

Host macOS: /Users/user/data
    ↓ (file sharing layer)
Linux VM: /Users/user/data (mounted)
    ↓
Container: /data

Performance: Slower (VM file sharing overhead)
Impact: 2-5x slower for file operations
```

**Networking Differences:**

**Linux:**
```bash
docker run -p 8080:80 nginx

Host network: Direct access
Container port 80 → Host port 8080
Access: http://localhost:8080 ✓
```

**macOS:**
```bash
docker run -p 8080:80 nginx

VM network: Bridged to macOS
Container port 80 → VM port 8080 → macOS port 8080
Access: http://localhost:8080 ✓ (transparent)

Note: Container cannot access macOS localhost
Use: host.docker.internal (special DNS name)
```

**Real-World Implications:**

**Development (macOS):**
```
Developer running:
- Database containers (PostgreSQL, Redis)
- Application container
- Development with hot reload

Impact:
- File watching slower (VM layer)
- I/O intensive operations slower
- CPU/Memory: Acceptable
- Generally fine for development
```

**Production (Linux):**
```
Production server running:
- 100+ containers
- High throughput
- Low latency requirements

Why Linux:
- Native performance critical
- No VM overhead
- Better resource utilization
- Lower costs at scale
```

**Interview Summary Points:**

1. **Fundamental Difference:**
   - Linux: Native containers
   - macOS/Windows: Containers in Linux VM

2. **Why:**
   - Containers need Linux kernel features
   - macOS/Windows have different kernels

3. **Performance:**
   - Linux: Best
   - macOS/Windows: Slight overhead

4. **Usage:**
   - Development: macOS/Windows acceptable
   - Production: Always Linux

5. **User Experience:**
   - Same docker commands everywhere
   - Transparent to most users
   - File sharing difference on macOS/Windows

</details>

</details>

---

[← Previous: 1.2 Why Containers?](02-why-containers.md) | [Next: 2.1 Docker Engine Components →](../02-docker-engine/01-engine-components.md)