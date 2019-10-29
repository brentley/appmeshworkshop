---
title: "Update traffic route"
date: 2018-09-18T17:39:30-05:00
weight: 60
---

* We are ready to start shifting traffic to crystal-sd-vanilla. We only need to update the weight assigned to the crystal-sd-virtual node (from 0 to 2).

```bash
# Define variables #
SPEC=$(cat <<-EOF
  { 
    "httpRoute": {
      "action": { 
        "weightedTargets": [
          {
            "virtualNode": "crystal-lb-vanilla",
            "weight": 1
          },
          {
            "virtualNode": "crystal-sd-vanilla",
            "weight": 2
          }
        ]
      },
      "match": {
        "prefix": "/"
      }
    },
    "priority": 10
  }
EOF
); \
# Create app mesh route #
aws appmesh update-route \
  --mesh-name appmesh-workshop \
  --virtual-router-name crystal-router \
  --route-name crystal-traffic-route \
  --spec "$SPEC"
```