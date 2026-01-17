# 2.2 Linux Kernel Features

Understanding the Linux kernel features that enable container isolation: namespaces and cgroups.

---

## Overview

Docker containers rely on two critical Linux kernel features:

1. **Namespaces** → Isolation (what you can see)
2. **cgroups** → Resource limits (what you can use)

```
Container Isolation = Namespaces + cgroups

┌─────────────────────────────────────┐
│         Linux Kernel                │
│                                     │
│  ┌────────────┐  ┌────────────┐     │
│  │ Namespaces │  │  cgroups   │     │
│  │            │  │            │     │
│  │ Isolation  │  │  Limits    │     │
│  └────────────┘  └────────────┘     │
│         ↓              ↓            │
│         └──────┬───────┘            │
│                ▼                    │
│         Container Process           │
└─────────────────────────────────────┘
```

---

## Namespaces

**Namespaces** provide **isolation** by giving processes their own view of system resources.

### Concept

> Each namespace creates an isolated "view" of a specific resource.

Think of it like apartments in a building:
- Each apartment (namespace) has its own view
- Residents can't see into other apartments
- Building infrastructure is shared (kernel)

### Types of Namespaces

Linux provides 7 types of namespaces:

| Namespace | Isolates | Example |
|-----------|----------|---------|
| **PID** | Process IDs | Process 1 in container, actually process 5432 on host |
| **NET** | Network stack | Own IP, ports, routing tables |
| **MNT** | Filesystem mounts | Own filesystem view |
| **UTS** | Hostname | Container has own hostname |
| **IPC** | Inter-process communication | Message queues, shared memory |
| **USER** | User and group IDs | UID 0 in container ≠ UID 0 on host |
| **Cgroup** | Cgroup root directory | Own cgroup hierarchy view |

---

### 1. PID Namespace (Process Isolation)

**What it does:** Each container thinks it's the only system running.

**Without PID Namespace:**
```bash
# On host
$ ps aux
USER   PID  COMMAND
root     1  /sbin/init
root   542  nginx
root   543  postgres
root   544  redis

# Every process sees every other process
```

**With PID Namespace:**
```bash
# Inside container
$ ps aux
USER   PID  COMMAND
root     1  nginx

# Container sees only its own processes
# Thinks nginx is PID 1

# On host (outside container)
$ ps aux | grep nginx
root  5432  nginx  # Actually PID 5432 on host

# Host sees real PID
```

**Real-World Example:**

```
Web Server Container:
├── Inside container view:
│   PID 1: nginx (master)
│   PID 2: nginx (worker 1)
│   PID 3: nginx (worker 2)
│
└── Host view:
    PID 5432: nginx (master) - Container 1
    PID 5433: nginx (worker 1) - Container 1
    PID 5434: nginx (worker 2) - Container 1

Database Container:
├── Inside container view:
│   PID 1: postgres
│
└── Host view:
    PID 6789: postgres - Container 2

Isolation:
- nginx can't see postgres
- postgres can't see nginx
- Each thinks it's PID 1 (the init process)
```

**Why This Matters:**

```
Security:
- Container processes can't kill host processes
- Can't see other containers' processes
- Reduced attack surface

Process Management:
- Clean PID space per container
- PID 1 gets special treatment (signal handling)
- Container lifecycle tied to PID 1
```

---

### 2. NET Namespace (Network Isolation)

**What it does:** Each container has its own network stack.

**Network Components Isolated:**
```
Each container gets:
├── Network interfaces (eth0, lo)
├── IP addresses (172.17.0.2)
├── Routing tables
├── Firewall rules (iptables)
├── Port numbers
└── Network statistics
```

**Visual Example:**

