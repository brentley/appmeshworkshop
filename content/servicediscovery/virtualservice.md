---
title: "Update the virtual service"
date: 2018-09-18T17:39:30-05:00
weight: 50
---

* Update the virtual service to use the virtual router provider you just created.

```bash
# Define variables #
SPEC=$(cat <<-EOF
    { 
      "provider": {
        "virtualRouter": { 
          "virtualRouterName": "crystal-router"
        }
      }
    }
EOF
); \
# Create app mesh virtual service #
aws appmesh update-virtual-service \
      --mesh-name appmesh-workshop \
      --virtual-service-name crystal.appmeshworkshop.hosted.local \
      --spec "$SPEC"
```

Finally, after confirming traffic is routing succesfully, we can shift 100% of our traffic over to the new virtual node. This also means deprovisioning the existing internal load balancer, the ECS service and the tasks running behind it. The CNAME record crystal.appmeshworkshop.hosted.local can now reference crystal.appmeshworkshop.pvt.local.