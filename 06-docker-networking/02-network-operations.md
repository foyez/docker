# 6.2 Network Operations

Managing Docker networks: creating, inspecting, connecting, disconnecting, and removing networks.

---

## Creating Networks

### docker network create

**Purpose:** Create a new network.

**Syntax:**
```bash
docker network create [OPTIONS] NETWORK
```

**Basic Creation:**
```bash
# Create bridge network (default)
docker network create my-network

# Specify driver
docker network create --driver bridge my-network

# Create overlay network (Swarm)
docker network create --driver overlay my-overlay
```

**With Subnet:**
```bash
# Custom subnet
docker network create \
    --subnet=172.20.0.0/16 \
    my-network

# Multiple subnets
docker network create \
    --subnet=172.20.0.0/16 \
    --subnet=2001:db8::/64 \
    dual-stack-net

# Gateway
docker network create \
    --subnet=172.20.0.0/16 \
    --gateway=172.20.0.1 \
    my-network

# IP range (DHCP pool)
docker network create \
    --subnet=172.20.0.0/16 \
    --ip-range=172.20.0.0/24 \
    my-network
```

**With Labels:**
```bash
# Add metadata
docker network create \
    --label env=production \
    --label team=backend \
    prod-network

# Multiple labels
docker network create \
    --label project=myapp \
    --label version=1.0 \
    --label backup=daily \
    myapp-network
```

**Internal Network:**
```bash
# No external connectivity
docker network create --internal private-network

# Containers can communicate internally
# But cannot access internet
```

**Advanced Options:**
```bash
# IPv6 support
docker network create \
    --ipv6 \
    --subnet=fd00::/64 \
    ipv6-network

# MTU setting
docker network create \
    --opt com.docker.network.driver.mtu=9000 \
    jumbo-network

# Custom bridge name
docker network create \
    --opt com.docker.network.bridge.name=br-myapp \
    my-network
```

---

## Listing Networks

### docker network ls

**Purpose:** List all networks.

**Syntax:**
```bash
docker network ls [OPTIONS]
```

**Basic Usage:**
```bash
# List all networks
docker network ls

# Output:
# NETWORK ID     NAME              DRIVER    SCOPE
# abc123def456   bridge            bridge    local
# def456ghi789   host              host      local
# ghi789jkl012   none              null      local
# jkl012mno345   my-network        bridge    local
```

**With Filters:**
```bash
# Filter by driver
docker network ls --filter driver=bridge

# Filter by label
docker network ls --filter label=env=production

# Filter by name pattern
docker network ls --filter name=my*

# Filter by ID
docker network ls --filter id=abc123

# Multiple filters
docker network ls \
    --filter driver=bridge \
    --filter label=env=production
```

**Custom Format:**
```bash
# Just names
docker network ls --format '{{.Name}}'

# Name and driver
docker network ls --format '{{.Name}}: {{.Driver}}'

# Table format
docker network ls --format 'table {{.Name}}\t{{.Driver}}\t{{.Scope}}'

# JSON
docker network ls --format '{{json .}}'
```

**Quiet Mode:**
```bash
# Just IDs
docker network ls -q

# Use in scripts
for net in $(docker network ls -q); do
    echo "Network: $net"
done
```

---

## Inspecting Networks

### docker network inspect

**Purpose:** Display detailed information about networks.

**Syntax:**
```bash
docker network inspect [OPTIONS] NETWORK [NETWORK...]
```

**Basic Inspection:**
```bash
# Inspect network
docker network inspect my-network

# Output (JSON):
[
    {
        "Name": "my-network",
        "Id": "abc123def456...",
        "Created": "2024-01-17T10:00:00Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "a1b2c3d4...": {
                "Name": "web",
                "EndpointID": "...",
                "MacAddress": "...",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

**Extract Specific Fields:**
```bash
# Get subnet
docker network inspect -f '{{range .IPAM.Config}}{{.Subnet}}{{end}}' my-network

# Get gateway
docker network inspect -f '{{range .IPAM.Config}}{{.Gateway}}{{end}}' my-network

# Get driver
docker network inspect -f '{{.Driver}}' my-network

# List connected containers
docker network inspect -f '{{range $k,$v := .Containers}}{{$v.Name}} {{end}}' my-network

