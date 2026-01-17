# 2.1 Docker Engine Components

Understanding the three main components of Docker Engine: CLI, Daemon, and REST API.

---

## Docker Engine Overview

**Docker Engine** is the core component that enables you to build and run containers. It consists of three main parts working together:

```
┌─────────────────────────────────────────┐
│         Docker Engine                   │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  1. Docker CLI (Client)           │  │
│  │     - User interface              │  │
│  │     - Command processor           │  │
│  └─────────────┬─────────────────────┘  │
│                │                        │
│                │ REST API               │
│                │                        │
│  ┌─────────────▼─────────────────────┐  │
│  │  2. Docker REST API               │  │
│  │     - Communication layer         │  │
│  │     - HTTP/Unix socket            │  │
│  └─────────────┬─────────────────────┘  │
│                │                        │
│                ▼                        │
│  ┌───────────────────────────────────┐  │
│  │  3. Docker Daemon (dockerd)       │  │
│  │     - Container runtime           │  │
│  │     - Image management            │  │
│  │     - Network management          │  │
│  │     - Volume management           │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

---

## 1. Docker CLI (Client)

The **Docker CLI** is the command-line tool that users interact with.

### What It Does

```
User types command → CLI processes → Sends to daemon

Example:
$ docker run nginx
         ↓
    Parse command
         ↓
    Create API request
         ↓
    Send to daemon
```

### Common CLI Commands

```bash
# Container operations
docker run nginx          # Create and start container
docker ps                 # List running containers
docker stop container_id  # Stop container
docker rm container_id    # Remove container

# Image operations
docker pull nginx         # Download image
docker build -t app .     # Build image
docker images             # List images
docker rmi image_id       # Remove image

# Information
docker logs container_id  # View container logs
docker inspect container  # Detailed info
docker stats              # Resource usage
```

### How CLI Works

**Example: `docker run nginx`**

```
Step 1: Parse Command
─────────────────────
Input: "docker run nginx"

Parsed as:
├── Command: run
├── Image: nginx
├── Options: (none, using defaults)
└── Arguments: (none)

Step 2: Validate
─────────────────
✓ Valid command
✓ Valid image name
✓ No conflicting options

Step 3: Create API Request
──────────────────────────
POST /v1.41/containers/create
Content-Type: application/json

{
  "Image": "nginx:latest",
  "HostConfig": {
    "NetworkMode": "default"
  }
}

Step 4: Send to Daemon
──────────────────────
Via: /var/run/docker.sock (Unix socket)
Or: tcp://localhost:2375 (if remote)

Step 5: Receive Response
────────────────────────
{
  "Id": "a1b2c3d4e5f6...",
  "Warnings": []
}

Step 6: Display to User
───────────────────────
a1b2c3d4e5f6
```

### CLI Configuration

```bash
# Default socket location
/var/run/docker.sock

# Connect to remote daemon
docker -H tcp://remote-host:2375 ps

# Set default host
export DOCKER_HOST=tcp://remote-host:2375
docker ps  # Now connects to remote

# Use TLS
docker -H tcp://remote-host:2376 --tlsverify ps
```

### CLI Features

**1. Command Aliases**
```bash
docker ps      = docker container ls
docker images  = docker image ls
docker rmi     = docker image rm
docker rm      = docker container rm
```

**2. Output Formatting**
```bash
# Default table format
docker ps

# Custom format
docker ps --format "table {{.Names}}\t{{.Status}}"

# JSON output
docker inspect container_id --format='{{json .}}'

# Get specific field
docker inspect --format='{{.State.Running}}' container_id
```

**3. Filters**
```bash
# Filter by status
docker ps -a --filter "status=exited"

# Filter by name
docker ps --filter "name=web"

# Filter by label
docker ps --filter "label=env=production"
```

---

## 2. Docker REST API

The **REST API** is the communication protocol between CLI and daemon.

### API Structure

```
Protocol: HTTP/1.1
Transport: Unix socket or TCP
Format: JSON

Endpoint structure:
http://localhost/v1.41/{resource}/{action}

