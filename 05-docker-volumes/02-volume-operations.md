# 5.2 Volume Operations

Managing Docker volumes: creating, listing, inspecting, removing, and advanced operations.

---

## Creating Volumes

### docker volume create

**Purpose:** Create a new volume.

**Syntax:**
```bash
docker volume create [OPTIONS] [VOLUME]
```

**Basic Creation:**
```bash
# Create named volume
docker volume create my-volume

# Docker generates unique name
# my-volume

# Create multiple volumes
docker volume create data
docker volume create logs
docker volume create uploads
```

**With Driver:**
```bash
# Specify driver
docker volume create --driver local my-volume

# With driver options
docker volume create \
    --driver local \
    --opt type=tmpfs \
    --opt device=tmpfs \
    --opt o=size=100m \
    tmp-volume
```

**With Labels:**
```bash
# Add metadata labels
docker volume create \
    --label env=production \
    --label app=database \
    --label backup=daily \
    prod-db-volume

# Multiple labels
docker volume create \
    --label project=myapp \
    --label component=cache \
    --label owner=devops \
    cache-volume
```

---

## Listing Volumes

### docker volume ls

**Purpose:** List all volumes.

**Syntax:**
```bash
docker volume ls [OPTIONS]
```

**Basic Usage:**
```bash
# List all volumes
docker volume ls

# Output:
# DRIVER    VOLUME NAME
# local     my-volume
# local     postgres-data
# local     redis-cache
```

**With Filters:**
```bash
# Filter by driver
docker volume ls --filter driver=local

# Filter by label
docker volume ls --filter label=env=production

# Filter by dangling (unused)
docker volume ls --filter dangling=true

# Filter by name pattern
docker volume ls --filter name=postgres*

# Multiple filters
docker volume ls \
    --filter driver=local \
    --filter label=env=production
```

**Custom Format:**
```bash
# Just names
docker volume ls --format '{{.Name}}'

# Names and drivers
docker volume ls --format '{{.Name}}: {{.Driver}}'

# Table format
docker volume ls --format 'table {{.Name}}\t{{.Driver}}\t{{.Labels}}'

# JSON output
docker volume ls --format '{{json .}}'
```

**Quiet Mode:**
```bash
# Just volume names (for scripting)
docker volume ls -q

# Use in scripts
for volume in $(docker volume ls -q); do
    echo "Processing: $volume"
done
```

---

## Inspecting Volumes

### docker volume inspect

**Purpose:** Display detailed information about volumes.

**Syntax:**
```bash
docker volume inspect [OPTIONS] VOLUME [VOLUME...]
```

**Basic Inspection:**
```bash
# Inspect single volume
docker volume inspect my-volume

# Output (JSON):
[
    {
        "CreatedAt": "2024-01-17T10:00:00Z",
        "Driver": "local",
        "Labels": {
            "env": "production"
        },
        "Mountpoint": "/var/lib/docker/volumes/my-volume/_data",
        "Name": "my-volume",
        "Options": {},
        "Scope": "local"
    }
]
```

**Inspect Multiple:**
```bash
# Multiple volumes at once
docker volume inspect vol1 vol2 vol3
```

**Extract Specific Field:**
```bash
# Get mountpoint
docker volume inspect -f '{{.Mountpoint}}' my-volume
# /var/lib/docker/volumes/my-volume/_data

# Get driver
docker volume inspect -f '{{.Driver}}' my-volume
# local

# Get labels
docker volume inspect -f '{{json .Labels}}' my-volume | jq
# {
#   "env": "production",
#   "app": "database"
# }

# Get creation date
docker volume inspect -f '{{.CreatedAt}}' my-volume
# 2024-01-17T10:00:00Z
```

**Scripting Examples:**
```bash
# Check if volume exists
if docker volume inspect my-volume > /dev/null 2>&1; then
    echo "Volume exists"
else
    echo "Volume does not exist"
fi

# Get volume size (requires root)
MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' my-volume)
sudo du -sh "$MOUNTPOINT"

# List all volumes with labels
for vol in $(docker volume ls -q); do
    labels=$(docker volume inspect -f '{{json .Labels}}' $vol)
    echo "$vol: $labels"
done
```

---

