---
title: "Enable X-Ray for NodeJS App"
date: 2018-09-18T16:01:14-05:00
weight: 11
---

Let's now enable the X-Ray integration to the NodeJS app currently running on the EKS cluster. 

The first step is to 


AUTO_SCALING_GROUP=$(jq < cfn-output.json -r '.RubyAutoScalingGroupName');
INSTANCE_PROFILE_NAME=$(aws ec2 describe-instances --filters Name=tag:aws:autoscaling:groupName,Values=$AUTO_SCALING_GROUP | jq -r ' .Reservations | first | .Instances | first | .IamInstanceProfile | .Arn' | cut -d '/' -f 2)
EKS_WORKERS_ROLE=$(aws iam get-instance-profile --instance-profile-name $INSTANCE_PROFILE_NAME | jq -r '.InstanceProfile.Roles[].RoleName')



```bash
helm upgrade -i appmesh-inject eks/appmesh-inject \
--namespace appmesh-system \
--set mesh.create=false \
--set mesh.name=appmesh-workshop \
--set tracing.enabled=true \
--set tracing.provider=x-ray
```