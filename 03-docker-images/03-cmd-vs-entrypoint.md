# 3.3 CMD vs ENTRYPOINT

Understanding the difference between CMD and ENTRYPOINT, and when to use each.

---

## Overview

Both **CMD** and **ENTRYPOINT** specify what command runs when a container starts, but they work differently.

```
Analogy:
ENTRYPOINT = The program
CMD = Default arguments to the program

Example:
ENTRYPOINT ["python"]
CMD ["app.py"]

Result: python app.py
```

---

## CMD Instruction

**Purpose:** Provide default command and/or arguments for the container.

### Syntax Forms

```dockerfile
# 1. Exec form (preferred)
CMD ["executable", "param1", "param2"]

# 2. Shell form
CMD command param1 param2

# 3. As parameters to ENTRYPOINT
CMD ["param1", "param2"]
```

### How CMD Works

**Basic Usage:**
```dockerfile
FROM ubuntu:20.04
CMD ["echo", "Hello World"]
```

```bash
# Run with default CMD
docker run myapp
# Output: Hello World

# Override CMD
docker run myapp echo "Goodbye"
# Output: Goodbye
# CMD completely replaced
```

**Easily Overridden:**
```dockerfile
FROM nginx:alpine
CMD ["nginx", "-g", "daemon off;"]
```

```bash
# Default: runs nginx
docker run nginx-image
# Executes: nginx -g daemon off;

# Override: run bash instead
docker run nginx-image bash
# Executes: bash
# nginx command completely ignored
```

---

## ENTRYPOINT Instruction

**Purpose:** Configure a container to run as an executable.

### Syntax Forms

```dockerfile
# 1. Exec form (preferred)
ENTRYPOINT ["executable", "param1", "param2"]

# 2. Shell form
ENTRYPOINT command param1 param2
```

### How ENTRYPOINT Works

**Basic Usage:**
```dockerfile
FROM ubuntu:20.04
ENTRYPOINT ["echo"]
```

```bash
# Run with arguments
docker run myapp "Hello World"
# Output: Hello World
# Executes: echo "Hello World"

# Without arguments
docker run myapp
# Output: (blank line)
# Executes: echo
```

**Harder to Override:**
```dockerfile
FROM alpine:latest
ENTRYPOINT ["ping"]
```

```bash
# Add arguments (easy)
docker run myapp google.com
# Executes: ping google.com

# Override ENTRYPOINT (requires flag)
docker run --entrypoint sh myapp
# Executes: sh
# Must use --entrypoint flag
```

---

## CMD vs ENTRYPOINT Comparison

### Side-by-Side Examples

**Example 1: Simple Command**

```dockerfile
# Using CMD
FROM ubuntu:20.04
CMD ["ls", "-la"]
```

```bash
docker run myapp
# Executes: ls -la

docker run myapp pwd
# Executes: pwd
# ls -la completely replaced
```

```dockerfile
# Using ENTRYPOINT
FROM ubuntu:20.04
ENTRYPOINT ["ls"]
```

```bash
docker run myapp
# Executes: ls

docker run myapp -la /app
# Executes: ls -la /app
# Can add arguments, not replace
```

---

**Example 2: Web Server**

```dockerfile
# CMD approach
FROM nginx:alpine
CMD ["nginx", "-g", "daemon off;"]
```

```bash
# Default: runs nginx
docker run webapp
# nginx -g daemon off;

# Easy to run different command
docker run webapp sh
# sh
# nginx not running!
```

```dockerfile
# ENTRYPOINT approach
FROM nginx:alpine
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

```bash
# Default: runs nginx with default args
docker run webapp
# nginx -g daemon off;

# Can change nginx arguments
docker run webapp -g "daemon off;" -c /custom.conf
# nginx -g "daemon off;" -c /custom.conf
# Still running nginx, different config

# Need flag to run different command
docker run --entrypoint sh webapp
# sh
```

---

## Combining CMD and ENTRYPOINT

**This is the most powerful pattern!**

```dockerfile
FROM alpine:latest

ENTRYPOINT ["ping"]
CMD ["localhost"]
```

**How It Works:**
```
Final command = ENTRYPOINT + CMD