## Removing Volumes

### docker volume rm

**Purpose:** Remove one or more volumes.

**Syntax:**
```bash
docker volume rm [OPTIONS] VOLUME [VOLUME...]
```

**Basic Removal:**
```bash
# Remove single volume
docker volume rm my-volume

# Remove multiple volumes
docker volume rm vol1 vol2 vol3

# Force remove (even if in use - not recommended)
docker volume rm -f my-volume
```

**Safe Removal Check:**
```bash
# Check if volume is in use
docker ps -a --filter volume=my-volume

# If no containers listed, safe to remove
docker volume rm my-volume
```

---

### docker volume prune

**Purpose:** Remove all unused volumes.

**Syntax:**
```bash
docker volume prune [OPTIONS]
```

**Basic Prune:**
```bash
# Remove all unused volumes (prompts for confirmation)
docker volume prune

# WARNING! This will remove all volumes not used by at least one container.
# Are you sure you want to continue? [y/N]

# Force (no confirmation)
docker volume prune -f
```

**With Filters:**
```bash
# Prune volumes with specific label
docker volume prune --filter label=temporary=true

# Prune volumes older than 24 hours
docker volume prune --filter until=24h

# Multiple filters
docker volume prune \
    --filter label=env=dev \
    --filter until=48h
```

**Get Info Before Pruning:**
```bash
# See what would be deleted
docker volume ls --filter dangling=true

# Count dangling volumes
docker volume ls --filter dangling=true -q | wc -l

# Then prune
docker volume prune -f
```

---

## Using Volumes in Containers

### Mount Volume at Runtime

```bash
# Named volume
docker run -d \
    --name web \
    -v my-volume:/app/data \
    nginx

# Multiple volumes
docker run -d \
    --name web \
    -v data:/app/data \
    -v logs:/app/logs \
    -v config:/etc/myapp \
    nginx

# Read-only volume
docker run -d \
    --name web \
    -v my-volume:/app/data:ro \
    nginx

# With --mount (more explicit)
docker run -d \
    --name web \
    --mount source=my-volume,target=/app/data \
    nginx

# Read-only with --mount
docker run -d \
    --name web \
    --mount source=my-volume,target=/app/data,readonly \
    nginx
```

### Volume from Another Container

```bash
# Create data container
docker create --name data-container \
    -v data-volume:/data \
    alpine

# Use volumes from data-container
docker run -d \
    --name web \
    --volumes-from data-container \
    nginx

# Multiple containers sharing same volumes
docker run -d --name app1 --volumes-from data-container myapp
docker run -d --name app2 --volumes-from data-container myapp
docker run -d --name app3 --volumes-from data-container myapp
```

---

## Advanced Operations

### Copying Data Between Volumes

```bash
# Copy from volume1 to volume2
docker run --rm \
    -v volume1:/from \
    -v volume2:/to \
    alpine \
    sh -c "cp -av /from/. /to/"

# Verify copy
docker run --rm \
    -v volume2:/data \
    alpine ls -la /data
```

### Cloning Volumes

```bash
# Clone volume
VOLUME_NAME="my-volume"
CLONE_NAME="my-volume-clone"

# Create new volume
docker volume create $CLONE_NAME

# Copy data
docker run --rm \
    -v $VOLUME_NAME:/source:ro \
    -v $CLONE_NAME:/dest \
    alpine \
    sh -c "cp -av /source/. /dest/"

# Verify
docker volume inspect $CLONE_NAME
```

### Volume Size

```bash
# Get volume mountpoint
MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' my-volume)

# Check size (requires root)
sudo du -sh "$MOUNTPOINT"

# Detailed breakdown
sudo du -h "$MOUNTPOINT" | sort -rh | head -20

# For all volumes
for vol in $(docker volume ls -q); do
    MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' $vol)
    SIZE=$(sudo du -sh "$MOUNTPOINT" 2>/dev/null | cut -f1)
    echo "$vol: $SIZE"
done
```

### Migrating Volumes

**Between containers:**
```bash
# Old container with volume
docker stop old-container
docker rm old-container

# New container with same volume
docker run -d \
    --name new-container \
    -v old-volume:/data \
    new-image
```

