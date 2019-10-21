---
title: "Re-start the ECS tasks"
date: 2018-09-18T17:39:30-05:00
weight: 15
---

* Re-start the ECS tasks to update the container images.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
# Stop ECS tasks #
aws ecs list-tasks \
  --cluster $CLUSTER_NAME \
  --service crystal-service-sd-green | \
jq -r ' .taskArns[] | [.] | @tsv' | \
  while IFS=$'\t' read -r taskArn; do 
    aws ecs stop-task --cluster $CLUSTER_NAME --task $taskArn;
  done
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
        --service crystal-service-sd-green | \
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