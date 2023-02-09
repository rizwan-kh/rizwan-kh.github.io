---
title: "Klar"
date: 2020-03-14T17:20:26+04:00
draft: false
toc: false
images:
tags:
  - docker
  - static image analysis
  - vulnerabilities
---




### Introduction
[Klar](https://github.com/optiopay/klar) is a static binary tool to analyze images stored in a private or public Docker registry for security vulnerabilities using [Clair](https://github.com/coreos/clair). Klar is designed to be used as an integration tool so it relies on environment variables. It's a single binary which requires no dependencies and can be plugged and/or integrated into our CI CD pipelines.

Klar serves as a client which coordinates the image checks between the Docker registry and Clair. We heavily use klar along with Clair in our internal CI CD pipeline to scan the newly built Docker images.

---
#### Installation
Download the latest release (for OSX and Linux) from https://github.com/optiopay/klar/releases/ and put the binary in a directory which is present in your PATH variable (make sure it has execute permission set).

---
#### Usage
Klar process returns 0 if the number of detected high-severity vulnerabilities in an image is less than or equal to a threshold (see below) and 1 if there were more. It will return 2 if an error has prevented the image from being analyzed.

Klar can be configured via the following environment variables:


| Variable | Description |
|------|------|
|**CLAIR_ADDR** | address of Clair server. It has a form of protocol://host:port - protocol and port default to http and 6060 respectively and may be omitted. You can also specify basic authentication in the URL: protocol://login:password@host:port |
|**CLAIR_OUTPUT** | severity level threshold, vulnerabilities with severity level higher than or equal to this threshold will be outputted. Supported levels are Unknown, Negligible, Low, Medium, High, Critical, Defcon1. Default is Unknown |
|**CLAIR_THRESHOLD** | how many outputted vulnerabilities Klar can tolerate before returning 1. Default is 0 |
|**CLAIR_TIMEOUT** | timeout in minutes before Klar cancels the image scanning. Default is 1 |
|**DOCKER_USER** | Docker registry account name |
|**DOCKER_PASSWORD** | Docker registry account password |
|**DOCKER_TOKEN** | Docker registry account token. (Can be used in place of DOCKER_USER and DOCKER_PASSWORD) |
|**DOCKER_INSECURE** | Allow Klar to access registries with bad SSL certificates. Default is false. Clair will need to be booted with -insecure-tls for this to work |
|**DOCKER_TIMEOUT** | timeout in minutes when trying to fetch layers from a docker registry |
|**DOCKER_PLATFORM_OS** | The operating system of the Docker image. Default is linux. This only needs to be set if the image specified references a Docker ManifestList instead of a usual manifest |
|**DOCKER_PLATFORM_ARCH** | The architecture the Docker image is optimized for. Default is amd64. This only needs to be set if the image specified references a Docker ManifestList instead of a usual manifest |
|**REGISTRY_INSECURE** | Allow Klar to access insecure registries (HTTP only). Default is false |
|**JSON_OUTPUT** | Output JSON, not plain text. Default is false |
|**FORMAT_OUTPUT** | Output format of the vulnerabilities. Supported formats are standard, json, table. Default is standard. If JSON_OUTPUT is set to true, this option is ignored |
|**WHITELIST_FILE** | Path to the YAML file with the CVE whitelist. Look at whitelist-example.yaml for the file format|
|**IGNORE_UNFIXED** | Do not count vulnerabilities without a fix towards the threshold |

#### Usage:

```
CLAIR_ADDR=localhost;CLAIR_OUTPUT=High;CLAIR_THRESHOLD=10;DOCKER_USER=docker;DOCKER_PASSWORD=secret;
klar mysql:latest
```
##### Debug Output
You can enable more verbose output by setting KLAR_TRACE to true.

run `export KLAR_TRACE=true` to persist between runs.
---
#### GitLab CI Usage
We have the below job defined in `.gitlab-ci.yml` post Dockerbuild stage

```
image_analysis:
  stage: analyse
  image: registry.mycompany.com/ci/kubernetes-deploy:klar
  script:
    - export PATH=$PATH:$CI_PROJECT_DIR
    - export TAG="$CI_BUILD_REF_NAME"
    - CLAIR_THRESHOLD=1000 DOCKER_TIMEOUT=5 CLAIR_TIMEOUT=5 CLAIR_ADDR=https://ngclair.mycompany.com:443 DOCKER_USER=gitlab-ci-token DOCKER_PASSWORD=$CI_BUILD_TOKEN klar $CI_REGISTRY_IMAGE:$TAG | tee scan.txt
  artifacts:
    paths:
      - $CI_PROJECT_DIR/scan.txt
    expire_in: 1 hour
  tags:
    - my_runner

```

This image `registry.mycompany.com/ci/kubernetes-deploy:klar` contains the klar binary and the clair server is running on https://ngclair.mycompany.com:443
