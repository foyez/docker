# 5.1 Data Persistence

Understanding Docker data persistence: volumes, bind mounts, tmpfs, and when to use each.

---

## The Problem: Container Ephemeral Storage

**Containers are ephemeral** - when removed, all data inside is lost.

### Container Filesystem

```
Container Filesystem (Writable Layer):
┌─────────────────────────────────┐
│  Writable Layer (temporary)     │  ← Lost on removal
├─────────────────────────────────┤
│  Image Layers (read-only)       │  ← Preserved
└─────────────────────────────────┘

When container is removed:
✗ Writable layer deleted
✗ All data created/modified lost
✗ Database data lost
✗ Uploaded files lost
✗ Generated reports lost
```

**Example:**
```bash
# Start database container
docker run -d --name postgres postgres

# Add data
docker exec postgres psql -U postgres -c \
    "CREATE TABLE users (id INT, name TEXT);"
docker exec postgres psql -U postgres -c \
    "INSERT INTO users VALUES (1, 'Alice');"

# Remove container
docker rm -f postgres

# Start new container
docker run -d --name postgres postgres

# Data is gone!
docker exec postgres psql -U postgres -c \
    "SELECT * FROM users;"
# ERROR: relation "users" does not exist
```

---

## Data Persistence Solutions

Docker provides three ways to persist data:

```
1. Volumes (Recommended)
   - Managed by Docker
   - Best performance
   - Easy backup
   - Works on all platforms

2. Bind Mounts
   - Mount host directory
   - Direct file access
   - Development use
   - Platform-specific paths

3. tmpfs Mounts
   - In-memory only
   - Very fast
   - Not persisted
   - Sensitive data
```

---

## Docker Volumes

**Volumes** are the preferred way to persist data in Docker.

### What are Volumes?

```
Volume = Storage managed by Docker

Location (Linux): /var/lib/docker/volumes/
Managed by: Docker
Independent of: Container lifecycle
Portable: Yes
```

### Volume Lifecycle

```
┌──────────────────────────────────────┐
│         Docker Host                  │
│                                      │
│  /var/lib/docker/volumes/           │
│  ├── my-volume/                     │
│  │   └── _data/                     │
│  │       ├── file1.txt              │
│  │       └── file2.txt              │
│                                      │
│  Container 1                         │
│  └── /app/data ──────────┐          │
│                          │          │
│  Container 2             │          │
│  └── /backup/data ───────┼───────┐  │
│                          │       │  │
│                          ▼       ▼  │
│                    my-volume        │
└──────────────────────────────────────┘

Volume persists:
✓ After container stops
✓ After container is removed
✓ Can be shared between containers
✓ Until explicitly deleted
```

### Creating Volumes

**Explicit Creation:**
```bash
# Create named volume
docker volume create my-volume

# Create with driver options
docker volume create \
    --driver local \
    --opt type=nfs \
    --opt o=addr=192.168.1.1,rw \
    --opt device=:/path/to/dir \
    my-nfs-volume

# Create with labels
docker volume create \
    --label env=production \
    --label app=database \
    prod-db-volume
```

**Implicit Creation:**
```bash
# Volume created automatically if doesn't exist
docker run -v my-volume:/data nginx

# Anonymous volume (auto-generated name)
docker run -v /data nginx
```

### Using Volumes

**Named Volumes:**
```bash
# Mount named volume
docker run -d \
    --name postgres \
    -v postgres-data:/var/lib/postgresql/data \
    postgres

# Volume structure:
# Host: /var/lib/docker/volumes/postgres-data/_data
# Container: /var/lib/postgresql/data
```

**Anonymous Volumes:**
```bash
# Docker generates name
docker run -d \
    --name postgres \
    -v /var/lib/postgresql/data \
    postgres

# Volume gets random name like:
# a1b2c3d4e5f6...
```

**Read-Only Volumes:**
```bash
# Mount as read-only
docker run -v my-volume:/data:ro nginx

# Container can read, cannot write
```

**Volume with Options:**
```bash
# Multiple options
docker run -v my-volume:/data:rw,Z nginx

# Options:
# rw = read-write (default)
# ro = read-only
# z = shared selinux label
# Z = private selinux label
```

---

## Bind Mounts

**Bind mounts** map a host directory to a container directory.

### What are Bind Mounts?

```
Bind Mount = Direct host directory mount

Host Path: Any directory on host
Container Path: Any directory in container
Management: Manual (not managed by Docker)
Performance: May be slower (especially on Mac/Windows)
```

### Using Bind Mounts

**Basic Syntax:**
```bash
# Mount host directory
docker run -v /host/path:/container/path nginx

# Or with --mount (preferred)
docker run --mount type=bind,source=/host/path,target=/container/path nginx
```

**Real-World Examples:**

