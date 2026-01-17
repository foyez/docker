# 4.2 Container Operations

Managing and interacting with running containers: executing commands, viewing logs, inspecting, monitoring resources, and more.

---

## Executing Commands in Containers

### docker exec

**Purpose:** Execute a command in a running container.

**Syntax:**
```bash
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

**Common Use Cases:**

**1. Interactive Shell:**
```bash
# Bash shell
docker exec -it my-container bash

# Alpine uses sh (no bash)
docker exec -it my-container sh

# As specific user
docker exec -it -u postgres my-container bash

# With environment variables
docker exec -it -e DEBUG=1 my-container bash
```

**2. One-off Commands:**
```bash
# Check logs inside container
docker exec my-container cat /var/log/app.log

# Check process list
docker exec my-container ps aux

# Check disk usage
docker exec my-container df -h

# Network diagnostics
docker exec my-container ping google.com
```

**3. Database Operations:**
```bash
# PostgreSQL
docker exec -it postgres psql -U postgres

# Execute SQL
docker exec postgres psql -U postgres -c "SELECT * FROM users;"

# MySQL
docker exec -it mysql mysql -u root -p

# MongoDB
docker exec -it mongo mongosh

# Redis
docker exec -it redis redis-cli
```

**4. Application Management:**
```bash
# Django migrations
docker exec web python manage.py migrate

# Clear cache
docker exec web python manage.py clear_cache

# Create superuser
docker exec -it web python manage.py createsuperuser

# Node.js REPL
docker exec -it node-app node
```

**Options:**
```bash
-i, --interactive    # Keep STDIN open
-t, --tty           # Allocate pseudo-TTY
-d, --detach        # Run in background
-e, --env           # Set environment variables
-u, --user          # User to run as
-w, --workdir       # Working directory
--privileged        # Give extended privileges
```

---

### docker attach

**Purpose:** Attach to a running container's STDIN, STDOUT, and STDERR.

**Syntax:**
```bash
docker attach [OPTIONS] CONTAINER
```

**Examples:**
```bash
# Attach to container
docker attach my-container

# Attach with no STDIN
docker attach --no-stdin my-container

# Attach with sig-proxy disabled
docker attach --sig-proxy=false my-container
```

**Difference from exec:**
```
docker exec:
- Creates NEW process in container
- Can run any command
- Multiple sessions possible
- Doesn't affect container if you exit

docker attach:
- Attaches to MAIN process (PID 1)
- Can't run different command
- Only one session
- Stopping attached session stops container
```

**Use Cases:**
```bash
# View real-time output of main process
docker attach my-app

# Debug container startup
docker run -it ubuntu bash
# Later: docker attach <container>

# Interactive applications
docker attach interactive-game
```

**Detaching Without Stopping:**
```bash
# Press: Ctrl+P, Ctrl+Q (sequence)
# Container keeps running
```

---

## Viewing Logs

### docker logs

**Purpose:** Fetch logs from a container.

**Syntax:**
```bash
docker logs [OPTIONS] CONTAINER
```

**Basic Usage:**
```bash
# View all logs
docker logs my-container

# Follow logs (like tail -f)
docker logs -f my-container

# Last 100 lines
docker logs --tail 100 my-container

# Logs since timestamp
docker logs --since 2024-01-17T10:00:00 my-container

# Logs in last hour
docker logs --since 1h my-container

# Logs until timestamp
docker logs --until 2024-01-17T11:00:00 my-container
```

**Advanced Options:**
```bash
# Follow with timestamps
docker logs -f --timestamps my-container

# Specific time range
docker logs --since "2024-01-17T09:00:00" \
            --until "2024-01-17T10:00:00" \
            my-container

# Last 50 lines and follow
docker logs --tail 50 -f my-container

# Show timestamps with details
docker logs -t --details my-container
```

**Filtering Logs:**
```bash
# Search for errors
docker logs my-container 2>&1 | grep ERROR

# Count errors
docker logs my-container 2>&1 | grep -c ERROR

# Filter by date
docker logs my-container 2>&1 | grep "2024-01-17"

# Multiple filters
docker logs my-container 2>&1 | grep -E "ERROR|WARN"
```

**Real-World Examples:**

**1. Debugging Application:**
```bash
# Follow logs for debugging
docker logs -f --tail 100 my-app

# Look for specific error
docker logs my-app 2>&1 | grep "DatabaseError"

# Check startup
docker logs my-app | head -50
```

**2. Performance Analysis:**
```bash
# Find slow requests
docker logs nginx | grep "request_time" | awk '{if ($10 > 1.0) print}'

# Count requests per minute
docker logs nginx | grep "$(date +%Y-%m-%d\ %H:%M)" | wc -l

# Top error codes
docker logs nginx | grep "HTTP/" | awk '{print $9}' | sort | uniq -c | sort -rn
```

**3. Security Monitoring:**
```bash
# Failed login attempts
docker logs auth-service | grep "Failed login"

# Suspicious IPs
docker logs web | grep "403" | awk '{print $1}' | sort | uniq -c | sort -rn

