# 4.1 Container Lifecycle

Understanding container states and lifecycle management from creation to removal.

---

## Container States

A Docker container can be in one of several states throughout its lifecycle:

```
Container States:

Created  → Container exists but hasn't started
Running  → Container is executing
Paused   → Container is frozen (processes suspended)
Stopped  → Container exited (can be restarted)
Removed  → Container deleted (cannot recover)
```

### State Diagram

```
                    docker create
                         │
                         ▼
                   ┌──────────┐
                   │ CREATED  │
                   └─────┬────┘
                         │ docker start
                         ▼
    ┌───────────────────────────────────┐
    │                                   │
    │        ┌──────────┐               │
    │    ┌───│ RUNNING  │───┐           │
    │    │   └──────────┘   │           │
    │    │                  │           │
    │    │ docker pause     │ docker    │
    │    │                  │ stop      │
    │    ▼                  ▼           │
    │ ┌────────┐      ┌─────────┐       │
    │ │ PAUSED │      │ STOPPED │       │
    │ └────┬───┘      └────┬────┘       │
    │      │               │            │
    │      │ docker        │ docker     │
    │      │ unpause       │ start      │
    │      │               │            │
    │      └───────┬───────┘            │
    │              │                    │
    └──────────────┘                    │
                                        │ docker rm
                                        ▼
                                  ┌──────────┐
                                  │ REMOVED  │
                                  └──────────┘
```

---

## Creating Containers

### docker create

**Purpose:** Create a container without starting it.

**Syntax:**
```bash
docker create [OPTIONS] IMAGE [COMMAND] [ARG...]
```

**Examples:**
```bash
# Basic creation
docker create nginx
# Container ID: a1b2c3d4e5f6

# With name
docker create --name my-nginx nginx

# With port mapping
docker create -p 8080:80 nginx

# With environment variables
docker create -e NODE_ENV=production node:18

# With volume
docker create -v /data:/app/data nginx

# With resource limits
docker create --memory 512m --cpus 1.5 nginx
```

**State After Creation:**
```bash
docker ps -a
CONTAINER ID   STATUS
a1b2c3d4e5f6   Created   # Not running yet
```

**When to Use:**
```
✓ Pre-create containers for later use
✓ Separate creation and starting
✓ Prepare container configuration
✓ Automated provisioning
```

---

### docker run

**Purpose:** Create AND start a container in one command.

**Syntax:**
```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

**Equivalent to:**
```bash
docker create [OPTIONS] IMAGE [COMMAND] [ARG...]
docker start [CONTAINER]
```

**Common Options:**

```bash
# Detached mode (background)
docker run -d nginx

# Interactive with terminal
docker run -it ubuntu bash

# Remove after exit
docker run --rm nginx

# Named container
docker run --name web nginx

# Port mapping
docker run -p 8080:80 nginx
docker run -p 127.0.0.1:8080:80 nginx  # Bind to localhost only

# Environment variables
docker run -e NODE_ENV=production node:18
docker run --env-file .env node:18

# Volume mounting
docker run -v /host/data:/container/data nginx
docker run -v my-volume:/data nginx

# Network
docker run --network my-network nginx

# Resource limits
docker run --memory 1g --cpus 2 nginx

# Restart policy
docker run --restart always nginx
docker run --restart unless-stopped nginx
```

---

## Starting and Stopping

### docker start

**Purpose:** Start one or more stopped containers.

**Syntax:**
```bash
docker start [OPTIONS] CONTAINER [CONTAINER...]
```

**Examples:**
```bash
# Start single container
docker start my-nginx

# Start multiple containers
docker start web api database

# Start and attach to output
docker start -a my-nginx

# Start interactively
docker start -ai my-ubuntu
```

**State Transition:**
```
CREATED  ──start──> RUNNING
STOPPED  ──start──> RUNNING
```

---

### docker stop

**Purpose:** Stop one or more running containers gracefully.

**Syntax:**
```bash
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

**How It Works:**
```
1. Send SIGTERM to container (PID 1)
2. Wait for graceful shutdown (default: 10 seconds)
3. If still running after timeout → Send SIGKILL
4. Container enters STOPPED state
```