ENTRYPOINT ["ping"]
CMD ["localhost"]
Result: ping localhost
```

**Usage:**
```bash
# Default: ping localhost
docker run myapp
# ping localhost

# Override CMD (just provide new target)
docker run myapp google.com
# ping google.com

# Override ENTRYPOINT (need --entrypoint)
docker run --entrypoint traceroute myapp google.com
# traceroute google.com
```

---

## Exec Form vs Shell Form

### Exec Form (Recommended)

```dockerfile
# Exec form: ["executable", "param1", "param2"]
CMD ["nginx", "-g", "daemon off;"]
ENTRYPOINT ["python", "app.py"]
```

**Characteristics:**
```
✓ Runs directly (no shell)
✓ Proper signal handling
✓ Cleaner process tree
✓ Faster startup
✓ No shell variable expansion
```

**Process Tree:**
```bash
# Exec form: PID 1 is nginx
docker run myapp
# PID 1: nginx

# Proper signal handling
docker stop myapp
# Sends SIGTERM to nginx (PID 1)
# nginx can shutdown gracefully
```

---

### Shell Form (Not Recommended)

```dockerfile
# Shell form: command param1 param2
CMD nginx -g daemon off;
ENTRYPOINT python app.py
```

**Characteristics:**
```
✗ Runs in shell (/bin/sh -c)
✗ Poor signal handling
✗ Extra process overhead
✓ Shell variable expansion works
```

**Process Tree:**
```bash
# Shell form: PID 1 is /bin/sh
docker run myapp
# PID 1: /bin/sh -c nginx -g daemon off;
# PID 2: nginx

# Poor signal handling
docker stop myapp
# Sends SIGTERM to /bin/sh (PID 1)
# /bin/sh doesn't forward to nginx
# nginx doesn't receive signal
# Container killed after timeout
```

---

## Real-World Examples

### Example 1: Python Application

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

# ENTRYPOINT: The interpreter
ENTRYPOINT ["python"]

# CMD: Default script
CMD ["app.py"]
```

**Usage:**
```bash
# Run default app
docker run myapp
# python app.py

# Run different script
docker run myapp manage.py migrate
# python manage.py migrate

# Run with flags
docker run myapp -m pytest
# python -m pytest

# Interactive Python
docker run -it myapp
# python (interactive shell)
```

---

### Example 2: CLI Tool (curl wrapper)

```dockerfile
FROM alpine:latest

RUN apk add --no-cache curl

# ENTRYPOINT: The tool
ENTRYPOINT ["curl"]

# CMD: Default behavior (help)
CMD ["--help"]
```

**Usage:**
```bash
# Show help
docker run curltool
# curl --help

# Make request
docker run curltool -I https://google.com
# curl -I https://google.com

# Download file
docker run curltool -o output.txt https://example.com/file.txt
# curl -o output.txt https://example.com/file.txt

# With verbose
docker run curltool -v https://api.example.com
# curl -v https://api.example.com
```

---

### Example 3: Database with Entrypoint Script

```dockerfile
FROM postgres:15

# Copy entrypoint script
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

# ENTRYPOINT: Wrapper script
ENTRYPOINT ["docker-entrypoint.sh"]

# CMD: Default command
CMD ["postgres"]
```

**docker-entrypoint.sh:**
```bash
#!/bin/bash
set -e

# Initialization logic
if [ ! -d "$PGDATA" ]; then
    echo "Initializing database..."
    initdb
fi

# Run migrations
if [ -f "/migrations/schema.sql" ]; then
    psql -f /migrations/schema.sql
fi

# Execute CMD (postgres)
exec "$@"
```

**Usage:**
```bash
# Default: runs postgres after initialization
docker run db-image
# docker-entrypoint.sh postgres

# With custom config
docker run db-image postgres -c config_file=/custom.conf
# docker-entrypoint.sh postgres -c config_file=/custom.conf

# Run psql instead
docker run db-image psql
# docker-entrypoint.sh psql
```

---

### Example 4: Node.js with Development/Production

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

# ENTRYPOINT: Node
ENTRYPOINT ["node"]