Examples:
/v1.41/containers/create
/v1.41/containers/{id}/start
/v1.41/images/create
/v1.41/networks/create
```

### API Version

```bash
# Check API version
docker version

Output:
Client: Docker Engine - Community
 Version:           24.0.5
 API version:       1.43

Server: Docker Engine - Community
 Engine:
  Version:          24.0.5
  API version:      1.43 (minimum version 1.12)
```

### Communication Methods

**1. Unix Socket (Default on Linux/macOS)**
```
Location: /var/run/docker.sock
Type: Unix domain socket
Security: File permissions
Speed: Fast (local)

Request flow:
CLI → /var/run/docker.sock → Daemon
```

**2. TCP Socket (Remote connections)**
```
Default insecure: tcp://localhost:2375
Default TLS: tcp://localhost:2376

Request flow:
CLI → Network → TCP:2375 → Daemon
```

### API Examples

**Create Container:**
```bash
# Using CLI
docker run -d --name web nginx

# Equivalent API call
curl --unix-socket /var/run/docker.sock \
  -H "Content-Type: application/json" \
  -d '{"Image":"nginx","name":"web"}' \
  -X POST http://localhost/v1.41/containers/create

# Response
{
  "Id": "a1b2c3d4e5f6...",
  "Warnings": []
}
```

**Start Container:**
```bash
# Using CLI
docker start web

# Equivalent API call
curl --unix-socket /var/run/docker.sock \
  -X POST http://localhost/v1.41/containers/web/start
```

**List Containers:**
```bash
# Using CLI
docker ps -a

# Equivalent API call
curl --unix-socket /var/run/docker.sock \
  http://localhost/v1.41/containers/json?all=true

# Response
[
  {
    "Id": "a1b2c3d4e5f6...",
    "Names": ["/web"],
    "Image": "nginx",
    "State": "running",
    "Status": "Up 5 minutes"
  }
]
```

### Real-World API Usage

**Monitoring Tool Example:**
```python
import docker

# Connect to Docker daemon
client = docker.from_env()  # Uses /var/run/docker.sock

# List all containers
containers = client.containers.list(all=True)

for container in containers:
    stats = container.stats(stream=False)
    
    cpu_usage = stats['cpu_stats']['cpu_usage']['total_usage']
    memory_usage = stats['memory_stats']['usage']
    
    print(f"{container.name}:")
    print(f"  CPU: {cpu_usage}")
    print(f"  Memory: {memory_usage / 1024 / 1024:.2f} MB")
```

**CI/CD Pipeline Example:**
```python
# Build and push in automated pipeline
import docker

client = docker.from_env()

# Build image
image, build_logs = client.images.build(
    path=".",
    tag="myapp:v1.2.3"
)

# Push to registry
for line in client.images.push("myapp:v1.2.3", stream=True):
    print(line)
```

---

## 3. Docker Daemon (dockerd)

The **Docker daemon** is the background service that does all the actual work.

### What It Does

```
Docker Daemon Responsibilities:

1. Container Management
   ├── Create containers
   ├── Start/stop containers
   ├── Remove containers
   └── Monitor container health

2. Image Management
   ├── Pull images from registry
   ├── Build images from Dockerfile
   ├── Push images to registry
   ├── Store images locally
   └── Manage image layers

3. Network Management
   ├── Create networks
   ├── Connect containers to networks
   ├── DNS resolution
   └── Port mapping

4. Volume Management
   ├── Create volumes
   ├── Mount volumes to containers
   ├── Manage volume lifecycle
   └── Volume drivers

5. System Management
   ├── Resource allocation
   ├── Event logging
   ├── Plugin management
   └── System cleanup
```

### Daemon Process

**Starting the Daemon:**
```bash
# Linux (systemd)
sudo systemctl start docker
sudo systemctl enable docker  # Start on boot

# Check status
sudo systemctl status docker

# Manual start (debugging)
sudo dockerd

