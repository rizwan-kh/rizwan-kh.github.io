---
title: "Trivy - Scan Container Images"
date: 2021-12-14T23:10:21+04:00
draft: false
toc: false
images:
tags:
  - container
  - static image analysis
  - vulnerabilities
---
![trivy](/trivy.png)
## Trivy 
[Trivy](https://github.com/aquasecurity/trivy) is a scanner for vulnerabilities in container images, file systems, git repositories and configuration. It's an [Aqua Security](https://aquasec.com/) open-source project that can be easily used to integrate with our existing CI CD pipeline or used as a stand-alone tool to scan container images deployed on the Kubernetes cluster. 

We were using [Clair](https://github.com/coreos/clair) previous to our switch to Trivy as we analyzed both the tools and found Trivy to be more helpful for the longer run - it was faster, had more CVE database updates as compared to Clair and didn't depend on external clients to interact/scan.

## Usage
For stand-alone usage, simply [download the binary](https://github.com/aquasecurity/trivy/releases) on your Mac/Linux system and scan images or file systems or configuration as below

```
# for scanning images
trivy image [image-name]
trivy image alpine:latest

# for scanning file system
trivy fs --security-checks vuln,config [YOUR_PROJECT_DIR]
trivy fs --security-checks vuln,config my_project/

# for scanning configuration
trivy config [YOUR_IAC_DIR]
trivy config terraform/src

``` 

Trivy is a [feature-rich tool](https://github.com/aquasecurity/trivy#features) and once you start using the same, you will recommend it to everyone who isn't using it due to its simplicity.

## Use-case
We scan the images in our build CI pipeline, but once deployed on the Kubernetes cluster, we were trying to find a tool which would scan the deployed images, we found Trivy can be used as an alternative if you don't have run-time scanners like Palo Alto Twistlock Defender which does more than just scanning.

We were re-scanning the images from our registry, but only the ones which are already running on our cluster.

We get the list of images using the below `kubectl` API call and then iterate over those images to generate either HTML or JSON format reports as per the need.

```
# to get the list of images running on the cluster
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" |tr -s '[[:space:]]' '\n' |sort |uniq -c

# then we use the output list from the above command to iterate over each image to get the desired output
# to get only critical vulnerabilities on library layer
trivy image -s CRITICAL --vuln-type library --ignore-unfixed --format json -o my-app-1.4.3.json registry.mycompany.com/containers/my-app:1.4.3

# to get all vulnerabilities in HTML format
trivy image --format template "@contrib/template.html" -o my-app-1.4.3.html registry.mycompany.com/containers/my-app:1.4.3
```