# CMD: Production entry point
CMD ["dist/server.js"]
```

**Usage:**
```bash
# Production
docker run -e NODE_ENV=production myapp
# node dist/server.js

# Development
docker run -e NODE_ENV=development myapp src/server.js
# node src/server.js

# Run tests
docker run myapp --test
# node --test

# Debug mode
docker run myapp --inspect=0.0.0.0:9229 dist/server.js
# node --inspect=0.0.0.0:9229 dist/server.js
```

---

### Example 5: Multi-purpose Container

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

# Entrypoint script handles different commands
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["serve"]
```

**entrypoint.sh:**
```bash
#!/bin/bash

case "$1" in
    serve)
        exec python app.py
        ;;
    worker)
        exec celery -A tasks worker
        ;;
    migrate)
        exec python manage.py migrate
        ;;
    shell)
        exec python manage.py shell
        ;;
    *)
        exec "$@"
        ;;
esac
```

**Usage:**
```bash
# Default: serve
docker run myapp
# python app.py

# Run worker
docker run myapp worker
# celery -A tasks worker

# Run migrations
docker run myapp migrate
# python manage.py migrate

# Interactive shell
docker run -it myapp shell
# python manage.py shell

# Custom command
docker run myapp python custom_script.py
# python custom_script.py
```

---

## Common Patterns

### Pattern 1: Application Server

```dockerfile
# Web application
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

ENTRYPOINT ["node"]
CMD ["server.js"]

# Flexible: can run server or utilities
```

### Pattern 2: CLI Tool

```dockerfile
# Command-line tool
FROM alpine:latest
RUN apk add --no-cache jq

ENTRYPOINT ["jq"]
CMD ["--help"]

# Acts like jq command
# docker run jqtool '.' data.json
```

### Pattern 3: Service with Initialization

```dockerfile
# Database or service
FROM postgres:15

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["postgres"]

# Entrypoint handles setup
# CMD is the service to run
```

### Pattern 4: Multi-Mode Application

```dockerfile
# Application with modes
FROM python:3.9

COPY entrypoint.sh /
ENTRYPOINT ["/entrypoint.sh"]
CMD ["web"]

# Modes: web, worker, migrate, shell
```

---

## Decision Guide

### When to Use CMD Only

```
Use CMD when:
✓ Simple containers
✓ Default command easily overridden
✓ No special initialization needed
✓ Running different commands common

Example: Development containers
```

```dockerfile
FROM node:18
WORKDIR /app
CMD ["npm", "start"]

# Easy to override for different tasks
# docker run myapp npm test
# docker run myapp npm run build
```

---

### When to Use ENTRYPOINT Only

```
Use ENTRYPOINT when:
✓ Container as executable
✓ Always runs same base command
✓ Arguments vary, not command
✓ CLI tool wrapper

Example: Utility containers
```

```dockerfile
FROM alpine:latest
RUN apk add --no-cache curl
ENTRYPOINT ["curl"]

# Always runs curl
# docker run curltool -I https://example.com
```

---

### When to Use ENTRYPOINT + CMD

```
Use ENTRYPOINT + CMD when:
✓ Main command fixed
✓ Default arguments provided
✓ Arguments easily overridden
✓ Professional production images

Example: Application servers
```

```dockerfile
FROM python:3.9
ENTRYPOINT ["python"]
CMD ["app.py"]

# python app.py by default
# Easy to pass different args
# docker run myapp -m pytest
```

---

## Best Practices

### 1. Prefer Exec Form

```dockerfile
# Good
CMD ["nginx", "-g", "daemon off;"]
ENTRYPOINT ["python", "app.py"]

# Avoid
CMD nginx -g daemon off;
ENTRYPOINT python app.py
```

### 2. Use ENTRYPOINT + CMD Pattern

```dockerfile
# Best practice for applications
ENTRYPOINT ["node"]
CMD ["server.js"]

# Flexible and predictable
```

### 3. Don't Mix Forms

```dockerfile
# Bad: Shell ENTRYPOINT + Exec CMD
ENTRYPOINT python
CMD ["app.py"]
# Results in: /bin/sh -c python app.py
# Poor signal handling

# Good: Both exec form
ENTRYPOINT ["python"]
CMD ["app.py"]
# Results in: python app.py
# Clean execution
```