# With custom configuration
sudo dockerd --config-file /etc/docker/daemon.json
```

**Daemon Configuration:**
```json
// /etc/docker/daemon.json
{
  "data-root": "/var/lib/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-address-pools": [
    {
      "base": "172.17.0.0/16",
      "size": 24
    }
  ],
  "insecure-registries": ["registry.company.local:5000"],
  "registry-mirrors": ["https://mirror.gcr.io"]
}
```

### Daemon Architecture

```
┌──────────────────────────────────────────┐
│         Docker Daemon (dockerd)          │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │      API Server                    │  │
│  │  (Listens on Unix socket/TCP)      │  │
│  └─────────────┬──────────────────────┘  │
│                │                         │
│                ▼                         │
│  ┌────────────────────────────────────┐  │
│  │      Container Manager             │  │
│  │  - Lifecycle management            │  │
│  │  - State tracking                  │  │
│  └─────────────┬──────────────────────┘  │
│                │                         │
│                ▼                         │
│  ┌────────────────────────────────────┐  │
│  │      containerd                    │  │
│  │  - Container runtime               │  │
│  │  - Image management                │  │
│  └─────────────┬──────────────────────┘  │
│                │                         │
│                ▼                         │
│  ┌────────────────────────────────────┐  │
│  │      runc                          │  │
│  │  - OCI runtime                     │  │
│  │  - Actually runs containers        │  │
│  └────────────────────────────────────┘  │
│                                          │
└──────────────────────────────────────────┘
                 │
                 ▼
         Linux Kernel
     (namespaces, cgroups)
```

### How Daemon Processes Requests

**Example: Creating and Starting a Container**

```
Request: docker run nginx

1. API Server receives request
   ├── Validates request
   ├── Checks permissions
   └── Routes to container manager

2. Container Manager
   ├── Checks if image exists locally
   ├── No → Trigger image pull
   │   ├── Connect to registry
   │   ├── Download layers
   │   └── Store in local cache
   └── Yes → Proceed

3. Container Creation
   ├── Generate container ID
   ├── Create writable layer
   ├── Prepare configuration
   └── Pass to containerd

4. containerd
   ├── Prepare container bundle
   ├── Set up namespaces
   ├── Configure cgroups
   └── Call runc

5. runc
   ├── Create container process
   ├── Apply namespace isolation
   ├── Apply cgroup limits
   ├── Execute entry point
   └── Return to containerd

6. Container Manager
   ├── Track container state
   ├── Monitor health
   ├── Log events
   └── Return success to API

7. API Server
   └── Send response to CLI

8. CLI
   └── Display container ID to user
```

### Daemon Storage

```
Default location: /var/lib/docker/

Directory structure:
/var/lib/docker/
├── containers/        # Container data
│   └── a1b2c3d4.../
│       ├── config.v2.json
│       ├── hostconfig.json
│       └── hostname
│
├── image/            # Image metadata
│   └── overlay2/
│       ├── imagedb/
│       └── layerdb/
│
├── overlay2/         # Image and container layers
│   ├── l/            # Symbolic links
│   └── a1b2c3d4.../  # Actual layer data
│
├── volumes/          # Named volumes
│   └── my-volume/
│       └── _data/
│
├── network/          # Network configuration
│   └── files/
│
└── buildkit/         # Build cache
    └── cache/
```

### Daemon Logs

```bash
# View daemon logs (systemd)
sudo journalctl -u docker.service

# Follow daemon logs
sudo journalctl -u docker.service -f

# Last 100 lines
sudo journalctl -u docker.service -n 100

# Logs since time
sudo journalctl -u docker.service --since "1 hour ago"
```

---

## How Components Work Together

### Complete Flow Example

**Scenario: Developer runs `docker run -p 8080:80 nginx`**

```
┌─────────────┐
│  Developer  │
│   Terminal  │
└──────┬──────┘
       │
       │ 1. Types: docker run -p 8080:80 nginx
       │
       ▼
┌─────────────────────────────────────────┐
│  Docker CLI                             │
│                                         │
│  Parses command:                        │
│  ├── Command: run                       │
│  ├── Port: 8080:80                      │
│  └── Image: nginx                       │
│                                         │
│  Creates API request:                   │
│  POST /containers/create                │
│  {                                      │
│    "Image": "nginx",                    │
│    "HostConfig": {                      │
│      "PortBindings": {                  │
│        "80/tcp": [{"HostPort": "8080"}] │
│      }                                  │
│    }                                    │
│  }                                      │
└──────┬──────────────────────────────────┘
       │
       │ 2. Sends via Unix socket
       │    /var/run/docker.sock
       │
       ▼