# Get container IPs
docker network inspect -f '{{range $k,$v := .Containers}}{{$v.Name}}: {{$v.IPv4Address}}{{"\n"}}{{end}}' my-network
```

**Inspect Multiple:**
```bash
# Multiple networks
docker network inspect net1 net2 net3

# All networks
docker network ls -q | xargs docker network inspect
```

---

## Connecting Containers

### docker network connect

**Purpose:** Connect a container to a network.

**Syntax:**
```bash
docker network connect [OPTIONS] NETWORK CONTAINER
```

**Basic Connection:**
```bash
# Connect running container
docker network connect my-network my-container

# Verify
docker network inspect my-network
```

**With IP Address:**
```bash
# Assign specific IP
docker network connect \
    --ip 172.18.0.10 \
    my-network \
    my-container

# IPv6
docker network connect \
    --ip6 2001:db8::10 \
    my-network \
    my-container
```

**With Alias:**
```bash
# Add network alias (additional DNS name)
docker network connect \
    --alias web-server \
    --alias api \
    my-network \
    my-container

# Container accessible as:
# - my-container (container name)
# - web-server (alias)
# - api (alias)
```

**With Link (Deprecated):**
```bash
# Legacy linking
docker network connect \
    --link other-container:alias \
    my-network \
    my-container
```

**Multiple Networks:**
```bash
# Container on multiple networks
docker network connect network1 my-container
docker network connect network2 my-container
docker network connect network3 my-container

# Verify
docker inspect my-container --format='{{json .NetworkSettings.Networks}}' | jq
```

---

## Disconnecting Containers

### docker network disconnect

**Purpose:** Disconnect a container from a network.

**Syntax:**
```bash
docker network disconnect [OPTIONS] NETWORK CONTAINER
```

**Basic Disconnection:**
```bash
# Disconnect container
docker network disconnect my-network my-container

# Force disconnect
docker network disconnect -f my-network my-container
```

**Use Cases:**
```bash
# Network maintenance
docker network disconnect old-network app
docker network connect new-network app

# Temporary isolation
docker network disconnect public-network suspicious-container

# Multi-stage deployment
docker network disconnect blue-network app
docker network connect green-network app
```

---

## Removing Networks

### docker network rm

**Purpose:** Remove one or more networks.

**Syntax:**
```bash
docker network rm NETWORK [NETWORK...]
```

**Basic Removal:**
```bash
# Remove network
docker network rm my-network

# Remove multiple
docker network rm net1 net2 net3

# Remove all custom networks (dangerous!)
docker network rm $(docker network ls -q)
```

**Safe Removal:**
```bash
# Check if in use
docker network inspect my-network

# If containers connected, must disconnect first
docker network disconnect my-network container1
docker network disconnect my-network container2

# Then remove
docker network rm my-network
```

---

### docker network prune

**Purpose:** Remove all unused networks.

**Syntax:**
```bash
docker network prune [OPTIONS]
```

**Basic Prune:**
```bash
# Remove unused networks
docker network prune

# WARNING! This will remove all custom networks not used by at least one container.
# Are you sure you want to continue? [y/N]

# Force (no confirmation)
docker network prune -f
```

**With Filters:**
```bash
# Prune networks with label
docker network prune --filter label=temporary=true

# Prune old networks
docker network prune --filter until=24h

# Multiple filters
docker network prune \
    --filter label=env=dev \
    --filter until=48h
```

---

## Real-World Patterns

### Pattern 1: Blue-Green Deployment

```bash
# Create two networks
docker network create blue-network
docker network create green-network

# Current production (blue)
docker run -d --name app-v1 --network blue-network myapp:1.0
docker run -d --name lb --network blue-network nginx

# Deploy new version (green)
docker run -d --name app-v2 --network green-network myapp:2.0

# Test green
docker network connect green-network lb
# Test traffic to app-v2

# Switch traffic
docker network disconnect blue-network lb
# Now only on green-network

# Rollback if needed
docker network disconnect green-network lb
docker network connect blue-network lb
```

---

### Pattern 2: Network Migration

```bash
# Migrate containers to new network

# Create new network
docker network create \
    --subnet=172.20.0.0/16 \
    new-network

