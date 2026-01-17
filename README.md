# Docker Complete Guide

A comprehensive, interview-ready guide to Docker covering fundamentals through advanced topics with real-world examples, diagrams, and practice questions.

---

## Table of Contents

### 1. Fundamentals
- [1.1 Infrastructure Evolution](01-fundamentals/01-infrastructure-evolution.md)
  - Bare Metal
  - Virtual Machines
  - Containers
  - VMs vs Containers Comparison
- [1.2 Why Containers?](01-fundamentals/02-why-containers.md)
  - Real-World Scenarios
  - Use Cases
- [1.3 Docker Basics](01-fundamentals/03-docker-basics.md)
  - What is Docker?
  - Docker Image vs Container
  - Docker Architecture

### 2. Docker Engine
- [2.1 Docker Engine Components](02-docker-engine/01-engine-components.md)
  - Docker CLI
  - Docker Daemon
  - Docker REST API
  - Architecture Diagram
- [2.2 Linux Kernel Features](02-docker-engine/02-kernel-features.md)
  - Namespaces
  - Control Groups (cgroups)
  - Resource Limits

### 3. Docker Images
- [3.1 Working with Images](03-docker-images/01-image-basics.md)
  - Building Images
  - Pulling & Pushing
  - Image Management
- [3.2 Dockerfile](03-docker-images/02-dockerfile.md)
  - Dockerfile Instructions
  - Best Practices
  - Multi-stage Builds
- [3.3 CMD vs ENTRYPOINT](03-docker-images/03-cmd-vs-entrypoint.md)
  - Differences
  - Use Cases
  - Best Practices

### 4. Docker Containers
- [4.1 Container Lifecycle](04-docker-containers/01-container-lifecycle.md)
  - Running Containers
  - Stopping & Starting
  - Container States
- [4.2 Container Operations](04-docker-containers/02-container-operations.md)
  - Interactive Mode
  - Executing Commands
  - Logs & Debugging

### 5. Docker Volumes
- [5.1 Data Persistence](05-docker-volumes/01-volumes-basics.md)
  - Why Volumes?
  - Volume Types
  - Volume Management
- [5.2 Volume Operations](05-docker-volumes/02-volume-operations.md)
  - Creating Volumes
  - Mounting Volumes
  - Data Backup

### 6. Docker Networking
- [6.1 Network Fundamentals](06-docker-networking/01-network-basics.md)
  - Network Types
  - Network Drivers
  - Container Communication
- [6.2 Network Operations](06-docker-networking/02-network-operations.md)
  - Creating Networks
  - Connecting Containers
  - Network Isolation

### 7. Docker Registry
- [7.1 Registry Basics](07-docker-registry/01-registry-basics.md)
  - What is a Registry?
  - DockerHub
  - Private Registries
- [7.2 Registry Operations](07-docker-registry/02-registry-operations.md)
  - Pushing Images
  - Pulling Images
  - Registry Management

### 8. Docker Compose
- [8.1 Docker Compose Fundamentals](08-docker-compose/01-compose-basics.md)
  - What is Docker Compose?
  - Why Docker Compose?
  - docker-compose.yml Structure
- [8.2 Compose Services](08-docker-compose/02-compose-services.md)
  - Defining Services
  - Dependencies
  - Environment Variables
- [8.3 Compose Operations](08-docker-compose/03-compose-operations.md)
  - Starting Applications
  - Scaling Services
  - Docker Commands vs Compose

### 9. Container Orchestration
- [9.1 From Compose to Orchestration](09-orchestration/01-orchestration-intro.md)
  - Limitations of Docker Compose
  - Why Kubernetes?
  - Orchestration Overview
- [9.2 Kubernetes Overview](09-orchestration/02-kubernetes-overview.md)
  - What is Kubernetes?
  - Key Concepts
  - When to Use K8s

### 10. Best Practices & Production
- [10.1 Dockerfile Best Practices](10-best-practices/01-dockerfile-practices.md)
  - Layer Optimization
  - Security
  - Image Size
- [10.2 Production Considerations](10-best-practices/02-production.md)
  - Health Checks
  - Logging
  - Monitoring
  - Security Hardening

### 11. Quick Reference
- [11.1 Command Cheatsheet](11-reference/01-command-cheatsheet.md)
- [11.2 Troubleshooting Guide](11-reference/02-troubleshooting.md)
- [11.3 Common Patterns](11-reference/03-common-patterns.md)

---

## How to Use This Guide

### For Learning
1. Start with **Fundamentals** (Section 1-2)
2. Practice with **Images and Containers** (Section 3-4)
3. Learn **Data Persistence** (Section 5)
4. Master **Networking** (Section 6)
5. Explore **Multi-container Apps** (Section 8)

### For Interviews
- Review practice questions at the end of each section
- Focus on **Interview Questions** sections
- Understand real-world scenarios
- Practice explaining concepts with diagrams

### For Reference
- Use the **Quick Reference** section (Section 11)
- Each section is self-contained
- Search by topic number (e.g., 3.2 for Dockerfile)

---

## Learning Approach

Each section follows this structure:
1. **Concept Explanation** with real-world analogies
2. **Diagrams** for visual understanding
3. **Code Examples** from production scenarios
4. **Practice Questions** (fill-in-the-blank, true/false, MCQ)
5. **Interview Questions** with detailed answers

---

## Prerequisites

- Basic Linux command line knowledge
- Understanding of software development concepts
- Familiarity with web applications (helpful but not required)

---

## Additional Resources

- [Official Docker Documentation](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Play with Docker](https://labs.play-with-docker.com/)

---

## Contributing

This guide is designed to be comprehensive and continuously updated. Sections are organized to allow easy addition of new topics while maintaining structure.

---

**Last Updated:** January 2026