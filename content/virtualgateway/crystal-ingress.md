---
title: "Virtual Gateway Ingress to Crystal service"
date: 2018-09-18T17:39:30-05:00
weight: 20
---

As we can see during the workshop, the AWS App Mesh services enable you to add services to the mesh, whether they are running on EKS, ECS or EC2. So, what we are going to do now is to add the Crystal service, which is running on the ECS cluster, enabling access from outside the mesh to both App Mesh services inside de cluster.

We just need to add a gateway route to the Crystal service running en ECS.


```bash
VIRTUAL_GATEWAY=$(aws appmesh list-virtual-gateways --mesh-name appmesh-workshop | jq -r ".virtualGateways[].virtualGatewayName")
VIRTUAL_SERVICE=$(aws appmesh list-virtual-services --mesh-name appmesh-workshop | jq -r ' .virtualServices[] | select( .virtualServiceName | contains("crystal")).virtualServiceName') 

# Create gatweay route spec json
cat <<-EOF > ~/environment/eks-scripts/gateway-route-ecs-spec.json
{
  "httpRoute": {
    "action": {
      "target": {
        "virtualService": {
          "virtualServiceName": "$VIRTUAL_SERVICE"
        }
      }
    },
    "match": {
      "prefix": "/ecs"
    }
  }
}
EOF

aws appmesh create-gateway-route --gateway-route-name gateway-route-ecs --mesh-name appmesh-workshop --spec file://~/environment/eks-scripts/gateway-route-ecs-spec.json --virtual-gateway-name $VIRTUAL_GATEWAY
```

Let’s test our new route.

Connect to the EC2 instance outside the mesh in order to test reachability from external requests to the mesh using the virtual gateway ingress
