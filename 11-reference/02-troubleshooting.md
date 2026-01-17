# 11.2 Troubleshooting Guide

Comprehensive guide to diagnosing and fixing common Docker issues.

---

## Container Won't Start

### Symptom: Container Exits Immediately

**Check Exit Code:**
```bash
docker ps -a
# Look for STATUS: Exited (1), Exited (137), etc.

# Get exit code
docker inspect myapp --format='{{.State.ExitCode}}'
```

**Exit Codes:**
```
0   = Success (normal exit)
1   = Application error
125 = Docker daemon error
126 = Command cannot be invoked
127 = Command not found
137 = SIGKILL (out of memory)
139 = SIGSEGV (segmentation fault)
143 = SIGTERM (graceful termination)
```

---

**Check Logs:**
```bash
docker logs myapp

# Last 100 lines
docker logs --tail 100 myapp

# With timestamps
docker logs -t myapp
```

---

**Common Causes:**

**1. Command Not Found:**
```dockerfile
# Bad
CMD ["nod", "server.js"]  # Typo: nod instead of node

# Check
docker run myapp which node
# /usr/local/bin/node

# Fix
CMD ["node", "server.js"]
```

**2. Missing Dependencies:**
```bash
# Check logs
docker logs myapp
# Error: Cannot find module 'express'

# Fix: Install dependencies
docker run myapp npm list express
```

**3. Permission Issues:**
```bash
# Check logs
docker logs myapp
# Error: EACCES: permission denied, open '/app/data.json'

# Check permissions
docker exec myapp ls -la /app
# -rw-r--r-- 1 root root data.json

# Fix: Change ownership
docker run --user node myapp
```

**4. Port Already in Use:**
```bash
# Check logs
Error: listen EADDRINUSE: address already in use :::3000

# Check what's using port
netstat -tlnp | grep 3000

# Fix: Use different port
docker run -p 3001:3000 myapp
```

---

### Symptom: Container Starts Then Crashes

**Monitor Startup:**
```bash
# Follow logs in real-time
docker logs -f myapp

# Check events
docker events --filter container=myapp
```

**Common Causes:**

**1. Application Crash:**
```bash
# Check logs
docker logs myapp
# Uncaught Exception: ...

# Debug
docker run -it --entrypoint /bin/sh myapp
# Run manually: node server.js
```

**2. Out of Memory:**
```bash
# Check exit code
docker inspect myapp --format='{{.State.ExitCode}}'
# 137 (SIGKILL - OOM)

# Check memory limit
docker inspect myapp --format='{{.HostConfig.Memory}}'

# Fix: Increase memory
docker run -m 512m myapp
```

**3. Health Check Failure:**
```bash
# Check health status
docker inspect myapp --format='{{.State.Health.Status}}'

# Check health logs
docker inspect myapp --format='{{json .State.Health}}' | jq

# Debug health check
docker exec myapp /bin/sh -c "curl -f http://localhost/health || exit 1"
```

---

## Network Issues

### Cannot Connect to Container

**Check Container Running:**
```bash
docker ps | grep myapp
```

**Check Port Mapping:**
```bash
docker ps
# PORTS: 0.0.0.0:8080->80/tcp

# Or
docker port myapp
# 80/tcp -> 0.0.0.0:8080
```

**Test Connection:**
```bash
# From host
curl http://localhost:8080

# If fails, check if container is listening
docker exec myapp netstat -tlnp
```

**Fix Wrong Port:**
```bash
# Remove container
docker rm -f myapp

# Run with correct port
docker run -p 8080:80 myapp
```

---

### Containers Cannot Communicate

**Check Network:**
```bash
# List networks
docker network ls

# Inspect network
docker network inspect bridge

# Check if containers on same network
docker inspect myapp --format='{{json .NetworkSettings.Networks}}' | jq
```

**Test Connectivity:**
```bash
# From one container to another
docker exec web ping api

# Check DNS resolution
docker exec web nslookup api

# Check if port is open
docker exec web nc -zv api 3000
```

**Fix: Put on Same Network:**
```bash
# Create custom network
docker network create myapp-network

# Connect containers
docker network connect myapp-network web
docker network connect myapp-network api

# Test
docker exec web ping api
```

---

### DNS Resolution Fails

**Symptom:**
```bash
docker exec myapp ping api
# ping: bad address 'api'
```

**Check Network Type:**
```bash
docker network inspect bridge --format='{{.Driver}}'
# bridge (default) - no DNS

# Custom bridge - has DNS
docker network inspect myapp-network --format='{{.Driver}}'
```

**Fix:**
```bash
# Use custom network (not default bridge)
docker network create myapp-network
docker run --network myapp-network --name api myapi
docker run --network myapp-network --name web myweb

# Now DNS works
docker exec web ping api
```

