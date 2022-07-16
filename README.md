# Docker

- Docker is a set of tools to deliver software in containers.
- Containers are packages of software.
- Containers are isolated so that they don’t interfere with each other or the software running outside of the containers.

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

## Virtual Machine vs Docker

![image](https://user-images.githubusercontent.com/11992095/125337394-c40d9980-e370-11eb-8bca-29546750357e.png)

source: from internet

## Basic commands

```sh
> docker # show docker commands & management commands

> docker -v # show docker version

> docker version # show docker version info
```

## Container commands

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


> In fish, **$** is used only for variables. Correct notation equivalent to bash **$(command)** is just **(command)** in fish.

## Image commands

Here `<image>` equals **image name** or **image id**

| command                             | explain                            | shorthand/notes                                      |
| ----------------------------------- | ---------------------------------- | ---------------------------------------------------- |
| `docker image ls`                   | Lists all images                   | `docker images`                                      |
| `docker image pull <image>`         | Pulls image from a docker registry | `docker pull`                                        |
| `docker image rm <image>`           | Removes an image                   | `docker rmi`                                         |
| `docker rmi $(docker images -a -q)` | Removes all images                 |                                                      |
| `docker tag <old_name> <new_name>`  | Renames image                      | then remove old image using, `docker rmi <old_name>` |
