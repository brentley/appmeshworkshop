---
title: "Enable X-Ray for crystal"
date: 2018-09-18T16:01:14-05:00
weight: 10
---

* Register a new task definition to add the x-ray container, and enable tracing for the envoy container.

```bash
# Define variables #
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
      jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1)
TASK_DEF_OLD=$(aws ecs describe-task-definition --task-definition $TASK_DEF_ARN);
TASK_DEF_NEW=$(echo $TASK_DEF_OLD \
      | jq ' .taskDefinition' \
      | jq ' .containerDefinitions |= map(
            if .name == "envoy" then .environment +=
              [
                {
                  "name": "ENABLE_ENVOY_XRAY_TRACING",
                  "value": "1"
                }
              ]
            else . end) ' \
      | jq ' .containerDefinitions += 
            [
              {
                "image": "amazon/aws-xray-daemon",
                "essential": true,
                "name": "xray",
                "portMappings": [
                  {
                    "hostPort": 2000,
                    "protocol": "udp",
                    "containerPort": 2000
                  }
                ]
              }
            ]' \
      | jq ' del(.status, .compatibilities, .taskDefinitionArn, .requiresAttributes, .revision) '
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
      --service crystal-service-sd-v1 \
      --task-definition "$(echo $TASK_DEF_ARN)"
```