**Alternative: Use Links (Legacy):**
```bash
docker run --name api myapi
docker run --link api:api --name web myweb

# Can ping via link
docker exec web ping api
```

---

## Volume Issues

### Data Not Persisting

**Check Volume Mount:**
```bash
# Inspect container
docker inspect myapp --format='{{json .Mounts}}' | jq

# Should show:
# "Type": "volume",
# "Name": "mydata",
# "Source": "/var/lib/docker/volumes/mydata/_data",
# "Destination": "/data"
```

**Common Mistakes:**

**1. Using Anonymous Volume:**
```bash
# Bad: Anonymous volume (lost on container removal)
docker run -v /data myapp

# Good: Named volume
docker run -v mydata:/data myapp
```

**2. Wrong Path:**
```bash
# Check where app writes data
docker exec myapp ls -la /app/data
# /app/data not mounted

# Fix mount path
docker run -v mydata:/app/data myapp
```

**3. Volume Not Created:**
```bash
# Check if volume exists
docker volume ls | grep mydata

# Create if missing
docker volume create mydata
```

---

### Permission Denied in Volume

**Symptom:**
```bash
docker logs myapp
# Error: EACCES: permission denied, open '/data/file.txt'
```

**Check Ownership:**
```bash
# Inside container
docker exec myapp ls -la /data
# drwxr-xr-x 2 root root

# Check user running app
docker exec myapp whoami
# node (UID 1000)
```

**Fix: Change Ownership:**
```bash
# Option 1: Fix in Dockerfile
FROM node:18-alpine
RUN mkdir -p /data && chown -R node:node /data
USER node

# Option 2: Fix at runtime
docker run -v mydata:/data myapp sh -c "chown -R node:node /data && node server.js"

# Option 3: Use volume options
docker run -v mydata:/data:rw,uid=1000,gid=1000 myapp
```

---

### Bind Mount Not Working

**Symptom:**
```bash
# Changed files on host, not reflected in container
docker run -v $(pwd)/src:/app/src myapp
```

**Check Mount:**
```bash
docker inspect myapp --format='{{json .Mounts}}' | jq
# "Type": "bind"
# "Source": "/host/path/src"
```

**Common Issues:**

**1. Path Not Absolute:**
```bash
# Bad
docker run -v ./src:/app/src myapp

# Good
docker run -v $(pwd)/src:/app/src myapp
```

**2. File Not Found:**
```bash
# Check source exists
ls -la $(pwd)/src

# Check inside container
docker exec myapp ls -la /app/src
```

**3. Windows Path Issues:**
```bash
# Windows (PowerShell)
docker run -v ${PWD}/src:/app/src myapp

# Windows (CMD)
docker run -v %cd%/src:/app/src myapp
```

---

## Build Issues

### Build Fails

**Check Build Context:**
```bash
# See what's being sent
docker build --no-cache .

# If slow, context too large
# Add .dockerignore
```

**.dockerignore:**
```
node_modules
.git
*.log
*.md
dist
build
```

---

### Layer Caching Not Working

**Symptom:**
```
Step 5/10 : RUN npm install
# Always reinstalls even when package.json unchanged
```

**Fix:**
```dockerfile
# Bad: Copies everything first
COPY . .
RUN npm install

# Good: Copy package.json first
COPY package*.json ./
RUN npm install
COPY . .
```

---

### COPY Command Fails

**Symptom:**
```
COPY failed: stat /var/lib/docker/.../src: no such file or directory
```

**Causes:**

**1. File Not in Build Context:**
```dockerfile
# Bad: File outside build context
COPY /etc/config.json /app/

# Good: File in build context
COPY config.json /app/
```

**2. File in .dockerignore:**
```bash
# Check .dockerignore
cat .dockerignore
# config.json

# Remove from .dockerignore or use different file
```

---

### Out of Disk Space

**Symptom:**
```
failed to register layer: Error processing tar file: write /...: no space left on device
```

**Check Disk Usage:**
```bash
docker system df

# Detailed
docker system df -v
```

**Clean Up:**
```bash
# Remove unused data
docker system prune

# Remove everything
docker system prune -a --volumes

# Specific cleanup
docker image prune -a
docker container prune
docker volume prune
docker network prune
```

---

## Performance Issues

### Container Running Slow

**Check Resource Usage:**
```bash
# Real-time stats
docker stats myapp

# Look for:
# - High CPU %
# - High memory usage
# - High block I/O
```

**Check Resource Limits:**
```bash
docker inspect myapp --format='{{.HostConfig.Memory}}'
docker inspect myapp --format='{{.HostConfig.CpuQuota}}'
```

