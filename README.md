# Docker

## Bare Metal

> Bare metal is a computer system without a base operating system (OS) or installed applications. It is a computer's hardware assembly, structure and components that is installed with either the firmware or basic input/output system (BIOS) software utility or no software at all. <sup>[ref](https://www.techopedia.com/definition/2153/bare-metal)</sup>

## Virtual Machines

> A virtual machine (VM) is a virtual environment that functions as a virtual computer system with its own CPU, memory, network interface, and storage, created on a physical hardware system (located off- or on-premises). Software called a hypervisor separates the machine’s resources from the hardware and provisions them appropriately so they can be used by the VM. VM allows to have multiple guest instances of OS (e.g. Linux, Windows, etc) running inside of a host instance of OS. <sup>[ref](https://www.redhat.com/en/topics/virtualization/what-is-a-virtual-machine)</sup>

## Container

> Container using a few features of Linux together to achieve isolation.

## Benefits from containers

#### Scenario 1: Works on my machine

You create a software. It works on your computer. It may even go through a testing pipeline working perfectly. You send it to the server and it does not work. This known as the “__works on my machine__” problem.\
\
Containers solve this problem by allowing the developer to personally run the application inside a container, which then includes all of the dependencies required for the app to work.

#### Scenario 2: Isolated environments

### Scenario 3: Development

You will setup a web app which uses these services when running: a postgres database, mongodb, redis and a number of others. Simple enough, you install whatever is required to run the application and all of the applications that it depends on.

#### Scenario 4: Scaling

What happens when one application dies? Container orchestration system notices it, splits traffic between the working replicas and spin up a new container to replace the dead one.

# Docker

- Docker is a set of tools to deliver software in containers.
- Containers are packages of software.
- Containers are isolated so that they don’t interfere with each other or the software running outside of the containers.

## Virtual Machine vs Docker

![image](https://user-images.githubusercontent.com/11992095/125337394-c40d9980-e370-11eb-8bca-29546750357e.png)

source: from internet

## Docker Image vs Container

A Docker image packs up the application and environment required by the application to run, and a container is a running instance of the image. Images are the packing part of Docker, analogous to "source code" or a "program". Containers are the execution part of Docker, analogous to a "process". <sup>[ref](https://stackoverflow.com/questions/23735149/what-is-the-difference-between-a-docker-image-and-a-containe)</sup>

## Basic commands

```sh
> docker # show docker commands & management commands

> docker -v # show docker version

> docker version # show docker version info

> docker run -d -p 6000:6379 --name boring_agnesi redis # docker run -d -p <HOST_PORT:CONTAINER_PORT> --name <CONTAINER_NAME> <IMAGE | CONTAINER ID | NAME>

> docker exec -it d58a6b05e3b9 /bin/bash # executes command inside container
# or, docker exec -it d58a6b05e3b9 bash
# or, docker exec -it d58a6b05e3b9 sh
```

## Docker Image

- **Build an Image from Dockerfile**

```sh
# docker build -t image_name path_of_dockerfile
docker build -t myapp .
```

- **List all local images**

```sh
docker image ls

# Old way
docker images
```

- **Pull an image from docker repository**

```sh
# docker pull registry_omain/image_name:tag
docker pull private-registry-domain/myapp:1.0

# if we don't give repository domain,
# it will pull from docker hub by default
docker pull node:lts
```

- **Push image to docker registry**

```sh
# docker push registry_omain/image_name:tag
docker push private-registry-domain/myapp:1.0

# if we don't give repository domain,
# it will push to docker hub by default
docker pull myapp:lts
```

- **Remove a local image**

```sh
docker image rm myapp:1.0 # docker image rm image_name/image_id

# old way
docker rmi myapp:1.0
```

- **Rename image name**

```sh
# docker tag old_image_name new_image_name
docker tag myapp:1.0 myapp:latest
```

- **Save an image to a tar archive**

```sh
# docker save -o name.tar image_name:tag
docker save -o myapp.tar mypp.1.0
```

- **Load an image from a tar archive**

```sh
# docker load -i name.tar
docker load -i myapp.tar
```

- **Remove unused images**

```sh
docker image prune
```

## Docker Container

Container Information

| CONTAINER ID | IMAGE | COMMAND                | CREATED       | STATUS       | PORTS                  | NAMES         |
| ------------ | ----- | ---------------------- | ------------- | ------------ | ---------------------- | ------------- |
| d58a6b05e3b9 | redis | "docker-entrypoint.s…" | 3 minutes ago | Up 3 minutes | 0.0.0.0:6000->6379/tcp | boring_agnesi |

- **Run a container from an image**

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

docker run -d \
-p 8080:8081 \
--name myapp \
myapp:1.0
```

- **List containers**

```sh
# list all running containers
docker container ls # OR, docker ps

# list all containers including stopped ones
docker container ls -a # OR, docker ps -a
```

- **Stop a running container**

```sh
# docker stop container_name/container_id
docker stop myapp
```

- **Start a stopped container**

```sh
# docker start container_name/container_id
docker start myapp
```

- **Execute command in running container**

> It is useful for debugging, troubleshooting, or when we need to run commands directly inside the container. Once the container is running in interactive mode, we can execute commands within the container just as if we were inside a regular terminal.

```sh
# execute a specific command (sh or bash) in running container
# docker exec -it container_name/container_id sh/bash
docker exec -it myapp sh # OR, docker exec -it myapp /bin/sh
```

- **Run a container in interactive mode**

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

- **Remove a container**

```sh
# remove a stopped container
# docker container rm container_name/container_id
docker container rm myapp # OR, docker rm myapp

# remove a running container forcefully
docker container rm -f myapp # docker rm -f myapp
```

- **View container logs**

> It is useful for debugging, and troubleshooting.

```sh
docker logs myapp
```

- **Pause/unpause a running container**

```sh
docker pause myapp

docker unpause myapp
```

Here `<container>` equals __container name__ or __container id__

| command                                           | explain                                   | examples/notes                          |
| ------------------------------------------------- | ----------------------------------------- | --------------------------------------- |
| `docker run <image>`                              | Runs a container from an image            | combine: `docker pull` + `docker start` |
| `docker run -d <image>`                           | Runs a container in detached mode         | docker starts in background             |
| `docker run -p<HOST_PORT:CONTAINER_PORT> <image>` | Runs a container (binding port with host) | `docker run -p4000:3000`                |
| `docker ls -a`                                    | Lists all containers                      | `docker ps -a`                          |
| `docker ls -a &#124; grep <container_name>`       | Filters container by name                 |                                         |
| `docker rm <container>`                           | Removes a container                       | `docker rm`                             |
| `docker stop <container>`                         | Stops a container                         | `docker stop`                           |
| `docker stop $(docker ps -aq)`                    | Stops all running containers              |                                         |
| `docker exec <container>`                         | Executes a command inside the container   | `docker exec`                           |
| `docker rm <container>`                           | removes a stop container                  |                                         |
| `docker rm -f <container>`                        | removes a running container forcefully    |                                         |
| `docker rm <container> <container>`               | removes multiple containers               |                                         |
| `docker rm $(docker ps -aq)`                      | removes all containers                    |                                         |
| `docker logs <container>`                         | gets logs                                 |                                         |
| `docker top <container>`                          | list processes running in container       |                                         |
| `docker container inspect <container>`                          |  Display detailed information on one or more containers       |    `docker container inspect postgres15`                                     |


> In fish, **$** is used only for variables. Correct notation equivalent to bash **$(command)** is just **(command)** in fish.

## Docker Volumes

> Docker valumes are used for data persistence. Docker relies on virtual file system. When a container is restarted or removed, the data is lost. Therefore, creating volumes becomes essential to preserve data even across container lifecycle events.

> Docker volumes mean that a pysical file system path is mounted into the virtual file system path in Docker. This allows for synchronization between the virtual file system and the host file system. When virtual file system is updated, the host file system gets automatically replicated, or vice varsa.

- **Create a named volume**

```sh
docker volume create volume_name
```

- **List all volumes**

```sh
docker volume ls
```

- **Remove a volume**

```sh
docker volume rm volume_name
```

- **Run a container with a volume**

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

- **Copy files between a contain and a volume**

```sh
# copy from container to volume
docker cp container_name:/path/to/source/file_or_directory /path/in/volume

# copy from volume to container
docker cp /path/in/volume container_name:/path/in/container
```

## Docker Network

- **Create a user-defined bridge network**

```sh
docker network create network_name
```

- **List all networks**

```sh
docker network ls
```

- **Connect a container to a network**

```sh
docker network connect network_name container_name
```

- **Disconnect a container from a network**

```sh
docker network disconnect network_name container_name
```

- **Run a container within a specific network**

```sh
docker run --network network_name image_name # OR, docker run -net network_name image_name
```

- **Inspect details of an image, container, volume, and network**

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

## Docker Compose