# Get containers on old network
CONTAINERS=$(docker network inspect old-network \
    --format='{{range $k,$v := .Containers}}{{$v.Name}} {{end}}')

# Migrate each container
for container in $CONTAINERS; do
    echo "Migrating $container..."
    
    # Connect to new network
    docker network connect new-network $container
    
    # Test connectivity
    docker exec $container ping -c 1 172.20.0.1
    
    # If successful, disconnect from old
    if [ $? -eq 0 ]; then
        docker network disconnect old-network $container
        echo "✓ Migrated $container"
    else
        echo "✗ Failed to migrate $container"
    fi
done

# Remove old network
docker network rm old-network
```

---

### Pattern 3: Network Segmentation

```bash
#!/bin/bash
# Setup multi-tier network architecture

# Create networks
docker network create frontend
docker network create backend
docker network create --internal database

# Deploy web tier
docker run -d \
    --name web \
    --network frontend \
    -p 80:80 \
    nginx

# Deploy application tier
docker run -d \
    --name app \
    myapp
docker network connect frontend app
docker network connect backend app

# Deploy database tier
docker run -d \
    --name db \
    --network database \
    postgres

# Connect app to database network
docker network connect database app

# Verify connectivity
docker exec web ping app     # ✓ Works (frontend)
docker exec app ping db      # ✓ Works (database)
docker exec web ping db      # ✗ Fails (isolated)
```

---

### Pattern 4: Dynamic Service Discovery

```bash
# Service registration
register_service() {
    local service=$1
    local network=$2
    
    # Start service
    docker run -d \
        --name $service \
        --network $network \
        --label service.type=backend \
        myapp
    
    # Register with discovery
    docker network connect discovery-network $service
}

# Service discovery
discover_services() {
    local network=$1
    
    # Find all containers on network
    docker network inspect $network \
        --format='{{range $k,$v := .Containers}}{{$v.Name}} {{end}}'
}

# Usage
register_service api-1 backend-network
register_service api-2 backend-network
register_service api-3 backend-network

# Discover
services=$(discover_services backend-network)
echo "Available services: $services"
```

---

### Pattern 5: Network Monitoring

```bash
#!/bin/bash
# Monitor network usage

# List all networks with stats
for net in $(docker network ls -q); do
    NAME=$(docker network inspect -f '{{.Name}}' $net)
    DRIVER=$(docker network inspect -f '{{.Driver}}' $net)
    COUNT=$(docker network inspect -f '{{len .Containers}}' $net)
    
    echo "Network: $NAME"
    echo "  Driver: $DRIVER"
    echo "  Containers: $COUNT"
    
    if [ $COUNT -gt 0 ]; then
        echo "  Connected containers:"
        docker network inspect -f \
            '{{range $k,$v := .Containers}}    - {{$v.Name}} ({{$v.IPv4Address}}){{"\n"}}{{end}}' \
            $net
    fi
    
    echo ""
done
```

---

## Troubleshooting

### Check Container Network

```bash
# View all networks for container
docker inspect my-container --format='{{json .NetworkSettings.Networks}}' | jq

# Get IP addresses
docker inspect my-container --format='{{range $k,$v := .NetworkSettings.Networks}}{{$k}}: {{$v.IPAddress}}{{"\n"}}{{end}}'

# Get gateway
docker inspect my-container --format='{{range $k,$v := .NetworkSettings.Networks}}{{$k}} gateway: {{$v.Gateway}}{{"\n"}}{{end}}'
```

### Network Connectivity Test

```bash
# Test from container
docker exec my-container ping -c 3 other-container

# Test specific network
docker exec my-container ping -c 3 172.18.0.1

# DNS test
docker exec my-container nslookup other-container

# Port test
docker exec my-container nc -zv other-container 8000
```

### Cannot Remove Network

```bash
# Error: network has active endpoints

# Find connected containers
docker network inspect my-network \
    --format='{{range $k,$v := .Containers}}{{$v.Name}} {{end}}'

# Disconnect all
for container in $(docker network inspect my-network \
    --format='{{range $k,$v := .Containers}}{{$v.Name}} {{end}}'); do
    docker network disconnect my-network $container
done