### 4. Use Wrapper Scripts for Complex Logic

```dockerfile
# For complex initialization
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["serve"]
```

### 5. Document Expected Usage

```dockerfile
# At top of Dockerfile
# Usage:
#   docker run myapp                    # Run server
#   docker run myapp worker             # Run worker
#   docker run myapp migrate            # Run migrations
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. __________ provides default command that can be easily overridden.
2. __________ configures the container to run as an executable.
3. The exec form uses __________ format, while shell form uses plain text.
4. When both ENTRYPOINT and CMD are present, the final command is __________ + __________.
5. To override ENTRYPOINT, you must use the __________ flag.
6. __________ form is preferred because it provides better signal handling.

### True/False

1. ⬜ CMD can be easily overridden by passing arguments to docker run
2. ⬜ ENTRYPOINT can be easily overridden by passing arguments to docker run
3. ⬜ Shell form runs the command wrapped in /bin/sh -c
4. ⬜ You can have both ENTRYPOINT and CMD in the same Dockerfile
5. ⬜ Only the last CMD instruction in a Dockerfile takes effect
6. ⬜ Exec form allows shell variable expansion
7. ⬜ When using exec form, PID 1 is your application

### Multiple Choice

1. What happens when you use `docker run myapp echo hello` if CMD is `["nginx"]`?
   - A) Runs nginx
   - B) Runs echo hello
   - C) Error
   - D) Runs both

2. How do you override ENTRYPOINT?
   - A) Just pass arguments to docker run
   - B) Use --entrypoint flag
   - C) Cannot be overridden
   - D) Use --cmd flag

3. Which form provides better signal handling?
   - A) Shell form
   - B) Exec form
   - C) Both equal
   - D) Neither

4. If ENTRYPOINT is `["python"]` and CMD is `["app.py"]`, what runs?
   - A) python
   - B) app.py
   - C) python app.py
   - D) Error

5. What is the PID 1 process when using shell form?
   - A) Your application
   - B) /bin/sh
   - C) docker-init
   - D) systemd

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. CMD
2. ENTRYPOINT
3. JSON (or array)
4. ENTRYPOINT, CMD
5. --entrypoint
6. Exec

**True/False:**
1. ✅ True - CMD easily overridden by arguments
2. ❌ False - ENTRYPOINT requires --entrypoint flag to override
3. ✅ True - Shell form runs in /bin/sh -c
4. ✅ True - Common pattern: ENTRYPOINT + CMD
5. ✅ True - Only last CMD instruction used
6. ❌ False - Exec form doesn't expand variables (shell form does)
7. ✅ True - Exec form runs app as PID 1

**Multiple Choice:**
1. **B** - Runs echo hello (CMD overridden)
2. **B** - Use --entrypoint flag
3. **B** - Exec form (app is PID 1)
4. **C** - python app.py (ENTRYPOINT + CMD)
5. **B** - /bin/sh (shell wraps the command)

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Explain the practical difference between CMD and ENTRYPOINT with examples

<details>
<summary><strong>View Answer</strong></summary>

**Core Difference:**

**CMD** = Default command, **easily overridden**  
**ENTRYPOINT** = Main executable, **harder to override**

---

**Scenario 1: Web Server**

**Using CMD:**
```dockerfile
FROM nginx:alpine
CMD ["nginx", "-g", "daemon off;"]
```

```bash
# Default: runs nginx
docker run webapp
→ nginx -g daemon off;

# Problem: Easy to accidentally not run nginx
docker run webapp bash
→ bash
# nginx NOT running!
# Container starts but doesn't serve web traffic
```

**Using ENTRYPOINT + CMD:**
```dockerfile
FROM nginx:alpine
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

```bash
# Default: runs nginx
docker run webapp
→ nginx -g daemon off;

# Override arguments: still runs nginx
docker run webapp -c /custom.conf
→ nginx -c /custom.conf
# nginx ALWAYS runs

# To run something else (need flag)
docker run --entrypoint bash webapp
→ bash
# Clear intention to not run nginx
```

---

**Scenario 2: Python Application**

