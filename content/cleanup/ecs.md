---
title: "Cleanup the ECS cluster"
date: 2018-08-07T12:37:34-07:00
weight: 10
---

* Delete the services running in your cluster.

```bash
 # Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
# Delete ecs services #
aws ecs list-services \
  --cluster $CLUSTER_NAME | \
jq -r ' .serviceArns[] | [.] | @tsv ' | \
  while IFS=$'\t' read -r serviceArn; do 
    aws ecs delete-service \
      --cluster $CLUSTER_NAME \
      --service $serviceArn \
      --force
  done
```

* Stop all running tasks.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
# Stop ecs tasks #
aws ecs list-tasks \
  --cluster $CLUSTER_NAME | \
jq -r ' .taskArns[] | [.] | @tsv' | \
  while IFS=$'\t' read -r taskArn; do 
    aws ecs stop-task --cluster $CLUSTER_NAME --task $taskArn;
  done
```

* Delete the cluster.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
# Delete the ecs cluster #
aws ecs delete-cluster --cluster $CLUSTER_NAME
```

* De-register all task definitions.

```bash
# Define variables #
TASK_DEF_FAMILY=$(jq < cfn-output.json -r '.CrystalTaskDefinition' | \
  cut -d'/' -f2 | cut -d':' -f1)
# Delete ecs task definitions #
aws ecs list-task-definitions \
  --family-prefix $TASK_DEF_FAMILY | \
jq -r ' .taskDefinitionArns[] | [.] | @tsv' | \
  while IFS=$'\t' read -r taskDefinitionArn; do 
    aws ecs deregister-task-definition --task-definition $taskDefinitionArn;
  done
```