# Access patterns
docker logs web | grep "POST /admin" | tail -20
```

---

## Inspecting Containers

### docker inspect

**Purpose:** Return detailed information about containers.

**Syntax:**
```bash
docker inspect [OPTIONS] CONTAINER [CONTAINER...]
```

**Basic Usage:**
```bash
# Full JSON output
docker inspect my-container

# Pretty print
docker inspect my-container | jq

# Specific field
docker inspect -f '{{.State.Status}}' my-container
```

**Common Inspections:**

**1. Network Information:**
```bash
# IP address
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-container

# All network settings
docker inspect -f '{{json .NetworkSettings}}' my-container | jq

# Port mappings
docker inspect -f '{{json .NetworkSettings.Ports}}' my-container | jq

# Gateway
docker inspect -f '{{.NetworkSettings.Gateway}}' my-container
```

**2. State Information:**
```bash
# Container state
docker inspect -f '{{.State.Status}}' my-container

# Running status
docker inspect -f '{{.State.Running}}' my-container

# PID
docker inspect -f '{{.State.Pid}}' my-container

# Started at
docker inspect -f '{{.State.StartedAt}}' my-container

# Exit code
docker inspect -f '{{.State.ExitCode}}' my-container
```

**3. Configuration:**
```bash
# Environment variables
docker inspect -f '{{json .Config.Env}}' my-container | jq

# Command
docker inspect -f '{{.Config.Cmd}}' my-container

# Image
docker inspect -f '{{.Config.Image}}' my-container

# Working directory
docker inspect -f '{{.Config.WorkingDir}}' my-container
```

**4. Mounts and Volumes:**
```bash
# All mounts
docker inspect -f '{{json .Mounts}}' my-container | jq

# Source and destination
docker inspect -f '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' my-container

# Volume names
docker inspect -f '{{range .Mounts}}{{.Name}}{{"\n"}}{{end}}' my-container
```

**5. Resource Limits:**
```bash
# Memory limit
docker inspect -f '{{.HostConfig.Memory}}' my-container

# CPU shares
docker inspect -f '{{.HostConfig.CpuShares}}' my-container

# CPU quota
docker inspect -f '{{.HostConfig.CpuQuota}}' my-container
```

**Real-World Debugging:**

```bash
# Container won't start - check exit code
docker inspect -f '{{.State.ExitCode}}' failed-container
docker inspect -f '{{.State.Error}}' failed-container

# Network issues - verify IP and ports
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-app
docker inspect -f '{{json .NetworkSettings.Ports}}' my-app | jq

# Volume missing - check mounts
docker inspect -f '{{json .Mounts}}' my-app | jq

# Environment variable issue
docker inspect -f '{{json .Config.Env}}' my-app | jq | grep DATABASE_URL
```

---

## Monitoring Resources

### docker stats

**Purpose:** Display live resource usage statistics.

**Syntax:**
```bash
docker stats [OPTIONS] [CONTAINER...]
```

**Basic Usage:**
```bash
# All containers
docker stats

# Specific containers
docker stats web api database

# No streaming (one-time)
docker stats --no-stream

# Format output
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

**Output Fields:**
```
CONTAINER ID   # Container ID
NAME          # Container name
CPU %         # CPU percentage
MEM USAGE     # Memory usage / limit
MEM %         # Memory percentage
NET I/O       # Network I/O
BLOCK I/O     # Disk I/O
PIDS          # Number of processes
```

**Custom Formatting:**
```bash
# CPU and memory only
docker stats --format "{{.Name}}: CPU={{.CPUPerc}} MEM={{.MemPerc}}"

# JSON output
docker stats --no-stream --format '{{json .}}'

# Table format
docker stats --format "table {{.Name}}\t{{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

**Monitoring Script:**
```bash
#!/bin/bash
# Monitor high resource usage

while true; do
    docker stats --no-stream --format \
        "{{.Name}}\t{{.CPUPerc}}\t{{.MemPerc}}" | \
    while read name cpu mem; do
        cpu_val=${cpu%\%}
        mem_val=${mem%\%}
        
        if (( $(echo "$cpu_val > 80" | bc -l) )); then
            echo "⚠️  HIGH CPU: $name using $cpu CPU"
        fi
        
        if (( $(echo "$mem_val > 80" | bc -l) )); then
            echo "⚠️  HIGH MEMORY: $name using $mem memory"
        fi
    done
    
    sleep 10
done
```

---

### docker top

**Purpose:** Display running processes in a container.

**Syntax:**
```bash
docker top CONTAINER [ps OPTIONS]
```

**Examples:**
```bash
# Default output
docker top my-container

# With specific ps options
docker top my-container aux

# Show threads
docker top my-container -eLf
```

**Output:**
```
UID    PID    PPID   C   STIME   TTY   TIME       CMD
root   1234   1200   0   10:00   ?     00:00:01   nginx: master
nginx  1235   1234   0   10:00   ?     00:00:05   nginx: worker
```

---

## Container Information

### docker ps

**Purpose:** List containers.

**Syntax:**
```bash
docker ps [OPTIONS]
```

**Common Options:**
```bash
# Running containers
docker ps

# All containers (including stopped)
docker ps -a

# Latest created
docker ps -l

