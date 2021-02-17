---
title: "Enable CloudWatch for Crystal"
date: 2018-09-18T16:01:14-05:00
weight: 10
---

You will create a new task definition that adds logging to the Envoy container. Once you have enabled it, you will access CloudWatch Logs to consume the logs produced by the Envoy proxy. By default, Envoy will produce application and access logs intermingled in the same CloudWatch Log file. Make sure you configured your Virtual Node representing your application to send logs to /dev/stdout


* Register a new task definition to add logging to the Envoy container.

```bash
# Define variables #
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1)
TASK_DEF_OLD=$(aws ecs describe-task-definition --task-definition $TASK_DEF_ARN);
TASK_DEF_NEW=$(echo $TASK_DEF_OLD \
  | jq ' .taskDefinition' \
  | jq --arg AWS_REGION $AWS_REGION ' .containerDefinitions |= map(
        if .name == "envoy" then . +=
          {
            "logConfiguration": {
              "logDriver": "awslogs",
              "options": {
                "awslogs-create-group": "true",
                "awslogs-region": $AWS_REGION,
                "awslogs-group": "appmesh-workshop-crystal-envoy",
                "awslogs-stream-prefix": "fargate"
              }
            }
          }
        else . end) ' \
  | jq ' del(.status, .compatibilities, .taskDefinitionArn, .requiresAttributes, .revision, .registeredAt, .registeredBy) '
); \
TASK_DEF_FAMILY=$(echo $TASK_DEF_ARN | cut -d"/" -f2 | cut -d":" -f1);
echo $TASK_DEF_NEW > /tmp/$TASK_DEF_FAMILY.json && 
# Register ecs task definition #
aws ecs register-task-definition \
  --cli-input-json file:///tmp/$TASK_DEF_FAMILY.json
```

* Update the service.

```bash
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1)
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service crystal-service-lb \
  --task-definition "$(echo $TASK_DEF_ARN)"
```

* Wait for the service tasks to be in a running state.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1);
# Get task state #
_list_tasks() {
  aws ecs list-tasks \
    --cluster $CLUSTER_NAME \
    --service crystal-service-lb | \
  jq -r ' .taskArns | @text' | \
    while read taskArns; do 
      aws ecs describe-tasks --cluster $CLUSTER_NAME --tasks $taskArns;
    done | \
  jq -r --arg TASK_DEF_ARN $TASK_DEF_ARN \
    ' [.tasks[] | select( (.taskDefinitionArn == $TASK_DEF_ARN) 
                    and (.lastStatus == "RUNNING" ))] | length'
}
until [ $(_list_tasks) == "3" ]; do
  echo "Tasks are starting ..."
  sleep 10s
  if [ $(_list_tasks) == "3" ]; then
    echo "Tasks started"
    break
  fi
done
```