**Examples:**
```bash
# Stop single container
docker stop my-nginx

# Stop multiple containers
docker stop web api database

# Custom timeout
docker stop -t 30 my-app  # Wait 30 seconds before SIGKILL

# Stop all running containers
docker stop $(docker ps -q)
```

**Graceful Shutdown Example:**
```python
# Python application with signal handling
import signal
import sys

def graceful_shutdown(signum, frame):
    print("Received SIGTERM, shutting down...")
    # Close connections
    db.close()
    # Finish current requests
    server.shutdown()
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)

# When docker stop is called:
# 1. SIGTERM sent
# 2. graceful_shutdown() executes
# 3. Clean shutdown
```

---

### docker restart

**Purpose:** Restart one or more containers.

**Syntax:**
```bash
docker restart [OPTIONS] CONTAINER [CONTAINER...]
```

**Equivalent to:**
```bash
docker stop CONTAINER
docker start CONTAINER
```

**Examples:**
```bash
# Restart single container
docker restart my-nginx

# Restart with timeout
docker restart -t 5 my-app

# Restart all containers
docker restart $(docker ps -q)
```

---

### docker kill

**Purpose:** Force stop a container immediately.

**Syntax:**
```bash
docker kill [OPTIONS] CONTAINER [CONTAINER...]
```

**How It Works:**
```
1. Send SIGKILL to container (PID 1)
2. No graceful shutdown
3. Immediate termination
4. Container enters STOPPED state
```

**Examples:**
```bash
# Kill container (SIGKILL)
docker kill my-nginx

# Send specific signal
docker kill -s SIGINT my-app
docker kill -s SIGUSR1 my-app

# Kill all running containers
docker kill $(docker ps -q)
```

**When to Use:**
```
✓ Container not responding to stop
✓ Need immediate termination
✓ Debugging/emergency situations

⚠ Avoid in production (use stop for graceful shutdown)
```

---

## Pausing and Unpausing

### docker pause

**Purpose:** Suspend all processes in a container.

**Syntax:**
```bash
docker pause CONTAINER [CONTAINER...]
```

**How It Works:**
```
Uses cgroups freezer to suspend all processes
- No CPU time allocated
- Processes frozen in memory
- Network connections preserved
- Very fast operation
```

**Examples:**
```bash
# Pause container
docker pause my-nginx

# Pause multiple
docker pause web api worker
```

**Use Cases:**
```
✓ Temporarily free CPU resources
✓ Checkpoint container state
✓ Debugging (freeze for inspection)
✓ Resource management

Not for:
✗ Long-term storage (use stop)
✗ Saving state across reboots
```

---

### docker unpause

**Purpose:** Resume all processes in a paused container.

**Syntax:**
```bash
docker unpause CONTAINER [CONTAINER...]
```

**Examples:**
```bash
# Unpause container
docker unpause my-nginx

# Unpause multiple
docker unpause web api worker
```

**State Transition:**
```
RUNNING ──pause──> PAUSED
PAUSED  ──unpause──> RUNNING
```

---

## Removing Containers

### docker rm

**Purpose:** Remove one or more stopped containers.

**Syntax:**
```bash
docker rm [OPTIONS] CONTAINER [CONTAINER...]
```

**Examples:**
```bash
# Remove stopped container
docker rm my-nginx

# Remove multiple containers
docker rm web api database

# Force remove running container
docker rm -f my-nginx

# Remove and volumes
docker rm -v my-nginx

# Remove all stopped containers
docker rm $(docker ps -aq -f status=exited)

# Remove containers older than 24 hours
docker container prune --filter "until=24h"
```

**Options:**
```bash
-f, --force      # Remove running container
-v, --volumes    # Remove associated volumes
-l, --link       # Remove specified link
```

---

## Container Lifecycle Patterns

### Pattern 1: One-off Command

```bash
# Run command and auto-remove
docker run --rm ubuntu echo "Hello World"

# Process:
1. Create container
2. Start container
3. Execute command
4. Exit
5. Remove container automatically

# Use case: Scripts, batch jobs, testing
```

### Pattern 2: Long-running Service

```bash
# Start service in background
docker run -d --name web \
  --restart unless-stopped \
  -p 8080:80 \
  nginx

# Process:
1. Create container
2. Start in detached mode
3. Auto-restart if fails
4. Runs indefinitely

# Use case: Web servers, APIs, databases
```

### Pattern 3: Interactive Session