# Last N containers
docker ps -n 5

# Quiet (IDs only)
docker ps -q

# Specific format
docker ps --format "{{.Names}}: {{.Status}}"

# Filter by status
docker ps -a --filter "status=exited"

# Filter by name
docker ps --filter "name=web"

# Show sizes
docker ps -s
```

**Filtering:**
```bash
# By status
docker ps -a --filter "status=running"
docker ps -a --filter "status=exited"
docker ps -a --filter "status=paused"

# By name pattern
docker ps --filter "name=web*"

# By label
docker ps --filter "label=env=production"

# By network
docker ps --filter "network=my-network"

# By volume
docker ps --filter "volume=my-volume"

# Multiple filters
docker ps --filter "status=running" --filter "name=web*"
```

**Custom Output:**
```bash
# Custom columns
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# JSON output
docker ps --format '{{json .}}'

# Just names
docker ps --format '{{.Names}}'

# Names and IPs
docker ps --format '{{.Names}}: {{.Networks}}'
```

---

## Copying Files

### docker cp

**Purpose:** Copy files/folders between container and host.

**Syntax:**
```bash
# Container to host
docker cp CONTAINER:SRC_PATH DEST_PATH

# Host to container
docker cp SRC_PATH CONTAINER:DEST_PATH
```

**Examples:**

**From Container to Host:**
```bash
# Copy file
docker cp my-container:/app/config.json ./config.json

# Copy directory
docker cp my-container:/var/log/app ./logs

# Copy with different name
docker cp my-container:/app/data.db ./backup.db

# Copy multiple files
docker cp my-container:/etc/nginx/nginx.conf .
docker cp my-container:/etc/nginx/sites-enabled .
```

**From Host to Container:**
```bash
# Copy file
docker cp ./config.json my-container:/app/config.json

# Copy directory
docker cp ./dist my-container:/app/dist

# Copy to different location
docker cp ./backup.sql my-container:/tmp/restore.sql
```

**Real-World Use Cases:**

**1. Backup Database:**
```bash
# Export database dump
docker exec postgres pg_dump -U postgres mydb > dump.sql

# Copy from container
docker cp postgres:/tmp/dump.sql ./backups/dump.sql
```

**2. Update Configuration:**
```bash
# Edit config locally
vim nginx.conf

# Copy to container
docker cp nginx.conf my-nginx:/etc/nginx/nginx.conf

# Reload nginx
docker exec my-nginx nginx -s reload
```

**3. Debugging:**
```bash
# Copy logs for analysis
docker cp my-app:/var/log/app ./logs

# Copy core dump
docker cp crashed-app:/core ./core.dump
```

**4. Development Workflow:**
```bash
# Copy built assets to container
docker cp ./dist/bundle.js web:/app/static/bundle.js

# Extract generated files
docker cp web:/app/generated ./generated
```

---

## Port Management

### docker port

**Purpose:** List port mappings for a container.

**Syntax:**
```bash
docker port CONTAINER [PRIVATE_PORT[/PROTO]]
```

**Examples:**
```bash
# All port mappings
docker port my-container

# Specific port
docker port my-container 80

# Specific port and protocol
docker port my-container 80/tcp
```

**Output:**
```bash
docker port web

80/tcp -> 0.0.0.0:8080
80/tcp -> :::8080
443/tcp -> 0.0.0.0:8443
```

**Finding Exposed Ports:**
```bash
# Get host port for container port 80
HOST_PORT=$(docker port my-container 80 | cut -d: -f2)
echo "Access at http://localhost:$HOST_PORT"

# All mapped ports
docker inspect -f '{{json .NetworkSettings.Ports}}' my-container | jq
```

---

## Renaming Containers

### docker rename

**Purpose:** Rename a container.

**Syntax:**
```bash
docker rename OLD_NAME NEW_NAME
```

**Examples:**
```bash
# Rename container
docker rename old-web new-web

# Fix naming
docker rename crazy_hermann my-app

# Versioning
docker rename api api-v1
```

---

## Waiting for Containers

### docker wait

**Purpose:** Block until one or more containers stop, then print exit codes.

**Syntax:**
```bash
docker wait CONTAINER [CONTAINER...]
```

**Examples:**
```bash
# Wait for container to exit
docker wait my-container
# Blocks until container stops
# Returns exit code

# Multiple containers
docker wait web api database
# Returns exit codes for all

# Use in scripts
EXIT_CODE=$(docker wait my-container)
if [ $EXIT_CODE -ne 0 ]; then
    echo "Container failed with code $EXIT_CODE"
fi
```

**Use Cases:**
```bash
# Wait for migration to complete
docker run -d --name migrate myapp migrate
docker wait migrate
echo "Migration complete"

# Sequential processing
docker run -d --name step1 processor step1
docker wait step1
docker run -d --name step2 processor step2
docker wait step2
echo "Pipeline complete"
```

---

## Real-World Operational Patterns

### Pattern 1: Health Check and Restart

```bash
#!/bin/bash
# Monitor and restart unhealthy containers

CONTAINER="my-app"
HEALTH_URL="http://localhost:8080/health"