```
Host:
├── Interface: eth0 (192.168.1.100)
├── Routing table
└── Ports: 22 (SSH), 80 (host web server)

Container 1:
├── Interface: eth0 (172.17.0.2) - virtual
├── Own routing table
├── Ports: 80 (nginx) - isolated
└── Loopback: 127.0.0.1 (separate from host)

Container 2:
├── Interface: eth0 (172.17.0.3) - virtual
├── Own routing table
├── Ports: 80 (apache) - isolated
└── Can use same port as Container 1!

Communication:
Host eth0 ←→ Virtual Bridge ←→ Container eth0
```

**Port Mapping Example:**

```bash
# Two containers, both using port 80 internally
docker run -d -p 8080:80 nginx  # Container 1
docker run -d -p 8081:80 nginx  # Container 2

Inside Container 1:
nginx listens on: 0.0.0.0:80

Inside Container 2:
nginx listens on: 0.0.0.0:80  # Same port, different namespace!

On Host:
Port 8080 → Container 1 port 80
Port 8081 → Container 2 port 80

No port conflict!
```

**Real-World Scenario:**

```
E-commerce Application:

Web Container (172.17.0.2):
├── Listen: port 3000
├── Connect to: api:8000
└── DNS resolves "api" to 172.17.0.3

API Container (172.17.0.3):
├── Listen: port 8000
├── Connect to: db:5432
└── DNS resolves "db" to 172.17.0.4

Database Container (172.17.0.4):
├── Listen: port 5432
└── No outbound connections

Network Isolation Benefits:
- Each service has clean port space
- Services communicate via Docker network
- External access via port mapping only
- Database not exposed to host network
```

---

### 3. MNT Namespace (Filesystem Isolation)

**What it does:** Each container has its own filesystem view.

**Without MNT Namespace:**
```
/ (root)
├── /home
├── /var
├── /etc
└── /usr

Everyone sees same filesystem
Changes affect everyone
```

**With MNT Namespace:**
```
Container 1 View:
/ (root)
├── /app
│   └── application files
├── /etc
│   └── container config
└── /var
    └── container logs

Container 2 View:
/ (root)
├── /app
│   └── different application
├── /etc
│   └── different config
└── /var
    └── different logs

Host View:
/ (real root)
├── /home
├── /var/lib/docker/
│   ├── overlay2/container1/
│   └── overlay2/container2/
└── /etc
```

**Mount Isolation Example:**

```bash
# Container 1
docker run -it ubuntu bash
root@container1:/# mount | grep "on / "
overlay on / type overlay

root@container1:/# ls /
app  bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root

# Container 2
docker run -it ubuntu bash
root@container2:/# ls /
bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root

# Different filesystem layers
# Each container sees own /
```

**Volume Mounts:**

```bash
docker run -v /host/data:/container/data nginx

Container View:
/container/data → sees /host/data content
                  but thinks it's local

Host View:
/host/data → actual data location

Isolation:
- Container can only access mounted paths
- Can't see other host directories
- Safe data sharing between host and container
```

---

### 4. UTS Namespace (Hostname Isolation)

**What it does:** Each container can have its own hostname.

```bash
# Host
$ hostname
production-server-1

# Container 1
$ docker run -it --hostname web-app ubuntu bash
root@web-app:/# hostname
web-app

# Container 2
$ docker run -it --hostname database ubuntu bash
root@database:/# hostname
database

# Each container thinks it has a unique hostname
# Doesn't affect host or other containers
```

**Use Case:**

```
Microservices logging:

Web Service:
hostname: web-prod-01
logs: [web-prod-01] Request received

API Service:
hostname: api-prod-01
logs: [api-prod-01] Processing request

Database:
hostname: db-prod-01
logs: [db-prod-01] Query executed

Easy to identify which container generated which log
```

---

### 5. IPC Namespace (Inter-Process Communication)

**What it does:** Isolates IPC resources (message queues, semaphores, shared memory).