# Now remove
docker network rm my-network
```

### Network Conflicts

```bash
# Subnet conflicts with existing network

# List all subnets in use
for net in $(docker network ls -q); do
    SUBNET=$(docker network inspect -f \
        '{{range .IPAM.Config}}{{.Subnet}} {{end}}' $net)
    NAME=$(docker network inspect -f '{{.Name}}' $net)
    echo "$NAME: $SUBNET"
done

# Choose different subnet
docker network create \
    --subnet=172.30.0.0/16 \
    my-network
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The command __________ creates a new Docker network.
2. Use __________ to connect a running container to a network.
3. The __________ flag creates a network with no external connectivity.
4. To remove all unused networks, use __________.
5. __________ disconnects a container from a network.
6. Network __________ provide additional DNS names for containers.

### True/False

1. ⬜ A container can only be connected to one network at a time
2. ⬜ Removing a network automatically disconnects all containers
3. ⬜ Internal networks cannot access the internet
4. ⬜ You can specify a custom IP address when connecting to a network
5. ⬜ Network prune removes all networks including default ones
6. ⬜ Containers must be stopped before connecting to a network
7. ⬜ Network aliases provide additional DNS names

### Multiple Choice

1. Which command lists all networks?
   - A) docker network list
   - B) docker network ls
   - C) docker networks
   - D) docker ls networks

2. What does --internal do when creating a network?
   - A) Makes it faster
   - B) Prevents external connectivity
   - C) Enables IPv6
   - D) Nothing

3. How do you assign a specific IP when connecting?
   - A) --address
   - B) --ip
   - C) --ipaddr
   - D) --static-ip

4. Which removes unused networks?
   - A) docker network clean
   - B) docker network prune
   - C) docker network remove
   - D) docker network delete

5. Can a container be on multiple networks?
   - A) No, only one
   - B) Yes, unlimited
   - C) Yes, maximum 2
   - D) Only with overlay networks

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. docker network create
2. docker network connect
3. --internal
4. docker network prune
5. docker network disconnect
6. aliases

**True/False:**
1. ❌ False - Containers can be on multiple networks
2. ❌ False - Must disconnect containers first
3. ✅ True - Internal networks are isolated
4. ✅ True - Use --ip flag
5. ❌ False - Only removes unused networks, not default
6. ❌ False - Can connect to running containers
7. ✅ True - Aliases provide additional DNS names

**Multiple Choice:**
1. **B** - docker network ls
2. **B** - Prevents external connectivity
3. **B** - --ip
4. **B** - docker network prune
5. **B** - Yes, unlimited

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: How would you isolate a compromised container from the network?

<details>
<summary><strong>View Answer</strong></summary>

**Immediate Response Steps:**

---

**Step 1: Identify the Container**

```bash
# Container showing suspicious activity
CONTAINER="suspicious-app"

# Verify it's running
docker ps --filter name=$CONTAINER
```

---

**Step 2: Quick Isolation (Disconnect from All Networks)**

```bash
# List current networks
docker inspect $CONTAINER \
    --format='{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}'

# Disconnect from each network
for net in $(docker inspect $CONTAINER \
    --format='{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}'); do
    
    echo "Disconnecting from $net..."
    docker network disconnect $net $CONTAINER
done

# Verify isolation
docker inspect $CONTAINER \
    --format='{{json .NetworkSettings.Networks}}' | jq
# Should be empty: {}
```

---

**Step 3: Move to Isolated Network (For Investigation)**

```bash
# Create isolated investigation network
docker network create \
    --internal \
    --subnet=192.168.99.0/24 \
    investigation-net

# Connect container to isolated network
docker network connect investigation-net $CONTAINER

# Now container has network access only to:
# - Other containers on investigation-net
# - No internet access (internal network)
```

---

**Step 4: Capture Network State (Before Isolation)**

```bash
# Export network configuration
docker inspect $CONTAINER > /tmp/container-network-state.json

# Check active connections (if possible)
docker exec $CONTAINER netstat -tunap > /tmp/active-connections.txt

# Check established connections
docker exec $CONTAINER ss -tunap > /tmp/socket-stats.txt

# DNS cache
docker exec $CONTAINER cat /etc/resolv.conf > /tmp/dns-config.txt
```