**Between hosts:**
```bash
# Export on source host
docker run --rm \
    -v my-volume:/data \
    alpine \
    tar cz -C /data . | ssh user@target-host \
    "docker run --rm -i -v my-volume:/data alpine tar xz -C /data"

# Or save to file and transfer
docker run --rm \
    -v my-volume:/data \
    alpine tar cz -C /data . > volume-backup.tar.gz

scp volume-backup.tar.gz user@target-host:/tmp/

# Import on target host
docker volume create my-volume
docker run --rm -i \
    -v my-volume:/data \
    alpine tar xz -C /data < /tmp/volume-backup.tar.gz
```

---

## Real-World Patterns

### Pattern 1: Database with Backup

```bash
# Create volume
docker volume create postgres-data

# Run database
docker run -d \
    --name postgres \
    -v postgres-data:/var/lib/postgresql/data \
    -e POSTGRES_PASSWORD=secret \
    postgres:15

# Regular backup
docker run --rm \
    -v postgres-data:/data:ro \
    -v $(pwd)/backups:/backup \
    alpine \
    tar czf /backup/db-$(date +%Y%m%d).tar.gz -C /data .

# Restore from backup
docker stop postgres
docker run --rm \
    -v postgres-data:/data \
    -v $(pwd)/backups:/backup \
    alpine \
    tar xzf /backup/db-20240117.tar.gz -C /data
docker start postgres
```

### Pattern 2: Shared Volume for Microservices

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx
    volumes:
      - static-files:/usr/share/nginx/html:ro
    ports:
      - "80:80"

  app:
    image: myapp
    volumes:
      - static-files:/app/static
    command: python manage.py collectstatic

  worker:
    image: myapp
    volumes:
      - static-files:/app/static:ro
    command: celery worker

volumes:
  static-files:
```

### Pattern 3: Volume Cleanup Script

```bash
#!/bin/bash
# cleanup-volumes.sh

# Remove dangling volumes
echo "Removing dangling volumes..."
docker volume prune -f

# Remove old volumes (by label)
echo "Removing old dev volumes..."
docker volume prune --filter label=temporary=true -f

# Remove large volumes
echo "Checking large volumes..."
for vol in $(docker volume ls -q); do
    MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' $vol)
    SIZE_MB=$(sudo du -sm "$MOUNTPOINT" 2>/dev/null | cut -f1)
    
    if [ "$SIZE_MB" -gt 5000 ]; then
        echo "Large volume found: $vol ($SIZE_MB MB)"
        # Check if in use
        if ! docker ps -a --filter volume=$vol --format '{{.Names}}' | grep -q .; then
            echo "Not in use. Consider removing: docker volume rm $vol"
        fi
    fi
done
```

### Pattern 4: Volume Monitoring

```bash
#!/bin/bash
# monitor-volumes.sh

echo "=== Docker Volume Status ==="
echo ""

# Total volumes
TOTAL=$(docker volume ls -q | wc -l)
echo "Total volumes: $TOTAL"

# Dangling volumes
DANGLING=$(docker volume ls --filter dangling=true -q | wc -l)
echo "Dangling volumes: $DANGLING"

# In-use volumes
IN_USE=$((TOTAL - DANGLING))
echo "In-use volumes: $IN_USE"

# Total size
TOTAL_SIZE=0
for vol in $(docker volume ls -q); do
    MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' $vol)
    SIZE=$(sudo du -sb "$MOUNTPOINT" 2>/dev/null | cut -f1)
    TOTAL_SIZE=$((TOTAL_SIZE + SIZE))
done

TOTAL_SIZE_GB=$(echo "scale=2; $TOTAL_SIZE / 1024 / 1024 / 1024" | bc)
echo "Total size: ${TOTAL_SIZE_GB}GB"

echo ""
echo "=== Largest Volumes ==="
for vol in $(docker volume ls -q); do
    MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' $vol)
    SIZE=$(sudo du -sh "$MOUNTPOINT" 2>/dev/null | cut -f1)
    echo "$vol: $SIZE"
done | sort -rh | head -10
```

### Pattern 5: Development Volume Setup

```yaml
# docker-compose.yml for development
version: '3.8'