while true; do
    if ! curl -sf $HEALTH_URL > /dev/null; then
        echo "Health check failed, restarting..."
        docker restart $CONTAINER
        sleep 60
    fi
    sleep 30
done
```

### Pattern 2: Log Rotation

```bash
#!/bin/bash
# Rotate container logs

CONTAINER="my-app"
LOG_DIR="/var/log/containers"
MAX_SIZE_MB=100

# Get log file size
LOG_FILE=$(docker inspect -f '{{.LogPath}}' $CONTAINER)
SIZE_MB=$(du -m "$LOG_FILE" | cut -f1)

if [ $SIZE_MB -gt $MAX_SIZE_MB ]; then
    echo "Rotating logs for $CONTAINER"
    
    # Copy log
    cp "$LOG_FILE" "$LOG_DIR/${CONTAINER}-$(date +%Y%m%d).log"
    
    # Truncate
    truncate -s 0 "$LOG_FILE"
    
    # Signal container to reopen log files
    docker kill --signal=USR1 $CONTAINER
fi
```

### Pattern 3: Resource Monitoring and Alerts

```bash
#!/bin/bash
# Alert on high resource usage

THRESHOLD_CPU=80
THRESHOLD_MEM=80

docker stats --no-stream --format \
    "{{.Name}}\t{{.CPUPerc}}\t{{.MemPerc}}" | \
while read name cpu mem; do
    cpu_val=${cpu%\%}
    mem_val=${mem%\%}
    
    if (( $(echo "$cpu_val > $THRESHOLD_CPU" | bc -l) )); then
        # Send alert
        curl -X POST https://alerts.company.com/webhook \
            -d "{\"container\": \"$name\", \"metric\": \"CPU\", \"value\": $cpu}"
    fi
    
    if (( $(echo "$mem_val > $THRESHOLD_MEM" | bc -l) )); then
        # Send alert
        curl -X POST https://alerts.company.com/webhook \
            -d "{\"container\": \"$name\", \"metric\": \"Memory\", \"value\": $mem}"
    fi
done
```

### Pattern 4: Backup Automation

```bash
#!/bin/bash
# Automated database backup

CONTAINER="postgres"
BACKUP_DIR="/backups"
RETENTION_DAYS=7

# Create backup
BACKUP_FILE="${BACKUP_DIR}/backup-$(date +%Y%m%d-%H%M%S).sql"
docker exec $CONTAINER pg_dump -U postgres mydb > "$BACKUP_FILE"

# Compress
gzip "$BACKUP_FILE"

# Remove old backups
find "$BACKUP_DIR" -name "backup-*.sql.gz" -mtime +$RETENTION_DAYS -delete

# Verify backup
if [ -f "${BACKUP_FILE}.gz" ]; then
    echo "Backup successful: ${BACKUP_FILE}.gz"
else
    echo "Backup failed!"
    exit 1
fi
```

### Pattern 5: Container Update with Verification

```bash
#!/bin/bash
# Update container with health verification

CONTAINER="web"
IMAGE="myapp:latest"
HEALTH_URL="http://localhost:8080/health"

# Pull new image
docker pull $IMAGE

# Stop old container
docker stop $CONTAINER

# Rename old container (for rollback)
docker rename $CONTAINER "${CONTAINER}-old"

# Start new container
docker run -d \
    --name $CONTAINER \
    --network app-network \
    -p 8080:8080 \
    $IMAGE

# Wait for health
MAX_ATTEMPTS=30
ATTEMPT=0

while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
    if curl -sf $HEALTH_URL > /dev/null; then
        echo "New container is healthy!"
        
        # Remove old container
        docker rm "${CONTAINER}-old"
        
        echo "Update successful!"
        exit 0
    fi
    
    ATTEMPT=$((ATTEMPT + 1))
    sleep 2
done

# Health check failed - rollback
echo "Health check failed, rolling back..."
docker stop $CONTAINER
docker rm $CONTAINER
docker rename "${CONTAINER}-old" $CONTAINER
docker start $CONTAINER

echo "Rollback complete"
exit 1
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The __________ command executes a new process in a running container.
2. Use __________ to view real-time resource usage of containers.
3. The __________ command copies files between container and host.
4. __________ displays running processes inside a container.
5. Use __________ to attach to a running container's main process.
6. The __________ option follows log output in real-time.

### True/False

1. ⬜ docker exec creates a new process in the container
2. ⬜ docker attach can run any command in the container
3. ⬜ docker logs works on both running and stopped containers
4. ⬜ docker stats shows historical resource usage
5. ⬜ docker cp only works with running containers
6. ⬜ You can execute multiple docker exec sessions simultaneously
7. ⬜ docker top shows processes from the host's perspective

### Multiple Choice

1. What's the difference between docker exec and docker attach?
   - A) They are the same
   - B) exec creates new process, attach connects to main process
   - C) attach creates new process, exec connects to main process
   - D) exec is for files, attach is for commands

2. How do you follow logs in real-time?
   - A) docker logs container
   - B) docker logs -f container
   - C) docker logs --follow container
   - D) Both B and C

3. What does docker stats --no-stream do?
   - A) Disables network monitoring
   - B) Shows stats once instead of continuously
   - C) Hides streaming containers
   - D) Stops all containers

