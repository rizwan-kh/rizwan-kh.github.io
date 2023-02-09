---
title: "Reload nginx without restarting the container/pod"
date: 2023-02-08T20:04:03-04:00
draft: false
toc: false
tags:
  - nginx container
  - nginx pod

---
![nginx](/nginx.png)
## Introduction
Today in a troubleshooting session, one of the pods we were working with had an Angular application exposed via an nginx server and we wanted to change a few settings in the custom-nginx.conf, for which the developer informed us that he would need to rebuild the image and the CI pipeline will take around 10-15mins for the build, scan and deploy.

I suggested two approaches
- map custom-nginx.conf as a configmap in the pod
- restart/reload nginx process

We went ahead with the second approach as it was the more time-saving option. 

## Steps
Reload the nginx process on the pod using kubectl exec
```
kubectl exec -ti app-nginx-ifbsdy -n ui -- nginx -s reload
2023/02/09 01:39:02 [notice] 3269#3269: signal process started
```

If you're running a docker container, exec inside the container

```
docker exec -it app-nginx nginx -s reload
2023/02/09 01:39:02 [notice] 3269#3269: signal process started
```