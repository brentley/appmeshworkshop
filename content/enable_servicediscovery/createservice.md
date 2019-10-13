---
title: "Create a new crystal service"
date: 2018-09-18T17:39:30-05:00
weight: 30
---

* Register a new task definition pointing to the crystal-sd-v1 virtual node

```bash
# Define variables #
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
      jq -r ' .taskDefinitionArns[] | select( . | contains("Crystal"))' | tail -1)
TASK_DEF_OLD=$(aws ecs describe-task-definition --task-definition $TASK_DEF_ARN);
TASK_DEF_NEW=$(echo $TASK_DEF_OLD \
      | jq ' .taskDefinition' \
      | jq ' .containerDefinitions[].environment |= map(
            if .name=="APPMESH_VIRTUAL_NODE_NAME" then 
                  .value="mesh/appmesh-workshop/virtualNode/crystal-sd-v1" 
            else . end) ' \
      | jq ' del(.status, .compatibilities, .taskDefinitionArn, .requiresAttributes, .revision) '
); \
TASK_DEF_FAMILY=$(echo $TASK_DEF_ARN | cut -d"/" -f2 | cut -d":" -f1);
echo $TASK_DEF_NEW > /tmp/$TASK_DEF_FAMILY.json && 
# Register ecs task definition #
aws ecs register-task-definition \
      --cli-input-json file:///tmp/$TASK_DEF_FAMILY.json
```

* Create a new service in ECS. This time use a service registry to register task instances.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
      jq -r ' .taskDefinitionArns[] | select( . | contains("Crystal"))' | tail -1)
SUBNET_ONE=$(jq < cfn-output.json -r '.PrivateSubnetOne');
SUBNET_TWO=$(jq < cfn-output.json -r '.PrivateSubnetTwo');
SUBNET_THREE=$(jq < cfn-output.json -r '.PrivateSubnetThree');
SECURITY_GROUP=$(jq < cfn-output.json -r '.ContainerSecurityGroup');
CMAP_SVC_ARN=$(aws servicediscovery list-services | \
      jq -r '.Services[] | select(.Name == "crystal") | .Arn');
# Create ecs service #
aws ecs create-service \
      --cluster $CLUSTER_NAME \
      --service-name crystal-service-sd-v1 \
      --task-definition "$(echo $TASK_DEF_ARN)" \
      --desired-count 3 \
      --platform-version LATEST \
      --service-registries "registryArn=$CMAP_SVC_ARN" \
      --launch-type FARGATE \
      --network-configuration \
          "awsvpcConfiguration={subnets=[$SUBNET_ONE,$SUBNET_TWO,$SUBNET_THREE],
            securityGroups=[$SECURITY_GROUP],
            assignPublicIp=DISABLED}"
```