4. How do you copy a file from container to host?
   - A) docker cp host:/file container:/file
   - B) docker cp container:/file ./file
   - C) docker copy container:/file ./file
   - D) docker export container:/file

5. What does docker wait do?
   - A) Waits for container to start
   - B) Waits for container to stop and returns exit code
   - C) Pauses container execution
   - D) Delays container startup

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. docker exec
2. docker stats
3. docker cp
4. docker top
5. docker attach
6. -f (or --follow)

**True/False:**
1. ✅ True - exec creates new process in existing container
2. ❌ False - attach connects to main process (PID 1), cannot run different command
3. ✅ True - logs are persisted, available for stopped containers
4. ❌ False - stats shows live/current usage, not historical
5. ❌ False - cp works with both running and stopped containers
6. ✅ True - can have multiple exec sessions simultaneously
7. ✅ True - shows actual PIDs from host, not container namespace

**Multiple Choice:**
1. **B** - exec creates new process, attach connects to main process
2. **D** - Both -f and --follow work
3. **B** - Shows stats once instead of continuously
4. **B** - docker cp container:/file ./file
5. **B** - Waits for container to stop and returns exit code

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: What's the difference between docker exec and docker attach, and when would you use each?

<details>
<summary><strong>View Answer</strong></summary>

**Core Difference:**

**docker exec:**
- Creates NEW process in container
- Can run any command
- Multiple sessions possible
- Exiting doesn't affect container

**docker attach:**
- Attaches to MAIN process (PID 1)
- Cannot run different command
- Only one session at a time
- Exiting may stop container

---

**docker exec - New Process**

**How It Works:**
```
Container has:
PID 1: nginx (main process)

docker exec -it myapp bash

Creates:
PID 15: bash (new process)

You interact with PID 15
PID 1 keeps running
```

**Examples:**
```bash
# Interactive shell
docker exec -it myapp bash
# New bash process created
# Exit bash → container keeps running

# Run command
docker exec myapp ls /app
# ls process runs and exits
# Container unaffected

# Multiple sessions
Terminal 1: docker exec -it myapp bash
Terminal 2: docker exec -it myapp bash
Terminal 3: docker exec -it myapp bash
# All work simultaneously
```

**Use Cases:**
```
✓ Debugging running containers
✓ Running maintenance tasks
✓ Checking logs/files
✓ Database queries
✓ Running scripts
✓ Multiple administrative sessions
```

---

**docker attach - Main Process**

**How It Works:**
```
Container has:
PID 1: bash (interactive shell)

docker attach myapp

Attaches to PID 1
You interact with main process
```

**Examples:**
```bash
# Start interactive container
docker run -it --name myapp ubuntu bash
# You're attached to bash (PID 1)

# Detach without stopping
Press: Ctrl+P, Ctrl+Q
# Container keeps running

# Later, re-attach
docker attach myapp
# Back to same bash session
```

**Use Cases:**
```
✓ Viewing output of main process
✓ Interactive containers
✓ Resuming interactive sessions
✓ Debugging container startup
✓ Applications expecting TTY
```

---

**Real-World Scenarios:**

**Scenario 1: Production Web Server**

```bash
# Container running nginx
docker run -d --name web nginx

# Need to check configuration
docker exec web cat /etc/nginx/nginx.conf  ✓
# New process, safe

# Don't use attach
docker attach web  ✗
# Attaches to nginx
# If you Ctrl+C → stops nginx → downtime!
```

**Scenario 2: Development Database**

```bash
# PostgreSQL running
docker run -d --name postgres postgres

# Need to run queries
docker exec -it postgres psql -U postgres  ✓
# New psql session
# Exit doesn't affect database

# Don't use attach
docker attach postgres  ✗
# Attaches to postgres main process
# Can't run psql
# Stopping it stops database
```

**Scenario 3: Interactive Container**

```bash
# Running Python interpreter
docker run -it --name python python

# Accidentally detached
# Press Ctrl+P, Ctrl+Q

# Resume same session
docker attach python  ✓
# Same Python session
# Same variables, history

# Using exec would start new session
docker exec -it python python  ✗
# New Python interpreter
# Lost previous session state
```

---

**Comparison Table:**

| Feature | docker exec | docker attach |
|---------|-------------|---------------|
| **Process** | Creates new | Connects to PID 1 |
| **Command** | Any command | Cannot change |
| **Sessions** | Multiple | Single |
| **Exit impact** | No impact | May stop container |
| **Use for** | Commands, debugging | Interactive sessions |
| **Production** | Safe | Risky |

---

**Best Practices:**

**Use exec for:**
```bash
# Debugging
docker exec -it web bash

# Maintenance
docker exec web python manage.py migrate

# Queries
docker exec -it db psql -U postgres

# Inspecting
docker exec web cat /var/log/app.log

# Multiple admins
Team member 1: docker exec -it web bash
Team member 2: docker exec -it web bash
# Both can work simultaneously
```