**Using CMD:**
```dockerfile
FROM python:3.9
COPY app.py .
CMD ["python", "app.py"]
```

```bash
# Works, but limited
docker run myapp
→ python app.py

# Want to run tests?
docker run myapp python -m pytest
# Redundant "python" because CMD has full command

# Want Python REPL?
docker run -it myapp python
# Again redundant
```

**Using ENTRYPOINT + CMD:**
```dockerfile
FROM python:3.9
COPY app.py .
ENTRYPOINT ["python"]
CMD ["app.py"]
```

```bash
# Default app
docker run myapp
→ python app.py

# Run tests (clean syntax)
docker run myapp -m pytest
→ python -m pytest

# Interactive Python
docker run -it myapp
→ python

# Run different script
docker run myapp manage.py migrate
→ python manage.py migrate

# All commands use python!
```

---

**Scenario 3: CLI Tool (jq wrapper)**

**Using CMD:**
```dockerfile
FROM alpine
RUN apk add --no-cache jq
CMD ["jq", "--help"]
```

```bash
# Show help
docker run jqtool
→ jq --help

# Process JSON
docker run jqtool jq '.' data.json
→ jq '.' data.json
# Works but awkward (jq twice)
```

**Using ENTRYPOINT:**
```dockerfile
FROM alpine
RUN apk add --no-cache jq
ENTRYPOINT ["jq"]
CMD ["--help"]
```

```bash
# Show help
docker run jqtool
→ jq --help

# Process JSON (clean)
docker run jqtool '.' data.json
→ jq '.' data.json

# With options
docker run jqtool -c '.' data.json
→ jq -c '.' data.json

# Container behaves exactly like jq command!
```

---

**Real-World Production Example:**

**Database Container (PostgreSQL pattern):**

```dockerfile
FROM postgres:15

COPY init-scripts/ /docker-entrypoint-initdb.d/
COPY entrypoint.sh /usr/local/bin/

ENTRYPOINT ["entrypoint.sh"]
CMD ["postgres"]
```

**entrypoint.sh:**
```bash
#!/bin/bash
set -e

# Initialization logic
if [ ! -s "$PGDATA/PG_VERSION" ]; then
    initdb --username=postgres
    # Run init scripts
    for f in /docker-entrypoint-initdb.d/*; do
        psql -f "$f"
    done
fi

# Start whatever was passed as CMD
exec "$@"
```

**Usage:**
```bash
# Default: runs postgres
docker run db-image
→ entrypoint.sh postgres
# Initializes DB, then runs postgres

# Run with custom config
docker run db-image postgres -c max_connections=200
→ entrypoint.sh postgres -c max_connections=200
# Still initializes, then runs with custom args

# Run psql client
docker run -it db-image psql
→ entrypoint.sh psql
# Initialization runs (idempotent), then psql

# Run bash for debugging
docker run -it db-image bash
→ entrypoint.sh bash
# Initialization runs, then bash
```

**Why This Pattern?**
```
✓ ENTRYPOINT ensures initialization always runs
✓ CMD provides default (postgres)
✓ Can override CMD for different tools
✓ Cannot accidentally skip entrypoint
✓ Flexible for different use cases
```

---

**Summary Table:**

| Use Case | Use CMD | Use ENTRYPOINT | Use Both |
|----------|---------|----------------|----------|
| Simple app | ✓ | | |
| CLI tool | | ✓ | ✓ |
| Web server | | | ✓ |
| Database | | | ✓ |
| Utility | | ✓ | |
| Development | ✓ | | |
| Production | | | ✓ |

</details>

### Question 2: What's wrong with shell form and why is exec form preferred?

<details>
<summary><strong>View Answer</strong></summary>

**Problems with Shell Form:**

**1. Poor Signal Handling (Most Critical)**

**Shell Form:**
```dockerfile
FROM python:3.9
CMD python app.py
```

**What Happens:**
```
Process Tree:
PID 1: /bin/sh -c python app.py
PID 7: python app.py

docker stop myapp
→ Sends SIGTERM to PID 1 (/bin/sh)
→ /bin/sh doesn't forward signal to python
→ python keeps running
→ After 10s timeout: SIGKILL sent
→ python killed abruptly
→ No graceful shutdown
→ Connections dropped
→ Data loss possible
```