**1. Development - Source Code:**
```bash
# Mount source code for live reload
docker run -d \
    --name dev \
    -v $(pwd)/src:/app/src \
    -p 3000:3000 \
    node:18 npm run dev

# Changes to local files immediately reflected in container
```

**2. Configuration Files:**
```bash
# Mount nginx config
docker run -d \
    --name nginx \
    -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
    -p 80:80 \
    nginx

# Can edit nginx.conf on host
# Reload: docker exec nginx nginx -s reload
```

**3. Log Files:**
```bash
# Mount log directory
docker run -d \
    --name app \
    -v /var/log/myapp:/app/logs \
    myapp

# Logs written to /var/log/myapp on host
# Easy to monitor, backup, rotate
```

**4. Data Exchange:**
```bash
# Share data between host and container
docker run --rm \
    -v $(pwd)/input:/input:ro \
    -v $(pwd)/output:/output \
    image-processor

# Read from ./input
# Write to ./output
```

---

## tmpfs Mounts

**tmpfs mounts** store data in host memory (RAM).

### What are tmpfs Mounts?

```
tmpfs = Temporary filesystem in memory

Storage: Host RAM
Persistence: None (lost on stop)
Speed: Very fast
Use case: Sensitive temporary data
```

### Using tmpfs

```bash
# Create tmpfs mount
docker run -d \
    --tmpfs /tmp \
    nginx

# With options
docker run -d \
    --tmpfs /tmp:rw,size=64m,mode=1770 \
    nginx

# Or with --mount
docker run -d \
    --mount type=tmpfs,target=/tmp,tmpfs-size=64m \
    nginx
```

**Use Cases:**

```bash
# 1. Sensitive data (passwords, tokens)
docker run -d \
    --tmpfs /run/secrets:mode=700 \
    myapp

# 2. Cache/temporary processing
docker run -d \
    --tmpfs /cache:size=1g \
    processor

# 3. Build artifacts (CI/CD)
docker run --rm \
    --tmpfs /tmp \
    -v $(pwd):/src \
    builder
```

---

## Comparison: Volumes vs Bind Mounts vs tmpfs

### Feature Comparison

| Feature | Volumes | Bind Mounts | tmpfs |
|---------|---------|-------------|-------|
| **Managed by** | Docker | User | Docker |
| **Host location** | /var/lib/docker/volumes | Any path | RAM |
| **Container location** | Any path | Any path | Any path |
| **Persistence** | Yes | Yes | No |
| **Performance** | Best | Good | Fastest |
| **Backup** | Easy | Manual | N/A |
| **Sharing** | Easy | Possible | No |
| **Platform** | All | Path-dependent | Linux/Mac |
| **Production** | ✓ Recommended | Use sparingly | Specific cases |
| **Development** | ✓ Good | ✓ Great | Testing |

### When to Use Each

**Use Volumes:**
```
✓ Production databases
✓ Application data
✓ User uploads
✓ Anything that needs persistence
✓ Data shared between containers
✓ Easy backup/restore needed
```

**Use Bind Mounts:**
```
✓ Development (source code)
✓ Configuration files
✓ Sharing data with host
✓ Testing with host data
✓ CI/CD pipelines (accessing workspace)
```

**Use tmpfs:**
```
✓ Secrets/sensitive data (temporary)
✓ Cache that doesn't need persistence
✓ Build artifacts
✓ Session data
✓ Performance-critical temp files
```

---

## Real-World Examples

### Example 1: PostgreSQL Database

```bash
# Create volume for database
docker volume create postgres-data

# Run PostgreSQL with volume
docker run -d \
    --name postgres \
    -e POSTGRES_PASSWORD=secret \
    -v postgres-data:/var/lib/postgresql/data \
    -p 5432:5432 \
    postgres:15

# Add data
docker exec -it postgres psql -U postgres -c \
    "CREATE TABLE users (id INT, name TEXT);"
docker exec -it postgres psql -U postgres -c \
    "INSERT INTO users VALUES (1, 'Alice'), (2, 'Bob');"

# Stop and remove container
docker stop postgres
docker rm postgres

# Start new container with same volume
docker run -d \
    --name postgres \
    -e POSTGRES_PASSWORD=secret \
    -v postgres-data:/var/lib/postgresql/data \
    -p 5432:5432 \
    postgres:15

# Data is still there!
docker exec -it postgres psql -U postgres -c \
    "SELECT * FROM users;"
#  id | name  
# ----+-------
#   1 | Alice
#   2 | Bob
```

---

### Example 2: Web Application Development