┌─────────────────────────────────────────┐
│  Docker REST API                        │
│                                         │
│  ├── Receives HTTP POST request         │
│  ├── Validates JSON payload             │
│  ├── Checks API version compatibility   │
│  └── Routes to daemon handler           │
└──────┬──────────────────────────────────┘
       │
       │ 3. Forwards to daemon
       │
       ▼
┌─────────────────────────────────────────┐
│  Docker Daemon                          │
│                                         │
│  Step 1: Image Check                    │
│  ├── Check local image store            │
│  ├── nginx not found                    │
│  └── Initiate pull from Docker Hub      │
│                                         │
│  Step 2: Image Pull                     │
│  ├── Connect to registry.docker.io      │
│  ├── Download layers                    │
│  │   ├── Layer 1: 7MB                   │
│  │   ├── Layer 2: 50MB                  │
│  │   └── Layer 3: 2MB                   │
│  └── Store in /var/lib/docker/          │
│                                         │
│  Step 3: Container Creation             │
│  ├── Generate ID: a1b2c3d4e5f6          │
│  ├── Create writable layer              │
│  ├── Allocate network                   │
│  │   ├── IP: 172.17.0.2                 │
│  │   └── Map port 80 → 8080             │
│  └── Prepare configuration              │
│                                         │
│  Step 4: Start Container                │
│  ├── Setup namespaces (PID, NET, MNT)   │
│  ├── Apply cgroups                      │
│  ├── Mount filesystem                   │
│  ├── Execute: nginx -g 'daemon off;'    │
│  └── Container state: RUNNING           │
│                                         │
│  Step 5: Return Success                 │
│  └── Container ID: a1b2c3d4e5f6         │
└──────┬──────────────────────────────────┘
       │
       │ 4. Response
       │
       ▼
┌─────────────────────────────────────────┐
│  Docker REST API                        │
│                                         │
│  Returns HTTP 201 Created:              │
│  {                                      │
│    "Id": "a1b2c3d4e5f6...",             │
│    "Warnings": null                     │
│  }                                      │
└──────┬──────────────────────────────────┘
       │
       │ 5. Format response
       │
       ▼
┌─────────────────────────────────────────┐
│  Docker CLI                             │
│                                         │
│  Displays to user:                      │
│  a1b2c3d4e5f6                           │
└──────┬──────────────────────────────────┘
       │
       ▼
┌─────────────┐
│  Developer  │  
│  sees ID    │
└─────────────┘
```

---

## Remote Docker Daemon

### Connecting to Remote Daemon

**Setup on Remote Server:**
```bash
# Edit daemon config
sudo vim /etc/docker/daemon.json

{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"]
}

# Restart daemon
sudo systemctl restart docker
```

⚠️ **Security Warning:** Port 2375 is unencrypted! Use only in trusted networks.

**From Client:**
```bash
# Connect to remote daemon
docker -H tcp://192.168.1.100:2375 ps

# Set as default
export DOCKER_HOST=tcp://192.168.1.100:2375
docker ps  # Now uses remote daemon

# Build on remote
docker -H tcp://192.168.1.100:2375 build -t myapp .

# Run on remote
docker -H tcp://192.168.1.100:2375 run -d myapp
```

### Secure Remote Access (TLS)

**Generate Certificates:**
```bash
# On server
openssl genrsa -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
openssl genrsa -out server-key.pem 4096
# ... (additional certificate steps)
```

**Configure Daemon:**
```json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
  "tls": true,
  "tlsverify": true,
  "tlscacert": "/etc/docker/ca.pem",
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem"
}
```

**From Client:**
```bash
docker --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=cert.pem \
  --tlskey=key.pem \
  -H=tcp://192.168.1.100:2376 ps
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The Docker __________ is the command-line interface that users interact with.
2. The Docker daemon communicates with the CLI through a __________ API.
3. By default, Docker daemon listens on a __________ socket at /var/run/docker.sock.
4. The Docker daemon is responsible for __________ containers, managing __________, and handling __________.
5. When connecting to a remote Docker daemon, port __________ is used for insecure connections and port __________ for TLS.
6. The Docker CLI converts user commands into __________ API requests.