```bash
# Interactive bash session
docker run -it --rm ubuntu bash

# Process:
1. Create container
2. Start with TTY
3. Attach to bash
4. User interacts
5. Exit bash
6. Remove container

# Use case: Debugging, exploration, development
```

### Pattern 4: Development with Live Reload

```bash
# Mount source code and run
docker run -d --name dev \
  -v $(pwd):/app \
  -p 3000:3000 \
  node:18 npm run dev

# Process:
1. Create container
2. Mount local code
3. Start dev server
4. Code changes reflected immediately
5. Stop when done
6. Remove container

# Use case: Development, testing
```

---

## Real-World Examples

### Example 1: Web Application Deployment

```bash
# Initial deployment
docker run -d \
  --name production-web \
  --restart always \
  --memory 2g \
  --cpus 2 \
  -p 80:80 \
  -e NODE_ENV=production \
  -v /data/logs:/app/logs \
  myapp:v1.0

# Update to new version
docker stop production-web           # Graceful shutdown
docker rm production-web             # Remove old container
docker run -d \
  --name production-web \
  --restart always \
  --memory 2g \
  --cpus 2 \
  -p 80:80 \
  -e NODE_ENV=production \
  -v /data/logs:/app/logs \
  myapp:v1.1                        # New version

# Rollback if needed
docker stop production-web
docker rm production-web
docker run -d \
  --name production-web \
  --restart always \
  --memory 2g \
  --cpus 2 \
  -p 80:80 \
  -e NODE_ENV=production \
  -v /data/logs:/app/logs \
  myapp:v1.0                        # Previous version
```

### Example 2: Database Maintenance

```bash
# Stop database for maintenance
docker stop postgres-db              # Graceful shutdown (saves data)

# Create backup
docker run --rm \
  --volumes-from postgres-db \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/db-backup.tar.gz /var/lib/postgresql/data

# Perform maintenance
docker start postgres-db
docker exec postgres-db psql -U postgres -c "VACUUM FULL;"
docker exec postgres-db psql -U postgres -c "REINDEX DATABASE myapp;"

# Restart if needed
docker restart postgres-db
```

### Example 3: Testing Pipeline

```bash
# Run unit tests
docker run --rm \
  -v $(pwd):/app \
  -w /app \
  node:18 npm test

# Run integration tests
docker run --rm \
  --network test-network \
  -e DATABASE_URL=postgres://testdb:5432/test \
  -v $(pwd):/app \
  -w /app \
  node:18 npm run test:integration

# Run linting
docker run --rm \
  -v $(pwd):/app \
  -w /app \
  node:18 npm run lint

# All tests passed, build production image
docker build -t myapp:v1.2 .

# All containers auto-removed after completion
```

### Example 4: Blue-Green Deployment

```bash
# Current (blue) running
docker ps
# blue-web running on port 80

# Start new version (green)
docker run -d \
  --name green-web \
  -p 8080:80 \
  myapp:v2.0

# Test green
curl http://localhost:8080/health

# Switch traffic (update load balancer)
# ...

# Stop blue
docker stop blue-web

# Wait for confirmation
sleep 300

# Remove blue
docker rm blue-web

# Rename green to blue for next deployment
docker rename green-web blue-web
docker stop blue-web
docker run -d \
  --name blue-web \
  -p 80:80 \
  myapp:v2.0
docker rm green-web
```

---

## Lifecycle Management Best Practices

### 1. Use Restart Policies

```bash
# Always restart (including on boot)
docker run --restart always nginx

# Restart unless manually stopped
docker run --restart unless-stopped nginx

# Restart on failure (max 3 times)
docker run --restart on-failure:3 nginx
```

### 2. Clean Up Regularly

```bash
# Remove stopped containers
docker container prune

# Remove containers older than 24h
docker container prune --filter "until=24h"

# Remove all stopped containers
docker rm $(docker ps -aq -f status=exited)
```

### 3. Use --rm for Temporary Containers

```bash
# Auto-remove after exit
docker run --rm alpine echo "test"

# Development testing
docker run --rm -v $(pwd):/app node:18 npm test
```

### 4. Graceful Shutdown

```bash
# Stop with generous timeout
docker stop -t 60 my-app

# In application, handle SIGTERM
signal.signal(signal.SIGTERM, graceful_shutdown)
```