services:
  web:
    build: .
    volumes:
      # Source code (bind mount)
      - ./src:/app/src:cached
      
      # Dependencies (volume for speed)
      - node_modules:/app/node_modules
      
      # Build output (volume)
      - build:/app/build
      
      # Logs (bind mount for easy access)
      - ./logs:/app/logs
      
    environment:
      - NODE_ENV=development

volumes:
  node_modules:
  build:
```

---

## Troubleshooting

### Volume Not Mounting

```bash
# Check volume exists
docker volume ls | grep my-volume

# Inspect volume
docker volume inspect my-volume

# Check container mounts
docker inspect -f '{{json .Mounts}}' my-container | jq

# Verify data in volume
docker run --rm -v my-volume:/data alpine ls -la /data
```

### Volume Permission Issues

```bash
# Check ownership in volume
docker run --rm -v my-volume:/data alpine ls -ln /data

# Fix permissions
docker run --rm -v my-volume:/data alpine chown -R 1000:1000 /data

# Or run container as specific user
docker run -u 1000:1000 -v my-volume:/data myapp
```

### Cannot Remove Volume

```bash
# Error: volume is in use

# Find containers using volume
docker ps -a --filter volume=my-volume

# Stop and remove containers
docker stop $(docker ps -a --filter volume=my-volume -q)
docker rm $(docker ps -a --filter volume=my-volume -q)

# Now remove volume
docker volume rm my-volume
```

### Volume Data Lost

```bash
# Check if volume still exists
docker volume ls | grep my-volume

# Check if correct volume mounted
docker inspect -f '{{json .Mounts}}' my-container | jq

# Common mistake: typo in volume name
# Creates new volume instead of using existing
docker run -v myvolume:/data nginx  # Creates 'myvolume'
docker run -v my-volume:/data nginx # Uses 'my-volume'
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The command __________ creates a new Docker volume.
2. Use __________ to remove all unused volumes.
3. The __________ option mounts a volume as read-only.
4. To inspect a volume's details, use __________.
5. __________ removes one or more specific volumes.
6. Volumes are stored at __________ on Linux by default.

### True/False

1. ⬜ docker volume prune removes all volumes including those in use
2. ⬜ You can mount the same volume to multiple containers
3. ⬜ Removing a container automatically removes its volumes
4. ⬜ Volumes can have labels for organization
5. ⬜ Anonymous volumes get automatically generated names
6. ⬜ You need root access to inspect volume contents on the host
7. ⬜ Volumes can be created implicitly when running containers

### Multiple Choice

1. Which command lists all volumes?
   - A) docker volume list
   - B) docker volume ls
   - C) docker volumes
   - D) docker ls volumes

2. How do you mount a volume as read-only?
   - A) -v volume:/path:r
   - B) -v volume:/path:readonly
   - C) -v volume:/path:ro
   - D) --readonly volume:/path

3. What happens to volumes when using docker volume prune?
   - A) All volumes are removed
   - B) Only unused volumes are removed
   - C) Only anonymous volumes are removed
   - D) Nothing, it just lists volumes

4. How can you share volumes between containers?
   - A) Use same volume name in -v flag
   - B) Use --volumes-from
   - C) Use --mount with same source
   - D) All of the above

5. Where does docker volume inspect show the mountpoint?
   - A) In the Mountpoint field
   - B) In the Path field
   - C) In the Location field
   - D) It doesn't show mountpoint

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. docker volume create
2. docker volume prune
3. :ro (or ro, readonly)
4. docker volume inspect
5. docker volume rm
6. /var/lib/docker/volumes

**True/False:**
1. ❌ False - Only removes unused volumes
2. ✅ True - Volumes can be shared across containers
3. ❌ False - Named volumes persist; use -v flag to auto-remove anonymous
4. ✅ True - Volumes support labels via --label flag
5. ✅ True - Get random hash-like names
6. ✅ True - Volume data directory requires root access
7. ✅ True - Created automatically if doesn't exist

**Multiple Choice:**
1. **B** - docker volume ls
2. **C** - -v volume:/path:ro
3. **B** - Only unused volumes are removed
4. **D** - All of the above
5. **A** - In the Mountpoint field

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: How do you find and clean up unused volumes safely?

<details>
<summary><strong>View Answer</strong></summary>

