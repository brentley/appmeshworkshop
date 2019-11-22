---
title: "Remove NodeJS Load Balancer"
date: 2018-09-18T17:39:30-05:00
weight: 90
---

Now that the frontend ec2 instances are talking to the NodeJS backend service using the Cloud Map service discovery integration, we can delete the Load Balancer that is fronting this application. To do so, let's delete the `nodejs-app-service`

```bash
kubectl delete service nodejs-app-service -n appmesh-workshop-ns
```

And now, changing the Route53 hosted zone in order to point it to the new domain name:

```bash
# Define variables
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name \
    --dns-name appmeshworkshop.hosted.local \
    --max-items 1 | \
  jq -r ' .HostedZones | first | .Id');
RECORD_SET=$(aws route53 list-resource-record-sets --hosted-zone-id=$HOSTED_ZONE_ID | \
 jq -r '.ResourceRecordSets[] | select (.Name == "nodejs.appmeshworkshop.hosted.local.")');
cat <<-EOF > /tmp/update_nodejs_r53.json
{
  "Comment": "UPDATE nodejs.appmeshworkshop.hosted.local",
  "Changes": [
    {
      "Action": "DELETE",
      "ResourceRecordSet": $RECORD_SET
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "nodejs.appmeshworkshop.hosted.local",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [
          { "Value": "nodejs.appmeshworkshop.pvt.local." }
        ]
      }
    }
  ]
}
EOF
# Change route53 record set
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch file:///tmp/update_nodejs_r53.json

```

Finally, we need to change the security group in the Kubernetes worker nodes in order to allow traffic from the Frontend instances to the port 3000 of the worker nodes:

```bash
# Set environment variables
WORKER_NODE_DNS=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
WORKER_NODES_SG_ID=$(aws ec2 describe-instances --filter "Name=private-dns-name,Values=$WORKER_NODE_DNS" | \
jq -r .Reservations[].Instances[].SecurityGroups[0].GroupId)
AUTO_SCALING_GROUP=$(jq < cfn-output.json -r '.RubyAutoScalingGroupName');
FRONTEND_SG_ID=$(aws ec2 describe-instances --filters Name=tag:aws:autoscaling:groupName,Values=$AUTO_SCALING_GROUP | \
jq -r '.Reservations[0].Instances[0].NetworkInterfaces[0].Groups[0].GroupId')

# Update kubernetes worker nodes sg
aws ec2 authorize-security-group-ingress --group-id $WORKER_NODES_SG_ID --protocol tcp --port 3000 --source-group $FRONTEND_SG_ID
```



You can confirm that the Load Balancer is not there anymore by acessing the [EC2 console](http://console.aws.amazon.com/ec2/home) and clicking in `Load balancers` on the left side menu. You will notice that at this moment you just have the public load balancer that is fronting your EC2 instances that are serving the Ruby frontend application.