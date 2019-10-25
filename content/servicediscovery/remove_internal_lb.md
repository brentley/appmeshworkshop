---
title: "Remove internal Load Balancer"
date: 2018-09-18T17:39:30-05:00
weight: 70
---

Now that we could see that the Crystal backend service is working without using the Load Balancer, we can shift 100% of the traffic to it and delete the old service and the internal LB.

To do so, let's start shifting 100% of the traffic to the virtual node with the Cloud Map integration:

```bash
# Define variables #
SPEC=$(cat <<-EOF
  { 
    "httpRoute": {
      "action": { 
        "weightedTargets": [
          {
            "virtualNode": "crystal-sd-blue",
            "weight": 1
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
# Update app mesh route #
aws appmesh update-route \
  --mesh-name appmesh-workshop \
  --virtual-router-name crystal-router \
  --route-name crystal-traffic-route \
  --spec "$SPEC"
```

After redirecting all the traffic to the new virtual node with the service discovery attributes, let's delete the old ECS service and the internal Load Balancer:

```bash
# Define variables
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
INTERNAL_LB_ARN=$(jq < cfn-output.json -r '.InternalLoadBalancerArn');

# Delete ECS Service
aws ecs delete-service \
--cluster $CLUSTER_NAME \
--service crystal-service-lb-blue \
--force

# Delete Load Balancer
aws elbv2 delete-load-balancer \
--load-balancer-arn $INTERNAL_LB_ARN
```

You can now go ahead and test your application again to make sure everything is still working.