**Use attach for:**
```bash
# Interactive development container
docker run -it --name dev ubuntu bash
# Work, then Ctrl+P, Ctrl+Q to detach
# Later: docker attach dev to resume

# Viewing main process output
docker attach logging-app
# See real-time output
# Ctrl+P, Ctrl+Q to detach

# Interactive applications
docker attach game-server
# Interact with game console
```

**Avoid attach in production:**
```bash
# Never attach to production services
docker attach production-web  ✗

# Risk: Accidentally stopping service
# Risk: Single session only
# Risk: No audit trail

# Use exec instead
docker exec -it production-web bash  ✓
```

</details>

### Question 2: How do you troubleshoot a container that keeps restarting?

<details>
<summary><strong>View Answer</strong></summary>

**Systematic Troubleshooting Approach:**

---

**Step 1: Check Container Status**

```bash
# View container status
docker ps -a --filter "name=my-container"

# Output shows:
# STATUS: Restarting (5) seconds ago
# Exit code in parentheses
```

**Common Exit Codes:**
```
0   - Success (shouldn't restart)
1   - Application error
125 - Docker daemon error
126 - Command cannot be invoked
127 - Command not found
137 - SIGKILL (OOM or killed)
139 - SIGSEGV (segmentation fault)
143 - SIGTERM (graceful termination)
```

---

**Step 2: View Recent Logs**

```bash
# Last 100 lines
docker logs --tail 100 my-container

# With timestamps
docker logs -t --tail 100 my-container

# Since last restart
docker logs --since $(date -d '1 minute ago' -Iseconds) my-container
```

**What to Look For:**
```
✓ Error messages
✓ Stack traces
✓ Failed dependencies
✓ Missing environment variables
✓ Permission errors
✓ Port conflicts
```

---

**Step 3: Inspect Exit Code**

```bash
# Get detailed state
docker inspect my-container | jq '.[0].State'

# Output:
{
  "Status": "restarting",
  "Running": false,
  "Paused": false,
  "Restarting": true,
  "OOMKilled": false,
  "Dead": false,
  "Pid": 0,
  "ExitCode": 1,
  "Error": "",
  "StartedAt": "2024-01-17T10:00:00Z",
  "FinishedAt": "2024-01-17T10:00:05Z"
}
```

**Analyze Exit Code:**
```bash
EXIT_CODE=$(docker inspect -f '{{.State.ExitCode}}' my-container)

case $EXIT_CODE in
    1)
        echo "Application error - check logs"
        ;;
    127)
        echo "Command not found - check CMD/ENTRYPOINT"
        ;;
    137)
        echo "OOM killed or docker kill - check memory"
        ;;
    139)
        echo "Segmentation fault - check app bugs"
        ;;
esac
```

---

**Step 4: Check Resource Limits**

```bash
# Check if OOM killed
docker inspect -f '{{.State.OOMKilled}}' my-container
# true = killed due to memory

# Check memory limit
docker inspect -f '{{.HostConfig.Memory}}' my-container

# Check actual memory usage (before crash)
docker stats --no-stream my-container
```

**OOM Scenario:**
```bash
# Container with 512MB limit
docker inspect -f '{{.HostConfig.Memory}}' my-container
# Output: 536870912 (512MB)

# Logs show:
docker logs my-container
# Fatal error: Out of memory
# Killed

# Solution: Increase memory
docker update --memory 1g my-container
# or recreate with higher limit
```

---

**Step 5: Test Without Restart Policy**

```bash
# Stop the container
docker stop my-container

# Remove it
docker rm my-container

# Run without restart policy
docker run --name my-container \
    --rm \
    my-image

# Watch for immediate errors
# Container will stay stopped on failure
# Allows you to see error messages
```

---

**Step 6: Check Dependencies**

```bash
# Inspect network connections
docker inspect -f '{{json .NetworkSettings.Networks}}' my-container | jq

# Test database connection
docker exec my-container nc -zv database 5432

# Check DNS resolution
docker exec my-container nslookup database

# Verify environment variables
docker inspect -f '{{json .Config.Env}}' my-container | jq
```

---

**Real-World Examples:**

**Example 1: Missing Environment Variable**

```bash
# Container keeps restarting
docker logs web
# Error: DATABASE_URL not set

# Check environment
docker inspect -f '{{json .Config.Env}}' web | jq
# DATABASE_URL not in list

# Solution
docker stop web
docker rm web
docker run -d --name web \
    -e DATABASE_URL=postgres://db:5432/myapp \
    my-image
```

**Example 2: Database Not Ready**

```bash
# Application logs
docker logs api
# Error: could not connect to database
# Retrying...
# Error: could not connect to database

# Database not ready yet
docker logs postgres
# PostgreSQL is starting up
# LOG: database system is ready

# Solution: Add health check and depends_on
```

**docker-compose.yml:**
```yaml
services:
  postgres:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    image: my-api
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://postgres@postgres:5432/myapp
```

**Example 3: Port Already in Use**

```bash
# Container restarts immediately
docker logs web
# Error: bind: address already in use

# Check what's using port
lsof -i :8080
# Another container or process using port

# Solution: Use different port or stop conflicting service
docker run -d --name web -p 8081:8080 my-image
```

**Example 4: File Permission Error**