### 5. Health Checks

```dockerfile
# In Dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost/health || exit 1
```

```bash
# Check container health
docker ps
# Shows (healthy), (unhealthy), or (starting)
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The __________ command creates a container but doesn't start it.
2. When you run __________, the container is first stopped with SIGTERM, then SIGKILL after timeout.
3. The __________ state suspends all processes in a container without stopping it.
4. Containers in __________ state can be restarted, while containers in __________ state cannot.
5. The __________ flag automatically removes a container when it exits.
6. Using __________ immediately terminates a container without graceful shutdown.

### True/False

1. ⬜ docker run is equivalent to docker create followed by docker start
2. ⬜ A paused container consumes CPU resources
3. ⬜ docker stop immediately kills the container
4. ⬜ You can restart a removed container
5. ⬜ The --restart always policy will restart containers even after Docker daemon restart
6. ⬜ docker rm can remove running containers without the -f flag
7. ⬜ docker create starts the container immediately

### Multiple Choice

1. What happens when docker stop times out?
   - A) Container continues running
   - B) Sends SIGKILL to force stop
   - C) Waits indefinitely
   - D) Returns error and exits

2. Which state uses the least resources?
   - A) Running
   - B) Paused
   - C) Stopped
   - D) All equal

3. What is the default timeout for docker stop?
   - A) 5 seconds
   - B) 10 seconds
   - C) 30 seconds
   - D) 60 seconds

4. Which command force removes a running container?
   - A) docker rm container
   - B) docker rm -f container
   - C) docker kill container
   - D) docker stop -f container

5. What does --restart unless-stopped do?
   - A) Never restart
   - B) Restart always, even after manual stop
   - C) Restart unless manually stopped
   - D) Restart only on failure

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. docker create
2. docker stop
3. PAUSED
4. STOPPED, REMOVED
5. --rm
6. docker kill

**True/False:**
1. ✅ True - docker run = create + start
2. ❌ False - Paused containers are frozen, no CPU usage
3. ❌ False - Sends SIGTERM first, waits, then SIGKILL if needed
4. ❌ False - Removed containers are permanently deleted
5. ✅ True - Restarts on daemon restart too
6. ❌ False - Need -f flag to remove running containers
7. ❌ False - docker create only creates, doesn't start

**Multiple Choice:**
1. **B** - Sends SIGKILL to force stop
2. **C** - Stopped (no process running, minimal memory)
3. **B** - 10 seconds
4. **B** - docker rm -f container
5. **C** - Restart unless manually stopped

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the difference between docker stop and docker kill

<details>
<summary><strong>View Answer</strong></summary>

**Core Difference:**

**docker stop** = Graceful shutdown (SIGTERM → wait → SIGKILL)  
**docker kill** = Immediate termination (SIGKILL)

---

**docker stop - Graceful Shutdown**

**Process:**
```
1. Send SIGTERM to PID 1
2. Wait for timeout (default: 10 seconds)
3. Application can shutdown gracefully
4. If still running after timeout → SIGKILL
5. Container enters STOPPED state
```

**Timeline:**
```
t=0s:   docker stop myapp
        → SIGTERM sent to container

t=0-10s: Application receives signal
         → Closes database connections
         → Finishes current requests
         → Saves state
         → Exits cleanly

t=10s:  If still running → SIGKILL sent
        → Forced termination
```

**Application Handling:**
```python
import signal
import sys

def graceful_shutdown(signum, frame):
    print("Shutting down gracefully...")
    
    # 1. Stop accepting new requests
    server.stop_accepting()
    
    # 2. Finish current requests
    for request in active_requests:
        request.complete()
    
    # 3. Close database connections
    db.close()
    
    # 4. Save application state
    save_state()
    
    # 5. Exit cleanly
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)

# With docker stop:
# graceful_shutdown() is called
# Clean exit after finishing work
```

---

**docker kill - Immediate Termination**

**Process:**
```
1. Send SIGKILL to PID 1
2. No waiting
3. No graceful shutdown
4. Immediate termination
5. Container enters STOPPED state
```

**Timeline:**
```
t=0s: docker kill myapp
      → SIGKILL sent
      → Process terminated immediately
      → No cleanup possible
