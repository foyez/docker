# Docker

## Basic commands

```sh
> docker # show docker commands & management commands

> docker -v # show docker version

> docker version # show docker version info
```

## Container commands

```sh
> docker container stop [CONTAINER_ID] # stop a running container

> docker stop $(docker ps -aq) # stop all running containers
# In fish, $ is used only for variables. Correct notation equivalent to bash $(command) is just (command) in fish.
```
