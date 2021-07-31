---
title: "Vagrant"
date: 2020-03-14T17:20:26+04:00
draft: false
toc: false
images:
tags:
  - hashicorp
  - vm
---

## Introduction

---
## Getting started with Vagrant
Download the install the [Vagrant](https://www.vagrantup.com/downloads) from the Download section

```
vagrant version
Installed Version: 2.2.10
Latest Version: 2.2.10

You're running an up-to-date version of Vagrant!
```

Download and install [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

Now if both **Vagrant** and **VirtualBox** are up and running, all we need to do is run the below 3 commands to have a Ubuntu VM up and available for our use in no time.
```
vagrant init hashicorp/bionic64
vagrant up
vagrant ssh
```
This is a pretty simple way to spin up a new Ubuntu VM, we can do a lot more which I will try to write up in later posts.
