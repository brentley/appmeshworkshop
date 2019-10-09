---
title: "Create a new crystal service"
date: 2018-09-18T17:39:30-05:00
weight: 30
---

* Register a new task definition pointing to the crystal-srv-v1 virtual node

```bash
TASK_DEF_ARN=$(aws ecs list-task-definitions | jq --raw-output ' .taskDefinitionArns | last');
TASK_DEF_OLD=$(aws ecs describe-task-definition --task-definition $TASK_DEF_ARN);
TASK_DEF_NEW=$(echo $TASK_DEF_OLD \
  | jq ' .taskDefinition' \
  | jq ' .containerDefinitions[].environment |= map(
        if .name=="APPMESH_VIRTUAL_NODE_NAME" then 
           .value="mesh/appmesh-workshop/virtualNode/crystal-srv-v1" 
        else . end) ' \
  | jq ' del( .status, .compatibilities, .taskDefinitionArn, .requiresAttributes, .revision) '
); \
TASK_DEF_FAMILY=$(echo $TASK_DEF_ARN | cut -d"/" -f2 | cut -d":" -f1);
echo $TASK_DEF_NEW > /tmp/$TASK_DEF_FAMILY.json && 
aws ecs register-task-definition \
      --cli-input-json file:///tmp/$TASK_DEF_FAMILY.json
```

* Create a new service in ECS. This time use a service registry to register task instances.

```bash
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
TASK_DEF_ARN=$(aws ecs list-task-definitions | jq --raw-output ' .taskDefinitionArns | last');
PRIVSUB1=$(jq < cfn-output.json -r '.PrivateSubnetOne');
PRIVSUB2=$(jq < cfn-output.json -r '.PrivateSubnetTwo');
PRIVSUB3=$(jq < cfn-output.json -r '.PrivateSubnetThree');
SECGROUP=$(jq < cfn-output.json -r '.ContainerSecurityGroup');
CMAP_SVC_ARN=$(aws servicediscovery list-services \
      | jq --raw-output '.Services[] | select(.Name == "crystal") | .Arn');
aws ecs create-service \
      --cluster $CLUSTER_NAME \
      --service-name CrystalService-SRV \
      --task-definition "$(echo $TASK_DEF_ARN)" \
      --desired-count 3 \
      --platform-version LATEST \
      --service-registries "registryArn=$CMAP_SVC_ARN,port=3000" \
      --launch-type FARGATE \
      --network-configuration \
          "awsvpcConfiguration={subnets=[$PRIVSUB1,$PRIVSUB2,$PRIVSUB3],
            securityGroups=[$SECGROUP],assignPublicIp=DISABLED}"
```