```

**What Happens:**
```
✗ Active requests interrupted
✗ Database connections not closed
✗ Transactions may be incomplete
✗ Data might be lost
✗ Temporary files not cleaned up
✗ Locks not released
```

---

**Real-World Scenario:**

**E-commerce Checkout Service:**

```bash
# 100 users checking out when update needed
```

**Using docker stop (Good):**
```bash
docker stop -t 30 checkout-service

Timeline:
t=0s:   SIGTERM sent
t=0s:   Stop accepting new checkout requests
t=1s:   95 checkouts complete
t=5s:   98 checkouts complete
t=10s:  100 checkouts complete
t=10s:  Database connections closed
t=10s:  Clean exit

Result:
✓ All 100 checkouts completed
✓ No lost transactions
✓ No data corruption
✓ Happy customers
```

**Using docker kill (Bad):**
```bash
docker kill checkout-service

Timeline:
t=0s: SIGKILL sent
t=0s: Process terminated

Result:
✗ 100 checkouts interrupted
✗ Payment processed but order not created
✗ Money taken but no confirmation
✗ Database in inconsistent state
✗ Customer complaints
✗ Manual cleanup needed
✗ Lost revenue
```

---

**When to Use Each:**

**docker stop:**
```
Production deployments        ✓
Rolling updates              ✓
Normal shutdowns             ✓
Maintenance windows          ✓
Scheduled restarts           ✓
```

**docker kill:**
```
Unresponsive containers      ✓
Emergency situations         ✓
Container in bad state       ✓
Development debugging        ✓
Testing failure scenarios    ✓

Production use              ✗
```

---

**Custom Timeout:**

```bash
# Default: 10 seconds
docker stop myapp

# Custom timeout: 60 seconds
docker stop -t 60 myapp

# Use case: Long-running tasks
# Example: Video encoding, batch processing
docker stop -t 300 video-encoder

# Gives 5 minutes to finish current encoding
```

---

**Comparison Table:**

| Feature | docker stop | docker kill |
|---------|-------------|-------------|
| Signal | SIGTERM | SIGKILL |
| Graceful | Yes | No |
| Timeout | Configurable (default 10s) | None |
| App can cleanup | Yes | No |
| Data safety | High | Low |
| Speed | Slower | Instant |
| Production use | Yes | Emergency only |

---

**Best Practice:**

```bash
# Always try stop first
docker stop -t 30 myapp

# If that doesn't work (container hung)
docker ps  # Still running?
docker kill myapp  # Force kill as last resort

# Check why stop didn't work
docker logs myapp
```

</details>

### Question 2: What are the different container restart policies and when would you use each?

<details>
<summary><strong>View Answer</strong></summary>

**Restart Policies:**

Docker provides four restart policies that control when containers restart:

1. **no** (default)
2. **on-failure[:max-retries]**
3. **always**
4. **unless-stopped**

---

### 1. no (Default)

**Behavior:**
```
Never restart the container automatically
Container stays stopped after exit
```

**Usage:**
```bash
docker run --restart no myapp
# or just
docker run myapp  # no is default
```

**When to Use:**
```
✓ One-off commands
✓ Batch jobs
✓ Testing/development
✓ Scripts that should run once
```

**Example:**
```bash
# Database migration (run once)
docker run --rm \
  --restart no \
  myapp migrate

# Exits after migration
# Does not restart
```

---

### 2. on-failure[:max-retries]

**Behavior:**
```
Restart only if container exits with non-zero status
Can limit number of restart attempts
Does not restart on daemon restart
```

**Usage:**
```bash
# Restart on failure, unlimited attempts
docker run --restart on-failure myapp

# Restart maximum 3 times
docker run --restart on-failure:3 myapp
```

**When to Use:**
```
✓ Applications that might have transient failures
✓ Services with external dependencies
✓ Want automatic recovery but not infinite retries
✓ Testing/staging environments
```

**Example:**
```bash
# Worker that processes queue
docker run -d \
  --name worker \
  --restart on-failure:5 \
  worker-app

# Scenario 1: Success (exit 0)
# → Does not restart

# Scenario 2: Failure (exit 1)
# → Restarts
# → Fails again
# → Restarts (attempt 2)
# → Succeeds
# → Stays running