```bash
# Project structure:
project/
├── src/
│   ├── app.js
│   └── views/
├── package.json
└── docker-compose.yml

# docker-compose.yml
version: '3.8'

services:
  web:
    image: node:18
    working_dir: /app
    command: npm run dev
    ports:
      - "3000:3000"
    volumes:
      # Source code (bind mount for development)
      - ./src:/app/src
      - ./package.json:/app/package.json
      
      # Dependencies (volume for performance)
      - node-modules:/app/node_modules
      
      # Logs (bind mount for easy access)
      - ./logs:/app/logs
    environment:
      - NODE_ENV=development

volumes:
  node-modules:
```

**Why this setup:**
```
./src → Bind mount
  - Edit locally, see changes immediately
  - Hot reload works

node-modules → Volume
  - Better performance (especially on Mac/Windows)
  - Consistent dependencies

./logs → Bind mount
  - Easy to view/analyze on host
  - Log rotation on host
```

---

### Example 3: Multi-Container Application

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx
    volumes:
      # Static files (volume shared with app)
      - static-files:/usr/share/nginx/html
      # Config (bind mount)
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "80:80"

  app:
    image: myapp
    volumes:
      # Static files (shared with nginx)
      - static-files:/app/static
      # Uploads (persistent)
      - uploads:/app/uploads
      # Temp files (tmpfs)
      - type: tmpfs
        target: /tmp
    environment:
      - DATABASE_URL=postgres://db/myapp

  db:
    image: postgres:15
    volumes:
      # Database data (persistent)
      - db-data:/var/lib/postgresql/data
      # Init scripts (bind mount)
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    environment:
      - POSTGRES_PASSWORD=secret

  backup:
    image: backup-tool
    volumes:
      # Database volume (read-only)
      - db-data:/data:ro
      # Backup destination (bind mount)
      - ./backups:/backups
    command: backup /data /backups

volumes:
  static-files:    # Shared between nginx and app
  uploads:         # User uploads
  db-data:         # Database persistence
```

---

### Example 4: Data Processing Pipeline

```bash
# Process CSV files
docker run --rm \
    -v $(pwd)/input:/input:ro \
    -v $(pwd)/output:/output \
    -v processing-cache:/cache \
    --tmpfs /tmp:size=2g \
    data-processor

# How it works:
# 1. Read CSV from ./input (bind mount, read-only)
# 2. Use /cache for intermediate results (volume)
# 3. Use /tmp for temporary calculations (tmpfs, fast)
# 4. Write results to ./output (bind mount)

# Benefits:
# - Input protected (read-only)
# - Fast processing (tmpfs for temp data)
# - Cache reused between runs (volume)
# - Results on host (bind mount)
```

---

## Volume Drivers

Docker supports different volume drivers for various storage backends.

### Built-in Driver

```bash
# Local driver (default)
docker volume create \
    --driver local \
    my-volume
```

### NFS Volume

```bash
# NFS mount as volume
docker volume create \
    --driver local \
    --opt type=nfs \
    --opt o=addr=192.168.1.1,rw \
    --opt device=:/path/to/share \
    nfs-volume

# Use it
docker run -v nfs-volume:/data nginx
```

### Cloud Storage

```bash
# AWS EFS (requires plugin)
docker volume create \
    --driver rexray/efs \
    --opt volumeType=gp2 \
    aws-volume

# Azure File Share (requires plugin)
docker volume create \
    --driver azure/file \
    --opt share=myshare \
    azure-volume
```

---

## Data Persistence Patterns

### Pattern 1: Database with Backups

```bash
# Production database
docker run -d \
    --name db \
    -v db-data:/var/lib/postgresql/data \
    postgres:15

# Regular backups
docker run --rm \
    -v db-data:/data:ro \
    -v $(pwd)/backups:/backup \
    postgres:15 \
    pg_dump -U postgres > /backup/dump.sql

# Restore
docker run --rm \
    -v db-data:/data \
    -v $(pwd)/backups:/backup \
    postgres:15 \
    psql -U postgres < /backup/dump.sql
```

### Pattern 2: Configuration Management

```bash
# Base configuration (bind mount)
docker run -d \
    --name app \
    -v ./config.yml:/app/config.yml:ro \
    -v app-data:/app/data \
    myapp

# Override for different environments
docker run -d \
    --name app-dev \
    -v ./config.dev.yml:/app/config.yml:ro \
    -v app-dev-data:/app/data \
    myapp
```

### Pattern 3: Multi-Stage Processing

```bash
# Stage 1: Extract
docker run --rm \
    -v raw-data:/data \
    extractor

# Stage 2: Transform
docker run --rm \
    -v raw-data:/input:ro \
    -v processed-data:/output \
    transformer

# Stage 3: Load
docker run --rm \
    -v processed-data:/data:ro \
    loader
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Docker __________ are managed by Docker and stored in /var/lib/docker/volumes on Linux.
2. __________ mounts map a host directory directly to a container directory.
3. __________ mounts store data in RAM and are not persisted.
4. When a container is removed, data in the writable layer is __________.
5. Volumes can be mounted as __________ to prevent containers from modifying data.
6. __________ volumes get random names generated by Docker.