```bash
# Container crashes
docker logs app
# PermissionError: [Errno 13] Permission denied: '/data/file.txt'

# Check user
docker inspect -f '{{.Config.User}}' app
# User: appuser (UID 1000)

# Volume mounted with wrong permissions
docker inspect -f '{{json .Mounts}}' app | jq
# Source: /host/data (owned by root)

# Solution: Fix permissions
sudo chown -R 1000:1000 /host/data

# Or run as root (not recommended)
docker run --user root ...
```

**Example 5: OOM Killed**

```bash
# Check if OOM
docker inspect -f '{{.State.OOMKilled}}' app
# true

# Logs show memory issues
docker logs app
# Warning: Memory usage high
# Killed

# Check limit
docker inspect -f '{{.HostConfig.Memory}}' app
# 268435456 (256MB)

# Solution: Increase memory
docker stop app
docker rm app
docker run -d --name app --memory 1g my-image

# Or fix memory leak in application
```

---

**Troubleshooting Checklist:**

```bash
#!/bin/bash
# Container restart troubleshooter

CONTAINER="$1"

echo "=== Container Status ==="
docker ps -a --filter "name=$CONTAINER"

echo "=== Exit Code ==="
docker inspect -f '{{.State.ExitCode}}' $CONTAINER

echo "=== OOM Killed? ==="
docker inspect -f '{{.State.OOMKilled}}' $CONTAINER

echo "=== Recent Logs ==="
docker logs --tail 50 $CONTAINER

echo "=== Environment ==="
docker inspect -f '{{json .Config.Env}}' $CONTAINER | jq

echo "=== Network ==="
docker inspect -f '{{json .NetworkSettings.Networks}}' $CONTAINER | jq

echo "=== Mounts ==="
docker inspect -f '{{json .Mounts}}' $CONTAINER | jq

echo "=== Resource Limits ==="
echo "Memory: $(docker inspect -f '{{.HostConfig.Memory}}' $CONTAINER)"
echo "CPU: $(docker inspect -f '{{.HostConfig.CpuQuota}}' $CONTAINER)"

echo "=== Last 5 State Changes ==="
docker events --filter "container=$CONTAINER" --since 5m
```

---

**Prevention Best Practices:**

```dockerfile
# In Dockerfile:

# 1. Add health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD curl -f http://localhost/health || exit 1

# 2. Proper signal handling
# Application should handle SIGTERM gracefully

# 3. Logging to stdout/stderr
# Not to files (for docker logs to work)
```

```bash
# When running:

# 1. Set appropriate restart policy
docker run --restart unless-stopped ...

# 2. Set resource limits
docker run --memory 1g --cpus 2 ...

# 3. Use health checks
docker run --health-cmd="curl -f http://localhost/health" ...

# 4. Proper dependency ordering
# Use docker-compose depends_on with health checks
```

</details>

### Question 3: How do you efficiently gather logs from multiple containers for debugging?

<details>
<summary><strong>View Answer</strong></summary>

**Multiple Approaches:**

---

### Approach 1: Quick View (Development)

```bash
# View logs from all containers
docker-compose logs

# Follow logs from all containers
docker-compose logs -f

# Specific services
docker-compose logs -f web api database

# With timestamps
docker-compose logs -f -t

# Last 100 lines from each
docker-compose logs --tail=100
```

---

### Approach 2: Unified Log Stream

```bash
# Stream logs from multiple containers to stdout
docker logs -f web &
docker logs -f api &
docker logs -f worker &
wait

# Or with labels
for container in $(docker ps -q --filter "label=app=myapp"); do
    docker logs -f $container 2>&1 | sed "s/^/[$container] /" &
done
wait
```

---

### Approach 3: Collect to Files

```bash
#!/bin/bash
# Collect logs from all containers

OUTPUT_DIR="./logs/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTPUT_DIR"

# Get all running containers
for container in $(docker ps --format '{{.Names}}'); do
    echo "Collecting logs from $container..."
    
    # Get logs
    docker logs $container > "$OUTPUT_DIR/${container}.log" 2>&1
    
    # Get container info
    docker inspect $container > "$OUTPUT_DIR/${container}-inspect.json"
    
    # Get stats snapshot
    docker stats --no-stream $container > "$OUTPUT_DIR/${container}-stats.txt"
done

# Create archive
tar -czf "logs-$(date +%Y%m%d-%H%M%S).tar.gz" "$OUTPUT_DIR"

echo "Logs collected in logs-*.tar.gz"
```

---

### Approach 4: Centralized Logging (Production)

**Using Docker Log Driver:**

```bash
# Configure logging driver
docker run -d \
    --log-driver=syslog \
    --log-opt syslog-address=tcp://logserver:514 \
    --log-opt tag="{{.Name}}" \
    my-app
```

**daemon.json configuration:**
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "labels": "app,environment"
  }
}
```

**Using Fluentd:**

```yaml
# docker-compose.yml
version: '3'

services:
  web:
    image: my-web
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: web

  api:
    image: my-api
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: api

  fluentd:
    image: fluent/fluentd
    ports:
      - "24224:24224"
    volumes:
      - ./fluentd/conf:/fluentd/etc
      - ./logs:/fluentd/log