**Comprehensive Volume Cleanup Strategy:**

---

**Step 1: Identify Unused Volumes**

```bash
# List all dangling (unused) volumes
docker volume ls --filter dangling=true

# Output shows volumes not attached to any container:
# DRIVER    VOLUME NAME
# local     old-db-data
# local     tmp-cache-123
# local     abandoned-volume
```

**Step 2: Understand What "Unused" Means**

```
Unused (Dangling) = Volume not mounted to ANY container
  - Not used by running containers
  - Not used by stopped containers
  - Not used by any container

Safe to remove = Truly unused
Careful with = Might be needed for stopped containers
```

**Step 3: Safe Investigation**

```bash
# Check each volume before removal
for vol in $(docker volume ls --filter dangling=true -q); do
    echo "=== Volume: $vol ==="
    
    # Check size
    MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' $vol)
    SIZE=$(sudo du -sh "$MOUNTPOINT" 2>/dev/null | cut -f1)
    echo "Size: $SIZE"
    
    # Check creation date
    CREATED=$(docker volume inspect -f '{{.CreatedAt}}' $vol)
    echo "Created: $CREATED"
    
    # Check labels
    LABELS=$(docker volume inspect -f '{{json .Labels}}' $vol)
    echo "Labels: $LABELS"
    
    # Verify no containers reference it
    REFS=$(docker ps -a --filter volume=$vol --format '{{.Names}}')
    if [ -z "$REFS" ]; then
        echo "Status: No containers reference this volume"
    else
        echo "Status: Referenced by: $REFS"
    fi
    
    echo ""
done
```

---

**Step 4: Preview What Will Be Removed**

```bash
# Dry run - see what would be deleted
echo "Volumes that would be pruned:"
docker volume ls --filter dangling=true

# Count
COUNT=$(docker volume ls --filter dangling=true -q | wc -l)
echo "Total volumes to be removed: $COUNT"

# Calculate total size
TOTAL_SIZE=0
for vol in $(docker volume ls --filter dangling=true -q); do
    MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' $vol)
    SIZE=$(sudo du -sb "$MOUNTPOINT" 2>/dev/null | cut -f1)
    TOTAL_SIZE=$((TOTAL_SIZE + SIZE))
done
TOTAL_GB=$(echo "scale=2; $TOTAL_SIZE / 1024 / 1024 / 1024" | bc)
echo "Total space to be freed: ${TOTAL_GB}GB"
```

---

**Step 5: Safe Cleanup Options**

**Option A: Interactive Removal**
```bash
# Remove volumes one by one with confirmation
for vol in $(docker volume ls --filter dangling=true -q); do
    echo "Volume: $vol"
    
    # Show info
    docker volume inspect $vol | jq
    
    # Ask for confirmation
    read -p "Remove this volume? (y/N) " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        docker volume rm $vol
        echo "✓ Removed"
    else
        echo "✗ Skipped"
    fi
done
```

**Option B: Filtered Cleanup**
```bash
# Remove only volumes with specific label
docker volume prune --filter label=temporary=true -f

# Remove volumes older than 7 days
docker volume prune --filter until=168h -f

# Remove old development volumes
docker volume prune --filter label=env=development -f
```

**Option C: Size-Based Cleanup**
```bash
# Remove large unused volumes
for vol in $(docker volume ls --filter dangling=true -q); do
    MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' $vol)
    SIZE_MB=$(sudo du -sm "$MOUNTPOINT" 2>/dev/null | cut -f1)
    
    # Remove if larger than 1GB
    if [ "$SIZE_MB" -gt 1024 ]; then
        echo "Removing large volume: $vol ($SIZE_MB MB)"
        docker volume rm $vol
    fi
done
```

---

**Step 6: Full Cleanup Script**