### True/False

1. ⬜ Volumes are the recommended way to persist data in Docker
2. ⬜ Data in volumes is lost when the container is removed
3. ⬜ Bind mounts can only be used for read-only access
4. ⬜ tmpfs mounts are faster than volumes because they use RAM
5. ⬜ Multiple containers can share the same volume
6. ⬜ Bind mount paths must be absolute paths
7. ⬜ Volumes have better performance than bind mounts on Mac/Windows

### Multiple Choice

1. Where are Docker volumes stored by default on Linux?
   - A) /var/lib/docker/containers
   - B) /var/lib/docker/volumes
   - C) /etc/docker/volumes
   - D) /opt/docker/volumes

2. Which persistence option is best for production databases?
   - A) Writable layer
   - B) Bind mounts
   - C) Volumes
   - D) tmpfs

3. What happens to tmpfs data when container stops?
   - A) Saved to disk
   - B) Lost permanently
   - C) Moved to volume
   - D) Backed up automatically

4. Which mount type gives Docker the most control?
   - A) Bind mount
   - B) Volume
   - C) tmpfs
   - D) All equal

5. When should you use bind mounts in production?
   - A) Always
   - B) Never
   - C) Only for specific cases like config files
   - D) Only for databases

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. volumes
2. Bind
3. tmpfs
4. lost (or deleted, removed)
5. read-only (or ro)
6. Anonymous

**True/False:**
1. ✅ True - Volumes are the recommended persistence mechanism
2. ❌ False - Volumes persist independently of containers
3. ❌ False - Bind mounts can be read-write or read-only
4. ✅ True - tmpfs uses RAM, much faster than disk
5. ✅ True - Volumes can be shared across multiple containers
6. ✅ True - Bind mounts require absolute paths
7. ✅ True - Volumes have optimizations on Mac/Windows

**Multiple Choice:**
1. **B** - /var/lib/docker/volumes
2. **C** - Volumes (managed by Docker, easy backup)
3. **B** - Lost permanently (tmpfs is in RAM)
4. **B** - Volume (fully managed by Docker)
5. **C** - Only for specific cases like config files

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the difference between volumes, bind mounts, and tmpfs mounts

<details>
<summary><strong>View Answer</strong></summary>

**Three Data Persistence Mechanisms:**

---

**1. Volumes (Recommended)**

**What it is:**
```
Managed storage area controlled by Docker
Location: /var/lib/docker/volumes/
Lifecycle: Independent of containers
```

**How it works:**
```bash
# Create volume
docker volume create my-data

# Use volume
docker run -v my-data:/app/data nginx

# File path:
Host: /var/lib/docker/volumes/my-data/_data/file.txt
Container: /app/data/file.txt
```

**Characteristics:**
```
✓ Fully managed by Docker
✓ Easy backup/restore
✓ Can use volume drivers (NFS, cloud storage)
✓ Better performance on Mac/Windows
✓ Can be shared between containers
✓ Named or anonymous
✓ Persists after container removal
```

**Use cases:**
```
✓ Production databases
✓ Application data
✓ User uploads
✓ Shared data between containers
✓ Anything requiring persistence
```

---

**2. Bind Mounts**

**What it is:**
```
Direct mount of host directory into container
Location: Any directory on host
Lifecycle: Managed by user
```

**How it works:**
```bash
# Mount host directory
docker run -v /host/path:/container/path nginx

# File path:
Host: /host/path/file.txt
Container: /container/path/file.txt
(Same file, direct access)
```

**Characteristics:**
```
⚠ User manages host directory
⚠ Full access to host filesystem
⚠ Path-dependent (may differ across systems)
✓ Real-time sync (changes immediately visible)
✓ Direct file access from host
✓ Can mount individual files
✗ Security risk (host access)
✗ Performance issues on Mac/Windows
```

**Use cases:**
```
✓ Development (source code)
✓ Configuration files
✓ Logs (for easy monitoring)
✓ CI/CD (accessing build workspace)
✓ Sharing data with host tools
```

---

**3. tmpfs Mounts**

**What it is:**
```
Temporary filesystem in RAM
Location: Host memory (RAM)
Lifecycle: Exists while container runs
```

**How it works:**
```bash
# Create tmpfs mount
docker run --tmpfs /tmp:size=64m nginx

# Storage:
RAM: 64MB allocated
Disk: Nothing written
```

**Characteristics:**
```
✓ Very fast (RAM speed)
✓ Never written to disk
✓ Automatically cleaned up
✗ Not persisted (lost on stop)
✗ Limited by available RAM
✗ Linux/Mac only (not Windows)
```

