---
title: "Create traffic routes"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

* Add a retry policy so that if a request to crystal-sd-green resulted in any server error status codes as “500 or 503”, a retry attempt would be made with a max limit of 10 retries.

```bash
# Define variables #
SPEC=$(cat <<-EOF
    { 
      "httpRoute": {
        "action": { 
          "weightedTargets": [
            {
              "virtualNode": "crystal-sd-green",
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
        },
        "retryPolicy": {
          "maxRetries": 10,
          "perRetryTimeout": {
            "unit": "s",
            "value": 2
          },
          "httpRetryEvents": [
            "server-error"
          ]
        }        
      },
      "priority": 1
    }
EOF
); \
# Create app mesh route #
aws appmesh update-route \
      --mesh-name appmesh-workshop \
      --virtual-router-name crystal-router \
      --route-name crystal-header-route \
      --spec "$SPEC"
```