# Scenario 3: Persistent failure
# → Fails (attempt 1)
# → Fails (attempt 2)
# → Fails (attempt 3)
# → Fails (attempt 4)
# → Fails (attempt 5)
# → Stops trying (max retries reached)
```

**Use Case - Database Connection:**
```python
# Application with database connection
import sys
import time

max_attempts = 5
attempt = 0

while attempt < max_attempts:
    try:
        db.connect()
        # Success - run application
        app.run()
        sys.exit(0)  # Clean exit, no restart
    except ConnectionError:
        attempt += 1
        if attempt >= max_attempts:
            sys.exit(1)  # Failure, trigger restart
        time.sleep(5)
```

```bash
docker run --restart on-failure:3 myapp

# Database down:
# Attempt 1: Fail → restart
# Attempt 2: Fail → restart
# Attempt 3: Fail → restart
# Attempt 4: Fail → stop (max retries)

# Database comes up:
# Attempt 1: Success → keep running
```

---

### 3. always

**Behavior:**
```
Always restart the container if it stops
Restarts even after daemon restart
Restarts even if manually stopped (until rm)
```

**Usage:**
```bash
docker run --restart always myapp
```

**When to Use:**
```
✓ Critical production services
✓ Must always be running
✓ System services
✓ Infrastructure components
```

**Example:**
```bash
# Load balancer (must always run)
docker run -d \
  --name lb \
  --restart always \
  -p 80:80 \
  nginx

# Scenarios:
# 1. Container crashes → Restarts automatically
# 2. Server reboots → Restarts after Docker daemon starts
# 3. Manually stopped → Restarts immediately
# 4. Only stops when: docker rm -f lb
```

**Real-World Example:**
```bash
# Production API server
docker run -d \
  --name production-api \
  --restart always \
  -p 443:443 \
  --memory 2g \
  api:v1.0

# High availability ensured:
# - Process crash → Restart
# - OOM kill → Restart
# - Server reboot → Restart
# - Manual stop → Restart
# - Only removed when explicitly deleted
```

---

### 4. unless-stopped

**Behavior:**
```
Restart unless manually stopped
Does NOT restart after daemon restart if was manually stopped
Like 'always' but respects manual stops
```

**Usage:**
```bash
docker run --restart unless-stopped myapp
```

**When to Use:**
```
✓ Production services
✓ Want auto-restart for crashes
✓ But respect manual maintenance stops
✓ Most common for production
```

**Behavior Comparison:**

```bash
docker run --restart unless-stopped myapp

# Scenario 1: Container crashes
# → Restarts automatically ✓

# Scenario 2: OOM killed
# → Restarts automatically ✓

# Scenario 3: Manually stopped
docker stop myapp
# → Stays stopped ✓
# → Server reboot → Still stopped ✓

# Scenario 4: Auto-stopped by Docker
# → Restarts automatically ✓
```

**vs always:**
```bash
docker run --restart always myapp

# Manually stopped
docker stop myapp
# → Restarts immediately

# Server reboots
# → Starts again

# Difference:
# always = Ignores manual stops
# unless-stopped = Respects manual stops
```

---

**Decision Matrix:**

```
┌─────────────────────────────────────────────────────┐
│ Should container auto-restart?                      │
│                                                      │
│ No (one-off task)                                   │
│   → --restart no                                    │
│                                                      │
│ Yes, but limited attempts                           │
│   → --restart on-failure:N                          │
│                                                      │
│ Yes, always running, ignore manual stops            │
│   → --restart always                                │
│                                                      │
│ Yes, but respect manual maintenance                 │
│   → --restart unless-stopped                        │
└─────────────────────────────────────────────────────┘
```

---

**Production Architecture Example:**

```bash
# Core services (must always run)
docker run -d --restart always --name traefik traefik
docker run -d --restart always --name consul consul

# Application services (respect maintenance)
docker run -d --restart unless-stopped --name api api:v1
docker run -d --restart unless-stopped --name web web:v1
docker run -d --restart unless-stopped --name worker worker:v1

# Database (respect manual stops for backups)
docker run -d --restart unless-stopped --name postgres postgres

# Batch jobs (run once)
docker run --restart no --name migration migration:latest

# Monitoring (retry on failure)
docker run -d --restart on-failure:5 --name monitor monitor:latest
```

---

**Restart Policy Changes:**

```bash
# Change restart policy of running container
docker update --restart unless-stopped myapp

