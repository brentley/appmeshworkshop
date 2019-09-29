---
title: "Enable service discovery"
date: 2018-09-18T17:39:30-05:00
weight: 30
---

A virtual service is an abstraction of a real service that is provided by a virtual node in the mesh. Virtual nodes act as a logical pointer to a particular task group, such as an EC2 Auto Scaling Group, an Amazon ECS service or a Kubernetes deployment.

* Start by creating the virtual node

```bash
SPEC=$(cat <<-EOF
    { 
      "serviceDiscovery": {
        "awsCloudMap": { 
          "namespaceName": "appmeshworkshop.hosted.local",
          "serviceName": "crystal"
        }
      },
      "logging": {
        "accessLog": {
          "file": {
            "path": "/dev/stdout"
          }
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
      --virtual-node-name crystal-v1 \
      --spec "$SPEC"
```

* Now, we are ready to create the virtual service

```bash
SPEC=$(cat <<-EOF
    { 
      "provider": {
        "virtualNode": { 
          "virtualNodeName": "crystal-v1"
        }
      }
    }
EOF
); \
aws appmesh create-virtual-service \
      --mesh-name AppMesh-Workshop \
      --virtual-service-name crystal.appmeshworkshop.hosted.local \
      --spec "$SPEC"
```