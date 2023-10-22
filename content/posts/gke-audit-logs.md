title: "GKE Audit logs - Find the cluster resources audit logs"
date: 2023-28-02T05:40:00+04:00
draft: true
toc: false
images:
tags:
  - gke
  - autopilot
  - gcp
---
## Introduction



## Commands
The below command has to be run in the Cloud Logging console, please don't forget to replace the cluster name within the quotes in the last line

### Cluster Created
```
resource.type="gke_cluster"
(log_id("cloudaudit.googleapis.com/data_access") OR
log_id("cloudaudit.googleapis.com/activity"))
protoPayload.methodName:"CreateCluster"
resource.labels.cluster_name="autopilot-cluster-1"
```