```bash
#!/bin/bash
# safe-volume-cleanup.sh

set -e

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo "=== Docker Volume Cleanup Utility ==="
echo ""

# Find dangling volumes
DANGLING=$(docker volume ls --filter dangling=true -q)

if [ -z "$DANGLING" ]; then
    echo "${GREEN}No unused volumes found. Nothing to clean up.${NC}"
    exit 0
fi

# Count and size
COUNT=$(echo "$DANGLING" | wc -l)
TOTAL_SIZE=0

echo "Found $COUNT unused volumes:"
echo ""

# List each volume with details
for vol in $DANGLING; do
    MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' $vol)
    SIZE=$(sudo du -sh "$MOUNTPOINT" 2>/dev/null | cut -f1)
    CREATED=$(docker volume inspect -f '{{.CreatedAt}}' $vol)
    
    echo "  - $vol"
    echo "    Size: $SIZE"
    echo "    Created: $CREATED"
    
    # Calculate total
    SIZE_BYTES=$(sudo du -sb "$MOUNTPOINT" 2>/dev/null | cut -f1)
    TOTAL_SIZE=$((TOTAL_SIZE + SIZE_BYTES))
done

# Show total
TOTAL_GB=$(echo "scale=2; $TOTAL_SIZE / 1024 / 1024 / 1024" | bc)
echo ""
echo "Total space to be freed: ${YELLOW}${TOTAL_GB}GB${NC}"
echo ""

# Confirm
read -p "Proceed with cleanup? (y/N) " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    echo "${YELLOW}Cleanup cancelled.${NC}"
    exit 0
fi

# Cleanup
echo ""
echo "Cleaning up..."
docker volume prune -f

echo ""
echo "${GREEN}✓ Cleanup complete. Freed ${TOTAL_GB}GB${NC}"
```

---

**Step 7: Automated Cleanup (Cron)**

```bash
#!/bin/bash
# auto-cleanup-volumes.sh

# Log file
LOG="/var/log/docker-volume-cleanup.log"

echo "=== Volume Cleanup: $(date) ===" >> $LOG

# Remove volumes older than 30 days
docker volume prune --filter until=720h -f >> $LOG 2>&1

# Remove volumes with temporary label
docker volume prune --filter label=temporary=true -f >> $LOG 2>&1

echo "Cleanup complete" >> $LOG
echo "" >> $LOG
```

**Cron job:**
```bash
# Run weekly on Sunday at 2 AM
0 2 * * 0 /usr/local/bin/auto-cleanup-volumes.sh
```

---

**Step 8: Prevent Unused Volumes**

**Pattern 1: Use named volumes consistently**
```yaml
# docker-compose.yml
services:
  db:
    image: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data  # Named

volumes:
  postgres-data:  # Explicit declaration
```

**Pattern 2: Label volumes for management**
```bash
# Create volumes with labels
docker volume create \
    --label project=myapp \
    --label temporary=false \
    --label backup-schedule=daily \
    myapp-data

# Easy to find and manage
docker volume ls --filter label=project=myapp
```

**Pattern 3: Cleanup in development**
```bash
# Use --rm and -v flags
docker run --rm -v /data myapp

# This removes anonymous volume when container exits
```

---

**Warning Signs to Check:**

```bash
# 1. Rapidly growing volume count
CURRENT=$(docker volume ls -q | wc -l)
YESTERDAY=$(cat /tmp/volume-count 2>/dev/null || echo 0)
echo $CURRENT > /tmp/volume-count

if [ $CURRENT -gt $((YESTERDAY + 10)) ]; then
    echo "⚠️  Volume count increased by $(($CURRENT - $YESTERDAY))"
    echo "Investigate: docker volume ls --filter dangling=true"
fi

# 2. Large dangling volumes
for vol in $(docker volume ls --filter dangling=true -q); do
    MOUNTPOINT=$(docker volume inspect -f '{{.Mountpoint}}' $vol)
    SIZE_GB=$(sudo du -sg "$MOUNTPOINT" 2>/dev/null | cut -f1)
    
    if [ "$SIZE_GB" -gt 5 ]; then
        echo "⚠️  Large unused volume: $vol (${SIZE_GB}GB)"
    fi
done
```

---

**Best Practices Summary:**

```
1. Regular monitoring
   ✓ Check weekly for dangling volumes
   ✓ Monitor total disk usage
   ✓ Track volume growth

2. Safe removal process
   ✓ Inspect before removing
   ✓ Check labels and metadata
   ✓ Verify no containers need it
   ✓ Back up important data first

3. Prevention
   ✓ Use named volumes
   ✓ Label volumes appropriately
   ✓ Use --rm for temporary containers
   ✓ Document volume purposes

4. Automation
   ✓ Schedule regular cleanup
   ✓ Set retention policies
   ✓ Alert on unusual growth
   ✓ Log all cleanup operations
```

