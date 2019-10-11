---
title: "Create a virtual router"
date: 2018-09-18T17:39:30-05:00
weight: 40
---

Virtual routers handle traffic for one or more virtual services within your mesh. 
We will create a virtual router and associate routes to direct incoming requests to the different virtual node destinations we have for the Crystal backend.

* Begin by creating the virtual router.

```bash
# Define variables #
SPEC=$(cat <<-EOF
    { 
      "listeners": [
        {
          "portMapping": { "port": 3000, "protocol": "http" }
        }
      ]
    }
EOF
); \
# Create app mesh virtual router #
aws appmesh create-virtual-router \
      --mesh-name appmesh-workshop \
      --virtual-router-name crystal-router \
      --spec "$SPEC"
```

* Create a route to direct every incomming request to crystal-srv-v1 (weight: 1).

```bash
# Define variables #
SPEC=$(cat <<-EOF
    { 
      "httpRoute": {
        "action": { 
          "weightedTargets": [
            {
              "virtualNode": "crystal-alb-v1",
              "weight": 0
            },
            {
              "virtualNode": "crystal-srv-v1",
              "weight": 1
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
aws appmesh create-route \
      --mesh-name appmesh-workshop \
      --virtual-router-name crystal-router \
      --route-name crystal-default-route \
      --spec "$SPEC"
```