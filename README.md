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

## Basic commands

```sh
> docker # show docker commands & management commands

> docker -v # show docker version

> docker version # show docker version info
```

## Container commands

Here `<container>` equals __container name__ or __container id__
  
| command                                     | explain                                 | shorthand/notes |
| ------------------------------------------- | --------------------------------------- | --------------- |
| docker container run <image>                | Runs a container from an image          | docker run      |
| docker container ls -a                      | Lists all containers                    | docker ps -a    |
| docker container rm <container>             | Removes a container                     | docker rm       |
| docker container stop <container>           | Stops a container                       | docker stop     |
| docker stop $(docker ps -aq)                | Stops all running containers            |                 |
| docker container exec <container>           | Executes a command inside the container | docker exec     |
| docker container rm <container>             | removes a stop container                |                 |
| docker container rm -f <container>          | removes a running container forcefully  |                 |
| docker container rm <container> <container> | removes multiple containers             |                 |
| docker rm $(docker ps -aq)                  | removes all containers                  |                 |
| docker logs <container>                     | gets logs                               |                 |
| docker container top <container>            | list processes running in container     |                 |

> In fish, __$__ is used only for variables. Correct notation equivalent to bash __$(command)__ is just __(command)__ in fish.

## Image commands
  
Here `<image>` equals __image name__ or __image id__
  
| command                           | explain                                 | shorthand     |
| --------------------------------- | --------------------------------------- | ------------- |
| docker image ls                   | Lists all images                        | docker images |
| docker image rm <image>           | Removes an image                        | docker rmi    |
| docker image pull <image>         | Pulls image from a docker registry      | docker pull   |