</details>

### Question 2: Explain how to share data between containers using volumes

<details>
<summary><strong>View Answer</strong></summary>

**Multiple Methods for Sharing Data:**

---

### Method 1: Shared Named Volume (Most Common)

**How it works:**
```
Volume created once, mounted to multiple containers

┌─────────────┐
│   Volume    │
│  my-data    │
└──────┬──────┘
       │
       ├──────────┬──────────┬──────────┐
       │          │          │          │
       ▼          ▼          ▼          ▼
   Container  Container  Container  Container
      1          2          3          4
```

**Implementation:**
```bash
# Create shared volume
docker volume create shared-data

# Container 1: Producer (writes data)
docker run -d \
    --name producer \
    -v shared-data:/data \
    myapp generate-data

# Container 2: Consumer (reads data)
docker run -d \
    --name consumer \
    -v shared-data:/data:ro \
    myapp process-data

# Container 3: Another consumer
docker run -d \
    --name analyzer \
    -v shared-data:/data:ro \
    myapp analyze-data
```

**Use Case: Static Files**
```yaml
version: '3.8'

services:
  # App generates static files
  app:
    image: myapp
    volumes:
      - static-files:/app/static
    command: python manage.py collectstatic

  # Nginx serves static files
  nginx:
    image: nginx
    volumes:
      - static-files:/usr/share/nginx/html:ro
    ports:
      - "80:80"

volumes:
  static-files:
```

---

### Method 2: Using --volumes-from

**How it works:**
```
Data container defines volumes
Other containers inherit its volumes
```

**Implementation:**
```bash
# Create data container
docker create --name data-container \
    -v /data \
    -v /logs \
    -v /config \
    alpine

# App container uses volumes
docker run -d \
    --name app \
    --volumes-from data-container \
    myapp

# Worker uses same volumes
docker run -d \
    --name worker \
    --volumes-from data-container \
    myapp worker

# Logger uses same volumes
docker run -d \
    --name logger \
    --volumes-from data-container \
    myapp logger
```

**Pattern: Data Container**
```yaml
version: '3.8'

services:
  data:
    image: alpine
    volumes:
      - app-data:/data
      - app-logs:/logs
    command: "true"  # Just creates volumes

  app:
    image: myapp
    volumes_from:
      - data

  worker:
    image: myapp
    volumes_from:
      - data
    command: celery worker

volumes:
  app-data:
  app-logs:
```

---

### Method 3: Read-Write vs Read-Only

**Access Control:**
```bash
# One writer, multiple readers
docker run -d \
    --name writer \
    -v shared-data:/data \
    myapp write

docker run -d \
    --name reader1 \
    -v shared-data:/data:ro \
    myapp read

docker run -d \
    --name reader2 \
    -v shared-data:/data:ro \
    myapp read

# Prevents readers from modifying data
```

**Real Example: Log Processing**
```yaml
services:
  # Application writes logs
  app:
    image: myapp
    volumes:
      - logs:/var/log/myapp  # Read-write

  # Log shipper reads logs
  logstash:
    image: logstash
    volumes:
      - logs:/logs:ro  # Read-only

  # Log analyzer reads logs
  analyzer:
    image: analyzer
    volumes:
      - logs:/logs:ro  # Read-only

volumes:
  logs:
```

---

### Method 4: Multi-Stage Data Pipeline

**Sequential Processing:**
```yaml
version: '3.8'

services:
  # Stage 1: Extract
  extract:
    image: extractor
    volumes:
      - raw-data:/output
    command: extract-data

  # Stage 2: Transform
  transform:
    image: transformer
    volumes:
      - raw-data:/input:ro
      - processed-data:/output
    command: transform-data
    depends_on:
      - extract

  # Stage 3: Load
  load:
    image: loader
    volumes:
      - processed-data:/input:ro
    command: load-data
    depends_on:
      - transform

volumes:
  raw-data:
  processed-data:
```

---

### Real-World Examples