---

**Step 5: Complete Isolation (No Network)**

```bash
# Disconnect from investigation network
docker network disconnect investigation-net $CONTAINER

# Connect to 'none' network (no networking)
docker network connect none $CONTAINER

# Verify complete isolation
docker exec $CONTAINER ip addr
# Only loopback (127.0.0.1)

docker exec $CONTAINER ping 8.8.8.8
# Network unreachable
```

---

**Alternative: Use Network None from Start**

```bash
# Pause container first
docker pause $CONTAINER

# Get container config
docker inspect $CONTAINER > /tmp/container-config.json

# Stop container
docker stop $CONTAINER

# Remove container
docker rm $CONTAINER

# Recreate with no network
docker create \
    --name $CONTAINER-isolated \
    --network none \
    $(docker inspect $CONTAINER --format='{{.Config.Image}}')

# Start in isolated state
docker start $CONTAINER-isolated
```

---

**Step 6: Monitoring During Isolation**

```bash
# Monitor for any network attempts
docker logs -f $CONTAINER 2>&1 | grep -i "network\|connection"

# Check processes
docker top $CONTAINER

# Resource usage
docker stats $CONTAINER --no-stream

# File changes
docker diff $CONTAINER
```

---

**Complete Isolation Script:**

```bash
#!/bin/bash
# isolate-container.sh

CONTAINER=$1
BACKUP_DIR="/var/log/docker/isolation"

mkdir -p $BACKUP_DIR

echo "=== Isolating Container: $CONTAINER ==="

# Step 1: Capture current state
echo "Capturing network state..."
docker inspect $CONTAINER > \
    $BACKUP_DIR/${CONTAINER}-state-$(date +%Y%m%d-%H%M%S).json

# Step 2: Capture active connections
echo "Capturing connections..."
docker exec $CONTAINER netstat -tunap > \
    $BACKUP_DIR/${CONTAINER}-connections-$(date +%Y%m%d-%H%M%S).txt 2>&1

# Step 3: Get current networks
NETWORKS=$(docker inspect $CONTAINER \
    --format='{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}')

echo "Currently on networks: $NETWORKS"

# Step 4: Disconnect from all networks
for net in $NETWORKS; do
    echo "Disconnecting from $net..."
    docker network disconnect $net $CONTAINER
done

# Step 5: Connect to isolated network
echo "Creating isolated network..."
docker network create --internal isolation-net 2>/dev/null || true

echo "Connecting to isolation network..."
docker network connect isolation-net $CONTAINER

# Step 6: Verify isolation
echo ""
echo "=== Isolation Status ==="
echo "Networks:"
docker inspect $CONTAINER \
    --format='{{range $k,$v := .NetworkSettings.Networks}}  - {{$k}}: {{$v.IPAddress}}{{"\n"}}{{end}}'

echo ""
echo "Testing external connectivity..."
if docker exec $CONTAINER ping -c 1 8.8.8.8 > /dev/null 2>&1; then
    echo "  ✗ WARNING: Can still reach internet!"
else
    echo "  ✓ No external connectivity"
fi

echo ""
echo "Container isolated. Logs saved to: $BACKUP_DIR"
```

---

**Usage:**

```bash
chmod +x isolate-container.sh
./isolate-container.sh suspicious-app
```

---

**Best Practices:**

```
1. Document Everything
   ✓ Capture state before changes
   ✓ Log all actions taken
   ✓ Preserve evidence

2. Gradual Isolation
   ✓ Don't immediately destroy
   ✓ Isolate first, investigate
   ✓ Preserve for forensics

3. Monitoring
   ✓ Watch logs during isolation
   ✓ Monitor for suspicious activity
   ✓ Track resource usage

4. Backup Plan
   ✓ Save network config
   ✓ Can restore if false alarm
   ✓ Document normal behavior
```

---

**Recovery (If False Alarm):**

```bash
# Restore original networks
ORIGINAL_NETWORKS=$(cat /tmp/container-network-state.json | \
    jq -r '.NetworkSettings.Networks | keys[]')

# Disconnect from isolation
docker network disconnect isolation-net $CONTAINER

# Reconnect to original networks
for net in $ORIGINAL_NETWORKS; do
    echo "Reconnecting to $net..."
    docker network connect $net $CONTAINER
done
```

