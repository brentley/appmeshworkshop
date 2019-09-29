---
title: "Add the envoy sidecar proxy"
date: 2018-09-18T17:40:09-05:00
weight: 20
---

Time to add the envoy sidecar proxy to our Crystal backend service. 
Since the service is runing in Fargate we will need to create a new revision of the Task Definition.

* Register a new task definition with the envoy sidecar proxy.

```bash
ENVOY_REGISTRY="111345817488.dkr.ecr.$AWS_REGION.amazonaws.com";
TASK_DEF_ARN=$(jq < cfn-output.json -r '.CrystalTaskDefinition');
TASK_DEF_OLD=$(aws ecs describe-task-definition --task-definition $TASK_DEF_ARN);
TASK_DEF_NEW=$(echo $TASK_DEF_OLD \
  | jq ' .taskDefinition' \
  | jq --arg ENVOY_REGISTRY $ENVOY_REGISTRY '.containerDefinitions += 
      [
        {
          "environment": [
            {
              "name": "APPMESH_VIRTUAL_NODE_NAME",
              "value": "mesh/AppMesh-Workshop/virtualNode/crystal-v1"
            }
          ],
          "image": ($ENVOY_REGISTRY + "/aws-appmesh-envoy:v1.11.1.1-prod"),
          "healthCheck": {
            "retries": 3,
            "command": [
              "CMD-SHELL",
              "curl -s http://localhost:9901/server_info | grep state | grep -q LIVE"
            ],
            "timeout": 2,
            "interval": 5,
            "startPeriod": 10
          },
          "essential": true,
          "user": "1337",
          "name": "envoy"
        }
      ]' \
  | jq ' .containerDefinitions[0] +=
      { 
        "dependsOn": [ 
          { 
            "containerName": "envoy",
            "condition": "HEALTHY" 
          }
        ] 
      }' \
  | jq ' . += 
      { 
        "proxyConfiguration": {
          "type": "APPMESH",
          "containerName": "envoy",
          "properties": [
            { "name": "IgnoredUID", "value": "1390"},
            { "name": "ProxyIngressPort", "value": "1500"},
            { "name": "ProxyEgressPort", "value": "1501"},
            { "name": "AppPorts", "value": "3000"},
            { "name": "EgressIgnoredIPs", "value": "169.254.170.2,169.254.169.254"}
          ]
        }
      }' \
  | jq ' del( .status, .compatibilities, .taskDefinitionArn, .requiresAttributes, .revision) '
); \
TASK_DEF_FAMILY=$(echo $TASK_DEF_ARN | cut -d"/" -f2 | cut -d":" -f1);
echo $TASK_DEF_NEW > /tmp/$TASK_DEF_FAMILY.json && 
aws ecs register-task-definition \
      --cli-input-json file:///tmp/$TASK_DEF_FAMILY.json
```