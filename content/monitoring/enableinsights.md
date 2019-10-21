---
title: "Enable CloudWatch Insights"
date: 2018-09-18T16:01:14-05:00
weight: 5
---

* Enable container insights.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
# Update ecs cluster settings #
aws ecs update-cluster-settings \
  --cluster $CLUSTER_NAME \
  --settings name=containerInsights,value=enabled
```