### True/False

1. ⬜ The Docker CLI directly manages containers without the daemon
2. ⬜ The Docker daemon can run on a different machine than the CLI
3. ⬜ REST API communication uses JSON format
4. ⬜ Multiple CLI clients can connect to the same daemon
5. ⬜ The daemon stores container data in /var/lib/docker by default
6. ⬜ You can only use Unix sockets for CLI-daemon communication
7. ⬜ The Docker daemon handles image builds, pulls, and pushes

### Multiple Choice

1. What does the Docker CLI do with a user command?
   - A) Directly creates containers
   - B) Converts it to an API request
   - C) Stores it in a database
   - D) Compiles it to binary

2. Where does the Docker daemon listen by default on Linux?
   - A) tcp://localhost:2375
   - B) tcp://localhost:2376
   - C) /var/run/docker.sock
   - D) /tmp/docker.sock

3. Which component actually creates and runs containers?
   - A) Docker CLI
   - B) Docker REST API
   - C) Docker Daemon
   - D) Docker Hub

4. What protocol does the Docker REST API use?
   - A) FTP
   - B) SSH
   - C) HTTP
   - D) WebSocket

5. How can you connect to a remote Docker daemon?
   - A) Only through VPN
   - B) Using -H flag with host address
   - C) Not possible
   - D) Only through SSH tunnel

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. CLI (or client)
2. REST (or HTTP)
3. Unix (or unix domain)
4. running (or creating), images, networks (or volumes) - order may vary
5. 2375, 2376
6. HTTP (or REST, API)

**True/False:**
1. ❌ False - CLI sends commands to daemon, daemon manages containers
2. ✅ True - Can connect to remote daemon over network
3. ✅ True - REST API uses JSON for requests and responses
4. ✅ True - Multiple clients can connect to same daemon
5. ✅ True - Default data directory is /var/lib/docker
6. ❌ False - Can also use TCP sockets for remote connections
7. ✅ True - Daemon handles all image operations

**Multiple Choice:**
1. **B** - Converts it to an API request
2. **C** - /var/run/docker.sock (Unix socket)
3. **C** - Docker Daemon
4. **C** - HTTP (REST API over HTTP)
5. **B** - Using -H flag with host address

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the role of each Docker Engine component and how they interact

<details>
<summary><strong>View Answer</strong></summary>

**Three Components:**

**1. Docker CLI (Client)**
- **Role:** User interface
- **Function:** Accepts commands, converts to API calls
- **Example:** `docker run nginx`

**2. Docker REST API**
- **Role:** Communication layer
- **Function:** Protocol for CLI-Daemon communication
- **Transport:** Unix socket or TCP

**3. Docker Daemon (dockerd)**
- **Role:** Core engine
- **Function:** Executes operations, manages resources
- **Responsibilities:** Containers, images, networks, volumes

**Interaction Flow:**

```
Example: docker run nginx

Step 1: User → CLI
─────────────────────
User types: docker run nginx
CLI receives command

Step 2: CLI → API
─────────────────────
CLI creates API request:
POST /v1.41/containers/create
{
  "Image": "nginx"
}

Sends via Unix socket:
/var/run/docker.sock

Step 3: API → Daemon
─────────────────────
API forwards request to daemon
Daemon receives JSON payload

Step 4: Daemon Processing
─────────────────────
Daemon:
1. Checks if nginx image exists
2. Pulls image if needed
3. Creates container
4. Starts nginx process
5. Returns container ID

Step 5: Daemon → API → CLI
─────────────────────
Response flows back:
Daemon → API → CLI

Step 6: CLI → User
─────────────────────
CLI displays:
a1b2c3d4e5f6
```

**Key Points:**

1. **Separation of Concerns**
```
CLI: User interface (presentation)
API: Communication (transport)
Daemon: Business logic (execution)
```

2. **Flexibility**
```
Can replace CLI with:
- Custom scripts
- Web UI (Portainer)
- API clients (Python, Go)

Daemon remains unchanged
```