```
Without IPC Namespace:
Process A → Message Queue → Process B (any process can access)

With IPC Namespace:
Container 1:
  Process A → Message Queue 1 → Process B
  (only processes in Container 1 can access)

Container 2:
  Process C → Message Queue 2 → Process D
  (only processes in Container 2 can access)

Isolation:
- Container 1 can't read Container 2's message queue
- Shared memory segments are isolated
- Prevents data leaks between containers
```

---

### 6. USER Namespace (User ID Isolation)

**What it does:** Maps user IDs between container and host.

**The Problem Without USER Namespace:**
```
Container runs as root (UID 0)
Root in container = Root on host
If container escapes = Root access to host
SECURITY RISK!
```

**With USER Namespace:**
```
Inside Container:
User: root (UID 0)
Can do anything inside container

On Host:
Same process runs as: user (UID 100000)
Limited permissions
Can't affect host even if container escapes
```

**Example:**

```bash
# Run container with user namespace
docker run --userns-remap=default nginx

Inside container:
$ id
uid=0(root) gid=0(root)  # Appears as root

On host:
$ ps aux | grep nginx
100000  5432  nginx  # Actually UID 100000

Benefit:
- Root in container ≠ root on host
- Security boundary
- Container escape doesn't give host root
```

---

### 7. Cgroup Namespace

**What it does:** Isolates cgroup hierarchy view.

```
Prevents container from seeing:
- Host cgroup configuration
- Other containers' cgroup settings
- System-wide resource allocation

Container only sees:
- Its own cgroup hierarchy
- Its own resource limits
```

---

## Control Groups (cgroups)

**cgroups** provide **resource limits** and accounting.

### Concept

> cgroups limit how much a process can **use** (CPU, memory, I/O).

Analogy: Hotel room limits
- Namespaces = Can't see into other rooms (isolation)
- cgroups = $100/day room service limit (resource limit)

### What cgroups Control

```
Resource Types:

1. CPU
   ├── CPU shares (relative weight)
   ├── CPU quota (absolute limit)
   └── CPU cores (pin to specific cores)

2. Memory
   ├── Memory limit (max RAM)
   ├── Swap limit
   └── OOM (out-of-memory) behavior

3. Block I/O
   ├── Read/write bytes per second
   ├── IOPS (operations per second)
   └── Device priorities

4. Network
   ├── Bandwidth limits
   └── Packet priorities

5. Devices
   └── Access control to devices
```

---

### CPU Limits

**CPU Shares (Relative Weight):**
```bash
# Container A: 1024 shares (default)
docker run --cpu-shares 1024 nginx

# Container B: 512 shares (half of A)
docker run --cpu-shares 512 nginx

When both compete for CPU:
Container A gets: 66% (1024/1536)
Container B gets: 33% (512/1536)

When only one is active:
Active container gets: 100% of available CPU
```

**CPU Quota (Absolute Limit):**
```bash
# Limit to 1.5 CPU cores
docker run --cpus 1.5 nginx

Behavior:
- Can use up to 150% of one core
- Equivalent to 1.5 cores
- Never exceeds this limit even if CPU is idle

# Limit to 50% of one core
docker run --cpus 0.5 nginx

Behavior:
- Maximum 50% of one CPU core
- Other 50% available to other processes
```

**CPU Pinning:**
```bash
# Run on specific cores (0 and 1)
docker run --cpuset-cpus 0,1 nginx

# Run on cores 0-3
docker run --cpuset-cpus 0-3 nginx

Use case:
- NUMA optimization
- Isolate critical workloads
- Prevent CPU migration overhead
```

**Real-World Example:**

```
Production Server: 8 CPU cores

Web Containers (high priority):
docker run --cpus 2.0 --cpu-shares 1024 web

API Containers (medium priority):
docker run --cpus 1.5 --cpu-shares 768 api

Background Jobs (low priority):
docker run --cpus 0.5 --cpu-shares 256 worker

Result:
- Web gets guaranteed 2 cores
- API gets guaranteed 1.5 cores
- Worker gets 0.5 core minimum
- During low load, any container can use spare CPU
```

---

### Memory Limits