**Example 1: WordPress + MySQL**
```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8
    volumes:
      - db-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret

  wordpress:
    image: wordpress
    volumes:
      - wp-content:/var/www/html/wp-content
      - wp-uploads:/var/www/html/wp-content/uploads
    ports:
      - "80:80"
    depends_on:
      - mysql

  # Backup service
  backup:
    image: backup-tool
    volumes:
      - db-data:/db-data:ro
      - wp-uploads:/wp-uploads:ro
      - ./backups:/backups
    command: backup-all

volumes:
  db-data:       # MySQL database files
  wp-content:    # WordPress content
  wp-uploads:    # Shared with backup
```

---

**Example 2: Microservices**
```yaml
version: '3.8'

services:
  # API generates reports
  api:
    image: api
    volumes:
      - reports:/app/reports
      - uploads:/app/uploads

  # Worker processes reports
  worker:
    image: worker
    volumes:
      - reports:/app/reports:ro
      - uploads:/app/uploads:ro

  # Web serves files
  web:
    image: nginx
    volumes:
      - reports:/usr/share/nginx/html/reports:ro
      - uploads:/usr/share/nginx/html/uploads:ro
    ports:
      - "80:80"

  # Cleanup service
  cleanup:
    image: alpine
    volumes:
      - reports:/reports
    command: sh -c "while true; do find /reports -mtime +30 -delete; sleep 86400; done"

volumes:
  reports:
  uploads:
```

---

**Example 3: Development Environment**
```yaml
version: '3.8'

services:
  # Node.js app
  app:
    build: .
    volumes:
      # Source code (bind mount)
      - ./src:/app/src:cached
      
      # Dependencies (shared volume)
      - node_modules:/app/node_modules
      
      # Build output (shared volume)
      - build:/app/build

  # Builder service
  builder:
    image: node:18
    volumes:
      - ./src:/app/src:cached
      - node_modules:/app/node_modules
      - build:/app/build
    working_dir: /app
    command: npm run build

  # Test runner
  test:
    image: node:18
    volumes:
      - ./src:/app/src:cached
      - node_modules:/app/node_modules
    working_dir: /app
    command: npm test

volumes:
  node_modules:  # Shared dependencies
  build:         # Shared build output
```

---

### Synchronization and Conflicts

**Handling Concurrent Writes:**

```
Problem: Multiple containers writing to same file

Container A: Writes to /data/file.txt
Container B: Writes to /data/file.txt
Result: Race condition, corruption possible
```

**Solutions:**

**1. Coordinator Pattern**
```yaml
services:
  coordinator:
    image: myapp
    volumes:
      - data:/data
    command: coordinator  # Single writer

  worker1:
    image: myapp
    volumes:
      - data:/data:ro  # Read-only
    command: worker

  worker2:
    image: myapp
    volumes:
      - data:/data:ro  # Read-only
    command: worker
```

**2. Separate Directories**
```yaml
services:
  worker1:
    volumes:
      - data:/data/worker1  # Own directory

  worker2:
    volumes:
      - data:/data/worker2  # Own directory

  aggregator:
    volumes:
      - data:/data:ro  # Reads all directories
```

**3. File Locking**
```python
# In application code
import fcntl

with open('/data/file.txt', 'w') as f:
    fcntl.flock(f, fcntl.LOCK_EX)
    f.write('data')
    fcntl.flock(f, fcntl.LOCK_UN)
```

---

### Best Practices

```
1. Access Control
   ✓ One writer, multiple readers
   ✓ Use :ro for read-only access
   ✓ Prevent accidental modification

2. Clear Ownership
   ✓ Document which service writes
   ✓ Document which services read
   ✓ Prevent conflicts

3. Volume Naming
   ✓ Descriptive names
   ✓ Indicate purpose
   ✓ Include project/app name

4. Cleanup
   ✓ Remove data when services stop
   ✓ Regular maintenance
   ✓ Monitor volume growth

5. Performance
   ✓ Minimize concurrent writes
   ✓ Use appropriate volume drivers
   ✓ Consider caching strategies
```

</details>

</details>

---

[← Previous: 5.1 Data Persistence](01-data-persistence.md) | [Next: 6.1 Network Basics →](../06-docker-networking/01-network-basics.md)