# View current restart policy
docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' myapp
```

---

**Summary Table:**

| Policy | Crash | Manual Stop | Daemon Restart | Use Case |
|--------|-------|-------------|----------------|----------|
| no | No | - | No | One-off tasks |
| on-failure | Yes (limited) | - | No | Retry on failure |
| always | Yes | Yes | Yes | Critical services |
| unless-stopped | Yes | No | No | Production apps |

</details>

### Question 3: How would you perform a zero-downtime deployment using containers?

<details>
<summary><strong>View Answer</strong></summary>

**Requirement:**
```
Deploy new version with:
✓ Zero downtime
✓ Zero dropped requests
✓ Rollback capability
✓ Health verification
```

**Strategy: Rolling Update with Load Balancer**

---

**Architecture:**

```
                Load Balancer (nginx/traefik)
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
    Container 1     Container 2     Container 3
       (old)           (old)           (old)
```

**Deployment Process:**

**Step 1: Start New Container**
```bash
# Current: v1.0 running
docker ps
# web-1, web-2, web-3 running (v1.0)

# Start new version (not in load balancer yet)
docker run -d \
  --name web-4 \
  --network app-network \
  -e INSTANCE_ID=4 \
  myapp:v1.1

# Health check
while ! curl -f http://web-4:8080/health; do
    echo "Waiting for web-4 health..."
    sleep 2
done
echo "web-4 is healthy"
```

**Step 2: Add to Load Balancer**
```bash
# Update load balancer config
cat > /etc/nginx/conf.d/upstream.conf <<EOF
upstream backend {
    server web-1:8080;
    server web-2:8080;
    server web-3:8080;
    server web-4:8080;  # New container added
}
EOF

# Reload nginx (graceful)
docker exec lb nginx -s reload