**Memory Limit:**
```bash
# Limit to 512MB
docker run --memory 512m nginx

# Limit to 1GB
docker run --memory 1g nginx

Behavior:
- Process can use up to limit
- Exceeding limit → OOM (Out of Memory) kill
- Container terminates
```

**Memory + Swap:**
```bash
# Memory: 512MB, Swap: 512MB
docker run --memory 512m --memory-swap 1g nginx

Total available: 512MB RAM + 512MB swap = 1GB

# Disable swap
docker run --memory 512m --memory-swap 512m nginx
```

**Memory Reservation:**
```bash
# Reserve 256MB, limit 512MB
docker run --memory 512m --memory-reservation 256m nginx

Behavior:
- Guaranteed: 256MB
- Can use up to: 512MB
- Under memory pressure: shrinks to reservation
```

**OOM Behavior:**
```bash
# What happens when memory limit exceeded:

Container using 510MB (limit: 512MB):
✓ Running normally

Container tries to allocate 3MB more:
❌ Total would be 513MB > 512MB limit
❌ Kernel OOM killer activates
❌ Container process killed
❌ Container exits

# View in logs:
$ docker logs container_id
...
Killed (OOM - Out of Memory)

# Container status:
$ docker ps -a
STATUS
Exited (137) # Exit code 137 = killed by signal 9 (SIGKILL from OOM)
```

**Real-World Scenario:**

```
Database Container:
docker run -d \
  --name postgres \
  --memory 2g \
  --memory-reservation 1g \
  postgres

Normal operation:
- Uses 800MB: ✓ Under reservation
- Performance: Optimal

Under load:
- Uses 1.5GB: ✓ Under limit
- Performance: Good

Memory leak:
- Tries to use 2.1GB: ❌ Exceeds limit
- Container killed
- Automatic restart (if restart policy set)
- Prevents affecting other containers
```

---

### Disk I/O Limits

```bash
# Limit read speed to 1MB/s
docker run --device-read-bps /dev/sda:1mb nginx

# Limit write speed to 10MB/s
docker run --device-write-bps /dev/sda:10mb nginx

# Limit IOPS (operations per second)
docker run --device-read-iops /dev/sda:100 nginx

Use case:
- Prevent one container from starving disk I/O
- Quality of Service for databases
- Ensure fair access to storage
```

---

## How Namespaces and cgroups Work Together

**Combined Example:**

```bash
docker run -d \
  --name web-app \
  --hostname web-prod-01 \
  --cpus 2 \
  --memory 1g \
  -p 8080:80 \
  nginx
```

**What Docker Creates:**

```
1. PID Namespace:
   ├── nginx appears as PID 1 inside container
   └── Actually PID 7654 on host

2. NET Namespace:
   ├── Container IP: 172.17.0.2
   ├── Container port 80 mapped to host 8080
   └── Own routing table

3. MNT Namespace:
   ├── Container sees own filesystem
   ├── nginx files at /usr/share/nginx
   └── Can't access host filesystem (except volumes)

4. UTS Namespace:
   └── Hostname: web-prod-01

5. IPC Namespace:
   └── Isolated message queues

6. USER Namespace (if enabled):
   ├── Root inside = UID 100000 outside
   └── Limited host permissions

7. CPU cgroup:
   ├── Maximum 2 CPU cores
   └── Can't use more even if available

8. Memory cgroup:
   ├── Maximum 1GB RAM
   ├── Exceeding → OOM kill
   └── Protects host and other containers

Result:
- Isolated environment (namespaces)
- Resource limits (cgroups)
- Secure container
```

---

## Verification Commands