**Use cases:**
```
✓ Secrets/sensitive data (temporary)
✓ Session data
✓ Build artifacts
✓ Cache/temp files
✓ Performance-critical processing
```

---

**Real-World Comparison:**

**Scenario: E-commerce Application**

```yaml
services:
  web:
    image: nginx
    volumes:
      # 1. Static files: Volume (shared with app)
      - static:/usr/share/nginx/html
      
      # 2. Config: Bind mount (easy to update)
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      
      # 3. Temp files: tmpfs (fast, not needed)
      - type: tmpfs
        target: /tmp

  app:
    image: myapp
    volumes:
      # 1. User uploads: Volume (persistent)
      - uploads:/app/uploads
      
      # 2. Source code: Bind mount (development)
      - ./src:/app/src  # Development only!
      
      # 3. Cache: tmpfs (fast, temporary)
      - type: tmpfs
        target: /app/cache
        tmpfs:
          size: 512m

  db:
    image: postgres
    volumes:
      # 1. Database: Volume (persistent, critical)
      - db-data:/var/lib/postgresql/data
      
      # 2. Init scripts: Bind mount (configuration)
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      
      # 3. Temp tables: tmpfs (performance)
      - type: tmpfs
        target: /dev/shm
        tmpfs:
          size: 2g

volumes:
  static:
  uploads:
  db-data:
```

**Why each choice:**

```
Volumes (static, uploads, db-data):
- Need persistence ✓
- Need backup ✓
- Production-ready ✓
- Easy management ✓

Bind Mounts (nginx.conf, init.sql, src):
- Need to edit on host ✓
- Development workflow ✓
- Configuration management ✓
- Not for production data ✗

tmpfs (/tmp, /app/cache, /dev/shm):
- Don't need persistence ✓
- Speed critical ✓
- Temporary data ✓
- Sensitive data (gone after stop) ✓
```

---

**Performance Comparison:**

```
Operation: Write 1GB file

Volume (Linux):
└── Time: 2.5 seconds ✓

Bind Mount (Linux):
└── Time: 2.5 seconds ✓

Bind Mount (Mac):
└── Time: 45 seconds ✗
└── Reason: File sync overhead

tmpfs (Linux/Mac):
└── Time: 0.8 seconds ✓✓
└── Reason: RAM speed
```

---

**Decision Tree:**

```
Need data persistence?
├─ Yes
│  ├─ Production use?
│  │  ├─ Yes → Volume ✓
│  │  └─ No → Bind mount (development)
│  │
│  └─ Need direct host access?
│     ├─ Yes → Bind mount
│     └─ No → Volume ✓
│
└─ No (temporary)
   ├─ Speed critical?
   │  ├─ Yes → tmpfs ✓
   │  └─ No → Volume or tmpfs
   │
   └─ Sensitive data?
      └─ Yes → tmpfs ✓
         (not written to disk)
```

</details>

### Question 2: How would you backup and restore Docker volumes?

<details>
<summary><strong>View Answer</strong></summary>

**Complete Backup/Restore Strategy:**

---

### Method 1: Using Temporary Container (Recommended)

**Backup:**
```bash
# Backup volume to tar file
docker run --rm \
    -v my-volume:/data \
    -v $(pwd):/backup \
    alpine \
    tar czf /backup/my-volume-backup.tar.gz -C /data .

# Explanation:
# --rm              : Remove container after backup
# -v my-volume:/data: Mount volume to backup
# -v $(pwd):/backup : Mount current dir for output
# tar czf           : Create compressed archive
# -C /data .        : Change to /data, archive everything
```

**Restore:**
```bash
# Restore from tar file
docker run --rm \
    -v my-volume:/data \
    -v $(pwd):/backup \
    alpine \
    tar xzf /backup/my-volume-backup.tar.gz -C /data

# Or create new volume and restore
docker volume create my-volume-restored
docker run --rm \
    -v my-volume-restored:/data \
    -v $(pwd):/backup \
    alpine \
    tar xzf /backup/my-volume-backup.tar.gz -C /data
```

---

### Method 2: Using docker cp

**Backup:**
```bash
# Create temporary container
docker create --name temp -v my-volume:/data alpine

# Copy data from volume
docker cp temp:/data ./backup-data

# Clean up
docker rm temp

# Archive
tar czf backup-data.tar.gz backup-data
```

**Restore:**
```bash
# Extract archive
tar xzf backup-data.tar.gz

# Create temporary container
docker create --name temp -v my-volume:/data alpine

# Copy data to volume
docker cp ./backup-data/. temp:/data

# Clean up
docker rm temp
```

---

### Method 3: Database-Specific Backups