**Increase Limits:**
```bash
docker run -m 1g --cpus 2 myapp
```

---

### Slow Network

**Test Network Speed:**
```bash
# Install iperf3
docker exec web apk add iperf3
docker exec api apk add iperf3

# Server
docker exec api iperf3 -s

# Client
docker exec web iperf3 -c api
```

**Check Network Driver:**
```bash
docker network inspect myapp-network --format='{{.Driver}}'

# Bridge is slower than host for same-host containers
# Consider host networking for performance-critical apps
docker run --network host myapp
```

---

### High Memory Usage

**Check Memory Leak:**
```bash
# Monitor over time
watch -n 5 docker stats myapp

# Node.js heap snapshot
docker exec myapp kill -USR2 $(docker exec myapp pidof node)

# Copy heap dump
docker cp myapp:/tmp/heapdump-*.heapsnapshot ./
```

**Fix:**
```bash
# Set memory limit
docker run -m 512m myapp

# Container will be killed if exceeds limit
# Fix memory leak in application
```

---

## Docker Compose Issues

### Service Won't Start

**Check Status:**
```bash
docker compose ps

# View logs
docker compose logs service-name

# Follow logs
docker compose logs -f service-name
```

**Check Dependencies:**
```yaml
services:
  web:
    depends_on:
      api:
        condition: service_healthy

  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
```

---

### Environment Variables Not Working

**Check Variable Loading:**
```bash
# Check if .env file exists
cat .env

# Check if variables are set
docker compose config

# Debug
docker compose run --rm web env
```

**Common Issues:**

**1. Wrong .env Location:**
```bash
# .env must be in same directory as docker-compose.yml
ls -la
# .env
# docker-compose.yml
```

**2. Syntax Error:**
```env
# Bad
DB_PASSWORD = secret  # Spaces around =

# Good
DB_PASSWORD=secret
```

---

### Port Already in Use

**Symptom:**
```
Error: Bind for 0.0.0.0:3000 failed: port is already allocated
```

**Find Process:**
```bash
# Linux/Mac
lsof -i :3000

# Windows
netstat -ano | findstr :3000
```

**Fix:**
```bash
# Change port in docker-compose.yml
ports:
  - "3001:3000"

# Or stop the other process
kill -9 <PID>
```

---

## Image Issues

### Image Too Large

**Check Size:**
```bash
docker images myapp

# Check layer sizes
docker history myapp
```

**Reduce Size:**

**1. Use Multi-Stage Build:**
```dockerfile
FROM node:18 AS builder
RUN npm install
RUN npm run build

FROM node:18-alpine
COPY --from=builder /app/dist ./dist
# Reduction: 1GB → 200MB
```

**2. Use Alpine:**
```dockerfile
FROM node:18-alpine  # Instead of node:18
# Reduction: 900MB → 180MB
```

**3. Clean Up in Same Layer:**
```dockerfile
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

---

### Cannot Pull Image

**Symptom:**
```
Error response from daemon: Get https://registry-1.docker.io/v2/: unauthorized
```

**Check Login:**
```bash
# Login to Docker Hub
docker login

# For private registry
docker login myregistry.com
```

**Check Image Name:**
```bash
# Wrong
docker pull myapp

# Right
docker pull username/myapp
docker pull myregistry.com/myapp
```

---

### Image Build Failed

**Check Dockerfile Syntax:**
```dockerfile
# Bad
FROM node:18
RUN npm install  # Missing COPY

# Good
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
```

**Build with No Cache:**
```bash
docker build --no-cache -t myapp .
```

---

## Debugging Techniques

### Interactive Debugging

**Access Container Shell:**
```bash
# Running container
docker exec -it myapp /bin/sh

# New container
docker run -it myapp /bin/sh

# Override entrypoint
docker run -it --entrypoint /bin/sh myapp
```

**Debug Failed Container:**
```bash
# Run with different command
docker run -it myapp /bin/sh

# Check logs of failed container
docker logs <container-id>
```

---

### Network Debugging

**Install Tools:**
```bash
# Alpine
docker exec myapp apk add curl netcat-openbsd bind-tools

# Debian/Ubuntu
docker exec myapp apt-get update && apt-get install -y curl netcat dnsutils
```

**Test Connectivity:**
```bash
# Ping
docker exec myapp ping -c 3 api

# DNS lookup
docker exec myapp nslookup api

# Port check
docker exec myapp nc -zv api 3000

# HTTP request
docker exec myapp curl -v http://api:3000/health
```

---

### Process Debugging

**Check Processes:**
```bash
# List processes
docker top myapp

# Inside container
docker exec myapp ps aux

# Find process
docker exec myapp pidof node
```

**Check Resource Usage:**
```bash
# Stats
docker stats myapp

