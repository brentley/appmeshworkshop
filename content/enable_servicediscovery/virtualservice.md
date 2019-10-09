---
title: "Update the virtual service"
date: 2018-09-18T17:39:30-05:00
weight: 20
---

* Update the virtual service to use the virtual router provider you just created.

```bash
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
aws appmesh update-virtual-service \
      --mesh-name appmesh-workshop \
      --virtual-service-name crystal.appmeshworkshop.hosted.local \
      --spec "$SPEC"
```