3. **Remote Operation**
```
CLI (Laptop) → Network → API → Daemon (Server)
Same architecture works locally and remotely
```

**Real-World Example:**

```
DevOps team:
- Developers: Use CLI locally
- CI/CD pipeline: Uses API directly (Python SDK)
- Monitoring tool: Connects via API
- All interact with same daemon
```

</details>

### Question 2: How would you debug an issue where Docker CLI commands are failing?

<details>
<summary><strong>View Answer</strong></summary>

**Systematic Debugging Approach:**

**Step 1: Check Daemon Status**
```bash
# Is daemon running?
sudo systemctl status docker

# If not running
sudo systemctl start docker

# Check daemon logs
sudo journalctl -u docker.service -n 50

# Common error: Daemon not started
Error: Cannot connect to Docker daemon
Solution: Start daemon
```

**Step 2: Check Socket Connection**
```bash
# Verify socket exists
ls -la /var/run/docker.sock

# Expected output:
srw-rw---- 1 root docker 0 Jan 17 10:00 /var/run/docker.sock

# Check permissions
# Current user must be in docker group
groups

# If not in docker group:
sudo usermod -aG docker $USER
# Log out and back in

# Test socket
curl --unix-socket /var/run/docker.sock http://localhost/version

# Should return JSON with version info
```

**Step 3: Check CLI-Daemon Communication**
```bash
# Set debug mode
export DOCKER_DEBUG=1
docker ps

# Or use verbose flag
docker --debug ps

# Check what API endpoint CLI is using
docker context ls

# Should show:
NAME      DESCRIPTION     DOCKER ENDPOINT
default * Current         unix:///var/run/docker.sock
```

**Step 4: Verify Daemon Configuration**
```bash
# Check daemon config
cat /etc/docker/daemon.json

# Common issues:
# - Invalid JSON syntax
# - Wrong socket configuration
# - Conflicting settings

# Validate JSON
jq . /etc/docker/daemon.json

# If error, fix JSON syntax
```

**Step 5: Check Network Issues (Remote Daemon)**
```bash
# If using remote daemon
docker -H tcp://remote-host:2375 version

# Test connectivity
telnet remote-host 2375

# Check firewall
sudo iptables -L -n | grep 2375

# Common issue: Port blocked
Solution: Open port in firewall
```

**Step 6: Review Daemon Logs in Detail**
```bash
# Full daemon logs
sudo journalctl -u docker.service --no-pager

# Filter for errors
sudo journalctl -u docker.service | grep -i error

# Check disk space (common issue)
df -h /var/lib/docker

# If disk full:
docker system prune -a --volumes
```

**Common Issues and Solutions:**

**Issue 1: Permission Denied**
```
Error: permission denied while trying to connect to the 
       Docker daemon socket

Diagnosis:
$ ls -la /var/run/docker.sock
srw-rw---- 1 root docker 0 Jan 17 10:00 /var/run/docker.sock
$ groups
user : user

Solution:
sudo usermod -aG docker $USER
# Log out and log back in
```

**Issue 2: Daemon Not Running**
```
Error: Cannot connect to the Docker daemon at 
       unix:///var/run/docker.sock. Is the docker 
       daemon running?

Diagnosis:
$ sudo systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded
   Active: inactive (dead)

Solution:
sudo systemctl start docker
sudo systemctl enable docker
```

**Issue 3: Socket File Missing**
```
Error: dial unix /var/run/docker.sock: connect: 
       no such file or directory

Diagnosis:
$ ls /var/run/docker.sock
ls: cannot access '/var/run/docker.sock': No such file

Solution:
# Daemon crashed or misconfigured
sudo systemctl restart docker

# Check daemon config
cat /etc/docker/daemon.json
# Ensure "hosts" includes unix:///var/run/docker.sock
```

**Issue 4: API Version Mismatch**
```
Error: client version 1.43 is too new. Maximum supported 
       API version is 1.41

Diagnosis:
$ docker version
Client: API version: 1.43
Server: API version: 1.41

Solution:
# Downgrade client or upgrade daemon
# Or set specific API version
export DOCKER_API_VERSION=1.41
docker ps
```

