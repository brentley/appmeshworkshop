---
title: "Create traffic routes"
date: 2018-09-18T17:39:30-05:00
weight: 40
---

We will  try sending 30% of the traffic over to our canary. If our monitoring indicates that the service is healthy, we could start granualy increasing the load with automated rollouts (and rollback if issues are indicated), but we're keeping things simple for the workshop.

* Start shifting traffic to your canary virtual node. Traffic will be distributed between the crystal-sd-blue and crystal-sd-greens virtual nodes at a 2:1 ratio respectively.

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
  --route-name crystal-random-route \
  --spec "$SPEC"
```