**Check Namespaces:**
```bash
# Get container PID
docker inspect -f '{{.State.Pid}}' container_name
# Output: 7654

# View container's namespaces
sudo ls -la /proc/7654/ns/
lrwxrwxrwx 1 root root 0 Jan 17 10:00 ipc -> ipc:[4026532508]
lrwxrwxrwx 1 root root 0 Jan 17 10:00 mnt -> mnt:[4026532506]
lrwxrwxrwx 1 root root 0 Jan 17 10:00 net -> net:[4026532511]
lrwxrwxrwx 1 root root 0 Jan 17 10:00 pid -> pid:[4026532509]
lrwxrwxrwx 1 root root 0 Jan 17 10:00 uts -> uts:[4026532507]

# Each number is unique namespace ID
```

**Check cgroups:**
```bash
# Find container cgroup path
docker inspect -f '{{.Id}}' container_name
# Output: a1b2c3d4e5f6...

# View CPU limits
cat /sys/fs/cgroup/cpu/docker/a1b2c3d4e5f6.../cpu.cfs_quota_us
# Output: 200000 (2 cores)

# View memory limits
cat /sys/fs/cgroup/memory/docker/a1b2c3d4e5f6.../memory.limit_in_bytes
# Output: 1073741824 (1GB)

# Current memory usage
cat /sys/fs/cgroup/memory/docker/a1b2c3d4e5f6.../memory.usage_in_bytes
# Output: 536870912 (512MB)
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. __________ provide isolation by giving containers their own view of resources.
2. __________ limit how much resources a container can use.
3. The __________ namespace isolates process IDs.
4. The __________ namespace gives each container its own network stack.
5. If a container exceeds its memory limit, the Linux __________ kills the process.
6. CPU __________ provide relative weight, while CPU __________ provide absolute limits.

### True/False

1. ⬜ Each container sees all processes running on the host
2. ⬜ Two containers can both listen on port 80 without conflict
3. ⬜ cgroups provide isolation, namespaces provide resource limits
4. ⬜ Root user in a container is the same as root on the host (without user namespaces)
5. ⬜ A container can see the filesystem of other containers by default
6. ⬜ Setting `--cpus 2` means the container will always use exactly 2 CPU cores
7. ⬜ Memory limits prevent a container from using more RAM than specified

### Multiple Choice

1. What happens when a container exceeds its memory limit?
   - A) It slows down but continues running
   - B) It gets killed by the OOM killer
   - C) It automatically gets more memory
   - D) Nothing, limits are just suggestions

2. Which namespace allows two containers to have the same hostname?
   - A) PID
   - B) NET
   - C) MNT
   - D) UTS

3. What does `--cpus 1.5` mean?
   - A) Use CPU cores 1 and 5
   - B) Use 150% of total CPU
   - C) Use up to 1.5 CPU cores
   - D) Use 1.5GHz CPU

4. Which namespace prevents a container from seeing other containers' processes?
   - A) NET namespace
   - B) PID namespace
   - C) MNT namespace
   - D) USER namespace

5. What is the purpose of the USER namespace?
   - A) Isolate network users
   - B) Map container UIDs to different host UIDs
   - C) Manage user permissions
   - D) Create container users

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. Namespaces
2. cgroups (or Control Groups)
3. PID
4. NET (or Network)
5. OOM killer (or Out-of-Memory killer)
6. shares, quota (or cpus)

**True/False:**
1. ❌ False - PID namespace isolates processes, container only sees its own
2. ✅ True - NET namespace provides separate network stack per container
3. ❌ False - Reversed: Namespaces provide isolation, cgroups provide limits
4. ✅ True - Without user namespace mapping, root in container = root on host
5. ❌ False - MNT namespace isolates filesystem view
6. ❌ False - It's a maximum limit, container can use less if not needed
7. ✅ True - Memory cgroup enforces hard limit, kills process if exceeded

**Multiple Choice:**
1. **B** - It gets killed by the OOM killer
2. **D** - UTS namespace (Unix Timesharing System)
3. **C** - Use up to 1.5 CPU cores
4. **B** - PID namespace
5. **B** - Map container UIDs to different host UIDs

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the difference between namespaces and cgroups

<details>
<summary><strong>View Answer</strong></summary>

**Simple Answer:**

**Namespaces** = What you can **SEE** (Isolation)  
**cgroups** = What you can **USE** (Resource limits)

**Detailed Explanation:**

**Namespaces - Isolation:**
```
Purpose: Create isolated views of system resources