# Detailed
docker exec myapp top
```

---

### Log Debugging

**View Logs:**
```bash
# All logs
docker logs myapp

# Last N lines
docker logs --tail 50 myapp

# Follow
docker logs -f myapp

# With timestamps
docker logs -t myapp

# Since time
docker logs --since 2024-01-17T10:00:00 myapp

# Between times
docker logs --since 2024-01-17T10:00:00 --until 2024-01-17T11:00:00 myapp
```

**Search Logs:**
```bash
# Search for errors
docker logs myapp 2>&1 | grep -i error

# Count errors
docker logs myapp 2>&1 | grep -c ERROR

# Filter by timestamp
docker logs -t myapp | grep "2024-01-17T10"
```

---

## Common Error Messages

### "no space left on device"

**Check:**
```bash
df -h
docker system df
```

**Fix:**
```bash
docker system prune -a --volumes
```

---

### "Cannot connect to Docker daemon"

**Check:**
```bash
# Is Docker running?
systemctl status docker

# Check socket
ls -la /var/run/docker.sock
```

**Fix:**
```bash
# Start Docker
sudo systemctl start docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

---

### "conflict: unable to remove repository reference"

**Cause:** Image is being used by container

**Fix:**
```bash
# Stop and remove containers
docker rm -f $(docker ps -aq --filter ancestor=myapp)

# Then remove image
docker rmi myapp
```

---

### "layer does not exist" or "failed to compute cache key"

**Fix:**
```bash
# Clean build cache
docker builder prune

# Rebuild with no cache
docker build --no-cache .
```

---

## Troubleshooting Checklist

```
Container Issues:
☐ Check container status (docker ps -a)
☐ Check exit code
☐ Review logs (docker logs)
☐ Check resource limits
☐ Verify health check
☐ Check disk space

Network Issues:
☐ Verify container running
☐ Check port mapping
☐ Test connectivity
☐ Check DNS resolution
☐ Verify network configuration
☐ Check firewall rules

Volume Issues:
☐ Check mount points
☐ Verify volume exists
☐ Check permissions
☐ Test read/write
☐ Check disk space

Build Issues:
☐ Check Dockerfile syntax
☐ Verify build context
☐ Check .dockerignore
☐ Clear build cache
☐ Check disk space

Performance Issues:
☐ Check resource usage
☐ Review logs for errors
☐ Check network speed
☐ Monitor memory usage
☐ Profile application
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Exit code __________ indicates the container was killed due to out of memory.
2. To view logs of a stopped container, use __________.
3. Containers must be on the same __________ to communicate using service names.
4. Use __________ to clean up unused Docker resources.
5. Check container resource usage with __________.
6. Override entrypoint with __________ flag.

### True/False

1. ⬜ Exit code 0 means the container failed
2. ⬜ Default bridge network provides DNS resolution
3. ⬜ Bind mounts require absolute paths
4. ⬜ docker system prune removes running containers
5. ⬜ Anonymous volumes persist after container removal
6. ⬜ Can access stopped container with docker exec
7. ⬜ Exit code 137 indicates out of memory

### Multiple Choice

1. Container exits immediately, what to check first?
   - A) Network settings
   - B) Logs and exit code
   - C) Disk space
   - D) Dockerfile

2. Containers cannot communicate, likely cause?
   - A) Wrong image
   - B) Different networks
   - C) No volumes
   - D) No labels

3. "No space left on device" error, what to do?
   - A) Restart Docker
   - B) docker system prune
   - C) Rebuild images
   - D) Change network

4. How to debug DNS issues?
   - A) docker logs
   - B) docker exec container nslookup
   - C) docker inspect
   - D) docker stats

5. Exit code 127 means?
   - A) Success
   - B) Command not found
   - C) Out of memory
   - D) Permission denied

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. 137
2. docker logs <container-id>
3. network (custom network)
4. docker system prune
5. docker stats
6. --entrypoint

**True/False:**
1. ❌ False - Exit code 0 is success
2. ❌ False - Custom networks have DNS, not default bridge
3. ✅ True - Bind mounts need absolute paths
4. ❌ False - Only removes stopped containers
5. ❌ False - Anonymous volumes are removed with container
6. ❌ False - Container must be running for exec
7. ✅ True - 137 is SIGKILL (OOM)

**Multiple Choice:**
1. **B** - Logs and exit code
2. **B** - Different networks
3. **B** - docker system prune
4. **B** - docker exec container nslookup
5. **B** - Command not found

</details>

</details>

---

[← Previous: 10.2 Production Considerations](../10-best-practices/02-production-considerations.md) | [Next: 11.3 Common Patterns →](03-common-patterns.md)