**PostgreSQL:**
```bash
# Backup
docker exec postgres \
    pg_dump -U postgres mydb \
    > backup.sql

# Or dump to volume
docker exec postgres \
    pg_dump -U postgres mydb \
    > /var/lib/postgresql/backup/dump.sql

# Restore
docker exec -i postgres \
    psql -U postgres mydb \
    < backup.sql
```

**MySQL:**
```bash
# Backup
docker exec mysql \
    mysqldump -u root -p mydb \
    > backup.sql

# Restore
docker exec -i mysql \
    mysql -u root -p mydb \
    < backup.sql
```

**MongoDB:**
```bash
# Backup
docker exec mongo \
    mongodump --db mydb --out /backup

# Restore
docker exec mongo \
    mongorestore --db mydb /backup/mydb
```

---

### Method 4: Automated Backup Script

```bash
#!/bin/bash
# backup-volumes.sh

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d-%H%M%S)
RETENTION_DAYS=7

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Get all volumes
VOLUMES=$(docker volume ls -q)

for volume in $VOLUMES; do
    echo "Backing up volume: $volume"
    
    # Create backup
    docker run --rm \
        -v $volume:/data:ro \
        -v $BACKUP_DIR:/backup \
        alpine \
        tar czf /backup/${volume}-${DATE}.tar.gz -C /data .
    
    if [ $? -eq 0 ]; then
        echo "✓ Backup successful: ${volume}-${DATE}.tar.gz"
    else
        echo "✗ Backup failed: $volume"
    fi
done

# Remove old backups
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup complete. Location: $BACKUP_DIR"
```

---

### Method 5: Production Backup Pattern

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data
      - db-backups:/backups
    environment:
      - POSTGRES_PASSWORD=secret

  backup:
    image: postgres:15
    volumes:
      - db-data:/data:ro  # Read-only
      - db-backups:/backups
      - ./backup-script.sh:/backup.sh:ro
    command: /backup.sh
    depends_on:
      - db

  # Backup to S3
  backup-s3:
    image: amazon/aws-cli
    volumes:
      - db-backups:/backups:ro
    environment:
      - AWS_ACCESS_KEY_ID
      - AWS_SECRET_ACCESS_KEY
    command: >
      s3 sync /backups 
      s3://my-bucket/backups/
      --delete

volumes:
  db-data:
  db-backups:
```

**backup-script.sh:**
```bash
#!/bin/bash
set -e

while true; do
    DATE=$(date +%Y%m%d-%H%M%S)
    
    # Backup database
    pg_dump -U postgres -h db mydb \
        > /backups/backup-${DATE}.sql
    
    # Compress
    gzip /backups/backup-${DATE}.sql
    
    # Remove backups older than 7 days
    find /backups -name "backup-*.sql.gz" \
        -mtime +7 -delete
    
    echo "Backup complete: backup-${DATE}.sql.gz"
    
    # Wait 24 hours
    sleep 86400
done
```

---

### Method 6: Volume Migration

**Between hosts:**
```bash
# On source host
docker run --rm \
    -v my-volume:/data \
    alpine \
    tar cz -C /data . \
    | ssh user@target-host \
    "docker run --rm -i -v my-volume:/data alpine tar xz -C /data"

# Or using file transfer
# Source host
docker run --rm \
    -v my-volume:/data \
    alpine \
    tar cz -C /data . \
    > my-volume.tar.gz

# Transfer file
scp my-volume.tar.gz user@target-host:/tmp/

# Target host
docker volume create my-volume
docker run --rm \
    -v my-volume:/data \
    -v /tmp:/backup \
    alpine \
    tar xzf /backup/my-volume.tar.gz -C /data
```

---

### Method 7: Cloud Backup

**AWS S3:**
```bash
# Install AWS CLI in container
docker run --rm \
    -v my-volume:/data:ro \
    -v ~/.aws:/root/.aws:ro \
    amazon/aws-cli \
    s3 sync /data s3://my-bucket/backups/my-volume/

# Restore
docker run --rm \
    -v my-volume:/data \
    -v ~/.aws:/root/.aws:ro \
    amazon/aws-cli \
    s3 sync s3://my-bucket/backups/my-volume/ /data
```

**Google Cloud Storage:**
```bash
# Backup
docker run --rm \
    -v my-volume:/data:ro \
    -v ~/.config/gcloud:/root/.config/gcloud:ro \
    google/cloud-sdk \
    gsutil -m rsync -r /data gs://my-bucket/backups/my-volume/

# Restore
docker run --rm \
    -v my-volume:/data \
    -v ~/.config/gcloud:/root/.config/gcloud:ro \
    google/cloud-sdk \
    gsutil -m rsync -r gs://my-bucket/backups/my-volume/ /data
