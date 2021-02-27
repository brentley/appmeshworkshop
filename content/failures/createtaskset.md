---
title: "Create a new ECS task set"
date: 2018-09-18T17:39:30-05:00
weight: 15
---

* Register a new task definition using our error container, and pointing to the crystal-sd-error virtual node.

```bash
# Define variables #
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1)
TASK_DEF_OLD=$(aws ecs describe-task-definition --task-definition $TASK_DEF_ARN);
TASK_DEF_NEW=$(echo $TASK_DEF_OLD \
  | jq ' .taskDefinition' \
  | jq ' .containerDefinitions[0].image |= sub("epoch"; "error") ' \
  | jq ' .containerDefinitions[].environment |= map(
        if .name=="APPMESH_VIRTUAL_NODE_NAME" then 
              .value="mesh/appmesh-workshop/virtualNode/crystal-sd-error" 
        else . end) ' \
  | jq ' del(.status, .compatibilities, .taskDefinitionArn, .requiresAttributes, .revision, .registeredBy, .registeredAt) '
); \
TASK_DEF_FAMILY=$(echo $TASK_DEF_ARN | cut -d"/" -f2 | cut -d":" -f1);
echo $TASK_DEF_NEW > /tmp/$TASK_DEF_FAMILY.json && 
# Register ecs task definition #
aws ecs register-task-definition \
  --cli-input-json file:///tmp/$TASK_DEF_FAMILY.json
```

* Create a new task set.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
SERVICE_ARN=$(aws ecs list-services --cluster $CLUSTER_NAME | \
  jq -r ' .serviceArns[] | select( . | contains("sd"))' | tail -1)
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1)
SUBNET_ONE=$(jq < cfn-output.json -r '.PrivateSubnetOne');
SUBNET_TWO=$(jq < cfn-output.json -r '.PrivateSubnetTwo');
SUBNET_THREE=$(jq < cfn-output.json -r '.PrivateSubnetThree');
SECURITY_GROUP=$(jq < cfn-output.json -r '.ContainerSecurityGroup');
CMAP_SVC_ARN=$(aws servicediscovery list-services | \
  jq -r '.Services[] | select(.Name == "crystal") | .Arn');
# Create ecs task set #
aws ecs create-task-set \
  --service $SERVICE_ARN \
  --cluster $CLUSTER_NAME \
  --external-id error-task-set \
  --task-definition "$(echo $TASK_DEF_ARN)" \
  --service-registries "registryArn=$CMAP_SVC_ARN" \
  --launch-type FARGATE \
  --scale value=50,unit=PERCENT \
  --network-configuration \
      "awsvpcConfiguration={subnets=[$SUBNET_ONE,$SUBNET_TWO,$SUBNET_THREE],
        securityGroups=[$SECURITY_GROUP],
        assignPublicIp=DISABLED}"
```

* Wait for the service tasks to be in a running state and marked healthy for service discovery.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1);
CMAP_SVC_ID=$(aws servicediscovery list-services | \
  jq -r '.Services[] | select(.Name == "crystal") | .Id');
# Get task state #
_list_tasks() {
  aws ecs list-tasks \
    --cluster $CLUSTER_NAME \
    --service crystal-service-sd | \
  jq -r ' .taskArns | @text' | \
    while read taskArns; do
      aws ecs describe-tasks --cluster $CLUSTER_NAME --tasks $taskArns;
    done | \
  jq -r --arg TASK_DEF_ARN $TASK_DEF_ARN \
    ' [.tasks[] | select( (.taskDefinitionArn == $TASK_DEF_ARN)
                    and (.lastStatus == "RUNNING" ))] | length'
}
# Get instances health status #
_list_instances() {
  aws servicediscovery get-instances-health-status \
    --service-id $CMAP_SVC_ID | \
  jq ' [.Status | to_entries[] | select( .value == "HEALTHY")] | length'
}
until [ $(_list_tasks) == "3" ]; do
  echo "Tasks are starting ..."
  sleep 10s
  if [ $(_list_tasks) == "3" ]; then
    echo "Tasks started"
    break
  fi
done
sleep 10s
until [ $(_list_instances) == "9" ]; do
  echo "Instances are registering ..."
  sleep 10s
  if [ $(_list_instances) == "9" ]; then
    echo "Instances registered"
    break
  fi
done
```
