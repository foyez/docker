# 11.1 Docker Command Cheatsheet

Quick reference for all Docker and Docker Compose commands.

---

## Docker Basics

### Version & Info

```bash
# Check Docker version
docker --version
docker version  # Detailed version info

# System information
docker info

# Display system-wide information
docker system df  # Disk usage
docker system events  # Real-time events
docker system prune  # Clean up unused resources
```

---

## Docker Images

### Building Images

```bash
# Build image from Dockerfile
docker build -t image_name:tag .
docker build -t myapp:1.0 .

# Build with specific Dockerfile
docker build -f Dockerfile.dev -t myapp:dev .

# Build with build arguments
docker build --build-arg NODE_ENV=production -t myapp:prod .

# Build without cache
docker build --no-cache -t myapp:1.0 .

# Multi-platform build
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 .
```

### Listing Images

```bash
# List all images
docker images
docker image ls

# List with specific format
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# List only image IDs
docker images -q
```

### Pulling & Pushing Images

```bash
# Pull from Docker Hub
docker pull nginx:latest
docker pull postgres:15

# Pull from private registry
docker pull registry.example.com/myapp:1.0

# Push to Docker Hub
docker push username/myapp:1.0

# Push to private registry
docker push registry.example.com/myapp:1.0

# Tag and push
docker tag myapp:1.0 username/myapp:1.0
docker push username/myapp:1.0
```

### Managing Images

```bash
# Remove image
docker rmi image_name:tag
docker image rm image_name:tag
docker rmi image_id

# Remove multiple images
docker rmi image1 image2 image3

# Remove dangling images (untagged)
docker image prune

# Remove all unused images
docker image prune -a

# Remove with force
docker rmi -f image_name
```

### Inspecting Images

```bash
# Inspect image
docker inspect image_name:tag
docker image inspect myapp:1.0

# View image history
docker history image_name:tag

# Save image to tar file
docker save -o myapp.tar myapp:1.0

# Load image from tar file
docker load -i myapp.tar

# Export container to tar
docker export container_name > container.tar

# Import container from tar
docker import container.tar myapp:imported
```

### Tagging Images

```bash
# Tag image
docker tag source_image:tag target_image:tag
docker tag myapp:1.0 myapp:latest
docker tag myapp:1.0 registry.example.com/myapp:1.0
```

---

## Docker Containers

### Running Containers

```bash
# Run container (attached mode)
docker run image_name

# Run in detached mode
docker run -d image_name

# Run with name
docker run --name container_name image_name
docker run --name myapp nginx

# Run with port mapping
docker run -p host_port:container_port image_name
docker run -p 8080:80 nginx
docker run -p 8080:80 -p 443:443 nginx

# Run with environment variables
docker run -e VAR=value image_name
docker run -e NODE_ENV=production myapp

# Run with volume
docker run -v /host/path:/container/path image_name
docker run -v mydata:/data postgres

# Run with network
docker run --network network_name image_name

# Run with resource limits
docker run --cpus=2 --memory=1g image_name

# Run in interactive mode
docker run -it image_name bash
docker run -it ubuntu /bin/bash

# Run with restart policy
docker run --restart unless-stopped nginx
docker run --restart always nginx

# Run with all common options
docker run -d \
  --name myapp \
  -p 8080:80 \
  -e NODE_ENV=production \
  -v app-data:/data \
  --network mynetwork \
  --restart unless-stopped \
  myapp:1.0
```

### Listing Containers

```bash
# List running containers
docker ps
docker container ls

# List all containers (including stopped)
docker ps -a
docker container ls -a

# List only container IDs
docker ps -q

# List with custom format
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### Container Lifecycle

```bash
# Stop container
docker stop container_name
docker stop container_id

# Stop with timeout
docker stop -t 30 container_name

# Start container
docker start container_name

# Restart container
docker restart container_name

# Pause container
docker pause container_name

# Unpause container
docker unpause container_name

# Kill container (force stop)
docker kill container_name

# Remove container
docker rm container_name
docker container rm container_name

# Remove with force (running container)
docker rm -f container_name

# Remove all stopped containers
docker container prune

# Stop and remove
docker stop container_name && docker rm container_name
```

### Executing Commands

```bash
# Execute command in running container
docker exec container_name command
docker exec myapp ls -la

# Interactive shell
docker exec -it container_name bash
docker exec -it container_name sh

# Execute as specific user
docker exec -u user_name container_name command

# Execute with environment variable
docker exec -e VAR=value container_name command

# Run command in new container (one-time)
docker run --rm image_name command
docker run --rm nginx ls /etc/nginx
```

### Viewing Logs

```bash
# View logs
docker logs container_name

# Follow logs (real-time)
docker logs -f container_name

# View last N lines
docker logs --tail 100 container_name

# View logs with timestamps
docker logs -t container_name

# View logs since specific time
docker logs --since 2023-01-01T00:00:00 container_name
docker logs --since 10m container_name
```

### Inspecting Containers

```bash
# Inspect container
docker inspect container_name

