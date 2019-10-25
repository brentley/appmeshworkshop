---
title: "Update traffic route"
date: 2018-09-18T17:39:30-05:00
weight: 5
---

* We are ready to start shifting traffic to crystal-sd-blue. We only need to update the weight assigned to the crystal-sd-virtual node (from 0 to 2).

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
aws appmesh update-route \
  --mesh-name appmesh-workshop \
  --virtual-router-name crystal-router \
  --route-name crystal-traffic-route \
  --spec "$SPEC"
```

After confirming traffic is routing succesfully, we can shift 100% of our traffic over to the new virtual node. This also means deprovisioning the existing internal load balancer, the ECS service and the tasks running behind it. The CNAME record crystal.appmeshworkshop.hosted.local can now reference crystal.appmeshworkshop.pvt.local.