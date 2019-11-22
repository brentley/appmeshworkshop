---
title: "Update the virtual service"
date: 2018-09-18T17:39:30-05:00
weight: 40
---
Virtual Services require a provider and you can specify either a Virtual Node or a Virtual Router as the provider of any given Virtual Service. The main difference between the two is that Virtual Routers allow to split traffic among multiple endpoints. If you plan to support A/B testing or Canary releases of new micro services version, then go with Virtual Routers. So far we have been using a Virtual node as the provider of the Crystal backend. Now we will switch to a Virtual Router as we want to gradually move traffic over the just created ECS service. Should things go wrong, can can simply adjust the traffic shaping percentages in our routes to move 100% of the incoming traffic to the old service version.


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
