---
title: "(Linux) Remove systemd process"
date: 2018-03-15T00:28:21+04:00
draft: false
toc: false
images:
tags:
  - linux
  - systemd
---
# Removing a systemd process from your Linux machine

The below steps are to be performed to remove a systemd process

## View the service unit file
```sh
systemctl cat [servicename]
```

## Stop and disable the service 
```sh
systemctl stop [servicename]
systemctl disable [servicename]
```

## Remove the service file
```sh
rm /etc/systemd/system/[servicename]
rm /etc/systemd/system/[servicename] #symlinks that might be related
```

## Reload systemd configuration
```sh
systemctl daemon-reload
```

## Reset the failed systemd service
```sh
systemctl reset-failed
```