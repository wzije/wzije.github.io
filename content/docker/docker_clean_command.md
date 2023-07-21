+++
title= "Docker Clean Command"
date= 2022-03-01T15:10:38+07:00
tags= ['docker','clean']
+++

## Docker – How to cleanup (unused) resources

Once in a while, you may need to cleanup resources (containers, volumes, images, networks) …

## delete volumes

```bash
$ docker volume rm $(docker volume ls -qf dangling=true)
$ docker volume ls -qf dangling=true | xargs -r docker volume rm

# delete networks
$ docker network ls
$ docker network ls | grep "bridge"
$ docker network rm $(docker network ls | grep "bridge" | awk '/ / { print $1 }')
```

## remove docker images

```bash
$ docker images
$ docker rmi $(docker images --filter "dangling=true" -q --no-trunc)
$ docker images | grep "none"
$ docker rmi $(docker images | grep "none" | awk '/ / { print $3 }')
```

## remove docker containers
```bash
$ docker ps
$ docker ps -a
$ docker rm $(docker ps -qa --no-trunc --filter "status=exited")
```

## Resize disk space for docker vm
```bash
$ docker-machine create --driver virtualbox --virtualbox-disk-size "40000" default
```

Referensi: 
- [Clean Up Volume](https://github.com/chadoe/docker-cleanup-volumes)
- [Remove unused](http://stackoverflow.com/questions/32723111/how-to-remove-old-and-unused-docker-images) 