**Exec Form:**
```dockerfile
FROM python:3.9
CMD ["python", "app.py"]
```

**What Happens:**
```
Process Tree:
PID 1: python app.py

docker stop myapp
→ Sends SIGTERM to PID 1 (python)
→ python receives signal directly
→ Can shutdown gracefully
→ Close connections properly
→ Save state
→ Clean exit
```

**Real Impact:**
```python
# app.py with signal handling
import signal
import sys

def shutdown(signum, frame):
    print("Shutting down gracefully...")
    # Close database connections
    db.close()
    # Finish current requests
    server.close()
    # Save state
    save_state()
    sys.exit(0)

signal.signal(signal.SIGTERM, shutdown)

# Shell form: shutdown() NEVER called
# Exec form: shutdown() called properly
```

---

**2. Extra Process Overhead**

**Shell Form:**
```dockerfile
CMD nginx -g daemon off;
```

```
Container has TWO processes:
PID 1: /bin/sh -c nginx -g daemon off;
PID 7: nginx -g daemon off;

Overhead:
- Extra memory (~5-10 MB for shell)
- Extra CPU (shell keeps running)
- More complex process tree
```

**Exec Form:**
```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

```
Container has ONE process:
PID 1: nginx -g daemon off;

Benefits:
- Less memory
- Simpler process tree
- Cleaner ps output
```

---

**3. Signal Propagation Issues**

**Problem Scenario:**
```dockerfile
# Shell form
CMD python -m celery worker
```

**When container receives signals:**
```
SIGTERM sent
→ /bin/sh receives it (PID 1)
→ Shell doesn't know how to handle it
→ Doesn't forward to celery
→ Celery keeps working
→ Tasks interrupted mid-execution
→ 10s timeout → SIGKILL
→ Tasks lost
```

**Fixed with Exec Form:**
```dockerfile
CMD ["python", "-m", "celery", "worker"]
```

```
SIGTERM sent
→ python receives it directly (PID 1)
→ Celery signal handlers work
→ Finishes current tasks
→ Graceful shutdown
→ No task loss
```

---

**4. No Variable Expansion in Exec Form**

**This is actually a FEATURE, not a bug:**

**Shell Form:**
```dockerfile
ENV NAME=world
CMD echo Hello $NAME
```

```bash
docker run myapp
→ /bin/sh -c echo Hello $NAME
→ Shell expands $NAME
→ Output: Hello world

But: Need shell running
Vulnerability: Shell injection possible
```

**Exec Form:**
```dockerfile
ENV NAME=world
CMD ["echo", "Hello $NAME"]
```

```bash
docker run myapp
→ echo Hello $NAME
→ NO variable expansion
→ Output: Hello $NAME

If you need variables, use explicit shell:
CMD ["/bin/sh", "-c", "echo Hello $NAME"]
```

---

**5. Difficult to Debug**

**Shell Form Process Tree:**
```bash
docker exec myapp ps aux

PID  COMMAND
1    /bin/sh -c python app.py
7    python app.py

Which is your app? PID 1 or 7?
Which to send signals to?
Which to check logs for?
```

**Exec Form Process Tree:**
```bash
docker exec myapp ps aux

PID  COMMAND
1    python app.py

Clear: PID 1 is your app
Easy to debug
Easy to monitor
```

---

**Real-World Disaster Scenario:**

**Company Production Issue:**

```dockerfile
# Original Dockerfile (shell form)
FROM python:3.9
COPY app.py .
CMD python app.py
```

**What Happened:**
```
Production deployment:
- High traffic e-commerce site
- Black Friday
- Need to deploy update

Deploy process:
1. docker stop old-container
   → SIGTERM sent to /bin/sh (PID 1)
   → Shell doesn't forward to python
   → Python keeps processing requests
   → 10s timeout
   → SIGKILL sent
   → Python killed immediately

Result:
- 500 active customer checkouts interrupted
- Transactions incomplete
- Customer complaints
- Revenue loss
- Database in inconsistent state
```

**Fixed Version (exec form):**
```dockerfile
FROM python:3.9
COPY app.py .
CMD ["python", "app.py"]
```

**With Signal Handling:**
```python
# app.py
import signal

