---
title: "Create a new ECS task set"
date: 2018-09-18T17:39:30-05:00
weight: 15
---

To get everything working, and deploy the updated container images, we need to create a new task set within the existing `crystal-service-sd` service ECS.

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
  jq -r '.Services[] | select(.Name == "crystal-green") | .Arn');
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

* Update the `crystal-service-sd` and scale it to 0%.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
SERVICE_ARN=$(aws ecs list-services --cluster $CLUSTER_NAME | \
  jq -r ' .serviceArns[] | select( . | contains("sd"))' | tail -1)
TASK_SET_ARN=$(aws ecs describe-task-sets --cluster $CLUSTER_NAME | \
  jq -r ' .serviceArns[] | select( . | contains("sd"))' | tail -1)
# Update ecs task set #
aws ecs update-task-set \
  --service $SERVICE_ARN \
  --cluster $CLUSTER_NAME \
  --task-set epoch-task-set \
  --scale value=50,unit=PERCENT
```


* Wait for the service tasks to be in a running state and marked healthy for service discovery.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1);
CMAP_SVC_ID=$(aws servicediscovery list-services | \
  jq -r '.Services[] | select(.Name == "crystal-green") | .Id');
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
until [ $(_list_instances) == "3" ]; do
  echo "Instances are registering ..."
  sleep 10s
  if [ $(_list_instances) == "3" ]; then
    echo "Instances registered"
    break
  fi
done
```