# Now traffic split 4 ways
# web-4 receives 25% of traffic
```

**Step 3: Monitor New Container**
```bash
# Monitor for 5 minutes
for i in {1..60}; do
    STATUS=$(curl -s http://web-4:8080/health)
    ERRORS=$(docker logs web-4 2>&1 | grep -c ERROR)
    
    if [ "$ERRORS" -gt "10" ]; then
        echo "Too many errors, rolling back!"
        # Remove from LB
        docker exec lb nginx -s reload  # Remove web-4
        docker stop web-4
        docker rm web-4
        exit 1
    fi
    
    echo "Minute $i: Status=$STATUS, Errors=$ERRORS"
    sleep 60
done

echo "web-4 stable, continuing deployment"
```

**Step 4: Remove Old Container**
```bash
# Remove web-1 from load balancer
cat > /etc/nginx/conf.d/upstream.conf <<EOF
upstream backend {
    server web-2:8080;
    server web-3:8080;
    server web-4:8080;
}
EOF

docker exec lb nginx -s reload

# Wait for existing connections to drain
sleep 30

# Stop old container
docker stop web-1

# Keep for rollback
# Don't remove yet
```

**Step 5: Repeat for Remaining Containers**
```bash
# Start web-5 (v1.1)
docker run -d --name web-5 --network app-network myapp:v1.1
# Health check, add to LB, monitor
# Remove web-2 from LB, stop

# Start web-6 (v1.1)
docker run -d --name web-6 --network app-network myapp:v1.1
# Health check, add to LB, monitor
# Remove web-3 from LB, stop

# Final state:
# web-4, web-5, web-6 running (v1.1)
# web-1, web-2, web-3 stopped (v1.0) - kept for rollback
```

---

**Complete Automated Script:**

```bash
#!/bin/bash
set -e

OLD_VERSION="v1.0"
NEW_VERSION="v1.1"
CONTAINERS=("web-1" "web-2" "web-3")
HEALTH_ENDPOINT="/health"
DRAIN_TIME=30

deploy_container() {
    local old_container=$1
    local new_id=$2
    local new_container="web-${new_id}"
    
    echo "Deploying ${new_container}..."
    
    # Start new container
    docker run -d \
        --name ${new_container} \
        --network app-network \
        -e INSTANCE_ID=${new_id} \
        myapp:${NEW_VERSION}
    
    # Wait for health
    echo "Waiting for ${new_container} to be healthy..."
    max_attempts=30
    attempt=0
    while [ $attempt -lt $max_attempts ]; do
        if curl -sf http://${new_container}:8080${HEALTH_ENDPOINT}; then
            echo "${new_container} is healthy!"
            break
        fi
        attempt=$((attempt + 1))
        sleep 2
    done
    
    if [ $attempt -eq $max_attempts ]; then
        echo "Health check failed for ${new_container}"
        docker stop ${new_container}
        docker rm ${new_container}
        return 1
    fi
    
    # Add to load balancer
    echo "Adding ${new_container} to load balancer..."
    # (nginx config update)
    docker exec lb nginx -s reload
    
    # Monitor for stability (2 minutes)
    echo "Monitoring ${new_container} stability..."
    for i in {1..12}; do
        errors=$(docker logs ${new_container} 2>&1 | grep -c "ERROR" || true)
        if [ $errors -gt 10 ]; then
            echo "Too many errors in ${new_container}, rolling back!"
            # Remove from LB
            docker exec lb nginx -s reload
            docker stop ${new_container}
            docker rm ${new_container}
            return 1
        fi
        sleep 10
    done
    
    # Remove old container from LB
    echo "Removing ${old_container} from load balancer..."
    # (nginx config update)
    docker exec lb nginx -s reload
    
    # Drain connections
    echo "Draining connections from ${old_container}..."
    sleep ${DRAIN_TIME}
    
    # Stop old container
    echo "Stopping ${old_container}..."
    docker stop ${old_container}
    
    echo "${new_container} deployed successfully!"
    return 0
}

# Main deployment
echo "Starting zero-downtime deployment..."
echo "Old version: ${OLD_VERSION}"
echo "New version: ${NEW_VERSION}"

new_id=4
for container in "${CONTAINERS[@]}"; do
    if ! deploy_container $container $new_id; then
        echo "Deployment failed at ${container}"
        echo "Rollback required!"
        exit 1
    fi
    new_id=$((new_id + 1))
done

echo "Deployment complete!"
echo "Old containers stopped (kept for rollback): ${CONTAINERS[@]}"
echo "New containers running: web-4, web-5, web-6"
```

---

**Rollback Procedure:**

```bash
#!/bin/bash
# Quick rollback to v1.0

echo "Rolling back to ${OLD_VERSION}..."

# Start old containers
docker start web-1 web-2 web-3

# Wait for health
for c in web-1 web-2 web-3; do
    while ! curl -sf http://${c}:8080/health; do
        sleep 2
    done
done

# Add to LB
# (Update nginx config to include web-1, web-2, web-3)
docker exec lb nginx -s reload

# Remove new containers from LB
# (Update nginx config to remove web-4, web-5, web-6)
docker exec lb nginx -s reload

# Drain connections from new
sleep 30

# Stop new containers
docker stop web-4 web-5 web-6
docker rm web-4 web-5 web-6

echo "Rollback complete!"
```

---

**Key Points:**

```
1. Health Checks
   - Verify new container before adding to LB
   - Monitor for errors after adding
   - Automatic rollback on failures

2. Connection Draining
   - Wait for existing connections to finish
   - Don't drop active requests
   - Configurable drain time

3. Gradual Rollout
   - One container at a time
   - Verify each step
   - Stop on first error

4. Rollback Ready
   - Keep old containers stopped
   - Can restart quickly
   - No rebuild needed

5. Zero Downtime
   - Always N containers running
   - Load balancer handles routing
   - Transparent to users
```

**Timeline:**
```
t=0:    3 containers (v1.0) running
t=1:    Start web-4 (v1.1)
t=2:    4 containers running (3 old, 1 new)
t=3:    Stop web-1
t=4:    3 containers running (2 old, 1 new)
t=5:    Start web-5 (v1.1)
t=6:    4 containers running (2 old, 2 new)
t=7:    Stop web-2
t=8:    3 containers running (1 old, 2 new)
t=9:    Start web-6 (v1.1)
t=10:   4 containers running (1 old, 3 new)
t=11:   Stop web-3
t=12:   3 containers running (all new)

At every point: 3+ containers serving traffic
Zero downtime achieved ✓
```

</details>

</details>

---

[← Previous: 3.3 CMD vs ENTRYPOINT](../03-docker-images/03-cmd-vs-entrypoint.md) | [Next: 4.2 Container Operations →](02-container-operations.md)