```

---

### Best Practices

**1. Regular Schedule:**
```bash
# Cron job for daily backups
0 2 * * * /path/to/backup-volumes.sh
```

**2. Verify Backups:**
```bash
# Test restore to temporary volume
docker volume create test-restore
docker run --rm \
    -v test-restore:/data \
    -v $(pwd):/backup \
    alpine \
    tar xzf /backup/my-volume-backup.tar.gz -C /data

# Verify data
docker run --rm \
    -v test-restore:/data \
    alpine \
    ls -la /data

# Clean up
docker volume rm test-restore
```

**3. Backup Metadata:**
```bash
# Save volume configuration
docker volume inspect my-volume > my-volume-metadata.json

# Save container configuration
docker inspect my-container > my-container-metadata.json
```

**4. Incremental Backups:**
```bash
# Using rsync for incremental backups
docker run --rm \
    -v my-volume:/data:ro \
    -v backup-volume:/backup \
    alpine \
    rsync -av --delete /data/ /backup/
```

**5. Encryption:**
```bash
# Encrypted backup
docker run --rm \
    -v my-volume:/data:ro \
    -v $(pwd):/backup \
    alpine \
    tar cz -C /data . \
    | gpg --encrypt --recipient user@example.com \
    > /backup/encrypted-backup.tar.gz.gpg

# Restore
gpg --decrypt /backup/encrypted-backup.tar.gz.gpg \
    | docker run --rm -i \
    -v my-volume:/data \
    alpine \
    tar xz -C /data
```

</details>

### Question 3: What are the performance implications of different mount types, especially on Mac/Windows?

<details>
<summary><strong>View Answer</strong></summary>

**Performance Comparison:**

---

### Linux (Native Docker)

**All mount types perform similarly on Linux:**

```
Volumes:
└── Performance: Excellent ✓✓✓
    └── Direct access to host filesystem
    └── No translation layer

Bind Mounts:
└── Performance: Excellent ✓✓✓
    └── Direct filesystem access
    └── Same as volumes

tmpfs:
└── Performance: Excellent ✓✓✓✓
    └── RAM speed
    └── Fastest option
```

**Benchmark (Linux):**
```bash
# Write 1GB file test

# Volume
time docker run --rm \
    -v test-volume:/data \
    alpine dd if=/dev/zero of=/data/test bs=1M count=1024
# Result: 2.5 seconds

# Bind mount
time docker run --rm \
    -v $(pwd)/test:/data \
    alpine dd if=/dev/zero of=/data/test bs=1M count=1024
# Result: 2.5 seconds

# tmpfs
time docker run --rm \
    --tmpfs /data:size=2g \
    alpine dd if=/dev/zero of=/data/test bs=1M count=1024
# Result: 0.8 seconds
```

---

### macOS (Docker Desktop)

**Significant performance differences:**

```
Volumes (with caching):
└── Performance: Good ✓✓
    └── Docker Desktop optimizations
    └── Virtual filesystem layer
    └── ~60% of native speed

Bind Mounts:
└── Performance: Poor ✗
    └── File sync overhead (osxfs/gRPC)
    └── Two-way sync required
    └── ~5-10% of native speed

tmpfs:
└── Performance: Excellent ✓✓✓
    └── Inside VM (no sync)
    └── RAM speed
```

**Benchmark (macOS):**
```bash
# Write 1GB file test

# Volume
time docker run --rm \
    -v test-volume:/data \
    alpine dd if=/dev/zero of=/data/test bs=1M count=1024
# Result: 4 seconds

# Bind mount (default)
time docker run --rm \
    -v $(pwd)/test:/data \
    alpine dd if=/dev/zero of=/data/test bs=1M count=1024
# Result: 45 seconds  ← Very slow!

# Bind mount (cached)
time docker run --rm \
    -v $(pwd)/test:/data:cached \
    alpine dd if=/dev/zero of=/data/test bs=1M count=1024
# Result: 35 seconds  ← Still slow

# tmpfs
time docker run --rm \
    --tmpfs /data:size=2g \
    alpine dd if=/dev/zero of=/data/test bs=1M count=1024
# Result: 0.8 seconds
```

---

### Windows (Docker Desktop)

**Similar to macOS:**

```
Volumes:
└── Performance: Good ✓✓
    └── Optimized file sharing
    └── ~50-70% of native speed

Bind Mounts:
└── Performance: Poor ✗
    └── Windows file sharing overhead
    └── SMB protocol overhead
    └── ~5-15% of native speed

tmpfs:
└── Not supported on Windows ✗
```

---

### Real-World Impact

**Node.js Application on macOS:**

**Scenario 1: Bind Mount (No Optimization)**
```yaml
# docker-compose.yml
services:
  web:
    image: node:18
    volumes:
      - ./:/app
    command: npm start
```

**Performance:**
```
npm install:
├── Linux: 20 seconds
├── macOS (bind mount): 180 seconds  ← 9x slower!
└── macOS (volume): 30 seconds

