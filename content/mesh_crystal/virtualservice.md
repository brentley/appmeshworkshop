---
title: "Create the virtual service"
date: 2018-09-18T17:39:30-05:00
weight: 10
---

In the previous section, we created the service mesh. Lets now add a representation of the Crystal ECS Service. We will do so by creating the following two artifacts in App Mesh, a virtual node and a virtual service.

A **virtual service** is an abstraction of a real service that is provided by a **virtual node** in the mesh. Virtual nodes act as a logical pointer to a particular task group, such as an EC2 Auto Scaling Group, an Amazon ECS service or a Kubernetes deployment. Remember our Crystal service is running as a Amazon ECS (Fargate) service.

Virtual nodes have friendly names. We will name this virtual node **crystal-lb-vanilla**. Inside a Virtual nodes  you will also define the service discovery mechanism for your service. It support both DNS and Cloud Map based service discovery. We will start by using DNS and most specifically we will leverage the DNS name given to the internal ALB that is already fronting our ECS Service.

The Crystal service is running as a service on ECS Fargate and is reachable via an internal Application Load Balancer. Lets use the DNS name given to it by AWS as part of the CloudFormation  template run in section Start the Workshop.

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
  --virtual-node-name crystal-lb-vanilla \
  --spec "$SPEC"
```

Now that we have our virtual node in place, we are ready to create the virtual service.

We will supply two important pieces of information to the definition of the virtual service. First, we will select the virtual node named **crystal-lb-vanilla** created above as the provider of the virtual service. Second, we will give it a service name. The name of a service is a FQDN and is the name used by clients interested in contacting the service. In our example, the Ruby Frontend will issue HTTP requests to  **crystal.appmeshworkshop.hosted.local** in order to interact with the Crystal service. 

In Route53, we have already created a private hosted zone named **appmeshworkshop.hosted.local** and inside, an A ALIAS record named **crystal** with value the FQDN of the internal ALB.

* Create the virtual service.

```bash
# Define variables #
SPEC=$(cat <<-EOF
  { 
    "provider": {
      "virtualNode": { 
        "virtualNodeName": "crystal-lb-vanilla"
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