def graceful_shutdown(signum, frame):
    print("Received SIGTERM, finishing current requests...")
    # Stop accepting new requests
    server.stop_accepting()
    # Wait for current requests (max 30s)
    server.wait_for_completion(timeout=30)
    # Close connections
    db.close()
    # Exit
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

**New Deployment:**
```
Deploy process:
1. docker stop old-container
   → SIGTERM sent to python (PID 1)
   → Python receives signal
   → graceful_shutdown() called
   → Finishes 500 checkouts
   → Closes cleanly in 3 seconds

Result:
- Zero interrupted transactions
- Zero customer complaints
- Clean deployment
- Happy customers
```

---

**When Shell Form Is Acceptable:**

```dockerfile
# Only acceptable when you specifically need shell features

# Example: Complex initialization
CMD ./setup.sh && python app.py

# Better: Use explicit shell in exec form
CMD ["/bin/sh", "-c", "./setup.sh && python app.py"]

# Best: Use entrypoint script
COPY entrypoint.sh /
CMD ["/entrypoint.sh"]
```

---

**Migration Guide:**

**From:**
```dockerfile
CMD python -m flask run --host=0.0.0.0
```

**To:**
```dockerfile
CMD ["python", "-m", "flask", "run", "--host=0.0.0.0"]
```

**From:**
```dockerfile
CMD npm start
```

**To:**
```dockerfile
CMD ["npm", "start"]
```

**From (needs shell):**
```dockerfile
CMD source /env.sh && python app.py
```

**To:**
```dockerfile
CMD ["/bin/sh", "-c", "source /env.sh && python app.py"]
# Explicit shell usage
```

---

**Summary:**

```
Shell Form Problems:
✗ Poor signal handling
✗ Extra process overhead
✗ No direct signal to app
✗ Complex process tree
✗ 10s timeout on stop
✗ Abrupt termination

Exec Form Benefits:
✓ Direct signal handling
✓ Minimal overhead
✓ App is PID 1
✓ Clean process tree
✓ Graceful shutdown
✓ Better monitoring

Always use exec form unless you have a specific reason not to.
```

</details>

### Question 3: Design a Dockerfile for a microservice that needs to run migrations before starting

<details>
<summary><strong>View Answer</strong></summary>

**Requirements:**
```
1. Run database migrations before starting
2. Handle migration failures gracefully
3. Support different modes (web, worker, migrate-only)
4. Proper signal handling
5. Non-root user
6. Health checks
```

**Solution:**

```dockerfile
FROM python:3.9-slim AS base

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        postgresql-client \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Create non-root user
RUN useradd -m -u 1000 appuser

# Copy application
COPY --chown=appuser:appuser . .

# Copy entrypoint script
COPY --chown=appuser:appuser entrypoint.sh /
RUN chmod +x /entrypoint.sh

# Switch to non-root
USER appuser

EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD python healthcheck.py || exit 1

# Entrypoint handles migrations
ENTRYPOINT ["/entrypoint.sh"]

# Default: run web server
CMD ["web"]
```