npm run dev (with file watching):
├── Linux: Fast hot reload (<1s)
├── macOS (bind mount): Slow (3-5s per change)
└── macOS (volume): Not real-time (can't see changes)
```

---

**Scenario 2: Optimized Setup**
```yaml
# docker-compose.yml
services:
  web:
    image: node:18
    volumes:
      # Source code: bind mount with delegated
      - ./src:/app/src:delegated
      
      # Dependencies: named volume (not bind mount!)
      - node_modules:/app/node_modules
      
      # Build output: volume
      - dist:/app/dist
      
    command: npm run dev

volumes:
  node_modules:  # Dramatically faster
  dist:
```

**Performance:**
```
npm install:
├── macOS (bind mount): 180 seconds
└── macOS (optimized): 35 seconds  ← 5x faster!

File changes:
├── macOS (bind mount): 3-5 seconds delay
└── macOS (optimized delegated): 1-2 seconds delay

Build output:
├── macOS (bind mount): 120 seconds
└── macOS (volume): 15 seconds  ← 8x faster!
```

---

### Optimization Strategies

**1. Bind Mount Caching (macOS/Windows)**

```yaml
# Three caching modes:

# consistent (default) - perfect consistency
- ./data:/data

# cached - host → container priority
- ./data:/data:cached
# Use: Source code (container reads, host writes)

# delegated - container → host priority  
- ./data:/data:delegated
# Use: Build output (container writes, host reads)
```

**Real example:**
```yaml
services:
  web:
    volumes:
      # Source: Host writes, container reads
      - ./src:/app/src:cached
      
      # Build: Container writes, host reads
      - ./dist:/app/dist:delegated
      
      # Both: Use default (consistent)
      - ./config.yml:/app/config.yml
```

---

**2. Use Volumes for node_modules**

```dockerfile
# Dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install  # Install in image
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    volumes:
      - ./src:/app/src:cached
      # node_modules from image, not bind mount
      # No bind mount for node_modules = fast!
```

**Alternative: Named volume**
```yaml
volumes:
  - ./src:/app/src:cached
  - node_modules:/app/node_modules  # Override with volume

volumes:
  node_modules:
```

---

**3. Development Container Pattern**

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    volumes:
      # Source code only
      - ./src:/app/src:cached
      - ./package.json:/app/package.json:cached
      
      # Everything else: volumes
      - node_modules:/app/node_modules
      - build_cache:/app/.next
      - logs:/app/logs

volumes:
  node_modules:
  build_cache:
  logs:
```

**Benefits:**
```
Source editing: Real-time (bind mount)
npm install: Fast (volume)
Build: Fast (volume)
Logs: Fast write (volume)
```

---

### Performance Benchmarks

**File Operations (macOS Docker Desktop):**

```
Operation: Create 10,000 small files

Bind Mount (default):
└── Time: 180 seconds
    └── Each file: sync to host

Bind Mount (delegated):
└── Time: 45 seconds
    └── Batched sync

Volume:
└── Time: 3 seconds
    └── No host sync

tmpfs:
└── Time: 1.2 seconds
    └── RAM speed
```

---

**Compilation Speed (macOS):**

```
TypeScript compilation (3000 files):

Bind Mount:
├── Read: Slow
├── Write: Slow
└── Total: 240 seconds

Volume (source + output):
├── Read: Can't edit on host
└── Not practical for development

Hybrid (source bind, output volume):
├── Read source: Acceptable (cached)
├── Write output: Fast (volume)
└── Total: 35 seconds  ← Best option
```

---

### Recommendations

**Linux:**
```
Use whatever makes sense:
✓ Volumes: Production data
✓ Bind mounts: Development code
✓ tmpfs: Temporary fast data

All perform well
```

**macOS/Windows Development:**
```
Optimize carefully:
✓ Source code: Bind mount with :cached
✓ Dependencies (node_modules): Volume
✓ Build output: Volume or :delegated
✓ Temporary data: tmpfs (macOS) or volume (Windows)
✗ Avoid: Large bind mounts with many files
```

**macOS/Windows Production:**
```
Always use volumes:
✓ Better performance
✓ Docker-managed
✓ Consistent across platforms
✗ Never bind mounts in production
```

---

**Quick Decision Matrix:**

```
Platform: Linux
├── All options perform well
└── Choose based on use case

Platform: macOS/Windows
├── Production: Always volumes
├── Development:
│   ├── Source code: Bind mount (:cached)
│   ├── Dependencies: Volumes
│   ├── Build output: Volumes or :delegated
│   └── Temp/cache: Volumes
└── Avoid: Large bind mounts
```

</details>

</details>

---

[← Previous: 4.2 Container Operations](../04-docker-containers/02-container-operations.md) | [Next: 5.2 Volume Operations →](02-volume-operations.md)