---
title: "Create the virtual service"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

A virtual service is an abstraction of a real service that is provided by a virtual node in the mesh. Virtual nodes act as a logical pointer to a particular task group, such as an EC2 Auto Scaling Group, an Amazon ECS service or a Kubernetes deployment.

* Start by creating the virtual node.

```bash
# Define variables #
INT_LOAD_BALANCER=$(jq < cfn-output.json -r '.InternalLoadBalancerDNS');
SPEC=$(cat <<-EOF
    { 
      "serviceDiscovery": {
        "dns": { 
          "hostname": "$INT_LOAD_BALANCER"
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
          "healthCheck": {
            "healthyThreshold": 3,
            "intervalMillis": 10000,
            "path": "/health",
            "port": 3000,
            "protocol": "http",
            "timeoutMillis": 5000,
            "unhealthyThreshold": 3
          },
          "portMapping": { "port": 3000, "protocol": "http" }
        }
      ]
    }
EOF
); \
# Create app mesh virual node #
aws appmesh create-virtual-node \
      --mesh-name appmesh-workshop \
      --virtual-node-name crystal-lb-blue \
      --spec "$SPEC"
```

* Now, you are ready to create the virtual service.

```bash
# Define variables #
SPEC=$(cat <<-EOF
    { 
      "provider": {
        "virtualNode": { 
          "virtualNodeName": "crystal-lb-blue"
        }
      }
    }
EOF
); \
# Create app mesh virtual service #
aws appmesh create-virtual-service \
      --mesh-name appmesh-workshop \
      --virtual-service-name crystal.appmeshworkshop.hosted.local \
      --spec "$SPEC"
```