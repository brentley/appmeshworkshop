---
title: "Create the virtual service"
date: 2018-09-18T17:39:30-05:00
weight: 20
---

A virtual service is an abstraction of a real service that is provided by a virtual node in the mesh. Virtual nodes act as a logical pointer to a particular task group, such as an EC2 Auto Scaling Group, an Amazon ECS service or a Kubernetes deployment.

* Let’s start creating the virtual node

```
SPEC=$(cat <<-EOF
    { 
      "serviceDiscovery": {
        "awsCloudMap": { 
          "attributes": [ { "key": "VERSION", "value": "1" } ],
          "namespaceName": "appmeshworkshop.service",
          "serviceName": "frontend"
        }
      },
      "listeners": [
        {
          "portMapping": { "port": 3000, "protocol": "http" }
        }
      ]
    }
EOF
); \
aws appmesh create-virtual-node \
      --mesh-name AppMesh-Workshop \
      --virtual-node-name frontend-v1 \
      --spec "$SPEC"
```

* Now, we are ready to create the virtual service

```
SPEC=$(cat <<-EOF
    { 
      "provider": {
        "virtualNode": { 
          "virtualNodeName": "frontend-v1"
        }
      }
    }
EOF
); \
aws appmesh create-virtual-service \
      --mesh-name AppMesh-Workshop \
      --virtual-service-name frontend.appmeshworkshop.service \
      --spec "$SPEC"
```