Example:
Container A sees:
├── Processes: PID 1, 2, 3 (own processes only)
├── Network: IP 172.17.0.2
├── Filesystem: /app, /etc, /var
└── Hostname: web-app

Container B sees:
├── Processes: PID 1, 2, 3 (different processes!)
├── Network: IP 172.17.0.3
├── Filesystem: /app, /etc, /var (different files!)
└── Hostname: database

Result:
- Each container thinks it's alone
- Can't see other containers
- Can't interfere with others
```

**cgroups - Resource Limits:**
```
Purpose: Limit and account for resource usage

Example:
Container A:
├── CPU: Maximum 2 cores
├── Memory: Maximum 1GB
├── Disk I/O: Maximum 100MB/s
└── If exceeded: Throttled or killed

Container B:
├── CPU: Maximum 1 core
├── Memory: Maximum 512MB
└── Prevents resource starvation

Result:
- Fair resource allocation
- Prevents one container from monopolizing
- Guarantees performance
```

**Real-World Analogy:**

```
Hotel Building:

Namespaces (Privacy):
- Room walls (can't see into other rooms)
- Own key (can't access other rooms)
- Own bathroom (isolated facilities)
- Own view from window

cgroups (Limits):
- Room service limit: $100/day
- Water usage limit: 50 gallons/day
- Electricity limit: 10 kWh/day
- Can't exceed even if building has spare capacity
```

**Why Both Are Needed:**

```
Without namespaces (only cgroups):
Container A: Limited to 1GB RAM ✓
But: Can see and kill Container B's processes ✗
Result: No security

Without cgroups (only namespaces):
Container A: Can't see Container B ✓
But: Can use 100% CPU, starve Container B ✗
Result: No resource fairness

With both:
Container A:
- Isolated from B (namespaces) ✓
- Limited resources (cgroups) ✓
Result: Secure and fair
```

**Interview Summary:**
"Namespaces provide isolation so containers can't see or interfere with each other. cgroups provide resource limits so containers can't monopolize system resources. Together, they enable secure multi-tenant container execution."

</details>

### Question 2: What would happen if you removed namespace isolation from Docker?

<details>
<summary><strong>View Answer</strong></summary>

**Consequences of No Namespaces:**

**1. No PID Isolation**
```
All containers see all processes:

Container A (Web):
$ ps aux
PID   COMMAND
1     /sbin/init (host)
542   nginx (me)
543   postgres (Container B)
544   redis (Container C)
...

Problems:
- Can kill other containers' processes
  $ kill 543  # Kills database in Container B!
  
- No process privacy
- Security breach
- Can monitor other containers' processes
```

**2. No Network Isolation**
```
All containers share network:

Problem 1: Port conflicts
Container A: nginx on port 80
Container B: apache on port 80
Result: Second container fails to start

Problem 2: No network privacy
Container A can sniff Container B's traffic
All containers see each other's connections

Problem 3: IP conflicts
Can't assign separate IPs
Single network interface shared
```

**3. No Filesystem Isolation**
```
All containers see same filesystem:

Container A modifies /etc/nginx/nginx.conf
Container B's nginx breaks (uses same file)

Container A creates /tmp/data
Container B can read, modify, or delete it

Container A installs library version 1.0
Container B needs library version 2.0
Conflict!

No isolation = No multi-tenancy
```

**4. No Hostname Isolation**
```
All containers same hostname:

Container A: hostname = docker-host
Container B: hostname = docker-host
Container C: hostname = docker-host

Problems:
- Can't identify which container in logs
- Services confused about identity
- Distributed systems break (expect unique hostnames)
```

**5. No IPC Isolation**
```
Shared message queues:

Container A creates message queue "orders"
Container B creates message queue "orders"
Result: Both access same queue!

Data leakage:
Container A: Sensitive payment data
Container B: Can read Container A's shared memory

Security violation!
```

**Real-World Disaster Scenario:**

```
E-commerce Platform (without namespaces):

Deployment:
- Web Container
- API Container  
- Database Container

What Goes Wrong:

Day 1:
Web container starts nginx on port 80 ✓

Day 2:
API container tries to start on port 80 ✗
Fails: Address already in use

Solution: Manually configure different ports
Web: port 8080
API: port 8081
(Defeats purpose of containerization!)

Day 3:
Database process killed
Investigation shows: API container had bug
Bug killed process with same PID as database
Both thought they were PID 543

Day 4:
Web container hacked
Attacker gains access
Can see all processes: ps aux
Finds database process
Reads database memory
Steals customer data

Total Failure:
- No isolation
- No security
- Containers interfere with each other
- Can't run multiple similar services
- Might as well run on bare metal
```

**What We'd Lose:**

```
1. Multi-Tenancy
   Can't run multiple customers' apps safely

2. Process Isolation
   Containers can kill each other

3. Network Isolation
   Can't run same service multiple times
   Port conflicts
   No network privacy

4. Filesystem Isolation
   Shared files = conflicts
   Can't have different versions of libraries

5. Security
   One compromised container = all compromised
   No containment

6. Resource Accountability
   Can't track which container uses what

7. Clean Shutdown
   Can't cleanly stop one container
   Affects all processes
```

**Why Docker Without Namespaces = Useless:**

```
Docker's value proposition:
"Package and run isolated applications"

Without namespaces:
- Not isolated ✗
- Processes interfere ✗
- Security compromised ✗
- Can't run multiple instances ✗

Conclusion:
Namespaces are fundamental to containers
Without them, containers are just fancy chroot
No better than running processes directly
```

</details>

### Question 3: How do CPU limits actually work? If I set `--cpus 2`, what happens under the hood?

<details>
<summary><strong>View Answer</strong></summary>

**How `--cpus 2` Works:**

**1. Docker Creates CPU cgroup:**
```bash
# Command
docker run --cpus 2 nginx

# Docker creates cgroup at:
/sys/fs/cgroup/cpu/docker/<container-id>/

# Sets two parameters:
cpu.cfs_period_us = 100000  # 100ms period
cpu.cfs_quota_us = 200000   # 200ms quota

# Math:
quota/period = 200000/100000 = 2.0 cores
```

**2. Kernel Enforcement:**
```
CFS Scheduler (Completely Fair Scheduler):

Every 100ms period:
├── Container can execute for 200ms total
├── Across all cores
└── Then throttled until next period

Example timeline:
Time 0ms:    Container starts executing
Time 100ms:  Used 200ms (2 cores worth)
             Throttled for rest of period
Time 200ms:  New period starts
             Can use 200ms again
```

**Real Execution Examples:**

**Example 1: Using Full Quota**
```
Container with --cpus 2 on 8-core system:

Timeline:
t=0-100ms:   Uses 200ms CPU time (2 cores)
t=100ms:     Quota exhausted, throttled
t=100-200ms: Waiting (throttled)
t=200ms:     New period, quota reset
t=200-300ms: Uses 200ms CPU time again

Effective utilization: 2 cores constant
```

**Example 2: Under-Utilization**
```
Container with --cpus 2, light workload:

Timeline:
t=0-100ms:   Uses only 50ms CPU time (0.5 core)
t=100ms:     Unused quota (150ms) discarded
t=100-200ms: Still only needs 50ms
t=200-300ms: Still only needs 50ms

Effective utilization: 0.5 core average
Note: Unused quota doesn't accumulate!
```

**Example 3: Burst Usage**
```
Container with --cpus 2:

Timeline:
t=0-20ms:    Idle (0 CPU)
t=20-40ms:   Heavy burst, uses 40ms on 8 cores
             Actually consumes 320ms quota!
t=40ms:      Quota exceeded (320ms > 200ms)
             Throttled for 60ms
t=100ms:     New period starts

Behavior:
- Can burst across all cores briefly
- But total time limited
- Throttled when quota exhausted
```

**Viewing Actual Throttling:**

```bash
# Start CPU-intensive container
docker run -d --name cpu-test --cpus 2 stress --cpu 8

# Check throttling stats
docker stats cpu-test

# Or directly from cgroup:
cat /sys/fs/cgroup/cpu/docker/<container-id>/cpu.stat

Output:
nr_periods 1342         # Number of periods
nr_throttled 892        # Times throttled
throttled_time 45234ms  # Total time throttled

Analysis:
- 892/1342 = 66% of periods hit limit
- Container is CPU-bound
- Regularly using full 2-core quota
```

**Multi-Core Distribution:**

```
Container with --cpus 2 on 4-core system:

Scenario A: Single-threaded application
Core 0: ████████████████ 100%
Core 1: ░░░░░░░░░░░░░░░░   0%
Core 2: ░░░░░░░░░░░░░░░░   0%
Core 3: ░░░░░░░░░░░░░░░░   0%
Total: 1 core (under limit)

Scenario B: Multi-threaded (4 threads)
Core 0: ████████░░░░░░░░  50%
Core 1: ████████░░░░░░░░  50%
Core 2: ████████░░░░░░░░  50%
Core 3: ████████░░░░░░░░  50%
Total: 2 cores (at limit)

Scenario C: Multi-threaded (trying to use all)
Core 0: ████████░░░░░░░░  50%
Core 1: ████████░░░░░░░░  50%
Core 2: ████████░░░░░░░░  50%
Core 3: ████████░░░░░░░░  50%
Total: 2 cores (throttled to limit)
Wants 4 cores, only gets 2
```

**Comparison: --cpus vs --cpuset-cpus:**

```bash
# --cpus 2 (anywhere on system)
docker run --cpus 2 stress

Result:
- Can use any 2 cores' worth
- Scheduler decides which physical cores
- Can migrate between cores
- Total time limited

# --cpuset-cpus 0,1 (pinned to cores)
docker run --cpuset-cpus 0,1 stress

Result:
- Only uses cores 0 and 1
- Can use 100% of those cores
- No migration to other cores
- No time limit within those cores

# Combined (best for NUMA)
docker run --cpus 2 --cpuset-cpus 0,1 stress

Result:
- Limited to cores 0 and 1
- Limited to 2 cores worth of time
- Predictable NUMA locality
```

**Production Example:**

```
Server: 16 cores

Web Containers (customer-facing):
docker run --cpus 4 web

Database (critical):
docker run --cpus 8 --cpuset-cpus 0-7 db

Background Jobs:
docker run --cpus 2 worker

Under load:
- Web: Uses up to 4 cores anywhere (cores 8-15 usually)
- DB: Uses up to 8 cores on cores 0-7 (NUMA node 0)
- Worker: Uses up to 2 cores anywhere

Benefits:
- DB has dedicated NUMA node
- Web can't starve DB
- Fair resource distribution
- All guaranteed minimums
```

**Key Takeaways:**

```
--cpus 2 means:
1. Maximum 2 CPU cores worth of time
2. Measured per 100ms period
3. Can burst across all cores
4. But total time limited
5. Throttled when quota exhausted
6. Unused quota doesn't accumulate

Not:
❌ Always uses exactly 2 cores
❌ Reserved 2 cores (other containers can't use)
❌ Runs on specific cores
❌ Guarantees 2 cores
```

</details>

</details>

---

[← Previous: 2.1 Docker Engine Components](01-engine-components.md) | [Next: 3.1 Working with Images →](../03-docker-images/01-image-basics.md)