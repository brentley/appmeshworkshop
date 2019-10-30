---
title: "Create a new virtual node"
date: 2018-09-18T17:39:30-05:00
weight: 20
---

* Create a second virtual node for the Crystal backend, and declare Cloud Map as the service discovery mechanism (instead of DNS).

```bash
# Define variables #
SPEC=$(cat <<-EOF
  { 
    "serviceDiscovery": {
      "awsCloudMap": {
        "namespaceName": "appmeshworkshop.pvt.local",
        "serviceName": "crystal",
        "attributes": [
          {
            "key": "ECS_TASK_SET_EXTERNAL_ID",
            "value": "vanilla-task-set"
          }
        ]
      }
    },
    "logging": {
      "accessLog": {
        "file": {
          "path": "/dev/stdout"
        }
      }
    },      
    "listeners": [
      {
        "healthCheck": {
          "healthyThreshold": 3,
          "intervalMillis": 10000,
          "path": "/health",
          "port": 3000,
          "protocol": "http",
          "timeoutMillis": 5000,
          "unhealthyThreshold": 3
        },
        "portMapping": { "port": 3000, "protocol": "http" }
      }
    ]
  }
EOF
); \
# Create app mesh virtual node #
aws appmesh create-virtual-node \
  --mesh-name appmesh-workshop \
  --virtual-node-name crystal-sd-vanilla \
  --spec "$SPEC"
```