# Get specific field
docker inspect --format='{{.NetworkSettings.IPAddress}}' container_name

# View container stats
docker stats container_name

# View all containers stats
docker stats

# View container processes
docker top container_name

# View container changes
docker diff container_name

# Copy files from container
docker cp container_name:/path/in/container /path/on/host

# Copy files to container
docker cp /path/on/host container_name:/path/in/container
```

---

## Docker Volumes

### Volume Management

```bash
# Create volume
docker volume create volume_name

# List volumes
docker volume ls

# Inspect volume
docker volume inspect volume_name

# Remove volume
docker volume rm volume_name

# Remove unused volumes
docker volume prune

# Remove all volumes (dangerous!)
docker volume prune -a
```

### Using Volumes

```bash
# Named volume
docker run -v volume_name:/path/in/container image_name
docker run -v mydata:/var/lib/mysql mysql

# Bind mount (host path)
docker run -v /host/path:/container/path image_name
docker run -v $(pwd)/app:/usr/src/app node

# Read-only volume
docker run -v mydata:/data:ro image_name

# Multiple volumes
docker run \
  -v vol1:/data1 \
  -v vol2:/data2 \
  -v /host:/data3 \
  image_name
```

---

## Docker Networks

### Network Management

```bash
# Create network
docker network create network_name

# Create bridge network with subnet
docker network create \
  --driver bridge \
  --subnet 172.18.0.0/16 \
  --gateway 172.18.0.1 \
  my-network

# List networks
docker network ls

# Inspect network
docker network inspect network_name

# Remove network
docker network rm network_name

# Remove unused networks
docker network prune

# Connect container to network
docker network connect network_name container_name

# Disconnect container from network
docker network disconnect network_name container_name
```

### Using Networks

```bash
# Run container in specific network
docker run --network network_name image_name

# Run container with network alias
docker run --network network_name --network-alias db postgres

# Run container with no network
docker run --network none image_name

# Run container with host network
docker run --network host image_name
```

---

## Docker Compose

### Basic Commands

```bash
# Start services
docker-compose up

# Start in detached mode
docker-compose up -d

# Start specific services
docker-compose up service1 service2

# Stop services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Stop services (keep containers)
docker-compose stop

# Start stopped services
docker-compose start

# Restart services
docker-compose restart
```

### Building & Pulling

```bash
# Build services
docker-compose build

# Build without cache
docker-compose build --no-cache

# Build specific service
docker-compose build service_name

# Pull service images
docker-compose pull

# Push service images
docker-compose push
```

### Viewing & Monitoring

```bash
# List containers
docker-compose ps

# View logs
docker-compose logs

# Follow logs
docker-compose logs -f

# View logs for specific service
docker-compose logs service_name

# View logs with timestamps
docker-compose logs -t

# Execute command in service
docker-compose exec service_name command
docker-compose exec web bash

# Run one-off command
docker-compose run service_name command
docker-compose run web npm test

# View service configuration
docker-compose config

# Validate compose file
docker-compose config --quiet
```

### Scaling

```bash
# Scale service
docker-compose up -d --scale service_name=num
docker-compose up -d --scale web=3

# Scale multiple services
docker-compose up -d --scale web=3 --scale worker=5
```

### Managing Services

```bash
# Pause services
docker-compose pause

# Unpause services
docker-compose unpause

# View service ports
docker-compose port service_name port_number

# List images used by services
docker-compose images

# Remove stopped service containers
docker-compose rm

# Force remove
docker-compose rm -f
```

---

## Docker Registry

### Login & Logout

```bash
# Login to Docker Hub
docker login
docker login -u username -p password

# Login to private registry
docker login registry.example.com
docker login -u username -p password registry.example.com

# Logout
docker logout
docker logout registry.example.com
```

### Working with Registries

```bash
# Search Docker Hub
docker search nginx
docker search --limit 5 nginx

# Pull from specific registry
docker pull registry.example.com/myapp:1.0

# Push to specific registry
docker push registry.example.com/myapp:1.0

# Run local registry
docker run -d -p 5000:5000 --name registry registry:2

# Push to local registry
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0

# List images in registry (REST API)
curl -X GET http://localhost:5000/v2/_catalog
```

---

## System Maintenance

### Cleanup Commands

```bash
# Remove unused data
docker system prune

# Remove all unused data (including volumes)
docker system prune -a --volumes

# Remove stopped containers
docker container prune

# Remove unused images
docker image prune

# Remove unused volumes
docker volume prune

# Remove unused networks
docker network prune

# Remove everything (nuclear option)
docker system prune -a --volumes -f
```

### System Information

```bash
# Disk usage
docker system df

# Detailed disk usage
docker system df -v

# Real-time events
docker system events

# Filter events
docker system events --filter type=container
docker system events --since '2023-01-01T00:00:00'
```

---

## Dockerfile Commands Reference

```dockerfile
# Base image
FROM node:18

# Build arguments
ARG NODE_ENV=production

# Environment variables
ENV NODE_ENV=$NODE_ENV
ENV PORT=3000