```

**fluentd.conf:**
```
<source>
  @type forward
  port 24224
</source>

<match **>
  @type file
  path /fluentd/log/${tag}
  <buffer tag,time>
    @type file
    timekey 1d
    timekey_wait 10m
    flush_mode interval
    flush_interval 30s
  </buffer>
</match>
```

---

### Approach 5: Search and Filter

```bash
# Search errors across all containers
for container in $(docker ps -q); do
    echo "=== $container ==="
    docker logs $container 2>&1 | grep -i error
done

# Find which container has specific error
for container in $(docker ps --format '{{.Names}}'); do
    if docker logs $container 2>&1 | grep -q "Database connection failed"; then
        echo "Found in: $container"
        docker logs $container 2>&1 | grep -A 5 "Database connection failed"
    fi
done

# Count errors per container
for container in $(docker ps --format '{{.Names}}'); do
    errors=$(docker logs $container 2>&1 | grep -c ERROR)
    echo "$container: $errors errors"
done | sort -t: -k2 -rn
```

---

### Approach 6: Timeline Analysis

```bash
#!/bin/bash
# Create unified timeline from all containers

SINCE="10m"
OUTPUT="timeline.log"

> $OUTPUT  # Clear file

# Collect logs with timestamps
for container in $(docker ps --format '{{.Names}}'); do
    docker logs --timestamps --since $SINCE $container 2>&1 | \
        sed "s/^/[$container] /" >> $OUTPUT
done

# Sort by timestamp
sort $OUTPUT -o $OUTPUT

echo "Timeline created: $OUTPUT"
```

**Output:**
```
[web] 2024-01-17T10:00:01Z GET /api/users
[api] 2024-01-17T10:00:02Z Processing request
[database] 2024-01-17T10:00:03Z SELECT * FROM users
[api] 2024-01-17T10:00:04Z Response sent
[web] 2024-01-17T10:00:05Z 200 OK
```

---

### Approach 7: Advanced Log Analysis

```bash
#!/bin/bash
# Comprehensive log analysis

CONTAINERS=$(docker ps --format '{{.Names}}')
REPORT="log-analysis-$(date +%Y%m%d-%H%M%S).txt"

{
    echo "=== Docker Log Analysis Report ==="
    echo "Generated: $(date)"
    echo ""

    for container in $CONTAINERS; do
        echo "=== Container: $container ==="
        echo ""
        
        echo "Total log lines:"
        docker logs $container 2>&1 | wc -l
        
        echo ""
        echo "Error count:"
        docker logs $container 2>&1 | grep -ci error
        
        echo ""
        echo "Warning count:"
        docker logs $container 2>&1 | grep -ci warn
        
        echo ""
        echo "Top 10 error messages:"
        docker logs $container 2>&1 | grep -i error | \
            sort | uniq -c | sort -rn | head -10
        
        echo ""
        echo "Last 5 errors:"
        docker logs $container 2>&1 | grep -i error | tail -5
        
        echo ""
        echo "Container status:"
        docker inspect -f '{{.State.Status}}' $container
        
        echo ""
        echo "Memory usage:"
        docker stats --no-stream --format \
            "{{.MemUsage}}" $container
        
        echo ""
        echo "====================================="
        echo ""
    done
} > $REPORT

echo "Report saved: $REPORT"
```

---

### Approach 8: Real-time Correlation

```bash
#!/bin/bash
# Correlate logs from multiple services

mkfifo /tmp/web_logs /tmp/api_logs /tmp/db_logs

# Stream logs from each container
docker logs -f web > /tmp/web_logs &
docker logs -f api > /tmp/api_logs &
docker logs -f db > /tmp/db_logs &

# Merge and display with service names
{
    sed 's/^/[WEB] /' < /tmp/web_logs &
    sed 's/^/[API] /' < /tmp/api_logs &
    sed 's/^/[DB]  /' < /tmp/db_logs &
} | while read line; do
    echo "$(date +%H:%M:%S) $line"
done

# Cleanup
rm /tmp/web_logs /tmp/api_logs /tmp/db_logs
```

---

### Production Best Practices:

**1. Structured Logging:**
```python
# Application code
import logging
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'message': record.getMessage(),
            'service': 'web',
            'trace_id': getattr(record, 'trace_id', None)
        })

logging.basicConfig(
    format='%(message)s',
    handlers=[
        logging.StreamHandler()
    ]
)
logger.setFormatter(JSONFormatter())
```

**2. Log Aggregation:**
```yaml
# ELK Stack setup
version: '3'

services:
  elasticsearch:
    image: elasticsearch:7.17.0
    
  logstash:
    image: logstash:7.17.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    
  kibana:
    image: kibana:7.17.0
    ports:
      - "5601:5601"
    
  app:
    image: my-app
    logging:
      driver: gelf
      options:
        gelf-address: "udp://logstash:12201"
```

**3. Log Retention:**
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "compress": "true"
  }
}
```

</details>

</details>

---

[← Previous: 4.1 Container Lifecycle](01-container-lifecycle.md) | [Next: 5.1 Volume Basics →](../05-docker-volumes/01-volume-basics.md)