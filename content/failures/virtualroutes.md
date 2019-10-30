---
title: "Create traffic routes"
date: 2018-09-18T17:39:30-05:00
weight: 20
---

Since we know that the Crystal application will start replying requests with `503` error messages, we need to to add a retry policy in the route, so that if a request to crystal-sd-error resulted in any server error status codes as “500 or 503”, a retry attempt would be made with a max limit of 10 retries:

```bash
# Define variables #
SPEC=$(cat <<-EOF
  {
    "httpRoute": {
      "action": {
        "weightedTargets": [
          {
            "virtualNode": "crystal-sd-error",
            "weight": 1
          }
        ]
      },
      "match": {
        "prefix": "/",
        "headers": [
          {
            "name": "canary_fleet",
            "match": {
              "exact": "true"
            }
          }
        ]
      },
      "retryPolicy": {
        "maxRetries": 10,
        "perRetryTimeout": {
          "unit": "s",
          "value": 2
        },
        "httpRetryEvents": [
          "server-error"
        ]
      }        
    },
    "priority": 1
  }
EOF
); \
# Create app mesh route #
aws appmesh update-route \
  --mesh-name appmesh-workshop \
  --virtual-router-name crystal-router \
  --route-name crystal-header-route \
  --spec "$SPEC"
```

* Update your `crystal-random-route` to route to `crystal-sd-error` instead of `crystal-sd-epoch`. This will let you check the default traffic behavior (without the presence of a header).

```bash
# Define variables #
SPEC=$(cat <<-EOF
  { 
    "httpRoute": {
      "action": { 
        "weightedTargets": [
          {
            "virtualNode": "crystal-sd-vanilla",
            "weight": 2
          },
          {
            "virtualNode": "crystal-sd-error",
            "weight": 1
          }
        ]
      },
      "match": {
        "prefix": "/"
      }
    },
    "priority": 5
  }
EOF
); \
# Create app mesh route #
aws appmesh update-route \
  --mesh-name appmesh-workshop \
  --virtual-router-name crystal-router \
  --route-name crystal-random-route \
  --spec "$SPEC"
```

* We no longer need the `epoch-task-set` task set. Let's scale it to 0%.

```bash
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
SERVICE_ARN=$(aws ecs list-services --cluster $CLUSTER_NAME | \
  jq -r ' .serviceArns[] | select( . | contains("sd"))' | tail -1)
TASK_SET_ARN=$(aws ecs describe-services --cluster $CLUSTER_NAME --service $SERVICE_ARN | \
  jq -r ' .services | first | .taskSets[] | select( .externalId == "epoch-task-set") | .taskSetArn')
# Update ecs task set #
aws ecs update-task-set \
  --service $SERVICE_ARN \
  --cluster $CLUSTER_NAME \
  --task-set $TASK_SET_ARN \
  --scale value=0,unit=PERCENT
```