</details>

### Question 2: Explain how to set up network monitoring and logging

<details>
<summary><strong>View Answer</strong></summary>

**Comprehensive Network Monitoring Setup:**

---

### Level 1: Built-in Docker Logging

**Container Network Stats:**

```bash
# Real-time network stats
docker stats

# Output:
# CONTAINER  CPU %  MEM USAGE  NET I/O
# web        0.5%   100MB      1.2MB / 800KB
# api        2.1%   200MB      5.5MB / 3.2MB
# db         1.0%   500MB      2.1MB / 1.5MB

# Specific container
docker stats web --no-stream

# JSON format for parsing
docker stats --no-stream --format '{{json .}}'
```

**Network Inspection:**

```bash
# Network details
docker network inspect my-network

# Container connections
docker network inspect my-network \
    --format='{{range $k,$v := .Containers}}{{$v.Name}}: {{$v.IPv4Address}}{{"\n"}}{{end}}'
```

---

### Level 2: Network Traffic Capture

**Using tcpdump:**

```bash
# Capture traffic from container
docker run -d --name web nginx
PID=$(docker inspect -f '{{.State.Pid}}' web)

# Capture on container's network namespace
sudo nsenter -t $PID -n tcpdump -i eth0 -w /tmp/web-traffic.pcap

# Analyze later with Wireshark
wireshark /tmp/web-traffic.pcap
```

**Dedicated Network Monitor Container:**

```bash
# Run monitoring container on same network
docker run -d \
    --name monitor \
    --network my-network \
    --cap-add NET_ADMIN \
    nicolaka/netshoot \
    tcpdump -i eth0 -w /capture/traffic.pcap
```

---

### Level 3: Centralized Logging

**Fluentd Setup:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Fluentd for log aggregation
  fluentd:
    image: fluent/fluentd:latest
    ports:
      - "24224:24224"
    volumes:
      - ./fluentd/conf:/fluentd/etc
      - ./logs:/fluentd/log
    networks:
      - logging

  # Application with custom logging
  web:
    image: nginx
    logging:
      driver: fluentd
      options:
        fluentd-address: fluentd:24224
        tag: web
    networks:
      - app-network
      - logging

  api:
    image: myapi
    logging:
      driver: fluentd
      options:
        fluentd-address: fluentd:24224
        tag: api
    networks:
      - app-network
      - logging

networks:
  app-network:
  logging:
```

**Fluentd Configuration (fluentd.conf):**

```
<source>
  @type forward
  port 24224
</source>

# Parse nginx access logs
<filter web>
  @type parser
  format nginx
  key_name log
</filter>

# Output to file
<match **>
  @type file
  path /fluentd/log/${tag}
  <buffer tag,time>
    @type file
    timekey 1d
    timekey_wait 10m
  </buffer>
</match>

# Output to Elasticsearch (optional)
<match **>
  @type elasticsearch
  host elasticsearch
  port 9200
  index_name docker-logs
</match>
```

---

### Level 4: Prometheus + Grafana

**Complete Monitoring Stack:**

```yaml
version: '3.8'

services:
  # Prometheus
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    networks:
      - monitoring

  # cAdvisor (container metrics)
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    networks:
      - monitoring

  # Node Exporter (host metrics)
  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"
    networks:
      - monitoring

  # Grafana
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - monitoring

networks:
  monitoring:

volumes:
  prometheus-data:
  grafana-data:
```

**Prometheus Configuration (prometheus.yml):**

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  # cAdvisor (container metrics)
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  # Node Exporter (host metrics)
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # Docker daemon metrics
  - job_name: 'docker'
    static_configs:
      - targets: ['host.docker.internal:9323']
```

---

### Level 5: ELK Stack (Elasticsearch, Logstash, Kibana)

```yaml
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    networks:
      - elk

  logstash:
    image: logstash:8.11.0
    ports:
      - "5000:5000"
      - "9600:9600"
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    environment:
      - "LS_JAVA_OPTS=-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - elk
    depends_on:
      - elasticsearch

  # Application with GELF logging
  web:
    image: nginx
    logging:
      driver: gelf
      options:
        gelf-address: "udp://logstash:12201"
        tag: "web"
    networks:
      - app-network
      - elk

networks:
  elk:
  app-network:

volumes:
  es-data:
```

