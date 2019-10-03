---
title: "Install the envoy proxy"
date: 2018-09-18T17:40:09-05:00
weight: 20
---

It is time to install the envoy proxy on our EC2 instances. We will use **Systems Manager** to configure the EC2 instances that serve the frontend microservice. SSM uses documents to specify configuration.

* Create a file named **configure-instance.yml** in your Cloud9 environment, and copy/paste the following content:

```yaml
---
schemaVersion: '2.2'
description: Configure instances
parameters: 
      region:
        type: String
      meshName:
        type: String
      vNodeName:
        type: String         
mainSteps:
- action: aws:configureDocker
      name: configureDocker
      inputs:
        action: Install
- action: aws:runShellScript
      name: runEnvoy
      inputs:
        runCommand: 
          - |
            #! /bin/bash
            docker run --detach \ 
              --env AWS_ACCESS_KEY_ID=$aws_access_key_id \
              --env AWS_SECRET_ACCESS_KEY=$aws_access_secret_key \
              --env AWS_SESSION_TOKEN=$aws_session_token \
              --env APPMESH_VIRTUAL_NODE_NAME=mesh/{{meshName}}/virtualNode/{{vNodeName}} \
              -u 1337 --network host \ 
              111345817488.dkr.ecr.{{region}}.amazonaws.com/aws-appmesh-envoy:v1.11.1.1-prod
```

* Call the SSM API to create the document
```bash
aws ssm create-document \
      --name AppMesh-Workshop-ConfigureInstance \
      --document-format YAML \
      --content file://configure-instance.yml \
      --document-type Command
```

* Create an association using State Manager. An association is a binding of the intent (described in the document) to a target specified by either a list of instance IDs or a tag query. Use the following command to create an association. Note that you are specifying a tag query in the target parameter and using the Frontend Auto Scaling group name. **Replace {serviceid} with the service id you wrote down previousy**.

```bash
AUTOSCALING_GROUP=$(jq < cfn-output.json -r '.RubyAutoScalingGroupName'); \
aws ssm create-association \
      --name AppMesh-Workshop-ConfigureInstance \
      --targets "Key=tag:aws:autoscaling:groupName,Values=$AUTOSCALING_GROUP" \
      --schedule-expression "cron(0 0/30 * 1/1 * ? *)" \
      --max-errors 0 \
      --max-concurrency 50% \
      --parameters "serviceid={serviceid}"
```