---
title: "Add the envoy sidecar proxy"
date: 2018-09-18T17:40:09-05:00
weight: 20
---

Time to add the envoy sidecar proxy to your Crystal backend service. 
Since the service is runing in Fargate you will need to create a new revision of the Task Definition.
The  task definition you are about to create will include a new container for the Envoy proxy that will run alongside (hence the term sidecar) the existing containerized Crystal micro service. 

At this point you may be wondering how is it possible for Envoy to actually intercept and process all the traffic that is sent to the Crystal container.

The ECS integration for AWS App Mesh leverages iptables provided by the Linux OS. Whenever you launch an ECS service based on a task definition that includes the Envoy proxy, it will apply a set of iptables rules such that all the ingress traffic targetted at the Crystal container port (3000 in our case) gets intercepted and sent instead to port 15000 where the Envoy Proxy listens for ingress traffic.
After processing its rules, the Envoy proxy establishes an HTTP connection to the app on port 3000 and forward the request. Once the Crystal app is done processing the request it send its response back to the Envoy process over the same HTTP connection. Finally the Envoy process takes the response sent by the app and replies to the client. 

* Register a new task definition with the envoy sidecar proxy.

```bash
# Define variables #
ENVOY_REGISTRY="840364872350.dkr.ecr.$AWS_REGION.amazonaws.com";
TASK_DEF_ARN=$(jq < cfn-output.json -r '.CrystalTaskDefinition');
TASK_DEF_OLD=$(aws ecs describe-task-definition --task-definition $TASK_DEF_ARN);
TASK_DEF_NEW=$(echo $TASK_DEF_OLD \
  | jq ' .taskDefinition' \
  | jq --arg ENVOY_REGISTRY $ENVOY_REGISTRY ' .containerDefinitions += 
        [
          {
            "environment": [
              {
                "name": "APPMESH_VIRTUAL_NODE_NAME",
                "value": "mesh/appmesh-workshop/virtualNode/crystal-lb-blue"
              }
            ],
            "image": ($ENVOY_REGISTRY + "/aws-appmesh-envoy:v1.11.2.0-prod"),
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
              { "name": "IgnoredUID", "value": "1337"},
              { "name": "ProxyIngressPort", "value": "15000"},
              { "name": "ProxyEgressPort", "value": "15001"},
              { "name": "AppPorts", "value": "3000"},
              { "name": "EgressIgnoredIPs", "value": "169.254.170.2,169.254.169.254"}
            ]
          }
        }' \
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
# Define variables #
CLUSTER_NAME=$(jq < cfn-output.json -r '.EcsClusterName');
TASK_DEF_ARN=$(aws ecs list-task-definitions | \
  jq -r ' .taskDefinitionArns[] | select( . | contains("crystal"))' | tail -1);
# Update ecs service #
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service crystal-service-lb-blue \
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
    --service crystal-service-lb-blue | \
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

Next, you will validate Envoy is actually proxing the requests in its way in and out of the crystal microservice. To do so, we will issue a curl command to the IP address of any of the 3 tasks running as part of the crystal service.

* Start a Session Manager session with any of the EC2 instances.

```bash
AUTO_SCALING_GROUP=$(jq < cfn-output.json -r '.RubyAutoScalingGroupName');
TARGET_EC2=$(aws ec2 describe-instances \
    --filters Name=tag:aws:autoscaling:groupName,Values=$AUTO_SCALING_GROUP | \
  jq -r ' .Reservations | first | .Instances | first | .InstanceId')
aws ssm start-session --target $TARGET_EC2
```

* Curl the crystal microservice.

```bash
TARGET_IP=$(dig +short crystal.appmeshworkshop.hosted.local | head -1)
curl -v $TARGET_IP:3000/crystal
```

* Terminate the session.

```bash
exit 
```

Notice the presence of the **server** header. This is in its own a confirmation that Envoy is proxing requests.