**Logstash Pipeline (logstash.conf):**

```
input {
  gelf {
    port => 12201
  }
}

filter {
  # Parse Docker container name
  mutate {
    add_field => {
      "container_name" => "%{container_name}"
    }
  }

  # Parse nginx logs
  if [tag] == "web" {
    grok {
      match => {
        "message" => "%{COMBINEDAPACHELOG}"
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "docker-logs-%{+YYYY.MM.dd}"
  }
}
```

---

### Level 6: Custom Network Monitoring

**Network Traffic Monitor:**

```python
#!/usr/bin/env python3
# network-monitor.py

import docker
import time
import json

client = docker.from_env()

def monitor_network(network_name):
    network = client.networks.get(network_name)
    
    while True:
        network.reload()
        
        print(f"\n=== Network: {network.name} ===")
        print(f"Driver: {network.attrs['Driver']}")
        print(f"Containers: {len(network.attrs['Containers'])}")
        
        for container_id, details in network.attrs['Containers'].items():
            container = client.containers.get(container_id)
            stats = container.stats(stream=False)
            
            # Network I/O
            net_io = stats['networks']['eth0']
            rx_bytes = net_io['rx_bytes']
            tx_bytes = net_io['tx_bytes']
            
            print(f"\n  {details['Name']}:")
            print(f"    IP: {details['IPv4Address']}")
            print(f"    RX: {rx_bytes / 1024 / 1024:.2f} MB")
            print(f"    TX: {tx_bytes / 1024 / 1024:.2f} MB")
        
        time.sleep(10)

if __name__ == "__main__":
    monitor_network("my-network")
```

---

### Level 7: Alerting

**Alert Script:**

```bash
#!/bin/bash
# network-alerts.sh

THRESHOLD_RX_MBPS=100
THRESHOLD_TX_MBPS=100
ALERT_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK"

while true; do
    # Check each container
    for container in $(docker ps -q); do
        NAME=$(docker inspect -f '{{.Name}}' $container | sed 's/\///')
        
        # Get network stats
        STATS=$(docker stats $container --no-stream --format '{{.NetIO}}')
        
        # Parse RX/TX
        RX=$(echo $STATS | awk '{print $1}' | sed 's/[^0-9.]//g')
        TX=$(echo $STATS | awk '{print $3}' | sed 's/[^0-9.]//g')
        
        # Alert if threshold exceeded
        if (( $(echo "$RX > $THRESHOLD_RX_MBPS" | bc -l) )); then
            curl -X POST $ALERT_WEBHOOK \
                -H 'Content-Type: application/json' \
                -d "{\"text\":\"⚠️ High RX on $NAME: ${RX}MB\"}"
        fi
        
        if (( $(echo "$TX > $THRESHOLD_TX_MBPS" | bc -l) )); then
            curl -X POST $ALERT_WEBHOOK \
                -H 'Content-Type: application/json' \
                -d "{\"text\":\"⚠️ High TX on $NAME: ${TX}MB\"}"
        fi
    done
    
    sleep 60
done
```

---

**Best Practices:**

```
1. Layer Monitoring
   ✓ Container metrics (cAdvisor)
   ✓ Host metrics (Node Exporter)
   ✓ Application logs (Fluentd)
   ✓ Network traffic (tcpdump)

2. Centralize Logs
   ✓ Use Fluentd or Logstash
   ✓ Store in Elasticsearch
   ✓ Visualize in Kibana/Grafana

3. Set Alerts
   ✓ High network traffic
   ✓ Connection failures
   ✓ Unusual patterns

4. Retention Policy
   ✓ Keep recent logs (7-30 days)
   ✓ Archive old logs
   ✓ Rotate files

5. Security
   ✓ Encrypt log transmission
   ✓ Access control
   ✓ Audit log access
```

</details>

</details>

---

[← Previous: 6.1 Network Fundamentals](01-network-fundamentals.md) | [Next: 7.1 Compose Basics →](../07-docker-compose/01-compose-basics.md)