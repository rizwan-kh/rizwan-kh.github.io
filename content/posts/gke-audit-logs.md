---
title: "GKE Audit logs - Find the cluster resources audit logs"
date: 2023-28-02T05:40:00+04:00
draft: false
toc: false
images:
tags:
  - gke
  - autopilot
  - gcp
---
## Introduction
In a scenario where you need to validate the creator of a specific Google Kubernetes Engine (GKE) cluster, especially when there is no clear ownership or standard naming conventions, leveraging the Google Logging Console can provide valuable insights. This guide aims to demonstrate how you can utilize Google Cloud's Logging Console to identify the creator of a GKE cluster.


## Commands
The following command is designed to be executed in the Cloud Logging console. Ensure to replace the placeholder with the actual cluster name within the quotes in the last line.

### Cluster Created
```
resource.type="gke_cluster"
(log_id("cloudaudit.googleapis.com/data_access") OR
log_id("cloudaudit.googleapis.com/activity"))
protoPayload.methodName:"CreateCluster"
resource.labels.cluster_name="autopilot-cluster-1"
```
## Additional Examples
### Cluster Deleted
To identify who deleted a GKE cluster:
```
resource.type="gke_cluster"
(log_id("cloudaudit.googleapis.com/data_access") OR
log_id("cloudaudit.googleapis.com/activity"))
protoPayload.methodName:"DeleteCluster"
resource.labels.cluster_name="your-cluster-name"
```

### Node Pool Created
To find out who created a specific GKE node pool:
```
resource.type="gke_nodepool"
(log_id("cloudaudit.googleapis.com/data_access") OR
log_id("cloudaudit.googleapis.com/activity"))
protoPayload.methodName:"CreateNodePool"
resource.labels.node_pool_name="your-node-pool-name"
```

These commands can be adapted based on your specific use case, allowing you to gain insights into various GKE cluster activities through the Google Logging Console.