**entrypoint.sh:**
```bash
#!/bin/bash
set -e

# Color output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1"
}

error() {
    echo -e "${RED}[$(date +'%Y-%m-%d %H:%M:%S')] ERROR:${NC} $1"
}

warn() {
    echo -e "${YELLOW}[$(date +'%Y-%m-%d %H:%M:%S')] WARN:${NC} $1"
}

# Wait for database to be ready
wait_for_db() {
    log "Waiting for database..."
    
    max_attempts=30
    attempt=0
    
    while [ $attempt -lt $max_attempts ]; do
        if python -c "
import sys
from app import db
try:
    db.connect()
    sys.exit(0)
except Exception as e:
    sys.exit(1)
        "; then
            log "Database is ready!"
            return 0
        fi
        
        attempt=$((attempt + 1))
        warn "Database not ready, attempt $attempt/$max_attempts"
        sleep 2
    done
    
    error "Database failed to become ready"
    return 1
}

# Run migrations
run_migrations() {
    log "Running database migrations..."
    
    if python manage.py migrate; then
        log "Migrations completed successfully"
        return 0
    else
        error "Migrations failed"
        return 1
    fi
}

# Graceful shutdown handler
shutdown() {
    log "Received shutdown signal, stopping gracefully..."
    
    # Send SIGTERM to child process
    if [ ! -z "$child_pid" ]; then
        kill -TERM "$child_pid" 2>/dev/null
        wait "$child_pid"
    fi
    
    log "Shutdown complete"
    exit 0
}

# Trap signals
trap shutdown SIGTERM SIGINT

# Main logic
case "$1" in
    web)
        log "Starting in WEB mode"
        wait_for_db || exit 1
        run_migrations || exit 1
        
        log "Starting web server..."
        exec python -m gunicorn app:app \
            --bind 0.0.0.0:8000 \
            --workers 4 \
            --timeout 30 \
            --graceful-timeout 30 &
        
        child_pid=$!
        wait "$child_pid"
        ;;
        
    worker)
        log "Starting in WORKER mode"
        wait_for_db || exit 1
        run_migrations || exit 1
        
        log "Starting Celery worker..."
        exec python -m celery worker \
            --app=app.celery \
            --loglevel=info \
            --max-tasks-per-child=1000 &
        
        child_pid=$!
        wait "$child_pid"
        ;;
        
    migrate)
        log "Starting in MIGRATE-ONLY mode"
        wait_for_db || exit 1
        run_migrations || exit 1
        
        log "Migrations complete, exiting"
        exit 0
        ;;
        
    shell)
        log "Starting interactive shell"
        wait_for_db || exit 1
        
        exec python manage.py shell
        ;;
        
    *)
        log "Custom command: $@"
        exec "$@"
        ;;
esac
```

**Usage:**

```bash
# Web server (default)
docker run myapp
# Waits for DB → Runs migrations → Starts web server

# Worker
docker run myapp worker
# Waits for DB → Runs migrations → Starts Celery worker

# Run migrations only
docker run myapp migrate
# Waits for DB → Runs migrations → Exits

# Interactive shell
docker run -it myapp shell
# Waits for DB → Opens Python shell

# Custom command
docker run myapp python custom_script.py
# Executes custom script
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 5s
      timeout: 5s
      retries: 5

  migrate:
    build: .
    command: migrate
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://myapp:secret@db:5432/myapp

  web:
    build: .
    command: web
    ports:
      - "8000:8000"
    depends_on:
      migrate:
        condition: service_completed_successfully
    environment:
      DATABASE_URL: postgresql://myapp:secret@db:5432/myapp

  worker:
    build: .
    command: worker
    depends_on:
      migrate:
        condition: service_completed_successfully
    environment:
      DATABASE_URL: postgresql://myapp:secret@db:5432/myapp
      CELERY_BROKER_URL: redis://redis:6379/0

  redis:
    image: redis:7-alpine

volumes:
  postgres-data:
```

**Startup Sequence:**
```
1. db starts
   └─ Waits for health check (pg_isready)

2. migrate runs
   ├─ Waits for db health
   ├─ Runs migrations
   └─ Exits successfully

3. web starts
   ├─ Waits for migrate completion
   ├─ Starts web server
   └─ Serves traffic

4. worker starts
   ├─ Waits for migrate completion
   ├─ Starts Celery worker
   └─ Processes tasks
```

**Benefits:**

```
✓ Migrations run once before any service starts
✓ Graceful shutdown handling
✓ Health checks
✓ Non-root user
✓ Multiple operational modes
✓ Proper error handling
✓ Colored logging
✓ Database connection waiting
✓ Signal handling
✓ Clean process management
```

**Production Ready:**
```
✓ Handles database not ready
✓ Handles migration failures
✓ Proper shutdown on SIGTERM
✓ No data loss on deployment
✓ Supports rolling updates
✓ Supports blue-green deployment
✓ Idempotent migrations
✓ Comprehensive logging
```

</details>

</details>

---

[← Previous: 3.2 Dockerfile](02-dockerfile.md) | [Next: 4.1 Container Lifecycle →](../04-docker-containers/01-container-lifecycle.md)