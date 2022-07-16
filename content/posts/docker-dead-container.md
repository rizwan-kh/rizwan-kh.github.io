---
title: "Working with dead container"
date: 2022-07-16T11:39:13-05:00
draft: false

tags:
- docker
- container
---
![docker](/docker.png)
I came across this question from a team member where they wanted to troubleshoot a dead container. I use the below process and I thought why not do a small write up to help wider audience.

## Usage
Process is very simple - to save/commit the dead container to a new image and then start a new container with a `sh` entrypoint and debug the container. I consider you're using docker, although similar equivalent commands for other container runtime alternatives like podman, ctr, etc. can also be found, but the process will be similar.

```
# commit the stopped container to a new image
docker commit 25836caaa158 dead/test

# run docker images to see the image in the list of images present on the host
docker images
REPOSITORY    TAG     IMAGE ID       CREATED         SIZE  
dead/test     latest  cc9db32dcc2d   2 seconds ago   280.7MB

# start a new container with shell entrypoint to debug
docker run -it --rm --entrypoint sh dead/test
```

After you debugging is completed, feel free to delete the image
```
docker rmi dead/test
```