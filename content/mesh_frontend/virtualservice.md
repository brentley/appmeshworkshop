---
title: "Create the virtual service"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

A virtual service is an abstraction of a real service that is provided by a virtual node in the mesh. Virtual nodes act as a logical pointer to a particular task group, such as an EC2 Auto Scaling Group, an Amazon ECS service or a Kubernetes deployment.

* Start by creating the virtual node

```bash
EXT_LOAD_BALANCER=$(jq < cfn-output.json -r '.ExternalLoadBalancerDNS');
SPEC=$(cat <<-EOF
    { 
      "serviceDiscovery": {
        "dns": { 
          "hostname": "$EXT_LOAD_BALANCER"
        }
      },
      "backends": [
        {
          "virtualService": {
            "virtualServiceName": "crystal.appmeshworkshop.hosted.local"
          }
        }
      ],      
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
      --mesh-name appmesh-workshop \
      --virtual-node-name frontend-v1 \
      --spec "$SPEC"
```

* Now, we are ready to create the virtual service

```bash
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
      --mesh-name appmesh-workshop \
      --virtual-service-name frontend.appmeshworkshop.hosted.local \
      --spec "$SPEC"
```