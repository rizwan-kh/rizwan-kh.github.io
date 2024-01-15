---
title: "How to use service accounts for Kubernetes imagePullSecrets"
date: 2020-01-02T17:40:27+04:00
draft: false
tags:
- kubernetes
- serviceAccounts
---
![Kubernetes](/kubernetes.jpg)
## What are Service Accounts in Kubernetes?
As per [Kubernetes.io](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) - A service account provides an identity for processes that run in a Pod.

One can think of service accounts as service users for pods. They help pods authenticate with the api-server and interact with it.

##

Many times, we come across a situation where our organization uses a private Docker registry to store the Docker images and to make this available we need to create a `docker-registry` kubernetes secret and pass as `imagePullSecrets` in the deployment manifest.


```sh
kubectl create secret docker-registry registry-cred \
 --docker-server=my.private-registry.com \
 --docker-username=my_username \
 --docker-password="my_superr_strong_password" \
 --docker-email=my.email@mycompany.com -n my-namespace
```


Then, we pass this secret in the deployment manifest as below.
```yml
...
      imagePullSecrets:
      - name: registry-cred
...

```


## The problem with this approach?
Not many that I can think of, except the below ones:
* The deployment yaml is generally developed by Developers who don't need to know about these credentials
* If there are a large number of pods in the namespace, then each manifest needs to be updated, whenever the password is rotated


## The solution
`serviceAccounts` - your Kubernetes administrator can just patch serviceAccounts with the registry credential secret and you don't need to worry about replacing or adding it in your manifest yaml each time.


```sh
kubectl patch serviceaccount default \
-p '{"imagePullSecrets": [{"name": "registry-cred"}]}' -n my-namespace
```