**Issue 5: Daemon Out of Resources**
```
Error: Error response from daemon: mkdir /var/lib/docker: 
       no space left on device

Diagnosis:
$ df -h /var/lib/docker
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        20G   20G     0 100% /

Solution:
# Clean up Docker resources
docker system prune -a --volumes

# Or increase disk space
```

**Debugging Checklist:**

```
□ Is daemon running?
  → sudo systemctl status docker

□ Can user access socket?
  → groups | grep docker

□ Does socket file exist?
  → ls -la /var/run/docker.sock

□ Is daemon listening?
  → curl --unix-socket /var/run/docker.sock http://localhost/version

□ Any daemon errors?
  → sudo journalctl -u docker.service -n 50

□ Sufficient disk space?
  → df -h /var/lib/docker

□ Network connectivity (if remote)?
  → telnet remote-host 2375

□ API version compatible?
  → docker version
```

</details>

### Question 3: Why does Docker use a client-server architecture instead of direct execution?

<details>
<summary><strong>View Answer</strong></summary>

**Key Reasons:**

**1. Remote Management**

```
Without client-server:
├── Can only manage local containers
└── Must SSH to each server

With client-server:
├── Manage containers on any server
├── One CLI controls multiple daemons
└── No SSH required

Example:
# From laptop, manage production
docker -H tcp://prod-server:2376 ps
docker -H tcp://staging-server:2376 ps
docker -H tcp://dev-server:2376 ps
```

**2. Multiple Clients**

```
Different interfaces to same daemon:

CLI (developers)
    ↓
Web UI (Portainer) → Docker Daemon → Containers
    ↓
API (monitoring tools)
    ↓
CI/CD pipeline
```

**3. Privilege Separation**

```
Security benefit:

Docker Daemon:
├── Runs as root
├── Needs kernel access
└── High privileges

Docker CLI:
├── Runs as regular user
├── No special privileges needed
└── Communicates via socket

User doesn't need root access
Only needs permission to access socket
```

**4. Scalability**

```
One daemon can handle:
├── Multiple CLI sessions
├── API clients
├── Monitoring tools
├── Orchestration systems

All simultaneously without conflicts
```

**5. Platform Flexibility**

```
CLI on macOS/Windows:
└── Connects to daemon in Linux VM

Same CLI works on:
├── Linux (native)
├── macOS (via VM)
├── Windows (via WSL2)
└── Remote servers
```

**Real-World Example: Netflix**

```
Netflix Infrastructure:

Developer Laptops (macOS):
└── Docker CLI

    ↓ (network)

Production Servers (Linux):
└── Docker Daemon
    ├── Running 1000+ containers
    ├── Handling API requests from:
    │   ├── Developers (CLI)
    │   ├── Kubernetes (API)
    │   ├── Monitoring (API)
    │   └── CI/CD (API)
    └── All coordinated through daemon

Benefits:
- Developers manage prod without SSH
- Kubernetes orchestrates containers
- Monitoring tools track resources
- CI/CD deploys automatically
- All through same daemon API
```

**Alternative Designs (Why Not Used):**

```
Direct Execution (like podman rootless):
Pros:
- Simpler architecture
- No daemon required
- Better security (no root daemon)

Cons:
- Harder remote management
- More complex for multiple clients
- Platform limitations

Docker chose client-server for:
- Remote management
- Multiple client support
- Established patterns
```

**Technical Benefits:**

```
1. API Versioning
   - CLI can upgrade independently
   - Daemon can support multiple API versions
   - Backward compatibility

2. Load Distribution
   - Daemon handles heavy operations
   - CLI stays lightweight
   - Clear responsibility separation

3. State Management
   - Daemon tracks all container states
   - Survives CLI disconnection
   - Centralized control

4. Security Boundaries
   - Socket permissions control access
   - TLS for remote connections
   - User doesn't need root
```

</details>

</details>

---

[← Previous: 1.3 Docker Basics](../01-fundamentals/03-docker-basics.md) | [Next: 2.2 Linux Kernel Features →](02-kernel-features.md)