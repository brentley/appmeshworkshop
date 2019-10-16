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

* Create a route to start shifting traffic to your new virtual node. The traffic will be distributed between the crystal-alb-blue and crystal-sd-blue virtual nodes at a 2:1 ratio (i.e., the crystal-sd-blue node will receive two thirds of the traffic).

```bash
# Define variables #
SPEC=$(cat <<-EOF
    { 
      "httpRoute": {
        "action": { 
          "weightedTargets": [
            {
              "virtualNode": "crystal-lb-blue",
              "weight": 1
            },
            {
              "virtualNode": "crystal-sd-blue",
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
aws appmesh create-route \
      --mesh-name appmesh-workshop \
      --virtual-router-name crystal-router \
      --route-name crystal-traffic-route \
      --spec "$SPEC"
```