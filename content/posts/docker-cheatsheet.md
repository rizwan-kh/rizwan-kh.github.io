---
title: "Docker Cheat Sheet"
date: 2019-10-11T05:11:13+04:00
draft: false

tags:
- docker
---
![docker](/docker.png)
Below are few of the main and basic commands used in Docker, an easy pick-up and good-to-go command page for docker troubleshooting.

## Alias
---
If you are a lazy developer/sysadmin like me, the first thing you should do on your system is to make easy alias of all the long commands, below are the ones I often use on any system I use on a  daily basis:

These can be imported on ~/.bashrc (if you use bash) or ~/.zshrc (if you are a MAC user and use ZSH)

{{< gist rizwan-kh 032e587751e54a2fd26f44c0267ea5c5 >}}

### Commands and their usage
Mostly used commands are aliased above, but to explain what each does, please read on
```
# to build a docker image with a certain name and certain tag, use the below Docker build comamnd
docker build --tag imagename:tagname --file /path/to/Dockerfile


# to check the docker images
docker images


# to run a docker container in detach mode publishing hostport:containerport and mounting a host vol to container vol, giving a name to the runnging container and the hostname to container
docker run --tty --interactive --publish 2222:22 --hostname my-x-host --volume /hostvolume:/containervol --name name-of-running-container --detach imagename:tagname
# docker run -ti -p 2222:22 -h my-x-host -v /hostvolume:/containervol -n name-of-running-container -d imagename:tagname (this is shorter version of the above command)


# to see all running containers
docker ps


# to see all containers (running, stopped, exited, etc.)
docker ps -a


# to get inside a running container
docker exec -ti CONTAINERNAME/CONTAINERID bash


# to stop a running container
docker stop CONTAINERNAME/CONTAINERID


# to remove a stopped container
docker rm CONTAINERNAME/CONTAINERID

# to see logs from containers
docker logs CONTAINERNAME/CONTAINERID
```
