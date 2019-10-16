---
title: "Create traffic routes"
date: 2018-09-18T17:39:30-05:00
weight: 40
---

Initally, we will only send 30% of the traffic over to it. If our monitoring indicates that the service is healthy, we'll increase it to 60%, then finally to 100%. In the real world, you might choose more granular increases with automated rollout (and rollback if issues are indicated), but we're keeping things simple for the workshop.

* Create a route to start shifting traffic to your new virtual node. The traffic will be distributed between the crystal-alb-blue and crystal-sd-blue virtual nodes at a 2:1 ratio (i.e., the crystal-sd-blue node will receive two thirds of the traffic).

```bash
# Define variables #
SPEC=$(cat <<-EOF
    { 
      "httpRoute": {
        "action": { 
          "weightedTargets": [
            {
              "virtualNode": "crystal-sd-blue",
              "weight": 2
            },
            {
              "virtualNode": "crystal-sd-green",
              "weight": 1
            }          
          ]
        },
        "match": {
          "prefix": "/"
        }
      },
      "priority": 5
    }
EOF
); \
# Create app mesh route #
aws appmesh create-route \
      --mesh-name appmesh-workshop \
      --virtual-router-name crystal-router \
      --route-name crystal-canary-route \
      --spec "$SPEC"
```