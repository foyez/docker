# Docker

## Bare Metal

**Bare metal** refers to a physical computer system with no pre-installed operating system or applications. Only firmware such as BIOS or UEFI may be present.

**Key points:**

* No virtualization layer
* Full access to hardware resources
* High performance, low abstraction

---

## Virtual Machines (VMs)

A **virtual machine (VM)** is a fully virtualized computer running its own operating system on top of a **hypervisor**.

**Characteristics:**

* Each VM includes a full guest OS
* Strong isolation
* Higher resource overhead
* Slower startup compared to containers

**Example:**

* One physical server running Linux
* Multiple VMs running Linux and Windows

---

## Containers

A **container** is a lightweight, isolated environment that packages an application with all its dependencies. Containers share the host OS kernel instead of running a full OS.

**Key technologies used:**

* Linux namespaces (process, network, filesystem isolation)
* cgroups (resource limits)

**Why containers are fast:**

* No guest OS
* Minimal overhead

---

## Virtual Machines vs Containers

![Virtual Machines vs Contaienrs](image.png)

source: [k21academy](https://k21academy.com/wp-content/uploads/2020/11/Docker-and-Vm-blog-image_result-1.webp)

<details>

<summary><strong>Details >></strong></summary>

### High-Level Difference

| Virtual Machines           | Containers                   |
| -------------------------- | ---------------------------- |
| Virtualize **hardware**    | Virtualize **applications**  |
| Each VM has its **own OS** | Share the **host OS kernel** |
| Heavyweight                | Lightweight                  |
| Slower to start            | Start in seconds             |
| Strong isolation           | Process-level isolation      |

---

### Architecture Comparison

#### Virtual Machines

```
Applications
Guest OS
-----------
Hypervisor
-----------
Host OS
-----------
Hardware
```

* Each VM runs a **full operating system**
* Hypervisor manages hardware allocation
* Higher memory and CPU usage

---

#### Containers

```
Applications
-----------
Container Runtime (Docker)
-----------
Host OS Kernel
-----------
Hardware
```

* Containers **do not include an OS**
* Share the host kernel
* Much lower overhead

---

### Resource Usage

| Feature      | Virtual Machine | Container |
| ------------ | --------------- | --------- |
| Disk size    | GBs             | MBs       |
| Memory usage | High            | Low       |
| CPU overhead | High            | Minimal   |
| Boot time    | Minutes         | Seconds   |

---

### Isolation & Security

| Aspect            | Virtual Machine   | Container                |
| ----------------- | ----------------- | ------------------------ |
| Isolation         | Strong (OS-level) | Moderate (process-level) |
| Kernel            | Separate per VM   | Shared                   |
| Security boundary | Hardware-level    | OS-level                 |

> VMs are better for **strong isolation**, containers are better for **speed and density**.

---

### Portability

|                             | Virtual Machine | Container |
| --------------------------- | --------------- | --------- |
| Cross-platform              | Limited         | Excellent |
| Environment consistency     | Medium          | Very high |
| “Works on my machine” issue | Possible        | Rare      |

Containers package the app and dependencies together, making them highly portable.

---

### Scaling & Deployment

#### Virtual Machines

* Scaling means **creating new VMs**
* Slow provisioning
* Less efficient for microservices

#### Containers

* Scale by **spawning new containers**
* Very fast
* Ideal for microservices & cloud-native apps

---

### Use Cases

#### Use Virtual Machines When:

* You need **different operating systems**
* Strong security isolation is required
* Running legacy applications
* Compliance requires OS-level isolation

#### Use Containers When:

* Building **microservices**
* CI/CD pipelines
* Cloud-native applications
* Fast startup and scaling are important

---

### Real-World Example

#### VM Example

Running:

* Windows Server VM
* Ubuntu VM
* CentOS VM
  All on the same physical machine

#### Container Example

Running:

* Node.js API
* Redis
* PostgreSQL
  All sharing the same Linux kernel

</details>


## Why Containers Are Useful

### Scenario 1: “Works on My Machine”

Containers package the application **with its exact dependencies**, ensuring consistent behavior across development, testing, and production environments.

---

### Scenario 2: Isolated Environments

Each container runs independently:

* No dependency conflicts
* Safe experimentation
* Easy cleanup

---

### Scenario 3: Development with Multiple Services

Example application dependencies:

* PostgreSQL
* MongoDB
* Redis

Containers allow each service to run independently without polluting the host system.

---

### Scenario 4: Scaling & Reliability

With container orchestration (e.g., Kubernetes):

* Failed containers are restarted automatically
* Traffic is load-balanced across replicas
* Horizontal scaling becomes simple

---

## What Is Docker?

**Docker** is a set of tools that enables developers to:

* Build container images
* Run containers
* Share images via registries

**In short:**

> Docker helps you package, ship, and run applications consistently.

---

## Docker Image vs Docker Container

| Docker Image   | Docker Container  |
| -------------- | ----------------- |
| Blueprint      | Running instance  |
| Immutable      | Mutable (runtime) |
| Like a program | Like a process    |

**Example:**

```sh
docker run nginx
```

* `nginx` → image
* Running nginx instance → container

---

## Basic Docker Commands

```sh
docker --version
docker version
```

---

## Docker Images

### Build an Image

```sh
# docker build -t image_name path_of_dockerfile
docker build -t myapp .
```

### List Images

```sh
docker image ls

# Old way
docker images
```

### Pull an image from docker repository

```sh
# docker pull registry_omain/image_name:tag
docker pull private-registry-domain/myapp:1.0

# if we don't give repository domain,
# it will pull from docker hub by default
docker pull node:lts
```

### Push Image

```sh
# docker push registry_omain/image_name:tag
docker push private-registry-domain/myapp:1.0

# if we don't give repository domain,
# it will push to docker hub by default
docker pull myapp:lts
```

### Remove Image

```sh
docker image rm myapp:1.0 # docker image rm image_name/image_id

# old way
docker rmi myapp:1.0
```

### Rename image name

```sh
# docker tag old_image_name new_image_name
docker tag myapp:1.0 myapp:latest
```

### Load an image from a tar archive

```sh
# docker load -i name.tar
docker load -i myapp.tar
```

### Remove unused images

```sh
docker image prune
```

---

## Docker Containers

Container Information

| CONTAINER ID | IMAGE | COMMAND                | CREATED       | STATUS       | PORTS                  | NAMES         |
| ------------ | ----- | ---------------------- | ------------- | ------------ | ---------------------- | ------------- |
| d58a6b05e3b9 | redis | "docker-entrypoint.s…" | 3 minutes ago | Up 3 minutes | 0.0.0.0:6000->6379/tcp | boring_agnesi |

### Run a Container

```sh
# run a container in attached mode with a random name
docker run myapp:1.0

# run a container in detached mode with a random name
docker run -d myapp:1.0

# run a container with the provided name
# docker run --name container_name image_name:tag
docker run --name myapp myapp:1.0

# run a container with binding port
# docker run -p host_port:container_port image_name:tag
docker run -p 8080:8081 myapp:1.0

docker run -d -p 8080:8081 --name myapp myapp:1.0
```

### List Containers

```sh
# list all running containers
docker container ls # OR, docker ps

# list all containers including stopped ones
docker container ls -a # OR, docker ps -a
```

### Stop / Start

```sh
# docker stop container_name/container_id
docker stop myapp

# docker start container_name/container_id
docker start myapp
```

### Execute command in running container

> It is useful for debugging, troubleshooting, or when we need to run commands directly inside the container. Once the container is running in interactive mode, we can execute commands within the container just as if we were inside a regular terminal.

```sh
# execute a specific command (sh or bash) in running container
# docker exec -it container_name/container_id sh/bash
docker exec -it myapp sh # OR, docker exec -it myapp /bin/sh
```

### Run a container in interactive mode

> It is useful for debugging, troubleshooting, or when we need to run commands directly inside the container. Once the container is running in interactive mode, we can execute commands within the container just as if we were inside a regular terminal. 

```sh
# run container in interactive mode with default command
# -i means interactive and -t means terminal
docker run -it myapp

# run container in interactive mode with overriding the default command with "sh" or "bash"
dcoker run -it myapp bash
```

type `exit` or `Ctrl + D`: To exit the interactive mode and stoping the container. \
`Ctrl + P` followed by `Ctrl + Q`: To exit the interactive mode without stopping the container.

### Remove a container

```sh
# remove a stopped container
# docker container rm container_name/container_id
docker container rm myapp # OR, docker rm myapp

# remove a running container forcefully
docker container rm -f myapp # docker rm -f myapp
```

### View container logs

> It is useful for debugging, and troubleshooting.

```sh
docker logs myapp
```

### Pause/unpause a running container

```sh
docker pause myapp

docker unpause myapp
```

---

## Docker Volumes

Docker containers use an **isolated filesystem**.
By default, **any data written inside a container is lost** when the container is stopped, removed, or recreated.

To solve this problem, Docker provides **volumes**.

---

### What Is a Docker Volume?

A **Docker volume** is a way to **store data outside the container’s filesystem** so that the data **persists even after the container is deleted**.

In simple terms:

> A Docker volume connects a directory inside a container to a directory on the host machine (or to Docker-managed storage).

---

### Why Volumes Are Important

Without volumes:

* Container is deleted → data is lost

With volumes:

* Container is deleted → **data remains**
* New container can reuse the same data

---

### How Docker Volumes Work

* The container writes data to a directory
* That directory is **mounted** to a location outside the container
* Docker keeps this data independent from the container lifecycle

```
Container filesystem  --->  Volume (persistent storage)
```

---

### Data Synchronization

* When data changes **inside the container**, it is immediately reflected in the volume
* When data changes **in the volume**, the container sees the updated data

This works because both paths point to the **same storage location**.

---

### Simple Example

```sh
docker run -v my-data:/var/lib/mysql mysql
```

* `/var/lib/mysql` → path inside the container
* `my-data` → Docker-managed persistent volume
* Database data stays safe even if the container is removed

---

> **Containers are temporary. Volumes are permanent.**

Use Docker volumes whenever your application needs to store:

* Databases
* User uploads
* Logs
* Configuration data

### Create a named volume

```sh
docker volume create volume_name
```

### List all volumes

```sh
docker volume ls
```

### Remove a volume

```sh
docker volume rm volume_name
```

### Run a container with a volume

```sh
# This is called host vaolumes
# Here, user defines the host file system path
# -v /path/in/host:/path/in/container
docker run -v /home/app-data:/var/lib/mysql/data myapp

# This is called annonymous vaolumes
# Here, docker defines the host file system path with a random name
# -v /path/in/container
docker run -v /var/lib/mysql/data myapp

# This is called named vaolumes
# Here, docker defines the host file system path with the provided name
# -v volume_name:/path/in/container
docker run -v my-data:/var/lib/mysql/data myapp
```

### Copy files between a contain and a volume

```sh
# copy from container to volume
docker cp container_name:/path/to/source/file_or_directory /path/in/volume

# copy from volume to container
docker cp /path/in/volume container_name:/path/in/container
```

## Docker Network

By default, Docker containers are **isolated from each other** and from the outside world.
They cannot communicate unless Docker explicitly allows it.

To control **how containers talk to each other and to external systems**, Docker provides **networks**.

---

### What Is a Docker Network?

A **Docker network** is a virtual network that connects containers together and controls:

* Which containers can communicate
* How they discover each other
* How traffic flows in and out of containers

In simple terms:

> A Docker network acts like a **private LAN** for containers.

---

### Why Docker Networks Are Important

Without Docker networks:

* Containers cannot easily talk to each other
* You must use IP addresses (which change)
* Communication becomes fragile

With Docker networks:

* Containers can communicate using **container names**
* Network isolation improves security
* Multi-container applications become manageable

---

### How Docker Networks Work

* Docker creates a **virtual network layer**
* Each container gets:

  * A virtual network interface
  * An internal IP address
* Docker provides built-in **DNS**, so containers can find each other by name

```
[ web ] ----\
             ---> Docker Network ---> Internet
[ db  ] ----/
```

---

### Container-to-Container Communication

If two containers are on the **same Docker network**:

```sh
docker network create app-network

docker run --name db --network app-network postgres
docker run --name api --network app-network my-api
```

Inside `api`, you can connect to the database using:

```
db:5432
```

No IP addresses needed.

---

### Container-to-Host / Internet Communication

* Containers can access the internet by default
* To access a container from the host or browser, **ports must be published**

```sh
docker run -p 8080:80 nginx
```

* Host port `8080` → Container port `80`

---

### Types of Docker Networks (Common Ones)

| Network Type | Purpose                 | When to Use                     |
| ------------ | ----------------------- | ------------------------------- |
| bridge       | Default private network | Most local development          |
| host         | Share host network      | High performance, low isolation |
| none         | No networking           | Maximum isolation               |
| overlay      | Multi-host networking   | Docker Swarm / Kubernetes       |

---

### Network Isolation

* Containers on **different networks cannot communicate**
* This improves security by limiting access

Example:

* `frontend` network → web app
* `backend` network → database

Only containers connected to both can act as bridges.

---

### Simple Mental Model

> **Docker Network = Virtual switch + DNS for containers**

---

### Key Takeaway

* Containers communicate through Docker networks
* Names are more important than IPs
* Networks provide isolation, discovery, and control

---

### Create a user-defined bridge network

```sh
docker network create network_name
```

### List all networks

```sh
docker network ls
```

### Connect a container to a network

```sh
docker network connect network_name container_name
```

### Disconnect a container from a network

```sh
docker network disconnect network_name container_name
```

### Run a container within a specific network

```sh
docker run --network network_name image_name # OR, docker run -net network_name image_name
```

### Inspect details of an image, container, volume, and network

```sh
docker image inspect myapp:1.0 # docker image inspect image_name:tag
docker container inspect myapp # docker container inspect container_name
docker volume inspect my-data # docker volume inspect volume_name
docker network app-network # docker network inspect network_name

# to format and filter the JSON output
docker image inspect myapp | jq

# we can also use "docker inspect" for all of them
docker inspect myapp
```

---

## Dockerfile

A **Dockerfile** is a plain text file that contains **step-by-step instructions** telling Docker how to build an image.

In simple terms:

> A Dockerfile is a **recipe** for creating a Docker image.

* Each instruction creates a **layer** in the image
* Layers are cached to speed up rebuilds
* The final image is immutable

---

### Basic Dockerfile Structure

```Dockerfile
# Base image
FROM node:18

# Set working directory
WORKDIR /app

# Copy files
COPY package.json .

# Install dependencies
RUN npm install

# Copy application code
COPY . .

# Start the application
CMD ["npm", "start"]
```

---

### Dockerfile Instructions (One by One)

#### FROM

```Dockerfile
FROM node:18
```

* Defines the **base image**
* Must be the **first instruction** (except ARG)
* Every image starts from another image

Think of it as:

> "Start building my image on top of this image"

---

#### ARG

```Dockerfile
ARG NODE_ENV=production
```

* Defines **build-time variables**
* Available only during `docker build`
* Not available at runtime

```sh
docker build --build-arg NODE_ENV=development .
```

---

#### ENV

```Dockerfile
ENV NODE_ENV=production
```

* Defines **runtime environment variables**
* Available inside the container
* Persists in the final image

Difference from ARG:

* `ARG` → build time only
* `ENV` → runtime

---

#### WORKDIR

```Dockerfile
WORKDIR /app
```

* Sets the working directory inside the image
* Automatically creates the directory if it doesn’t exist
* All following commands run from this directory

Equivalent to:

```sh
cd /app
```

---

#### COPY

```Dockerfile
COPY . .
```

* Copies files from **host → image**
* Simple and predictable

Best practice:

* Prefer `COPY` over `ADD`

---

#### ADD (Use Carefully)

```Dockerfile
ADD archive.tar.gz /app
```

* Can extract tar files automatically
* Can download files from URLs

⚠️ Often causes confusion. Use only when needed.

---

#### RUN

```Dockerfile
RUN npm install
```

* Executes commands **at build time**
* Creates a new image layer
* Used to install dependencies or build artifacts

Think of it as:

> "Prepare the image"

---

#### EXPOSE

```Dockerfile
EXPOSE 3000
```

* Documents which port the container listens on
* Does **not** publish the port

Publishing still requires:

```sh
docker run -p 3000:3000 myapp
```

---

#### USER

```Dockerfile
USER node
```

* Runs the container as a non-root user
* Improves security

---

#### VOLUME

```Dockerfile
VOLUME /data
```

* Declares a mount point for persistent data
* Actual volume is created at runtime

---

#### HEALTHCHECK

```Dockerfile
HEALTHCHECK CMD curl --fail http://localhost:3000 || exit 1
```

* Tells Docker how to check container health
* Useful for orchestration systems

---

#### SHELL

```Dockerfile
SHELL ["/bin/bash", "-c"]
```

* Changes the default shell for RUN commands

---

#### LABEL

```Dockerfile
LABEL maintainer="dev@example.com"
```

* Adds metadata to the image
* Useful for documentation and automation

---

### 4. CMD vs ENTRYPOINT

Both **CMD** and **ENTRYPOINT** define what runs when a container starts, but they serve different purposes.

---

### CMD

* Provides **default arguments**
* Can be easily overridden at runtime

**Dockerfile example:**

```Dockerfile
FROM ubuntu
CMD ["echo", "Hello World"]
```

**Run:**

```sh
docker run myimage
# Output: Hello World

# Override CMD
docker run myimage echo "Hi"
# Output: Hi
```

Think of CMD as:

> "Default behavior"

---

### ENTRYPOINT

* Defines the **main command**
* Not overridden unless explicitly requested

**Dockerfile example:**

```Dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
```

**Run:**

```sh
docker run myimage Hello
# Output: Hello
```

Think of ENTRYPOINT as:

> "This container *is* this command"

---

### ENTRYPOINT + CMD (Best Practice)

Use **ENTRYPOINT** for the executable and **CMD** for default arguments.

**Dockerfile example:**

```Dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
CMD ["Hello World"]
```

**Behavior:**

```sh
docker run myimage
# Output: Hello World

docker run myimage Hi
# Output: Hi
```

* ENTRYPOINT → executable
* CMD → default arguments

---

### Summary Table of CMD vs Entrypoint

| Feature     | CMD          | ENTRYPOINT          |
| ----------- | ------------ | ------------------- |
| Purpose     | Default args | Main command        |
| Overridable | Yes          | No (by default)     |
| Common use  | Defaults     | Production behavior |

---

### Key Takeaways

* Dockerfile defines **how an image is built**
* Each instruction creates a layer
* Use ENTRYPOINT for executables
* Use CMD for defaults

> **Good Dockerfiles are predictable, minimal, and explicit.**


## CMD vs ENTRYPOINT

Both **CMD** and **ENTRYPOINT** define what command runs when a container starts.

---

### CMD

* Provides **default arguments**
* Can be easily overridden at runtime

**Dockerfile example:**

```Dockerfile
FROM ubuntu
CMD ["echo", "Hello World"]
```

**Run:**

```sh
docker run myimage
# Output: Hello World

# Override CMD
docker run myimage echo "Hi"
# Output: Hi
```

---

### ENTRYPOINT

* Defines the **main command**
* Not overridden unless explicitly requested

**Dockerfile example:**

```Dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
```

**Run:**

```sh
docker run myimage Hello
# Output: Hello
```

---

### ENTRYPOINT + CMD (Best Practice)

Use **ENTRYPOINT** for the executable and **CMD** for default arguments.

**Dockerfile example:**

```Dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
CMD ["Hello World"]
```

**Behavior:**

```sh
docker run myimage
# Output: Hello World

docker run myimage Hi
# Output: Hi
```

---

### Summary Table

| Feature     | CMD               | ENTRYPOINT       |
| ----------- | ----------------- | ---------------- |
| Purpose     | Default arguments | Main command     |
| Overridable | Yes               | No (by default)  |
| Common use  | Dev defaults      | Production entry |

---