# Working directory
WORKDIR /app

# Copy files
COPY package.json .
COPY . .

# Add files (can extract archives)
ADD archive.tar.gz /app

# Run commands
RUN npm install
RUN apt-get update && apt-get install -y curl

# Expose ports
EXPOSE 3000

# Set user
USER node

# Create volume mount point
VOLUME /data

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:3000/ || exit 1

# Default command
CMD ["npm", "start"]

# Entry point
ENTRYPOINT ["node"]

# Labels
LABEL maintainer="dev@example.com"
LABEL version="1.0"

# Shell form vs Exec form
RUN npm install          # Shell form
RUN ["npm", "install"]   # Exec form (preferred)

# Multi-stage build
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm run build

FROM node:18-alpine
COPY --from=builder /app/dist /app
CMD ["node", "server.js"]
```

---

## Quick Reference Tables

### Container States

| State | Description | Can Start? | Can Stop? |
|-------|-------------|------------|-----------|
| created | Container created, not started | ✅ Yes | ❌ No |
| running | Container is running | ❌ No | ✅ Yes |
| paused | Container is paused | ❌ No | ✅ Yes (unpause first) |
| exited | Container has stopped | ✅ Yes | ❌ No |
| dead | Container crashed | ❌ No | ❌ No (remove it) |

### Port Mapping

```bash
# Syntax: -p [host_ip:]host_port:container_port[/protocol]

-p 8080:80              # Map host 8080 to container 80
-p 127.0.0.1:8080:80    # Bind to localhost only
-p 8080:80/tcp          # Specify protocol
-p 8080:80 -p 443:443   # Map multiple ports
```

### Restart Policies

```bash
--restart no              # Never restart (default)
--restart always          # Always restart
--restart unless-stopped  # Restart unless manually stopped
--restart on-failure      # Restart on non-zero exit
--restart on-failure:3    # Restart max 3 times
```

### Resource Limits

```bash
--cpus=2              # Limit to 2 CPU cores
--cpus=0.5            # Limit to 50% of 1 core
--memory=1g           # Limit to 1GB RAM
--memory=512m         # Limit to 512MB RAM
--memory-swap=2g      # Total memory (RAM + swap)
```

---

## Common Workflows

### Development Workflow

```bash
# 1. Build image
docker build -t myapp:dev .

# 2. Run with live reload
docker run -d \
  --name myapp-dev \
  -p 3000:3000 \
  -v $(pwd):/app \
  myapp:dev

# 3. View logs
docker logs -f myapp-dev

# 4. Test changes
docker exec -it myapp-dev npm test

# 5. Stop and clean up
docker stop myapp-dev
docker rm myapp-dev
```

### Production Deployment

```bash
# 1. Build production image
docker build -t myapp:1.0 .

# 2. Tag for registry
docker tag myapp:1.0 registry.example.com/myapp:1.0

# 3. Push to registry
docker push registry.example.com/myapp:1.0

# 4. Pull on production server
docker pull registry.example.com/myapp:1.0

# 5. Run with production settings
docker run -d \
  --name myapp \
  -p 80:8080 \
  -e NODE_ENV=production \
  --restart unless-stopped \
  --memory=2g \
  --cpus=2 \
  registry.example.com/myapp:1.0
```

### Debugging Workflow

```bash
# 1. Check if container is running
docker ps -a

# 2. View logs
docker logs container_name

# 3. Check container stats
docker stats container_name

# 4. Inspect container
docker inspect container_name

# 5. Execute shell
docker exec -it container_name bash

# 6. Check processes
docker top container_name

# 7. View container changes
docker diff container_name
```

---

## Keyboard Shortcuts (Interactive Mode)

```
Ctrl + P, Ctrl + Q  - Detach from container (keep running)
Ctrl + D            - Exit and stop container
Ctrl + C            - Stop current process in container
```

---

## Common Patterns

### One-liner Commands

```bash
# Stop all running containers
docker stop $(docker ps -q)

# Remove all containers
docker rm $(docker ps -a -q)

# Remove all images
docker rmi $(docker images -q)

# Get container IP address
docker inspect -f '{{.NetworkSettings.IPAddress}}' container_name

# Remove exited containers
docker rm $(docker ps -a -f status=exited -q)

# Remove dangling images
docker rmi $(docker images -f "dangling=true" -q)
```

### Useful Aliases

```bash
# Add to ~/.bashrc or ~/.zshrc

alias dps='docker ps'
alias dpsa='docker ps -a'
alias di='docker images'
alias dex='docker exec -it'
alias dlog='docker logs -f'
alias dstop='docker stop'
alias drm='docker rm'
alias drmi='docker rmi'
alias dprune='docker system prune -a'

# Docker Compose
alias dc='docker-compose'
alias dcu='docker-compose up -d'
alias dcd='docker-compose down'
alias dcl='docker-compose logs -f'
alias dcp='docker-compose ps'
alias dcb='docker-compose build'
```

---

**Remember:** Use `docker COMMAND --help` for detailed help on any command!