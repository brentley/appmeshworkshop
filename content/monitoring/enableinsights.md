---
title: "Enable CloudWatch Insights"
date: 2018-09-18T16:01:14-05:00
weight: 5
---

CloudWatch Container Insights is a fully managed service that collects, aggregates, and summarizes Amazon ECS metrics and logs. The CloudWatch Container Insights dashboard gives you access to the following information:

* CPU and memory utilization
* Task and service counts
* Read/write storage
* Network Rx/Tx
* Container instance counts for clusters, services, and tasks

* Enable container insights.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
# Update ecs cluster settings #
aws ecs update-cluster-settings \
  --cluster $CLUSTER_NAME \
  --settings name=containerInsights,value=enabled
```

Read and follow the instructions provided at [https://aws.amazon.com/blogs/mt/introducing-container-insights-for-amazon-ecs/](https://aws.amazon.com/blogs/mt/introducing-container-insights-for-amazon-ecs/) to access metrics collected by CloudWatch Container Insights.
