---
title: "Restart nginx without restarting the container/pod"
date: 2023-02-08T20:04:03-04:00
draft: false
toc: false
tags:
  - nginx container
  - nginx pod

---
![nginx](/nginx.png)
## Introduction
Today in a troublshooting session, one of the pod we were working with had Angular application exposed via a nginx server and we wanted to change few setting in the custom-nginx.conf, for which the developer informed us that he would need to re-build the image and the CI pipeline will take around 10-15mins for build, scan and deploy.

I suggested below two approaches
- map custom-nginx.conf as a configmap in the pod
- restart/reload nginx process

We went ahead with the second approach as it was the more time-saving option. 

## Steps
Reload the nginx process on the pod using kubectl exec
```
kubectl exec -ti app-nginx-ifbsdy -n ui -- nginx -s reload
```

If you're running a docker container, exec inside the container

```
docker exec -it app-nginx nginx -s reload
```