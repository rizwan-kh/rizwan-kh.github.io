---
title: "Gitlab CI & Kaniko to build Docker Images"
date: 2020-02-12T12:11:00+04:00
draft: false

tags:
- gitlab
- docker
- dind
- kaniko
---
![GitLab](/gitlab.jpeg)
### Introduction
You can build container images from a Dockerfile inside a container or a Kubernetes cluster, though *Jérôme Petazzoni* strongly discourages from doing so. He wrote a detailed blog that can be read [here](http://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/) on why not to build container images using Dockerfile inside a container or a Kubernetes cluster.

---
#### Context
You will get ```N``` number of blogs on how to use the CI/CD of GitLab; here we will see an easy reference point to extend a file and create CI/CD for Docker Image to be built and stored in the same GitLab registry using [kaniko](https://cloud.google.com/blog/products/gcp/introducing-kaniko-build-container-images-in-kubernetes-and-google-container-builder-even-without-root-access). This file needs to be created in the individual project in the GitLab using the template available with the name .gitlab-ci.yml.

---
#### What is Kaniko?
```Note: Kaniko is not an officially supported Google product```
It is a tool to build container images from a Dockerfile inside a container or a Kubernetes cluster. It doesn't depend on the Docker daemon to run each Dockerfile command.

It comes with it's own limitations, but we don't run the risk of using Docker-in-Docker

---
#### Prerequisites
- Access to GitLab (either private self hosted or managed)
- GitLab project with a Dockerfile

---
#### CI YAML for auto-devops
```
# .gitlab-ci.yml
variables:
    GIT_SSL_NO_VERIFY: "true"

before_script:
  - echo "Random image creation, user = $GITLABUSER"

stages:
  - build

build_image:
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  stage: build
  script:
    - ls
    - pwd
    - export CI_REGISTRY_IMAGE=mygitlab.com/base-project/subproject/project
    - echo "{\"auths\":{\"mygitlab.com\":{\"username\":\"gitlab-ci-token\",\"password\":\"$CI_BUILD_TOKEN\"},\"repository.xyz-company.io\":{\"username\":\"user\",\"password\":\"123random\"}}}" > /kaniko/.docker/config.json
    - wget https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem | xargs cat lets-encrypt-x3-cross-signed.pem >> /kaniko/ssl/certs/ca-certificates.crt
    - /kaniko/executor --skip-tls-verify --context $CI_PROJECT_DIR --dockerfile $CI_PROJECT_DIR/Dockerfile --destination $CI_REGISTRY_IMAGE:$CI_BUILD_REF_NAME

```

---
**variables**: These are static values which aren't going to change and is used at multiple location in the gitlab-ci.yml file

**before_script**: Set(s) of commands or echo statement we want to print

**stages**: Stages are block of code for an identical job or set of jobs viz. build, test, clean-up, delete, deploy, etc. This executes in the order it's defined in the YAML. A dot(.) in front of any job(block of code) disables it and it won't be executed or available neither as an automatic or manual job.

**Jobs (Each block of code)**: Each block of individual stage contains key-value pair or set of commands to it. We can define each block of code to point to a particular stage and all the set of commands it requires to perform that function in the script block. It can be made to run automatically and also manual (start the job manually by clicking a button). The variables like password, access/secret key can be defined in the CI/CD settings under secret variables section so it's not available in plain text format.

---

**Note**: If you want to use the GitLab docker registry and store docker images in the GitLab project; this by default is disabled and needs to be enabled in the General setting section.
