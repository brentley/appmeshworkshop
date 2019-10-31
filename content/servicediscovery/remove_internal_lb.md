---
title: "Remove internal load balancer"
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
            "virtualNode": "crystal-sd-vanilla",
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
# Delete ecs service
aws ecs delete-service \
--cluster $CLUSTER_NAME \
--service crystal-service-lb \
--force
# Delete load lalancer
aws elbv2 delete-load-balancer \
--load-balancer-arn $INTERNAL_LB_ARN
```

* Update the CNAME record in Route53 to crystal.appmeshworkshop.pvt.local

```bash
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name \
    --dns-name appmeshworkshop.hosted.local \
    --max-items 1 | \
  jq -r ' .HostedZones | first | .Id');
RECORD_SET=$(aws route53 list-resource-record-sets --hosted-zone-id=$HOSTED_ZONE_ID | \
 jq -r '.ResourceRecordSets[] | select (.Name == "crystal.appmeshworkshop.hosted.local.")');
cat <<-EOF > /tmp/update_r53.json
{
  "Comment": "UPDATE crystal.appmeshworkshop.hosted.local",
  "Changes": [
    {
      "Action": "DELETE",
      "ResourceRecordSet": $RECORD_SET
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "crystal.appmeshworkshop.hosted.local",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [
          { "Value": "crystal.appmeshworkshop.pvt.local." }
        ]
      }
    }
  ]
}
EOF
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch file:///tmp/update_r53.json
```

You can now go ahead and test your application again to make sure everything is still working.

