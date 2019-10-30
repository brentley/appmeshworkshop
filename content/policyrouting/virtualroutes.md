---
title: "Create traffic routes"
date: 2018-09-18T17:39:30-05:00
weight: 5
---

* Instead of randomly distributing traffic using weights among virtual nodes, direct the traffic to our canary only if the request header canary_fleet equals true.

```bash
# Define variables #
SPEC=$(cat <<-EOF
  { 
    "httpRoute": {
      "action": { 
        "weightedTargets": [
          {
            "virtualNode": "crystal-sd-epoch",
            "weight": 1
          }
        ]
      },
      "match": {
        "prefix": "/",
        "headers": [
          {
            "name": "canary_fleet",
            "match": {
              "exact": "true"
            }
          }
        ]
      }
    },
    "priority": 1
  }
EOF
); \
# Create app mesh route #
aws appmesh create-route \
  --mesh-name appmesh-workshop \
  --virtual-router-name crystal-router \
  --route-name crystal-header-route \
  --spec "$SPEC"
```